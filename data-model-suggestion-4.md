# Data Model Suggestion 4: Graph-Relational Hybrid

> Project: Constituent Relationship Management · Created: 2026-05-22

## Philosophy

This model combines conventional relational tables for operational CRUD (cases, staff, departments, service requests) with a property graph layer for relationship-heavy queries. The graph layer models the network of connections between constituents, households, elected officials, departments, cases, and agencies, enabling queries that are impractical in a purely relational model: "show all cases affecting members of this household," "find all constituents connected to this elected official through any path," or "detect potential conflicts of interest in case assignment."

Constituent relationship management is, at its core, a graph problem. A constituent belongs to a household, is represented by an elected official, has filed cases with multiple departments, is related to other constituents who have their own cases, and interacts with staff who serve specific geographic boundaries. Traversing these connections to answer cross-cutting questions — the "360-degree view" that every CRM promises — is naturally expressed as graph queries.

PostgreSQL 18 introduces native SQL/PGQ (Property Graph Queries) support, and the Apache AGE extension brings openCypher query capability to PostgreSQL 14+. This model uses a graph layer that can be implemented with either approach. The relational tables handle the operational workload (case creation, status updates, SLA tracking) where PostgreSQL excels, while the graph layer handles relationship traversal, network analysis, and cross-entity discovery queries.

**Best for:** Agencies managing complex constituent networks (elected offices, social services, housing authorities) where understanding relationships between constituents, cross-agency case patterns, and conflict-of-interest detection are critical requirements.

**Trade-offs:**
- (+) Natural expression of multi-hop relationship queries ("2 degrees of separation from this constituent")
- (+) Cross-agency constituent journey visibility — the key underserved area identified in feature research
- (+) Conflict-of-interest detection for case assignment (staff related to constituent)
- (+) Household and family unit analysis for social services casework
- (+) Network-effect analysis: identify constituents who influence others
- (-) Graph query performance requires careful index management and query tuning
- (-) Dual data model (relational + graph) increases operational complexity
- (-) Graph databases/extensions (AGE, SQL/PGQ) are less mature than core PostgreSQL
- (-) Fewer developers are experienced with graph query languages (Cypher, GQL)
- (-) Synchronisation between relational tables and graph layer adds complexity

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| Open311 GeoReport v2 | Service request operational tables mirror Open311 schema; graph edges connect requests to geographic nodes |
| NIEM Human Services Domain | Person and organisation entities in graph nodes align with NIEM person/organisation type semantics |
| ISO 3166-1/2 | Jurisdiction nodes in the graph use ISO 3166 codes as properties |
| SQL/PGQ (ISO 9075:2023 Part 16) | Graph queries use the emerging SQL standard for property graph queries |
| Apache TinkerPop / openCypher | Alternative graph query interface via Apache AGE extension |
| GovStack Building Blocks | Graph model supports federated identity and cross-agency data exchange patterns from GovStack GovSpecs 2.0 |

---

## Graph Layer

```sql
-- Graph nodes: every major entity is also a graph node
CREATE TABLE graph_node (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    node_type       VARCHAR(50) NOT NULL,           -- constituent, household, department, staff,
                                                     -- elected_official, agency, geographic_area,
                                                     -- case, service_request
    entity_id       UUID NOT NULL,                   -- FK to the corresponding relational table
    label           VARCHAR(255) NOT NULL,           -- human-readable label for display
    properties      JSONB NOT NULL DEFAULT '{}',     -- denormalised properties for graph queries
    -- properties example for constituent node:
    -- {
    --   "constituent_type": "individual",
    --   "display_name": "Maria Santos",
    --   "ward": 7,
    --   "sentiment_trend": "declining",
    --   "open_cases": 3
    -- }
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, node_type, entity_id)
);

CREATE INDEX idx_node_tenant_type ON graph_node(tenant_id, node_type);
CREATE INDEX idx_node_entity ON graph_node(entity_id);
CREATE INDEX idx_node_properties ON graph_node USING GIN(properties jsonb_path_ops);

-- Graph edges: relationships between any two nodes
CREATE TABLE graph_edge (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    source_node_id  UUID NOT NULL REFERENCES graph_node(id) ON DELETE CASCADE,
    target_node_id  UUID NOT NULL REFERENCES graph_node(id) ON DELETE CASCADE,
    edge_type       VARCHAR(50) NOT NULL,           -- see edge type catalogue below
    properties      JSONB NOT NULL DEFAULT '{}',
    -- properties example for FILED_CASE edge:
    -- {"role": "requester", "filed_at": "2026-05-15T10:30:00Z"}
    weight          DECIMAL(5, 2) DEFAULT 1.0,       -- for weighted graph algorithms
    is_active       BOOLEAN NOT NULL DEFAULT true,
    valid_from      TIMESTAMPTZ DEFAULT now(),
    valid_to        TIMESTAMPTZ,                     -- NULL = currently active
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_edge_source ON graph_edge(source_node_id);
CREATE INDEX idx_edge_target ON graph_edge(target_node_id);
CREATE INDEX idx_edge_type ON graph_edge(tenant_id, edge_type);
CREATE INDEX idx_edge_tenant ON graph_edge(tenant_id);
CREATE INDEX idx_edge_active ON graph_edge(is_active, valid_from, valid_to);
CREATE INDEX idx_edge_properties ON graph_edge USING GIN(properties jsonb_path_ops);

-- Edge type catalogue
-- Constituent relationships:
--   HOUSEHOLD_MEMBER   constituent -> household
--   FAMILY_OF          constituent -> constituent
--   EMPLOYER_OF        organisation -> constituent
--   REPRESENTS         elected_official -> constituent (via geographic_area)
--   REFERRED_BY        constituent -> constituent
--
-- Case relationships:
--   FILED_CASE         constituent -> case (properties: role=requester|subject|witness)
--   ASSIGNED_TO        case -> staff
--   BELONGS_TO_DEPT    case -> department
--   RELATED_CASE       case -> case (properties: relation=duplicate|followup|escalation)
--   SERVICE_REQUEST    case -> service (properties: service_code, Open311 fields)
--
-- Organisational:
--   WORKS_IN           staff -> department
--   MANAGES            staff -> staff
--   SUPERVISES         staff -> department
--   SERVES_AREA        department -> geographic_area
--
-- Geographic:
--   LOCATED_IN         constituent -> geographic_area
--   PART_OF            geographic_area -> geographic_area (ward -> district -> city)
--   WITHIN_JURISDICTION geographic_area -> jurisdiction
--
-- Communication:
--   CONTACTED          staff -> constituent (properties: channel, direction, timestamp)
--   CAMPAIGN_TARGET    campaign -> constituent
```

## Relational Tables (Operational CRUD)

```sql
-- These tables handle the day-to-day operational workload
-- The graph layer references these via graph_node.entity_id

CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    jurisdiction_id UUID REFERENCES jurisdiction(id),
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
    timezone        VARCHAR(50),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE constituent (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    constituent_type VARCHAR(20) NOT NULL,
    display_name    VARCHAR(255) NOT NULL,
    first_name      VARCHAR(100),
    last_name       VARCHAR(100),
    organisation_name VARCHAR(255),
    preferred_language VARCHAR(10) DEFAULT 'en',
    preferred_channel VARCHAR(20) DEFAULT 'email',
    external_id     VARCHAR(255),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    custom_fields   JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_constituent_tenant ON constituent(tenant_id);
CREATE INDEX idx_constituent_name ON constituent(tenant_id, last_name, first_name);

CREATE TABLE constituent_channel (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    constituent_id  UUID NOT NULL REFERENCES constituent(id) ON DELETE CASCADE,
    channel_type    VARCHAR(20) NOT NULL,
    value           VARCHAR(255) NOT NULL,
    is_primary      BOOLEAN NOT NULL DEFAULT false,
    properties      JSONB NOT NULL DEFAULT '{}',
    is_verified     BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_channel_constituent ON constituent_channel(constituent_id);

CREATE TABLE household (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255),
    address         JSONB,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE department (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    parent_id       UUID REFERENCES department(id),
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
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE UNIQUE INDEX idx_staff_email ON staff_user(tenant_id, email);

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

CREATE TABLE geographic_area (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    area_type       VARCHAR(50) NOT NULL,            -- ward, district, precinct, zone, city, county
    geom            GEOMETRY(MultiPolygon, 4326),
    properties      JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_area_geom ON geographic_area USING GIST(geom);

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
    ai_metadata     JSONB NOT NULL DEFAULT '{}',
    resolution      JSONB,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE UNIQUE INDEX idx_case_number ON constituent_case(tenant_id, case_number);
CREATE INDEX idx_case_status ON constituent_case(tenant_id, status);
CREATE INDEX idx_case_assigned ON constituent_case(assigned_to) WHERE assigned_to IS NOT NULL;

CREATE TABLE service (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    service_code    VARCHAR(50) NOT NULL,
    service_name    VARCHAR(255) NOT NULL,
    description     TEXT,
    group_name      VARCHAR(100),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

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
    metadata        JSONB NOT NULL DEFAULT '{}',
    performed_by    UUID REFERENCES staff_user(id),
    performed_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_activity_case ON activity(case_id) WHERE case_id IS NOT NULL;
CREATE INDEX idx_activity_constituent ON activity(constituent_id) WHERE constituent_id IS NOT NULL;

CREATE TABLE campaign (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    campaign_type   VARCHAR(50) NOT NULL,
    status          VARCHAR(30) NOT NULL DEFAULT 'draft',
    audience_filter JSONB NOT NULL DEFAULT '{}',
    content         JSONB NOT NULL DEFAULT '{}',
    metrics         JSONB NOT NULL DEFAULT '{}',
    created_by      UUID REFERENCES staff_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE attachment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    entity_type     VARCHAR(50) NOT NULL,
    entity_id       UUID NOT NULL,
    filename        VARCHAR(500) NOT NULL,
    content_type    VARCHAR(100) NOT NULL,
    size_bytes      BIGINT NOT NULL,
    storage_key     VARCHAR(500) NOT NULL,
    uploaded_by     UUID REFERENCES staff_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    actor_id        UUID,
    actor_type      VARCHAR(20) NOT NULL,
    action          VARCHAR(50) NOT NULL,
    entity_type     VARCHAR(50) NOT NULL,
    entity_id       UUID,
    changes         JSONB,
    metadata        JSONB,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_tenant ON audit_log(tenant_id, created_at);
```

## Graph Query Examples

### Find all cases affecting a household

```sql
-- Using recursive CTE (works on standard PostgreSQL)
-- "Show all open cases filed by any member of Maria Santos' household"
WITH household_members AS (
    -- Find Maria's household
    SELECT e1.target_node_id AS household_node_id
    FROM graph_node n
    JOIN graph_edge e1 ON e1.source_node_id = n.id
    WHERE n.entity_id = 'maria-uuid'
      AND n.node_type = 'constituent'
      AND e1.edge_type = 'HOUSEHOLD_MEMBER'
      AND e1.is_active = true
),
members AS (
    -- Find all members of that household
    SELECT gn.entity_id AS constituent_id
    FROM household_members hm
    JOIN graph_edge e2 ON e2.target_node_id = hm.household_node_id
    JOIN graph_node gn ON gn.id = e2.source_node_id
    WHERE e2.edge_type = 'HOUSEHOLD_MEMBER'
      AND e2.is_active = true
      AND gn.node_type = 'constituent'
),
member_cases AS (
    -- Find all cases filed by those members
    SELECT gn_case.entity_id AS case_id
    FROM members m
    JOIN graph_node gn_const ON gn_const.entity_id = m.constituent_id AND gn_const.node_type = 'constituent'
    JOIN graph_edge e3 ON e3.source_node_id = gn_const.id AND e3.edge_type = 'FILED_CASE'
    JOIN graph_node gn_case ON gn_case.id = e3.target_node_id AND gn_case.node_type = 'case'
)
SELECT cc.*
FROM constituent_case cc
JOIN member_cases mc ON mc.case_id = cc.id
WHERE cc.status IN ('open', 'in_progress', 'escalated');
```

### Conflict of interest detection for case assignment

```sql
-- "Before assigning case X to staff member Y, check if Y has any relationship
--  (direct or 2-hop) with the case requester"
WITH case_constituents AS (
    -- Find all constituents linked to the case
    SELECT gn.id AS constituent_node_id
    FROM graph_edge e
    JOIN graph_node gn ON gn.id = e.source_node_id
    WHERE e.target_node_id = (
        SELECT id FROM graph_node WHERE entity_id = 'case-uuid' AND node_type = 'case'
    )
    AND e.edge_type = 'FILED_CASE'
    AND e.is_active = true
),
staff_connections AS (
    -- Find all constituents connected to the staff member within 2 hops
    SELECT DISTINCT e2.target_node_id AS connected_node_id
    FROM graph_node staff_node
    -- Direct connections
    JOIN graph_edge e1 ON e1.source_node_id = staff_node.id
    LEFT JOIN graph_edge e2 ON e2.source_node_id = e1.target_node_id
    WHERE staff_node.entity_id = 'staff-uuid'
      AND staff_node.node_type = 'staff'
      AND e1.edge_type IN ('FAMILY_OF', 'HOUSEHOLD_MEMBER', 'REFERRED_BY')
      AND e1.is_active = true
    UNION
    -- Direct connections (target to source direction)
    SELECT e1.source_node_id
    FROM graph_node staff_node
    JOIN graph_edge e1 ON e1.target_node_id = staff_node.id
    WHERE staff_node.entity_id = 'staff-uuid'
      AND staff_node.node_type = 'staff'
      AND e1.edge_type IN ('FAMILY_OF', 'HOUSEHOLD_MEMBER')
      AND e1.is_active = true
)
SELECT EXISTS (
    SELECT 1
    FROM case_constituents cc
    JOIN staff_connections sc ON sc.connected_node_id = cc.constituent_node_id
) AS has_conflict;
```

### Cross-agency constituent journey

```sql
-- "Show all interactions and cases for constituent X across all agencies"
-- (requires federated graph or cross-tenant graph access)
SELECT
    gn_case.properties->>'case_number' AS case_number,
    gn_dept.label AS department,
    e.properties->>'role' AS constituent_role,
    gn_case.properties->>'status' AS status,
    e.valid_from AS filed_at
FROM graph_node gn_const
JOIN graph_edge e ON e.source_node_id = gn_const.id AND e.edge_type = 'FILED_CASE'
JOIN graph_node gn_case ON gn_case.id = e.target_node_id
LEFT JOIN graph_edge e_dept ON e_dept.source_node_id = gn_case.id AND e_dept.edge_type = 'BELONGS_TO_DEPT'
LEFT JOIN graph_node gn_dept ON gn_dept.id = e_dept.target_node_id
WHERE gn_const.entity_id = 'constituent-uuid'
  AND gn_const.node_type = 'constituent'
ORDER BY e.valid_from DESC;
```

### Constituent influence network

```sql
-- "Find the most connected constituents in Ward 7 (potential community leaders)"
SELECT
    gn.label AS constituent_name,
    gn.properties->>'display_name' AS display_name,
    COUNT(DISTINCT e.id) AS connection_count,
    COUNT(DISTINCT CASE WHEN e.edge_type = 'FILED_CASE' THEN e.id END) AS cases_filed,
    COUNT(DISTINCT CASE WHEN e.edge_type = 'REFERRED_BY' THEN e.id END) AS referrals_made
FROM graph_node gn
JOIN graph_edge e ON (e.source_node_id = gn.id OR e.target_node_id = gn.id)
WHERE gn.node_type = 'constituent'
  AND gn.tenant_id = 'tenant-uuid'
  AND gn.properties @> '{"ward": 7}'
  AND gn.is_active = true
  AND e.is_active = true
GROUP BY gn.id, gn.label, gn.properties
ORDER BY connection_count DESC
LIMIT 20;
```

### Using Apache AGE (openCypher syntax)

```sql
-- If using Apache AGE extension, the household query becomes:
SELECT * FROM cypher('constituent_graph', $$
    MATCH (c:constituent {entity_id: 'maria-uuid'})-[:HOUSEHOLD_MEMBER]->(h:household)
          <-[:HOUSEHOLD_MEMBER]-(member:constituent)
          -[:FILED_CASE]->(case:case)
    WHERE case.status IN ['open', 'in_progress', 'escalated']
    RETURN member.display_name, case.case_number, case.subject, case.status
$$) AS (member_name agtype, case_number agtype, subject agtype, status agtype);

-- Conflict of interest with path query:
SELECT * FROM cypher('constituent_graph', $$
    MATCH path = (staff:staff {entity_id: 'staff-uuid'})
                 -[:FAMILY_OF|HOUSEHOLD_MEMBER*1..2]-
                 (constituent:constituent)
                 -[:FILED_CASE]->
                 (case:case {entity_id: 'case-uuid'})
    RETURN path
$$) AS (path agtype);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Graph Layer | 2 | graph_node, graph_edge |
| Core Identity | 2 | tenant, jurisdiction |
| Constituent Management | 3 | constituent, constituent_channel, household |
| Case Management | 2 | constituent_case, service |
| Department & Staff | 4 | department, staff_user, role, staff_role, geographic_area |
| Activity | 1 | activity |
| Communications | 1 | campaign |
| Attachments | 1 | attachment |
| Audit | 1 | audit_log |
| **Total** | **17+2 graph** | 17 relational tables + 2 graph tables; graph edges replace many junction tables |

---

## Key Design Decisions

1. **Dual-model architecture** — relational tables handle operational CRUD where PostgreSQL's query planner, indexes, and transaction semantics excel. The graph layer handles relationship traversal and network analysis. This avoids forcing either paradigm where it does not fit.

2. **Graph nodes reference relational entities** — `graph_node.entity_id` is a logical foreign key to the corresponding relational table. The graph node also carries denormalised `properties` (JSONB) for graph-only queries that do not need to join back to relational tables.

3. **Temporal edges** — `valid_from` and `valid_to` on edges enable temporal relationship queries ("who was this constituent's elected representative in 2024?"). Edges are never deleted, only deactivated by setting `valid_to`, preserving historical relationship data.

4. **Edge weights for network analysis** — the `weight` column on edges enables weighted graph algorithms (PageRank, community detection, influence scoring) that can identify community leaders, predict case routing effectiveness, or detect anomalous relationship patterns.

5. **Household as a first-class node** — rather than modelling households as a relationship between constituents, the household is a separate graph node that members connect to. This simplifies queries ("all members of household X") and supports households with changing membership over time.

6. **Graph synchronisation via triggers or events** — when a relational record is created/updated, a trigger or application event creates/updates the corresponding graph node and edges. This keeps the graph layer consistent without requiring all writes to go through the graph.

7. **Compatible with Apache AGE and SQL/PGQ** — the `graph_node`/`graph_edge` table structure works with raw SQL recursive CTEs today and can be migrated to Apache AGE (openCypher) or native SQL/PGQ (PostgreSQL 18+) as those features mature. The model is not locked into a specific graph query engine.

8. **Cross-agency graph federation** — the graph model naturally supports federated constituent identity. When a constituent appears across multiple tenant graphs (identified via `external_id` or login.gov federation), a cross-tenant edge can link them, enabling the "cross-agency constituent journey" that no existing product offers.

9. **Conflict-of-interest detection is a graph query** — checking whether a staff member has a personal relationship with a case constituent is a 2-hop graph traversal. In a relational model, this would require multiple self-joins across relationship tables; in the graph model, it is a natural path query.

10. **Graph indexes for performance** — indexes on `source_node_id`, `target_node_id`, and `edge_type` ensure that graph traversals start fast. For Apache AGE, additional graph-specific indexes can be created on node and edge labels.
