# Roadmap: GroundControl_RAG

**Milestone:** v1 — Production RAG Platform
**Created:** 2026-06-14
**Granularity:** Standard
**Requirements covered:** 20/20

---

## Phases

- [ ] **Phase 1: Foundation** — Runnable environment with schema, auth, and health checks that every subsequent phase builds on
- [ ] **Phase 2: Document Ingestion Pipeline** — Documents are uploaded, parsed, chunked, embedded, and stored; knowledge base has content
- [ ] **Phase 3: Query Pipeline** — Users ask natural language questions and receive grounded, cited answers; system refuses low-confidence queries
- [ ] **Phase 4: Telemetry and Evaluation** — Every request is instrumented with per-stage timing and scored asynchronously for faithfulness, relevance, and correctness
- [ ] **Phase 5: Frontend — Admin Dashboard and Query UI** — Admins can monitor system health and inspect requests; users interact via a chat interface with inline citations

---

## Phase Details

### Phase 1: Foundation
**Goal**: Developers can run all services with a single `docker-compose up`, register a user, and authenticate via JWT — the infrastructure every subsequent phase depends on is stable and verified
**Depends on**: Nothing (first phase)
**Requirements**: FOUND-01, FOUND-02, FOUND-03, FOUND-04
**Success Criteria** (what must be TRUE):
  1. Running `docker-compose up` brings FastAPI, PostgreSQL, and ChromaDB online with health checks passing and no startup race conditions
  2. A new user can register with email and password; credentials are stored with bcrypt hashing in the PostgreSQL `users` table
  3. A registered user can log in and receive a JWT access token; a protected route returns 401 without a valid token and 200 with one
  4. Every service exposes `GET /health` and Docker Compose `depends_on: service_healthy` prevents dependent services from starting until their upstream is ready
  5. PostgreSQL schema has all five tables (users, documents, document_chunks, ai_requests, evaluations) created via Alembic migration on first boot
**Plans**: 5 plans
Plans:
- [ ] 01-01-PLAN.md — Project scaffold: Docker Compose, Dockerfile, requirements.txt, directory skeleton, alembic.ini
- [ ] 01-02-PLAN.md — Core modules: config.py, database.py, chroma.py, embedding.py, security.py, main.py skeleton
- [ ] 01-03-PLAN.md — ORM models (all 5 tables) + Alembic async env.py + initial migration
- [ ] 01-04-PLAN.md — Auth routes (register/login), deps.py, user_service.py, router wiring, main.py finalized
- [ ] 01-05-PLAN.md — Integration test suite: conftest.py, test_auth.py, test_health.py, test_db.py, pyproject.toml

### Phase 2: Document Ingestion Pipeline
**Goal**: An admin can upload a PDF, DOCX, or TXT file and verify that its content is chunked, embedded, and retrievable in ChromaDB — the knowledge base has content before the query pipeline is built
**Depends on**: Phase 1
**Requirements**: ING-01, ING-02, ING-03, ING-04
**Success Criteria** (what must be TRUE):
  1. Admin can drag and drop a document in the React upload UI; the file is sent to `POST /documents` and ingestion runs as a background task without blocking the response
  2. Uploaded documents have text extracted with page number and section header metadata; PDF, DOCX, and TXT formats each produce valid output
  3. Extracted text is chunked at 512 tokens with 64-token overlap, sentence-boundary-aware; each chunk is tagged with domain, doc_id, filename, page, section, and chunk index
  4. Chunks are embedded with `all-MiniLM-L6-v2` and stored in ChromaDB with numbered chunk IDs; the vector count in ChromaDB matches the chunk count in PostgreSQL for every uploaded document
  5. Container restart does not cause data loss: documents uploaded before restart are still retrievable in ChromaDB after restart
**Plans**: TBD
**UI hint**: yes

### Phase 3: Query Pipeline
**Goal**: An authenticated user can submit a natural language question and receive a grounded answer with inline citations, or a graceful refusal when evidence is insufficient
**Depends on**: Phase 2
**Requirements**: QUERY-01, QUERY-02, QUERY-03, QUERY-04, REL-01, REL-02
**Success Criteria** (what must be TRUE):
  1. A user query is embedded with the same `all-MiniLM-L6-v2` model used at ingestion (startup assertion enforces consistency); the top 20 candidate chunks are retrieved from ChromaDB
  2. Retrieved 20 chunks are reranked by a cross-encoder; the top 5 scoring chunks are passed to context assembly; chunks scoring below 0.35 trigger the fallback handler instead of generation
  3. Claude generates a response that references `[SOURCE N]` markers; the API response includes parsed citations with document name, page number, and section for each reference
  4. The response validator checks that the generated answer is grounded in the retrieved context before returning; a response that fails grounding check triggers the fallback path
  5. When retrieval confidence is low or grounding fails, the endpoint returns a graceful "I could not find reliable information on this topic" message with `fallback_triggered: true` in the response
**Plans**: TBD

### Phase 4: Telemetry and Evaluation
**Goal**: Every completed query request is instrumented with per-stage timing and error classification, and asynchronously scored for faithfulness, relevance, and correctness — quality is measurable
**Depends on**: Phase 3
**Requirements**: REL-03, REL-04
**Success Criteria** (what must be TRUE):
  1. Every request record in `ai_requests` contains stage-level latency fields (embed_ms, retrieve_ms, rerank_ms, generate_ms, validate_ms), token usage, model version, and error classification if applicable
  2. After a query response is returned to the user, an async evaluation fires via FastAPI BackgroundTasks; the `evaluations` table receives a row with faithfulness, relevance, and correctness scores
  3. LLM-as-judge evaluation uses explicit rubrics with score anchors and calibration examples at temperature=0; score distribution across a 20-example golden dataset does not cluster at 0.8+ regardless of answer quality
**Plans**: TBD

### Phase 5: Frontend — Admin Dashboard and Query UI
**Goal**: Admins can monitor system health and drill into individual requests via a dashboard; users can submit questions and read grounded answers with inline citations in a chat-style interface
**Depends on**: Phase 4
**Requirements**: DASH-01, DASH-02, UI-01, UI-02
**Success Criteria** (what must be TRUE):
  1. Admin dashboard shows total requests, average latency, failure rate, and average evaluation scores populated from live data
  2. Admin can browse a paginated request log; clicking any entry shows the full question, generated answer, source citations, and evaluation scores for that request
  3. User can type a natural language question in a chat-style interface and see the generated answer without page reload
  4. Answer display includes inline source citations showing document name, page number, and section derived from `[SOURCE N]` references; citations are human-readable and link to the correct chunk metadata
**Plans**: TBD
**UI hint**: yes

---

## Progress Table

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. Foundation | 0/5 | Planned | - |
| 2. Document Ingestion Pipeline | 0/? | Not started | - |
| 3. Query Pipeline | 0/? | Not started | - |
| 4. Telemetry and Evaluation | 0/? | Not started | - |
| 5. Frontend — Admin Dashboard and Query UI | 0/? | Not started | - |

---

## Coverage Map

| Requirement | Phase |
|-------------|-------|
| FOUND-01 | Phase 1 |
| FOUND-02 | Phase 1 |
| FOUND-03 | Phase 1 |
| FOUND-04 | Phase 1 |
| ING-01 | Phase 2 |
| ING-02 | Phase 2 |
| ING-03 | Phase 2 |
| ING-04 | Phase 2 |
| QUERY-01 | Phase 3 |
| QUERY-02 | Phase 3 |
| QUERY-03 | Phase 3 |
| QUERY-04 | Phase 3 |
| REL-01 | Phase 3 |
| REL-02 | Phase 3 |
| REL-03 | Phase 4 |
| REL-04 | Phase 4 |
| DASH-01 | Phase 5 |
| DASH-02 | Phase 5 |
| UI-01 | Phase 5 |
| UI-02 | Phase 5 |

**Coverage: 20/20 v1 requirements mapped. No orphans.**

---
*Roadmap created: 2026-06-14*
*Last updated: 2026-06-15 — Phase 1 planned: 5 plans across 5 waves*
