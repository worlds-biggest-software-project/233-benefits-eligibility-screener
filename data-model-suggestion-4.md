# Data Model Suggestion 4: Graph-Relational Hybrid

> Project: Benefits Eligibility Screener · Created: 2026-05-22

## Philosophy

This model combines a relational backbone for operational CRUD with a property-graph layer for relationship-intensive queries. The core insight is that benefits eligibility is fundamentally a graph problem: persons belong to households, households exist in jurisdictions, jurisdictions offer programmes, programmes have rules that reference other programmes (categorical eligibility), and referral networks form webs between agencies, CBOs, and healthcare providers. A graph layer makes these relationship traversals natural and efficient.

The relational tables handle transactional operations (creating sessions, recording income, storing documents) where ACID guarantees and foreign-key constraints matter. The graph layer -- implemented as `graph_node` and `graph_edge` tables in PostgreSQL with recursive CTEs, or optionally backed by a dedicated graph database like Neo4j -- handles queries that would otherwise require complex multi-table JOINs or recursive lookups: "find all programmes this household qualifies for, including programmes they become categorically eligible for by qualifying for another programme" or "trace the referral network: which CBOs serve this ZIP code for food assistance, and which of them have capacity?"

Graph-relational hybrids are used in fraud detection (following money trails), social network analysis, and supply-chain management. For benefits eligibility, the graph excels at cross-programme bundling (the AI-native advantage described in the project vision), household relationship modelling (custody arrangements, tax-filing units vs. SNAP households vs. Medicaid households), and community resource mapping.

**Best for:** Deployments that prioritise cross-programme bundling, complex household relationship modelling, community resource network mapping, and scenarios where the relationships between entities are as important as the entities themselves.

**Trade-offs:**
- (+) Cross-programme bundling queries are natural graph traversals, not complex SQL JOINs
- (+) Categorical eligibility chains (qualifying for A makes you eligible for B) are simple edge traversals
- (+) Community resource and referral networks are naturally graph-shaped
- (+) Household composition variants (SNAP household vs. MAGI tax unit vs. physical household) are parallel edges, not separate tables
- (+) Extensible: new relationship types are new edge types, not schema changes
- (-) Graph query patterns (recursive CTEs or Cypher) are less familiar to most developers
- (-) Dual-write to relational and graph layers requires consistency management
- (-) Graph databases add operational complexity if using a separate system (Neo4j)
- (-) PostgreSQL recursive CTEs are powerful but can be slow for deep traversals without careful indexing
- (-) Not as natural for simple CRUD operations or flat reporting

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ACA MAGI Rules (45 CFR Part 155) | MAGI tax household composition modelled as a specific edge type (`MAGI_TAX_UNIT_MEMBER`) separate from physical household edges; enables correct MAGI household identification |
| Social Security Act Title XIX/XXI | Categorical eligibility between programmes modelled as `CATEGORICAL_ELIGIBILITY` edges between programme nodes |
| FHIR SDOH Clinical Care IG | Referral network modelled as graph edges between patient/household nodes and organisation/programme nodes, mapping to FHIR ServiceRequest and Task resources |
| OpenFisca Entity Model | OpenFisca's entity hierarchy (person -> family -> household) maps to graph node types with `MEMBER_OF` edges |
| ICD-10 Z-codes / LOINC | SDOH screening responses attached as properties on person nodes, coded using standard terminologies |
| ISO 3166-1/2 | Jurisdiction hierarchy (country -> state -> county) modelled as a graph with `SUBDIVISION_OF` edges |
| Gravity Project SDOH | Community resource organisations and their service areas modelled as organisation nodes with `SERVES_AREA` edges to jurisdiction nodes |

---

## Relational Core Tables

```sql
-- ============================================================
-- PERSONS (relational, for transactional operations)
-- ============================================================
CREATE TABLE person (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    external_id     TEXT,
    first_name      TEXT NOT NULL,
    last_name       TEXT NOT NULL,
    date_of_birth   DATE,
    ssn_hash        TEXT,
    gender          TEXT,
    citizenship_status TEXT,
    disability_status BOOLEAN DEFAULT FALSE,
    veteran_status  BOOLEAN DEFAULT FALSE,
    pregnant        BOOLEAN DEFAULT FALSE,
    student_status  TEXT,
    primary_language TEXT DEFAULT 'en',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_person_external_id ON person(external_id);
CREATE INDEX idx_person_ssn_hash ON person(ssn_hash);

-- ============================================================
-- INCOME (relational, for calculations)
-- ============================================================
CREATE TABLE income (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    person_id       UUID NOT NULL REFERENCES person(id) ON DELETE CASCADE,
    income_type     TEXT NOT NULL,
    gross_amount    NUMERIC(12,2) NOT NULL,
    period          TEXT NOT NULL DEFAULT 'monthly',
    employer_name   TEXT,
    start_date      DATE,
    end_date        DATE,
    verified        BOOLEAN DEFAULT FALSE,
    verification_method TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_income_person ON income(person_id);

-- ============================================================
-- SCREENING SESSIONS (relational, transactional)
-- ============================================================
CREATE TABLE screening_session (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    household_node_id UUID NOT NULL,                   -- references graph_node.id for the household
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    channel         TEXT NOT NULL DEFAULT 'web',
    locale          TEXT DEFAULT 'en',
    status          TEXT NOT NULL DEFAULT 'in_progress',
    started_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ,
    intake_data     JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_session_household ON screening_session(household_node_id);
CREATE INDEX idx_session_tenant ON screening_session(tenant_id);
CREATE INDEX idx_session_status ON screening_session(status);

-- ============================================================
-- ELIGIBILITY RESULTS (relational, transactional)
-- ============================================================
CREATE TABLE eligibility_result (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id      UUID NOT NULL REFERENCES screening_session(id) ON DELETE CASCADE,
    household_node_id UUID NOT NULL,
    program_node_id UUID NOT NULL,                     -- references graph_node.id for the programme
    jurisdiction_code TEXT NOT NULL,
    status          TEXT NOT NULL,
    confidence      NUMERIC(3,2),
    estimated_monthly_benefit NUMERIC(10,2),
    estimated_annual_benefit NUMERIC(12,2),
    rule_results    JSONB NOT NULL DEFAULT '[]',
    explanation_summary TEXT,
    triggered_by    UUID,                              -- if this determination was triggered by categorical eligibility
                                                        -- from another programme, reference that result
    determined_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_result_session ON eligibility_result(session_id);
CREATE INDEX idx_result_program ON eligibility_result(program_node_id);
CREATE INDEX idx_result_status ON eligibility_result(status);
CREATE INDEX idx_result_triggered ON eligibility_result(triggered_by);

-- ============================================================
-- DOCUMENTS (relational, transactional)
-- ============================================================
CREATE TABLE document (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    person_id       UUID REFERENCES person(id),
    session_id      UUID REFERENCES screening_session(id),
    document_type   TEXT NOT NULL,
    file_name       TEXT NOT NULL,
    storage_path    TEXT NOT NULL,
    extraction_status TEXT DEFAULT 'pending',
    extracted_data  JSONB,
    uploaded_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_doc_person ON document(person_id);
CREATE INDEX idx_doc_session ON document(session_id);
```

## Graph Layer Tables

```sql
-- ============================================================
-- GRAPH NODES
-- ============================================================
CREATE TABLE graph_node (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    node_type       TEXT NOT NULL,                     -- node type taxonomy (see below)
    label           TEXT NOT NULL,                     -- human-readable label
    properties      JSONB DEFAULT '{}',                -- node-type-specific properties
    -- Reference to relational entity (optional; not all nodes have relational counterparts)
    entity_table    TEXT,                               -- 'person', 'tenant', NULL for pure graph nodes
    entity_id       UUID,                              -- FK to the relational table (not enforced to allow flexibility)
    is_active       BOOLEAN DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Node Type Taxonomy:
-- 'person'          -> individual person (links to person table)
-- 'household'       -> a household unit (physical, SNAP, MAGI tax unit, etc.)
-- 'program'         -> a benefit programme (SNAP, Medicaid, WIC, etc.)
-- 'jurisdiction'    -> a geographic/administrative area (US, US-CA, US-CA-037)
-- 'organisation'    -> a CBO, agency, health system, or service provider
-- 'rule'            -> an eligibility rule node (for rule-dependency traversal)
-- 'sdoh_need'       -> an SDOH category (food insecurity, housing instability, etc.)
-- 'service'         -> a specific service offered by an organisation

CREATE INDEX idx_gn_type ON graph_node(node_type);
CREATE INDEX idx_gn_entity ON graph_node(entity_table, entity_id);
CREATE INDEX idx_gn_label ON graph_node(label);
CREATE INDEX idx_gn_properties ON graph_node USING GIN (properties);

-- ============================================================
-- GRAPH EDGES
-- ============================================================
CREATE TABLE graph_edge (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    from_node_id    UUID NOT NULL REFERENCES graph_node(id) ON DELETE CASCADE,
    to_node_id      UUID NOT NULL REFERENCES graph_node(id) ON DELETE CASCADE,
    edge_type       TEXT NOT NULL,                     -- edge type taxonomy (see below)
    properties      JSONB DEFAULT '{}',                -- edge-type-specific properties
    weight          NUMERIC(5,2) DEFAULT 1.0,          -- for weighted traversals (e.g. referral priority)
    effective_date  DATE,                              -- temporal edges: when this relationship starts
    expiration_date DATE,                              -- temporal edges: when this relationship ends (NULL = current)
    is_active       BOOLEAN DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Edge Type Taxonomy:
--
-- Person-Household relationships:
--   'MEMBER_OF'                -> person is a member of a household
--                                 properties: {relationship: "head"|"spouse"|"child"|..., household_type: "physical"|"snap"|"magi_tax_unit"}
--   'TAX_DEPENDENT_OF'         -> person is a tax dependent of another person
--   'CUSTODIAL_PARENT_OF'      -> person has custody of another person
--
-- Household-Jurisdiction relationships:
--   'LOCATED_IN'               -> household is located in a jurisdiction
--
-- Programme-Jurisdiction relationships:
--   'AVAILABLE_IN'             -> programme is available in a jurisdiction
--                                 properties: {local_name: "CalFresh", application_url: "..."}
--
-- Programme-Programme relationships:
--   'CATEGORICAL_ELIGIBILITY'  -> qualifying for programme A grants categorical eligibility for programme B
--                                 properties: {direction: "a_qualifies_for_b", conditions: "..."}
--   'BUNDLED_WITH'             -> programmes commonly applied for together
--                                 properties: {bundle_priority: 1}
--   'PREREQUISITE_FOR'         -> must apply for A before B
--
-- Organisation-Programme relationships:
--   'ADMINISTERS'              -> organisation administers this programme
--   'PROVIDES_APPLICATION_HELP'-> CBO helps applicants apply for this programme
--
-- Organisation-Jurisdiction relationships:
--   'SERVES_AREA'              -> organisation serves this geographic area
--                                 properties: {service_types: ["food", "housing"], capacity: "available"}
--
-- Referral relationships:
--   'REFERRED_TO'              -> household referred to organisation for a programme
--                                 properties: {referral_id: "uuid", status: "pending", referred_at: "..."}
--
-- Jurisdiction hierarchy:
--   'SUBDIVISION_OF'           -> county is a subdivision of a state, state of country
--
-- Rule dependencies:
--   'DEPENDS_ON'               -> rule evaluation depends on another rule's outcome
--   'OVERRIDES'                -> jurisdiction-specific rule overrides federal default

CREATE INDEX idx_ge_from ON graph_edge(from_node_id);
CREATE INDEX idx_ge_to ON graph_edge(to_node_id);
CREATE INDEX idx_ge_type ON graph_edge(edge_type);
CREATE INDEX idx_ge_from_type ON graph_edge(from_node_id, edge_type);
CREATE INDEX idx_ge_to_type ON graph_edge(to_node_id, edge_type);
CREATE INDEX idx_ge_properties ON graph_edge USING GIN (properties);
CREATE INDEX idx_ge_temporal ON graph_edge(effective_date, expiration_date)
    WHERE effective_date IS NOT NULL;
```

## Programme & Rule Graph Nodes

```sql
-- ============================================================
-- Example: Inserting the programme graph
-- ============================================================

-- Federal programmes as graph nodes
INSERT INTO graph_node (id, node_type, label, properties) VALUES
    ('a1000000-0000-0000-0000-000000000001', 'program', 'SNAP',
     '{"code": "snap", "level": "federal", "income_methodology": "gross_and_net",
       "has_asset_test": true, "administering_agency": "USDA FNS"}'),
    ('a1000000-0000-0000-0000-000000000002', 'program', 'Medicaid (MAGI)',
     '{"code": "medicaid_magi", "level": "federal", "income_methodology": "magi",
       "has_asset_test": false}'),
    ('a1000000-0000-0000-0000-000000000003', 'program', 'CHIP',
     '{"code": "chip", "level": "federal", "income_methodology": "magi",
       "has_asset_test": false}'),
    ('a1000000-0000-0000-0000-000000000004', 'program', 'WIC',
     '{"code": "wic", "level": "federal", "income_methodology": "gross",
       "has_asset_test": false}'),
    ('a1000000-0000-0000-0000-000000000005', 'program', 'SSI',
     '{"code": "ssi", "level": "federal", "income_methodology": "ssi_methodology",
       "has_asset_test": true}'),
    ('a1000000-0000-0000-0000-000000000006', 'program', 'EITC',
     '{"code": "eitc", "level": "federal", "income_methodology": "magi",
       "has_asset_test": false}'),
    ('a1000000-0000-0000-0000-000000000007', 'program', 'LIHEAP',
     '{"code": "liheap", "level": "federal", "income_methodology": "gross",
       "has_asset_test": false}');

-- Categorical eligibility edges
INSERT INTO graph_edge (from_node_id, to_node_id, edge_type, properties) VALUES
    -- Receiving SSI makes you categorically eligible for SNAP
    ('a1000000-0000-0000-0000-000000000005', 'a1000000-0000-0000-0000-000000000001',
     'CATEGORICAL_ELIGIBILITY',
     '{"direction": "receiving_a_qualifies_for_b", "bypasses_rules": ["gross_income_test", "net_income_test", "asset_test"]}'),
    -- Receiving SNAP makes you categorically eligible for free school meals (if added)
    -- Receiving Medicaid makes you categorically eligible for WIC adjunctive eligibility
    ('a1000000-0000-0000-0000-000000000002', 'a1000000-0000-0000-0000-000000000004',
     'CATEGORICAL_ELIGIBILITY',
     '{"direction": "receiving_a_qualifies_for_b", "condition": "pregnant_or_postpartum_or_child_under_5"}');

-- Jurisdiction hierarchy as graph
INSERT INTO graph_node (id, node_type, label, properties) VALUES
    ('b1000000-0000-0000-0000-000000000001', 'jurisdiction', 'United States',
     '{"code": "US", "level": "country", "iso_3166_1": "US"}'),
    ('b1000000-0000-0000-0000-000000000002', 'jurisdiction', 'California',
     '{"code": "US-CA", "level": "state", "iso_3166_2": "US-CA", "fips": "06", "medicaid_expansion": true}'),
    ('b1000000-0000-0000-0000-000000000003', 'jurisdiction', 'Los Angeles County',
     '{"code": "US-CA-037", "level": "county", "fips": "06037"}');

INSERT INTO graph_edge (from_node_id, to_node_id, edge_type) VALUES
    ('b1000000-0000-0000-0000-000000000002', 'b1000000-0000-0000-0000-000000000001', 'SUBDIVISION_OF'),
    ('b1000000-0000-0000-0000-000000000003', 'b1000000-0000-0000-0000-000000000002', 'SUBDIVISION_OF');

-- Programmes available in jurisdictions
INSERT INTO graph_edge (from_node_id, to_node_id, edge_type, properties) VALUES
    ('a1000000-0000-0000-0000-000000000001', 'b1000000-0000-0000-0000-000000000002',
     'AVAILABLE_IN',
     '{"local_name": "CalFresh", "application_url": "https://benefitscal.com/"}');
```

## Multi-Tenancy & RBAC

```sql
-- ============================================================
-- TENANTS (also graph nodes for referral network queries)
-- ============================================================
CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    tenant_type     TEXT NOT NULL,
    jurisdiction_code TEXT,
    subdomain       TEXT UNIQUE,
    graph_node_id   UUID REFERENCES graph_node(id),    -- link to the graph for network queries
    is_active       BOOLEAN DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- USERS & ROLES
-- ============================================================
CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    email           TEXT NOT NULL UNIQUE,
    display_name    TEXT NOT NULL,
    auth_provider   TEXT DEFAULT 'login_gov',
    auth_subject_id TEXT,
    is_active       BOOLEAN DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE role (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL UNIQUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE user_role (
    user_id         UUID NOT NULL REFERENCES app_user(id) ON DELETE CASCADE,
    role_id         UUID NOT NULL REFERENCES role(id) ON DELETE CASCADE,
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    PRIMARY KEY (user_id, role_id, tenant_id)
);

-- ============================================================
-- AUDIT LOG
-- ============================================================
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID,
    user_id         UUID,
    entity_type     TEXT NOT NULL,
    entity_id       UUID NOT NULL,
    action          TEXT NOT NULL,
    changes         JSONB,
    context         JSONB DEFAULT '{}',
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_occurred ON audit_log(occurred_at);
```

## Rule Parameters (Relational for Computation)

```sql
-- ============================================================
-- RULE PARAMETERS (relational for fast computation)
-- Rules themselves are graph nodes; parameters are relational for efficient calculation
-- ============================================================
CREATE TABLE rule_parameter (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    program_code    TEXT NOT NULL,
    jurisdiction_code TEXT NOT NULL,
    rule_code       TEXT NOT NULL,
    parameter_name  TEXT NOT NULL,
    parameter_value NUMERIC(14,4) NOT NULL,
    household_size  INTEGER,
    effective_date  DATE NOT NULL,
    expiration_date DATE,
    source_citation TEXT,
    rule_node_id    UUID REFERENCES graph_node(id),    -- link to rule in graph for dependency traversal
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rp_program ON rule_parameter(program_code, jurisdiction_code);
CREATE INDEX idx_rp_rule ON rule_parameter(rule_code);
CREATE INDEX idx_rp_effective ON rule_parameter(effective_date, expiration_date);

-- ============================================================
-- REFERENCE DATA
-- ============================================================
CREATE TABLE reference_data (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    data_type       TEXT NOT NULL,
    fiscal_year     INTEGER NOT NULL,
    jurisdiction_code TEXT DEFAULT 'US',
    data            JSONB NOT NULL,
    effective_date  DATE NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(data_type, fiscal_year, jurisdiction_code)
);
```

## Example Graph Queries

```sql
-- ============================================================
-- QUERY 1: Cross-programme bundling via categorical eligibility
-- "If household X qualifies for SSI, what other programmes do they
--  automatically qualify for (transitively)?"
-- ============================================================
WITH RECURSIVE eligible_programs AS (
    -- Start with the programme the household already qualifies for
    SELECT
        gn.id AS program_node_id,
        gn.label AS program_name,
        gn.properties->>'code' AS program_code,
        0 AS depth,
        ARRAY[gn.id] AS path
    FROM graph_node gn
    WHERE gn.node_type = 'program'
      AND gn.properties->>'code' = 'ssi'

    UNION ALL

    -- Traverse CATEGORICAL_ELIGIBILITY edges
    SELECT
        target.id,
        target.label,
        target.properties->>'code',
        ep.depth + 1,
        ep.path || target.id
    FROM eligible_programs ep
    JOIN graph_edge ge ON ge.from_node_id = ep.program_node_id
        AND ge.edge_type = 'CATEGORICAL_ELIGIBILITY'
        AND ge.is_active = TRUE
    JOIN graph_node target ON target.id = ge.to_node_id
    WHERE NOT (target.id = ANY(ep.path))  -- prevent cycles
      AND ep.depth < 5                      -- limit traversal depth
)
SELECT DISTINCT program_code, program_name, depth
FROM eligible_programs
ORDER BY depth;

-- Result: ssi (depth 0), snap (depth 1), ...

-- ============================================================
-- QUERY 2: Find all programmes available in a household's location
-- (traversing jurisdiction hierarchy)
-- ============================================================
WITH RECURSIVE jurisdiction_chain AS (
    -- Start with the household's county
    SELECT gn.id AS jurisdiction_id, gn.label, gn.properties->>'code' AS code, 0 AS depth
    FROM graph_node gn
    WHERE gn.node_type = 'jurisdiction'
      AND gn.properties->>'code' = 'US-CA-037'  -- LA County

    UNION ALL

    -- Walk up the jurisdiction hierarchy
    SELECT parent.id, parent.label, parent.properties->>'code', jc.depth + 1
    FROM jurisdiction_chain jc
    JOIN graph_edge ge ON ge.from_node_id = jc.jurisdiction_id
        AND ge.edge_type = 'SUBDIVISION_OF'
    JOIN graph_node parent ON parent.id = ge.to_node_id
)
SELECT DISTINCT
    prog.label AS program_name,
    prog.properties->>'code' AS program_code,
    ge_avail.properties->>'local_name' AS local_name,
    jc.label AS jurisdiction_level
FROM jurisdiction_chain jc
JOIN graph_edge ge_avail ON ge_avail.to_node_id = jc.jurisdiction_id
    AND ge_avail.edge_type = 'AVAILABLE_IN'
    AND ge_avail.is_active = TRUE
JOIN graph_node prog ON prog.id = ge_avail.from_node_id
    AND prog.node_type = 'program'
ORDER BY program_name;

-- Result: All federal, California, and LA County programmes

-- ============================================================
-- QUERY 3: Referral network -- find CBOs that serve this area
-- and provide food assistance
-- ============================================================
SELECT
    org.label AS organisation_name,
    org.properties->>'website' AS website,
    ge.properties->>'capacity' AS capacity,
    ge.properties->'service_types' AS services
FROM graph_node org
JOIN graph_edge ge ON ge.from_node_id = org.id
    AND ge.edge_type = 'SERVES_AREA'
    AND ge.is_active = TRUE
JOIN graph_node jurisdiction ON jurisdiction.id = ge.to_node_id
    AND jurisdiction.properties->>'code' = 'US-CA-037'
WHERE org.node_type = 'organisation'
  AND ge.properties->'service_types' @> '"food"'::JSONB;

-- ============================================================
-- QUERY 4: Multiple household compositions for one person
-- "Show all household groupings for person X (physical, SNAP, MAGI)"
-- ============================================================
SELECT
    hh.label AS household_label,
    ge.properties->>'household_type' AS household_type,
    ge.properties->>'relationship' AS relationship
FROM graph_node person_node
JOIN graph_edge ge ON ge.from_node_id = person_node.id
    AND ge.edge_type = 'MEMBER_OF'
    AND ge.is_active = TRUE
JOIN graph_node hh ON hh.id = ge.to_node_id
    AND hh.node_type = 'household'
WHERE person_node.entity_table = 'person'
  AND person_node.entity_id = '550e8400-e29b-41d4-a716-446655440000';

-- Result:
-- "Smith Physical Household" | "physical"   | "head"
-- "Smith SNAP Household"     | "snap"        | "head"
-- "Smith MAGI Tax Unit"      | "magi_tax_unit"| "filer"
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity (Person) | 1 | person (households are graph nodes) |
| Financial | 1 | income |
| Graph Layer | 2 | graph_node, graph_edge |
| Screening & Results | 2 | screening_session, eligibility_result |
| Rule Parameters | 2 | rule_parameter, reference_data |
| Documents | 1 | document |
| Multi-Tenancy & RBAC | 4 | tenant, app_user, role, user_role |
| Audit | 1 | audit_log |
| **Total** | **14** | Fewer tables because households, programmes, jurisdictions, and organisations live in the graph |

---

## Key Design Decisions

1. **Households, programmes, jurisdictions, and organisations are graph nodes, not relational tables.** This is the most distinctive choice. Instead of separate `household`, `benefit_program`, `jurisdiction`, and `organisation` tables, these entities are rows in `graph_node` with type-specific properties in JSONB. This unifies the entity model and makes any-to-any relationship queries trivial via edge traversal.

2. **Persons remain relational because they carry structured PII.** Person data (name, DOB, SSN hash, citizenship status) benefits from column-level constraints, indexing, and encryption-at-rest controls that are easier to manage in a typed relational table than in JSONB properties. The `graph_node` table has an `entity_table`/`entity_id` pair that links person nodes back to the `person` table.

3. **Multiple household compositions modelled as parallel MEMBER_OF edges.** A single person can be a member of a "physical household", a "SNAP household", and a "MAGI tax unit" simultaneously, each as a separate `MEMBER_OF` edge with a `household_type` property. This directly solves the problem that SNAP, Medicaid MAGI, and physical households define "household" differently.

4. **Categorical eligibility as `CATEGORICAL_ELIGIBILITY` edges between programme nodes.** When qualifying for SSI makes a person categorically eligible for SNAP, this relationship is an edge, not a hardcoded rule. The eligibility engine traverses these edges recursively to discover all downstream programmes a household qualifies for. Adding new categorical eligibility relationships is an INSERT, not a code change.

5. **Temporal edges via `effective_date` / `expiration_date`.** Relationships change over time: a person may leave a household, a programme may become available in a new jurisdiction, an organisation may stop serving an area. Temporal edge columns handle this without separate history tables.

6. **Rule parameters remain relational for computational efficiency.** While rules are graph nodes (enabling dependency traversal), their numeric parameters (income thresholds, FPL percentages) live in a relational `rule_parameter` table for fast indexed lookups during eligibility computation. The `rule_node_id` column links each parameter set to its graph node.

7. **Community resource network is a first-class graph citizen.** Organisations (CBOs, agencies, health systems) are graph nodes with `SERVES_AREA` edges to jurisdiction nodes and `PROVIDES_APPLICATION_HELP` edges to programme nodes. This makes referral routing a graph query: "find organisations that serve this ZIP code and help with SNAP applications." This directly supports the closed-loop referral integration feature.

8. **PostgreSQL-native graph via recursive CTEs rather than a separate graph database.** The `graph_node`/`graph_edge` tables with recursive CTEs keep everything in one database, avoiding the operational complexity of dual-database deployments. For production deployments with very large graphs (millions of nodes), the schema is designed to be compatible with Neo4j migration: nodes and edges can be bulk-exported to Neo4j's CSV import format.

9. **`triggered_by` column on eligibility results tracks categorical eligibility chains.** When a household qualifies for SNAP because they qualified for SSI, the SNAP eligibility result's `triggered_by` column references the SSI result. This creates an auditable chain of reasoning that explains the full cross-programme bundling logic to the user.
