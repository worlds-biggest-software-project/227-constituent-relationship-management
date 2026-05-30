# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Constituent Relationship Management · Created: 2026-05-22

## Philosophy

This model follows the classic normalized relational approach: every domain concept gets its own table, relationships are expressed through foreign keys and junction tables, and reference data is separated into lookup tables aligned with industry standards. The schema is designed for maximum data integrity and query flexibility across the full constituent lifecycle.

The approach mirrors the patterns used by Salesforce Public Sector Solutions (Person Accounts, Cases, Participants, Complaints, Referrals, Benefits) and CiviCRM (200+ tables with junction tables for many-to-many relationships), but expressed in PostgreSQL with UUID primary keys and modern conventions. Every field that maps to a standard (ISO 3166 for jurisdictions, Open311 GeoReport v2 for service requests, NIEM for inter-agency exchange) is annotated accordingly.

This is the most conservative and well-understood architecture. It works well with existing ORMs, reporting tools, and data warehouses. It trades some write-time flexibility for maximum read-time query power and referential integrity.

**Best for:** Agencies that need strong referential integrity, complex cross-entity reporting, and standards-compliant data exchange with other government systems.

**Trade-offs:**
- (+) Maximum data integrity via foreign keys and constraints
- (+) Excellent for complex cross-entity queries and ad hoc reporting
- (+) Clear alignment with NIEM, Open311, and Salesforce PSS data models
- (+) Well-understood by database administrators and ORMs
- (-) High table count increases schema complexity and migration burden
- (-) Adding jurisdiction-specific fields requires schema changes (ALTER TABLE)
- (-) Many-to-many junction tables add write overhead and join complexity
- (-) Schema evolution is slow; every new concept requires a migration

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| Open311 GeoReport v2 | `service_request` table fields map directly to GeoReport v2 schema (service_code, status, requested_datetime, expected_datetime, address_string, lat/long, media_url) |
| NIEM Human Services Domain | `constituent`, `case`, `case_participant`, and `referral` entities align with NIEM person, case, and referral exchange elements |
| ISO 3166-1/2 | `jurisdiction` table uses ISO 3166 alpha-2 country codes and subdivision codes for state/province |
| ISO 8601 | All datetime fields use TIMESTAMPTZ with ISO 8601 formatting |
| WCAG 2.1 AA | Constituent portal accessibility metadata stored in `portal_config` |
| OAuth 2.0 / OpenID Connect | `oauth_client` and `user_session` tables support iGov Profile authentication |
| FedRAMP / NIST 800-53 | `audit_log` table captures all data access and modification events per NIST AU controls |
| OData v4 | API layer exposes entities following OData conventions for filtering, pagination, and relationship traversal |

---

## Core Identity & Multi-Tenancy

```sql
-- Each deploying agency/organisation is a tenant
CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    jurisdiction_id UUID REFERENCES jurisdiction(id),
    tier            VARCHAR(50) NOT NULL DEFAULT 'standard',  -- standard, enterprise, federal
    settings        JSONB NOT NULL DEFAULT '{}',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_tenant_slug ON tenant(slug);

-- ISO 3166-aligned jurisdiction reference data
CREATE TABLE jurisdiction (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    country_code    CHAR(2) NOT NULL,          -- ISO 3166-1 alpha-2
    subdivision_code VARCHAR(6),               -- ISO 3166-2 (e.g., US-CA)
    name            VARCHAR(255) NOT NULL,
    level           VARCHAR(50) NOT NULL,       -- federal, state, county, municipality, district
    parent_id       UUID REFERENCES jurisdiction(id),
    timezone        VARCHAR(50),                -- IANA timezone
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_jurisdiction_country ON jurisdiction(country_code);
CREATE INDEX idx_jurisdiction_parent ON jurisdiction(parent_id);
```

## Constituent Management

```sql
-- Core constituent record (person or organisation)
CREATE TABLE constituent (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    constituent_type VARCHAR(20) NOT NULL CHECK (constituent_type IN ('individual', 'organisation', 'household')),
    -- Individual fields
    first_name      VARCHAR(100),
    middle_name     VARCHAR(100),
    last_name       VARCHAR(100),
    prefix          VARCHAR(20),
    suffix          VARCHAR(20),
    date_of_birth   DATE,
    gender          VARCHAR(20),
    -- Organisation fields
    organisation_name VARCHAR(255),
    -- Common fields
    display_name    VARCHAR(255) NOT NULL,
    preferred_language VARCHAR(10) DEFAULT 'en',  -- BCP 47 language tag
    preferred_channel VARCHAR(20) DEFAULT 'email', -- email, sms, phone, mail
    external_id     VARCHAR(255),                  -- ID from external system (e.g., login.gov sub)
    is_deceased     BOOLEAN NOT NULL DEFAULT false,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_constituent_tenant ON constituent(tenant_id);
CREATE INDEX idx_constituent_name ON constituent(tenant_id, last_name, first_name);
CREATE INDEX idx_constituent_display ON constituent(tenant_id, display_name);
CREATE INDEX idx_constituent_external ON constituent(tenant_id, external_id);
CREATE INDEX idx_constituent_type ON constituent(tenant_id, constituent_type);

-- Constituent contact channels (email, phone, address)
CREATE TABLE constituent_channel (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    constituent_id  UUID NOT NULL REFERENCES constituent(id) ON DELETE CASCADE,
    channel_type    VARCHAR(20) NOT NULL CHECK (channel_type IN ('email', 'phone', 'address', 'social')),
    is_primary      BOOLEAN NOT NULL DEFAULT false,
    -- Email
    email           VARCHAR(255),
    -- Phone
    phone_number    VARCHAR(30),
    phone_type      VARCHAR(20),  -- mobile, home, work, fax
    -- Address (NIEM-aligned)
    street_line_1   VARCHAR(255),
    street_line_2   VARCHAR(255),
    city            VARCHAR(100),
    state_code      VARCHAR(10),     -- ISO 3166-2 subdivision
    postal_code     VARCHAR(20),
    country_code    CHAR(2),         -- ISO 3166-1 alpha-2
    latitude        DECIMAL(10, 7),
    longitude       DECIMAL(10, 7),
    -- Social
    social_platform VARCHAR(50),
    social_handle   VARCHAR(255),
    -- Metadata
    is_verified     BOOLEAN NOT NULL DEFAULT false,
    verified_at     TIMESTAMPTZ,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_channel_constituent ON constituent_channel(constituent_id);
CREATE INDEX idx_channel_email ON constituent_channel(email) WHERE email IS NOT NULL;
CREATE INDEX idx_channel_phone ON constituent_channel(phone_number) WHERE phone_number IS NOT NULL;

-- Relationships between constituents (household member, employer, elected representative, etc.)
CREATE TABLE constituent_relationship (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    constituent_a_id    UUID NOT NULL REFERENCES constituent(id) ON DELETE CASCADE,
    constituent_b_id    UUID NOT NULL REFERENCES constituent(id) ON DELETE CASCADE,
    relationship_type_id UUID NOT NULL REFERENCES relationship_type(id),
    is_active           BOOLEAN NOT NULL DEFAULT true,
    start_date          DATE,
    end_date            DATE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT chk_no_self_relationship CHECK (constituent_a_id <> constituent_b_id)
);

CREATE INDEX idx_rel_a ON constituent_relationship(constituent_a_id);
CREATE INDEX idx_rel_b ON constituent_relationship(constituent_b_id);

-- Relationship type lookup
CREATE TABLE relationship_type (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(100) NOT NULL,       -- e.g., 'household_member', 'employer', 'elected_representative'
    label_a_to_b    VARCHAR(100) NOT NULL,       -- e.g., 'is household member of'
    label_b_to_a    VARCHAR(100) NOT NULL,       -- e.g., 'has household member'
    is_bidirectional BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Constituent tags / segments for audience targeting
CREATE TABLE tag (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(100) NOT NULL,
    category        VARCHAR(50),                  -- interest, demographic, service_area, campaign
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_tag_tenant ON tag(tenant_id);

CREATE TABLE constituent_tag (
    constituent_id  UUID NOT NULL REFERENCES constituent(id) ON DELETE CASCADE,
    tag_id          UUID NOT NULL REFERENCES tag(id) ON DELETE CASCADE,
    tagged_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (constituent_id, tag_id)
);
```

## Case Management

```sql
-- Case types (service request, complaint, enquiry, casework, referral)
CREATE TABLE case_type (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(100) NOT NULL,
    description     TEXT,
    default_sla_hours INTEGER,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Core case record
CREATE TABLE constituent_case (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    case_number     VARCHAR(50) NOT NULL,
    case_type_id    UUID NOT NULL REFERENCES case_type(id),
    status          VARCHAR(30) NOT NULL DEFAULT 'open'
                    CHECK (status IN ('open', 'in_progress', 'pending', 'escalated', 'resolved', 'closed')),
    priority        VARCHAR(20) NOT NULL DEFAULT 'normal'
                    CHECK (priority IN ('low', 'normal', 'high', 'urgent')),
    subject         VARCHAR(500) NOT NULL,
    description     TEXT,
    -- Assignment
    department_id   UUID REFERENCES department(id),
    assigned_to     UUID REFERENCES staff_user(id),
    -- SLA tracking
    sla_due_at      TIMESTAMPTZ,
    sla_breached    BOOLEAN NOT NULL DEFAULT false,
    -- AI classification
    ai_category     VARCHAR(100),
    ai_priority_score DECIMAL(5, 4),              -- 0.0000 to 1.0000
    ai_sentiment    VARCHAR(20),                   -- positive, neutral, negative, urgent
    -- GIS location (Open311-aligned)
    latitude        DECIMAL(10, 7),
    longitude       DECIMAL(10, 7),
    address_string  VARCHAR(500),
    -- Resolution
    resolution_notes TEXT,
    resolved_at     TIMESTAMPTZ,
    closed_at       TIMESTAMPTZ,
    -- Metadata
    source_channel  VARCHAR(30),                   -- web, email, phone, sms, mobile_app, walk_in, open311
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE UNIQUE INDEX idx_case_number ON constituent_case(tenant_id, case_number);
CREATE INDEX idx_case_tenant_status ON constituent_case(tenant_id, status);
CREATE INDEX idx_case_assigned ON constituent_case(assigned_to) WHERE assigned_to IS NOT NULL;
CREATE INDEX idx_case_department ON constituent_case(department_id);
CREATE INDEX idx_case_sla ON constituent_case(tenant_id, sla_due_at) WHERE sla_breached = false AND status NOT IN ('resolved', 'closed');
CREATE INDEX idx_case_created ON constituent_case(tenant_id, created_at);

-- Case participants (constituent involvement in a case)
-- Mirrors Salesforce PSS CaseParticipant
CREATE TABLE case_participant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    case_id         UUID NOT NULL REFERENCES constituent_case(id) ON DELETE CASCADE,
    constituent_id  UUID NOT NULL REFERENCES constituent(id),
    role            VARCHAR(50) NOT NULL,           -- requester, subject, witness, representative, related_party
    is_primary      BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_case_participant_case ON case_participant(case_id);
CREATE INDEX idx_case_participant_constituent ON case_participant(constituent_id);
```

## Service Request (Open311 Alignment)

```sql
-- Open311 service catalogue
CREATE TABLE service (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    service_code    VARCHAR(50) NOT NULL,           -- Open311 service_code
    service_name    VARCHAR(255) NOT NULL,          -- Open311 service_name
    description     TEXT,
    group_name      VARCHAR(100),                   -- Open311 group
    metadata        BOOLEAN NOT NULL DEFAULT false,  -- Open311 metadata flag
    type            VARCHAR(20) NOT NULL DEFAULT 'realtime'
                    CHECK (type IN ('realtime', 'batch', 'blackbox')),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE UNIQUE INDEX idx_service_code ON service(tenant_id, service_code);

-- Open311 service definition attributes
CREATE TABLE service_attribute (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    service_id      UUID NOT NULL REFERENCES service(id) ON DELETE CASCADE,
    attribute_code  VARCHAR(50) NOT NULL,
    description     TEXT NOT NULL,
    datatype        VARCHAR(20) NOT NULL,            -- string, number, datetime, text, singlevaluelist, multivaluelist
    is_required     BOOLEAN NOT NULL DEFAULT false,
    sort_order      INTEGER NOT NULL DEFAULT 0,
    values          JSONB,                           -- for list types: [{"key": "val1", "name": "Value 1"}, ...]
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Service request (extends case with Open311 fields)
CREATE TABLE service_request (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    case_id         UUID NOT NULL REFERENCES constituent_case(id) ON DELETE CASCADE,
    service_id      UUID NOT NULL REFERENCES service(id),
    service_request_id VARCHAR(100),                 -- Open311 service_request_id
    token           VARCHAR(100),                    -- Open311 token for async requests
    media_url       TEXT,                            -- Open311 media_url
    attribute_values JSONB NOT NULL DEFAULT '{}',    -- submitted service attribute values
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_sr_case ON service_request(case_id);
CREATE INDEX idx_sr_service ON service_request(service_id);
CREATE INDEX idx_sr_open311_id ON service_request(service_request_id) WHERE service_request_id IS NOT NULL;
```

## Department & Staff

```sql
CREATE TABLE department (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    parent_id       UUID REFERENCES department(id),
    jurisdiction_id UUID REFERENCES jurisdiction(id),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_dept_tenant ON department(tenant_id);
CREATE INDEX idx_dept_parent ON department(parent_id);

CREATE TABLE staff_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    email           VARCHAR(255) NOT NULL,
    display_name    VARCHAR(255) NOT NULL,
    department_id   UUID REFERENCES department(id),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE UNIQUE INDEX idx_staff_email ON staff_user(tenant_id, email);

-- Role-based access control
CREATE TABLE role (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(100) NOT NULL,
    permissions     JSONB NOT NULL DEFAULT '[]',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE staff_role (
    staff_user_id   UUID NOT NULL REFERENCES staff_user(id) ON DELETE CASCADE,
    role_id         UUID NOT NULL REFERENCES role(id) ON DELETE CASCADE,
    PRIMARY KEY (staff_user_id, role_id)
);

-- GIS boundary data for automatic routing
CREATE TABLE geographic_boundary (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    boundary_type   VARCHAR(50) NOT NULL,           -- ward, district, precinct, zone
    department_id   UUID REFERENCES department(id),
    assigned_to     UUID REFERENCES staff_user(id),
    geom            GEOMETRY(MultiPolygon, 4326),   -- PostGIS geometry
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_boundary_geom ON geographic_boundary USING GIST(geom);
CREATE INDEX idx_boundary_tenant ON geographic_boundary(tenant_id);
```

## Activity & Interaction History

```sql
-- Activity types (phone call, email, note, status change, etc.)
CREATE TABLE activity_type (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(100) NOT NULL UNIQUE,
    category        VARCHAR(50),                     -- communication, internal, system, ai
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Activity log (all interactions with a constituent or case)
CREATE TABLE activity (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    activity_type_id UUID NOT NULL REFERENCES activity_type(id),
    case_id         UUID REFERENCES constituent_case(id),
    subject         VARCHAR(500),
    details         TEXT,
    direction       VARCHAR(10),                     -- inbound, outbound, internal
    channel         VARCHAR(30),                     -- email, phone, sms, web, in_person
    -- AI metadata
    ai_generated    BOOLEAN NOT NULL DEFAULT false,
    ai_confidence   DECIMAL(5, 4),
    -- Metadata
    performed_by    UUID REFERENCES staff_user(id),
    performed_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_activity_tenant ON activity(tenant_id);
CREATE INDEX idx_activity_case ON activity(case_id) WHERE case_id IS NOT NULL;
CREATE INDEX idx_activity_performed ON activity(performed_at);

-- Many-to-many: activity <-> constituent
CREATE TABLE activity_constituent (
    activity_id     UUID NOT NULL REFERENCES activity(id) ON DELETE CASCADE,
    constituent_id  UUID NOT NULL REFERENCES constituent(id) ON DELETE CASCADE,
    role            VARCHAR(30) NOT NULL DEFAULT 'target',  -- source, target, cc
    PRIMARY KEY (activity_id, constituent_id)
);

CREATE INDEX idx_act_const_constituent ON activity_constituent(constituent_id);
```

## Communications & Outreach

```sql
-- Campaign / outreach
CREATE TABLE campaign (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    campaign_type   VARCHAR(50),                     -- email_blast, sms, newsletter, town_hall, survey
    status          VARCHAR(30) NOT NULL DEFAULT 'draft',
    scheduled_at    TIMESTAMPTZ,
    sent_at         TIMESTAMPTZ,
    created_by      UUID REFERENCES staff_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Message templates
CREATE TABLE message_template (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    channel         VARCHAR(20) NOT NULL,            -- email, sms
    subject         VARCHAR(500),
    body_html       TEXT,
    body_text       TEXT,
    variables       JSONB,                           -- [{name: "constituent_name", type: "string"}, ...]
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Outbound message log
CREATE TABLE outbound_message (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    campaign_id     UUID REFERENCES campaign(id),
    template_id     UUID REFERENCES message_template(id),
    constituent_id  UUID NOT NULL REFERENCES constituent(id),
    case_id         UUID REFERENCES constituent_case(id),
    channel         VARCHAR(20) NOT NULL,
    recipient       VARCHAR(255) NOT NULL,           -- email address or phone number
    subject         VARCHAR(500),
    body            TEXT,
    status          VARCHAR(20) NOT NULL DEFAULT 'queued',  -- queued, sent, delivered, bounced, failed
    sent_at         TIMESTAMPTZ,
    delivered_at    TIMESTAMPTZ,
    opened_at       TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_outbound_tenant ON outbound_message(tenant_id);
CREATE INDEX idx_outbound_constituent ON outbound_message(constituent_id);
CREATE INDEX idx_outbound_campaign ON outbound_message(campaign_id) WHERE campaign_id IS NOT NULL;
```

## Attachments & Documents

```sql
CREATE TABLE attachment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    entity_type     VARCHAR(50) NOT NULL,            -- case, activity, constituent, service_request
    entity_id       UUID NOT NULL,
    filename        VARCHAR(500) NOT NULL,
    content_type    VARCHAR(100) NOT NULL,
    size_bytes      BIGINT NOT NULL,
    storage_key     VARCHAR(500) NOT NULL,            -- S3/blob storage key
    uploaded_by     UUID REFERENCES staff_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_attachment_entity ON attachment(entity_type, entity_id);
CREATE INDEX idx_attachment_tenant ON attachment(tenant_id);
```

## Audit & Compliance

```sql
-- Immutable audit log (NIST 800-53 AU controls)
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    actor_id        UUID,                             -- staff_user.id or NULL for system
    actor_type      VARCHAR(20) NOT NULL,             -- staff, system, api, ai_agent
    action          VARCHAR(50) NOT NULL,             -- create, read, update, delete, login, export
    entity_type     VARCHAR(50) NOT NULL,             -- constituent, case, activity, etc.
    entity_id       UUID,
    old_values      JSONB,
    new_values      JSONB,
    ip_address      INET,
    user_agent      TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Partition by month for performance
-- CREATE TABLE audit_log_2026_05 PARTITION OF audit_log FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');

CREATE INDEX idx_audit_tenant ON audit_log(tenant_id);
CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_actor ON audit_log(actor_id) WHERE actor_id IS NOT NULL;
CREATE INDEX idx_audit_created ON audit_log(created_at);
```

## Reporting & Analytics

```sql
-- Saved searches / smart groups (CiviCRM SearchKit equivalent)
CREATE TABLE saved_search (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    entity_type     VARCHAR(50) NOT NULL,
    query_definition JSONB NOT NULL,                  -- structured query AST
    is_smart_group  BOOLEAN NOT NULL DEFAULT false,
    created_by      UUID REFERENCES staff_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Dashboard widget configurations
CREATE TABLE dashboard_widget (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    widget_type     VARCHAR(50) NOT NULL,            -- case_volume, response_time, sla_compliance, heatmap
    query_definition JSONB NOT NULL,
    display_config  JSONB NOT NULL DEFAULT '{}',
    created_by      UUID REFERENCES staff_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Core Identity & Multi-Tenancy | 2 | tenant, jurisdiction |
| Constituent Management | 5 | constituent, constituent_channel, constituent_relationship, relationship_type, constituent_tag + tag |
| Case Management | 3 | constituent_case, case_type, case_participant |
| Service Request (Open311) | 3 | service, service_attribute, service_request |
| Department & Staff | 5 | department, staff_user, role, staff_role, geographic_boundary |
| Activity & Interaction | 3 | activity_type, activity, activity_constituent |
| Communications & Outreach | 3 | campaign, message_template, outbound_message |
| Attachments | 1 | attachment |
| Audit & Compliance | 1 | audit_log |
| Reporting & Analytics | 2 | saved_search, dashboard_widget |
| **Total** | **28** | |

---

## Key Design Decisions

1. **UUID primary keys everywhere** — enables distributed ID generation, safe cross-system merging (NIEM exchanges), and prevents enumeration attacks on government data.

2. **Tenant-per-row multi-tenancy** — a `tenant_id` column on all operational tables with indexes enables a single database to serve multiple agencies while keeping infrastructure simple. Row-Level Security (RLS) policies can enforce tenant isolation at the database level.

3. **Constituent is polymorphic** — a single `constituent` table handles individuals, organisations, and households via `constituent_type` discriminator, following the CiviCRM contact model. This avoids the complexity of separate tables while keeping the 360-degree view unified.

4. **Case and Service Request are separate but linked** — `constituent_case` is the universal case record; `service_request` extends it with Open311-specific fields. This allows the same case management workflow for complaints, enquiries, and casework that do not originate from Open311.

5. **Open311 GeoReport v2 fields embedded directly** — the `service`, `service_attribute`, and `service_request` tables mirror the Open311 specification field-for-field, ensuring a standards-compliant API can be generated directly from the schema.

6. **GIS-native routing via PostGIS** — the `geographic_boundary` table stores department/ward boundaries as PostGIS geometries, enabling automatic case assignment with `ST_Contains(geom, ST_Point(lng, lat))` queries.

7. **Activity as the universal interaction record** — all constituent interactions (calls, emails, notes, AI-generated responses) flow through the `activity` table with a junction table to constituents. This mirrors CiviCRM's activity model and provides the unified interaction history required by the feature specification.

8. **AI metadata on case and activity records** — `ai_category`, `ai_priority_score`, `ai_sentiment`, `ai_generated`, and `ai_confidence` columns store ML model outputs alongside human data, supporting the explainable AI routing requirement.

9. **Audit log partitioned by month** — the `audit_log` table is designed for time-range partitioning to manage the high volume of audit events required by NIST 800-53 AU controls without degrading query performance.

10. **JSONB used sparingly** — only where the data is genuinely variable (service attribute values, template variables, dashboard configs, role permissions). Core fields are always typed columns with constraints.
