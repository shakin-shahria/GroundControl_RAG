# Research Summary: GroundControl_RAG

**Synthesized:** 2026-06-14
**Sources:** STACK.md, FEATURES.md, ARCHITECTURE.md, PITFALLS.md, PROJECT.md
**Overall Confidence:** HIGH (stack versions verified from live registries; architecture patterns are well-established production RAG patterns)

---

## Executive Summary

GroundControl_RAG is a single-tenant, document-grounded Q&A platform for enterprise internal knowledge bases. The core value proposition — grounded, cited answers with explicit refusal to hallucinate — is non-trivial to implement correctly and easy to implement badly. The research shows that the distinguishing quality comes not from the LLM (Claude handles grounding well by default) but from three interacting layers: ingestion quality (chunking strategy, metadata preservation), retrieval quality (cross-encoder reranking on top of vector search), and validation quality (LLM-as-judge evaluation with calibrated rubrics, not naive prompting).

The recommended architecture is an in-process FastAPI monolith for v1. All pipeline stages (embedding, retrieval, reranking, generation, validation, evaluation) run inside the same FastAPI process using Python async machinery and FastAPI BackgroundTasks. This is intentional: at fewer than 1K documents, the per-request throughput does not justify separate microservices, and in-process architecture dramatically reduces debugging complexity. The architecture is designed for extraction — the Embedding Service, Reranker, and Evaluation Engine are each behind a clear interface boundary so they can be pulled into separate workers in v2 when telemetry data justifies it.

The biggest risks are silent failures: embedding model mismatch between ingestion and query time (retrieval collapses with no error), ChromaDB persistence misconfiguration (data silently lost on container restart), and LLM-as-judge score inflation (evaluation always returns 0.8+ regardless of actual quality). Each of these fails without throwing an exception, making them invisible until a human notices that answers are wrong. The mitigation for all three is the same: write startup assertions that fail loudly, and build a golden test dataset early.

---

## Recommended Stack (Definitive)

### Backend

| Technology | Version | Role |
|------------|---------|------|
| Python | 3.12 | Runtime |
| FastAPI | 0.136.3 | API framework |
| Uvicorn | 0.49.0 | ASGI server |
| Pydantic | 2.13.4 | Data validation |
| SQLAlchemy | 2.0.50 | ORM (async) |
| asyncpg | 0.31.0 | PostgreSQL async driver |
| Alembic | 1.18.4 | Database migrations |
| anthropic | 0.109.1 | Claude API client (generation + evaluation) |
| sentence-transformers | 5.5.1 | Embeddings (all-MiniLM-L6-v2) |
| chromadb | 1.5.9 | Vector store |
| PyJWT | 2.13.0 | JWT auth (NOT python-jose — has CVEs) |
| bcrypt | 5.0.0 | Password hashing (NOT passlib — low maintenance) |
| tenacity | 9.1.4 | Retry logic for LLM API calls |
| structlog | 26.1.0 | Structured JSON logging |
| pdfplumber | 0.11.9 | PDF parsing |
| python-docx | 1.2.0 | DOCX parsing |
| pytest | 9.1.0 | Test runner |
| pytest-asyncio | 1.4.0 | Async test support |

### Frontend

| Technology | Version | Role |
|------------|---------|------|
| React | 19.2.7 | UI framework |
| TypeScript | 6.0.3 | Type safety |
| Tailwind CSS | 4.3.1 | Styling (v4 is CSS-first, no config file) |
| Vite | 8.0.16 | Build tool |
| React Router | 7.17.0 | Client routing |
| TanStack Query | 5.101.0 | Server state / data fetching |
| Zustand | 5.0.14 | Client state (auth + UI) |
| Recharts | 3.8.1 | Admin dashboard charts |
| Axios | 1.17.0 | HTTP client (JWT interceptors) |

### Infrastructure

| Technology | Version | Role |
|------------|---------|------|
| Docker Compose | v2 | Service orchestration |
| PostgreSQL | 16-alpine | Relational store |
| ChromaDB | chromadb/chroma:latest | Vector DB (local persistent mode) |

---

## Embeddings Decision (Resolved)

**Chosen: sentence-transformers with all-MiniLM-L6-v2**

Anthropic has no standalone embeddings API endpoint — this was an open question in PROJECT.md and is now resolved. The embedding model runs in-process via sentence-transformers. The model is 80MB, runs on CPU, and produces 384-dimensional vectors in ~10ms per chunk. No external API dependency, no network latency per chunk, no additional vendor.

Pin the model name in config: `EMBEDDING_MODEL = "all-MiniLM-L6-v2"`. Never allow implicit upgrades — changing the model after ingestion requires re-embedding all documents. The Embedding Service must be the single source of truth for vectors; pass embeddings manually to ChromaDB rather than using ChromaDB's built-in embedding function.

---

## Table Stakes Features (Must Be in V1)

1. **Document ingestion pipeline** — PDF, DOCX, TXT with metadata stamped per document and per chunk (doc_id, filename, page number, section heading, chunk index)
2. **Fixed-size chunking with overlap** — 512 tokens per chunk, 64-token overlap, sentence-boundary-aware; boilerplate (headers/footers) stripped before chunking
3. **Consistent embedding at ingest and query** — same model, same version, validated at startup with a hard assertion
4. **Vector similarity retrieval (top-20 candidates)** — ChromaDB cosine similarity search, domain metadata filter support
5. **Cross-encoder reranking to top-5** — cross-encoder/ms-marco-MiniLM-L-6-v2 via sentence-transformers; profile CPU latency at k=20 before shipping
6. **Context construction with numbered source labels** — [SOURCE 1]: ... format; token budget enforced before Claude API call
7. **Claude generation (temperature=0, pinned model version)** — factual RAG is not a creative task; pin model name in config so eval scores remain comparable
8. **Citation generation** — parse [SOURCE N] references from Claude output, map to retrieved chunk metadata, validate citation numbers within chunk count
9. **Response validation and fallback** — retrieval confidence + faithfulness threshold; "I don't know" response when evidence is insufficient; fallback_triggered flag
10. **JWT authentication with user and admin roles** — access token (15 min) + refresh token (7 days); JWT secret from environment variable only
11. **Per-request telemetry** — stage-level latency (embed_ms, retrieve_ms, rerank_ms, generate_ms, eval_ms), token usage, retrieval scores, error classification
12. **LLM-as-judge evaluation (async, non-blocking)** — faithfulness, relevance, correctness scored via Claude with explicit rubrics; dispatched via BackgroundTasks after response is returned
13. **Admin dashboard** — total requests, average latency, failure rate, recent request log with drill-down, document list with ingestion status
14. **Docker Compose single-command startup** — health checks with service_healthy condition to prevent startup race conditions

---

## Differentiating Features (Production-Grade vs Demo)

1. **Cross-encoder reranking** — Most basic RAG uses vector similarity alone. Reranking over 20 initial candidates improves NDCG@5 by 15-25%. Critical for policy docs where user phrasing varies from document language.

2. **Grounding-enforced fallback gate** — The system explicitly refuses to answer when evidence is insufficient. Most RAG systems answer anyway. This is trust infrastructure for HR/policy/compliance use cases where a wrong answer has real consequences.

3. **Structured citation output** — Machine-readable citations (document, page, section). The metadata chain from ingest through context construction to response is the feature — requires ingestion metadata → chunk metadata → source labels in prompt → citation parsing → bounds validation.

4. **Per-request faithfulness/relevance/correctness evaluation** — Every response is evaluated asynchronously. Enables quality regression detection and surfacing low-quality responses before they appear in user complaints.

5. **Stage-level telemetry** — Per-stage timing stored per request enables bottleneck identification that total latency alone cannot provide.

6. **Request explorer in admin dashboard** — Drill into any request: see exact chunks retrieved, reranking scores, answer, and evaluation scores. Critical for auditing a multi-domain system.

7. **Domain-tagged document metadata** — Prevents cross-domain retrieval noise (HR question should not return API docs).

---

## Critical Pitfalls (Top 5)

### 1. Embedding Model Mismatch (Silent, Catastrophic)

Changing the embedding model after documents are ingested — including minor version bumps — invalidates all stored vectors. Retrieval collapses silently; the system returns 200 OK with wrong content. **Prevention:** Pin sentence-transformers to an exact version in requirements.txt. Store the model name in ChromaDB collection metadata at creation. Assert on startup that the configured model matches stored metadata. Add a smoke test: embed a known query, verify top chunk matches expected content.

### 2. ChromaDB Persistence Misconfiguration (Silent Data Loss)

Using `chromadb.Client()` instead of `chromadb.PersistentClient(path=...)`, or forgetting the Docker volume mount, causes data loss on container restart with no error. **Prevention:** Use `chromadb.PersistentClient(path="/data/chroma")` exclusively. Declare a named Docker volume (`chroma_data:/data/chroma`). Add startup health check: if collection exists but `count()` returns 0, raise an alert. Test persistence: ingest a document, restart the container, verify it survives.

### 3. PostgreSQL/ChromaDB State Divergence (Orphaned Records)

A crash between the PostgreSQL write and the ChromaDB write leaves a document record with no corresponding vectors. **Prevention:** Use an explicit ingestion state machine (pending → chunking → embedding → indexing → complete). Only write complete after both are confirmed. Build a reconciliation check: count PostgreSQL chunk records vs ChromaDB vector count per document at startup; alert on mismatches.

### 4. LLM-as-Judge Score Inflation (Evaluation Becomes Meaningless)

Claude evaluating its own outputs without a calibrated rubric produces consistently high scores (0.8-0.9) regardless of actual quality. Fallback thresholds never trigger. **Prevention:** Write explicit rubrics with score anchors in the judge prompt. Include calibration examples (clearly faithful, clearly unfaithful, borderline). Evaluate at temperature=0. Build a manually labeled golden dataset (20-30 examples) and benchmark the judge against it.

### 5. Citation Generation Without Source Tracking (Trust Failure)

Building generation first and adding citations as an afterthought produces citations attached to wrong chunks. Worse than no citations. **Prevention:** Number each chunk in the prompt (`[SOURCE 1]: ...`). Instruct Claude to reference sources by number only. Parse source references at the response layer, map to chunk metadata, validate that every cited source number is within the chunk count.

---

## Build Order (Recommended Phase Sequence)

### Phase 1: Foundation
**Delivers:** Runnable Docker Compose environment with schema and auth that every subsequent phase builds on.
- PostgreSQL schema + Alembic migrations
- Docker Compose skeleton with health checks and named volumes
- JWT auth middleware (FastAPI Depends; startup secret validation; test all failure modes)
- FastAPI lifespan events (model loading, ChromaDB collection init)

**Must avoid:** Docker Compose startup race conditions; JWT secret hardcoded; ChromaDB persistence misconfiguration.

---

### Phase 2: Ingestion Pipeline
**Delivers:** Documents uploaded via admin UI are parsed, chunked, embedded, and stored. Knowledge base has content.
- Document parser (pdfplumber, python-docx, stdlib); boilerplate stripping; garbage-text validation
- Chunker (512 tokens, 64-token overlap, sentence-boundary-aware, metadata stamping per chunk)
- Embedding Service singleton (sentence-transformers; loaded once at startup; startup model validation assertion)
- ChromaDB collection initialization (cosine similarity, manual embedding mode, volume verified)
- Ingestion Service (state machine: pending → chunking → embedding → indexing → complete)
- POST /admin/documents endpoint (admin role required; SHA-256 deduplication)
- Admin document list UI (upload drag-and-drop, ingestion status)

**Smoke test:** Upload a real document from the actual corpus, verify chunk count in PostgreSQL matches vector count in ChromaDB, query a known fact directly via ChromaDB.

---

### Phase 3: Query Pipeline
**Delivers:** Users can ask questions and receive cited, grounded answers. System refuses low-confidence questions.
- Query Processing (normalization, search query extraction)
- Retrieval Service (ChromaDB top-20, domain filter support)
- Reranker (cross-encoder scoring all 20 pairs; select top-5; CPU latency profiled at k=20; hard timeout with raw-similarity fallback)
- Context Builder (numbered source labels, token budget enforcement, system prompt)
- LLM Service (Claude API, temperature=0, pinned model, token usage + latency captured)
- Citation Generator (parse [SOURCE N] references, map to chunk metadata, validate bounds, deduplicate)
- Response Validator + Fallback Handler (retrieval confidence threshold + faithfulness threshold)
- POST /query endpoint (JWT protected, request_id generation)
- Query UI (React chat interface, citation display)

**Build within phase:** Test each stage in isolation with hardcoded inputs before composing. Full smoke test (upload doc, query it, get cited answer) before adding telemetry or evaluation.

---

### Phase 4: Telemetry
**Delivers:** Every query request is instrumented with per-stage timing, token usage, and error classification.
- TelemetryContext object (per-request dict, passed as parameter — no global state)
- Stage timing wrappers (embed_ms, retrieve_ms, rerank_ms, generate_ms)
- ai_requests INSERT at end of request (synchronous, before response returned)
- Failure reason classification enum
- GET /admin/stats endpoint

**Why after query pipeline:** Telemetry is additive; adding to a working pipeline avoids noise during query debugging.

---

### Phase 5: Evaluation Engine
**Delivers:** Every response is asynchronously scored for faithfulness, relevance, and correctness.
- Evaluation Engine (three LLM-as-judge prompts with explicit rubrics and calibration examples; temperature=0)
- evaluations INSERT (request_id FK, three scores)
- FastAPI BackgroundTasks dispatch in query endpoint (fire-and-forget after response returned)
- GET /admin/evaluations endpoint
- Golden dataset (20-30 manually labeled examples; benchmark judge; track score distribution)

**Note:** Batch all three judge calls into one structured Claude call to minimize latency overhead.

---

### Phase 6: Admin Dashboard + Frontend Polish
**Delivers:** Admins can monitor system health, inspect individual requests, and manage documents.
- Admin dashboard stats (total requests, avg latency, failure rate, avg evaluation scores)
- Recent request log (last 50, paginated)
- Per-request detail view (question → chunks → reranking scores → answer → eval scores)
- Document management UI
- Frontend polish (loading states, error boundaries)

**Note:** Frontend can be built in parallel with Phases 3-5 if the API contract is agreed upon early. Mock API responses during frontend development to avoid blocking.

---

## Key Decisions Resolved

| Decision | Resolution | Rationale |
|----------|-----------|-----------|
| Embeddings provider | sentence-transformers, all-MiniLM-L6-v2 | Anthropic has no embeddings API; local model = no vendor dependency, no network latency per chunk, sufficient quality at < 1K docs |
| JWT library | PyJWT 2.13.0 | python-jose has CVEs and slower maintenance cadence |
| Password hashing | bcrypt 5.0.0 directly | passlib maintenance has slowed; bcrypt direct is cleaner |
| Async DB driver | asyncpg | psycopg2 is synchronous; asyncpg required for SQLAlchemy 2.0 async mode |
| Evaluation framework | Custom LLM-as-judge prompts (not RAGAS) | RAGAS requires OpenAI embeddings by default; custom Claude prompts give same signal with full rubric control |
| Frontend state management | Zustand | Auth + UI state is minimal; Redux is overkill |
| Build tool | Vite | CRA is unmaintained since 2023 |
| Evaluation async strategy | FastAPI BackgroundTasks | Celery adds significant operational overhead; BackgroundTasks is sufficient at v1 throughput |
| ChromaDB constructor | PersistentClient(path=...) | Settings-based constructor behavior changed in v0.4+; PersistentClient is the correct v1.x API |
| ChromaDB embedding mode | Manual embedding (not built-in embedding function) | Embedding Service must be the single source of truth for vectors to prevent model drift between ingest and query |

---

## Conflicts Resolved

**ARCHITECTURE.md vs STACK.md on ChromaDB embedding approach:**
ARCHITECTURE.md recommends passing embeddings manually. STACK.md shows using `SentenceTransformerEmbeddingFunction` passed to the collection.

**Resolution: Use manual embedding.** The Embedding Service singleton must be the authoritative vector producer. Letting ChromaDB call the model separately creates a second code path that could diverge in model version or configuration. Pass resulting vectors directly to `collection.add(embeddings=[...])`.

**FEATURES.md vs ARCHITECTURE.md on chunking overlap:**
FEATURES.md recommends 50-token overlap. ARCHITECTURE.md specifies 64-token overlap.

**Resolution: 64-token overlap.** The difference is marginal. 64 tokens provides slightly better boundary coverage for longer-sentence policy documents. Tune empirically after ingesting real corpus.

---

## Open Questions Remaining

1. **Which Claude model to pin for generation?** Research uses examples but the actual model for v1 (claude-3-5-sonnet vs claude-opus-4 vs other) needs a team decision. Affects cost, latency, and quality.

2. **What fallback thresholds to start with?** Suggested defaults: retrieval similarity < 0.35 and faithfulness < 0.5. These need empirical calibration against the actual document corpus during Phase 3 or 5.

3. **What document domains to tag initially?** The domain taxonomy (e.g., hr, product, technical) affects the ChromaDB metadata schema and cannot be changed without re-ingestion. Needs product input before Phase 2.

4. **How are admin accounts created?** Options: seeded in Alembic migration, one-time CLI command, or environment variable bootstrap. Needs decision before Phase 1 ships.

5. **What is the acceptable p95 query latency target?** Total pipeline may be 3-6 seconds on CPU. If the team has an SLA target, it needs to be stated before Phase 3 performance testing.

---

## Research Flags for Phase Planning

| Phase | Research Needed? | Notes |
|-------|-----------------|-------|
| Phase 1: Foundation | No | PostgreSQL, FastAPI, JWT, Docker Compose are well-documented with no ambiguity |
| Phase 2: Ingestion | Minimal | PDF quality is corpus-dependent; test with real documents early |
| Phase 3: Query Pipeline | No | Cross-encoder reranking and RAG pipeline patterns are well-established |
| Phase 4: Telemetry | No | Standard instrumentation pattern; no novel decisions |
| Phase 5: Evaluation | Yes — judge prompt calibration | LLM-as-judge prompt engineering requires iteration; plan time for calibration against a golden dataset |
| Phase 6: Frontend | No | React, Tailwind v4, TanStack Query, Recharts are all well-documented |

---

## Confidence Assessment

| Area | Confidence | Basis |
|------|------------|-------|
| Stack versions | HIGH | Verified from PyPI JSON API and npm registry 2026-06-14 |
| Embeddings approach | HIGH | Anthropic confirmed no embeddings API; sentence-transformers + ChromaDB integration verified |
| Architecture patterns | HIGH | RAG with reranking, async evaluation, and telemetry are stable production patterns |
| Chunking defaults | MEDIUM | 512/64-token defaults are empirically supported starting points; optimal values are corpus-dependent |
| Pitfalls | HIGH | Core pitfalls are well-documented production failure modes with strong evidence |
| LLM-as-judge rubrics | MEDIUM | Score inflation is documented; specific rubric wording requires iteration against actual corpus |
| Fallback thresholds | LOW | Suggested values are reasonable starting points with no empirical basis for this corpus |

---

## Do Not Use

| Library | Reason |
|---------|--------|
| LangChain / LlamaIndex | Heavy abstractions; debugging pipeline failures becomes opaque |
| RAGAS | Requires OpenAI embeddings by default; opaque scoring; harder to tune than custom prompts |
| python-jose | CVE history; use PyJWT |
| passlib | Low maintenance; use bcrypt directly |
| Create React App | Unmaintained since 2023 |
| Prometheus + Grafana | Out of scope v1; lightweight custom dashboard is sufficient |
| pgvector | Not needed; ChromaDB is the vector store |
| Redis | Not needed at < 1K doc scale |
| Celery | FastAPI BackgroundTasks is sufficient for v1 evaluation throughput |
| chromadb.EphemeralClient | Never use in production code paths |

---

## Sources

- STACK.md (PyPI JSON API + npm registry, 2026-06-14) — HIGH confidence
- FEATURES.md (project spec + domain knowledge) — MEDIUM-HIGH confidence
- ARCHITECTURE.md (project base.md + system_overview.png + production RAG patterns) — HIGH confidence
- PITFALLS.md (ChromaDB official behavior + FastAPI production patterns + Zheng et al. 2023 on LLM-as-judge) — HIGH confidence
- PROJECT.md (authoritative project spec) — HIGH confidence
