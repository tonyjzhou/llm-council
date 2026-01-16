# LLM Council Security & Architecture Hardening Plan

## Overview

Transform LLM Council from a "vibe-coded prototype" (Security Grade F) into a production-ready system (Grade A-) by implementing 30 improvements across 6 phases: critical security fixes, resilience patterns, SQLite migration, security hardening, code quality, and future enhancements.

**Complexity:** Complex (30 items, ~18 days, 15+ files)
**Confidence:** High (all vulnerabilities verified, patterns proven)

---

## Current State

**Security Vulnerabilities (17 total):**
- `storage.py:16-18` - Path traversal: conversation_id directly in file path
- `config.py:9` - No API key validation, sends `Bearer None` if missing
- `main.py:25-31` - CORS allows `*` methods/headers with credentials
- `main.py:70-207` - All endpoints unprotected (no auth)
- `main.py:134-207` - No rate limiting on message endpoints
- `main.py:40-43` - No input size limits
- `main.py:196-198` - Raw exceptions sent to client
- `openrouter.py:47-49` - Error details logged to console

**Architectural Issues (5 total):**
- `storage.py` - JSON files, no ACID, no concurrent write safety
- `council.py:176-209` - Brittle regex parsing for rankings
- `openrouter.py` - No retry logic, no circuit breakers
- `config.py` - Hardcoded models, requires restart to change
- No tests, incomplete type hints

---

## Desired End State

| Category | Current | Target |
|----------|---------|--------|
| Security | F | A- |
| Reliability | D | A |
| Data Integrity | D | A |
| Maintainability | B | A |
| Observability | F | B+ |

**Key Outcomes:**
- Cannot exploit path traversal or access unauthorized data
- System self-heals from model failures (circuit breakers)
- All data in SQLite with ACID transactions
- 80%+ test coverage on critical paths
- Structured logging with request tracing

---

## NOT Doing

- OAuth/SSO (basic JWT sufficient for now)
- PostgreSQL (SQLite adequate for single-instance)
- Redis caching (defer to Phase 6)
- Kubernetes deployment configs
- Frontend authentication UI (backend-only in this plan)
- Dynamic model configuration UI

---

## Dependencies

**New Python Packages:**
- slowapi - Rate limiting
- PyJWT - JWT authentication
- tenacity - Retry logic with backoff
- instructor - Structured LLM output
- sqlmodel - SQLAlchemy + Pydantic
- aiosqlite - Async SQLite driver
- python-json-logger - Structured logging
- pytest - Testing
- pytest-asyncio - Async test support
- pytest-httpx - HTTP mocking

**Phase Dependencies:**
Phase 1 (Security) -> Phase 2 (Resilience) -> Phase 3 (SQLite) -> Phase 4 (Hardening) -> Phase 5 (Quality) -> Phase 6 (Future)

---

## Phase 1: Critical Security Fixes

**Goal:** Block vulnerabilities exploitable TODAY
**Effort:** ~2 days
**Security Grade:** F -> C

### 1.1 Fix Path Traversal Vulnerability

**File:** `backend/storage.py:16-18`

**Current:**
```python
def get_conversation_path(conversation_id: str) -> str:
    return os.path.join(DATA_DIR, f"{conversation_id}.json")
```

**Change to:**
```python
import uuid
from pathlib import Path

def validate_conversation_id(conversation_id: str) -> str:
    """Validate conversation_id is a valid UUID to prevent path traversal."""
    try:
        uuid.UUID(conversation_id)
    except ValueError:
        raise ValueError(f"Invalid conversation_id format: {conversation_id}")
    return conversation_id

def get_conversation_path(conversation_id: str) -> str:
    """Get the file path for a conversation with path traversal protection."""
    validated_id = validate_conversation_id(conversation_id)
    safe_path = Path(DATA_DIR) / f"{validated_id}.json"

    # Verify path is within DATA_DIR (defense in depth)
    try:
        safe_path.resolve().relative_to(Path(DATA_DIR).resolve())
    except ValueError:
        raise ValueError(f"Path traversal attempt detected: {conversation_id}")

    return str(safe_path)
```

**Success (Automated):**
- Request with `..%2F..%2Fetc%2Fpasswd` returns 422, NOT file contents

---

### 1.2 Add API Key Validation

**File:** `backend/config.py:9`

**Current:**
```python
OPENROUTER_API_KEY = os.getenv("OPENROUTER_API_KEY")
```

**Change to:**
```python
import sys

OPENROUTER_API_KEY = os.getenv("OPENROUTER_API_KEY")

# Fail fast if API key is missing
if not OPENROUTER_API_KEY:
    print("FATAL: OPENROUTER_API_KEY environment variable is required", file=sys.stderr)
    print("Please set it in your .env file:", file=sys.stderr)
    print("  OPENROUTER_API_KEY=sk-or-v1-your-key-here", file=sys.stderr)
    sys.exit(1)

# Warn if format looks wrong
if not OPENROUTER_API_KEY.startswith("sk-or-"):
    print("WARNING: OPENROUTER_API_KEY may be invalid (expected sk-or-* prefix)", file=sys.stderr)
```

**Success (Automated):**
- Unset API key causes exit code 1 with error message

---

### 1.3 Fix CORS Configuration

**File:** `backend/main.py:25-31`

**Current:**
```python
app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:5173", "http://localhost:3000"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

**Change to:**
```python
app.add_middleware(
    CORSMiddleware,
    allow_origins=[
        "http://localhost:5173",
        "http://localhost:3000",
    ],
    allow_credentials=True,
    allow_methods=["GET", "POST", "OPTIONS"],
    allow_headers=["Content-Type", "Authorization"],
    max_age=3600,
)
```

**Success (Manual):** OPTIONS preflight returns explicit allowed methods

---

### 1.4 Add JWT Authentication

**New File:** `backend/auth.py`

```python
"""JWT authentication for LLM Council API."""

import os
from datetime import datetime, timedelta
from typing import Optional

import jwt
from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials

SECRET_KEY = os.getenv("JWT_SECRET_KEY", "CHANGE-ME-IN-PRODUCTION")
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_HOURS = 24

security = HTTPBearer(auto_error=False)


def create_access_token(user_id: str, expires_delta: Optional[timedelta] = None) -> str:
    """Create a new JWT access token."""
    expire = datetime.utcnow() + (expires_delta or timedelta(hours=ACCESS_TOKEN_EXPIRE_HOURS))
    to_encode = {"sub": user_id, "exp": expire, "iat": datetime.utcnow()}
    return jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)


async def get_current_user(
    credentials: Optional[HTTPAuthorizationCredentials] = Depends(security),
) -> str:
    """Extract and validate user from JWT token."""
    if credentials is None:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Authentication required",
            headers={"WWW-Authenticate": "Bearer"},
        )

    try:
        payload = jwt.decode(credentials.credentials, SECRET_KEY, algorithms=[ALGORITHM])
        user_id: Optional[str] = payload.get("sub")
        if user_id is None:
            raise HTTPException(status_code=401, detail="Invalid token: missing user identifier")
        return user_id
    except jwt.ExpiredSignatureError:
        raise HTTPException(status_code=401, detail="Token has expired")
    except jwt.JWTError as e:
        raise HTTPException(status_code=401, detail=f"Invalid token: {str(e)}")
```

**Update main.py:** Add `user_id: str = Depends(get_current_user)` to protected endpoints

**Success (Automated):**
- Without token: 401 Unauthorized
- With valid token: 200 OK

---

### 1.5 Add Rate Limiting

**File:** `backend/main.py`

**Add:**
```python
from slowapi import Limiter, _rate_limit_exceeded_handler
from slowapi.util import get_remote_address
from slowapi.errors import RateLimitExceeded

limiter = Limiter(key_func=get_remote_address)
app.state.limiter = limiter
app.add_exception_handler(RateLimitExceeded, _rate_limit_exceeded_handler)

@app.post("/api/conversations/{conversation_id}/message/stream")
@limiter.limit("10/minute")
async def send_message_stream(request: Request, ...):
    ...
```

**Success (Automated):**
- 11th request in 1 minute returns 429 Too Many Requests

---

### Phase 1 Success Criteria

**Automated:**
- test_path_traversal_blocked PASS
- test_missing_api_key_fails_startup PASS
- test_auth_required PASS
- test_rate_limiting_enforced PASS

**Manual:**
- [ ] CORS preflight returns explicit allowed methods
- [ ] Invalid JWT returns 401 with clear message

---

## Phase 2: Resilience & Reliability

**Goal:** Prevent cascading failures from model issues
**Effort:** ~3 days
**Reliability Grade:** D -> B

### 2.1 Add Circuit Breaker Pattern

**New File:** `backend/circuit_breaker.py`

```python
"""Circuit breaker pattern for model health tracking."""

import time
from collections import defaultdict
from dataclasses import dataclass
from typing import Dict, Optional
import logging

logger = logging.getLogger(__name__)


@dataclass
class ModelHealth:
    failure_count: int = 0
    last_failure: Optional[float] = None
    disabled_until: Optional[float] = None
    total_requests: int = 0
    total_failures: int = 0


class CircuitBreaker:
    def __init__(self, failure_threshold: int = 3, cooldown_seconds: int = 300):
        self.failure_threshold = failure_threshold
        self.cooldown_seconds = cooldown_seconds
        self._health: Dict[str, ModelHealth] = defaultdict(ModelHealth)

    def is_available(self, model: str) -> bool:
        health = self._health[model]
        if health.disabled_until is None:
            return True
        if time.time() >= health.disabled_until:
            logger.info(f"Circuit breaker: Re-enabling {model} after cooldown")
            health.disabled_until = None
            health.failure_count = 0
            return True
        return False

    def record_success(self, model: str) -> None:
        health = self._health[model]
        health.failure_count = 0
        health.total_requests += 1

    def record_failure(self, model: str) -> None:
        health = self._health[model]
        health.failure_count += 1
        health.total_failures += 1
        health.total_requests += 1
        health.last_failure = time.time()
        if health.failure_count >= self.failure_threshold:
            health.disabled_until = time.time() + self.cooldown_seconds
            logger.warning(f"Circuit breaker: Disabling {model} for {self.cooldown_seconds}s")

    def get_status(self) -> Dict[str, dict]:
        return {
            model: {
                "available": self.is_available(model),
                "failure_count": health.failure_count,
                "total_requests": health.total_requests,
                "total_failures": health.total_failures,
            }
            for model, health in self._health.items()
        }


circuit_breaker = CircuitBreaker()
```

---

### 2.2 Add Retry Logic with Tenacity

**File:** `backend/openrouter.py`

**Update:**
```python
from tenacity import retry, stop_after_attempt, wait_exponential, retry_if_exception_type

RETRY_EXCEPTIONS = (httpx.TimeoutException, httpx.NetworkError, httpx.HTTPStatusError)

@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=1, max=10),
    retry=retry_if_exception_type(RETRY_EXCEPTIONS),
    reraise=True,
)
async def _query_model_with_retry(client, model, messages):
    headers = {"Authorization": f"Bearer {OPENROUTER_API_KEY}", "Content-Type": "application/json"}
    payload = {"model": model, "messages": messages}
    response = await client.post(OPENROUTER_API_URL, headers=headers, json=payload)
    response.raise_for_status()
    return response.json()
```

---

### 2.3 Add Structured Output with Instructor

**File:** `backend/council.py`

**Add Pydantic model:**
```python
from pydantic import BaseModel, Field
from typing import List, Optional
import instructor

class RankingEvaluation(BaseModel):
    evaluations: List[str] = Field(description="Individual evaluation of each response")
    final_ranking: List[str] = Field(description="Ordered list from best to worst")
    reasoning: Optional[str] = Field(default=None, description="Brief explanation")


async def get_structured_ranking(model: str, ranking_prompt: str, timeout: float = 120.0):
    """Query a model for structured ranking output using instructor."""
    try:
        async with httpx.AsyncClient(timeout=timeout) as client:
            instructor_client = instructor.from_openai(client, mode=instructor.Mode.JSON)
            response = await instructor_client.chat.completions.create(
                model=model,
                response_model=RankingEvaluation,
                messages=[{"role": "user", "content": ranking_prompt}],
                max_retries=2,
            )
            return response
    except Exception as e:
        logger.error(f"Structured ranking failed for {model}: {e}")
        return None
```

**Update stage2_collect_rankings:** Use `get_structured_ranking()` with fallback to regex parsing

---

### 2.4 Add Input Size Limits

**File:** `backend/main.py:40-43`

```python
from pydantic import Field

class SendMessageRequest(BaseModel):
    content: str = Field(min_length=1, max_length=10000)
```

---

### 2.5 Add Response Size Limits

**File:** `backend/openrouter.py`

```python
MAX_RESPONSE_LENGTH = 50000

# In query_model(), after getting response:
if len(content) > MAX_RESPONSE_LENGTH:
    logger.warning(f"Truncating {model} response from {len(content)} chars")
    content = content[:MAX_RESPONSE_LENGTH] + "\n\n[Response truncated]"
```

---

### 2.6 Add Error Message Sanitization

**File:** `backend/main.py:196-198`

```python
except Exception as e:
    logger.exception("Error in message stream")
    yield f"data: {json.dumps({'type': 'error', 'message': 'An error occurred. Please try again.', 'code': 'INTERNAL_ERROR'})}\n\n"
```

---

### Phase 2 Success Criteria

**Automated:**
- test_circuit_breaker_opens_after_failures PASS
- test_retry_on_transient_errors PASS
- test_structured_ranking_output PASS
- test_input_size_limits PASS

**Manual:**
- [ ] Disable one model -> System continues with other models
- [ ] Trigger exception -> Client sees generic error, not stack trace

---

## Phase 3: Data Integrity (SQLite Migration)

**Goal:** Replace JSON files with ACID-compliant SQLite
**Effort:** ~4 days
**Data Safety Grade:** D -> A

### 3.1 Define Database Schema

**New File:** `backend/models.py`

```python
from datetime import datetime
from typing import List, Optional
from sqlmodel import SQLModel, Field, Relationship
import uuid


class Conversation(SQLModel, table=True):
    __tablename__ = "conversations"
    id: str = Field(default_factory=lambda: str(uuid.uuid4()), primary_key=True)
    user_id: str = Field(index=True)
    title: str = Field(default="New Conversation")
    created_at: datetime = Field(default_factory=datetime.utcnow)
    updated_at: datetime = Field(default_factory=datetime.utcnow)
    schema_version: str = Field(default="1.0")
    messages: List["Message"] = Relationship(back_populates="conversation")


class Message(SQLModel, table=True):
    __tablename__ = "messages"
    id: str = Field(default_factory=lambda: str(uuid.uuid4()), primary_key=True)
    conversation_id: str = Field(foreign_key="conversations.id", index=True)
    role: str = Field(index=True)
    content: Optional[str] = None
    created_at: datetime = Field(default_factory=datetime.utcnow)
    conversation: Optional[Conversation] = Relationship(back_populates="messages")
    model_responses: List["ModelResponse"] = Relationship(back_populates="message")


class ModelResponse(SQLModel, table=True):
    __tablename__ = "model_responses"
    id: str = Field(default_factory=lambda: str(uuid.uuid4()), primary_key=True)
    message_id: str = Field(foreign_key="messages.id", index=True)
    model: str = Field(index=True)
    stage: int = Field(index=True)
    response: str
    parsed_ranking: Optional[str] = None
    created_at: datetime = Field(default_factory=datetime.utcnow)
    message: Optional[Message] = Relationship(back_populates="model_responses")
```

---

### 3.2 Create Database Module

**New File:** `backend/database.py`

```python
from contextlib import asynccontextmanager
from typing import AsyncGenerator
from sqlmodel import SQLModel
from sqlmodel.ext.asyncio.session import AsyncSession
from sqlalchemy.ext.asyncio import create_async_engine
from sqlalchemy.orm import sessionmaker
from .config import DATABASE_URL

engine = create_async_engine(DATABASE_URL, echo=False, future=True)
async_session_factory = sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)


async def init_db() -> None:
    async with engine.begin() as conn:
        await conn.run_sync(SQLModel.metadata.create_all)


@asynccontextmanager
async def get_session_context() -> AsyncGenerator[AsyncSession, None]:
    async with async_session_factory() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise
```

---

### 3.3 Update Configuration

**File:** `backend/config.py`

```python
DATABASE_URL = os.getenv("DATABASE_URL", f"sqlite+aiosqlite:///{DATA_DIR}/council.db")
```

---

### 3.4 Rewrite Storage Layer

**File:** `backend/storage.py` - Complete rewrite using SQLModel

Key functions to implement:
- `create_conversation(user_id: str)` -> Create in DB, return dict
- `get_conversation(conversation_id: str)` -> Query with joins, format as dict
- `list_conversations(user_id: Optional[str])` -> Query metadata only
- `add_user_message(conversation_id, content)` -> Insert Message row
- `add_assistant_message(conversation_id, stage1, stage2, stage3)` -> Insert Message + ModelResponse rows
- `update_conversation_title(conversation_id, title)` -> Update Conversation row

---

### 3.5 Create Migration Script

**New File:** `backend/migrate_json_to_sqlite.py`

- Read all JSON files from `data/conversations/`
- Insert into SQLite tables
- Archive JSON files to `data/conversations/archived/`
- Log migration progress

---

### Phase 3 Success Criteria

**Automated:**
- SQLite database created at `data/conversations/council.db`
- test_create_conversation PASS
- test_add_messages PASS
- test_concurrent_writes PASS

**Manual:**
- [ ] All existing conversations accessible after migration
- [ ] JSON files archived
- [ ] New conversations created in SQLite

---

## Phase 4: Security Hardening & Observability

**Goal:** Defense in depth + operational visibility
**Effort:** ~2 days
**Security Grade:** C -> B+

### 4.1 Add Security Headers Middleware

**New File:** `backend/middleware.py`

```python
from starlette.middleware.base import BaseHTTPMiddleware

class SecurityHeadersMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request, call_next):
        response = await call_next(request)
        response.headers["X-Content-Type-Options"] = "nosniff"
        response.headers["X-Frame-Options"] = "DENY"
        response.headers["X-XSS-Protection"] = "1; mode=block"
        response.headers["Referrer-Policy"] = "strict-origin-when-cross-origin"
        return response
```

---

### 4.2 Add Comprehensive Health Check

**File:** `backend/main.py`

```python
@app.get("/health")
async def health_check():
    health = {"status": "healthy", "service": "llm-council-backend", "checks": {}}

    # Check database
    try:
        async with engine.connect() as conn:
            await conn.execute("SELECT 1")
        health["checks"]["database"] = "ok"
    except Exception:
        health["checks"]["database"] = "error"
        health["status"] = "degraded"

    # Check OpenRouter
    # Check circuit breaker status
    # Return health dict
```

---

### 4.3 Add Request ID Tracing

**File:** `backend/middleware.py`

```python
class RequestIDMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request, call_next):
        request_id = str(uuid.uuid4())
        request.state.request_id = request_id
        response = await call_next(request)
        response.headers["X-Request-ID"] = request_id
        return response
```

---

### 4.4 Add Structured Logging

**New File:** `backend/logging_config.py`

```python
from pythonjsonlogger import jsonlogger

def setup_logging(level="INFO"):
    logger = logging.getLogger()
    handler = logging.StreamHandler(sys.stdout)
    formatter = jsonlogger.JsonFormatter(fmt="%(asctime)s %(levelname)s %(name)s %(message)s")
    handler.setFormatter(formatter)
    logger.addHandler(handler)
```

---

### 4.5 Add Frontend API URL Configuration

**File:** `frontend/src/api.js:5`

```javascript
const API_BASE = import.meta.env.VITE_API_BASE || 'http://localhost:8001';
```

---

### Phase 4 Success Criteria

**Automated:**
- All responses include X-Content-Type-Options, X-Frame-Options headers
- Health endpoint returns structured status

**Manual:**
- [ ] Logs are JSON formatted
- [ ] Request IDs appear in logs and response headers

---

## Phase 5: Code Quality

**Goal:** Enable confident refactoring with types and tests
**Effort:** ~3 days
**Maintainability Grade:** B -> A

### 5.1 Add Type Hints

**Add `mypy.ini`:**
```ini
[mypy]
python_version = 3.11
strict = true
```

Run `mypy backend/ --strict` and fix all issues.

---

### 5.2 Add Unit Tests

**New File:** `tests/test_council.py`

Test cases:
- `test_ranking_parser_strict_format`
- `test_ranking_parser_fallback`
- `test_aggregate_rankings_unanimous`
- `test_aggregate_rankings_split`

---

### 5.3 Add Integration Tests

**New File:** `tests/test_api.py`

Test cases:
- `test_health_check`
- `test_list_conversations_requires_auth`
- `test_create_conversation`
- `test_message_input_validation`

---

### 5.4 Add pytest Configuration

**New File:** `pytest.ini`

```ini
[pytest]
asyncio_mode = auto
testpaths = tests
addopts = -v --tb=short
```

---

### Phase 5 Success Criteria

**Automated:**
- `mypy backend/ --strict` passes
- `pytest tests/` passes
- Coverage > 80% on council.py

---

## Phase 6: Future Enhancements (Optional)

**Goal:** Operational excellence
**Effort:** ~4 days (can be deferred)

- 6.1 Dynamic Model Configuration (database-driven)
- 6.2 Settings UI (frontend model management)
- 6.3 Semantic Caching (hash or embedding-based)
- 6.4 Prometheus Metrics (/metrics endpoint)
- 6.5 Documentation (ADRs, deployment guide)

---

## Risks & Mitigations

| Risk | Mitigation | Severity |
|------|------------|----------|
| SQLite migration loses data | Backup JSON files, archive after migration | High |
| instructor library changes | Pin version, fallback to regex | Medium |
| Circuit breaker too aggressive | Configurable thresholds | Medium |
| JWT secret leaked | Use env var, rotate keys | High |

---

## Rollback Strategy

**Phase 1-2:** Revert commits, no data migration
**Phase 3:** Restore from archived JSON files
**Phase 4-5:** Middleware/tests are additive

---

## References

- Research: `thoughts/research/2026-01-11-architecture-critical-analysis.md`
- Backend: `backend/main.py`, `backend/storage.py`, `backend/council.py`
- Frontend: `frontend/src/api.js`

---

## Implementation Checklist

### Phase 1: Critical Security (~2 days)
- [ ] 1.1 Path traversal fix
- [ ] 1.2 API key validation
- [ ] 1.3 CORS configuration
- [ ] 1.4 JWT authentication
- [ ] 1.5 Rate limiting

### Phase 2: Resilience (~3 days)
- [ ] 2.1 Circuit breaker
- [ ] 2.2 Retry logic (tenacity)
- [ ] 2.3 Structured output (instructor)
- [ ] 2.4 Input size limits
- [ ] 2.5 Response size limits
- [ ] 2.6 Error sanitization

### Phase 3: SQLite Migration (~4 days)
- [ ] 3.1 Database schema
- [ ] 3.2 Database module
- [ ] 3.3 Configuration update
- [ ] 3.4 Storage rewrite
- [ ] 3.5 Migration script

### Phase 4: Hardening (~2 days)
- [ ] 4.1 Security headers
- [ ] 4.2 Health check
- [ ] 4.3 Request ID tracing
- [ ] 4.4 Structured logging
- [ ] 4.5 Frontend config

### Phase 5: Quality (~3 days)
- [ ] 5.1 Type hints
- [ ] 5.2 Unit tests
- [ ] 5.3 Integration tests
- [ ] 5.4 pytest config

### Phase 6: Future (Optional)
- [ ] 6.1 Dynamic config
- [ ] 6.2 Settings UI
- [ ] 6.3 Caching
- [ ] 6.4 Metrics
- [ ] 6.5 Documentation
