# Dev Patrika: Priority-Based Development Roadmap

This document outlines the development priorities for the **Dev Patrika** backend. To ensure stability and focus, we divide the roadmap into three progressive implementation phases: LangChain Core, LangGraph Agents, and LangSmith Observability.

---

## 🎯 Priority 1: LangChain Core & DB Foundation (Current Focus)
This phase builds the foundational storage engine, collects raw feeds, and runs linear LLM chains (using LangChain Core) to summarize, classify, vectorise, and search data.

### Milestone 1.1: Project Skeleton & Database Setup (`v0.1.0-alpha`)
* **Objective**: Build the application shell and relational storage.
* **Backend Deliverables**:
  * Set up FastAPI structure and configurations (`app/`, `main.py`, `.env`).
  * Initialize SQLite via SQLAlchemy/SQLModel (`dev_patrika.db`).
  * Define tables for `NewsItem`, `WikiEntry`, and `GitHubRadar`.
  * Establish API health-check (`GET /api/health`).

### Milestone 1.2: Feed Ingestion Services (`v0.2.0-alpha`)
* **Objective**: Automate content collection from developer sources.
* **Backend Deliverables**:
  * Write APIs/scrapers for Hacker News, Dev.to, GitHub Trending, and arXiv.
  * Implement background scheduling to periodically pull new stories.
  * Integrate duplicate-prevention logic using URL hash checks.

### Milestone 1.3: LangChain Core Processing Pipeline (`v0.3.0-alpha`)
* **Objective**: Process raw stories into structured summaries and tags using LLMs.
* **Backend Deliverables**:
  * Create LangChain `RunnableSequence` pipelines.
  * **Classifier**: Classify stories into *AI, Web Dev, Cybersecurity, Startups, Open Source, Cloud/DevOps*.
  * **Summarizer**: Generate bullet-point summaries and credit sources.
  * Implement failover switching (e.g., fall back from Groq to Gemini if rate-limited).

### Milestone 1.4: Dev Wiki & Local Chroma Vector DB (`v0.4.0-alpha`)
* **Objective**: Detect trending terms, create Wiki definitions, and search them semantically.
* **Backend Deliverables**:
  * Integrate local Chroma DB to index Wiki definitions.
  * **Wiki Curator Chain**: Extract trending keywords from the news, write entries (definition, importance, references), and store/embed them.
  * Build unified search API (`GET /api/search`) querying SQLite (news) and Chroma (wiki) in parallel.

### Milestone 1.5: Model Router & Basic Chat endpoint (`v0.5.0-beta`)
* **Objective**: Allow runtime LLM switching and deploy a basic conversational assistant.
* **Backend Deliverables**:
  * Endpoint `GET /api/ai/models` to toggle between Gemini, Groq, and Hugging Face.
  * Baseline `POST /api/ai/chat` endpoint using LangChain's memory wrapper for simple Q&A.
  * Support user localStorage preferences (bookmarks, filter tags).

---

## 🕸️ Priority 2: LangGraph Autonomous Agents (Next Phase)
Once the database and linear chains are stable, we will transition to stateful, loop-capable agent networks using LangGraph.

### Milestone 2.1: Multi-Agent Collaboration Graph (`v0.6.0-beta`)
* **Objective**: Replace simple linear pipelines with specialized autonomous agents.
* **Backend Deliverables**:
  * Set up LangGraph state schemas and stateful nodes.
  * **Daily Brief Agent**: Autonomous fetch -> dedupe -> summarize -> classify graph.
  * **Wiki Curator Agent**: Manage trending term definitions and resolve wiki conflicts.
  * **Research Digest Agent**: Simplify arXiv PDFs.
  * **Explain Why Agent**: Pull github PRs and documentation to explain news context.

### Milestone 2.2: MCP Integration & Human Checkpoints (`v0.7.0-beta`)
* **Objective**: Standardize agent tools and insert editorial approval gates.
* **Backend Deliverables**:
  * Build Model Context Protocol (MCP) server endpoints for tools (News search, GitHub lookup, Web scraper).
  * Implement **Human-in-the-Loop** approval states in the graph, saving state to SQLite before publishing content.
  * Configure persistent SQLite-backed graph checkpointers.

---

## 📊 Priority 3: LangSmith Observability & Release (Final Phase)
This phase focuses on quality, optimization, tracing, cost control, and production freeze.

### Milestone 3.1: LangSmith Tracing & Evaluations (`v0.8.0-rc`)
* **Objective**: Add deep logging, QA evaluations, and cost monitoring.
* **Backend Deliverables**:
  * Integrate LangSmith SDK to trace all agent node transitions, prompt templates, and tool calls.
  * Setup automatic evaluation suites in LangSmith:
    * Summarization quality (faithfulness/hallucination checks).
    * Search relevance score.
  * Manage prompts dynamically using the LangSmith Prompt Hub.
  * Log execution costs and latencies.

### Milestone 3.2: Production Freeze & Deploy (`v1.0.0`)
* **Objective**: Stabilize API security, caching, rate limiting, and bundle the app.
* **Backend Deliverables**:
  * Complete API schemas & documentation.
  * Build CORS configs and api rate limits.
  * Write Docker files and setup guides.
