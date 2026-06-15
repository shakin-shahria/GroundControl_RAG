---
phase: 01-foundation
plan: "01"
subsystem: infrastructure
tags: [docker, compose, fastapi, postgresql, chromadb, alembic, python]
dependency_graph:
  requires: []
  provides:
    - docker-compose.yml with health-checked service orchestration
    - backend/Dockerfile for Python 3.12-slim FastAPI container
    - backend/requirements.txt with pinned dependencies
    - backend/alembic/ async migration scaffold
    - by-layer Python package skeleton
  affects:
    - All subsequent plans depend on docker-compose.yml and requirements.txt
tech_stack:
  added:
    - FastAPI 0.136.3
    - SQLAlchemy 2.0.50 async
    - asyncpg 0.31.0
    - Alembic 1.18.4
    - PyJWT 2.13.0
    - bcrypt 5.0.0
    - ChromaDB 1.5.9
    - sentence-transformers 5.5.1
    - structlog 26.1.0
    - Python 3.12-slim (Docker)
    - PostgreSQL 16-alpine (Docker)
  patterns:
    - Docker Compose health-check ordering with condition: service_healthy
    - pg_isready for PostgreSQL health; curl /api/v1/heartbeat for ChromaDB
    - Alembic env.py reads DATABASE_URL from Pydantic Settings at runtime (no URL in alembic.ini)
    - By-layer directory structure: api/, models/, services/, core/
key_files:
  created:
    - docker-compose.yml
    - .env.example
    - backend/Dockerfile
    - backend/requirements.txt
    - backend/alembic.ini
    - backend/alembic/env.py
    - backend/alembic/script.py.mako
    - backend/app/__init__.py
    - backend/app/api/__init__.py
    - backend/app/api/v1/__init__.py
    - backend/app/core/__init__.py
    - backend/app/models/__init__.py
    - backend/app/services/__init__.py
    - backend/tests/__init__.py
    - .gitignore
  modified: []
decisions:
  - "Alembic sqlalchemy.url omitted from alembic.ini — injected at runtime via env.py reading from Pydantic Settings"
  - "ChromaDB pinned to 1.5.9 (exact version) rather than using latest tag for reproducibility"
  - "bcrypt and PyJWT used directly (no passlib, no python-jose) per CLAUDE.md security constraints"
metrics:
  duration: "33 minutes"
  completed: "2026-06-15"
  tasks_completed: 2
  tasks_total: 2
  files_created: 14
  files_modified: 0
---

# Phase 1 Plan 01: Docker Compose Scaffold and Backend Skeleton Summary

**One-liner:** Three-service Docker Compose with pg_isready and /api/v1/heartbeat health checks gating backend startup, Python 3.12-slim Dockerfile, and pinned requirements for FastAPI 0.136.3 + SQLAlchemy 2.0 async.

## Tasks Completed

| Task | Name | Commit | Key Files |
|------|------|--------|-----------|
| 0 | Verify package legitimacy (checkpoint) | (human gate) | — |
| 1 | Docker Compose, Dockerfile, requirements.txt | 366ffd5 | docker-compose.yml, backend/Dockerfile, backend/requirements.txt, .env.example |
| 2 | Directory skeleton and alembic.ini | de00165 | backend/alembic/, backend/app/**/__init__.py, .gitignore |

## What Was Built

**Task 1** created the full container orchestration layer:
- `docker-compose.yml` — three services (postgres, chromadb, backend) with named volumes (pgdata, chromadata) and health-checked startup ordering. Backend `depends_on` both postgres and chromadb with `condition: service_healthy`. PostgreSQL healthcheck uses `pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}`; ChromaDB healthcheck uses `curl -f http://localhost:8000/api/v1/heartbeat` (not /health).
- `backend/Dockerfile` — `FROM python:3.12-slim`, installs curl (for healthcheck CMD in the image), copies and installs requirements, exposes port 8000.
- `backend/requirements.txt` — all dependencies pinned to exact CLAUDE.md versions including fastapi==0.136.3, PyJWT==2.13.0, bcrypt==5.0.0, chromadb==1.5.9, sqlalchemy==2.0.50, asyncpg==0.31.0.
- `.env.example` — committed template with DATABASE_URL using `postgresql+asyncpg://` prefix; no real secrets.

**Task 2** created the Python package skeleton and Alembic async migration config:
- `backend/app/{api/v1,core,models,services}/__init__.py` — by-layer package structure matching D-01 decision.
- `backend/tests/__init__.py` — test package.
- `backend/alembic.ini` — `script_location = alembic`, `sqlalchemy.url` intentionally omitted (injected at runtime by env.py from Pydantic Settings to avoid duplicating secrets in config files).
- `backend/alembic/env.py` — full async migration template using `async_engine_from_config` and `asyncio.run()`, reads `settings.DATABASE_URL` and imports `app.models.Base` for autogenerate support.
- `backend/alembic/script.py.mako` — standard migration file template.
- `.gitignore` — protects `.env` (secrets); `.env.example` is NOT ignored.

## Deviations from Plan

### Auto-applied adjustments

**1. [Rule 2 - Missing Critical Functionality] Added alembic/script.py.mako**
- **Found during:** Task 2
- **Issue:** The plan specified `alembic init -t async` which generates script.py.mako alongside env.py. Since alembic is not installed in the host environment (only inside the Docker container), the files were hand-crafted.
- **Fix:** Created `backend/alembic/script.py.mako` with the standard async template content. Without this file, `alembic revision` inside the container would fail with a missing template error.
- **Files modified:** `backend/alembic/script.py.mako`
- **Commit:** de00165

**2. [Rule 2 - Missing Critical Functionality] Added additional .gitignore entries**
- **Found during:** Task 2
- **Issue:** Plan specified minimum .gitignore entries. Added `.venv/`, `venv/`, `env/`, `.vscode/`, `.idea/`, and `node_modules/` — all standard exclusions for this project type.
- **Fix:** Extended .gitignore with common Python and Node project entries.
- **Files modified:** `.gitignore`
- **Commit:** de00165

## Known Stubs

None — this plan contains only configuration and structure files, no application logic.

## Threat Surface Scan

No new network endpoints, auth paths, or trust boundaries introduced beyond what the threat model anticipated. `.env.example` contains only placeholder values; `.gitignore` protects `.env` from accidental commits.

## Self-Check

**Files exist:**
- docker-compose.yml: FOUND
- .env.example: FOUND
- backend/Dockerfile: FOUND
- backend/requirements.txt: FOUND
- backend/alembic.ini: FOUND
- backend/alembic/env.py: FOUND
- backend/app/api/v1/__init__.py: FOUND
- backend/app/core/__init__.py: FOUND
- backend/app/models/__init__.py: FOUND
- backend/app/services/__init__.py: FOUND
- .gitignore: FOUND

**Commits exist:**
- 366ffd5: FOUND (Task 1)
- de00165: FOUND (Task 2)

## Self-Check: PASSED
