# Contact Enrichment Engine -- Development Plan

> Project: 044-contact-enrichment-engine
> Created: 2026-05-25
> Based on: research.md, features.md, standards.md, data-model-suggestion-1 through 4, README.md

---

## Technology Decisions

### Primary Stack

| Layer | Technology | Rationale |
|-------|-----------|-----------|
| Language | **TypeScript (Node.js 22 LTS)** | API-first product; strong ecosystem for HTTP servers, async I/O, and data transformation. PDL, Apollo, Hunter, Lusha, and HubSpot all provide JavaScript SDKs or well-documented REST APIs consumed via fetch/axios. TypeScript gives compile-time safety on enrichment response payloads. |
| API Framework | **Fastify** | Fastest Node.js HTTP framework; built-in JSON Schema validation maps directly to OpenAPI 3.1 spec requirement from standards.md. Plugin architecture supports modular data-source adapters. |
| Database | **PostgreSQL 16** | All four data model suggestions target PostgreSQL. Chosen for: JSONB with GIN indexes (hybrid model), Row-Level Security (multi-tenant isolation), ltree extension (industry taxonomy), and partitioning (enrichment request history). |
| Data Model | **Hybrid Relational + JSONB (Model 3)** with select elements from Model 1 and Model 2 | Model 3 offers the fastest development velocity with 11 tables, no-migration schema evolution for new enrichment fields, and direct API-response alignment. We adopt Model 1's separate `emails` and `phones` tables (not JSONB arrays) for cross-entity email lookups and independent verification tracking. We adopt Model 2's append-only `enrichment_events` audit log for GDPR provenance without full event-sourcing complexity. |
| Cache | **Redis 7** | Rate-limit counters (per API key), enrichment result caching (avoid redundant upstream API calls within TTL), and waterfall step deduplication. |
| Queue | **BullMQ (Redis-backed)** | Bulk enrichment jobs, CRM sync tasks, email verification pipeline, and job-change detection scheduling. BullMQ provides retries, rate limiting, and priority queues without introducing Kafka complexity at MVP scale. |
| Email Verification | **Custom SMTP prober + DNS resolver** | RFC 5321/5322 compliance requires direct MX lookup and SMTP handshake. No open-source library covers the full pipeline (syntax, MX, SMTP, catch-all, disposable, role-based). Build in-house to control quality. |
| Authentication | **API Key (primary) + OAuth 2.0 (CRM integrations)** | API key for server-to-server enrichment calls (matches PDL, Apollo, Hunter patterns). OAuth 2.0 authorization-code flow for Salesforce/HubSpot CRM connections. JWT (RFC 7519) for session tokens in the management dashboard. |
| API Spec | **OpenAPI 3.1** | Industry standard per standards.md; enables auto-generated client SDKs, Swagger UI, and contract testing. Published as part of the public API. |
| Testing | **Vitest** (unit/integration) + **Playwright** (E2E dashboard) + **Pactflow** (contract) | Vitest for speed; Playwright for CRM integration UI flows; Pact for contract testing between enrichment API and data-source adapters. |
| CI/CD | **GitHub Actions** | Standard for open-source projects; matrix builds for Node 22 + PostgreSQL 16. |
| Containerisation | **Docker + Docker Compose** (dev) / **Kubernetes** (production) | Self-hosted deployment option is a key differentiator from SaaS-only competitors. |
| Observability | **OpenTelemetry + Prometheus + Grafana** | Distributed tracing across waterfall steps; per-source latency and cost dashboards; alerting on fill-rate degradation. |

### Data Source Adapter Architecture

Each external enrichment provider (PDL, Apollo, Hunter, SEC EDGAR, Companies House, RDAP) is implemented as a **DataSourceAdapter** conforming to a common interface. This enables:
- Waterfall orchestration without provider-specific logic in the core engine
- Independent testing via adapter-specific mocks
- New provider integration without touching core enrichment logic

```typescript
interface DataSourceAdapter {
  readonly name: string;
  readonly supportedFields: EnrichmentField[];
  readonly costPerLookup: number;
  readonly lawfulBasis: LawfulBasis;

  enrichPerson(input: PersonLookupInput): Promise<EnrichmentResult>;
  enrichCompany(input: CompanyLookupInput): Promise<EnrichmentResult>;
  healthCheck(): Promise<HealthStatus>;
}
```

---

## Project Structure

```
contact-enrichment-engine/
  packages/
    core/                     # Shared types, schemas, utilities
      src/
        types/                # TypeScript types for Person, Company, Email, etc.
        schemas/              # JSON Schema definitions (OpenAPI 3.1 aligned)
        errors/               # Custom error classes
        constants/            # Seniority levels, departments, verification statuses
    api/                      # Fastify REST API server
      src/
        routes/               # /v1/person/enrich, /v1/company/enrich, /v1/bulk/*
        middleware/           # Auth, rate limiting, tenant isolation
        plugins/              # Fastify plugins (database, cache, queue)
    engine/                   # Enrichment orchestration logic
      src/
        waterfall/            # Waterfall executor, source selector, cost tracker
        enrichment/           # Person/company enrichment pipelines
        verification/         # Email verification (SMTP prober, DNS, disposable check)
        dedup/                # Identity resolution and deduplication
        job-change/           # Job change detection logic
    adapters/                 # Data source adapters
      src/
        pdl/                  # People Data Labs adapter
        apollo/               # Apollo.io adapter
        hunter/               # Hunter.io adapter (email finding/verification)
        sec-edgar/            # SEC EDGAR adapter (US public companies)
        companies-house/      # UK Companies House adapter
        rdap/                 # ICANN RDAP adapter (domain data)
        crunchbase/           # Crunchbase adapter (funding data)
    crm/                      # CRM integration module
      src/
        salesforce/           # Salesforce sync adapter
        hubspot/              # HubSpot sync adapter
        sync-engine/          # Bidirectional sync orchestration
    dashboard/                # Management UI (Next.js)
      src/
        app/                  # App router pages
        components/           # Shared UI components
    db/                       # Database migrations and seeds
      migrations/             # Numbered SQL migration files
      seeds/                  # Seed data (data sources, industry taxonomy)
  docker/                     # Dockerfiles and compose
  docs/                       # OpenAPI spec, architecture diagrams
  tests/
    integration/              # Cross-package integration tests
    e2e/                      # End-to-end API tests
    fixtures/                 # Test data and mock responses
```

---

## Phase Dependency Graph

```
Phase 1: Foundation
    |
    v
Phase 2: Core Enrichment Engine
    |
    +------------------+
    |                  |
    v                  v
Phase 3: Email       Phase 4: Waterfall
Verification         Orchestration
    |                  |
    +--------+---------+
             |
             v
Phase 5: Bulk Enrichment & API Polish
             |
    +--------+---------+
    |                  |
    v                  v
Phase 6: CRM         Phase 7: Job Change
Integration           Detection
    |                  |
    +--------+---------+
             |
             v
Phase 8: GDPR Compliance & Data Provenance
             |
             v
Phase 9: Dashboard & Self-Service
             |
             v
Phase 10: Advanced Features (Intent, Technographics)
             |
             v
Phase 11: Performance, Scale & Observability
             |
             v
Phase 12: MCP Server & AI Agent Integration
```

---

## Phase 1: Foundation

**Goal:** Establish project skeleton, database schema, authentication, and the data-source adapter interface. All subsequent phases build on this foundation.

**Duration:** 2 weeks

### Task 1.1: Project Scaffolding & Monorepo Setup

**What:** Initialize the monorepo with TypeScript, package structure, linting, CI, and Docker Compose for local development (PostgreSQL 16, Redis 7).

**Design:**
```typescript
// Root tsconfig.json with project references
// packages/core/tsconfig.json extends root
// Turborepo or npm workspaces for monorepo management

// docker/docker-compose.yml
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: enrichment
      POSTGRES_USER: enrichment
      POSTGRES_PASSWORD: dev_password
    ports: ["5432:5432"]
    volumes: ["pgdata:/var/lib/postgresql/data"]

  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]
```

**Testing:**
- `npm run build` compiles all packages without errors
- `docker compose up` starts PostgreSQL and Redis; both accept connections
- `npm run lint` passes on empty project skeleton
- CI pipeline (GitHub Actions) runs build + lint on push to main

---

### Task 1.2: Database Schema -- Core Tables

**What:** Create the initial database migration with tenant, person, company, email, phone, data_source, and enrichment_request tables. Adopts the Hybrid model (Model 3) with separate email/phone tables from Model 1.

**Design:**
```sql
-- Migration 001_core_schema.sql

CREATE EXTENSION IF NOT EXISTS "pgcrypto";  -- for gen_random_uuid()
CREATE EXTENSION IF NOT EXISTS "ltree";     -- for industry taxonomy (Phase 10)

CREATE TABLE tenants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    plan            TEXT NOT NULL DEFAULT 'free'
                    CHECK (plan IN ('free', 'starter', 'pro', 'enterprise')),
    settings        JSONB NOT NULL DEFAULT '{}',
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
    expires_at      TIMESTAMPTZ,
    revoked_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE data_sources (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL UNIQUE,
    display_name    TEXT NOT NULL,
    source_type     TEXT NOT NULL CHECK (source_type IN (
        'api_provider', 'public_registry', 'web_scrape', 'user_contributed', 'manual'
    )),
    lawful_basis    TEXT NOT NULL CHECK (lawful_basis IN (
        'legitimate_interest', 'consent', 'public_task', 'legal_obligation', 'contract'
    )),
    priority        INT NOT NULL DEFAULT 100,
    cost_per_lookup NUMERIC(10,6),
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    config          JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE companies (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenants(id),
    name                TEXT NOT NULL,
    domain              TEXT,
    industry            TEXT,
    employee_count      INT,
    employee_range      TEXT,
    hq_country_code     TEXT,
    company_type        TEXT,
    firmographics       JSONB NOT NULL DEFAULT '{}',
    locations           JSONB NOT NULL DEFAULT '[]',
    technologies        JSONB NOT NULL DEFAULT '[]',
    funding             JSONB NOT NULL DEFAULT '[]',
    social_profiles     JSONB NOT NULL DEFAULT '{}',
    provenance          JSONB NOT NULL DEFAULT '{}',
    enrichment_score    NUMERIC(3,2),
    last_enriched_at    TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE persons (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenants(id),
    full_name           TEXT,
    first_name          TEXT,
    last_name           TEXT,
    job_title           TEXT,
    seniority           TEXT,
    department          TEXT,
    current_company_id  UUID REFERENCES companies(id),
    primary_email       TEXT,
    linkedin_url        TEXT,
    location_country    TEXT,
    social_profiles     JSONB NOT NULL DEFAULT '{}',
    location            JSONB NOT NULL DEFAULT '{}',
    experience          JSONB NOT NULL DEFAULT '[]',
    education           JSONB NOT NULL DEFAULT '[]',
    skills              JSONB NOT NULL DEFAULT '[]',
    provenance          JSONB NOT NULL DEFAULT '{}',
    privacy             JSONB NOT NULL DEFAULT '{}',
    enrichment_score    NUMERIC(3,2),
    last_enriched_at    TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Separate email table for cross-person lookups and verification tracking
CREATE TABLE emails (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    person_id       UUID NOT NULL REFERENCES persons(id) ON DELETE CASCADE,
    address         TEXT NOT NULL,
    email_type      TEXT NOT NULL DEFAULT 'work',
    is_primary      BOOLEAN NOT NULL DEFAULT FALSE,
    verification_status TEXT NOT NULL DEFAULT 'unverified',
    verified_at     TIMESTAMPTZ,
    source_id       UUID REFERENCES data_sources(id),
    confidence      NUMERIC(3,2),
    collected_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (person_id, address)
);

-- Separate phone table for verification tracking
CREATE TABLE phones (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    person_id       UUID NOT NULL REFERENCES persons(id) ON DELETE CASCADE,
    number          TEXT NOT NULL,
    phone_type      TEXT NOT NULL DEFAULT 'work_direct',
    is_primary      BOOLEAN NOT NULL DEFAULT FALSE,
    is_verified     BOOLEAN NOT NULL DEFAULT FALSE,
    verified_at     TIMESTAMPTZ,
    source_id       UUID REFERENCES data_sources(id),
    confidence      NUMERIC(3,2),
    collected_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (person_id, number)
);

-- Enrichment audit log (from Model 2, simplified)
CREATE TABLE enrichment_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    entity_type     TEXT NOT NULL CHECK (entity_type IN ('person', 'company')),
    entity_id       UUID NOT NULL,
    event_type      TEXT NOT NULL,
    payload         JSONB NOT NULL,
    data_source_id  UUID REFERENCES data_sources(id),
    lawful_basis    TEXT,
    tenant_id       UUID NOT NULL,
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Indexes (see data-model-suggestion-1.md and 3.md for full set)
CREATE INDEX idx_persons_tenant ON persons(tenant_id);
CREATE INDEX idx_persons_name ON persons(tenant_id, last_name, first_name);
CREATE INDEX idx_persons_email ON persons(primary_email);
CREATE INDEX idx_persons_company ON persons(current_company_id);
CREATE INDEX idx_persons_linkedin ON persons(linkedin_url) WHERE linkedin_url IS NOT NULL;
CREATE INDEX idx_companies_tenant ON companies(tenant_id);
CREATE INDEX idx_companies_domain ON companies(tenant_id, domain);
CREATE INDEX idx_emails_address ON emails(address);
CREATE INDEX idx_emails_person ON emails(person_id);
CREATE INDEX idx_phones_person ON phones(person_id);
CREATE INDEX idx_enrichment_events_entity ON enrichment_events(entity_type, entity_id);
CREATE INDEX idx_enrichment_events_tenant ON enrichment_events(tenant_id, occurred_at);

-- Row-Level Security
ALTER TABLE persons ENABLE ROW LEVEL SECURITY;
ALTER TABLE companies ENABLE ROW LEVEL SECURITY;
ALTER TABLE emails ENABLE ROW LEVEL SECURITY;
ALTER TABLE phones ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation_persons ON persons
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);
CREATE POLICY tenant_isolation_companies ON companies
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);
```

**Testing:**
- Migration runs successfully against clean PostgreSQL 16 instance
- Migration is idempotent (running twice does not error)
- RLS policies prevent cross-tenant data access (write integration test inserting data for tenant A, querying as tenant B, asserting empty result)
- All indexes exist and are used by EXPLAIN ANALYZE on representative queries
- Rollback migration drops all tables cleanly

---

### Task 1.3: API Key Authentication & Rate Limiting

**What:** Implement API key creation, hashing (bcrypt), validation middleware, and Redis-backed rate limiting (per key, sliding window).

**Design:**
```typescript
// packages/api/src/middleware/auth.ts
import { FastifyRequest, FastifyReply } from 'fastify';
import bcrypt from 'bcrypt';

export async function authenticateApiKey(
  request: FastifyRequest,
  reply: FastifyReply
): Promise<void> {
  const apiKey = request.headers['x-api-key'] as string;
  if (!apiKey) {
    reply.code(401).send({ error: 'Missing X-API-Key header' });
    return;
  }

  // Look up key by prefix (first 8 chars stored in plaintext for lookup)
  const prefix = apiKey.substring(0, 8);
  const keyRecord = await db.apiKeys.findByPrefix(prefix);

  if (!keyRecord || !await bcrypt.compare(apiKey, keyRecord.key_hash)) {
    reply.code(401).send({ error: 'Invalid API key' });
    return;
  }

  if (keyRecord.revoked_at || (keyRecord.expires_at && keyRecord.expires_at < new Date())) {
    reply.code(401).send({ error: 'API key expired or revoked' });
    return;
  }

  // Set tenant context for RLS
  await db.query("SELECT set_config('app.current_tenant_id', $1, true)", [keyRecord.tenant_id]);
  request.tenantId = keyRecord.tenant_id;
  request.apiKeyId = keyRecord.id;
}

// packages/api/src/middleware/rateLimit.ts
// Sliding window rate limiter using Redis ZRANGEBYSCORE
export async function checkRateLimit(
  apiKeyId: string,
  limitRpm: number
): Promise<{ allowed: boolean; remaining: number; resetAt: Date }> {
  const key = `rate:${apiKeyId}`;
  const now = Date.now();
  const windowStart = now - 60_000; // 1-minute window

  const pipeline = redis.pipeline();
  pipeline.zremrangebyscore(key, 0, windowStart);
  pipeline.zadd(key, now, `${now}:${crypto.randomUUID()}`);
  pipeline.zcard(key);
  pipeline.expire(key, 120);
  const results = await pipeline.exec();

  const count = results[2][1] as number;
  return {
    allowed: count <= limitRpm,
    remaining: Math.max(0, limitRpm - count),
    resetAt: new Date(now + 60_000),
  };
}
```

**Testing:**
- Valid API key returns 200 with correct tenant context
- Invalid API key returns 401 with error message
- Expired API key returns 401
- Revoked API key returns 401
- Rate limit: 61st request within 1 minute returns 429 (for default 60 RPM limit)
- Rate limit headers (`X-RateLimit-Remaining`, `X-RateLimit-Reset`) are present on all responses
- API key creation returns the key only once; subsequent lookups use the hash

---

### Task 1.4: Data Source Adapter Interface & First Adapter (PDL)

**What:** Define the `DataSourceAdapter` interface and implement the People Data Labs adapter as the first concrete implementation.

**Design:**
```typescript
// packages/core/src/types/adapter.ts
export interface PersonLookupInput {
  email?: string;
  firstName?: string;
  lastName?: string;
  company?: string;
  linkedinUrl?: string;
}

export interface CompanyLookupInput {
  domain?: string;
  name?: string;
  ticker?: string;
}

export type EnrichmentField =
  | 'full_name' | 'first_name' | 'last_name'
  | 'job_title' | 'seniority' | 'department'
  | 'emails' | 'phones' | 'linkedin_url'
  | 'company_name' | 'company_domain' | 'employee_count'
  | 'industry' | 'location' | 'experience' | 'education' | 'skills';

export interface EnrichmentResult {
  matched: boolean;
  confidence: number;
  fields: Partial<Record<EnrichmentField, unknown>>;
  rawResponse?: Record<string, unknown>;
  durationMs: number;
  costUsd: number;
}

export interface DataSourceAdapter {
  readonly name: string;
  readonly displayName: string;
  readonly supportedFields: EnrichmentField[];
  readonly costPerLookup: number;
  readonly lawfulBasis: string;

  enrichPerson(input: PersonLookupInput): Promise<EnrichmentResult>;
  enrichCompany(input: CompanyLookupInput): Promise<EnrichmentResult>;
  healthCheck(): Promise<{ healthy: boolean; latencyMs: number }>;
}

// packages/adapters/src/pdl/pdl-adapter.ts
import PDLJS from 'peopledatalabs';

export class PdlAdapter implements DataSourceAdapter {
  readonly name = 'pdl';
  readonly displayName = 'People Data Labs';
  readonly supportedFields: EnrichmentField[] = [
    'full_name', 'first_name', 'last_name', 'job_title', 'seniority',
    'department', 'emails', 'phones', 'linkedin_url', 'company_name',
    'company_domain', 'employee_count', 'industry', 'location',
    'experience', 'education', 'skills',
  ];
  readonly costPerLookup = 0.004;
  readonly lawfulBasis = 'legitimate_interest';

  private client: PDLJS;

  constructor(apiKey: string) {
    this.client = new PDLJS({ apiKey });
  }

  async enrichPerson(input: PersonLookupInput): Promise<EnrichmentResult> {
    const start = Date.now();
    try {
      const response = await this.client.person.enrichment({
        email: input.email,
        first_name: input.firstName,
        last_name: input.lastName,
        company: input.company,
        profile: input.linkedinUrl,
      });

      return {
        matched: true,
        confidence: response.likelihood ?? 0,
        fields: this.mapPersonFields(response.data),
        rawResponse: response.data,
        durationMs: Date.now() - start,
        costUsd: this.costPerLookup,
      };
    } catch (error: any) {
      if (error.status === 404) {
        return {
          matched: false, confidence: 0, fields: {},
          durationMs: Date.now() - start, costUsd: 0,
        };
      }
      throw error;
    }
  }

  // ... enrichCompany, healthCheck, mapPersonFields implementations
}
```

**Testing:**
- PDL adapter returns correctly mapped fields for a known test email (use PDL sandbox/test key)
- PDL adapter returns `{ matched: false }` for a non-existent email
- PDL adapter handles 429 (rate limit) from PDL API and throws retryable error
- PDL adapter handles network timeout and throws appropriate error
- PDL adapter `healthCheck()` returns healthy status and latency when PDL API is reachable
- Adapter conforms to `DataSourceAdapter` interface (TypeScript compilation check)
- Field mapping covers all supported fields (unit test with fixture PDL response)

---

### Definition of Done -- Phase 1

- [ ] Monorepo builds and lints without errors
- [ ] Docker Compose starts PostgreSQL 16 + Redis 7 locally
- [ ] Database migration creates all core tables with RLS enabled
- [ ] API key auth middleware validates keys and sets tenant context
- [ ] Rate limiter enforces RPM limits via Redis sliding window
- [ ] PDL adapter implements `DataSourceAdapter` interface with full field mapping
- [ ] CI pipeline runs build + lint + unit tests on every push
- [ ] All tests pass; no TypeScript errors; no lint warnings

---

## Phase 2: Core Enrichment Engine

**Goal:** Implement single-record person and company enrichment via a single data source, with result persistence and the enrichment_events audit log.

**Duration:** 2 weeks

### Task 2.1: Person Enrichment Pipeline

**What:** Build the person enrichment service that accepts lookup parameters, calls a data source adapter, persists the enriched person record, stores associated emails/phones in their own tables, and appends an enrichment event to the audit log.

**Design:**
```typescript
// packages/engine/src/enrichment/person-enrichment.ts
export class PersonEnrichmentService {
  constructor(
    private db: Database,
    private adapters: Map<string, DataSourceAdapter>,
    private eventLogger: EnrichmentEventLogger,
  ) {}

  async enrich(
    tenantId: string,
    input: PersonLookupInput,
    sourceName: string = 'pdl',
  ): Promise<EnrichedPerson> {
    const adapter = this.adapters.get(sourceName);
    if (!adapter) throw new AdapterNotFoundError(sourceName);

    // Check for existing person match
    const existing = await this.findExistingPerson(tenantId, input);

    // Call adapter
    const result = await adapter.enrichPerson(input);
    if (!result.matched) {
      await this.eventLogger.log({
        entityType: 'person',
        entityId: existing?.id ?? null,
        eventType: 'person.enrichment_attempted',
        payload: { source: sourceName, matched: false, input },
        dataSourceId: await this.getSourceId(sourceName),
        tenantId,
      });
      throw new PersonNotFoundError(input);
    }

    // Upsert person record
    const person = await this.upsertPerson(tenantId, existing?.id, result, sourceName);

    // Upsert emails into separate table
    if (result.fields.emails) {
      await this.upsertEmails(person.id, result.fields.emails, sourceName);
    }

    // Upsert phones into separate table
    if (result.fields.phones) {
      await this.upsertPhones(person.id, result.fields.phones, sourceName);
    }

    // Log enrichment event
    await this.eventLogger.log({
      entityType: 'person',
      entityId: person.id,
      eventType: 'person.enriched',
      payload: {
        source: sourceName,
        fieldsEnriched: Object.keys(result.fields),
        confidence: result.confidence,
        costUsd: result.costUsd,
        durationMs: result.durationMs,
      },
      dataSourceId: await this.getSourceId(sourceName),
      lawfulBasis: adapter.lawfulBasis,
      tenantId,
    });

    return this.assemblePersonResponse(person);
  }
}
```

**Testing:**
- Enrich a person by email; person record is created with all mapped fields
- Enrich same person again; record is updated (upserted), not duplicated
- Emails are stored in `emails` table with correct `person_id` and `source_id`
- Phones are stored in `phones` table with correct `person_id`
- Enrichment event is appended to `enrichment_events` with correct payload
- Non-existent person returns 404 with `PersonNotFoundError`
- Invalid adapter name returns 400 with `AdapterNotFoundError`
- Database transaction rolls back all inserts if any step fails

---

### Task 2.2: Company Enrichment Pipeline

**What:** Build the equivalent company enrichment service with domain-based lookup, firmographic data persistence, and JSONB fields for technologies/funding/locations.

**Design:**
```typescript
// packages/engine/src/enrichment/company-enrichment.ts
export class CompanyEnrichmentService {
  async enrich(
    tenantId: string,
    input: CompanyLookupInput,
    sourceName: string = 'pdl',
  ): Promise<EnrichedCompany> {
    const adapter = this.adapters.get(sourceName);
    const result = await adapter.enrichCompany(input);

    if (!result.matched) {
      throw new CompanyNotFoundError(input);
    }

    const company = await this.upsertCompany(tenantId, result, sourceName);

    await this.eventLogger.log({
      entityType: 'company',
      entityId: company.id,
      eventType: 'company.enriched',
      payload: {
        source: sourceName,
        fieldsEnriched: Object.keys(result.fields),
        confidence: result.confidence,
      },
      dataSourceId: await this.getSourceId(sourceName),
      lawfulBasis: adapter.lawfulBasis,
      tenantId,
    });

    return company;
  }
}
```

**Testing:**
- Enrich a company by domain; company record is created with firmographics in JSONB
- Technologies, funding, and locations stored in respective JSONB columns
- Enrichment event appended with company entity type
- Duplicate domain within same tenant upserts rather than creating duplicate
- Company linked to person via `current_company_id` when person enrichment includes company data

---

### Task 2.3: REST API Endpoints (Person & Company Enrich)

**What:** Expose `POST /v1/person/enrich` and `POST /v1/company/enrich` endpoints with JSON Schema request validation and OpenAPI-aligned response format.

**Design:**
```typescript
// packages/api/src/routes/v1/person.ts
const personEnrichSchema = {
  body: {
    type: 'object',
    properties: {
      email: { type: 'string', format: 'email' },
      first_name: { type: 'string' },
      last_name: { type: 'string' },
      company: { type: 'string' },
      linkedin_url: { type: 'string', format: 'uri' },
    },
    anyOf: [
      { required: ['email'] },
      { required: ['linkedin_url'] },
      { required: ['first_name', 'last_name', 'company'] },
    ],
  },
  response: {
    200: {
      type: 'object',
      properties: {
        id: { type: 'string', format: 'uuid' },
        full_name: { type: 'string' },
        first_name: { type: 'string' },
        last_name: { type: 'string' },
        job_title: { type: 'string' },
        seniority: { type: 'string' },
        department: { type: 'string' },
        emails: { type: 'array', items: { $ref: '#/$defs/Email' } },
        phones: { type: 'array', items: { $ref: '#/$defs/Phone' } },
        company: { $ref: '#/$defs/CompanySummary' },
        linkedin_url: { type: 'string' },
        confidence: { type: 'number' },
        sources: { type: 'array', items: { type: 'string' } },
      },
    },
  },
};

app.post('/v1/person/enrich', {
  schema: personEnrichSchema,
  preHandler: [authenticateApiKey],
}, async (request, reply) => {
  const result = await personEnrichmentService.enrich(
    request.tenantId,
    request.body,
  );
  return reply.code(200).send(result);
});
```

**Testing:**
- `POST /v1/person/enrich` with valid email returns 200 with enriched person
- Request missing all of email, linkedin_url, and (first_name + last_name + company) returns 400
- Request without `X-API-Key` header returns 401
- Request with revoked key returns 401
- Response body matches OpenAPI schema (JSON Schema validation)
- Response includes `X-RateLimit-Remaining` header
- `POST /v1/company/enrich` with valid domain returns 200 with enriched company

---

### Task 2.4: Enrichment Request Logging

**What:** Create the `enrichment_requests` table and log every API call with input parameters, result status, matched entity IDs, and performance metrics.

**Design:**
```sql
-- Migration 002_enrichment_requests.sql
CREATE TABLE enrichment_requests (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    api_key_id      UUID REFERENCES api_keys(id),
    request_type    TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'pending',
    input_params    JSONB NOT NULL,
    matched_person_id UUID REFERENCES persons(id),
    matched_company_id UUID REFERENCES companies(id),
    match_confidence NUMERIC(3,2),
    waterfall_log   JSONB NOT NULL DEFAULT '[]',
    total_cost_usd  NUMERIC(10,6),
    duration_ms     INT,
    credits_consumed INT NOT NULL DEFAULT 1,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ
);
```

**Testing:**
- Every enrichment API call creates a row in `enrichment_requests`
- Status is `completed` on success, `not_found` when no match, `error` on failure
- `duration_ms` is accurate (within 50ms of actual execution time)
- `GET /v1/usage` endpoint returns request count and credit usage for the tenant

---

### Definition of Done -- Phase 2

- [ ] `POST /v1/person/enrich` returns enriched person data from PDL
- [ ] `POST /v1/company/enrich` returns enriched company data from PDL
- [ ] Person and company records are persisted to PostgreSQL
- [ ] Emails and phones stored in separate tables with source attribution
- [ ] Enrichment events appended to audit log for every enrichment
- [ ] Enrichment requests logged with status, duration, and cost
- [ ] Request validation rejects invalid inputs with descriptive 400 errors
- [ ] Integration tests cover success, not-found, and error paths

---

## Phase 3: Email Verification

**Goal:** Build the email verification pipeline: syntax validation (RFC 5322), MX lookup, SMTP probe (RFC 5321), catch-all detection, disposable domain detection, and role-based email detection.

**Duration:** 2 weeks

### Task 3.1: Email Syntax Validator (RFC 5322)

**What:** Implement RFC 5322-compliant email address syntax validation, rejecting malformed addresses before any network operations.

**Design:**
```typescript
// packages/engine/src/verification/syntax-validator.ts
// Full RFC 5322 local-part + domain validation
// Handle: quoted strings, dot-atom, domain literals
// Reject: bare IP addresses, consecutive dots, trailing dots

export function validateEmailSyntax(email: string): SyntaxResult {
  // 1. Split on last '@'
  // 2. Validate local-part per RFC 5322 Section 3.2.3
  // 3. Validate domain per RFC 5321 Section 2.3.5
  // 4. Check overall length (max 254 chars per RFC 5321)
  return { valid: boolean, reason?: string };
}
```

**Testing:**
- Valid addresses: `user@example.com`, `"user name"@example.com`, `user+tag@sub.domain.co.uk`
- Invalid addresses: `user@`, `@domain.com`, `user@.com`, `user@domain..com`, `user name@domain.com`
- Edge cases: maximum-length addresses (254 chars), internationalized domain names
- Reject addresses exceeding 254 total characters

---

### Task 3.2: MX Record Lookup & SMTP Prober

**What:** Implement DNS MX record resolution and SMTP RCPT TO probe to verify mailbox existence without sending email.

**Design:**
```typescript
// packages/engine/src/verification/smtp-prober.ts
import { Resolver } from 'dns/promises';
import net from 'net';

export class SmtpProber {
  async probe(email: string): Promise<SmtpProbeResult> {
    const domain = email.split('@')[1];

    // 1. Resolve MX records, sorted by priority
    const mxRecords = await this.resolveMx(domain);
    if (mxRecords.length === 0) {
      return { mxFound: false, verdict: 'invalid' };
    }

    // 2. Connect to lowest-priority MX host
    // 3. EHLO → MAIL FROM:<> → RCPT TO:<email>
    // 4. Interpret response code:
    //    250 → valid
    //    550 → invalid (mailbox not found)
    //    452 → risky (mailbox full)
    //    421/451 → temporary, retry later

    // 5. Detect catch-all: probe with random UUID prefix
    //    If random address also returns 250, domain is catch-all
  }
}
```

**Testing:**
- Known valid email returns `{ verdict: 'valid', smtpCode: 250 }`
- Known invalid email at valid domain returns `{ verdict: 'invalid', smtpCode: 550 }`
- Domain with no MX records returns `{ mxFound: false, verdict: 'invalid' }`
- Catch-all domain detected correctly (probe with random address returns 250)
- SMTP timeout after 10 seconds returns `{ verdict: 'unknown', reason: 'timeout' }`
- Disposable domain detection (check against disposable-email-domains list)
- Role-based detection: `info@`, `admin@`, `support@`, `sales@` flagged as role-based

---

### Task 3.3: Email Verification Table & API Endpoint

**What:** Create the `email_verifications` table storing full probe results and expose `POST /v1/email/verify` endpoint.

**Design:**
```sql
-- Migration 003_email_verifications.sql
CREATE TABLE email_verifications (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email_id            UUID REFERENCES emails(id) ON DELETE CASCADE,
    email_address       TEXT NOT NULL,
    syntax_valid        BOOLEAN NOT NULL,
    mx_found            BOOLEAN,
    mx_hostname         TEXT,
    smtp_connectable    BOOLEAN,
    smtp_response_code  INT,
    smtp_response_text  TEXT,
    mailbox_exists      BOOLEAN,
    is_catch_all        BOOLEAN,
    is_disposable       BOOLEAN,
    is_role_based       BOOLEAN,
    verdict             TEXT NOT NULL,
    confidence          NUMERIC(3,2),
    verification_ms     INT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

**Testing:**
- `POST /v1/email/verify` with valid email returns full verification result
- Verification result updates `emails.verification_status` and `emails.verified_at`
- Verification result is stored in `email_verifications` table with all probe details
- Rate limiting on verification endpoint (max 10 req/sec to prevent SMTP abuse)
- Cached verification result returned if email was verified within last 24 hours

---

### Definition of Done -- Phase 3

- [ ] RFC 5322 syntax validation with 100% test coverage on edge cases
- [ ] MX record lookup resolves and sorts by priority
- [ ] SMTP prober connects, probes, and interprets response codes correctly
- [ ] Catch-all, disposable, and role-based detection working
- [ ] `POST /v1/email/verify` endpoint returns structured verification result
- [ ] Verification results persisted with full probe details
- [ ] Integration test against real email addresses (test accounts)

---

## Phase 4: Waterfall Orchestration

**Goal:** Implement the multi-source waterfall engine that queries data sources in configurable priority order until all requested fields are filled, with cost tracking and fill-rate reporting.

**Duration:** 3 weeks

### Task 4.1: Waterfall Configuration Model

**What:** Create per-tenant waterfall configurations that define source ordering, conditional execution (by geography, industry), cost budgets, and timeout limits.

**Design:**
```sql
-- Migration 004_waterfall_configs.sql
CREATE TABLE waterfall_configs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            TEXT NOT NULL,
    request_type    TEXT NOT NULL CHECK (request_type IN ('person', 'company')),
    is_default      BOOLEAN NOT NULL DEFAULT FALSE,
    steps           JSONB NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, name)
);
```

```typescript
// packages/engine/src/waterfall/types.ts
export interface WaterfallStep {
  sourceName: string;          // adapter name
  priority: number;            // execution order
  fieldsRequested: EnrichmentField[];
  conditions?: {
    countryIn?: string[];      // only execute for these countries
    industryIn?: string[];
    seniorityIn?: string[];
  };
  maxCostUsd?: number;         // skip if cost exceeds budget
  timeoutMs?: number;          // per-step timeout
  fallbackOnly?: boolean;      // only execute if prior steps missed fields
}

export interface WaterfallConfig {
  name: string;
  requestType: 'person' | 'company';
  steps: WaterfallStep[];
}
```

**Testing:**
- CRUD endpoints for waterfall configurations: create, read, update, delete
- Default configuration created for new tenants
- Validation rejects configs referencing non-existent data sources
- Validation rejects configs with duplicate priorities

---

### Task 4.2: Waterfall Executor

**What:** Build the core waterfall execution engine that iterates through configured sources, tracks which fields are still missing, and stops when all fields are filled or all sources are exhausted.

**Design:**
```typescript
// packages/engine/src/waterfall/executor.ts
export class WaterfallExecutor {
  async execute(
    config: WaterfallConfig,
    input: PersonLookupInput | CompanyLookupInput,
    requestedFields: EnrichmentField[],
    context: WaterfallContext,
  ): Promise<WaterfallResult> {
    const missingFields = new Set(requestedFields);
    const mergedResult: Partial<Record<EnrichmentField, unknown>> = {};
    const stepResults: WaterfallStepResult[] = [];
    let totalCostUsd = 0;

    const sortedSteps = config.steps
      .filter(step => this.evaluateConditions(step, context))
      .sort((a, b) => a.priority - b.priority);

    for (const step of sortedSteps) {
      if (missingFields.size === 0) break;
      if (step.fallbackOnly && stepResults.some(s => s.matched)) continue;
      if (step.maxCostUsd && totalCostUsd + step.costPerLookup > step.maxCostUsd) continue;

      const adapter = this.adapters.get(step.sourceName);
      const result = await this.executeStep(adapter, input, step.timeoutMs);

      stepResults.push({
        source: step.sourceName,
        status: result.matched ? 'success' : 'not_found',
        fieldsReturned: Object.keys(result.fields),
        confidence: result.confidence,
        durationMs: result.durationMs,
        costUsd: result.costUsd,
      });

      if (result.matched) {
        // Merge fields: keep higher confidence values
        for (const [field, value] of Object.entries(result.fields)) {
          if (missingFields.has(field as EnrichmentField) || 
              result.confidence > (mergedResult[`${field}_confidence`] ?? 0)) {
            mergedResult[field as EnrichmentField] = value;
            missingFields.delete(field as EnrichmentField);
          }
        }
        totalCostUsd += result.costUsd;
      }
    }

    return {
      fields: mergedResult,
      stepResults,
      totalCostUsd,
      fillRate: (requestedFields.length - missingFields.size) / requestedFields.length,
      sourcesQueried: stepResults.map(s => s.source),
      sourcesMatched: stepResults.filter(s => s.status === 'success').map(s => s.source),
    };
  }
}
```

**Testing:**
- Waterfall queries sources in priority order and stops when all fields are filled
- If source 1 returns partial data, source 2 fills remaining fields
- Conditional steps are skipped when conditions are not met (e.g., US-only source skipped for EU contact)
- Cost budget enforcement: step is skipped when cumulative cost exceeds `maxCostUsd`
- Timeout enforcement: step returns `error` status after `timeoutMs`
- Fallback-only steps are skipped when prior steps found a match
- Fill rate is calculated correctly (e.g., 0.75 when 3/4 requested fields are returned)
- Empty waterfall config returns 0% fill rate with no steps executed
- Waterfall log is persisted to `enrichment_requests.waterfall_log`

---

### Task 4.3: Additional Data Source Adapters

**What:** Implement adapters for Apollo, Hunter.io, SEC EDGAR, and Companies House UK to provide meaningful waterfall diversity.

**Design:**
```typescript
// packages/adapters/src/apollo/apollo-adapter.ts
export class ApolloAdapter implements DataSourceAdapter {
  readonly name = 'apollo';
  readonly costPerLookup = 0.47;
  // REST API calls to docs.apollo.io/reference/people-enrichment
}

// packages/adapters/src/hunter/hunter-adapter.ts
export class HunterAdapter implements DataSourceAdapter {
  readonly name = 'hunter';
  readonly costPerLookup = 0.03;
  // Email Finder + Email Verifier APIs
}

// packages/adapters/src/sec-edgar/sec-edgar-adapter.ts
export class SecEdgarAdapter implements DataSourceAdapter {
  readonly name = 'sec_edgar';
  readonly costPerLookup = 0;  // free public API
  readonly lawfulBasis = 'public_task';
  // REST API calls to data.sec.gov
}

// packages/adapters/src/companies-house/companies-house-adapter.ts
export class CompaniesHouseAdapter implements DataSourceAdapter {
  readonly name = 'companies_house';
  readonly costPerLookup = 0;  // free public API
  readonly lawfulBasis = 'public_task';
  // REST API calls to developer.company-information.service.gov.uk
}
```

**Testing:**
- Each adapter conforms to `DataSourceAdapter` interface (TypeScript check)
- Each adapter maps response fields correctly (unit tests with fixture data)
- Each adapter handles 404/not-found responses gracefully
- Each adapter handles rate-limit (429) responses with retryable error
- SEC EDGAR adapter returns company data for known CIK (e.g., Apple Inc.)
- Companies House adapter returns company data for known company number
- Hunter adapter returns email addresses for a known domain

---

### Definition of Done -- Phase 4

- [ ] Waterfall configs are CRUD-manageable via API
- [ ] Waterfall executor queries sources in priority order with conditional execution
- [ ] Fill rate and cost tracking per request
- [ ] At least 5 adapters implemented: PDL, Apollo, Hunter, SEC EDGAR, Companies House
- [ ] Waterfall log persisted on every enrichment request
- [ ] `GET /v1/usage/fill-rates` returns per-source fill rate statistics
- [ ] Integration tests cover multi-source waterfall with partial fills

---

## Phase 5: Bulk Enrichment & API Polish

**Goal:** Add batch enrichment endpoints, CSV upload/download, webhook callbacks for async jobs, and finalize the OpenAPI 3.1 specification.

**Duration:** 2 weeks

### Task 5.1: Bulk Person Enrichment API

**What:** Implement `POST /v1/bulk/person/enrich` that accepts arrays of up to 1,000 lookup records, processes them via BullMQ, and returns results via webhook or polling.

**Design:**
```typescript
// packages/api/src/routes/v1/bulk.ts
app.post('/v1/bulk/person/enrich', {
  schema: {
    body: {
      type: 'object',
      properties: {
        records: {
          type: 'array',
          items: { $ref: '#/$defs/PersonLookupInput' },
          maxItems: 1000,
        },
        webhook_url: { type: 'string', format: 'uri' },
        waterfall_config: { type: 'string' },  // config name
      },
      required: ['records'],
    },
  },
}, async (request, reply) => {
  const job = await bulkEnrichmentQueue.add('bulk-person-enrich', {
    tenantId: request.tenantId,
    records: request.body.records,
    webhookUrl: request.body.webhook_url,
    waterfallConfig: request.body.waterfall_config,
  });
  return reply.code(202).send({
    job_id: job.id,
    status: 'processing',
    poll_url: `/v1/bulk/jobs/${job.id}`,
    total_records: request.body.records.length,
  });
});
```

**Testing:**
- Bulk enrichment with 10 records returns 202 and job_id
- `GET /v1/bulk/jobs/{jobId}` returns current status (processing, completed, failed)
- Completed job includes per-record results with match status and confidence
- Webhook URL receives POST with results when job completes
- Job with > 1,000 records returns 400 with validation error
- Partial failures (some records enriched, some not) reported accurately
- Rate limiting applies to upstream API calls within bulk jobs

---

### Task 5.2: CSV Upload/Download

**What:** Support CSV file upload for bulk enrichment and CSV download of results.

**Design:**
```typescript
// POST /v1/bulk/csv/upload
// Accepts multipart/form-data with CSV file
// Parses CSV, validates headers, creates bulk enrichment job
// Returns job_id for polling

// GET /v1/bulk/jobs/{jobId}/download
// Returns CSV with original columns + enriched columns
// Content-Type: text/csv
// Content-Disposition: attachment; filename="enriched_{jobId}.csv"
```

**Testing:**
- CSV with email column creates bulk enrichment job
- CSV with first_name, last_name, company columns creates job
- Invalid CSV (missing required columns) returns 400 with column mapping guidance
- Download endpoint returns CSV with original + enriched columns
- Large CSV (10,000 rows) processes without memory issues (streaming parser)

---

### Task 5.3: OpenAPI 3.1 Specification

**What:** Generate and publish the complete OpenAPI 3.1 specification from Fastify route schemas, covering all endpoints, request/response schemas, authentication, and error formats.

**Design:**
- Use `@fastify/swagger` to auto-generate from route schemas
- Publish at `GET /v1/openapi.json` and `GET /v1/docs` (Swagger UI)
- Include `securitySchemes` for API key authentication
- Define reusable `$defs` for Person, Company, Email, Phone, Error

**Testing:**
- Generated OpenAPI spec validates against OAS 3.1 JSON Schema
- All routes have documented request/response schemas
- Swagger UI renders at `/v1/docs` and all endpoints are testable
- Contract tests validate API responses against OpenAPI spec

---

### Definition of Done -- Phase 5

- [ ] Bulk person enrichment processes up to 1,000 records per request
- [ ] CSV upload and download working for bulk jobs
- [ ] Webhook notifications delivered on job completion
- [ ] Job polling endpoint returns accurate status and progress
- [ ] OpenAPI 3.1 spec published and validates
- [ ] Swagger UI serves interactive documentation
- [ ] Performance: bulk job of 100 records completes within 5 minutes

---

## Phase 6: CRM Integration

**Goal:** Implement bidirectional sync with Salesforce and HubSpot, including OAuth connection flow, field mapping, and scheduled re-enrichment.

**Duration:** 3 weeks

### Task 6.1: CRM Connection OAuth Flow

**What:** Implement OAuth 2.0 authorization-code flow for connecting Salesforce and HubSpot accounts, with encrypted token storage and automatic refresh.

**Design:**
```sql
-- Migration 005_crm.sql
CREATE TABLE crm_connections (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    crm_type        TEXT NOT NULL CHECK (crm_type IN ('salesforce', 'hubspot', 'pipedrive')),
    credentials_enc BYTEA NOT NULL,
    instance_url    TEXT,
    config          JSONB NOT NULL DEFAULT '{}',
    last_synced_at  TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE crm_sync_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    crm_connection_id UUID NOT NULL REFERENCES crm_connections(id),
    entity_type     TEXT NOT NULL,
    entity_id       UUID NOT NULL,
    crm_record_id   TEXT NOT NULL,
    sync_details    JSONB NOT NULL,
    status          TEXT NOT NULL DEFAULT 'success',
    error_message   TEXT,
    synced_at       TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

**Testing:**
- Salesforce OAuth flow: redirect to Salesforce, callback receives code, token exchanged and stored encrypted
- HubSpot OAuth flow: same pattern with HubSpot endpoints
- Token refresh: expired token is refreshed automatically before sync
- Revoked token: sync fails gracefully and marks connection as `needs_reauth`
- Credentials are AES-256 encrypted at rest; never logged in plaintext

---

### Task 6.2: CRM Field Mapping & Sync Engine

**What:** Build configurable field mappings between enrichment fields and CRM fields, with sync logic that respects overwrite rules (only-if-empty, always-overwrite).

**Design:**
```typescript
// packages/crm/src/sync-engine/sync.ts
export class CrmSyncEngine {
  async pushEnrichedData(
    connection: CrmConnection,
    person: EnrichedPerson,
  ): Promise<SyncResult> {
    const adapter = this.getCrmAdapter(connection.crm_type);
    const mappings = connection.config.field_mappings;

    // Build CRM record payload from enrichment data + mappings
    const crmPayload = this.mapFields(person, mappings, connection.config.overwrite_rules);

    // Find or create CRM record
    const existingRecord = await adapter.findByEmail(connection, person.primary_email);

    if (existingRecord) {
      return adapter.updateRecord(connection, existingRecord.id, crmPayload);
    } else {
      return adapter.createRecord(connection, crmPayload);
    }
  }
}
```

**Testing:**
- Enriched person creates new Contact in Salesforce with mapped fields
- Enriched person updates existing Contact (matched by email)
- Overwrite rules: `only_if_empty` skips populated CRM fields; `always_overwrite` updates regardless
- Enriched company creates/updates Account in Salesforce
- HubSpot sync creates/updates Contact and Company objects
- Sync log records every create/update/skip with field details
- Sync failure (API error) is logged and does not block subsequent records

---

### Task 6.3: Scheduled CRM Re-enrichment

**What:** Implement scheduled jobs that periodically pull stale CRM contacts, re-enrich them through the waterfall, and push updated data back to the CRM.

**Design:**
```typescript
// BullMQ repeatable job
crmSyncQueue.add('scheduled-re-enrichment', {
  connectionId: connection.id,
  staleThresholdDays: 90,  // re-enrich contacts not updated in 90 days
  batchSize: 100,
}, {
  repeat: { cron: '0 2 * * *' },  // daily at 2 AM
});
```

**Testing:**
- Scheduled job pulls contacts with `last_enriched_at` older than threshold
- Re-enrichment runs through waterfall and updates CRM if data changed
- Job respects CRM API rate limits (Salesforce: 100 requests/day for bulk API)
- Job processes in batches of configurable size
- Job idempotent: running twice does not duplicate CRM updates

---

### Definition of Done -- Phase 6

- [ ] Salesforce and HubSpot OAuth connection flows working
- [ ] Configurable field mappings between enrichment and CRM fields
- [ ] Push enriched data to CRM (create or update records)
- [ ] Overwrite rules respected (only-if-empty, always-overwrite)
- [ ] Sync log records every CRM interaction
- [ ] Scheduled re-enrichment runs daily for stale contacts
- [ ] Token refresh handles expired credentials automatically

---

## Phase 7: Job Change Detection

**Goal:** Detect when enriched contacts change jobs by comparing re-enrichment results against stored data, generate notifications, and optionally trigger CRM updates.

**Duration:** 2 weeks

### Task 7.1: Job Change Detection Logic

**What:** Compare incoming enrichment data against stored person records to detect changes in job_title, company, or seniority. Emit job change events and store them for notification.

**Design:**
```sql
-- Migration 006_job_changes.sql
CREATE TABLE job_change_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    person_id       UUID NOT NULL REFERENCES persons(id),
    change_data     JSONB NOT NULL,
    notification_sent BOOLEAN NOT NULL DEFAULT FALSE,
    notification_sent_at TIMESTAMPTZ,
    detected_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_job_changes_tenant ON job_change_events(tenant_id, detected_at);
CREATE INDEX idx_job_changes_unsent ON job_change_events(notification_sent)
    WHERE notification_sent = FALSE;
```

```typescript
// packages/engine/src/job-change/detector.ts
export class JobChangeDetector {
  async detect(
    existingPerson: Person,
    newEnrichment: EnrichmentResult,
  ): Promise<JobChangeEvent | null> {
    const newTitle = newEnrichment.fields.job_title;
    const newCompany = newEnrichment.fields.company_name;

    // Heuristics:
    // 1. Company name changed (fuzzy match to handle "Inc" vs "Inc.")
    // 2. Job title changed AND company changed (role change + company change)
    // 3. Seniority level changed significantly (e.g., IC to VP)
    // 4. LinkedIn URL profile summary changed (if available)

    if (this.isCompanyChange(existingPerson, newCompany)) {
      return {
        personId: existingPerson.id,
        previous: {
          companyName: existingPerson.currentCompany?.name,
          title: existingPerson.job_title,
          seniority: existingPerson.seniority,
        },
        current: {
          companyName: newCompany,
          title: newTitle,
          seniority: newEnrichment.fields.seniority,
        },
        confidence: this.calculateConfidence(existingPerson, newEnrichment),
        detectionSource: 'enrichment_delta',
      };
    }
    return null;
  }
}
```

**Testing:**
- Person changes company: job change event is created with previous and new data
- Person changes title at same company: no job change event (title changes are not company changes)
- Fuzzy company matching: "Acme Corp" vs "Acme Corporation" treated as same company
- Confidence score reflects how many signals confirm the change
- Job change event is appended to `enrichment_events` audit log

---

### Task 7.2: Job Change Notification System

**What:** Implement webhook and email notifications for detected job changes, with configurable notification preferences per tenant.

**Design:**
```typescript
// packages/engine/src/job-change/notifier.ts
export class JobChangeNotifier {
  async notifyAll(tenantId: string): Promise<void> {
    const unsentChanges = await this.db.jobChangeEvents.findUnsent(tenantId);
    const tenant = await this.db.tenants.findById(tenantId);

    if (tenant.settings.job_change_webhook_url) {
      await this.sendWebhook(tenant.settings.job_change_webhook_url, unsentChanges);
    }

    if (tenant.settings.job_change_email_recipients) {
      await this.sendEmailDigest(tenant.settings.job_change_email_recipients, unsentChanges);
    }

    // Mark as sent
    await this.db.jobChangeEvents.markSent(unsentChanges.map(e => e.id));
  }
}
```

**Testing:**
- Webhook notification sent within 5 minutes of job change detection
- Email digest sent with configurable frequency (immediate, daily, weekly)
- Webhook payload includes previous and new company/title/seniority
- Failed webhook delivery retried 3 times with exponential backoff
- `GET /v1/job-changes` endpoint returns detected changes for the tenant with pagination

---

### Definition of Done -- Phase 7

- [ ] Job change detection identifies company changes via enrichment deltas
- [ ] Job change events stored with previous/new data and confidence scores
- [ ] Webhook notifications delivered for detected job changes
- [ ] `GET /v1/job-changes` endpoint returns paginated job change history
- [ ] Job changes visible in `enrichment_events` audit log
- [ ] Fuzzy company name matching prevents false positives

---

## Phase 8: GDPR Compliance & Data Provenance

**Goal:** Implement full GDPR compliance: data subject request handling (access, erasure, portability), field-level provenance tracking, lawful basis documentation, and opt-out mechanisms.

**Duration:** 2 weeks

### Task 8.1: Data Subject Request Handling

**What:** Implement `POST /v1/gdpr/requests` endpoint supporting access, erasure, rectification, and portability requests per GDPR Articles 15-22. Erasure must cascade through all tables (persons, emails, phones, enrichment_events).

**Design:**
```sql
-- Migration 007_gdpr.sql
CREATE TABLE data_subject_requests (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    person_id       UUID REFERENCES persons(id),
    request_type    TEXT NOT NULL CHECK (request_type IN (
        'access', 'rectification', 'erasure', 'restriction',
        'portability', 'objection', 'ccpa_do_not_sell', 'ccpa_delete'
    )),
    status          TEXT NOT NULL DEFAULT 'received',
    requester_email TEXT NOT NULL,
    details         JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

```typescript
// packages/engine/src/gdpr/request-handler.ts
export class GdprRequestHandler {
  async handleErasure(request: DataSubjectRequest): Promise<void> {
    const person = await this.db.persons.findById(request.person_id);

    // 1. Export data before erasure (for audit trail)
    const exportData = await this.exportPersonData(person.id);

    // 2. Delete emails, phones (CASCADE)
    // 3. Clear person PII fields (set to '[ERASED]')
    // 4. Clear provenance JSONB
    // 5. Retain anonymised enrichment_events (for audit)
    // 6. Log erasure event

    await this.eventLogger.log({
      entityType: 'person',
      entityId: person.id,
      eventType: 'person.gdpr.erased',
      payload: {
        requestId: request.id,
        fieldsErased: ['full_name', 'emails', 'phones', 'experience', 'education'],
        completedAt: new Date().toISOString(),
      },
      tenantId: request.tenant_id,
    });

    // 7. Update request status
    await this.db.dataSubjectRequests.complete(request.id);
  }

  async handleAccess(request: DataSubjectRequest): Promise<PersonDataExport> {
    // Return all stored data for the person in machine-readable format
    // Include: person fields, emails, phones, provenance, enrichment history
    // Format: JSON + optional vCard (RFC 6350)
  }

  async handlePortability(request: DataSubjectRequest): Promise<Buffer> {
    // Return data in structured, commonly-used format (JSON + CSV)
    // Per GDPR Article 20
  }
}
```

**Testing:**
- Erasure request removes all PII from person record, emails, phones
- Erasure retains anonymised audit log entry (event type + timestamp, no PII)
- Access request returns all stored data for the person in JSON format
- Portability request returns data in JSON and CSV formats
- Request status transitions: received -> verified -> processing -> completed
- Duplicate erasure request for already-erased person returns 200 (idempotent)
- Erasure cascades to CRM sync (delete record from connected CRM)

---

### Task 8.2: Field-Level Provenance Dashboard Data

**What:** Implement the provenance JSONB column tracking which data source provided each field, with confidence scores and collection dates. Expose via API for the dashboard.

**Design:**
```typescript
// Provenance tracking is updated on every enrichment
// packages/engine/src/enrichment/provenance-tracker.ts
export class ProvenanceTracker {
  updateProvenance(
    existingProvenance: Record<string, FieldProvenance>,
    newFields: Record<string, unknown>,
    source: string,
    confidence: number,
  ): Record<string, FieldProvenance> {
    const updated = { ...existingProvenance };
    for (const [field, value] of Object.entries(newFields)) {
      const existing = updated[field];
      // Keep higher confidence source, or newer source if confidence is equal
      if (!existing || confidence >= existing.confidence) {
        updated[field] = {
          source,
          confidence,
          collectedAt: new Date().toISOString(),
          lawfulBasis: this.getLawfulBasis(source),
          previousValue: existing?.source ? { source: existing.source, value: existing.value } : undefined,
        };
      }
    }
    return updated;
  }
}
```

**Testing:**
- Provenance JSONB updated after every enrichment with source, confidence, date
- Higher-confidence source overwrites lower-confidence source for the same field
- `GET /v1/person/{id}/provenance` returns per-field provenance breakdown
- Provenance includes lawful basis for each field per GDPR Article 14

---

### Task 8.3: Opt-Out Mechanism

**What:** Implement do-not-contact and do-not-sell flags with API endpoints for data subjects to opt out, and enforcement in the enrichment pipeline.

**Design:**
```typescript
// POST /v1/gdpr/opt-out
// Body: { email: "person@example.com", type: "do_not_contact" | "ccpa_do_not_sell" }

// Enrichment pipeline checks opt-out status before returning data
// If person has do_not_contact: true, enrichment returns 451 Unavailable For Legal Reasons
```

**Testing:**
- Opt-out request sets `privacy.do_not_contact: true` on person record
- Subsequent enrichment requests for opted-out person return 451
- CCPA do-not-sell flag prevents person data from being included in bulk exports
- Opt-out status is checked before CRM sync (opted-out persons not pushed to CRM)

---

### Definition of Done -- Phase 8

- [ ] GDPR erasure deletes all PII while retaining anonymised audit trail
- [ ] GDPR access returns all stored data for a person
- [ ] GDPR portability exports data in JSON and CSV formats
- [ ] Field-level provenance tracks source, confidence, and lawful basis per field
- [ ] Opt-out flags enforced in enrichment pipeline and CRM sync
- [ ] Data subject request lifecycle tracked with status transitions
- [ ] All GDPR endpoints require authentication and tenant context

---

## Phase 9: Dashboard & Self-Service

**Goal:** Build a management dashboard (Next.js) for tenant self-service: waterfall configuration, usage analytics, CRM connection management, API key management, and GDPR request tracking.

**Duration:** 3 weeks

### Task 9.1: Dashboard Authentication & Layout

**What:** Implement Next.js app with email/password login (JWT sessions), tenant-scoped navigation, and responsive layout.

**Design:**
- Next.js 15 App Router with server components
- Authentication via credentials provider (email/password stored in `users` table)
- JWT session tokens (RFC 7519) with tenant_id claim
- Layout: sidebar navigation with sections for Enrichment, Waterfall, CRM, GDPR, Usage, Settings

**Testing:**
- Login with valid credentials creates JWT session
- Invalid credentials return error with rate limiting (5 attempts per 15 minutes)
- Session expiry redirects to login
- Navigation renders all sections with correct active state

---

### Task 9.2: Usage Analytics Dashboard

**What:** Display enrichment request volume, fill rates by source, cost breakdown, and credit consumption over time.

**Design:**
```typescript
// API: GET /v1/analytics/usage
// Returns: {
//   period: "30d",
//   total_requests: 12450,
//   total_cost_usd: 342.80,
//   fill_rate: 0.82,
//   by_source: [
//     { source: "pdl", requests: 8200, fill_rate: 0.78, cost_usd: 32.80 },
//     { source: "apollo", requests: 3100, fill_rate: 0.85, cost_usd: 145.70 },
//   ],
//   by_day: [ { date: "2026-05-01", requests: 420, cost_usd: 11.20 }, ... ]
// }
```

- Charts: line chart (requests/day), bar chart (fill rate by source), pie chart (cost by source)

**Testing:**
- Analytics endpoint returns accurate counts matching `enrichment_requests` table
- Dashboard renders charts without errors for zero-data tenants (empty state)
- Date range selector filters analytics correctly (7d, 30d, 90d, custom)
- Cost breakdown matches sum of `waterfall_log` costs

---

### Task 9.3: Waterfall Configuration UI

**What:** Visual waterfall builder allowing drag-and-drop source ordering, conditional rules, and cost budget configuration.

**Testing:**
- Create new waterfall config with 3 sources in specified order
- Drag-and-drop reorder persists priority changes
- Conditional rules (country filter) render and save correctly
- Delete a waterfall config with confirmation dialog

---

### Task 9.4: API Key & CRM Connection Management

**What:** UI for creating/revoking API keys and managing CRM OAuth connections.

**Testing:**
- Create API key displays the key once, then shows masked version
- Revoke API key marks it as revoked; subsequent API calls fail
- Connect Salesforce triggers OAuth flow and shows connected status
- Disconnect CRM with confirmation dialog

---

### Definition of Done -- Phase 9

- [ ] Dashboard login and session management working
- [ ] Usage analytics with charts for requests, fill rates, and costs
- [ ] Waterfall configuration builder with drag-and-drop
- [ ] API key management (create, revoke, list)
- [ ] CRM connection management (connect, disconnect, status)
- [ ] GDPR request tracking UI showing request lifecycle
- [ ] Responsive layout works on desktop and tablet

---

## Phase 10: Advanced Features (Intent, Technographics)

**Goal:** Add technology stack detection from public sources (job postings, websites), company funding signals from Crunchbase, and intent signal overlay.

**Duration:** 3 weeks

### Task 10.1: Technographic Enrichment

**What:** Detect technology stacks from public job postings and company websites, storing results in the companies `technologies` JSONB column.

**Design:**
- Parse job postings for technology keywords (mapping: "React" -> category "Frontend Framework")
- Detect website technologies via HTTP headers, meta tags, JavaScript libraries (similar to BuiltWith/Wappalyzer approach)
- Store in `companies.technologies` JSONB array with `first_seen_at`, `last_seen_at`, `source`

**Testing:**
- Company with public job posting mentioning "Salesforce" has `{ name: "Salesforce", category: "CRM" }` in technologies
- Technology detection from website headers (e.g., `X-Powered-By: Express` -> Node.js/Express)
- `GET /v1/company/{id}` response includes `technologies` array
- Technology keyword dictionary covers 200+ common B2B technologies

---

### Task 10.2: Crunchbase Funding Data Adapter

**What:** Implement Crunchbase API adapter for company funding round data.

**Testing:**
- Known funded company returns funding rounds with amounts, dates, investors
- Funding data stored in `companies.funding` JSONB array
- Company without Crunchbase data returns enrichment with `funding: []`

---

### Task 10.3: RDAP Domain Enrichment

**What:** Implement ICANN RDAP adapter for domain registration data (company age, registrant, MX provider detection).

**Testing:**
- Domain lookup returns registration date, registrant organization
- MX provider detection: Google Workspace, Microsoft 365, custom
- RDAP data stored alongside company domain information

---

### Definition of Done -- Phase 10

- [ ] Technology stack detection from job postings and websites
- [ ] Crunchbase funding data integration
- [ ] RDAP domain enrichment with registrant and MX provider
- [ ] Industry taxonomy with ltree for hierarchical classification
- [ ] All advanced features accessible via enrichment API responses

---

## Phase 11: Performance, Scale & Observability

**Goal:** Optimize for production scale: connection pooling, query optimization, caching strategy, distributed tracing, and alerting.

**Duration:** 2 weeks

### Task 11.1: Caching Layer

**What:** Implement Redis caching for enrichment results (TTL-based), email verification results, and company data to reduce upstream API calls and costs.

**Design:**
```typescript
// Cache key: `enrich:person:{sha256(input_params)}`
// TTL: configurable per source (PDL: 24h, SEC EDGAR: 7d, Companies House: 30d)
// Cache invalidation: on re-enrichment or GDPR erasure

export class EnrichmentCache {
  async getOrEnrich(
    input: PersonLookupInput,
    enrichFn: () => Promise<EnrichmentResult>,
    ttlSeconds: number = 86400,
  ): Promise<EnrichmentResult> {
    const cacheKey = this.buildKey(input);
    const cached = await this.redis.get(cacheKey);
    if (cached) return JSON.parse(cached);

    const result = await enrichFn();
    await this.redis.setex(cacheKey, ttlSeconds, JSON.stringify(result));
    return result;
  }
}
```

**Testing:**
- Second enrichment request for same input returns cached result (no upstream API call)
- Cache TTL expires correctly and triggers fresh enrichment
- GDPR erasure invalidates cache for erased person
- Cache hit rate metrics exposed via `/v1/metrics`

---

### Task 11.2: Database Query Optimization

**What:** Optimize slow queries identified via `pg_stat_statements`, add missing indexes, implement connection pooling with PgBouncer, and partition `enrichment_events` by month.

**Testing:**
- p99 latency for `POST /v1/person/enrich` under 500ms at 100 concurrent requests
- `enrichment_events` table partitioned by month; queries on recent data use partition pruning
- Connection pool handles 200 concurrent connections without exhaustion
- `EXPLAIN ANALYZE` shows index scans (not seq scans) for all primary query patterns

---

### Task 11.3: Observability Stack

**What:** Instrument all services with OpenTelemetry traces, Prometheus metrics, and Grafana dashboards for waterfall latency, fill rates, error rates, and upstream API health.

**Design:**
- Distributed trace for each enrichment request spanning: API -> engine -> adapter -> upstream API
- Metrics: request_count, request_duration, fill_rate, cache_hit_rate, upstream_error_rate (per source)
- Dashboards: Enrichment Overview, Waterfall Performance, Source Health, Cost Tracking

**Testing:**
- Traces visible in Grafana/Tempo for end-to-end enrichment requests
- Prometheus metrics scrape endpoint at `/metrics`
- Alert fires when upstream source error rate exceeds 10% for 5 minutes
- Alert fires when fill rate drops below 50% for 15 minutes

---

### Definition of Done -- Phase 11

- [ ] Redis caching reduces upstream API calls by 40%+ for repeated lookups
- [ ] p99 API latency under 500ms at 100 concurrent requests
- [ ] Enrichment events table partitioned by month
- [ ] OpenTelemetry traces for all enrichment requests
- [ ] Prometheus metrics with Grafana dashboards for key SLIs
- [ ] Alerting on error rate spikes and fill rate degradation

---

## Phase 12: MCP Server & AI Agent Integration

**Goal:** Publish an MCP (Model Context Protocol) server that exposes enrichment capabilities to AI agents (Claude, GPT), enabling LLM-powered workflows to call enrichment as a tool.

**Duration:** 2 weeks

### Task 12.1: MCP Server Implementation

**What:** Build an MCP server that exposes `enrich_person`, `enrich_company`, `verify_email`, and `search_contacts` as tools consumable by AI agent frameworks.

**Design:**
```typescript
// packages/mcp-server/src/server.ts
import { McpServer } from '@modelcontextprotocol/sdk/server';

const server = new McpServer({
  name: 'contact-enrichment-engine',
  version: '1.0.0',
});

server.tool('enrich_person', {
  description: 'Enrich a person contact with verified email, phone, title, and company data',
  inputSchema: {
    type: 'object',
    properties: {
      email: { type: 'string', description: 'Email address to enrich' },
      first_name: { type: 'string' },
      last_name: { type: 'string' },
      company: { type: 'string' },
      linkedin_url: { type: 'string' },
    },
    anyOf: [
      { required: ['email'] },
      { required: ['linkedin_url'] },
      { required: ['first_name', 'last_name', 'company'] },
    ],
  },
}, async (input) => {
  const result = await enrichmentService.enrich(input);
  return { content: [{ type: 'text', text: JSON.stringify(result, null, 2) }] };
});

server.tool('verify_email', { /* ... */ });
server.tool('enrich_company', { /* ... */ });
server.tool('search_contacts', { /* ... */ });
```

**Testing:**
- MCP server starts and registers all tools
- `enrich_person` tool returns enriched data when called with valid email
- `verify_email` tool returns verification result
- Tool input validation rejects malformed requests
- MCP server handles concurrent tool calls without race conditions

---

### Task 12.2: AI-Powered Research Endpoint

**What:** Implement an experimental `POST /v1/research` endpoint that uses LLM-parsed public data (news articles, job postings, conference talks) to generate unstructured enrichment insights about a person or company.

**Testing:**
- Research endpoint returns structured insights about a known public figure
- Rate limiting prevents abuse (max 10 research requests per hour per tenant)
- Research results are not cached (always fresh from public sources)
- Response includes source URLs for all cited information

---

### Definition of Done -- Phase 12

- [ ] MCP server published with 4 tools (enrich_person, enrich_company, verify_email, search_contacts)
- [ ] MCP server consumable by Claude and other MCP-compatible agents
- [ ] npm package published for easy installation: `npx contact-enrichment-mcp`
- [ ] AI research endpoint returns structured insights from public data
- [ ] Documentation includes MCP server setup guide

---

## Cross-Phase Definition of Done

Each phase must satisfy these criteria before being marked complete:

1. **All unit tests pass** with >80% code coverage on new code
2. **Integration tests** cover success, error, and edge-case paths
3. **No TypeScript errors** (`tsc --noEmit` passes)
4. **No lint warnings** (`eslint` passes with zero warnings)
5. **API endpoints** return correct HTTP status codes and error messages
6. **Database migrations** are reversible (up + down)
7. **Docker Compose** environment starts and runs all tests locally
8. **CI pipeline** passes on the phase branch before merge to main
9. **OpenAPI spec** updated to reflect any new/changed endpoints
10. **CHANGELOG** updated with user-facing changes
