# Copy-Trading Secure Foundation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Establish the secure, tenant-isolated, auditable backend foundation required before any target can reach simulated execution.

**Architecture:** Build a Python 3.12 modular monolith with a small FastAPI composition root, explicit domain primitives, PostgreSQL-backed identity records, fail-closed tenant context, append-only audit ingestion, and schema-filtered structured logging. This milestone intentionally contains no strategy-target or broker-submission endpoint.

**Tech Stack:** Python 3.12, FastAPI with Pydantic v2, SQLAlchemy 2.0 async APIs with Psycopg 3, Alembic, PostgreSQL 18, PyJWT/cryptography, pytest, pytest-asyncio, Hypothesis, Ruff, and mypy.

**Spec:** `docs/superpowers/specs/2026-09-25-copy-trading-platform-design.md`

## Global Constraints

- PostgreSQL is authoritative; Redis must never become required for correctness.
- Money, prices, quantities, margin, and P&L must reject binary floating-point inputs.
- Missing tenant context and mismatched tenant/follower attributes fail closed before repository access.
- Application roles may insert and select audit rows but may not update or delete them.
- Raw-object logging and unrestricted exception serialization are prohibited.
- No strategy target, allocation, risk, execution, or broker endpoint is introduced in this milestone.
- All changes follow test-driven development and end in an independently reviewable commit.
- Local implementation uses the bundled Python 3.12 runtime at `C:\Users\yoges\.cache\codex-runtimes\codex-primary-runtime\dependencies\python\python.exe`; CI uses Python 3.12.
- PostgreSQL integration tests require PostgreSQL 18 through Docker or CI services. If Docker is unavailable locally, unit checks may proceed, but the milestone cannot pass its exit gate until the PostgreSQL suite passes in CI or an equivalent PostgreSQL 18 environment.

## Review Focus

- Missing, malformed, expired, wrong-issuer, or wrong-audience identity tokens must return 401 without creating application context.
- Validly signed tokens with inactive or cross-tenant membership must return 403 and expose no resource existence.
- Attempts to update or delete audit records must fail at the database boundary, including direct SQL through the runtime connection.
- Nested authorization headers, bearer tokens, database URLs, API keys, and exception messages must be redacted before log output.
- Float inputs, NaN, infinity, tenant mismatches, and duplicate membership identities must fail deterministically rather than being coerced.

---

### Task 1: Reproducible Python service skeleton

**Files:**
- Create: `.gitignore`
- Create: `.env.example`
- Create: `pyproject.toml`
- Create: `README.md`
- Create: `src/copytrader/__init__.py`
- Create: `src/copytrader/http/__init__.py`
- Create: `src/copytrader/http/app.py`
- Create: `tests/unit/http/test_app.py`

**Interfaces:**
- Produces: `copytrader.http.app.create_app() -> FastAPI`
- Produces: `GET /health/live -> {"status": "alive"}`
- Consumes: no earlier project interfaces

- [ ] **Step 1: Write the failing application-factory test**

```python
# tests/unit/http/test_app.py
from fastapi.testclient import TestClient

from copytrader.http.app import create_app


def test_liveness_is_available_without_authentication() -> None:
    response = TestClient(create_app()).get("/health/live")

    assert response.status_code == 200
    assert response.json() == {"status": "alive"}


def test_openapi_has_no_trading_routes_in_foundation() -> None:
    schema = create_app().openapi()

    assert set(schema["paths"]) == {"/health/live"}
```

- [ ] **Step 2: Add project metadata and install the development environment**

Create `pyproject.toml` with these dependency ranges and tool settings:

```toml
[build-system]
requires = ["hatchling>=1.27,<2"]
build-backend = "hatchling.build"

[project]
name = "copytrader"
version = "0.1.0"
requires-python = ">=3.12,<3.13"
dependencies = [
  "alembic>=1.16,<2",
  "cryptography>=45,<47",
  "fastapi>=0.120,<1",
  "psycopg[binary,pool]>=3.2,<4",
  "pydantic>=2.12,<3",
  "pydantic-settings>=2.11,<3",
  "pyjwt[crypto]>=2.10,<3",
  "sqlalchemy[asyncio]>=2.0.54,<2.1",
  "structlog>=25.4,<26",
  "uvicorn[standard]>=0.35,<1",
]

[project.optional-dependencies]
dev = [
  "httpx>=0.28,<1",
  "hypothesis>=6.140,<7",
  "mypy>=1.18,<2",
  "pytest>=8.4,<9",
  "pytest-asyncio>=1.2,<2",
  "ruff>=0.13,<1",
]

[tool.hatch.build.targets.wheel]
packages = ["src/copytrader"]

[tool.pytest.ini_options]
addopts = "-ra --strict-config --strict-markers"
testpaths = ["tests"]
asyncio_mode = "auto"

[tool.ruff]
target-version = "py312"
line-length = 100

[tool.ruff.lint]
select = ["E", "F", "I", "UP", "B", "ASYNC", "S"]

[tool.mypy]
python_version = "3.12"
strict = true
packages = ["copytrader"]
```

Run:

```powershell
& 'C:\Users\yoges\.cache\codex-runtimes\codex-primary-runtime\dependencies\python\python.exe' -m venv .venv
& '.\.venv\Scripts\python.exe' -m pip install --upgrade pip
& '.\.venv\Scripts\python.exe' -m pip install -e '.[dev]'
```

Expected: editable installation completes and `.venv` remains ignored by Git.

- [ ] **Step 3: Run the focused test and verify the import fails**

Run: `& '.\.venv\Scripts\python.exe' -m pytest tests/unit/http/test_app.py -q`

Expected: FAIL with `ModuleNotFoundError` for `copytrader.http.app` or `create_app`.

- [ ] **Step 4: Implement the minimal application factory**

```python
# src/copytrader/http/app.py
from fastapi import FastAPI


def create_app() -> FastAPI:
    app = FastAPI(title="Copy-Trading Platform", version="0.1.0")

    @app.get("/health/live", include_in_schema=True)
    async def liveness() -> dict[str, str]:
        return {"status": "alive"}

    return app
```

Set `.gitignore` to exclude `.venv/`, `__pycache__/`, `.pytest_cache/`, `.mypy_cache/`, `.ruff_cache/`, `.env`, coverage outputs, build outputs, and editor metadata. Set `.env.example` to contain names only: `APP_ENV`, `DATABASE_URL`, `OIDC_ISSUER`, `OIDC_AUDIENCE`, and `OIDC_PUBLIC_KEY_PATH`. Document the environment creation, test, lint, and type-check commands in `README.md`.

- [ ] **Step 5: Verify and commit the skeleton**

Run:

```powershell
& '.\.venv\Scripts\python.exe' -m pytest tests/unit/http/test_app.py -q
& '.\.venv\Scripts\python.exe' -m ruff check src tests
& '.\.venv\Scripts\python.exe' -m mypy src
```

Expected: all commands pass.

```powershell
git add .gitignore .env.example pyproject.toml README.md src/copytrader tests/unit/http/test_app.py
git commit -m "chore: scaffold secure backend service"
```

---

### Task 2: Exact domain primitives and transition guards

**Files:**
- Create: `src/copytrader/core/__init__.py`
- Create: `src/copytrader/core/ids.py`
- Create: `src/copytrader/core/numbers.py`
- Create: `src/copytrader/core/state_machine.py`
- Create: `tests/unit/core/test_ids.py`
- Create: `tests/unit/core/test_numbers.py`
- Create: `tests/unit/core/test_state_machine.py`

**Interfaces:**
- Produces: `TenantId`, `UserId`, `FollowerId`, `MembershipId` UUID value types
- Produces: `ExactDecimal.parse(value: Decimal | int | str) -> ExactDecimal`
- Produces: `TransitionTable.require(current, target) -> None`
- Consumes: Python standard-library `UUID` and `Decimal`

- [ ] **Step 1: Write failing identifier and decimal tests**

```python
# tests/unit/core/test_numbers.py
from decimal import Decimal

import pytest

from copytrader.core.numbers import ExactDecimal


@pytest.mark.parametrize("value", [float("nan"), float("inf"), 1.25])
def test_exact_decimal_rejects_binary_float(value: float) -> None:
    with pytest.raises(TypeError, match="binary floating-point"):
        ExactDecimal.parse(value)  # type: ignore[arg-type]


def test_exact_decimal_preserves_string_precision() -> None:
    value = ExactDecimal.parse("123.4500")

    assert value.value == Decimal("123.4500")
```

```python
# tests/unit/core/test_ids.py
from uuid import UUID

import pytest

from copytrader.core.ids import TenantId


def test_tenant_id_round_trips_uuid_text() -> None:
    text = "018f4e58-4fa0-7b4a-8f35-e8fbe736f8f8"

    assert str(TenantId.parse(text)) == text


def test_tenant_id_rejects_non_uuid() -> None:
    with pytest.raises(ValueError):
        TenantId.parse("tenant-one")
```

```python
# tests/unit/core/test_state_machine.py
from enum import StrEnum

import pytest

from copytrader.core.state_machine import InvalidTransition, TransitionTable


class ExampleState(StrEnum):
    CREATED = "created"
    ACTIVE = "active"
    CLOSED = "closed"


def test_transition_table_allows_only_declared_edges() -> None:
    table = TransitionTable(
        {
            ExampleState.CREATED: frozenset({ExampleState.ACTIVE}),
            ExampleState.ACTIVE: frozenset({ExampleState.CLOSED}),
            ExampleState.CLOSED: frozenset(),
        }
    )

    table.require(ExampleState.CREATED, ExampleState.ACTIVE)

    with pytest.raises(InvalidTransition, match="created -> closed"):
        table.require(ExampleState.CREATED, ExampleState.CLOSED)
```

- [ ] **Step 2: Run the tests and verify missing-module failures**

Run: `& '.\.venv\Scripts\python.exe' -m pytest tests/unit/core -q`

Expected: FAIL because the core modules do not exist.

- [ ] **Step 3: Implement immutable typed identifiers and exact decimals**

```python
# src/copytrader/core/ids.py
from dataclasses import dataclass
from typing import Self
from uuid import UUID


@dataclass(frozen=True, slots=True)
class EntityId:
    value: UUID

    @classmethod
    def parse(cls, value: UUID | str) -> Self:
        return cls(value if isinstance(value, UUID) else UUID(value))

    def __str__(self) -> str:
        return str(self.value)


class TenantId(EntityId):
    pass


class UserId(EntityId):
    pass


class FollowerId(EntityId):
    pass


class MembershipId(EntityId):
    pass
```

```python
# src/copytrader/core/numbers.py
from dataclasses import dataclass
from decimal import Decimal
from typing import Self


@dataclass(frozen=True, slots=True)
class ExactDecimal:
    value: Decimal

    @classmethod
    def parse(cls, value: Decimal | int | str) -> Self:
        if isinstance(value, float):
            raise TypeError("binary floating-point inputs are prohibited")
        parsed = value if isinstance(value, Decimal) else Decimal(value)
        if not parsed.is_finite():
            raise ValueError("decimal value must be finite")
        return cls(parsed)
```

```python
# src/copytrader/core/state_machine.py
from collections.abc import Mapping
from enum import StrEnum
from typing import Generic, TypeVar


StateT = TypeVar("StateT", bound=StrEnum)


class InvalidTransition(ValueError):
    pass


class TransitionTable(Generic[StateT]):
    def __init__(self, allowed: Mapping[StateT, frozenset[StateT]]) -> None:
        self._allowed = dict(allowed)

    def require(self, current: StateT, target: StateT) -> None:
        if target not in self._allowed.get(current, frozenset()):
            raise InvalidTransition(f"invalid transition: {current.value} -> {target.value}")
```

- [ ] **Step 4: Add property coverage for arbitrary exact decimal strings**

Use Hypothesis to generate signed integer and fractional strings, parse them through `ExactDecimal`, and assert equality with `Decimal(text)`. Include explicit `NaN`, `Infinity`, and `-Infinity` rejection tests.

- [ ] **Step 5: Verify and commit the primitives**

Run:

```powershell
& '.\.venv\Scripts\python.exe' -m pytest tests/unit/core -q
& '.\.venv\Scripts\python.exe' -m ruff check src tests
& '.\.venv\Scripts\python.exe' -m mypy src
```

Expected: all commands pass.

```powershell
git add src/copytrader/core tests/unit/core
git commit -m "feat: add exact domain primitives"
```

---

### Task 3: PostgreSQL configuration, sessions, and identity schema

**Files:**
- Create: `compose.yaml`
- Create: `alembic.ini`
- Create: `migrations/env.py`
- Create: `migrations/script.py.mako`
- Create: `migrations/versions/20260925_0001_identity_foundation.py`
- Create: `src/copytrader/bootstrap/__init__.py`
- Create: `src/copytrader/bootstrap/settings.py`
- Create: `src/copytrader/database/__init__.py`
- Create: `src/copytrader/database/base.py`
- Create: `src/copytrader/database/session.py`
- Create: `src/copytrader/identity/__init__.py`
- Create: `src/copytrader/identity/models.py`
- Create: `tests/conftest.py`
- Create: `tests/integration/database/test_identity_schema.py`

**Interfaces:**
- Produces: `Settings.database_url: str`
- Produces: `create_session_factory(settings) -> async_sessionmaker[AsyncSession]`
- Produces: SQLAlchemy models `Tenant`, `User`, `Membership`
- Consumes: typed UUID values from Task 2

- [ ] **Step 1: Write the failing PostgreSQL schema test**

```python
# tests/integration/database/test_identity_schema.py
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

from copytrader.identity.models import Membership, MembershipStatus, Tenant, User


async def test_identity_rows_are_persisted_with_tenant_boundary(
    db_session: AsyncSession,
) -> None:
    tenant = Tenant(display_name="Alpha")
    user = User(subject="oidc|alice")
    db_session.add_all([tenant, user])
    await db_session.flush()
    membership = Membership(
        tenant_id=tenant.id,
        user_id=user.id,
        status=MembershipStatus.ACTIVE,
        roles=["administrator"],
    )
    db_session.add(membership)
    await db_session.commit()

    loaded = await db_session.scalar(
        select(Membership).where(Membership.tenant_id == tenant.id)
    )

    assert loaded is not None
    assert loaded.roles == ["administrator"]
```

- [ ] **Step 2: Add PostgreSQL service and test-session fixtures**

Use this `compose.yaml` service:

```yaml
services:
  postgres:
    image: postgres:18
    environment:
      POSTGRES_DB: copytrader_test
      POSTGRES_USER: copytrader
      POSTGRES_PASSWORD: copytrader
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U copytrader -d copytrader_test"]
      interval: 2s
      timeout: 2s
      retries: 20
```

`tests/conftest.py` must read `TEST_DATABASE_URL`, create the async engine, run Alembic upgrade to `head` once per session, and wrap each test in a transaction that is rolled back. Do not silently substitute SQLite.

- [ ] **Step 3: Run the focused test and verify schema failure**

Run:

```powershell
$env:TEST_DATABASE_URL='postgresql+psycopg://copytrader:copytrader@localhost:5432/copytrader_test'
& '.\.venv\Scripts\python.exe' -m pytest tests/integration/database/test_identity_schema.py -q
```

Expected: FAIL because models and migration are absent.

- [ ] **Step 4: Implement settings, database sessions, models, and migration**

Implement settings and the async session factory as follows:

```python
# src/copytrader/bootstrap/settings.py
from functools import lru_cache

from pydantic import Field
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", extra="forbid")

    app_env: str = Field(pattern=r"^(development|test|production)$")
    database_url: str = Field(pattern=r"^postgresql\+psycopg://")
    oidc_issuer: str
    oidc_audience: str
    oidc_public_key_path: str


@lru_cache
def get_settings() -> Settings:
    return Settings()  # type: ignore[call-arg]
```

```python
# src/copytrader/database/session.py
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker, create_async_engine

from copytrader.bootstrap.settings import Settings


def create_session_factory(settings: Settings) -> async_sessionmaker[AsyncSession]:
    engine = create_async_engine(settings.database_url, pool_pre_ping=True)
    return async_sessionmaker(engine, expire_on_commit=False)
```

Model constraints must be exact:

```python
class MembershipStatus(StrEnum):
    ACTIVE = "active"
    SUSPENDED = "suspended"
    REVOKED = "revoked"


class Tenant(Base):
    __tablename__ = "tenants"
    id: Mapped[UUID] = mapped_column(Uuid, primary_key=True, default=new_uuid)
    display_name: Mapped[str] = mapped_column(String(120), nullable=False)


class User(Base):
    __tablename__ = "users"
    id: Mapped[UUID] = mapped_column(Uuid, primary_key=True, default=new_uuid)
    subject: Mapped[str] = mapped_column(String(255), nullable=False, unique=True)


class Membership(Base):
    __tablename__ = "memberships"
    __table_args__ = (
        UniqueConstraint("tenant_id", "user_id", name="uq_membership_tenant_user"),
    )
    id: Mapped[UUID] = mapped_column(Uuid, primary_key=True, default=new_uuid)
    tenant_id: Mapped[UUID] = mapped_column(ForeignKey("tenants.id"), nullable=False)
    user_id: Mapped[UUID] = mapped_column(ForeignKey("users.id"), nullable=False)
    status: Mapped[MembershipStatus] = mapped_column(Enum(MembershipStatus), nullable=False)
    roles: Mapped[list[str]] = mapped_column(ARRAY(String(64)), nullable=False)
```

Define `new_uuid()` as `return uuid4()`. Python 3.12 lacks `uuid.uuid7`, so do not mislabel UUID4 as UUID7. The migration creates all constraints and indexes on `memberships(tenant_id, status)` and `users(subject)`.

- [ ] **Step 5: Verify migration upgrade/downgrade and commit**

Run:

```powershell
& '.\.venv\Scripts\alembic.exe' upgrade head
& '.\.venv\Scripts\python.exe' -m pytest tests/integration/database/test_identity_schema.py -q
& '.\.venv\Scripts\alembic.exe' downgrade base
& '.\.venv\Scripts\alembic.exe' upgrade head
```

Expected: migration round-trip succeeds and the focused test passes.

```powershell
git add compose.yaml alembic.ini migrations src/copytrader/bootstrap src/copytrader/database src/copytrader/identity tests/conftest.py tests/integration/database
git commit -m "feat: add tenant identity persistence"
```

---

### Task 4: Fail-closed tenant context and repository boundaries

**Files:**
- Create: `src/copytrader/identity/context.py`
- Create: `src/copytrader/identity/errors.py`
- Create: `src/copytrader/identity/repository.py`
- Create: `tests/unit/identity/test_context.py`
- Create: `tests/integration/identity/test_membership_repository.py`

**Interfaces:**
- Produces: `PrincipalContext(subject, user_id, tenant_id, follower_id, roles, auth_strength)`
- Produces: `require_role(context, role) -> None`
- Produces: `MembershipRepository.get(context, membership_id) -> Membership | None`
- Consumes: identity models and session factory from Task 3

- [ ] **Step 1: Write failing cross-tenant repository tests**

```python
async def test_repository_hides_membership_from_other_tenant(
    db_session: AsyncSession,
    tenant_a_context: PrincipalContext,
    tenant_b_membership: Membership,
) -> None:
    repository = MembershipRepository(db_session)

    result = await repository.get(tenant_a_context, tenant_b_membership.id)

    assert result is None


async def test_repository_rejects_missing_context(db_session: AsyncSession) -> None:
    repository = MembershipRepository(db_session)

    with pytest.raises(MissingTenantContext):
        await repository.get(None, uuid4())
```

- [ ] **Step 2: Run tests and verify the missing-boundary failures**

Run: `& '.\.venv\Scripts\python.exe' -m pytest tests/unit/identity tests/integration/identity -q`

Expected: FAIL because context and repository interfaces are absent.

- [ ] **Step 3: Implement immutable principal context and role checks**

```python
@dataclass(frozen=True, slots=True)
class PrincipalContext:
    subject: str
    user_id: UUID
    tenant_id: UUID
    follower_id: UUID | None
    roles: frozenset[str]
    auth_strength: str


def require_role(context: PrincipalContext | None, role: str) -> None:
    if context is None:
        raise MissingTenantContext("tenant context is required")
    if role not in context.roles:
        raise PermissionDenied("required role is absent")
```

- [ ] **Step 4: Implement tenant-scoped repository queries**

Every repository public method takes a non-optional `PrincipalContext`. Keep the runtime `None` guard despite the type declaration so untyped callers fail closed. Implement the lookup at the query boundary:

```python
class MembershipRepository:
    def __init__(self, session: AsyncSession) -> None:
        self._session = session

    async def get(
        self,
        context: PrincipalContext,
        membership_id: UUID,
    ) -> Membership | None:
        if context is None:  # type: ignore[comparison-overlap]
            raise MissingTenantContext("tenant context is required")
        statement = select(Membership).where(
            Membership.id == membership_id,
            Membership.tenant_id == context.tenant_id,
        )
        return await self._session.scalar(statement)
```

Never query by resource ID alone and then compare the tenant in memory.

- [ ] **Step 5: Verify all tenant-boundary tests and commit**

Run:

```powershell
& '.\.venv\Scripts\python.exe' -m pytest tests/unit/identity tests/integration/identity -q
& '.\.venv\Scripts\python.exe' -m mypy src
```

Expected: missing context raises, cross-tenant lookup returns indistinguishable not-found, and same-tenant lookup succeeds.

```powershell
git add src/copytrader/identity tests/unit/identity tests/integration/identity
git commit -m "feat: enforce tenant context in repositories"
```

---

### Task 5: Signed identity claims and active-membership authorization

**Files:**
- Create: `src/copytrader/identity/tokens.py`
- Create: `src/copytrader/identity/authorizer.py`
- Create: `src/copytrader/http/dependencies.py`
- Modify: `src/copytrader/http/app.py`
- Create: `tests/unit/identity/test_tokens.py`
- Create: `tests/integration/http/test_context_endpoint.py`

**Interfaces:**
- Produces: `TokenVerifier.verify(token: str) -> TokenClaims`
- Produces: `MembershipAuthorizer.authorize(claims: TokenClaims) -> PrincipalContext`
- Produces: FastAPI dependency `current_principal() -> PrincipalContext`
- Produces: authenticated diagnostic endpoint `GET /v1/context`
- Consumes: membership repository from Task 4

- [ ] **Step 1: Write failing token-verification tests**

Use a test RSA key pair and issue tokens containing `sub`, `iss`, `aud`, `iat`, `exp`, `jti`, `tenant_id`, optional `follower_id`, `roles`, and `acr`. Tests must cover valid tokens, expired tokens, wrong audience, wrong issuer, unsupported algorithm, missing tenant, and malformed UUID claims.

Define `TokenFactory` in `tests/unit/identity/conftest.py`. It owns the generated private key, fixed issuer/audience, and the `TokenVerifier` built from the matching public key. Its `issue(**overrides)` method starts from a complete valid payload, applies overrides, and signs only with RS256.

```python
def test_wrong_audience_is_rejected(token_factory: TokenFactory) -> None:
    token = token_factory.issue(audience="different-service")

    with pytest.raises(InvalidIdentityToken):
        token_factory.verifier.verify(token)
```

- [ ] **Step 2: Run the token tests and verify failure**

Run: `& '.\.venv\Scripts\python.exe' -m pytest tests/unit/identity/test_tokens.py -q`

Expected: FAIL because `TokenVerifier` does not exist.

- [ ] **Step 3: Implement RS256-only verification and typed claims**

```python
from collections.abc import Mapping


@dataclass(frozen=True, slots=True)
class TokenClaims:
    subject: str
    tenant_id: UUID
    follower_id: UUID | None
    roles: frozenset[str]
    auth_strength: str
    token_id: str

    @classmethod
    def from_mapping(cls, payload: Mapping[str, object]) -> "TokenClaims":
        raw_roles = payload.get("roles", [])
        if not isinstance(raw_roles, list) or not all(isinstance(role, str) for role in raw_roles):
            raise ValueError("roles must be a string list")
        follower = payload.get("follower_id")
        return cls(
            subject=str(payload["sub"]),
            tenant_id=UUID(str(payload["tenant_id"])),
            follower_id=None if follower is None else UUID(str(follower)),
            roles=frozenset(raw_roles),
            auth_strength=str(payload.get("acr", "unknown")),
            token_id=str(payload["jti"]),
        )


class TokenVerifier:
    def __init__(self, public_key: str, issuer: str, audience: str) -> None:
        self._public_key = public_key
        self._issuer = issuer
        self._audience = audience

    def verify(self, token: str) -> TokenClaims:
        try:
            payload = jwt.decode(
                token,
                self._public_key,
                algorithms=["RS256"],
                issuer=self._issuer,
                audience=self._audience,
                options={"require": ["sub", "iss", "aud", "iat", "exp", "jti", "tenant_id"]},
            )
            return TokenClaims.from_mapping(payload)
        except (jwt.PyJWTError, KeyError, TypeError, ValueError) as exc:
            raise InvalidIdentityToken("identity token is invalid") from exc
```

Do not log the token or decoded payload on failure.

- [ ] **Step 4: Revalidate membership and expose the protected context endpoint**

`MembershipAuthorizer` resolves `User.subject`, requires an active membership for the claimed tenant, intersects token roles with stored membership roles, and rejects mismatched follower boundaries. `GET /v1/context` returns only `tenant_id`, `follower_id`, effective roles, and authentication strength. Authentication failures return 401; inactive or mismatched membership returns 403; neither response reveals membership existence.

- [ ] **Step 5: Verify identity failure modes and commit**

Run:

```powershell
& '.\.venv\Scripts\python.exe' -m pytest tests/unit/identity/test_tokens.py tests/integration/http/test_context_endpoint.py -q
& '.\.venv\Scripts\python.exe' -m ruff check src tests
& '.\.venv\Scripts\python.exe' -m mypy src
```

Expected: all token and membership cases pass and OpenAPI contains only `/health/live` plus `/v1/context`.

```powershell
git add src/copytrader/identity src/copytrader/http tests/unit/identity tests/integration/http
git commit -m "feat: authenticate tenant-bound principals"
```

---

### Task 6: Append-only audit ingestion

**Files:**
- Create: `migrations/versions/20260925_0002_append_only_audit.py`
- Create: `src/copytrader/audit/__init__.py`
- Create: `src/copytrader/audit/models.py`
- Create: `src/copytrader/audit/service.py`
- Create: `tests/integration/audit/test_append_only_audit.py`

**Interfaces:**
- Produces: `AuditService.record(session, event: NewAuditEvent) -> AuditEvent`
- Produces: globally ordered `audit_events.audit_sequence BIGINT GENERATED ALWAYS AS IDENTITY`
- Consumes: active SQLAlchemy transaction and `PrincipalContext`

- [ ] **Step 1: Write failing audit immutability tests**

```python
async def test_runtime_connection_cannot_update_or_delete_audit_event(
    db_session: AsyncSession,
    audit_event: AuditEvent,
) -> None:
    with pytest.raises(DBAPIError):
        await db_session.execute(
            text("UPDATE audit_events SET action = 'tampered' WHERE id = :id"),
            {"id": audit_event.id},
        )
        await db_session.commit()

    await db_session.rollback()

    with pytest.raises(DBAPIError):
        await db_session.execute(
            text("DELETE FROM audit_events WHERE id = :id"),
            {"id": audit_event.id},
        )
        await db_session.commit()
```

Also test that two inserted records have strictly increasing `audit_sequence`, required correlation/causation identifiers, and redacted structured details rather than secret payload copies.

- [ ] **Step 2: Run tests and verify missing audit schema**

Run: `& '.\.venv\Scripts\python.exe' -m pytest tests/integration/audit -q`

Expected: FAIL because the audit table and service do not exist.

- [ ] **Step 3: Implement the audit model and typed command**

```python
@dataclass(frozen=True, slots=True)
class NewAuditEvent:
    tenant_id: UUID
    actor_subject: str
    action: str
    resource_type: str
    resource_id: str
    correlation_id: UUID
    causation_id: UUID | None
    reason: str | None
    outcome: str
    details: dict[str, str | int | bool | None]
```

The SQLAlchemy model stores `audit_sequence`, UUID ID, tenant, actor, action, resource references, correlation and causation IDs, reason, outcome, typed JSON details, and UTC database timestamp. `AuditService.record` only adds and flushes; the caller controls the surrounding transaction so audit and business mutation can later commit atomically.

- [ ] **Step 4: Implement database immutability controls**

The migration creates a trigger function that raises SQLSTATE `55000` on `UPDATE` or `DELETE`, attaches it to `audit_events`, and indexes `(tenant_id, audit_sequence)` and `(correlation_id)`. Its downgrade removes the trigger before dropping the table and function. The application model exposes no update helper.

- [ ] **Step 5: Verify immutability and commit**

Run:

```powershell
& '.\.venv\Scripts\alembic.exe' upgrade head
& '.\.venv\Scripts\python.exe' -m pytest tests/integration/audit -q
& '.\.venv\Scripts\python.exe' -m pytest tests/integration/database -q
```

Expected: inserts succeed, sequence increases, and direct update/delete fail.

```powershell
git add migrations/versions/20260925_0002_append_only_audit.py src/copytrader/audit tests/integration/audit
git commit -m "feat: add append-only audit ingestion"
```

---

### Task 7: Schema-filtered logging and automated secret interception

**Files:**
- Create: `src/copytrader/observability/__init__.py`
- Create: `src/copytrader/observability/events.py`
- Create: `src/copytrader/observability/logging.py`
- Create: `tools/check_logging_calls.py`
- Create: `tests/unit/observability/test_logging.py`
- Create: `tests/unit/tools/test_check_logging_calls.py`

**Interfaces:**
- Produces: `SafeLogger.emit(event: LogEvent) -> None`
- Produces: approved event dataclasses `RequestCompleted`, `AuthorizationRejected`, `AuditWriteFailed`
- Produces: `redact(value: object) -> JSONValue`
- Consumes: no raw application objects

- [ ] **Step 1: Write failing recursive-redaction tests**

```python
def test_nested_secrets_and_exception_text_are_redacted(capsys: pytest.CaptureFixture[str]) -> None:
    logger = configured_test_logger()
    event = AuthorizationRejected(
        correlation_id="018f4e58-4fa0-7b4a-8f35-e8fbe736f8f8",
        reason="upstream failed: Bearer eyJ.secret.payload",
        metadata={
            "Authorization": "Bearer abc.def.ghi",
            "database_url": "postgresql://user:password@db/copytrader",
            "safe_code": "membership_inactive",
        },
    )

    logger.emit(event)
    output = capsys.readouterr().out

    assert "abc.def.ghi" not in output
    assert "eyJ.secret.payload" not in output
    assert "user:password" not in output
    assert "membership_inactive" in output
    assert "[REDACTED]" in output
```

- [ ] **Step 2: Write the failing static logging-policy test**

The AST checker must fail on `logging.info(...)`, `logger.info(...)`, `print(...)`, or `structlog.get_logger(...)` anywhere below `src/copytrader` except `observability/logging.py`. It must ignore tests and generated migrations. The test creates a temporary violating Python file and asserts the checker returns exit code 1 with its path and line.

- [ ] **Step 3: Run tests and verify logging modules are missing**

Run: `& '.\.venv\Scripts\python.exe' -m pytest tests/unit/observability tests/unit/tools -q`

Expected: FAIL because the safe logger and checker do not exist.

- [ ] **Step 4: Implement typed events, recursive redaction, and the AST gate**

`redact` must:

- Replace values for case-insensitive keys matching `authorization`, `cookie`, `token`, `secret`, `password`, `api_key`, or `connection_string`.
- Redact bearer-token patterns and credentials embedded in PostgreSQL URLs inside arbitrary strings.
- Recurse through mappings and sequences while limiting depth to six and collection length to 50.
- Return only strings, integers, booleans, null, lists, and string-keyed dictionaries.
- Convert unknown objects to their type name, never `repr(object)`.

`SafeLogger.emit` accepts only a frozen approved event dataclass, converts it with a schema-specific serializer, applies redaction, and emits JSON through the one configured structlog pipeline. Exception logging records an approved error code and exception type, not the raw traceback message, unless a schema-specific scrubber is present.

Use these event schemas; adding a field requires changing the corresponding dataclass and test:

```python
from collections.abc import Mapping


@dataclass(frozen=True, slots=True)
class RequestCompleted:
    correlation_id: str
    route_name: str
    status_code: int
    duration_ms: int


@dataclass(frozen=True, slots=True)
class AuthorizationRejected:
    correlation_id: str
    reason: str
    metadata: Mapping[str, object]


@dataclass(frozen=True, slots=True)
class AuditWriteFailed:
    correlation_id: str
    error_code: str
    exception_type: str
```

- [ ] **Step 5: Verify redaction, run the static gate, and commit**

Run:

```powershell
& '.\.venv\Scripts\python.exe' -m pytest tests/unit/observability tests/unit/tools -q
& '.\.venv\Scripts\python.exe' tools/check_logging_calls.py src/copytrader
& '.\.venv\Scripts\python.exe' -m ruff check src tests tools
```

Expected: canary secrets are absent, the production tree contains no bypass logging calls, and all checks pass.

```powershell
git add src/copytrader/observability tools/check_logging_calls.py tests/unit/observability tests/unit/tools
git commit -m "feat: enforce schema-filtered application logging"
```

---

### Task 8: Milestone verification, CI, and secure-foundation evidence

**Files:**
- Create: `.github/workflows/ci.yml`
- Create: `scripts/verify.ps1`
- Create: `docs/runbooks/development.md`
- Create: `docs/evidence/secure-foundation.md`
- Modify: `README.md`

**Interfaces:**
- Produces: `scripts/verify.ps1` as the single local verification entry point
- Produces: CI evidence for lint, types, unit tests, PostgreSQL integration tests, migration round-trip, and logging-policy checks
- Consumes: all Task 1–7 verification commands

- [ ] **Step 1: Add a failing verification-script smoke test**

Create a PowerShell parse check in CI before using the script:

```powershell
$errors = $null
[System.Management.Automation.Language.Parser]::ParseFile(
  (Resolve-Path 'scripts/verify.ps1'),
  [ref]$null,
  [ref]$errors
) | Out-Null
if ($errors.Count -ne 0) { throw ($errors | Out-String) }
```

Expected before `scripts/verify.ps1` exists: FAIL because the path cannot be resolved.

- [ ] **Step 2: Implement the local verification script**

`scripts/verify.ps1` must stop on the first failure and run exactly:

```powershell
$ErrorActionPreference = 'Stop'
$python = Join-Path $PSScriptRoot '..\.venv\Scripts\python.exe'
& $python -m ruff check src tests tools
& $python -m ruff format --check src tests tools
& $python -m mypy src
& $python tools/check_logging_calls.py src/copytrader
& $python -m pytest tests/unit -q
& $python -m pytest tests/integration -q
& (Join-Path $PSScriptRoot '..\.venv\Scripts\alembic.exe') downgrade base
& (Join-Path $PSScriptRoot '..\.venv\Scripts\alembic.exe') upgrade head
```

- [ ] **Step 3: Add CI with a real PostgreSQL 18 service**

Create this workflow, retaining the commands in the shown order:

```yaml
name: ci
on:
  pull_request:
  push:
    branches: [main]

jobs:
  verify:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:18
        env:
          POSTGRES_DB: copytrader_test
          POSTGRES_USER: copytrader
          POSTGRES_PASSWORD: copytrader
        ports: ["5432:5432"]
        options: >-
          --health-cmd "pg_isready -U copytrader -d copytrader_test"
          --health-interval 2s --health-timeout 2s --health-retries 20
    env:
      APP_ENV: test
      TEST_DATABASE_URL: postgresql+psycopg://copytrader:copytrader@localhost:5432/copytrader_test
      DATABASE_URL: postgresql+psycopg://copytrader:copytrader@localhost:5432/copytrader_test
      OIDC_ISSUER: https://issuer.invalid/
      OIDC_AUDIENCE: copytrader-api
      OIDC_PUBLIC_KEY_PATH: tests/fixtures/public-key.pem
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
          cache: pip
      - run: python -m pip install --upgrade pip
      - run: python -m pip install -e '.[dev]'
      - run: python -m ruff check src tests tools
      - run: python -m ruff format --check src tests tools
      - run: python -m mypy src
      - run: python tools/check_logging_calls.py src/copytrader
      - run: python -m pytest tests/unit -q
      - run: python -m pytest tests/integration -q
      - run: alembic downgrade base
      - run: alembic upgrade head
```

Cache only downloaded Python packages; never cache `.env`, database data, tokens, or test keys. Generate test RSA keys at test runtime instead of committing a private key; `OIDC_PUBLIC_KEY_PATH` is not opened by tests that inject their verifier.

- [ ] **Step 4: Record the milestone evidence and operating instructions**

`docs/evidence/secure-foundation.md` must list each Milestone 1 exit gate from the spec and the exact test file or command proving it. `docs/runbooks/development.md` must document PostgreSQL startup, environment variables, migrations, verification, teardown, and the rule that the integration suite must never fall back to SQLite.

- [ ] **Step 5: Run the complete milestone gate and commit**

Run:

```powershell
$env:TEST_DATABASE_URL='postgresql+psycopg://copytrader:copytrader@localhost:5432/copytrader_test'
& '.\scripts\verify.ps1'
git status --short
```

Expected: every verification command passes and Git shows only the intended Task 8 files.

```powershell
git add .github/workflows/ci.yml scripts/verify.ps1 docs/runbooks/development.md docs/evidence/secure-foundation.md README.md
git commit -m "ci: verify secure foundation milestone"
```

---

## Milestone completion gate

Do not begin the instrument/simulator plan until all of the following are true:

1. The full `scripts/verify.ps1` run passes against PostgreSQL 18.
2. CI passes from a clean checkout.
3. Cross-tenant, missing-context, invalid-token, inactive-membership, audit-mutation, float-rejection, and canary-secret cases all have explicit passing tests.
4. OpenAPI contains no target, allocation, risk, execution, or broker endpoint.
5. `docs/evidence/secure-foundation.md` maps every Milestone 1 spec gate to reproducible evidence.
6. A fresh reviewer checks the complete milestone diff against Sections 5, 7, 18, 19, 20, and 25.1 of the specification.
