# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: IP Management Platform · Created: 2026-05-12

## Philosophy

The Event-Sourced / Audit-First model treats the append-only event store as the single source of truth. Every state change to a patent, trademark, deadline, or party is recorded as an immutable domain event. Current state is derived by replaying events (or maintained in materialised read models via CQRS projections). This architecture is purpose-built for the IP management domain where the history of what happened and when is as important as the current state.

In IP prosecution, a missed deadline can cause irreversible loss of patent rights, and regulatory environments (ISO 27001, NIST SP 800-92, trade secret regulations) demand comprehensive audit trails. Traditional CRUD databases with an afterthought audit log table can lose context -- which field changed, who authorised it, and what the system state was at the time. Event sourcing makes the audit trail the primary data structure rather than a secondary artifact. Every annuity decision, deadline completion, correspondence receipt, and status change is a first-class event with full context.

The CQRS (Command Query Responsibility Segregation) pattern separates the write path (events appended to the event store) from the read path (materialised views optimised for specific queries like "all upcoming deadlines" or "patent family status across jurisdictions"). This allows the read models to be denormalized and fast without compromising the integrity of the write-side event log. The pattern is adopted by 57% of organisations building event-driven architectures and is particularly well-suited for domains requiring legal-grade auditability.

**Best for:** Organisations requiring full temporal audit trails, bi-temporal querying ("what was the status of this patent on date X as known on date Y"), AI-powered change analytics, and regulatory compliance environments.

**Trade-offs:**
- PRO: Complete, immutable audit trail -- every change is preserved with full context, meeting ISO 27001 and NIST SP 800-92 requirements natively
- PRO: Bi-temporal querying: reconstruct the state of any IP asset at any point in time
- PRO: Event replay enables AI analytics on change patterns (e.g., "which patents had the most status changes before abandonment?")
- PRO: New read models can be added without modifying the write path -- just replay events into a new projection
- PRO: Natural fit for patent office correspondence processing: each incoming document becomes an event
- CON: Higher complexity -- developers must understand event sourcing, CQRS, and projection maintenance
- CON: Eventual consistency between event store and read models requires careful handling
- CON: Event schema evolution (versioning events over time) adds migration complexity
- CON: Read model rebuilds from large event stores can be slow without snapshots
- CON: Debugging current state requires understanding the event sequence, not just reading a row

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| WIPO ST.96 | Incoming patent office data (ST.96 XML) is ingested as `PatentOfficeCorrespondenceReceived` events; the ST.96 structure is preserved in the event payload |
| ISO 27001 / NIST SP 800-92 | The event store IS the audit trail -- no separate log needed; events are immutable, timestamped, and attributable to users |
| ISO 3166-1/2 | Jurisdiction codes in event payloads use ISO 3166 alpha-2 codes |
| NICE Classification | Trademark class assignments recorded as `TrademarkClassAssigned` events referencing NCL class numbers |
| Paris Convention | Priority claim events include computed 12-month deadline |
| PCT | National phase entry events track jurisdiction-specific deadlines (30/31/34 months) |
| Madrid Protocol | Madrid designation and renewal events |
| OAuth 2.0 / OIDC | User identity in event metadata for attribution |
| GDPR | Right-of-erasure handled via crypto-shredding: personal data encrypted with per-party keys stored separately; key deletion makes personal data in events unreadable without altering the event log |

---

## Event Store (Write Side)

```sql
-- ============================================================
-- EVENT STORE -- the single source of truth
-- ============================================================

CREATE TABLE event_store (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id       UUID NOT NULL,                 -- aggregate root ID (patent, trademark, etc.)
    stream_type     TEXT NOT NULL,                  -- 'Patent', 'Trademark', 'Deadline', 'Party', etc.
    event_type      TEXT NOT NULL,                  -- e.g., 'PatentFiled', 'DeadlineCompleted'
    event_version   INTEGER NOT NULL,               -- version within the stream (optimistic concurrency)
    organisation_id UUID NOT NULL,
    payload         JSONB NOT NULL,                 -- event-specific data
    metadata        JSONB NOT NULL DEFAULT '{}',    -- user_id, ip_address, correlation_id, causation_id
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_id, event_version)               -- optimistic concurrency control
);

-- Primary query pattern: replay events for a single aggregate
CREATE INDEX idx_events_stream ON event_store(stream_id, event_version);

-- Organisation-scoped queries
CREATE INDEX idx_events_org ON event_store(organisation_id, created_at);

-- Event type queries (for projections that process specific event types)
CREATE INDEX idx_events_type ON event_store(event_type, created_at);

-- Time-range queries for audit
CREATE INDEX idx_events_created ON event_store(created_at);

-- Partition by month for performance at scale
-- CREATE TABLE event_store (...) PARTITION BY RANGE (created_at);
```

## Event Type Catalogue

```sql
-- ============================================================
-- EVENT TYPE REGISTRY (documentation & versioning)
-- ============================================================

CREATE TABLE event_type_registry (
    event_type      TEXT PRIMARY KEY,
    stream_type     TEXT NOT NULL,
    description     TEXT NOT NULL,
    schema_version  INTEGER NOT NULL DEFAULT 1,
    payload_schema  JSONB NOT NULL,        -- JSON Schema for validating event payloads
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Example event types and their payload structures:

-- PatentFiled:
-- {
--   "application_number": "US17/123456",
--   "title": "Method for Quantum Error Correction",
--   "filing_date": "2026-03-15",
--   "jurisdiction_code": "US",
--   "patent_type": "utility",
--   "inventors": [{"party_id": "uuid", "name": "Jane Smith"}],
--   "assignee_party_id": "uuid",
--   "priority_claims": [{"country": "GB", "app_number": "GB2025123", "date": "2025-03-16"}]
-- }

-- PatentStatusChanged:
-- {
--   "old_status": "pending",
--   "new_status": "under_examination",
--   "reason": "First office action issued",
--   "effective_date": "2026-05-10"
-- }

-- DeadlineCreated:
-- {
--   "deadline_type": "office_action_response",
--   "due_date": "2026-08-10",
--   "grace_period_end": "2026-11-10",
--   "jurisdiction_code": "US",
--   "fee_amount": 0,
--   "assigned_to_user_id": "uuid"
-- }

-- DeadlineCompleted:
-- {
--   "completed_by_user_id": "uuid",
--   "completed_at": "2026-07-25T14:30:00Z",
--   "action_taken": "Response filed with USPTO"
-- }

-- AnnuityDecisionMade:
-- {
--   "decision": "pay",
--   "decided_by_user_id": "uuid",
--   "ai_recommendation": "pay",
--   "ai_confidence": 0.87,
--   "citation_score": 7.5,
--   "forward_citation_count": 23,
--   "business_alignment_score": 8.2,
--   "licensing_potential_score": 6.1,
--   "cost_if_paid": 1520.00,
--   "currency": "USD",
--   "reasoning": "High forward citation count and active licensing interest"
-- }

-- PatentOfficeCorrespondenceReceived:
-- {
--   "document_type": "office_action",
--   "source_office": "uspto",
--   "received_date": "2026-05-01",
--   "file_name": "OA_20260501.pdf",
--   "storage_path": "s3://docs/patent/uuid/OA_20260501.pdf",
--   "st96_data": { ... },
--   "ai_extracted_deadlines": [
--     {"type": "office_action_response", "due_date": "2026-08-01"}
--   ]
-- }

-- PriorArtSearchCompleted:
-- {
--   "search_type": "fto",
--   "query_text": "quantum error correction surface codes",
--   "data_sources": ["uspto", "epo_ops"],
--   "result_count": 47,
--   "overall_risk_score": 0.65,
--   "high_risk_patents": ["US10123456", "EP3456789"]
-- }

-- TrademarkFiled:
-- {
--   "application_number": "97/123456",
--   "mark_text": "QUANTUM SHIELD",
--   "mark_type": "word",
--   "filing_date": "2026-04-01",
--   "jurisdiction_code": "US",
--   "nice_classes": [9, 42],
--   "goods_services": {"9": "Computer software...", "42": "SaaS services..."}
-- }

-- InventionDisclosed:
-- {
--   "disclosure_number": "INV-2026-042",
--   "title": "Novel Approach to ...",
--   "inventors": [{"party_id": "uuid", "contribution_pct": 60.0}],
--   "technology_area": "quantum_computing"
-- }

-- LicenceAgreementExecuted:
-- {
--   "agreement_number": "LIC-2026-015",
--   "licensee_party_id": "uuid",
--   "licence_type": "non_exclusive",
--   "field_of_use": "Consumer electronics",
--   "territory": "US, EU",
--   "effective_date": "2026-06-01",
--   "royalty_type": "percentage",
--   "royalty_rate": 3.5,
--   "patent_ids": ["uuid1", "uuid2"]
-- }
```

## Snapshots (Performance Optimisation)

```sql
-- ============================================================
-- SNAPSHOTS -- periodic state captures for fast replay
-- ============================================================

CREATE TABLE snapshots (
    stream_id       UUID NOT NULL,
    stream_type     TEXT NOT NULL,
    snapshot_version INTEGER NOT NULL,      -- event_version at snapshot time
    state           JSONB NOT NULL,         -- full aggregate state at this version
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_id, snapshot_version)
);

-- To rebuild current state: load latest snapshot + replay events after snapshot_version
-- Snapshots should be created every N events (e.g., every 100 events per stream)
```

## Read Models (Query Side -- CQRS Projections)

```sql
-- ============================================================
-- READ MODEL: Patent Portfolio View
-- Materialised from: PatentFiled, PatentStatusChanged,
--   PatentClassificationAssigned, DeadlineCreated, etc.
-- ============================================================

CREATE TABLE rm_patents (
    id                  UUID PRIMARY KEY,
    organisation_id     UUID NOT NULL,
    patent_number       TEXT,
    application_number  TEXT NOT NULL,
    title               TEXT NOT NULL,
    abstract            TEXT,
    filing_date         DATE NOT NULL,
    grant_date          DATE,
    expiry_date         DATE,
    priority_date       DATE,
    jurisdiction_code   CHAR(2) NOT NULL,
    status              TEXT NOT NULL,
    patent_type         TEXT NOT NULL,
    family_id           UUID,
    pct_application_number TEXT,
    cost_to_date        NUMERIC(12,2) DEFAULT 0,
    -- Denormalized for fast reads:
    inventor_names      TEXT[],
    assignee_name       TEXT,
    primary_cpc_code    TEXT,
    primary_ipc_code    TEXT,
    next_deadline_date  DATE,
    next_deadline_type  TEXT,
    forward_citation_count INTEGER DEFAULT 0,
    ai_portfolio_score  NUMERIC(5,2),
    last_event_version  INTEGER NOT NULL,   -- tracks projection position
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_patents_org ON rm_patents(organisation_id);
CREATE INDEX idx_rm_patents_status ON rm_patents(status);
CREATE INDEX idx_rm_patents_jurisdiction ON rm_patents(jurisdiction_code);
CREATE INDEX idx_rm_patents_next_deadline ON rm_patents(next_deadline_date);
CREATE INDEX idx_rm_patents_family ON rm_patents(family_id);

-- ============================================================
-- READ MODEL: Deadline Dashboard
-- Materialised from: DeadlineCreated, DeadlineCompleted,
--   DeadlineRescheduled, DeadlineMissed
-- ============================================================

CREATE TABLE rm_deadlines (
    id                  UUID PRIMARY KEY,
    organisation_id     UUID NOT NULL,
    patent_id           UUID,
    trademark_id        UUID,
    asset_title         TEXT,              -- denormalized patent/trademark title
    asset_number        TEXT,              -- denormalized application number
    deadline_type       TEXT NOT NULL,
    due_date            DATE NOT NULL,
    grace_period_end    DATE,
    jurisdiction_code   CHAR(2),
    jurisdiction_name   TEXT,              -- denormalized
    status              TEXT NOT NULL,
    assigned_to_user_id UUID,
    assigned_to_name    TEXT,              -- denormalized
    fee_amount          NUMERIC(10,2),
    fee_currency        CHAR(3),
    completed_at        TIMESTAMPTZ,
    days_until_due      INTEGER GENERATED ALWAYS AS (due_date - CURRENT_DATE) STORED,
    last_event_version  INTEGER NOT NULL,
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_deadlines_org ON rm_deadlines(organisation_id);
CREATE INDEX idx_rm_deadlines_due ON rm_deadlines(due_date);
CREATE INDEX idx_rm_deadlines_status ON rm_deadlines(status);
CREATE INDEX idx_rm_deadlines_assigned ON rm_deadlines(assigned_to_user_id);

-- ============================================================
-- READ MODEL: Annuity Decision Queue
-- Materialised from: DeadlineCreated (annuity type),
--   AnnuityDecisionMade, AnnuityDecisionOverridden
-- ============================================================

CREATE TABLE rm_annuity_queue (
    id                  UUID PRIMARY KEY,
    organisation_id     UUID NOT NULL,
    patent_id           UUID NOT NULL,
    patent_title        TEXT,
    patent_number       TEXT,
    jurisdiction_code   CHAR(2),
    due_date            DATE NOT NULL,
    fee_amount          NUMERIC(10,2),
    fee_currency        CHAR(3),
    decision            TEXT,              -- NULL if undecided
    decided_by_name     TEXT,
    decided_at          TIMESTAMPTZ,
    ai_recommendation   TEXT,
    ai_confidence       NUMERIC(3,2),
    citation_score      NUMERIC(5,2),
    forward_citation_count INTEGER,
    business_alignment  NUMERIC(5,2),
    licensing_potential  NUMERIC(5,2),
    portfolio_coverage  NUMERIC(5,2),
    last_event_version  INTEGER NOT NULL,
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_annuity_org ON rm_annuity_queue(organisation_id);
CREATE INDEX idx_rm_annuity_due ON rm_annuity_queue(due_date);
CREATE INDEX idx_rm_annuity_decision ON rm_annuity_queue(decision);

-- ============================================================
-- READ MODEL: Trademark Portfolio View
-- Materialised from: TrademarkFiled, TrademarkStatusChanged,
--   TrademarkClassAssigned, MadridDesignationFiled
-- ============================================================

CREATE TABLE rm_trademarks (
    id                  UUID PRIMARY KEY,
    organisation_id     UUID NOT NULL,
    registration_number TEXT,
    application_number  TEXT NOT NULL,
    mark_text           TEXT,
    mark_type           TEXT NOT NULL,
    filing_date         DATE NOT NULL,
    registration_date   DATE,
    expiry_date         DATE,
    jurisdiction_code   CHAR(2) NOT NULL,
    status              TEXT NOT NULL,
    nice_classes        SMALLINT[],        -- denormalized array
    owner_name          TEXT,
    madrid_reg_number   TEXT,
    next_renewal_date   DATE,
    last_event_version  INTEGER NOT NULL,
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_trademarks_org ON rm_trademarks(organisation_id);
CREATE INDEX idx_rm_trademarks_status ON rm_trademarks(status);

-- ============================================================
-- READ MODEL: Portfolio Analytics / Cost Summary
-- Materialised from: InvoiceRecorded, FeePaymentMade,
--   AnnuityDecisionMade, CounselAssigned
-- ============================================================

CREATE TABLE rm_portfolio_costs (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL,
    patent_id           UUID,
    trademark_id        UUID,
    asset_title         TEXT,
    jurisdiction_code   CHAR(2),
    cost_category       TEXT NOT NULL,      -- 'filing', 'prosecution', 'annuity', 'counsel', 'search'
    total_spent         NUMERIC(12,2) NOT NULL DEFAULT 0,
    currency_code       CHAR(3),
    year                INTEGER,
    last_event_version  INTEGER NOT NULL,
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_costs_org ON rm_portfolio_costs(organisation_id);
CREATE INDEX idx_rm_costs_year ON rm_portfolio_costs(year);

-- ============================================================
-- READ MODEL: Prior Art & FTO Risk Dashboard
-- Materialised from: PriorArtSearchCompleted, FTOAnalysisCompleted
-- ============================================================

CREATE TABLE rm_fto_risk (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL,
    patent_id           UUID,
    search_type         TEXT NOT NULL,
    searched_at         TIMESTAMPTZ,
    overall_risk_score  NUMERIC(3,2),
    high_risk_count     INTEGER DEFAULT 0,
    medium_risk_count   INTEGER DEFAULT 0,
    low_risk_count      INTEGER DEFAULT 0,
    data_sources        TEXT[],
    last_event_version  INTEGER NOT NULL,
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_fto_org ON rm_fto_risk(organisation_id);
```

## Projection Tracking

```sql
-- ============================================================
-- PROJECTION CHECKPOINTS
-- Tracks which events each read model has processed
-- ============================================================

CREATE TABLE projection_checkpoints (
    projection_name TEXT PRIMARY KEY,       -- e.g., 'rm_patents', 'rm_deadlines'
    last_event_id   UUID NOT NULL,
    last_event_at   TIMESTAMPTZ NOT NULL,
    events_processed BIGINT NOT NULL DEFAULT 0,
    last_rebuilt_at  TIMESTAMPTZ,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Supporting Tables (Shared State)

```sql
-- ============================================================
-- REFERENCE DATA (not event-sourced -- static reference)
-- ============================================================

CREATE TABLE jurisdictions (
    country_code        CHAR(2) PRIMARY KEY,
    country_name        TEXT NOT NULL,
    is_pct_member       BOOLEAN NOT NULL DEFAULT false,
    is_paris_member     BOOLEAN NOT NULL DEFAULT false,
    is_madrid_member    BOOLEAN NOT NULL DEFAULT false,
    pct_national_phase_months INTEGER DEFAULT 30,
    patent_term_years   INTEGER DEFAULT 20,
    currency_code       CHAR(3)
);

CREATE TABLE nice_classes (
    class_number    SMALLINT PRIMARY KEY CHECK (class_number BETWEEN 1 AND 45),
    class_type      TEXT NOT NULL CHECK (class_type IN ('goods', 'services')),
    class_heading   TEXT NOT NULL
);

CREATE TABLE organisations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    org_type        TEXT NOT NULL,
    country_code    CHAR(2),
    settings        JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    email           TEXT NOT NULL,
    display_name    TEXT NOT NULL,
    role            TEXT NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, email)
);

-- ============================================================
-- GDPR: CRYPTO-SHREDDING KEY STORE
-- Per-party encryption keys for personal data in events
-- ============================================================

CREATE TABLE party_encryption_keys (
    party_id        UUID PRIMARY KEY,
    organisation_id UUID NOT NULL,
    encryption_key_encrypted BYTEA NOT NULL, -- encrypted with master key
    is_active       BOOLEAN NOT NULL DEFAULT true,
    shredded_at     TIMESTAMPTZ,            -- set when GDPR erasure requested
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Example Queries

### Replay Patent State from Events

```sql
-- Rebuild the current state of a patent by replaying all events
SELECT
    event_type,
    event_version,
    payload,
    metadata,
    created_at
FROM event_store
WHERE stream_id = '550e8400-e29b-41d4-a716-446655440000'
  AND stream_type = 'Patent'
ORDER BY event_version ASC;

-- With snapshot optimisation:
-- 1. Load latest snapshot
SELECT state, snapshot_version
FROM snapshots
WHERE stream_id = '550e8400-e29b-41d4-a716-446655440000'
ORDER BY snapshot_version DESC
LIMIT 1;

-- 2. Replay only events after the snapshot
SELECT event_type, payload, metadata, created_at
FROM event_store
WHERE stream_id = '550e8400-e29b-41d4-a716-446655440000'
  AND event_version > 42  -- snapshot_version
ORDER BY event_version ASC;
```

### Bi-Temporal Query: "What was this patent's status on March 1, as we knew it on April 15?"

```sql
-- Point-in-time state reconstruction
SELECT event_type, payload, created_at
FROM event_store
WHERE stream_id = '550e8400-e29b-41d4-a716-446655440000'
  AND stream_type = 'Patent'
  AND event_type = 'PatentStatusChanged'
  AND created_at <= '2026-04-15T23:59:59Z'          -- as known on April 15
  AND (payload->>'effective_date')::date <= '2026-03-01'  -- effective on March 1
ORDER BY event_version DESC
LIMIT 1;
```

### All Annuity Decisions with Full History

```sql
-- Every annuity decision ever made, with AI scores at time of decision
SELECT
    e.stream_id AS patent_id,
    e.payload->>'decision' AS decision,
    e.payload->>'ai_recommendation' AS ai_recommendation,
    (e.payload->>'ai_confidence')::numeric AS ai_confidence,
    (e.payload->>'cost_if_paid')::numeric AS cost,
    e.metadata->>'user_id' AS decided_by,
    e.created_at
FROM event_store e
WHERE e.organisation_id = 'org-uuid'
  AND e.event_type = 'AnnuityDecisionMade'
ORDER BY e.created_at DESC;
```

### AI Analytics: Patents with Most Status Changes Before Abandonment

```sql
-- Find patterns: patents that churned through many statuses before being abandoned
WITH abandoned_patents AS (
    SELECT DISTINCT stream_id
    FROM event_store
    WHERE event_type = 'PatentStatusChanged'
      AND payload->>'new_status' = 'abandoned'
),
status_change_counts AS (
    SELECT
        e.stream_id,
        COUNT(*) AS change_count,
        MIN(e.created_at) AS first_change,
        MAX(e.created_at) AS last_change
    FROM event_store e
    JOIN abandoned_patents ap ON e.stream_id = ap.stream_id
    WHERE e.event_type = 'PatentStatusChanged'
    GROUP BY e.stream_id
)
SELECT
    sc.stream_id,
    rp.title,
    sc.change_count,
    sc.last_change - sc.first_change AS prosecution_duration
FROM status_change_counts sc
JOIN rm_patents rp ON sc.stream_id = rp.id
ORDER BY sc.change_count DESC
LIMIT 20;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store (Write Side) | 3 | event_store, event_type_registry, snapshots |
| Projection Tracking | 1 | projection_checkpoints |
| Read Model: Patents | 1 | rm_patents (denormalized) |
| Read Model: Deadlines | 1 | rm_deadlines (denormalized) |
| Read Model: Annuities | 1 | rm_annuity_queue (denormalized) |
| Read Model: Trademarks | 1 | rm_trademarks (denormalized) |
| Read Model: Costs | 1 | rm_portfolio_costs |
| Read Model: FTO Risk | 1 | rm_fto_risk |
| Reference Data | 2 | jurisdictions, nice_classes |
| Identity | 2 | organisations, users |
| GDPR Support | 1 | party_encryption_keys |
| **Total** | **15** | Plus additional read models as needed |

---

## Key Design Decisions

1. **Single event_store table for all aggregate types** rather than separate event tables per entity. The `stream_type` and `stream_id` columns partition events logically. This simplifies the infrastructure (one table to back up, partition, and index) and enables cross-entity audit queries.

2. **Optimistic concurrency via `(stream_id, event_version)` unique constraint**. Two concurrent writers to the same patent stream will fail at the database level, preventing lost updates without application-level locks. This is essential for deadline management where concurrent edits could cause missed deadlines.

3. **JSONB event payloads** rather than strongly-typed event tables. Each event type has a different field set, and new event types can be added without schema migrations. The `event_type_registry` stores JSON Schema definitions for runtime validation.

4. **Denormalized read models** -- each read model (`rm_patents`, `rm_deadlines`, `rm_annuity_queue`) contains all the fields needed for its dashboard without joins. The `rm_deadlines` table includes a computed `days_until_due` column for instant deadline prioritisation.

5. **Snapshot strategy for replay performance**. For patents with long prosecution histories (10+ years, hundreds of events), replaying from the beginning is expensive. Snapshots at every 100 events provide bounded replay time. The snapshot table uses a composite primary key `(stream_id, snapshot_version)` to support multiple snapshots per stream.

6. **Crypto-shredding for GDPR compliance** rather than deleting events (which would break the immutable log). Personal data in event payloads is encrypted with per-party keys. When a GDPR erasure request is received, the party's key is destroyed, making all personal data in their events cryptographically unrecoverable without altering the event stream integrity.

7. **Projection checkpoints** track exactly which event each read model has processed. If a projection crashes, it can resume from its checkpoint rather than replaying all events. This also enables monitoring for projection lag.

8. **No separate audit log table** -- the event store IS the audit trail. Every event includes metadata (user_id, ip_address, correlation_id) providing attribution. This eliminates the common problem of audit logs that diverge from actual state.

9. **Event metadata includes correlation and causation IDs** to trace chains of events. For example, a `PatentOfficeCorrespondenceReceived` event (causation) triggers `DeadlineCreated` events (correlated). This enables full causal chain reconstruction for debugging and compliance.

10. **Read models are disposable and rebuildable**. If a read model is corrupted or a new view is needed, it can be rebuilt by replaying all events from the event store through the projection logic. The `last_rebuilt_at` timestamp in `projection_checkpoints` tracks when this was last done.
