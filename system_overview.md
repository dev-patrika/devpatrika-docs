# Dev Patrika: Backend Engine Overview

This document provides a professional reference summary of the **Dev Patrika** backend engine architecture, capabilities, and API endpoints implemented up to `v0.4.0-alpha`.

---

## 🏗️ System Architecture

Dev Patrika's backend is a modular developer intelligence engine built using **FastAPI**, **SQLModel (SQLAlchemy)**, and **LangChain**. The system combines a relational storage layer (**SQLite**) for structured records and a vector storage layer (**Chroma DB**) for semantic conceptual lookups.

```mermaid
graph TD
    subgraph Data Sources
        HN[Hacker News API]
        DevTo[Dev.to REST API]
        arXiv[arXiv Atom API]
        GH[GitHub Scraper]
    end

    subgraph Ingestion Layer
        Orch[Ingestion Orchestrator]
        Dedup[Deduplication Scorer]
        Sched[asyncio 6-Hour Scheduler]
    end

    subgraph Storage Layer
        DB[(SQLite DB)]
        VectorDB[(Chroma Vector DB)]
    end

    subgraph AI Pipeline Layer
        Pipeline[LangChain Processing pipeline]
        LLM[Groq / Gemini Fallback Manager]
        Curator[Wiki Curator Pipeline]
    end

    HN & DevTo & arXiv & GH --> Orch
    Orch --> Dedup
    Dedup --> DB
    DB --> Pipeline
    Pipeline --> LLM
    Pipeline --> DB
    
    DB --> Curator
    Curator --> LLM
    Curator --> DB
    Curator --> VectorDB
```

---

## 🎯 Key Capabilities & Core Engines

### 1. Multi-Source Ingestion Engine
* Automatically crawls and extracts tech updates from the developer ecosystem:
  * **Hacker News**: Firebase REST API (top stories).
  * **Dev.to**: RSS/API endpoints matching tech tags (Python, Web Dev, DevOps, AI, Security).
  * **arXiv**: Public Atom feeds for Computer Science and Machine Learning preprints (`cs.AI`, `cs.LG`, `cs.SE`).
  * **GitHub**: Scrapes daily trending repository metrics (languages, descriptions, stars).

### 2. Duplication & Overlap Control
* Prevents data pollution through a two-tiered check:
  * **Unique URL Index**: SQLite constraint blocks duplicate URLs on insertion.
  * **Fuzzy Title Scorer**: Computes Token-based Jaccard similarity. Stories with **>80% similarity** to articles ingested in the last 24 hours are skipped, maintaining diverse topics.

### 3. Asynchronous Scheduler
* Integrates a non-blocking `asyncio` task loop running inside the FastAPI lifespan context.
* Polls data feeds automatically every **6 hours** and sequences three distinct phases: Ingestion -> News Summarization -> Auto-Wiki Curation.

### 4. AI Summarization & Classification Pipeline
* Leverages LangChain's `RunnableSequence` to transform raw descriptions/abstracts into structured, professional markdown.
* **Unified Summary Format**:
  * **Overview**: Concise 1-2 sentence introduction of the news item.
  * **Key Details**: 3-4 bullet-point takeaways.
  * **Community & Traction**: Popularity, developer activity, and adoption context.
* **AI Categorizer**: Classifies stories into tech buckets (*AI, Web Dev, Cybersecurity, Startups, Open Source, Cloud/DevOps*).
* **GitHub "Why it Matters" Radar**: Evaluates trending repos to detail architectural highlights and developer-focused summaries.

### 5. Multi-Provider LLM Fallback (Failover Engine)
* Uses native LangChain model fallbacks to guarantee uptime:
  * **Primary Model**: Groq (`openai/gpt-oss-120b`).
  * **Secondary Backup**: Google Gemini (`gemini-2.5-flash`).
  * If the primary model encounters rate limits, timeouts, or authentication issues, it fails over to the backup instantly and silently.

### 6. Dev Wiki Compiler
* Automatically compiles a dictionary of technical concepts on-demand.
* Extracts structured definitions, context on why the term is trending, and verified official resource links.

### 7. Local Vector Store & Semantic Search Engine
* Connects to a persistent local Chroma DB database located at `Backend/chroma_db/`.
* Converts text definitions into vectors using Google GenAI's **`gemini-embedding-2`** embeddings model.
* Performs semantic cosine similarity lookup: returns matching concept glossary definitions based on meaning, bypassing strict keyword matches (e.g., query `"stateful multi-agent systems"` matches `"LangGraph"`).

### 8. Automated Wiki Curation
* Scans news stories summarized during the scheduler cycle.
* Prompts the LLM (using the fallback chain) to extract key new developer terms or libraries.
* Automatically creates wiki definitions for missing concepts and indexes them in Chroma DB, building a self-expanding technical glossary.

---

## 🔌 API Endpoints Summary

| Method | Endpoint | Description | Lifecycle Group |
|:---|:---|:---|:---|
| **GET** | `/api/health` | Service health status check. | System |
| **GET** | `/api/news` | Retrieve daily news feeds with category, query, and limit filters. | News |
| **POST** | `/api/news/ingest` | Triggers background crawlers and initiates LLM processing. | News |
| **POST** | `/api/news/process` | Processes pending raw items in the DB via AI pipeline. | News |
| **GET** | `/api/github/trending` | Returns stored repository radar with AI summaries. | GitHub Radar |
| **GET** | `/api/wiki` | Returns list of concept definitions with autocomplete query support. | Dev Wiki |
| **GET** | `/api/wiki/{term}` | Fetch case-insensitive wiki entries. | Dev Wiki |
| **POST** | `/api/wiki/generate` | Dispatch LangChain worker to generate concept definitions. | Dev Wiki |
| **GET** | `/api/search` | Unified parallel search querying news & repos (SQL) and wiki concepts (Chroma). | Search |
