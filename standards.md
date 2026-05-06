# Standards & API Reference

> Project: Contact Enrichment Engine · Generated: 2026-05-06

## Industry Standards & Specifications

### IETF / RFC Standards

**RFC 5321 — Simple Mail Transfer Protocol (SMTP)**
- URL: https://www.rfc-editor.org/rfc/rfc5321.html
- Every email enrichment engine must validate deliverability using SMTP. RFC 5321 defines the exact command sequence (EHLO → MAIL FROM → RCPT TO) used to probe a recipient mail server without sending a message. The MX record lookup that precedes an SMTP probe is also governed by this standard: if MX records exist for a domain, only the MX targets may be used for mail delivery (A/AAAA fallback is prohibited).

**RFC 5322 — Internet Message Format**
- URL: https://www.rfc-editor.org/rfc/rfc5322.html
- Defines the syntax of email address local-parts and headers. Enrichment engines must validate candidate email addresses against this spec before returning them to callers; an address that is syntactically invalid per RFC 5322 will always bounce.

**RFC 6350 — vCard Format Specification (vCard 4.0)**
- URL: https://www.rfc-editor.org/rfc/rfc6350.html
- The canonical IETF standard for contact data interchange. Enrichment engines that need to export or import contact records in a portable format should emit vCard 4.0. The standard covers structured name, organisation, email, telephone, geographic location, and URLs, and defines a URI scheme for each field type.

**RFC 9554 — vCard Format Extensions for JSContact**
- URL: https://datatracker.ietf.org/doc/html/rfc9554
- Updates RFC 6350 by adding new properties and aligning vCard with the JSContact format (RFC 8984). Relevant for enrichment engines targeting CalDAV/CardDAV server ecosystems (e.g., Apple iCloud, Google Contacts, Fastmail).

**RFC 6749 — The OAuth 2.0 Authorization Framework**
- URL: https://www.rfc-editor.org/rfc/rfc6749.html
- The authentication standard used by all major enrichment platform APIs (ZoomInfo, HubSpot, Cognism). Any enrichment engine offering an API consumed by third-party applications should implement OAuth 2.0 client-credentials flow for server-to-server access and authorisation-code flow for user-delegated access. ZoomInfo uses a username+password → token model over this framework.

**RFC 7519 — JSON Web Token (JWT)**
- URL: https://www.rfc-editor.org/rfc/rfc7519.html
- Defines a compact, URL-safe token format for securely transmitting claims between parties. Enrichment APIs that use OAuth 2.0 typically issue JWTs as access tokens; the token encodes the caller's scopes, expiry, and identity without requiring a server-side session store.

**RFC 7946 — The GeoJSON Format**
- URL: https://www.rfc-editor.org/rfc/rfc7946.html
- GeoJSON is the standard for encoding geographic data structures in JSON. Enrichment output that includes office location, headquarters coordinates, or regional coverage data should use GeoJSON Point features for geographic fields to enable downstream mapping and geospatial filtering.

**ICANN RDAP (Registration Data Access Protocol)**
- URL: https://www.icann.org/rdap/
- IETF standard: https://datatracker.ietf.org/wg/regext/documents/
- RDAP is the modern, JSON-based successor to WHOIS for querying domain registration data. Unlike WHOIS, RDAP delivers structured RESTful responses (base URL: `https://rdap.verisign.com/com/v1/`) and supports differentiated access levels. Enrichment engines can use RDAP to infer company age (domain registration date), registrant organisation, and MX provider from domain records — all with a clear legal basis (public registry data).

### W3C Standards

**Schema.org — Person and Organization Vocabularies**
- Person: https://schema.org/Person
- Organization: https://schema.org/Organization
- ContactPoint: https://schema.org/ContactPoint
- Schema.org defines the semantic vocabulary for representing people, organisations, and contact points as structured data (typically serialised as JSON-LD). Enrichment engines returning contact records should model their output schema against Schema.org Person/Organization to enable interoperability with knowledge graph systems, SEO pipelines, and downstream data consumers that expect structured-data conventions. Key properties include `name`, `jobTitle`, `worksFor`, `email`, `telephone`, `address`, and `sameAs` (for linking to LinkedIn, GitHub, Twitter profiles).

**JSON-LD 1.1**
- URL: https://www.w3.org/TR/json-ld11/
- JSON-LD is the W3C standard for linking JSON data to semantic vocabularies such as Schema.org. Contact enrichment APIs that embed JSON-LD context in their responses make their output machine-interpretable without requiring the consumer to know the enrichment provider's proprietary field names.

### Data Model & API Specifications

**OpenAPI Specification 3.1 (OAS 3.1)**
- URL: https://spec.openapis.org/oas/v3.1.0.html
- The industry standard for documenting RESTful APIs. OAS 3.1 is fully compatible with JSON Schema Draft 2020-12, enabling type-safe data model definitions for enrichment request and response payloads. PDL, Apollo, Lusha, Cognism, and ZoomInfo all publish or reference OAS-compatible API definitions. Any new open-source enrichment engine should publish an OpenAPI 3.1 specification as part of its public API contract.

**JSON Schema Draft 2020-12**
- URL: https://json-schema.org/specification.html
- The data-model definition language underlying OAS 3.1. JSON Schema defines the structure of enrichment request bodies (e.g., `{email, firstName, lastName, company}`) and response payloads (e.g., `{title, seniority, department, verifiedEmail, confidence}`). Field-level validation, required properties, and enumerated values for categorical fields (seniority level, department name) should all be expressed as JSON Schema.

### Regulatory & Compliance Frameworks

**GDPR — General Data Protection Regulation (EU) 2016/679**
- URL: https://gdpr-info.eu/
- The EU's primary data protection regulation. Three articles are directly relevant to contact enrichment:
  - **Article 6 (Lawful Basis)**: Processing of personal data (including B2B contact enrichment) requires a documented lawful basis. Most enrichment vendors rely on Article 6(1)(f) — legitimate interest — but this is increasingly scrutinised by European Data Protection Authorities.
  - **Article 14 (Transparency)**: When contact data is obtained from a third-party source (enrichment engine, purchased list), the data controller must inform the data subject at the time of first contact — including the data source, purpose, legal basis, and right to object.
  - **Article 21 (Right to Object)**: Data subjects may object to processing based on legitimate interest; controllers must cease processing unless they can demonstrate "compelling legitimate grounds" that override the individual's interests.
  - In 2026, a Legitimate Interest Assessment (LIA) documenting the three-part test (purpose, necessity, balancing) is the minimum documentation required to rely on Article 6(1)(f) for B2B enrichment.

**CCPA / CPRA — California Consumer Privacy Act / California Privacy Rights Act**
- URL: https://oag.ca.gov/privacy/ccpa
- Since January 2023 (CPRA), the B2B data exemption was removed: California residents acting in a business capacity now have full CCPA privacy rights. From January 2026, new requirements include mandatory risk assessments for high-risk processing, phased-in cybersecurity audits, and rules for Automated Decision-Making Technology (ADMT). Enrichment engines must support a "Do Not Sell or Share" opt-out mechanism for California-resident contacts and maintain audit trails of data provenance.

**ISO/IEC 27001:2022 — Information Security Management Systems**
- URL: https://www.iso.org/standard/27001
- The international standard for information security management. European enterprise buyers increasingly require ISO 27001 certification as a contractual prerequisite for any B2B data vendor. ISO 27001:2022 (the current version; all audits are now against the 2022 edition) introduces enriched control attributes and expanded cloud and AI-related controls. An enrichment engine handling personal data at scale should pursue ISO 27001 certification to unlock enterprise procurement.

---

## Similar Products — Developer Documentation & APIs

### People Data Labs (PDL)

- **Description:** API-first B2B data infrastructure provider with 3B+ person and company records. Widely used as the data backbone by enrichment aggregators (Clay, FullEnrich). Offers the most developer-friendly API in the category with free tier access.
- **API Documentation:** https://docs.peopledatalabs.com/
- **SDKs/Libraries:**
  - Python: https://github.com/peopledatalabs/peopledatalabs-python
  - JavaScript/TypeScript: https://github.com/peopledatalabs/peopledatalabs-js
  - Go: https://github.com/peopledatalabs/peopledatalabs-go
  - Ruby: https://github.com/peopledatalabs/peopledatalabs-ruby
  - Rust: https://github.com/peopledatalabs/peopledatalabs-rust
- **Developer Guide:** https://docs.peopledatalabs.com/docs/quickstart
- **OpenAPI Specification:** https://github.com/peopledatalabs/openAPI-specifications
- **Standards:** REST/JSON; OpenAPI specification published on GitHub
- **Authentication:** API Key (passed as `api_key` query parameter or `X-API-Key` header)
- **Key Endpoints:** Person Enrichment (`/v5/person/enrich`), Company Enrichment (`/v5/company/enrich`), Person Search (`/v5/person/search` — SQL-like query syntax), Bulk Person Enrichment

### Apollo.io

- **Description:** 275M+ contact database combined with sales engagement. Offers both single-record and bulk enrichment APIs. Authentication via API key; integrates with Salesforce, HubSpot, Pipedrive. Provides native waterfall enrichment across connected third-party data sources.
- **API Documentation:** https://docs.apollo.io/
- **API Overview:** https://docs.apollo.io/docs/api-overview
- **Key Endpoints:**
  - People Enrichment: https://docs.apollo.io/reference/people-enrichment
  - Bulk People Enrichment: https://docs.apollo.io/reference/bulk-people-enrichment
  - Organization Enrichment: https://docs.apollo.io/reference/organization-enrichment
- **SDKs/Libraries:** No official SDK; language-agnostic REST API usable from any HTTP client
- **Developer Guide:** https://knowledge.apollo.io/hc/en-us/articles/4416173158541-Use-the-Apollo-REST-API
- **Standards:** REST/JSON; API key authentication; bulk endpoints limited to 10 records per call
- **Authentication:** API Key (passed as `api_key` in request body or `x-api-key` header)

### ZoomInfo

- **Description:** Market-leading B2B data platform with 260M+ contacts. Two-tier API: Legacy Enterprise API (being deprecated) and new API. Standard suite includes Search API (filter-based) and Enrich API (full-profile retrieval by matched identifier).
- **API Documentation:** https://docs.zoominfo.com/
- **Legacy API Docs:** https://api-docs.zoominfo.com/
- **SDKs/Libraries:** No official SDK; REST API; Python examples available in third-party community guides
- **Developer Guide:** https://aeroleads.com/blog/zoominfo-api-documentation-developers-getting-started/
- **Standards:** REST/JSON; token-based authentication; tokens expire after 60 minutes and must be refreshed
- **Authentication:** OAuth 2.0-style token endpoint (`POST https://api.zoominfo.com/oauth/token` with username/password → Bearer token)

### Cognism

- **Description:** European-headquartered B2B data provider; strongest GDPR compliance story and best mobile number accuracy in EU markets. Enrich API performs one-to-one contact matching against their proprietary dataset.
- **API Documentation:** https://developers.cognism.com/
- **Help Center API Docs:** https://help.cognism.com/hc/en-gb (Enrich, Redeem, and Search APIs documented)
- **SDKs/Libraries:** No official SDK; REST API consumable from any HTTP client or ETL tool
- **Developer Guide:** https://help.cognism.com/hc/en-gb/articles/7703988771730-Using-Enrich-and-Redeem-APIs
- **Standards:** REST/JSON; bearer token authentication; entitlement-based access controls for GDPR-compliant Diamond Data fields
- **Authentication:** Bearer token (configured via developer dashboard)

### Hunter.io

- **Description:** Email finder and verifier with domain-based enrichment. Finds email addresses for a given domain (Domain Search), derives emails from name+domain (Email Finder), and verifies deliverability with confidence scores (Email Verifier). Person Enrichment converts an email or LinkedIn handle into a fully enriched record.
- **API Documentation:** https://hunter.io/api-documentation
- **Email Finder API:** https://hunter.io/api/email-finder
- **Email Verifier API:** https://hunter.io/api/email-verifier
- **SDKs/Libraries:** Community libraries in Python, Ruby, PHP; no official SDK
- **Developer Guide:** https://help.hunter.io/en/articles/12633706-how-to-verify-emails-via-api-with-code-examples
- **Standards:** REST/JSON; rate-limited to 10 requests/sec (300/min) for verification endpoint
- **Authentication:** API Key (`api_key` query parameter, `X-API-KEY` header, or `Authorization: Bearer` header)

### Lusha

- **Description:** B2B data API for prospecting and enrichment with person and company endpoints. Offers real-time signals (job changes, funding events). All responses in JSON over HTTPS; API key access restricted to Premium and Scale plans.
- **API Documentation:** https://docs.lusha.com/
- **Person Enrichment Endpoint:** https://docs.lusha.com/apis/openapi/person-enrichment
- **Company Enrichment Endpoint:** https://docs.lusha.com/apis/openapi/enrichment
- **Tutorials:** https://docs.lusha.com/tutorials
- **Standards:** REST/JSON; OpenAPI specification available within docs portal
- **Authentication:** API Key required for all requests

### Crunchbase

- **Description:** Company-level enrichment provider covering funding rounds, founding date, executive team, acquisition history, technographic signals, and investor relationships. API v4.0 provides Entity Lookup endpoints for organisations, people, funding rounds, and acquisitions. No person-level email/phone data.
- **API Documentation:** https://data.crunchbase.com/docs/using-the-api
- **CRM Enrichment Guide:** https://data.crunchbase.com/docs/crm-enrichment
- **Developer Portal:** https://developer.edgar-online.com/docs (EDGAR Online integration)
- **SDKs/Libraries:** No official SDK; REST API with CSV bulk export option
- **Standards:** REST/JSON; token-based authentication; API v4.0 current
- **Authentication:** API Key passed as `user_key` query parameter

### HubSpot CRM (Enrichment Integration Target)

- **Description:** HubSpot is the most common CRM target for enrichment write-back. The 2026-03 API version introduced date-based versioning (replacing `/crm/v3/` paths). Enrichment engines must support creating and updating Contact and Company objects, handling deduplication via `hs_unique_email_address` and `hs_object_id`.
- **API Documentation:** https://developers.hubspot.com/docs/api-reference/latest/overview
- **Integration Overview:** https://developers.hubspot.com/
- **SDKs/Libraries:**
  - Node.js: `@hubspot/api-client`
  - Python: `hubspot-api-client`
  - PHP, Ruby, Java SDKs available on HubSpot GitHub
- **Developer Guide:** https://www.getknit.dev/blog/hubspot-api-directory-oD0RSt
- **Standards:** REST/JSON; date-versioned endpoints (e.g., `/crm/objects/2026-03/contacts`); OAuth 2.0 for public apps; Private App access tokens for internal integrations
- **Authentication:** Private App token (recommended) or OAuth 2.0 authorisation-code flow

### SEC EDGAR (Public Company Data Source)

- **Description:** U.S. Securities and Exchange Commission's public data API. Provides programmatic access to all public company filings: 10-K, 10-Q, 8-K, proxy statements. No API key required; only a User-Agent header identifying the requester. Primary relevance to the enrichment engine: free, legally clean source of firmographic data (employee count from 10-K, executive names from proxy statements, SIC industry code, financial figures) for US public companies.
- **API Documentation:** https://www.sec.gov/search-filings/edgar-application-programming-interfaces
- **Developer Resources:** https://www.sec.gov/about/developer-resources
- **Data Endpoint Base:** https://data.sec.gov/
- **Key Endpoints:** `/api/xbrl/companyfacts/{CIK}`, `/api/xbrl/companyconcept/{CIK}`, `/submissions/{CIK}.json`
- **SDKs/Libraries:** `edgartools` (Python, open source, MIT); `sec-api` (Python, commercial)
- **Standards:** REST/JSON; XBRL for financial data; no authentication required; User-Agent header mandatory
- **Authentication:** None required (public API)

### Companies House UK (Public Company Registry)

- **Description:** UK government's official company registry API. Provides legal name, company number, status, registered address, director names and appointment history, and filing history for all UK limited companies. One of the cleanest open registry APIs globally; well-documented and free to use for lookup purposes.
- **API Documentation:** https://developer.company-information.service.gov.uk/overview
- **Developer Portal:** https://developer.company-information.service.gov.uk/
- **API Specification:** https://developer-specs.company-information.service.gov.uk/companies-house-public-data-api/reference
- **SDKs/Libraries:** No official SDK; REST API consumable from any HTTP client
- **Getting Started:** https://developer.company-information.service.gov.uk/get-started
- **Standards:** REST/JSON; versioned API paths; rate limits apply per API key
- **Authentication:** API Key (HTTP Basic Auth with key as username and blank password)

---

## Notes

**Clearbit / Breeze Intelligence API Deprecation:** Clearbit's standalone enrichment API (formerly at `/v2/companies/find`, `/v2/people/find`) is effectively sunsetted following HubSpot's acquisition in December 2023. In 2026, Clearbit data is exclusively accessible via HubSpot Breeze Intelligence as a CRM-native feature; there is no independent developer API pathway. Any enrichment engine previously depending on Clearbit endpoints should migrate to PDL, Hunter, or Apollo as API data sources.

**Clay's API Status:** Clay does not expose a public API. It operates as a SaaS enrichment orchestration UI and cannot be called programmatically by third-party systems. Developers needing similar waterfall orchestration must build it themselves against the underlying source APIs (PDL, Apollo, Hunter, etc.) or use webhook-based workarounds.

**LinkedIn API Constraints:** LinkedIn's official API does not permit B2B contact enrichment use cases for commercial third-party applications. The Member Data Portability API and Marketing API are scoped to first-party use cases (users accessing their own data, ad targeting). Any enrichment engine relying on LinkedIn data must use compliant access paths (e.g., official Partner Programme integration or data licensed directly from LinkedIn). The hiQ v. LinkedIn litigation, while not fully settled, does not provide a safe harbour for bulk automated enrichment.

**Emerging Standard — MCP Server for Enrichment:** The Model Context Protocol (MCP) enables AI agents to call enrichment APIs as tools. PDL, Apollo, and community contributors have begun publishing MCP server wrappers that expose enrichment endpoints to LLM agents (e.g., Claude, GPT). An open-source enrichment engine publishing an MCP server specification would make it natively consumable by AI agent frameworks without requiring custom integration code.
