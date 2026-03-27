# Architecture — Design Decisions & Trade-offs

## 1. System Goals

This platform must satisfy three competing demands simultaneously:

- **Low-latency serving** — the recommendation API must respond in under 200 ms for active users.
- **High-throughput ingestion** — thousands of concurrent conversations must be processed without blocking.
- **Analytical depth** — campaign performance and engagement patterns must be queryable across millions of historical records.

No single database technology satisfies all three goals equally well. This is why the architecture is **polyglot** — each store is chosen for what it does best, and the pipeline stitches them together.

---

## 2. Database Selection Rationale

### MongoDB — Document Store (Raw Conversation Data)

**Why:** Conversation data is naturally semi-structured. Messages vary in length, metadata fields may differ between channels, and the schema will evolve as new integrations are added. A rigid relational schema would require frequent migrations. MongoDB's flexible document model allows new fields without downtime.

**What it stores:** `{ user_id, session_id, message, channel, timestamp, lineage_id }` per conversation turn.

**Trade-off:** MongoDB is not ideal for complex multi-table joins or aggregations. We deliberately offload those to SQLite/BigQuery. MongoDB here is purely a write-optimised raw store and source-of-truth for unprocessed data.

**Alternative considered:** PostgreSQL with JSONB — ruled out because operational overhead (schema migrations, connection pooling) outweighs the benefits at prototype scale, and we're not using any relational join patterns in the hot path.

---

### Milvus — Vector Database (Embeddings)

**Why:** Vector similarity search is the core of the recommendation engine. We need Approximate Nearest Neighbour (ANN) search across millions of 1024-dimensional embeddings with sub-100 ms P99 latency. Milvus is purpose-built for this: it supports IVF_FLAT and HNSW indices, has native metadata filtering, and scales horizontally.

**What it stores:** One vector per message `(user_id, message_id, vector[1024])` with user-level aggregated embeddings computed periodically.

**Embedding model:** `intfloat/e5-large-v2` — chosen specifically because it outputs exactly **1024 dimensions** as required, runs fully on CPU with no GPU needed, and consistently ranks among the top-performing open-source embedding models on the MTEB benchmark. Models like `all-mpnet-base-v2` only output 768 dimensions and were therefore ruled out.

**Index chosen:** `IVF_FLAT` at prototype scale (exact recall, reasonable speed). At 10M+ users, switch to `HNSW` (graph-based ANN, sub-10 ms at scale).

**Trade-off:** Milvus requires more operational overhead than FAISS (which is a library, not a server). However, Milvus provides persistence, metadata filtering, and a query API that FAISS cannot, making it the right choice for production.

**Alternative considered:** Pinecone (managed) — ruled out to avoid cloud vendor lock-in in the prototype; the architecture can migrate to Pinecone by swapping the Milvus client.

---

### Neo4j — Graph Database (Relationship Mapping)

**Why:** The connections between users, campaigns, and intents form a graph — not a table. A user may belong to multiple campaigns; a campaign targets multiple intents; intents cluster by user behaviour. Graph traversal in Neo4j (`MATCH (u)-[:ENGAGED_WITH]->(c)`) is orders of magnitude faster than equivalent SQL joins with junction tables, especially at 2–3 hops.

**What it stores:** Nodes for `User`, `Campaign`, `Intent`. Edges for `ENGAGED_WITH`, `TARGETS`, `TRIGGERED`. Each edge carries weight (engagement count) and timestamp.

**Trade-off:** Neo4j's Cypher query language has a learning curve. Write throughput is lower than MongoDB. For this reason, we write to Neo4j asynchronously (upsert edge weights in batches) rather than in the hot ingestion path.

**Alternative considered:** Amazon Neptune — omitted to keep the architecture cloud-agnostic and locally runnable.

---

### SQLite / BigQuery — Analytics Layer

**Why:** Aggregated metrics (total interactions per user, campaign CTR, engagement frequency) are read-heavy, query-heavy, and best served by a columnar or relational engine. SQLite is the local prototype stand-in; BigQuery is the production target. Both expose standard SQL.

**What it stores:** `user_interaction_summary(user_id, campaign_id, interaction_count, last_seen, channel)` — pre-aggregated, updated by the batch pipeline.

**Trade-off:** SQLite has no horizontal scalability and no concurrent write safety beyond WAL mode. This is acceptable at prototype scale. The migration path to BigQuery is a one-line client swap — the schema and query patterns are identical.

**Alternative considered:** DuckDB — an excellent local analytical engine, but BigQuery compatibility was a stated requirement, so SQLite-as-BigQuery-mock was the more honest approximation.

---

### Redis — Cache & Session Store

**Why:** The recommendation endpoint would otherwise make 3 database round-trips on every request (Milvus → Neo4j → SQLite). For active users (queried repeatedly within a session), caching the result set in Redis reduces P99 latency from ~200 ms to ~2 ms.

**What it stores:** `recommendations:{user_id}` → JSON-serialised ranked list of campaigns. TTL of 5 minutes. Also stores `session:{user_id}` → recent context window for the conversation.

**Trade-off:** Cached recommendations can be stale by up to TTL seconds. This is acceptable for marketing personalisation (recommendations don't need millisecond freshness). For higher-stakes use cases, reduce TTL or implement cache invalidation on new interactions.

**Alternative considered:** Memcached — ruled out because Redis also handles pub/sub (useful for future real-time event streaming) and supports richer data types (sorted sets for leaderboards).

---

## 3. Pipeline Design

### Orchestration Strategy

The pipeline is implemented as a **Python DAG** with clearly separated task functions. At prototype scale this is simpler to run and debug than Airflow. The structure is designed so that each task function can be lifted into an Airflow `PythonOperator` or a Prefect `task` with zero modification — only the scheduler wrapper changes.

### Schema Validation

Every ingested record is validated against a Pydantic model before any downstream write. Invalid records are routed to a dead-letter queue (a separate MongoDB collection) with the original payload and the validation error. This ensures the pipeline never silently drops data.

### Data Lineage

Every record carries a `lineage_id` (UUID) generated at ingestion time. This ID is propagated to all four stores:
- MongoDB document: `lineage_id` field
- Milvus vector: stored as metadata
- Neo4j edge: property on the relationship
- SQLite row: `lineage_id` column

This makes it possible to trace any recommendation back to its source message without joining across stores at query time.

---

## 4. API Design

### Hybrid Retrieval

The `/recommendations/<user_id>` endpoint performs a **three-stage retrieval** that deliberately separates concerns:

1. **Milvus** answers "who is similar to this user?" (vector space proximity)
2. **Neo4j** answers "what campaigns have those similar users engaged with?" (graph traversal)
3. **SQLite** answers "which of those campaigns performs best?" (frequency ranking)

This separation means each component can be tuned or replaced independently. Adding a new ranking signal (e.g., recency decay) only touches stage 3.

### Caching Strategy

Responses are cached in Redis with a `user_id`-keyed key and a 5-minute TTL. On cache miss, the full three-stage retrieval runs and the result is written back to Redis. On cache hit, the result is returned directly with no database contact.

---

## 5. Observability Design

### What is logged

Every pipeline run emits:
- Task start/end timestamps and duration (latency)
- Record count processed per task
- Error type and count if any task fails
- A run-level summary with overall status

Every API request emits:
- `user_id`, cache hit/miss, total response time, and per-stage breakdown (Milvus ms, Neo4j ms, SQLite ms)

A **Streamlit dashboard** (`dashboard.py`) surfaces all of the above visually — pipeline run history, top campaigns by engagement, and a live recommendation demo where any user_id can be queried interactively.

### Anomaly Detection

The pipeline checks for:
- **Empty embeddings** — a vector of all zeros (or below a norm threshold) indicates the Sentence-Transformer failed silently.
- **Missing relationships** — if a user has MongoDB records but no Neo4j node, referential integrity is broken.
- **Throughput drop** — if records processed per minute drops below a configured threshold, an alert is logged (extensible to PagerDuty/Slack webhook).

---

## 6. Real-Time vs Batch Flows

| Dimension         | Real-Time (per message)             | Batch (scheduled)                  |
|-------------------|-------------------------------------|------------------------------------|
| **Trigger**       | New message ingested                | Cron / scheduler (e.g., 5 min)    |
| **Stores written**| MongoDB, Milvus, Neo4j, Redis       | SQLite / BigQuery                  |
| **Target latency**| < 500 ms end-to-end                 | Throughput-optimised, no SLA       |
| **Failure mode**  | Dead-letter queue, retry 3×         | Idempotent upsert, re-run safe     |
| **Scaling**       | Horizontal (more pipeline workers)  | Parallelise by user_id partition   |

The split exists because SQLite/BigQuery aggregations are expensive to compute per-message. Pre-aggregating on a schedule decouples write latency from analytical complexity.

---

## 7. Docker Compose Topology

All services run in a single `docker-compose.yml` for local development:

| Service        | Image                          | Port  | Purpose                        |
|---------------|-------------------------------|-------|-------------------------------|
| `mongodb`      | `mongo:7`                      | 27017 | Document store                 |
| `redis`        | `redis:7-alpine`               | 6379  | Cache                          |
| `milvus`       | `milvusdb/milvus:v2.4.0`       | 19530 | Vector store                   |
| `etcd`         | `quay.io/coreos/etcd:v3.5`     | 2379  | Milvus metadata (dependency)   |
| `minio`        | `minio/minio`                  | 9000  | Milvus object storage (dep.)   |
| `neo4j`        | `neo4j:5`                      | 7474/7687 | Graph database             |
| `pipeline`     | Custom (Python 3.11)           | —     | ETL worker                     |
| `api`          | Custom (Python 3.11)           | 8000  | FastAPI server                 |

`pipeline` and `api` depend on all storage services via `depends_on` with `condition: service_healthy`.

---

## 8. Key Trade-offs Summary

| Decision                              | What we gain                         | What we give up                        |
|--------------------------------------|--------------------------------------|----------------------------------------|
| Polyglot storage (5 databases)        | Each DB optimised for its workload   | Operational complexity, more services  |
| Async Neo4j writes                   | Lower ingestion latency              | Slight lag before graph reflects reality |
| SQLite as BigQuery mock              | Zero cloud cost in development       | Not identical behaviour at scale       |
| Redis TTL caching                    | Near-zero API latency on cache hit   | Up to 5 min stale recommendations      |
| Airflow DAG (production-ready)       | Retry, scheduling, monitoring UI     | Heavier infra, needs its own DB        |
| IVF_FLAT Milvus index                | Exact recall, good for small corpus  | Slower than HNSW at 10M+ vectors       |
| Pydantic validation at ingestion     | Data quality guaranteed downstream   | Small CPU overhead per record          |