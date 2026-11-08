# Project Structure

Status: planned. Update this file whenever folders are added, moved, or removed.

```
ZENVYQ/
├── docs/                         # All documentation (this folder)
├── app/
│   ├── layout.tsx                # Root: fonts, theme tokens, <html>, analytics beacon
│   ├── globals.css               # Tailwind + design tokens (CSS variables)
│   ├── not-found.tsx  error.tsx  global-error.tsx
│   ├── sitemap.ts  robots.ts
│   ├── (site)/                   # Public site (cached)
│   │   ├── layout.tsx            # TopNav, Footer, Mascot, preview-mode bar
│   │   ├── page.tsx              # Home — renders sections in CMS order
│   │   ├── services/[slug]/page.tsx
│   │   ├── work/page.tsx
│   │   ├── work/[slug]/page.tsx
│   │   ├── about/page.tsx
│   │   ├── contact/page.tsx
│   │   └── (legal)/privacy/page.tsx, terms/page.tsx
│   ├── (auth)/
│   │   ├── login/page.tsx
│   │   ├── forgot-password/page.tsx
│   │   └── actions.ts
│   ├── auth/confirm/route.ts     # verifyOtp for invite + recovery links
│   ├── api/
│   │   ├── leads/route.ts
│   │   ├── draft/route.ts        # CMS preview (draft mode)
│   │   └── revalidate/route.ts
│   └── portal/                   # Private, dynamic
│       ├── layout.tsx            # Sidebar + header + bell; inactive / password-not-set guards
│       ├── error.tsx  loading.tsx
│       ├── page.tsx              # Role dashboard
│       ├── set-password/  profile/  notifications/  search/
│       ├── users/                # page, new, [id] (incl. offboarding), actions.ts
│       ├── directory/
│       ├── documents/            # page, review/, actions.ts
│       ├── projects/             # page, new, [id]/(board|list|members|files), actions.ts
│       ├── tasks/
│       ├── announcements/
│       ├── leads/
│       ├── cms/                  # page, [collection]/, actions.ts
│       └── audit/
├── components/
│   ├── ui/                       # shadcn/ui primitives (button, dialog, input, ...)
│   ├── site/                     # TopNav, sections/Hero, sections/Services, ... Footer
│   ├── portal/                   # Sidebar, DataTable, KanbanBoard, FileUpload, EmptyState, ...
│   └── mascot/
│       ├── Mascot.tsx            # Client wrapper: lazy load, fallback, quick-nav
│       ├── MascotCanvas.tsx      # R3F Canvas
│       ├── MascotModel.tsx       # Geometry built from primitives
│       ├── useMascotState.ts     # Section → pose state machine
│       └── mascot-fallback.svg
├── lib/
│   ├── supabase/
│   │   ├── server.ts             # createServerClient (cookies) — RSC & actions
│   │   ├── client.ts             # createBrowserClient — uploads, realtime
│   │   ├── admin.ts              # service role, imports 'server-only'
│   │   └── public.ts             # anon client for cached site queries
│   ├── env.ts                    # zod-validated environment
│   ├── auth.ts                   # getSessionUser, requireRole, requirePermission
│   ├── permissions.ts            # Role matrix + MANAGEABLE (mirrors docs/ROLES_PERMISSIONS.md)
│   ├── site-data.ts              # Tagged cached CMS readers, draft-mode aware, placeholder fallback
│   ├── audit.ts                  # logAudit (service role, non-DB events only)
│   ├── errors.ts                 # DB/trigger error → user message
│   ├── pagination.ts             # offset + cursor helpers
│   ├── redirects.ts              # safe relative-path allow-list
│   ├── email.ts                  # Resend templates
│   ├── turnstile.ts
│   ├── sanitize.ts               # sanitizeHtml, sanitizeSvg
│   └── validation/               # zod schemas per feature (lead.ts, user.ts, task.ts, cms.ts, ...)
├── types/
│   └── database.ts               # Generated: supabase gen types typescript
├── supabase/
│   ├── config.toml
│   ├── migrations/               # 0001_extensions_enums.sql … 0011_cron.sql (never edit applied ones)
│   ├── seed.sql                  # Demo CMS content + users per role (local only)
│   ├── templates/                # Auth email templates (invite.html, recovery.html)
│   ├── tests/                    # pgTAP: rls_*.test.sql, guards.test.sql, rpc.test.sql, hygiene.test.sql
│   └── functions/
│       ├── send-reminders/       # Expiry reminder emails
│       └── cleanup-orphans/      # Weekly storage orphan cleanup
├── .github/
│   ├── workflows/                # ci, keepalive, backup-db, backup-storage, usage-check
│   └── dependabot.yml
├── tests/
│   ├── unit/                     # Vitest
│   ├── integration/              # Vitest + local Supabase
│   └── e2e/                      # Playwright
├── public/                       # Static assets (favicon fallback, fonts if self-hosted)
├── proxy.ts                      # Session refresh + /portal guard (middleware.ts on Next ≤ 15)
├── next.config.ts                # Images remote pattern, security headers
├── netlify.toml
├── .env.example
└── package.json
```

## Conventions
- **Feature folders:** a portal feature keeps its page, components used only there, and `actions.ts` together.
- **Server vs client:** components are server by default; client components have `'use client'` at the top and are kept small.
- **Naming:** components `PascalCase.tsx`, hooks `useThing.ts`, utilities `kebab-case.ts`, SQL `snake_case`.
- **Types:** never hand-write DB types; regenerate `types/database.ts` after each migration.
- **Imports:** absolute via `@/` alias.
- **File size:** 200–400 lines typical, 800 max.
- **Migrations:** one concern per file; never edit an applied migration — add a new one.
