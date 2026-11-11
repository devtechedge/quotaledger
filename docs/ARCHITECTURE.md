# System Architecture

**Product:** QUOTALEDGER  
**Stack:** Next.js 16.4 (App Router), React 19.3, TypeScript 5.9 (`strict: true`), PostgreSQL (Neon in production, PostgreSQL 17 in CI, embedded PGLite local zero-config fallback), Drizzle ORM 0.45, Zod 4, Tailwind CSS 4.

---

## 1. Architectural Principles

1. **Tenant Isolation by Construction:** Every domain table carries a non-null `tenant_id` foreign key indexed as the leading column of composite lookup keys. Workspace context is resolved exclusively on the server from the `httpOnly` session cookie (or workspace API key on `/api/v1/*`), never from client-supplied headers or query parameters.
2. **Deterministic, Pure Financial Core (`src/engine/`):** The commission calculation engine is a zero-I/O, deterministic pure TypeScript module operating strictly on `bigint` minor units and parts-per-million (`10^6`) fixed-point rates. Database services hydrate an immutable input snapshot, invoke the pure engine, verify invariants, and persist the resulting calculation run, line items, and rep statements inside a single database transaction.
3. **Append-Only Financial Corrections:** Once a payout period enters `LOCKED` or `PAID`, underlying deal rows and approved statements cannot be silently mutated. Disputed payouts, late CRM reversals, or clawbacks either require an audited period reopen by `OWNER`/`ADMIN`/`FINANCE` with a mandatory reason or generate an explicit append-only `adjustments` / clawback entry applied to an open period.
4. **Grounded Intelligence Only:** The Commission Explanation Assistant (`src/services/intelligence.ts`) queries authorized calculation runs, line-item traces, exclusion logs, and plan versions first. Its deterministic citation engine produces exact line-by-line answers; when an optional server-side `OPENAI_API_KEY` is configured, the LLM may only summarize the retrieved citation bundle and can never compute or alter monetary figures.
5. **Honest External Adapters:** Billing (Stripe, Razorpay) and third-party integrations (HubSpot, Salesforce, Slack, Google Sheets, Outbound Webhooks) report their genuine configuration and verification state. Missing merchant or OAuth credentials activate an explicit **Non-Charging Demo Mode** / **Setup Required** state rather than fabricated green checkmarks.

---

## 2. Layered Module Topology

```text
src/
  env.ts                      # Zod-validated server/public env & demo/prod mode gates
  lib/
    crypto.ts                 # scrypt password hashing, AES-256-GCM vault, HMAC signed URLs, SHA-256
    tabular-parser.ts         # Deterministic CSV & XLSX workbook parser (ZIP/XML + worksheet reader)
    plan-templates.ts         # Curated B2B SaaS compensation plan templates
    format.ts                 # Currency minor-unit & rate formatters (tabular-nums safe)
  engine/                     # Pure, zero-float financial calculation engine
    money.ts                  # BigInt minor units, ppm rates, HALF_EVEN / HALF_UP, largest-remainder
    fx-and-splits.ts          # Documented FX conversion & multi-rep split credit allocation
    types.ts                  # Canonical PlanRuleConfig, EngineInputSnapshot, CalculationTrace types
    plan-validator.ts         # Zod + semantic invariant validator & plain-English plan explainer
    tiers.ts                  # Flat, marginal (bracket-straddling), and cliff accelerator evaluators
    ramps-and-draws.ts        # New-hire quota relief, recoverable/non-recoverable draws, caps, cliffs
    spiffs-clawbacks-overrides.ts # Windowed SPIFFs, refund/churn clawbacks, manager team overrides
    calculator.ts             # Period orchestrator + SHA-256 input/output reproducibility verifier
  db/
    schema/                   # Drizzle PostgreSQL schema (core, org, ledger, payouts, commercial)
    seeds/                    # Deterministic synthetic workspace seed (Acme Cloud Systems)
    client.ts                 # Pooled node-postgres (Neon/PG) + PGLite fallback + fresh test DB factory
    bootstrap.ts              # Generated DDL executor for embedded PGLite
    url.ts                    # libpq SSL normalization
  auth/
    session.ts                # httpOnly SHA-256 cookie sessions, workspace switching, revocation
    rbac.ts                   # Role matrix (OWNER, ADMIN, FINANCE, SALES_MANAGER, REPRESENTATIVE, ANALYST)
  services/                   # Transactional tenant-scoped domain services
    request-guard.ts          # Same-origin CSRF check & hashed-IP sliding-window rate limiter
    audit.ts                  # Append-only audit event writer with secret masking
    auth.ts                   # Signup, login, password change, recovery tokens, demo persona sign-in
    entitlements.ts           # Plan tier limits (Starter/Growth/Scale/Enterprise seats, plans, features)
    workspace.ts              # Onboarding 9-step state, invitations, data export, typed-confirm deletion
    plans.ts                  # Plan creation, immutable version publishing, sandbox simulator
    representatives-and-quotas.ts # Roster hierarchy, ramp schedules, quota assignments
    deals-and-imports.ts      # Manual deals, reversals, CSV/XLSX field mapper, idempotent import batches
    reconciliation.ts         # Imported source totals vs eligible commissionable revenue & exclusions
    commissions.ts            # Calculation run execution, statement generation, SHA-256 replay check
    workflows.ts              # Draft -> Calculated -> Manager Review -> Finance Approved -> Locked -> Paid
    intelligence.ts           # Grounded Q&A over calculation traces, what-if quota simulations, audit logs
    billing.ts                # Stripe + Razorpay checkout/webhooks + non-charging demo + strict MRR math
    integrations.ts           # AES-256-GCM token storage, connection test probes, webhook delivery + DLQ
    leads.ts                  # Demo requests, contact inquiries, referral attribution, outbound CSV export
  components/                 # Terminal UI primitives, theme toggle, charts, interactive calculators
  app/                        # 13 public routes, auth/onboarding flows, 20 /app/* routes, REST APIs
```

---

## 3. Request & Authorization Lifecycle

1. **Edge / Header Layer (`next.config.ts`):** Applies `Content-Security-Policy`, `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, `Referrer-Policy: strict-origin-when-cross-origin`, `Permissions-Policy`, and production `Strict-Transport-Security`.
2. **Mutating Route Guard (`src/services/request-guard.ts`):** Validates `Origin` against the request `Host` (CSRF mitigation) except on externally signed webhook endpoints (`/api/billing/webhooks/*`), which verify provider HMAC signatures (`Stripe-Signature`, `X-Razorpay-Signature`) against raw request bodies.
3. **Session & Tenant Resolution (`src/auth/session.ts`):** Reads the `ql_session` `httpOnly` cookie, hashes the raw bearer token with SHA-256, joins `sessions -> users -> memberships -> tenants`, verifies `expires_at > now()` and `revoked_at IS NULL`, and returns an immutable `WorkspaceActor` context.
4. **Role & Row-Level Scoping (`src/auth/rbac.ts`):**
   - `OWNER` / `ADMIN`: Full tenant control.
   - `FINANCE`: Full financial operations, period lock/reopen, approvals, dispute resolution, reports, audit read.
   - `SALES_MANAGER`: Scoped to own representative profile plus direct-report team members for deals, statements, disputes, and `MANAGER_REVIEW` approvals.
   - `REPRESENTATIVE`: Strictly restricted at the query filter level (`WHERE tenant_id = $1 AND representative_id = $2`) so a representative can never enumerate or inspect another representative's deals, quotas, calculation items, statements, or disputes.
   - `ANALYST`: Read-only access across tenant reporting, ledger, attainment, and reconciliation surfaces; all mutating endpoints return `403 Forbidden`.

---

## 4. Database & Runtime Deployment Modes

- **Production (`Vercel + Neon PostgreSQL`):** Uses `DATABASE_URL` (PgBouncer pooled connection) at runtime and `DATABASE_URL_UNPOOLED` (direct connection) for `drizzle-kit` migrations via `npm run db:migrate`.
- **CI (`GitHub Actions + PostgreSQL 17 Service`):** Runs migrations, seeds deterministic synthetic fixtures, executes unit + SQL integration tests, builds the production Next.js bundle, and runs Playwright end-to-end journeys + WCAG 2.2 AA axe checks against `next start`.
- **Zero-Config Local Development (`Embedded PGLite`):** When `DATABASE_URL` is omitted locally, `src/db/client.ts` boots an in-process `@electric-sql/pglite` instance using `src/db/ddl.ts` (auto-generated directly from `drizzle/*.sql`) and seeds the deterministic demo workspace so `npm run dev` works immediately after clone.
