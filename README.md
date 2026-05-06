# Constituent Relationship Management

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, open-source CRM for government agencies, elected offices, and civic organisations to manage case management, service requests, and constituent outreach.

Constituent Relationship Management (CRM) systems unify constituent profiles, case management, and multi-channel outreach across government agencies and elected offices. This project targets municipal governments, state agencies, elected officials, and nonprofit advocacy organisations who currently choose between expensive proprietary platforms and a single ageing open-source option that lacks AI capabilities.

---

## Why Constituent Relationship Management?

- Incumbent government CRM platforms (Granicus, OpenGov, Salesforce Government Cloud, Dynamics 365) are priced at $25,000–$300,000/year for mid-sized agencies, with enterprise deployments running into seven figures — pricing that excludes smaller municipalities and most elected offices.
- The leading open-source alternative, CiviCRM (AGPL-3.0), has no native AI features as of 2026 and a constituent-facing portal that requires significant theming to reach consumer-grade usability.
- Lightweight tools for elected officials (CivicTrack, FiscalNote Fireside) are proprietary, expose no documented public API, and are limited to specific workflow segments rather than spanning agency and elected-office use cases.
- AI capabilities present in incumbents (Einstein, Copilot, Granicus AI agents) are black-box and gated behind enterprise licensing; agencies face accountability requirements that demand auditability of routing decisions.
- Cross-agency constituent journey visibility is absent across the entire market — every agency operates a silo, even within the same jurisdiction.

---

## Key Features

### Unified Constituent Record and Intake

- Unified constituent profile with full interaction history across web, email, phone, SMS, mobile app, and web form
- Multi-channel intake with automatic de-duplication and consolidation across channels
- Role-based access control with department-level data scoping
- Constituent self-service portal for request submission and real-time status tracking

### Case Management and Routing

- Case creation, assignment, routing, and closure workflow with audit trail
- AI-powered classification, priority scoring, and basic deduplication on inbound mail
- GIS-based routing using geographic boundary data for automatic case assignment
- AI escalation risk scoring and SLA breach prediction with supervisor alerting

### Outbound Communications

- Templated email and SMS responses with AI-assisted draft generation
- Audience segmentation and targeted outbound campaign tools
- Reporting dashboard: case volume, response time, resolution rate by department

### Standards and Interoperability

- Open311 GeoReport v2 API endpoint for interoperability with third-party civic reporting apps (FixMyStreet and similar)
- Accessibility compliance module: WCAG 2.1 AA audit tooling for the constituent portal
- NIEM-aligned data exchange for case and service request sharing between agencies

### Advanced Capabilities (Backlog)

- Customer data platform: cross-channel behaviour analytics and sentiment scoring
- RAG-powered knowledge base surfacing for case workers during case handling
- Virtual Town Hall / mass engagement broadcasting module
- Cross-agency constituent journey view with federated identity (login.gov or equivalent)
- Policy tracking integration for elected-office deployments

---

## AI-Native Advantage

AI is built into the core workflow rather than bolted on as a premium tier. Transformer-based classifiers handle informal constituent language for inbound routing far better than rule-based keyword matching; LLM drafting with policy guardrails reduces case-worker response time on templated and semi-custom replies; NLP analysis of communication tone flags cases at risk of escalation before complaints reach elected officials or media; and ML models trained on historical case data predict resolution time and SLA breach risk for proactive supervisor intervention. Routing decisions are designed to be auditable and explainable to meet government accountability requirements.

---

## Tech Stack & Deployment

The project targets self-hosted and cloud deployment with FedRAMP-compatible architecture as a goal for state and federal procurement. Open standards are first-class: Open311 GeoReport v2 for service request interoperability, NIEM for cross-agency data exchange, Section 508 / WCAG 2.1 AA for accessibility, and OAuth 2.0 for authentication. A REST API surface (modelled on patterns from CiviCRM APIv4 and Salesforce Public Sector Solutions) provides full entity access for integrators, with SDKs and connectors for common payment processors, SMS gateways, and mailing providers.

---

## Market Context

The broader CRM market reached $101.83 billion in 2026 at a 14.6% CAGR, with government and education among the fastest-growing verticals. Government-dedicated platforms charge $25,000–$300,000/year depending on agency size; enterprise platforms (Salesforce Government Cloud, Dynamics 365) run seven-figure contracts. Primary buyers include municipal governments managing 311-style service requests, state agencies coordinating constituent outreach, elected officials managing casework, and nonprofits and advocacy groups managing stakeholder relationships.

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
