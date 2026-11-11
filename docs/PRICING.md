# Pricing, Entitlements, and MRR Accounting Model

**Product:** QUOTALEDGER  
**Primary Billing Currency:** USD (with INR billing support via Razorpay and multi-currency workspace calculation ledgers in USD, INR, EUR, GBP, CAD, AUD, SGD, JPY).

---

## 1. Published SaaS Subscription Tiers

| Tier | Monthly Billing | Annual Billing (15% Discount Stated Separately) | Representative Cap | Active Plans | Key Capabilities & Server-Enforced Entitlements |
| --- | --- | --- | --- | --- | --- |
| **Starter** | `$249 / mo` (Rs 20,900 / mo) | `$212 / mo` (`$2,544 / yr`, saves `$444 / yr`) | Up to **10** reps | **1** active plan | Flat & tiered commissions, CSV/XLSX deal imports, deterministic calculation runs, individual employee PDF/CSV statements, basic audit log. |
| **Growth** | `$699 / mo` (Rs 58,900 / mo) | `$594 / mo` (`$7,128 / yr`, saves `$1,260 / yr`) | Up to **35** reps | **10** active plans | Marginal bracket accelerators, splits, ramps, draws, SPIFFs, clawbacks, Manager Review -> Finance Approval workflows, employee dispute desk, HubSpot/Stripe/Slack adapters, Explainable Commission Intelligence. |
| **Scale** | `$1,299 / mo` (Rs 1,09,900 / mo) | `$1,104 / mo` (`$13,248 / yr`, saves `$2,340 / yr`) | Up to **100** reps | **Unlimited** plans | Multi-team manager overrides, multi-currency FX tables, source-to-ledger reconciliation reports, scenario forecasting, REST API + key rotation, signed outbound webhooks, priority support. |
| **Enterprise** | `$2,500+ / mo` (Custom quote) | Custom annual agreement | **100+** negotiated seats | **Unlimited** plans | Custom seat blocks, dedicated migration & parallel-run audit onboarding, custom retention policies, dedicated SLA. |

---

## 2. Server-Side Entitlement Enforcement (`src/services/entitlements.ts`)

Entitlements are never enforced solely by hiding UI buttons:
- Creating or activating a representative checks `active_rep_count < plan_entitlements.max_representatives`.
- Publishing an additional active compensation plan checks `active_plan_count < plan_entitlements.max_active_plans`.
- Minting REST API keys or outbound webhooks checks `plan_entitlements.api_access === true` (`Scale` and `Enterprise`, plus explicit preview evaluation in labeled demo workspaces).

---

## 3. Commercial Target Mathematics (Business Objective, Not Forecast)

QUOTALEDGER is priced so a focused B2B customer base can sustain `$5,000+` in Monthly Recurring Revenue without requiring hundreds of low-ARPU accounts:

- `4` Growth workspaces x `$699 / month` = `$2,796`
- `2` Scale workspaces x `$1,299 / month` = `$2,598`
- **Total Target Cohort:** `6` paying B2B customers = **`$5,394 MRR`** (`$64,728 ARR`)

This is an explicit unit-economics target scenario, not claimed or projected revenue.

---

## 4. Strict MRR Accounting & Demo Honesty Rules

1. **Recognized MRR (`recognized_mrr_cents`):** Computed exclusively from subscriptions where:
   - `provider` is `STRIPE` or `RAZORPAY` (never `DEMO_NON_CHARGING`), AND
   - `status === 'ACTIVE'` (verified via signed provider webhook), AND
   - Payment collection succeeded.
2. **Annual Normalization:** Annual subscriptions contribute exactly `Math.round(annual_amount_cents / 12)` to monthly normalized MRR.
3. **Strict Exclusions:** Free trials (`TRIALING`), unpaid or failed renewals (`PAST_DUE`, `UNPAID`), canceled subscriptions (`CANCELED`), one-time onboarding fees, and `DEMO_NON_CHARGING` workspaces always contribute **`$0.00`** to recognized MRR.
4. **Missing Merchant Keys:** When `STRIPE_SECRET_KEY` / `RAZORPAY_KEY_ID` are absent in an environment, `/app/billing` renders an explicit **Non-Charging Demo Billing Banner** so evaluators can test plan switching and entitlement gates without entering card details or creating false revenue records.
