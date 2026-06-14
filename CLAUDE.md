<!-- GSD:project-start source:PROJECT.md -->
## Project

**GroundControl_RAG**

GroundControl_RAG is a production-grade Retrieval Augmented Generation platform that lets users query multi-domain knowledge bases (internal docs, customer support content, technical references) using natural language. Every response is grounded in retrieved source documents with citations, scored by an evaluation engine, and protected by fallback logic that refuses to hallucinate when evidence is weak. It targets employees, customers, and developer/AI teams within the same deployment.

**Core Value:** Users get grounded, cited answers from company knowledge — and the system tells you when it doesn't know rather than making things up.

### Constraints

- **Tech Stack (Backend)**: Python + FastAPI — specified in base.md
- **Tech Stack (Frontend)**: React + TypeScript + Tailwind — specified in base.md
- **Tech Stack (Vector DB)**: ChromaDB — chosen for simplicity at < 1K doc scale
- **Tech Stack (LLM)**: Claude (Anthropic) — generation and evaluation
- **Tech Stack (Database)**: PostgreSQL — user, document, request, evaluation tables
- **Deployment**: Docker Compose — must run with a single `docker-compose up`
- **Evaluation**: Custom metrics only — no RAGAS or external eval framework dependency
- **Auth**: JWT-based — sessions for users, no API key auth in v1
- **Embeddings**: TBD (constraint: not Anthropic native API) — resolve in research
<!-- GSD:project-end -->

<!-- GSD:stack-start source:research/STACK.md -->
## Technology Stack

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
## Embeddings Decision (Critical)
| Option | Pros | Cons | Decision |
|--------|------|------|----------|
| sentence-transformers (`all-MiniLM-L6-v2`) | Free, runs locally in Docker, no external API dependency, ChromaDB has a built-in `SentenceTransformerEmbeddingFunction`, 384-dim vectors are fast for < 1K docs | Requires ~90MB model download on first run; adds torch dependency | **CHOSEN** |
| OpenAI text-embedding-3-small | High quality, managed, no torch dep | Adds a second external API vendor dependency (cost + another key to manage + network latency per embed), couples ingestion to OpenAI availability | Rejected |
| Cohere Embed v3 | High quality multilingual support | Same vendor-dependency problem as OpenAI; overkill for v1 English docs | Rejected |
# config.py
# chroma_client.py
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
## Installation
# Backend (requirements.txt)
# Auth
# Database
# Vector DB
# AI/ML
# Document parsing
# Testing
# Frontend (package.json key deps)
## Key Integration Patterns
### Anthropic SDK (generation + LLM-as-judge)
# Generation
# LLM-as-judge evaluation uses the same client with a structured judge prompt
### ChromaDB with sentence-transformers
# Ingest — ChromaDB calls the embedding function automatically
# Query — embedding function applied to query automatically
### FastAPI BackgroundTasks for document ingestion
## Confidence Assessment
| Area | Confidence | Notes |
|------|------------|-------|
| All versions | HIGH | Verified directly from PyPI JSON API and npm registry at time of research (2026-06-14) |
| Embeddings recommendation | HIGH | Verified Anthropic has no embeddings API; ChromaDB's SentenceTransformerEmbeddingFunction is documented in chromadb package |
| Anthropic SDK patterns | MEDIUM | SDK version verified; specific method signatures may drift — verify against anthropic==0.109.1 changelog before implementation |
| ChromaDB persistent client API | MEDIUM | Version verified; `PersistentClient` is the correct API for local persistence in chromadb 0.4+ / 1.x |
| FastAPI + Pydantic v2 | HIGH | Well-established, no breaking changes expected at these patch versions |
| Frontend versions | HIGH | Verified from npm registry; Tailwind v4 is a significant rewrite — CSS-first config, no tailwind.config.js by default |
## Sources
- PyPI JSON API: `https://pypi.org/pypi/{package}/json` — verified 2026-06-14
- npm Registry API: `https://registry.npmjs.org/{package}/latest` — verified 2026-06-14
- Anthropic Python SDK: confirmed no embeddings endpoint (generation and messages API only)
- ChromaDB embedding functions: `chromadb.utils.embedding_functions.SentenceTransformerEmbeddingFunction` is a first-party integration
- Confidence: MEDIUM on API patterns (training data); HIGH on version numbers (live registry verification)
<!-- GSD:stack-end -->

<!-- GSD:conventions-start source:CONVENTIONS.md -->
## Conventions

Conventions not yet established. Will populate as patterns emerge during development.
<!-- GSD:conventions-end -->

<!-- GSD:architecture-start source:ARCHITECTURE.md -->
## Architecture

Architecture not yet mapped. Follow existing patterns found in the codebase.
<!-- GSD:architecture-end -->

<!-- GSD:workflow-start source:GSD defaults -->
## GSD Workflow Enforcement

Before using Edit, Write, or other file-changing tools, start work through a GSD command so planning artifacts and execution context stay in sync.

Use these entry points:
- `/gsd:quick` for small fixes, doc updates, and ad-hoc tasks
- `/gsd:debug` for investigation and bug fixing
- `/gsd:execute-phase` for planned phase work

Do not make direct repo edits outside a GSD workflow unless the user explicitly asks to bypass it.
<!-- GSD:workflow-end -->



<!-- GSD:profile-start -->
## Developer Profile

> Profile not yet configured. Run `/gsd:profile-user` to generate your developer profile.
> This section is managed by `generate-claude-profile` -- do not edit manually.
<!-- GSD:profile-end -->
