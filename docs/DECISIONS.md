# Decision Log

Newest first. Format: date · decision · why · alternatives rejected. A superseded decision stays in the log, marked **Superseded by D-xxx**.

### D-013 · 2026-11-08 · Notifications and audit entries written by database triggers
**Why:** an action can't forget them and users can't forge them (`notify()` / `log_audit()` executable by the service role only). One place to read for "what notifies whom".
**Rejected:** calling `notify()`/`log_audit()` from each server action (easy to miss; callable by any user if exposed).

### D-012 · 2026-11-08 · Column- and transition-level rules in guard triggers + privileged RPCs
**Why:** RLS works per row, not per column. Without guards, an assignee could rewrite any task field and a document owner could approve their own document. Role changes and disabling run in `security definer` RPCs so rank rules live in SQL.
**Rejected:** Postgres column grants (apply to the whole `authenticated` role, can't differ between owner and reviewer); service-role writes for all admin actions (moves security out of the DB).

### D-011 · 2026-11-08 · Email links use `token_hash` + `verifyOtp`, not PKCE code exchange
**Why:** invites are sent by the server and links are often opened on another device, so the browser has no PKCE verifier and `exchangeCodeForSession` would fail.
**Rejected:** default `?code=` PKCE links.

### D-010 · 2026-11-08 · Assignees can move their own tasks to Done; Review is a convention
**Why:** PRD acceptance criteria and E2E flow 4 require the employee to move a task to Done. A hard review gate adds friction for a small team. PMs can reopen any card; `completed_at` records when.
**Rejected:** only the PM can move Review → Done (contradicted the acceptance criteria). Revisit if quality issues appear (OPEN_QUESTIONS #15).

### D-009 · 2026-11-08 · Only super admins create or manage admins
**Why:** the old matrix ("admin: not super_admin") conflicted with the rank rule ("lower rank only"). Restricting admin management to the owner is the safer reading and leaves one rule: `can_manage_role()`.
**Rejected:** admins managing peer admins (one admin could disable every other admin).

### D-008 · 2026-11-08 · Host on Netlify free plan instead of Vercel Hobby — supersedes D-002
**Why:** Vercel Hobby is for personal, non-commercial use; an agency site that solicits paid work is commercial from day one. Netlify's free plan allows commercial use and supports Next.js (ISR, tag revalidation, draft mode) through its runtime. Keeps the $0 goal.
**Rejected:** Vercel Hobby (terms), Vercel Pro ($20/month — fine later, breaks $0 now), Cloudflare Workers via OpenNext (viable, but more cache configuration for ISR/tags).
**Revisit:** if the Netlify runtime lags behind the needed Next.js version, or usage exceeds the free allowance (OPEN_QUESTIONS #17).

### D-007 · 2026-11-08 · Mascot built from code primitives, original design
**Why:** zero asset cost, tiny bundle, easy to recolour when the design arrives, no IP risk.
**Rejected:** Spline embed (heavier, external runtime), purchased GLB (cost), copying an existing brand mascot (not allowed).

### D-006 · 2026-11-08 · Landing content fully CMS-driven, cached with tag revalidation
**Why:** admin must edit the site without a deploy; caching keeps Supabase usage inside the free tier.
**Rejected:** headless CMS (Sanity/Contentful — extra service, limits), hard-coded content.

### D-005 · 2026-11-08 · Server actions instead of a REST API
**Why:** less code, type-safe, one deploy. Route handlers only where HTTP is required (lead form, email link handler, draft mode, revalidate webhook).
**Rejected:** separate Express/Nest backend.

### D-004 · 2026-11-08 · Database as the security boundary, mirrored in `lib/permissions.ts`
**Why:** a bug in UI or an action cannot leak data; the database refuses.
**Rejected:** app-only checks with the service-role key everywhere.

### D-003 · 2026-11-08 · Accounts by invitation only
**Why:** requirement — admin creates staff accounts; reduces abuse surface.
**Rejected:** open signup with approval queue.

### D-002 · 2026-11-08 · Hosting on Vercel Hobby at launch — **Superseded by D-008**
**Why (original):** best Next.js support, free. Superseded because Hobby terms are non-commercial.

### D-001 · 2026-11-08 · Next.js + Supabase, single app
**Why:** budget is tight; Supabase free tier covers Postgres, auth, storage, realtime, cron; one Next.js app serves site and portal.
**Rejected:** Next.js + own Postgres/Prisma/Auth.js (more code), Laravel (team preference unknown, two runtimes).
