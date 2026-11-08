# Security

## Principles
1. **The database is the boundary.** RLS on every table decides rows; guard triggers decide columns and state transitions; privileged RPCs hold rank rules. Policies match [ROLES_PERMISSIONS.md](ROLES_PERMISSIONS.md). UI checks are only for UX.
2. **Least-privilege keys.** The browser uses the anon key + user session. The service-role key exists only in `lib/supabase/admin.ts` (imports `server-only`), Edge Functions, and GitHub Actions secrets — and in the app it is used only after a DB permission check.
3. **Validate at every boundary** with zod: forms, server actions, route handlers, CMS JSON.
4. **Side effects can't be forged.** `notify()` and `log_audit()` are executable by `service_role` only; users cause them only indirectly through audited data changes.

## Checklist (must pass before each release)
**Secrets & config**
- [ ] No secrets in the repo; `.env*` git-ignored; `.env.example` has names only.
- [ ] No `NEXT_PUBLIC_` variable contains a secret.
- [ ] `lib/env.ts` validates all variables at startup.

**Supabase**
- [ ] Public signup disabled in Auth settings.
- [ ] Email templates use `/auth/confirm?token_hash=…` (invite + recovery).
- [ ] Redirect URL allow-list contains only the production and staging/preview domains.
- [ ] JWT expiry 1 h with refresh; minimum password length 10 in Auth settings.
- [ ] RLS enabled on all tables (`select tablename from pg_tables where schemaname='public' and not rowsecurity` returns nothing).
- [ ] All views in `public` use `security_invoker = true` (pgTAP hygiene test).
- [ ] Every `security definer` function sets `search_path` and has explicit grants; Supabase Security Advisor shows no errors.
- [ ] `notify`, `log_audit`, `hit_rate_limit` not executable by `anon`/`authenticated`.
- [ ] RLS + guard-trigger test suite passes for every role.

**Storage**
- [ ] Bucket `file_size_limit` and `allowed_mime_types` set; private buckets not public.
- [ ] Storage policies enforce path ownership.
- [ ] Signed URLs expire in ≤ 60 s (5 min for the offboarding export only).
- [ ] SVG accepted only through the sanitising server action; filenames slugified.

**App**
- [ ] Rich text sanitised with the DOMPurify allow-list before storing **and** when rendering.
- [ ] Lead form: zod, honeypot, Turnstile verified server-side, 3 per IP per 10 min + 50 global per hour, IP stored only as a salted hash.
- [ ] Auth: password ≥ 10 chars + zxcvbn ≥ 3; generic login errors; reset flow doesn't reveal whether an email exists.
- [ ] Redirect params (`next`, `path`) allow only relative paths with the expected prefix (no `//`, no scheme, no backslash) — no open redirect.
- [ ] Rank rules: no self role change/disable, admins cannot manage admins/super admins, last super admin protected.
- [ ] Disabled users: banned in Auth and blocked by RLS helpers.
- [ ] Draft mode only enabled for an admin session.
- [ ] CMS `collection` param checked against an allow-list.
- [ ] Security headers in `next.config.ts`: CSP (self, Supabase URL + realtime `wss`, `challenges.cloudflare.com` for Turnstile, `static.cloudflareinsights.com` for analytics), HSTS, X-Content-Type-Options, Referrer-Policy `strict-origin-when-cross-origin`, Permissions-Policy, `frame-ancestors 'none'`.
- [ ] Server actions return generic errors; details logged server-side only.
- [ ] Audit log populated for role changes, enable/disable, document review, CMS publish, lead status/owner, document purge, invites.
- [ ] `npm audit` has no high/critical issues; Dependabot enabled.

**Ops**
- [ ] Backup artifacts are encrypted; restore drill done this quarter ([DEPLOYMENT](DEPLOYMENT.md#backups)).
- [ ] GitHub repo private; Actions secrets set; branch protection on `main`.

> Supabase "leaked password protection" (HaveIBeenPwned check) is a **paid-plan** feature at the time of writing. On the free plan we rely on the length + zxcvbn check. Enable it when moving to Pro.

## Personal data
**Employee documents** (ID proofs, contracts) are the most sensitive data.
- Stored only in the private `employee-docs` bucket; visible only to the owner and super_admin/admin/hr.
- Rejected/replaced files are deleted, not kept.
- Offboarding: disable → export → purge (`purgeUserDocuments`, audited). See [APP_FLOW §3a](APP_FLOW.md#3a-offboarding).
- Encrypted storage backups are kept at most 90 days.

**Leads** (prospect name, email, company, message)
- Used only to respond to the inquiry; described in the privacy policy.
- `lost` leads older than 24 months are deleted during the quarterly review (automate post-v1); deletion on request via `deleteLead` (super_admin).
- IP addresses are never stored — only a salted hash, and only for rate limiting.

**Analytics:** Cloudflare Web Analytics is cookieless and collects no personal data; Turnstile is strictly necessary for security. No cookie banner needed; the privacy policy lists both.
