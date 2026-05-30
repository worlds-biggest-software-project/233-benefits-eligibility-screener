# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: Benefits Eligibility Screener · Created: 2026-05-22

## Philosophy

This model treats every state change as an immutable event appended to an event store. The event stream is the single source of truth; all queryable state is derived by replaying or projecting events into materialised read models. The Command-Query Responsibility Segregation (CQRS) pattern separates the write path (commands produce events) from the read path (projections serve queries).

This architecture is a natural fit for benefits eligibility because government systems require complete audit trails, temporal queries ("what was this household's eligibility status on March 15th?"), and the ability to re-evaluate past determinations when rules change retroactively. Healthcare systems, financial trading platforms, and government compliance systems have adopted event sourcing precisely because regulators demand that no data is ever lost or silently mutated. The OECD Rules-as-Code initiative emphasises traceability of policy decisions, which event sourcing delivers by design.

The model uses PostgreSQL as both the event store and the projection database, avoiding the operational complexity of a separate event-streaming infrastructure. Events are stored in a small number of append-only tables, while read-optimised projections are maintained in standard relational tables that can be rebuilt at any time from the event stream.

**Best for:** Deployments where full audit trails, temporal queries, regulatory explainability, and the ability to retroactively re-evaluate eligibility under changed rules are top priorities.

**Trade-offs:**
- (+) Complete, immutable audit trail by construction -- no separate audit log needed
- (+) Temporal queries ("what was true on date X?") are trivial: replay events up to that date
- (+) Re-evaluation: when rules change retroactively, replay events through the new rule version
- (+) Natural fit for AI analytics on change patterns and eligibility trajectories
- (+) Projections can be rebuilt or new projections added without changing the source of truth
- (-) Higher write complexity: every mutation is an event, not a simple UPDATE
- (-) Eventual consistency between event store and projections requires careful design
- (-) Projection rebuild can be slow for large event volumes without snapshotting
- (-) Developers must learn event-sourcing patterns; less familiar than CRUD
- (-) Reporting queries run against projections, not the event store directly

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ACA MAGI Rules (45 CFR Part 155) | MAGI calculations are captured as `IncomeCalculated` events with full breakdown of AGI components; historical recalculation via event replay |
| Social Security Act Title XIX/XXI | Program eligibility events reference specific rule versions, enabling audit of which statutory interpretation was applied |
| OECD Rules-as-Code Framework | Event sourcing directly implements the RaC principle of traceable, reproducible policy decisions |
| HIPAA | PII is stored only in identity events with encryption; projections can be built with or without PII depending on access level |
| FHIR SDOH Clinical Care IG | Referral lifecycle events map to FHIR Task status transitions (requested -> accepted -> in-progress -> completed) |
| OpenFisca Variable Model | Rule evaluation events capture the equivalent of OpenFisca variable calculations with parameter snapshots |
| NIST SP 800-53 Rev 5 (AU controls) | Event store inherently satisfies AU-3 (content of audit records), AU-9 (protection of audit information), and AU-11 (audit record retention) |

---

## Event Store Tables

```sql
-- ============================================================
-- EVENT STORE: The single source of truth
-- ============================================================
CREATE TABLE event_store (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id       UUID NOT NULL,                    -- aggregate root ID (household, person, session, etc.)
    stream_type     TEXT NOT NULL,                     -- 'household', 'person', 'screening_session', 'program'
    event_type      TEXT NOT NULL,                     -- e.g. 'HouseholdCreated', 'PersonAdded', 'IncomeReported',
                                                       -- 'EligibilityDetermined', 'RuleParameterUpdated'
    event_version   INTEGER NOT NULL,                  -- sequence number within the stream (optimistic concurrency)
    event_data      JSONB NOT NULL,                    -- the event payload
    metadata        JSONB DEFAULT '{}',                -- tenant_id, user_id, correlation_id, causation_id, ip_address
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    -- Ensure ordered, gap-free event sequences per stream
    UNIQUE(stream_id, event_version)
);

-- Primary query pattern: read all events for a stream in order
CREATE INDEX idx_es_stream ON event_store(stream_id, event_version);

-- Query by event type for projections and analytics
CREATE INDEX idx_es_event_type ON event_store(event_type, occurred_at);

-- Query by time range for temporal replay
CREATE INDEX idx_es_occurred ON event_store(occurred_at);

-- Multi-tenant filtering via metadata
CREATE INDEX idx_es_tenant ON event_store((metadata->>'tenant_id'), occurred_at);

-- GIN index on event_data for ad-hoc JSONB queries
CREATE INDEX idx_es_data ON event_store USING GIN (event_data);

-- ============================================================
-- SNAPSHOTS (optional: accelerate projection rebuilds)
-- ============================================================
CREATE TABLE event_snapshot (
    snapshot_id     UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id       UUID NOT NULL,
    stream_type     TEXT NOT NULL,
    snapshot_version INTEGER NOT NULL,                 -- event_version at snapshot time
    snapshot_data   JSONB NOT NULL,                    -- serialised aggregate state
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(stream_id, snapshot_version)
);

CREATE INDEX idx_snap_stream ON event_snapshot(stream_id, snapshot_version DESC);
```

## Event Type Catalogue

```sql
-- ============================================================
-- EVENT TYPE REGISTRY (documentation and schema validation)
-- ============================================================
CREATE TABLE event_type_registry (
    event_type      TEXT PRIMARY KEY,
    category        TEXT NOT NULL,                     -- 'identity', 'financial', 'screening', 'determination',
                                                       -- 'document', 'referral', 'policy', 'system'
    description     TEXT NOT NULL,
    json_schema     JSONB,                             -- JSON Schema for validating event_data payloads
    version         INTEGER DEFAULT 1,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Example event types:
-- Identity Events
--   'PersonCreated'         -> {first_name, last_name, date_of_birth, citizenship_status, ...}
--   'PersonUpdated'         -> {field_name, old_value, new_value}
--   'HouseholdCreated'      -> {address, state_code, county_fips, housing_type, ...}
--   'HouseholdMemberAdded'  -> {person_id, relationship_to_head, is_tax_filer, ...}
--   'HouseholdMemberRemoved'-> {person_id, reason}
--
-- Financial Events
--   'IncomeReported'        -> {person_id, income_type, gross_amount, period, employer_name, ...}
--   'IncomeVerified'        -> {income_source_id, verification_method, verified_amount}
--   'IncomeEnded'           -> {income_source_id, end_date, reason}
--   'DeductionReported'     -> {person_id, deduction_type, amount, applicable_program}
--   'AssetReported'         -> {person_id, asset_type, current_value, is_countable}
--   'MAGICalculated'        -> {household_id, magi_amount, components: [{type, amount}, ...]}
--
-- Screening Events
--   'ScreeningStarted'      -> {household_id, channel, locale, tenant_id}
--   'QuestionAnswered'      -> {question_code, response_value, response_type}
--   'ScreeningCompleted'    -> {total_questions, duration_seconds}
--   'ScreeningAbandoned'    -> {last_question_code, reason}
--
-- Determination Events
--   'EligibilityDetermined' -> {program_id, status, confidence_score, estimated_benefit,
--                                rules_version, rule_results: [{rule_code, result, ...}, ...]}
--   'DeterminationOverridden' -> {determination_id, new_status, reason, caseworker_id}
--
-- Document Events
--   'DocumentUploaded'      -> {document_type, file_name, storage_path}
--   'DocumentExtractionCompleted' -> {document_id, extracted_fields: {...}}
--   'DocumentExtractionFailed'    -> {document_id, error_message}
--
-- Referral Events
--   'ReferralCreated'       -> {program_id, receiving_org, referral_type}
--   'ReferralAccepted'      -> {referral_id, accepted_by}
--   'ReferralCompleted'     -> {referral_id, outcome}
--   'ReferralDeclined'      -> {referral_id, reason}
--
-- Policy Events
--   'RuleCreated'           -> {program_id, rule_code, rule_type, statutory_citation}
--   'RuleParameterUpdated'  -> {rule_id, parameter_name, old_value, new_value, effective_date}
--   'FPLThresholdUpdated'   -> {fiscal_year, household_size, amount, region}
--   'ProgramActivated'      -> {program_id, jurisdiction_code}
--   'ProgramDeactivated'    -> {program_id, jurisdiction_code, reason}
```

## Read-Model Projections

```sql
-- ============================================================
-- PROJECTION: Current Household State
-- (Rebuilt by replaying identity and financial events)
-- ============================================================
CREATE TABLE proj_household (
    household_id    UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    head_person_id  UUID,
    member_count    INTEGER NOT NULL DEFAULT 1,
    address_line1   TEXT,
    city            TEXT,
    state_code      TEXT,
    county_fips     TEXT,
    zip_code        TEXT,
    housing_type    TEXT,
    total_gross_monthly_income NUMERIC(12,2) DEFAULT 0,
    total_countable_assets NUMERIC(14,2) DEFAULT 0,
    magi_annual     NUMERIC(12,2),
    last_screened_at TIMESTAMPTZ,
    last_event_version INTEGER NOT NULL,              -- tracks which event this projection is current to
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_proj_hh_tenant ON proj_household(tenant_id);
CREATE INDEX idx_proj_hh_state ON proj_household(state_code);

-- ============================================================
-- PROJECTION: Current Person State
-- ============================================================
CREATE TABLE proj_person (
    person_id       UUID PRIMARY KEY,
    first_name      TEXT,
    last_name       TEXT,
    date_of_birth   DATE,
    age             INTEGER,                          -- computed from DOB
    citizenship_status TEXT,
    disability_status BOOLEAN,
    veteran_status  BOOLEAN,
    pregnant        BOOLEAN,
    student_status  TEXT,
    primary_language TEXT,
    last_event_version INTEGER NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- PROJECTION: Current Household Members
-- ============================================================
CREATE TABLE proj_household_member (
    household_id    UUID NOT NULL,
    person_id       UUID NOT NULL,
    relationship_to_head TEXT,
    is_tax_filer    BOOLEAN,
    is_tax_dependent BOOLEAN,
    PRIMARY KEY (household_id, person_id)
);

-- ============================================================
-- PROJECTION: Latest Eligibility Results per Household-Program
-- ============================================================
CREATE TABLE proj_eligibility (
    household_id    UUID NOT NULL,
    program_code    TEXT NOT NULL,
    program_name    TEXT,
    determination_status TEXT NOT NULL,
    confidence_score NUMERIC(3,2),
    estimated_monthly_benefit NUMERIC(10,2),
    estimated_annual_benefit NUMERIC(12,2),
    explanation_text TEXT,
    rules_version   TEXT,
    determined_at   TIMESTAMPTZ NOT NULL,
    session_id      UUID,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (household_id, program_code)
);

CREATE INDEX idx_proj_elig_status ON proj_eligibility(determination_status);

-- ============================================================
-- PROJECTION: Screening Session Summary
-- ============================================================
CREATE TABLE proj_screening_session (
    session_id      UUID PRIMARY KEY,
    household_id    UUID NOT NULL,
    tenant_id       UUID NOT NULL,
    channel         TEXT,
    locale          TEXT,
    status          TEXT,
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    question_count  INTEGER DEFAULT 0,
    programs_eligible INTEGER DEFAULT 0,
    programs_ineligible INTEGER DEFAULT 0,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_proj_ss_tenant ON proj_screening_session(tenant_id);
CREATE INDEX idx_proj_ss_household ON proj_screening_session(household_id);
CREATE INDEX idx_proj_ss_started ON proj_screening_session(started_at);

-- ============================================================
-- PROJECTION: Active Referrals
-- ============================================================
CREATE TABLE proj_referral (
    referral_id     UUID PRIMARY KEY,
    household_id    UUID NOT NULL,
    program_code    TEXT,
    receiving_organisation TEXT,
    referral_type   TEXT,
    status          TEXT NOT NULL,
    referred_at     TIMESTAMPTZ,
    last_status_change TIMESTAMPTZ,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_proj_ref_household ON proj_referral(household_id);
CREATE INDEX idx_proj_ref_status ON proj_referral(status);

-- ============================================================
-- PROJECTION: Rule Parameter Timeline (for policy analysis)
-- ============================================================
CREATE TABLE proj_rule_parameter_timeline (
    program_code    TEXT NOT NULL,
    rule_code       TEXT NOT NULL,
    parameter_name  TEXT NOT NULL,
    parameter_value NUMERIC(14,4) NOT NULL,
    household_size  INTEGER,
    effective_date  DATE NOT NULL,
    expiration_date DATE,
    source_citation TEXT,
    event_id        UUID NOT NULL,                    -- which event set this parameter
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_proj_rpt_program ON proj_rule_parameter_timeline(program_code, rule_code);
CREATE INDEX idx_proj_rpt_effective ON proj_rule_parameter_timeline(effective_date);
```

## Multi-Tenancy & RBAC (Command-Side)

```sql
-- ============================================================
-- TENANTS (command-side reference data)
-- ============================================================
CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    tenant_type     TEXT NOT NULL,
    jurisdiction_code TEXT,
    subdomain       TEXT UNIQUE,
    is_active       BOOLEAN DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- USERS & ROLES (command-side, minimal)
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
```

## Example: Temporal Query via Event Replay

```sql
-- "What was household X's eligibility for SNAP on March 15, 2026?"
-- Replay all events for the household up to that date, then evaluate.

-- Step 1: Get all events for the household stream up to the target date
SELECT event_type, event_data, occurred_at
FROM event_store
WHERE stream_id = '550e8400-e29b-41d4-a716-446655440000'  -- household UUID
  AND stream_type = 'household'
  AND occurred_at <= '2026-03-15T23:59:59Z'
ORDER BY event_version ASC;

-- Step 2: Also get the screening session events linked to this household
SELECT es.event_type, es.event_data, es.occurred_at
FROM event_store es
WHERE es.stream_type = 'screening_session'
  AND es.event_data->>'household_id' = '550e8400-e29b-41d4-a716-446655440000'
  AND es.occurred_at <= '2026-03-15T23:59:59Z'
ORDER BY es.event_version ASC;

-- The application layer replays these events to reconstruct the household state
-- at that point in time and re-runs the eligibility rules that were effective
-- on March 15, 2026 (using RuleParameterUpdated events to determine the correct thresholds).
```

## Example: Re-evaluation When Rules Change

```sql
-- When FPL thresholds are updated retroactively, find all households
-- that were determined under the old thresholds and need re-evaluation.

-- Step 1: Find the rule parameter change event
SELECT event_id, event_data, occurred_at
FROM event_store
WHERE event_type = 'FPLThresholdUpdated'
  AND event_data->>'fiscal_year' = '2026'
ORDER BY occurred_at DESC
LIMIT 1;

-- Step 2: Find all households with determinations after the effective date
-- of the old threshold but before the update was applied
SELECT DISTINCT es.event_data->>'household_id' AS household_id
FROM event_store es
WHERE es.event_type = 'EligibilityDetermined'
  AND es.event_data->>'program_code' = 'snap'
  AND es.occurred_at >= '2026-01-01'  -- old threshold effective date
  AND es.occurred_at < '2026-04-15';  -- date new threshold was published

-- Step 3: Application re-replays each household's event stream through
-- the updated rule parameters and emits new EligibilityDetermined events
-- with a metadata tag indicating re-evaluation.
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 2 | event_store, event_snapshot |
| Event Registry | 1 | event_type_registry |
| Projections: Identity | 3 | proj_household, proj_person, proj_household_member |
| Projections: Screening | 1 | proj_screening_session |
| Projections: Eligibility | 1 | proj_eligibility |
| Projections: Referrals | 1 | proj_referral |
| Projections: Policy | 1 | proj_rule_parameter_timeline |
| Multi-Tenancy & RBAC | 4 | tenant, app_user, role, user_role |
| **Total** | **14** | 3 core + 7 projections + 4 RBAC |

---

## Key Design Decisions

1. **Single `event_store` table rather than one table per aggregate type.** This simplifies the infrastructure (one table to partition, index, and back up) and allows cross-aggregate queries for analytics. The `stream_type` column enables efficient filtering. For very high volumes, the table can be partitioned by `occurred_at` (monthly or quarterly).

2. **Events carry the full payload, not just diffs.** Each event contains all the information needed to understand what happened without looking up prior events. This makes individual events self-describing for audit purposes and simplifies projection logic. The trade-off is larger event payloads, but storage is cheap relative to the value of complete audit records.

3. **Projections are explicitly labelled as derived data.** Every projection table is prefixed with `proj_` to make it clear these are rebuilt from events. If a projection is corrupted or a new reporting need arises, the projection can be dropped and rebuilt from the event stream. No projection is the source of truth.

4. **Optimistic concurrency via `event_version`.** The UNIQUE constraint on `(stream_id, event_version)` prevents concurrent writes to the same aggregate from producing inconsistent state. The application must read the current version before appending and retry if a conflict occurs.

5. **Temporal queries by design, not by accident.** Event replay to a point in time is the fundamental operation. This eliminates the need for bi-temporal columns (`valid_from`, `valid_to`) on every table, as the temporal dimension is inherent in the event stream.

6. **Event type registry for governance.** The `event_type_registry` table with JSON Schema validation ensures that events conform to a contract. This is essential in a government context where multiple teams may produce events and consumers must rely on consistent structure.

7. **Snapshots are optional performance optimisation.** For households with long histories, rebuilding state from events can be slow. Periodic snapshots capture the aggregate state at a point in time, allowing replay to start from the snapshot rather than from the first event. Snapshots are never the source of truth.

8. **RBAC tables are command-side, not event-sourced.** User roles and permissions are mutable operational state that does not benefit from event sourcing. They use simple relational tables. The event store's `metadata` column records which user and tenant produced each event for audit purposes.

9. **No separate audit_log table needed.** The event store IS the audit log. Every state change is an immutable event with a timestamp, user ID, and full payload. This satisfies NIST SP 800-53 AU controls and HIPAA audit requirements without maintaining a parallel audit infrastructure.
