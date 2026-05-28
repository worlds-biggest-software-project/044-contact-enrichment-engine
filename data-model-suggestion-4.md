# Data Model Suggestion 4: Graph-Relational Hybrid

> Project: Contact Enrichment Engine · Created: 2026-05-12

## Philosophy

The Graph-Relational Hybrid model combines a relational backbone for operational CRUD with a property graph layer for relationship-heavy queries. Contact enrichment is fundamentally a relationship problem: people work at companies, companies own domains, people know other people, companies belong to industries, enrichment sources link to contacts through data provenance chains, and job changes create temporal relationships between people and companies. A graph model makes these relationships first-class queryable entities rather than implicit foreign keys.

This architecture uses PostgreSQL's `ltree` extension for hierarchical data (company org charts, industry taxonomies) and a lightweight graph-edge table for arbitrary relationships (person-works-at-company, person-knows-person, company-owns-domain, contact-enriched-by-source). The relational tables store entity properties; the graph layer stores typed, weighted, temporal edges between them. This pattern is used by LinkedIn's Economic Graph, Palantir's knowledge graph, and modern identity resolution platforms that need to traverse relationship chains.

The graph layer enables queries that are impractical in a purely relational model: "Find all people at companies in the same industry as Company X who changed jobs in the last 6 months" or "Show me the chain of data sources that contributed to this contact record." These traversal queries use recursive CTEs against the edge table rather than requiring a separate graph database (Neo4j, Neptune).

**Best for:** Teams building an enrichment platform that emphasises relationship intelligence, identity resolution across multiple sources, and network-based prospecting -- where understanding connections between people and companies is as valuable as the enrichment data itself.

**Trade-offs:**
- (+) Relationship queries (paths, neighborhoods, clusters) are natural and efficient
- (+) Identity resolution: merge/split logic via edges rather than destructive record merging
- (+) Temporal edges enable "relationship at point in time" queries
- (+) Flexible: new relationship types require no DDL changes (just a new edge_type)
- (+) Supports conflict-of-interest analysis, org chart traversal, and network-based prospecting
- (-) More complex query patterns: recursive CTEs and edge-table joins
- (-) Graph traversal performance degrades with very deep paths (>5 hops)
- (-) Edge table grows proportionally to relationships, not entities -- can be very large
- (-) Developers need to think in graph terms (nodes, edges, traversals)
- (-) Less mature tooling ecosystem than pure relational or pure graph databases

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| RFC 5322 (Email Format) | Email validation on person nodes |
| RFC 5321 (SMTP) | Verification results stored on email sub-nodes |
| RFC 6350 (vCard 4.0) | Person node properties map to vCard fields |
| Schema.org Person/Organization | Entity types and property names align with Schema.org |
| ISO 3166-1/2 (Country/Region) | Location properties on person and company nodes use ISO 3166 |
| ISO 17442 (LEI) | Company identifier edges link to LEI records |
| GDPR Articles 6, 14, 21 | Data provenance edges carry lawful_basis and source attribution |
| W3C PROV-O | Provenance edges follow W3C PROV-O ontology (entity, activity, agent) |
| SKOS (Simple Knowledge Organization) | Industry and seniority taxonomies modeled as SKOS concept schemes |
| OpenAPI 3.1 | API responses serialize graph traversal results as nested JSON |

---

## Entity (Node) Tables

```sql
-- ============================================================
-- GRAPH NODE REGISTRY
-- All entities in the system are registered as nodes.
-- Properties are stored on the domain-specific tables.
-- ============================================================

CREATE TABLE graph_nodes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    node_type       TEXT NOT NULL,           -- 'person', 'company', 'email', 'phone', 'data_source', 'domain'
    tenant_id       UUID NOT NULL,
    label           TEXT,                    -- human-readable label for graph visualisation
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_graph_nodes_type ON graph_nodes(node_type);
CREATE INDEX idx_graph_nodes_tenant ON graph_nodes(tenant_id);

-- ============================================================
-- PERSONS (node properties)
-- ============================================================

CREATE TABLE persons (
    node_id             UUID PRIMARY KEY REFERENCES graph_nodes(id),
    tenant_id           UUID NOT NULL,
    -- Identity
    full_name           TEXT,
    first_name          TEXT,
    last_name           TEXT,
    -- Current role (denormalised from WORKS_AT edge)
    job_title           TEXT,
    seniority           TEXT CHECK (seniority IN (
        'entry', 'individual_contributor', 'manager', 'director',
        'vp', 'c_level', 'partner', 'owner', 'intern'
    )),
    department          TEXT,
    -- Location
    location_city       TEXT,
    location_region     TEXT,
    location_country    TEXT,               -- ISO 3166-1 alpha-2
    -- Social profiles
    linkedin_url        TEXT,
    github_url          TEXT,
    twitter_url         TEXT,
    -- Privacy
    do_not_contact      BOOLEAN NOT NULL DEFAULT FALSE,
    gdpr_opt_out        BOOLEAN NOT NULL DEFAULT FALSE,
    -- Quality
    enrichment_score    NUMERIC(3,2),
    last_enriched_at    TIMESTAMPTZ,
    -- Properties that vary by source or region
    extended_properties JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "skills": ["Python", "PostgreSQL"],
    --   "languages": ["English", "Spanish"],
    --   "bio": "VP of Engineering with 15 years experience..."
    -- }
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_persons_tenant ON persons(tenant_id);
CREATE INDEX idx_persons_name ON persons(tenant_id, last_name, first_name);
CREATE INDEX idx_persons_linkedin ON persons(linkedin_url) WHERE linkedin_url IS NOT NULL;

-- ============================================================
-- COMPANIES (node properties)
-- ============================================================

CREATE TABLE companies (
    node_id             UUID PRIMARY KEY REFERENCES graph_nodes(id),
    tenant_id           UUID NOT NULL,
    -- Identity
    name                TEXT NOT NULL,
    legal_name          TEXT,
    domain              TEXT,
    -- Firmographics
    industry            TEXT,
    sic_code            TEXT,
    naics_code          TEXT,
    employee_count      INT,
    employee_range      TEXT,
    revenue_range       TEXT,
    founded_year        INT,
    company_type        TEXT CHECK (company_type IN (
        'public', 'private', 'nonprofit', 'government', 'education', 'self_employed'
    )),
    -- Location (HQ)
    hq_city             TEXT,
    hq_region           TEXT,
    hq_country_code     TEXT,               -- ISO 3166-1 alpha-2
    -- Identifiers
    ticker_symbol       TEXT,
    sec_cik             TEXT,
    lei                 TEXT,               -- ISO 17442
    duns_number         TEXT,
    -- Social
    linkedin_url        TEXT,
    crunchbase_url      TEXT,
    -- Extended
    extended_properties JSONB NOT NULL DEFAULT '{}',
    -- Quality
    enrichment_score    NUMERIC(3,2),
    last_enriched_at    TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_companies_tenant ON companies(tenant_id);
CREATE INDEX idx_companies_domain ON companies(tenant_id, domain);

-- ============================================================
-- EMAILS (node)
-- ============================================================

CREATE TABLE email_nodes (
    node_id             UUID PRIMARY KEY REFERENCES graph_nodes(id),
    address             TEXT NOT NULL,       -- RFC 5322 validated
    domain              TEXT NOT NULL,       -- extracted from address
    email_type          TEXT NOT NULL DEFAULT 'work',
    -- Verification
    verification_status TEXT NOT NULL DEFAULT 'unverified',
    verification_data   JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "syntax_valid": true,
    --   "mx_found": true,
    --   "mx_hostname": "alt1.aspmx.l.google.com",
    --   "smtp_code": 250,
    --   "is_catch_all": false,
    --   "is_disposable": false,
    --   "is_role_based": false,
    --   "verified_at": "2026-05-12T10:00:00Z"
    -- }
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_email_nodes_address ON email_nodes(address);
CREATE INDEX idx_email_nodes_domain ON email_nodes(domain);

-- ============================================================
-- PHONE NODES
-- ============================================================

CREATE TABLE phone_nodes (
    node_id             UUID PRIMARY KEY REFERENCES graph_nodes(id),
    number              TEXT NOT NULL,       -- E.164 format
    phone_type          TEXT NOT NULL DEFAULT 'work_direct',
    is_verified         BOOLEAN NOT NULL DEFAULT FALSE,
    verified_at         TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_phone_nodes_number ON phone_nodes(number);

-- ============================================================
-- DOMAIN NODES
-- ============================================================

CREATE TABLE domain_nodes (
    node_id             UUID PRIMARY KEY REFERENCES graph_nodes(id),
    domain              TEXT NOT NULL UNIQUE,
    mx_provider         TEXT,
    rdap_data           JSONB NOT NULL DEFAULT '{}',
    -- Example rdap_data:
    -- {
    --   "registrant_org": "Acme Corporation",
    --   "registration_date": "2005-03-15",
    --   "expiry_date": "2027-03-15",
    --   "queried_at": "2026-05-01"
    -- }
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_domain_nodes_domain ON domain_nodes(domain);

-- ============================================================
-- DATA SOURCE NODES
-- ============================================================

CREATE TABLE data_source_nodes (
    node_id             UUID PRIMARY KEY REFERENCES graph_nodes(id),
    name                TEXT NOT NULL UNIQUE,
    display_name        TEXT NOT NULL,
    source_type         TEXT NOT NULL,
    lawful_basis        TEXT NOT NULL,
    priority            INT NOT NULL DEFAULT 100,
    cost_per_lookup     NUMERIC(10,6),
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    config              JSONB NOT NULL DEFAULT '{}',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Graph Edge Table

```sql
-- ============================================================
-- GRAPH EDGES (typed, weighted, temporal relationships)
-- ============================================================

CREATE TABLE graph_edges (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    -- Source and target nodes
    source_node_id  UUID NOT NULL REFERENCES graph_nodes(id),
    target_node_id  UUID NOT NULL REFERENCES graph_nodes(id),
    -- Edge metadata
    edge_type       TEXT NOT NULL,
    -- Example edge_types:
    --   'WORKS_AT'          person → company (with title, seniority in properties)
    --   'WORKED_AT'         person → company (historical employment)
    --   'HAS_EMAIL'         person → email
    --   'HAS_PHONE'         person → phone
    --   'OWNS_DOMAIN'       company → domain
    --   'SUBSIDIARY_OF'     company → company
    --   'ENRICHED_BY'       person/company → data_source (provenance)
    --   'KNOWS'             person → person (co-workers, shared company)
    --   'INVESTED_IN'       company → company (funding relationship)
    --   'STUDIED_AT'        person → education institution
    --   'USES_TECHNOLOGY'   company → technology (implied node)
    --   'JOB_CHANGED_FROM'  person → company (job change source)
    --   'JOB_CHANGED_TO'    person → company (job change target)

    -- Edge properties
    properties      JSONB NOT NULL DEFAULT '{}',
    -- Example (WORKS_AT):
    -- {
    --   "title": "VP of Engineering",
    --   "seniority": "vp",
    --   "department": "Engineering",
    --   "start_date": "2024-03-01"
    -- }
    -- Example (ENRICHED_BY):
    -- {
    --   "fields_provided": ["job_title", "emails"],
    --   "confidence": 0.88,
    --   "lawful_basis": "legitimate_interest",
    --   "cost_usd": 0.004,
    --   "request_id": "req_abc123"
    -- }
    -- Example (KNOWS):
    -- {
    --   "relationship": "former_colleague",
    --   "shared_company": "Acme Corp",
    --   "overlap_period": "2020-2024"
    -- }

    -- Temporal properties
    weight          NUMERIC(5,3) NOT NULL DEFAULT 1.0,  -- relationship strength
    confidence      NUMERIC(3,2),
    valid_from      TIMESTAMPTZ,            -- when this relationship started
    valid_to        TIMESTAMPTZ,            -- when this relationship ended (NULL = current)
    is_current      BOOLEAN NOT NULL DEFAULT TRUE,
    -- Metadata
    tenant_id       UUID NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Primary traversal indexes
CREATE INDEX idx_edges_source ON graph_edges(source_node_id, edge_type);
CREATE INDEX idx_edges_target ON graph_edges(target_node_id, edge_type);
CREATE INDEX idx_edges_type ON graph_edges(edge_type);
CREATE INDEX idx_edges_tenant ON graph_edges(tenant_id);
CREATE INDEX idx_edges_current ON graph_edges(source_node_id, edge_type, is_current)
    WHERE is_current = TRUE;
CREATE INDEX idx_edges_temporal ON graph_edges(valid_from, valid_to)
    WHERE valid_to IS NULL;

-- Prevent duplicate current edges of the same type between same nodes
CREATE UNIQUE INDEX idx_edges_unique_current
    ON graph_edges(source_node_id, target_node_id, edge_type)
    WHERE is_current = TRUE AND edge_type NOT IN ('ENRICHED_BY');
```

## Tenant & Pipeline Tables

```sql
-- ============================================================
-- TENANTS
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

-- ============================================================
-- API KEYS
-- ============================================================

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
-- WATERFALL CONFIGURATIONS
-- ============================================================

CREATE TABLE waterfall_configs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            TEXT NOT NULL,
    request_type    TEXT NOT NULL,
    steps           JSONB NOT NULL,
    is_default      BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, name)
);

-- ============================================================
-- ENRICHMENT REQUESTS
-- ============================================================

CREATE TABLE enrichment_requests (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    api_key_id      UUID REFERENCES api_keys(id),
    request_type    TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'pending',
    input_params    JSONB NOT NULL,
    -- Graph result references
    matched_node_id UUID REFERENCES graph_nodes(id),
    match_confidence NUMERIC(3,2),
    -- Waterfall log
    waterfall_log   JSONB NOT NULL DEFAULT '[]',
    total_cost_usd  NUMERIC(10,6),
    duration_ms     INT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ
);

CREATE INDEX idx_enrich_req_tenant ON enrichment_requests(tenant_id, created_at);

-- ============================================================
-- CRM CONNECTIONS
-- ============================================================

CREATE TABLE crm_connections (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    crm_type        TEXT NOT NULL,
    credentials_enc BYTEA NOT NULL,
    instance_url    TEXT,
    config          JSONB NOT NULL DEFAULT '{}',
    last_synced_at  TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- GDPR / COMPLIANCE
-- ============================================================

CREATE TABLE data_subject_requests (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    person_node_id  UUID REFERENCES graph_nodes(id),
    request_type    TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'received',
    requester_email TEXT NOT NULL,
    details         JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Industry Taxonomy (Using ltree)

```sql
-- ============================================================
-- INDUSTRY TAXONOMY (hierarchical classification)
-- ============================================================

CREATE EXTENSION IF NOT EXISTS ltree;

CREATE TABLE industry_taxonomy (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            TEXT NOT NULL UNIQUE,    -- e.g., 'technology.software.saas'
    path            ltree NOT NULL,          -- e.g., 'technology.software.saas'
    label           TEXT NOT NULL,           -- e.g., 'SaaS'
    description     TEXT,
    sic_codes       TEXT[],                  -- mapped SIC codes
    naics_codes     TEXT[],                  -- mapped NAICS codes
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_taxonomy_path ON industry_taxonomy USING GIST (path);

-- Example data:
-- INSERT INTO industry_taxonomy (code, path, label) VALUES
--   ('technology', 'technology', 'Technology'),
--   ('technology.software', 'technology.software', 'Software'),
--   ('technology.software.saas', 'technology.software.saas', 'SaaS'),
--   ('technology.software.infrastructure', 'technology.software.infrastructure', 'Infrastructure'),
--   ('finance', 'finance', 'Finance'),
--   ('finance.banking', 'finance.banking', 'Banking'),
--   ('finance.fintech', 'finance.fintech', 'FinTech');

-- Query: Find all companies in any software sub-industry
-- SELECT c.* FROM companies c
-- JOIN industry_taxonomy t ON c.industry = t.code
-- WHERE t.path <@ 'technology.software';
```

## Graph Traversal Queries

```sql
-- ============================================================
-- "Find all people who work at the same company as person X"
-- ============================================================

SELECT p.full_name, p.job_title, e_colleague.properties->>'title' AS their_title
FROM graph_edges e_target
-- Find which company person X works at
JOIN graph_edges e_colleague ON e_colleague.target_node_id = e_target.target_node_id
JOIN persons p ON p.node_id = e_colleague.source_node_id
WHERE e_target.source_node_id = 'person-x-node-id'
  AND e_target.edge_type = 'WORKS_AT'
  AND e_target.is_current = TRUE
  AND e_colleague.edge_type = 'WORKS_AT'
  AND e_colleague.is_current = TRUE
  AND e_colleague.source_node_id != 'person-x-node-id';

-- ============================================================
-- "Show the complete data provenance chain for this person"
-- ============================================================

SELECT ds.display_name AS data_source,
       e.properties->>'fields_provided' AS fields_provided,
       e.properties->>'confidence' AS confidence,
       e.properties->>'lawful_basis' AS lawful_basis,
       e.created_at AS enriched_at
FROM graph_edges e
JOIN data_source_nodes ds ON ds.node_id = e.target_node_id
WHERE e.source_node_id = 'person-node-id'
  AND e.edge_type = 'ENRICHED_BY'
ORDER BY e.created_at DESC;

-- ============================================================
-- "Find people who changed jobs in the last 6 months"
-- (Traverse JOB_CHANGED_FROM → JOB_CHANGED_TO edge pairs)
-- ============================================================

SELECT p.full_name,
       e_from.properties->>'title' AS previous_title,
       c_from.name AS previous_company,
       e_to.properties->>'title' AS new_title,
       c_to.name AS new_company,
       e_to.created_at AS change_detected_at
FROM graph_edges e_to
JOIN graph_edges e_from ON e_from.source_node_id = e_to.source_node_id
    AND e_from.edge_type = 'JOB_CHANGED_FROM'
    AND e_from.properties->>'change_group' = e_to.properties->>'change_group'
JOIN persons p ON p.node_id = e_to.source_node_id
JOIN companies c_from ON c_from.node_id = e_from.target_node_id
JOIN companies c_to ON c_to.node_id = e_to.target_node_id
WHERE e_to.edge_type = 'JOB_CHANGED_TO'
  AND e_to.created_at >= now() - INTERVAL '6 months';

-- ============================================================
-- "Find the shortest path between two people" (via shared companies)
-- Uses recursive CTE for breadth-first traversal
-- ============================================================

WITH RECURSIVE path_search AS (
    -- Start from person A
    SELECT
        e.source_node_id AS current_node,
        e.target_node_id AS next_node,
        ARRAY[e.source_node_id, e.target_node_id] AS path,
        1 AS depth,
        e.edge_type
    FROM graph_edges e
    WHERE e.source_node_id = 'person-a-node-id'
      AND e.edge_type IN ('WORKS_AT', 'WORKED_AT', 'KNOWS')
      AND e.tenant_id = 'tenant-uuid'

    UNION ALL

    -- Traverse outward
    SELECT
        ps.next_node,
        e.target_node_id,
        ps.path || e.target_node_id,
        ps.depth + 1,
        e.edge_type
    FROM path_search ps
    JOIN graph_edges e ON e.source_node_id = ps.next_node
    WHERE e.target_node_id != ALL(ps.path)   -- no cycles
      AND ps.depth < 4                       -- max 4 hops
      AND e.edge_type IN ('WORKS_AT', 'WORKED_AT', 'KNOWS')
)
SELECT path, depth
FROM path_search
WHERE next_node = 'person-b-node-id'
ORDER BY depth ASC
LIMIT 1;

-- ============================================================
-- "What was this person's employment history?"
-- (All WORKS_AT and WORKED_AT edges, ordered temporally)
-- ============================================================

SELECT c.name AS company,
       e.properties->>'title' AS title,
       e.properties->>'seniority' AS seniority,
       e.valid_from AS started,
       e.valid_to AS ended,
       e.is_current
FROM graph_edges e
JOIN companies c ON c.node_id = e.target_node_id
WHERE e.source_node_id = 'person-node-id'
  AND e.edge_type IN ('WORKS_AT', 'WORKED_AT')
ORDER BY e.valid_from DESC NULLS FIRST;

-- ============================================================
-- "Find companies connected to Company X" (subsidiaries, investors)
-- ============================================================

WITH RECURSIVE company_network AS (
    SELECT
        e.target_node_id AS company_node_id,
        e.edge_type AS relationship,
        1 AS depth
    FROM graph_edges e
    WHERE e.source_node_id = 'company-x-node-id'
      AND e.edge_type IN ('SUBSIDIARY_OF', 'INVESTED_IN', 'OWNS_DOMAIN')

    UNION ALL

    SELECT
        e.target_node_id,
        e.edge_type,
        cn.depth + 1
    FROM company_network cn
    JOIN graph_edges e ON e.source_node_id = cn.company_node_id
    WHERE cn.depth < 3
      AND e.edge_type IN ('SUBSIDIARY_OF', 'INVESTED_IN')
)
SELECT c.name, cn.relationship, cn.depth
FROM company_network cn
JOIN companies c ON c.node_id = cn.company_node_id;
```

## Identity Resolution via Graph

```sql
-- ============================================================
-- IDENTITY RESOLUTION: merge candidates via shared edges
-- When two person nodes share emails, phones, or LinkedIn URLs,
-- they may be the same person. The graph makes this queryable.
-- ============================================================

-- Find potential duplicate persons (share an email or phone)
SELECT p1.node_id AS person_a,
       p1.full_name AS name_a,
       p2.node_id AS person_b,
       p2.full_name AS name_b,
       shared.node_type AS shared_via,
       CASE shared.node_type
           WHEN 'email' THEN (SELECT address FROM email_nodes WHERE node_id = shared.id)
           WHEN 'phone' THEN (SELECT number FROM phone_nodes WHERE node_id = shared.id)
       END AS shared_value
FROM graph_edges e1
JOIN graph_edges e2 ON e2.target_node_id = e1.target_node_id
    AND e2.source_node_id != e1.source_node_id
    AND e2.edge_type = e1.edge_type
JOIN graph_nodes shared ON shared.id = e1.target_node_id
JOIN persons p1 ON p1.node_id = e1.source_node_id
JOIN persons p2 ON p2.node_id = e2.source_node_id
WHERE e1.edge_type IN ('HAS_EMAIL', 'HAS_PHONE')
  AND e1.tenant_id = 'tenant-uuid'
  AND p1.node_id < p2.node_id;  -- avoid duplicating pairs

-- Record a merge decision as an edge (non-destructive)
-- INSERT INTO graph_edges (source_node_id, target_node_id, edge_type, properties, tenant_id)
-- VALUES ('person-duplicate-id', 'person-canonical-id', 'MERGED_INTO', '{"merged_by":"user_abc","reason":"shared_email"}', 'tenant-uuid');
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Graph Infrastructure | 2 | graph_nodes, graph_edges |
| Person Nodes | 1 | persons (properties on graph node) |
| Company Nodes | 1 | companies (properties on graph node) |
| Contact Nodes | 2 | email_nodes, phone_nodes |
| Domain Nodes | 1 | domain_nodes |
| Data Source Nodes | 1 | data_source_nodes |
| Taxonomy | 1 | industry_taxonomy (ltree) |
| Tenant & Access | 2 | tenants, api_keys |
| Enrichment Pipeline | 2 | waterfall_configs, enrichment_requests |
| CRM Integration | 1 | crm_connections |
| GDPR Compliance | 1 | data_subject_requests |
| **Total** | **15** | Plus ltree extension |

---

## Key Design Decisions

1. **Dual-table entity pattern (graph_nodes + property table).** Every entity has a row in `graph_nodes` (for uniform graph traversal) and a row in its property table (persons, companies, etc.) for typed, indexed attributes. This avoids the "property soup" problem where a single key-value table stores everything, while retaining uniform graph traversal.

2. **Edges are temporal by default.** Every edge has `valid_from` and `valid_to` timestamps. When a person changes jobs, their `WORKS_AT` edge gets `valid_to` set and `is_current` set to FALSE; a new `WORKS_AT` edge is created. This preserves full employment history without separate history tables.

3. **Data provenance as `ENRICHED_BY` edges.** Instead of a provenance table or JSONB field, each enrichment creates an edge from the enriched entity to the data source. The edge properties carry which fields were provided, confidence scores, cost, and lawful basis. This makes provenance queryable via graph traversal.

4. **Identity resolution via shared edges, not destructive merging.** When two person nodes share an email or phone, they appear as merge candidates via graph queries. The `MERGED_INTO` edge type records merge decisions without deleting the duplicate -- allowing merges to be reversed if they were incorrect.

5. **Industry taxonomy with ltree.** The `ltree` extension enables hierarchical industry classification with efficient ancestor/descendant queries (`@>`, `<@` operators). Companies can be classified at any level of granularity (e.g., "technology.software.saas") and queried at any level ("find all technology companies").

6. **Recursive CTEs for path queries.** PostgreSQL's `WITH RECURSIVE` enables multi-hop traversal (shortest path, network analysis) without a separate graph database. Performance is acceptable for paths up to 4-5 hops; deeper traversals would benefit from Neo4j or Apache AGE.

7. **Contact data (emails, phones) as separate graph nodes rather than properties.** This enables multiple persons to share the same email node (useful for identity resolution), and edges from email nodes to domain nodes (which email provider does this person use?). The trade-off is more joins for simple enrichment lookups.

8. **Edge types are schemaless.** New relationship types (e.g., `ATTENDED_CONFERENCE`, `SPOKE_AT`, `AUTHORED_PATENT`) can be added by inserting edges with new `edge_type` values -- no DDL changes required. This makes the model extensible for future enrichment sources that provide new types of intelligence.

9. **Unique constraint on current edges** prevents duplicate active relationships (a person can only `WORKS_AT` one company at a time). The `ENRICHED_BY` edge type is excluded from this constraint because a person can be enriched by the same source multiple times.

10. **Graph visualisation-ready.** The `label` field on `graph_nodes` and the structured `properties` on edges are designed to feed directly into graph visualisation libraries (D3.js, Cytoscape.js, vis.js) without transformation. A front-end can render the contact's relationship network by querying nodes and edges within 2 hops.
