# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: Contact Enrichment Engine · Created: 2026-05-12

## Philosophy

The Event-Sourced model treats every enrichment action, data change, and system interaction as an immutable event appended to a single event store. The event log is the source of truth; current contact and company state is derived by replaying events into materialised read models. This architecture implements Command Query Responsibility Segregation (CQRS), where write operations append events and read operations query denormalized projections.

This approach is inspired by financial ledger systems, audit-heavy regulatory environments, and modern event-driven architectures (e.g., Kafka-backed microservices). In the contact enrichment domain, it solves a critical pain point: **data provenance**. When a customer asks "where did this phone number come from?" or "what was this person's title 6 months ago?", an event-sourced system can answer by replaying history. GDPR Article 14 requires transparency about data sources -- an event store inherently provides this because every field change is an event with a source attribution.

PostgreSQL serves as both event store (append-only `events` table with JSONB payloads) and read model host (materialised projection tables updated by triggers or async workers). This eliminates the need for a separate event streaming system (Kafka, EventStoreDB) while retaining the full audit trail benefit. Periodic snapshots compress the event stream for frequently-queried aggregates.

**Best for:** Teams building a compliance-first enrichment platform where full audit trails, temporal queries ("what did we know on date X?"), and GDPR data provenance are non-negotiable requirements.

**Trade-offs:**
- (+) Complete audit trail: every field change is an immutable event with timestamp, source, and actor
- (+) Temporal queries: reconstruct any entity's state at any point in time by replaying events up to that timestamp
- (+) GDPR provenance is inherent: each event records which data source provided which data
- (+) Waterfall debugging: the exact sequence of source queries and their results are preserved as events
- (+) Schema evolution is easier: new event types can be added without migrating existing data
- (-) Higher write amplification: every change creates an event + updates projections
- (-) Read model consistency lag: projections may be milliseconds behind the event store
- (-) More complex querying: ad-hoc queries against event payloads require JSONB operators
- (-) Snapshot management needed for entities with thousands of events
- (-) Developers must understand the event-sourcing mental model

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| RFC 5322 (Email Format) | Email validation events store RFC 5322 conformance results |
| RFC 5321 (SMTP) | SMTP probe events capture full handshake details per RFC 5321 |
| RFC 6350 (vCard 4.0) | Export projection maps to vCard properties via event replay |
| Schema.org Person/Organization | Read model field names align with Schema.org vocabulary |
| ISO 3166-1/2 (Country/Region) | Location events use ISO 3166 codes in event payloads |
| GDPR Articles 6, 14, 21 | Every event carries `lawful_basis` and `data_source` -- provenance is the architecture |
| CCPA/CPRA | Deletion events create an immutable audit record even after data removal |
| CloudEvents 1.0 | Event envelope structure follows CloudEvents specification |
| OCSF (Open Cybersecurity Schema) | Security-relevant events (API key usage, data access) follow OCSF patterns |
| OpenAPI 3.1 | Read model projections map directly to API response schemas |

---

## Event Store (Write Side)

```sql
-- ============================================================
-- CORE EVENT STORE
-- ============================================================

-- The single source of truth. All state changes are recorded here.
-- Events are immutable -- only INSERT, never UPDATE or DELETE.
CREATE TABLE events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    -- Aggregate identification
    aggregate_type  TEXT NOT NULL,           -- 'person', 'company', 'enrichment_request', 'tenant'
    aggregate_id    UUID NOT NULL,           -- the entity this event belongs to
    -- Event metadata
    event_type      TEXT NOT NULL,           -- e.g., 'person.created', 'person.email.added', 'person.enriched'
    event_version   INT NOT NULL DEFAULT 1,  -- schema version of this event type
    -- Event payload (CloudEvents-aligned envelope)
    payload         JSONB NOT NULL,          -- the event data
    -- Provenance
    tenant_id       UUID NOT NULL,
    actor_type      TEXT NOT NULL CHECK (actor_type IN ('user', 'api_key', 'system', 'data_source')),
    actor_id        UUID,                   -- user.id, api_key.id, or data_source.id
    data_source_id  UUID,                   -- which enrichment source produced this data
    lawful_basis    TEXT,                    -- GDPR lawful basis for this data operation
    -- Ordering
    sequence_num    BIGINT NOT NULL,         -- monotonically increasing per aggregate
    -- Timestamp
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Unique constraint ensures no duplicate events per aggregate
CREATE UNIQUE INDEX idx_events_aggregate_seq ON events(aggregate_type, aggregate_id, sequence_num);

-- Primary query patterns
CREATE INDEX idx_events_aggregate ON events(aggregate_type, aggregate_id, sequence_num);
CREATE INDEX idx_events_type ON events(event_type, occurred_at);
CREATE INDEX idx_events_tenant ON events(tenant_id, occurred_at);
CREATE INDEX idx_events_source ON events(data_source_id) WHERE data_source_id IS NOT NULL;

-- Partition by month for performance at scale
-- (In production, use declarative partitioning on occurred_at)

-- ============================================================
-- EVENT TYPE REGISTRY
-- ============================================================

CREATE TABLE event_types (
    event_type      TEXT PRIMARY KEY,
    description     TEXT NOT NULL,
    payload_schema  JSONB NOT NULL,          -- JSON Schema for the event payload
    version         INT NOT NULL DEFAULT 1,
    deprecated      BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Example event type registrations:
-- INSERT INTO event_types VALUES
--   ('person.created',        'A new person aggregate was created', '{"type":"object",...}', 1, false, now()),
--   ('person.email.added',    'An email address was added to a person', '{"type":"object",...}', 1, false, now()),
--   ('person.email.verified', 'An email was verified via SMTP probe', '{"type":"object",...}', 1, false, now()),
--   ('person.enriched',       'Person was enriched from a data source', '{"type":"object",...}', 1, false, now()),
--   ('person.job_changed',    'Job change detected for a person', '{"type":"object",...}', 1, false, now()),
--   ('person.gdpr.erased',    'Person data erased per GDPR request', '{"type":"object",...}', 1, false, now()),
--   ('company.created',       'A new company aggregate was created', '{"type":"object",...}', 1, false, now()),
--   ('company.enriched',      'Company was enriched from a data source', '{"type":"object",...}', 1, false, now());

-- ============================================================
-- SNAPSHOTS (performance optimisation for replay)
-- ============================================================

CREATE TABLE snapshots (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    aggregate_type  TEXT NOT NULL,
    aggregate_id    UUID NOT NULL,
    snapshot_data   JSONB NOT NULL,          -- full entity state at this point
    last_event_seq  BIGINT NOT NULL,         -- replay from here instead of from 0
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE UNIQUE INDEX idx_snapshots_aggregate ON snapshots(aggregate_type, aggregate_id);
```

### Example Event Payloads

```sql
-- person.created event
-- {
--   "full_name": "Sean Thorne",
--   "first_name": "Sean",
--   "last_name": "Thorne",
--   "linkedin_url": "https://linkedin.com/in/seanthorne"
-- }

-- person.email.added event
-- {
--   "email": "sean@example.com",
--   "email_type": "work",
--   "source": "pdl",
--   "confidence": 0.92
-- }

-- person.email.verified event
-- {
--   "email": "sean@example.com",
--   "verdict": "valid",
--   "syntax_valid": true,
--   "mx_found": true,
--   "mx_hostname": "alt1.aspmx.l.google.com",
--   "smtp_response_code": 250,
--   "is_catch_all": false,
--   "confidence": 0.98
-- }

-- person.enriched event (waterfall step result)
-- {
--   "source": "apollo",
--   "fields_enriched": ["job_title", "seniority", "department"],
--   "data": {
--     "job_title": "VP of Engineering",
--     "seniority": "vp",
--     "department": "Engineering"
--   },
--   "confidence": 0.85,
--   "request_id": "req_abc123",
--   "waterfall_step": 2,
--   "cost_usd": 0.47
-- }

-- person.job_changed event
-- {
--   "previous": {
--     "company_name": "Acme Corp",
--     "title": "Senior Engineer"
--   },
--   "current": {
--     "company_name": "NewCo Inc",
--     "title": "VP of Engineering"
--   },
--   "detection_source": "linkedin_signal",
--   "confidence": 0.91
-- }

-- person.gdpr.erased event
-- {
--   "request_id": "dsr_xyz789",
--   "request_type": "erasure",
--   "fields_erased": ["full_name", "emails", "phones", "job_experiences"],
--   "requester_email": "sean@example.com",
--   "completed_at": "2026-05-12T14:30:00Z"
-- }
```

## Tenant & Configuration Tables (Mutable)

```sql
-- ============================================================
-- TENANT & ACCESS (mutable operational data)
-- ============================================================

CREATE TABLE tenants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    plan            TEXT NOT NULL DEFAULT 'free',
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

-- ============================================================
-- DATA SOURCES (reference data for provenance)
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
    api_base_url    TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- WATERFALL CONFIGURATIONS
-- ============================================================

CREATE TABLE waterfall_configs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            TEXT NOT NULL,
    request_type    TEXT NOT NULL,
    source_order    JSONB NOT NULL,          -- ordered list of source configs
    -- Example source_order:
    -- [
    --   {"source_id": "...", "priority": 1, "conditions": {"country": "US"}},
    --   {"source_id": "...", "priority": 2, "max_cost_usd": 0.50}
    -- ]
    is_default      BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, name)
);
```

## Read Model Projections (Materialised from Events)

```sql
-- ============================================================
-- PERSON PROJECTION (current state, derived from events)
-- ============================================================

CREATE TABLE person_projection (
    id                  UUID PRIMARY KEY,       -- same as aggregate_id in events
    tenant_id           UUID NOT NULL,
    -- Current state (rebuilt from events)
    full_name           TEXT,
    first_name          TEXT,
    last_name           TEXT,
    job_title           TEXT,
    seniority           TEXT,
    department          TEXT,
    current_company_id  UUID,
    -- Location
    location_city       TEXT,
    location_region     TEXT,
    location_country    TEXT,
    -- Social
    linkedin_url        TEXT,
    github_url          TEXT,
    twitter_url         TEXT,
    -- Privacy flags
    do_not_contact      BOOLEAN NOT NULL DEFAULT FALSE,
    gdpr_opt_out        BOOLEAN NOT NULL DEFAULT FALSE,
    is_erased           BOOLEAN NOT NULL DEFAULT FALSE,
    -- Computed from events
    email_count         INT NOT NULL DEFAULT 0,
    phone_count         INT NOT NULL DEFAULT 0,
    enrichment_score    NUMERIC(3,2),
    total_enrichments   INT NOT NULL DEFAULT 0,
    sources_used        TEXT[] NOT NULL DEFAULT '{}',
    -- Event tracking
    last_event_seq      BIGINT NOT NULL,        -- last processed event sequence
    first_seen_at       TIMESTAMPTZ NOT NULL,
    last_enriched_at    TIMESTAMPTZ,
    last_updated_at     TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_person_proj_tenant ON person_projection(tenant_id);
CREATE INDEX idx_person_proj_name ON person_projection(tenant_id, last_name, first_name);
CREATE INDEX idx_person_proj_company ON person_projection(current_company_id);
CREATE INDEX idx_person_proj_linkedin ON person_projection(linkedin_url) WHERE linkedin_url IS NOT NULL;

-- ============================================================
-- PERSON EMAILS PROJECTION
-- ============================================================

CREATE TABLE person_email_projection (
    person_id       UUID NOT NULL,
    email           TEXT NOT NULL,
    email_type      TEXT NOT NULL DEFAULT 'work',
    is_primary      BOOLEAN NOT NULL DEFAULT FALSE,
    verification_status TEXT NOT NULL DEFAULT 'unverified',
    verified_at     TIMESTAMPTZ,
    confidence      NUMERIC(3,2),
    source_name     TEXT,
    added_at        TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (person_id, email)
);

CREATE INDEX idx_email_proj_email ON person_email_projection(email);

-- ============================================================
-- PERSON PHONES PROJECTION
-- ============================================================

CREATE TABLE person_phone_projection (
    person_id       UUID NOT NULL,
    phone           TEXT NOT NULL,
    phone_type      TEXT NOT NULL DEFAULT 'work_direct',
    is_primary      BOOLEAN NOT NULL DEFAULT FALSE,
    is_verified     BOOLEAN NOT NULL DEFAULT FALSE,
    confidence      NUMERIC(3,2),
    source_name     TEXT,
    added_at        TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (person_id, phone)
);

-- ============================================================
-- COMPANY PROJECTION
-- ============================================================

CREATE TABLE company_projection (
    id                  UUID PRIMARY KEY,
    tenant_id           UUID NOT NULL,
    name                TEXT NOT NULL,
    legal_name          TEXT,
    domain              TEXT,
    industry            TEXT,
    employee_count      INT,
    employee_range      TEXT,
    revenue_range       TEXT,
    founded_year        INT,
    company_type        TEXT,
    -- Location
    hq_city             TEXT,
    hq_region           TEXT,
    hq_country_code     TEXT,
    -- Identifiers
    ticker_symbol       TEXT,
    sec_cik             TEXT,
    lei                 TEXT,
    -- Social
    linkedin_url        TEXT,
    crunchbase_url      TEXT,
    -- Computed
    enrichment_score    NUMERIC(3,2),
    total_enrichments   INT NOT NULL DEFAULT 0,
    technology_count    INT NOT NULL DEFAULT 0,
    -- Event tracking
    last_event_seq      BIGINT NOT NULL,
    first_seen_at       TIMESTAMPTZ NOT NULL,
    last_enriched_at    TIMESTAMPTZ,
    last_updated_at     TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_company_proj_tenant ON company_projection(tenant_id);
CREATE INDEX idx_company_proj_domain ON company_projection(tenant_id, domain);

-- ============================================================
-- ENRICHMENT REQUEST PROJECTION
-- ============================================================

CREATE TABLE enrichment_request_projection (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    request_type    TEXT NOT NULL,
    status          TEXT NOT NULL,
    input_params    JSONB NOT NULL,
    matched_person_id UUID,
    matched_company_id UUID,
    match_confidence NUMERIC(3,2),
    -- Waterfall summary
    sources_queried TEXT[],
    sources_matched TEXT[],
    total_cost_usd  NUMERIC(10,6),
    duration_ms     INT,
    -- Timestamps
    requested_at    TIMESTAMPTZ NOT NULL,
    completed_at    TIMESTAMPTZ
);

CREATE INDEX idx_enrich_proj_tenant ON enrichment_request_projection(tenant_id, requested_at);

-- ============================================================
-- JOB CHANGE PROJECTION
-- ============================================================

CREATE TABLE job_change_projection (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    person_id       UUID NOT NULL,
    tenant_id       UUID NOT NULL,
    prev_company    TEXT,
    prev_title      TEXT,
    new_company     TEXT,
    new_title       TEXT,
    detected_at     TIMESTAMPTZ NOT NULL,
    confidence      NUMERIC(3,2),
    notification_sent BOOLEAN NOT NULL DEFAULT FALSE
);

CREATE INDEX idx_job_change_proj_tenant ON job_change_projection(tenant_id, detected_at);
CREATE INDEX idx_job_change_proj_unsent ON job_change_projection(tenant_id, notification_sent)
    WHERE notification_sent = FALSE;
```

## Projection Update Functions

```sql
-- ============================================================
-- EVENT HANDLER: Update person projection when events arrive
-- ============================================================

CREATE OR REPLACE FUNCTION project_person_event() RETURNS TRIGGER AS $$
BEGIN
    -- Only handle person aggregate events
    IF NEW.aggregate_type != 'person' THEN
        RETURN NEW;
    END IF;

    CASE NEW.event_type
        WHEN 'person.created' THEN
            INSERT INTO person_projection (
                id, tenant_id, full_name, first_name, last_name,
                linkedin_url, last_event_seq, first_seen_at, last_updated_at
            ) VALUES (
                NEW.aggregate_id,
                NEW.tenant_id,
                NEW.payload->>'full_name',
                NEW.payload->>'first_name',
                NEW.payload->>'last_name',
                NEW.payload->>'linkedin_url',
                NEW.sequence_num,
                NEW.occurred_at,
                NEW.occurred_at
            );

        WHEN 'person.enriched' THEN
            UPDATE person_projection SET
                job_title = COALESCE(NEW.payload->'data'->>'job_title', job_title),
                seniority = COALESCE(NEW.payload->'data'->>'seniority', seniority),
                department = COALESCE(NEW.payload->'data'->>'department', department),
                total_enrichments = total_enrichments + 1,
                sources_used = array_append(
                    sources_used,
                    NEW.payload->>'source'
                ),
                last_event_seq = NEW.sequence_num,
                last_enriched_at = NEW.occurred_at,
                last_updated_at = NEW.occurred_at
            WHERE id = NEW.aggregate_id;

        WHEN 'person.email.added' THEN
            INSERT INTO person_email_projection (
                person_id, email, email_type, confidence, source_name, added_at
            ) VALUES (
                NEW.aggregate_id,
                NEW.payload->>'email',
                COALESCE(NEW.payload->>'email_type', 'work'),
                (NEW.payload->>'confidence')::NUMERIC,
                NEW.payload->>'source',
                NEW.occurred_at
            ) ON CONFLICT (person_id, email) DO UPDATE SET
                confidence = GREATEST(
                    person_email_projection.confidence,
                    (NEW.payload->>'confidence')::NUMERIC
                );

            UPDATE person_projection SET
                email_count = email_count + 1,
                last_event_seq = NEW.sequence_num,
                last_updated_at = NEW.occurred_at
            WHERE id = NEW.aggregate_id;

        WHEN 'person.email.verified' THEN
            UPDATE person_email_projection SET
                verification_status = NEW.payload->>'verdict',
                verified_at = NEW.occurred_at,
                confidence = (NEW.payload->>'confidence')::NUMERIC
            WHERE person_id = NEW.aggregate_id
              AND email = NEW.payload->>'email';

        WHEN 'person.gdpr.erased' THEN
            UPDATE person_projection SET
                full_name = '[ERASED]',
                first_name = NULL,
                last_name = NULL,
                job_title = NULL,
                linkedin_url = NULL,
                is_erased = TRUE,
                last_event_seq = NEW.sequence_num,
                last_updated_at = NEW.occurred_at
            WHERE id = NEW.aggregate_id;

            DELETE FROM person_email_projection WHERE person_id = NEW.aggregate_id;
            DELETE FROM person_phone_projection WHERE person_id = NEW.aggregate_id;

        ELSE
            -- Unknown event type: log but don't fail
            NULL;
    END CASE;

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_project_person
    AFTER INSERT ON events
    FOR EACH ROW
    EXECUTE FUNCTION project_person_event();
```

## Temporal Query Examples

```sql
-- ============================================================
-- "What was this person's job title on January 1, 2026?"
-- ============================================================

SELECT payload->'data'->>'job_title' AS job_title,
       occurred_at
FROM events
WHERE aggregate_type = 'person'
  AND aggregate_id = '550e8400-e29b-41d4-a716-446655440000'
  AND event_type = 'person.enriched'
  AND payload->'data' ? 'job_title'
  AND occurred_at <= '2026-01-01T00:00:00Z'
ORDER BY occurred_at DESC
LIMIT 1;

-- ============================================================
-- "Show me the full enrichment history for this person"
-- ============================================================

SELECT event_type,
       occurred_at,
       payload->>'source' AS data_source,
       payload->'data' AS enriched_fields,
       lawful_basis
FROM events
WHERE aggregate_type = 'person'
  AND aggregate_id = '550e8400-e29b-41d4-a716-446655440000'
ORDER BY sequence_num ASC;

-- ============================================================
-- "Which data source provided this person's phone number?"
-- ============================================================

SELECT payload->>'phone' AS phone,
       payload->>'source' AS data_source,
       lawful_basis,
       occurred_at
FROM events
WHERE aggregate_type = 'person'
  AND aggregate_id = '550e8400-e29b-41d4-a716-446655440000'
  AND event_type = 'person.phone.added'
ORDER BY occurred_at DESC;

-- ============================================================
-- "How many enrichments per source in the last 30 days?"
-- ============================================================

SELECT payload->>'source' AS data_source,
       COUNT(*) AS enrichment_count,
       AVG((payload->>'confidence')::NUMERIC) AS avg_confidence,
       SUM((payload->>'cost_usd')::NUMERIC) AS total_cost
FROM events
WHERE event_type = 'person.enriched'
  AND occurred_at >= now() - INTERVAL '30 days'
GROUP BY payload->>'source'
ORDER BY enrichment_count DESC;

-- ============================================================
-- GDPR: "Generate a full data export for this person"
-- ============================================================

SELECT event_type,
       occurred_at,
       payload,
       (SELECT ds.display_name FROM data_sources ds WHERE ds.id = e.data_source_id) AS source,
       lawful_basis
FROM events e
WHERE aggregate_type = 'person'
  AND aggregate_id = '550e8400-e29b-41d4-a716-446655440000'
  AND event_type NOT LIKE '%.internal.%'
ORDER BY sequence_num ASC;
```

## CRM Integration (Event-Driven)

```sql
-- ============================================================
-- CRM SYNC OUTBOX (events to push to CRM)
-- ============================================================

CREATE TABLE crm_sync_outbox (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    crm_type        TEXT NOT NULL,
    -- What changed
    event_id        UUID NOT NULL REFERENCES events(id),
    entity_type     TEXT NOT NULL,
    entity_id       UUID NOT NULL,
    -- Sync status
    status          TEXT NOT NULL DEFAULT 'pending' CHECK (status IN (
        'pending', 'syncing', 'synced', 'failed', 'skipped'
    )),
    attempts        INT NOT NULL DEFAULT 0,
    last_error      TEXT,
    -- Timestamps
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    synced_at       TIMESTAMPTZ
);

CREATE INDEX idx_crm_outbox_pending ON crm_sync_outbox(status, created_at)
    WHERE status = 'pending';

-- CRM connections table (same as model 1)
CREATE TABLE crm_connections (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    crm_type        TEXT NOT NULL,
    access_token_enc BYTEA,
    refresh_token_enc BYTEA,
    token_expires_at TIMESTAMPTZ,
    instance_url    TEXT,
    sync_enabled    BOOLEAN NOT NULL DEFAULT TRUE,
    field_mappings  JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 3 | events, event_types, snapshots |
| Tenant & Config | 4 | tenants, api_keys, data_sources, waterfall_configs |
| Person Projections | 3 | person_projection, person_email_projection, person_phone_projection |
| Company Projections | 1 | company_projection |
| Request Projections | 1 | enrichment_request_projection |
| Job Change Projections | 1 | job_change_projection |
| CRM Integration | 2 | crm_connections, crm_sync_outbox |
| **Total** | **15** | Plus 1 trigger function |

---

## Key Design Decisions

1. **Single `events` table as the source of truth.** Every state change -- person creation, email addition, enrichment from a source, GDPR erasure -- is an immutable event. The events table is append-only; no UPDATEs or DELETEs are ever executed against it (GDPR erasure records an erasure event; the person data in the projection is cleared, but the event log retains the fact that erasure occurred).

2. **JSONB payloads with registered schemas.** Each event type has a JSON Schema definition in `event_types`, enabling validation at insert time. The JSONB payload allows different event types to carry different data without requiring DDL changes -- adding a new enrichment field means adding it to the event payload, not running ALTER TABLE.

3. **Synchronous projections via PostgreSQL triggers.** The `project_person_event()` trigger updates read models in the same transaction as the event insert. This provides strong consistency (read model is never behind) at the cost of write performance. For higher throughput, this can be switched to async processing via LISTEN/NOTIFY or a background worker.

4. **Snapshots for performance.** Entities with hundreds of enrichment events would be slow to rebuild from scratch. The `snapshots` table stores periodic full-state snapshots; replay starts from the most recent snapshot rather than from event #1.

5. **Temporal queries are first-class.** Reconstructing historical state is a simple query: filter events by `occurred_at <= target_date` and replay. No separate "history" table or temporal extension needed.

6. **CRM sync via outbox pattern.** Rather than pushing to CRM synchronously, enrichment events create entries in `crm_sync_outbox`. A background worker processes the outbox, handling retries and failures without blocking the enrichment pipeline.

7. **GDPR erasure preserves the audit trail.** When a person exercises their right to erasure, a `person.gdpr.erased` event is recorded. The projection is cleared, but the event store retains the metadata (that erasure occurred, when, by whom) without retaining the personal data -- the erasure event's payload lists which fields were erased, not their values.

8. **Event-driven job change detection.** When a `person.enriched` event arrives with a different `job_title` or `company` than the current projection, the system automatically emits a `person.job_changed` event. This is a natural pattern in event-sourced systems and eliminates the need for separate change-detection logic.

9. **CloudEvents-aligned envelope.** The event structure follows the CloudEvents 1.0 specification (source, type, subject, time, data), making it compatible with cloud-native event routing systems if the architecture evolves to use Kafka, NATS, or Google Pub/Sub.

10. **Lower table count than normalized model.** With 15 tables vs. 25, the event-sourced model achieves the same functionality with less schema complexity. The trade-off is that complexity shifts from DDL to application logic (event handlers, projection rebuilders).
