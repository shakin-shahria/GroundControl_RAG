# Requirements: GroundControl_RAG

**Defined:** 2026-06-14
**Core Value:** Users get grounded, cited answers from company knowledge — and the system tells you when it doesn't know rather than making things up.

## v1 Requirements

### Foundation

- [ ] **FOUND-01**: System has a PostgreSQL schema with tables for users, documents, document_chunks, ai_requests, and evaluations
- [ ] **FOUND-02**: All services (FastAPI, ChromaDB, PostgreSQL) run via a single `docker-compose up` with health checks and named volumes for data persistence
- [ ] **FOUND-03**: User can register with email and password, log in, and access protected routes via JWT (PyJWT + bcrypt)
- [ ] **FOUND-04**: Each service exposes a `GET /health` endpoint used by Docker Compose `depends_on` for startup ordering

### Document Ingestion

- [ ] **ING-01**: Admin can upload PDF, DOCX, and TXT files; system extracts text with page number and section header metadata
- [ ] **ING-02**: Uploaded documents are chunked at 512 tokens with 50-token overlap, sentence-boundary aware, and tagged with domain (hr / product / technical)
- [ ] **ING-03**: Chunks are embedded with `all-MiniLM-L6-v2` (sentence-transformers) and stored in ChromaDB with numbered chunk IDs for citation architecture (`[SOURCE N]`)
- [ ] **ING-04**: Admin can upload documents via a React drag-and-drop UI that calls `POST /documents`; ingestion runs as a FastAPI BackgroundTask

### Query Pipeline

- [ ] **QUERY-01**: User query is embedded with the same `all-MiniLM-L6-v2` model (startup assertion enforces consistency) and top 20 chunks are retrieved from ChromaDB
- [ ] **QUERY-02**: Retrieved 20 chunks are reranked by a cross-encoder (`ms-marco-MiniLM-L-6-v2`); chunks scoring below 0.35 trigger the fallback handler instead of generation
- [ ] **QUERY-03**: Context is assembled as: system prompt + numbered chunks + user question, capped at 80K tokens to stay within Claude's context window
- [ ] **QUERY-04**: Claude generates a grounded response referencing `[SOURCE N]` markers; system parses citations and maps them to chunk metadata (document name, page, section)

### Reliability

- [ ] **REL-01**: Response validator checks that the generated answer is grounded in the retrieved context before returning it to the user
- [ ] **REL-02**: Fallback handler returns a graceful "I could not find reliable information on this topic" message when retrieval confidence is low or grounding check fails
- [ ] **REL-03**: Per-request telemetry is captured — latency for each stage (embed, retrieve, rerank, generate, validate), token usage, model version, and errors — stored in the `ai_requests` table
- [ ] **REL-04**: Each completed request triggers an async evaluation (FastAPI BackgroundTasks) that scores faithfulness, relevance, and correctness via Claude LLM-as-judge with a calibrated rubric; scores are stored in the `evaluations` table

### Admin Dashboard

- [ ] **DASH-01**: Admin dashboard shows a stats overview: total requests, average latency, failure rate, and average evaluation scores
- [ ] **DASH-02**: Admin can browse a paginated request log showing question, generated answer, source citations, and evaluation scores for each past query

### Query Interface

- [ ] **UI-01**: User can submit a natural language question and see the generated answer in a chat-style interface
- [ ] **UI-02**: Answer displays inline source citations (document name, page number, section) derived from `[SOURCE N]` references in the response

## v2 Requirements

### Admin Dashboard Enhancements

- **DASH-03**: Document manager — list all ingested documents with delete capability
- **DASH-04**: Model comparison view — side-by-side accuracy and latency metrics across Claude model versions

### Query Interface Enhancements

- **UI-03**: Per-answer confidence indicator showing faithfulness and relevance scores to the user
- **UI-04**: Per-user query history — list of past questions with answers

### Retrieval Improvements

- **RETR-01**: Hybrid search combining BM25 keyword search with vector similarity for improved recall on exact-term queries
- **RETR-02**: Domain-scoped filtering in the query UI — user can restrict search to a specific document domain

### Observability

- **OBS-01**: Prometheus metrics export + Grafana dashboard for production monitoring
- **OBS-02**: Alerting on high failure rate or latency regression

### Fallback Expansion

- **FALL-01**: Human escalation routing — low-confidence queries flagged and queued for human review
- **FALL-02**: Clarification request flow — system asks the user a clarifying question before attempting retrieval

## Out of Scope

| Feature | Reason |
|---------|--------|
| LangChain / LlamaIndex | Frameworks obscure custom pipeline; debugging becomes harder with no benefit at this scale |
| Anthropic native embeddings | Anthropic does not offer an embeddings API; sentence-transformers is the local alternative |
| Celery / Redis task queue | FastAPI BackgroundTasks sufficient for v1 volume; Celery adds operational overhead |
| python-jose for JWT | Active CVE history; PyJWT is the maintained alternative |
| Multi-modal documents (images, video) | Significant parser complexity; text-only corpus sufficient for v1 |
| Model fine-tuning / retraining | Operational overhead; RAG pipeline quality improvement is the v1 priority |
| Agentic / tool-use workflows | Out of scope for a reliability-focused RAG platform in v1 |
| Multi-tenancy | Single-tenant v1; org isolation is a v2 concern |
| RAGAS evaluation framework | External dependency; custom LLM-as-judge metrics are sufficient and more tunable |

## Traceability

| Requirement | Phase | Status |
|-------------|-------|--------|
| FOUND-01 | Phase 1 | Pending |
| FOUND-02 | Phase 1 | Pending |
| FOUND-03 | Phase 1 | Pending |
| FOUND-04 | Phase 1 | Pending |
| ING-01 | Phase 2 | Pending |
| ING-02 | Phase 2 | Pending |
| ING-03 | Phase 2 | Pending |
| ING-04 | Phase 2 | Pending |
| QUERY-01 | Phase 3 | Pending |
| QUERY-02 | Phase 3 | Pending |
| QUERY-03 | Phase 3 | Pending |
| QUERY-04 | Phase 3 | Pending |
| REL-01 | Phase 3 | Pending |
| REL-02 | Phase 3 | Pending |
| REL-03 | Phase 4 | Pending |
| REL-04 | Phase 4 | Pending |
| DASH-01 | Phase 5 | Pending |
| DASH-02 | Phase 5 | Pending |
| UI-01 | Phase 5 | Pending |
| UI-02 | Phase 5 | Pending |

**Coverage:**
- v1 requirements: 20 total
- Mapped to phases: 20
- Unmapped: 0

---
*Requirements defined: 2026-06-14*
*Last updated: 2026-06-14 after roadmap creation — traceability populated*
