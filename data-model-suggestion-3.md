# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Benefits Eligibility Screener · Created: 2026-05-22

## Philosophy

This model uses a relational backbone for core identity, screening, and determination entities, but employs PostgreSQL JSONB columns extensively for jurisdiction-specific rule parameters, variable-structure intake data, and programme-specific eligibility criteria. The guiding principle is: **shared structure is relational; variable structure is JSONB**.

Benefits eligibility is inherently variable across jurisdictions. SNAP in California ("CalFresh") has different deduction rules than SNAP in Texas. Medicaid expansion states have entirely different eligibility pathways than non-expansion states. A fully normalized model would need hundreds of junction tables to capture every jurisdictional variant, or it would force all variants into a lowest-common-denominator structure. The hybrid approach avoids both extremes: the core screening workflow and household identity are relational (they are consistent everywhere), while the programme-specific rule definitions, intake questionnaires, and jurisdiction-specific parameters live in JSONB columns with GIN indexes.

This pattern is widely used in modern SaaS platforms that serve diverse customers. Salesforce Public Sector Solutions uses custom fields (effectively a flexible schema) for jurisdiction-specific data. PolicyEngine and OpenFisca encode rules in YAML/JSON rather than relational tables. The hybrid model brings this flexibility into the database itself, enabling rapid MVP development and fast iteration on jurisdiction-specific features without schema migrations.

**Best for:** Rapid MVP development, multi-jurisdiction deployments where programme rules and intake forms vary significantly by state/county, and teams that want to ship quickly while maintaining enough relational structure for integrity and reporting.

**Trade-offs:**
- (+) Dramatically fewer tables than a fully normalized model
- (+) New jurisdictions and programmes can be added without schema migration
- (+) JSONB columns can store jurisdiction-specific fields without ALTER TABLE
- (+) GIN indexes on JSONB enable efficient containment and key-existence queries
- (+) Natural fit for OpenFisca/PolicyEngine-style rule definitions stored as structured JSON
- (+) Rapid iteration: change the JSONB structure, not the schema
- (-) JSONB columns lack foreign-key constraints; referential integrity depends on application validation
- (-) Complex JSONB queries can be less readable than simple JOINs
- (-) Schema validation must be enforced via CHECK constraints, JSON Schema, or application code
- (-) Reporting tools may struggle with nested JSONB structures
- (-) Risk of "JSONB everything" -- discipline needed to keep core fields relational

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ACA MAGI Rules (45 CFR Part 155) | MAGI calculation components stored as structured JSONB in income records; jurisdiction-specific MAGI thresholds in programme rule JSONB |
| Social Security Act Title XIX/XXI | Programme definitions with jurisdiction-specific rule JSONB encode Medicaid pathways (expansion vs. non-expansion) as different JSON structures |
| Federal Poverty Level (FPL) | FPL thresholds stored as JSONB arrays in a reference table, indexed by year and region |
| OpenFisca Variable/Parameter Format | Rule definitions in JSONB mirror OpenFisca's YAML variable structure: variable name, value_type, default_value, formula references |
| PolicyEngine Household Schema | Intake data JSONB follows PolicyEngine's household/person/tax_unit entity structure for API interoperability |
| FHIR SDOH Clinical Care IG | SDOH screening responses stored as JSONB that maps to FHIR QuestionnaireResponse structure |
| ISO 3166-1/2 | Jurisdiction codes use ISO 3166 in relational columns; jurisdiction-specific metadata in JSONB |
| JSON Schema (Draft 2020-12) | JSONB columns reference JSON Schema definitions for validation |

---

## Core Tables

```sql
-- ============================================================
-- PERSONS (relational core)
-- ============================================================
CREATE TABLE person (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    external_id     TEXT,
    first_name      TEXT NOT NULL,
    last_name       TEXT NOT NULL,
    date_of_birth   DATE,
    ssn_hash        TEXT,
    gender          TEXT,
    primary_language TEXT DEFAULT 'en',
    -- Core eligibility-relevant attributes (relational for indexing and joins)
    citizenship_status TEXT,
    disability_status BOOLEAN DEFAULT FALSE,
    veteran_status  BOOLEAN DEFAULT FALSE,
    pregnant        BOOLEAN DEFAULT FALSE,
    student_status  TEXT,
    -- Variable/jurisdiction-specific attributes (JSONB for flexibility)
    extended_attributes JSONB DEFAULT '{}',
    -- Example extended_attributes:
    -- {
    --   "tribal_member": true,
    --   "foster_care_aged_out": false,
    --   "formerly_incarcerated": false,
    --   "work_registration_exempt": true,
    --   "work_requirement_hours_met": 80,
    --   "immigration_detail": {
    --     "visa_type": "U",
    --     "entry_date": "2020-03-15",
    --     "qualified_alien_category": "refugee"
    --   }
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_person_external_id ON person(external_id);
CREATE INDEX idx_person_ssn_hash ON person(ssn_hash);
CREATE INDEX idx_person_dob ON person(date_of_birth);
CREATE INDEX idx_person_extended ON person USING GIN (extended_attributes);

-- ============================================================
-- HOUSEHOLDS (relational core + JSONB for variable housing data)
-- ============================================================
CREATE TABLE household (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    state_code      TEXT NOT NULL,                     -- ISO 3166-2:US (e.g. 'US-CA')
    county_fips     TEXT,                              -- 5-digit FIPS code
    zip_code        TEXT,
    -- Core address fields (relational)
    address_line1   TEXT,
    city            TEXT,
    -- Variable housing and utility data (JSONB)
    housing_data    JSONB DEFAULT '{}',
    -- Example housing_data:
    -- {
    --   "type": "rent",
    --   "monthly_rent": 1500.00,
    --   "utilities_included": false,
    --   "utility_costs": {
    --     "heating": 120.00,
    --     "electric": 85.00,
    --     "water": 40.00,
    --     "phone": 50.00
    --   },
    --   "standard_utility_allowance_eligible": true,
    --   "homeless_shelter": false
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_household_state ON household(state_code);
CREATE INDEX idx_household_county ON household(county_fips);
CREATE INDEX idx_household_zip ON household(zip_code);
CREATE INDEX idx_household_housing ON household USING GIN (housing_data);

-- ============================================================
-- HOUSEHOLD MEMBERS
-- ============================================================
CREATE TABLE household_member (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    household_id    UUID NOT NULL REFERENCES household(id) ON DELETE CASCADE,
    person_id       UUID NOT NULL REFERENCES person(id) ON DELETE CASCADE,
    relationship_to_head TEXT NOT NULL,
    tax_filing      JSONB DEFAULT '{}',
    -- Example tax_filing:
    -- {
    --   "is_filer": true,
    --   "filing_status": "head_of_household",
    --   "dependents_claimed": ["uuid-of-child-1", "uuid-of-child-2"],
    --   "is_dependent_of": null
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(household_id, person_id)
);

CREATE INDEX idx_hm_household ON household_member(household_id);
CREATE INDEX idx_hm_person ON household_member(person_id);
```

## Financial Data (Hybrid)

```sql
-- ============================================================
-- INCOME (relational core + JSONB for variable components)
-- ============================================================
CREATE TABLE income (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    person_id       UUID NOT NULL REFERENCES person(id) ON DELETE CASCADE,
    income_type     TEXT NOT NULL,                     -- 'employment', 'self_employment', 'social_security', etc.
    gross_amount    NUMERIC(12,2) NOT NULL,
    period          TEXT NOT NULL DEFAULT 'monthly',
    start_date      DATE,
    end_date        DATE,
    -- Variable details per income type (JSONB)
    details         JSONB DEFAULT '{}',
    -- Example details for employment:
    -- {
    --   "employer_name": "Acme Corp",
    --   "hours_per_week": 32,
    --   "is_seasonal": false,
    --   "tips_included": true,
    --   "overtime_regular": false
    -- }
    -- Example details for self_employment:
    -- {
    --   "business_name": "Jane's Catering",
    --   "business_type": "sole_proprietor",
    --   "gross_revenue": 4500.00,
    --   "business_expenses": 2100.00,
    --   "net_self_employment": 2400.00
    -- }
    verification    JSONB DEFAULT '{}',
    -- Example verification:
    -- {
    --   "verified": true,
    --   "method": "pay_stub",
    --   "verified_date": "2026-04-20",
    --   "document_id": "uuid-of-uploaded-doc",
    --   "ai_confidence": 0.95
    -- }
    -- Programme-specific countability flags
    countability    JSONB DEFAULT '{}',
    -- Example countability:
    -- {
    --   "snap_gross": true,
    --   "snap_net": true,
    --   "magi": true,
    --   "ssi": false,
    --   "tanf": true
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_income_person ON income(person_id);
CREATE INDEX idx_income_type ON income(income_type);
CREATE INDEX idx_income_details ON income USING GIN (details);
CREATE INDEX idx_income_countability ON income USING GIN (countability);

-- ============================================================
-- EXPENSES & DEDUCTIONS (JSONB-heavy since deduction types vary by programme)
-- ============================================================
CREATE TABLE expense (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    household_id    UUID NOT NULL REFERENCES household(id) ON DELETE CASCADE,
    person_id       UUID REFERENCES person(id),        -- NULL if household-level expense
    expense_type    TEXT NOT NULL,                      -- 'childcare', 'medical', 'shelter', 'child_support_paid',
                                                        -- 'dependent_care', 'transportation'
    monthly_amount  NUMERIC(10,2) NOT NULL,
    -- Programme-specific deduction applicability (JSONB)
    deductibility   JSONB DEFAULT '{}',
    -- Example deductibility:
    -- {
    --   "snap": {"deductible": true, "type": "dependent_care", "capped_at": null},
    --   "magi": {"deductible": false},
    --   "tanf": {"deductible": true, "type": "work_expense"}
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_expense_household ON expense(household_id);
CREATE INDEX idx_expense_type ON expense(expense_type);
CREATE INDEX idx_expense_deductibility ON expense USING GIN (deductibility);
```

## Programme & Rule Definitions (JSONB-First)

```sql
-- ============================================================
-- BENEFIT PROGRAMMES (core metadata relational, rules in JSONB)
-- ============================================================
CREATE TABLE benefit_program (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            TEXT NOT NULL UNIQUE,
    name            TEXT NOT NULL,
    program_level   TEXT NOT NULL,                     -- 'federal', 'state', 'county', 'nonprofit'
    description     TEXT,
    website_url     TEXT,
    application_url TEXT,
    is_active       BOOLEAN DEFAULT TRUE,
    -- Programme metadata and configuration (JSONB)
    config          JSONB DEFAULT '{}',
    -- Example config:
    -- {
    --   "income_methodology": "gross_and_net",  -- or "magi", "ssi_methodology"
    --   "has_asset_test": true,
    --   "categorical_eligibility_programs": ["snap"],
    --   "application_time_estimate_minutes": 30,
    --   "renewal_period_months": 12,
    --   "documents_required": ["proof_of_income", "proof_of_identity", "proof_of_residence"],
    --   "sdoh_categories": ["food_insecurity"],
    --   "fhir_condition_codes": [{"system": "icd-10-cm", "code": "Z59.41"}]
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- PROGRAMME RULES (per-jurisdiction rule sets as JSONB)
-- ============================================================
CREATE TABLE program_rule_set (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    program_id      UUID NOT NULL REFERENCES benefit_program(id) ON DELETE CASCADE,
    jurisdiction_code TEXT NOT NULL,                    -- 'US' for federal, 'US-CA' for California, 'US-CA-037' for LA County
    version         INTEGER NOT NULL DEFAULT 1,
    is_active       BOOLEAN DEFAULT TRUE,
    effective_date  DATE NOT NULL,
    expiration_date DATE,
    -- The complete rule set as structured JSONB
    rules           JSONB NOT NULL,
    -- Example rules for SNAP in California:
    -- {
    --   "eligibility_rules": [
    --     {
    --       "rule_code": "gross_income_test",
    --       "rule_type": "threshold",
    --       "description": "Gross monthly income must be at or below 200% FPL (CalFresh BBCE)",
    --       "statutory_citation": "California WIC § 18901.5",
    --       "test": {
    --         "field": "household_gross_monthly_income",
    --         "operator": "<=",
    --         "threshold_type": "fpl_percentage",
    --         "threshold_value": 200,
    --         "thresholds_by_household_size": {
    --           "1": 2510, "2": 3407, "3": 4303, "4": 5200,
    --           "5": 6097, "6": 6993, "7": 7890, "8": 8787,
    --           "each_additional": 897
    --         }
    --       }
    --     },
    --     {
    --       "rule_code": "net_income_test",
    --       "rule_type": "threshold",
    --       "description": "Net monthly income at or below 100% FPL",
    --       "test": {
    --         "field": "household_net_monthly_income",
    --         "operator": "<=",
    --         "threshold_type": "fpl_percentage",
    --         "threshold_value": 100,
    --         "thresholds_by_household_size": {
    --           "1": 1255, "2": 1704, "3": 2152, "4": 2600,
    --           "5": 3049, "6": 3497, "7": 3945, "8": 4394,
    --           "each_additional": 449
    --         }
    --       }
    --     },
    --     {
    --       "rule_code": "categorical_eligibility",
    --       "rule_type": "categorical",
    --       "description": "BBCE: households receiving TANF/SSI are categorically eligible",
    --       "test": {
    --         "any_member_receives": ["tanf", "ssi", "ga"]
    --       }
    --     },
    --     {
    --       "rule_code": "citizenship_test",
    --       "rule_type": "categorical",
    --       "description": "Must be US citizen, US national, or qualified alien",
    --       "test": {
    --         "field": "citizenship_status",
    --         "operator": "in",
    --         "values": ["us_citizen", "us_national", "permanent_resident", "refugee", "asylee"]
    --       }
    --     }
    --   ],
    --   "benefit_calculation": {
    --     "method": "snap_allotment_table",
    --     "max_allotment_by_hh_size": {
    --       "1": 292, "2": 536, "3": 768, "4": 975,
    --       "5": 1158, "6": 1390, "7": 1536, "8": 1756
    --     },
    --     "formula": "max_allotment - (0.30 * net_income)"
    --   },
    --   "deductions": [
    --     {"code": "standard", "amount": 198, "applies_to": "all"},
    --     {"code": "earned_income", "rate": 0.20, "applies_to": "earned_income"},
    --     {"code": "dependent_care", "max": 200, "max_child_under_2": 200, "applies_to": "dependents"},
    --     {"code": "medical_elderly_disabled", "threshold": 35, "applies_to": "elderly_disabled_members"},
    --     {"code": "shelter", "max": 672, "uncapped_for": "homeless", "applies_to": "shelter_costs"}
    --   ]
    -- }
    statutory_citations JSONB DEFAULT '[]',
    author_notes    TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(program_id, jurisdiction_code, version)
);

CREATE INDEX idx_prs_program ON program_rule_set(program_id);
CREATE INDEX idx_prs_jurisdiction ON program_rule_set(jurisdiction_code);
CREATE INDEX idx_prs_effective ON program_rule_set(effective_date, expiration_date);
CREATE INDEX idx_prs_rules ON program_rule_set USING GIN (rules);

-- ============================================================
-- REFERENCE DATA (FPL, standard utility allowances, etc.)
-- ============================================================
CREATE TABLE reference_data (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    data_type       TEXT NOT NULL,                     -- 'fpl', 'standard_utility_allowance', 'snap_max_allotment'
    fiscal_year     INTEGER NOT NULL,
    jurisdiction_code TEXT DEFAULT 'US',
    data            JSONB NOT NULL,
    -- Example data for 'fpl':
    -- {
    --   "region": "contiguous",
    --   "thresholds": {
    --     "1": 15650, "2": 21150, "3": 26650, "4": 32150,
    --     "5": 37650, "6": 43150, "7": 48650, "8": 54150,
    --     "each_additional": 5500
    --   },
    --   "source": "91 FR 12345",
    --   "effective_date": "2026-01-12"
    -- }
    source_citation TEXT,
    effective_date  DATE NOT NULL,
    expiration_date DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(data_type, fiscal_year, jurisdiction_code)
);

CREATE INDEX idx_refdata_type ON reference_data(data_type, fiscal_year);
CREATE INDEX idx_refdata_jurisdiction ON reference_data(jurisdiction_code);
```

## Screening & Determination Tables

```sql
-- ============================================================
-- SCREENING SESSIONS
-- ============================================================
CREATE TABLE screening_session (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    household_id    UUID NOT NULL REFERENCES household(id),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    channel         TEXT NOT NULL DEFAULT 'web',
    locale          TEXT DEFAULT 'en',
    status          TEXT NOT NULL DEFAULT 'in_progress',
    started_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ,
    -- All intake responses as a single JSONB document
    intake_data     JSONB DEFAULT '{}',
    -- Example intake_data:
    -- {
    --   "household_size": 3,
    --   "has_elderly_or_disabled": false,
    --   "anyone_pregnant": true,
    --   "has_children_under_18": true,
    --   "youngest_child_age": 2,
    --   "total_monthly_gross_income": 2400,
    --   "income_sources": [
    --     {"type": "employment", "amount": 2000, "person": "head"},
    --     {"type": "child_support_received", "amount": 400, "person": "head"}
    --   ],
    --   "total_countable_assets": 1200,
    --   "housing_type": "rent",
    --   "monthly_rent": 1500,
    --   "receives_other_benefits": ["wic"],
    --   "all_citizens": true
    -- }
    -- Session metadata
    session_metadata JSONB DEFAULT '{}',
    -- Example session_metadata:
    -- {
    --   "ip_address": "192.168.1.1",
    --   "user_agent": "Mozilla/5.0...",
    --   "chatbot_conversation_id": "conv-uuid",
    --   "questions_answered": 14,
    --   "duration_seconds": 342,
    --   "abandonment_point": null
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_session_household ON screening_session(household_id);
CREATE INDEX idx_session_tenant ON screening_session(tenant_id);
CREATE INDEX idx_session_status ON screening_session(status);
CREATE INDEX idx_session_started ON screening_session(started_at);
CREATE INDEX idx_session_intake ON screening_session USING GIN (intake_data);

-- ============================================================
-- ELIGIBILITY RESULTS (one row per programme per session)
-- ============================================================
CREATE TABLE eligibility_result (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id      UUID NOT NULL REFERENCES screening_session(id) ON DELETE CASCADE,
    household_id    UUID NOT NULL REFERENCES household(id),
    program_id      UUID NOT NULL REFERENCES benefit_program(id),
    rule_set_id     UUID REFERENCES program_rule_set(id),
    jurisdiction_code TEXT NOT NULL,
    status          TEXT NOT NULL,                     -- 'likely_eligible', 'likely_ineligible',
                                                       -- 'needs_verification', 'unable_to_determine'
    confidence      NUMERIC(3,2),
    estimated_monthly_benefit NUMERIC(10,2),
    estimated_annual_benefit NUMERIC(12,2),
    -- Full rule evaluation results as JSONB
    rule_results    JSONB NOT NULL DEFAULT '[]',
    -- Example rule_results:
    -- [
    --   {
    --     "rule_code": "gross_income_test",
    --     "result": "pass",
    --     "actual_value": 2400,
    --     "threshold": 3407,
    --     "explanation": "Gross monthly income $2,400 is below the $3,407 limit for household size 3 (200% FPL)"
    --   },
    --   {
    --     "rule_code": "net_income_test",
    --     "result": "pass",
    --     "actual_value": 1720,
    --     "threshold": 2152,
    --     "explanation": "Net monthly income $1,720 (after deductions) is below the $2,152 limit (100% FPL)"
    --   },
    --   {
    --     "rule_code": "citizenship_test",
    --     "result": "pass",
    --     "explanation": "All household members are US citizens"
    --   }
    -- ]
    explanation_summary TEXT,                          -- AI-generated plain-language summary
    determined_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_result_session ON eligibility_result(session_id);
CREATE INDEX idx_result_household ON eligibility_result(household_id);
CREATE INDEX idx_result_program ON eligibility_result(program_id);
CREATE INDEX idx_result_status ON eligibility_result(status);
CREATE INDEX idx_result_rules ON eligibility_result USING GIN (rule_results);
```

## Documents, Referrals & Multi-Tenancy

```sql
-- ============================================================
-- DOCUMENTS
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

-- ============================================================
-- REFERRALS
-- ============================================================
CREATE TABLE referral (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id      UUID REFERENCES screening_session(id),
    household_id    UUID NOT NULL REFERENCES household(id),
    program_id      UUID NOT NULL REFERENCES benefit_program(id),
    tenant_id       UUID REFERENCES tenant(id),
    receiving_org   TEXT NOT NULL,
    referral_type   TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'pending',
    -- FHIR integration metadata
    fhir_data       JSONB DEFAULT '{}',
    -- Example fhir_data:
    -- {
    --   "service_request_id": "fhir-sr-uuid",
    --   "task_id": "fhir-task-uuid",
    --   "sdoh_category": "food-insecurity",
    --   "icd10_code": "Z59.41",
    --   "loinc_code": "88122-7",
    --   "consent_given": true,
    --   "consent_scope": "food_assistance_referral"
    -- }
    notes           TEXT,
    referred_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_referral_household ON referral(household_id);
CREATE INDEX idx_referral_status ON referral(status);
CREATE INDEX idx_referral_fhir ON referral USING GIN (fhir_data);

-- ============================================================
-- TENANTS
-- ============================================================
CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    tenant_type     TEXT NOT NULL,
    jurisdiction_code TEXT,
    subdomain       TEXT UNIQUE,
    config          JSONB DEFAULT '{}',
    -- Example config:
    -- {
    --   "branding": {"logo_url": "...", "primary_color": "#003366"},
    --   "programs_enabled": ["snap", "medicaid_magi", "wic", "liheap"],
    --   "default_locale": "es",
    --   "referral_partners": [{"name": "United Way", "api_endpoint": "..."}],
    --   "intake_questionnaire_id": "custom-q-001"
    -- }
    is_active       BOOLEAN DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- USERS & ROLES (same as Model 1)
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
    description     TEXT,
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
    changes         JSONB,                             -- {field: {old: ..., new: ...}} or full event payload
    context         JSONB DEFAULT '{}',                -- ip_address, user_agent, correlation_id
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_occurred ON audit_log(occurred_at);
CREATE INDEX idx_audit_tenant ON audit_log(tenant_id);
```

## Example Queries

```sql
-- Find all programmes where a household with gross income $2,400 and size 3
-- in California would pass the gross income test:

SELECT bp.code, bp.name, prs.jurisdiction_code,
       rule_elem->>'rule_code' AS rule_code,
       (rule_elem->'test'->>'threshold_value')::NUMERIC AS fpl_pct,
       (rule_elem->'test'->'thresholds_by_household_size'->>'3')::NUMERIC AS threshold
FROM program_rule_set prs
JOIN benefit_program bp ON bp.id = prs.program_id
CROSS JOIN LATERAL jsonb_array_elements(prs.rules->'eligibility_rules') AS rule_elem
WHERE prs.jurisdiction_code IN ('US', 'US-CA')
  AND prs.is_active = TRUE
  AND rule_elem->>'rule_code' = 'gross_income_test'
  AND (rule_elem->'test'->'thresholds_by_household_size'->>'3')::NUMERIC >= 2400;

-- Find all screening sessions where the intake data indicates pregnancy:
SELECT id, household_id, started_at
FROM screening_session
WHERE intake_data @> '{"anyone_pregnant": true}'
  AND status = 'completed';

-- Get the current SNAP max allotment for a household of 4 in the contiguous US:
SELECT data->'thresholds'->>'4' AS max_allotment
FROM reference_data
WHERE data_type = 'snap_max_allotment'
  AND fiscal_year = 2026
  AND jurisdiction_code = 'US';
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity (Person, Household) | 3 | person, household, household_member |
| Financial | 2 | income, expense |
| Programme & Rules | 3 | benefit_program, program_rule_set, reference_data |
| Screening & Results | 2 | screening_session, eligibility_result |
| Documents & Referrals | 2 | document, referral |
| Multi-Tenancy & RBAC | 4 | tenant, app_user, role, user_role |
| Audit | 1 | audit_log |
| **Total** | **17** | 5 fewer than the normalized model |

---

## Key Design Decisions

1. **Programme rules stored as JSONB rather than separate rule/parameter tables.** A single `program_rule_set` row contains the complete rule definition for a programme in a jurisdiction. This means adding a new jurisdiction is an INSERT of one row with a JSON document, not a migration of rule/parameter/threshold tables. The trade-off is that rule integrity depends on JSON Schema validation rather than foreign keys.

2. **Intake data as a single JSONB column on `screening_session`.** Rather than a separate `screening_response` table with one row per question, all intake responses are stored as a single structured JSON document. This dramatically simplifies the screening workflow (one document in, one document out) and mirrors the PolicyEngine API's household-as-JSON pattern. Individual fields can still be queried efficiently via GIN indexes and containment operators (`@>`).

3. **Rule evaluation results as JSONB on `eligibility_result`.** The `rule_results` JSONB array captures which rules passed, failed, and what values were tested. This avoids a separate `determination_rule_result` junction table while still providing full explainability. The trade-off is that reporting on individual rule outcomes requires JSONB array unpacking rather than simple JOINs.

4. **`countability` JSONB on income records.** Different programmes count income differently. Rather than separate boolean columns for each programme (`is_countable_snap`, `is_countable_magi`, `is_countable_ssi`), a JSONB object maps programme codes to countability flags. New programmes can be added without ALTER TABLE.

5. **Reference data in a generic table with typed JSONB.** FPL thresholds, SNAP allotment tables, standard utility allowances, and other reference data all go in one `reference_data` table with a `data_type` discriminator. This avoids creating separate tables for each type of reference data while keeping each dataset clearly typed and indexed.

6. **FHIR integration data as JSONB on referrals.** Rather than modelling FHIR resources as separate tables (which would effectively duplicate the FHIR data model), the FHIR-relevant identifiers and codes are stored in a `fhir_data` JSONB column. This keeps the screener's data model independent of FHIR while maintaining enough linkage for interoperability.

7. **Discipline boundary: relational for identity and workflow, JSONB for variation.** The schema draws a clear line: core entities (person, household, session) and their relationships are relational with foreign keys. Everything that varies by programme, jurisdiction, or intake form is JSONB. This avoids the "JSONB everything" anti-pattern while still delivering the flexibility needed for multi-jurisdiction deployment.

8. **Version column on `program_rule_set` enables rule history without temporal tables.** When rules change, a new row is inserted with an incremented version and the old row's `expiration_date` is set. This provides rule history without the complexity of bi-temporal columns, though it is less powerful than the event-sourced model for arbitrary point-in-time queries.
