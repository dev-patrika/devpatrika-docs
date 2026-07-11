# Dev Patrika — Frozen Schema Reference (Pre-Migration)
> **Snapshot Date**: 2026-07-11
> **Database**: SQLite (`dev_patrika.db` — 1.4 MB)
> **ORM**: SQLModel 0.0.22

This document captures the exact database schema before the Neon (Postgres + pgvector) migration.
All 7 models will be reused nearly 1:1 on Postgres, with dialect-specific fixes noted.

---

## Table 1: `news_items` (Model: `NewsItem`)

**File**: `app/models/news.py`

| Column | Type | Constraints | Notes |
|--------|------|-------------|-------|
| `id` | `Optional[int]` | PRIMARY KEY, auto-increment | — |
| `title` | `str` | NOT NULL | — |
| `url` | `str` | UNIQUE, INDEXED | Dedup key |
| `summary` | `Optional[str]` | — | LLM-generated markdown; NULL = pending processing |
| `category` | `Optional[TechCategory]` | INDEXED | Enum: AI, Web Dev, Cybersecurity, Startups, Open Source, Cloud/DevOps |
| `source` | `NewsSource` | INDEXED | Enum: Hacker News, Dev.to, GitHub Trending, arXiv |
| `published_at` | `datetime` | NOT NULL | Source publish timestamp |
| `created_at` | `datetime` | NOT NULL, default=utcnow | Ingestion timestamp |
| `raw_content` | `Optional[str]` | — | Original scraped content |
| `freshness_tag` | `Optional[str]` | — | Human-readable relative time ("2 hours ago") |

---

## Table 2: `github_radar` (Model: `GitHubRadar`)

**File**: `app/models/github_radar.py`

| Column | Type | Constraints | Notes |
|--------|------|-------------|-------|
| `id` | `Optional[int]` | PRIMARY KEY, auto-increment | — |
| `repo_name` | `str` | INDEXED | Format: "owner/name" |
| `repo_url` | `str` | UNIQUE | GitHub URL |
| `description` | `Optional[str]` | — | Repo description |
| `why_it_matters_summary` | `Optional[str]` | — | LLM-generated; NULL = pending |
| `stars_count` | `int` | default=0 | Star count at scrape time |
| `created_at` | `datetime` | NOT NULL, default=utcnow | — |

---

## Table 3: `wiki_entries` (Model: `WikiEntry`)

**File**: `app/models/wiki.py`

| Column | Type | Constraints | Notes |
|--------|------|-------------|-------|
| `id` | `Optional[int]` | PRIMARY KEY, auto-increment | — |
| `term` | `str` | UNIQUE, INDEXED | Technical term name |
| `definition` | `str` | NOT NULL | LLM-generated markdown definition |
| `why_trending` | `str` | NOT NULL | Explanation of current relevance |
| `related_links` | `Optional[str]` | default="[]" | ⚠️ **JSON stored as TEXT string** — migrate to JSONB on Postgres |
| `created_at` | `datetime` | NOT NULL, default=utcnow | — |
| `updated_at` | `datetime` | NOT NULL, default=utcnow | — |

> **Migration Note**: `related_links` uses `json.dumps()` / `json.loads()` for serialization. On Postgres, switch to native `JSONB` column type using `sa_column=Column(JSONB)`.

**Property methods on model:**
- `links_list` (property) → returns `json.loads(self.related_links)`
- `set_links(links: list)` → sets `self.related_links = json.dumps(links)`

---

## Table 4: `chat_messages` (Model: `ChatMessage`)

**File**: `app/models/chat_history.py`

| Column | Type | Constraints | Notes |
|--------|------|-------------|-------|
| `id` | `Optional[int]` | PRIMARY KEY, auto-increment | — |
| `session_id` | `str` | INDEXED | Groups messages by chat session |
| `role` | `str` | NOT NULL | "user" or "assistant" |
| `content` | `str` | NOT NULL | Message text |
| `created_at` | `datetime` | NOT NULL, default=utcnow | — |

---

## Table 5: `trending_topics` (Model: `TrendingTopic`)

**File**: `app/models/trending_topic.py`

| Column | Type | Constraints | Notes |
|--------|------|-------------|-------|
| `id` | `Optional[int]` | PRIMARY KEY, auto-increment | — |
| `term` | `str` | UNIQUE, INDEXED | Must match a wiki_entries term |
| `frequency` | `int` | default=0 | Mention count in last 7 days of news |
| `trend_direction` | `str` | default="stable" | "up", "down", or "stable" |
| `updated_at` | `datetime` | NOT NULL, default=utcnow | — |

---

## Table 6: `weekly_reports` (Model: `WeeklyReport`)

**File**: `app/models/weekly_report.py`

| Column | Type | Constraints | Notes |
|--------|------|-------------|-------|
| `id` | `Optional[int]` | PRIMARY KEY, auto-increment | — |
| `title` | `str` | INDEXED | Report title with date range |
| `content` | `str` | NOT NULL | Full compiled markdown report |
| `start_date` | `datetime` | NOT NULL | Report coverage start |
| `end_date` | `datetime` | NOT NULL | Report coverage end |
| `created_at` | `datetime` | NOT NULL, default=utcnow | — |

---

## Table 7: `personal_notes` (Model: `PersonalNote`)

**File**: `app/models/notes.py`

| Column | Type | Constraints | Notes |
|--------|------|-------------|-------|
| `id` | `Optional[int]` | PRIMARY KEY, auto-increment | — |
| `title` | `str` | INDEXED | Note title |
| `content` | `str` | NOT NULL | Note body |
| `created_at` | `datetime` | NOT NULL, default=utcnow | — |
| `updated_at` | `datetime` | NOT NULL, default=utcnow | — |

> **Note**: No router exists for this table yet — it's a dormant model.

---

## Enums

### `TechCategory` (app/core/constants.py)
```
AI | Web Dev | Cybersecurity | Startups | Open Source | Cloud/DevOps
```

### `NewsSource` (app/core/constants.py)
```
Hacker News | Dev.to | GitHub Trending | arXiv
```

---

## ChromaDB Collections (Pre-Migration Snapshot)

| Collection | Embedding Model | Source Table | Document Format |
|------------|-----------------|-------------|-----------------|
| `wiki_entries` | `gemini-embedding-2` | `wiki_entries` | "Term: ...\nDefinition: ...\nWhy trending: ..." |
| `news_items` | `gemini-embedding-2` | `news_items` | "Category: ...\nTitle: ...\nSummary: ...\nContent: ..." (capped at 8000 chars) |

**Storage**: `chroma_db/` directory with 2 HNSW index collections
**Total vector data size**: ~18 MB (SQLite metadata + binary vectors)

---

## SQLite-Specific Code to Update During Migration

| File | What | Migration Action |
|------|------|-----------------|
| `config.py` | `DATABASE_URL = "sqlite:///./dev_patrika.db"` | Change to Neon Postgres connection string |
| `database.py` | `connect_args = {"check_same_thread": False}` | Remove — Postgres doesn't need this |
| `models/wiki.py` | `related_links: Optional[str]` (JSON-as-string) | Change to `JSONB` column type |
| `models/wiki.py` | `links_list` property + `set_links()` | Simplify — JSONB handles serialization natively |
