# Ideal Customer Profile, Onboarding, and Go-To-Market Playbook

**Product:** QUOTALEDGER  
**Positioning:** The explainable, finance-grade commission ledger for 10-100 rep B2B SaaS and technology revenue teams graduating from brittle spreadsheets.

---

## 1. Ideal Customer Profile (ICP)

- **Company Profile:** B2B SaaS, developer tools, cybersecurity, fintech, and B2B technology companies with **10 to 100 commission-eligible employees** (typically `$2M` to `$35M ARR`, Series A through Series C, or profitable bootstrapped SaaS).
- **Economic & Champion Buyers:**
  - **VP Finance / Controller / CFO:** Owns payroll sign-off, audit readiness, clawback recovery, and payout accuracy. Hates black-box numbers and last-minute spreadsheet formula bugs on the 5th business day of the month.
  - **Head of Revenue Operations / Sales Ops:** Owns compensation plan rollout, CRM export hygiene, territory/quota updates, and answering rep "how was this calculated?" Slack messages.
  - **VP of Sales:** Wants reps focused on closing pipeline rather than maintaining personal "shadow accounting" spreadsheets.

---

## 2. Self-Serve 9-Step Onboarding Sequence (`/onboarding` & `/app/overview`)

QUOTALEDGER is engineered so a RevOps manager or Finance Controller can run a parallel payout audit in under 30 minutes without waiting on a vendor implementation consultant:

1. **Create Organization:** Name workspace, slug, and fiscal year start month.
2. **Choose Industry & Base Currency:** Configure `USD`, `INR`, `EUR`, or `GBP` primary ledger currency plus documented FX conversion rates.
3. **Invite Representatives:** Add AE, SDR/BDR, SE, and Sales Manager profiles with hire dates and ramp durations.
4. **Import Historical Deals:** Upload a CRM CSV or XLSX export (or download the pre-mapped QUOTALEDGER CSV template) with preview validation and `source_deal_id` duplicate detection.
5. **Configure Quota & Compensation Plan:** Pick a proven B2B SaaS template (AE Marginal Accelerator, SDR Pipeline Bonus, Manager Team Override) or build custom brackets in the Plan Studio.
6. **Run Sample Commission Calculation:** Execute the deterministic engine and inspect line-by-line calculation traces.
7. **Review Representative Statements:** Open individual employee statements with quota attainment bars and PDF/CSV export.
8. **Invite Finance & Manager Approvers:** Assign `SALES_MANAGER` and `FINANCE` reviewers.
9. **Complete First Payout-Period Simulation:** Walk a period through `Calculated -> Manager Review -> Finance Approved -> Locked`.

---

## 3. Product-Led Acquisition & SEO Infrastructure

1. **Interactive Public Calculators (`/guides` & `/guides/[slug]`):**
   - `commission-calculator`: Marginal vs. cliff commission bracket calculator with instant deal breakdown.
   - `saas-sales-commission`: New business vs. expansion ACV, multi-year SPIFF, and OTE split planner.
   - `accelerator-calculator`: Quota attainment straddle analyzer showing exact marginal dollar lift above 100% and 125% quota.
   - `commission-statement`: Interactive anatomy of an audit-ready employee commission statement.
   - `sales-compensation-software`: Spreadsheet parallel-run checklist and vendor evaluation rubric.
2. **Instant Synthetic Demo Workspace (`/demo`):**
   - Visitors can launch directly into a populated multi-role SaaS workspace (**Acme Cloud Systems**, 8 sales personnel across AMER/EMEA/INDIA, multi-currency USD/EUR/INR deals, marginal accelerators, split deals, clawbacks, and an open rep dispute) as Owner, Finance Controller, Sales Manager, Account Executive, or Analyst.
3. **Lead Capture, Referral Attribution & CRM-Ready Export:**
   - `/demo` and `/contact` capture qualified company size, rep count, current tool (Excel/Sheets, CRM report, legacy ICM), and UTM/referral attribution (`?ref=`) directly into `marketing_leads`.
   - Workspace admins can export structured CRM-ready lead CSVs and outbound prospecting templates from `/app/reports`.
4. **Strict Commercial Honesty:**
   - Zero fabricated customer logos, zero invented testimonials, zero unverified ROI claims. Primary industry statistics cite their exact third-party publication and date (such as The SaaS CFO 1Q25 Tech Stack Survey, June 2025, n=637).
