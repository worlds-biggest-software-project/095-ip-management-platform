# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: IP Management Platform · Created: 2026-05-12

## Philosophy

The Hybrid Relational + JSONB model uses strongly-typed relational columns for core fields that are universal across all jurisdictions and IP types (application number, filing date, status, jurisdiction code), while storing variable, jurisdiction-specific, and evolving fields in JSONB columns. This approach acknowledges a fundamental reality of IP management: the core lifecycle is consistent (file, prosecute, grant, maintain, expire), but the details vary enormously across 150+ jurisdictions, each with different rules, fee structures, forms, and procedural requirements.

A fully normalized schema forces you to either (a) add nullable columns for every jurisdiction-specific field (resulting in sparse, wide tables), or (b) create separate tables per jurisdiction (resulting in an unmanageable table count). The JSONB hybrid avoids both extremes. Core fields that are indexed and queried frequently live in typed columns. Jurisdiction-specific details (local agent requirements, specific form numbers, jurisdiction-specific fee categories, language requirements, local classification codes) live in JSONB columns where they can vary per record without schema migrations.

This pattern is widely used by modern SaaS platforms that serve diverse markets. It enables rapid MVP development (add new jurisdiction support without migrations), supports the plugin architecture described in the project's README (annuity service providers can store provider-specific data in JSONB fields), and naturally accommodates the AI-extracted data from patent office correspondence (which varies by office and document type).

**Best for:** Rapid MVP development, multi-jurisdiction flexibility, plugin architectures where third-party integrations store provider-specific data, and teams that want to iterate quickly on jurisdiction coverage.

**Trade-offs:**
- PRO: Significantly fewer tables than a fully normalized model (25-30 vs 60+)
- PRO: New jurisdictions and their specific fields can be supported without schema migrations
- PRO: JSONB columns naturally store AI-extracted data whose structure evolves as the AI improves
- PRO: Plugin-compatible: annuity service providers, patent office connectors, and analytics tools can store their data in JSONB without new tables
- PRO: PostgreSQL GIN indexes on JSONB enable fast queries on semi-structured data
- CON: JSONB fields lack foreign key constraints -- referential integrity for nested data must be enforced at the application level
- CON: JSONB queries use different syntax (e.g., `->>'field'`) than standard SQL, increasing learning curve
- CON: Type safety in JSONB is weaker -- a string "30" vs integer 30 can cause subtle bugs
- CON: Complex JSONB queries can be slower than equivalent queries on indexed typed columns
- CON: Schema documentation must be maintained separately since JSONB structure is not self-documenting in the DDL

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| WIPO ST.96 | Incoming ST.96 XML is parsed; core fields populate typed columns, full ST.96 payload stored in `st96_data` JSONB for completeness |
| ISO 3166-1/2 | `jurisdiction_code` is a typed CHAR(2) column on core tables; jurisdiction-specific rules in JSONB on `jurisdictions.rules` |
| NICE Classification | Class numbers in typed columns; class-specific goods/services descriptions in JSONB |
| Paris Convention | Priority claims as structured JSONB array on patent records, with computed deadline in typed column |
| PCT | National phase deadlines computed from typed `filing_date` + jurisdiction JSONB rules |
| Madrid Protocol | Madrid registration data as JSONB on trademark records; designation details in JSONB array |
| ISO 27001 / NIST SP 800-92 | Audit trail in typed `audit_log` table; changed fields captured as JSONB diffs |
| CPC / IPC | Primary classification in typed column; full classification hierarchy in JSONB |
| OpenAPI 3.1 / JSON Schema | JSONB column schemas documented via JSON Schema definitions stored in `schema_definitions` table |

---

## Core Tables

```sql
-- ============================================================
-- ORGANISATIONS & USERS
-- ============================================================

CREATE TABLE organisations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    org_type        TEXT NOT NULL CHECK (org_type IN ('corporation', 'law_firm', 'university', 'government', 'individual')),
    country_code    CHAR(2) NOT NULL,
    settings        JSONB NOT NULL DEFAULT '{}',
    -- settings example:
    -- {
    --   "default_currency": "USD",
    --   "deadline_alert_days": [90, 60, 30, 14, 7],
    --   "annuity_service_provider": "dennemeyer",
    --   "enabled_jurisdictions": ["US", "EP", "JP", "CN", "KR"],
    --   "ai_features": {"annuity_scoring": true, "auto_docketing": true}
    -- }
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
    preferences     JSONB NOT NULL DEFAULT '{}',
    -- preferences example:
    -- {
    --   "timezone": "America/New_York",
    --   "notification_channels": ["email", "in_app"],
    --   "dashboard_layout": "compact",
    --   "default_jurisdiction_filter": ["US", "EP"]
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, email)
);

CREATE INDEX idx_users_org ON users(organisation_id);
CREATE INDEX idx_users_email ON users(email);
```

## Jurisdictions (Reference Data with JSONB Rules)

```sql
-- ============================================================
-- JURISDICTIONS -- typed core + JSONB rules per jurisdiction
-- ============================================================

CREATE TABLE jurisdictions (
    country_code    CHAR(2) PRIMARY KEY,
    country_name    TEXT NOT NULL,
    is_pct_member   BOOLEAN NOT NULL DEFAULT false,
    is_paris_member BOOLEAN NOT NULL DEFAULT false,
    is_madrid_member BOOLEAN NOT NULL DEFAULT false,
    currency_code   CHAR(3),
    rules           JSONB NOT NULL DEFAULT '{}',
    -- rules example for US:
    -- {
    --   "pct_national_phase_months": 30,
    --   "patent_term_years": 20,
    --   "patent_term_adjustment": true,
    --   "maintenance_fee_years": [4, 8, 12],
    --   "grace_period_months": 6,
    --   "grace_period_surcharge_pct": 50,
    --   "office_action_response_months": 3,
    --   "extension_available": true,
    --   "max_extensions": 3,
    --   "extension_fee_per_month": 230,
    --   "working_requirement": false,
    --   "language": "en",
    --   "local_agent_required": false,
    --   "annuity_start_year": 4
    -- }
    --
    -- rules example for JP:
    -- {
    --   "pct_national_phase_months": 30,
    --   "patent_term_years": 20,
    --   "maintenance_fee_years": [1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20],
    --   "office_action_response_months": 3,
    --   "request_for_examination_years": 3,
    --   "local_agent_required": true,
    --   "language": "ja",
    --   "translation_required": true,
    --   "annuity_start_year": 1
    -- }
    fee_schedule    JSONB NOT NULL DEFAULT '[]',
    -- fee_schedule example:
    -- [
    --   {"type": "filing", "amount": 320, "currency": "USD"},
    --   {"type": "examination", "amount": 800, "currency": "USD"},
    --   {"type": "annuity", "year": 4, "amount": 1600, "currency": "USD"},
    --   {"type": "annuity", "year": 8, "amount": 3600, "currency": "USD"},
    --   {"type": "annuity", "year": 12, "amount": 7400, "currency": "USD"},
    --   {"type": "late_surcharge", "amount": 150, "currency": "USD", "grace_months": 6}
    -- ]
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_jurisdictions_pct ON jurisdictions(is_pct_member) WHERE is_pct_member;
```

## IP Assets (Unified Table)

```sql
-- ============================================================
-- IP ASSETS -- unified table for patents and trademarks
-- with JSONB for type-specific and jurisdiction-specific fields
-- ============================================================

CREATE TABLE ip_assets (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    asset_type          TEXT NOT NULL CHECK (asset_type IN ('patent', 'trademark', 'design')),

    -- Universal typed columns (all IP types share these)
    application_number  TEXT NOT NULL,
    registration_number TEXT,              -- patent number or trademark registration number
    title               TEXT NOT NULL,
    filing_date         DATE NOT NULL,
    registration_date   DATE,              -- grant_date for patents, registration_date for trademarks
    expiry_date         DATE,
    priority_date       DATE,
    jurisdiction_code   CHAR(2) NOT NULL REFERENCES jurisdictions(country_code),
    status              TEXT NOT NULL,
    family_id           UUID,              -- for patent family grouping
    cost_to_date        NUMERIC(12,2) DEFAULT 0,
    notes               TEXT,

    -- Type-specific fields in JSONB
    patent_data         JSONB,
    -- patent_data example:
    -- {
    --   "patent_type": "utility",
    --   "abstract": "A method for quantum error correction...",
    --   "pct_application_number": "PCT/US2025/012345",
    --   "claims_count": 24,
    --   "independent_claims_count": 3,
    --   "primary_cpc": "H03M13/00",
    --   "classifications": [
    --     {"type": "cpc", "code": "H03M13/00", "primary": true},
    --     {"type": "ipc", "code": "H03M13/29", "primary": false}
    --   ],
    --   "claims": [
    --     {"number": 1, "type": "independent", "text": "A method comprising..."},
    --     {"number": 2, "type": "dependent", "depends_on": 1, "text": "The method of claim 1..."}
    --   ]
    -- }

    trademark_data      JSONB,
    -- trademark_data example:
    -- {
    --   "mark_text": "QUANTUM SHIELD",
    --   "mark_type": "word",
    --   "nice_classes": [9, 42],
    --   "goods_services": {
    --     "9": "Computer software for quantum computing error correction",
    --     "42": "Software as a service for quantum computing"
    --   },
    --   "madrid_registration": {
    --     "international_reg_number": "1234567",
    --     "registration_date": "2026-01-15",
    --     "renewal_due": "2036-01-15",
    --     "designations": [
    --       {"jurisdiction": "EP", "status": "protected", "protection_date": "2026-06-01"},
    --       {"jurisdiction": "JP", "status": "pending"}
    --     ]
    --   },
    --   "logo_file": "s3://marks/quantum-shield-logo.png"
    -- }

    -- Jurisdiction-specific fields
    jurisdiction_data   JSONB,
    -- jurisdiction_data example (US patent):
    -- {
    --   "patent_term_adjustment_days": 45,
    --   "terminal_disclaimer": false,
    --   "provisional_filed": true,
    --   "provisional_app_number": "63/123456",
    --   "provisional_filing_date": "2025-01-15",
    --   "art_unit": "2111",
    --   "examiner_name": "Smith, John"
    -- }
    --
    -- jurisdiction_data example (EP patent):
    -- {
    --   "validated_countries": ["DE", "FR", "GB", "NL", "IT"],
    --   "opposition_period_end": "2027-03-15",
    --   "unitary_patent_opted": false,
    --   "designated_states": ["AT","BE","BG","CH","CY","CZ","DE","DK","EE","ES"]
    -- }

    -- AI-generated analytics
    ai_scores           JSONB,
    -- ai_scores example:
    -- {
    --   "portfolio_score": 7.8,
    --   "citation_score": 6.5,
    --   "forward_citation_count": 23,
    --   "business_alignment": 8.2,
    --   "licensing_potential": 6.1,
    --   "abandonment_risk": 0.15,
    --   "scored_at": "2026-05-10T14:30:00Z",
    --   "model_version": "annuity-scorer-v2.1"
    -- }

    -- Integration/sync metadata
    sync_data           JSONB,
    -- sync_data example:
    -- {
    --   "last_sync_at": "2026-05-12T08:00:00Z",
    --   "source": "epo_ops",
    --   "external_id": "EP3456789",
    --   "st96_last_import": "2026-05-11",
    --   "sync_status": "current"
    -- }

    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Core typed column indexes
CREATE INDEX idx_ip_assets_org ON ip_assets(organisation_id);
CREATE INDEX idx_ip_assets_type ON ip_assets(asset_type);
CREATE INDEX idx_ip_assets_jurisdiction ON ip_assets(jurisdiction_code);
CREATE INDEX idx_ip_assets_status ON ip_assets(status);
CREATE INDEX idx_ip_assets_filing_date ON ip_assets(filing_date);
CREATE INDEX idx_ip_assets_app_number ON ip_assets(application_number);
CREATE INDEX idx_ip_assets_family ON ip_assets(family_id);

-- JSONB GIN indexes for semi-structured queries
CREATE INDEX idx_ip_assets_patent_data ON ip_assets USING GIN (patent_data jsonb_path_ops)
    WHERE asset_type = 'patent';
CREATE INDEX idx_ip_assets_trademark_data ON ip_assets USING GIN (trademark_data jsonb_path_ops)
    WHERE asset_type = 'trademark';
CREATE INDEX idx_ip_assets_ai_scores ON ip_assets USING GIN (ai_scores jsonb_path_ops);

-- Expression index for common JSONB queries
CREATE INDEX idx_ip_assets_primary_cpc ON ip_assets ((patent_data->>'primary_cpc'))
    WHERE asset_type = 'patent';
CREATE INDEX idx_ip_assets_mark_text ON ip_assets ((trademark_data->>'mark_text'))
    WHERE asset_type = 'trademark';
```

## Deadlines

```sql
-- ============================================================
-- DEADLINES
-- ============================================================

CREATE TABLE deadlines (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    ip_asset_id         UUID NOT NULL REFERENCES ip_assets(id),
    deadline_type       TEXT NOT NULL CHECK (deadline_type IN (
        'office_action_response', 'annuity_payment', 'pct_national_phase',
        'priority_claim', 'trademark_renewal', 'madrid_renewal',
        'examination_request', 'opposition_period', 'appeal_deadline',
        'maintenance_fee', 'use_declaration', 'custom'
    )),
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

    -- Annuity decision fields (populated when deadline_type = 'annuity_payment')
    annuity_decision    JSONB,
    -- annuity_decision example:
    -- {
    --   "decision": "pay",
    --   "decided_by": "user-uuid",
    --   "decided_at": "2026-05-08T10:00:00Z",
    --   "ai_recommendation": "pay",
    --   "ai_confidence": 0.87,
    --   "scoring": {
    --     "citation_score": 7.5,
    --     "forward_citations": 23,
    --     "business_alignment": 8.2,
    --     "licensing_potential": 6.1,
    --     "portfolio_coverage": 7.0
    --   },
    --   "reasoning": "High citation count and active licensing interest"
    -- }

    -- Jurisdiction-specific deadline details
    details             JSONB,
    -- details example (office action response):
    -- {
    --   "office_action_type": "non_final_rejection",
    --   "office_action_date": "2026-05-01",
    --   "extensions_available": 3,
    --   "extension_fee_per_month": 230,
    --   "art_unit": "2111",
    --   "examiner": "Smith, John"
    -- }

    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_deadlines_org ON deadlines(organisation_id);
CREATE INDEX idx_deadlines_asset ON deadlines(ip_asset_id);
CREATE INDEX idx_deadlines_due ON deadlines(due_date);
CREATE INDEX idx_deadlines_status ON deadlines(status);
CREATE INDEX idx_deadlines_type ON deadlines(deadline_type);
CREATE INDEX idx_deadlines_assigned ON deadlines(assigned_to);
```

## Parties & Relationships

```sql
-- ============================================================
-- PARTIES -- inventors, assignees, counsel, licensees
-- ============================================================

CREATE TABLE parties (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    is_person       BOOLEAN NOT NULL DEFAULT true,
    display_name    TEXT NOT NULL,          -- computed: "Last, First" or org name
    contact         JSONB NOT NULL DEFAULT '{}',
    -- contact example:
    -- {
    --   "first_name": "Jane",
    --   "last_name": "Smith",
    --   "email": "jane.smith@example.com",
    --   "phone": "+1-555-0123",
    --   "address": {
    --     "street": "123 Innovation Dr",
    --     "city": "San Jose",
    --     "state": "CA",
    --     "country": "US",
    --     "postal_code": "95134"
    --   },
    --   "organisation_name": "Acme Corp",
    --   "bar_number": "CA-123456",
    --   "firm_name": "Smith & Associates LLP"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_parties_org ON parties(organisation_id);
CREATE INDEX idx_parties_name ON parties(display_name);

-- ============================================================
-- IP ASSET <-> PARTY relationships (many-to-many with role)
-- ============================================================

CREATE TABLE ip_asset_parties (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    ip_asset_id     UUID NOT NULL REFERENCES ip_assets(id) ON DELETE CASCADE,
    party_id        UUID NOT NULL REFERENCES parties(id),
    role            TEXT NOT NULL CHECK (role IN (
        'inventor', 'assignee', 'applicant', 'attorney_of_record',
        'agent', 'owner', 'opponent', 'licensee'
    )),
    is_current      BOOLEAN NOT NULL DEFAULT true,
    assignment_date DATE,
    details         JSONB,
    -- details example (inventor):
    -- {"contribution_pct": 60.0, "citizenship": "US"}
    -- details example (counsel):
    -- {"matter_number": "MAT-2026-042", "hourly_rate": 450, "currency": "USD"}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (ip_asset_id, party_id, role)
);

CREATE INDEX idx_ip_asset_parties_asset ON ip_asset_parties(ip_asset_id);
CREATE INDEX idx_ip_asset_parties_party ON ip_asset_parties(party_id);
```

## Documents & Correspondence

```sql
-- ============================================================
-- DOCUMENTS
-- ============================================================

CREATE TABLE documents (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    ip_asset_id     UUID REFERENCES ip_assets(id),
    document_type   TEXT NOT NULL CHECK (document_type IN (
        'office_action', 'response', 'specification', 'claims',
        'drawings', 'assignment', 'power_of_attorney', 'fee_receipt',
        'correspondence', 'search_report', 'examination_report',
        'grant_certificate', 'renewal_receipt', 'invention_disclosure',
        'fto_report', 'prior_art_report', 'licence_agreement', 'other'
    )),
    title           TEXT NOT NULL,
    file_name       TEXT NOT NULL,
    file_size_bytes BIGINT,
    mime_type       TEXT,
    storage_path    TEXT NOT NULL,
    received_date   DATE,
    source          TEXT CHECK (source IN ('patent_office', 'counsel', 'internal', 'ai_generated')),

    -- AI-extracted and office-specific data
    extracted_data  JSONB,
    -- extracted_data example (office action):
    -- {
    --   "ai_model": "docket-extractor-v3",
    --   "extracted_at": "2026-05-01T14:00:00Z",
    --   "confidence": 0.94,
    --   "action_type": "non_final_rejection",
    --   "mailing_date": "2026-04-28",
    --   "response_deadline": "2026-07-28",
    --   "rejected_claims": [1, 3, 5],
    --   "prior_art_cited": ["US10123456", "EP3456789"],
    --   "examiner_notes": "Claims 1, 3, 5 rejected under 35 USC 103..."
    -- }

    -- Full ST.96 payload if imported from patent office
    st96_data       JSONB,

    uploaded_by     UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_documents_org ON documents(organisation_id);
CREATE INDEX idx_documents_asset ON documents(ip_asset_id);
CREATE INDEX idx_documents_type ON documents(document_type);
```

## Citations & Prior Art

```sql
-- ============================================================
-- CITATIONS
-- ============================================================

CREATE TABLE citations (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    citing_asset_id     UUID NOT NULL REFERENCES ip_assets(id) ON DELETE CASCADE,
    cited_asset_id      UUID REFERENCES ip_assets(id),  -- NULL if external
    cited_reference     TEXT,                             -- external patent number or publication
    citation_type       TEXT NOT NULL CHECK (citation_type IN (
        'examiner_cited', 'applicant_cited', 'third_party', 'foreign', 'npl'
    )),
    citation_category   TEXT,                             -- EPO categories: X, Y, A, D, etc.
    details             JSONB,
    -- details example:
    -- {
    --   "relevance_to_claims": [1, 3],
    --   "cited_jurisdiction": "US",
    --   "cited_filing_date": "2020-03-15",
    --   "cited_title": "Method for..."
    -- }
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_citations_citing ON citations(citing_asset_id);
CREATE INDEX idx_citations_cited ON citations(cited_asset_id);

-- ============================================================
-- PRIOR ART SEARCHES & FTO
-- ============================================================

CREATE TABLE searches (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    ip_asset_id     UUID REFERENCES ip_assets(id),
    search_type     TEXT NOT NULL CHECK (search_type IN ('patentability', 'fto', 'invalidity', 'landscape')),
    query_text      TEXT,
    data_sources    TEXT[],
    searched_by     UUID REFERENCES users(id),
    searched_at     TIMESTAMPTZ NOT NULL DEFAULT now(),

    results         JSONB NOT NULL DEFAULT '[]',
    -- results example:
    -- [
    --   {
    --     "patent_number": "US10123456",
    --     "title": "Quantum Error Correction Method",
    --     "relevance_score": 0.89,
    --     "risk_level": "high",
    --     "claim_overlap": "Claims 1-3 overlap with claims 1, 4 of cited patent",
    --     "jurisdiction": "US",
    --     "assignee": "QuantumCorp Inc."
    --   },
    --   {
    --     "patent_number": "EP3456789",
    --     "title": "Surface Code Implementation",
    --     "relevance_score": 0.72,
    --     "risk_level": "medium",
    --     "claim_overlap": "Claim 5 partially overlaps",
    --     "jurisdiction": "EP"
    --   }
    -- ]

    summary         JSONB,
    -- summary example:
    -- {
    --   "total_results": 47,
    --   "high_risk_count": 3,
    --   "medium_risk_count": 8,
    --   "low_risk_count": 36,
    --   "overall_risk_score": 0.65,
    --   "ai_model": "fto-analyzer-v2",
    --   "analyzed_at": "2026-05-10T16:00:00Z"
    -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_searches_org ON searches(organisation_id);
CREATE INDEX idx_searches_asset ON searches(ip_asset_id);
```

## Invention Disclosures & Licensing

```sql
-- ============================================================
-- INVENTION DISCLOSURES
-- ============================================================

CREATE TABLE invention_disclosures (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    disclosure_number TEXT NOT NULL,
    title           TEXT NOT NULL,
    description     TEXT,
    disclosed_date  DATE NOT NULL,
    status          TEXT NOT NULL CHECK (status IN (
        'submitted', 'under_review', 'approved_for_filing',
        'filed', 'licensed', 'abandoned', 'rejected'
    )),
    ip_asset_id     UUID REFERENCES ip_assets(id),
    details         JSONB,
    -- details example:
    -- {
    --   "technology_area": "quantum_computing",
    --   "inventors": [
    --     {"party_id": "uuid", "name": "Jane Smith", "contribution_pct": 60},
    --     {"party_id": "uuid", "name": "Bob Jones", "contribution_pct": 40}
    --   ],
    --   "prior_disclosures": ["Conference paper at QIP 2025"],
    --   "potential_licensees": ["QuantumCorp", "TechGiant Inc."],
    --   "estimated_market_value": "high"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_disclosures_org ON invention_disclosures(organisation_id);

-- ============================================================
-- LICENCE AGREEMENTS
-- ============================================================

CREATE TABLE licence_agreements (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    agreement_number TEXT NOT NULL,
    licensee_party_id UUID NOT NULL REFERENCES parties(id),
    licence_type    TEXT NOT NULL CHECK (licence_type IN (
        'exclusive', 'non_exclusive', 'sole', 'cross_licence', 'compulsory'
    )),
    effective_date  DATE NOT NULL,
    expiry_date     DATE,
    status          TEXT NOT NULL CHECK (status IN ('draft', 'active', 'expired', 'terminated')),

    -- Licensed assets (array of IP asset IDs)
    licensed_asset_ids UUID[] NOT NULL DEFAULT '{}',

    terms           JSONB NOT NULL DEFAULT '{}',
    -- terms example:
    -- {
    --   "field_of_use": "Consumer electronics excluding automotive",
    --   "territory": ["US", "EU", "JP"],
    --   "royalty_type": "percentage",
    --   "royalty_rate": 3.5,
    --   "minimum_royalty": 50000,
    --   "minimum_royalty_currency": "USD",
    --   "payment_frequency": "quarterly",
    --   "payment_schedule": [
    --     {"period": "Q1", "due_date": "2026-04-30"},
    --     {"period": "Q2", "due_date": "2026-07-31"}
    --   ],
    --   "milestones": [
    --     {"event": "First commercial sale", "payment": 100000}
    --   ],
    --   "sublicensing_allowed": false,
    --   "audit_rights": true
    -- }

    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_licences_org ON licence_agreements(organisation_id);
CREATE INDEX idx_licences_licensee ON licence_agreements(licensee_party_id);
CREATE INDEX idx_licences_assets ON licence_agreements USING GIN (licensed_asset_ids);
```

## Cost Tracking

```sql
-- ============================================================
-- COST ENTRIES (unified cost tracking)
-- ============================================================

CREATE TABLE cost_entries (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    ip_asset_id     UUID REFERENCES ip_assets(id),
    cost_type       TEXT NOT NULL CHECK (cost_type IN (
        'filing_fee', 'examination_fee', 'grant_fee', 'annuity_fee',
        'counsel_fee', 'translation', 'search_fee', 'agent_fee',
        'trademark_registration', 'trademark_renewal', 'other'
    )),
    amount          NUMERIC(12,2) NOT NULL,
    currency_code   CHAR(3) NOT NULL,
    incurred_date   DATE NOT NULL,
    vendor_name     TEXT,
    invoice_number  TEXT,
    status          TEXT NOT NULL CHECK (status IN ('incurred', 'approved', 'paid', 'disputed')),
    details         JSONB,
    -- details example:
    -- {
    --   "counsel_assignment_id": "uuid",
    --   "hours_billed": 4.5,
    --   "hourly_rate": 450,
    --   "matter_number": "MAT-2026-042",
    --   "description": "Response to non-final office action"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_costs_org ON cost_entries(organisation_id);
CREATE INDEX idx_costs_asset ON cost_entries(ip_asset_id);
CREATE INDEX idx_costs_date ON cost_entries(incurred_date);
CREATE INDEX idx_costs_type ON cost_entries(cost_type);
```

## Audit Log & Integration

```sql
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
    -- changes example:
    -- {
    --   "old": {"status": "pending"},
    --   "new": {"status": "under_examination"},
    --   "fields_changed": ["status"]
    -- }
    ip_address      INET,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_org ON audit_log(organisation_id);
CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_created ON audit_log(created_at);

-- ============================================================
-- PATENT OFFICE CONNECTIONS
-- ============================================================

CREATE TABLE integrations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    integration_type TEXT NOT NULL CHECK (integration_type IN (
        'uspto', 'epo_ops', 'wipo', 'jpo', 'cnipa', 'euipo',
        'dennemeyer', 'questel_pavis', 'cpa_global', 'custom'
    )),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    config          JSONB NOT NULL DEFAULT '{}',
    -- config example (epo_ops):
    -- {
    --   "api_endpoint": "https://ops.epo.org/3.2/rest-services",
    --   "auth_type": "oauth2",
    --   "client_id": "encrypted:...",
    --   "client_secret": "encrypted:...",
    --   "sync_frequency_hours": 24,
    --   "last_sync_at": "2026-05-12T08:00:00Z",
    --   "sync_status": "healthy"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- SCHEMA DEFINITIONS (documents JSONB field structures)
-- ============================================================

CREATE TABLE schema_definitions (
    schema_name     TEXT PRIMARY KEY,
    schema_version  INTEGER NOT NULL DEFAULT 1,
    json_schema     JSONB NOT NULL,        -- JSON Schema Draft 2020-12
    description     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Store JSON Schema definitions for all JSONB column structures
-- This enables runtime validation and self-documentation
-- Example: schema_name = 'patent_data', 'trademark_data', 'jurisdiction_rules', etc.
```

## Example Queries

### All Upcoming Deadlines with Asset Details

```sql
SELECT
    d.id,
    d.deadline_type,
    d.due_date,
    d.status,
    d.fee_amount,
    d.fee_currency,
    ia.title AS asset_title,
    ia.application_number,
    ia.asset_type,
    ia.jurisdiction_code,
    u.display_name AS assigned_to_name
FROM deadlines d
JOIN ip_assets ia ON d.ip_asset_id = ia.id
LEFT JOIN users u ON d.assigned_to = u.id
WHERE d.organisation_id = 'org-uuid'
  AND d.status IN ('upcoming', 'due_soon')
  AND d.due_date <= CURRENT_DATE + INTERVAL '90 days'
ORDER BY d.due_date ASC;
```

### Patents by CPC Classification (JSONB query)

```sql
SELECT
    id, title, application_number, filing_date, status,
    patent_data->>'primary_cpc' AS primary_cpc,
    ai_scores->>'portfolio_score' AS portfolio_score
FROM ip_assets
WHERE organisation_id = 'org-uuid'
  AND asset_type = 'patent'
  AND patent_data->>'primary_cpc' LIKE 'H03M%'
ORDER BY (ai_scores->>'portfolio_score')::numeric DESC NULLS LAST;
```

### Trademarks by NICE Class (JSONB containment)

```sql
SELECT
    id, title, application_number, status,
    trademark_data->>'mark_text' AS mark,
    trademark_data->'nice_classes' AS classes
FROM ip_assets
WHERE organisation_id = 'org-uuid'
  AND asset_type = 'trademark'
  AND trademark_data->'nice_classes' @> '9';  -- contains class 9
```

### PCT National Phase Deadline Computation

```sql
SELECT
    ia.id,
    ia.title,
    ia.filing_date,
    j.country_code,
    j.country_name,
    ia.filing_date + ((j.rules->>'pct_national_phase_months')::int * INTERVAL '1 month') AS national_phase_deadline,
    j.rules->>'local_agent_required' AS needs_local_agent,
    j.rules->>'language' AS filing_language,
    j.rules->>'translation_required' AS needs_translation
FROM ip_assets ia
CROSS JOIN jurisdictions j
WHERE ia.id = 'pct-patent-uuid'
  AND ia.patent_data->>'pct_application_number' IS NOT NULL
  AND j.is_pct_member = true
ORDER BY national_phase_deadline ASC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity | 2 | organisations, users |
| Reference Data | 1 | jurisdictions (with JSONB rules and fee_schedule) |
| IP Assets | 1 | ip_assets (unified patents + trademarks + designs, with JSONB) |
| Deadlines | 1 | deadlines (with JSONB annuity_decision and details) |
| Parties | 2 | parties, ip_asset_parties |
| Documents | 1 | documents (with JSONB extracted_data and st96_data) |
| Citations & Search | 2 | citations, searches (with JSONB results) |
| Disclosures | 1 | invention_disclosures (with JSONB details) |
| Licensing | 1 | licence_agreements (with JSONB terms) |
| Cost Tracking | 1 | cost_entries (with JSONB details) |
| Audit & Integration | 3 | audit_log, integrations, schema_definitions |
| **Total** | **16** | |

---

## Key Design Decisions

1. **Unified `ip_assets` table** for patents, trademarks, and designs instead of separate tables. The `asset_type` discriminator and type-specific JSONB columns (`patent_data`, `trademark_data`) provide the structural variation. This enables a single query for "all IP assets expiring this year" without UNION across multiple tables, which is the most common portfolio reporting query.

2. **Jurisdiction rules in JSONB** on the `jurisdictions` table rather than separate rule tables. Each of the 150+ jurisdictions has different deadline rules, fee structures, agent requirements, and language requirements. Storing these as structured JSONB means a new jurisdiction can be added by inserting a row with the appropriate rules object, rather than migrating the schema.

3. **Fee schedules embedded in jurisdictions JSONB** rather than a separate `fee_schedules` table. Fee schedules are jurisdiction-specific reference data that changes infrequently (typically annually). Embedding them in JSONB keeps the table count low and allows each jurisdiction's fee structure to have a different shape (US has 3 maintenance fee windows; JP has 20 annual annuity amounts; EP has validation fees per country).

4. **JSONB GIN indexes with partial index conditions** (`WHERE asset_type = 'patent'`) ensure JSONB queries on patent-specific fields only index patent rows, not the entire table. This keeps index sizes manageable.

5. **Search results stored as JSONB arrays** on the `searches` table rather than in a separate `search_results` table. Each search produces a variable number of results with variable fields; JSONB arrays are a natural fit and avoid a high-cardinality junction table.

6. **Schema definitions table** provides self-documentation for all JSONB column structures using JSON Schema (Draft 2020-12). This compensates for JSONB's lack of intrinsic schema and enables runtime validation of JSONB payloads before insertion.

7. **Licensed asset IDs stored as UUID array** on `licence_agreements` rather than a junction table. This is a pragmatic trade-off: licence-to-asset relationships are typically queried from the licence side ("which assets does this licence cover?"), and PostgreSQL's GIN index on UUID arrays makes these queries efficient.

8. **AI scores as a dedicated JSONB column** on `ip_assets` rather than in a separate scoring table. The scores evolve as the AI model improves (new fields, new model versions), and storing them as JSONB avoids schema migrations when the scoring model changes. The `model_version` field in the JSONB tracks which model produced the scores.

9. **Integration configs in JSONB** on the `integrations` table rather than separate tables per integration type. Each patent office and annuity service provider has different connection requirements (OAuth credentials, API keys, sync schedules). JSONB accommodates this variation without separate tables per provider, supporting the plugin architecture described in the project README.

10. **16 tables total** vs 39 in the normalized model and 15 in the event-sourced model. This represents the sweet spot: enough structure for referential integrity on core relationships, enough flexibility in JSONB for the long tail of jurisdiction-specific and integration-specific variation.
