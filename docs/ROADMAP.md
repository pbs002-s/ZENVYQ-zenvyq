# Roadmap & Progress

Tick items as they are completed (`[x]`). Add new tasks where they belong. Keep the phase status column current.
Definition of done for every item: code + relevant tests (unit / integration / database) pass in CI, docs updated, CHANGELOG entry.

| Phase | Name | Status |
|---|---|---|
| 0 | Documentation | ✅ Done (2026-11-08, revised to v2 the same day) |
| 1 | Setup, auth, ops baseline | ⏳ Not started |
| 2 | Landing page (static content) | ⏳ |
| 3 | 3D mascot | ⏳ |
| 4 | CMS + dynamic landing | ⏳ |
| 5 | Users, roles, directory | ⏳ |
| 6 | Documents | ⏳ |
| 7 | Projects & tasks | ⏳ |
| 8 | Announcements, leads, notifications, audit, reminders | ⏳ |
| 9 | Design pass, QA, launch | ⏳ |

## Phase 0 — Documentation
- [x] PRD, TRD, app flow, schema, roles, API, structure, design system, security, testing, deployment
- [x] Doc maintenance rules (docs/README.md)
- [x] v2 review: auth link flow, guard triggers, rank rule, hosting, backups, pagination, error states, offboarding

## Phase 1 — Setup, auth, ops baseline
- [ ] Answer blocking OPEN_QUESTIONS 1–3 (brand, domain, design timing)
- [ ] Pin Next.js version (TRD §9) + TypeScript strict + Tailwind v4 + shadcn/ui; record versions in CHANGELOG
- [ ] ESLint, Prettier, Vitest, Playwright, pgTAP configured; `ci.yml`
- [ ] Supabase local; migrations `0001`–`0002` (extensions, enums, `set_updated_at`, profiles, helpers, `can_manage_role`, privileged RPCs, profile triggers)
- [ ] `lib/supabase/*`, `lib/env.ts`, `lib/auth.ts`, `lib/permissions.ts`, `lib/errors.ts`, `lib/redirects.ts` + unit tests (incl. `MANAGEABLE` ↔ `can_manage_role` parity)
- [ ] `proxy.ts` guard, `/login`, `/forgot-password`, `/auth/confirm`, `/portal/set-password`, custom email templates
- [ ] Portal shell (sidebar, topbar, `error.tsx`, `loading.tsx`) filtered by role
- [ ] Seed users per role (local)
- [ ] Production Supabase project + Netlify site; first super admin set
- [ ] `keepalive.yml`, `backup-db.yml` (+ one restore test), `usage-check.yml`

## Phase 2 — Landing page (static content)
- [ ] Design tokens in `globals.css`
- [ ] TopNav (open → solid), mobile menu, footer
- [ ] All 11 home sections with placeholder content
- [ ] Service detail, work list, case study, about, contact, legal, 404
- [ ] Leads migration (`0006`) + `rate_limits` / `hit_rate_limit`
- [ ] Contact form + `/api/leads` (honeypot, Turnstile, IP + global rate limit, 503 fallback) + Resend alert
- [ ] SEO: metadata, sitemap, robots, JSON-LD; Cloudflare Web Analytics; security headers + CSP

## Phase 3 — 3D mascot
- [ ] Model from primitives, idle + eye tracking
- [ ] Section state machine (idle, wave, point, think, celebrate, sleep) + speech bubble
- [ ] Quick-nav popover (accessible)
- [ ] Lazy load, reduced-motion + low-power fallback, bundle ≤ 180 KB gz

## Phase 4 — CMS + dynamic landing
- [ ] CMS tables + RLS + seed content; `site-media` bucket
- [ ] `lib/site-data.ts` tagged cached readers + placeholder fallback when DB unreachable
- [ ] CMS editors: settings (draft/publish), hero, sections order, services, case studies, clients, testimonials, stats, process, models, FAQ, legal, mascot lines
- [ ] Image upload with compression; SVG sanitise action
- [ ] Draft-mode preview (`/api/draft`), publish + tag invalidation, concurrent-edit warning

## Phase 5 — Users, roles, directory
- [ ] Invite flow (`assert_can_invite`), resend invite, edit, `change_user_role`, `set_user_active` + Auth ban
- [ ] Guard / last-super-admin / no-self-change triggers + pgTAP tests
- [ ] Directory with search; my profile + avatar (`avatars` bucket); `get_public_team()` on /about
- [ ] Create `staging` Supabase project; point deploy previews at it

## Phase 6 — Documents
- [ ] `employee-docs` bucket + policies; documents + requests migration with guard triggers
- [ ] Upload with progress, retry, type/size validation
- [ ] Review queue (paginated), approve/reject with reason, no self-review, replace file
- [ ] Document requests + auto-fulfil
- [ ] Signed URL preview/download
- [ ] Offboarding: export + purge documents (audited)
- [ ] `backup-storage.yml`; `cleanup-orphans` Edge Function + cron

## Phase 7 — Projects & tasks
- [ ] Projects CRUD + members (auto-add creator/lead)
- [ ] Kanban (dnd-kit, keyboard), position + rebalance, list view with filters
- [ ] Guard trigger tests (assignee limits, employee self-assign only, interns)
- [ ] Comments with @mentions (member picker), attachments (`task-files`)
- [ ] `project_progress` view, My tasks page

## Phase 8 — Announcements, leads, notifications, audit, reminders
- [ ] Notifications table, `notify()`, all notification triggers, realtime bell (+ reconnect)
- [ ] Audit triggers + `log_audit()` + viewer (cursor pagination)
- [ ] Announcements with audience + read receipts
- [ ] Leads pipeline + notes + owner (PM read-only)
- [ ] pg_cron jobs + `send-reminders` Edge Function
- [ ] Global search (`search_portal`)
- [ ] Role dashboards

## Phase 9 — Design pass, QA, launch
- [ ] Apply uploaded design to tokens/components
- [ ] Accessibility audit (axe), Lighthouse ≥ 90
- [ ] SECURITY.md checklist fully ticked; Supabase Security Advisor clean
- [ ] Real content entered via CMS; privacy policy and terms finalised
- [ ] Production smoke test; restore drill; launch
