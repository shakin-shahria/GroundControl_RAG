# Phase 1: Foundation - Context

**Gathered:** 2026-06-15
**Status:** Ready for planning

<domain>
## Phase Boundary

Deliver a runnable development environment: `docker-compose up` brings FastAPI, PostgreSQL, and ChromaDB online with health checks; the PostgreSQL schema has all five tables via an Alembic migration; a user can register, log in, and call a protected route with a JWT. No document ingestion, no query pipeline, no frontend — just the infrastructure every subsequent phase builds on.

</domain>

<decisions>
## Implementation Decisions

### Project Layout
- **D-01:** Backend uses a **by-layer** structure: `app/api/` (routes), `app/models/` (SQLAlchemy ORM models + Pydantic schemas collocated), `app/services/` (business logic), `app/core/` (shared infrastructure). This is the template all future phases follow.
- **D-02:** Shared infrastructure lives in `app/core/`: `config.py` (Pydantic Settings), `database.py` (SQLAlchemy async engine + session factory), `chroma.py` (ChromaDB **HttpClient** connecting to the separate `chromadb` Docker service — not PersistentClient), `embedding.py` (sentence-transformers model singleton). Downstream phases import clients from `app/core/`.
- **D-03:** API routes are versioned under `/api/v1/`. Router files live in `app/api/v1/` (e.g., `app/api/v1/auth.py`, `app/api/v1/health.py`). The main `app/api/router.py` aggregates all v1 routers and mounts them.
- **D-04:** Pydantic request/response schemas are **collocated** with their SQLAlchemy model in `app/models/`. Example: `app/models/user.py` contains both the `User` table class and `UserCreate` / `UserResponse` schemas. One file per domain entity.

### Claude's Discretion
- **Admin bootstrap:** How the initial admin account is seeded is left to Claude. Reasonable approaches: env-var credentials read at startup and inserted if no admin exists, or a seeded Alembic migration with a default admin. Choose the approach that is simplest to override in Docker Compose.
- **JWT behavior:** Access-token-only for v1 simplicity (no refresh tokens). Token expiry window and claim shape are left to Claude. Use the `sub` claim for user ID.
- **Env var structure:** Single `.env` file at repo root, loaded by `python-dotenv`. `.env.example` committed to the repo. Docker Compose reads the same `.env` file via `env_file:`.

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Requirements
- `.planning/REQUIREMENTS.md` — Phase 1 requirements: FOUND-01 (schema), FOUND-02 (Docker Compose + health), FOUND-03 (JWT auth), FOUND-04 (health endpoints). Read the full success criteria before planning.
- `.planning/ROADMAP.md` — Phase 1 goal statement and 5 success criteria (exact pass/fail conditions for verification).

### Tech Stack Decisions
- `CLAUDE.md` — Full locked tech stack with pinned versions: FastAPI 0.136.3, SQLAlchemy 2.0.50, asyncpg 0.31.0, Alembic 1.18.4, PyJWT 2.13.0, bcrypt 5.0.0, ChromaDB 1.5.9, Python 3.12. **Use these exact versions.** Also documents what NOT to use (python-jose, passlib, LangChain, Celery).

### Architecture
- `base.md` — System specification with architecture diagrams and pipeline overview. Read for context on how Phase 1 components connect to later phases.

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- None — greenfield project. No existing code to reuse.

### Established Patterns
- None yet — Phase 1 establishes the patterns all subsequent phases follow.

### Integration Points
- Phase 1's `app/core/database.py` and `app/core/chroma.py` are the contracts Phase 2 imports when it starts writing embedding and ingestion services.
- Phase 1's `app/models/user.py` User table is referenced by the JWT claims generated in Phase 1 and used in all subsequent protected routes.

</code_context>

<specifics>
## Specific Ideas

No specific requirements — open to standard approaches within the locked stack.

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope.

</deferred>

---

*Phase: 1-Foundation*
*Context gathered: 2026-06-15*
