# Security Policy & Threat Model

**Product:** QUOTALEDGER (`devtechedge/quotaledger`)  
**Last Reviewed:** 2026-11-11

---

## 1. Supported Versions & Vulnerability Reporting

| Version | Supported |
| --- | --- |
| `1.x` (`main`) | Yes |

If you discover a security vulnerability, tenant-isolation bypass, or financial calculation integrity defect, please report it privately via GitHub Security Advisories on `https://github.com/devtechedge/quotaledger/security/advisories` rather than opening a public issue containing exploit details.

---

## 2. Threat Model & Implemented Controls

### A. Multi-Tenant Data Isolation (Cross-Tenant & IDOR Prevention)
- Every tenant-owned table (`representatives`, `teams`, `territories`, `compensation_plans`, `plan_versions`, `quota_assignments`, `deals`, `deal_splits`, `import_batches`, `payout_periods`, `calculation_runs`, `calculation_line_items`, `statements`, `approval_events`, `disputes`, `adjustments`, `subscriptions`, `integration_connections`, `workspace_api_keys`, `audit_events`) includes a non-null `tenant_id` foreign key.
- All database reads, mutations, exports, and assistant queries derive `tenant_id` and `role` exclusively from the verified server-side session (`ql_session` cookie -> SHA-256 lookup in `sessions`) or workspace API key (`ql_live_*` -> SHA-256 lookup in `workspace_api_keys`).
- Client-supplied `tenantId` or `role` fields are never trusted for authorization.
- Users with the `REPRESENTATIVE` role are additionally scoped at the SQL filter layer to their linked `representative_id`, preventing horizontal privilege escalation (IDOR) across sibling reps in the same company.

### B. Authentication & Session Security
- **Passwords:** Hashed with `crypto.scryptSync` (`N=16384, r=8, p=1`, 64-byte derived key) with a unique 16-byte cryptographic salt per user and verified via `crypto.timingSafeEqual`.
- **Sessions:** Raw 32-byte random tokens are set solely in `httpOnly`, `SameSite=Lax`, `Secure` (in production) cookies; only the SHA-256 digest (`token_hash`) is persisted in PostgreSQL.
- **Session Revocation:** Password changes revoke sibling sessions; users and workspace owners can revoke individual or all active sessions immediately from `/app/settings`.
- **Demo Accounts:** Public synthetic demo accounts (`*@quotaledger.demo`) are strictly gated behind `APP_ENV=demo`. Setting `APP_ENV=production` returns `403 Forbidden` on demo persona endpoints and requires explicit `SESSION_SECRET`, `ENCRYPTION_KEY`, and `DATABASE_URL`.

### C. Encryption of Integration Secrets at Rest
- Third-party OAuth tokens, webhook signing secrets, and connector credentials stored in `integration_connections` are encrypted before persistence using **AES-256-GCM** (`iv:authTag:ciphertext` base64url envelope) keyed by server-only `ENCRYPTION_KEY`.
- API responses return only a masked preview and key fingerprint, never raw secret material.

### D. Signed Statement Downloads & Replay Protection
- Download links for private CSV/printable employee statements use short-lived HMAC-SHA256 signatures binding `(tenantId, statementId, actorUserId, expiresAt)` verified with `crypto.timingSafeEqual`.
- Mutating browser API routes enforce same-origin `Origin`/`Host` verification (CSRF defense) and hashed-IP sliding-window rate limiting.
- Stripe and Razorpay billing webhooks verify cryptographic provider signatures against the raw request payload and enforce event-ID idempotency via `billing_webhook_events` (`UNIQUE(provider, external_event_id)`).

### E. Financial Integrity & Auditability
- Monetary calculations execute exclusively in `bigint` minor units (zero IEEE-754 floating-point math).
- Period locks (`LOCKED`, `PAID`) execute inside database transactions and block silent edits to underlying deals or approved statements. Reopening a locked period requires `OWNER`, `ADMIN`, or `FINANCE` role plus a mandatory reason recorded in both `approval_events` and append-only `audit_events`.

---

## 3. Accepted Residual Risks on Public Demo Deployment

- On the public evaluation deployment (`APP_ENV=demo`), pre-seeded `@quotaledger.demo` accounts share a documented workspace so visitors can inspect multi-role workflows immediately without email verification friction. Do not enter confidential company or PII records into the shared public demo workspace; self-serve signup creates an isolated tenant workspace.
- Email delivery (`RESEND_API_KEY`), live payment charging (`STRIPE_SECRET_KEY`, `RAZORPAY_KEY_SECRET`), and optional LLM summarization (`OPENAI_API_KEY`) default to deterministic local/demo sinks until production credentials are supplied.
