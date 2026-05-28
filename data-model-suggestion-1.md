# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Contact Enrichment Engine · Created: 2026-05-12

## Philosophy

The Entity-Centric Normalized Relational model applies classical database normalization (3NF/BCNF) to decompose every concept in the contact enrichment domain into its own table. Each person, company, email address, phone number, job experience, enrichment request, and data source gets a dedicated table with strict foreign key relationships. This approach maximises data integrity through referential constraints and eliminates redundancy by ensuring each fact is stored exactly once.

This architecture mirrors how enterprise CRM systems (Salesforce, HubSpot) and mature B2B data providers (People Data Labs, ZoomInfo) structure their internal data. PDL's person schema, for example, separates person records from their email arrays, phone arrays, experience arrays, and education arrays -- each queryable independently. The normalized model makes this separation explicit at the database level rather than hiding it inside JSON arrays.

The normalized approach is best suited for teams that prioritise query flexibility, data integrity, and compliance auditability over rapid schema evolution. When every field has a defined column with a type constraint, bad data is rejected at insert time rather than discovered downstream. The trade-off is more tables, more joins, and more migration overhead when the schema needs to change.

**Best for:** Teams building a long-lived, compliance-heavy enrichment platform where data integrity, complex cross-entity queries, and regulatory auditability are paramount.

**Trade-offs:**
- (+) Maximum data integrity via foreign keys and CHECK constraints
- (+) Complex analytical queries (joins across persons, companies, enrichment sources) are natural
- (+) Each field has explicit type, nullability, and constraint documentation
- (+) GDPR audit: every enriched field traces to a source record with documented lawful basis
- (-) High table count (~30+ tables) increases migration complexity
- (-) Schema changes require DDL migrations for every new enrichment field
- (-) Many-to-many relationships (person-emails, person-phones) require junction tables or array-of-rows patterns
- (-) Read-heavy workloads require careful indexing or denormalized views

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| RFC 5322 (Email Format) | `emails.address` validated against RFC 5322 syntax before insert |
| RFC 5321 (SMTP) | `email_verifications.smtp_response_code` stores SMTP probe results per RFC 5321 |
| RFC 6350 (vCard 4.0) | Person fields map to vCard properties: `FN`, `N`, `ORG`, `EMAIL`, `TEL`, `TITLE` |
| Schema.org Person/Organization | Field names align: `full_name`, `job_title`, `works_for`, `same_as` (social URLs) |
| ISO 3166-1/2 (Country/Region) | `addresses.country_code` and `companies.hq_country_code` use ISO 3166 codes |
| ISO/IEC 27001:2022 | Audit trail tables support information security management evidence requirements |
| GDPR Articles 6, 14, 21 | `data_sources.lawful_basis`, `consent_records`, and `data_subject_requests` tables |
| CCPA/CPRA | `opt_out_requests` table with California-specific fields; `do_not_sell` flag on persons |
| OpenAPI 3.1 / JSON Schema | Table structures map directly to JSON Schema definitions for API request/response |
| ICANN RDAP | `company_domains` stores RDAP-sourced registration data with `rdap_queried_at` |

---

## Core Identity Tables

```sql
-- ============================================================
-- TENANT & ACCESS CONTROL
-- ============================================================

CREATE TABLE tenants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    plan            TEXT NOT NULL DEFAULT 'free' CHECK (plan IN ('free', 'starter', 'pro', 'enterprise')),
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    email           TEXT NOT NULL,
    name            TEXT NOT NULL,
    role            TEXT NOT NULL DEFAULT 'member' CHECK (role IN ('owner', 'admin', 'member', 'readonly')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);

CREATE TABLE api_keys (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    user_id         UUID REFERENCES users(id),
    key_hash        TEXT NOT NULL UNIQUE,  -- bcrypt hash of the API key
    name            TEXT NOT NULL,
    scopes          TEXT[] NOT NULL DEFAULT '{}',  -- e.g., {'person:enrich', 'company:enrich'}
    rate_limit_rpm  INT NOT NULL DEFAULT 60,
    expires_at      TIMESTAMPTZ,
    revoked_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_api_keys_tenant ON api_keys(tenant_id);
CREATE INDEX idx_api_keys_hash ON api_keys(key_hash);
```

## Person & Company Tables

```sql
-- ============================================================
-- PERSONS
-- ============================================================

CREATE TABLE persons (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenants(id),
    -- Core identity (aligns with vCard N / Schema.org Person)
    full_name           TEXT,               -- vCard FN / Schema.org name
    first_name          TEXT,               -- vCard N given-name
    last_name           TEXT,               -- vCard N family-name
    middle_name         TEXT,               -- vCard N additional-name
    name_prefix         TEXT,               -- vCard N honorific-prefix
    name_suffix         TEXT,               -- vCard N honorific-suffix
    -- Current employment
    job_title           TEXT,               -- vCard TITLE / Schema.org jobTitle
    seniority           TEXT CHECK (seniority IN (
        'entry', 'individual_contributor', 'manager', 'director',
        'vp', 'c_level', 'partner', 'owner', 'intern'
    )),
    department          TEXT,
    current_company_id  UUID REFERENCES companies(id),
    -- Social profiles (Schema.org sameAs)
    linkedin_url        TEXT,
    github_url          TEXT,
    twitter_url         TEXT,
    personal_website    TEXT,
    -- Location
    location_city       TEXT,
    location_region     TEXT,               -- state/province
    location_country    TEXT,               -- ISO 3166-1 alpha-2
    -- Metadata
    gender              TEXT CHECK (gender IN ('male', 'female', 'non_binary', 'unknown')),
    birth_year          INT,
    -- Data quality
    enrichment_score    NUMERIC(3,2),       -- 0.00 to 1.00 overall confidence
    last_enriched_at    TIMESTAMPTZ,
    -- Privacy
    do_not_contact      BOOLEAN NOT NULL DEFAULT FALSE,
    gdpr_opt_out        BOOLEAN NOT NULL DEFAULT FALSE,
    ccpa_do_not_sell    BOOLEAN NOT NULL DEFAULT FALSE,
    -- Timestamps
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_persons_tenant ON persons(tenant_id);
CREATE INDEX idx_persons_name ON persons(tenant_id, last_name, first_name);
CREATE INDEX idx_persons_company ON persons(current_company_id);
CREATE INDEX idx_persons_linkedin ON persons(linkedin_url) WHERE linkedin_url IS NOT NULL;
CREATE INDEX idx_persons_enrichment ON persons(tenant_id, last_enriched_at);

-- ============================================================
-- COMPANIES
-- ============================================================

CREATE TABLE companies (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenants(id),
    -- Core identity (Schema.org Organization)
    name                TEXT NOT NULL,
    legal_name          TEXT,
    domain              TEXT,               -- primary website domain
    -- Firmographics
    industry            TEXT,
    sic_code            TEXT,               -- SEC SIC code
    naics_code          TEXT,               -- NAICS classification
    employee_count      INT,
    employee_range      TEXT CHECK (employee_range IN (
        '1-10', '11-50', '51-200', '201-500', '501-1000',
        '1001-5000', '5001-10000', '10001+'
    )),
    revenue_range       TEXT,
    founded_year        INT,
    company_type        TEXT CHECK (company_type IN (
        'public', 'private', 'nonprofit', 'government', 'education', 'self_employed'
    )),
    -- Location (headquarters)
    hq_street           TEXT,
    hq_city             TEXT,
    hq_region           TEXT,
    hq_country_code     TEXT,               -- ISO 3166-1 alpha-2
    hq_postal_code      TEXT,
    -- Identifiers
    ticker_symbol       TEXT,               -- for public companies
    sec_cik             TEXT,               -- SEC Central Index Key
    lei                 TEXT,               -- Legal Entity Identifier (ISO 17442)
    duns_number         TEXT,               -- D-U-N-S Number
    -- Social
    linkedin_url        TEXT,
    twitter_url         TEXT,
    crunchbase_url      TEXT,
    -- Metadata
    enrichment_score    NUMERIC(3,2),
    last_enriched_at    TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_companies_tenant ON companies(tenant_id);
CREATE INDEX idx_companies_domain ON companies(tenant_id, domain);
CREATE INDEX idx_companies_name ON companies(tenant_id, name);
CREATE INDEX idx_companies_sec ON companies(sec_cik) WHERE sec_cik IS NOT NULL;
```

## Contact Data Tables

```sql
-- ============================================================
-- EMAILS (one person can have multiple emails)
-- ============================================================

CREATE TABLE emails (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    person_id       UUID NOT NULL REFERENCES persons(id) ON DELETE CASCADE,
    address         TEXT NOT NULL,           -- RFC 5322 validated
    email_type      TEXT NOT NULL DEFAULT 'work' CHECK (email_type IN ('work', 'personal', 'other')),
    is_primary      BOOLEAN NOT NULL DEFAULT FALSE,
    -- Verification status
    verification_status TEXT NOT NULL DEFAULT 'unverified' CHECK (verification_status IN (
        'valid', 'invalid', 'risky', 'catch_all', 'disposable', 'role_based', 'unverified'
    )),
    verified_at     TIMESTAMPTZ,
    -- Data provenance
    source_id       UUID REFERENCES data_sources(id),
    confidence      NUMERIC(3,2),           -- 0.00 to 1.00
    collected_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE UNIQUE INDEX idx_emails_person_address ON emails(person_id, address);
CREATE INDEX idx_emails_address ON emails(address);
CREATE INDEX idx_emails_verification ON emails(verification_status);

-- ============================================================
-- PHONE NUMBERS
-- ============================================================

CREATE TABLE phones (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    person_id       UUID NOT NULL REFERENCES persons(id) ON DELETE CASCADE,
    number          TEXT NOT NULL,           -- E.164 format
    phone_type      TEXT NOT NULL DEFAULT 'work' CHECK (phone_type IN (
        'work_direct', 'work_main', 'mobile', 'personal', 'other'
    )),
    is_primary      BOOLEAN NOT NULL DEFAULT FALSE,
    is_verified     BOOLEAN NOT NULL DEFAULT FALSE,  -- phone-verified (Cognism Diamond-style)
    verified_at     TIMESTAMPTZ,
    source_id       UUID REFERENCES data_sources(id),
    confidence      NUMERIC(3,2),
    collected_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE UNIQUE INDEX idx_phones_person_number ON phones(person_id, number);
CREATE INDEX idx_phones_number ON phones(number);
```

## Email Verification Tables

```sql
-- ============================================================
-- EMAIL VERIFICATION RESULTS
-- ============================================================

CREATE TABLE email_verifications (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email_id            UUID NOT NULL REFERENCES emails(id) ON DELETE CASCADE,
    -- Verification pipeline stages
    syntax_valid        BOOLEAN NOT NULL,
    mx_found            BOOLEAN,
    mx_hostname         TEXT,                   -- MX record used for probe
    smtp_connectable    BOOLEAN,
    smtp_response_code  INT,                    -- e.g., 250, 550
    smtp_response_text  TEXT,
    mailbox_exists      BOOLEAN,
    is_catch_all        BOOLEAN,
    is_disposable       BOOLEAN,
    is_role_based       BOOLEAN,                -- e.g., info@, admin@
    -- Result
    verdict             TEXT NOT NULL CHECK (verdict IN (
        'valid', 'invalid', 'risky', 'catch_all', 'disposable', 'role_based', 'unknown'
    )),
    confidence          NUMERIC(3,2),
    -- Metadata
    verified_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
    verification_ms     INT,                    -- time taken in milliseconds
    provider            TEXT                    -- which verification service was used
);

CREATE INDEX idx_email_verifications_email ON email_verifications(email_id);
CREATE INDEX idx_email_verifications_verdict ON email_verifications(verdict);
```

## Employment & Experience Tables

```sql
-- ============================================================
-- JOB EXPERIENCES (employment history)
-- ============================================================

CREATE TABLE job_experiences (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    person_id       UUID NOT NULL REFERENCES persons(id) ON DELETE CASCADE,
    company_id      UUID REFERENCES companies(id),
    company_name    TEXT NOT NULL,           -- denormalized for cases where company not in DB
    title           TEXT,
    seniority       TEXT,
    department      TEXT,
    start_date      DATE,
    end_date        DATE,                   -- NULL = current position
    is_current      BOOLEAN NOT NULL DEFAULT FALSE,
    source_id       UUID REFERENCES data_sources(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_job_experiences_person ON job_experiences(person_id);
CREATE INDEX idx_job_experiences_company ON job_experiences(company_id);
CREATE INDEX idx_job_experiences_current ON job_experiences(person_id, is_current) WHERE is_current = TRUE;

-- ============================================================
-- EDUCATION
-- ============================================================

CREATE TABLE education (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    person_id       UUID NOT NULL REFERENCES persons(id) ON DELETE CASCADE,
    school_name     TEXT NOT NULL,
    degree          TEXT,
    field_of_study  TEXT,
    start_year      INT,
    end_year        INT,
    source_id       UUID REFERENCES data_sources(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_education_person ON education(person_id);
```

## Company Enrichment Tables

```sql
-- ============================================================
-- COMPANY DOMAINS
-- ============================================================

CREATE TABLE company_domains (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    company_id      UUID NOT NULL REFERENCES companies(id) ON DELETE CASCADE,
    domain          TEXT NOT NULL,
    is_primary      BOOLEAN NOT NULL DEFAULT FALSE,
    -- RDAP / WHOIS data
    registrant_org  TEXT,
    registration_date DATE,
    mx_provider     TEXT,                   -- detected email provider (Google, Microsoft, etc.)
    rdap_queried_at TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE UNIQUE INDEX idx_company_domains_unique ON company_domains(company_id, domain);
CREATE INDEX idx_company_domains_domain ON company_domains(domain);

-- ============================================================
-- COMPANY TECHNOLOGIES (technographic data)
-- ============================================================

CREATE TABLE company_technologies (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    company_id      UUID NOT NULL REFERENCES companies(id) ON DELETE CASCADE,
    technology_name TEXT NOT NULL,
    category        TEXT,                   -- e.g., 'CRM', 'Marketing Automation', 'Cloud Provider'
    first_seen_at   TIMESTAMPTZ,
    last_seen_at    TIMESTAMPTZ,
    source_id       UUID REFERENCES data_sources(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_company_tech_company ON company_technologies(company_id);
CREATE INDEX idx_company_tech_name ON company_technologies(technology_name);

-- ============================================================
-- COMPANY FUNDING ROUNDS (Crunchbase-style)
-- ============================================================

CREATE TABLE funding_rounds (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    company_id      UUID NOT NULL REFERENCES companies(id) ON DELETE CASCADE,
    round_type      TEXT,                   -- 'seed', 'series_a', 'series_b', etc.
    amount_usd      BIGINT,
    announced_date  DATE,
    investors       TEXT[],
    source_url      TEXT,
    source_id       UUID REFERENCES data_sources(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_funding_company ON funding_rounds(company_id);
```

## Data Source & Provenance Tables

```sql
-- ============================================================
-- DATA SOURCES (providers in the waterfall)
-- ============================================================

CREATE TABLE data_sources (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL UNIQUE,    -- e.g., 'pdl', 'apollo', 'hunter', 'sec_edgar'
    display_name    TEXT NOT NULL,
    source_type     TEXT NOT NULL CHECK (source_type IN (
        'api_provider', 'public_registry', 'web_scrape', 'user_contributed', 'manual'
    )),
    -- GDPR compliance
    lawful_basis    TEXT NOT NULL CHECK (lawful_basis IN (
        'legitimate_interest', 'consent', 'public_task', 'legal_obligation', 'contract'
    )),
    lawful_basis_doc TEXT,                  -- link to LIA or consent record
    data_region     TEXT,                   -- where the source stores data
    -- Configuration
    api_base_url    TEXT,
    priority        INT NOT NULL DEFAULT 100,  -- lower = higher priority in waterfall
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    -- Cost tracking
    cost_per_lookup NUMERIC(10,6),          -- USD per enrichment call
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- FIELD PROVENANCE (which source provided which field)
-- ============================================================

CREATE TABLE field_provenance (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    entity_type     TEXT NOT NULL CHECK (entity_type IN ('person', 'company')),
    entity_id       UUID NOT NULL,          -- person.id or company.id
    field_name      TEXT NOT NULL,           -- e.g., 'job_title', 'employee_count'
    field_value     TEXT,                    -- the value that was set
    source_id       UUID NOT NULL REFERENCES data_sources(id),
    confidence      NUMERIC(3,2),
    collected_at    TIMESTAMPTZ NOT NULL,
    superseded_at   TIMESTAMPTZ,            -- when a newer value replaced this one
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_provenance_entity ON field_provenance(entity_type, entity_id);
CREATE INDEX idx_provenance_source ON field_provenance(source_id);
CREATE INDEX idx_provenance_field ON field_provenance(entity_type, entity_id, field_name);
```

## Enrichment Pipeline Tables

```sql
-- ============================================================
-- ENRICHMENT REQUESTS (API calls from tenants)
-- ============================================================

CREATE TABLE enrichment_requests (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    user_id         UUID REFERENCES users(id),
    api_key_id      UUID REFERENCES api_keys(id),
    -- Request details
    request_type    TEXT NOT NULL CHECK (request_type IN ('person', 'company', 'bulk_person', 'bulk_company')),
    input_params    JSONB NOT NULL,         -- the lookup parameters provided
    -- Result
    status          TEXT NOT NULL DEFAULT 'pending' CHECK (status IN (
        'pending', 'processing', 'completed', 'partial', 'not_found', 'error'
    )),
    matched_person_id UUID REFERENCES persons(id),
    matched_company_id UUID REFERENCES companies(id),
    match_confidence NUMERIC(3,2),
    -- Waterfall tracking
    sources_queried TEXT[] NOT NULL DEFAULT '{}',
    sources_matched TEXT[] NOT NULL DEFAULT '{}',
    total_sources_tried INT NOT NULL DEFAULT 0,
    -- Performance
    duration_ms     INT,
    credits_consumed INT NOT NULL DEFAULT 1,
    -- Timestamps
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ
);

CREATE INDEX idx_enrichment_req_tenant ON enrichment_requests(tenant_id);
CREATE INDEX idx_enrichment_req_status ON enrichment_requests(status);
CREATE INDEX idx_enrichment_req_created ON enrichment_requests(tenant_id, created_at);

-- ============================================================
-- WATERFALL STEPS (per-source results within a request)
-- ============================================================

CREATE TABLE waterfall_steps (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    request_id      UUID NOT NULL REFERENCES enrichment_requests(id) ON DELETE CASCADE,
    source_id       UUID NOT NULL REFERENCES data_sources(id),
    step_order      INT NOT NULL,           -- 1, 2, 3... in waterfall sequence
    -- Result
    status          TEXT NOT NULL CHECK (status IN (
        'success', 'partial', 'not_found', 'error', 'skipped', 'rate_limited'
    )),
    fields_returned TEXT[],                 -- which fields this source provided
    confidence      NUMERIC(3,2),
    -- Performance & cost
    duration_ms     INT,
    cost_usd        NUMERIC(10,6),
    error_message   TEXT,
    -- Raw response (for debugging)
    raw_response    JSONB,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_waterfall_request ON waterfall_steps(request_id);
CREATE INDEX idx_waterfall_source ON waterfall_steps(source_id);

-- ============================================================
-- WATERFALL CONFIGURATIONS (per-tenant source ordering)
-- ============================================================

CREATE TABLE waterfall_configs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            TEXT NOT NULL,           -- e.g., 'default', 'europe_contacts', 'us_enterprise'
    request_type    TEXT NOT NULL CHECK (request_type IN ('person', 'company')),
    is_default      BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, name)
);

CREATE TABLE waterfall_config_sources (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    config_id       UUID NOT NULL REFERENCES waterfall_configs(id) ON DELETE CASCADE,
    source_id       UUID NOT NULL REFERENCES data_sources(id),
    priority        INT NOT NULL,           -- execution order (1 = first)
    -- Conditional execution
    condition_field TEXT,                   -- e.g., 'location_country'
    condition_value TEXT,                   -- e.g., 'US'
    max_cost_usd    NUMERIC(10,6),          -- skip if cost exceeds budget
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_wf_config_sources ON waterfall_config_sources(config_id, priority);
```

## GDPR & Compliance Tables

```sql
-- ============================================================
-- CONSENT RECORDS
-- ============================================================

CREATE TABLE consent_records (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    person_id       UUID NOT NULL REFERENCES persons(id) ON DELETE CASCADE,
    consent_type    TEXT NOT NULL CHECK (consent_type IN (
        'enrichment', 'marketing', 'data_sharing', 'profiling'
    )),
    status          TEXT NOT NULL CHECK (status IN ('granted', 'withdrawn', 'expired')),
    lawful_basis    TEXT NOT NULL,
    granted_at      TIMESTAMPTZ,
    withdrawn_at    TIMESTAMPTZ,
    expires_at      TIMESTAMPTZ,
    evidence_url    TEXT,                   -- link to consent form or record
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_consent_person ON consent_records(person_id);

-- ============================================================
-- DATA SUBJECT REQUESTS (GDPR Art. 15-22 / CCPA)
-- ============================================================

CREATE TABLE data_subject_requests (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    person_id       UUID REFERENCES persons(id),
    request_type    TEXT NOT NULL CHECK (request_type IN (
        'access', 'rectification', 'erasure', 'restriction',
        'portability', 'objection', 'ccpa_do_not_sell', 'ccpa_delete'
    )),
    status          TEXT NOT NULL DEFAULT 'received' CHECK (status IN (
        'received', 'verified', 'processing', 'completed', 'rejected'
    )),
    requester_email TEXT NOT NULL,
    request_details TEXT,
    completed_at    TIMESTAMPTZ,
    response_sent_at TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_dsr_person ON data_subject_requests(person_id);
CREATE INDEX idx_dsr_status ON data_subject_requests(status);
```

## Job Change Detection Tables

```sql
-- ============================================================
-- JOB CHANGE EVENTS
-- ============================================================

CREATE TABLE job_change_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    person_id       UUID NOT NULL REFERENCES persons(id),
    -- Previous position
    prev_company_id UUID REFERENCES companies(id),
    prev_company_name TEXT,
    prev_title      TEXT,
    -- New position
    new_company_id  UUID REFERENCES companies(id),
    new_company_name TEXT,
    new_title       TEXT,
    -- Detection metadata
    detected_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    detection_source TEXT NOT NULL,          -- e.g., 'linkedin_signal', 'enrichment_delta', 'manual'
    confidence      NUMERIC(3,2),
    -- Notification tracking
    notification_sent BOOLEAN NOT NULL DEFAULT FALSE,
    notification_sent_at TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_job_changes_person ON job_change_events(person_id);
CREATE INDEX idx_job_changes_detected ON job_change_events(detected_at);
CREATE INDEX idx_job_changes_unsent ON job_change_events(notification_sent) WHERE notification_sent = FALSE;
```

## CRM Integration Tables

```sql
-- ============================================================
-- CRM CONNECTIONS
-- ============================================================

CREATE TABLE crm_connections (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    crm_type        TEXT NOT NULL CHECK (crm_type IN ('salesforce', 'hubspot', 'pipedrive', 'dynamics')),
    -- OAuth credentials (encrypted)
    access_token_enc BYTEA,
    refresh_token_enc BYTEA,
    token_expires_at TIMESTAMPTZ,
    instance_url    TEXT,                   -- e.g., https://myorg.my.salesforce.com
    -- Sync settings
    sync_enabled    BOOLEAN NOT NULL DEFAULT TRUE,
    sync_direction  TEXT NOT NULL DEFAULT 'push' CHECK (sync_direction IN ('push', 'pull', 'bidirectional')),
    last_synced_at  TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_crm_tenant ON crm_connections(tenant_id);

-- ============================================================
-- CRM FIELD MAPPINGS
-- ============================================================

CREATE TABLE crm_field_mappings (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    crm_connection_id   UUID NOT NULL REFERENCES crm_connections(id) ON DELETE CASCADE,
    enrichment_field    TEXT NOT NULL,       -- e.g., 'job_title', 'email.work'
    crm_object          TEXT NOT NULL,       -- e.g., 'Contact', 'Lead', 'Account'
    crm_field           TEXT NOT NULL,       -- e.g., 'Title', 'Email', 'hs_job_title'
    sync_direction      TEXT NOT NULL DEFAULT 'push' CHECK (sync_direction IN ('push', 'pull', 'bidirectional')),
    overwrite_existing  BOOLEAN NOT NULL DEFAULT FALSE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_crm_mappings_conn ON crm_field_mappings(crm_connection_id);

-- ============================================================
-- CRM SYNC LOG
-- ============================================================

CREATE TABLE crm_sync_log (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    crm_connection_id   UUID NOT NULL REFERENCES crm_connections(id),
    entity_type         TEXT NOT NULL,       -- 'person' or 'company'
    entity_id           UUID NOT NULL,
    crm_record_id       TEXT NOT NULL,       -- external CRM record ID
    sync_type           TEXT NOT NULL CHECK (sync_type IN ('create', 'update', 'skip', 'error')),
    fields_synced       TEXT[],
    error_message       TEXT,
    synced_at           TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_crm_sync_conn ON crm_sync_log(crm_connection_id);
CREATE INDEX idx_crm_sync_entity ON crm_sync_log(entity_type, entity_id);
```

## Row-Level Security

```sql
-- ============================================================
-- ROW-LEVEL SECURITY POLICIES
-- ============================================================

ALTER TABLE persons ENABLE ROW LEVEL SECURITY;
ALTER TABLE companies ENABLE ROW LEVEL SECURITY;
ALTER TABLE emails ENABLE ROW LEVEL SECURITY;
ALTER TABLE phones ENABLE ROW LEVEL SECURITY;
ALTER TABLE enrichment_requests ENABLE ROW LEVEL SECURITY;

-- Tenant isolation: users can only see their own tenant's data
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
| Tenant & Access Control | 3 | tenants, users, api_keys |
| Person Identity | 5 | persons, emails, phones, job_experiences, education |
| Company Identity | 4 | companies, company_domains, company_technologies, funding_rounds |
| Email Verification | 1 | email_verifications |
| Data Provenance | 2 | data_sources, field_provenance |
| Enrichment Pipeline | 4 | enrichment_requests, waterfall_steps, waterfall_configs, waterfall_config_sources |
| GDPR & Compliance | 2 | consent_records, data_subject_requests |
| Job Change Detection | 1 | job_change_events |
| CRM Integration | 3 | crm_connections, crm_field_mappings, crm_sync_log |
| **Total** | **25** | |

---

## Key Design Decisions

1. **Separate tables for emails and phones** rather than arrays or JSONB on the person record. This enables direct querying (find all persons at a domain), independent verification tracking per email, and per-field provenance without parsing nested structures.

2. **Field-level provenance table** tracks which data source provided which value and when, enabling GDPR Article 14 transparency (telling data subjects where their data came from) and supporting waterfall conflict resolution (when two sources disagree, the higher-confidence or more-recent value wins).

3. **Waterfall configuration is per-tenant and per-request-type**, allowing different customers to prioritise different sources (a European customer might prioritise Cognism for GDPR-compliant phone numbers; a US customer might prioritise Apollo for cost).

4. **Email verification is a separate table** with full SMTP probe results (MX hostname, response code, catch-all detection), not just a boolean flag. This supports re-verification scheduling and historical analysis of deliverability trends.

5. **PostgreSQL Row-Level Security** for multi-tenant isolation. The `tenant_id` on every data table combined with RLS policies ensures that a tenant's API key can never access another tenant's enriched contacts, even if the application layer has a bug.

6. **Explicit GDPR/CCPA tables** for consent records and data subject requests. When a deletion request arrives, the system can trace every field back to its source via `field_provenance`, delete all records for that person, and log the completed request -- all with referential integrity enforced by foreign keys.

7. **Job change events as a first-class entity** rather than derived from diffs in `job_experiences`. This makes it easy to query unsent notifications, track detection accuracy over time, and build re-engagement triggers.

8. **Company identifiers use international standards** -- SEC CIK, LEI (ISO 17442), D-U-N-S Number -- enabling cross-referencing with public registries and reducing deduplication ambiguity.

9. **CRM sync is tracked bidirectionally** with field-level mapping configuration. Each tenant can map enrichment fields to their specific CRM field names, and the sync log provides an audit trail of what was pushed/pulled and when.

10. **Cost tracking at the waterfall step level** enables per-request cost reporting and budget enforcement. Each step records its actual cost, and the waterfall config can enforce a max cost per source to prevent runaway spend.
