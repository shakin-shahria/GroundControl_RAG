# Architecture Patterns: GroundControl_RAG

**Domain:** Production RAG Platform
**Researched:** 2026-06-14
**Sources:** Project base.md, system_overview.png, PROJECT.md, production RAG architecture patterns

---

## Components

### Component Map

| Component | Layer | Responsibility | Communicates With |
|-----------|-------|---------------|-------------------|
| React Frontend | Client | Query UI, admin dashboard, file upload drag-and-drop | FastAPI via REST/JSON |
| FastAPI Gateway | API | Auth enforcement, request ID generation, routing, telemetry capture | Frontend, all internal services, PostgreSQL |
| Auth Middleware | API Cross-Cut | JWT validation, role check (user vs admin) | FastAPI (every request) |
| Ingestion Service | Offline Pipeline | Parse PDF/DOCX/TXT, chunk text, call embedding model, write to ChromaDB + PostgreSQL | Embedding Service, ChromaDB, PostgreSQL |
| Query Processing | Online Pipeline | Normalize query text, extract intent, prepare search query | Embedding Service |
| Embedding Service | Shared | Convert text to vectors (sentence-transformers `all-MiniLM-L6-v2`) | Ingestion Service, Query Processing |
| ChromaDB | Storage | Vector persistence, similarity search (cosine), metadata filtering | Ingestion Service, Retrieval Service |
| Retrieval Service | Online Pipeline | Run similarity search, return top-N chunks with metadata | ChromaDB, Reranker |
| Reranker | Online Pipeline | Re-score top-N chunks by query relevance, select top-5 | Retrieval Service, Context Builder |
| Context Builder | Online Pipeline | Assemble: system prompt + reranked chunks + user question → prompt string | Reranker, LLM Service |
| LLM Service | Online Pipeline | Call Claude API (Anthropic SDK), capture token counts + latency | Context Builder, Citation Generator |
| Citation Generator | Online Pipeline | Parse LLM output, attach source document/page/section metadata | LLM Service, Response Validator |
| Response Validator | Online Pipeline | Check grounding score + confidence threshold; route to fallback or return | Citation Generator, Fallback Handler |
| Fallback Handler | Online Pipeline | Return "I don't know" or clarification prompt when confidence is low | Response Validator |
| Evaluation Engine | Async | LLM-as-judge scoring of faithfulness, relevance, correctness; write to `evaluations` table | PostgreSQL (read request), Claude API |
| Telemetry Collector | Cross-Cut | Capture per-stage latency, token usage, retrieval scores, errors; write to `ai_requests` | FastAPI, all pipeline stages |
| PostgreSQL | Storage | Users, documents, document_chunks, ai_requests, evaluations tables | Ingestion Service, FastAPI, Evaluation Engine |
| Admin Dashboard | Client | Show totals, latency charts, failure rate, request explorer | FastAPI (read endpoints) |

### Component Boundaries — Key Rules

1. **ChromaDB is write-only from Ingestion; read-only from Retrieval.** No other component touches ChromaDB directly. This isolates vector storage behind two clear service boundaries.

2. **The Embedding Service is shared but stateless.** Both Ingestion and Query Processing call it identically. This prevents embedding drift (same model weights produce comparable vectors for stored and queried text).

3. **Evaluation Engine has NO write path into the query response.** It consumes the completed `ai_request` record asynchronously. The query pipeline does not wait for evaluation scores before returning to the user.

4. **FastAPI Gateway is the single entry point.** The frontend never calls ChromaDB, PostgreSQL, or Claude directly. All traffic routes through FastAPI, which enforces JWT auth before passing requests to pipeline services.

5. **Telemetry is cross-cutting, not inline.** Each pipeline stage emits timing + metadata to a telemetry context object that is committed to PostgreSQL at the end of the request, not inline during the pipeline. This prevents telemetry writes from adding to user-facing latency.

6. **The Reranker is CPU-only in v1.** Cross-encoder reranking with a small model (e.g., `cross-encoder/ms-marco-MiniLM-L-6-v2`) runs in-process inside FastAPI. No separate microservice needed at < 1K doc scale. Extract to a worker if latency exceeds SLA.

---

## Data Flows

### Pipeline 1: Document Ingestion (Offline)

```
Admin uploads file via drag-and-drop UI
    |
    v
POST /admin/documents  [multipart/form-data]
    |
    v
FastAPI: JWT check (admin role required)
    |
    v
Ingestion Service
    |
    +-- Parse file (PyMuPDF for PDF, python-docx for DOCX, stdlib for TXT)
    |       → raw text + page numbers + section headers
    |
    +-- Chunk text
    |       Strategy: sliding window, 512 tokens, 64-token overlap
    |       → list of (chunk_text, page_number, section, doc_id)
    |
    +-- Embed chunks
    |       Call Embedding Service (sentence-transformers)
    |       → list of (chunk_text, embedding_vector[384], metadata)
    |
    +-- Write to ChromaDB
    |       Collection: "documents_v1"
    |       Document ID: chunk_uuid
    |       Embedding: vector[384]
    |       Metadata: {doc_id, filename, page, section, chunk_index}
    |
    +-- Write to PostgreSQL
            INSERT INTO documents (id, filename, source, uploaded_at)
            INSERT INTO document_chunks (id, document_id, content, embedding_id, metadata)
    |
    v
Return: {document_id, chunk_count, status: "ingested"}
```

**Write order within ingestion:** File save → Parse → Chunk → Embed → ChromaDB write → PostgreSQL write. PostgreSQL write is last so `embedding_id` references the ChromaDB-assigned ID.

---

### Pipeline 2: Query (Online, synchronous, user-facing)

```
User types question in frontend
    |
    v
POST /query  {question: "..."}
    |
    v
FastAPI: JWT check, generate request_id + trace_id, start telemetry context
    |
    v
Query Processing
    → normalize text, extract search terms
    → output: search_query string
    |
    v
Embedding Service
    → embed search_query → vector[384]
    |
    v
Retrieval Service
    → ChromaDB: query(embedding=vector, n_results=20, include=["documents","metadatas","distances"])
    → returns: [{chunk_text, metadata, distance_score}, ...] × 20
    |
    v
Reranker
    → cross-encoder score each of 20 chunks against original question
    → select top 5 by rerank score
    → output: [{chunk_text, metadata, rerank_score}, ...] × 5
    |
    v
Context Builder
    → assemble prompt:
        SYSTEM: "You are a company knowledge assistant. Answer only from context below."
        CONTEXT: [chunk_1]...[chunk_5] with source labels [DOC-1]...[DOC-5]
        QUESTION: "{user_question}"
    → record: context_token_count
    |
    v
LLM Service (Claude API)
    → anthropic.messages.create(model="claude-3-5-sonnet", messages=[...])
    → capture: response_text, input_tokens, output_tokens, latency_ms
    |
    v
Citation Generator
    → parse response_text for [DOC-N] references
    → resolve references to {filename, page, section} from reranked metadata
    → output: {answer: "...", sources: [{document, page, section}, ...]}
    |
    v
Response Validator
    → grounding check: are cited sources in context? (string match)
    → confidence check: max(rerank_scores) vs threshold (0.4 default)
    |
    +-- confidence >= threshold → return cited answer
    |
    +-- confidence < threshold → Fallback Handler
            → return {answer: "I could not find reliable information on this topic.", sources: []}
    |
    v
Commit telemetry to PostgreSQL (ai_requests row)
    → {request_id, user_id, question, response, model, input_tokens, output_tokens,
       total_latency_ms, embedding_ms, retrieval_ms, rerank_ms, llm_ms, 
       retrieval_scores: [top-5 scores], fallback_triggered: bool}
    |
    v
Dispatch evaluation task to background worker (fire-and-forget)
    |
    v
Return response to user
```

**Critical timing constraint:** The evaluation dispatch happens AFTER the telemetry commit but BEFORE `return`. Use FastAPI `BackgroundTasks` to fire the evaluation job without awaiting it. The response returns to the user in ~1-2 seconds; evaluation completes within ~3-5 additional seconds asynchronously.

---

### Pipeline 3: Evaluation (Async, non-blocking)

```
BackgroundTask receives: {request_id, question, context_chunks, answer}
    |
    v
Evaluation Engine
    |
    +-- Faithfulness check (LLM-as-judge)
    |       Prompt Claude: "Does the answer [answer] only contain claims 
    |       supported by this context [context]? Score 0.0-1.0"
    |       → faithfulness_score: float
    |
    +-- Relevance check (LLM-as-judge)
    |       Prompt Claude: "Does the answer address the question [question]? Score 0.0-1.0"
    |       → relevance_score: float
    |
    +-- Correctness check
    |       Either: keyword overlap heuristic (fast, no LLM call)
    |       Or: LLM judge (slower, more accurate)
    |       → correctness_score: float
    |
    v
INSERT INTO evaluations (request_id, faithfulness_score, relevance_score, correctness_score)
    |
    v
Admin dashboard reads evaluation scores on next poll/load
```

**Do not block on evaluation.** Three Claude API calls in sequence adds ~3-6 seconds to the hot path. FastAPI `BackgroundTasks` runs after response is sent. If a task fails, log the failure — do not retry synchronously.

---

### Pipeline 4: Observability (Cross-cutting)

```
Each pipeline stage: record (stage_name, start_time, end_time, metadata)
    → stored in in-request telemetry context dict (in-memory, per request)
    |
    v
End of request: serialize telemetry context → INSERT INTO ai_requests
    |
    v
Admin dashboard polls GET /admin/stats
    → aggregate queries on ai_requests + evaluations tables
    → return: {total_requests, avg_latency_ms, failure_rate, avg_faithfulness}
```

---

## ChromaDB Collection Design

Single collection: `documents_v1`

```python
collection = chroma_client.get_or_create_collection(
    name="documents_v1",
    metadata={"hnsw:space": "cosine"},  # cosine similarity for text embeddings
    embedding_function=None              # pass embeddings manually, don't let ChromaDB call an embedding model
)
```

**Why manual embeddings (not ChromaDB's built-in embedding function):**
- The same Embedding Service must produce vectors for both ingestion and query time.
- Letting ChromaDB call the model separately risks using a different model version or configuration.
- Manual embedding keeps the Embedding Service as the single source of truth.

**Metadata schema per chunk:**
```json
{
  "doc_id": "uuid",
  "filename": "refund_policy.pdf",
  "page": 4,
  "section": "Returns Policy",
  "chunk_index": 12,
  "char_count": 487
}
```

**Query pattern:**
```python
results = collection.query(
    query_embeddings=[query_vector],
    n_results=20,
    include=["documents", "metadatas", "distances"],
    where={"doc_id": {"$in": allowed_doc_ids}}  # future: per-domain filtering
)
```

**v1 constraint:** Single collection, no per-tenant isolation. < 1K docs fits comfortably in ChromaDB local persistence mode (SQLite + HNSW index). No need for ChromaDB server mode in v1.

---

## Build Order

The components have hard dependency chains. Build in this order — each block unblocks the next.

### Block 1: Foundation (nothing depends on this being done first, but everything else does)

```
PostgreSQL schema + migrations
    → enables: every service that writes state
Docker Compose skeleton (all services defined, ports mapped)
    → enables: local development from day one
JWT auth middleware (FastAPI dependency)
    → enables: all protected endpoints
```

**Block 1 must exist before any API endpoint is tested.** The schema is load-bearing; getting it wrong causes cascading migrations.

---

### Block 2: Ingestion Pipeline (unblocks Query Pipeline)

```
Embedding Service (sentence-transformers wrapper)
    → enables: Ingestion AND Query (shared dependency)
ChromaDB client + collection init
    → enables: Ingestion writes, Retrieval reads
Document Parser (PyMuPDF, python-docx, text)
    → enables: Ingestion
Chunker
    → enables: Ingestion
Ingestion Service (compose parser + chunker + embedder + chroma + postgres)
    → enables: knowledge base to have content
POST /admin/documents endpoint
    → enables: frontend to trigger ingestion
```

**You cannot test the query pipeline until documents are ingested.** Build and test ingestion first with a smoke-test document.

---

### Block 3: Query Pipeline (unblocks Evaluation + Observability)

```
Query Processing (normalization)
Retrieval Service (ChromaDB query)
Reranker (cross-encoder)
Context Builder (prompt assembly)
LLM Service (Claude API call)
Citation Generator
Response Validator + Fallback Handler
POST /query endpoint
```

Build and test each stage in isolation with hardcoded inputs before composing them. The full pipeline should pass an end-to-end smoke test (upload a doc, query it, get a cited answer) before adding evaluation or observability.

---

### Block 4: Telemetry (add to query pipeline, non-breaking)

```
Telemetry context object (per-request dict)
Stage timing wrappers around each pipeline step
ai_requests INSERT at end of request
GET /admin/stats endpoint (aggregation queries)
```

Telemetry is additive — it does not change pipeline logic, only instruments it. Add after query pipeline is working to avoid noise during debugging.

---

### Block 5: Evaluation Engine (async, non-blocking)

```
Evaluation Engine (3 LLM-as-judge prompts)
evaluations INSERT
FastAPI BackgroundTasks dispatch in query endpoint
GET /admin/evaluations endpoint
```

Evaluation depends on a working query pipeline (needs real answers to score) and a working PostgreSQL schema (needs to write scores). Add last among backend services.

---

### Block 6: Admin Dashboard + Frontend Polish

```
React admin dashboard (stats, request explorer)
File upload UI (triggers POST /admin/documents)
Query UI (chat interface)
```

Frontend can be built in parallel with Block 3+ if the API contract is agreed upon early. Mock API responses during frontend development to avoid blocking.

---

## API Design Patterns for FastAPI RAG Services

### Request/Response Contracts

```python
# POST /query
class QueryRequest(BaseModel):
    question: str = Field(..., min_length=1, max_length=2000)

class Citation(BaseModel):
    document: str
    page: int | None
    section: str | None

class QueryResponse(BaseModel):
    request_id: str
    answer: str
    sources: list[Citation]
    confidence: float
    fallback_triggered: bool
    latency_ms: int

# POST /admin/documents
# multipart/form-data: file field
class IngestResponse(BaseModel):
    document_id: str
    filename: str
    chunk_count: int
    status: Literal["ingested", "failed"]
```

### Dependency Injection for Pipeline Services

Use FastAPI's `Depends()` to inject shared services. This keeps the pipeline composable and testable.

```python
# services/embedding.py
class EmbeddingService:
    def __init__(self):
        self.model = SentenceTransformer("all-MiniLM-L6-v2")
    def embed(self, text: str) -> list[float]:
        return self.model.encode(text).tolist()

# Singleton via lru_cache
@lru_cache(maxsize=1)
def get_embedding_service() -> EmbeddingService:
    return EmbeddingService()

# In router
@router.post("/query")
async def query(
    req: QueryRequest,
    embedding_svc: EmbeddingService = Depends(get_embedding_service),
    background_tasks: BackgroundTasks = BackgroundTasks(),
    ...
):
```

**Why singleton embedding service:** The sentence-transformers model is ~90MB. Loading it once at startup (via lifespan event) and reusing it avoids 2-3 seconds of cold load per request.

### Async Evaluation Dispatch (BackgroundTasks)

```python
@router.post("/query", response_model=QueryResponse)
async def query(
    req: QueryRequest,
    background_tasks: BackgroundTasks,
    ...
):
    # ... run full pipeline synchronously ...
    result = await run_query_pipeline(req.question)
    
    # Commit telemetry synchronously (must be done before return)
    request_record = await save_request_to_db(result)
    
    # Dispatch evaluation — runs AFTER response is sent, does NOT block return
    background_tasks.add_task(
        evaluate_request,
        request_id=request_record.id,
        question=req.question,
        context=result.context_chunks,
        answer=result.answer,
    )
    
    return QueryResponse(...)
```

**BackgroundTasks vs asyncio.create_task:** Use `BackgroundTasks` (FastAPI built-in) for v1. It runs after the response is sent, is framework-managed, and requires no separate worker infrastructure. `create_task` is fine too but needs careful error handling. Do NOT use Celery or Redis queues for v1 — overkill at this scale.

### Lifespan for Model Loading

```python
# main.py
from contextlib import asynccontextmanager

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup: load models once
    get_embedding_service()        # warms the LRU cache
    get_reranker()                 # loads cross-encoder
    init_chroma_collection()       # connects to ChromaDB, creates collection if needed
    yield
    # Shutdown: cleanup if needed

app = FastAPI(lifespan=lifespan)
```

Models loaded at startup, not on first request. This prevents the first user hitting a cold-start penalty.

### Per-Stage Telemetry Pattern

```python
import time

class TelemetryContext:
    def __init__(self, request_id: str):
        self.request_id = request_id
        self.stages: dict[str, float] = {}
        self.metadata: dict = {}
    
    def record(self, stage: str, latency_ms: float, **kwargs):
        self.stages[stage] = latency_ms
        self.metadata.update(kwargs)

# Usage in each pipeline stage:
t0 = time.perf_counter()
embedding = embedding_svc.embed(query)
telemetry.record("embedding", (time.perf_counter() - t0) * 1000)
```

Pass `TelemetryContext` through the pipeline as a parameter — don't use global state or thread-locals. This is safe for async FastAPI handlers.

### Error Handling Strategy

```python
# Define domain exceptions
class RetrievalError(Exception): pass
class LLMError(Exception): pass
class EmbeddingError(Exception): pass

# Global handler in FastAPI
@app.exception_handler(RetrievalError)
async def retrieval_error_handler(request, exc):
    # Log + return graceful error (don't expose internals)
    return JSONResponse(status_code=503, content={"error": "retrieval_unavailable"})
```

Pipeline errors should never return 500 stack traces to the user. Map domain exceptions to HTTP codes in handlers.

---

## Scalability Notes (v1 context)

| Concern | At < 1K docs (v1) | At 100K docs (future) |
|---------|-------------------|-----------------------|
| Embedding model | In-process, sentence-transformers | Separate embedding microservice |
| ChromaDB | Local persistence, SQLite+HNSW | ChromaDB server mode or Qdrant |
| Reranker | In-process cross-encoder | GPU-backed reranker service |
| Evaluation | BackgroundTasks (in-process) | Celery + Redis worker queue |
| LLM calls | Direct Claude API per request | Batching, caching common queries |

v1 is intentionally in-process. Extract to microservices only when throughput data from telemetry justifies it.

---

## Potential Pitfalls Flagged for Later Research

| Topic | Risk |
|-------|------|
| Embedding model choice | `all-MiniLM-L6-v2` vs OpenAI embeddings — must be decided before any data is ingested; changing models requires re-embedding all docs |
| ChromaDB persistence in Docker | Must mount a volume; default in-container storage is lost on container restart |
| Token budget management | Context Builder must enforce a max token limit per prompt or Claude API will reject requests; chunk count × chunk size must stay under model context window |
| Reranker latency | Cross-encoder over 20 chunks adds ~100-300ms; benchmark before committing to this approach |
| Async evaluation error isolation | If evaluation Claude API calls fail, must not surface errors to users; wrap in try/except inside BackgroundTask |
| JWT secret management | Must be in environment variable, never hardcoded; Docker Compose env file must not be committed |
