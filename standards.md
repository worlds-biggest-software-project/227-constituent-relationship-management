# Standards & API Reference

> Project: Constituent Relationship Management · Generated: 2026-05-03

## Industry Standards & Specifications

### ISO Standards

**ISO/IEC 27001:2022 — Information Security Management Systems**
- URL: https://www.iso.org/standard/27001
- The primary international standard for information security management; establishes requirements for protecting sensitive government data (citizen records, case files, national databases). Government procurement increasingly requires ISO 27001 certification from SaaS vendors alongside FedRAMP.

**ISO/IEC 18013-5 — Mobile Driver's Licence (mDL)**
- URL: https://blog.pacificcert.com/iso-iec-18013-19794-digital-identity-global-standards/
- The mDL standard defines how governments issue and verify digital identity credentials on mobile devices. Relevant to constituent portal authentication where a government-issued digital ID is used to verify constituent identity before case submission or benefit access.

**ISO/IEC 27000 Family — Information Security Management**
- URL: https://www.iso.org/standard/iso-iec-27000-family
- Suite of standards covering information security controls (27002), risk management (27005), privacy (27701), and cloud security (27017/27018); collectively applicable to CRM deployments handling personally identifiable constituent data.

---

### W3C & IETF Standards

**WCAG 2.1 Level AA — Web Content Accessibility Guidelines**
- URL: https://www.w3.org/TR/WCAG21/
- Mandatory accessibility standard for US state and local government websites and applications. The DOJ ADA Title II rule (April 2026) extended the compliance deadline for large jurisdictions to April 26, 2027. Constituent-facing portals must meet WCAG 2.1 AA for perceivability, operability, understandability, and robustness.

**RFC 6749 — The OAuth 2.0 Authorization Framework**
- URL: https://datatracker.ietf.org/doc/html/rfc6749
- The foundational standard for delegated authorisation used by all major government CRM platforms (Salesforce, Dynamics 365, OpenGov) for API access. Constituent portal authentication and third-party integrations depend on OAuth 2.0 flows.

**RFC 6750 — Bearer Token Usage (OAuth 2.0)**
- URL: https://datatracker.ietf.org/doc/html/rfc6750
- Defines how bearer tokens are transmitted in API requests; directly referenced by CiviCRM APIv4 authentication implementation.

**OpenID Connect 1.0**
- URL: https://openid.net/connect/
- Authentication layer on top of OAuth 2.0; the standard for constituent identity federation. Login.gov (US federal identity provider) implements OpenID Connect with the iGov Profile for government service authentication.

**iGov Profile for OAuth 2.0 — OpenID Foundation**
- URL: https://openid.net/specs/openid-igov-oauth2-1_0.html
- Government-specific profile of OAuth 2.0 mandating stronger security constraints (PKCE, restricted grant types, signed request objects) for government-facing applications. Login.gov conforms to this profile.

**RFC 7231 — HTTP/1.1 Semantics and Content**
- URL: https://datatracker.ietf.org/doc/html/rfc7231
- Baseline HTTP standard underlying all REST API interactions in constituent management platforms.

---

### Data Model & API Specifications

**Open311 GeoReport v2**
- URL: https://wiki.open311.org/GeoReport_v2/
- The canonical open standard for government service request management. Defines REST API methods (GET services, GET service definitions, POST service request, GET service requests) using XML or JSON. SeeClickFix 311 CRM implements this standard; it enables interoperability with third-party civic reporting apps (FixMyStreet, mySociety, etc.). Location is required for all service requests.

**Open311 Inquiry v1**
- URL: https://wiki.open311.org/Inquiry_v1/
- Extension of GeoReport v2 for non-location-based government service inquiries; broadens Open311 beyond physical infrastructure requests to general constituent inquiries and information requests.

**OpenAPI Specification v3.1.1**
- URL: https://spec.openapis.org/oas/v3.1.1.html
- The standard format for documenting REST APIs using JSON or YAML; used by OpenGov, Salesforce, and Dynamics 365 to publish API schemas. Full JSON Schema 2020-12 alignment in v3.1. An AI-native CRM should expose an OpenAPI 3.1-compliant API for integration.

**JSON Schema Draft 2020-12**
- URL: https://json-schema.org/specification
- The data validation specification underlying OpenAPI 3.1; governs the structure of request/response payloads in REST APIs. Use for constituent data model validation and schema-driven form generation.

**NIEM (National Information Exchange Model)**
- URL: https://www.niem.gov/
- US government data interoperability standard providing a common vocabulary for information exchange between agencies. Contains 17 domain-specific models covering justice, health, emergency management, and human services — all relevant to constituent case data exchange. NIEM is XML-based (NIEM-JSON support exists); mandated for inter-agency data sharing in many federal and state contexts.

**OData v4 (ISO/IEC 20802)**
- URL: https://www.odata.org/
- The REST protocol underlying Microsoft Dynamics 365 Web API; provides standardised query, filtering, and relationship traversal for CRM entity data. ISO/IEC 20802 part 1 and 2 standardise the protocol.

---

### Security & Authentication Standards

**FedRAMP (Federal Risk and Authorization Management Program)**
- URL: https://www.fedramp.gov/
- US government-wide cloud security authorisation programme. As of May 2026 there are 511 FedRAMP-authorised cloud services. CRM SaaS products serving US federal agencies must be FedRAMP authorised. FedRAMP 20x (2026) is modernising the authorisation process. Compliance requires independent 3PAO assessment against NIST SP 800-53 controls.

**NIST SP 800-53 Rev 5 — Security and Privacy Controls**
- URL: https://csrc.nist.gov/publications/detail/sp/800-53/rev-5/final
- The foundational control catalogue underlying FedRAMP. Constituent management systems handling PII must implement relevant controls covering access management, audit logging, incident response, and data protection.

**Section 508 (Rehabilitation Act) / ADA Title II**
- URL: https://www.section508.gov/
- US federal accessibility mandate; constituent-facing portals must meet WCAG 2.1 AA standards. The April 2026 DOJ rule extended Title II ADA compliance to all state and local government digital services.

**CJIS Security Policy**
- URL: https://le.fbi.gov/informational-resources/cjis/cjis-security-policy-resource-center
- FBI security policy applying to any CRM that integrates with criminal justice information. Relevant where constituent management intersects with law enforcement case data (e.g., public safety service requests, municipal court case management).

**GDPR (General Data Protection Regulation)**
- URL: https://gdpr-info.eu/
- EU data protection regulation; relevant for constituent management platforms deployed in or processing data of EU residents. Key articles: Article 5 (data minimisation), Article 17 (right to erasure), Article 20 (data portability) all create implementation requirements for CRM data models.

**CCPA (California Consumer Privacy Act)**
- URL: https://oag.ca.gov/privacy/ccpa
- California consumer data privacy law applicable to local government agencies operating in California; requires constituent data access and deletion request workflows within the CRM.

---

### MCP Server Specifications

**Model Context Protocol (MCP)**
- URL: https://modelcontextprotocol.io/
- Anthropic's open standard for connecting AI models to external data sources and tools. Relevant for an AI-native constituent CRM that exposes case data, constituent profiles, and service request history to AI agents via MCP servers. An MCP server wrapping the CRM's OpenAPI endpoints would allow AI models to query constituent records, draft responses, and update case status in real time.

**GovStack Building Blocks Specifications (GovSpecs 2.0)**
- URL: https://govstack.global/
- International framework for interoperable government digital infrastructure, used by 20+ countries. GovSpecs 2.0 (2025–2027) embeds AI-readiness, machine learning APIs, and ethical safeguards into government building block standards. GovStack's CRM and communication building blocks define API contracts relevant to constituent management platforms targeting non-US markets.

---

## Similar Products — Developer Documentation & APIs

### Granicus / GovDelivery Communications Cloud

- **Description:** Government communications and constituent engagement platform; GovDelivery API enables automated creation of communication topics, subscriber management, and message delivery.
- **API Documentation:** https://support.granicus.com/s/article/TMS-API-Documentation-Index
- **SDKs/Libraries:** .NET SDK (SOAP): https://github.com/Granicus/platform-api-net
- **Developer Guide:** https://support.granicus.com/s/article/API-Integration
- **Standards:** REST (GovDelivery TMS API); SOAP (Platform API 1.x)
- **Authentication:** API key; OAuth for newer endpoints

---

### OpenGov Public Service Platform

- **Description:** Government platform spanning CRM, permitting, asset management, and budgeting; Developer Portal provides REST API documentation and quickstart guides for integration.
- **API Documentation:** https://developer.opengov.com/docs/overview
- **SDKs/Libraries:** No official SDK; REST JSON API
- **Developer Guide:** https://developer.opengov.com/docs/quickstart
- **Standards:** REST/JSON; OpenAPI documented on developer portal
- **Authentication:** API Key (https://developer.opengov.com/docs/app-management/api-key)

---

### CiviCRM

- **Description:** Open-source constituent relationship management platform for nonprofits and civic organisations; APIv4 provides full entity access over HTTP/REST.
- **API Documentation:** https://docs.civicrm.org/dev/en/latest/api/v4/rest/
- **SDKs/Libraries:** No official SDK; PHP, JavaScript, and REST bindings documented. API Explorer at /civicrm/api4 in admin.
- **Developer Guide:** https://docs.civicrm.org/dev/en/latest/
- **Standards:** REST/JSON; APIv4 endpoint: `civicrm/ajax/api4/{ENTITY}/{ACTION}`
- **Authentication:** HTTP Parameter (`?_authx=Bearer+MY_API_KEY`) or `X-Civi-Auth` header; API key per user

---

### Salesforce Public Sector Solutions

- **Description:** FedRAMP-authorised Salesforce CRM with pre-built public sector data models for licensing, case management, benefits, and grants.
- **API Documentation:** https://developer.salesforce.com/docs/atlas.en-us.psc_api.meta/psc_api/api_psc_overview.htm
- **SDKs/Libraries:** Salesforce SDKs for Java, JavaScript, Python, .NET, iOS, Android (https://developer.salesforce.com/tools/sdks)
- **Developer Guide:** https://resources.docs.salesforce.com/latest/latest/en-us/sfdc/pdf/api_rest.pdf (v66.0 Spring '26)
- **Standards:** REST API v66.0; Bulk API 2.0; Pub/Sub API; OData; GraphQL; Composite API; OpenAPI
- **Authentication:** OAuth 2.0 Web Server Flow with My Domain–specific endpoints (recommended 2026)

---

### Microsoft Dynamics 365 Customer Engagement (Government)

- **Description:** Azure Government–hosted enterprise CRM; Web API provides OData v4 access to all CRM entities including constituent records and cases.
- **API Documentation:** https://learn.microsoft.com/en-us/rest/dynamics365/
- **SDKs/Libraries:** .NET SDK (Microsoft.Xrm.Sdk); JavaScript/TypeScript (Xrm client API); community SDKs for Python and Java
- **Developer Guide:** https://learn.microsoft.com/en-us/dynamics365/customerengagement/on-premises/developer/overview
- **Standards:** OData v4 (ISO/IEC 20802); REST/JSON; OpenAPI available via Power Platform
- **Authentication:** OAuth 2.0 (Azure AD / Entra ID); service principal for server-to-server

---

### SeeClickFix 311 CRM (CivicPlus)

- **Description:** Open-standards 311 service request management for local governments; implements Open311 GeoReport v2 and a proprietary REST API v2.
- **API Documentation:** https://dev.seeclickfix.com/ (API v2); https://seeclickfix.com/open311/v2/docs (Open311)
- **SDKs/Libraries:** No official SDK; REST JSON API
- **Developer Guide:** https://www.civicplus.help/seeclickfix/docs/seeclickfix-api-information
- **Standards:** Open311 GeoReport v2 (open standard); REST/JSON (proprietary v2 API)
- **Authentication:** CivicPlus Account Personal Access Token

---

### Login.gov (US Federal Identity Provider)

- **Description:** US federal identity service providing OpenID Connect–based authentication for government applications; enables constituent identity verification without per-agency account creation.
- **API Documentation:** https://developers.login.gov/oidc/getting-started/
- **SDKs/Libraries:** Community libraries; OIDC standard clients in all major languages
- **Developer Guide:** https://developers.login.gov/
- **Standards:** OpenID Connect 1.0; iGov Profile for OAuth 2.0; PKCE required
- **Authentication:** OpenID Connect with iGov Profile; authorization_code grant only (implicit grant prohibited)

---

### FiscalNote Fireside

- **Description:** Constituent management platform for legislative offices combining casework CRM with policy tracking; no public developer API documented.
- **API Documentation:** Not publicly available
- **SDKs/Libraries:** Not publicly available
- **Developer Guide:** Not publicly available
- **Standards:** Undocumented; assumed REST/JSON internally
- **Authentication:** Not publicly documented

---

## Notes

**Emerging standard — AI-native government CRM**: No existing standard governs AI model integration with government CRM systems. GovStack GovSpecs 2.0 is beginning to address this with AI-readiness guidelines for government building blocks, but concrete API standards for AI agent access to constituent records are not yet defined. MCP (Model Context Protocol) is the most credible candidate for this role given its adoption by major AI model providers.

**Open311 maturity ceiling**: Open311 GeoReport v2 is widely implemented for service request intake but covers only location-based requests. Open311 Inquiry v1 extends this to non-location requests, but neither standard covers constituent case management, outbound communications, or profile data — limiting its applicability as a comprehensive interoperability standard.

**FedRAMP 20x modernisation**: The FedRAMP 20x initiative (2026) is streamlining the authorisation process. Platforms targeting US federal agencies should monitor Phase 2 pilot outcomes, as the new continuous authorisation approach may lower barriers for AI-native government SaaS.

**NIEM adoption complexity**: NIEM is mandated for inter-agency data sharing but requires significant XML expertise. The NIEM JSON profile (https://niem.github.io/json/) is gaining adoption and would be the preferred approach for a modern CRM implementing NIEM-compliant data exchange.
