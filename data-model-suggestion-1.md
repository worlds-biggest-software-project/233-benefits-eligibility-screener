# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Benefits Eligibility Screener · Created: 2026-05-22

## Philosophy

This model follows a fully normalized relational design where every domain concept occupies its own table with strict foreign-key constraints. The schema separates concerns into distinct layers: identity (persons, households), programme catalogue (programs, rules, parameters), screening workflow (sessions, responses, determinations), and integration (documents, referrals). Every relationship is explicit and enforced at the database level.

Normalized relational models are the backbone of government eligibility systems worldwide. IBM Curam, Salesforce Public Sector Solutions, and most state-built Medicaid systems use this pattern because it provides referential integrity, clear audit trails through join tables, and straightforward SQL queries for reporting. The approach mirrors how policy analysts think about benefits: programs have rules, rules have parameters, households have members, and members have attributes that feed into rule evaluation.

This design is best when data integrity and regulatory compliance are the top priorities, when complex cross-entity queries are needed for reporting and analytics, and when the development team has strong relational database expertise.

**Best for:** Agencies and organisations that need maximum data integrity, clear regulatory compliance, and straightforward SQL-based reporting across programs, households, and eligibility determinations.

**Trade-offs:**
- (+) Full referential integrity enforced at the database level
- (+) Straightforward to reason about, audit, and explain to non-technical stakeholders
- (+) Standard SQL tooling for reporting and analytics
- (+) Natural fit for government procurement and compliance requirements
- (-) High table count increases migration and schema-evolution complexity
- (-) Jurisdictional variations require schema changes or proliferation of junction tables
- (-) MAGI income calculation logic lives in application code, not in the schema
- (-) Rule parameter updates require careful schema migration for threshold tables

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ACA MAGI Rules (45 CFR Part 155) | Income calculation tables model MAGI components (AGI adjustments, tax-filing status, household composition) as separate entities with typed income sources |
| Social Security Act Title XIX/XXI | Program and rule tables encode Medicaid/CHIP eligibility pathways (MAGI vs. non-MAGI) as distinct rule sets with separate parameter tables |
| Federal Poverty Level (FPL) | Dedicated `fpl_threshold` table stores annual FPL values by household size, referenced by rules that use percentage-of-FPL tests |
| OpenFisca Variable Model | Variable/parameter separation mirrors OpenFisca's distinction between input variables (person attributes) and parameters (policy thresholds) |
| FHIR SDOH Clinical Care IG | Referral and screening tables map to FHIR ServiceRequest, QuestionnaireResponse, and Observation profiles |
| WCAG 2.2 / Section 508 | Not directly schema-relevant, but the `locale` and `accessibility_preference` fields on screening sessions support compliance tracking |
| ISO 3166-1/2 | Jurisdiction codes use ISO 3166 country and subdivision codes for state/county identification |
| HIPAA | PII fields are identified in schema comments; encryption-at-rest and access logging are application-layer concerns informed by the schema design |

---

## Core Identity Tables

```sql
-- ============================================================
-- PERSONS
-- ============================================================
CREATE TABLE person (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    external_id     TEXT,                            -- ID from upstream system (Medicaid MIS, etc.)
    first_name      TEXT NOT NULL,
    last_name       TEXT NOT NULL,
    date_of_birth   DATE,
    ssn_hash        TEXT,                            -- HMAC hash of SSN for matching; never store plaintext
    gender          TEXT,                            -- 'male', 'female', 'non_binary', 'prefer_not_to_say'
    citizenship_status TEXT,                         -- 'us_citizen', 'permanent_resident', 'qualified_alien', etc.
    immigration_status TEXT,
    disability_status BOOLEAN DEFAULT FALSE,
    veteran_status  BOOLEAN DEFAULT FALSE,
    pregnant        BOOLEAN DEFAULT FALSE,
    student_status  TEXT,                            -- 'full_time', 'part_time', 'not_student'
    primary_language TEXT DEFAULT 'en',              -- ISO 639-1 language code
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_person_external_id ON person(external_id);
CREATE INDEX idx_person_ssn_hash ON person(ssn_hash);
CREATE INDEX idx_person_dob ON person(date_of_birth);

-- ============================================================
-- HOUSEHOLDS
-- ============================================================
CREATE TABLE household (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT,                            -- optional label, e.g. "Smith Family"
    address_line1   TEXT,
    address_line2   TEXT,
    city            TEXT,
    state_code      TEXT,                            -- ISO 3166-2:US subdivision (e.g. 'US-CA')
    county_fips     TEXT,                            -- 5-digit FIPS code for county-level program matching
    zip_code        TEXT,
    country_code    TEXT DEFAULT 'US',               -- ISO 3166-1 alpha-2
    housing_type    TEXT,                            -- 'rent', 'own', 'shelter', 'homeless', 'other'
    monthly_rent    NUMERIC(10,2),
    monthly_mortgage NUMERIC(10,2),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_household_state ON household(state_code);
CREATE INDEX idx_household_county ON household(county_fips);
CREATE INDEX idx_household_zip ON household(zip_code);

-- ============================================================
-- HOUSEHOLD MEMBERS (junction)
-- ============================================================
CREATE TABLE household_member (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    household_id    UUID NOT NULL REFERENCES household(id) ON DELETE CASCADE,
    person_id       UUID NOT NULL REFERENCES person(id) ON DELETE CASCADE,
    relationship_to_head TEXT NOT NULL,              -- 'head', 'spouse', 'child', 'parent', 'sibling', 'other_relative', 'unrelated'
    is_tax_filer    BOOLEAN DEFAULT FALSE,
    is_tax_dependent BOOLEAN DEFAULT FALSE,
    tax_filing_status TEXT,                          -- 'single', 'married_joint', 'married_separate', 'head_of_household'
    is_head_of_household BOOLEAN DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(household_id, person_id)
);

CREATE INDEX idx_hh_member_household ON household_member(household_id);
CREATE INDEX idx_hh_member_person ON household_member(person_id);
```

## Income & Assets Tables

```sql
-- ============================================================
-- INCOME SOURCES
-- ============================================================
CREATE TABLE income_source (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    person_id       UUID NOT NULL REFERENCES person(id) ON DELETE CASCADE,
    income_type     TEXT NOT NULL,                   -- 'employment', 'self_employment', 'social_security', 'ssi',
                                                     -- 'unemployment', 'child_support', 'alimony', 'pension',
                                                     -- 'investment', 'rental', 'other'
    employer_name   TEXT,
    gross_amount    NUMERIC(12,2) NOT NULL,          -- per-period gross amount
    period          TEXT NOT NULL DEFAULT 'monthly', -- 'weekly', 'biweekly', 'monthly', 'annual'
    is_countable_snap BOOLEAN DEFAULT TRUE,          -- whether counted for SNAP gross/net income test
    is_countable_magi BOOLEAN DEFAULT TRUE,          -- whether counted for MAGI calculation
    start_date      DATE,
    end_date        DATE,                            -- NULL = ongoing
    verified        BOOLEAN DEFAULT FALSE,
    verification_method TEXT,                        -- 'pay_stub', 'tax_return', 'employer_contact', 'self_attestation'
    verification_date DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_income_person ON income_source(person_id);
CREATE INDEX idx_income_type ON income_source(income_type);

-- ============================================================
-- DEDUCTIONS (for SNAP net income, MAGI adjustments)
-- ============================================================
CREATE TABLE deduction (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    person_id       UUID NOT NULL REFERENCES person(id) ON DELETE CASCADE,
    deduction_type  TEXT NOT NULL,                   -- 'standard_deduction', 'earned_income_deduction',
                                                     -- 'dependent_care', 'medical_elderly_disabled',
                                                     -- 'child_support_paid', 'shelter', 'ira_contribution',
                                                     -- 'student_loan_interest', 'hsa_contribution'
    applicable_program TEXT NOT NULL,                -- 'snap', 'magi', 'both'
    amount          NUMERIC(12,2) NOT NULL,
    period          TEXT NOT NULL DEFAULT 'monthly',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_deduction_person ON deduction(person_id);

-- ============================================================
-- ASSETS (for programs with asset tests, e.g. SNAP in non-expanded states)
-- ============================================================
CREATE TABLE asset (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    person_id       UUID NOT NULL REFERENCES person(id) ON DELETE CASCADE,
    asset_type      TEXT NOT NULL,                   -- 'bank_account', 'vehicle', 'real_property',
                                                     -- 'retirement_account', 'investment', 'other'
    description     TEXT,
    current_value   NUMERIC(14,2) NOT NULL,
    is_countable    BOOLEAN DEFAULT TRUE,            -- some assets are excluded (e.g. primary residence)
    verified        BOOLEAN DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_asset_person ON asset(person_id);
```

## Program Catalogue & Rules Tables

```sql
-- ============================================================
-- BENEFIT PROGRAMS
-- ============================================================
CREATE TABLE benefit_program (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            TEXT NOT NULL UNIQUE,             -- 'snap', 'medicaid_magi', 'medicaid_abd', 'chip',
                                                      -- 'ssi', 'eitc', 'ctc', 'wic', 'liheap', 'section8'
    name            TEXT NOT NULL,
    description     TEXT,
    program_level   TEXT NOT NULL,                    -- 'federal', 'state', 'county', 'municipal', 'nonprofit'
    administering_agency TEXT,
    website_url     TEXT,
    application_url TEXT,
    is_active       BOOLEAN DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- PROGRAM JURISDICTIONS (which jurisdictions offer this program)
-- ============================================================
CREATE TABLE program_jurisdiction (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    program_id      UUID NOT NULL REFERENCES benefit_program(id) ON DELETE CASCADE,
    jurisdiction_type TEXT NOT NULL,                  -- 'federal', 'state', 'county'
    jurisdiction_code TEXT NOT NULL,                  -- ISO 3166-2 for states, FIPS for counties, 'US' for federal
    local_name      TEXT,                             -- state-specific name, e.g. 'CalFresh' for SNAP in CA
    local_application_url TEXT,
    is_active       BOOLEAN DEFAULT TRUE,
    effective_date  DATE NOT NULL,
    expiration_date DATE,                             -- NULL = currently active
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(program_id, jurisdiction_code)
);

CREATE INDEX idx_pj_program ON program_jurisdiction(program_id);
CREATE INDEX idx_pj_jurisdiction ON program_jurisdiction(jurisdiction_code);

-- ============================================================
-- ELIGIBILITY RULES
-- ============================================================
CREATE TABLE eligibility_rule (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    program_id      UUID NOT NULL REFERENCES benefit_program(id) ON DELETE CASCADE,
    jurisdiction_code TEXT,                            -- NULL = federal default; specific code = state/county override
    rule_code       TEXT NOT NULL,                     -- 'income_gross_test', 'income_net_test', 'asset_test',
                                                       -- 'age_test', 'citizenship_test', 'categorical_eligibility',
                                                       -- 'household_size_test', 'pregnancy_test', 'disability_test'
    rule_type       TEXT NOT NULL,                     -- 'threshold', 'categorical', 'formula', 'lookup'
    description     TEXT,
    statutory_citation TEXT,                           -- e.g. '7 USC § 2014(c)' for SNAP gross income test
    is_active       BOOLEAN DEFAULT TRUE,
    effective_date  DATE NOT NULL,
    expiration_date DATE,
    priority_order  INTEGER DEFAULT 0,                -- evaluation order within a program
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rule_program ON eligibility_rule(program_id);
CREATE INDEX idx_rule_jurisdiction ON eligibility_rule(jurisdiction_code);
CREATE INDEX idx_rule_effective ON eligibility_rule(effective_date, expiration_date);

-- ============================================================
-- RULE PARAMETERS (thresholds, rates, limits)
-- ============================================================
CREATE TABLE rule_parameter (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    rule_id         UUID NOT NULL REFERENCES eligibility_rule(id) ON DELETE CASCADE,
    parameter_name  TEXT NOT NULL,                     -- 'fpl_percentage', 'max_gross_income', 'asset_limit',
                                                       -- 'min_age', 'max_age', 'deduction_rate'
    parameter_value NUMERIC(14,4) NOT NULL,
    household_size  INTEGER,                           -- NULL if not size-dependent; 1-10+ for size-specific thresholds
    unit            TEXT,                               -- 'percent_fpl', 'usd_monthly', 'usd_annual', 'years', 'count'
    effective_date  DATE NOT NULL,
    expiration_date DATE,
    source_citation TEXT,                              -- e.g. 'FY2026 FPL Table, 91 FR 12345'
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_param_rule ON rule_parameter(rule_id);
CREATE INDEX idx_param_effective ON rule_parameter(effective_date, expiration_date);
CREATE INDEX idx_param_hh_size ON rule_parameter(household_size);

-- ============================================================
-- FEDERAL POVERTY LEVEL REFERENCE TABLE
-- ============================================================
CREATE TABLE fpl_threshold (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    fiscal_year     INTEGER NOT NULL,
    household_size  INTEGER NOT NULL,                  -- 1 through 8+
    annual_amount   NUMERIC(10,2) NOT NULL,            -- base FPL amount
    per_additional  NUMERIC(10,2),                     -- increment for each person above 8
    region          TEXT DEFAULT 'contiguous',          -- 'contiguous', 'alaska', 'hawaii'
    effective_date  DATE NOT NULL,
    source_citation TEXT,                              -- Federal Register citation
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(fiscal_year, household_size, region)
);

CREATE INDEX idx_fpl_year ON fpl_threshold(fiscal_year);
```

## Screening Session & Determination Tables

```sql
-- ============================================================
-- SCREENING SESSIONS
-- ============================================================
CREATE TABLE screening_session (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    household_id    UUID NOT NULL REFERENCES household(id),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),    -- multi-tenant: which agency/CBO
    channel         TEXT NOT NULL DEFAULT 'web',             -- 'web', 'mobile', 'chatbot', 'phone', 'in_person'
    locale          TEXT DEFAULT 'en',                       -- ISO 639-1
    session_status  TEXT NOT NULL DEFAULT 'in_progress',     -- 'in_progress', 'completed', 'abandoned', 'expired'
    started_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ,
    ip_address      INET,
    user_agent      TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_session_household ON screening_session(household_id);
CREATE INDEX idx_session_tenant ON screening_session(tenant_id);
CREATE INDEX idx_session_status ON screening_session(session_status);
CREATE INDEX idx_session_started ON screening_session(started_at);

-- ============================================================
-- SCREENING RESPONSES (individual question-answer pairs)
-- ============================================================
CREATE TABLE screening_response (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id      UUID NOT NULL REFERENCES screening_session(id) ON DELETE CASCADE,
    question_code   TEXT NOT NULL,                            -- 'household_size', 'monthly_income', 'is_pregnant', etc.
    response_value  TEXT NOT NULL,                            -- stringified value; parsed by application layer
    response_type   TEXT NOT NULL,                            -- 'integer', 'decimal', 'boolean', 'text', 'date', 'enum'
    question_order  INTEGER,
    answered_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_response_session ON screening_response(session_id);
CREATE INDEX idx_response_question ON screening_response(question_code);

-- ============================================================
-- ELIGIBILITY DETERMINATIONS
-- ============================================================
CREATE TABLE eligibility_determination (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id      UUID NOT NULL REFERENCES screening_session(id) ON DELETE CASCADE,
    household_id    UUID NOT NULL REFERENCES household(id),
    program_id      UUID NOT NULL REFERENCES benefit_program(id),
    jurisdiction_code TEXT NOT NULL,
    determination_status TEXT NOT NULL,                       -- 'likely_eligible', 'likely_ineligible',
                                                              -- 'needs_verification', 'unable_to_determine'
    confidence_score NUMERIC(3,2),                            -- 0.00 to 1.00
    estimated_monthly_benefit NUMERIC(10,2),
    estimated_annual_benefit NUMERIC(12,2),
    explanation_text TEXT,                                     -- human-readable explanation of determination
    rules_version   TEXT,                                      -- version tag of the rule set used
    determined_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_det_session ON eligibility_determination(session_id);
CREATE INDEX idx_det_household ON eligibility_determination(household_id);
CREATE INDEX idx_det_program ON eligibility_determination(program_id);
CREATE INDEX idx_det_status ON eligibility_determination(determination_status);

-- ============================================================
-- DETERMINATION RULE RESULTS (which rules passed/failed for each determination)
-- ============================================================
CREATE TABLE determination_rule_result (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    determination_id UUID NOT NULL REFERENCES eligibility_determination(id) ON DELETE CASCADE,
    rule_id         UUID NOT NULL REFERENCES eligibility_rule(id),
    result          TEXT NOT NULL,                             -- 'pass', 'fail', 'skip', 'inconclusive'
    input_values    TEXT,                                      -- serialised input values used for this rule
    threshold_used  NUMERIC(14,4),                             -- the parameter value the rule tested against
    actual_value    NUMERIC(14,4),                             -- the household's actual value
    explanation     TEXT,                                       -- "Gross monthly income $2,400 < limit $2,510 for HH size 3"
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_drr_determination ON determination_rule_result(determination_id);
CREATE INDEX idx_drr_rule ON determination_rule_result(rule_id);
```

## Multi-Tenancy & Access Control Tables

```sql
-- ============================================================
-- TENANTS (agencies, CBOs, white-label partners)
-- ============================================================
CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    tenant_type     TEXT NOT NULL,                     -- 'state_agency', 'county_agency', 'nonprofit', 'health_system', 'cbo'
    jurisdiction_code TEXT,                            -- primary jurisdiction served
    subdomain       TEXT UNIQUE,                       -- for white-label: 'lacounty', 'unitedway-chicago', etc.
    branding_config JSONB DEFAULT '{}',                -- logo URL, colours, custom text
    is_active       BOOLEAN DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- USERS (caseworkers, admins, navigators)
-- ============================================================
CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    email           TEXT NOT NULL UNIQUE,
    display_name    TEXT NOT NULL,
    auth_provider   TEXT DEFAULT 'login_gov',          -- 'login_gov', 'saml', 'oidc', 'local'
    auth_subject_id TEXT,                              -- external IdP subject identifier
    is_active       BOOLEAN DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_user_tenant ON app_user(tenant_id);
CREATE INDEX idx_user_auth ON app_user(auth_provider, auth_subject_id);

-- ============================================================
-- ROLES
-- ============================================================
CREATE TABLE role (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL UNIQUE,               -- 'screener_admin', 'caseworker', 'navigator', 'policy_analyst', 'viewer'
    description     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- USER-ROLE ASSIGNMENTS
-- ============================================================
CREATE TABLE user_role (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES app_user(id) ON DELETE CASCADE,
    role_id         UUID NOT NULL REFERENCES role(id) ON DELETE CASCADE,
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    granted_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(user_id, role_id, tenant_id)
);

CREATE INDEX idx_ur_user ON user_role(user_id);
CREATE INDEX idx_ur_role ON user_role(role_id);
```

## Document & Referral Tables

```sql
-- ============================================================
-- DOCUMENTS (uploaded pay stubs, tax returns, ID documents)
-- ============================================================
CREATE TABLE document (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    person_id       UUID REFERENCES person(id),
    household_id    UUID REFERENCES household(id),
    session_id      UUID REFERENCES screening_session(id),
    document_type   TEXT NOT NULL,                     -- 'pay_stub', 'tax_return', 'bank_statement', 'id_document',
                                                       -- 'proof_of_residence', 'disability_documentation', 'other'
    file_name       TEXT NOT NULL,
    file_size_bytes BIGINT,
    mime_type       TEXT,
    storage_path    TEXT NOT NULL,                     -- S3 / Azure Blob path; never store file content in DB
    extraction_status TEXT DEFAULT 'pending',          -- 'pending', 'processing', 'completed', 'failed'
    extracted_data  JSONB,                             -- AI-extracted structured data from the document
    -- Example extracted_data for a pay_stub:
    -- {
    --   "employer_name": "Acme Corp",
    --   "pay_period": {"start": "2026-04-01", "end": "2026-04-15"},
    --   "gross_pay": 2400.00,
    --   "net_pay": 1920.00,
    --   "deductions": [{"type": "federal_tax", "amount": 288.00}, ...]
    -- }
    uploaded_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_doc_person ON document(person_id);
CREATE INDEX idx_doc_session ON document(session_id);
CREATE INDEX idx_doc_type ON document(document_type);

-- ============================================================
-- REFERRALS (closed-loop referrals to CBOs and application assistance)
-- ============================================================
CREATE TABLE referral (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id      UUID REFERENCES screening_session(id),
    household_id    UUID NOT NULL REFERENCES household(id),
    program_id      UUID NOT NULL REFERENCES benefit_program(id),
    referring_tenant_id UUID REFERENCES tenant(id),
    receiving_organisation TEXT NOT NULL,              -- name of CBO or agency
    referral_type   TEXT NOT NULL,                     -- 'application_assistance', 'enrollment_help',
                                                       -- 'document_collection', 'transportation', 'other'
    status          TEXT NOT NULL DEFAULT 'pending',   -- 'pending', 'accepted', 'in_progress', 'completed',
                                                       -- 'declined', 'expired'
    -- Maps to FHIR SDOH ServiceRequest status
    fhir_service_request_id TEXT,                      -- external FHIR resource ID if integrated with EHR
    notes           TEXT,
    referred_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    status_updated_at TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_referral_household ON referral(household_id);
CREATE INDEX idx_referral_program ON referral(program_id);
CREATE INDEX idx_referral_status ON referral(status);

-- ============================================================
-- AUDIT LOG
-- ============================================================
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID REFERENCES tenant(id),
    user_id         UUID REFERENCES app_user(id),
    entity_type     TEXT NOT NULL,                     -- 'person', 'household', 'screening_session', 'determination', etc.
    entity_id       UUID NOT NULL,
    action          TEXT NOT NULL,                     -- 'create', 'read', 'update', 'delete', 'export'
    old_values      JSONB,
    new_values      JSONB,
    ip_address      INET,
    user_agent      TEXT,
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_user ON audit_log(user_id);
CREATE INDEX idx_audit_occurred ON audit_log(occurred_at);
CREATE INDEX idx_audit_tenant ON audit_log(tenant_id);
```

## Row-Level Security (Multi-Tenant Isolation)

```sql
-- Enable RLS on tenant-scoped tables
ALTER TABLE screening_session ENABLE ROW LEVEL SECURITY;
ALTER TABLE app_user ENABLE ROW LEVEL SECURITY;
ALTER TABLE referral ENABLE ROW LEVEL SECURITY;
ALTER TABLE audit_log ENABLE ROW LEVEL SECURITY;

-- Policy: users can only see data belonging to their tenant
CREATE POLICY tenant_isolation_session ON screening_session
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);

CREATE POLICY tenant_isolation_user ON app_user
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);

CREATE POLICY tenant_isolation_referral ON referral
    USING (referring_tenant_id = current_setting('app.current_tenant_id')::UUID);

CREATE POLICY tenant_isolation_audit ON audit_log
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity (Person, Household) | 3 | person, household, household_member |
| Financial (Income, Deductions, Assets) | 3 | income_source, deduction, asset |
| Program Catalogue & Rules | 4 | benefit_program, program_jurisdiction, eligibility_rule, rule_parameter |
| Reference Data | 1 | fpl_threshold |
| Screening & Determination | 4 | screening_session, screening_response, eligibility_determination, determination_rule_result |
| Multi-Tenancy & RBAC | 4 | tenant, app_user, role, user_role |
| Documents & Referrals | 2 | document, referral |
| Audit | 1 | audit_log |
| **Total** | **22** | |

---

## Key Design Decisions

1. **Separate `person` and `household` entities with a junction table** rather than embedding persons in households. This supports the common benefits scenario where a person belongs to multiple households over time (e.g. a child in shared custody) and where MAGI household composition differs from the physical household.

2. **Income sources typed by program countability** (`is_countable_snap`, `is_countable_magi`) because SNAP and Medicaid count income differently. SNAP uses gross and net income tests with specific deductions, while MAGI uses Modified Adjusted Gross Income with different exclusions. The schema makes this explicit at the data level.

3. **Rule parameters are time-bounded** with `effective_date` and `expiration_date` columns. When FPL thresholds change annually or states adjust Medicaid income limits, new parameter rows are inserted with updated dates rather than overwriting existing rows. This preserves the ability to re-evaluate past determinations under the rules that were in effect at the time.

4. **`determination_rule_result` provides rule-level explainability.** Each determination records which rules passed, failed, or were skipped, along with the actual and threshold values. This supports the "why did I qualify/not qualify" explanation required by fair-administration principles and enables the AI explanation generation feature.

5. **Multi-tenant isolation via PostgreSQL RLS** rather than schema-per-tenant. A shared schema with `tenant_id` columns and RLS policies is simpler to maintain and migrate than separate schemas, while still providing strong data isolation suitable for government deployments.

6. **Documents store metadata only; file content lives in object storage.** The `storage_path` column points to S3/Azure Blob. AI-extracted structured data goes in `extracted_data` JSONB, which is the one concession to denormalization in this otherwise fully normalized model.

7. **FHIR integration via reference IDs** (`fhir_service_request_id` on referrals) rather than replicating the full FHIR resource model. The screener is not an EHR; it stores enough to link back to FHIR resources for SDOH Clinical Care IG compliance without duplicating the healthcare data model.

8. **FPL thresholds in a dedicated reference table** rather than hardcoded in application logic. This allows annual updates to be a simple INSERT operation and supports historical lookups for redetermination.
