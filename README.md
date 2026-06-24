# Agentic RAG Research Assistant

<div align="center">
  <p>A production-grade AI research assistant that automatically ingests academic papers from arXiv and answers research questions using agentic RAG with hybrid search.</p>

  <img src="https://img.shields.io/badge/Python-3.12+-blue.svg" alt="Python">
  <img src="https://img.shields.io/badge/FastAPI-0.115+-green.svg" alt="FastAPI">
  <img src="https://img.shields.io/badge/OpenSearch-2.19-orange.svg" alt="OpenSearch">
  <img src="https://img.shields.io/badge/LangGraph-Agentic_RAG-purple.svg" alt="LangGraph">
  <img src="https://img.shields.io/badge/Docker-Compose-blue.svg" alt="Docker">
</div>

---

## What It Does

Fetches papers daily from the arXiv API, parses full PDF content using Docling, chunks and embeds them into OpenSearch, and exposes a conversational RAG API backed by a LangGraph agent. The agent evaluates retrieval quality, rewrites queries when needed, and detects out-of-domain questions — all observable through Langfuse tracing with Redis-cached responses.

---

## Architecture

```
arXiv API → Airflow DAG → Docling PDF Parser → PostgreSQL
                                                    ↓
                                          Chunker + Jina Embeddings
                                                    ↓
                                              OpenSearch
                                           (BM25 + Vector)
                                                    ↓
                                         LangGraph Agentic RAG
                                      ┌──────────────────────┐
                                      │  Guardrail Node       │
                                      │  Retrieve Node        │
                                      │  Document Grader      │
                                      │  Query Rewriter       │
                                      │  Generate Node        │
                                      └──────────────────────┘
                                                    ↓
                              FastAPI + Gradio UI + Telegram Bot
                                                    ↓
                                    Langfuse Tracing + Redis Cache
```

---

## Tech Stack

| Layer | Technology |
|---|---|
| API | FastAPI, Server-Sent Events (streaming) |
| Database | PostgreSQL 16 |
| Search | OpenSearch 2.19 — BM25 + vector hybrid with RRF fusion |
| Embeddings | Jina AI (`jina-embeddings-v3`, 1024-dim) |
| PDF Parsing | Docling |
| Orchestration | Apache Airflow 3.0 |
| Agent Framework | LangGraph |
| LLM | Ollama (local, `llama3.2`) |
| Observability | Langfuse |
| Caching | Redis |
| UI | Gradio |
| Bot | Telegram |
| Dev | UV, Ruff, MyPy, Pytest, Docker Compose |

---

## Key Features

**Hybrid Search** — RRF fusion of BM25 keyword search and Jina semantic embeddings. Achieves Precision@10 of 0.84 vs 0.67 for BM25 alone.

**Agentic RAG** — LangGraph workflow with guardrail detection, document relevance grading, automatic query rewriting on poor retrieval, and adaptive multi-attempt retrieval.

**Streaming Responses** — Server-Sent Events for real-time LLM output via `/api/v1/stream`.

**Full Observability** — End-to-end Langfuse tracing for every retrieval and generation step with latency and cost tracking.

**Redis Caching** — Exact-match query cache with configurable TTL. Up to 150-400x speedup on repeated queries.

**Telegram Bot** — Mobile-first conversational interface with async handling and full agent reasoning transparency.

**Automated Pipeline** — Daily Airflow DAG handles arXiv fetching, PDF parsing, chunking, embedding, and indexing with no manual intervention.

---

## Quick Start

### Prerequisites

- Docker Desktop
- Python 3.12+
- [UV](https://docs.astral.sh/uv/getting-started/installation/)
- 8GB+ RAM, 20GB+ disk

### Setup

```bash
git clone https://github.com/jamwithai/arxiv-paper-curator
cd arxiv-paper-curator

cp .env.example .env
# Add your JINA_API_KEY (free tier works)
# Optionally add TELEGRAM__BOT_TOKEN and LANGFUSE keys

uv sync
docker compose up --build -d

curl http://localhost:8000/api/v1/health
```

### Services

| Service | URL |
|---|---|
| API Docs | http://localhost:8000/docs |
| Gradio Chat UI | http://localhost:7861 |
| Langfuse Dashboard | http://localhost:3000 |
| Airflow | http://localhost:8080 |
| OpenSearch Dashboards | http://localhost:5601 |

> Airflow credentials: `airflow/simple_auth_manager_passwords.json.generated`

---

## API

```bash
# Hybrid search
curl -X POST http://localhost:8000/api/v1/hybrid-search/ \
  -H "Content-Type: application/json" \
  -d '{"query": "efficient transformer architectures", "use_hybrid": true, "size": 10}'

# Ask a question (streaming)
curl -X POST http://localhost:8000/api/v1/stream \
  -H "Content-Type: application/json" \
  -d '{"question": "What are the latest approaches to reducing LLM inference cost?"}'

# Agentic RAG (with guardrails + query rewriting)
curl -X POST http://localhost:8000/api/v1/agentic-ask \
  -H "Content-Type: application/json" \
  -d '{"question": "Summarize recent work on mixture-of-experts models"}'
```

| Endpoint | Method | Description |
|---|---|---|
| `/api/v1/health` | GET | Health check |
| `/api/v1/papers` | GET | List indexed papers |
| `/api/v1/search` | POST | BM25 keyword search |
| `/api/v1/hybrid-search/` | POST | Hybrid BM25 + vector search |
| `/api/v1/ask` | POST | RAG question answering |
| `/api/v1/stream` | POST | Streaming RAG response |
| `/api/v1/agentic-ask` | POST | Agentic RAG with reasoning |

---

## Project Structure

```
arxiv-paper-curator/
├── src/
│   ├── routers/                  # API endpoints
│   │   ├── search.py             # BM25 search
│   │   ├── hybrid_search.py      # Hybrid search
│   │   ├── ask.py                # RAG + streaming
│   │   └── agentic_ask.py        # Agentic RAG
│   ├── services/
│   │   ├── arxiv/                # arXiv API client
│   │   ├── pdf_parser/           # Docling integration
│   │   ├── opensearch/           # Search client
│   │   ├── indexing/             # Chunker + hybrid indexer
│   │   ├── embeddings/           # Jina AI client
│   │   ├── ollama/               # LLM client
│   │   ├── agents/               # LangGraph nodes + workflow
│   │   │   └── nodes/            # guardrail, retrieve, grade, rewrite, generate
│   │   ├── cache/                # Redis caching
│   │   ├── langfuse/             # Tracing
│   │   └── telegram/             # Bot handlers
│   ├── models/                   # SQLAlchemy models
│   ├── schemas/                  # Pydantic schemas
│   └── config.py
├── airflow/
│   └── dags/
│       └── arxiv_paper_ingestion.py
├── gradio_app.py
├── gradio_launcher.py
├── tests/
└── compose.yml
```

---

## Configuration

```bash
# Required
JINA_API_KEY=your_key                    # Jina embeddings (free tier)

# Optional
TELEGRAM__BOT_TOKEN=your_token           # Telegram bot
LANGFUSE__PUBLIC_KEY=your_key            # Observability
LANGFUSE__SECRET_KEY=your_key

# Defaults (work out of the box)
ARXIV__SEARCH_CATEGORY=cs.AI
ARXIV__MAX_RESULTS=15
CHUNKING__CHUNK_SIZE=600
CHUNKING__OVERLAP_SIZE=100
EMBEDDINGS__MODEL=jina-embeddings-v3
EMBEDDINGS__DIMENSIONS=1024
OLLAMA_MODEL=llama3.2:1b
```

---

## Development

```bash
make start        # Start all services
make stop         # Stop all services
make health       # Health check all services
make test         # Run test suite
make test-cov     # Tests with coverage report
make lint         # Ruff + MyPy
make format       # Auto-format code
make logs         # Tail service logs
make clean        # Full teardown
```

---

## Troubleshooting

**Services not starting** — wait 2-3 min after `docker compose up`, then check `docker compose logs`.

**Port conflicts** — ensure 8000, 8080, 5432, 9200, 3000, 6379 are free.

**Memory issues** — allocate at least 8GB to Docker Desktop; Docling and OpenSearch are memory-intensive.

**Full reset** — `docker compose down --volumes && docker compose up --build -d`

---

## License

MIT — see [LICENSE](LICENSE) for details.