# Technology Stack

**Project:** GroundControl_RAG
**Researched:** 2026-06-14
**Overall confidence:** HIGH (versions verified from PyPI and npm registries directly)

---

## Recommended Stack

### Backend

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| Python | 3.12 | Runtime | sentence-transformers requires >=3.10; 3.12 is current stable with best performance |
| FastAPI | 0.136.3 | API framework | Specified constraint; async-native, automatic OpenAPI docs, strong Pydantic integration |
| Uvicorn | 0.49.0 | ASGI server | Standard pairing with FastAPI; `uvicorn[standard]` includes uvloop for perf |
| Pydantic | 2.13.4 | Data validation | FastAPI 0.100+ requires Pydantic v2; v2 is 5-50x faster than v1 via Rust core |
| python-multipart | 0.0.32 | File upload parsing | FastAPI dependency for `UploadFile` / `Form` — required for document ingestion endpoint |
| python-dotenv | 1.2.2 | Config from .env | Standard for Docker Compose env var management; keeps secrets out of code |
| httpx | 0.28.1 | Async HTTP client | Used by Anthropic SDK internally; also useful for inter-service calls in tests |
| tenacity | 9.1.4 | Retry logic | Essential for LLM API calls (rate limits, transient errors); decorator-based, clean API |
| structlog | 26.1.0 | Structured logging | JSON-formatted logs are necessary for per-request telemetry; superior to stdlib logging |

### Authentication

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| PyJWT | 2.13.0 | JWT encode/decode | **Use PyJWT, not python-jose.** python-jose has had CVEs and is less actively maintained. PyJWT 2.x is the actively maintained standard. |
| bcrypt | 5.0.0 | Password hashing | Industry standard; use `bcrypt` directly rather than passlib (passlib is minimally maintained) |

### Database

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| PostgreSQL | 16 (Docker image) | Relational store | Specified constraint; users, documents, requests, evaluations tables |
| SQLAlchemy | 2.0.50 | ORM + query builder | v2 is fully async-native with `AsyncSession`; use `asyncpg` driver for async |
| asyncpg | 0.31.0 | PostgreSQL async driver | Fastest async Postgres driver for Python; pairs with SQLAlchemy 2.0 async |
| Alembic | 1.18.4 | Database migrations | Standard SQLAlchemy migration tool; required for schema evolution |

### Vector Database

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| ChromaDB | 1.5.9 | Vector store | Specified constraint; local persistent mode requires zero infra for < 1K docs; Python-native API |

### AI / ML

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| anthropic | 0.109.1 | Claude API client | Official Anthropic Python SDK; use for generation AND LLM-as-judge evaluation calls |
| sentence-transformers | 5.5.1 | Embedding generation | **Recommended for embeddings** — see decision below |

### Frontend

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| React | 19.2.7 | UI framework | Specified constraint; v19 is current stable with improved concurrent rendering |
| TypeScript | 6.0.3 | Type safety | Specified constraint; v6 is current stable |
| Tailwind CSS | 4.3.1 | Styling | Specified constraint; v4 rewrites the engine (no config file required, CSS-first) |
| Vite | 8.0.16 | Build tool | Fastest dev server and build; default for React+TS projects in 2025/2026 |
| React Router | 7.17.0 | Client routing | v7 merges Remix and React Router into one; file-based routing available but not required |
| TanStack Query | 5.101.0 | Server state / data fetching | Standard for async data in React; handles caching, refetching, loading/error states cleanly |
| Zustand | 5.0.14 | Client state | Minimal API for auth state and UI state; no boilerplate vs Redux; works well alongside TanStack Query |
| Recharts | 3.8.1 | Admin dashboard charts | Composable React charting built on D3; appropriate for the lightweight latency/failure rate dashboard |
| Axios | 1.17.0 | HTTP client | Interceptors make it easy to attach JWT Bearer tokens globally; cleaner than raw fetch for auth headers |

### Infrastructure

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| Docker Compose | v2 (latest) | Service orchestration | Specified constraint; single `docker-compose up` target |
| PostgreSQL (Docker) | postgres:16-alpine | DB container | Alpine image reduces size; pgdata volume for persistence |
| ChromaDB (Docker) | chromadb/chroma:latest | Vector DB container | Official image; mount a host volume at `/chroma/chroma` for persistence |

### Testing

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| pytest | 9.1.0 | Test runner | Standard Python testing |
| pytest-asyncio | 1.4.0 | Async test support | Required for testing FastAPI async endpoints and async SQLAlchemy calls |

---

## Embeddings Decision (Critical)

**Recommendation: sentence-transformers with `all-MiniLM-L6-v2`**

**Rationale:**

Anthropic confirmed does not offer a standalone embeddings API endpoint. The three realistic options are:

| Option | Pros | Cons | Decision |
|--------|------|------|----------|
| sentence-transformers (`all-MiniLM-L6-v2`) | Free, runs locally in Docker, no external API dependency, ChromaDB has a built-in `SentenceTransformerEmbeddingFunction`, 384-dim vectors are fast for < 1K docs | Requires ~90MB model download on first run; adds torch dependency | **CHOSEN** |
| OpenAI text-embedding-3-small | High quality, managed, no torch dep | Adds a second external API vendor dependency (cost + another key to manage + network latency per embed), couples ingestion to OpenAI availability | Rejected |
| Cohere Embed v3 | High quality multilingual support | Same vendor-dependency problem as OpenAI; overkill for v1 English docs | Rejected |

**Why sentence-transformers wins for this project:**

1. **No external API calls during ingestion.** Document ingestion runs entirely in-process. Adding OpenAI/Cohere means every chunk embed is a network call, adding latency and failure modes to what should be a local pipeline.
2. **Docker-contained.** The model weights are downloaded once into the container image (or cached volume). Zero operational dependency on a third-party embeddings API being up.
3. **ChromaDB native integration.** ChromaDB ships `chromadb.utils.embedding_functions.SentenceTransformerEmbeddingFunction` — this means you pass the embedding function directly to the ChromaDB collection and ChromaDB handles embedding automatically at both ingest and query time. You write less code.
4. **`all-MiniLM-L6-v2` is the right model for this scale.** At < 1K docs, 384-dimensional vectors are more than sufficient. The model is 80MB, runs on CPU, and produces embeddings in ~10ms per chunk. No GPU required.

**Implementation note:** Pin the model name in config so it can be swapped in v2 without code changes.

```python
# config.py
EMBEDDING_MODEL = "all-MiniLM-L6-v2"  # swap to "all-mpnet-base-v2" in v2 if quality needs improvement

# chroma_client.py
from chromadb.utils.embedding_functions import SentenceTransformerEmbeddingFunction

embedding_fn = SentenceTransformerEmbeddingFunction(model_name=settings.EMBEDDING_MODEL)
collection = chroma_client.get_or_create_collection(
    name="documents",
    embedding_function=embedding_fn,
)
```

---

## Alternatives Considered

| Category | Recommended | Alternative | Why Not |
|----------|-------------|-------------|---------|
| Embeddings | sentence-transformers | OpenAI text-embedding-3-small | Vendor dependency, network latency per chunk, costs money |
| Embeddings | sentence-transformers | Cohere Embed v3 | Same vendor-dependency problem; multilingual features not needed |
| JWT | PyJWT | python-jose | python-jose has CVEs, slower maintenance cadence |
| Password hashing | bcrypt (direct) | passlib | passlib maintenance has slowed significantly; bcrypt direct is cleaner |
| ORM | SQLAlchemy 2.0 async | Tortoise-ORM / databases | SQLAlchemy is the ecosystem standard, best Alembic integration |
| State management | Zustand | Redux Toolkit | Redux is overkill for auth + UI state; Zustand has 1/10th the boilerplate |
| Charts | Recharts | Chart.js / Victory | Recharts is React-native (no DOM imperative API), composable, well-maintained |
| Build tool | Vite | Create React App | CRA is unmaintained as of 2023; Vite is the community standard |
| Async HTTP driver | asyncpg | psycopg2 | psycopg2 is synchronous; asyncpg is required for SQLAlchemy 2.0 async mode |
| Logging | structlog | stdlib logging | structlog produces JSON-formatted structured logs essential for per-request telemetry |

---

## What NOT to Use

| Library | Reason to Avoid |
|---------|----------------|
| RAGAS | Project explicitly excludes external eval frameworks; custom metrics only |
| LangChain / LlamaIndex | Heavy abstractions that obscure the pipeline; debugging becomes hard; not needed when building custom RAG with direct Anthropic SDK + ChromaDB |
| python-jose | CVE history; use PyJWT instead |
| passlib | Low maintenance; use bcrypt directly |
| Create React App | Unmaintained since 2023 |
| Prometheus + Grafana | Explicitly out of scope for v1; lightweight custom dashboard only |
| pgvector | Not needed; ChromaDB is the specified vector store |
| Redis | Not needed for v1; no caching layer required at < 1K doc scale |
| Celery | Async document ingestion can be handled with FastAPI BackgroundTasks for v1 scale; Celery adds significant operational overhead |

---

## Installation

```bash
# Backend (requirements.txt)
fastapi==0.136.3
uvicorn[standard]==0.49.0
pydantic==2.13.4
pydantic-settings==2.10.1
python-multipart==0.0.32
python-dotenv==1.2.2
httpx==0.28.1
tenacity==9.1.4
structlog==26.1.0

# Auth
PyJWT==2.13.0
bcrypt==5.0.0

# Database
sqlalchemy==2.0.50
asyncpg==0.31.0
alembic==1.18.4

# Vector DB
chromadb==1.5.9

# AI/ML
anthropic==0.109.1
sentence-transformers==5.5.1

# Document parsing
pdfplumber==0.11.9
python-docx==1.2.0

# Testing
pytest==9.1.0
pytest-asyncio==1.4.0
httpx==0.28.1  # also used as test client for FastAPI
```

```bash
# Frontend (package.json key deps)
react@19.2.7
react-dom@19.2.7
typescript@6.0.3
tailwindcss@4.3.1
vite@8.0.16
react-router-dom@7.17.0
@tanstack/react-query@5.101.0
zustand@5.0.14
recharts@3.8.1
axios@1.17.0
```

---

## Key Integration Patterns

### Anthropic SDK (generation + LLM-as-judge)

```python
import anthropic

client = anthropic.Anthropic(api_key=settings.ANTHROPIC_API_KEY)

# Generation
message = client.messages.create(
    model="claude-opus-4-5",  # or claude-sonnet-4-5 for cost/speed tradeoff
    max_tokens=1024,
    messages=[{"role": "user", "content": prompt}]
)

# LLM-as-judge evaluation uses the same client with a structured judge prompt
```

### ChromaDB with sentence-transformers

```python
import chromadb
from chromadb.utils.embedding_functions import SentenceTransformerEmbeddingFunction

client = chromadb.PersistentClient(path="/data/chroma")
embedding_fn = SentenceTransformerEmbeddingFunction(model_name="all-MiniLM-L6-v2")
collection = client.get_or_create_collection("documents", embedding_function=embedding_fn)

# Ingest — ChromaDB calls the embedding function automatically
collection.add(documents=[chunk_text], metadatas=[metadata], ids=[chunk_id])

# Query — embedding function applied to query automatically
results = collection.query(query_texts=[user_question], n_results=10)
```

### FastAPI BackgroundTasks for document ingestion

```python
from fastapi import BackgroundTasks

@router.post("/documents")
async def upload_document(file: UploadFile, background_tasks: BackgroundTasks):
    doc_id = await save_document_record(file)
    background_tasks.add_task(process_and_embed_document, doc_id, file)
    return {"id": doc_id, "status": "processing"}
```

---

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| All versions | HIGH | Verified directly from PyPI JSON API and npm registry at time of research (2026-06-14) |
| Embeddings recommendation | HIGH | Verified Anthropic has no embeddings API; ChromaDB's SentenceTransformerEmbeddingFunction is documented in chromadb package |
| Anthropic SDK patterns | MEDIUM | SDK version verified; specific method signatures may drift — verify against anthropic==0.109.1 changelog before implementation |
| ChromaDB persistent client API | MEDIUM | Version verified; `PersistentClient` is the correct API for local persistence in chromadb 0.4+ / 1.x |
| FastAPI + Pydantic v2 | HIGH | Well-established, no breaking changes expected at these patch versions |
| Frontend versions | HIGH | Verified from npm registry; Tailwind v4 is a significant rewrite — CSS-first config, no tailwind.config.js by default |

---

## Sources

- PyPI JSON API: `https://pypi.org/pypi/{package}/json` — verified 2026-06-14
- npm Registry API: `https://registry.npmjs.org/{package}/latest` — verified 2026-06-14
- Anthropic Python SDK: confirmed no embeddings endpoint (generation and messages API only)
- ChromaDB embedding functions: `chromadb.utils.embedding_functions.SentenceTransformerEmbeddingFunction` is a first-party integration
- Confidence: MEDIUM on API patterns (training data); HIGH on version numbers (live registry verification)
