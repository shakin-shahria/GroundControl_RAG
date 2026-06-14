# Feature Landscape: GroundControl_RAG

**Domain:** Production RAG platform (multi-domain knowledge base, enterprise internal use)
**Researched:** 2026-06-14
**Confidence:** MEDIUM-HIGH (based on project spec + domain knowledge; external search unavailable)

---

## Table Stakes

Features that users and operators expect from any production RAG system. Missing any of these and the system either fails at its core purpose or is not deployable.

### 1. Document Ingestion Pipeline

| Sub-Feature | Why Expected | Complexity | Notes |
|-------------|--------------|------------|-------|
| PDF parsing with text extraction | PDFs are the universal enterprise doc format | Medium | Use `pdfplumber` or `PyMuPDF`; handle multi-column, headers/footers |
| DOCX parsing | Office docs dominate HR/policy content | Low | `python-docx` is sufficient for v1 |
| TXT ingestion | Plaintext technical docs, READMEs | Low | Trivial; still needs metadata stamping |
| Metadata preservation on ingest | Without doc name, page, section — citations are impossible | Medium | Extract and store: filename, page number, section heading, upload timestamp, doc type |
| Drag-and-drop upload UI | Admin expects UI-first; CLI batch import is a v2 feature | Medium | React dropzone + multipart upload to FastAPI |
| Ingestion status feedback | Admins need to know if a document processed or failed | Low | Job status field on document record; poll or SSE |

**Dependency chain:** Metadata preservation → Citation generation → Response validation

---

### 2. Chunking Strategy

This is the most under-specified but highest-impact ingestion decision. Poor chunking breaks retrieval regardless of model quality.

| Sub-Feature | Why Expected | Complexity | Notes |
|-------------|--------------|------------|-------|
| Configurable chunk size | One size does not fit all domains | Low | 512 tokens is a safe default for mixed enterprise docs |
| Chunk overlap | Without overlap, answers that span chunk boundaries are missed | Low | 10-20% overlap (50-100 tokens) prevents boundary blind spots |
| Metadata stamping per chunk | Each chunk must carry its source metadata for citation | Medium | Store: doc_id, chunk_index, page_number, section, char_offset |
| Sentence-boundary-aware splitting | Splitting mid-sentence breaks semantic coherence | Medium | Use `RecursiveCharacterTextSplitter` logic or spaCy sentence boundaries |
| Section-aware chunking | HR/policy docs have sections; splitting mid-section loses context | High | Detect H1/H2 boundaries in DOCX; paragraph breaks in PDF |

**Key decision:** Fixed-size with overlap (512 tokens, 10% overlap) is the correct default for v1 across mixed doc types. Hierarchical or semantic chunking are v2 improvements.

**Dependency:** Chunking quality directly limits retrieval recall ceiling — no retrieval improvement fixes bad chunks.

---

### 3. Embedding and Vector Storage

| Sub-Feature | Why Expected | Complexity | Notes |
|-------------|--------------|------------|-------|
| Sentence embedding of each chunk | Core RAG mechanic | Low | `all-MiniLM-L6-v2` (384-dim) is the default; `bge-large-en-v1.5` for higher quality |
| ChromaDB persistence | < 1K docs; local mode is sufficient | Low | `PersistentClient` with Docker volume mount |
| Embedding model consistency | Query must use same model as ingest | Low | Single embedding service; no mixing models |
| Batch embedding on ingest | Don't embed one chunk at a time; batch for throughput | Low | `sentence-transformers` encode() accepts lists |
| Metadata filter support | Users from different domains should only see relevant docs | Medium | ChromaDB `where` filter on `doc_type` or `domain` metadata field |

---

### 4. Natural Language Query Pipeline

| Sub-Feature | Why Expected | Complexity | Notes |
|-------------|--------------|------------|-------|
| Query embedding | Core retrieval mechanic | Low | Same embedding model as ingest |
| Vector similarity search (top-k) | Retrieve candidate chunks | Low | ChromaDB `query()` with `n_results=20` as initial candidate pool |
| Query normalization | Raw questions contain noise; cleaning improves recall | Low | Lowercase, strip punctuation, basic stopword handling |
| Request ID + trace ID generation | Production systems need traceable requests | Low | UUID4 per request; store from entry point |

---

### 5. Reranking

| Sub-Feature | Why Expected | Complexity | Notes |
|-------------|--------------|------------|-------|
| Cross-encoder reranking of top-k | Vector similarity is approximate; reranking improves precision significantly | Medium | `cross-encoder/ms-marco-MiniLM-L-6-v2` is the standard production choice; scores all (query, chunk) pairs |
| Select top-5 after reranking | Standard production pattern; limits context window consumption | Low | Configurable; 5 is correct default for mixed domains |
| Score threshold cutoff | Reranked chunks below a relevance floor should be dropped entirely | Medium | If best reranked score < 0.3, trigger low-confidence fallback |

**Why this matters:** In benchmarks, cross-encoder reranking over 20 initial candidates consistently improves NDCG@5 by 15-25% vs pure vector similarity alone (HIGH confidence — well-documented pattern).

---

### 6. Context Construction

| Sub-Feature | Why Expected | Complexity | Notes |
|-------------|--------------|------------|-------|
| System prompt with grounding instruction | LLM must be told to answer only from context | Low | "Answer based only on provided context. If information is absent, say so." |
| Chunk injection with source labels | Each chunk in context must be labeled with its source for citation linking | Medium | Format: `[Source: refund_policy.pdf, Page 4]\n{chunk_content}` |
| Token budget management | Context window limits exist; overfilling causes truncation or errors | Medium | Count tokens before LLM call; trim to fit within model limit (leave headroom for response) |
| User question appended last | Recency bias in transformers means the question at the end gets most attention | Low | Always append: "Question: {user_query}" as the final element |

---

### 7. LLM Generation

| Sub-Feature | Why Expected | Complexity | Notes |
|-------------|--------------|------------|-------|
| Claude API call with context | Core generation step | Low | Anthropic Python SDK `messages.create()` |
| Model version pinning | Unpinned model versions drift; evaluation scores become meaningless | Low | Pin to `claude-3-5-sonnet-20241022` or equivalent; store version in response record |
| Response + token usage recording | Required for telemetry and cost tracking | Low | Capture `usage.input_tokens`, `usage.output_tokens`, latency |
| Temperature = 0 for factual Q&A | Grounded RAG answers should be deterministic | Low | `temperature=0` — factual queries are not creative tasks |

---

### 8. Citation Generation

| Sub-Feature | Why Expected | Complexity | Notes |
|-------------|--------------|------------|-------|
| Source attribution per response | Core differentiator that separates RAG from chatbot; expected by users | Medium | Extract source labels from injected context chunks; attach to response |
| Citation format: doc name + page number | Minimum viable citation that lets user verify the answer | Low | `{"document": "refund_policy.pdf", "page": 4, "section": "Returns"}` |
| Multiple citations per response | Answers may draw from multiple chunks/documents | Medium | Return array; deduplicate by (doc_id, page) |

**Dependency:** Citation generation depends on metadata stamping during ingestion and chunk labeling during context construction.

---

### 9. Response Validation and Fallback

| Sub-Feature | Why Expected | Complexity | Notes |
|-------------|--------------|------------|-------|
| Grounding check | Is the answer supported by the retrieved context? | High | LLM-as-judge: "Is this answer supported by the following context? Yes/No" using Claude |
| Confidence threshold gate | If retrieval scores or grounding check fail, don't return the answer | Medium | Configurable threshold; default: grounding_score < 0.7 → fallback |
| "I don't know" response | System must gracefully refuse rather than hallucinate | Low | Return structured fallback: `{"answer": "I could not find reliable information...", "citations": [], "fallback": true}` |
| Clarification request option | When intent is ambiguous, ask rather than guess | Medium | Detected by zero relevant chunks retrieved (top similarity < 0.4) |

---

### 10. Evaluation Engine (Per-Request)

| Sub-Feature | Why Expected | Complexity | Notes |
|-------------|--------------|------------|-------|
| Faithfulness scoring | Is the generated answer supported by retrieved context? | High | LLM-as-judge prompt: score 0.0-1.0 |
| Relevance scoring | Does the answer address the user's actual question? | High | LLM-as-judge prompt: score 0.0-1.0 |
| Correctness scoring | Is the answer factually accurate relative to context? | High | LLM-as-judge prompt: score 0.0-1.0 |
| Retrieval quality score | Were the retrieved chunks actually useful? | Medium | Average reranker score for top-5 chunks; stored as float |
| Evaluation stored per request | Scores must be persisted to drive dashboard and improvement | Low | `evaluations` table with `request_id` FK |

**Note:** All three LLM-as-judge calls can be batched into one Claude call using structured output to reduce latency. This is a significant implementation optimization.

---

### 11. Per-Request Telemetry

| Sub-Feature | Why Expected | Complexity | Notes |
|-------------|--------------|------------|-------|
| Stage-level latency tracking | Cannot debug performance without per-stage timing | Medium | Instrument: embed_ms, retrieve_ms, rerank_ms, context_ms, generate_ms, validate_ms, eval_ms |
| Total end-to-end latency | Primary user-facing SLA metric | Low | Sum of stages or wall-clock time from request receipt |
| Token usage (input + output) | Cost and context budget awareness | Low | Captured from Claude API response |
| Retrieval score logging | Track retrieval quality trends over time | Low | Log top-5 scores and average |
| Error logging with stage tagging | Know which pipeline stage failed | Medium | Structured error: `{"stage": "reranking", "error": "timeout", "request_id": "req_123"}` |
| Failure reason classification | Distinguish: no-docs-found, low-confidence, model-error, validation-failed | Medium | Enum field on request record |

---

### 12. Authentication

| Sub-Feature | Why Expected | Complexity | Notes |
|-------------|--------------|------------|-------|
| JWT-based login | Multi-user product requires session auth | Medium | FastAPI + `python-jose` + bcrypt password hashing |
| Role distinction: user vs admin | Admin upload/dashboard must be gated separately from query | Medium | `role` field on user record; middleware checks on admin routes |
| Token refresh | Sessions need longevity without permanent tokens | Medium | Access token (15 min) + refresh token (7 days) is standard |
| Protected query endpoint | Query API must require valid JWT | Low | FastAPI `Depends(get_current_user)` on query routes |

---

### 13. Admin Dashboard

| Sub-Feature | Why Expected | Complexity | Notes |
|-------------|--------------|------------|-------|
| Total requests counter | Basic health metric | Low | SQL COUNT on `ai_requests` |
| Average latency (rolling 24h) | SLA monitoring | Low | SQL AVG on `latency_ms` |
| Failure rate percentage | Reliability metric | Low | COUNT(fallback=true) / COUNT(*) |
| Recent request log (last 50) | Debugging and audit | Low | Paginated SELECT with question, answer, scores, latency |
| Document list with status | Admins need to see what's in the knowledge base | Low | SELECT from `documents` with ingest status |
| Per-request detail view | Drill into a single request to see chunks, scores, answer | Medium | Request detail page: question → chunks → answer → eval scores |

---

## Differentiators

Features that elevate GroundControl_RAG above a basic "wrap an LLM with some docs" implementation. Not expected by default — but they're what makes this a platform, not a script.

### 1. LLM-as-Judge Evaluation (Custom Metrics)

**Value:** Most RAG systems skip evaluation entirely or use RAGAS (heavy dependency). Per-request faithfulness/relevance/correctness scoring using Claude as judge gives a quality signal on every single response.

**Complexity:** High (requires careful prompt engineering; scoring prompts need calibration)

**Why differentiating:** Enables quality regression detection, comparison across model versions, and surfacing low-quality responses automatically — none of which a basic RAG chatbot has.

**Note:** Batch all three judge calls into one structured Claude call to minimize latency overhead.

---

### 2. Grounding-Enforced Fallback (Anti-Hallucination Gate)

**Value:** The system explicitly refuses to answer when evidence is insufficient. This is rare in production deployments — most systems answer anyway.

**Complexity:** Medium (validation prompt + threshold tuning)

**Why differentiating:** Directly addresses the core LLM reliability problem. Makes the system trustworthy for HR/policy/compliance queries where a wrong answer has real consequences.

---

### 3. Structured Citation Output

**Value:** Every answer comes with machine-readable citations (document, page, section). Users can click through to source. This is trust infrastructure.

**Complexity:** Medium (metadata chain from ingest through to response must be intact)

**Why differentiating:** Most internal chatbots cite nothing. Citations enable user verification and are essential for regulated domains (HR policy, legal, compliance).

---

### 4. Cross-Encoder Reranking

**Value:** Initial vector retrieval casts a wide net; the cross-encoder selects the most semantically relevant chunks with a much more powerful relevance model.

**Complexity:** Medium (`cross-encoder/ms-marco-MiniLM-L-6-v2` is plug-and-play via sentence-transformers)

**Why differentiating:** Many basic RAG implementations skip reranking. The quality improvement is measurable and significant, especially for policy/procedure docs where phrasing varies.

---

### 5. Stage-Level Telemetry

**Value:** Most RAG apps log total latency at best. Per-stage timing (embed_ms, retrieve_ms, rerank_ms, generate_ms, eval_ms) lets you pinpoint bottlenecks.

**Complexity:** Medium (instrumentation must be consistent across all pipeline stages)

**Why differentiating:** Operationally, this is what separates a debuggable production system from a black box.

---

### 6. Request Explorer in Dashboard

**Value:** Drill into any request: see exact chunks retrieved, reranking scores, generated answer, and evaluation scores. This is a debugging and quality review tool that most dashboards don't offer.

**Complexity:** Medium (data must be stored; UI query + detail view)

**Why differentiating:** Enables engineers and AI teams to audit why a specific answer was given — crucial for a multi-domain system where different doc types behave differently.

---

### 7. Domain-Tagged Documents

**Value:** Store document domain (hr, product, technical) as metadata. Enables domain-scoped queries and per-domain quality analysis.

**Complexity:** Low (metadata field on ingest; filter in ChromaDB query)

**Why differentiating:** Multi-domain knowledge bases without scoping return irrelevant cross-domain chunks. An HR question should not retrieve API docs.

---

## Anti-Features

Features that production RAG platforms commonly add but that harm v1 by increasing complexity, delaying delivery, or solving problems that don't exist yet at this scale.

| Anti-Feature | Why to Avoid in V1 | What to Do Instead |
|--------------|-------------------|-------------------|
| RAGAS framework integration | Heavy dependency (requires OpenAI embeddings by default, opaque scoring logic); harder to tune than custom prompts | Custom LLM-as-judge prompts with Claude; same signal, full control |
| Prometheus + Grafana stack | Operational overhead disproportionate to a single-tenant < 1K doc deployment | PostgreSQL + lightweight React dashboard; add Prometheus in v2 if scale demands |
| Hybrid search (BM25 + vector) | Meaningful improvement for large corpora; at < 1K docs the gain is marginal | Pure vector search with reranking; add BM25 in v2 if retrieval recall is insufficient |
| Semantic chunking | Requires embedding every candidate boundary and clustering; expensive and complex at build time | Fixed-size with overlap (512 tokens, 50-token overlap); correct default for v1 |
| Query expansion / HyDE | Hypothetical Document Embedding improves recall for vague queries; adds latency and complexity | Rely on reranking to compensate; add query expansion in v2 if recall is demonstrably poor |
| Streaming responses | Good UX but requires SSE/WebSocket plumbing in both FastAPI and React | Synchronous response for v1; add streaming in v2 |
| Conversation history / multi-turn | Context management across turns is a significant engineering surface; not required for document Q&A | Stateless per-query in v1; each query is independent |
| Multi-tenancy / per-org isolation | DB schema changes, auth layer changes, vector namespace per tenant — major scope | Single tenant v1; ChromaDB collection-per-tenant pattern is a clean v2 migration path |
| Fine-tuning or RLHF | Separate operational concern; RAG quality comes from retrieval/evaluation, not model weights | Focus on pipeline quality; fine-tuning is a later optimization if base model underperforms |
| Human-in-the-loop escalation | Adds routing, notification, and ticketing surface; human fallback is rare at this corpus size | "I don't know" fallback covers v1 graceful degradation |
| Redis caching layer | Response caching requires cache invalidation logic when docs update; adds infra dependency | Acceptable latency at < 1K docs without caching; add if p95 > 5s in production |
| Vector DB managed service (Pinecone, Weaviate) | External dependency, cost, network latency; overkill at < 1K docs | ChromaDB local with Docker volume; migrate to managed DB when corpus > 10K docs |
| API key auth (external consumers) | Adds a separate auth surface; out-of-scope for internal deployment | JWT sessions for internal users; API keys are a v2 concern if external integrations emerge |

---

## Feature Dependencies

Critical chains where one feature gates another:

```
Document metadata stamping (ingest)
  → Chunk metadata per chunk (chunking)
    → Source labels in context construction
      → Citation generation
        → Response validation (grounding check references citations)

Embedding model selection
  → Consistent embedding at ingest
    → Consistent embedding at query time
      → Vector similarity search
        → Reranking (operates on initial k results)
          → Top-5 selection
            → Context construction
              → LLM generation
                → Evaluation engine (judges the generation against context)
                  → Telemetry storage (stores eval scores)
                    → Admin dashboard (reads telemetry)

Authentication
  → Protected query endpoint
  → Admin-only document upload
  → Admin dashboard access
```

---

## MVP Recommendation

### Priority 1 — Core Pipeline (ship first, nothing works without this)
1. Document ingestion with metadata (PDF/DOCX/TXT)
2. Fixed-size chunking with overlap + metadata per chunk
3. Sentence embedding + ChromaDB storage
4. Vector similarity search (top-20 candidates)
5. Cross-encoder reranking → top-5
6. Context construction with source labels
7. Claude generation (temperature=0, pinned model version)
8. Citation attachment from chunk metadata

### Priority 2 — Reliability Layer (what makes it production-grade)
9. Response validation (grounding check via LLM-as-judge)
10. Fallback handler ("I don't know" for low-confidence)
11. Per-request evaluation (faithfulness + relevance + correctness)
12. JWT authentication (user + admin roles)

### Priority 3 — Observability (what makes it debuggable)
13. Per-stage telemetry (embed_ms, retrieve_ms, rerank_ms, generate_ms, eval_ms)
14. Admin dashboard (request counts, avg latency, failure rate, request log)
15. Document management UI (upload + status)

### Defer to V2
- Streaming responses
- Hybrid search (BM25 + vector)
- Query expansion / HyDE
- Conversation history
- Multi-tenancy
- Redis caching
- Prometheus/Grafana
- Human escalation

---

## Chunking Strategy Reference

This deserves its own section because it is the highest-impact low-visibility decision in any RAG build.

### Recommended: Fixed-Size with Sentence-Boundary Awareness and Overlap

- **Chunk size:** 512 tokens (approximately 350-400 words)
- **Overlap:** 50 tokens (~10%)
- **Splitter logic:** Split at sentence boundaries where possible; never split mid-sentence
- **Metadata per chunk:** `doc_id`, `chunk_index`, `page_number`, `section_heading`, `char_start`, `char_end`
- **Why this size:** 512 tokens fits 3-5 policy paragraphs — enough context for a coherent answer, small enough for precision retrieval

### Chunk Size Trade-off

| Chunk Size | Retrieval Precision | Answer Completeness | Risk |
|------------|--------------------|--------------------|------|
| 128 tokens | High | Low (answer often spans multiple chunks) | Answer fragmentation |
| 256 tokens | Good | Medium | Acceptable for short factual Q&A |
| 512 tokens | Medium | Good | **Recommended default for policy docs** |
| 1024 tokens | Low | High | Top-k precision drops; retrieves too-broad chunks |

### Section-Aware Splitting (Optional V1 Enhancement)

For DOCX files, detect heading-level boundaries (H1/H2) and avoid splitting within a section unless forced by token limit. Store `section_heading` as chunk metadata. This improves citation quality significantly for structured HR/policy docs with no meaningful implementation overhead using `python-docx` heading detection.

---

## Retrieval Quality Feature Coverage

| Retrieval Concern | Feature | V1 or V2 |
|-------------------|---------|-----------|
| Low recall (relevant docs not retrieved) | Top-20 initial candidates; 512-token chunks | V1 |
| Low precision (irrelevant chunks in top-5) | Cross-encoder reranking | V1 |
| Boundary blind spots | 50-token chunk overlap | V1 |
| Cross-domain noise | Domain metadata filter on ChromaDB query | V1 |
| Keyword-dense queries not matching semantic embedding | — | V2 (hybrid BM25) |
| Short vague queries | — | V2 (query expansion / HyDE) |
| Stale documents returned | Doc version tracking + re-ingestion | V2 |

---

## Sources

- Project specification: `/base.md` (HIGH confidence — authoritative for this project's requirements)
- Project context: `/.planning/PROJECT.md` (HIGH confidence)
- RAG pipeline patterns: domain knowledge from well-established RAG literature (MEDIUM confidence — cross-encoder reranking improvement figures are widely documented; specific numbers should be validated against project's actual retrieval quality)
- Chunking strategy: established community practice (MEDIUM confidence — fixed-size with overlap is universally recommended starting point; optimal size is domain-dependent and should be tuned empirically)
