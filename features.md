# Benefits Eligibility Screener — Feature & Functionality Survey

> Candidate #233 · Researched: 2026-05-03

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| IBM Cúram | Enterprise rules platform | Commercial (proprietary) | https://www.ibm.com/docs/en/SS8S5A_7.0.0/com.ibm.curam.content.doc/product_overview/c_solution_modules.html |
| Salesforce Public Sector Solutions | Low-code SaaS | Commercial SaaS | https://www.salesforce.com/government/solutions/ |
| PolicyEngine | Open-source microsimulation | MIT / Open Source | https://www.policyengine.org/us |
| MyFriendBen | Open-source screener | Open Source (MIT) | https://www.myfriendben.org/ |
| NYC Benefits Platform / ACCESS NYC | Open-source screener + API | Open Source (MIT) | https://screeningapidocs.cityofnewyork.us/ |
| Findhelp (Aunt Bertha) | Social care referral SaaS | Commercial SaaS | https://company.findhelp.com/ |
| OpenFisca | Open-source rules engine | AGPL-3.0 | https://openfisca.org/en/ |
| Nava PBC / OSCER | Open-source eligibility component | Apache 2.0 / Open Source | https://github.com/navapbc/oscer |
| BenefitsCal (California) | Government-built multi-program portal | Government / Proprietary | https://benefitscal.com/ |
| Code for America / GetCalFresh | Civic-tech screener (now merged into BenefitsCal) | Open Source | https://codeforamerica.org/programs/social-safety-net/food-benefits/ |

---

## Feature Analysis by Solution

### IBM Cúram

**Core features**
- Rules-based eligibility determination engine (Cúram Eligibility and Entitlement Engine / CER) supporting complex multi-program rule sets
- Integrated case management: unified view of client demographics, eligibility history, benefits, and services
- Benefit calculation and payment management including adjustments and disbursements
- Workflow automation for case events and circumstance changes
- Reporting and analytics dashboards for program performance monitoring
- Configuration tooling for policy analysts to update program rules without code changes
- Income Support module covering means-testing, MAGI calculations, and program co-enrolment

**Differentiating features**
- CER (Custom Eligibility Rules) engine allows non-developers to encode legislative rules using a business rules DSL
- Proven at massive government scale (US states, UK DWP, Canadian provinces)
- Integrated payment engine handles entitlement calculations, overpayments, and clawbacks

**UX patterns**
- Caseworker-centric desktop interface; applicant-facing portal is a separate module
- Progressive disclosure of questions based on program type
- Task lists guide caseworkers through intake, verification, and adjudication

**Integration points**
- SOAP and REST APIs for inter-system data exchange
- Income and Eligibility Verification System (IEVS) connectors for SSA, IRS, and state data
- Third-party identity verification and document management integrations

**Known gaps**
- Applicant self-service UX is dated compared to modern consumer products
- Policy rule updates take weeks-to-months through IT change-management processes
- No AI-native conversational intake; bolt-on chatbots require custom integration
- Extremely high total cost of ownership limits adoption outside large agencies

**Licence / IP notes**
- Fully proprietary; no open-source components. Multi-million dollar government contracts required.

---

### Salesforce Public Sector Solutions

**Core features**
- Omnistudio-powered no-code guided application flows and prescreening questionnaires
- AI-powered eligibility pre-screening that matches applicants to eligible programs before full application
- Automated eligibility determination and benefit disbursement rules
- Case management with unified household view (household details, related cases, benefits)
- Document collection and verification workflows
- Analytics and reporting via Salesforce Einstein
- Configurable benefit program catalogue with eligibility criteria

**Differentiating features**
- Rapid deployment via pre-built data models and low-code workflow builder
- Einstein AI personalisation for benefit matching and next-best-action recommendations
- Tight integration with Salesforce CRM ecosystem (Marketing Cloud, Service Cloud)
- OmniStudio FlexCards for dynamic, role-based UI composition

**UX patterns**
- Constituent-facing guided digital application with step-by-step progress indicators
- Caseworker 360-degree household view with integrated task management
- Mobile-responsive design out of the box

**Integration points**
- REST and SOAP APIs via Salesforce platform
- FHIR connectors available through Health Cloud partnership
- MuleSoft integration for legacy system connectivity
- Open API / Swagger-described endpoints

**Known gaps**
- Licensing costs prohibitive for small or mid-size agencies without Salesforce contracts
- Rules management requires Salesforce expertise; policy analysts cannot update rules independently
- Limited community-resource referral capability compared to dedicated SDOH platforms
- No direct LLM/AI document extraction for income verification

**Licence / IP notes**
- Fully commercial SaaS. Data residency and sovereignty concerns for some state agencies.

---

### PolicyEngine

**Core features**
- Microsimulation of US federal and state tax-benefit system covering SNAP, Medicaid, SSI, TANF, EITC, CTC, ACA subsidies, and 50-state income tax
- Household-level eligibility and benefit-amount calculations
- Policy reform modelling: compare current law vs. proposed changes
- REST API for household eligibility queries
- Python package (`policyengine-us`) for programmatic integration
- AI-generated plain-language explanations of tax/benefit calculations (2025 feature)
- Annual updates for federal poverty levels, state tax parameters, and benefit thresholds

**Differentiating features**
- Simultaneous cross-program eligibility and dollar-value estimation in one call
- Policy simulation capability (unique: models how law changes affect a household's benefits)
- Open-source YAML-based rule definitions auditable by policy researchers
- TAXSIM-compatible microsimulation for large-scale population analysis

**UX patterns**
- Web app with household questionnaire and results summary page
- Cliff effect visualisation shows benefit loss as income rises
- API-first design enabling white-label integration by third parties (e.g., Illinois Benefit Hub)

**Integration points**
- REST API (JSON in/out); Python SDK via PyPI
- OpenFisca-compatible rule format (YAML/Python)
- Public GitHub repositories with full rule definitions

**Known gaps**
- No caseworker or case management functionality
- No document upload or identity verification features
- No referral or application submission workflow; calculation only
- US and UK coverage only; no localised rule sets for other countries

**Licence / IP notes**
- Core engine: Apache 2.0. Rule definitions: open data (CC0 or CC-BY). No patent encumbrances identified.

---

### MyFriendBen

**Core features**
- Eligibility screening for 40+ government benefits, tax credits, and nonprofit programs
- Benefit value estimation with dollar amounts and time-to-apply estimates
- Document requirements listed per program
- Mobile-first, accessible design
- Available in 12 languages
- Open-source React front end and Python/Django back-end rules API

**Differentiating features**
- Personalised results report with a prioritised application roadmap
- "Transparent" benefit information (value, time, documents) to help users plan before applying
- White-label embeddable tool for community organisations and county partners
- Modular architecture allows communities to add local/nonprofit programs alongside federal ones

**UX patterns**
- Single-session questionnaire completing in under six minutes
- Results page shows likely-eligible programs with estimated values and application links
- Progressive disclosure: questions adapt to prior answers

**Integration points**
- Public REST API (GitHub: MyFriendBen/benefits-api)
- Embeddable widget for partner organisation websites
- White-label deployment model for county and state partners

**Known gaps**
- No case management or ongoing case tracking
- No document upload or income verification
- No conversational AI; form-based questionnaire only
- Limited to Colorado and partner states; rule sets require localisation for other jurisdictions

**Licence / IP notes**
- Front end and back end: MIT licence. Full source on GitHub.

---

### NYC Benefits Platform / ACCESS NYC

**Core features**
- Eligibility screening for 40+ City, State, and Federal programs (SNAP, Cash Assistance, WIC, HEAP, etc.)
- Drools-based rules engine exposed as a public REST API
- Machine-readable eligibility criteria and calculation logic
- Quarterly rules updates maintained by NYC Opportunity
- Open-source front-end application (JavaScript SPA)
- Open API documentation at screeningapidocs.cityofnewyork.us

**Differentiating features**
- Fully open API allowing any front end to query the rules engine
- Rules maintained as open data alongside the API documentation
- Programme criteria documented in human-readable format alongside the machine-readable rules
- Serverless AWS Lambda deployment enabling cost-effective scaling

**UX patterns**
- Simple questionnaire interface; results show applicable programs with application links
- Designed for constituent self-service, not caseworker workflow

**Integration points**
- Public REST API (JSON); no authentication required for read access
- Open-source Drools rules available on GitHub (NYCOpportunity/ACCESS-NYC-Rules)
- NYC Open Data catalogue listing

**Known gaps**
- NYC-only rule coverage; significant effort to adapt for other jurisdictions
- Evaluating replatform away from Drools (specialist skills required to maintain)
- No document upload, identity verification, or application submission
- No case management or referral tracking

**Licence / IP notes**
- MIT licence. Rules and data available as open data.

---

### Findhelp (Aunt Bertha)

**Core features**
- National database of 600,000+ social care programs and community resources
- Social needs screening tools (configurable SDOH screening forms)
- Closed-loop referral management: send, track, and confirm referrals to CBOs
- Automatic mapping of screening responses to standardised SDOH codes (ICD-10 Z-codes, LOINC, SNOMED-CT)
- Granular, per-need consent management
- Outcomes tracking and care coordination metrics
- EHR integration (Epic, Cerner) and Medicaid system connectors

**Differentiating features**
- Largest US community-resource database with regular verification of program availability
- USCDI-compliant SDOH data exchange for Medicaid HRSN waiver reimbursement
- Consent architecture at need-category level (food vs. housing vs. employment)
- API partnerships enabling bidirectional referral data with health plans and social service agencies

**UX patterns**
- Configurable screening wizard surfacing relevant programs for local populations
- Care coordinator dashboard for referral management and outcome tracking
- EHR-embedded workflows for clinical teams

**Integration points**
- REST API with FHIR-compatible SDOH endpoints
- USCDI v3 social care data exchange
- Native EHR integrations (Epic App Orchard, Cerner marketplace)
- Webhooks for referral status updates

**Known gaps**
- Referral and resource-directory focus; not a deterministic eligibility engine
- No benefit dollar-value estimation
- Limited open-source components; proprietary program database
- Cost prohibitive for smaller community organisations

**Licence / IP notes**
- Fully commercial. Program database is proprietary. API access requires contract.

---

### OpenFisca

**Core features**
- Open-source Python framework for encoding tax and benefit rules as executable code
- REST Web API exposing a `/calculate` endpoint for household simulations
- YAML-based variable and parameter definitions for policy rules
- Support for multiple jurisdictions (France, Tunisia, Senegal, UK via PolicyEngine, etc.)
- Vectorised Python API for large-scale microsimulation
- Parameter versioning so rule history is trackable over time

**Differentiating features**
- Endorsed by OECD and UNDP as a Rules-as-Code reference implementation
- Jurisdiction-agnostic framework: any country or sub-national government can model its rules
- Legislative parameter traceability: each rule links to its statutory source
- Used by academic researchers and governments for both individual calculations and population-level policy analysis

**UX patterns**
- Developer-facing; no built-in end-user UI
- Intended as a computation back end for citizen-facing applications

**Integration points**
- REST Web API (JSON); Python package via PyPI (`openfisca-core`)
- Compatible with OpenAPI tooling for documentation generation
- GitHub-hosted rule repositories with CI/CD for rule testing

**Known gaps**
- No built-in applicant UI; requires separate front-end development
- No identity verification, document processing, or referral management
- Requires Python expertise to write and maintain rule definitions
- Limited tooling for non-technical policy analysts to author rules without developer assistance

**Licence / IP notes**
- Core engine: AGPL-3.0. Country-specific rule packages vary (GPL or LGPL).

---

### Nava PBC / OSCER

**Core features**
- Open-source Medicaid community engagement reporting and eligibility tool
- Configurable rules engine for HR1-compliant work requirement exemption screening
- Guided workflow for submitting qualifying activities (work, education, volunteering)
- Compliance report generation for state Medicaid agencies
- Sidecar architecture integrating with existing Medicaid eligibility systems
- U.S. Web Design System (USWDS) front end for accessibility compliance
- Cloud-agnostic Infrastructure as Code (IaC) deployment

**Differentiating features**
- Purpose-built for HR1 Medicaid work requirement compliance
- Designed so states retain full ownership and data sovereignty
- Minimal, well-defined integration touchpoints reduce risk of legacy system disruption

**UX patterns**
- Step-by-step guided intake; USWDS component library ensures Section 508 compliance
- Applicant self-service with caseworker administration back end

**Integration points**
- REST APIs for Medicaid eligibility system integration
- IaC modules for AWS, Azure, and GCP deployment
- Open-source on GitHub (navapbc/oscer)

**Known gaps**
- Narrowly scoped to community engagement / work requirement use case
- Not a general-purpose cross-program screener
- Requires state IT capacity to deploy and operate

**Licence / IP notes**
- Apache 2.0 licence. Fully open source.

---

## Cross-Cutting Feature Themes

### Table-Stakes Features
- Multi-program eligibility screening in a single session
- Plain-language questions adaptable to user answers (progressive disclosure)
- Eligibility results displayed with actionable next steps and application links
- Mobile-first, responsive design
- Accessibility compliance (WCAG 2.2 / Section 508 / ADA)
- At least one major language beyond English (Spanish most common)
- Annual or more frequent updates to reflect policy changes (poverty lines, benefit thresholds)

### Differentiating Features
- Dollar-value benefit estimation (time-to-apply, expected monthly value)
- Cross-program bundling with a prioritised application roadmap
- AI-powered conversational intake vs. static form trees
- Closed-loop referral tracking to community-based organisations
- Rules-as-Code with traceable legislative citations
- Embeddable widget or white-label deployment model for partner organisations
- Policy reform simulation (compare current law vs. proposed changes)
- Predictive cliff-effect visualisation showing how income changes affect benefit eligibility

### Underserved Areas / Opportunities
- Non-English languages beyond Spanish (many tools cover only 2–3 languages despite diverse applicant populations)
- Document understanding: automated extraction of income/household data from uploaded pay stubs, tax returns, and bank statements
- Proactive outreach: identifying likely-eligible non-participants from administrative data and initiating contact
- Real-time policy update cycle: most rule engines require IT intervention for even minor threshold changes; AI-assisted rule translation from statute text is absent
- Rural and low-connectivity populations: offline-capable or SMS-based screeners are rare
- Integrated identity verification without requiring an existing government account
- Cross-jurisdictional portability: very few tools cover federal + state + county + nonprofit programs in a unified model

### AI-Augmentation Candidates
- Conversational intake replacing form questionnaires (LLM-guided natural language interview)
- Automated rules extraction: LLMs translate statute/regulation text into executable policy rules
- Document intelligence: OCR + NLP to extract income, household composition, and asset data from uploaded documents
- Proactive eligibility prediction: ML models scoring likely eligibility from partial data to reduce abandonment
- Explanation generation: AI-produced plain-language explanations of why a household qualifies or does not qualify for each program

---

## Legal & IP Summary

The dominant open-source tools in this space (PolicyEngine, MyFriendBen, OpenFisca, ACCESS NYC, Nava OSCER) use permissive or reciprocal licences (MIT, Apache 2.0, AGPL-3.0) with no patent encumbrances identified. The AGPL-3.0 licence of OpenFisca requires any derivative web service to release modifications; this is compatible with open-source AI-native tools but would constrain proprietary SaaS products. The proprietary vendors (IBM Cúram, Salesforce, Findhelp) own their platform code and databases outright; no patent conflicts are anticipated from building an independent AI-native screener. Government-produced eligibility rules and program criteria are public domain under US law. Benefit program logos and brand assets are not subject to copyright reuse.

---

## Recommended Feature Scope

**Must-have (MVP)**
- Questionnaire-driven multi-program screening covering key federal programs (SNAP, Medicaid/CHIP, SSI, EITC, WIC, LIHEAP, housing vouchers)
- Rules-as-Code back end with versioned, auditable eligibility logic linked to regulatory sources
- Benefit dollar-value estimation and time-to-apply for each matched program
- Accessible, mobile-first front end meeting WCAG 2.2 / Section 508 requirements
- English and Spanish support
- Public REST API so third parties can build their own front ends against the rules engine
- Quarterly update process for federal poverty levels, benefit thresholds, and program rules

**Should-have (v1.1)**
- Conversational AI intake (LLM-powered natural language interview replacing form)
- Document upload with AI extraction (pay stubs, tax returns) to pre-populate application data
- Cross-program application roadmap with prioritised action steps
- Embeddable widget and white-label deployment for community partners
- Additional language support (Chinese, Vietnamese, Arabic, Portuguese, French Creole)
- Closed-loop referral integration with Findhelp or similar community-resource network

**Nice-to-have (backlog)**
- Policy reform simulation (household impact of proposed benefit changes)
- Cliff-effect visualisation showing benefit loss as income rises
- Proactive outreach module (scoring and contact campaigns for likely-eligible non-participants)
- Offline/SMS mode for low-connectivity populations
- Admin UI for non-technical policy analysts to update benefit thresholds without developer intervention
- OpenFisca and PolicyEngine rule-format import/export for interoperability with existing rule libraries
