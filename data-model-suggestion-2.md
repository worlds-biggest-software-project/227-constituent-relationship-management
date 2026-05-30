# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: Constituent Relationship Management · Created: 2026-05-22

## Philosophy

This model treats every state change as an immutable domain event stored in an append-only event store. The current state of any entity (constituent, case, service request) is derived by replaying its event stream. Read-optimised projections (materialised views) are maintained separately for fast queries, following the CQRS (Command Query Responsibility Segregation) pattern.

Government case management has an inherent audit requirement: every change to a constituent record, every case status transition, every assignment decision must be traceable and reproducible. Rather than bolting an audit log onto a mutable relational schema, this model makes the audit trail the source of truth. The event store captures not just what changed but why — the intent behind every action — which is critical for explainable AI routing decisions and regulatory compliance.

This architecture is used by financial trading systems (where regulatory audit trails are mandatory), healthcare systems (where patient record history must be reconstructable), and increasingly by government digital services (UK Government Digital Service uses event-driven architecture). It is particularly well-suited to AI-native applications because ML model predictions and their outcomes can be captured as events, enabling continuous model evaluation and bias detection.

**Best for:** Agencies requiring complete audit trails, temporal queries ("what was the case status on March 15?"), and AI-driven workflows where every routing decision must be explainable and reviewable.

**Trade-offs:**
- (+) Complete, immutable audit trail by design — not an afterthought
- (+) Full temporal query capability: reconstruct any entity state at any point in time
- (+) AI routing decisions captured as events with full context, enabling bias analysis
- (+) Event replay enables rebuilding read models, fixing projection bugs, and creating new views of historical data
- (+) Natural fit for async/event-driven integrations with other government systems
- (-) Higher implementation complexity: requires event store, projections, and eventual consistency management
- (-) Read models are eventually consistent — staff may see stale data briefly after writes
- (-) Event schema evolution requires careful versioning (upcasting old events)
- (-) Storage requirements are significantly higher than mutable models (every change is persisted)
- (-) Debugging requires understanding event replay rather than inspecting current state directly

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| Open311 GeoReport v2 | Service request events map to Open311 lifecycle; read projections expose Open311-compliant API responses |
| NIEM Human Services Domain | Event payloads for case and person events align with NIEM exchange element semantics |
| ISO 3166-1/2 | Jurisdiction codes in event metadata and projection tables |
| NIST 800-53 AU Controls | The event store IS the audit trail — every event is immutable and timestamped, satisfying AU-2, AU-3, AU-6, AU-9 |
| FedRAMP | Immutable event store with cryptographic integrity (event hashing) supports FedRAMP continuous monitoring |
| OAuth 2.0 / OpenID Connect | Actor identity captured in every event's metadata for non-repudiation |
| ISO 8601 | All event timestamps use TIMESTAMPTZ with ISO 8601 formatting |

---

## Event Store (Source of Truth)

```sql
-- Core event store: append-only, immutable
CREATE TABLE event_store (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    stream_type     VARCHAR(50) NOT NULL,           -- constituent, case, service_request, campaign, etc.
    stream_id       UUID NOT NULL,                   -- aggregate root ID
    event_type      VARCHAR(100) NOT NULL,           -- e.g., ConstituentCreated, CaseAssigned, SlaBreached
    event_version   INTEGER NOT NULL,                -- schema version for this event type
    sequence_number BIGINT NOT NULL,                 -- monotonic within stream
    payload         JSONB NOT NULL,                  -- event-specific data
    metadata        JSONB NOT NULL DEFAULT '{}',     -- actor, ip, correlation_id, causation_id
    -- Integrity
    event_hash      VARCHAR(64),                     -- SHA-256 hash of payload + previous hash (tamper detection)
    -- Timing
    occurred_at     TIMESTAMPTZ NOT NULL,            -- when the event actually happened
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now(), -- when it was written to the store
    CONSTRAINT uq_stream_sequence UNIQUE (stream_id, sequence_number)
);

-- Primary access pattern: read events for a specific stream in order
CREATE INDEX idx_event_stream ON event_store(stream_id, sequence_number);
-- Cross-stream queries by type and time
CREATE INDEX idx_event_type_time ON event_store(tenant_id, event_type, occurred_at);
-- Tenant-scoped time range queries for projections
CREATE INDEX idx_event_tenant_time ON event_store(tenant_id, recorded_at);

-- Partition the event store by month for manageability
-- CREATE TABLE event_store_2026_05 PARTITION OF event_store
--     FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');
```

### Event Payload Examples

```sql
-- ConstituentCreated event payload:
-- {
--   "constituent_type": "individual",
--   "first_name": "Maria",
--   "last_name": "Santos",
--   "display_name": "Maria Santos",
--   "preferred_language": "es",
--   "channels": [
--     {"type": "email", "value": "maria.santos@example.com", "is_primary": true},
--     {"type": "phone", "value": "+1-555-0147", "phone_type": "mobile"}
--   ]
-- }

-- CaseCreated event payload:
-- {
--   "case_number": "SR-2026-00472",
--   "case_type": "service_request",
--   "subject": "Pothole on Main Street near 5th Avenue",
--   "description": "Large pothole approximately 2 feet wide...",
--   "source_channel": "mobile_app",
--   "latitude": 40.7128,
--   "longitude": -74.0060,
--   "address_string": "Main Street & 5th Avenue",
--   "requester_id": "550e8400-e29b-41d4-a716-446655440000",
--   "service_code": "POTHOLE"
-- }

-- CaseRouted event payload (AI routing with explainability):
-- {
--   "department_id": "dept-public-works-uuid",
--   "assigned_to": "staff-user-uuid",
--   "routing_method": "ai_gis",
--   "ai_model_version": "route-classifier-v2.3",
--   "ai_confidence": 0.9234,
--   "ai_explanation": "Classified as 'road_maintenance' (93.2% confidence). GIS boundary match: Ward 7 → Public Works Division B.",
--   "gis_boundary_id": "boundary-ward-7-uuid",
--   "previous_department_id": null
-- }

-- CaseEscalated event payload:
-- {
--   "escalated_by": "system",
--   "reason": "sla_breach_prediction",
--   "ai_breach_probability": 0.87,
--   "sla_due_at": "2026-05-25T17:00:00Z",
--   "predicted_resolution_at": "2026-05-27T14:00:00Z",
--   "supervisor_id": "supervisor-uuid"
-- }

-- SentimentScored event payload:
-- {
--   "message_id": "activity-uuid",
--   "sentiment": "negative",
--   "sentiment_score": -0.72,
--   "escalation_risk": 0.65,
--   "model_version": "sentiment-v1.1",
--   "contributing_factors": ["repeated_contact", "unresolved_prior_case", "negative_language"]
-- }
```

## Event Type Catalogue

```sql
-- Registry of all event types with their JSON Schema definitions
CREATE TABLE event_type_registry (
    event_type      VARCHAR(100) PRIMARY KEY,
    stream_type     VARCHAR(50) NOT NULL,
    description     TEXT NOT NULL,
    schema_version  INTEGER NOT NULL DEFAULT 1,
    json_schema     JSONB NOT NULL,                  -- JSON Schema draft 2020-12 definition
    is_deprecated   BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Example event types:
-- Constituent lifecycle: ConstituentCreated, ConstituentUpdated, ConstituentMerged,
--     ConstituentDeactivated, ChannelAdded, ChannelRemoved, ChannelVerified,
--     RelationshipCreated, RelationshipEnded, TagAdded, TagRemoved
-- Case lifecycle: CaseCreated, CaseRouted, CaseAssigned, CaseReassigned,
--     CaseStatusChanged, CaseEscalated, CasePriorityChanged, CaseResolved,
--     CaseClosed, CaseReopened, SlaBreached, SlaBreachPredicted
-- Service Request: ServiceRequestSubmitted, ServiceRequestLinkedToCase,
--     Open311TokenIssued, Open311StatusUpdated
-- Interaction: InboundMessageReceived, OutboundMessageSent, MessageDelivered,
--     MessageBounced, PhoneCallLogged, NoteAdded, AttachmentUploaded
-- AI: AiClassificationPerformed, AiRoutingSuggested, AiDraftGenerated,
--     AiDraftAccepted, AiDraftRejected, SentimentScored, DeduplicationDetected
-- Campaign: CampaignCreated, CampaignScheduled, CampaignSent, CampaignCompleted
```

## Read Projections (Query Side)

```sql
-- Materialised projection: current constituent state
CREATE TABLE proj_constituent (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    constituent_type VARCHAR(20) NOT NULL,
    first_name      VARCHAR(100),
    last_name       VARCHAR(100),
    organisation_name VARCHAR(255),
    display_name    VARCHAR(255) NOT NULL,
    preferred_language VARCHAR(10),
    preferred_channel VARCHAR(20),
    external_id     VARCHAR(255),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    total_cases     INTEGER NOT NULL DEFAULT 0,
    open_cases      INTEGER NOT NULL DEFAULT 0,
    last_interaction_at TIMESTAMPTZ,
    sentiment_trend VARCHAR(20),                     -- improving, stable, declining
    -- Projection metadata
    last_event_sequence BIGINT NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_proj_const_tenant ON proj_constituent(tenant_id);
CREATE INDEX idx_proj_const_name ON proj_constituent(tenant_id, last_name, first_name);
CREATE INDEX idx_proj_const_display ON proj_constituent(tenant_id, display_name);

-- Materialised projection: current case state
CREATE TABLE proj_case (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    case_number     VARCHAR(50) NOT NULL,
    case_type       VARCHAR(100) NOT NULL,
    status          VARCHAR(30) NOT NULL,
    priority        VARCHAR(20) NOT NULL,
    subject         VARCHAR(500) NOT NULL,
    description     TEXT,
    department_name VARCHAR(255),
    department_id   UUID,
    assigned_to_id  UUID,
    assigned_to_name VARCHAR(255),
    requester_id    UUID,
    requester_name  VARCHAR(255),
    sla_due_at      TIMESTAMPTZ,
    sla_breached    BOOLEAN NOT NULL DEFAULT false,
    ai_category     VARCHAR(100),
    ai_priority_score DECIMAL(5, 4),
    ai_sentiment    VARCHAR(20),
    latitude        DECIMAL(10, 7),
    longitude       DECIMAL(10, 7),
    address_string  VARCHAR(500),
    source_channel  VARCHAR(30),
    -- Open311 fields
    service_code    VARCHAR(50),
    service_name    VARCHAR(255),
    open311_id      VARCHAR(100),
    media_url       TEXT,
    -- Timeline
    created_at      TIMESTAMPTZ NOT NULL,
    resolved_at     TIMESTAMPTZ,
    closed_at       TIMESTAMPTZ,
    -- Derived analytics
    total_interactions INTEGER NOT NULL DEFAULT 0,
    time_to_first_response_minutes INTEGER,
    -- Projection metadata
    last_event_sequence BIGINT NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_proj_case_tenant_status ON proj_case(tenant_id, status);
CREATE INDEX idx_proj_case_number ON proj_case(tenant_id, case_number);
CREATE INDEX idx_proj_case_sla ON proj_case(tenant_id, sla_due_at) WHERE NOT sla_breached AND status NOT IN ('resolved', 'closed');
CREATE INDEX idx_proj_case_assigned ON proj_case(assigned_to_id) WHERE assigned_to_id IS NOT NULL;

-- Materialised projection: constituent interaction timeline
CREATE TABLE proj_interaction_timeline (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    constituent_id  UUID NOT NULL,
    case_id         UUID,
    event_type      VARCHAR(100) NOT NULL,
    summary         TEXT NOT NULL,                    -- human-readable summary of event
    direction       VARCHAR(10),
    channel         VARCHAR(30),
    performed_by_name VARCHAR(255),
    ai_generated    BOOLEAN NOT NULL DEFAULT false,
    occurred_at     TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_proj_timeline_constituent ON proj_interaction_timeline(constituent_id, occurred_at DESC);
CREATE INDEX idx_proj_timeline_case ON proj_interaction_timeline(case_id, occurred_at DESC) WHERE case_id IS NOT NULL;

-- Materialised projection: department dashboard metrics
CREATE TABLE proj_department_metrics (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    department_id   UUID NOT NULL,
    metric_date     DATE NOT NULL,
    open_cases      INTEGER NOT NULL DEFAULT 0,
    new_cases       INTEGER NOT NULL DEFAULT 0,
    resolved_cases  INTEGER NOT NULL DEFAULT 0,
    escalated_cases INTEGER NOT NULL DEFAULT 0,
    sla_breaches    INTEGER NOT NULL DEFAULT 0,
    avg_resolution_hours DECIMAL(10, 2),
    avg_first_response_hours DECIMAL(10, 2),
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, department_id, metric_date)
);

-- Materialised projection: AI model performance tracking
CREATE TABLE proj_ai_model_metrics (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    model_name      VARCHAR(100) NOT NULL,
    model_version   VARCHAR(50) NOT NULL,
    metric_date     DATE NOT NULL,
    predictions_made INTEGER NOT NULL DEFAULT 0,
    predictions_accepted INTEGER NOT NULL DEFAULT 0,
    predictions_overridden INTEGER NOT NULL DEFAULT 0,
    avg_confidence  DECIMAL(5, 4),
    accuracy_rate   DECIMAL(5, 4),
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, model_name, model_version, metric_date)
);
```

## Snapshot Store (Performance Optimisation)

```sql
-- Periodic snapshots to avoid full event replay for long-lived aggregates
CREATE TABLE event_snapshot (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_type     VARCHAR(50) NOT NULL,
    stream_id       UUID NOT NULL,
    sequence_number BIGINT NOT NULL,                 -- snapshot taken at this sequence
    state           JSONB NOT NULL,                  -- full aggregate state at this point
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_snapshot_stream ON event_snapshot(stream_id, sequence_number DESC);

-- To reconstruct current state:
-- 1. Load latest snapshot for stream_id
-- 2. Replay events with sequence_number > snapshot.sequence_number
-- 3. Apply each event to snapshot state
```

## Reference Data (Shared Across Read/Write Sides)

```sql
CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    settings        JSONB NOT NULL DEFAULT '{}',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE jurisdiction (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    country_code    CHAR(2) NOT NULL,
    subdivision_code VARCHAR(6),
    name            VARCHAR(255) NOT NULL,
    level           VARCHAR(50) NOT NULL,
    parent_id       UUID REFERENCES jurisdiction(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE department (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    parent_id       UUID REFERENCES department(id),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE staff_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    email           VARCHAR(255) NOT NULL,
    display_name    VARCHAR(255) NOT NULL,
    department_id   UUID REFERENCES department(id),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE service_catalogue (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    service_code    VARCHAR(50) NOT NULL,
    service_name    VARCHAR(255) NOT NULL,
    description     TEXT,
    group_name      VARCHAR(100),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Example Queries

### Reconstruct case state at a specific point in time

```sql
-- "What was case SR-2026-00472 status on May 18, 2026 at 3:00 PM?"
SELECT payload
FROM event_store
WHERE stream_id = '550e8400-e29b-41d4-a716-446655440000'
  AND stream_type = 'case'
  AND occurred_at <= '2026-05-18T15:00:00Z'
ORDER BY sequence_number ASC;
-- Application code replays these events to reconstruct state
```

### Audit trail for a specific constituent

```sql
-- "Show all changes to constituent Maria Santos in the last 30 days"
SELECT event_type, payload, metadata, occurred_at
FROM event_store
WHERE stream_id = 'constituent-maria-uuid'
  AND stream_type = 'constituent'
  AND occurred_at >= now() - INTERVAL '30 days'
ORDER BY sequence_number ASC;
```

### AI routing decision audit

```sql
-- "Show all AI routing decisions and whether they were overridden"
SELECT
    r.payload->>'ai_model_version' AS model,
    r.payload->>'ai_confidence' AS confidence,
    r.payload->>'ai_explanation' AS explanation,
    r.occurred_at AS routed_at,
    o.occurred_at AS overridden_at,
    o.payload->>'reason' AS override_reason
FROM event_store r
LEFT JOIN event_store o
    ON o.stream_id = r.stream_id
    AND o.event_type = 'CaseReassigned'
    AND o.sequence_number > r.sequence_number
    AND o.sequence_number = (
        SELECT MIN(sequence_number)
        FROM event_store
        WHERE stream_id = r.stream_id
          AND event_type = 'CaseReassigned'
          AND sequence_number > r.sequence_number
    )
WHERE r.tenant_id = 'tenant-uuid'
  AND r.event_type = 'CaseRouted'
  AND r.occurred_at >= now() - INTERVAL '7 days'
ORDER BY r.occurred_at DESC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 3 | event_store, event_type_registry, event_snapshot |
| Read Projections | 5 | proj_constituent, proj_case, proj_interaction_timeline, proj_department_metrics, proj_ai_model_metrics |
| Reference Data | 5 | tenant, jurisdiction, department, staff_user, service_catalogue |
| **Total** | **13** | Much fewer tables than normalised model; complexity is in event processing logic |

---

## Key Design Decisions

1. **Event store as the single source of truth** — no mutable state tables. The `event_store` table is append-only; projections are derived and can be rebuilt from events at any time. This eliminates the "audit log drift" problem where a separate audit table diverges from the operational data.

2. **Event hashing for tamper detection** — each event includes a `event_hash` (SHA-256 of payload concatenated with the previous event's hash), creating a blockchain-like chain within each stream. This satisfies FedRAMP continuous monitoring requirements and makes tampering detectable.

3. **Rich event metadata** — every event captures `actor_id`, `actor_type`, `ip_address`, `correlation_id` (linking related events across streams), and `causation_id` (which event caused this one). This enables full request tracing across the system.

4. **AI decisions are first-class events** — routing decisions, sentiment scores, draft generation, and draft acceptance/rejection are all captured as events with model version, confidence, and explanation. This enables AI model performance tracking, bias detection, and the explainable routing requirement.

5. **Snapshot store for performance** — long-lived constituent streams (years of interactions) would be expensive to replay from scratch. Periodic snapshots (e.g., every 100 events) bound the replay cost.

6. **Event type registry with JSON Schema validation** — each event type has a registered JSON Schema definition. Events are validated against their schema on write, ensuring data quality in the append-only store. Schema versioning with upcasting enables evolution.

7. **Projections are disposable** — all `proj_*` tables can be dropped and rebuilt from the event store. This means projection bugs can be fixed retroactively, and new query patterns can be added by creating new projections over historical events.

8. **Eventual consistency is explicit** — read projections include `last_event_sequence` and `projected_at` metadata so the API can report staleness. For critical use cases (e.g., SLA breach checks), the system can read directly from the event store.

9. **Time-range partitioning on the event store** — partitioning by month keeps the event store manageable as it grows. Old partitions can be archived to cold storage while remaining queryable.

10. **Separation of `occurred_at` and `recorded_at`** — `occurred_at` is when the event actually happened (e.g., when the constituent called); `recorded_at` is when it was persisted. This distinction is critical for accurate temporal queries and handles late-arriving events correctly.
