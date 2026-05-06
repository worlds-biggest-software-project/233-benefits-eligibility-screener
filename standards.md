# Standards & API Reference

> Project: Benefits Eligibility Screener · Generated: 2026-05-03

---

## Industry Standards & Specifications

### Regulatory & Legal Standards

| Standard | Relevance |
|----------|-----------|
| **Social Security Act, Title IV / XIX / XXI** | Governing legislation for SNAP, Medicaid, CHIP, TANF, and LIHEAP eligibility rules; defines the federal requirements that any compliant screener must model. |
| **ACA MAGI Rules (45 CFR Part 155)** | Modified Adjusted Gross Income methodology mandated for marketplace and Medicaid eligibility calculations; defines exactly how household income is computed for benefits screening. |
| **IRS Internal Revenue Code § 6103** | Restricts the disclosure and matching of tax return data for benefit verification; directly constrains income-verification API design and data-sharing architecture. |
| **HIPAA (45 CFR Parts 160 & 164)** | Privacy and security requirements apply when a screener collects health-related information or connects to healthcare eligibility systems; governs data handling for Medicaid-linked screens. |
| **ADA Title II / Section 504** | Requires public-entity digital services to be accessible to individuals with disabilities; applies to state and local agency deployments of an eligibility screener. |
| **Section 508 of the Rehabilitation Act (29 USC § 794d)** | Mandates federal technology procurement to meet accessibility standards; any screener deployed by a federal agency or federally funded project must comply. |
| **Executive Order 13985 / DEIA Guidance (2021)** | Directs federal agencies to remove barriers to benefit access; supports the policy case for AI-native screeners reducing application friction. |

---

### W3C & IETF Standards

| Standard | URL | Relevance |
|----------|-----|-----------|
| **WCAG 2.2** (W3C Recommendation) | https://www.w3.org/TR/WCAG22/ | Web Content Accessibility Guidelines; Level AA is the de facto requirement for public-sector digital services in the US. Incorporates new criteria relevant to mobile and touch interfaces. |
| **RFC 9110 — HTTP Semantics** | https://www.rfc-editor.org/rfc/rfc9110 | Defines correct use of HTTP methods, status codes, and caching; foundational for a REST API exposing an eligibility calculation endpoint. |
| **RFC 8259 — JSON Data Interchange Format** | https://www.rfc-editor.org/rfc/rfc8259 | Defines JSON encoding; the universal wire format for eligibility screener APIs exchanging household data and results. |
| **RFC 6749 — OAuth 2.0** | https://www.rfc-editor.org/rfc/rfc6749 | Authorization framework used to protect administrative and agency-facing API endpoints while keeping the public screening endpoint open. |
| **RFC 7519 — JSON Web Token (JWT)** | https://www.rfc-editor.org/rfc/rfc7519 | Token format used in OAuth 2.0 flows; relevant to secure session handling for authenticated caseworker and admin interfaces. |
| **OpenID Connect 1.0** | https://openid.net/connect/ | Identity layer on top of OAuth 2.0 for SSO with government identity providers (Login.gov, state Medicaid portals). |
| **RFC 8288 — Web Linking** | https://www.rfc-editor.org/rfc/rfc8288 | Defines `Link` headers for pagination and hypermedia; relevant to REST API design for paginated program catalogue endpoints. |
| **ARIA 1.2** (W3C) | https://www.w3.org/TR/wai-aria-1.2/ | Accessible Rich Internet Applications specification; required alongside WCAG for screen-reader-compatible dynamic screener interfaces. |

---

### Data Model & API Specifications

| Standard | URL | Relevance |
|----------|-----|-----------|
| **OpenAPI Specification 3.1** | https://spec.openapis.org/oas/v3.1.0 | Industry standard for describing REST APIs; the eligibility screener REST API should publish an OpenAPI document for developer tooling, SDK generation, and contract testing. |
| **JSON Schema (Draft 2020-12)** | https://json-schema.org/draft/2020-12 | Used within OpenAPI 3.1 for validating household composition request bodies and eligibility result response schemas. |
| **HL7 FHIR R4 (4.0.1)** | https://hl7.org/fhir/R4/ | Fast Healthcare Interoperability Resources; the integration standard when a screener connects to EHRs, Medicaid management information systems (MMIS), or health plan APIs. |
| **FHIR US Core 6.x / USCDI v3** | https://www.hl7.org/fhir/us/core/ | US-specific FHIR profiles and data elements (mandated by ONC HTI-1 rule effective January 2026); governs patient/member data exchanged with healthcare stakeholders. |
| **HL7 FHIR SDOH Clinical Care IG v2.3** | https://hl7.org/fhir/us/sdoh-clinicalcare/ | Gravity Project implementation guide defining FHIR profiles for SDOH screening responses, conditions, goals, service requests, and closed-loop social care referrals. Key for interoperability with Findhelp, EHRs, and Medicaid HRSN waiver implementations. |
| **ICD-10-CM Z-codes (Social Determinants)** | https://www.cdc.gov/nchs/icd/icd10cm.htm | Standardised codes (Z55–Z65) for documenting social needs (food insecurity, housing instability, etc.) in clinical and administrative records; used for Medicaid HRSN billing. |
| **LOINC Social Determinant Panels** | https://loinc.org/ | Standardised codes for SDOH screening instruments (Hunger Vital Sign, AHC HRSN Screening Tool, PRAPARE); required for FHIR-compliant SDOH data exchange. |
| **SNOMED-CT SDOH Value Sets** | https://www.snomed.org/ | Clinical terminology used alongside LOINC for coding SDOH observations in FHIR resources. |

---

### Rules-as-Code & Policy Engine Standards

| Standard / Framework | URL | Relevance |
|----------------------|-----|-----------|
| **OpenFisca Web API & YAML Rule Format** | https://openfisca.org/doc/openfisca-web-api/input-output-data.html | De-facto open standard for encoding tax-benefit rules as executable code; OECD- and UNDP-endorsed. Defines the `/calculate` REST endpoint pattern and YAML variable-definition format widely adopted for benefits rule portability. |
| **OECD Rules as Code Initiative** | https://www.oecd.org/en/topics/sub-issues/digitalising-government/rules-as-code.html | OECD framework and best-practice guidelines for governments adopting machine-executable legislation; relevant to policy governance decisions around rule authoring and publication. |
| **G7 Regulatory Innovation Task Force — RaC Standards (2025)** | https://www.cigionline.org/static/documents/T7_TF2_Rapson_et_al.pdf | G7 working group on a shared best-practices compendium and interoperability standards for machine-executable benefit rules; emerging standard to watch. |

---

### Security & Compliance Standards

| Standard | URL | Relevance |
|----------|-----|-----------|
| **NIST SP 800-63B — Digital Identity Guidelines** | https://pages.nist.gov/800-63-4/ | Defines identity assurance levels (IAL) and authenticator assurance levels (AAL); relevant to identity proofing when a screener connects to Login.gov or state identity systems. |
| **NIST SP 800-53 Rev 5 — Security Controls** | https://csrc.nist.gov/publications/detail/sp/800-53/rev-5/final | Federal baseline security control catalogue; required for FedRAMP authorisation of any cloud-hosted screener serving federal agencies. |
| **FedRAMP** | https://www.fedramp.gov/ | US federal cloud security authorisation programme; a screener offered to federal agencies as a SaaS must achieve FedRAMP Moderate or High authorisation. |
| **OWASP Top 10 (2021)** | https://owasp.org/www-project-top-ten/ | Web application security baseline; injection, broken authentication, and insecure data exposure are the primary risks for a benefits screener handling sensitive household data. |
| **SMART on FHIR (OAuth 2.0 + FHIR Scopes)** | https://smarthealthit.org/ | Security and authorisation layer for FHIR-integrated screeners; controls which user roles or apps can read/write patient-linked SDOH data in an EHR context. |
| **GDPR (EU 2016/679)** | https://gdpr.eu/ | Applies if the screener is deployed in EU jurisdictions or processes EU residents' personal data; imposes consent, data minimisation, and right-to-erasure requirements. |

---

### Accessibility & Design System Standards

| Standard | URL | Relevance |
|----------|-----|-----------|
| **U.S. Web Design System (USWDS) 3.x** | https://designsystem.digital.gov/ | Official federal design system for US government digital services; used by Nava's OSCER and Code for America tools; ensures consistent, accessible, and on-brand government UX. |
| **Section 508 ICT Standards (Revised 2017)** | https://www.access-board.gov/ict/ | Revised 508 Standards (incorporating WCAG 2.0 Level A/AA) mandatory for federally procured ICT; the USWDS May 2025 VPAT 2.5 report is the reference conformance document. |

---

## Similar Products — Developer Documentation & APIs

### PolicyEngine API

- **Description:** Open-source microsimulation of the US (and UK) tax-benefit system. Calculates household eligibility and benefit amounts for SNAP, Medicaid, SSI, EITC, CTC, ACA subsidies, TANF, and all 50-state income tax systems in a single REST call.
- **API Documentation:** https://policyengine.github.io/policyengine.py/
- **SDKs/Libraries:** Python — `pip install policyengine-us` (PyPI); REST API available at `api.policyengine.org`
- **Developer Guide:** https://policyengine.org/us/research/api-v2
- **GitHub:** https://github.com/policyengine/policyengine-api
- **Standards:** REST/JSON; OpenFisca-compatible YAML rule format; Apache 2.0
- **Authentication:** API key for commercial use; open for research/non-profit use

---

### NYC Benefits Platform Screening API

- **Description:** Public REST API exposing a Drools-based eligibility rules engine for 40+ City, State, and Federal programs (SNAP, Cash Assistance, WIC, HEAP, etc.). Accepts household composition data and returns a list of likely-eligible programs.
- **API Documentation:** https://screeningapidocs.cityofnewyork.us/
- **SDKs/Libraries:** None official; JSON REST API directly consumable from any HTTP client
- **Developer Guide:** https://screeningapidocs.cityofnewyork.us/getting-started
- **GitHub:** https://github.com/CityOfNewYork/ACCESS-NYC · https://github.com/NYCOpportunity/ACCESS-NYC-Rules
- **Standards:** REST/JSON; open data (NYC Open Data); rules published as Drools DRL files
- **Authentication:** None required for read (public API); write/admin endpoints require API key

---

### MyFriendBen Benefits API

- **Description:** Open-source Python/Django REST API that accepts household demographic data and returns eligibility status and estimated dollar values for 40+ programs in Colorado, with a modular rule set for other jurisdictions.
- **API Documentation:** https://github.com/MyFriendBen/benefits-api (README)
- **SDKs/Libraries:** Python Django back end; React front end (`benefits-calculator`)
- **Developer Guide:** https://github.com/MyFriendBen/benefits-api
- **Standards:** REST/JSON; MIT licence; white-label deployment model
- **Authentication:** Open for local deployment; token-based auth for hosted instances

---

### OpenFisca Web API

- **Description:** JSON REST API serving as the computation layer for any OpenFisca tax-benefit rules package. Accepts a household situation (entities + variables) and returns calculated values for any encoded benefit or tax variable.
- **API Documentation:** https://openfisca.org/doc/openfisca-web-api/input-output-data.html
- **SDKs/Libraries:** Python (`pip install openfisca-core`; `pip install openfisca-france`, `openfisca-us` etc.); REST API self-hostable via Docker
- **Developer Guide:** https://openfisca.org/doc/
- **GitHub:** https://github.com/openfisca/openfisca-core
- **Standards:** REST/JSON; AGPL-3.0; OpenAPI-compatible endpoint descriptions
- **Authentication:** No authentication on self-hosted instances by default; production deployments may add OAuth 2.0 gateway

---

### Salesforce Public Sector Solutions API

- **Description:** Salesforce platform API exposing data models for Benefits, Programs, Benefit Applications, and Eligibility Determinations. Enables external systems to read and write benefits case data in Salesforce.
- **API Documentation:** https://developer.salesforce.com/docs/atlas.en-us.psc_api.meta/psc_api/
- **SDKs/Libraries:** Salesforce SDKs (JavaScript, Java, Python via `simple-salesforce`); REST and SOAP flavours
- **Developer Guide:** https://trailhead.salesforce.com/content/learn/modules/benefit-management-data-model-in-public-sector-solutions/
- **Standards:** REST/JSON and SOAP/XML; OpenAPI 3.0 specification available via Salesforce API Explorer; OData compatible
- **Authentication:** OAuth 2.0 (Connected App); SMART on FHIR via Health Cloud extension

---

### Findhelp (Aunt Bertha) API

- **Description:** Commercial API enabling bidirectional social care referral data exchange: send screening results, create referrals to community-based organisations, receive status updates, and retrieve program listings. USCDI v3 SDOH-coded data exchange.
- **API Documentation:** https://integrations.auntbertha.com/ (requires partner agreement)
- **SDKs/Libraries:** REST/JSON; partner-specific SDKs on request
- **Developer Guide:** https://company.findhelp.com/products/customer-integrations/
- **Standards:** REST/JSON; FHIR SDOH Clinical Care IG (Gravity Project) profiles; USCDI v3 social care data elements; ICD-10 Z-codes; LOINC SDOH panels
- **Authentication:** OAuth 2.0; API key for lower-trust integrations

---

### Argyle Income Verification API

- **Description:** Consent-based real-time payroll and income verification API. Applicants connect their payroll/bank accounts via Argyle Link; the API returns structured income, employment, and asset data for benefit eligibility determination.
- **API Documentation:** https://www.argyle.com/platform-overview/api
- **SDKs/Libraries:** JavaScript SDK (`argyle-link`); REST API (JSON); iOS and Android SDKs
- **Developer Guide:** https://argyle.com/docs
- **Standards:** REST/JSON; OpenAPI 3.0; consent-based data access aligned with CFPB Open Banking / Section 1033 of the Dodd-Frank Act
- **Authentication:** API key + OAuth 2.0 user-consent flow via Argyle Link widget

---

### Login.gov API (Identity & Authentication)

- **Description:** US federal shared identity service providing identity verification (IAL2) and authentication (AAL2) for government digital services. Benefits screeners requiring verified identity can integrate via Login.gov rather than building their own identity proofing.
- **API Documentation:** https://developers.login.gov/
- **SDKs/Libraries:** Ruby, Python, Java, .NET reference clients; OpenID Connect standard
- **Developer Guide:** https://developers.login.gov/oidc/getting-started/
- **Standards:** OpenID Connect 1.0; OAuth 2.0; PKCE; NIST SP 800-63B IAL2/AAL2
- **Authentication:** OIDC client credentials; PKCE flow for public clients

---

## Notes

- **Rules-as-Code interoperability is still maturing:** OpenFisca and PolicyEngine use compatible but not identical rule formats; no universal standard for portable benefits rule sets has yet been ratified. The G7 RaC initiative and OECD guidance are the most active standards-development bodies to watch.
- **FHIR SDOH mandate timeline:** ONC's HTI-1 rule requires USCDI v3 compliance (including SDOH elements) in certified health IT by January 2026, creating a forcing function for FHIR-based social care data exchange in Medicaid-adjacent screeners.
- **FedRAMP pathway:** A screener offered as SaaS to federal agencies must complete FedRAMP authorisation; this is a significant engineering and compliance investment. State-only deployments fall under state risk management frameworks (often NIST 800-53-derived) rather than FedRAMP.
- **SMART on FHIR scope standardisation:** The Gravity Project's FHIR IG defines resource types and profiles but does not yet define a standard SMART on FHIR scope vocabulary for SDOH screening applications; individual EHR vendors define their own scope strings.
