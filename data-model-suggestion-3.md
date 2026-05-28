# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Contact Enrichment Engine · Created: 2026-05-12

## Philosophy

The Hybrid Relational + JSONB model uses traditional relational columns for core identity fields that are queried, filtered, and joined frequently, while storing variable, source-specific, and jurisdiction-dependent enrichment data in JSONB columns. This approach recognises that contact enrichment data has a stable core (name, email, company, title) surrounded by a highly variable periphery (technographic signals, social profiles, source-specific metadata, jurisdiction-specific compliance fields) that changes with every new data source integration.

This pattern is used successfully by modern SaaS platforms that need to ship fast and evolve their schema without downtime migrations. HubSpot, for example, stores custom properties as key-value pairs alongside typed core fields. People Data Labs returns enrichment data as nested JSON with variable field availability depending on the data source and region. By embedding this variability in JSONB columns rather than separate tables, the model reduces join complexity while retaining PostgreSQL's full JSONB indexing capabilities (GIN indexes, containment queries, jsonpath).

The hybrid approach is ideal for an MVP/v1 enrichment engine where the enrichment schema is still evolving rapidly. New data sources can be integrated by adding fields to JSONB payloads rather than running DDL migrations. Once the schema stabilises, high-value JSONB fields can be promoted to dedicated columns.

**Best for:** Teams building an MVP or rapidly iterating enrichment engine where schema flexibility, fast development velocity, and easy integration of new data sources outweigh the rigour of full normalisation.

**Trade-offs:**
- (+) Fastest development velocity: new enrichment fields require no DDL migration
- (+) New data source integration is a config change, not a schema change
- (+) Fewer tables and joins: ~18 tables vs. ~25 for normalised
- (+) JSONB GIN indexes enable fast containment queries on variable fields
- (+) Multi-jurisdiction compliance fields stored naturally (EU fields differ from US fields)
- (-) JSONB fields lack column-level type constraints; validation must happen in application layer
- (-) Complex JSONB queries are harder to write and optimise than relational joins
- (-) Reporting and analytics queries on JSONB fields are slower without careful indexing
- (-) Risk of "schema-in-the-app" -- field definitions live in code rather than DDL
- (-) JSONB columns can accumulate stale or inconsistent fields over time

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| RFC 5322 (Email Format) | `emails` array entries validated against RFC 5322 before storage |
| RFC 5321 (SMTP) | Verification results in `emails[].verification` JSONB sub-object |
| RFC 6350 (vCard 4.0) | Core person fields map to vCard; `social_profiles` JSONB maps to vCard URL type |
| Schema.org Person/Organization | Relational field names align with Schema.org vocabulary |
| ISO 3166-1/2 (Country/Region) | `location.country_code` in JSONB and `hq_country_code` column use ISO 3166 |
| GDPR Articles 6, 14, 21 | `provenance` JSONB on each entity stores per-field source and lawful basis |
| CCPA/CPRA | Privacy flags as relational columns; jurisdiction-specific fields in `compliance` JSONB |
| JSON Schema Draft 2020-12 | JSONB column schemas defined as JSON Schema for application-layer validation |
| OpenAPI 3.1 | API request/response schemas mirror the JSONB structures directly |
| ICANN RDAP | Company domain RDAP data stored in `domains[].rdap` JSONB sub-object |

---

## Core Tables

```sql
-- ============================================================
-- TENANT & ACCESS CONTROL
-- ============================================================

CREATE TABLE tenants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    plan            TEXT NOT NULL DEFAULT 'free',
    settings        JSONB NOT NULL DEFAULT '{}',
    -- Example settings:
    -- {
    --   "default_waterfall": "us_standard",
    --   "auto_verify_emails": true,
    --   "enrichment_budget_monthly_usd": 500.00,
    --   "gdpr_mode": true,
    --   "crm_sync_enabled": true
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE api_keys (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    key_hash        TEXT NOT NULL UNIQUE,
    name            TEXT NOT NULL,
    scopes          TEXT[] NOT NULL DEFAULT '{}',
    rate_limit_rpm  INT NOT NULL DEFAULT 60,
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- Example metadata:
    -- {
    --   "created_by": "user_abc",
    --   "environment": "production",
    --   "ip_allowlist": ["10.0.0.0/8"]
    -- }
    expires_at      TIMESTAMPTZ,
    revoked_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_api_keys_tenant ON api_keys(tenant_id);
CREATE INDEX idx_api_keys_hash ON api_keys(key_hash);
```

## Person Table (Hybrid)

```sql
-- ============================================================
-- PERSONS: relational core + JSONB extensions
-- ============================================================

CREATE TABLE persons (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenants(id),

    -- =====================================================
    -- RELATIONAL CORE: indexed, typed, constrained
    -- These fields are queried/filtered most often
    -- =====================================================
    full_name           TEXT,
    first_name          TEXT,
    last_name           TEXT,
    job_title           TEXT,
    seniority           TEXT CHECK (seniority IN (
        'entry', 'individual_contributor', 'manager', 'director',
        'vp', 'c_level', 'partner', 'owner', 'intern'
    )),
    department          TEXT,
    current_company_id  UUID REFERENCES companies(id),
    primary_email       TEXT,               -- denormalized for fast lookup
    linkedin_url        TEXT,
    location_country    TEXT,               -- ISO 3166-1 alpha-2

    -- =====================================================
    -- JSONB EXTENSIONS: flexible, schema-on-read
    -- =====================================================

    -- All email addresses with verification status
    emails              JSONB NOT NULL DEFAULT '[]',
    -- Example:
    -- [
    --   {
    --     "address": "sean@example.com",
    --     "type": "work",
    --     "is_primary": true,
    --     "verification": {
    --       "status": "valid",
    --       "verdict": "valid",
    --       "mx_hostname": "alt1.aspmx.l.google.com",
    --       "smtp_code": 250,
    --       "is_catch_all": false,
    --       "verified_at": "2026-05-12T10:00:00Z"
    --     },
    --     "source": "pdl",
    --     "confidence": 0.92,
    --     "collected_at": "2026-05-01T08:00:00Z"
    --   },
    --   {
    --     "address": "sean.thorne@gmail.com",
    --     "type": "personal",
    --     "is_primary": false,
    --     "verification": { "status": "valid", "verified_at": "2026-05-02" },
    --     "source": "hunter",
    --     "confidence": 0.78
    --   }
    -- ]

    -- All phone numbers
    phones              JSONB NOT NULL DEFAULT '[]',
    -- Example:
    -- [
    --   {
    --     "number": "+14155551234",
    --     "type": "mobile",
    --     "is_verified": true,
    --     "source": "cognism",
    --     "confidence": 0.95
    --   }
    -- ]

    -- Social profiles beyond LinkedIn
    social_profiles     JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "github": "https://github.com/seanthorne",
    --   "twitter": "https://twitter.com/seanthorne",
    --   "personal_website": "https://seanthorne.dev",
    --   "stackoverflow": "https://stackoverflow.com/users/12345"
    -- }

    -- Location details (variable by region)
    location            JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "city": "San Francisco",
    --   "region": "California",
    --   "country_code": "US",
    --   "postal_code": "94105",
    --   "timezone": "America/Los_Angeles",
    --   "geo": { "type": "Point", "coordinates": [-122.3965, 37.7922] }
    -- }

    -- Employment history
    experience          JSONB NOT NULL DEFAULT '[]',
    -- Example:
    -- [
    --   {
    --     "company_name": "Acme Corp",
    --     "company_id": "uuid-here",
    --     "title": "VP of Engineering",
    --     "seniority": "vp",
    --     "department": "Engineering",
    --     "start_date": "2024-03",
    --     "end_date": null,
    --     "is_current": true,
    --     "source": "pdl"
    --   }
    -- ]

    -- Education history
    education           JSONB NOT NULL DEFAULT '[]',
    -- Example:
    -- [
    --   {
    --     "school": "Stanford University",
    --     "degree": "MS",
    --     "field": "Computer Science",
    --     "start_year": 2012,
    --     "end_year": 2014,
    --     "source": "pdl"
    --   }
    -- ]

    -- Skills and certifications
    skills              JSONB NOT NULL DEFAULT '[]',
    -- Example: ["Python", "PostgreSQL", "Machine Learning", "AWS"]

    -- Per-field data provenance (which source provided which field)
    provenance          JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "job_title": {
    --     "source": "apollo",
    --     "confidence": 0.85,
    --     "collected_at": "2026-05-01",
    --     "lawful_basis": "legitimate_interest"
    --   },
    --   "primary_email": {
    --     "source": "pdl",
    --     "confidence": 0.92,
    --     "collected_at": "2026-04-28",
    --     "lawful_basis": "legitimate_interest"
    --   }
    -- }

    -- Privacy and compliance
    privacy             JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "do_not_contact": false,
    --   "gdpr_opt_out": false,
    --   "ccpa_do_not_sell": false,
    --   "consent_records": [
    --     {
    --       "type": "enrichment",
    --       "status": "granted",
    --       "lawful_basis": "legitimate_interest",
    --       "granted_at": "2026-01-15"
    --     }
    --   ],
    --   "data_subject_requests": []
    -- }

    -- Raw enrichment responses (for debugging and re-processing)
    raw_enrichments     JSONB NOT NULL DEFAULT '[]',
    -- Example:
    -- [
    --   {
    --     "source": "pdl",
    --     "queried_at": "2026-05-01T08:00:00Z",
    --     "request_id": "req_abc123",
    --     "response_fields": ["full_name", "job_title", "emails", "phones"],
    --     "confidence": 0.88
    --   }
    -- ]

    -- =====================================================
    -- METADATA
    -- =====================================================
    enrichment_score    NUMERIC(3,2),
    last_enriched_at    TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Relational indexes for core query patterns
CREATE INDEX idx_persons_tenant ON persons(tenant_id);
CREATE INDEX idx_persons_name ON persons(tenant_id, last_name, first_name);
CREATE INDEX idx_persons_email ON persons(primary_email);
CREATE INDEX idx_persons_company ON persons(current_company_id);
CREATE INDEX idx_persons_linkedin ON persons(linkedin_url) WHERE linkedin_url IS NOT NULL;
CREATE INDEX idx_persons_country ON persons(tenant_id, location_country);

-- GIN indexes for JSONB queries
CREATE INDEX idx_persons_emails_gin ON persons USING GIN (emails jsonb_path_ops);
CREATE INDEX idx_persons_phones_gin ON persons USING GIN (phones jsonb_path_ops);
CREATE INDEX idx_persons_skills_gin ON persons USING GIN (skills jsonb_path_ops);
CREATE INDEX idx_persons_provenance_gin ON persons USING GIN (provenance jsonb_path_ops);
```

## Company Table (Hybrid)

```sql
-- ============================================================
-- COMPANIES: relational core + JSONB extensions
-- ============================================================

CREATE TABLE companies (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenants(id),

    -- =====================================================
    -- RELATIONAL CORE
    -- =====================================================
    name                TEXT NOT NULL,
    domain              TEXT,
    industry            TEXT,
    employee_count      INT,
    employee_range      TEXT,
    hq_country_code     TEXT,               -- ISO 3166-1 alpha-2
    company_type        TEXT CHECK (company_type IN (
        'public', 'private', 'nonprofit', 'government', 'education', 'self_employed'
    )),

    -- =====================================================
    -- JSONB EXTENSIONS
    -- =====================================================

    -- Additional domains
    domains             JSONB NOT NULL DEFAULT '[]',
    -- Example:
    -- [
    --   {
    --     "domain": "acme.com",
    --     "is_primary": true,
    --     "mx_provider": "Google Workspace",
    --     "rdap": {
    --       "registrant_org": "Acme Corporation",
    --       "registration_date": "2005-03-15",
    --       "queried_at": "2026-05-01"
    --     }
    --   },
    --   { "domain": "acme.io", "is_primary": false }
    -- ]

    -- Firmographic details
    firmographics       JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "legal_name": "Acme Corporation Inc.",
    --   "revenue_range": "$10M-$50M",
    --   "founded_year": 2005,
    --   "sic_code": "7372",
    --   "naics_code": "511210",
    --   "description": "Enterprise software company...",
    --   "ticker_symbol": null,
    --   "sec_cik": null,
    --   "lei": "5493001KJTIIGC8Y1R12",
    --   "duns_number": "123456789"
    -- }

    -- Location details
    locations           JSONB NOT NULL DEFAULT '[]',
    -- Example:
    -- [
    --   {
    --     "type": "headquarters",
    --     "city": "San Francisco",
    --     "region": "California",
    --     "country_code": "US",
    --     "postal_code": "94105",
    --     "street": "123 Market St"
    --   }
    -- ]

    -- Technology stack
    technologies        JSONB NOT NULL DEFAULT '[]',
    -- Example:
    -- [
    --   {
    --     "name": "Salesforce",
    --     "category": "CRM",
    --     "first_seen": "2024-06",
    --     "last_seen": "2026-05",
    --     "source": "job_postings"
    --   }
    -- ]

    -- Funding history
    funding             JSONB NOT NULL DEFAULT '[]',
    -- Example:
    -- [
    --   {
    --     "round": "series_b",
    --     "amount_usd": 25000000,
    --     "date": "2025-09-15",
    --     "investors": ["Sequoia Capital", "a16z"],
    --     "source": "crunchbase"
    --   }
    -- ]

    -- Social profiles
    social_profiles     JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "linkedin": "https://linkedin.com/company/acme",
    --   "twitter": "https://twitter.com/acme",
    --   "crunchbase": "https://crunchbase.com/organization/acme"
    -- }

    -- Per-field provenance
    provenance          JSONB NOT NULL DEFAULT '{}',

    -- =====================================================
    -- METADATA
    -- =====================================================
    enrichment_score    NUMERIC(3,2),
    last_enriched_at    TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_companies_tenant ON companies(tenant_id);
CREATE INDEX idx_companies_domain ON companies(tenant_id, domain);
CREATE INDEX idx_companies_name ON companies(tenant_id, name);
CREATE INDEX idx_companies_country ON companies(tenant_id, hq_country_code);
CREATE INDEX idx_companies_domains_gin ON companies USING GIN (domains jsonb_path_ops);
CREATE INDEX idx_companies_tech_gin ON companies USING GIN (technologies jsonb_path_ops);
```

## Data Source & Waterfall Tables

```sql
-- ============================================================
-- DATA SOURCES
-- ============================================================

CREATE TABLE data_sources (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL UNIQUE,
    display_name    TEXT NOT NULL,
    source_type     TEXT NOT NULL,
    lawful_basis    TEXT NOT NULL,
    priority        INT NOT NULL DEFAULT 100,
    cost_per_lookup NUMERIC(10,6),
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    -- Source-specific configuration in JSONB
    config          JSONB NOT NULL DEFAULT '{}',
    -- Example config:
    -- {
    --   "api_base_url": "https://api.peopledatalabs.com/v5",
    --   "auth_type": "api_key",
    --   "rate_limit_rpm": 100,
    --   "supported_fields": ["full_name", "job_title", "emails", "phones"],
    --   "coverage_regions": ["US", "EU", "APAC"],
    --   "lawful_basis_doc": "https://docs.example.com/lia"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- WATERFALL CONFIGURATIONS
-- ============================================================

CREATE TABLE waterfall_configs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            TEXT NOT NULL,
    request_type    TEXT NOT NULL,
    is_default      BOOLEAN NOT NULL DEFAULT FALSE,
    -- Full waterfall definition in JSONB
    steps           JSONB NOT NULL,
    -- Example steps:
    -- [
    --   {
    --     "source": "pdl",
    --     "priority": 1,
    --     "fields_requested": ["emails", "phones", "job_title"],
    --     "conditions": { "country_in": ["US", "CA"] },
    --     "max_cost_usd": 0.10,
    --     "timeout_ms": 5000
    --   },
    --   {
    --     "source": "cognism",
    --     "priority": 2,
    --     "fields_requested": ["phones"],
    --     "conditions": { "country_in": ["GB", "DE", "FR", "NL"] },
    --     "max_cost_usd": 0.50
    --   },
    --   {
    --     "source": "hunter",
    --     "priority": 3,
    --     "fields_requested": ["emails"],
    --     "fallback_only": true
    --   }
    -- ]
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, name)
);

CREATE INDEX idx_waterfall_tenant ON waterfall_configs(tenant_id);
```

## Enrichment Pipeline Tables

```sql
-- ============================================================
-- ENRICHMENT REQUESTS
-- ============================================================

CREATE TABLE enrichment_requests (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    api_key_id      UUID REFERENCES api_keys(id),
    request_type    TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'pending',
    -- Input
    input_params    JSONB NOT NULL,
    -- Results
    matched_person_id UUID REFERENCES persons(id),
    matched_company_id UUID REFERENCES companies(id),
    match_confidence NUMERIC(3,2),
    -- Waterfall execution log (complete history in JSONB)
    waterfall_log   JSONB NOT NULL DEFAULT '[]',
    -- Example:
    -- [
    --   {
    --     "step": 1,
    --     "source": "pdl",
    --     "status": "partial",
    --     "fields_returned": ["full_name", "job_title", "emails"],
    --     "fields_missing": ["phones"],
    --     "confidence": 0.88,
    --     "duration_ms": 342,
    --     "cost_usd": 0.004
    --   },
    --   {
    --     "step": 2,
    --     "source": "cognism",
    --     "status": "success",
    --     "fields_returned": ["phones"],
    --     "confidence": 0.95,
    --     "duration_ms": 567,
    --     "cost_usd": 0.50
    --   }
    -- ]
    -- Performance
    total_cost_usd  NUMERIC(10,6),
    duration_ms     INT,
    credits_consumed INT NOT NULL DEFAULT 1,
    -- Timestamps
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ
);

CREATE INDEX idx_enrichment_req_tenant ON enrichment_requests(tenant_id, created_at);
CREATE INDEX idx_enrichment_req_status ON enrichment_requests(status);
```

## Job Change Detection

```sql
-- ============================================================
-- JOB CHANGE EVENTS
-- ============================================================

CREATE TABLE job_change_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    person_id       UUID NOT NULL REFERENCES persons(id),
    -- Change details in JSONB for flexibility
    change_data     JSONB NOT NULL,
    -- Example:
    -- {
    --   "previous": {
    --     "company_name": "Acme Corp",
    --     "company_id": "uuid-here",
    --     "title": "Senior Engineer",
    --     "seniority": "individual_contributor"
    --   },
    --   "current": {
    --     "company_name": "NewCo Inc",
    --     "company_id": "uuid-there",
    --     "title": "VP of Engineering",
    --     "seniority": "vp"
    --   },
    --   "detection_source": "enrichment_delta",
    --   "confidence": 0.91,
    --   "signals": ["linkedin_title_changed", "new_company_domain_in_email"]
    -- }
    notification_sent BOOLEAN NOT NULL DEFAULT FALSE,
    notification_sent_at TIMESTAMPTZ,
    detected_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_job_changes_tenant ON job_change_events(tenant_id, detected_at);
CREATE INDEX idx_job_changes_person ON job_change_events(person_id);
CREATE INDEX idx_job_changes_unsent ON job_change_events(notification_sent)
    WHERE notification_sent = FALSE;
```

## CRM Integration

```sql
-- ============================================================
-- CRM CONNECTIONS
-- ============================================================

CREATE TABLE crm_connections (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    crm_type        TEXT NOT NULL,
    -- Auth (encrypted)
    credentials_enc BYTEA NOT NULL,
    instance_url    TEXT,
    -- All configuration in JSONB
    config          JSONB NOT NULL DEFAULT '{}',
    -- Example config:
    -- {
    --   "sync_enabled": true,
    --   "sync_direction": "bidirectional",
    --   "sync_interval_minutes": 15,
    --   "field_mappings": {
    --     "person.job_title": { "object": "Contact", "field": "Title" },
    --     "person.primary_email": { "object": "Contact", "field": "Email" },
    --     "person.phones[0].number": { "object": "Contact", "field": "Phone" },
    --     "company.name": { "object": "Account", "field": "Name" },
    --     "company.employee_count": { "object": "Account", "field": "NumberOfEmployees" }
    --   },
    --   "overwrite_rules": {
    --     "only_if_empty": ["Email"],
    --     "always_overwrite": ["Title", "Phone"]
    --   }
    -- }
    last_synced_at  TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_crm_tenant ON crm_connections(tenant_id);

-- ============================================================
-- CRM SYNC LOG
-- ============================================================

CREATE TABLE crm_sync_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    crm_connection_id UUID NOT NULL REFERENCES crm_connections(id),
    entity_type     TEXT NOT NULL,
    entity_id       UUID NOT NULL,
    crm_record_id   TEXT NOT NULL,
    -- Sync details in JSONB
    sync_details    JSONB NOT NULL,
    -- Example:
    -- {
    --   "sync_type": "update",
    --   "fields_synced": ["Title", "Phone"],
    --   "fields_skipped": ["Email"],
    --   "skip_reason": { "Email": "overwrite_rules.only_if_empty" }
    -- }
    status          TEXT NOT NULL DEFAULT 'success',
    error_message   TEXT,
    synced_at       TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_crm_sync_conn ON crm_sync_log(crm_connection_id, synced_at);
```

## GDPR Compliance

```sql
-- ============================================================
-- DATA SUBJECT REQUESTS
-- ============================================================

CREATE TABLE data_subject_requests (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    person_id       UUID REFERENCES persons(id),
    request_type    TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'received',
    requester_email TEXT NOT NULL,
    -- Full request/response details in JSONB
    details         JSONB NOT NULL DEFAULT '{}',
    -- Example (erasure request):
    -- {
    --   "fields_to_erase": ["all"],
    --   "verification_method": "email_confirmation",
    --   "verified_at": "2026-05-12T10:00:00Z",
    --   "data_exported": true,
    --   "export_sent_at": "2026-05-12T10:15:00Z",
    --   "erasure_completed_at": "2026-05-12T10:30:00Z",
    --   "fields_erased": ["full_name", "emails", "phones", "experience", "education"],
    --   "retained_for_legal": ["enrichment_requests"]
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_dsr_tenant ON data_subject_requests(tenant_id, status);
CREATE INDEX idx_dsr_person ON data_subject_requests(person_id);
```

## Example JSONB Queries

```sql
-- ============================================================
-- Find persons with a specific email domain
-- ============================================================

SELECT id, full_name, primary_email
FROM persons
WHERE tenant_id = 'tenant-uuid-here'
  AND emails @> '[{"address": "sean@example.com"}]';

-- ============================================================
-- Find persons with verified work emails
-- ============================================================

SELECT id, full_name,
       jsonb_path_query(emails, '$[*] ? (@.type == "work" && @.verification.status == "valid")') AS verified_emails
FROM persons
WHERE tenant_id = 'tenant-uuid-here'
  AND emails @@ '$[*].verification.status == "valid"';

-- ============================================================
-- Find companies using Salesforce
-- ============================================================

SELECT id, name, domain
FROM companies
WHERE tenant_id = 'tenant-uuid-here'
  AND technologies @> '[{"name": "Salesforce"}]';

-- ============================================================
-- Get provenance for a specific field
-- ============================================================

SELECT id, full_name,
       provenance->'job_title'->>'source' AS title_source,
       provenance->'job_title'->>'confidence' AS title_confidence,
       provenance->'job_title'->>'collected_at' AS title_collected_at,
       provenance->'job_title'->>'lawful_basis' AS title_lawful_basis
FROM persons
WHERE id = 'person-uuid-here';

-- ============================================================
-- Find companies that raised Series B+ funding
-- ============================================================

SELECT id, name, domain,
       jsonb_path_query(funding, '$[*] ? (@.round like_regex "series_[b-z]")') AS recent_rounds
FROM companies
WHERE tenant_id = 'tenant-uuid-here'
  AND funding @@ '$[*].round like_regex "series_[b-z]"';

-- ============================================================
-- Aggregate enrichment cost by source from waterfall logs
-- ============================================================

SELECT step->>'source' AS data_source,
       COUNT(*) AS total_steps,
       SUM((step->>'cost_usd')::NUMERIC) AS total_cost,
       AVG((step->>'duration_ms')::NUMERIC) AS avg_duration_ms
FROM enrichment_requests,
     jsonb_array_elements(waterfall_log) AS step
WHERE tenant_id = 'tenant-uuid-here'
  AND created_at >= now() - INTERVAL '30 days'
GROUP BY step->>'source'
ORDER BY total_cost DESC;
```

## Row-Level Security

```sql
ALTER TABLE persons ENABLE ROW LEVEL SECURITY;
ALTER TABLE companies ENABLE ROW LEVEL SECURITY;
ALTER TABLE enrichment_requests ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation_persons ON persons
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);

CREATE POLICY tenant_isolation_companies ON companies
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);

CREATE POLICY tenant_isolation_enrichment ON enrichment_requests
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Tenant & Access Control | 2 | tenants, api_keys |
| Person Data | 1 | persons (with JSONB for emails, phones, experience, education, provenance) |
| Company Data | 1 | companies (with JSONB for domains, technologies, funding, locations) |
| Data Sources & Waterfall | 2 | data_sources, waterfall_configs |
| Enrichment Pipeline | 1 | enrichment_requests (waterfall log embedded as JSONB) |
| Job Change Detection | 1 | job_change_events |
| CRM Integration | 2 | crm_connections, crm_sync_log |
| GDPR Compliance | 1 | data_subject_requests |
| **Total** | **11** | Significantly fewer tables than normalised model |

---

## Key Design Decisions

1. **Emails and phones as JSONB arrays on the person record** rather than separate tables. This eliminates joins for the most common query pattern (enrich a person and return all their contact data) and matches how enrichment APIs (PDL, Apollo) return data. The trade-off is that cross-person email lookups require GIN index containment queries rather than simple B-tree lookups.

2. **Per-field provenance embedded in JSONB** (`provenance` column on persons and companies). Each field maps to its source, confidence, collection date, and lawful basis. This satisfies GDPR Article 14 requirements without a separate `field_provenance` table and keeps provenance co-located with the data it describes.

3. **Waterfall execution log as JSONB array** on `enrichment_requests` rather than a separate `waterfall_steps` table. Each enrichment request carries its complete execution history inline, making it trivial to return in the API response without a join. Analytical queries across waterfall steps use `jsonb_array_elements()`.

4. **Company firmographics, technologies, and funding in JSONB** because these fields vary widely by company type and data source. A public company has SEC CIK and ticker; a startup has Crunchbase data; a European company has LEI. JSONB accommodates all without NULL-heavy columns.

5. **GIN indexes on JSONB columns** enable efficient containment queries (`@>` operator) and jsonpath queries (`@@` operator). The `jsonb_path_ops` GIN operator class is used for smaller index size and faster containment lookups.

6. **Denormalized `primary_email` and `location_country` on persons** for the most common filter/sort patterns. These are also stored in the JSONB arrays but duplicated as relational columns for B-tree index performance.

7. **Waterfall configuration stored as JSONB steps** rather than a junction table. This makes it easy to version configurations (just store the full JSON), diff changes, and render in a UI. The JSONB structure mirrors the API payload a user would send to configure their waterfall.

8. **CRM field mappings embedded in connection config JSONB** rather than a separate mapping table. This keeps all CRM configuration in one place and makes it easy to import/export configurations.

9. **Data subject requests use JSONB for variable details** because erasure requests, access requests, and rectification requests have different required fields. The JSONB `details` column accommodates all request types without separate tables.

10. **11 tables total vs. 25 in the normalised model.** This reduction comes entirely from embedding variable data in JSONB. The core query patterns remain performant through GIN indexes, but complex analytical queries (e.g., "which data source provides the most accurate phone numbers across all persons") require JSONB path expressions rather than simple GROUP BY queries.
