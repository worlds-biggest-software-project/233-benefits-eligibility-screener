# Benefits Eligibility Screener

> Candidate #233 · Researched: 2026-05-02

## Existing Products and Software Packages

| Tool | Description | Type | Pricing | Strengths / Weaknesses |
|------|-------------|------|---------|------------------------|
| Curam (IBM) | Social program management platform used by US agencies to automate eligibility determination for healthcare, income support, and child welfare | Commercial / Enterprise | Multi-million-dollar government contracts | Strengths: proven at scale, configurable rules engine; Weaknesses: extremely expensive, slow to update policy rules |
| Salesforce Public Sector Solutions | CRM-based benefits-intake and case-management platform with low-code eligibility workflow builder | Commercial SaaS | Per-user enterprise licensing | Strengths: rapid configuration, modern UX; Weaknesses: high licensing cost, requires implementation partner |
| Nava Benefits Screener (open-source tooling) | Open-source eligibility logic and intake components developed by Nava PBC for state benefits modernisation | Open Source / Consulting | Free code; consulting fees | Strengths: rules-as-code approach, transparent; Weaknesses: requires significant state-IT capacity |
| BenefitsCal (California) | State-built integrated benefits portal covering CalFresh, Medi-Cal, and CalWORKs | Government-built SaaS | N/A (state-funded) | Strengths: unified cross-program view; Weaknesses: state-specific, not replicable without major effort |
| Aunt Bertha (now Findhelp) | Social care network and screening tool connecting residents to local benefit programs and community resources | Commercial SaaS | Per-seat / enterprise | Strengths: large program database, community-resource focus; Weaknesses: referral-oriented rather than eligibility-determinative |
| Benefits Data Trust (BDT) | Nonprofit using data matching and outreach to enroll eligible individuals in benefits they are not receiving | Nonprofit / contract | Grant / government contract | Strengths: proactive enrollment focus; Weaknesses: not a self-service platform |
| Sagely (AI screener) | AI-guided benefits screening chatbot for older adults and caregivers | Commercial SaaS | Contact for quote | Strengths: conversational interface, specific population focus; Weaknesses: narrow demographic scope |

## Relevant Industry Standards or Protocols

- **Social Security Act, Title XIX/XXI** — Medicaid and CHIP eligibility rules; MAGI methodology mandates for income calculation
- **ACA MAGI Rules (45 CFR Part 155)** — Modified Adjusted Gross Income rules governing marketplace and Medicaid eligibility screening logic
- **IRS Disclosure Rules (IRC § 6103)** — Restrictions on sharing tax data for benefit-matching; relevant to income-verification integrations
- **HIPAA** — Privacy requirements when health information is collected during benefits screening
- **ADA / Section 508** — Accessibility for public-facing screener interfaces
- **Rules as Code (RaC) / Policy-as-Code standards** — Emerging OECD-led framework for encoding benefit policy logic in machine-readable form to ensure consistency across systems

## Available Research Materials

1. Nava PBC (2024). *Experimenting with AI-Powered Tools in Public Benefits*. https://www.navapbc.com/case-studies/ai-tools-public-benefits
2. Digital Government Hub (2025). *AI-Powered Rules as Code: Experiments with Public Benefits Policy*. https://digitalgovernmenthub.org/publications/ai-powered-rules-as-code-experiments-with-public-benefits-policy-summary/
3. Microsoft Industry Blogs (2026). *Right Benefit, Right Person, Right Time: How AI is Reshaping Administration of Benefits Programs Worldwide*. https://www.microsoft.com/en-us/industry/blog/government/public-health-social-services/2026/03/04/right-benefit-right-person-right-time-how-ai-is-reshaping-administration-of-benefits-programs-worldwide/
4. Servos.io (2025). *Modernizing Public Assistance: How AI is Reinventing Eligibility Screening*. https://servos.io/blog/aieligibility
5. InsightAce Analytic (2026). *AI in Government and Public Services Market Demand Analysis Report*. https://www.insightaceanalytic.com/report/ai-in-government-and-public-services-market/2749
6. CMS (2025). *Fact Sheet: Pledges from Medicaid Technology Companies to Support Community Engagement Implementation*. https://www.cms.gov/newsroom/fact-sheets/fact-sheet-pledges-medicaid-technology-companies-support-community-engagement-implementation-related
7. BenefitsUSA (2026). *Federal Poverty Level 2026: Updated Income Guidelines for Benefits Eligibility*. https://benefitsusa.org/en/blog/federal-poverty-level-2026

## Market Research

**Market Size:** The AI in Government and Public Services Market is forecast to grow at 16% CAGR through 2035; benefits-eligibility software is a significant segment of that, driven by Medicaid, SNAP, housing, and childcare programs. US federal and state benefit programs collectively distribute over $2 trillion annually, making even modest automation investments substantial.

**Funding:** IBM Curam commands large government contracts; Findhelp raised over $100 million; Nava PBC operates on government contracts and grants; numerous civic-tech startups are entering with AI-native approaches backed by venture and philanthropic capital.

**Pricing Landscape:** Government-procured systems range from hundreds of thousands to tens of millions of dollars; commercial SaaS screeners for nonprofits and smaller agencies range from $10,000–$200,000/year.

**Key Buyer Personas:** State Medicaid and SNAP agency directors, county health and human services departments, nonprofit social-service navigators, health systems with community benefit programs.

**Notable Trends:** Rules-as-code adoption accelerating to reduce costly policy-update cycles; AI chatbots replacing paper screeners; proactive data-matching to reach eligible but unenrolled populations; federal policy volatility in 2025–2026 increasing demand for configurable, rapidly updatable rule engines; SNAP and Medicaid unwinding driving urgency.

## AI-Native Opportunity

- Conversational AI intake that guides applicants through eligibility questions in plain language, adapting to their answers in real time rather than presenting static form trees
- Automated policy-rule translation (Rules as Code) using LLMs to convert statute and regulation text into executable eligibility logic, cutting update cycles from months to days
- Cross-program benefit bundling: AI identifies all programs for which a household qualifies in a single session and generates a prioritised application roadmap
- Document-understanding AI to extract income, household composition, and asset data from uploaded documents (pay stubs, tax returns) to pre-populate applications
- Predictive outreach: machine-learning models identify likely-eligible non-participants from administrative data and trigger proactive enrollment campaigns
