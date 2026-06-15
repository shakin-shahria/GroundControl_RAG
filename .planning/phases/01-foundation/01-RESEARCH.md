# Phase 1: Foundation - Research

**Researched:** 2026-06-15
**Domain:** Python/FastAPI infrastructure bootstrap — async database, JWT auth, Docker Compose orchestration
**Confidence:** HIGH

---

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

- **D-01:** Backend uses a **by-layer** structure: `app/api/` (routes), `app/models/` (SQLAlchemy ORM models + Pydantic schemas collocated), `app/services/` (business logic), `app/core/` (shared infrastructure). This is the template all future phases follow.
- **D-02:** Shared infrastructure lives in `app/core/`: `config.py` (Pydantic Settings), `database.py` (SQLAlchemy async engine + session factory), `chroma.py` (ChromaDB PersistentClient singleton), `embedding.py` (sentence-transformers model singleton). Downstream phases import clients from `app/core/`.
- **D-03:** API routes are versioned under `/api/v1/`. Router files live in `app/api/v1/` (e.g., `app/api/v1/auth.py`, `app/api/v1/health.py`). The main `app/api/router.py` aggregates all v1 routers and mounts them.
- **D-04:** Pydantic request/response schemas are **collocated** with their SQLAlchemy model in `app/models/`. Example: `app/models/user.py` contains both the `User` table class and `UserCreate` / `UserResponse` schemas. One file per domain entity.

### Claude's Discretion

- **Admin bootstrap:** How the initial admin account is seeded is left to Claude. Reasonable approaches: env-var credentials read at startup and inserted if no admin exists, or a seeded Alembic migration with a default admin. Choose the approach that is simplest to override in Docker Compose.
- **JWT behavior:** Access-token-only for v1 simplicity (no refresh tokens). Token expiry window and claim shape are left to Claude. Use the `sub` claim for user ID.
- **Env var structure:** Single `.env` file at repo root, loaded by `python-dotenv`. `.env.example` committed to the repo. Docker Compose reads the same `.env` file via `env_file:`.

### Deferred Ideas (OUT OF SCOPE)

None — discussion stayed within phase scope.
</user_constraints>

---

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| FOUND-01 | System has a PostgreSQL schema with tables for users, documents, document_chunks, ai_requests, and evaluations | SQLAlchemy ORM models + Alembic async migration pattern; all five table schemas documented |
| FOUND-02 | All services (FastAPI, ChromaDB, PostgreSQL) run via a single `docker-compose up` with health checks and named volumes | Docker Compose v2 `depends_on: condition: service_healthy`; pg_isready, HTTP curl healthcheck patterns |
| FOUND-03 | User can register with email and password, log in, and access protected routes via JWT (PyJWT + bcrypt) | PyJWT 2.13.0 encode/decode; bcrypt 5.0.0 hashpw/checkpw; FastAPI `get_current_user` dependency pattern |
| FOUND-04 | Each service exposes a `GET /health` endpoint used by Docker Compose `depends_on` for startup ordering | FastAPI health route; Docker healthcheck with curl -f; ChromaDB /api/v1/heartbeat; depends_on: service_healthy |
</phase_requirements>

---

## Summary

Phase 1 establishes the complete runnable infrastructure that all future phases depend on: a Docker Compose environment with FastAPI, PostgreSQL 16, and ChromaDB; a PostgreSQL schema created via Alembic async migration; and JWT-based user registration/authentication. There is no frontend, no document ingestion, and no query pipeline in this phase — only the foundation contracts (database.py, chroma.py, config.py, embedding.py stub) that phases 2–5 import.

The technical domain is well-understood and stable. All pinned versions from CLAUDE.md have been confirmed on PyPI and npm registries as of 2026-06-15. The primary complexity lies in two areas: (1) the async SQLAlchemy/Alembic wiring, which has a specific boilerplate structure that diverges from the sync pattern, and (2) Docker Compose startup ordering, which requires proper health checks to prevent race conditions between the FastAPI app, PostgreSQL, and ChromaDB.

**Primary recommendation:** Use `alembic init -t async` to generate the correct async env.py template. Use `async_sessionmaker` (not the older `sessionmaker`) for the session factory. Use `asyncio.run(run_async_migrations())` in Alembic env.py for the online mode. Implement admin seeding as a startup lifespan event that checks for admin existence and inserts from env vars if absent.

---

## Architectural Responsibility Map

| Capability | Primary Tier | Secondary Tier | Rationale |
|------------|-------------|----------------|-----------|
| User registration + password hashing | API / Backend | — | bcrypt hashing must never happen client-side; all credential handling in FastAPI service layer |
| JWT creation + validation | API / Backend | — | HS256 signing with server secret; dependency injection pattern protects routes |
| PostgreSQL schema creation | Database / Storage | API startup | Alembic migrations run at container start before FastAPI accepts requests |
| Service health reporting | API / Backend | Docker Compose | Each service exposes /health; Docker Compose polls it for depends_on ordering |
| ChromaDB client singleton | API / Backend (core/) | — | PersistentClient initialized once in app/core/chroma.py; imported by services in later phases |
| Config / env loading | API / Backend (core/) | — | Pydantic Settings reads .env; single source of truth for all environment configuration |
| Container orchestration | CDN / Static (Docker layer) | — | Docker Compose v2 handles startup ordering, named volumes, health conditions |
| sentence-transformers stub | API / Backend (core/) | — | embedding.py stub loads no model in Phase 1; Phase 2 instantiates actual model |

---

## Standard Stack

### Core

| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| Python | 3.12 | Runtime | [VERIFIED: pypi.org] Current stable; sentence-transformers requires >=3.10 |
| FastAPI | 0.136.3 | API framework | [VERIFIED: pypi.org] Locked in CLAUDE.md; async-native, auto OpenAPI, Pydantic v2 integration. Note: PyPI latest is 0.137.1 — pin to 0.136.3 per CLAUDE.md |
| Uvicorn | 0.49.0 | ASGI server | [VERIFIED: pypi.org] Standard pairing with FastAPI; install with `uvicorn[standard]` for uvloop |
| Pydantic | 2.13.4 | Data validation | [VERIFIED: pypi.org] FastAPI 0.100+ requires Pydantic v2 |
| pydantic-settings | 2.x | Env var loading | [ASSUMED] Required for `BaseSettings`; separate package from pydantic v2 core |
| SQLAlchemy | 2.0.50 | ORM + async engine | [VERIFIED: pypi.org] Locked; fully async-native with AsyncSession |
| asyncpg | 0.31.0 | PostgreSQL async driver | [VERIFIED: pypi.org] Fastest async Postgres driver; required for SQLAlchemy 2.0 async |
| Alembic | 1.18.4 | Database migrations | [VERIFIED: pypi.org] Standard SQLAlchemy migration tool |
| PyJWT | 2.13.0 | JWT encode/decode | [VERIFIED: pypi.org] Locked; HS256; replaces python-jose which has CVEs |
| bcrypt | 5.0.0 | Password hashing | [VERIFIED: pypi.org] Direct usage (no passlib); 5.x enforces 72-byte limit |
| ChromaDB | 1.5.9 | Vector store | [VERIFIED: pypi.org] Locked; PersistentClient mode for local persistence |
| python-dotenv | 1.2.2 | .env file loading | [ASSUMED] Standard for Docker Compose env var management |
| structlog | 26.1.0 | Structured logging | [ASSUMED] JSON-formatted logs for per-request telemetry |
| python-multipart | 0.0.32 | File upload parsing | [ASSUMED] FastAPI dependency for UploadFile (needed Phase 2 onward; install now) |

### Supporting

| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| httpx | 0.28.1 | Async HTTP client | [ASSUMED] For inter-service calls and testing async endpoints |
| pytest | 9.1.0 | Test runner | [ASSUMED] Standard Python testing |
| pytest-asyncio | 1.4.0 | Async test support | [ASSUMED] Required for testing FastAPI async endpoints |

### Alternatives Considered (locked — do not use)

| Instead of | Could Use | Why Locked Out |
|------------|-----------|----------------|
| PyJWT | python-jose | python-jose has CVE history (CLAUDE.md locked) |
| bcrypt direct | passlib | passlib maintenance has slowed (CLAUDE.md locked) |
| SQLAlchemy async | Tortoise-ORM | SQLAlchemy is ecosystem standard, best Alembic integration |
| asyncpg | psycopg2 | psycopg2 is synchronous; cannot use with SQLAlchemy 2.0 async mode |

**Installation (backend):**
```bash
pip install \
  fastapi==0.136.3 \
  uvicorn[standard]==0.49.0 \
  pydantic==2.13.4 \
  pydantic-settings \
  sqlalchemy==2.0.50 \
  asyncpg==0.31.0 \
  alembic==1.18.4 \
  PyJWT==2.13.0 \
  bcrypt==5.0.0 \
  chromadb==1.5.9 \
  python-dotenv==1.2.2 \
  structlog==26.1.0 \
  python-multipart==0.0.32 \
  httpx==0.28.1
```

---

## Package Legitimacy Audit

> slopcheck was unavailable at research time. All packages below are tagged `[ASSUMED]` for legitimacy and the planner must gate each `pip install` command behind confirming these are official packages. Core packages (FastAPI, SQLAlchemy, ChromaDB, PyJWT, bcrypt) are well-established and confirmed on PyPI with multi-year histories and millions of downloads — treat as effectively verified.

| Package | Registry | Age | Downloads | Source Repo | slopcheck | Disposition |
|---------|----------|-----|-----------|-------------|-----------|-------------|
| fastapi | PyPI | ~6 yrs | Very high | github.com/tiangolo/fastapi | unavailable | [ASSUMED] Approved — authoritative source confirmed |
| sqlalchemy | PyPI | ~18 yrs | Very high | github.com/sqlalchemy/sqlalchemy | unavailable | [ASSUMED] Approved |
| alembic | PyPI | ~13 yrs | Very high | github.com/sqlalchemy/alembic | unavailable | [ASSUMED] Approved |
| asyncpg | PyPI | ~8 yrs | High | github.com/MagicStack/asyncpg | unavailable | [ASSUMED] Approved |
| PyJWT | PyPI | ~10 yrs | Very high | github.com/jpadilla/pyjwt | unavailable | [ASSUMED] Approved |
| bcrypt | PyPI | ~10 yrs | Very high | github.com/pyca/bcrypt | unavailable | [ASSUMED] Approved |
| chromadb | PyPI | ~3 yrs | High | github.com/chroma-core/chroma | unavailable | [ASSUMED] Approved |
| pydantic-settings | PyPI | ~3 yrs | Very high | github.com/pydantic/pydantic-settings | unavailable | [ASSUMED] Approved — official Pydantic org |
| uvicorn | PyPI | ~7 yrs | Very high | github.com/encode/uvicorn | unavailable | [ASSUMED] Approved |
| structlog | PyPI | ~12 yrs | High | github.com/hynek/structlog | unavailable | [ASSUMED] Approved |
| python-dotenv | PyPI | ~10 yrs | Very high | github.com/theskumar/python-dotenv | unavailable | [ASSUMED] Approved |

**Packages removed due to slopcheck [SLOP] verdict:** none
**Packages flagged as suspicious [SUS]:** none

*slopcheck was unavailable at research time — all packages tagged `[ASSUMED]`. The planner should note that every listed package is a widely-known, multi-year-old library from official maintainer organizations. No novel or obscure packages are in this stack.*

---

## Architecture Patterns

### System Architecture Diagram

```
.env file (repo root)
       |
       v
[Docker Compose v2]
       |
       +---> [PostgreSQL 16] <---healthcheck: pg_isready---> healthy
       |            |
       |     [Alembic migration] (runs before FastAPI starts, in entrypoint)
       |
       +---> [ChromaDB 1.5.9] <---healthcheck: curl /api/v1/heartbeat---> healthy
       |            |
       |     /data volume (named Docker volume)
       |
       +---> [FastAPI + Uvicorn]
                    |
                    depends_on: postgres (service_healthy), chromadb (service_healthy)
                    |
                    v
            [lifespan startup]
                    |
                    +---> create_async_engine (asyncpg DSN)
                    +---> PersistentClient(path="/chroma/data")
                    +---> seed admin if not exists (env vars)
                    |
                    v
            [/api/v1/ routes]
                    |
                    +---> GET /health   (no auth)
                    +---> POST /api/v1/auth/register
                    +---> POST /api/v1/auth/login  --> returns JWT
                    +---> GET /api/v1/users/me  <-- protected (Depends(get_current_user))
```

### Recommended Project Structure

```
backend/
├── app/
│   ├── __init__.py
│   ├── main.py               # FastAPI app creation, lifespan, router mount
│   ├── api/
│   │   ├── __init__.py
│   │   ├── router.py         # aggregates all v1 routers
│   │   └── v1/
│   │       ├── __init__.py
│   │       ├── auth.py       # POST /register, POST /login
│   │       ├── health.py     # GET /health
│   │       └── users.py      # GET /users/me (protected)
│   ├── core/
│   │   ├── __init__.py
│   │   ├── config.py         # Pydantic Settings (BaseSettings)
│   │   ├── database.py       # async engine, async_session_maker, get_db
│   │   ├── chroma.py         # PersistentClient singleton
│   │   ├── embedding.py      # sentence-transformers stub (Phase 2 fills in)
│   │   └── security.py       # JWT encode/decode, bcrypt helpers
│   ├── models/
│   │   ├── __init__.py
│   │   ├── user.py           # User ORM + UserCreate/UserResponse schemas
│   │   ├── document.py       # Document + DocumentChunk ORM (tables only, Phase 2 uses)
│   │   ├── request.py        # AIRequest ORM (Phase 4 uses)
│   │   └── evaluation.py     # Evaluation ORM (Phase 4 uses)
│   └── services/
│       ├── __init__.py
│       └── user_service.py   # register, authenticate, get_by_id
├── alembic/
│   ├── env.py                # async template (alembic init -t async)
│   ├── script.py.mako
│   └── versions/
│       └── 0001_initial_schema.py
├── alembic.ini
├── requirements.txt
├── Dockerfile
└── tests/
    ├── __init__.py
    ├── conftest.py           # async test client, test DB setup
    └── test_auth.py          # register, login, protected route tests
docker-compose.yml
.env
.env.example
```

### Pattern 1: Pydantic Settings (app/core/config.py)

```python
# Source: https://docs.pydantic.dev/latest/concepts/pydantic_settings/
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        case_sensitive=False,
    )

    # Database
    database_url: str  # postgresql+asyncpg://user:pass@host:5432/dbname
    
    # JWT
    jwt_secret_key: str
    jwt_algorithm: str = "HS256"
    jwt_access_token_expire_minutes: int = 60

    # Admin bootstrap
    admin_email: str = "admin@example.com"
    admin_password: str  # required from env

    # ChromaDB
    chroma_path: str = "/chroma/data"

settings = Settings()
```

**Note:** `pydantic-settings` is a separate install from `pydantic` in v2. Do not import from `pydantic` — import from `pydantic_settings`. [CITED: docs.pydantic.dev]

### Pattern 2: SQLAlchemy Async Engine + Session Factory (app/core/database.py)

```python
# Source: https://docs.sqlalchemy.org/en/20/orm/extensions/asyncio.html
from typing import AsyncGenerator
from sqlalchemy.ext.asyncio import (
    AsyncSession,
    async_sessionmaker,
    create_async_engine,
)
from sqlalchemy.orm import DeclarativeBase

from app.core.config import settings

engine = create_async_engine(
    settings.database_url,  # MUST use postgresql+asyncpg:// prefix
    echo=False,
    pool_pre_ping=True,
)

async_session_maker = async_sessionmaker(
    engine,
    class_=AsyncSession,
    expire_on_commit=False,  # prevent lazy-load errors after commit
)

class Base(DeclarativeBase):
    pass

async def get_db() -> AsyncGenerator[AsyncSession, None]:
    async with async_session_maker() as session:
        yield session
```

**Critical DSN format:** `postgresql+asyncpg://user:password@host:5432/dbname`
The `+asyncpg` prefix is mandatory — using `postgresql://` without it will use the synchronous psycopg2 driver and fail. [CITED: docs.sqlalchemy.org]

### Pattern 3: FastAPI Lifespan Event (app/main.py)

```python
# Source: https://fastapi.tiangolo.com/advanced/events/
from contextlib import asynccontextmanager
from fastapi import FastAPI
from app.api.router import router
from app.core.database import engine
from app.core.chroma import init_chroma
from app.services.user_service import seed_admin_if_absent

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup assertions
    await seed_admin_if_absent()
    init_chroma()   # validates PersistentClient path is writable
    yield
    # Shutdown cleanup
    await engine.dispose()

app = FastAPI(lifespan=lifespan)
app.include_router(router, prefix="/api/v1")

@app.get("/health")
async def health():
    return {"status": "ok"}
```

**Do not** use `@app.on_event("startup")` — it is deprecated in FastAPI. Use the `lifespan` async context manager exclusively. [CITED: fastapi.tiangolo.com/advanced/events/]

### Pattern 4: Alembic Async env.py (alembic/env.py)

```python
# Source: https://github.com/sqlalchemy/alembic/blob/main/alembic/templates/async/env.py
import asyncio
from logging.config import fileConfig
from sqlalchemy import pool
from sqlalchemy.engine import Connection
from sqlalchemy.ext.asyncio import async_engine_from_config
from alembic import context
from app.models import Base  # import ALL models so autogenerate finds them

config = context.config
if config.config_file_name is not None:
    fileConfig(config.config_file_name)

target_metadata = Base.metadata

def do_run_migrations(connection: Connection) -> None:
    context.configure(connection=connection, target_metadata=target_metadata)
    with context.begin_transaction():
        context.run_migrations()

async def run_async_migrations() -> None:
    connectable = async_engine_from_config(
        config.get_section(config.config_ini_section, {}),
        prefix="sqlalchemy.",
        poolclass=pool.NullPool,
    )
    async with connectable.connect() as connection:
        await connection.run_sync(do_run_migrations)
    await connectable.dispose()

def run_migrations_online() -> None:
    asyncio.run(run_async_migrations())

if context.is_offline_mode():
    run_migrations_offline()
else:
    run_migrations_online()
```

Bootstrap command: `alembic init -t async` — generates the correct template. [CITED: alembic.sqlalchemy.org]

### Pattern 5: PyJWT encode/decode (app/core/security.py)

```python
# Source: https://pyjwt.readthedocs.io/en/stable/usage.html
import jwt
from datetime import datetime, timezone, timedelta
from fastapi import HTTPException, status
from app.core.config import settings

def create_access_token(user_id: int) -> str:
    payload = {
        "sub": str(user_id),
        "exp": datetime.now(tz=timezone.utc) + timedelta(
            minutes=settings.jwt_access_token_expire_minutes
        ),
    }
    return jwt.encode(payload, settings.jwt_secret_key, algorithm=settings.jwt_algorithm)

def decode_access_token(token: str) -> dict:
    try:
        return jwt.decode(token, settings.jwt_secret_key, algorithms=[settings.jwt_algorithm])
    except jwt.ExpiredSignatureError:
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED, detail="Token expired")
    except jwt.InvalidTokenError:
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED, detail="Invalid token")
```

**Key point:** `algorithms` parameter in `jwt.decode()` is a list, not a string. Omitting it or using a string raises a `DecodeError`. [CITED: pyjwt.readthedocs.io]

### Pattern 6: bcrypt 5.x password hashing (app/core/security.py)

```python
# Source: https://pypi.org/project/bcrypt/
import bcrypt

def hash_password(password: str) -> str:
    # bcrypt requires bytes input; encode first
    password_bytes = password.encode("utf-8")
    # bcrypt 5.x raises ValueError if password > 72 bytes — enforce this
    if len(password_bytes) > 72:
        raise ValueError("Password must be 72 bytes or fewer")
    hashed = bcrypt.hashpw(password_bytes, bcrypt.gensalt())
    return hashed.decode("utf-8")  # store as string in DB

def verify_password(plain: str, hashed: str) -> bool:
    return bcrypt.checkpw(plain.encode("utf-8"), hashed.encode("utf-8"))
```

**bcrypt 5.x breaking change:** Passwords longer than 72 bytes now raise `ValueError` (previously silently truncated). [CITED: pypi.org/project/bcrypt/]

### Pattern 7: JWT dependency for protected routes (app/api/deps.py)

```python
# Source: https://fastapi.tiangolo.com/tutorial/security/oauth2-jwt/
from typing import Annotated
from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer
from sqlalchemy.ext.asyncio import AsyncSession
from app.core.database import get_db
from app.core.security import decode_access_token

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/api/v1/auth/login")

async def get_current_user(
    token: Annotated[str, Depends(oauth2_scheme)],
    db: Annotated[AsyncSession, Depends(get_db)],
):
    payload = decode_access_token(token)  # raises 401 on failure
    user_id = payload.get("sub")
    if user_id is None:
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED, detail="Invalid token payload")
    # fetch user from DB
    from app.services.user_service import get_user_by_id
    user = await get_user_by_id(db, int(user_id))
    if user is None:
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED, detail="User not found")
    return user
```

### Pattern 8: ChromaDB PersistentClient singleton (app/core/chroma.py)

```python
# Source: https://docs.trychroma.com/reference/python/client
import chromadb
from app.core.config import settings

_client: chromadb.PersistentClient | None = None

def get_chroma_client() -> chromadb.PersistentClient:
    global _client
    if _client is None:
        _client = chromadb.PersistentClient(path=settings.chroma_path)
    return _client

def init_chroma() -> None:
    """Called at lifespan startup to validate path is writable."""
    client = get_chroma_client()
    # heartbeat raises if client cannot reach storage
    client.heartbeat()
```

**Docker volume:** Mount `/chroma/data` inside the container to a named Docker volume. The ChromaDB **server** container (when running as a service) uses `/data` — but when using `PersistentClient` from Python code (not the server container), the path is whatever you pass. In Docker Compose with a single FastAPI container using PersistentClient, mount a named volume to `/chroma/data` in the FastAPI container. [CITED: docs.trychroma.com/guides/deploy/docker]

### Pattern 9: Docker Compose health checks and startup ordering

```yaml
# Source: https://docs.docker.com/compose/how-tos/startup-order/
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 5s
      timeout: 5s
      retries: 10
      start_period: 10s

  chromadb:
    image: chromadb/chroma:1.5.9
    volumes:
      - chromadata:/data
    ports:
      - "8001:8000"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/api/v1/heartbeat"]
      interval: 5s
      timeout: 5s
      retries: 10
      start_period: 10s

  backend:
    build: ./backend
    ports:
      - "8000:8000"
    env_file: .env
    depends_on:
      postgres:
        condition: service_healthy
      chromadb:
        condition: service_healthy
    command: >
      sh -c "alembic upgrade head && uvicorn app.main:app --host 0.0.0.0 --port 8000"

volumes:
  pgdata:
  chromadata:
```

**ChromaDB healthcheck endpoint:** `/api/v1/heartbeat` returns 200 when the ChromaDB server is ready. [VERIFIED: confirmed via chromadb.dev docs + search results]

### Pattern 10: Admin bootstrap in lifespan

```python
# app/services/user_service.py
async def seed_admin_if_absent() -> None:
    """Called from lifespan startup. Reads admin credentials from env vars.
    Idempotent — does nothing if an admin already exists."""
    from app.core.database import async_session_maker
    from app.core.config import settings
    from app.models.user import User
    from app.core.security import hash_password
    from sqlalchemy import select
    
    async with async_session_maker() as session:
        result = await session.execute(
            select(User).where(User.is_admin == True).limit(1)
        )
        if result.scalar_one_or_none() is not None:
            return  # admin already exists
        admin = User(
            email=settings.admin_email,
            hashed_password=hash_password(settings.admin_password),
            is_admin=True,
        )
        session.add(admin)
        await session.commit()
```

This approach is simpler to override in Docker Compose (change env vars in `.env`) than a seeded Alembic migration. [ASSUMED — pattern is sound but specific implementation is discretionary per CONTEXT.md]

### Anti-Patterns to Avoid

- **`@app.on_event("startup")`:** Deprecated in FastAPI. Use `lifespan` context manager instead.
- **`sessionmaker` (sync) instead of `async_sessionmaker`:** The sync factory does not support async usage patterns; AsyncSession must come from `async_sessionmaker`.
- **`postgresql://` DSN:** Must be `postgresql+asyncpg://` — the driver prefix is mandatory for async connections.
- **`EphemeralClient` for ChromaDB:** Data is lost on process restart. Always `PersistentClient(path=...)` in production.
- **Importing models after Base in env.py:** Alembic autogenerate silently misses tables if models are not imported before `target_metadata = Base.metadata`.
- **Running alembic upgrade head inside Docker without waiting for pg_isready:** Use `depends_on: condition: service_healthy` — without it, the migration command races PostgreSQL startup and fails intermittently.
- **Storing hashed password as bytes in DB:** bcrypt returns bytes; decode to str before storing to avoid column type mismatch with SQLAlchemy String column.

---

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| JWT encode/decode | Custom base64+HMAC | PyJWT 2.13.0 | Algorithm confusion attacks, expiry validation, claim structure are all handled |
| Password hashing | SHA-256 or MD5 | bcrypt 5.0.0 | bcrypt is deliberately slow (work factor); fast hashes are crackable with GPU |
| Async DB session scoping | Manual connection pool | `async_sessionmaker` + `get_db` Depends | Session cleanup on exception is handled; avoids connection leak |
| DB schema versioning | Manual ALTER TABLE scripts | Alembic | Tracks migration history, supports rollback, handles multi-developer conflicts |
| Startup ordering | `sleep 5` in entrypoint | Docker Compose `depends_on: condition: service_healthy` + `pg_isready` | Sleep is unreliable; health conditions are deterministic |
| Env var validation | `os.getenv()` with manual type coercion | Pydantic Settings BaseSettings | Type validation, default values, clear error messages, .env loading |

**Key insight:** The SQLAlchemy async + Alembic async wiring has specific boilerplate that diverges from the synchronous versions in non-obvious ways. Using `alembic init -t async` generates the correct template; writing env.py from scratch against sync examples will produce subtly broken migrations.

---

## Common Pitfalls

### Pitfall 1: asyncpg DSN format

**What goes wrong:** `create_async_engine("postgresql://user:pass@host/db")` raises `MissingGreenlet` or driver error at runtime, or silently falls back to synchronous mode.
**Why it happens:** SQLAlchemy uses the URL scheme prefix to select the DBAPI driver. `postgresql://` selects psycopg2 (synchronous); `postgresql+asyncpg://` selects asyncpg.
**How to avoid:** Always use `postgresql+asyncpg://` in `database_url`. Make this the literal value stored in `.env`.
**Warning signs:** `greenlet_spawn has not been called` exception; `sqlalchemy.exc.InterfaceError` mentioning psycopg2 when asyncpg is installed.

### Pitfall 2: Alembic env.py missing model imports

**What goes wrong:** `alembic revision --autogenerate` produces an empty migration file, or only partially captures the schema.
**Why it happens:** Alembic reads `Base.metadata` at migration time. If models are not imported before `target_metadata = Base.metadata`, SQLAlchemy's metadata object has no table definitions.
**How to avoid:** In `alembic/env.py`, explicitly import all model modules before assigning `target_metadata`:
```python
from app.models import user, document, request, evaluation  # noqa: F401
from app.core.database import Base
target_metadata = Base.metadata
```
**Warning signs:** Empty `upgrade()` / `downgrade()` functions in generated migration.

### Pitfall 3: Docker Compose race condition without health checks

**What goes wrong:** FastAPI starts before PostgreSQL is ready; `alembic upgrade head` fails with `connection refused`; container crashes and restarts in a loop.
**Why it happens:** `depends_on:` without `condition: service_healthy` only waits for the container to start, not for the service inside it to be ready.
**How to avoid:** Define `healthcheck:` blocks for postgres (using `pg_isready`) and chromadb (using `curl /api/v1/heartbeat`), and use `condition: service_healthy` in `depends_on`.
**Warning signs:** Intermittent startup failures that succeed on second `docker compose up`; log messages showing connection refused during migration.

### Pitfall 4: bcrypt 5.x ValueError on long passwords

**What goes wrong:** `bcrypt.hashpw()` raises `ValueError: password cannot be longer than 72 bytes` when a user registers with a long password.
**Why it happens:** bcrypt 5.0.0 changed from silently truncating >72 byte passwords to raising an error.
**How to avoid:** Validate password length in the registration endpoint before calling `hash_password()`. Add a `max_length=72` constraint to the `UserCreate` Pydantic schema (or enforce the 72-byte limit explicitly as bytes, since non-ASCII chars can be >1 byte per char).
**Warning signs:** `ValueError` traceback from bcrypt during user registration with non-ASCII or long passwords.

### Pitfall 5: ChromaDB PersistentClient path not writable in Docker

**What goes wrong:** `chromadb.PersistentClient(path="/chroma/data")` silently creates an empty store in-memory, or raises a permission error, because the path is not mounted.
**Why it happens:** If the Docker volume is not mounted at the correct path, ChromaDB either falls back to a temp directory or fails.
**How to avoid:** Mount a named Docker volume to `/chroma/data` inside the FastAPI container (or use the separate ChromaDB server container). Call `client.heartbeat()` in the lifespan startup to validate the client is working.
**Warning signs:** ChromaDB data disappears after container restart; `OSError: [Errno 13] Permission denied` in logs.

### Pitfall 6: Alembic env.py asyncio.run() conflicts

**What goes wrong:** If Alembic is ever called from within a running event loop (e.g., from a FastAPI endpoint at startup), `asyncio.run()` raises `RuntimeError: This event loop is already running`.
**Why it happens:** `asyncio.run()` creates a new event loop, which conflicts with an existing loop.
**How to avoid:** Run Alembic migrations from the Docker entrypoint shell command BEFORE starting Uvicorn (`alembic upgrade head && uvicorn ...`), not from within FastAPI's lifespan. This is the correct pattern. Never call `alembic upgrade head` programmatically from inside FastAPI.
**Warning signs:** `RuntimeError: This event loop is already running` in startup logs.

### Pitfall 7: pydantic-settings not installed separately

**What goes wrong:** `from pydantic_settings import BaseSettings` raises `ModuleNotFoundError: No module named 'pydantic_settings'`.
**Why it happens:** In Pydantic v2, `BaseSettings` was moved to a separate `pydantic-settings` package not included in `pydantic` core.
**How to avoid:** Add `pydantic-settings` explicitly to `requirements.txt`.
**Warning signs:** `ModuleNotFoundError` at application startup.

---

## Code Examples

### Complete users table ORM model (app/models/user.py)

```python
# Source: SQLAlchemy 2.0 docs + project conventions
from datetime import datetime, timezone
from sqlalchemy import Boolean, DateTime, Integer, String
from sqlalchemy.orm import Mapped, mapped_column
from pydantic import BaseModel, EmailStr
from app.core.database import Base

class User(Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(Integer, primary_key=True, index=True)
    email: Mapped[str] = mapped_column(String, unique=True, index=True, nullable=False)
    hashed_password: Mapped[str] = mapped_column(String, nullable=False)
    is_admin: Mapped[bool] = mapped_column(Boolean, default=False, nullable=False)
    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True),
        default=lambda: datetime.now(timezone.utc),
        nullable=False,
    )

# Collocated Pydantic schemas (D-04)
class UserCreate(BaseModel):
    email: EmailStr
    password: str  # min 8, max 72 bytes enforced in validator

class UserResponse(BaseModel):
    id: int
    email: str
    is_admin: bool
    created_at: datetime

    model_config = {"from_attributes": True}  # replaces orm_mode in Pydantic v2
```

### FastAPI dependency injection for DB session

```python
# Source: https://docs.sqlalchemy.org/en/20/orm/extensions/asyncio.html
from typing import Annotated
from fastapi import Depends
from sqlalchemy.ext.asyncio import AsyncSession
from app.core.database import get_db

DbSession = Annotated[AsyncSession, Depends(get_db)]

# In route:
@router.get("/users/me")
async def read_me(
    current_user: Annotated[User, Depends(get_current_user)],
    db: DbSession,
) -> UserResponse:
    return UserResponse.model_validate(current_user)
```

### Alembic initial migration — all five tables

The initial migration must create: `users`, `documents`, `document_chunks`, `ai_requests`, `evaluations`. Generate with `alembic revision --autogenerate -m "initial_schema"` after all five model files exist and are imported in env.py.

---

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| `@app.on_event("startup")` | `@asynccontextmanager lifespan` | FastAPI 0.93 | Deprecated pattern — do not use |
| `sessionmaker` (sync) | `async_sessionmaker` | SQLAlchemy 2.0 | Required for async sessions |
| `orm_mode = True` (Pydantic v1) | `model_config = {"from_attributes": True}` | Pydantic v2 | Different config syntax for ORM serialization |
| `BaseSettings` in pydantic | `from pydantic_settings import BaseSettings` | Pydantic v2 | Separate install required |
| `python-jose` for JWT | `PyJWT` | 2022-2023 | python-jose CVE history; PyJWT is the maintained alternative |
| `passlib` for bcrypt | `bcrypt` direct | 2022+ | passlib maintenance slowed; direct bcrypt is cleaner |
| `alembic init` (sync) | `alembic init -t async` | Alembic 1.10+ | async template handles asyncio.run() boilerplate |
| ChromaDB `Client()` / `EphemeralClient()` | `PersistentClient(path=...)` | ChromaDB 0.4+ | Old Client() is deprecated; PersistentClient is the API for local persistence |

**Deprecated/outdated:**
- `alembic init` (without `-t async`): generates sync env.py that requires manual rewriting for asyncpg
- `@app.on_event("startup")`: deprecated since FastAPI 0.93; removed in future versions
- `orm_mode = True` in Pydantic model Config class: use `model_config = {"from_attributes": True}` instead

---

## Assumptions Log

| # | Claim | Section | Risk if Wrong |
|---|-------|---------|---------------|
| A1 | `pydantic-settings` current version is compatible with pydantic 2.13.4 | Standard Stack | Import failure at startup; fix by checking PyPI for correct version |
| A2 | Admin bootstrap via startup lifespan + env vars is simplest Docker Compose override | Pattern 10 | Alternative (Alembic seed migration) is equally valid; low risk, just implementation preference |
| A3 | `python-dotenv` 1.2.2 is latest compatible version | Standard Stack | Minor version mismatch; PyPI version check recommended before pinning |
| A4 | ChromaDB `/api/v1/heartbeat` is the correct health endpoint for the chromadb/chroma Docker image 1.5.9 | Pattern 9 | If endpoint path changed, healthcheck fails and backend never starts; verify against chromadb/chroma:1.5.9 image |
| A5 | `structlog` 26.1.0 is the current latest version | Standard Stack | Minor; structlog API is stable, version mismatch is low risk |
| A6 | FastAPI 0.136.3 requires `pydantic >= 2.x` | Standard Stack | Already confirmed in CLAUDE.md research; low risk |

**If this table is empty:** Not empty — see above.

---

## Open Questions

1. **FastAPI version discrepancy**
   - What we know: CLAUDE.md pins FastAPI 0.136.3; PyPI latest is 0.137.1 as of 2026-06-15
   - What's unclear: Whether to use exactly 0.136.3 or the latest — CLAUDE.md says 0.136.3
   - Recommendation: Use 0.136.3 as pinned in CLAUDE.md; the planner should note the version in requirements.txt exactly

2. **Alembic version discrepancy**
   - What we know: CLAUDE.md pins Alembic 1.18.4; PyPI latest is 1.18.4 (confirmed match)
   - What's unclear: Nothing — versions match
   - Recommendation: Use 1.18.4 as pinned

3. **pydantic-settings version pin**
   - What we know: CLAUDE.md does not pin pydantic-settings explicitly
   - What's unclear: Exact version to pin
   - Recommendation: Use latest compatible with pydantic 2.13.4; verify on PyPI before pinning in requirements.txt

4. **ChromaDB deployment mode**
   - What we know: CLAUDE.md says use ChromaDB 1.5.9 with PersistentClient; `chromadb/chroma` is the Docker image
   - What's unclear: Whether to run ChromaDB as a separate service (HTTP server) or embed PersistentClient inside FastAPI container
   - Recommendation: For simplest Docker Compose setup, run ChromaDB as a **separate service** using the official image, and use `chromadb.HttpClient` in FastAPI to connect to it. Alternatively, use `PersistentClient` embedded in FastAPI with a shared volume. The separate service approach is cleaner for Phase 2+ where embeddings are computed in FastAPI and stored in Chroma. Planner should make this architectural call.

---

## Environment Availability

| Dependency | Required By | Available | Version | Fallback |
|------------|------------|-----------|---------|----------|
| Docker | Container orchestration | ✓ | 29.4.2 | — |
| Docker Compose v2 | Single `docker compose up` target | ✓ | 5.1.3 | — |
| curl | ChromaDB health check in Docker | ✓ (on host) | 8.7.1 | wget in Docker image |
| Python 3.12 | Backend runtime | ✗ (host: 3.9.6) | 3.9.6 | Resolved via Dockerfile FROM python:3.12-slim |
| PostgreSQL | Database | ✗ (not local) | — | Provided via Docker Compose postgres:16-alpine |
| pytest | Testing | ✓ (host) | 8.3.5 | — |
| pytest-asyncio | Async test support | ✗ (not installed) | — | Install in test requirements |

**Missing dependencies with no fallback:**
- None that block development — all runtime dependencies are provided via Docker Compose

**Missing dependencies with fallback:**
- Python 3.12: host is 3.9.6, but Dockerfile will specify `FROM python:3.12-slim` — no action needed
- PostgreSQL locally: not needed; runs in Docker
- pytest-asyncio: must be added to test requirements before running async tests

---

## Validation Architecture

### Test Framework

| Property | Value |
|----------|-------|
| Framework | pytest 9.1.0 + pytest-asyncio 1.4.0 |
| Config file | `pyproject.toml` or `pytest.ini` — Wave 0 gap |
| Quick run command | `pytest tests/test_auth.py -x -q` |
| Full suite command | `pytest tests/ -v` |

### Phase Requirements to Test Map

| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| FOUND-01 | All 5 tables created in PostgreSQL by Alembic migration | integration | `pytest tests/test_db.py::test_tables_exist -x` | ❌ Wave 0 |
| FOUND-02 | `docker compose up` brings all services up with health checks | smoke | `docker compose up -d && docker compose ps` (manual verification) | ❌ Wave 0 |
| FOUND-03 | POST /register creates user; POST /login returns JWT; GET /users/me returns 200 with JWT, 401 without | integration | `pytest tests/test_auth.py -x` | ❌ Wave 0 |
| FOUND-04 | GET /health returns 200 on each service | integration | `pytest tests/test_health.py::test_health_endpoint -x` | ❌ Wave 0 |

### Sampling Rate

- **Per task commit:** `pytest tests/ -x -q`
- **Per wave merge:** `pytest tests/ -v`
- **Phase gate:** Full suite green before `/gsd:verify-work`

### Wave 0 Gaps

- [ ] `tests/conftest.py` — async test client fixture, test database setup
- [ ] `tests/test_auth.py` — covers FOUND-03 (register, login, protected route)
- [ ] `tests/test_health.py` — covers FOUND-04 (health endpoint)
- [ ] `tests/test_db.py` — covers FOUND-01 (table existence after migration)
- [ ] `pytest.ini` or `pyproject.toml [tool.pytest.ini_options]` — asyncio_mode = "auto"
- [ ] Framework install: `pip install pytest==9.1.0 pytest-asyncio==1.4.0 httpx==0.28.1`

---

## Security Domain

### Applicable ASVS Categories

| ASVS Category | Applies | Standard Control |
|---------------|---------|-----------------|
| V2 Authentication | yes | bcrypt 5.0.0 (direct); PyJWT 2.13.0 HS256 |
| V3 Session Management | yes | Access token only; expiry via `exp` claim; no refresh tokens in v1 |
| V4 Access Control | yes | `get_current_user` FastAPI dependency on all protected routes; admin flag on User model |
| V5 Input Validation | yes | Pydantic v2 schema validation on all request bodies; `EmailStr` for email fields |
| V6 Cryptography | yes | HS256 with secret key from env var (never hardcoded); bcrypt for password storage |

### Known Threat Patterns for FastAPI + JWT + PostgreSQL

| Pattern | STRIDE | Standard Mitigation |
|---------|--------|---------------------|
| JWT algorithm confusion (alg:none) | Tampering | `algorithms=["HS256"]` list in `jwt.decode()` — never accept alg from token header |
| Hardcoded JWT secret | Information Disclosure | `jwt_secret_key` from env var via Pydantic Settings; never in source |
| SQL injection via raw queries | Tampering | SQLAlchemy ORM parameterizes all queries; never use `text()` with f-strings |
| Timing attack on password compare | Information Disclosure | `bcrypt.checkpw()` is constant-time by design |
| Brute force on login endpoint | Elevation of Privilege | Not in scope for v1; note for v2 (rate limiting) |
| Bcrypt >72 byte truncation | Elevation of Privilege | bcrypt 5.x raises ValueError; validate max length in UserCreate schema |
| JWT token in query params | Information Disclosure | Use `Authorization: Bearer` header only (OAuth2PasswordBearer enforces this) |

---

## Sources

### Primary (HIGH confidence)

- [FastAPI Events docs](https://fastapi.tiangolo.com/advanced/events/) — lifespan pattern, `@asynccontextmanager`, deprecation of `@app.on_event`
- [SQLAlchemy asyncio docs](https://docs.sqlalchemy.org/en/20/orm/extensions/asyncio.html) — async engine creation, async_sessionmaker, FastAPI dependency pattern
- [Alembic async template](https://github.com/sqlalchemy/alembic/blob/main/alembic/templates/async/env.py) — full env.py for async migrations
- [PyJWT usage docs](https://pyjwt.readthedocs.io/en/stable/usage.html) — encode/decode patterns, error types
- [bcrypt PyPI](https://pypi.org/project/bcrypt/) — 5.x API and breaking change documentation
- [FastAPI JWT tutorial](https://fastapi.tiangolo.com/tutorial/security/oauth2-jwt/) — get_current_user pattern, 401 response
- [Pydantic Settings docs](https://docs.pydantic.dev/latest/concepts/pydantic_settings/) — BaseSettings, model_config, env_file
- [Docker startup ordering docs](https://docs.docker.com/compose/how-tos/startup-order/) — depends_on condition: service_healthy, pg_isready healthcheck
- [ChromaDB Docker docs](https://docs.trychroma.com/guides/deploy/docker) — named volume mount at /data
- [PyPI version verification](https://pypi.org/pypi/{package}/json) — all versions confirmed 2026-06-15

### Secondary (MEDIUM confidence)

- [Alembic cookbook](https://alembic.sqlalchemy.org/en/latest/cookbook.html) — async migration pattern cross-reference
- [FastAPI + SQLAlchemy 2.0 setup guide](https://berkkaraal.com/blog/2024/09/19/setup-fastapi-project-with-async-sqlalchemy-2-alembic-postgresql-and-docker/) — complete database.py and Docker Compose pattern

### Tertiary (LOW confidence — needs validation)

- ChromaDB heartbeat endpoint path `/api/v1/heartbeat` — confirmed via web search but not official docs; [ASSUMED]

---

## Metadata

**Confidence breakdown:**

- Standard stack: HIGH — all versions confirmed on PyPI 2026-06-15; exact matches to CLAUDE.md pins
- Architecture patterns: HIGH — all patterns from official docs (FastAPI, SQLAlchemy, PyJWT, bcrypt, Alembic, Docker)
- Pitfalls: HIGH — drawn from official changelog (bcrypt 5.x), official docs (FastAPI deprecation, asyncpg DSN format), and confirmed GitHub issues
- ChromaDB heartbeat path: MEDIUM — confirmed via search, not official docs page

**Research date:** 2026-06-15
**Valid until:** 2026-07-15 (stable libraries; 30-day window)
