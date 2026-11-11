# Financial Data Model & Ledger Invariants

**ORM:** Drizzle ORM (`src/db/schema/*`)  
**Database:** PostgreSQL 17 / Neon Lakebase Postgres (with 1:1 DDL compatibility for embedded PGLite)

---

## 1. Monetary, Rate, and FX Representation Rules

1. **No Floating-Point Columns:** All monetary fields (`amount_minor`, `eligible_amount_minor`, `quota_amount_minor`, `gross_commission_minor`, `net_payout_minor`, `draw_balance_minor`, `adjustment_amount_minor`) are stored as 64-bit signed integers (`BIGINT` in PostgreSQL, mapped to `bigint` in TypeScript) representing the currency's smallest minor unit:
   - `USD`, `EUR`, `GBP`, `CAD`, `AUD`, `SGD`: `100` minor units = `1.00` major unit (`$12,500.00` -> `1_250_000n`).
   - `INR`: `100` paise = `1.00` INR (`Rs 8,50,000.00` -> `85_000_000n`).
   - `JPY`: `1` minor unit = `1` JPY (`exponent = 0`).
2. **Explicit ISO-4217 Currency Pairing:** Every monetary column is accompanied by an explicit 3-letter `currency` column (`USD`, `INR`, `EUR`, `GBP`, etc.). Arithmetic across mismatched currencies throws `CurrencyMismatchError` unless converted through an explicit `fx_rates` record.
3. **Rates & Split Weights in Parts-Per-Million (`ppm`, `INTEGER` / `BIGINT`):**
   - `100%` = `1_000_000` ppm (`1%` = `10_000` ppm, `8.5%` = `85_000` ppm, `0.25%` = `2_500` ppm).
4. **FX Conversion Rates in `10^8` Fixed-Point (`rate_scaled`, `BIGINT`):**
   - `1.00000000` = `100_000_000n`. Every conversion records `source_currency`, `target_currency`, `rate_scaled`, `rate_source` (e.g. `RBI_REFERENCE`, `ECB_DAILY`, `WORKSPACE_TREASURY_LOCK`), and `effective_at`.
5. **Split Residual Allocation:** Multi-rep deal splits use deterministic **Largest-Remainder Allocation** ordered by `(split_ppm DESC, representative_id ASC)` so `SUM(allocated_minor) === deal.eligible_amount_minor` for every split deal.

---

## 2. Core Entity Catalog (26 Tables)

### A. Multi-Tenancy, Identity & Governance (`src/db/schema/core.ts`)
- `tenants`: Workspace root (`id`, `slug` unique, `name`, `industry`, `base_currency`, `fiscal_year_start_month`, `onboarding_state_json`, `is_demo_workspace`, `created_at`).
- `users`: Global user principals (`id`, `email` unique lowercase, `full_name`, `password_hash`, `is_demo_user`, `created_at`).
- `memberships`: Tenant-scoped RBAC link (`id`, `tenant_id`, `user_id`, `role` in `OWNER | ADMIN | FINANCE | SALES_MANAGER | REPRESENTATIVE | ANALYST`, `representative_id` nullable link for rep/manager self-service, unique `(tenant_id, user_id)`).
- `sessions`: Stateful revocable sessions (`id`, `token_hash` unique SHA-256, `user_id`, `tenant_id`, `ip_hash`, `user_agent`, `expires_at`, `revoked_at`, `created_at`).
- `invitations`: Organization invites (`id`, `tenant_id`, `email`, `role`, `token_hash`, `invited_by_user_id`, `expires_at`, `accepted_at`).
- `password_recovery_tokens`: Single-use time-boxed recovery tokens (`id`, `user_id`, `token_hash`, `expires_at`, `used_at`).
- `rate_limit_buckets`: Sliding-window abuse protection (`key`, `count`, `window_started_at`).
- `audit_events`: Append-only compliance log (`id`, `tenant_id`, `actor_user_id`, `actor_role`, `action`, `entity_type`, `entity_id`, `summary`, `metadata_json`, `ip_hash`, `created_at`).

### B. Organization Structure, Quotas & FX (`src/db/schema/org.ts`)
- `territories`: Sales regions (`id`, `tenant_id`, `code`, `name`, `region_group`, `rate_modifier_ppm`, unique `(tenant_id, code)`).
- `teams`: Functional pods (`id`, `tenant_id`, `code`, `name`, `manager_rep_id`, `team_bonus_threshold_ppm`, `team_bonus_amount_minor`, `currency`).
- `products`: Commissionable catalog (`id`, `tenant_id`, `sku`, `name`, `revenue_Category` in `RECURRING_ARR | ONE_TIME_SERVICES | USAGE`, `commissionable_pct_ppm`, `rate_modifier_ppm`).
- `fx_rates`: Documented currency table (`id`, `tenant_id`, `from_currency`, `to_currency`, `rate_scaled`, `source_label`, `effective_date`).
- `representatives`: Commission-eligible personnel (`id`, `tenant_id`, `employee_code`, `full_name`, `email`, `title`, `role_type` in `AE | SDR | BDR | SE | MANAGER | CHANNEL`, `team_id`, `territory_id`, `manager_rep_id`, `payout_currency`, `hire_date`, `ramp_months`, `active`, unique `(tenant_id, email)`).
- `quota_assignments`: Period/annual targets (`id`, `tenant_id`, `representative_id`, `period_key`, `cadence` in `MONTHLY | QUARTERLY | ANNUAL`, `quota_amount_minor`, `currency`, `ramp_override_ppm`, `notes`, unique `(tenant_id, representative_id, period_key)`).

### C. Compensation Plans, Deal Ledger & Imports (`src/db/schema/ledger.ts`)
- `compensation_plans`: Plan header (`id`, `tenant_id`, `code`, `name`, `description`, `target_role_type`, `payout_currency`, `status` in `DRAFT | ACTIVE | ARCHIVED`, `active_version_number`).
- `plan_versions`: Immutable published rule definitions (`id`, `tenant_id`, `plan_id`, `version_number`, `effective_from`, `effective_to`, `rules_json`, `rules_hash` SHA-256, `published_by_user_id`, `published_at`, unique `(plan_id, version_number)`).
- `rep_plan_assignments`: Effective-dated enrollment (`id`, `tenant_id`, `representative_id`, `plan_id`, `plan_version_id`, `effective_from`, `effective_to`).
- `import_batches`: CSV/XLSX upload audit (`id`, `tenant_id`, `filename`, `source_type`, `mapping_json`, `total_rows`, `inserted_rows`, `duplicate_rows`, `invalid_rows`, `source_total_minor`, `currency`, `imported_by_user_id`, `created_at`).
- `deals`: Authoritative revenue ledger (`id`, `tenant_id`, `source_deal_id`, `account_name`, `deal_name`, `primary_rep_id`, `manager_rep_id`, `product_id`, `territory_id`, `deal_type` in `NEW_BUSINESS | EXPANSION | RENEWAL`, `revenue_type` in `RECURRING | ONE_TIME`, `close_date`, `period_key`, `amount_minor`, `eligible_amount_minor`, `currency`, `payment_status` in `PAID | INVOICED | UNPAID | REFUNDED | CHURNED_CLAWBACK`, `contract_months`, `original_deal_id` for reversals/clawbacks, `is_locked`, `import_batch_id`, unique `(tenant_id, source_deal_id)`).
- `deal_splits`: Co-selling attribution (`id`, `tenant_id`, `deal_id`, `representative_id`, `split_role` in `PRIMARY | OVERLAY | CHANNEL | SDR_SOURCE`, `split_ppm`).

### D. Payout Periods, Calculation Snapshots, Statements & Workflows (`src/db/schema/payouts.ts`)
- `payout_periods`: Governance state machine (`id`, `tenant_id`, `period_key`, `label`, `start_date`, `end_date`, `status` in `DRAFT | CALCULATED | MANAGER_REVIEW | FINANCE_APPROVED | LOCKED | PAID`, `latest_run_id`, `locked_at`, `locked_by_user_id`, `reopen_reason`, unique `(tenant_id, period_key)`).
- `calculation_runs`: Immutable execution header (`id`, `tenant_id`, `payout_period_id`, `period_key`, `run_number`, `engine_version`, `input_snapshot_json`, `input_fingerprint`, `output_fingerprint`, `total_eligible_minor`, `total_gross_minor`, `total_adjustments_minor`, `total_net_minor`, `currency`, `executed_by_user_id`, `created_at`).
- `calculation_line_items`: Line-by-line deal/bonus/override/clawback traces (`id`, `tenant_id`, `run_id`, `representative_id`, `deal_id`, `plan_version_id`, `component_type` in `DEAL_COMMISSION | ACCELERATOR_KICKER | SPIFF_BONUS | TEAM_BONUS | MANAGER_OVERRIDE | DRAW_TOPUP | DRAW_RECOVERY | CLAWBACK_REVERSAL | MANUAL_ADJUSTMENT`, `source_amount_minor`, `source_currency`, `fx_rate_scaled`, `fx_source`, `credited_amount_minor`, `quota_snapshot_minor`, `prior_attainment_ppm`, `post_attainment_ppm`, `effective_rate_ppm`, `payout_amount_minor`, `payout_currency`, `explanation`, `trace_json`).
- `statements`: Individual employee payout statements (`id`, `tenant_id`, `payout_period_id`, `run_id`, `representative_id`, `period_key`, `plan_version_id`, `quota_amount_minor`, `credited_revenue_minor`, `attainment_ppm`, `deal_commission_minor`, `bonuses_minor`, `overrides_minor`, `clawbacks_minor`, `draw_net_minor`, `adjustments_minor`, `net_payout_minor`, `currency`, `status`, `statement_hash`, unique `(run_id, representative_id)`).
- `approval_events`: Period & statement sign-off chain (`id`, `tenant_id`, `payout_period_id`, `from_status`, `to_status`, `decision` in `SUBMITTED | APPROVED | REJECTED | LOCKED | REOPENED | MARKED_PAID`, `actor_user_id`, `actor_role`, `comment`, `attachment_meta_json`, `created_at`).
- `disputes` & `adjustments`: Employee inquiries & append-only ledger corrections (`disputes`: `id`, `tenant_id`, `statement_id`, `deal_id`, `raised_by_rep_id`, `reason_code`, `subject`, `description`, `claimed_delta_minor`, `currency`, `status` in `OPEN | UNDER_REVIEW | APPROVED_ADJUSTMENT | UPHELD | WITHDRAWN`, `resolution_note`, `resolved_by_user_id`, `evidence_json`; `adjustments`: `id`, `tenant_id`, `representative_id`, `target_period_key`, `dispute_id`, `amount_minor`, `currency`, `reason`, `approved_by_user_id`, `applied_in_run_id`, `created_at`).

### E. SaaS Billing, Integrations, API Keys & Growth (`src/db/schema/commercial.ts`)
- `subscriptions` & `billing_webhook_events`: Real recurring subscription records (`provider` in `STRIPE | RAZORPAY | DEMO_NON_CHARGING`, `plan_tier` in `STARTER | GROWTH | SCALE | ENTERPRISE`, `billing_interval` in `MONTHLY | ANNUAL`, `status` in `TRIALING | ACTIVE | PAST_DUE | CANCELED | DEMO`, `seat_limit`, `mrr_cents` strictly `0` unless `provider != DEMO_NON_CHARGING` and `status == ACTIVE`).
- `integration_connections`, `integration_sync_logs`, `workspace_api_keys`, `intelligence_queries`, and `marketing_leads`.
