# Benefits Eligibility Screener

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An open-source, AI-native screener that helps people discover every government, tax, and nonprofit benefit they qualify for in a single conversation.

The Benefits Eligibility Screener is a multi-program eligibility platform for residents, social-service navigators, and public agencies. It combines a Rules-as-Code back end with conversational AI intake to replace static form trees and multi-month policy-update cycles with plain-language interviews and rapidly maintainable rule sets.

---

## Why Benefits Eligibility Screener?

- Incumbent enterprise platforms such as IBM Cúram require multi-million-dollar government contracts and weeks-to-months change-management cycles for even minor policy rule updates.
- Salesforce Public Sector Solutions delivers modern UX but carries per-user enterprise licensing that is prohibitive for smaller agencies and nonprofits, and rule changes still require Salesforce expertise.
- Existing open-source tools (PolicyEngine, MyFriendBen, ACCESS NYC, OpenFisca, Nava OSCER) each solve part of the problem but lack a unified offering that combines cross-program eligibility, AI conversational intake, and document understanding.
- Findhelp's strength is community-resource referral rather than deterministic eligibility determination, leaving a gap for tools that compute dollar-value benefits across federal, state, county, and nonprofit programs.
- Federal policy volatility in 2025–2026, plus SNAP and Medicaid unwinding, is driving urgent demand for configurable, rapidly updatable rule engines that small agencies and CBOs can actually afford to run.

---

## Key Features

### Multi-Program Eligibility Screening

- Questionnaire-driven screening covering SNAP, Medicaid/CHIP, SSI, EITC, WIC, LIHEAP, and housing vouchers
- Plain-language progressive-disclosure questions that adapt to prior answers
- Benefit dollar-value estimation with time-to-apply for each matched program
- Cross-program application roadmap that prioritises next steps for the household

### Rules-as-Code Engine

- Versioned, auditable eligibility logic linked to regulatory sources (Title XIX/XXI, ACA MAGI, federal poverty levels)
- YAML/Python-style rule definitions inspired by OpenFisca and PolicyEngine for interoperability
- Quarterly update process for poverty lines, benefit thresholds, and program changes
- Public REST API so third parties can build their own front ends against the rules engine

### Conversational AI Intake

- LLM-powered natural-language interview replacing rigid form trees
- AI-generated explanations of why a household qualifies (or does not) for each program
- Document upload with AI extraction of income, household, and asset data from pay stubs and tax returns
- Predictive eligibility scoring from partial data to reduce questionnaire abandonment

### Accessibility and Reach

- Mobile-first front end meeting WCAG 2.2 / Section 508 / ADA requirements
- English and Spanish at launch, with Chinese, Vietnamese, Arabic, Portuguese, and French Creole on the roadmap
- Embeddable widget and white-label deployment for community partners and counties
- Closed-loop referral integration with community-resource networks

### Policy and Outreach Tooling

- Cliff-effect visualisation showing how income changes affect benefit eligibility
- Policy reform simulation to compare current law against proposed changes
- Proactive outreach scoring to identify likely-eligible non-participants from administrative data
- Admin UI for non-technical policy analysts to update thresholds without developer intervention

---

## AI-Native Advantage

AI capabilities turn this from a static screener into a living policy system. LLMs translate statute and regulation text into executable Rules-as-Code, cutting update cycles from months to days. Conversational intake replaces form trees with adaptive interviews in plain language, while document-understanding models extract income, household composition, and asset data directly from uploaded pay stubs and tax returns. Machine-learning models also identify likely-eligible non-participants from administrative data to drive proactive enrollment campaigns.

---

## Tech Stack & Deployment

The platform is designed as a Rules-as-Code back end with a public REST API and an accessible USWDS-style front end. Rule definitions follow OpenFisca- and PolicyEngine-compatible formats so existing rule libraries can be imported and exported. Deployment targets cloud-agnostic Infrastructure as Code for AWS, Azure, and GCP, with embeddable widgets and white-label configurations for community partners. Standards alignment includes ACA MAGI rules (45 CFR Part 155), HIPAA, Section 508, and the OECD-led Rules-as-Code framework.

---

## Market Context

US federal and state benefit programs collectively distribute over $2 trillion annually, and the AI in Government and Public Services market is forecast to grow at 16% CAGR through 2035 (InsightAce Analytic, 2026). Government-procured eligibility systems range from hundreds of thousands to tens of millions of dollars, while commercial SaaS screeners run $10,000–$200,000/year. Primary buyers are state Medicaid and SNAP agency directors, county health and human services departments, nonprofit social-service navigators, and health systems operating community benefit programs.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
