# Security

## Principles
1. **RLS is the boundary.** Every table has RLS enabled; policies match [ROLES_PERMISSIONS.md](ROLES_PERMISSIONS.md). UI checks are only for UX.
2. **Least privilege keys.** Browser uses the anon key + user session. Service-role key exists only in `lib/supabase/admin.ts` (imports `server-only`) and Edge Functions.
3. **Validate at every boundary** with zod: forms, server actions, route handlers, CMS JSON.

## Checklist (must pass before each release)
- [ ] No secrets in the repo; `.env*` git-ignored; `.env.example` has names only.
- [ ] No `NEXT_PUBLIC_` variable contains a secret.
- [ ] Public signup disabled in Supabase Auth settings.
- [ ] RLS enabled on all tables (`select tablename from pg_tables where schemaname='public' and not rowsecurity` returns nothing).
- [ ] RLS test suite passes for every role.
- [ ] Storage policies enforce path ownership; private buckets never public.
- [ ] Signed URLs expire in ≤ 60 s.
- [ ] File uploads: MIME + extension + size checked client and server side; SVG uploads sanitised; filenames slugified.
- [ ] Rich text sanitised with DOMPurify allow-list before storing **and** rendering.
- [ ] Lead form: Turnstile verified server-side, honeypot field, 3 per IP per 10 min, IP stored only as salted hash.
- [ ] Auth: password ≥ 10 chars with strength check; generic login errors; reset flow does not reveal whether an email exists.
- [ ] `/auth/callback` `next` param only allows relative `/portal/*` paths (no open redirect).
- [ ] Rank rules: no self role change, no escalation above own rank, last super admin protected (DB trigger).
- [ ] Disabled users: banned in Auth and blocked by RLS helpers.
- [ ] Security headers in `next.config.ts`: CSP (allow Supabase, Turnstile, Resend not needed client-side), HSTS, X-Content-Type-Options, Referrer-Policy, Permissions-Policy, frame-ancestors 'none'.
- [ ] Server actions return generic errors; details logged server-side only.
- [ ] Audit log written for role changes, disable, document review, CMS publish, lead status.
- [ ] `npm audit` has no high/critical issues.
- [ ] Supabase: enable leaked-password protection and email confirmation; set JWT expiry 1 h with refresh.

## Personal data
Employee documents (ID proofs, contracts) are sensitive.
- Stored only in the private `employee-docs` bucket.
- Visible only to the owner and super_admin/admin/hr.
- Rejected/replaced files are deleted, not kept.
- When a user is offboarded, admin can export and then delete their documents (manual step in v1).
- Privacy policy on the site must describe lead data use.
