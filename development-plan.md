# Constituent Relationship Management — Phased Development Plan

> Project: `227-constituent-relationship-management` · Created: 2026-05-29
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan implements an AI-native, multi-tenant, open-source CRM for government agencies, elected offices, and civic organisations. It is built around the MVP feature set from `features.md` (unified constituent profile, multi-channel intake, AI-classified case management, templated outbound responses with AI drafts, constituent self-service portal, reporting dashboards) and extends through the v1.1 set (GIS-based routing, AI escalation prediction, Open311 GeoReport v2, audience segmentation, accessibility audits). Standards alignment (Open311, NIEM, WCAG 2.1 AA, OAuth 2.0/OIDC with iGov, NIST 800-53 audit, OpenAPI 3.1, MCP) is baked into the schema and API from Phase 1.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary language (backend) | Python 3.12 | Best ecosystem for ML/NLP work (transformers, sentence-transformers, spaCy) that drives the AI-native differentiation; mature government adoption (US Digital Service, GDS); strong async story via FastAPI. |
| API framework | FastAPI 0.115+ | Native OpenAPI 3.1 emission (required by `standards.md`); async/await fits webhook + LLM workloads; Pydantic v2 model validation aligns with JSON Schema 2020-12. |
| Web frontend | Next.js 15 (App Router) + React 19 + TypeScript | Two distinct UIs needed (staff console + constituent portal). Next.js gives SSR for accessibility (WCAG 2.1 AA is hard with pure SPA); App Router supports server actions; widely used by government digital services (US Web Design System has React components). |
| UI component library | US Web Design System (USWDS) 3.x + shadcn/ui | USWDS for the constituent-facing portal (Section 508 compliant out of the box, recognisable to US government users). shadcn/ui for the staff console (faster iteration, customisable). |
| Database | PostgreSQL 16 + PostGIS 3.4 | Required for GIS-based routing (PostGIS `ST_Contains` for boundary lookups); JSONB + GIN indexes support the hybrid schema (Data Model Suggestion 3); mature replication, RLS for tenant isolation, and FedRAMP-authorised managed offerings. |
| ORM / query layer | SQLAlchemy 2.0 (async) + Alembic migrations | Async support fits FastAPI; explicit SQL when needed; Alembic gives reviewable migrations (mandatory for government audit trails). |
| Task queue | Celery 5 + Redis 7 broker | Async workloads: inbound email/SMS processing, AI classification, campaign sends, GIS routing batch jobs, projection rebuilds. Celery is the most mature Python option and supports priority queues for SLA-sensitive work. |
| LLM provider abstraction | LiteLLM | Single interface across OpenAI, Anthropic, AWS Bedrock (FedRAMP), Azure OpenAI Government; required because government deployments often mandate specific providers. |
| Embedding model | `sentence-transformers/all-MiniLM-L6-v2` (local, default) + provider-pluggable | Self-hostable for FedRAMP boundary; small enough for CPU inference; good enough for case classification and similarity (deduplication). |
| Classification model | Fine-tuned DistilBERT for case category + service code | Better than rule-based keyword routing per `research.md` AI-Native Opportunity; explainable via attention weights; small enough to run on CPU. |
| Vector store | `pgvector` extension in PostgreSQL | Avoids a second database; sufficient for the case-similarity / dedup / RAG knowledge-base use cases at the scales government agencies need. |
| Object storage | S3-compatible (MinIO for self-host, AWS S3 for cloud) | Attachments and message media (Open311 `media_url`). |
| Email transport | SMTP via aiosmtplib (outbound) + IMAP via aioimaplib (inbound) | Standards-based; integrates with any agency mail system. Pluggable via interface for SendGrid / AWS SES in cloud deployments. |
| SMS transport | Twilio (default) with pluggable interface | Dominant US SMS provider; FedRAMP-Moderate authorisation through Twilio Public Sector; interface allows swapping for VoodooSMS, AWS SNS, etc. |
| Authentication (staff) | OAuth 2.0 / OIDC via authlib | Required by `standards.md`; supports SSO with Microsoft Entra / Okta / Google Workspace used by agencies. |
| Authentication (constituents) | OIDC with iGov Profile + login.gov adapter | `standards.md` mandates iGov Profile; login.gov is the US federal identity service for constituent identity verification. |
| Authorisation | Role-Based Access Control with department scoping | Required by MVP feature list; permissions stored as JSON arrays on roles per Data Model Suggestion 3. |
| Containerisation | Docker + docker-compose for dev/self-host; Kubernetes manifests for cloud | Government procurement frequently mandates container deployment; docker-compose keeps local dev simple. |
| Testing framework | pytest + pytest-asyncio + httpx test client | Python standard; async support; httpx for end-to-end API tests. |
| Frontend testing | Vitest + React Testing Library + Playwright for E2E | Vitest is faster than Jest for component tests; Playwright supports accessibility assertions via `@axe-core/playwright`. |
| Accessibility testing | axe-core (via Playwright) + Pa11y for batch portal audits | WCAG 2.1 AA verification is a v1.1 feature; build the tooling early. |
| Code quality (Python) | ruff (lint + format), mypy (type check), pyright (IDE) | ruff is the fastest, replaces flake8 + black + isort; mypy for typing. |
| Code quality (TypeScript) | ESLint + Prettier + TypeScript strict | Standard Next.js setup. |
| API documentation | OpenAPI 3.1 (auto) + Redoc + Swagger UI | OpenAPI 3.1 required by `standards.md`; FastAPI emits it natively. |
| Observability | OpenTelemetry traces + Prometheus metrics + structured JSON logs | Government deployments require auditability; OTLP exporters work with most observability platforms. |
| AI agent integration | MCP server exposing CRM operations | `standards.md` highlights MCP as the credible standard for AI-native CRM. |
| Data model approach | **Data Model Suggestion 3 (Hybrid Relational + JSONB)** | Best fit for multi-jurisdiction deployments where each tenant has different custom fields; fewer tables than fully normalised; PostgreSQL JSONB + GIN gives queryability; field schema registry validates JSONB at the application layer. Event sourcing (Suggestion 2) is deferred — its complexity is not justified at MVP, but the audit log table captures the same intent. Graph layer (Suggestion 4) is deferred to a later phase for the cross-agency journey backlog feature. |
| Licence | AGPL-3.0 | Matches the leading open-source incumbent (CiviCRM) and is appropriate for a public-good government CRM; deters proprietary forks without contribution. |

### Project Structure

```
constituent-crm/
├── pyproject.toml                 # Poetry / uv project file
├── poetry.lock
├── Dockerfile                     # API + worker image
├── docker-compose.yml             # Postgres+PostGIS, Redis, MinIO, API, worker, frontend
├── docker-compose.test.yml        # Test overrides
├── Makefile                       # dev shortcuts (up, migrate, test, lint, seed)
├── .env.example
├── README.md
├── alembic.ini
├── api/
│   ├── alembic/
│   │   ├── env.py
│   │   └── versions/              # SQL migrations
│   ├── src/
│   │   └── crm/
│   │       ├── __init__.py
│   │       ├── main.py            # FastAPI app factory
│   │       ├── config.py          # pydantic-settings
│   │       ├── db/
│   │       │   ├── base.py        # SQLAlchemy declarative base
│   │       │   ├── session.py     # async session factory
│   │       │   ├── models/        # ORM models (one file per aggregate)
│   │       │   └── seeds/         # YAML seed data (services, roles, etc.)
│   │       ├── schemas/           # Pydantic request/response models
│   │       ├── api/
│   │       │   ├── deps.py        # auth, tenant, db deps
│   │       │   ├── routes/        # FastAPI routers per resource
│   │       │   ├── open311/       # GeoReport v2 endpoints
│   │       │   └── mcp/           # MCP server
│   │       ├── services/          # business logic (case_service, routing_service)
│   │       ├── ai/
│   │       │   ├── classifier.py  # case classification
│   │       │   ├── dedup.py       # constituent deduplication
│   │       │   ├── sentiment.py   # sentiment + escalation risk
│   │       │   ├── drafting.py    # outbound response drafting
│   │       │   ├── embeddings.py  # embedding service
│   │       │   └── prompts/       # prompt templates
│   │       ├── integrations/
│   │       │   ├── email/         # SMTP + IMAP
│   │       │   ├── sms/           # Twilio + interface
│   │       │   ├── storage/       # S3/MinIO adapter
│   │       │   ├── login_gov/     # OIDC iGov adapter
│   │       │   └── llm/           # LiteLLM wrapper
│   │       ├── gis/
│   │       │   ├── routing.py     # PostGIS boundary lookups
│   │       │   └── importer.py    # GeoJSON / Shapefile importer
│   │       ├── workers/
│   │       │   ├── celery_app.py
│   │       │   ├── intake_tasks.py
│   │       │   ├── ai_tasks.py
│   │       │   ├── campaign_tasks.py
│   │       │   └── sla_tasks.py
│   │       ├── audit/
│   │       │   ├── logger.py      # audit_log writer
│   │       │   └── middleware.py
│   │       ├── auth/
│   │       │   ├── oidc.py
│   │       │   ├── rbac.py
│   │       │   └── jwt.py
│   │       └── utils/
│   │           ├── pagination.py
│   │           ├── case_number.py
│   │           └── jsonschema_validator.py
│   └── tests/
│       ├── conftest.py            # fixtures: db, tenant, http client
│       ├── unit/
│       ├── integration/
│       ├── api/
│       └── fixtures/              # JSON test data, sample emails, GeoJSON
├── frontend/
│   ├── package.json
│   ├── next.config.ts
│   ├── tailwind.config.ts
│   ├── apps/
│   │   ├── staff/                 # Next.js app for staff console
│   │   │   ├── app/
│   │   │   ├── components/
│   │   │   ├── lib/
│   │   │   └── tests/
│   │   └── portal/                # Next.js app for constituent portal (USWDS)
│   │       ├── app/
│   │       ├── components/
│   │       ├── lib/
│   │       └── tests/
│   ├── packages/
│   │   ├── api-client/            # generated OpenAPI TypeScript client
│   │   ├── ui-staff/              # shadcn/ui components
│   │   └── ui-portal/             # USWDS React components
│   └── e2e/                       # Playwright tests
│       ├── staff.spec.ts
│       ├── portal.spec.ts
│       └── a11y.spec.ts
├── infra/
│   ├── k8s/                       # Kubernetes manifests
│   ├── terraform/                 # optional infra modules
│   └── helm/                      # Helm chart
└── docs/
    ├── api.md
    ├── architecture.md
    ├── deployment.md
    └── ai-models.md
```

---

## Phase 1: Foundation — Project Scaffolding, Multi-Tenancy, Auth, Audit

### Purpose
Establish the runnable monorepo, baseline infrastructure (Postgres + PostGIS + Redis + MinIO via docker-compose), schema-management workflow (Alembic), authentication (OIDC for staff), tenant isolation (RLS or app-level tenant guard), and the audit log primitive that every subsequent phase will write to. After this phase a developer can `make up`, log in as a staff user, and create an empty tenant — and every action is recorded in the audit log.

### Tasks

#### 1.1 — Repository scaffolding and docker-compose stack

**What**: Set up the monorepo directory structure, Python project (`pyproject.toml` with FastAPI, SQLAlchemy 2, Alembic, Celery, pytest), Next.js apps (`staff` and `portal`), and a `docker-compose.yml` that runs Postgres 16 with PostGIS + pgvector, Redis 7, MinIO, the API, the Celery worker, and both Next.js apps.

**Design**:
- `pyproject.toml` pinned to Python ^3.12 with dependency groups (`main`, `dev`, `test`, `ai`).
- `docker-compose.yml` services:
  - `db`: `postgis/postgis:16-3.4` with `POSTGRES_DB=crm`, `POSTGRES_USER=crm`, init script enabling `pgvector` and `uuid-ossp`.
  - `redis`: `redis:7-alpine`.
  - `minio`: `minio/minio` with default bucket `crm-attachments`.
  - `api`: built from `Dockerfile`, mounts `api/` for hot reload, port 8000.
  - `worker`: same image, runs `celery -A crm.workers.celery_app worker -l info`.
  - `beat`: Celery beat for scheduled tasks (SLA checks, projection refresh).
  - `staff-ui`: `node:22-alpine`, port 3000.
  - `portal-ui`: `node:22-alpine`, port 3001.
- `.env.example` with all required env vars; settings loaded via `pydantic-settings` (`crm.config.Settings`).
- `Makefile` targets: `up`, `down`, `migrate`, `seed`, `test`, `lint`, `format`, `e2e`, `shell`, `logs`.

**Testing**:
- `Integration: docker-compose up --wait → all services healthy within 60s`
- `Integration: docker-compose exec api alembic current → returns valid revision ID`
- `Integration: psql -c "SELECT extname FROM pg_extension" → contains postgis, pgvector, uuid-ossp`
- `Unit: Settings loads with .env.example → no validation errors`

#### 1.2 — Database schema baseline (tenant, jurisdiction, audit_log, field_schema_registry)

**What**: Create the initial Alembic migration with the four foundational tables required by every other phase.

**Design**:
```sql
CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    jurisdiction_id UUID REFERENCES jurisdiction(id),
    tier            VARCHAR(50) NOT NULL DEFAULT 'standard',
    config          JSONB NOT NULL DEFAULT '{}',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE jurisdiction (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    country_code    CHAR(2) NOT NULL,
    subdivision_code VARCHAR(6),
    name            VARCHAR(255) NOT NULL,
    level           VARCHAR(50) NOT NULL,
    parent_id       UUID REFERENCES jurisdiction(id),
    timezone        VARCHAR(50) NOT NULL DEFAULT 'UTC',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE field_schema_registry (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id) ON DELETE CASCADE,
    entity_type     VARCHAR(50) NOT NULL,
    field_group     VARCHAR(100) NOT NULL,
    json_schema     JSONB NOT NULL,
    ui_schema       JSONB,
    version         INTEGER NOT NULL DEFAULT 1,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, entity_type, field_group, version)
);

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    actor_id        UUID,
    actor_type      VARCHAR(20) NOT NULL CHECK (actor_type IN ('staff','system','api','ai_agent','constituent','anonymous')),
    action          VARCHAR(50) NOT NULL,
    entity_type     VARCHAR(50) NOT NULL,
    entity_id       UUID,
    changes         JSONB,
    request_metadata JSONB,
    ip_address      INET,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE TABLE audit_log_2026_q2 PARTITION OF audit_log
    FOR VALUES FROM ('2026-04-01') TO ('2026-07-01');

CREATE INDEX idx_audit_tenant_time ON audit_log(tenant_id, created_at);
CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_actor ON audit_log(actor_id) WHERE actor_id IS NOT NULL;
```
- SQLAlchemy 2.0 declarative models in `crm/db/models/tenant.py`, `jurisdiction.py`, `field_schema.py`, `audit.py`.
- An Alembic seed script seeds the US (`country_code=US`) and a sample state (`US-CA`).

**Testing**:
- `Unit: Tenant model save/load roundtrip preserves config JSONB`
- `Unit: Jurisdiction.parent_id self-referencing FK constraint enforced`
- `Unit: AuditLog actor_type CHECK constraint rejects invalid value`
- `Integration: alembic upgrade head → downgrade base → upgrade head succeeds`
- `Integration: audit_log partition created for next quarter via scheduled task`

#### 1.3 — Tenant resolution middleware

**What**: Every API request resolves a tenant from one of: `X-Tenant-Slug` header, subdomain, or JWT `tenant_id` claim. The resolved tenant is attached to the request context and used to scope all subsequent DB queries.

**Design**:
```python
class TenantContext(BaseModel):
    id: UUID
    slug: str
    tier: Literal["standard", "enterprise", "federal"]
    config: dict

async def resolve_tenant(
    request: Request,
    x_tenant_slug: str | None = Header(default=None),
    db: AsyncSession = Depends(get_db),
) -> TenantContext: ...
```
- Resolution order: JWT claim > header > subdomain (`{slug}.crm.example.gov`).
- Failure modes: 404 `tenant_not_found`, 403 `tenant_inactive`.
- Tenant context stored in `request.state.tenant` and made available via FastAPI dep `Depends(get_tenant)`.
- All ORM models include `tenant_id` and a `SQLAlchemy` event listener auto-injects it on `INSERT` if not provided.

**Testing**:
- `Unit: resolve_tenant with valid slug header → returns TenantContext`
- `Unit: resolve_tenant with unknown slug → raises 404`
- `Unit: resolve_tenant with inactive tenant → raises 403`
- `Integration: GET /v1/constituents with no tenant → 400 tenant_required`
- `Integration: GET /v1/constituents with mismatched tenant → 403`

#### 1.4 — Staff authentication via OIDC

**What**: Staff users sign in via an external OIDC provider (Microsoft Entra, Okta, Google Workspace, or Keycloak in dev). Successful sign-in mints a CRM session JWT containing `sub` (staff_user.id), `tenant_id`, `roles`, and `permissions`.

**Design**:
- Use `authlib` for OIDC flows; configurable per-tenant in `tenant.config.auth`.
- Dev: docker-compose includes Keycloak with a pre-seeded realm, client, and two users (`admin@crm.test`, `caseworker@crm.test`).
- Endpoints:
  - `GET /v1/auth/login?provider=keycloak` → redirects to OIDC authorize endpoint with PKCE.
  - `GET /v1/auth/callback?code=...&state=...` → exchanges code, finds/creates `staff_user`, mints CRM JWT (RS256), sets `__Host-crm_session` cookie.
  - `POST /v1/auth/logout` → revokes session.
  - `GET /v1/auth/me` → returns current staff_user with roles/permissions.
- Staff user table created here (deferred to Phase 2's RBAC for permission attachment):
```sql
CREATE TABLE staff_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    email           VARCHAR(255) NOT NULL,
    display_name    VARCHAR(255) NOT NULL,
    department_id   UUID,                      -- FK added in Phase 2
    external_subject VARCHAR(255),             -- OIDC sub
    roles           JSONB NOT NULL DEFAULT '[]',
    permissions     JSONB NOT NULL DEFAULT '[]',
    preferences     JSONB NOT NULL DEFAULT '{}',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX idx_staff_email ON staff_user(tenant_id, email);
CREATE INDEX idx_staff_external ON staff_user(tenant_id, external_subject);
```
- JWT verification middleware reads `__Host-crm_session` cookie or `Authorization: Bearer` header.

**Testing**:
- `Integration (Keycloak via docker-compose): full OIDC code flow → CRM JWT issued with correct tenant_id and sub`
- `Integration: GET /v1/auth/me with valid cookie → 200 with staff details`
- `Integration: GET /v1/auth/me without cookie → 401`
- `Unit: JWT with expired exp → verification rejects`
- `Unit: JWT signed with wrong key → verification rejects`
- `Unit: staff_user auto-provisioned on first OIDC login with same external_subject`

#### 1.5 — Audit log writer and middleware

**What**: Every state-changing API request and every model mutation writes an `audit_log` row. Read requests can opt in via decorator for sensitive resources (e.g., constituent PII reads under CCPA).

**Design**:
```python
class AuditEvent(BaseModel):
    tenant_id: UUID
    actor_id: UUID | None
    actor_type: Literal["staff","system","api","ai_agent","constituent","anonymous"]
    action: str                                # create | read | update | delete | login | export | route | ...
    entity_type: str                           # constituent | case | activity | ...
    entity_id: UUID | None
    changes: dict | None                       # {"field": {"old": ..., "new": ...}}
    request_metadata: dict                     # {"path": ..., "method": ..., "user_agent": ...}
    ip_address: IPv4Address | IPv6Address | None

async def write_audit(event: AuditEvent, db: AsyncSession) -> None: ...
```
- SQLAlchemy `before_update` / `before_delete` events compute a diff and stage an audit event in the request scope.
- FastAPI middleware flushes staged audit events after a successful response (within the same DB transaction so audit + change commit atomically).
- Field-level redaction: a `sensitive_fields` set per entity type masks values in `changes` (replaces with `"***"`).
- A monthly Celery beat job creates the next partition.

**Testing**:
- `Unit: AuditEvent diff for {"status": "open" → "in_progress"} → changes = {"status": {"old": "open", "new": "in_progress"}}`
- `Unit: sensitive field "date_of_birth" → masked in changes`
- `Integration: POST /v1/constituents → audit_log row with action=create, entity_type=constituent`
- `Integration: failed POST (validation error) → no audit_log row written (transaction rolled back)`
- `Integration: monthly beat task creates audit_log_2026_q3 partition`

#### 1.6 — OpenAPI 3.1 generation and TypeScript client

**What**: FastAPI emits OpenAPI 3.1; a build script generates a typed TypeScript client into `frontend/packages/api-client/` for use by both UIs.

**Design**:
- `GET /openapi.json` returns OpenAPI 3.1 (FastAPI default).
- `make generate-client` runs `openapi-typescript` and `openapi-fetch` to emit `frontend/packages/api-client/src/index.ts`.
- Custom `operationId` per route for stable generated function names.
- A CI check fails if the generated client is out of date.

**Testing**:
- `Integration: GET /openapi.json → valid OpenAPI 3.1.x document (validate via openapi-schema-validator)`
- `Unit: generated client compiles with TypeScript strict mode`
- `CI: generated client matches committed version (drift check)`

---

## Phase 2: Constituent and Department Management

### Purpose
Implement the core domain entities of any CRM: constituents (individuals, organisations, households) with embedded contact channels, plus departments, roles, and the geographic_boundary table that Phase 5's GIS routing will use. After this phase, staff can browse and edit constituent profiles and configure the organisational structure of the agency.

### Tasks

#### 2.1 — Constituent table, schemas, and CRUD API

**What**: Implement the `constituent` table from Data Model Suggestion 3 and a full CRUD API with search, pagination, and JSON Schema validation of `custom_fields` against the field_schema_registry.

**Design**:
- Migration adds the `constituent` table (full DDL from Suggestion 3) plus GIN indexes on `channels`, `tags`, `custom_fields`, `ai_metadata`.
- Pydantic schemas in `crm/schemas/constituent.py`:
```python
class ChannelEmail(BaseModel):
    type: Literal["email"]
    value: EmailStr
    is_primary: bool = False
    verified: bool = False

class ChannelPhone(BaseModel):
    type: Literal["phone"]
    value: str                                  # E.164
    phone_type: Literal["mobile","home","work","fax"] = "mobile"
    is_primary: bool = False
    verified: bool = False

class ChannelAddress(BaseModel):
    type: Literal["address"]
    street_line_1: str
    street_line_2: str | None = None
    city: str
    state: str                                  # ISO 3166-2 code
    postal: str
    country: str = "US"                         # ISO 3166-1 alpha-2
    lat: float | None = None
    lng: float | None = None

Channel = Annotated[
    ChannelEmail | ChannelPhone | ChannelAddress,
    Field(discriminator="type"),
]

class ConstituentCreate(BaseModel):
    constituent_type: Literal["individual","organisation","household"]
    display_name: str
    first_name: str | None = None
    last_name: str | None = None
    organisation_name: str | None = None
    preferred_language: str = "en"
    preferred_channel: Literal["email","sms","phone","mail"] = "email"
    external_id: str | None = None
    channels: list[Channel] = []
    relationships: list[Relationship] = []
    tags: list[str] = []
    custom_fields: dict = {}
```
- Endpoints:
  - `POST /v1/constituents` — create.
  - `GET /v1/constituents/{id}` — read.
  - `PATCH /v1/constituents/{id}` — partial update.
  - `DELETE /v1/constituents/{id}` — soft delete (sets `is_active=false`).
  - `GET /v1/constituents` — list with pagination + filters: `q` (full-text on `display_name`, `first_name`, `last_name`, `organisation_name`), `tag`, `channel_email`, `channel_phone`, `tenant_id` implicit from context.
- `custom_fields` validated at write time against the active schema in `field_schema_registry` (entity_type='constituent', field_group='custom').

**Testing**:
- `Unit: discriminated Channel union validates email vs phone vs address correctly`
- `Unit: ConstituentCreate with custom_fields={ward_number: "seven"} when schema requires integer → ValidationError`
- `Integration: POST /v1/constituents → 201 with generated UUID, audit_log row`
- `Integration: GET /v1/constituents?q=Santos → returns matching display_name results`
- `Integration: GET /v1/constituents?channel_email=maria@example.com → uses GIN index (verify via EXPLAIN)`
- `Integration: PATCH constituent in another tenant → 404`
- `Integration: DELETE constituent → is_active=false, not hard-deleted`

#### 2.2 — Department and staff_user RBAC

**What**: Implement `department` (hierarchical), `role`, and `staff_role` tables; complete `staff_user` schema; permission check helper.

**Design**:
- DDL:
```sql
CREATE TABLE department (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    parent_id       UUID REFERENCES department(id),
    config          JSONB NOT NULL DEFAULT '{}',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE role (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(100) NOT NULL,
    description     TEXT,
    permissions     JSONB NOT NULL DEFAULT '[]',
    is_system       BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
ALTER TABLE staff_user ADD CONSTRAINT staff_user_dept_fk
    FOREIGN KEY (department_id) REFERENCES department(id);
```
- Seed system roles per tenant: `admin`, `supervisor`, `case_worker`, `read_only`, `ai_reviewer`.
- Permission catalogue (`crm/auth/permissions.py`):
```python
PERMISSIONS = [
    "constituents:read","constituents:write","constituents:delete","constituents:export",
    "cases:read","cases:write","cases:assign","cases:close","cases:reopen",
    "campaigns:read","campaigns:write","campaigns:send",
    "reports:read","reports:export",
    "settings:read","settings:write",
    "ai:review","ai:override",
    "audit:read",
]
```
- Decorator `@require_permission("cases:write")` on routes.
- Department scoping: when permission is `cases:read:own_department`, queries are filtered by `department_id IN <user's departments>`.

**Testing**:
- `Unit: require_permission rejects staff missing permission → 403`
- `Unit: department-scoped query for case_worker → only own-department rows`
- `Integration: POST /v1/departments → created, audit_log row`
- `Integration: GET /v1/staff → admin sees all, read_only sees self only`
- `Integration: assign role to staff → permissions array recomputed`

#### 2.3 — Geographic boundary import and storage

**What**: Implement `geographic_boundary` table (PostGIS), and a CLI / API endpoint to import GeoJSON boundary files (wards, districts, zones). This table powers Phase 5's GIS routing.

**Design**:
- DDL:
```sql
CREATE TABLE geographic_boundary (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    boundary_type   VARCHAR(50) NOT NULL,           -- ward, district, precinct, zone, council_district
    external_code   VARCHAR(100),                    -- e.g. ward number
    department_id   UUID REFERENCES department(id),
    assigned_to     UUID REFERENCES staff_user(id),
    geom            GEOMETRY(MultiPolygon, 4326),
    properties      JSONB NOT NULL DEFAULT '{}',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_boundary_geom ON geographic_boundary USING GIST(geom);
CREATE INDEX idx_boundary_tenant_type ON geographic_boundary(tenant_id, boundary_type);
```
- CLI: `crm boundaries import --tenant <slug> --type ward --file wards.geojson [--dept-mapping wards-to-depts.csv]`.
- API: `POST /v1/boundaries/import` accepts multipart upload.
- Use `geoalchemy2` for SQLAlchemy ↔ PostGIS integration.
- Routing helper (exposed for Phase 5):
```python
async def find_boundary_for_point(
    tenant_id: UUID, lat: float, lng: float, boundary_type: str | None = None
) -> GeographicBoundary | None: ...
```

**Testing**:
- `Unit: CLI import of sample 2-ward GeoJSON → 2 boundary rows with valid geom`
- `Unit: find_boundary_for_point at known coordinate inside ward 1 → ward 1 returned`
- `Unit: find_boundary_for_point at coordinate outside all wards → None`
- `Integration: POST /v1/boundaries/import with malformed GeoJSON → 400 with line-level error`
- `Integration: query uses GIST index (EXPLAIN ANALYZE → Index Scan, not Seq Scan)`

#### 2.4 — Staff console: constituent list, profile, department admin

**What**: Build the first staff UI screens in the `staff` Next.js app: constituent search list, constituent detail page (with channels, tags, custom fields), and basic department/role admin.

**Design**:
- App structure:
```
apps/staff/app/
├── (auth)/login/page.tsx
├── (app)/
│   ├── layout.tsx                # sidebar nav + topbar
│   ├── constituents/
│   │   ├── page.tsx              # search list
│   │   └── [id]/page.tsx         # profile
│   └── admin/
│       ├── departments/page.tsx
│       ├── roles/page.tsx
│       └── staff/page.tsx
```
- Use `@tanstack/react-query` for data fetching, the generated `api-client`, and shadcn/ui components.
- All forms are server components where possible; client components for interactive search.
- Auth: middleware redirects unauthenticated requests to `/login`; login button calls `GET /v1/auth/login`.

**Testing**:
- `E2E (Playwright): login → constituent list → search "Santos" → click result → profile loads`
- `E2E: create constituent via form → list updates`
- `E2E: edit channels → save → audit log entry visible`
- `A11y (axe-core): all admin pages → 0 critical violations`
- `Component: ConstituentSearch debounces queries to 300ms`

---

## Phase 3: Case Management — The Core Workflow

### Purpose
Ship the case lifecycle: create, assign, route, comment, status-change, resolve, close. This is the heart of the product. Cases are linked to constituents (participants) and carry both typed status/SLA fields and JSONB metadata for AI classification and custom fields. Activities (the interaction log) are introduced here because every case event is an activity.

### Tasks

#### 3.1 — Case and activity tables, lifecycle services

**What**: Implement `constituent_case` and `activity` tables from Suggestion 3, plus a `CaseService` that encapsulates state transitions and emits activities + audit events.

**Design**:
- DDL:
```sql
CREATE TABLE constituent_case (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    case_number     VARCHAR(50) NOT NULL,
    case_type       VARCHAR(50) NOT NULL,
    status          VARCHAR(30) NOT NULL DEFAULT 'open',
    priority        VARCHAR(20) NOT NULL DEFAULT 'normal',
    subject         VARCHAR(500) NOT NULL,
    description     TEXT,
    department_id   UUID REFERENCES department(id),
    assigned_to     UUID REFERENCES staff_user(id),
    sla_due_at      TIMESTAMPTZ,
    sla_breached    BOOLEAN NOT NULL DEFAULT false,
    latitude        DECIMAL(10, 7),
    longitude       DECIMAL(10, 7),
    address_string  VARCHAR(500),
    source_channel  VARCHAR(30),
    participants    JSONB NOT NULL DEFAULT '[]',
    open311         JSONB,
    ai_classification JSONB NOT NULL DEFAULT '{}',
    resolution      JSONB,
    custom_fields   JSONB NOT NULL DEFAULT '{}',
    opened_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    first_response_at TIMESTAMPTZ,
    resolved_at     TIMESTAMPTZ,
    closed_at       TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT case_status_chk CHECK (status IN ('open','in_progress','pending','escalated','resolved','closed','reopened')),
    CONSTRAINT case_priority_chk CHECK (priority IN ('low','normal','high','urgent'))
);
CREATE UNIQUE INDEX idx_case_number ON constituent_case(tenant_id, case_number);
CREATE INDEX idx_case_status ON constituent_case(tenant_id, status);
CREATE INDEX idx_case_assigned ON constituent_case(assigned_to) WHERE assigned_to IS NOT NULL;
CREATE INDEX idx_case_department ON constituent_case(department_id);
CREATE INDEX idx_case_sla ON constituent_case(tenant_id, sla_due_at) WHERE NOT sla_breached AND status NOT IN ('resolved','closed');
CREATE INDEX idx_case_created ON constituent_case(tenant_id, created_at DESC);
CREATE INDEX idx_case_participants ON constituent_case USING GIN(participants jsonb_path_ops);
CREATE INDEX idx_case_open311 ON constituent_case USING GIN(open311 jsonb_path_ops);
CREATE INDEX idx_case_ai ON constituent_case USING GIN(ai_classification jsonb_path_ops);

CREATE TABLE activity (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    case_id         UUID REFERENCES constituent_case(id),
    constituent_id  UUID REFERENCES constituent(id),
    activity_type   VARCHAR(50) NOT NULL,
    direction       VARCHAR(10),
    channel         VARCHAR(30),
    subject         VARCHAR(500),
    body            TEXT,
    participants    JSONB NOT NULL DEFAULT '[]',
    metadata        JSONB NOT NULL DEFAULT '{}',
    performed_by    UUID REFERENCES staff_user(id),
    performed_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_activity_case ON activity(case_id) WHERE case_id IS NOT NULL;
CREATE INDEX idx_activity_constituent ON activity(constituent_id) WHERE constituent_id IS NOT NULL;
CREATE INDEX idx_activity_tenant_time ON activity(tenant_id, performed_at DESC);
CREATE INDEX idx_activity_type ON activity(tenant_id, activity_type);
```
- Case number generator: `{tenant_prefix}-{YYYY}-{6-digit sequence}` from a per-tenant `case_sequence` row (`SELECT nextval` style with row lock).
- Service methods:
```python
class CaseService:
    async def create(self, tenant_id, payload: CaseCreate, actor: Actor) -> Case
    async def assign(self, case_id, assignee_id, actor: Actor, reason: str | None) -> Case
    async def change_status(self, case_id, new_status, actor: Actor, note: str | None) -> Case
    async def change_priority(self, case_id, new_priority, actor: Actor) -> Case
    async def resolve(self, case_id, resolution: ResolutionPayload, actor: Actor) -> Case
    async def close(self, case_id, actor: Actor) -> Case
    async def reopen(self, case_id, actor: Actor, reason: str) -> Case
    async def add_comment(self, case_id, body, actor: Actor) -> Activity
```
- State machine enforces valid transitions: `open → in_progress | pending | escalated`, `in_progress → pending | escalated | resolved`, `pending → in_progress | resolved`, `resolved → closed | reopened`, `closed → reopened`.

**Testing**:
- `Unit: invalid transition (closed → open) → InvalidTransitionError`
- `Unit: case_number generator → unique sequence per tenant, monotonic per year`
- `Unit: assign() emits activity with type='assignment' and audit log`
- `Integration: POST /v1/cases with 3 participants → constituent_case row + 3 entries in participants JSONB`
- `Integration: resolve() sets resolved_at and writes resolution JSONB`
- `Integration: GET /v1/cases?status=open&assigned_to=me → uses idx_case_assigned`

#### 3.2 — Case API and JSON Schema validation for custom fields

**What**: REST endpoints for the case lifecycle, including filtering and pagination; validation of `custom_fields` and `open311.attributes` against the field schema registry.

**Design**:
- Endpoints:
  - `POST /v1/cases`
  - `GET /v1/cases?status=&priority=&assigned_to=&department=&service_code=&q=&sort=&page=&per_page=`
  - `GET /v1/cases/{id}` (includes full activity timeline)
  - `PATCH /v1/cases/{id}` (subject, description, priority, custom_fields)
  - `POST /v1/cases/{id}/assign`
  - `POST /v1/cases/{id}/status`
  - `POST /v1/cases/{id}/resolve`
  - `POST /v1/cases/{id}/close`
  - `POST /v1/cases/{id}/reopen`
  - `POST /v1/cases/{id}/comments`
  - `GET /v1/cases/{id}/activities`
- Pagination: cursor-based (`?cursor=...&limit=50`); cursor encodes `(created_at, id)`.
- JSON Schema validator (`crm/utils/jsonschema_validator.py`) loads active schema once per request and caches by `(tenant_id, entity_type, field_group)` for 60 s.

**Testing**:
- `Integration: POST /v1/cases with invalid custom_field value → 422 with JSON pointer to bad field`
- `Integration: list cases pagination → returns next_cursor, stable ordering`
- `Integration: POST /v1/cases/{id}/assign with insufficient permission → 403`
- `Integration: GET /v1/cases/{id} returns full activity timeline ordered DESC`

#### 3.3 — Staff console: case inbox, case detail, timeline

**What**: Three primary staff screens — `Cases` (filterable inbox), `Case Detail` (subject/description, participants, activity timeline, action toolbar), and `My Work` (cases assigned to me).

**Design**:
- Pages:
```
apps/staff/app/(app)/
├── cases/
│   ├── page.tsx               # filterable inbox (Server Component with searchParams)
│   ├── new/page.tsx           # case creation form
│   └── [id]/page.tsx          # case detail
├── my-work/page.tsx
```
- Case detail layout: left column (subject/description/SLA/priority), centre column (timeline of activities), right column (participants + custom fields + AI panel placeholder).
- Status changes via shadcn `Dialog` with required reason for `escalated` / `reopened`.
- Real-time refresh of the timeline via polling every 30 s (WebSockets deferred to a later phase).

**Testing**:
- `E2E: create case → appears in inbox → open detail → add comment → comment appears in timeline`
- `E2E: assign case to self → My Work shows the case`
- `E2E: resolve case → status pill changes, resolution form requires notes`
- `A11y: case detail page → all interactive controls have accessible names, focus order correct`

---

## Phase 4: Inbound Intake — Web Forms, Email, SMS, Webhook

### Purpose
Wire up the multi-channel intake required by the MVP. Cases originate from many places; this phase implements ingestion from web forms, inbound email (IMAP polling + inbound SMTP webhook), SMS (Twilio webhook), and a generic webhook endpoint. Each channel produces a case + initial activity + (eventually) an AI classification job.

### Tasks

#### 4.1 — Web form intake and constituent self-service portal scaffolding

**What**: Build the constituent-facing portal (Next.js with USWDS), with a dynamic form generator that renders an intake form from the field schema registry, and an endpoint that creates a case (and constituent if anonymous) from form submissions.

**Design**:
- `apps/portal/app/`:
```
├── (public)/
│   ├── page.tsx              # home + service catalogue
│   ├── services/[code]/page.tsx  # service detail + intake form
│   └── status/[token]/page.tsx   # anonymous status lookup
├── (authenticated)/
│   ├── dashboard/page.tsx
│   └── requests/[id]/page.tsx
```
- Form renderer (`apps/portal/components/SchemaForm.tsx`) consumes the JSON Schema + UI Schema from `field_schema_registry` (entity_type='service_request', field_group=service_code).
- Endpoint:
  - `POST /v1/intake/web` — body: `{service_code, subject, description, location?, channels[], custom_fields}`.
  - Anonymous submissions allowed if `service.metadata=false`; create-or-find constituent by email/phone.
  - Issues an Open311-compatible `token` for status lookup.
- USWDS theme via `@uswds/uswds` package; meets Section 508.

**Testing**:
- `Integration: POST /v1/intake/web with valid payload → 201, case created, constituent matched/created`
- `Integration: anonymous submission → token returned; GET /v1/intake/status/{token} → status visible`
- `E2E (Playwright): fill intake form → submit → confirmation page shows tracking number`
- `A11y (axe + Pa11y): portal home, service detail, intake form → 0 critical/serious WCAG 2.1 AA violations`

#### 4.2 — Email intake (IMAP poller + inbound webhook)

**What**: Two ingestion paths for email: (a) a Celery beat task polling an IMAP mailbox per tenant, and (b) an inbound webhook (e.g., SendGrid Inbound Parse, AWS SES). Both parse the message, find or create the constituent by `From:` address, and create a case (or reply to existing case if `In-Reply-To` matches).

**Design**:
- IMAP config in `tenant.config.email.imap = {host, port, username, password_secret_ref, ssl, mailbox}`.
- Worker `crm.workers.intake_tasks.poll_imap(tenant_id)` runs every 60 s per configured tenant.
- Parser uses `email` stdlib + `mail-parser`:
```python
class ParsedEmail(BaseModel):
    message_id: str
    in_reply_to: str | None
    from_addr: EmailStr
    from_name: str | None
    to: list[EmailStr]
    cc: list[EmailStr]
    subject: str
    body_text: str
    body_html: str | None
    attachments: list[ParsedAttachment]
    received_at: datetime
```
- Threading: if `in_reply_to` matches an existing outbound message's `Message-ID` (stored in `outbound_message.message_id`), append as activity to that case. Otherwise create a new case.
- Inbound webhook endpoint `POST /v1/intake/email/inbound` (HMAC-verified) accepts a normalised payload from the provider.
- Attachments uploaded to MinIO, recorded in `attachment` table (created here).

**Testing**:
- `Unit: parse multipart email with HTML+text+2 attachments → ParsedEmail with attachments list of size 2`
- `Unit: parse email with In-Reply-To matching existing outbound → routes to existing case`
- `Integration (mock IMAP with aioimaplib): poll fetches 2 unseen messages → 2 cases + 2 activities created`
- `Integration: webhook with invalid HMAC → 401`
- `Integration: SES-format webhook → 1 case created with attachment in MinIO`

#### 4.3 — SMS intake via Twilio webhook

**What**: Implement `POST /v1/intake/sms/twilio` to receive inbound SMS, validate Twilio signature, find/create constituent by phone, and create a case or thread reply.

**Design**:
- Twilio webhook signature validation per `twilio.request_validator.RequestValidator`.
- Body: `From`, `To`, `Body`, `MessageSid`, `NumMedia` etc.
- Find constituent by E.164 phone in `channels` JSONB; create with minimal profile if not found.
- Threading: if the SMS body starts with `CASE-...` token or replies within 24h of a previous outbound SMS to the same number, attach to that case.

**Testing**:
- `Unit: signature validator → rejects bad signature`
- `Integration: valid inbound SMS, new sender → constituent created, case created`
- `Integration: inbound SMS within 24h of outbound → activity appended, no new case`
- `Integration: MMS with media URL → media downloaded to MinIO, attachment row created`

#### 4.4 — Generic webhook intake and Open311 mobile-app submission

**What**: A generic `POST /v1/intake/webhook` endpoint that accepts a normalised payload (used by third-party mobile apps that aren't Open311-compatible). The full Open311 GeoReport v2 surface lands in Phase 7.

**Design**:
- Endpoint signature:
```http
POST /v1/intake/webhook
Authorization: Bearer <tenant_api_key>
Content-Type: application/json
{
  "source": "mobile_app",
  "service_code": "POTHOLE",
  "constituent": {"email": "...", "phone": "..."},
  "subject": "...",
  "description": "...",
  "location": {"lat": ..., "lng": ..., "address": "..."},
  "media_urls": [...],
  "custom_fields": {...}
}
```
- API keys stored in new `api_key` table (`tenant_id`, `name`, `hash`, `scopes`, `created_at`, `last_used_at`).

**Testing**:
- `Integration: webhook with valid token + payload → case created with source_channel=mobile_app`
- `Integration: webhook with revoked token → 401`
- `Integration: webhook with media_urls → attachments fetched and stored`

---

## Phase 5: AI-Native Workflows — Classification, Routing, Drafting

### Purpose
Deliver the differentiating AI capabilities of the MVP: automatic case classification + priority scoring, GIS-aware routing, basic deduplication, and AI-assisted response drafting. Every AI action is recorded as a structured activity with model version, confidence, and an explanation — satisfying the explainability requirement from `research.md` and `features.md`.

### Tasks

#### 5.1 — Case classification + priority scoring

**What**: A Celery task that, on case creation, runs a fine-tuned classifier to assign `ai_classification.category`, `ai_classification.priority_score`, and `ai_classification.confidence`. The result is written to `constituent_case.ai_classification` and an activity is recorded.

**Design**:
- `crm/ai/classifier.py`:
```python
class ClassificationResult(BaseModel):
    category: str
    confidence: float
    priority_score: float                        # 0.0-1.0
    suggested_service_code: str | None
    suggested_department_id: UUID | None
    model_name: str
    model_version: str
    explanation: str                             # human-readable reason
    top_features: list[tuple[str, float]]        # top 5 token contributions

class CaseClassifier:
    async def classify(
        self, subject: str, description: str, tenant_id: UUID
    ) -> ClassificationResult: ...
```
- Two-tier model:
  1. Fast: `sentence-transformers/all-MiniLM-L6-v2` embeddings + per-tenant logistic regression head trained on labelled historical cases. Falls back to zero-shot from a default model if the tenant has < 50 labelled cases.
  2. Optional LLM fallback: when confidence < 0.6, escalate to LiteLLM-routed model with a structured-output prompt asking for category and reason. Stored explanation is the LLM's reason.
- Worker task: `classify_case(case_id)` triggered by `case.created` signal.
- Activity recorded:
```json
{
  "activity_type": "ai_classification",
  "metadata": {
    "category": "road_maintenance",
    "confidence": 0.92,
    "priority_score": 0.78,
    "model_name": "case-classifier",
    "model_version": "v1.0.0",
    "explanation": "Keywords 'pothole', 'road', 'crater' match road_maintenance category (12 similar historical cases).",
    "top_features": [["pothole", 0.41], ["road", 0.22], ["crater", 0.18]]
  }
}
```

**Testing**:
- `Unit: classifier with 'There is a huge pothole on Main Street' → category='road_maintenance', confidence > 0.7`
- `Unit: low-confidence path triggers LLM fallback (mocked LiteLLM)`
- `Integration: case create → within 5 s ai_classification populated`
- `Integration: ai_review role can override category via PATCH /v1/cases/{id}/ai-classification`
- `Fixture: 50 labelled sample cases → trained head → >70% accuracy on holdout`

#### 5.2 — GIS-based routing + department assignment

**What**: When a case has `latitude`/`longitude`, automatically determine the geographic boundary, and combine it with the AI-suggested category to assign a department + (optionally) a primary case worker. Routing produces a structured explanation.

**Design**:
- `crm/ai/routing.py`:
```python
class RoutingDecision(BaseModel):
    department_id: UUID | None
    assigned_to: UUID | None
    routing_method: Literal["gis","category","gis+category","default","manual_required"]
    explanation: str
    boundary_id: UUID | None
    confidence: float

class CaseRouter:
    async def route(self, case: Case, classification: ClassificationResult) -> RoutingDecision: ...
```
- Algorithm:
  1. If lat/lng present → `find_boundary_for_point()` → boundary.department_id.
  2. If category present → look up `routing_rules` in `tenant.config.routing` keyed by category.
  3. Combine: prefer GIS for category in `{road_maintenance, sanitation, parks}`; prefer category for `{benefits, casework}`.
  4. If both indicate same department → assigned_to = department's least-loaded available staff.
  5. Otherwise → leave unassigned, `routing_method=manual_required`, explanation describes the conflict.
- The decision is applied as `assign()` with `reason=routing.explanation`, producing the standard activity + audit trail.
- Configurable per-tenant: `tenant.config.routing.auto_apply: bool` (default true for MVP; orgs can set false to merely suggest).

**Testing**:
- `Unit: route with lat/lng inside ward 1 + road_maintenance → ward 1's public works department`
- `Unit: route with category=benefits and no location → social services dept by rule`
- `Unit: GIS / category conflict → manual_required with explanation listing both signals`
- `Integration: case created with location → assigned within 10 s, activity 'routing' recorded`
- `Integration: explainability — every routing decision returns explanation ≥ 1 sentence`

#### 5.3 — Constituent deduplication suggestions

**What**: On constituent creation, find candidate duplicates and store them in `ai_metadata.dedup_candidates`. Provide a `POST /v1/constituents/{id}/merge` endpoint for staff to confirm a merge.

**Design**:
- Strategy:
  1. Deterministic: exact email or phone match → high-confidence candidate.
  2. Fuzzy name: trigram (`pg_trgm`) similarity on `display_name` > 0.85.
  3. Embedding: compare `display_name + city + first channel` embeddings, top-3 by cosine similarity via `pgvector`.
- Each candidate stored with `{constituent_id, score, signals: [...], method}`.
- Merge: target absorbs source — channels, tags, custom_fields merged; participants in cases re-pointed; source soft-deleted; merge recorded as an event in the audit log.

**Testing**:
- `Unit: two records with same email → dedup_candidates contains the other with score 1.0`
- `Unit: name "Maria Santos" vs "Maria S Santos" → trigram-based candidate with explained signal`
- `Integration: POST /merge → source soft-deleted, target has merged channels, cases re-linked`
- `Integration: merge with same constituent_id (self-merge) → 400`

#### 5.4 — AI-assisted outbound response drafting

**What**: A `POST /v1/cases/{id}/draft-response` endpoint returns 1-3 candidate response drafts using LLM with context from the case and the tenant's response template library. Staff can edit and send via Phase 6's outbound transports.

**Design**:
- Prompt template (`crm/ai/prompts/draft_response.md`):
```
You are a constituent services assistant for {tenant.name}, a {jurisdiction.level} government agency.
Draft a courteous, plain-language response (max 200 words, US 7th-grade reading level)
to the following case.

Case subject: {case.subject}
Description: {case.description}
Status: {case.status}
Department: {department.name}
Constituent name: {constituent.display_name}
Preferred language: {constituent.preferred_language}

You MUST:
- Address the constituent by their preferred name.
- Reference the case number {case.case_number}.
- Be specific about next steps and timeframe.
- Avoid promises that depend on factors outside the agency's control.
- Adhere to policy snippets below.

Relevant templates / policy:
{retrieved_templates}

Return a JSON object with shape:
{"drafts": [{"subject": "...", "body": "...", "rationale": "..."}, ...]}
```
- Retrieval: query `message_template` (introduced in Phase 6) by category match + embedding similarity to case description; supply top 3.
- LLM call via LiteLLM with `response_format={"type":"json_object"}`.
- Activity recorded: `activity_type=ai_draft`, metadata includes `model`, `confidence`, `accepted`, edit_distance computed when accepted.

**Testing**:
- `Unit: prompt assembly with mock case → contains required tokens`
- `Unit: LLM response parser handles missing 'drafts' key → DraftGenerationError`
- `Integration (mocked LiteLLM): POST draft-response → 3 drafts returned, activity recorded`
- `Integration: spanish-preference constituent → drafts in Spanish`
- `Integration: rate limit at 10 draft requests/minute per staff user → 429`

#### 5.5 — MCP server exposing CRM operations to AI agents

**What**: A Model Context Protocol server (mounted at `/mcp` or as a separate process) exposing tools for: search constituents, search cases, get case, create case, add comment, draft response, get service catalogue. Allows external AI agents (Claude, Copilot) to interact with the CRM with full audit logging.

**Design**:
- Use `mcp-python-sdk`. Server registers tools, each backed by the same service functions used by the API.
- Authentication via OAuth 2.0 Token Exchange or per-agent API keys with explicit scopes; every call writes an audit_log row with `actor_type=ai_agent` and the agent name.
- Resources also exposed: `crm://constituents/{id}`, `crm://cases/{id}`.

**Testing**:
- `Integration: MCP tools/list returns the registered tool set with input schemas`
- `Integration: tools/call search_cases with filter → results returned, audit row recorded`
- `Integration: unauthorised agent → tool call rejected with policy explanation`

---

## Phase 6: Outbound Communications and Templates

### Purpose
Complete the outbound side of the MVP: a templated message library, an `outbound_message` log, transport adapters for email (SMTP/SES) and SMS (Twilio), and the UI for staff to send a response from a case (optionally seeded by the AI draft). Audience segmentation arrives later in Phase 8.

### Tasks

#### 6.1 — Message templates and template variables

**What**: Implement `message_template` table and CRUD; templates support Jinja-style variable interpolation with a strict allow-list of variables.

**Design**:
```sql
CREATE TABLE message_template (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    channel         VARCHAR(20) NOT NULL CHECK (channel IN ('email','sms')),
    category        VARCHAR(100),
    content         JSONB NOT NULL,
    -- content example: { "subject": "...", "body_html": "...", "body_text": "...", "variables": [...] }
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_template_tenant_channel ON message_template(tenant_id, channel);
```
- Variable resolver allow-list: `{constituent.display_name, constituent.first_name, case.case_number, case.subject, case.status, department.name, tenant.name, link.portal_status}`.
- Renderer uses `jinja2.sandbox.SandboxedEnvironment` with the allow-list dict.

**Testing**:
- `Unit: render template with {{ constituent.display_name }} → substituted`
- `Unit: template with unknown variable {{ secret }} → renders as empty string + warning logged`
- `Unit: template with `{% raw %}{% include %}{% endraw %}` → blocked by sandbox`
- `Integration: POST /v1/templates → created; PATCH → updates; activity recorded`

#### 6.2 — Outbound message log + transport adapters

**What**: `outbound_message` table and transport adapter interface; SMTP, AWS SES, Twilio implementations. Sending records status transitions (queued → sent → delivered → bounced / failed).

**Design**:
```sql
CREATE TABLE outbound_message (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    campaign_id     UUID,                                -- nullable, links to campaign (Phase 8)
    template_id     UUID REFERENCES message_template(id),
    constituent_id  UUID NOT NULL REFERENCES constituent(id),
    case_id         UUID REFERENCES constituent_case(id),
    channel         VARCHAR(20) NOT NULL,
    recipient       VARCHAR(255) NOT NULL,
    subject         VARCHAR(500),
    body            TEXT,
    message_id      VARCHAR(255),                        -- transport message-id for threading
    status          VARCHAR(20) NOT NULL DEFAULT 'queued',
    provider        VARCHAR(50),
    provider_response JSONB,
    queued_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    sent_at         TIMESTAMPTZ,
    delivered_at    TIMESTAMPTZ,
    opened_at       TIMESTAMPTZ,
    bounced_at      TIMESTAMPTZ,
    failed_at       TIMESTAMPTZ,
    CONSTRAINT outbound_status_chk CHECK (status IN ('queued','sent','delivered','opened','bounced','failed','suppressed'))
);
CREATE INDEX idx_outbound_tenant_status ON outbound_message(tenant_id, status);
CREATE INDEX idx_outbound_constituent ON outbound_message(constituent_id);
CREATE INDEX idx_outbound_case ON outbound_message(case_id) WHERE case_id IS NOT NULL;
```
- Transport interface:
```python
class EmailTransport(Protocol):
    async def send(self, msg: OutboundEmail) -> SendResult: ...

class SmsTransport(Protocol):
    async def send(self, msg: OutboundSms) -> SendResult: ...
```
- Implementations: `SmtpEmailTransport`, `SesEmailTransport`, `TwilioSmsTransport`.
- Delivery webhooks `POST /v1/transport/email/webhook` and `/v1/transport/sms/webhook` update status; HMAC validated.
- Suppression list: unsubscribed/bounced recipients prevent further sends (per channel).

**Testing**:
- `Unit: SMTP transport with mock aiosmtplib → SendResult.message_id captured`
- `Integration: send outbound_message via /v1/cases/{id}/messages → row inserted, queued, async send fires`
- `Integration: SES bounce webhook → status=bounced, suppression record created`
- `Integration: send to suppressed recipient → 422 suppressed`

#### 6.3 — Staff console: send response from case

**What**: From a case detail page, staff can pick a template or "Use AI draft", edit, and send via email or SMS to one of the case participants. Sent messages appear as activities.

**Design**:
- Modal in case detail with three steps: choose channel + recipient → choose template or AI draft → preview + edit → send.
- Calls `POST /v1/cases/{id}/draft-response` if AI draft chosen.
- Sends via `POST /v1/cases/{id}/messages`.
- On success, the timeline refreshes and shows the new outbound activity.

**Testing**:
- `E2E: case detail → Reply → choose template → preview → send → activity appears`
- `E2E: AI draft path → 3 drafts shown → pick → edit → send`
- `Component: SendMessageModal disables Send button until recipient + body present`

---

## Phase 7: Open311 GeoReport v2 API + Service Catalogue

### Purpose
Ship the v1.1 standards-compliance feature: Open311 GeoReport v2 endpoints. This makes the CRM interoperable with third-party civic reporting apps (FixMyStreet etc.), satisfies the standards.md requirement, and packages the service catalogue (the `service` table) used by intake forms.

### Tasks

#### 7.1 — Service catalogue and service attributes

**What**: Implement the `service` and `service_attribute` tables and admin endpoints, plus seed Open311 standard services.

**Design**:
```sql
CREATE TABLE service (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    service_code    VARCHAR(50) NOT NULL,
    service_name    VARCHAR(255) NOT NULL,
    description     TEXT,
    group_name      VARCHAR(100),
    metadata        BOOLEAN NOT NULL DEFAULT false,
    type            VARCHAR(20) NOT NULL DEFAULT 'realtime' CHECK (type IN ('realtime','batch','blackbox')),
    keywords        TEXT[] NOT NULL DEFAULT '{}',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    default_department_id UUID REFERENCES department(id),
    sla_hours       INTEGER,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX idx_service_code ON service(tenant_id, service_code);

CREATE TABLE service_attribute (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    service_id      UUID NOT NULL REFERENCES service(id) ON DELETE CASCADE,
    attribute_code  VARCHAR(50) NOT NULL,
    description     TEXT NOT NULL,
    datatype        VARCHAR(20) NOT NULL CHECK (datatype IN ('string','number','datetime','text','singlevaluelist','multivaluelist')),
    is_required     BOOLEAN NOT NULL DEFAULT false,
    sort_order      INTEGER NOT NULL DEFAULT 0,
    values          JSONB,
    UNIQUE (service_id, attribute_code)
);
```
- Seed: import a default catalogue of 25 services from `api/src/crm/db/seeds/services.yaml` (potholes, graffiti, noise, parks, etc.).
- Admin endpoints under `/v1/services` for CRUD.

**Testing**:
- `Integration: GET /v1/services → seeded catalogue returned`
- `Integration: POST /v1/services with duplicate code → 409`
- `Integration: service_attribute datatype constraint enforced`

#### 7.2 — Open311 GeoReport v2 endpoints

**What**: Implement the standard endpoints under `/open311/v2/` per https://wiki.open311.org/GeoReport_v2/. JSON and XML output (JSON primary), with optional API-key auth.

**Design**:
- Routes (all under `/open311/v2/`):
  - `GET /services.json` — list active services.
  - `GET /services/{service_code}.json` — service definition (attributes).
  - `POST /requests.json` — submit a service request. Required: `service_code`, location (`lat`+`lng` or `address_string`). Optional: contact (`email`, `first_name`, `last_name`, `phone`), `description`, `media_url`, `device_id`, attribute fields (`attribute[code]=value`).
  - `GET /requests.json?service_code=&status=&start_date=&end_date=` — list requests.
  - `GET /requests/{service_request_id}.json` — single request.
  - `GET /tokens/{token}.json` — async token → request_id resolution.
- Outputs JSON by default; `.xml` suffix returns XML.
- Maps Open311 fields onto `constituent_case` + `constituent_case.open311` JSONB:
  - `service_request_id` → public case_number.
  - `token` for async creation flow stored in `open311.token`.
  - `media_url` stored on the case + linked to attachment.
- API-key auth required if `tenant.config.open311.require_api_key=true`.

**Testing**:
- `Conformance: run open311-test-cases (community fixture) against the API → all endpoints pass`
- `Integration: POST /open311/v2/requests.json with valid payload → 201 with service_request_id`
- `Integration: GET requests with date range → filtered`
- `Integration: XML output well-formed and matches schema`
- `Integration: missing required attribute → 400 with code=400 and description per spec`

#### 7.3 — Public status portal page + Open311 status updates

**What**: Constituents (anonymous or authenticated) can look up their request status by token or case_number. Open311 `status` (`open` / `closed`) and `status_notes` are kept in sync with internal case lifecycle.

**Design**:
- Mapping: internal status `open|in_progress|pending|escalated` → Open311 `open`; `resolved|closed` → Open311 `closed`. `status_notes` derived from latest public-facing activity.
- Portal page `apps/portal/app/(public)/status/[token]/page.tsx` shows request + timeline of public-visible activities (resident notifications only — internal notes hidden).

**Testing**:
- `Integration: case resolved internally → Open311 GET returns status=closed`
- `E2E: portal status lookup with valid token → shows progress milestones`
- `Integration: anonymous status lookup with invalid token → 404 (not 401, to avoid token enumeration distinction)`

---

## Phase 8: Campaigns, Audience Segmentation, Reporting Dashboards

### Purpose
Deliver the remaining v1.1 features: targeted outbound campaigns, audience segmentation against constituent attributes, and the staff reporting dashboard (case volume, response time, resolution rate, SLA compliance). Introduces the projection layer needed to keep dashboards fast.

### Tasks

#### 8.1 — Campaign model and audience segmentation engine

**What**: Implement `campaign` table; an audience query DSL (JSON) compiled to SQL; segment preview + send job.

**Design**:
```sql
CREATE TABLE campaign (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    campaign_type   VARCHAR(50) NOT NULL CHECK (campaign_type IN ('email_blast','sms','newsletter','town_hall_invite','survey')),
    status          VARCHAR(30) NOT NULL DEFAULT 'draft' CHECK (status IN ('draft','scheduled','sending','sent','failed','cancelled')),
    audience_filter JSONB NOT NULL DEFAULT '{}',
    content         JSONB NOT NULL DEFAULT '{}',
    schedule        JSONB,
    metrics         JSONB NOT NULL DEFAULT '{}',
    created_by      UUID REFERENCES staff_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```
- Audience DSL example:
```json
{
  "all": [
    {"field": "tags", "op": "contains", "value": "ward-7"},
    {"field": "custom_fields.property_owner", "op": "eq", "value": true},
    {"any": [
      {"field": "channels[].type", "op": "eq", "value": "email"},
      {"field": "channels[].type", "op": "eq", "value": "sms"}
    ]}
  ]
}
```
- Compiler in `crm/services/audience.py` translates DSL → SQLAlchemy expressions, using GIN containment for tags / custom_fields.
- Preview: `POST /v1/audiences/preview` returns count + 25 samples; never the full list to prevent PII bulk-extract.
- Send: `POST /v1/campaigns/{id}/send` enqueues a Celery task that creates one `outbound_message` per recipient with backoff/throttle.

**Testing**:
- `Unit: DSL with nested all/any → correct SQLAlchemy expression`
- `Unit: DSL referencing forbidden field (e.g., date_of_birth) → ValidationError`
- `Integration: preview with broad filter → returns count and 25 samples`
- `Integration: send → outbound_message rows created, status transitions tracked`
- `Integration: campaign respects suppression list`

#### 8.2 — Dashboard projections (refreshed materialised views)

**What**: Create `proj_department_metrics` and `proj_case_status_daily` materialised views, refreshed by a Celery beat task. Dashboards query the projections (not the raw cases table) to stay fast.

**Design**:
```sql
CREATE MATERIALIZED VIEW proj_case_status_daily AS
SELECT
    tenant_id,
    department_id,
    DATE(opened_at) AS metric_date,
    COUNT(*) FILTER (WHERE status = 'open') AS open_count,
    COUNT(*) FILTER (WHERE status = 'in_progress') AS in_progress_count,
    COUNT(*) FILTER (WHERE status = 'resolved') AS resolved_count,
    COUNT(*) FILTER (WHERE status = 'closed') AS closed_count,
    COUNT(*) FILTER (WHERE sla_breached) AS sla_breached_count,
    AVG(EXTRACT(EPOCH FROM (resolved_at - opened_at))/3600) FILTER (WHERE resolved_at IS NOT NULL) AS avg_resolution_hours,
    AVG(EXTRACT(EPOCH FROM (first_response_at - opened_at))/3600) FILTER (WHERE first_response_at IS NOT NULL) AS avg_first_response_hours,
    COUNT(*) AS total_cases
FROM constituent_case
GROUP BY tenant_id, department_id, DATE(opened_at);
CREATE UNIQUE INDEX idx_proj_csd ON proj_case_status_daily(tenant_id, department_id, metric_date);
```
- Refresh task: `REFRESH MATERIALIZED VIEW CONCURRENTLY proj_case_status_daily` every 5 minutes.
- Endpoints:
  - `GET /v1/reports/case-volume?from=&to=&department=&group_by=day|week|month`
  - `GET /v1/reports/response-time?from=&to=&department=`
  - `GET /v1/reports/sla-compliance?from=&to=&department=`
  - `GET /v1/reports/category-mix?from=&to=` (uses `ai_classification.category`).

**Testing**:
- `Unit: refresh task runs without locking writes (CONCURRENTLY)`
- `Integration: insert 100 sample cases → after refresh, projection reflects them`
- `Integration: report endpoint pagination + date filtering`
- `Performance: report endpoint at 10k cases responds in <300ms`

#### 8.3 — Staff dashboard UI

**What**: `apps/staff/app/(app)/dashboard/page.tsx` shows: open cases, cases by status pie, response time trend line, SLA compliance gauge, top categories bar chart, recent escalations list.

**Design**:
- Server component fetches all dashboard data in parallel.
- Charts via `recharts` (lightweight, accessible).
- Filter controls in URL (`?from=2026-05-01&to=2026-05-29&department=...`).
- Each chart has an "Export CSV" button (audit-logged).

**Testing**:
- `E2E: dashboard loads in <2s with seeded data`
- `E2E: change date range → charts update`
- `A11y: charts have aria-labels and an accessible data table fallback`

---

## Phase 9: SLA Tracking, Escalation Prediction, Sentiment Scoring

### Purpose
Wire up the proactive AI features from v1.1: SLA breach prediction (proactive alerting before the breach), real-time sentiment + escalation risk scoring on every inbound message, and supervisor notifications.

### Tasks

#### 9.1 — SLA computation and beat task

**What**: Compute `sla_due_at` from `service.sla_hours`, `case.priority`, and tenant defaults at case creation. Beat task scans for upcoming/overdue cases and creates alerts.

**Design**:
- SLA function `compute_sla_due(case, service, tenant) -> datetime`:
  - Start from `case.opened_at`.
  - Use `service.sla_hours` if set; else `tenant.config.sla_defaults[priority]`; else 24 h.
  - Skip non-working hours if `department.config.working_hours` set.
- Beat task `crm.workers.sla_tasks.scan_sla` runs every minute:
  - Cases due within 4 h and not `sla_warning_sent` → activity `sla_warning`, notify assignee + supervisor.
  - Cases overdue → set `sla_breached=true`, activity `sla_breached`, notify supervisor.

**Testing**:
- `Unit: compute_sla_due with priority=urgent → opened_at + 4h`
- `Unit: working hours skip → 5pm Friday opening with weekend skip → next Monday`
- `Integration: SLA beat scan picks up cases within 4 h → warning activity created once`
- `Integration: breach sets sla_breached=true → reflected in idx_case_sla`

#### 9.2 — Escalation risk + SLA breach prediction model

**What**: A model that, given case features (priority, age, response gap, sentiment of communications), predicts probability of SLA breach. Stored in `ai_classification.breach_probability`. Cases above threshold flagged for supervisor review.

**Design**:
- Features: `priority`, `category`, `hours_since_opened`, `hours_since_last_response`, `num_inbound_messages`, `latest_sentiment`, `is_repeat_constituent`.
- Model: gradient-boosted classifier (`lightgbm`) trained on historical resolved cases. Falls back to heuristic ((`hours_since_opened/sla_hours`) + `negative_sentiment_factor`) if no trained model.
- Worker `predict_breach(case_id)` runs on case create, on each inbound activity, and every hour.

**Testing**:
- `Unit: heuristic for case 80% through SLA window with negative sentiment → breach_probability > 0.6`
- `Integration: a case progresses through messages → breach_probability updates`
- `Integration: cases > 0.7 breach probability appear in /v1/reports/at-risk`

#### 9.3 — Sentiment scoring on inbound messages

**What**: Every inbound activity (email body, SMS body, web message) is scored for sentiment (-1.0 to +1.0) and an `escalation_risk` (0-1). Stored on the activity metadata and aggregated to the constituent's `ai_metadata.sentiment_trend`.

**Design**:
- Model: cross-encoder fine-tuned for sentiment; `cardiffnlp/twitter-roberta-base-sentiment-latest` as default; provider-pluggable.
- Triggered as a Celery task after each inbound activity write.
- Constituent-level rollup: rolling-window average of last 5 sentiments → trend `improving | stable | declining`.

**Testing**:
- `Unit: "This is the third time I've called and nothing happens." → sentiment < -0.5, escalation_risk > 0.5`
- `Integration: inbound message → activity metadata updated within 5 s`
- `Integration: 5 negative messages in a row → constituent.ai_metadata.sentiment_trend = 'declining'`

#### 9.4 — Notifications to supervisors

**What**: A simple notification fan-out: in-app (notifications table + WebSocket-less polling) + email + (optional) SMS based on staff preferences. Trigger sources: SLA warnings/breaches, predicted breaches, high escalation-risk inbound messages, ai_override events.

**Design**:
```sql
CREATE TABLE notification (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    staff_user_id   UUID NOT NULL REFERENCES staff_user(id),
    type            VARCHAR(50) NOT NULL,
    title           VARCHAR(255) NOT NULL,
    body            TEXT,
    entity_type     VARCHAR(50),
    entity_id       UUID,
    is_read         BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    read_at         TIMESTAMPTZ
);
CREATE INDEX idx_notif_user_unread ON notification(staff_user_id, is_read, created_at DESC);
```
- Dispatch interface with email/SMS implementations reusing transports from Phase 6.
- Staff preferences in `staff_user.preferences.notifications`.

**Testing**:
- `Integration: SLA breach → notification row inserted + email sent to supervisor`
- `Integration: user marks notification read → is_read=true, audit row`
- `E2E: bell icon shows unread count, opens dropdown`

---

## Phase 10: Accessibility Audit Tooling, Compliance Hardening, Production Readiness

### Purpose
Final v1.1 deliverables and production hardening: built-in WCAG 2.1 AA audit tooling for the constituent portal (`/admin/accessibility` runs pa11y/axe scans against a list of public URLs and reports issues), data-export and data-deletion flows (CCPA/GDPR), full observability, container hardening, Helm chart for cloud deployment.

### Tasks

#### 10.1 — Accessibility audit dashboard

**What**: An admin page that runs accessibility scans against the public portal URLs and shows the results, scan history, and trend.

**Design**:
```sql
CREATE TABLE accessibility_scan (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    target_url      VARCHAR(1000) NOT NULL,
    standard        VARCHAR(50) NOT NULL DEFAULT 'WCAG2AA',
    tool            VARCHAR(50) NOT NULL,                  -- axe, pa11y
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    violations      JSONB NOT NULL DEFAULT '[]',
    summary         JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```
- Worker spawns headless Chrome via Playwright + `@axe-core/playwright` to scan URLs; results parsed and stored.
- Endpoints: `POST /v1/accessibility/scans`, `GET /v1/accessibility/scans?target_url=...`.
- Beat task runs weekly scan of registered public URLs.
- UI: list scans, drill into violations, link to specific element selectors.

**Testing**:
- `Integration: trigger scan against portal home → scan row created, violations populated`
- `Integration: scan with known violation (inject bad fixture page) → violation detected`
- `E2E: open scan → see ordered violation list with severity and rule ID`

#### 10.2 — Data export and deletion (GDPR/CCPA)

**What**: Endpoints for constituents (via authenticated portal session) and staff (admin) to export and delete constituent data.

**Design**:
- `POST /v1/constituents/{id}/data-export` — async; enqueues a job that produces a ZIP (constituent record, channels, all cases, all activities, all outbound messages) and uploads to MinIO; returns a signed download URL valid 24 h.
- `POST /v1/constituents/{id}/data-deletion` — soft-deletes constituent and anonymises (replace channel values with `[redacted]`, blank free-text fields in cases unless legal hold applies). Original `audit_log` entries are retained (anonymised); the deletion itself is logged.
- `legal_hold` flag on constituent prevents deletion until cleared.

**Testing**:
- `Integration: data-export → ZIP with all expected sections`
- `Integration: data-deletion → constituent anonymised, channels redacted, cases preserved but freetext anonymised`
- `Integration: data-deletion with legal_hold → 409`
- `Integration: signed download URL expires after 24h`

#### 10.3 — Observability and operational endpoints

**What**: Prometheus `/metrics`, OpenTelemetry tracing across API + worker, structured JSON logs with `correlation_id`, health/readiness endpoints, request-id propagation.

**Design**:
- Use `opentelemetry-instrumentation-fastapi`, `opentelemetry-instrumentation-sqlalchemy`, `opentelemetry-instrumentation-celery`.
- Custom metrics: `crm_case_created_total{tenant,source_channel}`, `crm_ai_classification_duration_seconds`, `crm_sla_breach_total`, `crm_outbound_message_total{channel,status}`.
- Endpoints: `GET /healthz` (liveness), `GET /readyz` (DB + Redis + MinIO + SMTP connectivity), `GET /metrics`.
- Logs: JSON with `timestamp`, `level`, `correlation_id`, `tenant_id`, `actor_id`, `message`, `extras`.

**Testing**:
- `Integration: /metrics endpoint exposes registered metrics`
- `Integration: request → trace spans for API + DB + Celery task with same trace_id`
- `Integration: /readyz fails if Redis unavailable`

#### 10.4 — Container hardening, Helm chart, deployment docs

**What**: Distroless final-stage Dockerfile, non-root user, read-only root filesystem; Helm chart for Kubernetes; deployment guide for self-host and cloud; backup/restore docs.

**Design**:
- Multi-stage Dockerfile: build stage installs deps with `uv`; final stage is `gcr.io/distroless/python3-debian12`, runs as user `1001`, read-only root.
- Helm chart in `infra/helm/crm/` with values for: replicas (api, worker), resource requests/limits, ingress, OIDC config, Postgres DSN, Redis URL, MinIO/S3 creds.
- `docs/deployment.md`: docker-compose path, Helm path, FedRAMP-aligned cloud notes (AWS GovCloud, Azure Government).
- `docs/backup-restore.md`: `pg_dump` + WAL archive for Postgres; MinIO mirror for attachments.

**Testing**:
- `Security: trivy scan on final image → no HIGH/CRITICAL CVEs`
- `Security: container starts as non-root, denied to write to /`
- `Integration: helm install in kind cluster → all pods Ready, /readyz=200`
- `Manual: deployment guide walkthrough works for new developer in <60 min`

#### 10.5 — Documentation and seed data for demo

**What**: A `seed-demo` Make target loads a realistic demo tenant (City of Springfield) with 5 departments, 10 staff, 1000 constituents, 2500 cases at varying lifecycle stages, 10 service types, GeoJSON for 8 wards, and 50 templates. Plus README polish and architecture diagram.

**Design**:
- `crm seed demo --tenant springfield` invokes a script that generates the dataset using Faker + small canonical templates. Idempotent.
- Diagrams via Mermaid checked into `docs/architecture.md` (system context, container, key sequence flows: intake → classification → routing → resolution).
- API reference auto-published from OpenAPI to `docs/api/`.

**Testing**:
- `Integration: seed-demo on empty DB → all entities present, dashboards populated`
- `Integration: seed-demo run twice → second run is no-op`
- `Manual: README walkthrough creates a working environment from scratch in ≤15 minutes`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation (scaffolding, multi-tenancy, OIDC, audit)
    │
    ├─→ Phase 2: Constituent + Department Management
    │       │
    │       └─→ Phase 3: Case Management (core workflow)
    │               │
    │               ├─→ Phase 4: Inbound Intake (web, email, SMS, webhook)
    │               │       │
    │               │       └─→ Phase 5: AI Workflows (classification, routing, dedup, drafts, MCP)
    │               │               │
    │               │               ├─→ Phase 6: Outbound Communications + Templates
    │               │               │       │
    │               │               │       └─→ Phase 8: Campaigns + Reporting Dashboards
    │               │               │
    │               │               └─→ Phase 9: SLA + Escalation Prediction + Sentiment
    │               │
    │               └─→ Phase 7: Open311 GeoReport v2 + Service Catalogue
    │                       (depends on Phase 3; can parallel with Phase 4 & 5)
    │
    └─→ Phase 10: Accessibility, Compliance, Production Readiness
            (final phase; touches all earlier phases)
```

**Parallelism opportunities** after the linear spine (Phases 1 → 2 → 3) is complete:
- Phase 4 (intake) and Phase 7 (Open311) can be developed concurrently — they share the service catalogue and case lifecycle but are otherwise independent.
- Phase 6 (outbound) and Phase 9 (SLA/sentiment) can begin once Phase 5 lands AI primitives.
- Phase 8 (campaigns + dashboards) requires Phase 6 (outbound transports) but can be split: the audience DSL + projections (8.1, 8.2) can be built in parallel with the dashboard UI (8.3).
- Phase 10's accessibility tooling (10.1) and observability (10.3) can begin alongside Phase 6 once the portal exists.

---

## Definition of Done (per phase)

Every phase is considered complete only when **all** of the following are true:

1. All tasks in the phase are implemented per their Design sections.
2. Unit and integration test scenarios listed in the Testing sections pass; new tests have ≥ 80 % branch coverage on phase-introduced code.
3. `ruff check`, `ruff format --check`, and `mypy --strict crm/` all pass with zero issues.
4. Frontend phases additionally pass `eslint`, `tsc --noEmit`, `vitest`, and `playwright test`.
5. New Alembic migrations are present, reviewable, and reversible (`alembic downgrade -1 && upgrade head` works).
6. Any new schema fields, env vars, and config keys are documented in `.env.example`, `docs/deployment.md`, and `README.md`.
7. New REST endpoints appear in `/openapi.json`, are picked up by the generated TypeScript client, and have a published OpenAPI description.
8. New AI components have a model card in `docs/ai-models.md` recording: model name/version, training data summary, intended use, known limitations, evaluation metrics, and fallback behaviour.
9. New endpoints that mutate data write `audit_log` rows in the same transaction.
10. `docker compose up --build` succeeds from a clean clone and the resulting stack passes `/readyz`.
11. Playwright accessibility checks for any new public-facing pages report zero serious/critical violations.
12. A short demo script exists under `docs/demo-scripts/phase-N.md` that exercises the new capability end-to-end.
