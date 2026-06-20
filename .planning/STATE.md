---
gsd_state_version: 1.0
milestone: v1.0
milestone_name: milestone
status: executing
last_updated: "2026-06-15T15:36:32.664Z"
progress:
  total_phases: 5
  completed_phases: 0
  total_plans: 5
  completed_plans: 0
  percent: 0
---

# STATE: GroundControl_RAG

**Last updated:** 2026-06-15
**Updated by:** gsd-plan-phase

---

## Project Reference

**Core value:** Users get grounded, cited answers from company knowledge — and the system tells you when it doesn't know rather than making things up.

**Current focus:** Phase 01 — foundation

---

## Current Position

Phase: 01 (foundation) — EXECUTING
Plan: 1 of 5
**Phase:** 1 — Foundation
**Plan:** 5 plans created (01-01 through 01-05)
**Status:** Executing Phase 01

**Progress:**

```
Phase 1 [▓▓        ] Planned (5 plans, 5 waves)
Phase 2 [          ] Not started
Phase 3 [          ] Not started
Phase 4 [          ] Not started
Phase 5 [          ] Not started
```

**Overall milestone progress:** 0/5 phases complete

---

## Performance Metrics

| Metric | Value |
|--------|-------|
| Phases total | 5 |
| Phases complete | 0 |
| Plans written | 5 |
| Plans complete | 0 |

---

## Accumulated Context

### Key Decisions (from research)

| Decision | Resolution |
|----------|-----------|
| Embeddings provider | sentence-transformers, all-MiniLM-L6-v2 (Anthropic has no embeddings API) |
| JWT library | PyJWT 2.13.0 (not python-jose — has CVEs) |
| Password hashing | bcrypt 5.0.0 directly (not passlib) |
| Async DB driver | asyncpg (SQLAlchemy 2.0 async mode) |
| Evaluation framework | Custom LLM-as-judge prompts via Claude (not RAGAS) |
| ChromaDB constructor | PersistentClient(path=...) — never EphemeralClient |
| ChromaDB embedding mode | Manual embedding (Embedding Service is single source of truth for vectors) |
| Evaluation async strategy | FastAPI BackgroundTasks (not Celery) |

### Open Questions (carry into planning)

1. Which Claude model to pin for generation (claude-3-5-sonnet vs claude-opus-4)?
2. Fallback thresholds — retrieval similarity < 0.35 and faithfulness < 0.5 are starting defaults; calibrate against real corpus in Phase 3
3. Document domain taxonomy (hr, product, technical) — must be decided before Phase 2 ships; changing requires re-ingestion
4. How are admin accounts bootstrapped — seeded migration, CLI command, or env var?
5. Acceptable p95 query latency target — pipeline may be 3-6s on CPU; set SLA before Phase 3 performance testing

### Critical Pitfalls to Watch

1. Embedding model mismatch — pin sentence-transformers version; assert on startup that configured model matches ChromaDB collection metadata
2. ChromaDB persistence misconfiguration — use PersistentClient with named volume; test persistence across container restart in Phase 2
3. PostgreSQL/ChromaDB state divergence — use ingestion state machine (pending → chunking → embedding → indexing → complete)
4. LLM-as-judge score inflation — use calibrated rubrics with score anchors; validate against 20-30 example golden dataset in Phase 4
5. Citation generation without source tracking — number chunks in prompt as [SOURCE N], parse and validate citation bounds at response layer

### Todos

- [ ] Decide Claude model version before Phase 3 starts
- [ ] Define document domain taxonomy before Phase 2 starts
- [ ] Decide admin account bootstrap strategy before Phase 1 ships
- [ ] Build golden dataset (20-30 examples) during or after Phase 3 for Phase 4 evaluation calibration

### Blockers

None.

---

## Session Continuity

**To resume work:**

1. Read this file to understand current position
2. Read `.planning/ROADMAP.md` to see phase goals and success criteria
3. Run `/gsd:execute-phase 1` to execute Phase 1 — Foundation (planning complete)

**Phase sequence:**
1 → 2 → 3 → 4 → 5

---
*Initialized: 2026-06-14 by gsd-roadmapper*
