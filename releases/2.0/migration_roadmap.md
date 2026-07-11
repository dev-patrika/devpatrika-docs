# 🗺️ Dev Patrika — Migration Roadmap
### SQLite + ChromaDB + LangChain → Neon (Postgres + pgvector) + LangGraph

> **Goal**: Consolidate storage onto Neon (Postgres + pgvector), refactor the pipeline into LangGraph for traceability via LangSmith, without breaking the RAG chatbot.

---

## 🎯 Guiding Principle: Order Matters

Do the **data layer first**, then the **orchestration layer**. Reasoning:
- LangGraph's checkpointing (`PostgresSaver`) needs Postgres to already exist — no point setting it up against SQLite.
- Migrating vector store and relational DB together (both move to Neon) avoids doing double work.
- Once the foundation is stable, refactoring pipeline logic into graphs is lower-risk because you're not simultaneously debugging DB connection issues.

---

## Phase 0 — Prep & Safety Net (0.5 day)

| Task | Detail |
|---|---|
| Backup current SQLite DB | Copy `dev_patrika.db` somewhere safe |
| Backup ChromaDB folder | Copy `chroma_db/` — you'll need old embeddings for migration/validation |
| Freeze schema | Document current 7 SQLModel models as-is (you'll reuse these almost 1:1 on Postgres) |
| Create Neon project | Set up project, note pooled + direct connection strings |
| Enable `pgvector` extension | `CREATE EXTENSION IF NOT EXISTS vector;` on the Neon DB |

**Risk**: Low. Nothing in production yet, this is pure prep.

---

## Phase 1 — Neon (Postgres) Migration (1–2 days)

| Task | Detail |
|---|---|
| Swap DB driver | `sqlite` → `postgresql+asyncpg` (if async) or `psycopg2`/`psycopg` (if sync) in `database.py` |
| Update connection string | Use Neon's **pooled** connection string (pgbouncer) for the app; use **direct** connection only for migrations/DDL |
| Fix dialect-specific fields | `related_links` (currently JSON-as-string) → native **JSONB** column |
| Run `SQLModel.metadata.create_all()` | Against Neon — recreates all 7 tables |
| Data migration script | Simple Python script: read all rows from SQLite via SQLModel → bulk insert into Neon via same models. ~1.4MB db, this will be fast |
| Update `config.py` | New `DATABASE_URL` env var, remove SQLite-specific settings |
| Smoke test all 7 routers | Hit every existing endpoint against Neon, confirm reads/writes work |

**Risk**: Medium — mostly around connection pooling under async load. Test with a few concurrent requests before moving on.

**Don't touch yet**: LangChain pipeline code, ChromaDB. Keep those pointing to old Chroma folder for now — isolate variables.

---

## Phase 2 — Vector Store: ChromaDB → pgvector (1–2 days)

| Task | Detail |
|---|---|
| Add vector column | New table (or extend existing) with `vector(768)` or matching your `gemini-embedding-2` dimension |
| Install `langchain-postgres` | Swap `Chroma` import for `PGVector` |
| Re-embed strategy | **Don't try to migrate raw vectors** — just re-run your existing embedding pipeline against Neon. Since you already have the source text (wiki entries + news items) in SQLite/Neon, regenerating embeddings is simpler and safer than migrating binary vector data |
| Update `chroma_service.py` → rename to `vector_service.py` | Swap initialization; retrieval methods (`similarity_search`, `top_k`) keep nearly identical signatures |
| Update RAG chat service | Point semantic search calls (Wiki top-3, News top-3) to new `vector_service.py` — **logic unchanged**, only the client object changes |
| Validate retrieval quality | Run 5–10 sample chat queries, compare citations/relevance against old Chroma results before decommissioning Chroma |
| Decommission `chroma_db/` folder | Only after validation passes |

**Risk**: Medium — this is the one place where output quality (RAG answer relevance) could silently regress. Don't skip validation.

**Why this doesn't break your chatbot**: The RAG flow (`load history → semantic search → build citation map → stream LLM response`) stays structurally identical. Only the vector store backend changes underneath.

---

## Phase 3 — LangGraph Refactor (staged, 3–5 days total)

Do this **incrementally**, one subsystem at a time — don't rewrite everything at once.

### 3a. Processing Pipeline first (highest value, most isolated)
- Convert `pipeline.py` (LLM Router → News Analyzer → GitHub Analyzer) into a graph
- Nodes: `route_llm`, `analyze_news`, `analyze_github`, `save_to_db`
- Conditional edge: Groq fails → route to Gemini fallback (replaces try/except glue)
- State: shared `PipelineState` (TypedDict) carrying raw items → processed items

### 3b. Wiki Curator second
- Nodes: `extract_terms`, `check_exists` (conditional), `generate_definition`, `save_and_index`
- Your existing flowchart in the report *already* maps to this almost directly — minimal redesign needed

### 3c. Ingestion Orchestrator third
- Nodes per source: `fetch_hn`, `fetch_devto`, `fetch_arxiv`, `fetch_github` (can run as parallel branches in LangGraph)
- Then: `dedup_node` → `categorize_node` → `persist_node`

### 3d. Trending + Weekly Report last
- These are simpler, mostly single-pass — lowest priority to convert, can even stay as-is if time-constrained

### 3e. Add PostgresSaver checkpointing
- Once everything is on Neon, wire up `PostgresSaver` for graph state persistence
- Now a failed mid-cycle run (e.g. GitHub scraper timeout) can resume instead of restarting the whole 6-hour cycle from scratch

**Risk**: Medium-high on time, low on data risk (this is pure code refactor, not touching schema).

---

## Phase 4 — LangSmith Tracing (0.5–1 day)

| Task | Detail |
|---|---|
| Set env vars | `LANGCHAIN_TRACING_V2=true`, `LANGCHAIN_API_KEY`, `LANGCHAIN_PROJECT=dev-patrika` |
| Verify graph traces | Run one ingestion cycle, confirm node-by-node trace shows up in LangSmith UI |
| Add tags/metadata | Tag traces by cycle type (`ingestion`, `wiki_curation`, `chat`) for easier filtering later |

**This is the payoff step** — once here, every LLM call across all ~100+ calls/cycle is inspectable node-by-node.

---

## Phase 5 — Cutover & Cleanup (0.5 day)

| Task | Detail |
|---|---|
| Remove SQLite fallback code | Delete old `dev_patrika.db` references |
| Remove ChromaDB dependency | Uninstall `chromadb`, remove `chroma_db/` from repo |
| Update `requirements.txt` | Remove SQLite/Chroma-specific packages, add `asyncpg`/`psycopg`, `langchain-postgres`, `langgraph` |
| Update README/docs | Reflect new architecture |
| Final smoke test | Full ingestion cycle + chat session end-to-end on new stack |

---

## 📅 Suggested Timeline

| Phase | Duration | Can start after |
|---|---|---|
| 0. Prep | 0.5 day | — |
| 1. Neon migration | 1–2 days | Phase 0 |
| 2. pgvector migration | 1–2 days | Phase 1 |
| 3. LangGraph refactor | 3–5 days | Phase 2 (can overlap partially) |
| 4. LangSmith tracing | 0.5–1 day | Phase 3 (at least 3a done) |
| 5. Cutover | 0.5 day | All above |

**Total: ~7–11 days** for a solo dev working incrementally, assuming no major surprises.

---

## ⚠️ Key Risks to Watch

1. **Async connection pool exhaustion on Neon** — serverless Postgres has connection limits; use pooled endpoint + proper pool sizing in SQLAlchemy engine.
2. **Embedding dimension mismatch** — confirm `gemini-embedding-2` output dimension before creating the `vector(N)` column; wrong size = silent failures.
3. **RAG quality regression** — always A/B validate pgvector retrieval against old Chroma results before deleting Chroma.
4. **LangGraph state bloat** — keep `State` objects lean (IDs/references, not full raw content) to avoid checkpoint storage bloat in Postgres.
5. **Partial migration window** — during Phase 1–2, you'll temporarily have both old and new systems; make sure the scheduler doesn't run ingestion cycles against both simultaneously (double writes).

---

> **Recommended starting point**: Phase 0 + Phase 1 this week. Once Neon is stable and validated, move to Phase 2 (pgvector) since your chatbot depends on it — get that validated early rather than late.
