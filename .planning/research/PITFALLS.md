# Domain Pitfalls: Production RAG Platform

**Domain:** Retrieval Augmented Generation (RAG) with evaluation and citations
**Project:** GroundControl_RAG
**Researched:** 2026-06-14
**Confidence:** HIGH (core RAG patterns are stable and well-documented; ChromaDB specifics HIGH from official behavior; JWT/FastAPI HIGH from production incident reports)

---

## Critical Pitfalls

Mistakes that cause rewrites, data corruption, or fundamental product failure.

---

### Pitfall 1: Embedding Model Mismatch Between Ingestion and Query

**Description:**
The embedding model used at query time MUST be byte-for-byte identical to the model used during document ingestion. Changing the embedding model (even a minor version bump of `all-MiniLM-L6-v2`, or swapping from HuggingFace to OpenAI embeddings) after documents are ingested invalidates every stored vector. The similarity scores returned by ChromaDB will be meaningless — the query vector lives in a completely different geometric space than the stored document vectors. The system will appear to work (it returns results) but retrieval quality collapses silently.

**Why It Happens:**
- Teams update the embedding model mid-project to improve quality
- The model library (`sentence-transformers`) releases a new default version
- Staging and production use different model configs
- Docker image rebuilds pull a newer model version implicitly
- No validation exists to detect the mismatch at query time

**Consequences:**
- Retrieval returns wrong or irrelevant chunks
- Confidence scores look fine but answers are hallucinated or grounded on wrong documents
- The bug is invisible — the system returns 200 OK responses with bad content
- Full re-ingestion of all documents required to fix

**Prevention:**
1. Pin the embedding model name AND version in a single config constant (`EMBEDDING_MODEL = "sentence-transformers/all-MiniLM-L6-v2"` at a specific commit hash or package version).
2. Store the model name as metadata on every ChromaDB collection at creation time.
3. At application startup, assert that the configured model matches the stored collection metadata. Raise a hard error if they differ.
4. Never allow implicit model upgrades — pin `sentence-transformers` to an exact version in `requirements.txt` (not `>=`).
5. After any re-ingestion, run a smoke test: embed a known query, retrieve the top chunk, verify it matches expected content.

**Warning Signs:**
- Retrieval scores are uniformly low (< 0.3) for queries that should match
- The top retrieved chunk is obviously unrelated to the query
- Scores are identical across very different queries

**Phase:** Ingestion pipeline setup (Phase 1). Must be solved before any documents are ingested.

---

### Pitfall 2: ChromaDB Persistence Misconfiguration in Docker

**Description:**
ChromaDB's default in-memory mode loses all data when the container restarts. The `persist_directory` must be set explicitly AND the path inside the container must be bind-mounted to a Docker volume. Many projects configure the path in Python code but forget the volume mount in `docker-compose.yml`, or configure the mount but forget to set `persist_directory` in the client, or set both but mount to a non-persistent tmpfs location.

**Why It Happens:**
- ChromaDB's client API changed between v0.3.x and v0.4.x — `chromadb.Client()` vs `chromadb.PersistentClient(path=...)` are different constructors with different defaults.
- Docker volume mounts are in `docker-compose.yml` but developers test with `python main.py` locally without Docker, where the path exists but is not the container path.
- `chromadb.Client(Settings(persist_directory=...))` (old API) silently ignored in newer versions.

**Consequences:**
- All embedded documents are lost on container restart
- Re-ingestion required every time the service restarts
- Intermittent data loss in production if containers restart due to OOM or deployments

**Prevention:**
1. Use `chromadb.PersistentClient(path="/data/chroma")` (v0.4+ API). Do not use the Settings-based constructor.
2. In `docker-compose.yml`, declare an explicit named volume: `chroma_data:/data/chroma`.
3. Add a startup health check: after initialization, count the number of collections. If expected collections exist but document count is 0, raise an alert (not just a log).
4. Test persistence explicitly: ingest a document, restart the container, verify it's still there. Add this as a smoke test.
5. Never use `chromadb.EphemeralClient()` in production code paths.

**Warning Signs:**
- Document count resets to 0 after container restart
- ChromaDB collection exists but `collection.count()` returns 0
- Ingestion endpoint re-processes already-processed documents without error

**Phase:** Infrastructure setup / Docker Compose configuration (Phase 2).

---

### Pitfall 3: Chunk Size Tuned for Developer Convenience, Not Retrieval Quality

**Description:**
Teams pick chunk sizes (e.g., 512 tokens, 1000 characters) based on what "feels right" without empirical testing. Two failure modes: chunks too small (individual chunks lack enough context for the LLM to generate a coherent answer), or chunks too large (the top-k chunks fill the entire context window, leaving no room for the question or system prompt, and retrieval scores become diluted because each chunk covers multiple topics). The overlap between chunks is equally undertested — too little overlap means sentences that straddle chunk boundaries are unretrievable; too much overlap wastes tokens and inflates the collection size.

**Why It Happens:**
- Chunking is the first thing built and the last thing revisited
- PDF parsing libraries (PyMuPDF, pdfplumber) emit text with arbitrary whitespace, page headers, footers, and table content mixed into prose — naive character-count chunking produces semantically incoherent chunks
- Developers test on clean, well-formatted documents; production documents are messy

**Consequences:**
- Answers lack sufficient context (chunks too small)
- Context window overflow: the combined chunks + system prompt + user question exceeds Claude's context limit, causing truncation errors at runtime
- Retrieved chunks contain page headers, footer boilerplate, or table rows that confuse the LLM
- Low faithfulness scores in evaluation despite correct retrieval

**Prevention:**
1. Use token-count chunking (not character count) to avoid context window surprises. Use `tiktoken` or the embedding model's tokenizer to count.
2. Start with 400-600 tokens per chunk with 10-15% overlap (50-80 tokens). Adjust empirically.
3. Before chunking, strip known boilerplate: page numbers, repeated headers/footers, table of contents entries.
4. After chunking, log the distribution of chunk sizes. Any chunk under 50 tokens or over 800 tokens is suspicious.
5. Test chunking on the actual document corpus, not synthetic clean text.
6. Calculate worst-case context: `(top_k chunks * max_chunk_tokens) + system_prompt_tokens + max_question_tokens` must stay under the model's context limit with headroom.

**Warning Signs:**
- Claude returns "I don't have enough context" for questions with obvious answers in the documents
- `anthropic.BadRequestError` about prompt length at query time
- Evaluation faithfulness scores cluster near 0 despite correct documents being retrieved

**Phase:** Ingestion pipeline (Phase 1). Revisit during evaluation phase when scores reveal problems.

---

### Pitfall 4: Citation Generation Without Source Tracking

**Description:**
Teams build the generation step first (pass chunks to Claude, get an answer) and add citations as an afterthought. The result: you can tell the user "this came from document X" but you cannot tell them which sentence in the answer maps to which chunk. Worse, Claude may synthesize across multiple chunks and the attribution becomes ambiguous. If chunk metadata (document name, page number, section heading) is not stored and returned alongside the chunk content during retrieval, there is no reliable way to reconstruct citations at the response layer.

**Why It Happens:**
- The LLM generates fluent prose that blends information from multiple chunks
- Chunk metadata is stored in ChromaDB but not propagated through the retrieval → reranking → prompt construction pipeline
- The system prompt instructs Claude to cite sources, but Claude's citations reference content not position

**Consequences:**
- Citations are fabricated or attached to wrong chunks (a form of hallucination)
- No auditability: users cannot verify which document an answer came from
- Citation mismatch becomes a trust failure — worse than no citations at all

**Prevention:**
1. At ingestion time, store rich metadata per chunk: `{"document_id": ..., "document_name": ..., "page_number": ..., "chunk_index": ..., "section_heading": ...}`. This metadata must be stored in ChromaDB and in PostgreSQL.
2. Number each chunk in the prompt explicitly: `[SOURCE 1]: <chunk content>`, `[SOURCE 2]: <chunk content>`. Instruct Claude to reference sources by number only.
3. At the response layer, parse `[SOURCE N]` references from Claude's output and map them to the retrieved chunk metadata (not to general document metadata).
4. Validate citations: check that each cited source number corresponds to an actual retrieved chunk. If Claude cites `[SOURCE 6]` but only 5 chunks were provided, that citation is fabricated.
5. Store the full retrieved chunk set alongside each response in PostgreSQL for auditability.

**Warning Signs:**
- Citation page numbers don't match the actual document pages
- Claude references source numbers higher than the number of provided chunks
- Two different questions about the same document return different citation metadata

**Phase:** Query pipeline / generation layer (Phase 2). Must be designed into the prompt architecture before the first response is generated.

---

### Pitfall 5: LLM-as-Judge Evaluation Bias and Score Inflation

**Description:**
When Claude is used both as the response generator and the evaluator (judge), it exhibits a self-consistency bias: it tends to rate its own outputs as faithful and relevant even when they are not. Additionally, prompt framing causes score inflation — asking "is this response faithful (0-1)?" with no calibration examples produces mostly 0.8-0.9 scores regardless of quality. The evaluation becomes a vanity metric, not a signal.

**Why It Happens:**
- Same model, same temperature, same framing evaluating its own output
- No calibration examples (few-shot examples of what score 0.3 vs 0.8 looks like)
- Evaluation prompt does not give the judge an explicit rubric
- Judge sees the response and chunks at the same time, anchoring on surface similarity

**Consequences:**
- Evaluation scores are consistently high even when the system is broken
- Threshold-based fallback logic never triggers (because scores are always inflated)
- Engineering team believes the system is working; quality issues surface only from user complaints

**Prevention:**
1. Write explicit rubrics in the judge prompt. Not "rate faithfulness" but "score 1 if every factual claim in the response is directly supported by a quoted passage in the sources. Score 0 if any factual claim is absent from the sources. Score 0.5 if claims are implied but not stated."
2. Include calibration examples in the judge prompt (few-shot): one clearly faithful response (expected 0.9), one clearly unfaithful response (expected 0.2), one borderline response (expected 0.5).
3. Evaluate with lower temperature (0.0) for determinism and consistency.
4. Periodically benchmark the judge against a manually labeled golden dataset (20-30 examples scored by a human). If judge scores differ from human scores by more than 0.2 on average, the judge prompt needs revision.
5. Track score distribution over time. If 95% of scores cluster above 0.7, the rubric is too generous or the prompt is anchoring.

**Warning Signs:**
- Evaluation scores have very low variance (0.75-0.95 for everything)
- Fallback threshold never triggers in production
- Manual inspection finds bad answers that scored 0.8+

**Phase:** Evaluation engine (Phase 3). Score calibration is ongoing work, not a one-time setup.

---

## Moderate Pitfalls

Mistakes that degrade quality or require significant rework, but don't cause complete failure.

---

### Pitfall 6: Fallback Threshold Miscalibration

**Description:**
The confidence threshold for refusing to answer (fallback) is set once and never revisited. Set too low: the system answers everything including questions it shouldn't, producing hallucinated responses. Set too high: the system refuses most queries, making it useless. The threshold depends on the quality of the embedding model, the chunk size, the reranking model, and the specific document corpus — it cannot be set universally.

**Why It Happens:**
- Threshold is chosen arbitrarily (e.g., "0.7 seems reasonable") without empirical calibration
- Calibration on ingestion-time docs doesn't transfer to production docs with different characteristics
- The threshold combines retrieval similarity score and evaluation confidence score without clear semantics

**Prevention:**
1. Separate the fallback signal into two components: retrieval confidence (ChromaDB similarity score of top result) and generation confidence (evaluation faithfulness score). Each has its own threshold.
2. Calibrate retrieval threshold on a sample of "unanswerable" queries (questions clearly outside the corpus) and "answerable" queries. Pick the value that correctly classifies 90%+ of each.
3. Log every fallback trigger with the query and scores so you can inspect false positives (refused a valid question) and false negatives (answered a hallucinated response).
4. Default thresholds as a starting point: retrieval similarity < 0.35 triggers fallback; faithfulness < 0.5 triggers fallback. Tune from there.

**Warning Signs:**
- Users report "I don't know" for questions clearly in the documents
- OR users report confidently wrong answers that should have been refused
- Fallback trigger rate is 0% or 90%+

**Phase:** Query pipeline integration (Phase 2), calibrated during evaluation phase (Phase 3).

---

### Pitfall 7: JWT Security Mistakes in FastAPI

**Description:**
Common JWT implementation mistakes in FastAPI: (1) using `HS256` with a weak or hardcoded secret key, (2) not validating token expiration (`exp` claim) on every request, (3) storing the JWT secret in source code instead of environment variables, (4) not implementing token refresh — users get logged out after expiry with no graceful path, (5) setting expiry too long (30 days) to avoid logout friction, defeating the purpose of short-lived tokens.

**Why It Happens:**
- JWT tutorial code hardcodes secrets for simplicity; devs copy the tutorial
- `python-jose` and `PyJWT` have different APIs and different default validation behaviors
- FastAPI's `Depends()` system makes it easy to write auth middleware that looks correct but skips validation steps

**Prevention:**
1. Load `JWT_SECRET_KEY` from environment variables only. Never commit to source. Add `.env` to `.gitignore` and validate on startup that the var is set.
2. Set `JWT_SECRET_KEY` to at least 32 random bytes (use `secrets.token_urlsafe(32)`).
3. Set access token expiry to 15-30 minutes. Implement a refresh token endpoint with a longer (7-day) expiry.
4. Use `python-jose[cryptography]` (not bare `python-jose`) for ECDSA support if you need asymmetric keys.
5. Write an explicit test: call a protected endpoint with (a) no token, (b) expired token, (c) tampered token, (d) valid token. All must behave as expected.
6. In the FastAPI dependency, explicitly check `exp` claim and raise `HTTP 401` with a clear message, not a generic 500.

**Warning Signs:**
- JWT secret is a short string like `"secret"` or `"mysecretkey"` in any config file
- No test coverage of auth failure modes
- Token expiry is set to > 24 hours

**Phase:** Auth setup (Phase 1 or Phase 2, whichever introduces protected routes).

---

### Pitfall 8: Async Evaluation Blocking the Query Response

**Description:**
Running LLM-as-judge evaluation synchronously in the query request handler adds 2-5 seconds of latency to every user-facing query. The user waits for Claude to generate a response AND for Claude to evaluate that response before getting anything back. This latency is invisible to the developer (who tests with curl and doesn't care about 5 seconds) but is felt immediately by users in a chat UI.

**Why It Happens:**
- The evaluation is added to the same request handler as generation
- "I'll make it async later" never happens
- Background task queueing (Celery, ARQ, FastAPI BackgroundTasks) adds complexity that gets deferred

**Prevention:**
1. Use FastAPI's built-in `BackgroundTasks` for evaluation: return the response to the user immediately, then trigger evaluation as a background task. This is zero-dependency and available in FastAPI out of the box.
2. Store a `status: "pending_evaluation"` field on the request record in PostgreSQL. Update it to `"evaluated"` once the background task completes.
3. The admin dashboard should be able to display evaluations even if they arrive a few seconds after the query response.
4. For Phase 1/2, `BackgroundTasks` is sufficient. Only move to Celery/ARQ if evaluation volume exceeds FastAPI's concurrency model.
5. Design the response schema to separate `response` (immediate) from `evaluation` (deferred) from the start.

**Warning Signs:**
- Query endpoint p99 latency is > 8 seconds
- Evaluation always completes within milliseconds of the response (impossible if they're asynchronous — this means they're synchronous)
- Users see spinner for 5+ seconds before seeing any response

**Phase:** Query pipeline design (Phase 2). The async/sync decision must be made when designing the response handler, not retrofitted.

---

### Pitfall 9: PostgreSQL and ChromaDB State Divergence

**Description:**
PostgreSQL stores document metadata (document name, file path, status, chunk count). ChromaDB stores the actual vectors. They are updated in separate operations without a transaction. If the server crashes between the PostgreSQL write and the ChromaDB write, you get a document record in PostgreSQL with no corresponding vectors in ChromaDB — the system believes the document is ingested but retrieval finds nothing. The reverse (vectors in ChromaDB, no PostgreSQL record) is harder to detect and leaves orphaned vectors.

**Why It Happens:**
- ChromaDB is not a transactional system and cannot participate in a PostgreSQL transaction
- Error handling in the ingestion pipeline doesn't account for partial failures
- Status flags (`ingestion_status`) are set to "complete" before all steps succeed

**Prevention:**
1. Use a state machine for ingestion status: `pending` → `chunking` → `embedding` → `indexing` → `complete`. Each transition is written to PostgreSQL before attempting the next step.
2. Only set status to `complete` after both the PostgreSQL chunk records AND the ChromaDB vectors are confirmed written (verify with `collection.get(ids=[chunk_ids])`).
3. If any step fails, set status to `failed` with an error message. Surface this in the admin dashboard.
4. Build a reconciliation check: on startup (or on a schedule), count PostgreSQL chunk records vs ChromaDB vector count for each document. Alert on mismatches.
5. Never delete a document from PostgreSQL without first deleting its vectors from ChromaDB.

**Warning Signs:**
- Document shows "ingested" in the dashboard but queries about its content return nothing
- ChromaDB collection count doesn't match PostgreSQL chunk count
- Ingestion pipeline crashes mid-way and documents are stuck in `indexing` status

**Phase:** Ingestion pipeline (Phase 1). The state machine must be designed before ingestion is implemented.

---

### Pitfall 10: PDF Parsing Extracting Garbage Text

**Description:**
PDF is a layout format, not a text format. Many PDFs — especially scanned documents, PDFs generated from presentations, or PDFs with complex column layouts — produce garbage text when parsed with text extraction libraries. PyMuPDF extracts text in reading order but multi-column documents often interleave columns. Tables produce rows without column delimiters. Headers and footers repeat on every page and pollute every chunk with boilerplate text.

**Why It Happens:**
- Development is tested with clean, well-structured PDFs (annual reports, manuals)
- Production documents include scans, slide decks exported as PDF, and complex tables
- No validation of parsed text quality before chunking and embedding

**Prevention:**
1. After extraction, validate minimum text quality: reject pages with fewer than 50 characters of printable text (likely scanned pages that need OCR).
2. Implement a page-level boilerplate filter: if the same text block appears on more than 3 pages in the same document, treat it as a header/footer and strip it.
3. Log the first 200 characters of extracted text for every page during ingestion so developers can inspect extraction quality.
4. For v1, limit to well-structured PDFs and DOCX files. Make the limitation explicit in the admin upload UI ("Note: scanned PDFs are not supported in v1").
5. Test with at minimum 5 representative documents from the actual use case before considering ingestion "done."

**Warning Signs:**
- Chunks contain strings like "Page 1 of 47" or "CONFIDENTIAL — DO NOT DISTRIBUTE" repeatedly
- Chunk content is a series of disjointed short phrases without semantic coherence
- Retrieval returns chunks that are entirely table cell values with no surrounding context

**Phase:** Ingestion pipeline (Phase 1). Test with real documents, not synthetic ones.

---

### Pitfall 11: Docker Compose Service Startup Order Failures

**Description:**
`docker-compose up` starts all services simultaneously. FastAPI starts before PostgreSQL is ready to accept connections. FastAPI tries to run Alembic migrations before the DB is up. ChromaDB is still initializing when the first ingestion request arrives. The `depends_on` directive only waits for the container to start, not for the service inside to be ready.

**Why It Happens:**
- `depends_on` is commonly misunderstood as "wait until healthy" but it actually means "wait until the container starts"
- PostgreSQL takes 2-5 seconds to initialize after the container starts
- No health check defined on the PostgreSQL or ChromaDB services

**Prevention:**
1. Define health checks in `docker-compose.yml` for PostgreSQL: `test: ["CMD-SHELL", "pg_isready -U $POSTGRES_USER"]` with `interval: 5s`, `retries: 5`.
2. Use `depends_on: condition: service_healthy` (not just `depends_on: [postgres]`) for any service that needs the DB ready.
3. Add a startup retry loop in the FastAPI app: try to connect to PostgreSQL up to 10 times with 2-second delays before raising a fatal error. This handles the race even when health checks are imperfect.
4. Alembic migrations should run as a separate `command` override or init container before the API starts, or as the first thing in the FastAPI startup event with retry logic.
5. Test cold start: `docker-compose down -v && docker-compose up` — watch logs to confirm services start in the correct order.

**Warning Signs:**
- `docker-compose up` works on the second run but not the first
- `sqlalchemy.exc.OperationalError: could not connect to server` in FastAPI startup logs
- Health endpoints return 200 but database queries fail immediately after

**Phase:** Docker Compose setup (Phase 2 or infrastructure phase). Must be solved before any service integration testing.

---

## Minor Pitfalls

Issues that cause friction or debugging time but are not architectural.

---

### Pitfall 12: ChromaDB Metadata Filtering Type Mismatches

**Description:**
ChromaDB metadata filtering requires exact type matching. If `page_number` was stored as a string (`"5"`) but the filter uses an integer (`{"page_number": {"$eq": 5}}`), the filter returns zero results silently. The same applies to boolean metadata fields stored as Python `True` vs the string `"True"`.

**Prevention:**
1. Define a metadata schema at ingestion time and enforce types explicitly before calling `collection.add()`.
2. Store all numeric fields as Python `int` or `float` (never as strings).
3. If filtering by metadata is needed in production, write a test that ingests a document with known metadata and asserts the filter returns it.

**Warning Signs:**
- Metadata filters return 0 results when the data is clearly present
- `collection.get(where={"page": 5})` returns empty but `collection.get()` shows documents with `page: "5"`

**Phase:** Ingestion pipeline (Phase 1), caught during integration testing.

---

### Pitfall 13: Reranker Latency Killing Query Performance

**Description:**
Cross-encoder rerankers (e.g., `cross-encoder/ms-marco-MiniLM-L-6-v2`) run inference on every (query, chunk) pair. With top-k=20 initial retrieval and a reranker scoring all 20 pairs, total reranker latency can be 1-3 seconds per query on CPU. This is often tested with k=5 chunks (fast) and only discovered with real k values in production.

**Prevention:**
1. Profile reranker latency at the actual top-k value you intend to use (k=20 initial, rerank to top-5).
2. If CPU latency is unacceptable, run the reranker on a larger initial k (50) and batch inference, or reduce k.
3. Set a hard timeout on reranking (e.g., 2 seconds) and fall back to the raw embedding similarity ranking if exceeded.
4. Log reranker latency as a separate telemetry field so it can be isolated in the dashboard.

**Warning Signs:**
- Query latency is consistently 4-6 seconds but generation is only 1-2 seconds
- Reranker latency scales linearly with k value

**Phase:** Query pipeline (Phase 2), discovered during performance testing.

---

### Pitfall 14: Missing Idempotency in Document Ingestion

**Description:**
If a user uploads the same document twice (or the upload request is retried), the ingestion pipeline processes it again, creating duplicate chunks and vectors in ChromaDB. This inflates the collection, increases retrieval time, and can cause duplicate citations in responses.

**Prevention:**
1. Compute a content hash (SHA-256 of the file bytes) at upload time. Store it in PostgreSQL. Check for duplicates before starting ingestion.
2. On duplicate detection, return a 200 with the existing document record (not a 409 — the user doesn't care about the distinction).
3. Delete the old vectors from ChromaDB before re-ingesting if a re-upload with the same name but different content is allowed.

**Warning Signs:**
- Document count in PostgreSQL grows faster than expected
- The same chunk text appears multiple times in retrieval results
- ChromaDB collection size grows even when no new documents are added

**Phase:** Ingestion pipeline (Phase 1).

---

## Phase-Specific Warning Matrix

| Phase Topic | Likely Pitfall | Mitigation |
|-------------|----------------|------------|
| Ingestion pipeline design | Embedding model not pinned → silent version drift | Pin exact version, validate at startup |
| Ingestion pipeline design | PDF produces garbage text from complex layouts | Test with real corpus documents before finalizing chunking |
| Ingestion pipeline design | PostgreSQL/ChromaDB state divergence on crash | State machine with explicit status transitions |
| Docker Compose setup | Services start before dependencies are healthy | Health checks + `service_healthy` condition |
| Query pipeline | Citations have no chunk-level traceability | Number chunks in prompt, parse references, validate bounds |
| Query pipeline | Fallback threshold set once, never calibrated | Log every trigger, build calibration dataset |
| Query pipeline | Evaluation blocks query response latency | BackgroundTasks from day one of the query handler |
| Evaluation engine | LLM judge inflates scores for its own output | Rubric + calibration examples + human golden dataset |
| Auth setup | JWT secret hardcoded or weak | Environment variable + startup validation + test suite |
| Performance | Reranker CPU latency at real k values | Profile at production k before shipping |
| Data quality | Duplicate documents inflate ChromaDB | SHA-256 content hash deduplication at upload |

---

## Sources

**Confidence note:** WebSearch and WebFetch were unavailable in this environment. All findings are based on:

- Author's knowledge of ChromaDB v0.4+ API behavior (HIGH confidence — API changes are well-documented in official changelogs)
- Production RAG system patterns established in the community through mid-2025 (HIGH confidence — patterns are stable and widely reproduced)
- FastAPI security patterns from official FastAPI security documentation (HIGH confidence)
- LLM-as-judge evaluation bias — documented in academic literature (Zheng et al. 2023 "Judging LLM-as-a-Judge") and widely reproduced in production systems (HIGH confidence)
- Docker Compose health check behavior — official Docker documentation, stable behavior (HIGH confidence)

For verification before implementing, cross-reference:
- ChromaDB persistence: https://docs.trychroma.com/docs/run-chroma/persistent-client
- FastAPI BackgroundTasks: https://fastapi.tiangolo.com/tutorial/background-tasks/
- FastAPI security/JWT: https://fastapi.tiangolo.com/tutorial/security/oauth2-jwt/
- sentence-transformers model versioning: https://huggingface.co/sentence-transformers
