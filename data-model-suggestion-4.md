# Data Model Suggestion 4: Graph-Relational Hybrid

> Project: IP Management Platform · Created: 2026-05-12

## Philosophy

The Graph-Relational Hybrid model combines a property graph layer for relationship-heavy queries with relational tables for operational CRUD. IP management is inherently a graph problem: patents cite other patents (citation networks), patent families span jurisdictions (hierarchical trees), inventors collaborate across organisations (social networks), assignees form ownership chains through acquisitions and transfers, and FTO analysis requires traversing claim-to-claim relationships across entire technology landscapes. These relationship patterns are poorly served by JOIN-heavy relational queries but are the natural domain of graph databases.

This model uses PostgreSQL as the operational database for CRUD operations (filing patents, tracking deadlines, managing parties) while maintaining a graph layer in either (a) PostgreSQL using `graph_nodes` and `graph_edges` tables with recursive CTEs, or (b) a dedicated graph database like Neo4j for production deployments requiring complex traversals. The relational tables handle day-to-day IP management; the graph layer handles analytics, FTO traversals, citation network analysis, inventor network discovery, and portfolio relationship mapping.

Real-world precedent exists for this approach: IP Street (now part of Clarivate) uses Neo4j for inventor network analysis and patent citation mapping. PatSnap's competitive intelligence features involve graph traversals across patent citation networks. The open-source research community has published extensively on patent citation knowledge graphs built on Neo4j. For an open-source IPMS that aims to compete on analytics (portfolio optimisation, FTO analysis, competitive intelligence), a graph layer is a significant differentiator.

**Best for:** Organisations that need advanced patent analytics -- citation network analysis, inventor collaboration mapping, FTO claim traversal, portfolio relationship visualisation, and AI-powered competitive intelligence.

**Trade-offs:**
- PRO: Natural representation of patent citation networks, family trees, ownership chains, and inventor collaboration
- PRO: FTO analysis as graph traversal: "find all paths from this product's features to potentially blocking patent claims"
- PRO: Portfolio optimisation via graph centrality: identify patents that are citation hubs vs. peripheral
- PRO: Inventor network analysis for R&D strategy and talent acquisition
- PRO: Relationship discovery that is impractical with relational JOINs (e.g., "show all patents within 3 citation hops of this competitor's key patent")
- CON: Dual-database architecture increases operational complexity (sync between relational and graph)
- CON: Graph query languages (Cypher, GQL, recursive CTEs) require specialised knowledge
- CON: PostgreSQL-only graph implementation with recursive CTEs has performance limits for large traversals
- CON: Neo4j (if used) adds a second database to operate, back up, and scale
- CON: Graph data must be kept in sync with relational data -- eventual consistency risk

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| WIPO ST.96 | Patent data ingested via ST.96; citation relationships extracted and loaded into graph edges |
| ISO 3166-1/2 | Jurisdiction nodes in the graph use ISO 3166 alpha-2 codes as node properties |
| NICE Classification | NICE class nodes enable graph queries: "show all trademarks in class 9 and their owners" |
| Paris Convention | Priority claim relationships modeled as graph edges between patent nodes across jurisdictions |
| PCT | PCT family relationships as graph edges connecting the international application to national phase entries |
| CPC / IPC Classification | Classification nodes enable technology landscape graph queries |
| PatentsView / EPO OPS | External patent data imported as nodes with citation edges from USPTO/EPO citation data |
| ISO 27001 | Audit trail maintained in relational audit_log; graph access controlled via application-layer RBAC |

---

## Relational Layer (Operational CRUD)

```sql
-- ============================================================
-- CORE RELATIONAL TABLES
-- These handle day-to-day CRUD operations.
-- The graph layer (below) handles analytics and traversals.
-- ============================================================

CREATE TABLE organisations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    org_type        TEXT NOT NULL CHECK (org_type IN ('corporation', 'law_firm', 'university', 'government', 'individual')),
    country_code    CHAR(2),
    settings        JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    email           TEXT NOT NULL,
    display_name    TEXT NOT NULL,
    role            TEXT NOT NULL CHECK (role IN ('admin', 'ip_counsel', 'paralegal', 'inventor', 'viewer')),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, email)
);

CREATE TABLE jurisdictions (
    country_code    CHAR(2) PRIMARY KEY,
    country_name    TEXT NOT NULL,
    is_pct_member   BOOLEAN NOT NULL DEFAULT false,
    is_paris_member BOOLEAN NOT NULL DEFAULT false,
    is_madrid_member BOOLEAN NOT NULL DEFAULT false,
    pct_national_phase_months INTEGER DEFAULT 30,
    patent_term_years INTEGER DEFAULT 20,
    currency_code   CHAR(3)
);

-- ============================================================
-- PATENTS (operational table)
-- ============================================================

CREATE TABLE patents (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    patent_number       TEXT,
    application_number  TEXT NOT NULL,
    title               TEXT NOT NULL,
    abstract            TEXT,
    filing_date         DATE NOT NULL,
    grant_date          DATE,
    expiry_date         DATE,
    priority_date       DATE,
    jurisdiction_code   CHAR(2) NOT NULL REFERENCES jurisdictions(country_code),
    status              TEXT NOT NULL,
    patent_type         TEXT NOT NULL,
    family_id           UUID,
    pct_application_number TEXT,
    cost_to_date        NUMERIC(12,2) DEFAULT 0,
    -- Graph node reference
    graph_node_id       UUID,              -- links to graph_nodes table
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_patents_org ON patents(organisation_id);
CREATE INDEX idx_patents_jurisdiction ON patents(jurisdiction_code);
CREATE INDEX idx_patents_status ON patents(status);
CREATE INDEX idx_patents_family ON patents(family_id);
CREATE INDEX idx_patents_graph ON patents(graph_node_id);

-- ============================================================
-- TRADEMARKS (operational table)
-- ============================================================

CREATE TABLE trademarks (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    registration_number TEXT,
    application_number  TEXT NOT NULL,
    mark_text           TEXT,
    mark_type           TEXT NOT NULL,
    filing_date         DATE NOT NULL,
    registration_date   DATE,
    expiry_date         DATE,
    jurisdiction_code   CHAR(2) NOT NULL REFERENCES jurisdictions(country_code),
    status              TEXT NOT NULL,
    nice_classes        SMALLINT[],
    graph_node_id       UUID,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_trademarks_org ON trademarks(organisation_id);
CREATE INDEX idx_trademarks_jurisdiction ON trademarks(jurisdiction_code);

-- ============================================================
-- PARTIES (operational table)
-- ============================================================

CREATE TABLE parties (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    party_type      TEXT NOT NULL CHECK (party_type IN (
        'inventor', 'assignee', 'applicant', 'outside_counsel',
        'licensee', 'licensor', 'agent', 'examiner', 'competitor'
    )),
    is_person       BOOLEAN NOT NULL DEFAULT true,
    display_name    TEXT NOT NULL,
    email           TEXT,
    country_code    CHAR(2) REFERENCES jurisdictions(country_code),
    details         JSONB DEFAULT '{}',
    graph_node_id   UUID,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_parties_org ON parties(organisation_id);
CREATE INDEX idx_parties_type ON parties(party_type);
CREATE INDEX idx_parties_graph ON parties(graph_node_id);

-- ============================================================
-- DEADLINES (operational -- not in graph)
-- ============================================================

CREATE TABLE deadlines (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    patent_id           UUID REFERENCES patents(id),
    trademark_id        UUID REFERENCES trademarks(id),
    deadline_type       TEXT NOT NULL,
    due_date            DATE NOT NULL,
    grace_period_end    DATE,
    jurisdiction_code   CHAR(2) REFERENCES jurisdictions(country_code),
    status              TEXT NOT NULL CHECK (status IN (
        'upcoming', 'due_soon', 'overdue', 'completed', 'waived', 'missed'
    )),
    assigned_to         UUID REFERENCES users(id),
    completed_at        TIMESTAMPTZ,
    completed_by        UUID REFERENCES users(id),
    fee_amount          NUMERIC(10,2),
    fee_currency        CHAR(3),
    annuity_decision    JSONB,
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_deadlines_org ON deadlines(organisation_id);
CREATE INDEX idx_deadlines_due ON deadlines(due_date);
CREATE INDEX idx_deadlines_status ON deadlines(status);

-- ============================================================
-- DOCUMENTS (operational)
-- ============================================================

CREATE TABLE documents (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    patent_id       UUID REFERENCES patents(id),
    trademark_id    UUID REFERENCES trademarks(id),
    document_type   TEXT NOT NULL,
    title           TEXT NOT NULL,
    file_name       TEXT NOT NULL,
    storage_path    TEXT NOT NULL,
    source          TEXT,
    extracted_data  JSONB,
    uploaded_by     UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_documents_patent ON documents(patent_id);
CREATE INDEX idx_documents_trademark ON documents(trademark_id);

-- ============================================================
-- AUDIT LOG
-- ============================================================

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL,
    user_id         UUID,
    action          TEXT NOT NULL,
    entity_type     TEXT NOT NULL,
    entity_id       UUID,
    changes         JSONB,
    ip_address      INET,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_org ON audit_log(organisation_id);
CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_created ON audit_log(created_at);
```

## Graph Layer (PostgreSQL Implementation)

```sql
-- ============================================================
-- PROPERTY GRAPH -- generic node/edge tables
-- Enables graph traversals within PostgreSQL using recursive CTEs.
-- For production deployments with large citation networks (>1M nodes),
-- consider migrating to Neo4j or Apache AGE (PostgreSQL graph extension).
-- ============================================================

CREATE TABLE graph_nodes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID,                  -- NULL for external/public patent nodes
    node_type       TEXT NOT NULL CHECK (node_type IN (
        'patent', 'trademark', 'party', 'organisation',
        'jurisdiction', 'cpc_class', 'ipc_class', 'nice_class',
        'technology_area', 'product', 'claim',
        'external_patent'                  -- patents not in our system (from USPTO/EPO data)
    )),
    -- Reference to the relational entity (one of these will be populated)
    ref_patent_id       UUID REFERENCES patents(id),
    ref_trademark_id    UUID REFERENCES trademarks(id),
    ref_party_id        UUID REFERENCES parties(id),
    -- For external patents not in our system
    external_identifier TEXT,              -- e.g., "US10123456B2", "EP3456789A1"
    -- Node properties (semi-structured)
    label           TEXT NOT NULL,          -- display label for visualisation
    properties      JSONB NOT NULL DEFAULT '{}',
    -- properties examples:
    -- Patent node: {"filing_date": "2026-03-15", "status": "granted", "jurisdiction": "US"}
    -- Party node: {"type": "inventor", "country": "US", "h_index": 12}
    -- CPC node: {"code": "H03M13/00", "description": "Coding, decoding or code conversion"}
    -- External patent: {"title": "...", "assignee": "CompetitorCorp", "grant_date": "2024-06-01"}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_graph_nodes_org ON graph_nodes(organisation_id);
CREATE INDEX idx_graph_nodes_type ON graph_nodes(node_type);
CREATE INDEX idx_graph_nodes_ref_patent ON graph_nodes(ref_patent_id) WHERE ref_patent_id IS NOT NULL;
CREATE INDEX idx_graph_nodes_ref_party ON graph_nodes(ref_party_id) WHERE ref_party_id IS NOT NULL;
CREATE INDEX idx_graph_nodes_external ON graph_nodes(external_identifier) WHERE external_identifier IS NOT NULL;
CREATE INDEX idx_graph_nodes_properties ON graph_nodes USING GIN (properties jsonb_path_ops);

CREATE TABLE graph_edges (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_node_id  UUID NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    target_node_id  UUID NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    edge_type       TEXT NOT NULL CHECK (edge_type IN (
        -- Citation relationships
        'cites',                           -- patent A cites patent B
        'cited_by',                        -- inverse of cites (materialised for fast traversal)

        -- Family relationships
        'family_member',                   -- patents in the same family
        'continuation_of',                 -- continuation, CIP, divisional
        'priority_claim',                  -- priority claim relationship

        -- Party relationships
        'invented_by',                     -- patent -> inventor
        'assigned_to',                     -- patent -> assignee/owner
        'represented_by',                  -- patent -> attorney/agent
        'co_invented_with',               -- inventor -> inventor (inferred)
        'employed_by',                     -- person -> organisation

        -- Classification
        'classified_as',                   -- patent -> CPC/IPC class
        'subclass_of',                     -- CPC class hierarchy

        -- Licensing & FTO
        'licensed_to',                     -- patent -> licensee party
        'potentially_blocked_by',          -- product -> patent (FTO risk)
        'claim_overlaps_with',             -- claim -> claim (FTO detail)

        -- Competitive
        'competes_with',                   -- organisation -> organisation
        'operates_in_field',              -- organisation -> technology area

        -- Trademark
        'registered_in',                   -- trademark -> jurisdiction
        'owned_by'                         -- trademark -> party
    )),
    -- Edge properties
    properties      JSONB NOT NULL DEFAULT '{}',
    -- properties examples:
    -- cites: {"citation_type": "examiner", "category": "X", "relevant_claims": [1,3]}
    -- invented_by: {"contribution_pct": 60, "is_current": true}
    -- priority_claim: {"priority_date": "2025-03-16", "deadline": "2026-03-16"}
    -- potentially_blocked_by: {"risk_level": "high", "confidence": 0.89, "overlapping_claims": [1,4]}
    -- co_invented_with: {"shared_patents": 5, "collaboration_start": "2020"}
    weight          NUMERIC(5,2) DEFAULT 1.0,  -- for weighted graph algorithms
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_graph_edges_source ON graph_edges(source_node_id);
CREATE INDEX idx_graph_edges_target ON graph_edges(target_node_id);
CREATE INDEX idx_graph_edges_type ON graph_edges(edge_type);
CREATE INDEX idx_graph_edges_source_type ON graph_edges(source_node_id, edge_type);
CREATE INDEX idx_graph_edges_target_type ON graph_edges(target_node_id, edge_type);
CREATE INDEX idx_graph_edges_properties ON graph_edges USING GIN (properties jsonb_path_ops);

-- ============================================================
-- GRAPH ANALYTICS CACHE
-- Pre-computed graph metrics, refreshed periodically
-- ============================================================

CREATE TABLE graph_analytics (
    node_id         UUID NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    metric_type     TEXT NOT NULL CHECK (metric_type IN (
        'degree_centrality',               -- how many connections
        'betweenness_centrality',          -- how often on shortest paths
        'pagerank',                        -- importance in citation network
        'clustering_coefficient',          -- local clustering
        'citation_depth',                  -- max depth in citation chain
        'h_index',                         -- for inventor nodes
        'family_size',                     -- number of family members
        'forward_citation_count',          -- times cited by others
        'backward_citation_count'          -- number of references made
    )),
    value           NUMERIC(12,6) NOT NULL,
    computed_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (node_id, metric_type)
);

CREATE INDEX idx_graph_analytics_metric ON graph_analytics(metric_type, value DESC);
```

## Graph Query Examples (PostgreSQL Recursive CTEs)

```sql
-- ============================================================
-- QUERY 1: Patent Citation Chain (up to 3 hops)
-- "Show all patents within 3 citation hops of patent X"
-- ============================================================

WITH RECURSIVE citation_chain AS (
    -- Base case: start from the target patent
    SELECT
        gn.id AS node_id,
        gn.label,
        gn.external_identifier,
        gn.properties,
        0 AS depth,
        ARRAY[gn.id] AS path
    FROM graph_nodes gn
    WHERE gn.ref_patent_id = '550e8400-e29b-41d4-a716-446655440000'

    UNION ALL

    -- Recursive case: follow citation edges
    SELECT
        gn2.id,
        gn2.label,
        gn2.external_identifier,
        gn2.properties,
        cc.depth + 1,
        cc.path || gn2.id
    FROM citation_chain cc
    JOIN graph_edges ge ON ge.source_node_id = cc.node_id
    JOIN graph_nodes gn2 ON ge.target_node_id = gn2.id
    WHERE ge.edge_type = 'cites'
      AND cc.depth < 3                    -- max 3 hops
      AND NOT gn2.id = ANY(cc.path)       -- prevent cycles
)
SELECT
    node_id,
    label,
    external_identifier,
    depth,
    properties->>'status' AS status,
    properties->>'jurisdiction' AS jurisdiction,
    array_length(path, 1) AS path_length
FROM citation_chain
ORDER BY depth, label;


-- ============================================================
-- QUERY 2: Inventor Collaboration Network
-- "Find all inventors who have co-invented with inventor X"
-- ============================================================

WITH inventor_patents AS (
    -- All patents by the target inventor
    SELECT ge.source_node_id AS patent_node_id
    FROM graph_edges ge
    WHERE ge.target_node_id = (
        SELECT id FROM graph_nodes WHERE ref_party_id = 'inventor-uuid'
    )
    AND ge.edge_type = 'invented_by'
),
co_inventors AS (
    -- All other inventors on those patents
    SELECT DISTINCT
        gn.id AS inventor_node_id,
        gn.label AS inventor_name,
        gn.properties,
        COUNT(*) AS shared_patents
    FROM inventor_patents ip
    JOIN graph_edges ge ON ge.source_node_id = ip.patent_node_id
    JOIN graph_nodes gn ON ge.target_node_id = gn.id
    WHERE ge.edge_type = 'invented_by'
      AND gn.ref_party_id != 'inventor-uuid'
    GROUP BY gn.id, gn.label, gn.properties
)
SELECT
    inventor_name,
    shared_patents,
    properties->>'country' AS country,
    (properties->>'h_index')::int AS h_index
FROM co_inventors
ORDER BY shared_patents DESC;


-- ============================================================
-- QUERY 3: FTO Risk Traversal
-- "Find all patents that potentially block product features"
-- ============================================================

SELECT
    gn_product.label AS product_feature,
    gn_patent.label AS blocking_patent,
    gn_patent.external_identifier AS patent_number,
    ge.properties->>'risk_level' AS risk_level,
    (ge.properties->>'confidence')::numeric AS confidence,
    ge.properties->'overlapping_claims' AS overlapping_claims,
    gn_assignee.label AS patent_owner
FROM graph_nodes gn_product
JOIN graph_edges ge ON ge.source_node_id = gn_product.id
    AND ge.edge_type = 'potentially_blocked_by'
JOIN graph_nodes gn_patent ON ge.target_node_id = gn_patent.id
-- Get the assignee of the blocking patent
LEFT JOIN graph_edges ge_owner ON ge_owner.source_node_id = gn_patent.id
    AND ge_owner.edge_type = 'assigned_to'
LEFT JOIN graph_nodes gn_assignee ON ge_owner.target_node_id = gn_assignee.id
WHERE gn_product.node_type = 'product'
  AND gn_product.organisation_id = 'org-uuid'
ORDER BY
    CASE ge.properties->>'risk_level'
        WHEN 'high' THEN 1
        WHEN 'medium' THEN 2
        WHEN 'low' THEN 3
    END,
    (ge.properties->>'confidence')::numeric DESC;


-- ============================================================
-- QUERY 4: Patent Family Tree
-- "Show the complete family tree for this patent"
-- ============================================================

WITH RECURSIVE family_tree AS (
    SELECT
        gn.id AS node_id,
        gn.label,
        gn.properties,
        ge.edge_type AS relationship,
        0 AS depth,
        ARRAY[gn.id] AS path
    FROM graph_nodes gn
    WHERE gn.ref_patent_id = 'patent-uuid'

    UNION ALL

    SELECT
        gn2.id,
        gn2.label,
        gn2.properties,
        ge.edge_type,
        ft.depth + 1,
        ft.path || gn2.id
    FROM family_tree ft
    JOIN graph_edges ge ON (
        ge.source_node_id = ft.node_id OR ge.target_node_id = ft.node_id
    )
    JOIN graph_nodes gn2 ON (
        CASE
            WHEN ge.source_node_id = ft.node_id THEN ge.target_node_id
            ELSE ge.source_node_id
        END = gn2.id
    )
    WHERE ge.edge_type IN ('family_member', 'continuation_of', 'priority_claim')
      AND NOT gn2.id = ANY(ft.path)
      AND ft.depth < 10
)
SELECT
    label AS patent_title,
    properties->>'jurisdiction' AS jurisdiction,
    properties->>'status' AS status,
    properties->>'filing_date' AS filing_date,
    relationship,
    depth
FROM family_tree
ORDER BY depth, properties->>'filing_date';


-- ============================================================
-- QUERY 5: Portfolio Centrality Analysis
-- "Which patents are the most important in our citation network?"
-- ============================================================

SELECT
    gn.label AS patent_title,
    p.patent_number,
    p.jurisdiction_code,
    ga_pr.value AS pagerank,
    ga_bc.value AS betweenness_centrality,
    ga_fc.value AS forward_citations,
    p.cost_to_date
FROM graph_nodes gn
JOIN patents p ON gn.ref_patent_id = p.id
LEFT JOIN graph_analytics ga_pr ON gn.id = ga_pr.node_id AND ga_pr.metric_type = 'pagerank'
LEFT JOIN graph_analytics ga_bc ON gn.id = ga_bc.node_id AND ga_bc.metric_type = 'betweenness_centrality'
LEFT JOIN graph_analytics ga_fc ON gn.id = ga_fc.node_id AND ga_fc.metric_type = 'forward_citation_count'
WHERE p.organisation_id = 'org-uuid'
  AND p.status = 'granted'
ORDER BY ga_pr.value DESC NULLS LAST
LIMIT 20;
```

## Neo4j Schema (Alternative Graph Layer)

For production deployments with large patent citation networks (1M+ nodes), Neo4j provides better graph traversal performance than PostgreSQL recursive CTEs. The following Cypher schema mirrors the PostgreSQL graph layer.

```cypher
// ============================================================
// NEO4J NODE DEFINITIONS (for production graph deployments)
// ============================================================

// Patent nodes (both internal and external)
CREATE CONSTRAINT patent_id IF NOT EXISTS FOR (p:Patent) REQUIRE p.uuid IS UNIQUE;
CREATE CONSTRAINT ext_patent_id IF NOT EXISTS FOR (p:ExternalPatent) REQUIRE p.patent_number IS UNIQUE;

// Party nodes
CREATE CONSTRAINT party_id IF NOT EXISTS FOR (p:Party) REQUIRE p.uuid IS UNIQUE;

// Classification nodes
CREATE CONSTRAINT cpc_code IF NOT EXISTS FOR (c:CPCClass) REQUIRE c.code IS UNIQUE;
CREATE CONSTRAINT ipc_code IF NOT EXISTS FOR (c:IPCClass) REQUIRE c.code IS UNIQUE;
CREATE CONSTRAINT nice_class IF NOT EXISTS FOR (c:NICEClass) REQUIRE c.number IS UNIQUE;

// Jurisdiction nodes
CREATE CONSTRAINT jurisdiction_code IF NOT EXISTS FOR (j:Jurisdiction) REQUIRE j.code IS UNIQUE;

// Full-text indexes for search
CREATE FULLTEXT INDEX patent_search IF NOT EXISTS FOR (p:Patent) ON EACH [p.title, p.abstract];

// ============================================================
// EXAMPLE: Load a patent with relationships
// ============================================================

// Create patent node
CREATE (p:Patent {
    uuid: 'patent-uuid',
    patent_number: 'US11234567B2',
    title: 'Method for Quantum Error Correction',
    filing_date: date('2026-03-15'),
    grant_date: date('2027-01-10'),
    status: 'granted',
    jurisdiction: 'US',
    organisation_id: 'org-uuid'
});

// Citation relationship
MATCH (citing:Patent {uuid: 'patent-uuid'})
MATCH (cited:ExternalPatent {patent_number: 'US10123456B2'})
CREATE (citing)-[:CITES {
    citation_type: 'examiner',
    category: 'X',
    relevant_claims: [1, 3]
}]->(cited);

// Inventor relationship
MATCH (p:Patent {uuid: 'patent-uuid'})
MATCH (inv:Party {uuid: 'inventor-uuid'})
CREATE (p)-[:INVENTED_BY {contribution_pct: 60.0}]->(inv);

// ============================================================
// EXAMPLE: FTO risk query in Cypher
// ============================================================

MATCH (product:Product {organisation_id: 'org-uuid'})
      -[:POTENTIALLY_BLOCKED_BY]->(patent)
      -[:ASSIGNED_TO]->(owner:Party)
WHERE product.name = 'QuantumShield v2'
RETURN product.name AS product,
       patent.title AS blocking_patent,
       patent.patent_number AS patent_number,
       owner.name AS patent_owner
ORDER BY patent.risk_level DESC;

// ============================================================
// EXAMPLE: Citation network analysis in Cypher
// ============================================================

// Find all patents within 3 citation hops
MATCH path = (start:Patent {uuid: 'patent-uuid'})-[:CITES*1..3]->(cited)
RETURN cited.patent_number, cited.title,
       length(path) AS hops,
       cited.jurisdiction
ORDER BY hops, cited.title;

// PageRank on citation network
CALL gds.pageRank.stream('citation-graph', {
    relationshipTypes: ['CITES'],
    maxIterations: 20,
    dampingFactor: 0.85
})
YIELD nodeId, score
MATCH (p:Patent) WHERE id(p) = nodeId AND p.organisation_id = 'org-uuid'
RETURN p.title, p.patent_number, score
ORDER BY score DESC
LIMIT 20;
```

## Graph Sync Service

```sql
-- ============================================================
-- GRAPH SYNC TRACKING
-- Keeps the graph layer in sync with relational changes
-- ============================================================

CREATE TABLE graph_sync_queue (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    entity_type     TEXT NOT NULL,         -- 'patent', 'trademark', 'party', 'citation'
    entity_id       UUID NOT NULL,
    operation       TEXT NOT NULL CHECK (operation IN ('create', 'update', 'delete')),
    status          TEXT NOT NULL CHECK (status IN ('pending', 'processing', 'completed', 'failed')),
    payload         JSONB,                 -- the data to sync to the graph
    error_message   TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    processed_at    TIMESTAMPTZ
);

CREATE INDEX idx_graph_sync_status ON graph_sync_queue(status, created_at);

-- Trigger function to enqueue graph sync on patent changes
CREATE OR REPLACE FUNCTION enqueue_graph_sync()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO graph_sync_queue (entity_type, entity_id, operation, payload)
    VALUES (TG_TABLE_NAME, NEW.id, TG_OP, to_jsonb(NEW));
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER patent_graph_sync
    AFTER INSERT OR UPDATE ON patents
    FOR EACH ROW EXECUTE FUNCTION enqueue_graph_sync();

CREATE TRIGGER party_graph_sync
    AFTER INSERT OR UPDATE ON parties
    FOR EACH ROW EXECUTE FUNCTION enqueue_graph_sync();

CREATE TRIGGER trademark_graph_sync
    AFTER INSERT OR UPDATE ON trademarks
    FOR EACH ROW EXECUTE FUNCTION enqueue_graph_sync();
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity | 2 | organisations, users |
| Reference Data | 1 | jurisdictions |
| Patents | 1 | patents (with graph_node_id reference) |
| Trademarks | 1 | trademarks (with graph_node_id reference) |
| Parties | 1 | parties (with graph_node_id reference) |
| Deadlines | 1 | deadlines |
| Documents | 1 | documents |
| Audit | 1 | audit_log |
| Graph Layer | 3 | graph_nodes, graph_edges, graph_analytics |
| Graph Sync | 1 | graph_sync_queue |
| **Total (PostgreSQL)** | **13** | Plus Neo4j if used for production graph |

---

## Key Design Decisions

1. **Dual-layer architecture**: relational tables for CRUD operations, graph tables for analytics and traversals. This avoids forcing operational workflows (deadline management, docketing) through a graph query language while enabling graph-native analytics (citation networks, FTO traversals) that are impractical with relational JOINs alone.

2. **Generic `graph_nodes` / `graph_edges` schema** rather than typed graph tables (e.g., separate `citation_edges`, `family_edges`). The generic approach supports any relationship type via the `edge_type` discriminator and JSONB properties, making it easy to add new relationship types (e.g., a new "competes_with" edge for competitive intelligence) without schema changes.

3. **External patent nodes** (`node_type = 'external_patent'`) enable citation network analysis that extends beyond the organisation's own portfolio. When USPTO/EPO citation data is imported, cited patents that are not in the organisation's portfolio are created as external nodes. This is essential for FTO analysis and competitive intelligence, where the relevant patents are primarily owned by other organisations.

4. **Pre-computed graph analytics cache** (`graph_analytics` table) stores PageRank, betweenness centrality, and other metrics that are expensive to compute in real-time. These are refreshed periodically (e.g., nightly) by a background job. This enables instant "most important patents" and "most connected inventors" queries without running graph algorithms on every request.

5. **Bidirectional citation edges** (`cites` and `cited_by`) are materialised rather than computed. While this doubles the edge count for citations, it enables fast traversal in both directions without reverse-edge lookups. Forward citations (who cites this patent) are as important as backward citations (what does this patent cite) for portfolio scoring.

6. **Graph sync via trigger-based queue** rather than synchronous dual-writes. When a patent is created or updated in the relational layer, a trigger enqueues a sync job. A background worker processes the queue and updates the graph layer. This provides eventual consistency with clear visibility into sync status.

7. **`graph_node_id` on relational tables** provides a direct link from operational records to their graph representation, enabling hybrid queries that join relational and graph data (e.g., "show me the top-PageRank patents with their operational deadline status").

8. **Weight column on edges** supports weighted graph algorithms. Citation edges can be weighted by relevance category (X citations weighted higher than A citations), enabling more nuanced centrality and path analysis.

9. **Neo4j as an optional production upgrade**. The PostgreSQL graph layer (recursive CTEs) works for portfolios up to ~100K nodes. For organisations with larger citation networks (importing full USPTO/EPO citation data for competitive intelligence), Neo4j provides native graph storage with constant-time edge traversal via index-free adjacency. The Cypher schema mirrors the PostgreSQL graph layer for straightforward migration.

10. **FTO as graph traversal** -- the `potentially_blocked_by` edge type between product/feature nodes and patent/claim nodes transforms FTO analysis from an ad-hoc search into a structured graph query. As the AI FTO module identifies risk relationships, it creates edges that can be traversed, visualised, and monitored for changes when cited patents change legal status.
