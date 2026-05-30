# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Constituent Relationship Management · Created: 2026-05-22

## Philosophy

This model uses a smaller number of relational tables for core entities (constituents, cases, staff, departments) with strongly-typed columns for universal fields, while accommodating variable, jurisdiction-specific, and extension fields in JSONB columns. The result is a schema that can serve a municipality in rural Texas and a state agency in California from the same codebase without requiring schema migrations for each deployment.

Government CRM is inherently multi-jurisdictional. A pothole service request in Portland has different attributes than a benefits enquiry in Miami. Elected-office casework in a congressional district tracks different metadata than agency-level 311 operations. A fully normalised schema either becomes excessively wide (hundreds of nullable columns) or requires a complex EAV (Entity-Attribute-Value) pattern. JSONB offers a middle ground: core fields are typed and indexed for fast querying; variable fields live in a validated JSONB column with GIN indexes for containment queries.

This pattern is widely used in modern SaaS platforms (Shopify's metafields, Stripe's metadata, Notion's property system) and is well-supported by PostgreSQL's JSONB operators, GIN indexes, and JSON Schema validation via check constraints or application-level validation.

**Best for:** Multi-jurisdiction deployments where each agency needs custom fields, rapid MVP development, and scenarios where the full entity model is not yet known and will evolve through customer feedback.

**Trade-offs:**
- (+) Fewer tables and migrations: jurisdiction-specific fields do not require ALTER TABLE
- (+) Fast time-to-market: new field types can be added without database changes
- (+) JSONB columns are indexable (GIN) and queryable with PostgreSQL operators
- (+) Natural fit for API extensibility: custom fields are first-class in the JSON response
- (+) Reduces the "100 nullable columns" problem of wide normalised tables
- (-) JSONB fields lack database-level foreign key constraints; referential integrity for custom fields is application-enforced
- (-) Complex queries across JSONB fields are slower than typed column queries
- (-) Schema validation for JSONB content requires application-level logic or CHECK constraints
- (-) Reporting tools often struggle with JSONB columns; may need to extract to typed columns for BI
- (-) Risk of unstructured data drift if JSONB validation is not enforced rigorously

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| Open311 GeoReport v2 | Service request core fields are typed columns; service-specific attributes stored in `attributes` JSONB column matching Open311's extensible attribute model |
| NIEM Human Services Domain | Core NIEM-aligned fields (person name, address, case status) are typed columns; domain-specific NIEM extensions stored in `niem_extensions` JSONB |
| ISO 3166-1/2 | Jurisdiction codes as typed columns with foreign key to jurisdiction reference table |
| JSON Schema Draft 2020-12 | Every JSONB column has a registered JSON Schema definition in `field_schema_registry` for runtime validation |
| OpenAPI 3.1 | API schema generation includes JSONB field definitions from the schema registry, enabling self-documenting APIs |
| NIST 800-53 | Audit trail captures changes to both typed columns and JSONB fields |

---

## Field Schema Registry

```sql
-- Registry of custom/extension field schemas per entity type per tenant
-- This is the "schema for the JSONB columns" — ensures data quality
CREATE TABLE field_schema_registry (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    entity_type     VARCHAR(50) NOT NULL,           -- constituent, case, service_request, etc.
    field_group     VARCHAR(100) NOT NULL,           -- e.g., 'jurisdiction_fields', 'intake_form', 'agency_custom'
    json_schema     JSONB NOT NULL,                  -- JSON Schema draft 2020-12 definition
    ui_schema       JSONB,                           -- UI rendering hints (field order, widgets, sections)
    version         INTEGER NOT NULL DEFAULT 1,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, entity_type, field_group)
);

-- Example JSON Schema for a municipality's custom constituent fields:
-- {
--   "type": "object",
--   "properties": {
--     "ward_number": {"type": "integer", "minimum": 1, "maximum": 50},
--     "voter_registration_id": {"type": "string", "pattern": "^VR-[0-9]{8}$"},
--     "property_owner": {"type": "boolean"},
--     "accessibility_needs": {
--       "type": "array",
--       "items": {"type": "string", "enum": ["wheelchair", "visual", "hearing", "cognitive", "language"]}
--     },
--     "preferred_contact_times": {
--       "type": "array",
--       "items": {"type": "string", "enum": ["morning", "afternoon", "evening"]}
--     }
--   }
-- }
```

## Core Identity & Multi-Tenancy

```sql
CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    jurisdiction_id UUID REFERENCES jurisdiction(id),
    tier            VARCHAR(50) NOT NULL DEFAULT 'standard',
    config          JSONB NOT NULL DEFAULT '{}',
    -- config example:
    -- {
    --   "features": {"gis_routing": true, "ai_classification": true, "open311": true},
    --   "branding": {"primary_color": "#003366", "logo_url": "..."},
    --   "sla_defaults": {"high": 4, "normal": 24, "low": 72},
    --   "channels": ["web", "email", "phone", "sms", "mobile_app"]
    -- }
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
    timezone        VARCHAR(50),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Constituent Management

```sql
CREATE TABLE constituent (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    constituent_type VARCHAR(20) NOT NULL CHECK (constituent_type IN ('individual', 'organisation', 'household')),
    -- Universal typed fields (present for all deployments)
    display_name    VARCHAR(255) NOT NULL,
    first_name      VARCHAR(100),
    last_name       VARCHAR(100),
    organisation_name VARCHAR(255),
    preferred_language VARCHAR(10) DEFAULT 'en',
    preferred_channel VARCHAR(20) DEFAULT 'email',
    external_id     VARCHAR(255),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    -- Contact channels embedded as JSONB array (eliminates separate channel table)
    channels        JSONB NOT NULL DEFAULT '[]',
    -- channels example:
    -- [
    --   {"type": "email", "value": "maria@example.com", "is_primary": true, "verified": true},
    --   {"type": "phone", "value": "+1-555-0147", "phone_type": "mobile", "is_primary": false},
    --   {"type": "address", "street": "123 Main St", "city": "Portland", "state": "OR",
    --    "postal": "97201", "country": "US", "lat": 45.5152, "lng": -122.6784}
    -- ]
    -- Relationships embedded as JSONB array (lightweight alternative to junction table)
    relationships   JSONB NOT NULL DEFAULT '[]',
    -- relationships example:
    -- [
    --   {"constituent_id": "uuid", "type": "household_member", "label": "spouse", "since": "2020-01-15"},
    --   {"constituent_id": "uuid", "type": "elected_representative", "label": "council member"}
    -- ]
    -- Tags as simple array
    tags            JSONB NOT NULL DEFAULT '[]',
    -- tags example: ["ward-7", "property-owner", "spanish-speaker", "senior"]
    -- Jurisdiction-specific custom fields (validated against field_schema_registry)
    custom_fields   JSONB NOT NULL DEFAULT '{}',
    -- custom_fields example (varies per tenant):
    -- {
    --   "ward_number": 7,
    --   "voter_registration_id": "VR-20240319",
    --   "property_owner": true,
    --   "accessibility_needs": ["visual", "language"]
    -- }
    -- AI-derived metadata
    ai_metadata     JSONB NOT NULL DEFAULT '{}',
    -- ai_metadata example:
    -- {
    --   "sentiment_trend": "declining",
    --   "engagement_score": 0.72,
    --   "predicted_needs": ["benefit_renewal", "property_tax_exemption"],
    --   "dedup_candidates": ["uuid-1", "uuid-2"]
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_constituent_tenant ON constituent(tenant_id);
CREATE INDEX idx_constituent_name ON constituent(tenant_id, last_name, first_name);
CREATE INDEX idx_constituent_display ON constituent(tenant_id, display_name);
CREATE INDEX idx_constituent_external ON constituent(tenant_id, external_id);
-- GIN index for JSONB containment queries on channels
CREATE INDEX idx_constituent_channels ON constituent USING GIN(channels jsonb_path_ops);
-- GIN index for tag-based queries
CREATE INDEX idx_constituent_tags ON constituent USING GIN(tags jsonb_path_ops);
-- GIN index for custom field queries
CREATE INDEX idx_constituent_custom ON constituent USING GIN(custom_fields jsonb_path_ops);
```

### JSONB Query Examples for Constituents

```sql
-- Find all constituents with a specific email
SELECT * FROM constituent
WHERE tenant_id = 'tenant-uuid'
  AND channels @> '[{"type": "email", "value": "maria@example.com"}]';

-- Find all constituents tagged as "spanish-speaker" in ward 7
SELECT * FROM constituent
WHERE tenant_id = 'tenant-uuid'
  AND tags @> '["spanish-speaker"]'
  AND custom_fields @> '{"ward_number": 7}';

-- Find constituents with accessibility needs
SELECT * FROM constituent
WHERE tenant_id = 'tenant-uuid'
  AND custom_fields ? 'accessibility_needs'
  AND jsonb_array_length(custom_fields->'accessibility_needs') > 0;

-- Find constituents with declining sentiment
SELECT * FROM constituent
WHERE tenant_id = 'tenant-uuid'
  AND ai_metadata @> '{"sentiment_trend": "declining"}';
```

## Case Management

```sql
CREATE TABLE constituent_case (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    case_number     VARCHAR(50) NOT NULL,
    case_type       VARCHAR(50) NOT NULL,             -- service_request, complaint, enquiry, casework, referral
    status          VARCHAR(30) NOT NULL DEFAULT 'open',
    priority        VARCHAR(20) NOT NULL DEFAULT 'normal',
    subject         VARCHAR(500) NOT NULL,
    description     TEXT,
    -- Assignment (typed for fast queries)
    department_id   UUID REFERENCES department(id),
    assigned_to     UUID REFERENCES staff_user(id),
    -- SLA (typed for indexed queries)
    sla_due_at      TIMESTAMPTZ,
    sla_breached    BOOLEAN NOT NULL DEFAULT false,
    -- GIS (typed for PostGIS queries)
    latitude        DECIMAL(10, 7),
    longitude       DECIMAL(10, 7),
    address_string  VARCHAR(500),
    -- Source
    source_channel  VARCHAR(30),
    -- Participants embedded as JSONB (avoids junction table for common pattern)
    participants    JSONB NOT NULL DEFAULT '[]',
    -- participants example:
    -- [
    --   {"constituent_id": "uuid", "role": "requester", "is_primary": true},
    --   {"constituent_id": "uuid", "role": "subject"},
    --   {"constituent_id": "uuid", "role": "witness"}
    -- ]
    -- Open311 fields (only populated for service requests)
    open311         JSONB,
    -- open311 example:
    -- {
    --   "service_code": "POTHOLE",
    --   "service_name": "Pothole Repair",
    --   "service_request_id": "SR-2026-00472",
    --   "token": "abc123",
    --   "media_url": "https://...",
    --   "attributes": {"pothole_size": "large", "lane_affected": "right"}
    -- }
    -- AI classification and routing
    ai_classification JSONB NOT NULL DEFAULT '{}',
    -- ai_classification example:
    -- {
    --   "category": "road_maintenance",
    --   "priority_score": 0.9234,
    --   "sentiment": "negative",
    --   "routing_method": "ai_gis",
    --   "model_version": "route-classifier-v2.3",
    --   "confidence": 0.9234,
    --   "explanation": "Classified as road_maintenance (93.2%). GIS match: Ward 7."
    -- }
    -- Resolution
    resolution      JSONB,
    -- resolution example:
    -- {
    --   "notes": "Pothole repaired on 2026-05-20",
    --   "resolved_by": "staff-uuid",
    --   "resolved_at": "2026-05-20T14:30:00Z",
    --   "satisfaction_rating": 4,
    --   "satisfaction_comment": "Fixed quickly, thank you"
    -- }
    -- Jurisdiction-specific custom fields
    custom_fields   JSONB NOT NULL DEFAULT '{}',
    -- Metadata
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE UNIQUE INDEX idx_case_number ON constituent_case(tenant_id, case_number);
CREATE INDEX idx_case_status ON constituent_case(tenant_id, status);
CREATE INDEX idx_case_type ON constituent_case(tenant_id, case_type);
CREATE INDEX idx_case_assigned ON constituent_case(assigned_to) WHERE assigned_to IS NOT NULL;
CREATE INDEX idx_case_department ON constituent_case(department_id);
CREATE INDEX idx_case_sla ON constituent_case(tenant_id, sla_due_at) WHERE NOT sla_breached AND status NOT IN ('resolved', 'closed');
CREATE INDEX idx_case_created ON constituent_case(tenant_id, created_at);
CREATE INDEX idx_case_participants ON constituent_case USING GIN(participants jsonb_path_ops);
CREATE INDEX idx_case_open311 ON constituent_case USING GIN(open311 jsonb_path_ops);
CREATE INDEX idx_case_ai ON constituent_case USING GIN(ai_classification jsonb_path_ops);
```

## Activity / Interaction Log

```sql
CREATE TABLE activity (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    case_id         UUID REFERENCES constituent_case(id),
    constituent_id  UUID REFERENCES constituent(id),
    activity_type   VARCHAR(50) NOT NULL,             -- inbound_email, outbound_email, phone_call, note, status_change, ai_draft, etc.
    direction       VARCHAR(10),
    channel         VARCHAR(30),
    subject         VARCHAR(500),
    body            TEXT,
    -- Participants and metadata as JSONB
    participants    JSONB NOT NULL DEFAULT '[]',
    -- participants example:
    -- [{"constituent_id": "uuid", "role": "sender"}, {"staff_id": "uuid", "role": "recipient"}]
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- metadata example (varies by activity_type):
    -- For ai_draft: {"model": "gpt-4", "confidence": 0.88, "accepted": true, "edit_distance": 12}
    -- For email: {"message_id": "<...>", "in_reply_to": "<...>", "attachments": 2}
    -- For phone_call: {"duration_seconds": 342, "recording_url": "..."}
    performed_by    UUID REFERENCES staff_user(id),
    performed_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_activity_case ON activity(case_id) WHERE case_id IS NOT NULL;
CREATE INDEX idx_activity_constituent ON activity(constituent_id) WHERE constituent_id IS NOT NULL;
CREATE INDEX idx_activity_tenant_time ON activity(tenant_id, performed_at);
CREATE INDEX idx_activity_type ON activity(tenant_id, activity_type);
```

## Department & Staff

```sql
CREATE TABLE department (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    parent_id       UUID REFERENCES department(id),
    config          JSONB NOT NULL DEFAULT '{}',
    -- config example:
    -- {
    --   "sla_overrides": {"pothole": 48, "noise_complaint": 72},
    --   "routing_rules": [{"service_code": "POTHOLE", "assign_to": "staff-uuid"}],
    --   "working_hours": {"start": "08:00", "end": "17:00", "timezone": "America/New_York"}
    -- }
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE staff_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    email           VARCHAR(255) NOT NULL,
    display_name    VARCHAR(255) NOT NULL,
    department_id   UUID REFERENCES department(id),
    roles           JSONB NOT NULL DEFAULT '[]',
    -- roles example: ["case_worker", "supervisor", "admin"]
    permissions     JSONB NOT NULL DEFAULT '[]',
    -- permissions example: ["cases:read", "cases:write", "constituents:read", "reports:export"]
    preferences     JSONB NOT NULL DEFAULT '{}',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE UNIQUE INDEX idx_staff_email ON staff_user(tenant_id, email);

-- GIS boundaries for routing
CREATE TABLE geographic_boundary (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    boundary_type   VARCHAR(50) NOT NULL,
    department_id   UUID REFERENCES department(id),
    assigned_to     UUID REFERENCES staff_user(id),
    geom            GEOMETRY(MultiPolygon, 4326),
    metadata        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_boundary_geom ON geographic_boundary USING GIST(geom);
```

## Communications

```sql
CREATE TABLE campaign (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    campaign_type   VARCHAR(50) NOT NULL,
    status          VARCHAR(30) NOT NULL DEFAULT 'draft',
    -- Target audience as JSONB query definition
    audience_filter JSONB NOT NULL DEFAULT '{}',
    -- audience_filter example:
    -- {
    --   "tags": ["ward-7"],
    --   "custom_fields": {"property_owner": true},
    --   "language": "es"
    -- }
    content         JSONB NOT NULL DEFAULT '{}',
    -- content example:
    -- {
    --   "subject": "Ward 7 Town Hall Meeting",
    --   "body_html": "<h1>You're invited...</h1>",
    --   "body_text": "You're invited...",
    --   "sms_text": "Town Hall Meeting May 25 at City Hall. Details: ..."
    -- }
    schedule        JSONB,
    metrics         JSONB NOT NULL DEFAULT '{}',
    -- metrics example (updated as campaign runs):
    -- {"sent": 1243, "delivered": 1198, "opened": 456, "clicked": 89, "bounced": 45}
    created_by      UUID REFERENCES staff_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE message_template (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    channel         VARCHAR(20) NOT NULL,
    content         JSONB NOT NULL,
    -- content example:
    -- {
    --   "subject": "Re: Your service request {{case_number}}",
    --   "body_html": "<p>Dear {{constituent_name}}, ...</p>",
    --   "body_text": "Dear {{constituent_name}}, ...",
    --   "variables": ["constituent_name", "case_number", "department_name", "status"]
    -- }
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Attachments & Audit

```sql
CREATE TABLE attachment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    entity_type     VARCHAR(50) NOT NULL,
    entity_id       UUID NOT NULL,
    filename        VARCHAR(500) NOT NULL,
    content_type    VARCHAR(100) NOT NULL,
    size_bytes      BIGINT NOT NULL,
    storage_key     VARCHAR(500) NOT NULL,
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- metadata example: {"scanned_for_pii": true, "ocr_text": "...", "thumbnail_key": "..."}
    uploaded_by     UUID REFERENCES staff_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_attachment_entity ON attachment(entity_type, entity_id);

-- Audit log
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    actor_id        UUID,
    actor_type      VARCHAR(20) NOT NULL,
    action          VARCHAR(50) NOT NULL,
    entity_type     VARCHAR(50) NOT NULL,
    entity_id       UUID,
    changes         JSONB,
    -- changes example:
    -- {
    --   "typed_fields": {"status": {"old": "open", "new": "in_progress"}},
    --   "jsonb_fields": {
    --     "custom_fields.ward_number": {"old": null, "new": 7},
    --     "ai_classification.category": {"old": null, "new": "road_maintenance"}
    --   }
    -- }
    request_metadata JSONB,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_tenant_time ON audit_log(tenant_id, created_at);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Schema Management | 1 | field_schema_registry |
| Core Identity | 2 | tenant, jurisdiction |
| Constituent Management | 1 | constituent (channels, relationships, tags in JSONB) |
| Case Management | 1 | constituent_case (participants, Open311, AI metadata in JSONB) |
| Activity | 1 | activity |
| Department & Staff | 3 | department, staff_user, geographic_boundary |
| Communications | 2 | campaign, message_template |
| Attachments | 1 | attachment |
| Audit | 1 | audit_log |
| **Total** | **13** | JSONB eliminates ~15 junction/lookup tables compared to normalised model |

---

## Key Design Decisions

1. **Channels embedded in constituent JSONB** — rather than a separate `constituent_channel` table with multiple rows per constituent, contact channels are stored as a JSONB array on the constituent record. This eliminates a join for the most common query pattern (display constituent with all their contact info) while remaining queryable via GIN indexes.

2. **Participants embedded in case JSONB** — case participants are stored as a JSONB array rather than a junction table. For the typical case (1-3 participants), this avoids an extra join. For complex queries across participants, the GIN index supports containment queries.

3. **Field schema registry for JSONB validation** — every JSONB `custom_fields` column has a corresponding JSON Schema definition in `field_schema_registry`. The application layer validates incoming data against the schema before persisting, providing the data quality guarantees that typed columns normally provide.

4. **Open311 fields as a JSONB object on case** — rather than a separate `service_request` table, Open311-specific fields are stored as a JSONB object on the case. This avoids a 1:1 join for service request cases while keeping the case table universal.

5. **AI classification as JSONB** — ML model outputs (category, confidence, explanation, model version) are stored as a JSONB object on the case. This accommodates evolving AI models that may produce different output fields over time without requiring schema migrations.

6. **Roles and permissions as JSONB on staff_user** — rather than separate `role`, `staff_role`, and `permission` tables, roles and permissions are stored as JSONB arrays. This is simpler for the common case where a staff member has 1-3 roles, and the permission check is an application-level `@>` containment query.

7. **Campaign audience filter as JSONB** — audience targeting criteria are stored as a structured JSONB query definition that can be translated to SQL at campaign send time. This is more flexible than separate filter tables and mirrors how modern marketing platforms define audiences.

8. **Audit log captures JSONB field changes** — the `changes` column in the audit log distinguishes between typed column changes and JSONB field path changes, ensuring full traceability even for nested custom field modifications.

9. **Trade-off: fewer tables, more application logic** — this model shifts validation, referential integrity, and relationship management from the database layer to the application layer. This is the explicit trade-off: deployment speed and flexibility at the cost of database-enforced constraints.

10. **GIN indexes on all JSONB columns** — every JSONB column that will be queried has a `jsonb_path_ops` GIN index. The `jsonb_path_ops` variant is smaller and faster than the default GIN operator class for containment queries (`@>`), which is the primary query pattern for JSONB fields.
