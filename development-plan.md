# IP Management Platform -- Development Plan

> Project: IP Management Platform (Candidate #95)
> Created: 2026-05-25
> Status: Planning

---

## Table of Contents

1. [Technology Decisions](#technology-decisions)
2. [Project Structure](#project-structure)
3. [Phase Dependency Graph](#phase-dependency-graph)
4. [Phase 1: Foundation & Core Data Model](#phase-1-foundation--core-data-model)
5. [Phase 2: Patent & Trademark Docketing](#phase-2-patent--trademark-docketing)
6. [Phase 3: Deadline Engine & Calendar](#phase-3-deadline-engine--calendar)
7. [Phase 4: Patent Office Integration](#phase-4-patent-office-integration)
8. [Phase 5: Document Management & AI Docketing](#phase-5-document-management--ai-docketing)
9. [Phase 6: Annuity Management & Decision Support](#phase-6-annuity-management--decision-support)
10. [Phase 7: Parties, Counsel & Cost Tracking](#phase-7-parties-counsel--cost-tracking)
11. [Phase 8: Prior Art Search & FTO Analysis](#phase-8-prior-art-search--fto-analysis)
12. [Phase 9: Invention Disclosures & Licensing](#phase-9-invention-disclosures--licensing)
13. [Phase 10: Portfolio Analytics & AI Scoring](#phase-10-portfolio-analytics--ai-scoring)
14. [Phase 11: Plugin Architecture & Integrations](#phase-11-plugin-architecture--integrations)
15. [Phase 12: Production Hardening & Compliance](#phase-12-production-hardening--compliance)
16. [Definition of Done](#definition-of-done)

---

## Technology Decisions

### Database: PostgreSQL with Hybrid Relational + JSONB (Data Model Suggestion 3)

**Rationale:** Data Model Suggestion 3 (Hybrid Relational + JSONB) is selected as the primary data model for the following reasons:

1. **Table count balance.** 16 tables vs. 39 in the fully normalized model (Suggestion 1). For a startup/OSS project, schema complexity directly correlates with migration burden and onboarding friction.

2. **Multi-jurisdiction flexibility.** IP management spans 150+ jurisdictions, each with unique rules, fee structures, and procedural requirements. Storing jurisdiction-specific rules, fee schedules, and AI-extracted data in JSONB columns avoids the alternative of hundreds of nullable columns or per-jurisdiction tables. New jurisdiction support requires a data insert, not a schema migration.

3. **Plugin architecture alignment.** The README specifies a plugin architecture for annuity services and patent office connectors. JSONB `config` columns on integration tables and `details` columns on operational records let plugins store provider-specific data without schema changes.

4. **AI-score evolution.** AI models will change frequently (new fields, new model versions). JSONB `ai_scores` columns absorb model evolution without migrations.

5. **MVP velocity.** The unified `ip_assets` table enables "all IP assets expiring this year" without UNION queries, and the reduced table count accelerates initial development.

**What we take from other suggestions:**
- From Suggestion 1: explicit `priority_claims` and `pct_national_phases` typed columns on the deadlines table where legal precision is non-negotiable.
- From Suggestion 2: audit log design inspired by event-sourcing principles -- JSONB diffs capturing old/new values on every change, aligned with NIST SP 800-92.
- From Suggestion 4: graph analytics concepts adopted as a later-phase read model (Phase 10) rather than a primary storage layer, avoiding dual-database operational complexity in early phases.

### Backend: Node.js (TypeScript) with Fastify

**Rationale:**
- TypeScript provides type safety for complex domain models (patent statuses, jurisdiction rules, deadline types) while maintaining the ecosystem breadth needed for API integrations (USPTO, EPO OPS, WIPO).
- Fastify offers superior performance over Express for the high-volume API calls required by patent office sync jobs, with built-in JSON Schema validation that aligns with the OpenAPI 3.1 specification requirement from standards.md.
- The `python-epo-ops-client` SDK exists for EPO OPS, but a TypeScript implementation allows a single-language backend; the EPO OPS REST API is straightforward to consume directly.

### Frontend: Next.js 15 (App Router) with React 19

**Rationale:**
- Server Components reduce client bundle size for the document-heavy, table-heavy UI patterns required by IP management dashboards.
- App Router's nested layouts support the multi-panel interfaces (deadline sidebar + patent detail + document viewer) described in Anaqua and AppColl UX patterns.
- React 19's form actions simplify the complex form workflows (patent filing, office action response, annuity decision batch review).

### Authentication: OpenID Connect (OIDC) via NextAuth.js v5

**Rationale:**
- Standards.md mandates OIDC for enterprise SSO (Microsoft Entra ID, Okta, Google Workspace).
- NextAuth.js v5 supports the OIDC providers required for enterprise adoption and provides a local credentials fallback for self-hosted deployments.

### AI Layer: Claude API via Anthropic SDK

**Rationale:**
- Patent office correspondence extraction, annuity decision scoring, and FTO claim mapping require long-context reasoning across technical documents.
- Claude's 200K context window accommodates full patent specifications (typically 20-60 pages) in a single call.
- MCP server integration (noted in standards.md as an emerging pattern via PatSnap) can expose portfolio data to AI agents.

### File Storage: S3-compatible (MinIO for self-hosted, AWS S3 for cloud)

**Rationale:**
- Patent prosecution files include large PDFs, images (drawings), and XML payloads. Object storage is the standard pattern.
- MinIO provides S3 API compatibility for self-hosted deployments without vendor lock-in.

### Search: PostgreSQL full-text search (initial), Typesense (later)

**Rationale:**
- PostgreSQL `tsvector` handles initial patent/trademark search needs without additional infrastructure.
- Typesense (or Meilisearch) can be added in Phase 10 for faceted analytics search across portfolio data.

### CI/CD: GitHub Actions

**Rationale:** Standard for OSS projects. Provides free CI for public repositories.

---

## Project Structure

```
ip-management-platform/
  apps/
    web/                          # Next.js 15 frontend
      src/
        app/
          (auth)/                 # Login, SSO callbacks
          (dashboard)/            # Main authenticated layout
            patents/              # Patent docketing views
            trademarks/           # Trademark management views
            deadlines/            # Deadline calendar & dashboard
            portfolio/            # Portfolio analytics & reporting
            documents/            # Document management
            parties/              # Inventors, counsel, assignees
            annuities/            # Annuity decision queue
            searches/             # Prior art & FTO
            disclosures/          # Invention disclosures
            licences/             # Licence agreements
            settings/             # Organisation, users, integrations
          api/                    # Next.js API routes (BFF layer)
        components/
          ui/                     # Shared UI components (shadcn/ui)
          patents/                # Patent-specific components
          deadlines/              # Deadline-specific components
          documents/              # Document viewer components
          analytics/              # Charts, visualisations
        lib/
          api-client.ts           # Typed API client
          auth.ts                 # NextAuth configuration
          utils.ts
    api/                          # Fastify backend API
      src/
        modules/
          patents/                # Patent domain (routes, service, repository)
          trademarks/             # Trademark domain
          deadlines/              # Deadline engine
          documents/              # Document management
          parties/                # Party management
          annuities/              # Annuity decision logic
          searches/               # Prior art & FTO
          disclosures/            # Invention disclosures
          licences/               # Licensing
          integrations/           # Patent office connectors
            uspto/                # USPTO API client
            epo/                  # EPO OPS client
            wipo/                 # WIPO PATENTSCOPE client
          ai/                     # AI service layer
            docketing/            # Correspondence extraction
            annuity-scoring/      # Pay/abandon scoring
            fto/                  # FTO claim mapping
            portfolio-scoring/    # Portfolio optimisation
          audit/                  # Audit log service
          auth/                   # Authentication & RBAC
        plugins/                  # Plugin interface for external services
          annuity-providers/      # Dennemeyer, Questel PAVIS, CPA Global
          patent-offices/         # Additional patent office connectors
        shared/
          database/
            migrations/           # Database migrations (node-pg-migrate)
            seeds/                # Jurisdiction data, NICE classes, CPC codes
            schema.ts             # Drizzle ORM schema definitions
          types/                  # Shared TypeScript types
          constants/              # Jurisdiction rules, deadline types
          validators/             # Zod schemas for API validation
  packages/
    deadline-engine/              # Standalone deadline computation library
    st96-parser/                  # WIPO ST.96 XML parser
    patent-office-types/          # Shared types for patent office APIs
  infrastructure/
    docker/
      docker-compose.yml          # PostgreSQL, MinIO, Redis
      Dockerfile.api
      Dockerfile.web
    k8s/                          # Kubernetes manifests (optional)
  docs/
    api/                          # OpenAPI 3.1 specification
    architecture/                 # Architecture decision records
```

---

## Phase Dependency Graph

```
Phase 1: Foundation & Core Data Model
  |
  +---> Phase 2: Patent & Trademark Docketing
  |       |
  |       +---> Phase 3: Deadline Engine & Calendar
  |       |       |
  |       |       +---> Phase 4: Patent Office Integration
  |       |       |       |
  |       |       |       +---> Phase 5: Document Management & AI Docketing
  |       |       |
  |       |       +---> Phase 6: Annuity Management & Decision Support
  |       |
  |       +---> Phase 7: Parties, Counsel & Cost Tracking
  |
  +---> Phase 8: Prior Art Search & FTO Analysis  (requires Phase 4)
  |
  +---> Phase 9: Invention Disclosures & Licensing  (requires Phase 2, 7)
  |
  +---> Phase 10: Portfolio Analytics & AI Scoring  (requires Phase 6, 8)
  |
  +---> Phase 11: Plugin Architecture & Integrations  (requires Phase 4, 6)
  |
  +---> Phase 12: Production Hardening & Compliance  (requires all prior phases)
```

**Critical path:** Phase 1 -> Phase 2 -> Phase 3 -> Phase 4 -> Phase 5

**Parallelisation opportunities:**
- Phase 7 can run in parallel with Phases 3-4 (different domain, shared foundation)
- Phase 8 can begin once Phase 4 delivers patent office API clients
- Phase 9 can begin once Phase 2 and Phase 7 are complete
- Phases 10-11 can run in parallel once their prerequisites are met

---

## Phase 1: Foundation & Core Data Model

**Goal:** Establish project infrastructure, database schema, authentication, multi-tenancy, and the base API layer.

**Duration:** 3-4 weeks

### Task 1.1: Project Scaffolding & Development Environment

**What:** Initialise the monorepo with Next.js 15, Fastify API server, PostgreSQL via Docker Compose, and CI pipeline.

**Design:**

```typescript
// apps/api/src/server.ts
import Fastify from 'fastify';
import cors from '@fastify/cors';
import jwt from '@fastify/jwt';
import swagger from '@fastify/swagger';
import swaggerUi from '@fastify/swagger-ui';

const app = Fastify({ logger: true });

await app.register(cors, { origin: true });
await app.register(jwt, { secret: process.env.JWT_SECRET! });
await app.register(swagger, {
  openapi: {
    info: {
      title: 'IP Management Platform API',
      version: '0.1.0',
    },
    servers: [{ url: 'http://localhost:3001' }],
  },
});
await app.register(swaggerUi, { routePrefix: '/docs' });

// Module registration
await app.register(import('./modules/auth/routes'), { prefix: '/api/auth' });
await app.register(import('./modules/patents/routes'), { prefix: '/api/patents' });

await app.listen({ port: 3001, host: '0.0.0.0' });
```

```yaml
# infrastructure/docker/docker-compose.yml
services:
  postgres:
    image: postgres:17
    environment:
      POSTGRES_DB: ipms
      POSTGRES_USER: ipms
      POSTGRES_PASSWORD: ipms_dev
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data

  minio:
    image: minio/minio:latest
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin
    ports:
      - "9000:9000"
      - "9001:9001"
    volumes:
      - miniodata:/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

volumes:
  pgdata:
  miniodata:
```

**Testing:**
- `docker compose up` starts all services without error.
- `curl http://localhost:3001/docs` returns Swagger UI.
- `curl http://localhost:3000` returns Next.js landing page.
- GitHub Actions workflow runs lint, type-check, and unit tests on push.

---

### Task 1.2: Database Schema -- Core Tables

**What:** Create the foundational database tables: organisations, users, jurisdictions, and the audit log. Implement database migrations using node-pg-migrate.

**Design:**

```sql
-- migrations/001_core_tables.sql

CREATE TABLE organisations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    org_type        TEXT NOT NULL CHECK (org_type IN (
        'corporation', 'law_firm', 'university', 'government', 'individual'
    )),
    country_code    CHAR(2) NOT NULL,
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    email           TEXT NOT NULL,
    display_name    TEXT NOT NULL,
    role            TEXT NOT NULL CHECK (role IN (
        'admin', 'ip_counsel', 'paralegal', 'inventor', 'viewer'
    )),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    preferences     JSONB NOT NULL DEFAULT '{}',
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
    currency_code   CHAR(3),
    rules           JSONB NOT NULL DEFAULT '{}',
    fee_schedule    JSONB NOT NULL DEFAULT '[]',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

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

**Testing:**
- Migration runs forward and backward without error: `npm run migrate up && npm run migrate down && npm run migrate up`.
- Insert sample organisation, user, and jurisdiction rows; verify constraints reject invalid `org_type`, `role`, and `country_code` values.
- Verify `audit_log` indexes exist via `\d audit_log` in psql.
- Seed script populates all 195 ISO 3166-1 jurisdictions with PCT/Paris/Madrid membership flags and rules JSONB.

---

### Task 1.3: Authentication & RBAC

**What:** Implement OIDC-based authentication via NextAuth.js v5, JWT-based API authentication, and role-based access control middleware.

**Design:**

```typescript
// apps/web/src/lib/auth.ts
import NextAuth from 'next-auth';
import CredentialsProvider from 'next-auth/providers/credentials';
import MicrosoftEntraID from 'next-auth/providers/microsoft-entra-id';
import GoogleProvider from 'next-auth/providers/google';

export const { handlers, auth, signIn, signOut } = NextAuth({
  providers: [
    CredentialsProvider({
      // Local username/password for self-hosted deployments
      async authorize(credentials) {
        // Validate against users table
      },
    }),
    MicrosoftEntraID({
      clientId: process.env.AZURE_AD_CLIENT_ID!,
      clientSecret: process.env.AZURE_AD_CLIENT_SECRET!,
      tenantId: process.env.AZURE_AD_TENANT_ID!,
    }),
    GoogleProvider({
      clientId: process.env.GOOGLE_CLIENT_ID!,
      clientSecret: process.env.GOOGLE_CLIENT_SECRET!,
    }),
  ],
  callbacks: {
    async jwt({ token, user }) {
      if (user) {
        token.organisationId = user.organisationId;
        token.role = user.role;
      }
      return token;
    },
    async session({ session, token }) {
      session.user.organisationId = token.organisationId;
      session.user.role = token.role;
      return session;
    },
  },
});

// apps/api/src/modules/auth/rbac.ts
export type Role = 'admin' | 'ip_counsel' | 'paralegal' | 'inventor' | 'viewer';

const PERMISSIONS: Record<string, Role[]> = {
  'patents:create':  ['admin', 'ip_counsel', 'paralegal'],
  'patents:read':    ['admin', 'ip_counsel', 'paralegal', 'inventor', 'viewer'],
  'patents:update':  ['admin', 'ip_counsel', 'paralegal'],
  'patents:delete':  ['admin'],
  'deadlines:complete': ['admin', 'ip_counsel', 'paralegal'],
  'annuity:decide':  ['admin', 'ip_counsel'],
  'admin:manage':    ['admin'],
};

export function requirePermission(permission: string) {
  return async (request: FastifyRequest, reply: FastifyReply) => {
    const user = request.user;
    const allowedRoles = PERMISSIONS[permission];
    if (!allowedRoles?.includes(user.role)) {
      return reply.code(403).send({ error: 'Insufficient permissions' });
    }
  };
}
```

**Testing:**
- Login with local credentials returns a valid JWT with `organisationId` and `role` claims.
- OIDC login flow with a mock provider (using `next-auth/providers/credentials` in test mode) creates a user record and returns a session.
- API requests without a valid JWT return 401.
- API requests with a `viewer` role to `POST /api/patents` return 403.
- API requests with an `ip_counsel` role to `POST /api/patents` return 201.
- Multi-tenancy: user from Org A cannot access patents from Org B (returns 404, not 403, to prevent information leakage).

---

### Task 1.4: Multi-Tenancy & Organisation Management

**What:** Implement organisation CRUD, user invitation flow, and tenant-scoped data access middleware.

**Design:**

```typescript
// apps/api/src/shared/middleware/tenant-scope.ts
export async function tenantScope(request: FastifyRequest, reply: FastifyReply) {
  const organisationId = request.user.organisationId;
  if (!organisationId) {
    return reply.code(401).send({ error: 'No organisation context' });
  }
  // Attach to request for use in repositories
  request.organisationId = organisationId;
}

// apps/api/src/modules/patents/repository.ts
export class PatentRepository {
  constructor(private db: Pool) {}

  async findAll(organisationId: string, filters: PatentFilters) {
    const query = sql`
      SELECT * FROM ip_assets
      WHERE organisation_id = ${organisationId}
        AND asset_type = 'patent'
      ORDER BY filing_date DESC
    `;
    return this.db.query(query);
  }
}
```

**Testing:**
- Create two organisations via API; verify each has isolated data.
- User in Org A creates a patent; user in Org B queries patents and gets an empty list.
- Organisation settings JSONB stores and retrieves custom deadline alert days, default currency, and enabled jurisdictions.
- Admin user can invite new users to their organisation; invited user receives role assignment.

---

## Phase 2: Patent & Trademark Docketing

**Goal:** Implement the core IP asset management -- creating, reading, updating, and searching patent and trademark records.

**Duration:** 3-4 weeks

**Dependencies:** Phase 1

### Task 2.1: IP Assets Table & API

**What:** Create the unified `ip_assets` table and implement CRUD API endpoints for patents and trademarks.

**Design:**

```sql
-- migrations/002_ip_assets.sql

CREATE TABLE ip_assets (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    asset_type          TEXT NOT NULL CHECK (asset_type IN ('patent', 'trademark', 'design')),
    application_number  TEXT NOT NULL,
    registration_number TEXT,
    title               TEXT NOT NULL,
    filing_date         DATE NOT NULL,
    registration_date   DATE,
    expiry_date         DATE,
    priority_date       DATE,
    jurisdiction_code   CHAR(2) NOT NULL REFERENCES jurisdictions(country_code),
    status              TEXT NOT NULL,
    family_id           UUID,
    cost_to_date        NUMERIC(12,2) DEFAULT 0,
    notes               TEXT,
    patent_data         JSONB,
    trademark_data      JSONB,
    jurisdiction_data   JSONB,
    ai_scores           JSONB,
    sync_data           JSONB,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ip_assets_org ON ip_assets(organisation_id);
CREATE INDEX idx_ip_assets_type ON ip_assets(asset_type);
CREATE INDEX idx_ip_assets_jurisdiction ON ip_assets(jurisdiction_code);
CREATE INDEX idx_ip_assets_status ON ip_assets(status);
CREATE INDEX idx_ip_assets_filing_date ON ip_assets(filing_date);
CREATE INDEX idx_ip_assets_app_number ON ip_assets(application_number);
CREATE INDEX idx_ip_assets_family ON ip_assets(family_id);
CREATE INDEX idx_ip_assets_patent_data ON ip_assets USING GIN (patent_data jsonb_path_ops)
    WHERE asset_type = 'patent';
CREATE INDEX idx_ip_assets_trademark_data ON ip_assets USING GIN (trademark_data jsonb_path_ops)
    WHERE asset_type = 'trademark';
```

```typescript
// apps/api/src/modules/patents/routes.ts
import { FastifyPluginAsync } from 'fastify';
import { z } from 'zod';

const CreatePatentSchema = z.object({
  applicationNumber: z.string().min(1),
  title: z.string().min(1),
  filingDate: z.string().date(),
  jurisdictionCode: z.string().length(2),
  status: z.enum([
    'provisional', 'pending', 'published', 'under_examination',
    'allowed', 'granted', 'expired', 'abandoned', 'withdrawn', 'lapsed',
  ]),
  patentType: z.enum([
    'utility', 'design', 'plant', 'provisional', 'pct',
    'continuation', 'continuation_in_part', 'divisional', 'reissue',
  ]),
  abstract: z.string().optional(),
  pctApplicationNumber: z.string().optional(),
  priorityDate: z.string().date().optional(),
  classifications: z.array(z.object({
    type: z.enum(['cpc', 'ipc', 'uspc']),
    code: z.string(),
    primary: z.boolean().default(false),
  })).optional(),
  claims: z.array(z.object({
    number: z.number().int().positive(),
    type: z.enum(['independent', 'dependent']),
    dependsOn: z.number().int().positive().optional(),
    text: z.string(),
  })).optional(),
});

const routes: FastifyPluginAsync = async (app) => {
  app.post('/', {
    preHandler: [requirePermission('patents:create')],
    handler: async (request, reply) => {
      const body = CreatePatentSchema.parse(request.body);
      const patent = await patentService.create(request.organisationId, body);
      await auditService.log(request, 'create', 'patent', patent.id);
      return reply.code(201).send(patent);
    },
  });

  app.get('/', {
    preHandler: [requirePermission('patents:read')],
    handler: async (request, reply) => {
      const filters = PatentFilterSchema.parse(request.query);
      const patents = await patentService.findAll(request.organisationId, filters);
      return patents;
    },
  });

  app.get('/:id', {
    preHandler: [requirePermission('patents:read')],
    handler: async (request, reply) => {
      const patent = await patentService.findById(
        request.organisationId,
        request.params.id
      );
      if (!patent) return reply.code(404).send({ error: 'Patent not found' });
      return patent;
    },
  });
};
```

**Testing:**
- `POST /api/patents` with valid data creates a patent and returns 201 with the patent record including `id`.
- `POST /api/patents` with missing `applicationNumber` returns 400 with validation error.
- `POST /api/patents` with `jurisdictionCode: "ZZ"` (non-existent) returns 400 (foreign key violation).
- `GET /api/patents` returns paginated list filtered by `organisationId`.
- `GET /api/patents?status=granted&jurisdictionCode=US` returns only granted US patents.
- `GET /api/patents/:id` returns patent with `patent_data` JSONB including classifications and claims.
- Patent status transitions are validated: cannot move from `granted` to `pending`.
- Audit log records the creation with `entity_type: 'patent'` and the full JSONB payload in `changes.new`.

---

### Task 2.2: Patent Family Management

**What:** Implement patent family grouping, enabling users to link related applications (PCT, national phases, continuations, divisionals) into family trees.

**Design:**

```typescript
// apps/api/src/modules/patents/family-service.ts
export class PatentFamilyService {
  async createFamily(organisationId: string, data: {
    familyName?: string;
    memberIds: string[];
  }) {
    const familyId = uuid();
    await this.db.query(sql`
      UPDATE ip_assets
      SET family_id = ${familyId}, updated_at = now()
      WHERE id = ANY(${data.memberIds})
        AND organisation_id = ${organisationId}
    `);
    return { familyId, memberCount: data.memberIds.length };
  }

  async getFamilyMembers(organisationId: string, familyId: string) {
    return this.db.query(sql`
      SELECT id, title, application_number, jurisdiction_code,
             status, filing_date, patent_data->>'patent_type' AS patent_type
      FROM ip_assets
      WHERE family_id = ${familyId}
        AND organisation_id = ${organisationId}
      ORDER BY filing_date ASC
    `);
  }

  async getEarliestPriorityDate(familyId: string): Promise<Date | null> {
    const result = await this.db.query(sql`
      SELECT MIN(priority_date) AS earliest_priority_date
      FROM ip_assets
      WHERE family_id = ${familyId}
        AND priority_date IS NOT NULL
    `);
    return result.rows[0]?.earliest_priority_date;
  }
}
```

**Testing:**
- Create 3 patents (US provisional, US utility, EP national phase); group into a family.
- `GET /api/patents/families/:familyId` returns all 3 members ordered by filing date.
- Adding a patent to a family updates its `family_id` and produces an audit log entry.
- Removing a patent from a family (setting `family_id = null`) is tracked.
- Earliest priority date is correctly computed across family members.

---

### Task 2.3: Trademark CRUD & NICE Classification

**What:** Implement trademark-specific endpoints, NICE classification reference data, and Madrid Protocol data structures.

**Design:**

```typescript
// apps/api/src/modules/trademarks/routes.ts
const CreateTrademarkSchema = z.object({
  applicationNumber: z.string().min(1),
  title: z.string().min(1),
  filingDate: z.string().date(),
  jurisdictionCode: z.string().length(2),
  status: z.enum([
    'pending', 'published', 'opposed', 'registered',
    'renewed', 'expired', 'cancelled', 'withdrawn',
  ]),
  markText: z.string().optional(),
  markType: z.enum([
    'word', 'figurative', 'combined', 'three_dimensional',
    'sound', 'colour', 'motion', 'hologram', 'other',
  ]),
  niceClasses: z.array(z.number().int().min(1).max(45)),
  goodsServices: z.record(z.string(), z.string()).optional(),
  madridRegistration: z.object({
    internationalRegNumber: z.string(),
    registrationDate: z.string().date().optional(),
    renewalDue: z.string().date().optional(),
    designations: z.array(z.object({
      jurisdiction: z.string().length(2),
      status: z.enum(['pending', 'protected', 'refused', 'withdrawn']),
      protectionDate: z.string().date().optional(),
    })).optional(),
  }).optional(),
});
```

```sql
-- Seed NICE classification reference data
INSERT INTO schema_definitions (schema_name, schema_version, json_schema, description)
VALUES ('nice_classes', 1, '{
  "type": "array",
  "items": {
    "type": "object",
    "properties": {
      "classNumber": {"type": "integer", "minimum": 1, "maximum": 45},
      "classType": {"enum": ["goods", "services"]},
      "heading": {"type": "string"}
    }
  }
}', 'NICE Classification (NCL 13-2026) reference data');
```

**Testing:**
- `POST /api/trademarks` creates a trademark with NICE classes stored in `trademark_data.nice_classes`.
- Validation rejects NICE class numbers outside 1-45.
- Query `GET /api/trademarks?niceClass=9` returns trademarks containing class 9 (JSONB containment query).
- Madrid Protocol registration data is stored and retrieved with designation statuses.
- `GET /api/reference/nice-classes` returns all 45 classes with headings and goods/services type.

---

### Task 2.4: Patent & Trademark Search

**What:** Implement full-text search across IP assets using PostgreSQL `tsvector`.

**Design:**

```sql
-- migrations/003_search.sql
ALTER TABLE ip_assets ADD COLUMN search_vector tsvector
    GENERATED ALWAYS AS (
        setweight(to_tsvector('english', coalesce(title, '')), 'A') ||
        setweight(to_tsvector('english', coalesce(application_number, '')), 'A') ||
        setweight(to_tsvector('english', coalesce(registration_number, '')), 'A') ||
        setweight(to_tsvector('english', coalesce(notes, '')), 'C')
    ) STORED;

CREATE INDEX idx_ip_assets_search ON ip_assets USING GIN (search_vector);
```

```typescript
// apps/api/src/modules/search/service.ts
export class SearchService {
  async search(organisationId: string, query: string, filters?: {
    assetType?: 'patent' | 'trademark';
    jurisdictionCode?: string;
    status?: string;
  }) {
    return this.db.query(sql`
      SELECT id, asset_type, title, application_number, jurisdiction_code,
             status, filing_date,
             ts_rank(search_vector, websearch_to_tsquery('english', ${query})) AS rank
      FROM ip_assets
      WHERE organisation_id = ${organisationId}
        AND search_vector @@ websearch_to_tsquery('english', ${query})
        ${filters?.assetType ? sql`AND asset_type = ${filters.assetType}` : sql``}
        ${filters?.jurisdictionCode ? sql`AND jurisdiction_code = ${filters.jurisdictionCode}` : sql``}
        ${filters?.status ? sql`AND status = ${filters.status}` : sql``}
      ORDER BY rank DESC
      LIMIT 50
    `);
  }
}
```

**Testing:**
- Searching "quantum error correction" returns patents with that phrase in the title ranked highest.
- Searching by application number "US17/123456" returns the matching patent.
- Search respects tenant isolation: results only include the requesting organisation's assets.
- Filters combine with search: `query=quantum&assetType=patent&status=granted` returns only granted patents matching "quantum".
- Empty search query with filters returns filtered results ordered by filing date.

---

### Task 2.5: Frontend -- Patent List & Detail Views

**What:** Build the patent dashboard with list view, detail view, and creation form using Next.js App Router and shadcn/ui.

**Design:**

```typescript
// apps/web/src/app/(dashboard)/patents/page.tsx
import { DataTable } from '@/components/ui/data-table';
import { columns } from './columns';

export default async function PatentsPage({
  searchParams,
}: {
  searchParams: { status?: string; jurisdiction?: string; q?: string };
}) {
  const patents = await fetchPatents(searchParams);

  return (
    <div className="space-y-4">
      <div className="flex items-center justify-between">
        <h1 className="text-2xl font-bold">Patent Portfolio</h1>
        <Button asChild>
          <Link href="/patents/new">Add Patent</Link>
        </Button>
      </div>

      <PatentFilters />

      <DataTable
        columns={columns}
        data={patents.items}
        pagination={patents.pagination}
      />
    </div>
  );
}

// apps/web/src/app/(dashboard)/patents/[id]/page.tsx
export default async function PatentDetailPage({
  params,
}: {
  params: { id: string };
}) {
  const patent = await fetchPatent(params.id);

  return (
    <div className="grid grid-cols-3 gap-6">
      <div className="col-span-2">
        <PatentHeader patent={patent} />
        <Tabs defaultValue="details">
          <TabsList>
            <TabsTrigger value="details">Details</TabsTrigger>
            <TabsTrigger value="claims">Claims</TabsTrigger>
            <TabsTrigger value="deadlines">Deadlines</TabsTrigger>
            <TabsTrigger value="documents">Documents</TabsTrigger>
            <TabsTrigger value="family">Family</TabsTrigger>
            <TabsTrigger value="citations">Citations</TabsTrigger>
          </TabsList>
          <TabsContent value="details"><PatentDetails patent={patent} /></TabsContent>
          <TabsContent value="claims"><PatentClaims claims={patent.patentData?.claims} /></TabsContent>
          {/* ... */}
        </Tabs>
      </div>
      <div>
        <PatentSidebar patent={patent} />
      </div>
    </div>
  );
}
```

**Testing:**
- Patent list page loads with a table showing title, application number, jurisdiction, status, and filing date.
- Clicking a patent row navigates to the detail page at `/patents/:id`.
- Detail page shows all fields including JSONB data (classifications, claims) in formatted sections.
- "Add Patent" form validates required fields client-side and shows server-side errors.
- Patent list supports sorting by column and pagination.
- Status filter dropdown shows only statuses that exist in the current portfolio.

---

## Phase 3: Deadline Engine & Calendar

**Goal:** Build the multi-jurisdictional deadline computation engine -- the most legally critical component of the platform.

**Duration:** 4-5 weeks

**Dependencies:** Phase 2

### Task 3.1: Deadlines Table & Core CRUD

**What:** Create the deadlines table and implement CRUD endpoints for manual deadline creation and management.

**Design:**

```sql
-- migrations/004_deadlines.sql

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
    annuity_decision    JSONB,
    details             JSONB,
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

**Testing:**
- Create a deadline linked to a patent; verify foreign key constraint works.
- Mark a deadline as `completed`; verify `completed_at` and `completed_by` are set.
- Query `GET /api/deadlines?status=upcoming&dueBefore=2026-09-01` returns upcoming deadlines before the date.
- Attempting to complete a deadline that is already `completed` returns 409 Conflict.
- Audit log captures every deadline status change with old and new values.

---

### Task 3.2: Deadline Computation Engine (Standalone Package)

**What:** Build the `deadline-engine` package as a standalone, pure-function library that computes deadlines from patent/trademark data and jurisdiction rules. This is extracted as a package because it must be independently testable with high coverage -- deadline errors can cause irreversible loss of patent rights.

**Design:**

```typescript
// packages/deadline-engine/src/index.ts

export interface JurisdictionRules {
  pctNationalPhaseMonths: number;     // 30, 31, or 34
  patentTermYears: number;
  maintenanceFeeYears?: number[];     // e.g., [4, 8, 12] for US
  annuityStartYear?: number;          // 1 for JP, 4 for US
  officeActionResponseMonths: number;
  extensionAvailable?: boolean;
  maxExtensions?: number;
  gracePeriodMonths?: number;
  gracePeriodSurchargePercent?: number;
}

export interface DeadlineInput {
  filingDate: Date;
  priorityDate?: Date;
  grantDate?: Date;
  jurisdictionCode: string;
  rules: JurisdictionRules;
  patentType: string;
}

export interface ComputedDeadline {
  deadlineType: string;
  dueDate: Date;
  gracePeriodEnd?: Date;
  jurisdictionCode: string;
  feeAmount?: number;
  feeCurrency?: string;
  description: string;
}

// Paris Convention: 12 months from priority date
export function computePriorityDeadline(priorityDate: Date): ComputedDeadline {
  const dueDate = addMonths(priorityDate, 12);
  return {
    deadlineType: 'priority_claim',
    dueDate,
    description: 'Paris Convention 12-month priority window expires',
  };
}

// PCT National Phase: 30/31/34 months from priority date
export function computePCTNationalPhaseDeadlines(
  priorityDate: Date,
  targetJurisdictions: Array<{ code: string; rules: JurisdictionRules }>
): ComputedDeadline[] {
  return targetJurisdictions.map(jurisdiction => ({
    deadlineType: 'pct_national_phase',
    dueDate: addMonths(priorityDate, jurisdiction.rules.pctNationalPhaseMonths),
    jurisdictionCode: jurisdiction.code,
    description: `PCT national phase entry deadline for ${jurisdiction.code}`,
  }));
}

// US Maintenance Fees: years 4, 8, 12 from grant date
export function computeUSMaintenanceFees(
  grantDate: Date,
  feeSchedule: Array<{ year: number; amount: number; currency: string }>
): ComputedDeadline[] {
  return feeSchedule.map(fee => {
    const dueDate = addYears(grantDate, fee.year);
    const gracePeriodEnd = addMonths(dueDate, 6);
    return {
      deadlineType: 'maintenance_fee',
      dueDate,
      gracePeriodEnd,
      jurisdictionCode: 'US',
      feeAmount: fee.amount,
      feeCurrency: fee.currency,
      description: `US maintenance fee year ${fee.year}`,
    };
  });
}

// Annuity deadlines: varies by jurisdiction
export function computeAnnuityDeadlines(
  filingDate: Date,
  grantDate: Date | undefined,
  rules: JurisdictionRules,
  feeSchedule: Array<{ year: number; amount: number; currency: string }>
): ComputedDeadline[] {
  const startYear = rules.annuityStartYear ?? 1;
  const endYear = rules.patentTermYears;
  const baseDate = grantDate ?? filingDate;

  return feeSchedule
    .filter(fee => fee.year >= startYear && fee.year <= endYear)
    .map(fee => ({
      deadlineType: 'annuity_payment',
      dueDate: addYears(baseDate, fee.year),
      gracePeriodEnd: rules.gracePeriodMonths
        ? addMonths(addYears(baseDate, fee.year), rules.gracePeriodMonths)
        : undefined,
      jurisdictionCode: undefined,
      feeAmount: fee.amount,
      feeCurrency: fee.currency,
      description: `Annual patent fee year ${fee.year}`,
    }));
}
```

**Testing:**
- **Paris Convention:** `computePriorityDeadline(new Date('2025-03-15'))` returns `dueDate: 2026-03-15`.
- **PCT US (30 months):** priority date 2025-01-01 -> national phase deadline 2027-07-01.
- **PCT EP (31 months):** priority date 2025-01-01 -> national phase deadline 2027-08-01.
- **US Maintenance Fees:** grant date 2026-06-01 -> year 4 due 2030-06-01, year 8 due 2034-06-01, year 12 due 2038-06-01, each with 6-month grace period.
- **JP Annuities:** filing date 2025-01-01, annuity start year 1 -> 20 annual deadlines.
- **Edge case:** PCT filed on Feb 29 (leap year) -> national phase deadline handles month-end correctly (no Feb 30).
- **Edge case:** priority date Dec 31 -> 12-month deadline Dec 31 next year (not Jan 1).
- All functions are pure (no database calls) and covered at >95% branch coverage.

---

### Task 3.3: Automated Deadline Generation

**What:** When a patent or trademark is created or updated, automatically generate applicable deadlines based on jurisdiction rules.

**Design:**

```typescript
// apps/api/src/modules/deadlines/auto-generator.ts
export class DeadlineAutoGenerator {
  async onPatentCreated(patent: IPAsset) {
    const jurisdiction = await this.jurisdictionRepo.findByCode(
      patent.jurisdictionCode
    );
    const rules = jurisdiction.rules as JurisdictionRules;
    const deadlines: ComputedDeadline[] = [];

    // Paris Convention priority deadline (if priority date exists)
    if (patent.priorityDate) {
      deadlines.push(computePriorityDeadline(patent.priorityDate));
    }

    // PCT national phase deadlines (if PCT application)
    if (patent.patentData?.patentType === 'pct') {
      const pctJurisdictions = await this.jurisdictionRepo.findPCTMembers();
      deadlines.push(
        ...computePCTNationalPhaseDeadlines(
          patent.priorityDate ?? patent.filingDate,
          pctJurisdictions
        )
      );
    }

    // Maintenance/annuity fees (if granted)
    if (patent.registrationDate) {
      const feeSchedule = jurisdiction.feeSchedule;
      deadlines.push(
        ...computeAnnuityDeadlines(
          patent.filingDate,
          patent.registrationDate,
          rules,
          feeSchedule
        )
      );
    }

    // Persist computed deadlines
    for (const deadline of deadlines) {
      await this.deadlineRepo.create({
        organisationId: patent.organisationId,
        ipAssetId: patent.id,
        ...deadline,
        status: 'upcoming',
      });
    }

    return deadlines;
  }
}
```

**Testing:**
- Creating a US utility patent with a priority date auto-generates a Paris Convention priority deadline.
- Creating a PCT application auto-generates national phase entry deadlines for all PCT member jurisdictions.
- Creating a granted US patent auto-generates 3 maintenance fee deadlines (years 4, 8, 12).
- Creating a granted JP patent auto-generates 20 annual annuity deadlines.
- Updating a patent's status to `granted` (adding `registrationDate`) triggers generation of maintenance/annuity deadlines.
- No duplicate deadlines are created if the patent is updated multiple times.

---

### Task 3.4: Deadline Dashboard & Calendar UI

**What:** Build the deadline dashboard with list view, calendar view, and alert indicators.

**Design:**

```typescript
// apps/web/src/app/(dashboard)/deadlines/page.tsx
export default async function DeadlinesPage() {
  const deadlines = await fetchDeadlines({
    status: ['upcoming', 'due_soon', 'overdue'],
    limit: 100,
  });

  const urgencyBuckets = {
    overdue: deadlines.filter(d => d.status === 'overdue'),
    thisWeek: deadlines.filter(d => isThisWeek(d.dueDate)),
    thisMonth: deadlines.filter(d => isThisMonth(d.dueDate)),
    upcoming: deadlines.filter(d => isAfterThisMonth(d.dueDate)),
  };

  return (
    <div className="space-y-6">
      <DeadlineStats deadlines={deadlines} />

      <Tabs defaultValue="list">
        <TabsList>
          <TabsTrigger value="list">List</TabsTrigger>
          <TabsTrigger value="calendar">Calendar</TabsTrigger>
        </TabsList>

        <TabsContent value="list">
          {urgencyBuckets.overdue.length > 0 && (
            <DeadlineGroup
              title="Overdue"
              deadlines={urgencyBuckets.overdue}
              variant="destructive"
            />
          )}
          <DeadlineGroup
            title="This Week"
            deadlines={urgencyBuckets.thisWeek}
            variant="warning"
          />
          <DeadlineGroup
            title="This Month"
            deadlines={urgencyBuckets.thisMonth}
          />
          <DeadlineGroup
            title="Upcoming"
            deadlines={urgencyBuckets.upcoming}
          />
        </TabsContent>

        <TabsContent value="calendar">
          <DeadlineCalendar deadlines={deadlines} />
        </TabsContent>
      </Tabs>
    </div>
  );
}
```

**Testing:**
- Dashboard loads with deadlines grouped by urgency (overdue, this week, this month, upcoming).
- Overdue deadlines show red visual indicator.
- Calendar view renders deadlines on their due dates.
- Clicking a deadline opens a detail panel with the linked patent/trademark information.
- Completing a deadline from the dashboard updates its status and shows a success notification.
- Deadline counts in the navigation sidebar update in real time.

---

### Task 3.5: Deadline Notifications & Alerts

**What:** Implement configurable deadline alerts via email and in-app notifications.

**Design:**

```typescript
// apps/api/src/modules/deadlines/notification-service.ts
export class DeadlineNotificationService {
  // Run daily via cron job
  async checkAndNotify() {
    const orgs = await this.orgRepo.findAll();

    for (const org of orgs) {
      const alertDays = org.settings.deadline_alert_days ?? [90, 60, 30, 14, 7];

      for (const days of alertDays) {
        const targetDate = addDays(new Date(), days);
        const deadlines = await this.deadlineRepo.findByDueDate(
          org.id,
          targetDate
        );

        for (const deadline of deadlines) {
          if (deadline.assignedTo) {
            await this.sendAlert(deadline, days);
          }
        }
      }
    }
  }

  private async sendAlert(deadline: Deadline, daysUntilDue: number) {
    // In-app notification
    await this.notificationRepo.create({
      userId: deadline.assignedTo,
      type: 'deadline_alert',
      title: `Deadline in ${daysUntilDue} days`,
      message: `${deadline.deadlineType} for ${deadline.assetTitle} due ${deadline.dueDate}`,
      data: { deadlineId: deadline.id, ipAssetId: deadline.ipAssetId },
    });

    // Email notification
    await this.emailService.send({
      to: deadline.assignedToEmail,
      subject: `[IPMS] Deadline in ${daysUntilDue} days: ${deadline.assetTitle}`,
      template: 'deadline-alert',
      data: { deadline, daysUntilDue },
    });
  }
}
```

**Testing:**
- A deadline due in exactly 30 days triggers an alert when the alert schedule includes 30.
- Alerts are sent both as in-app notifications and emails.
- Organisation-specific alert schedules are respected (org with `[60, 30, 7]` does not get 14-day alerts).
- Completed deadlines do not trigger alerts.
- A user with multiple deadlines on the same date receives a single batched email.
- Alert deduplication: the same deadline/day combination does not produce duplicate notifications.

---

## Phase 4: Patent Office Integration

**Goal:** Connect to USPTO and EPO OPS APIs for automated patent status updates and correspondence import.

**Duration:** 3-4 weeks

**Dependencies:** Phase 3

### Task 4.1: USPTO API Client

**What:** Implement a client for the USPTO Open Data Portal APIs (Patent File Wrapper, TSDR).

**Design:**

```typescript
// packages/patent-office-types/src/uspto.ts
export interface USPTOPatentFileWrapper {
  applicationNumber: string;
  applicationStatusCode: string;
  applicationStatusDate: string;
  patentNumber?: string;
  filingDate: string;
  inventorNames: string[];
  assigneeName?: string;
  artUnit?: string;
  examinerName?: string;
}

// apps/api/src/modules/integrations/uspto/client.ts
export class USPTOClient {
  private baseUrl = 'https://data.uspto.gov/apis';

  async getPatentFileWrapper(applicationNumber: string): Promise<USPTOPatentFileWrapper> {
    const response = await fetch(
      `${this.baseUrl}/patent-file-wrapper/search?applicationNumber=${applicationNumber}`,
      { headers: { 'X-API-Key': this.apiKey } }
    );
    if (!response.ok) throw new USPTOApiError(response.status, await response.text());
    return response.json();
  }

  async getDocuments(applicationNumber: string): Promise<USPTODocument[]> {
    const response = await fetch(
      `${this.baseUrl}/patent-file-wrapper/documents?applicationNumber=${applicationNumber}`,
      { headers: { 'X-API-Key': this.apiKey } }
    );
    return response.json();
  }

  async getTrademarkStatus(serialNumber: string): Promise<USPTOTrademarkStatus> {
    const response = await fetch(
      `${this.baseUrl}/tsdr/${serialNumber}`,
      { headers: { 'X-API-Key': this.apiKey } }
    );
    return response.json();
  }
}

// apps/api/src/modules/integrations/uspto/sync-service.ts
export class USPTOSyncService {
  async syncPatent(organisationId: string, ipAssetId: string) {
    const patent = await this.patentRepo.findById(organisationId, ipAssetId);
    const fileWrapper = await this.usptoClient.getPatentFileWrapper(
      patent.applicationNumber
    );

    const updates: Partial<IPAsset> = {};
    if (fileWrapper.patentNumber && !patent.registrationNumber) {
      updates.registrationNumber = fileWrapper.patentNumber;
    }
    if (fileWrapper.applicationStatusCode !== patent.status) {
      updates.status = this.mapUSPTOStatus(fileWrapper.applicationStatusCode);
    }

    updates.syncData = {
      lastSyncAt: new Date().toISOString(),
      source: 'uspto',
      externalId: fileWrapper.applicationNumber,
      syncStatus: 'current',
    };

    if (Object.keys(updates).length > 0) {
      await this.patentRepo.update(organisationId, ipAssetId, updates);
      await this.auditService.log('sync_update', 'patent', ipAssetId, updates);
    }
  }
}
```

**Testing:**
- USPTO client fetches file wrapper data for a known application number (mock API in tests, real API in integration tests).
- Sync service updates local patent status when USPTO reports a status change.
- Sync service populates `registrationNumber` when USPTO reports a patent grant.
- Rate limiting is respected: client backs off after 429 responses.
- Network errors are retried with exponential backoff (max 3 retries).
- Sync status (`syncData.lastSyncAt`, `syncData.syncStatus`) is updated after each sync.

---

### Task 4.2: EPO OPS API Client

**What:** Implement a client for the European Patent Office Open Patent Services (OPS v3.2) API with OAuth 2.0 authentication.

**Design:**

```typescript
// apps/api/src/modules/integrations/epo/client.ts
export class EPOOPSClient {
  private baseUrl = 'https://ops.epo.org/3.2/rest-services';
  private tokenUrl = 'https://ops.epo.org/3.2/auth/accesstoken';
  private accessToken: string | null = null;
  private tokenExpiry: Date | null = null;

  constructor(
    private clientId: string,
    private clientSecret: string
  ) {}

  private async authenticate(): Promise<string> {
    if (this.accessToken && this.tokenExpiry && this.tokenExpiry > new Date()) {
      return this.accessToken;
    }

    const credentials = Buffer.from(
      `${this.clientId}:${this.clientSecret}`
    ).toString('base64');

    const response = await fetch(this.tokenUrl, {
      method: 'POST',
      headers: {
        'Authorization': `Basic ${credentials}`,
        'Content-Type': 'application/x-www-form-urlencoded',
      },
      body: 'grant_type=client_credentials',
    });

    const data = await response.json();
    this.accessToken = data.access_token;
    this.tokenExpiry = new Date(Date.now() + data.expires_in * 1000);
    return this.accessToken;
  }

  async getBibliographic(epodocNumber: string): Promise<EPOBibliographic> {
    const token = await this.authenticate();
    const response = await fetch(
      `${this.baseUrl}/published-data/publication/epodoc/${epodocNumber}/biblio`,
      { headers: { 'Authorization': `Bearer ${token}`, 'Accept': 'application/json' } }
    );
    return response.json();
  }

  async getLegalStatus(epodocNumber: string): Promise<EPOLegalStatus> {
    const token = await this.authenticate();
    const response = await fetch(
      `${this.baseUrl}/legal/${epodocNumber}`,
      { headers: { 'Authorization': `Bearer ${token}`, 'Accept': 'application/json' } }
    );
    return response.json();
  }

  async getPatentFamily(epodocNumber: string): Promise<EPOFamilyMembers> {
    const token = await this.authenticate();
    const response = await fetch(
      `${this.baseUrl}/family/publication/epodoc/${epodocNumber}`,
      { headers: { 'Authorization': `Bearer ${token}`, 'Accept': 'application/json' } }
    );
    return response.json();
  }
}
```

**Testing:**
- OAuth 2.0 token acquisition works with valid client credentials.
- Token is cached and reused until expiry.
- Bibliographic data retrieval returns title, abstract, classifications, and inventors.
- Legal status retrieval returns patent legal events (grants, lapses, renewals).
- Family member retrieval returns related patents across jurisdictions.
- Expired token triggers automatic re-authentication.
- Rate limiting: respects EPO's 4 GB/month free tier and backs off on throttle responses.

---

### Task 4.3: WIPO ST.96 Parser

**What:** Build a standalone ST.96 XML parser package for structured extraction of patent and trademark data from WIPO-standard XML documents.

**Design:**

```typescript
// packages/st96-parser/src/index.ts
import { XMLParser } from 'fast-xml-parser';

export interface ParsedST96Patent {
  applicationNumber: string;
  filingDate: string;
  title: string;
  abstract?: string;
  applicants: Array<{ name: string; country: string }>;
  inventors: Array<{ name: string; country: string }>;
  classifications: Array<{ type: 'ipc' | 'cpc'; code: string }>;
  priorityClaims: Array<{
    country: string;
    applicationNumber: string;
    filingDate: string;
  }>;
  legalStatus?: string;
}

export class ST96Parser {
  private parser = new XMLParser({
    ignoreAttributes: false,
    attributeNamePrefix: '@_',
  });

  parsePatentDocument(xml: string): ParsedST96Patent {
    const doc = this.parser.parse(xml);
    const patent = doc['pat:PatentPublication'] ?? doc['pat:PatentApplication'];

    return {
      applicationNumber: this.extractApplicationNumber(patent),
      filingDate: this.extractFilingDate(patent),
      title: this.extractTitle(patent),
      abstract: this.extractAbstract(patent),
      applicants: this.extractApplicants(patent),
      inventors: this.extractInventors(patent),
      classifications: this.extractClassifications(patent),
      priorityClaims: this.extractPriorityClaims(patent),
    };
  }

  // Each extraction method handles the specific ST.96 XML structure
  private extractApplicationNumber(patent: any): string {
    const appRef = patent?.['com:ApplicationReference'];
    const docId = appRef?.['com:DocumentIdentification'];
    return `${docId?.['com:IPOfficeCode']}${docId?.['com:DocumentNumber']}`;
  }
  // ... additional extraction methods
}
```

**Testing:**
- Parse a sample USPTO ST.96 XML document; verify all fields are correctly extracted.
- Parse a sample EPO ST.96 XML document; verify European-specific fields.
- Handle missing optional fields (abstract, priority claims) without throwing.
- Handle malformed XML gracefully with descriptive error messages.
- Parse a document with multiple inventors and priority claims.
- Round-trip: parse an ST.96 document, then verify extracted data against known values.

---

### Task 4.4: Background Sync Jobs

**What:** Implement scheduled background jobs that sync patent data from USPTO and EPO at configurable intervals.

**Design:**

```typescript
// apps/api/src/modules/integrations/sync-scheduler.ts
import { CronJob } from 'cron';

export class SyncScheduler {
  private jobs: Map<string, CronJob> = new Map();

  async initialise() {
    const integrations = await this.integrationRepo.findActive();

    for (const integration of integrations) {
      const schedule = integration.config.syncFrequencyHours
        ? `0 */${integration.config.syncFrequencyHours} * * *`
        : '0 2 * * *'; // Default: daily at 2 AM

      const job = new CronJob(schedule, async () => {
        await this.runSync(integration);
      });

      this.jobs.set(integration.id, job);
      job.start();
    }
  }

  private async runSync(integration: Integration) {
    const patents = await this.patentRepo.findByOrganisation(
      integration.organisationId
    );

    for (const patent of patents) {
      try {
        if (integration.integrationType === 'uspto') {
          await this.usptoSync.syncPatent(integration.organisationId, patent.id);
        } else if (integration.integrationType === 'epo_ops') {
          await this.epoSync.syncPatent(integration.organisationId, patent.id);
        }
      } catch (error) {
        await this.logSyncError(integration.id, patent.id, error);
      }
    }
  }
}
```

**Testing:**
- Sync job runs on schedule and updates patent statuses from patent office APIs.
- Failed syncs are logged with error details and do not prevent other patents from syncing.
- Sync frequency is configurable per integration (hourly, daily, weekly).
- Sync status dashboard shows last sync time, records processed, and errors per integration.
- Disabling an integration stops its sync job.

---

## Phase 5: Document Management & AI Docketing

**Goal:** Implement document upload, storage, and AI-powered extraction of docketing data from patent office correspondence.

**Duration:** 3-4 weeks

**Dependencies:** Phase 4

### Task 5.1: Document Storage & Upload

**What:** Implement document upload to S3-compatible storage (MinIO/S3), document metadata tracking, and file retrieval.

**Design:**

```typescript
// apps/api/src/modules/documents/storage-service.ts
import { S3Client, PutObjectCommand, GetObjectCommand } from '@aws-sdk/client-s3';
import { getSignedUrl } from '@aws-sdk/s3-request-presigner';

export class DocumentStorageService {
  private s3: S3Client;
  private bucket: string;

  async upload(file: Buffer, metadata: {
    organisationId: string;
    ipAssetId?: string;
    documentType: string;
    fileName: string;
    mimeType: string;
  }): Promise<Document> {
    const storagePath = `${metadata.organisationId}/${metadata.ipAssetId ?? 'general'}/${Date.now()}-${metadata.fileName}`;

    await this.s3.send(new PutObjectCommand({
      Bucket: this.bucket,
      Key: storagePath,
      Body: file,
      ContentType: metadata.mimeType,
      ServerSideEncryption: 'AES256',
    }));

    return this.documentRepo.create({
      ...metadata,
      storagePath,
      fileSizeBytes: file.length,
    });
  }

  async getDownloadUrl(document: Document): Promise<string> {
    const command = new GetObjectCommand({
      Bucket: this.bucket,
      Key: document.storagePath,
    });
    return getSignedUrl(this.s3, command, { expiresIn: 3600 });
  }
}
```

**Testing:**
- Upload a PDF file; verify it is stored in S3/MinIO with correct path structure.
- Download URL is signed and expires after 1 hour.
- Document metadata is persisted in the database with correct `ip_asset_id` link.
- Upload of files exceeding size limit (configurable, default 50 MB) returns 413.
- Document list for a patent returns all associated documents ordered by upload date.
- Server-side encryption (AES-256) is applied to all stored documents.

---

### Task 5.2: AI-Powered Correspondence Extraction

**What:** Build an AI service that reads uploaded patent office correspondence (office actions, examination reports) and extracts structured docketing data: deadline types, due dates, cited prior art, and rejected claims.

**Design:**

```typescript
// apps/api/src/modules/ai/docketing/extraction-service.ts
import Anthropic from '@anthropic-ai/sdk';

export class CorrespondenceExtractionService {
  private client: Anthropic;

  async extractDocketingData(document: Document): Promise<ExtractedDocketingData> {
    // Retrieve document content
    const fileBuffer = await this.storageService.download(document.storagePath);
    const base64Content = fileBuffer.toString('base64');

    const response = await this.client.messages.create({
      model: 'claude-sonnet-4-20250514',
      max_tokens: 4096,
      messages: [{
        role: 'user',
        content: [
          {
            type: 'document',
            source: {
              type: 'base64',
              media_type: document.mimeType as 'application/pdf',
              data: base64Content,
            },
          },
          {
            type: 'text',
            text: `You are an expert patent prosecution paralegal. Extract the following structured data from this patent office correspondence:

1. **Document Type**: office_action | restriction_requirement | notice_of_allowance | examination_report | search_report | other
2. **Mailing Date**: The official mailing date of the document
3. **Response Deadline**: Calculate the response deadline based on the mailing date and applicable rules
4. **Rejected/Objected Claims**: List claim numbers and the basis for rejection (35 USC 101, 102, 103, 112, etc.)
5. **Prior Art Cited**: List all cited references with patent numbers, publication numbers, and inventor names
6. **Examiner Name** and **Art Unit** (if visible)
7. **Key Requirements**: What the applicant must do in response

Return the data as JSON matching this schema:
{
  "documentType": string,
  "mailingDate": "YYYY-MM-DD",
  "responseDeadline": "YYYY-MM-DD",
  "gracePeriodEnd": "YYYY-MM-DD" | null,
  "rejectedClaims": [{"claimNumber": number, "basis": string, "details": string}],
  "priorArtCited": [{"patentNumber": string, "inventorName": string, "relevance": string}],
  "examinerName": string | null,
  "artUnit": string | null,
  "requirements": [string],
  "confidence": number  // 0.0 to 1.0
}`,
          },
        ],
      }],
    });

    const extracted = JSON.parse(
      response.content[0].type === 'text' ? response.content[0].text : ''
    ) as ExtractedDocketingData;

    // Persist extracted data on the document record
    await this.documentRepo.update(document.id, {
      extractedData: {
        ...extracted,
        aiModel: 'claude-sonnet-4-20250514',
        extractedAt: new Date().toISOString(),
      },
    });

    return extracted;
  }

  // Auto-create deadline from extracted data (requires paralegal approval)
  async proposeDocketingEntry(
    organisationId: string,
    document: Document,
    extracted: ExtractedDocketingData
  ): Promise<ProposedDeadline> {
    return {
      deadlineType: 'office_action_response',
      dueDate: extracted.responseDeadline,
      gracePeriodEnd: extracted.gracePeriodEnd,
      status: 'proposed',  // Not active until approved by paralegal
      details: {
        officeActionType: extracted.documentType,
        mailingDate: extracted.mailingDate,
        rejectedClaims: extracted.rejectedClaims,
        priorArtCited: extracted.priorArtCited,
        aiConfidence: extracted.confidence,
      },
    };
  }
}
```

**Testing:**
- Upload a sample USPTO non-final office action PDF; AI extracts mailing date, response deadline, rejected claims, and cited prior art.
- Extracted `responseDeadline` is correct (mailing date + 3 months for US non-final OA).
- AI confidence score is returned; low-confidence extractions (<0.7) are flagged for manual review.
- Proposed docketing entry is created with `status: 'proposed'` (not auto-activated).
- Paralegal approving the proposal converts it to an active deadline.
- Extraction results are stored on the document's `extracted_data` JSONB column.
- Processing a non-patent document (e.g., an invoice) returns a meaningful error rather than hallucinated data.

---

### Task 5.3: Document Viewer & Management UI

**What:** Build document list, upload, preview, and AI extraction review interfaces.

**Design:**

```typescript
// apps/web/src/app/(dashboard)/documents/page.tsx
export default async function DocumentsPage() {
  return (
    <div className="space-y-4">
      <DocumentUploadDropzone />

      <DataTable
        columns={documentColumns}
        data={documents}
        actions={[
          { label: 'Preview', icon: Eye, onClick: previewDocument },
          { label: 'Extract Data', icon: Sparkles, onClick: extractData },
          { label: 'Download', icon: Download, onClick: downloadDocument },
        ]}
      />
    </div>
  );
}

// AI extraction review component
function ExtractionReview({ extraction }: { extraction: ExtractedDocketingData }) {
  return (
    <Card>
      <CardHeader>
        <CardTitle>AI-Extracted Docketing Data</CardTitle>
        <Badge variant={extraction.confidence > 0.8 ? 'default' : 'destructive'}>
          Confidence: {(extraction.confidence * 100).toFixed(0)}%
        </Badge>
      </CardHeader>
      <CardContent>
        <dl className="grid grid-cols-2 gap-4">
          <dt>Document Type</dt>
          <dd>{extraction.documentType}</dd>
          <dt>Response Deadline</dt>
          <dd className="font-bold text-destructive">{extraction.responseDeadline}</dd>
          <dt>Rejected Claims</dt>
          <dd>{extraction.rejectedClaims.map(c => `Claim ${c.claimNumber}`).join(', ')}</dd>
        </dl>
        <div className="mt-4 flex gap-2">
          <Button onClick={approveAndCreateDeadline}>Approve & Create Deadline</Button>
          <Button variant="outline" onClick={editBeforeApproval}>Edit</Button>
          <Button variant="ghost" onClick={dismiss}>Dismiss</Button>
        </div>
      </CardContent>
    </Card>
  );
}
```

**Testing:**
- Drag-and-drop file upload creates a document record and uploads to storage.
- PDF documents are previewable in-browser using an embedded PDF viewer.
- "Extract Data" button triggers AI extraction and displays results in the review panel.
- Paralegal can edit extracted fields before approving.
- Approving an extraction creates an active deadline and links the document to the patent.
- Document list filters by document type, patent, and date range.

---

## Phase 6: Annuity Management & Decision Support

**Goal:** Build the annuity/renewal fee management system with AI-assisted pay/abandon decision scoring.

**Duration:** 3-4 weeks

**Dependencies:** Phase 3

### Task 6.1: Annuity Decision Queue

**What:** Build a batch review interface for upcoming annuity/maintenance fee decisions, displaying all relevant data for each pay/abandon decision.

**Design:**

```typescript
// apps/api/src/modules/annuities/service.ts
export class AnnuityService {
  async getDecisionQueue(organisationId: string, filters?: {
    jurisdictionCode?: string;
    dueBefore?: Date;
    decisionStatus?: 'pending' | 'decided';
  }) {
    return this.db.query(sql`
      SELECT
        d.id AS deadline_id,
        d.due_date,
        d.fee_amount,
        d.fee_currency,
        d.grace_period_end,
        d.annuity_decision,
        ia.id AS patent_id,
        ia.title,
        ia.application_number,
        ia.registration_number,
        ia.jurisdiction_code,
        ia.status AS patent_status,
        ia.cost_to_date,
        ia.ai_scores
      FROM deadlines d
      JOIN ip_assets ia ON d.ip_asset_id = ia.id
      WHERE d.organisation_id = ${organisationId}
        AND d.deadline_type IN ('annuity_payment', 'maintenance_fee')
        AND d.status IN ('upcoming', 'due_soon')
        ${filters?.jurisdictionCode ? sql`AND d.jurisdiction_code = ${filters.jurisdictionCode}` : sql``}
      ORDER BY d.due_date ASC
    `);
  }

  async recordDecision(
    organisationId: string,
    deadlineId: string,
    decision: {
      decision: 'pay' | 'abandon';
      reasoning?: string;
    },
    userId: string
  ) {
    await this.deadlineRepo.update(organisationId, deadlineId, {
      annuityDecision: {
        ...decision,
        decidedBy: userId,
        decidedAt: new Date().toISOString(),
      },
      status: decision.decision === 'pay' ? 'upcoming' : 'waived',
    });
  }
}
```

**Testing:**
- Decision queue returns all upcoming annuity deadlines with patent details and AI scores.
- Recording a `pay` decision keeps the deadline active.
- Recording an `abandon` decision marks the deadline as `waived` and the patent status transitions appropriately.
- Batch decision: user can select multiple deadlines and apply `pay` to all selected.
- Decision audit trail records who decided, when, and the reasoning.
- Filter by jurisdiction shows only deadlines for the selected jurisdiction.

---

### Task 6.2: AI Annuity Decision Scoring

**What:** Implement AI-powered scoring that analyses citation metrics, business alignment, and portfolio coverage to generate pay/abandon recommendations.

**Design:**

```typescript
// apps/api/src/modules/ai/annuity-scoring/service.ts
export class AnnuityScoringSevice {
  async scorePatentForRenewal(
    patent: IPAsset,
    citationData: CitationMetrics,
    portfolioContext: PortfolioContext
  ): Promise<AnnuityScore> {
    // Citation score: forward citations indicate patent importance
    const citationScore = this.computeCitationScore(citationData);

    // Business alignment: how well does this patent align with current business areas?
    const businessAlignment = await this.computeBusinessAlignment(
      patent, portfolioContext.businessAreas
    );

    // Portfolio coverage: would abandoning this patent create a coverage gap?
    const portfolioCoverage = this.computePortfolioCoverage(
      patent, portfolioContext.familyMembers
    );

    // Licensing potential: is this patent generating or likely to generate revenue?
    const licensingPotential = this.computeLicensingPotential(
      patent, portfolioContext.licences
    );

    // Cost efficiency: cost-to-date relative to remaining patent term
    const costEfficiency = this.computeCostEfficiency(patent);

    // Composite recommendation
    const compositeScore = (
      citationScore * 0.25 +
      businessAlignment * 0.30 +
      portfolioCoverage * 0.20 +
      licensingPotential * 0.15 +
      costEfficiency * 0.10
    );

    const recommendation: 'pay' | 'abandon' | 'review' =
      compositeScore >= 6.0 ? 'pay' :
      compositeScore <= 3.0 ? 'abandon' : 'review';

    const confidence = Math.abs(compositeScore - 5.0) / 5.0; // Higher when further from the boundary

    return {
      citationScore,
      forwardCitationCount: citationData.forwardCitations,
      businessAlignmentScore: businessAlignment,
      licensingPotentialScore: licensingPotential,
      portfolioCoverageScore: portfolioCoverage,
      aiRecommendation: recommendation,
      aiConfidence: Math.min(confidence, 1.0),
      costIfPaid: patent.nextAnnuityAmount,
    };
  }
}
```

**Testing:**
- A patent with high forward citations (>20), active licence, and central portfolio position scores "pay" with high confidence.
- A patent with zero citations, expired in multiple family jurisdictions, and no licensing activity scores "abandon" with high confidence.
- A patent with moderate signals scores "review" with low confidence.
- Scores are persisted on both the `annuity_decision` JSONB and the patent's `ai_scores` JSONB.
- Score computation uses only data available in the local database (no external API calls in the scoring path).
- Scoring 100 patents completes within 10 seconds (no AI API calls in the scoring loop).

---

### Task 6.3: Annuity Decision UI

**What:** Build the annuity decision queue interface with inline AI scoring display, batch operations, and decision workflow.

**Design:**

```typescript
// apps/web/src/app/(dashboard)/annuities/page.tsx
export default async function AnnuitiesPage() {
  const queue = await fetchAnnuityQueue();

  return (
    <div className="space-y-6">
      <AnnuitySummaryCards queue={queue} />

      <DataTable
        columns={annuityColumns}
        data={queue}
        selectable
        bulkActions={[
          { label: 'Pay Selected', action: batchPay },
          { label: 'Abandon Selected', action: batchAbandon },
        ]}
      />
    </div>
  );
}

// Inline scoring display per row
function AnnuityScoreCell({ scores }: { scores: AnnuityScore }) {
  return (
    <div className="flex items-center gap-2">
      <Badge variant={
        scores.aiRecommendation === 'pay' ? 'default' :
        scores.aiRecommendation === 'abandon' ? 'destructive' : 'secondary'
      }>
        {scores.aiRecommendation.toUpperCase()}
      </Badge>
      <span className="text-xs text-muted-foreground">
        {(scores.aiConfidence * 100).toFixed(0)}% confidence
      </span>
      <ScoreBreakdownTooltip scores={scores} />
    </div>
  );
}
```

**Testing:**
- Annuity queue shows all upcoming annuity decisions with patent details and AI scores.
- AI recommendation badge is colour-coded: green (pay), red (abandon), grey (review).
- Hovering over the score shows a breakdown tooltip with individual metric scores.
- Batch "Pay Selected" records pay decisions for all selected deadlines.
- Decision confirmation dialog shows total cost of selected pay decisions.
- Filtering by AI recommendation ("show me all 'abandon' recommendations") works.

---

## Phase 7: Parties, Counsel & Cost Tracking

**Goal:** Implement party management (inventors, assignees, counsel), outside counsel cost tracking, and invoice management.

**Duration:** 2-3 weeks

**Dependencies:** Phase 2

### Task 7.1: Party Management

**What:** Implement the parties table and CRUD for inventors, assignees, outside counsel, and other party types. Build the party-to-asset relationship management.

**Design:**

```sql
-- migrations/005_parties.sql

CREATE TABLE parties (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    is_person       BOOLEAN NOT NULL DEFAULT true,
    display_name    TEXT NOT NULL,
    contact         JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

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
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (ip_asset_id, party_id, role)
);
```

**Testing:**
- Create a party (inventor); link to a patent with role `inventor`.
- Same party linked to multiple patents with different roles (inventor on one, assignee on another).
- `GET /api/patents/:id/parties` returns all linked parties with roles.
- `GET /api/parties/:id/assets` returns all IP assets linked to a party.
- Removing a party from a patent sets `is_current = false` (soft delete for audit trail).
- Contact JSONB stores structured address, email, phone, bar number, and firm name.

---

### Task 7.2: Outside Counsel Cost Tracking

**What:** Implement cost entry tracking linked to IP assets and outside counsel assignments.

**Design:**

```typescript
// apps/api/src/modules/costs/service.ts
export class CostTrackingService {
  async recordCost(organisationId: string, data: {
    ipAssetId?: string;
    costType: CostType;
    amount: number;
    currencyCode: string;
    incurredDate: string;
    vendorName?: string;
    invoiceNumber?: string;
    details?: Record<string, any>;
  }) {
    const entry = await this.costRepo.create({ organisationId, ...data });

    // Update cost_to_date on the linked IP asset
    if (data.ipAssetId) {
      await this.db.query(sql`
        UPDATE ip_assets
        SET cost_to_date = cost_to_date + ${data.amount},
            updated_at = now()
        WHERE id = ${data.ipAssetId}
          AND organisation_id = ${organisationId}
      `);
    }

    return entry;
  }

  async getCostReport(organisationId: string, filters: {
    year?: number;
    jurisdictionCode?: string;
    costType?: string;
  }) {
    return this.db.query(sql`
      SELECT
        ce.cost_type,
        ia.jurisdiction_code,
        SUM(ce.amount) AS total,
        ce.currency_code,
        COUNT(*) AS entry_count
      FROM cost_entries ce
      LEFT JOIN ip_assets ia ON ce.ip_asset_id = ia.id
      WHERE ce.organisation_id = ${organisationId}
        ${filters.year ? sql`AND EXTRACT(YEAR FROM ce.incurred_date) = ${filters.year}` : sql``}
      GROUP BY ce.cost_type, ia.jurisdiction_code, ce.currency_code
      ORDER BY total DESC
    `);
  }
}
```

**Testing:**
- Recording a cost entry of $5,000 linked to a patent increases the patent's `cost_to_date` by $5,000.
- Cost report grouped by type and jurisdiction returns correct totals.
- Cost entries support multiple currencies; currency is stored per entry.
- Filtering cost report by year returns only entries for that year.
- Cost entry linked to outside counsel includes `details.counselAssignmentId`, `details.hoursBilled`, `details.hourlyRate`.

---

### Task 7.3: Counsel & Cost Management UI

**What:** Build party list, outside counsel assignment views, and cost reporting dashboards.

**Testing:**
- Party list page shows inventors, counsel, and assignees with linked asset counts.
- Party detail page shows all linked assets with roles and assignment dates.
- Cost report page shows spending by category, jurisdiction, and time period.
- Bar chart visualisation of spending by cost type (filing, prosecution, annuity, counsel).
- Cost entry form validates amount, currency, and links to IP asset.

---

## Phase 8: Prior Art Search & FTO Analysis

**Goal:** Implement prior art search via USPTO and EPO OPS APIs, and build AI-powered preliminary FTO analysis.

**Duration:** 4-5 weeks

**Dependencies:** Phase 4

### Task 8.1: Prior Art Search Integration

**What:** Build a search interface that queries USPTO PatentsView and EPO OPS for prior art and returns ranked results.

**Design:**

```typescript
// apps/api/src/modules/searches/prior-art-service.ts
export class PriorArtSearchService {
  async search(organisationId: string, params: {
    queryText: string;
    dataSources: Array<'uspto' | 'epo_ops'>;
    maxResults?: number;
  }): Promise<SearchResult> {
    const results: PriorArtResult[] = [];

    if (params.dataSources.includes('uspto')) {
      const usptoResults = await this.usptoClient.searchPatents(params.queryText);
      results.push(...usptoResults.map(this.normalizeUSPTOResult));
    }

    if (params.dataSources.includes('epo_ops')) {
      const epoResults = await this.epoClient.searchPatents(params.queryText);
      results.push(...epoResults.map(this.normalizeEPOResult));
    }

    // Deduplicate by patent family
    const deduped = this.deduplicateByFamily(results);

    // Store search and results
    const search = await this.searchRepo.create({
      organisationId,
      searchType: 'patentability',
      queryText: params.queryText,
      dataSources: params.dataSources,
      results: deduped,
      summary: {
        totalResults: deduped.length,
        highRiskCount: deduped.filter(r => r.riskLevel === 'high').length,
        mediumRiskCount: deduped.filter(r => r.riskLevel === 'medium').length,
        lowRiskCount: deduped.filter(r => r.riskLevel === 'low').length,
      },
    });

    return search;
  }
}
```

**Testing:**
- Searching "quantum error correction" returns results from both USPTO and EPO OPS.
- Results are deduplicated by patent family (same invention, different jurisdictions, shows once).
- Results include patent number, title, assignee, filing date, and relevance score.
- Search is persisted with results in `searches` table JSONB.
- Empty query or no results returns an empty result set, not an error.
- Rate limiting and error handling for external API calls.

---

### Task 8.2: AI-Powered FTO Preliminary Analysis

**What:** Build an AI service that maps product features to patent claim language and generates a preliminary FTO risk landscape.

**Design:**

```typescript
// apps/api/src/modules/ai/fto/analysis-service.ts
export class FTOAnalysisService {
  async analyseProductClearance(
    organisationId: string,
    productDescription: string,
    productFeatures: string[],
    relevantPatents: PriorArtResult[]
  ): Promise<FTOAnalysis> {
    const response = await this.anthropic.messages.create({
      model: 'claude-sonnet-4-20250514',
      max_tokens: 8192,
      messages: [{
        role: 'user',
        content: `You are an expert patent attorney conducting a preliminary freedom-to-operate analysis.

Product Description: ${productDescription}

Product Features:
${productFeatures.map((f, i) => `${i + 1}. ${f}`).join('\n')}

Potentially Relevant Patents:
${relevantPatents.map(p => `- ${p.patentNumber}: "${p.title}" (Claims: ${p.claimSummary})`).join('\n')}

For each product feature, assess whether any of the listed patents could potentially restrict the freedom to operate. Return JSON:
{
  "overallRiskScore": number (0.0-1.0),
  "featureAnalysis": [
    {
      "feature": string,
      "riskLevel": "high" | "medium" | "low" | "none",
      "potentiallyBlockingPatents": [
        {
          "patentNumber": string,
          "overlappingClaims": [number],
          "riskDescription": string,
          "mitigationSuggestions": [string]
        }
      ]
    }
  ],
  "disclaimer": "This is a preliminary AI-assisted analysis and does not constitute legal advice. A qualified patent attorney should review these findings before making business decisions."
}`,
      }],
    });

    const analysis = JSON.parse(response.content[0].text);

    await this.searchRepo.create({
      organisationId,
      searchType: 'fto',
      queryText: productDescription,
      results: analysis.featureAnalysis,
      summary: {
        overallRiskScore: analysis.overallRiskScore,
        highRiskCount: analysis.featureAnalysis.filter(f => f.riskLevel === 'high').length,
        disclaimer: analysis.disclaimer,
      },
    });

    return analysis;
  }
}
```

**Testing:**
- FTO analysis for a product with 5 features against 10 patents returns a per-feature risk assessment.
- High-risk features identify specific claim overlaps and mitigation suggestions.
- The mandatory disclaimer is always included in the response.
- Analysis is stored in the `searches` table for future reference.
- Running FTO on a product with no overlapping patents returns all features as "none" risk.
- AI response is validated against the expected JSON schema before storage.

---

### Task 8.3: Search & FTO UI

**What:** Build the prior art search interface and FTO analysis dashboard with risk visualisation.

**Testing:**
- Search page allows free-text query with data source selection (USPTO, EPO).
- Results display as a sortable table with relevance scores and risk indicators.
- FTO analysis page accepts product description and feature list.
- FTO results display as a colour-coded risk matrix (green/yellow/red per feature).
- Clicking a blocking patent shows claim overlap details and mitigation suggestions.
- FTO report is exportable as PDF with the disclaimer prominently displayed.

---

## Phase 9: Invention Disclosures & Licensing

**Goal:** Implement invention disclosure tracking (university tech transfer workflow) and licence agreement management.

**Duration:** 2-3 weeks

**Dependencies:** Phase 2, Phase 7

### Task 9.1: Invention Disclosure Workflow

**What:** Implement disclosure creation, status tracking, inventor assignment, and linkage to filed patents.

**Design:**

```typescript
// apps/api/src/modules/disclosures/service.ts
export class DisclosureService {
  async create(organisationId: string, data: {
    disclosureNumber: string;
    title: string;
    description?: string;
    disclosedDate: string;
    technologyArea?: string;
    inventors: Array<{ partyId: string; contributionPercent: number }>;
  }) {
    // Validate inventor contributions sum to 100%
    const totalContribution = data.inventors.reduce(
      (sum, inv) => sum + inv.contributionPercent, 0
    );
    if (Math.abs(totalContribution - 100) > 0.01) {
      throw new ValidationError('Inventor contributions must sum to 100%');
    }

    return this.disclosureRepo.create({
      organisationId,
      ...data,
      status: 'submitted',
      details: {
        inventors: data.inventors,
        technologyArea: data.technologyArea,
      },
    });
  }

  async transitionStatus(
    organisationId: string,
    disclosureId: string,
    newStatus: DisclosureStatus,
    metadata?: { patentId?: string }
  ) {
    const disclosure = await this.disclosureRepo.findById(organisationId, disclosureId);

    // Validate status transition
    const validTransitions: Record<string, string[]> = {
      submitted: ['under_review', 'rejected'],
      under_review: ['approved_for_filing', 'rejected'],
      approved_for_filing: ['filed'],
      filed: ['licensed', 'abandoned'],
    };

    if (!validTransitions[disclosure.status]?.includes(newStatus)) {
      throw new ValidationError(
        `Cannot transition from ${disclosure.status} to ${newStatus}`
      );
    }

    const updates: Partial<InventionDisclosure> = { status: newStatus };
    if (newStatus === 'filed' && metadata?.patentId) {
      updates.ipAssetId = metadata.patentId;
    }

    await this.disclosureRepo.update(organisationId, disclosureId, updates);
  }
}
```

**Testing:**
- Create a disclosure with 2 inventors at 60%/40% contribution.
- Validation rejects contributions that do not sum to 100%.
- Status transitions follow the defined workflow: submitted -> under_review -> approved_for_filing -> filed.
- Invalid transitions (e.g., submitted -> filed directly) return 400.
- Linking a disclosure to a filed patent via `ipAssetId` works.
- Disclosure list filters by status and technology area.

---

### Task 9.2: Licence Agreement Management

**What:** Implement licence agreement CRUD with royalty tracking and linked IP assets.

**Design:**

```typescript
// apps/api/src/modules/licences/service.ts
const CreateLicenceSchema = z.object({
  agreementNumber: z.string(),
  licenseePartyId: z.string().uuid(),
  licenceType: z.enum(['exclusive', 'non_exclusive', 'sole', 'cross_licence', 'compulsory']),
  effectiveDate: z.string().date(),
  expiryDate: z.string().date().optional(),
  licensedAssetIds: z.array(z.string().uuid()).min(1),
  terms: z.object({
    fieldOfUse: z.string().optional(),
    territory: z.array(z.string()).optional(),
    royaltyType: z.enum(['percentage', 'fixed', 'milestone', 'hybrid']),
    royaltyRate: z.number().optional(),
    minimumRoyalty: z.number().optional(),
    paymentFrequency: z.enum(['monthly', 'quarterly', 'annually']).optional(),
  }),
});
```

**Testing:**
- Create a licence linking 3 patents to a licensee with quarterly royalty payments.
- `GET /api/licences/:id` returns the licence with all linked patent titles and licensee name.
- Licence expiry generates a deadline for renewal review.
- Royalty payment tracking: record received payments against scheduled amounts.
- Overdue royalty payments are flagged in the licence dashboard.
- A patent linked to an active licence shows the licence information on its detail page.

---

## Phase 10: Portfolio Analytics & AI Scoring

**Goal:** Build portfolio-level analytics, AI-powered portfolio optimisation scoring, and data visualisations.

**Duration:** 3-4 weeks

**Dependencies:** Phase 6, Phase 8

### Task 10.1: Portfolio Dashboard & Reporting

**What:** Build the main portfolio analytics dashboard with key metrics, charts, and filterable views.

**Design:**

```typescript
// apps/api/src/modules/analytics/service.ts
export class PortfolioAnalyticsService {
  async getPortfolioSummary(organisationId: string) {
    const [assetCounts, costSummary, deadlineSummary, jurisdictionBreakdown] =
      await Promise.all([
        this.getAssetCounts(organisationId),
        this.getCostSummary(organisationId),
        this.getDeadlineSummary(organisationId),
        this.getJurisdictionBreakdown(organisationId),
      ]);

    return {
      assetCounts,        // patents by status, trademarks by status
      costSummary,        // total spend YTD, by category
      deadlineSummary,    // upcoming, overdue, completed this month
      jurisdictionBreakdown, // assets per jurisdiction
    };
  }

  async getPortfolioTimeline(organisationId: string, years: number = 5) {
    return this.db.query(sql`
      SELECT
        DATE_TRUNC('month', filing_date) AS month,
        COUNT(*) FILTER (WHERE asset_type = 'patent') AS patents_filed,
        COUNT(*) FILTER (WHERE asset_type = 'trademark') AS trademarks_filed,
        COUNT(*) FILTER (WHERE status = 'granted') AS grants,
        COUNT(*) FILTER (WHERE status = 'abandoned') AS abandonments
      FROM ip_assets
      WHERE organisation_id = ${organisationId}
        AND filing_date >= CURRENT_DATE - (${years} || ' years')::interval
      GROUP BY month
      ORDER BY month
    `);
  }
}
```

**Testing:**
- Portfolio summary returns correct counts of patents and trademarks by status.
- Cost summary matches the sum of all cost entries for the current year.
- Deadline summary correctly counts overdue, upcoming, and completed deadlines.
- Jurisdiction breakdown chart data includes all jurisdictions with at least one asset.
- Timeline chart shows filing activity over the past 5 years by month.

---

### Task 10.2: AI Portfolio Optimisation Scoring

**What:** Implement continuous portfolio scoring that identifies patents suitable for abandonment, licensing, or sale.

**Design:**

```typescript
// apps/api/src/modules/ai/portfolio-scoring/service.ts
export class PortfolioScoringService {
  async scoreEntirePortfolio(organisationId: string) {
    const patents = await this.patentRepo.findGranted(organisationId);
    const citations = await this.citationRepo.getMetricsForOrg(organisationId);
    const licences = await this.licenceRepo.findActive(organisationId);

    const scores: PatentScore[] = [];

    for (const patent of patents) {
      const score = await this.annuityScoringService.scorePatentForRenewal(
        patent,
        citations.get(patent.id) ?? { forwardCitations: 0, backwardCitations: 0 },
        {
          businessAreas: await this.getBusinessAreas(organisationId),
          familyMembers: await this.getFamilyMembers(patent.familyId),
          licences: licences.filter(l => l.licensedAssetIds.includes(patent.id)),
        }
      );

      scores.push({ patentId: patent.id, ...score });

      // Update ai_scores on the patent
      await this.patentRepo.update(organisationId, patent.id, {
        aiScores: {
          ...score,
          scoredAt: new Date().toISOString(),
          modelVersion: 'portfolio-scorer-v1.0',
        },
      });
    }

    return {
      totalScored: scores.length,
      recommendations: {
        pay: scores.filter(s => s.aiRecommendation === 'pay').length,
        abandon: scores.filter(s => s.aiRecommendation === 'abandon').length,
        review: scores.filter(s => s.aiRecommendation === 'review').length,
      },
      potentialSavings: scores
        .filter(s => s.aiRecommendation === 'abandon')
        .reduce((sum, s) => sum + (s.costIfPaid ?? 0), 0),
    };
  }
}
```

**Testing:**
- Scoring a portfolio of 50 patents completes within 30 seconds.
- Each patent's `ai_scores` JSONB is updated with current scores.
- Summary correctly calculates potential savings from recommended abandonments.
- Patents recommended for abandonment have low citation scores and no active licences.
- Portfolio scoring is idempotent: running twice produces the same results.

---

### Task 10.3: Analytics Visualisations

**What:** Build interactive charts and visualisations for portfolio data.

**Testing:**
- Jurisdiction map shows asset counts per country with colour coding.
- Technology area treemap shows CPC classification distribution.
- Cost trend line chart shows spending by category over time.
- AI score distribution histogram shows the spread of portfolio scores.
- All charts are interactive: clicking a segment filters the underlying data table.
- Charts export as PNG for inclusion in reports.

---

## Phase 11: Plugin Architecture & Integrations

**Goal:** Build the plugin interface for annuity service providers, additional patent offices, and MCP server support.

**Duration:** 3-4 weeks

**Dependencies:** Phase 4, Phase 6

### Task 11.1: Plugin Interface Definition

**What:** Define the plugin interface contract that annuity providers (Dennemeyer, Questel PAVIS, CPA Global) and patent office connectors must implement.

**Design:**

```typescript
// apps/api/src/plugins/types.ts
export interface AnnuityProviderPlugin {
  readonly name: string;
  readonly version: string;

  // Configuration schema (JSON Schema)
  getConfigSchema(): JsonSchema;

  // Submit a renewal instruction
  submitRenewal(instruction: RenewalInstruction): Promise<RenewalConfirmation>;

  // Check renewal status
  checkStatus(referenceId: string): Promise<RenewalStatus>;

  // Get available fee schedules
  getFeeSchedule(jurisdictionCode: string, year: number): Promise<FeeScheduleEntry[]>;
}

export interface PatentOfficePlugin {
  readonly name: string;
  readonly version: string;
  readonly supportedJurisdictions: string[];

  getConfigSchema(): JsonSchema;

  // Fetch patent status
  getPatentStatus(applicationNumber: string): Promise<PatentStatus>;

  // Fetch patent documents
  getDocuments(applicationNumber: string): Promise<PatentDocument[]>;

  // Search patents
  searchPatents(query: string, options?: SearchOptions): Promise<SearchResult[]>;
}

// Plugin registry
export class PluginRegistry {
  private annuityProviders = new Map<string, AnnuityProviderPlugin>();
  private patentOffices = new Map<string, PatentOfficePlugin>();

  registerAnnuityProvider(plugin: AnnuityProviderPlugin) {
    this.annuityProviders.set(plugin.name, plugin);
  }

  registerPatentOffice(plugin: PatentOfficePlugin) {
    this.patentOffices.set(plugin.name, plugin);
  }
}
```

**Testing:**
- Plugin interface can be implemented by a mock annuity provider.
- Plugin registry correctly registers and retrieves plugins by name.
- Plugin configuration schema is validated against user-provided config.
- Plugin errors are caught and do not crash the main application.
- A plugin can be disabled per organisation via `integrations` table.

---

### Task 11.2: MCP Server Implementation

**What:** Expose patent portfolio data and deadline queues to AI agents via the Model Context Protocol.

**Design:**

```typescript
// apps/api/src/modules/mcp/server.ts
import { McpServer } from '@modelcontextprotocol/sdk/server/mcp.js';
import { StdioServerTransport } from '@modelcontextprotocol/sdk/server/stdio.js';

const server = new McpServer({
  name: 'ip-management-platform',
  version: '1.0.0',
});

// Resource: Patent portfolio
server.resource('patent-portfolio', 'ip://portfolio/patents', async (uri) => {
  const patents = await patentService.findAll(organisationId, {});
  return {
    contents: [{
      uri,
      mimeType: 'application/json',
      text: JSON.stringify(patents),
    }],
  };
});

// Resource: Upcoming deadlines
server.resource('upcoming-deadlines', 'ip://deadlines/upcoming', async (uri) => {
  const deadlines = await deadlineService.findUpcoming(organisationId);
  return {
    contents: [{
      uri,
      mimeType: 'application/json',
      text: JSON.stringify(deadlines),
    }],
  };
});

// Tool: Search patents
server.tool('search-patents', { query: z.string() }, async ({ query }) => {
  const results = await searchService.search(organisationId, query);
  return { content: [{ type: 'text', text: JSON.stringify(results) }] };
});

// Tool: Get annuity decision queue
server.tool('annuity-queue', {}, async () => {
  const queue = await annuityService.getDecisionQueue(organisationId);
  return { content: [{ type: 'text', text: JSON.stringify(queue) }] };
});
```

**Testing:**
- MCP server starts and responds to resource list requests.
- `patent-portfolio` resource returns all patents for the authenticated organisation.
- `upcoming-deadlines` resource returns deadlines due within 90 days.
- `search-patents` tool accepts a query and returns patent search results.
- MCP server respects tenant isolation: only returns data for the authenticated org.

---

### Task 11.3: Integration Management UI

**What:** Build a settings page for managing patent office connections, annuity provider plugins, and sync status.

**Testing:**
- Integration settings page lists all available integrations (USPTO, EPO OPS, WIPO).
- Adding a USPTO integration prompts for API key and validates connectivity.
- Adding an EPO OPS integration prompts for OAuth client ID/secret.
- Sync status dashboard shows last sync time, records processed, and error count.
- Manual "Sync Now" button triggers an immediate sync for the selected integration.
- Plugin configuration is stored encrypted in the `integrations` table.

---

## Phase 12: Production Hardening & Compliance

**Goal:** Prepare for production deployment with security hardening, compliance documentation, performance optimisation, and operational tooling.

**Duration:** 3-4 weeks

**Dependencies:** All prior phases

### Task 12.1: Security Hardening

**What:** Implement encryption at rest and in transit, API rate limiting, input sanitisation, and vulnerability scanning.

**Design:**

```typescript
// apps/api/src/shared/middleware/security.ts
import rateLimit from '@fastify/rate-limit';
import helmet from '@fastify/helmet';

// Rate limiting
await app.register(rateLimit, {
  max: 100,           // 100 requests per minute per user
  timeWindow: '1 minute',
  keyGenerator: (request) => request.user?.id ?? request.ip,
});

// Security headers
await app.register(helmet, {
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'"],
      styleSrc: ["'self'", "'unsafe-inline'"],
    },
  },
});

// API key encryption for patent office credentials
import { createCipheriv, createDecipheriv, randomBytes } from 'crypto';

export class CredentialEncryption {
  private algorithm = 'aes-256-gcm';
  private masterKey: Buffer;

  encrypt(plaintext: string): { encrypted: string; iv: string; tag: string } {
    const iv = randomBytes(16);
    const cipher = createCipheriv(this.algorithm, this.masterKey, iv);
    let encrypted = cipher.update(plaintext, 'utf8', 'hex');
    encrypted += cipher.final('hex');
    const tag = cipher.getAuthTag().toString('hex');
    return { encrypted, iv: iv.toString('hex'), tag };
  }

  decrypt(data: { encrypted: string; iv: string; tag: string }): string {
    const decipher = createDecipheriv(
      this.algorithm,
      this.masterKey,
      Buffer.from(data.iv, 'hex')
    );
    decipher.setAuthTag(Buffer.from(data.tag, 'hex'));
    let decrypted = decipher.update(data.encrypted, 'hex', 'utf8');
    decrypted += decipher.final('utf8');
    return decrypted;
  }
}
```

**Testing:**
- API rate limiting returns 429 after 100 requests in 1 minute from the same user.
- Security headers (CSP, HSTS, X-Frame-Options) are present on all responses.
- Patent office API credentials are encrypted at rest in the database.
- Encrypted credentials can be decrypted with the correct master key.
- SQL injection attempts via API parameters are rejected by input validation.
- XSS payloads in patent titles are sanitised on display.
- Dependency vulnerability scan (npm audit) produces zero critical/high findings.

---

### Task 12.2: Audit Log & Compliance

**What:** Ensure comprehensive audit logging aligned with NIST SP 800-92 and ISO 27001 requirements. Implement data retention policies.

**Design:**

```typescript
// apps/api/src/modules/audit/service.ts
export class AuditService {
  async log(
    request: FastifyRequest,
    action: string,
    entityType: string,
    entityId: string,
    changes?: { old?: any; new?: any }
  ) {
    await this.db.query(sql`
      INSERT INTO audit_log (
        organisation_id, user_id, action, entity_type, entity_id,
        changes, ip_address
      ) VALUES (
        ${request.organisationId},
        ${request.user.id},
        ${action},
        ${entityType},
        ${entityId},
        ${changes ? JSON.stringify(changes) : null},
        ${request.ip}
      )
    `);
  }

  // Retention policy: archive logs older than 7 years
  async archiveOldLogs() {
    const cutoff = new Date();
    cutoff.setFullYear(cutoff.getFullYear() - 7);

    const archived = await this.db.query(sql`
      WITH moved AS (
        DELETE FROM audit_log
        WHERE created_at < ${cutoff}
        RETURNING *
      )
      INSERT INTO audit_log_archive
      SELECT * FROM moved
    `);

    return { archivedCount: archived.rowCount };
  }
}
```

**Testing:**
- Every API mutation (create, update, delete) produces an audit log entry.
- Audit log entries include user ID, IP address, entity type, entity ID, and JSONB changes.
- Audit log viewer in the admin panel shows filterable, paginated entries.
- Log retention: entries older than 7 years are archived to `audit_log_archive`.
- Audit logs cannot be modified or deleted by any user role (append-only).
- Exporting audit logs for a date range produces a compliant CSV/JSON report.

---

### Task 12.3: Performance Optimisation

**What:** Optimise database queries, implement connection pooling, add caching, and load test critical paths.

**Design:**

```typescript
// Connection pooling via pgBouncer or built-in pool
import { Pool } from 'pg';

const pool = new Pool({
  host: process.env.DB_HOST,
  database: 'ipms',
  max: 20,                    // Max connections per pool
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 5000,
});

// Redis caching for hot queries
import Redis from 'ioredis';

export class CacheService {
  private redis: Redis;

  async cacheQuery<T>(
    key: string,
    ttlSeconds: number,
    fetcher: () => Promise<T>
  ): Promise<T> {
    const cached = await this.redis.get(key);
    if (cached) return JSON.parse(cached);

    const result = await fetcher();
    await this.redis.setex(key, ttlSeconds, JSON.stringify(result));
    return result;
  }

  // Cache deadline counts for dashboard (60-second TTL)
  async getDeadlineCounts(organisationId: string) {
    return this.cacheQuery(
      `deadline-counts:${organisationId}`,
      60,
      () => this.deadlineRepo.getCounts(organisationId)
    );
  }
}
```

**Testing:**
- Database connection pool handles 20 concurrent connections without timeout.
- Deadline dashboard query returns in <100ms for portfolios of 1,000 patents with 10,000 deadlines.
- Redis cache reduces dashboard load time by 50%+ on repeated requests.
- Load test: 50 concurrent users performing typical operations (list patents, view deadlines, search) sustains <200ms p95 response time.
- Database query explain plans show index usage on all critical queries.

---

### Task 12.4: Deployment & Docker

**What:** Create production-ready Docker images, Kubernetes manifests, and deployment documentation.

**Design:**

```dockerfile
# infrastructure/docker/Dockerfile.api
FROM node:22-alpine AS builder
WORKDIR /app
COPY package*.json ./
COPY apps/api/package*.json ./apps/api/
COPY packages/ ./packages/
RUN npm ci --workspace=apps/api
COPY apps/api/ ./apps/api/
RUN npm run build --workspace=apps/api

FROM node:22-alpine
WORKDIR /app
COPY --from=builder /app/apps/api/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
EXPOSE 3001
CMD ["node", "dist/server.js"]
```

**Testing:**
- `docker compose up` starts the full stack (PostgreSQL, MinIO, Redis, API, web) and passes health checks.
- API container starts in <5 seconds and responds to health check endpoint.
- Web container serves the Next.js application on port 3000.
- Database migrations run automatically on container start via entrypoint script.
- Environment variables for secrets (JWT_SECRET, DB password, API keys) are read from environment, not baked into images.
- `docker compose down && docker compose up` (restart) preserves data in volumes.

---

### Task 12.5: Legal Disclaimers & Liability

**What:** Implement required legal disclaimers for deadline calculations, FTO analysis, and AI recommendations.

**Design:**

```typescript
// apps/api/src/shared/constants/disclaimers.ts
export const DISCLAIMERS = {
  deadlineCalculation: `IMPORTANT: Deadline calculations are provided as a convenience
    tool and must be independently verified by a qualified patent attorney before
    reliance. Errors in deadline calculations can result in irreversible loss of
    patent rights. The IP Management Platform and its contributors accept no
    liability for missed deadlines or incorrect calculations.`,

  ftoAnalysis: `DISCLAIMER: This preliminary freedom-to-operate analysis is
    generated by AI and does not constitute legal advice. It is intended as a
    starting point for discussion with qualified patent counsel. No business
    decisions should be made solely on the basis of this analysis.`,

  annuityRecommendation: `NOTE: AI-generated pay/abandon recommendations are
    advisory only. Annuity and maintenance fee decisions should be reviewed
    and approved by qualified IP counsel. The IP Management Platform accepts
    no liability for decisions made based on AI recommendations.`,
};
```

**Testing:**
- Deadline computation API responses include the deadline disclaimer in the response body.
- FTO analysis reports always include the FTO disclaimer.
- Annuity decision queue displays the annuity recommendation disclaimer.
- Disclaimers are displayed in the UI near the relevant data.
- Terms of service page includes all liability limitations.

---

## Definition of Done

A phase is considered complete when ALL of the following criteria are met:

### Code Quality
- [ ] All tasks in the phase are implemented and code-reviewed
- [ ] TypeScript strict mode passes with zero errors
- [ ] ESLint passes with zero warnings
- [ ] No `any` types except in explicitly justified locations

### Testing
- [ ] Unit test coverage >= 80% for business logic (services, domain functions)
- [ ] Unit test coverage >= 95% for the deadline-engine package (Phase 3+)
- [ ] Integration tests pass for all API endpoints
- [ ] E2E tests pass for critical UI workflows (Playwright)
- [ ] All edge cases documented in tests (leap years, month boundaries, timezone handling)

### Security
- [ ] No secrets committed to version control
- [ ] API endpoints validate and sanitise all inputs (Zod schemas)
- [ ] Tenant isolation verified: cross-org data access is impossible
- [ ] Audit log entries exist for all state-changing operations
- [ ] Dependencies scanned for vulnerabilities (npm audit, Snyk)

### Documentation
- [ ] OpenAPI 3.1 specification is up to date for all API endpoints
- [ ] Architecture Decision Records (ADRs) exist for significant design choices
- [ ] JSONB column schemas are documented in the `schema_definitions` table
- [ ] README updated with setup instructions for the current phase

### Operations
- [ ] Docker Compose configuration works for local development
- [ ] Database migrations are reversible (up and down)
- [ ] Health check endpoint returns correct status
- [ ] Error responses follow consistent format with correlation IDs

### Legal (Phases 3+)
- [ ] Deadline calculations include the required disclaimer
- [ ] FTO analysis includes the required disclaimer
- [ ] AI recommendations include the required disclaimer
- [ ] Terms of service reflect current feature set
