# Contact Enrichment Engine

A B2B contact and company data enrichment platform combining high-match-rate data aggregation with waterfall logic, GDPR compliance, and flexible API access—serving SMBs and mid-market teams who need rich contact data without ZoomInfo's pricing lock-in.

## Problem Statement

Contact enrichment is expensive and opaque:
- **ZoomInfo** dominates with 260M+ contacts and 85% match rates, but rigid annual contracts ($14,900+/year) and GDPR concerns lock out price-sensitive buyers
- **Apollo** offers better value ($49–$119/user/month) and combined data + engagement, but data quality lags ZoomInfo for enterprise contacts
- **Waterfall approaches** (Clay, FullEnrich) maximize fill rates by chaining multiple providers, but require orchestration expertise
- **GDPR compliance concerns** plague most commercial enrichment (ZoomInfo's legitimate-interest basis increasingly scrutinised)

## Key Differentiators

### Waterfall Enrichment for Maximum Fill Rates
- Chain across 75+ data providers (Apollo, PDL, Hunter, LinkedIn, etc.) to maximise hit rates
- Intelligent routing: use highest-confidence source first, fall back to secondary providers
- Customisable enrichment pipelines for SMB/mid-market use cases
- Cost per contact as low as $0.004/record at scale vs. $0.47–$0.71 with single-provider services

### Real-Time Job-Change Intelligence
- Automatically notify when an enriched contact switches roles or companies
- Trigger re-engagement sequences without manual monitoring
- Only integration feature differentiating contact enrichment from enrichment + engagement tools (Apollo)

### GDPR-Compliant Data Sourcing
- Enrich from permissioned and public data sources with auditable provenance
- Replace legitimate-interest concerns plaguing ZoomInfo with lawful basis transparency
- Appeals to European and regulated-industry buyers

### Flexible Deployment
- API-first architecture: embed in custom pipelines or use as standalone enrichment service
- Self-hosted option for data-sensitive teams
- CRM integrations: Salesforce, HubSpot, Pipedrive auto-update enrichment fields

## Market Context

- **Market size**: Data enrichment market $3.04–$3.26B in 2026, growing at CAGR 8.75–12.5% through 2035
- **Key competitors**: ZoomInfo ($1.2B+ revenue, 85% match rate), Clay ($185–$800/mo, power-user favorite), PDL ($98/mo API)
- **Positioning**: Best-value waterfall enrichment for RevOps and growth engineering teams; GDPR-safe alternative to ZoomInfo

## Recommended MVP Scope

- Real-time contact enrichment: email, direct dial, mobile, title, seniority, company data
- Waterfall logic across 3–5 primary data sources with configurable fallback chains
- Company-level enrichment: employee count, revenue, industry, technology stack
- Job-change detection with automatic alert/sequence triggering
- Bulk enrichment via CSV and API
- CRM bi-directional sync (Salesforce, HubSpot native)
- GDPR compliance dashboard: data lineage and lawful basis tracking

## Resources

**Research Materials**: [research.md](research.md) | **Feature Analysis**: [features.md](features.md)

## Target Users

- SDR/BDR managers and RevOps leads eliminating manual research and increasing rep productivity
- Marketing operations teams enriching inbound form fills for nurture sequencing
- Growth engineers building automated pipeline systems on top of enrichment APIs
- Data engineering teams building internal enrichment pipelines for privacy-sensitive deployments
