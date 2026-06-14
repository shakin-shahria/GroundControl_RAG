# GroundControl_RAG

## What This Is

GroundControl_RAG is a production-grade Retrieval Augmented Generation platform that lets users query multi-domain knowledge bases (internal docs, customer support content, technical references) using natural language. Every response is grounded in retrieved source documents with citations, scored by an evaluation engine, and protected by fallback logic that refuses to hallucinate when evidence is weak. It targets employees, customers, and developer/AI teams within the same deployment.

## Core Value

Users get grounded, cited answers from company knowledge — and the system tells you when it doesn't know rather than making things up.

## Requirements

### Validated

(None yet — ship to validate)

### Active

- [ ] Users can upload documents (PDF, DOCX, TXT) via admin UI
- [ ] Documents are chunked, embedded, and stored in ChromaDB vector store
- [ ] Users can query the knowledge base with natural language questions
- [ ] Queries are embedded, similarity-searched, and top chunks retrieved
- [ ] Retrieved chunks are reranked to select the top 5 most relevant
- [ ] Context is built from system prompt + reranked chunks + user question
- [ ] Claude generates responses grounded only in provided context
- [ ] Every response includes citations (document name, page, section)
- [ ] Response validator checks grounding and confidence before returning
- [ ] Low-confidence responses trigger fallback (clarification request or "I don't know")
- [ ] Every request is evaluated with custom faithfulness, relevance, and correctness metrics
- [ ] Per-request telemetry is captured (latency per stage, token usage, retrieval scores, errors)
- [ ] JWT-based authentication gates access to the query and admin interfaces
- [ ] Lightweight admin dashboard shows total requests, average latency, failure rate, and recent request logs
- [ ] All services are containerized and runnable via Docker Compose

### Out of Scope

- Multi-modal document support (images, video) — complexity for v1; add in v2
- Fine-tuning or retraining models — operational overhead; focus on RAG pipeline quality first
- Agent/agentic workflows — not in scope for v1 reliability-focused platform
- Human-in-the-loop escalation routing — deferred to v2 fallback expansion
- Full Prometheus/Grafana monitoring stack — lightweight dashboard covers v1 observability needs; external stack in v2
- Multi-tenancy / per-org isolation — single-tenant v1; multi-tenant in future
- Anthropic embeddings API — Anthropic does not offer a dedicated embeddings endpoint; will use an alternative (HuggingFace or OpenAI embeddings, to be decided in research phase)

## Context

- The user has a detailed system specification in `base.md` and `images/system_overview.png` covering the full architecture: ingestion pipeline, query pipeline, evaluation engine, and observability
- Corpus size is small (< 1K docs) for v1 — ChromaDB local mode is sufficient; no managed vector DB needed
- LLM: Claude (Anthropic) via the Anthropic Python SDK for generation and LLM-as-judge evaluation
- Embeddings: Anthropic does not offer a standalone embeddings API — research phase must resolve this (likely `all-MiniLM-L6-v2` via sentence-transformers or OpenAI embeddings)
- Evaluation uses custom metrics only (no RAGAS framework dependency)
- Document ingestion is via admin UI drag-and-drop; no CLI batch import in v1
- Deployment target is Docker Compose (self-contained); cloud deployment TBD post-v1

## Constraints

- **Tech Stack (Backend)**: Python + FastAPI — specified in base.md
- **Tech Stack (Frontend)**: React + TypeScript + Tailwind — specified in base.md
- **Tech Stack (Vector DB)**: ChromaDB — chosen for simplicity at < 1K doc scale
- **Tech Stack (LLM)**: Claude (Anthropic) — generation and evaluation
- **Tech Stack (Database)**: PostgreSQL — user, document, request, evaluation tables
- **Deployment**: Docker Compose — must run with a single `docker-compose up`
- **Evaluation**: Custom metrics only — no RAGAS or external eval framework dependency
- **Auth**: JWT-based — sessions for users, no API key auth in v1
- **Embeddings**: TBD (constraint: not Anthropic native API) — resolve in research

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| ChromaDB over FAISS/Milvus | < 1K docs for v1; ChromaDB offers persistence + simple API without managed infra overhead | — Pending |
| Claude for LLM generation | Best citation-grounded outputs, long context window, strong instruction following | — Pending |
| Custom evaluation metrics over RAGAS | Reduce external dependencies; easier to tune scoring thresholds for this domain | — Pending |
| JWT auth (not API key) | Multi-user product with different roles; session-based login fits the use case | — Pending |
| Docker Compose deployment | Decouple infra decision from build; lets the system run anywhere | — Pending |
| Lightweight dashboard in v1 | Full Prometheus/Grafana adds operational complexity; totals + latency charts sufficient to validate observability needs | — Pending |

## Evolution

This document evolves at phase transitions and milestone boundaries.

**After each phase transition** (via `/gsd:transition`):
1. Requirements invalidated? → Move to Out of Scope with reason
2. Requirements validated? → Move to Validated with phase reference
3. New requirements emerged? → Add to Active
4. Decisions to log? → Add to Key Decisions
5. "What This Is" still accurate? → Update if drifted

**After each milestone** (via `/gsd:complete-milestone`):
1. Full review of all sections
2. Core Value check — still the right priority?
3. Audit Out of Scope — reasons still valid?
4. Update Context with current state

---
*Last updated: 2026-06-14 after initialization*
