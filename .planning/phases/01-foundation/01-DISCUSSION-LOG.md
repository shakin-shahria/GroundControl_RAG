# Phase 1: Foundation - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-06-15
**Phase:** 1-Foundation
**Areas discussed:** Project layout

---

## Project layout

### Q1: Backend organization style

| Option | Description | Selected |
|--------|-------------|----------|
| By layer (Recommended) | app/api/, app/models/, app/services/, app/core/ | ✓ |
| By domain/feature | app/auth/, app/documents/, app/queries/ — each owns routes+models+services | |
| Flat with modules | app/routes.py, app/models.py, app/services.py | |

**User's choice:** By layer (Recommended)
**Notes:** Sets the template for all future phases.

---

### Q2: Shared infrastructure location

| Option | Description | Selected |
|--------|-------------|----------|
| app/core/ (Recommended) | config.py, database.py, chroma.py, embedding.py — downstream phases import from core/ | ✓ |
| app/dependencies.py | Single FastAPI dependency injection file | |
| app/infrastructure/ | Separate fourth layer | |

**User's choice:** app/core/ (Recommended)
**Notes:** None.

---

### Q3: API route namespace

| Option | Description | Selected |
|--------|-------------|----------|
| /api/v1/* versioned prefix (Recommended) | All routes under /api/v1/, router files in app/api/v1/ | ✓ |
| /api/* unversioned | No v1 nesting | |
| Top-level routes | /auth/*, /documents/*, /queries/* | |

**User's choice:** /api/v1/* versioned prefix (Recommended)
**Notes:** None.

---

### Q4: Pydantic schema location

| Option | Description | Selected |
|--------|-------------|----------|
| Collocated in app/models/ (Recommended) | ORM model + Pydantic schemas in same file per domain entity | ✓ |
| Separate app/schemas/ directory | ORM in models/, Pydantic in schemas/ | |
| Schemas next to routes | Pydantic schemas defined inline in route files | |

**User's choice:** Collocated in app/models/ (Recommended)
**Notes:** None.

---

## Claude's Discretion

- **Admin bootstrap:** User deferred to Claude. Will choose simplest approach for Docker Compose (env-var seeded admin or Alembic seed migration).
- **JWT behavior:** Access-token-only for v1. Token expiry and claim shape left to Claude.
- **Env var structure:** Single `.env` file with `.env.example` committed. Left to Claude.

## Deferred Ideas

None — discussion stayed within phase scope.
