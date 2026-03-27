# Architecture Diagram — Scalable Multi-Database Data Platform

## System Overview

```mermaid
flowchart TD
    %% ─── Input ───────────────────────────────────────────────
    A([👤 User Message\nuser_id · message · channel])

    %% ─── Pipeline ────────────────────────────────────────────
    subgraph PIPE["⚙️ ELT Pipeline  —  Python DAG / Airflow"]
        direction TB
        P1["📥 Ingest\nPydantic validation · Lineage tagging"]
        P2["🧠 Embed\nintfloat/e5-large-v2 · 1024-dim vector"]
        P3["📤 Store\nFan-out to 4 databases"]
        P1 --> P2 --> P3
    end

    A --> P1

    %% ─── Dead Letter Queue ───────────────────────────────────
    DLQ["🚫 Dead Letter Queue\nMongoDB — failed records"]
    P1 -- "invalid record" --> DLQ

    %% ─── Storage Layer ───────────────────────────────────────
    subgraph STORE["🗄️ Polyglot Storage Layer"]
        direction LR
        M["🍃 MongoDB\nRaw text + metadata\nSource of truth"]
        V["⚡ Milvus\n1024-dim embeddings\nIVF_FLAT index"]
        N["🕸️ Neo4j\nUser→Campaign→Intent\nGraph relationships"]
        S["📊 SQLite\nAggregated metrics\nEngagement counts"]
    end

    P3 -- "raw document" --> M
    P3 -- "1024-dim vector" --> V
    P3 -- "MERGE edges" --> N
    P3 -- "upsert counts" --> S

    %% ─── Batch Sync ──────────────────────────────────────────
    M -. "batch aggregation · every 5 mins" .-> S

    %% ─── Redis ───────────────────────────────────────────────
    R["⚡ Redis\nRecommendation cache · TTL 5 min"]
    P3 -- "🗑️ invalidate stale cache" --> R

    %% ─── Client ──────────────────────────────────────────────
    C([🌐 Client\nGET /recommendations/user_id])
    RESP(["📦 JSON Response\nranked campaigns · latency · cache_hit"])

    %% ─── API ─────────────────────────────────────────────────
    subgraph API["🚀 FastAPI — localhost:8000"]
        direction TB
        CHECK["🔍 Step 1 · Check Redis\nDoes cached result exist?"]
        STEP2["🔎 Step 2 · Milvus ANN search\nTop 5 similar users"]
        STEP3["🕸️ Step 3 · Neo4j Cypher\nCampaigns for those users"]
        STEP4["📊 Step 4 · SQLite scoring\nRank by engagement frequency"]
        STEP5["💾 Step 5 · Write to Redis\nCache result · TTL 5 min"]

        CHECK -- "MISS → go through full retrieval" --> STEP2
        STEP2 --> STEP3
        STEP3 --> STEP4
        STEP4 --> STEP5
    end

    C --> CHECK
    R -- "cached result" --> CHECK

    V -- "ANN query" --> STEP2
    N -- "Cypher query" --> STEP3
    S -- "engagement scores" --> STEP4
    STEP5 --> R

    %% ─── Both paths return to client ─────────────────────────
    CHECK -- "HIT → return immediately ~2ms" --> RESP
    STEP5 -- "return fresh result ~100ms" --> RESP

    %% ─── Observability ───────────────────────────────────────
    subgraph OBS["🔍 Observability"]
        direction LR
        O1["📝 Structured Logs\nConsole + logs/platform.log"]
        O2["⚠️ Anomaly Detection\nEmpty embeddings · DLQ spikes"]
        O3["📈 Streamlit Dashboard\nlocalhost:8501"]
    end

    PIPE --> O1
    API  --> O1
    O1 --> O2
    O1 --> O3

    %% ─── Streamlit dual role ─────────────────────────────────
    S -- "pipeline metrics\ncampaign stats" --> O3
    O3 -- "live recommendation demo" --> CHECK

    %% ─── Styles ──────────────────────────────────────────────
    classDef input    fill:#E1F5EE,stroke:#0F6E56,color:#085041
    classDef store    fill:#E6F1FB,stroke:#185FA5,color:#0C447C
    classDef cache    fill:#FAEEDA,stroke:#854F0B,color:#633806
    classDef api      fill:#EEEDFE,stroke:#534AB7,color:#3C3489
    classDef obs      fill:#F1EFE8,stroke:#5F5E5A,color:#444441
    classDef danger   fill:#FCEBEB,stroke:#A32D2D,color:#501313
    classDef response fill:#EAF3DE,stroke:#3B6D11,color:#173404

    class A,C input
    class M,V,N,S store
    class R cache
    class CHECK,STEP2,STEP3,STEP4,STEP5 api
    class O1,O2,O3 obs
    class DLQ danger
    class RESP response
```

---

## Data Flow Summary

### Real-Time Path (per message, P99 < 500 ms)

```
User message
  → Ingestion (validate + tag lineage)
  → Embedding (intfloat/e5-large-v2 — 1024-dim)
  → MongoDB  (store raw text + metadata)
  → Milvus   (upsert 1024-dim vector)
  → Neo4j    (upsert user–campaign–intent edges)
  → Redis    (cache session for active user)
```

### Batch Path (scheduled, every N minutes)

```
MongoDB change-stream / polling
  → Aggregate interaction counts per user/campaign
  → Upsert aggregated rows → SQLite / BigQuery
  → Refresh Redis TTL for top-active users
```

### Query Path (API, target P99 < 200 ms)

```
GET /recommendations/<user_id>
  → Redis  → cache hit?  → return immediately (~2ms)
  → Milvus → ANN top-5 similar user embeddings
  → Neo4j  → campaigns connected to those 5 users
  → SQLite → rank campaigns by engagement frequency
  → merge + return JSON response
  → write result to Redis (TTL = 5 min)
```

---

## Component Interaction Matrix

| From \ To       | MongoDB | Milvus | Neo4j | SQLite | Redis | FastAPI |
|----------------|---------|--------|-------|--------|-------|---------|
| **Pipeline**    | write   | write  | write | write  | write | —       |
| **Redis**       | read    | —      | —     | —      | —     | serve   |
| **FastAPI**     | —       | read   | read  | read   | r/w   | —       |
| **Observability** | read  | —      | —     | read   | read  | read    |

---

## Scaling & Fault-Tolerance

```mermaid
flowchart LR
    LB["Load Balancer\n(nginx / ALB)"]

    subgraph API_POOL["FastAPI Workers  (horizontal scale)"]
        W1["Worker 1"]
        W2["Worker 2"]
        W3["Worker N"]
    end

    subgraph DATA["Stateful Services  (vertical + sharding)"]
        MIL["Milvus Cluster\nHNSW index\nsharded collections"]
        NEO["Neo4j Cluster\nread replicas"]
        MON["MongoDB ReplicaSet\n3-node"]
        RD["Redis Cluster\nsentinel HA"]
        BQ["BigQuery\n(cloud, fully managed)"]
    end

    LB --> W1 & W2 & W3
    W1 & W2 & W3 --> MIL & NEO & MON & RD & BQ
```