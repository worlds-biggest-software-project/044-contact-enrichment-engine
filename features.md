# Contact Enrichment Engine — Feature & Functionality Survey

> Candidate #44 · Researched: 2026-05-01

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| ZoomInfo | Commercial SaaS | Proprietary; from $14,900/yr | https://zoominfo.com |
| Apollo.io | Commercial SaaS | Proprietary; $49–$119/user/mo | https://apollo.io |
| Clearbit / HubSpot Breeze Intelligence | Commercial SaaS | Proprietary; HubSpot-bundled | https://hubspot.com/products/marketing/breeze |
| Cognism | Commercial SaaS | Proprietary; from ~$15k/yr | https://cognism.com |
| Clay | Commercial SaaS | Proprietary; $185–$800/mo | https://clay.com |
| People Data Labs (PDL) | Commercial API | Proprietary; from $98/mo | https://peopledatalabs.com |
| FullEnrich | Commercial SaaS | Proprietary; from ~$29/mo | https://fullenrich.com |
| Lusha | Commercial SaaS | Proprietary; free–$59/user/mo | https://lusha.com |
| Crunchbase | Commercial SaaS | Proprietary; from $29/user/mo | https://crunchbase.com |

## Feature Analysis by Solution

### ZoomInfo

**Core features**
- Proprietary database of 260M+ contacts and 100M+ companies maintained through web crawling, partnerships, and user-contributed data
- Real-time contact enrichment: email, direct dial, mobile number, title, department, and seniority
- Technology install data (FormComplete / TechTarget): detects technology stack in use at target companies
- Intent data: behavioural signals indicating which companies are actively researching relevant topics
- CRM enrichment: automatic update of stale contact fields in Salesforce/HubSpot on a scheduled basis

**Differentiating features**
- Largest proprietary B2B contact database; ~85% match rate for North American enterprise contacts
- Technology install detection embedded in enrichment output; no separate technographic provider needed
- FormComplete: shortens web forms by auto-filling known fields for recognised visitors, improving conversion rates
- Intent data from TechTarget content consumption; most comprehensive intent signal of any enrichment provider

**UX patterns**
- Web application for manual prospecting; Chrome extension for LinkedIn prospecting and contact export
- CRM-native enrichment panel: HubSpot and Salesforce integrations surface ZoomInfo data within existing CRM records
- Bulk enrichment via CSV upload for list processing

**Integration points**
- Native: Salesforce, HubSpot, Microsoft Dynamics, Marketo, Pardot
- Chrome extension for LinkedIn and company website enrichment
- REST API for programmatic enrichment
- Privacy consent signals feed HubSpot/Salesforce GDPR lawful-basis fields

**Known gaps**
- 219 G2 reviews cite inaccurate data as the primary complaint; accuracy drops significantly for private companies, smaller organisations, and EU contacts
- Rigid annual contracts with significant minimum spend; price-sensitive buyers excluded
- GDPR legitimate-interest basis increasingly scrutinised by EU data protection authorities
- Single-source: no fallback when ZoomInfo lacks a contact; no built-in waterfall logic

**Licence / IP notes**
- Fully proprietary; data collected via web crawling, scraping, and user contributions; GDPR compliance claimed via legitimate interest basis but contested

---

### Apollo.io

**Core features**
- 275M+ contact database with enrichment API and integrated sales engagement
- Email and direct dial enrichment with A-grade verification (98%+ deliverability confidence threshold)
- AI job-change alerts: automatic notification when an enriched contact switches roles or companies
- Company-level enrichment: employee count, revenue range, industry, technology stack
- Bulk enrichment via CSV and API; CRM enrichment for Salesforce and HubSpot

**Differentiating features**
- Best value for SMB/mid-market: combined database + sequencing + enrichment in one subscription reduces tool stack cost
- Job-change detection triggers re-engagement sequences automatically; most actionable dynamic enrichment in class
- Buyer intent signals integrated with enrichment output

**UX patterns**
- Unified prospecting UI: filter contacts by enriched attributes, export or sequence directly
- Chrome extension for LinkedIn enrichment and one-click CRM contact creation
- API documentation designed for developer-friendly programmatic enrichment

**Integration points**
- Native: Salesforce, HubSpot, Pipedrive, Zoho
- REST API; Zapier/Make connectors
- Chrome extension

**Known gaps**
- Data quality lower than ZoomInfo for enterprise contacts and EU regions
- Match rate (~75–80% for general B2B lists) below ZoomInfo (~85%) in independent benchmarks
- Less intent data depth than ZoomInfo's TechTarget integration

**Licence / IP notes**
- Fully proprietary; data sourced from web, public records, and user contributions

---

### Clearbit / HubSpot Breeze Intelligence

**Core features**
- Real-time contact enrichment on web form submission (form shortening: auto-fills known fields for returning visitors)
- Company-level enrichment: employee count, industry, revenue range, technology stack, social profiles
- Person-level enrichment: title, seniority, department, email, LinkedIn URL
- Buyer intent signals: identifies companies visiting the website and maps to enrichment profiles
- Native HubSpot integration: enriched data populates HubSpot contact and company records automatically

**Differentiating features**
- Real-time form shortening is a unique conversion optimisation use case; reduces form friction for known visitors
- Deeply embedded in HubSpot workflow engine; enrichment triggers can immediately activate nurture sequences
- Previously available as an independent API (Clearbit); now fully integrated into HubSpot Customer Platform

**UX patterns**
- Configured entirely within HubSpot settings; no separate UI needed for HubSpot users
- Enrichment credits consumed per lookup; usage dashboard within HubSpot billing

**Integration points**
- HubSpot-native (exclusive from 2024 acquisition onwards)
- Limited standalone API access post-acquisition

**Known gaps**
- Now locked into HubSpot ecosystem; non-HubSpot users must use alternative enrichment vendors
- Coverage of SMB contacts and non-North-American contacts thinner than ZoomInfo
- Cost per contact (~$0.71) higher than Apollo and PDL for equivalent data points

**Licence / IP notes**
- Fully proprietary; data sourced from web crawling and public sources; GDPR legitimate interest basis

---

### Cognism

**Core features**
- Diamond Data: phone-verified mobile numbers (human-verified, not scraped) for UK, DACH, and Benelux markets
- GDPR-compliant data collection methodology; phone-verified contacts include documented consent or legitimate interest basis with cleaner audit trail than ZoomInfo
- Contact enrichment: email, direct dial, mobile number, title, department, company attributes
- Buyer intent data (partnered with Bombora and G2): intent signal overlay on enriched contacts

**Differentiating features**
- Best mobile number accuracy in the European market; phone-verified Diamond Data is differentiated from scraped competitor data
- Strongest GDPR compliance story in the category; documented legal basis for each contact record
- Primary choice for outbound teams selling into European markets where ZoomInfo coverage and compliance are weakest

**UX patterns**
- Web prospecting UI with geographic, industry, and seniority filters
- Chrome extension for LinkedIn enrichment
- CRM enrichment for Salesforce and HubSpot

**Integration points**
- Native: Salesforce, HubSpot, Outreach, Salesloft
- Chrome extension for LinkedIn
- API access on higher plans

**Known gaps**
- Database smaller than ZoomInfo for North American contacts; not the primary choice for US-focused outbound
- Pricing ($15k–$25k/yr) excludes small teams despite superior GDPR compliance

**Licence / IP notes**
- Fully proprietary; GDPR compliance via legitimate interest with enhanced documentation; phone verification adds defensibility

---

### Clay

**Core features**
- Waterfall enrichment orchestrator: queries 150+ data providers sequentially until requested fields are filled, maximising fill rates without manual source selection
- AI-powered research: natural-language prompts instruct Clay to research prospects using web search, LinkedIn data, and integrated sources (e.g., "find the VP of Engineering's GitHub profile and summarise their recent open-source contributions")
- Spreadsheet-like interface for non-technical users to build enrichment pipelines visually
- Automated CRM enrichment: enrich and update Salesforce/HubSpot records on a scheduled or trigger basis
- Outreach integration: enriched contacts can be directly enrolled in email sequences via Apollo, Instantly, or Outreach connections

**Differentiating features**
- Highest fill rates in the category via multi-source waterfall logic (150+ providers vs. ZoomInfo's single database)
- AI research capabilities extend enrichment beyond structured data to unstructured insights (recent news, job postings analysis, GitHub activity)
- Most flexible enrichment platform; RevOps power users can build custom enrichment workflows without writing code
- Beloved by growth engineers and RevOps teams for bespoke pipeline automation

**UX patterns**
- Spreadsheet-like table UI with column-per-enrichment-source metaphor
- Natural-language AI research prompt builder for custom enrichment logic
- Workflow templates for common use cases (inbound lead enrichment, outbound prospect research, CRM cleanup)

**Integration points**
- 150+ data provider integrations (Apollo, PDL, Hunter, LinkedIn via partner, Crunchbase, Clearbit, etc.)
- CRM: Salesforce, HubSpot, Pipedrive
- Outreach: Instantly, Apollo, Outreach, Salesloft
- REST API for custom automation

**Known gaps**
- Steep learning curve for non-technical users; full power requires understanding of waterfall logic and API concepts
- Expensive at scale ($800/mo for Scale tier; enterprise custom above)
- Dependent on underlying provider quality; orchestrates third-party data but does not own any proprietary database

**Licence / IP notes**
- Fully proprietary SaaS; Clay itself holds no proprietary data; underlying provider licences apply to the data returned

---

### People Data Labs (PDL)

**Core features**
- API-first B2B data infrastructure: 3B+ person and company records available via programmatic API
- Person enrichment: email, phone, LinkedIn URL, job history, education, skills, social profiles
- Company enrichment: employee count, funding history, technology stack, location, industry
- Bulk enrichment: CSV processing and batch API for large-scale enrichment jobs
- Used by Clay, FullEnrich, and other aggregators as an underlying data source

**Differentiating features**
- Broadest global coverage in the category: 3B+ records with strong non-North-American coverage
- Most developer-friendly API in the category; extensive documentation and SDKs (Python, Node, Go)
- Free tier (100 lookups/month) and transparent pay-as-you-go pricing ($0.004/record at bulk)
- Infrastructure-level product: not designed as an end-user tool but as an enrichment backbone for other applications

**UX patterns**
- API-only; no end-user UI; developer console for key management and usage monitoring
- Well-documented examples in Python, JavaScript, and curl for common enrichment patterns

**Integration points**
- Any application via REST API
- Commonly used as source within Clay, FullEnrich, and custom enrichment pipelines

**Known gaps**
- Raw data quality variable; cleaning and validation required for production use
- Not an end-user product; requires developer integration for all use cases
- No intent data, technographic signals, or phone verification overlay

**Licence / IP notes**
- Fully proprietary data; API usage terms restrict resale of raw data; GDPR legitimate interest claimed; California CCPA compliance documented

---

### Crunchbase

**Core features**
- Company-level enrichment: funding rounds (amount, date, investors), founding year, employee count range, description, social profiles, key executives
- Technology company intelligence: acquisition history, IPO data, subsidiary relationships
- Investor and fund database for BD and partnership use cases

**Differentiating features**
- Best funding and investment data in the category; no other enrichment provider matches Crunchbase for funding round intelligence
- Executive contact data for key leadership roles at funded companies (combined with funding signals)

**UX patterns**
- Web prospecting UI with company and investor search
- CSV export for list building
- Salesforce and HubSpot native integrations

**Integration points**
- Salesforce, HubSpot (native)
- REST API
- CSV export

**Known gaps**
- Does not provide person-level email or phone data; company-level only
- Funded startups and venture-backed companies are well covered; bootstrapped companies and SMBs are underrepresented
- Pricing ($29–$49/user/mo) reasonable for targeted use but not a general-purpose enrichment replacement

**Licence / IP notes**
- Fully proprietary; data sourced from public filings, news, and user submissions

---

## Cross-Cutting Feature Themes

### Table-Stakes Features
- Contact-level enrichment: verified email address, direct dial / mobile number, job title, seniority, department
- Company-level enrichment: employee count, revenue range, industry, location, technology stack
- Email verification: syntax, MX record, and deliverability validation before data is returned
- CRM integration: Salesforce and HubSpot field population and scheduled re-enrichment
- Bulk enrichment: CSV upload and batch API for list processing
- Match rate reporting: transparency about what percentage of input records were enriched
- Basic GDPR/CCPA compliance documentation and opt-out handling

### Differentiating Features
- Waterfall enrichment: multi-source sequencing to maximise fill rates when primary sources lack coverage (Clay model)
- Phone-verified mobile numbers (Cognism Diamond Data model); not just scraped numbers
- Intent data overlay: which companies are actively researching relevant topics (ZoomInfo/Bombora model)
- Job-change detection and automated re-enrichment triggers (Apollo model)
- Technology install / technographic signals: which software stack is in use at target companies
- AI-powered unstructured research: natural-language prompts returning insights from web search, LinkedIn, and news
- Real-time form shortening: auto-fill known fields for website visitors (Clearbit/Breeze model)
- GDPR-auditable data provenance: documented lawful basis and data source for each enriched field

### Underserved Areas / Opportunities
- **No production-ready OSS enrichment engine**: Despite high developer demand, no maintained open-source enrichment platform exists; tools on GitHub are largely ad-hoc scrapers with legal and maintenance risk
- **GDPR-compliant OSS enrichment**: An engine built exclusively on public-domain or clearly permissioned sources (government business registries, public LinkedIn, company websites, SEC filings) with auditable data lineage would serve European GTM teams currently excluded from ZoomInfo's pricing tier and compliance posture
- **Event-driven re-enrichment**: No tool automatically re-enriches contacts when a LinkedIn job change, company funding event, or executive departure is detected in real time; Apollo's job-change alerts are closest but still batch-oriented
- **Enrichment accuracy transparency**: No tool provides field-level confidence scores or data source provenance to the end user; all return enriched data without indicating which source provided each field or how recently it was validated

### AI-Augmentation Candidates
- Source selection in waterfall: manually configured provider sequences → AI selecting and ordering sources dynamically based on contact geography, industry, and historical fill rate feedback
- Data freshness management: scheduled re-crawl → event-driven re-enrichment triggered by public signals (job changes, funding rounds, company restructures detected via LLM-parsed news feeds)
- Unstructured enrichment: structured firmographic data only → LLM research generating insights from recent news, job postings, and conference appearances
- GDPR compliance classification: manual legal basis documentation → AI classifying each data source against applicable legal basis and jurisdiction automatically
- Data quality scoring: binary matched / not matched → field-level confidence score and source attribution per enriched record

## Legal & IP Summary

The contact enrichment space carries significant IP and regulatory risk. LinkedIn's Terms of Service prohibit automated data scraping; the ongoing hiQ v. LinkedIn litigation has not definitively settled the legality of scraping publicly available LinkedIn profiles, creating persistent legal exposure for enrichment tools that source LinkedIn data. GDPR Article 6 (lawful basis) and Article 14 (transparency with data subjects) apply to any enrichment of EU-based personal data; the "legitimate interest" basis claimed by ZoomInfo, Apollo, and most US-based providers is increasingly challenged by EU Data Protection Authorities. Cognism's phone-verified Diamond Data approach offers greater defensibility but at higher operational cost. An open-source enrichment engine should be built exclusively on data sources with a clear legal basis: government business registries (Companies House UK, SEC EDGAR, EIN/DUNS records), company websites, public job postings, and academic/conference speaker databases. No patent concerns specific to enrichment algorithms were identified.

## Recommended Feature Scope

**Must-have (MVP)**:
- Contact enrichment from public/permissioned sources: verified email, job title, seniority, department, LinkedIn URL
- Company enrichment: employee count, industry, location, technology stack (from public job postings and website)
- Email verification: syntax, MX record, SMTP-level deliverability validation
- Waterfall logic across at least 3 data sources with transparent fill rate reporting per source
- CRM sync: Salesforce and HubSpot field population via API
- GDPR-auditable data provenance: each returned field tagged with source and collection date

**Should-have (v1.1)**:
- Bulk enrichment API: batch processing of CSV lists with per-record match rate and confidence scores
- Job-change detection: re-enrich contacts and notify CRM when a role change is detected via public signals
- Intent signal overlay: identify companies showing buying intent via public content consumption signals
- Technographic data: technology stack detection from job postings, public GitHub, and company websites

**Nice-to-have (backlog)**:
- AI-powered unstructured research: natural-language prompt returns insights from news, job postings, and LinkedIn (within ToS-compliant access paths)
- Event-driven enrichment triggers: automatic re-enrichment on company funding event, executive hire/departure, or product launch detected via LLM news parsing
- Real-time form shortening: auto-fill known fields for website visitors using enrichment cache
- Confidence scoring: field-level probability score indicating enrichment accuracy with source attribution visible to end users
