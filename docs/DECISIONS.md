# Decision Log

Newest first. Format: date · decision · why · alternatives rejected.

### D-007 · 2026-11-08 · Mascot built from code primitives, original design
**Why:** zero asset cost, tiny bundle, easy to recolour when the design arrives, no IP risk.
**Rejected:** Spline embed (heavier, external runtime), purchased GLB (cost), copying an existing brand mascot (not allowed).

### D-006 · 2026-11-08 · Landing content fully CMS-driven, cached with tag revalidation
**Why:** admin must edit the site without a deploy; caching keeps Supabase usage inside free tier.
**Rejected:** headless CMS (Sanity/Contentful — extra service, limits), hard-coded content.

### D-005 · 2026-11-08 · Server actions instead of a REST API
**Why:** less code, type-safe, one deploy. Only the public lead form needs a route handler.
**Rejected:** separate Express/Nest backend.

### D-004 · 2026-11-08 · RLS as the security boundary, mirrored in `lib/permissions.ts`
**Why:** a bug in UI or an action cannot leak data; the database refuses.
**Rejected:** app-only checks with the service-role key everywhere.

### D-003 · 2026-11-08 · Accounts by invitation only
**Why:** requirement — admin creates staff accounts; reduces abuse surface.
**Rejected:** open signup with approval queue.

### D-002 · 2026-11-08 · Hosting on Vercel Hobby at launch
**Why:** best Next.js support, free. **Caveat:** Hobby terms are non-commercial. When the agency starts billing through the site or traffic grows, move to Netlify free (commercial allowed) or Vercel Pro ($20/month). Code stays the same.
**Rejected:** self-hosted VPS (cost and maintenance).

### D-001 · 2026-11-08 · Next.js + Supabase, single app
**Why:** budget is tight; Supabase free tier covers Postgres, auth, storage, realtime, cron; one Next.js app serves site and portal.
**Rejected:** Next.js + own Postgres/Prisma/Auth.js (more code), Laravel (team preference unknown, two runtimes).
