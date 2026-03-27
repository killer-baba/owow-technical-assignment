# Scaling Plan

How to evolve this prototype from a single-machine Docker Compose setup
to a cloud-native platform serving 10M+ users with sub-100ms vector queries
and cost-efficient infrastructure.

---

## Current State (Prototype)

| Dimension | Current |
|---|---|
| Users | ~100s in sample data |
| Vector index | Milvus standalone, IVF_FLAT, 1024-dim |
| Graph DB | Neo4j single node |
| MongoDB | Single node, no replica set |
| Redis | Single node |
| Analytics | SQLite file |
| Pipeline | Python DAG, single process |
| API | Single Uvicorn worker |
| Infra | Docker Compose, one machine |

---

## Target State (Production)

| Dimension | Target |
|---|---|
| Users | 10M+ |
| Vector queries | P99 < 100ms |
| Ingestion throughput | 10,000+ messages/second |
| API availability | 99.9% uptime |
| Cost model | Cloud-native, autoscaling |

---

## Phase 1 — Harden the Data Layer

### MongoDB → Replica Set + Sharding

At 10M users with multiple messages each, the conversations collection will
exceed 100M documents. A single MongoDB node becomes a write bottleneck and
a single point of failure.

**Changes:**
- Deploy a 3-node replica set (1 primary + 2 secondaries) for high availability
- Enable write concern `majority` to prevent data loss on failover
- Shard the `conversations` collection on `user_id` hash — this distributes
  writes evenly across shards as the user base grows
- Route analytical reads to secondaries to keep the primary free for writes

**Infrastructure:** MongoDB Atlas (managed) or a self-hosted replica set on
Kubernetes. Atlas is preferred for automatic failover and point-in-time recovery.

---

### Redis → Redis Cluster with Sentinel

A single Redis node failing brings cache hit rates to zero and puts full
retrieval load on all downstream databases simultaneously.

**Changes:**
- Deploy Redis Cluster with 3 primary shards, each with 1 replica
- Use Redis Sentinel for automatic failover
- Partition keys by consistent hashing (built into Redis Cluster)
- Increase TTL to 15 minutes for lower-churn users; reduce to 60s for
  high-activity users (implement adaptive TTL based on session activity)

**Infrastructure:** AWS ElastiCache for Redis (managed, multi-AZ) or
self-hosted Redis Cluster on Kubernetes.

---

### SQLite → BigQuery

SQLite is a single-file database with no concurrent write safety beyond WAL mode.
At 10M users generating millions of interaction events per day, SQLite is the
first bottleneck to hit.

**Changes:**
- Replace SQLite with BigQuery (or Redshift / ClickHouse for self-hosted)
- Migrate the `user_interaction_summary` and `pipeline_runs` tables as-is —
  the schema is already compatible with standard SQL
- Stream interaction events into BigQuery via the BigQuery Storage Write API
  for real-time ingestion (replaces the SQLite `upsert_interaction` call)
- Use BigQuery partitioning on `last_seen` date for cost-efficient queries
- Materialized views for the engagement scoring query used by the API

**Code change:** swap `sqlite_client.py` for a `bigquery_client.py` with the
same interface — the API and pipeline code are unchanged.

---

## Phase 2 — Scale the Vector Layer

This is the most technically demanding scaling challenge. Milvus standalone
with IVF_FLAT handles ~1M vectors comfortably. At 10M users with multiple
messages each, you may have 50–100M vectors.

### Switch Index: IVF_FLAT → HNSW

| Index | Recall | Query latency | Build time |
|---|---|---|---|
| IVF_FLAT (current) | 100% | ~50ms at 1M | Fast |
| HNSW (target) | 98%+ | ~5ms at 100M | Slower |

HNSW (Hierarchical Navigable Small World) is a graph-based ANN algorithm
that maintains sub-10ms query latency even at 100M+ vectors, at the cost
of ~98% recall instead of 100%. For recommendation use cases, 98% recall
is more than sufficient.

**Migration:** create a new Milvus collection with HNSW index, backfill from
the existing collection, then hot-swap the collection name in config.

### Milvus Distributed Mode

Milvus standalone runs all components in one process. Milvus distributed
separates them:

- **Proxy** — receives queries, routes to query nodes
- **Query nodes** — hold segments in memory, execute ANN search
- **Data nodes** — handle inserts, compaction
- **Index nodes** — build indices in background

This allows each component to scale independently. At 10M users, deploy
4–8 query nodes to parallelise search across shards of the vector space.

**Infrastructure:** Milvus on Kubernetes with the Milvus Operator.
Query nodes are the most memory-intensive component — use memory-optimised
instances (e.g., AWS `r6i.4xlarge`, 128 GB RAM).

### User-Level Aggregated Embeddings

Currently, every message generates a separate vector. At 10M users with
10 messages each, that is 100M vectors. ANN search over 100M vectors is
slower and more expensive than necessary.

**Optimisation:** compute a per-user aggregated embedding (mean-pool of all
their message vectors) and store that as the primary search vector. Retain
per-message vectors in a separate collection for audit and lineage.

This reduces the searchable collection from 100M to 10M vectors — a 10x
reduction in search cost with no loss in recommendation quality.

---

## Phase 3 — Scale the Graph Layer

Neo4j single-node can handle ~1B relationships before write throughput degrades.
At 10M users × 10 campaigns average = 100M `ENGAGED_WITH` edges — comfortably
within single-node range.

**Short term (up to 500M edges):**
- Add read replicas for the API's graph traversal queries (Cypher reads)
- Keep the primary for pipeline writes only
- Add Neo4j indexes on `:User(user_id)` and `:Campaign(campaign_id)` if not
  already present (the constraints we create already do this)

**Long term (500M+ edges):**
- Migrate to Neo4j AuraDB Enterprise (managed, horizontally sharded)
- Or evaluate Apache AGE (PostgreSQL extension) for cost efficiency if
  graph traversal depth stays at 2 hops

---

## Phase 4 — Scale the Ingestion Pipeline

The current Python DAG is single-process and synchronous. At 10,000+
messages/second, a queue-backed async pipeline is required.

### Add a Message Queue

```
User messages
    │
    ▼
Kafka / Pub-Sub topic: raw_conversations
    │
    ├── Pipeline consumer group 1: validate + embed + write MongoDB + Milvus
    └── Pipeline consumer group 2: write Neo4j + SQLite (batch)
```

**Why Kafka:** decouples ingestion rate from processing rate. If the embedding
service is slow (CPU-bound), the queue absorbs the backlog without dropping
messages. Kafka's consumer groups allow independent scaling of each processing
stage.

**Embedding throughput:** a single CPU process can embed ~50 messages/second
with `intfloat/e5-large-v2`. At 10,000 messages/second, deploy 200 embedding
workers — or switch to a GPU inference server (TGI, vLLM) that batches
requests and achieves 2,000+ embeddings/second on a single A10G GPU.

### Switch to Airflow for Orchestration

The Python DAG structure already mirrors Airflow's task model. Migration steps:

1. Wrap each stage function in an Airflow `PythonOperator`
2. Define task dependencies with `>>` operators
3. Deploy Airflow on Kubernetes with the Celery or Kubernetes executor
4. Use Airflow's built-in retry, alerting, and DAG visualisation

No pipeline logic changes — only the scheduler wrapper changes.

---

## Phase 5 — Scale the API Layer

### Horizontal API Scaling

```
Internet
    │
    ▼
AWS ALB / nginx (load balancer)
    │
    ├── FastAPI worker 1  (Uvicorn + Gunicorn)
    ├── FastAPI worker 2
    ├── FastAPI worker 3
    └── FastAPI worker N  (autoscale on CPU/RPS)
```

Deploy the FastAPI app on Kubernetes. Use a Horizontal Pod Autoscaler (HPA)
that scales worker count based on requests-per-second. At 10M users with
typical session patterns, 10–20 workers handle steady-state traffic; scale
to 50+ during peak hours.

### Async FastAPI Workers

The current implementation uses synchronous DB clients. At high concurrency,
each request blocks a thread waiting for Milvus/Neo4j responses.

**Change:** migrate to async clients:
- `motor` (async MongoDB driver) instead of `pymongo`
- `neo4j`'s async session API
- `asyncio`-compatible Redis client (`aioredis`)

This allows a single worker to handle hundreds of concurrent in-flight
requests without spawning hundreds of threads.

---

## Phase 6 — Cost Efficiency

| Component | Cost lever |
|---|---|
| Milvus query nodes | Right-size to memory needed for working set; use spot instances for non-primary replicas |
| BigQuery | Partition by date; cluster by `user_id`; use materialized views to avoid full table scans |
| Embedding inference | GPU spot instances for batch embedding; reserved instances for real-time API path |
| MongoDB Atlas | Use tiered storage — hot data on NVMe SSDs, cold data (>90 days) on object storage via Atlas Online Archive |
| Redis | Set appropriate TTLs; evict with `allkeys-lru` policy to keep memory bounded |
| Neo4j | Archive edges older than 12 months to a cold analytical store (e.g., BigQuery) |

---

## Sub-100ms Vector Query — Guarantee Path

At 10M users with HNSW index on Milvus distributed:

| Step | Target latency |
|---|---|
| Redis cache hit | < 5ms |
| Embed query text (GPU) | < 10ms |
| Milvus HNSW ANN search | < 20ms |
| Neo4j 2-hop traversal | < 30ms |
| SQLite / BigQuery score lookup | < 10ms |
| Serialise + network | < 5ms |
| **Total (cache miss, P99)** | **< 80ms** |

The Redis cache hit path (covering 80%+ of repeat users) delivers results in under 5ms, 
well within any real-time personalisation SLA.

---

## Migration Roadmap Summary

| Phase | Change | Impact |
|---|---|---|
| 1a | MongoDB replica set | HA, no data loss on failover |
| 1b | Redis Cluster | Cache HA, no cold-start spikes |
| 1c | SQLite → BigQuery | Analytical scale, SQL compatibility |
| 2a | IVF_FLAT → HNSW index | 10× faster ANN at 10M+ vectors |
| 2b | Milvus distributed | Horizontal vector search scale |
| 2c | User-level aggregated embeddings | 10× vector count reduction |
| 3 | Neo4j read replicas → AuraDB | Graph query scale |
| 4a | Kafka message queue | Decouple ingestion from processing |
| 4b | GPU embedding server | 40× embedding throughput |
| 4c | Airflow orchestration | Retry, monitoring, scheduling |
| 5a | Kubernetes + HPA | API horizontal scale |
| 5b | Async FastAPI clients | High concurrency per worker |
| 6 | Cost optimisation | Spot instances, tiered storage, TTL tuning |