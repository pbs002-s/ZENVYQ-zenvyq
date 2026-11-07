# Roadmap & Progress

Tick items as they are completed (`[x]`). Add new tasks where they belong. Keep the phase status column current.

| Phase | Name | Status |
|---|---|---|
| 0 | Documentation | ✅ Done (2026-11-08) |
| 1 | Setup & auth | ⏳ Not started |
| 2 | Landing page (static content) | ⏳ |
| 3 | 3D mascot | ⏳ |
| 4 | CMS + dynamic landing | ⏳ |
| 5 | Users, roles, directory | ⏳ |
| 6 | Documents | ⏳ |
| 7 | Projects & tasks | ⏳ |
| 8 | Announcements, leads, notifications, audit, reminders | ⏳ |
| 9 | Design pass, QA, deploy | ⏳ |

## Phase 0 — Documentation
- [x] PRD, TRD, app flow, schema, roles, API, structure, design system, security, testing, deployment
- [x] Doc maintenance rules (docs/README.md)

## Phase 1 — Setup & auth
- [ ] Next.js 15 + TypeScript strict + Tailwind v4 + shadcn/ui
- [ ] ESLint, Prettier, Vitest, Playwright configured
- [ ] Supabase local + `0001_init.sql` (enums, profiles, triggers, helpers)
- [ ] `lib/supabase/*`, `lib/env.ts`, `lib/auth.ts`, `lib/permissions.ts` + unit tests
- [ ] `middleware.ts` guard, `/login`, `/forgot-password`, `/auth/callback`, `/portal/set-password`
- [ ] Portal shell (sidebar, topbar) filtered by role
- [ ] Seed users per role (local)

## Phase 2 — Landing page (static content)
- [ ] Design tokens in `globals.css`
- [ ] TopNav (open → solid), mobile menu, footer
- [ ] All 11 home sections with placeholder content
- [ ] Service detail, work list, case study, about, contact, legal, 404
- [ ] Contact form + `/api/leads` (Turnstile, honeypot, rate limit) + Resend alert
- [ ] SEO: metadata, sitemap, robots, JSON-LD

## Phase 3 — 3D mascot
- [ ] Model from primitives, idle + eye tracking
- [ ] Section state machine + speech bubble
- [ ] Quick-nav popover (accessible)
- [ ] Lazy load, reduced-motion + low-power fallback, bundle ≤ 180 KB gz

## Phase 4 — CMS + dynamic landing
- [ ] CMS tables + RLS + seed content
- [ ] `lib/site-data.ts` cached readers with tags
- [ ] CMS editors: settings, hero, sections order, services, case studies, clients, testimonials, stats, process, models, FAQ, legal, mascot lines
- [ ] Image upload to `site-media` with compression
- [ ] Draft / publish + preview mode + revalidation

## Phase 5 — Users, roles, directory
- [ ] Invite flow with rank rules, edit, role change, disable
- [ ] Last-super-admin and self-role-change triggers + tests
- [ ] Directory with search; my profile + avatar

## Phase 6 — Documents
- [ ] Upload with progress, type/size validation
- [ ] Review queue, approve/reject with reason, re-upload
- [ ] Document requests
- [ ] Signed URL preview/download

## Phase 7 — Projects & tasks
- [ ] Projects CRUD + members
- [ ] Kanban (dnd-kit, keyboard) + list view with filters
- [ ] Comments with @mentions, attachments
- [ ] Progress view, My tasks page

## Phase 8 — Announcements, leads, notifications, audit, reminders
- [ ] Announcements with audience + read receipts
- [ ] Leads pipeline + notes + owner
- [ ] Notifications table, `notify()`, realtime bell
- [ ] Audit log + viewer
- [ ] pg_cron jobs + reminder Edge Function
- [ ] Global search
- [ ] Role dashboards

## Phase 9 — Design pass, QA, deploy
- [ ] Apply uploaded design to tokens/components
- [ ] Accessibility audit (axe), Lighthouse ≥ 90
- [ ] Security checklist in SECURITY.md fully ticked
- [ ] Production setup per DEPLOYMENT.md, backups
- [ ] Production smoke test
