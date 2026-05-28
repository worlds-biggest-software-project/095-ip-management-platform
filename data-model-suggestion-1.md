# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: IP Management Platform · Created: 2026-05-12

## Philosophy

The Entity-Centric Normalized Relational model treats every domain concept as a first-class table with strongly-typed columns, foreign key constraints, and junction tables for many-to-many relationships. This is the classical approach for systems where data integrity is non-negotiable -- patent prosecution deadlines, annuity payments, and priority claims all have legal consequences if data is corrupted or inconsistent. Every relationship is explicit and enforced at the database level.

This approach mirrors how enterprise IPMS vendors (Anaqua AQX, CPA Global, Dennemeyer DIAMS) historically structure their data: separate tables for patents, trademarks, deadlines, parties, jurisdictions, and documents, with referential integrity enforced by the RDBMS. The model aligns naturally with WIPO ST.96's component-based XML schema, where patents, trademarks, and designs each have distinct element structures with shared common components.

The trade-off is table count and join complexity. A fully normalized IP management schema requires 50-70+ tables, and queries that span patent families across jurisdictions with their deadlines, parties, and documents will involve multi-table joins. However, for a domain where a missed deadline can cause irreversible loss of patent rights, the data integrity guarantees of normalization outweigh the query complexity costs.

**Best for:** Enterprise deployments where data integrity, regulatory compliance, and complex cross-entity reporting are paramount.

**Trade-offs:**
- PRO: Maximum data integrity via foreign keys and constraints -- critical when deadline errors cause irreversible patent loss
- PRO: Clean alignment with WIPO ST.96 entity structure and PatentsView data model
- PRO: Standard SQL queries; no special database extensions required
- PRO: Well-understood by enterprise development teams and DBAs
- CON: High table count (60+ tables) increases schema complexity and migration burden
- CON: Multi-table joins for common queries (e.g., "all deadlines for this patent family") can be slow without careful indexing
- CON: Adding jurisdiction-specific fields requires schema migrations rather than flexible columns
- CON: Reporting queries across the full portfolio can be complex and require views or materialised tables

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| WIPO ST.96 | Patent, trademark, and design entity structures map directly to ST.96 XML component schemas; field names align with ST.96 element names where possible |
| ISO 3166-1/2 | `jurisdictions` reference table uses ISO 3166 alpha-2 country codes and subdivision codes |
| NICE Classification (NCL 13-2026) | `nice_classes` reference table stores all 45 classes with headings; trademarks link via junction table |
| Paris Convention | Priority claims modeled as explicit `priority_claims` table with 12-month window enforcement |
| PCT | National phase entries tracked in `pct_national_phases` with jurisdiction-specific deadline rules |
| Madrid Protocol | Trademark international registrations and designations modeled in `madrid_registrations` and `madrid_designations` |
| ISO 27001 / NIST SP 800-92 | Audit log table structure aligned with SP 800-92 requirements for log retention and tamper detection |
| OpenID Connect / OAuth 2.0 | User authentication modeled to support OIDC identity providers via `identity_providers` and `user_identities` |
| CPC / IPC Classification | Patent classification stored in `patent_classifications` with CPC and IPC code references |

---

## Core Identity & Multi-Tenancy

```sql
-- ============================================================
-- ORGANISATIONS & USERS
-- ============================================================

CREATE TABLE organisations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    org_type        TEXT NOT NULL CHECK (org_type IN ('corporation', 'law_firm', 'university', 'government', 'individual')),
    country_code    CHAR(2) REFERENCES jurisdictions(country_code),
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

CREATE INDEX idx_users_org ON users(organisation_id);
CREATE INDEX idx_users_email ON users(email);

CREATE TABLE identity_providers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    provider_type   TEXT NOT NULL CHECK (provider_type IN ('oidc', 'saml', 'local')),
    issuer_url      TEXT,
    client_id       TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE user_identities (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id),
    provider_id     UUID NOT NULL REFERENCES identity_providers(id),
    external_id     TEXT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (provider_id, external_id)
);
```

## Reference Data

```sql
-- ============================================================
-- JURISDICTIONS & CLASSIFICATIONS
-- ============================================================

CREATE TABLE jurisdictions (
    country_code    CHAR(2) PRIMARY KEY,  -- ISO 3166-1 alpha-2
    country_name    TEXT NOT NULL,
    region_code     TEXT,                  -- ISO 3166-2 subdivision where relevant
    is_pct_member   BOOLEAN NOT NULL DEFAULT false,
    is_paris_member BOOLEAN NOT NULL DEFAULT false,
    is_madrid_member BOOLEAN NOT NULL DEFAULT false,
    pct_national_phase_months INTEGER DEFAULT 30,  -- 30, 31, or 34 depending on jurisdiction
    patent_term_years INTEGER DEFAULT 20,
    currency_code   CHAR(3),               -- ISO 4217
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE nice_classes (
    class_number    SMALLINT PRIMARY KEY CHECK (class_number BETWEEN 1 AND 45),
    class_type      TEXT NOT NULL CHECK (class_type IN ('goods', 'services')),
    class_heading   TEXT NOT NULL,
    edition         TEXT NOT NULL DEFAULT '13-2026'  -- NCL edition
);

CREATE TABLE cpc_classes (
    cpc_code        TEXT PRIMARY KEY,      -- e.g., 'A61K31/00'
    cpc_section     CHAR(1) NOT NULL,
    cpc_class       TEXT NOT NULL,
    cpc_subclass    TEXT NOT NULL,
    description     TEXT NOT NULL
);

CREATE TABLE ipc_classes (
    ipc_code        TEXT PRIMARY KEY,
    description     TEXT NOT NULL,
    edition         TEXT
);
```

## Patent Management

```sql
-- ============================================================
-- PATENTS & APPLICATIONS
-- ============================================================

CREATE TABLE patents (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    patent_number       TEXT,                 -- granted patent number
    application_number  TEXT NOT NULL,
    title               TEXT NOT NULL,
    abstract            TEXT,
    filing_date         DATE NOT NULL,
    grant_date          DATE,
    expiry_date         DATE,
    priority_date       DATE,
    jurisdiction_code   CHAR(2) NOT NULL REFERENCES jurisdictions(country_code),
    status              TEXT NOT NULL CHECK (status IN (
        'provisional', 'pending', 'published', 'under_examination',
        'allowed', 'granted', 'expired', 'abandoned', 'withdrawn', 'lapsed'
    )),
    patent_type         TEXT NOT NULL CHECK (patent_type IN (
        'utility', 'design', 'plant', 'provisional', 'pct', 'continuation',
        'continuation_in_part', 'divisional', 'reissue'
    )),
    family_id           UUID REFERENCES patent_families(id),
    pct_application_number TEXT,             -- PCT/XX/YYYY/NNNNNN
    cost_to_date        NUMERIC(12,2) DEFAULT 0,
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_patents_org ON patents(organisation_id);
CREATE INDEX idx_patents_jurisdiction ON patents(jurisdiction_code);
CREATE INDEX idx_patents_status ON patents(status);
CREATE INDEX idx_patents_family ON patents(family_id);
CREATE INDEX idx_patents_filing_date ON patents(filing_date);
CREATE INDEX idx_patents_app_number ON patents(application_number);

CREATE TABLE patent_families (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    family_name     TEXT,
    earliest_priority_date DATE,
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE patent_classifications (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    patent_id       UUID NOT NULL REFERENCES patents(id) ON DELETE CASCADE,
    classification_type TEXT NOT NULL CHECK (classification_type IN ('cpc', 'ipc', 'uspc')),
    classification_code TEXT NOT NULL,
    is_primary      BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_patent_classifications_patent ON patent_classifications(patent_id);
CREATE INDEX idx_patent_classifications_code ON patent_classifications(classification_code);

CREATE TABLE patent_claims (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    patent_id       UUID NOT NULL REFERENCES patents(id) ON DELETE CASCADE,
    claim_number    INTEGER NOT NULL,
    claim_type      TEXT NOT NULL CHECK (claim_type IN ('independent', 'dependent')),
    depends_on      INTEGER,               -- claim number this depends on
    claim_text      TEXT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_patent_claims_patent ON patent_claims(patent_id);
```

## Priority Claims & PCT

```sql
-- ============================================================
-- PRIORITY CLAIMS & PCT NATIONAL PHASE
-- ============================================================

CREATE TABLE priority_claims (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    patent_id           UUID NOT NULL REFERENCES patents(id) ON DELETE CASCADE,
    priority_country    CHAR(2) NOT NULL REFERENCES jurisdictions(country_code),
    priority_app_number TEXT NOT NULL,
    priority_date       DATE NOT NULL,
    priority_deadline   DATE NOT NULL,      -- priority_date + 12 months (Paris Convention)
    is_claimed          BOOLEAN NOT NULL DEFAULT false,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_priority_claims_patent ON priority_claims(patent_id);
CREATE INDEX idx_priority_claims_deadline ON priority_claims(priority_deadline);

CREATE TABLE pct_national_phases (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    patent_id           UUID NOT NULL REFERENCES patents(id),  -- the PCT application
    target_jurisdiction CHAR(2) NOT NULL REFERENCES jurisdictions(country_code),
    national_app_id     UUID REFERENCES patents(id),           -- the resulting national application
    entry_deadline      DATE NOT NULL,
    entry_date          DATE,
    status              TEXT NOT NULL CHECK (status IN ('pending', 'entered', 'missed', 'waived')),
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_pct_phases_patent ON pct_national_phases(patent_id);
CREATE INDEX idx_pct_phases_deadline ON pct_national_phases(entry_deadline);
CREATE INDEX idx_pct_phases_status ON pct_national_phases(status);
```

## Trademark Management

```sql
-- ============================================================
-- TRADEMARKS
-- ============================================================

CREATE TABLE trademarks (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    registration_number TEXT,
    application_number  TEXT NOT NULL,
    mark_text           TEXT,
    mark_type           TEXT NOT NULL CHECK (mark_type IN (
        'word', 'figurative', 'combined', 'three_dimensional',
        'sound', 'colour', 'motion', 'hologram', 'other'
    )),
    filing_date         DATE NOT NULL,
    registration_date   DATE,
    expiry_date         DATE,
    jurisdiction_code   CHAR(2) NOT NULL REFERENCES jurisdictions(country_code),
    status              TEXT NOT NULL CHECK (status IN (
        'pending', 'published', 'opposed', 'registered',
        'renewed', 'expired', 'cancelled', 'withdrawn'
    )),
    madrid_registration_id UUID REFERENCES madrid_registrations(id),
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_trademarks_org ON trademarks(organisation_id);
CREATE INDEX idx_trademarks_jurisdiction ON trademarks(jurisdiction_code);
CREATE INDEX idx_trademarks_status ON trademarks(status);

CREATE TABLE trademark_nice_classes (
    trademark_id    UUID NOT NULL REFERENCES trademarks(id) ON DELETE CASCADE,
    class_number    SMALLINT NOT NULL REFERENCES nice_classes(class_number),
    goods_services_description TEXT,
    PRIMARY KEY (trademark_id, class_number)
);

CREATE TABLE madrid_registrations (
    id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id         UUID NOT NULL REFERENCES organisations(id),
    international_reg_number TEXT NOT NULL,
    base_trademark_id       UUID REFERENCES trademarks(id),
    registration_date       DATE,
    renewal_due_date        DATE,          -- every 10 years
    status                  TEXT NOT NULL CHECK (status IN ('active', 'expired', 'cancelled')),
    created_at              TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at              TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE madrid_designations (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    madrid_reg_id       UUID NOT NULL REFERENCES madrid_registrations(id) ON DELETE CASCADE,
    jurisdiction_code   CHAR(2) NOT NULL REFERENCES jurisdictions(country_code),
    designation_date    DATE NOT NULL,
    protection_granted  BOOLEAN DEFAULT false,
    refusal_date        DATE,
    status              TEXT NOT NULL CHECK (status IN ('pending', 'protected', 'refused', 'withdrawn')),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Deadlines & Annuities

```sql
-- ============================================================
-- DEADLINES & ANNUITY MANAGEMENT
-- ============================================================

CREATE TABLE deadlines (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    patent_id           UUID REFERENCES patents(id),
    trademark_id        UUID REFERENCES trademarks(id),
    deadline_type       TEXT NOT NULL CHECK (deadline_type IN (
        'office_action_response', 'annuity_payment', 'pct_national_phase',
        'priority_claim', 'trademark_renewal', 'madrid_renewal',
        'examination_request', 'opposition_period', 'appeal_deadline',
        'publication_deadline', 'rce_filing', 'foreign_filing',
        'maintenance_fee', 'use_declaration', 'custom'
    )),
    due_date            DATE NOT NULL,
    grace_period_end    DATE,              -- some deadlines have grace periods with surcharges
    jurisdiction_code   CHAR(2) REFERENCES jurisdictions(country_code),
    status              TEXT NOT NULL CHECK (status IN (
        'upcoming', 'due_soon', 'overdue', 'completed', 'waived', 'missed'
    )),
    assigned_to         UUID REFERENCES users(id),
    completed_at        TIMESTAMPTZ,
    completed_by        UUID REFERENCES users(id),
    fee_amount          NUMERIC(10,2),
    fee_currency        CHAR(3),
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    CHECK (patent_id IS NOT NULL OR trademark_id IS NOT NULL)
);

CREATE INDEX idx_deadlines_org ON deadlines(organisation_id);
CREATE INDEX idx_deadlines_patent ON deadlines(patent_id);
CREATE INDEX idx_deadlines_trademark ON deadlines(trademark_id);
CREATE INDEX idx_deadlines_due_date ON deadlines(due_date);
CREATE INDEX idx_deadlines_status ON deadlines(status);
CREATE INDEX idx_deadlines_assigned ON deadlines(assigned_to);

CREATE TABLE annuity_decisions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    patent_id       UUID NOT NULL REFERENCES patents(id),
    deadline_id     UUID NOT NULL REFERENCES deadlines(id),
    decision        TEXT NOT NULL CHECK (decision IN ('pay', 'abandon', 'pending_review')),
    decided_by      UUID REFERENCES users(id),
    decided_at      TIMESTAMPTZ,
    -- AI scoring fields
    citation_score      NUMERIC(5,2),
    forward_citation_count INTEGER,
    business_alignment_score NUMERIC(5,2),
    licensing_potential_score NUMERIC(5,2),
    portfolio_coverage_score NUMERIC(5,2),
    ai_recommendation   TEXT CHECK (ai_recommendation IN ('pay', 'abandon', 'review')),
    ai_confidence       NUMERIC(3,2),       -- 0.00 to 1.00
    cost_if_paid        NUMERIC(10,2),
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_annuity_decisions_patent ON annuity_decisions(patent_id);
CREATE INDEX idx_annuity_decisions_deadline ON annuity_decisions(deadline_id);

CREATE TABLE fee_schedules (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    jurisdiction_code   CHAR(2) NOT NULL REFERENCES jurisdictions(country_code),
    fee_type            TEXT NOT NULL CHECK (fee_type IN (
        'filing_fee', 'examination_fee', 'grant_fee', 'annuity',
        'maintenance_fee', 'trademark_registration', 'trademark_renewal',
        'late_surcharge', 'restoration_fee'
    )),
    year_or_period      INTEGER,           -- for annuities: the year number
    amount              NUMERIC(10,2) NOT NULL,
    currency_code       CHAR(3) NOT NULL,
    effective_from      DATE NOT NULL,
    effective_to        DATE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_fee_schedules_jurisdiction ON fee_schedules(jurisdiction_code);
```

## Parties & Relationships

```sql
-- ============================================================
-- INVENTORS, ASSIGNEES, COUNSEL
-- ============================================================

CREATE TABLE parties (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    party_type      TEXT NOT NULL CHECK (party_type IN (
        'inventor', 'assignee', 'applicant', 'outside_counsel',
        'licensee', 'licensor', 'agent', 'examiner'
    )),
    is_person       BOOLEAN NOT NULL DEFAULT true,
    first_name      TEXT,
    last_name       TEXT,
    organisation_name TEXT,
    email           TEXT,
    country_code    CHAR(2) REFERENCES jurisdictions(country_code),
    address         TEXT,
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_parties_org ON parties(organisation_id);
CREATE INDEX idx_parties_type ON parties(party_type);

CREATE TABLE patent_parties (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    patent_id       UUID NOT NULL REFERENCES patents(id) ON DELETE CASCADE,
    party_id        UUID NOT NULL REFERENCES parties(id),
    role            TEXT NOT NULL CHECK (role IN (
        'inventor', 'assignee', 'applicant', 'attorney_of_record', 'agent'
    )),
    assignment_date DATE,
    is_current      BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (patent_id, party_id, role)
);

CREATE INDEX idx_patent_parties_patent ON patent_parties(patent_id);
CREATE INDEX idx_patent_parties_party ON patent_parties(party_id);

CREATE TABLE trademark_parties (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    trademark_id    UUID NOT NULL REFERENCES trademarks(id) ON DELETE CASCADE,
    party_id        UUID NOT NULL REFERENCES parties(id),
    role            TEXT NOT NULL CHECK (role IN (
        'owner', 'applicant', 'attorney_of_record', 'agent', 'opponent'
    )),
    is_current      BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (trademark_id, party_id, role)
);

CREATE TABLE outside_counsel_assignments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    counsel_party_id UUID NOT NULL REFERENCES parties(id),
    patent_id       UUID REFERENCES patents(id),
    trademark_id    UUID REFERENCES trademarks(id),
    firm_name       TEXT NOT NULL,
    matter_number   TEXT,
    hourly_rate     NUMERIC(8,2),
    rate_currency   CHAR(3),
    authorised_budget NUMERIC(12,2),
    spent_to_date   NUMERIC(12,2) DEFAULT 0,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_counsel_org ON outside_counsel_assignments(organisation_id);
CREATE INDEX idx_counsel_patent ON outside_counsel_assignments(patent_id);
```

## Documents & Correspondence

```sql
-- ============================================================
-- DOCUMENTS & CORRESPONDENCE
-- ============================================================

CREATE TABLE documents (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    patent_id       UUID REFERENCES patents(id),
    trademark_id    UUID REFERENCES trademarks(id),
    document_type   TEXT NOT NULL CHECK (document_type IN (
        'office_action', 'response', 'specification', 'claims',
        'drawings', 'assignment', 'power_of_attorney', 'fee_receipt',
        'correspondence', 'search_report', 'examination_report',
        'grant_certificate', 'renewal_receipt', 'invention_disclosure',
        'fto_report', 'prior_art_report', 'other'
    )),
    title           TEXT NOT NULL,
    file_name       TEXT NOT NULL,
    file_size_bytes BIGINT,
    mime_type       TEXT,
    storage_path    TEXT NOT NULL,          -- S3 key or file path
    received_date   DATE,
    filed_date      DATE,
    source          TEXT CHECK (source IN ('patent_office', 'counsel', 'internal', 'ai_generated')),
    ai_extracted_data JSONB,               -- structured data extracted by AI from correspondence
    notes           TEXT,
    uploaded_by     UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_documents_org ON documents(organisation_id);
CREATE INDEX idx_documents_patent ON documents(patent_id);
CREATE INDEX idx_documents_trademark ON documents(trademark_id);
CREATE INDEX idx_documents_type ON documents(document_type);
```

## Citations & Prior Art

```sql
-- ============================================================
-- CITATIONS & PRIOR ART
-- ============================================================

CREATE TABLE patent_citations (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    citing_patent_id    UUID NOT NULL REFERENCES patents(id) ON DELETE CASCADE,
    cited_patent_id     UUID REFERENCES patents(id),          -- NULL if external patent
    cited_patent_number TEXT,                                   -- external patent number if not in system
    cited_jurisdiction  CHAR(2) REFERENCES jurisdictions(country_code),
    citation_type       TEXT NOT NULL CHECK (citation_type IN (
        'examiner_cited', 'applicant_cited', 'third_party', 'foreign'
    )),
    citation_category   TEXT CHECK (citation_category IN ('X', 'Y', 'A', 'D', 'E', 'L', 'O', 'P', 'T')),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_citations_citing ON patent_citations(citing_patent_id);
CREATE INDEX idx_citations_cited ON patent_citations(cited_patent_id);

CREATE TABLE prior_art_searches (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    patent_id       UUID REFERENCES patents(id),
    search_type     TEXT NOT NULL CHECK (search_type IN ('patentability', 'fto', 'invalidity', 'landscape')),
    query_text      TEXT,
    data_sources    TEXT[],                -- e.g., ['uspto', 'epo_ops', 'wipo']
    result_count    INTEGER,
    risk_score      NUMERIC(3,2),          -- AI-generated 0.00-1.00
    searched_by     UUID REFERENCES users(id),
    searched_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE prior_art_results (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    search_id       UUID NOT NULL REFERENCES prior_art_searches(id) ON DELETE CASCADE,
    external_patent_number TEXT NOT NULL,
    title           TEXT,
    relevance_score NUMERIC(3,2),
    claim_overlap   TEXT,                  -- which claims overlap
    risk_level      TEXT CHECK (risk_level IN ('high', 'medium', 'low', 'none')),
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Invention Disclosures & Licensing

```sql
-- ============================================================
-- INVENTION DISCLOSURES (University Tech Transfer)
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
    technology_area TEXT,
    patent_id       UUID REFERENCES patents(id),  -- linked once filed
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE disclosure_inventors (
    disclosure_id   UUID NOT NULL REFERENCES invention_disclosures(id) ON DELETE CASCADE,
    party_id        UUID NOT NULL REFERENCES parties(id),
    contribution_pct NUMERIC(5,2),
    PRIMARY KEY (disclosure_id, party_id)
);

-- ============================================================
-- LICENSING
-- ============================================================

CREATE TABLE licence_agreements (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    agreement_number TEXT NOT NULL,
    licensee_party_id UUID NOT NULL REFERENCES parties(id),
    licence_type    TEXT NOT NULL CHECK (licence_type IN (
        'exclusive', 'non_exclusive', 'sole', 'cross_licence', 'compulsory'
    )),
    field_of_use    TEXT,
    territory       TEXT,                  -- jurisdiction scope
    effective_date  DATE NOT NULL,
    expiry_date     DATE,
    royalty_type    TEXT CHECK (royalty_type IN ('percentage', 'fixed', 'milestone', 'hybrid')),
    royalty_rate    NUMERIC(8,4),
    minimum_royalty NUMERIC(12,2),
    status          TEXT NOT NULL CHECK (status IN ('draft', 'active', 'expired', 'terminated')),
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE licence_patents (
    licence_id      UUID NOT NULL REFERENCES licence_agreements(id) ON DELETE CASCADE,
    patent_id       UUID NOT NULL REFERENCES patents(id),
    PRIMARY KEY (licence_id, patent_id)
);

CREATE TABLE royalty_payments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    licence_id      UUID NOT NULL REFERENCES licence_agreements(id),
    payment_date    DATE NOT NULL,
    amount          NUMERIC(12,2) NOT NULL,
    currency_code   CHAR(3) NOT NULL,
    period_start    DATE,
    period_end      DATE,
    status          TEXT NOT NULL CHECK (status IN ('scheduled', 'received', 'overdue', 'waived')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Invoices & Cost Tracking

```sql
-- ============================================================
-- INVOICES & COST TRACKING
-- ============================================================

CREATE TABLE invoices (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    counsel_assignment_id UUID REFERENCES outside_counsel_assignments(id),
    invoice_number  TEXT NOT NULL,
    vendor_name     TEXT NOT NULL,
    invoice_date    DATE NOT NULL,
    due_date        DATE,
    total_amount    NUMERIC(12,2) NOT NULL,
    currency_code   CHAR(3) NOT NULL,
    status          TEXT NOT NULL CHECK (status IN ('draft', 'submitted', 'approved', 'paid', 'disputed')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE invoice_line_items (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    invoice_id      UUID NOT NULL REFERENCES invoices(id) ON DELETE CASCADE,
    patent_id       UUID REFERENCES patents(id),
    trademark_id    UUID REFERENCES trademarks(id),
    description     TEXT NOT NULL,
    quantity        NUMERIC(10,2) DEFAULT 1,
    unit_price      NUMERIC(10,2) NOT NULL,
    line_total      NUMERIC(12,2) NOT NULL,
    fee_type        TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Audit Log

```sql
-- ============================================================
-- AUDIT LOG (NIST SP 800-92 aligned)
-- ============================================================

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL,
    user_id         UUID,
    action          TEXT NOT NULL,          -- 'create', 'update', 'delete', 'view', 'export', 'login'
    entity_type     TEXT NOT NULL,          -- 'patent', 'trademark', 'deadline', etc.
    entity_id       UUID,
    old_values      JSONB,
    new_values      JSONB,
    ip_address      INET,
    user_agent      TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_log_org ON audit_log(organisation_id);
CREATE INDEX idx_audit_log_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_log_user ON audit_log(user_id);
CREATE INDEX idx_audit_log_created ON audit_log(created_at);

-- Partition audit log by month for performance
-- CREATE TABLE audit_log (...) PARTITION BY RANGE (created_at);
```

## Patent Office Integration

```sql
-- ============================================================
-- PATENT OFFICE INTEGRATION & SYNC
-- ============================================================

CREATE TABLE patent_office_connections (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    office_code     TEXT NOT NULL CHECK (office_code IN ('uspto', 'epo', 'wipo', 'jpo', 'cnipa', 'kipo', 'euipo')),
    api_endpoint    TEXT NOT NULL,
    auth_type       TEXT NOT NULL CHECK (auth_type IN ('api_key', 'oauth2', 'certificate')),
    credentials_encrypted BYTEA,           -- encrypted API credentials
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_sync_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE sync_jobs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    connection_id   UUID NOT NULL REFERENCES patent_office_connections(id),
    job_type        TEXT NOT NULL CHECK (job_type IN ('status_update', 'correspondence_import', 'bulk_import')),
    status          TEXT NOT NULL CHECK (status IN ('queued', 'running', 'completed', 'failed')),
    records_processed INTEGER DEFAULT 0,
    error_message   TEXT,
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Multi-Tenancy | 4 | organisations, users, identity_providers, user_identities |
| Reference Data | 4 | jurisdictions, nice_classes, cpc_classes, ipc_classes |
| Patent Management | 4 | patents, patent_families, patent_classifications, patent_claims |
| Priority & PCT | 2 | priority_claims, pct_national_phases |
| Trademark Management | 4 | trademarks, trademark_nice_classes, madrid_registrations, madrid_designations |
| Deadlines & Annuities | 3 | deadlines, annuity_decisions, fee_schedules |
| Parties & Relationships | 4 | parties, patent_parties, trademark_parties, outside_counsel_assignments |
| Documents | 1 | documents |
| Citations & Prior Art | 3 | patent_citations, prior_art_searches, prior_art_results |
| Invention Disclosures | 2 | invention_disclosures, disclosure_inventors |
| Licensing | 3 | licence_agreements, licence_patents, royalty_payments |
| Cost Tracking | 2 | invoices, invoice_line_items |
| Audit | 1 | audit_log |
| Integration | 2 | patent_office_connections, sync_jobs |
| **Total** | **39** | |

---

## Key Design Decisions

1. **Separate tables for patents and trademarks** rather than a single "ip_assets" table, because the field sets, lifecycle states, and regulatory frameworks differ fundamentally between patent and trademark prosecution. A combined table would require excessive nullable columns.

2. **Party/role pattern with junction tables** (patent_parties, trademark_parties) instead of separate inventor, assignee, and counsel tables. A single party can play multiple roles across multiple IP assets, and role changes over time are tracked via `is_current` flags.

3. **Unified deadlines table** covering all deadline types (office actions, annuities, PCT phases, trademark renewals) rather than separate deadline tables per IP type. This enables a single dashboard query for "all upcoming deadlines" across the portfolio -- the most common operational query in any IPMS.

4. **ISO 3166-based jurisdictions as a reference table** with PCT/Paris/Madrid membership flags and jurisdiction-specific patent term and national phase deadline rules. This supports the multi-jurisdictional deadline cascade computation required for PCT applications.

5. **NICE Classification as a reference table** (NCL 13-2026, all 45 classes) linked to trademarks via a junction table, supporting multiple class registrations per trademark as required by the Madrid Protocol.

6. **AI scoring fields directly on annuity_decisions** rather than in a separate AI results table, because the decision record is the natural home for the data that informs the decision. Citation scores, business alignment, and licensing potential are displayed alongside each pay/abandon choice.

7. **Audit log with old/new value JSONB columns** rather than a separate history table per entity. This provides a single queryable audit trail across all entity types, aligned with NIST SP 800-92 requirements. Monthly partitioning is recommended for production deployments.

8. **Patent office credentials encrypted at rest** (BYTEA column for encrypted credentials) rather than stored as plaintext, supporting ISO 27001 Control 5.32 requirements for protecting IP-related system access.

9. **Documents linked to both patents and trademarks** via nullable foreign keys with a CHECK constraint ensuring at least one is populated. Document metadata includes AI-extracted data as JSONB for correspondence that has been processed by the automated docketing AI.

10. **Forward reference from patents to patent_families** enables INPADOC-style family grouping, critical for portfolio analytics and for computing which jurisdictions are covered by related applications.
