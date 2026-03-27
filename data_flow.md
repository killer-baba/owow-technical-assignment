# Data Flow & Architecture Diagrams

Two reference diagrams showing exactly what happens to data as it moves through the platform — from a raw user message all the way to a ranked recommendation served by the API.

---

## Diagram 1 — Data Movement

What the data looks like, and how its structure changes, at every step.

```
╔══════════════════════════════════════════════════════════════╗
║  STEP 0 — Raw Input                                          ║
║  (what arrives from the outside world)                       ║
╠══════════════════════════════════════════════════════════════╣
║                                                              ║
║  {                                                           ║
║    "user_id":     "u001",                                    ║
║    "message":     "I'm looking for running shoes",           ║
║    "campaign_id": "camp_sports_q1",                          ║
║    "intent":      "purchase_intent",                         ║
║    "channel":     "web"                                      ║
║  }                                                           ║
║                                                              ║
╚══════════════════════════════════════════════════════════════╝
                          │
                          │  raw dict enters the pipeline
                          ▼
╔══════════════════════════════════════════════════════════════╗
║  STEP 1 — Validation & Lineage  (ingest.py)                  ║
║  Tool: Pydantic                                              ║
║  Job:  check data is clean, assign tracking IDs              ║
╠══════════════════════════════════════════════════════════════╣
║                                                              ║
║  CHECKS:                                                     ║
║    ✅ user_id has no spaces                                  ║
║    ✅ message is not blank                                   ║
║    ✅ campaign_id exists                                     ║
║                                                              ║
║  ADDS 2 NEW FIELDS:                                          ║
║    message_id: "abc-123"   ← unique ID for this message      ║
║    lineage_id: "xyz-456"   ← tracking ID across all stores   ║
║                                                              ║
║  OUTPUT:                                                     ║
║  {                                                           ║
║    "user_id":     "u001",                                    ║
║    "message":     "I'm looking for running shoes",           ║
║    "campaign_id": "camp_sports_q1",                          ║
║    "intent":      "purchase_intent",                         ║
║    "channel":     "web",                                     ║
║    "message_id":  "abc-123",          ← NEW                  ║
║    "lineage_id":  "xyz-456"           ← NEW                  ║
║  }                                                           ║
║                                                              ║
║  IF INVALID → goes to MongoDB DLQ, stops here                ║
║                                                              ║
╚══════════════════════════════════════════════════════════════╝
                          │
                          │  validated record moves forward
                          ▼
╔══════════════════════════════════════════════════════════════╗
║  STEP 2 — Embedding Generation  (embed.py)                   ║
║  Tool: intfloat/e5-large-v2 (Sentence Transformer model)     ║
║  Job:  convert message TEXT into NUMBERS                     ║
╠══════════════════════════════════════════════════════════════╣
║                                                              ║
║  INPUT (only the message text):                              ║
║    "I'm looking for running shoes"                           ║
║                                                              ║
║  PROCESS:                                                    ║
║    AI model reads the meaning of this sentence               ║
║    converts it into 1024 numbers                             ║
║                                                              ║
║  OUTPUT ADDED TO RECORD:                                     ║
║    "vector": [0.23, 0.87, 0.12, 0.56, 0.34, ...]             ║
║               └─────────── 1024 numbers total ───────────┘   ║
║                                                              ║
║  ANOMALY CHECK:                                              ║
║    if vector ≈ all zeros → flagged, excluded from Milvus     ║
║                                                              ║
╚══════════════════════════════════════════════════════════════╝
                          │
                          │  enriched record (text + vector)
                          ▼
╔══════════════════════════════════════════════════════════════╗
║  STEP 3 — Fan-out Storage  (store.py)                        ║
║  Job: write to 4 different databases simultaneously          ║
║  Each DB takes ONLY the fields it needs                      ║
╠══════════════════════════════════════════════════════════════╣
║                                                              ║
║  ┌─────────────────────────────────────────────────────┐     ║
║  │ Full enriched record                                │     ║
║  │ user_id, message, campaign_id, intent,              │     ║
║  │ channel, message_id, lineage_id, vector[1024]       │     ║
║  └──────┬──────────┬──────────┬───────────────┬────────┘     ║
║         │          │          │               │              ║
║         ▼          ▼          ▼               ▼              ║
║                                                              ║
║  MONGODB      MILVUS       NEO4J          SQLITE             ║
║  ─────────    ──────────   ────────────   ──────────────     ║
║  Takes:       Takes:       Takes:         Takes:             ║
║  ALL fields   vector       user_id        user_id            ║
║               user_id      campaign_id    campaign_id        ║
║               message_id   intent         channel            ║
║               lineage_id   lineage_id     timestamp          ║
║                                                              ║
║  Stores:      Stores:      Stores:        Stores:            ║
║  Raw JSON     1024 nums    Graph edges    Count rows         ║
║  document     per message  between        interaction        ║
║               in Milvus    nodes          frequency          ║
║                                                              ║
║  Why:         Why:         Why:           Why:               ║
║  Source of    Find         Find           Rank by            ║
║  truth,       similar      campaigns      popularity         ║
║  reprocess    users        for those                         ║
║  later                     users                             ║
║                                                              ║
╚══════════════════════════════════════════════════════════════╝
                          │
                          │  also writes to Redis
                          ▼
╔══════════════════════════════════════════════════════════════╗
║  STEP 4 — Cache Invalidation  (Redis)                        ║
║  Job: delete old cached recommendations for these users      ║
╠══════════════════════════════════════════════════════════════╣
║                                                              ║
║  Redis key deleted:  "recommendations:u001"                  ║
║                                                              ║
║  Why: u001 just sent a new message, their old cached         ║
║  recommendations are now stale. Force fresh lookup           ║
║  on their next API call.                                     ║
║                                                              ║
╚══════════════════════════════════════════════════════════════╝
                          │
                          │  pipeline done. Now API time.
                          ▼
╔══════════════════════════════════════════════════════════════╗
║  STEP 5 — API Request  (main.py)                             ║
║  Trigger: GET /recommendations/u001                          ║
╠══════════════════════════════════════════════════════════════╣
║                                                              ║
║  5a. Check Redis first                                       ║
║      key "recommendations:u001" exists?                      ║
║        YES → return immediately (1-2ms). DONE.               ║
║        NO  → continue below                                  ║
║                                                              ║
║  5b. Embed the user_id as a query                            ║
║      "user profile for u001" → [0.21, 0.85, ...] 1024 nums   ║
║                                                              ║
║  5c. Milvus ANN Search                                       ║
║      Input:  query vector [0.21, 0.85, ...]                  ║
║      Output: ["u019", "u013", "u004", "u002", "u010"]        ║
║              (5 users with most similar vectors)             ║
║                                                              ║
║  5d. Neo4j Graph Query                                       ║
║      Input:  ["u019", "u013", "u004", "u002", "u010"]        ║
║      Query:  what campaigns did these users engage with?     ║
║      Output: [                                               ║
║                {camp_tech_edu,  weight: 4},                  ║
║                {camp_sports_q1, weight: 2},                  ║
║                {camp_health_q1, weight: 1}                   ║
║              ]                                               ║
║                                                              ║
║  5e. SQLite Scoring                                          ║
║      Input:  [camp_tech_edu, camp_sports_q1, camp_health_q1] ║
║      Query:  total interactions for each campaign?           ║
║      Output: {camp_tech_edu: 4, camp_sports_q1: 3, ...}      ║
║                                                              ║
║  5f. Merge + Rank                                            ║
║      score = engagement_score + graph_weight                 ║
║      sort descending                                         ║
║      Output: ranked list of campaigns                        ║
║                                                              ║
║  5g. Write to Redis                                          ║
║      "recommendations:u001" → ranked list (TTL 5 mins)       ║
║                                                              ║
║  5h. Return JSON response                                    ║
║                                                              ║
╚══════════════════════════════════════════════════════════════╝
```

---

## Diagram 2 — System Architecture

All tools, layers, and how they connect to each other.

```
╔══════════════════════════════════════════════════════════════════╗
║                    INFRASTRUCTURE LAYER                          ║
║                    (Docker Compose)                              ║
║                                                                  ║
║   ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌───────────────┐    ║
║   │ MongoDB  │  │  Redis   │  │  Neo4j   │  │    Milvus     │    ║
║   │  :27017  │  │  :6379   │  │  :7687   │  │    :19530     │    ║
║   └──────────┘  └──────────┘  └──────────┘  └───────┬───────┘    ║
║                                                      │           ║
║                                              ┌───────┴───────┐   ║
║                                              │  etcd  MinIO  │   ║
║                                              │ (Milvus deps) │   ║
║                                              └───────────────┘   ║
╚══════════════════════════════════════════════════════════════════╝
         ▲              ▲              ▲              ▲
         │              │              │              │
         └──────────────┴──────────────┴──────────────┘
                              reads/writes
╔═════════════════════════════════════════════════════════════════╗
║                   APPLICATION LAYER                             ║
║                   (your local Python)                           ║
║                                                                 ║
║  ┌─────────────────────────────────────────────────────────┐    ║
║  │  PIPELINE  (run_pipeline.py)                            │    ║
║  │                                                         │    ║
║  │  ingest.py → embed.py → store.py                        │    ║
║  │  [validate]  [1024-dim]  [fan-out to 4 DBs]             │    ║
║  │                                                         │    ║
║  │  Orchestrated by: Python DAG / Airflow (dags/)          │    ║
║  └─────────────────────────────────────────────────────────┘    ║
║                                                                 ║
║  ┌─────────────────────────────────────────────────────────┐    ║
║  │  API  (main.py — FastAPI on port 8000)                  │    ║
║  │                                                         │    ║
║  │  GET /recommendations/{user_id}                         │    ║
║  │                                                         │    ║
║  │  Redis → Milvus → Neo4j → SQLite → Redis → Response     │    ║
║  └─────────────────────────────────────────────────────────┘    ║
║                                                                 ║
║  ┌─────────────────────────────────────────────────────────┐    ║
║  │  DASHBOARD  (dashboard.py — Streamlit on port 8501)     │    ║
║  │                                                         │    ║
║  │  Pipeline run history  (reads SQLite)                   │    ║
║  │  Top campaigns         (reads SQLite)                   │    ║
║  │  Live recommendations  (calls FastAPI)                  │    ║
║  └─────────────────────────────────────────────────────────┘    ║
╚═════════════════════════════════════════════════════════════════╝
```