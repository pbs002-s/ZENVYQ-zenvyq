# Project Structure

Status: planned. Update this file whenever folders are added, moved, or removed.

```
ZENVYQ/
├── docs/                         # All documentation (this folder)
├── app/
│   ├── layout.tsx                # Root: fonts, theme tokens, <html>
│   ├── globals.css               # Tailwind + design tokens (CSS variables)
│   ├── not-found.tsx
│   ├── sitemap.ts  robots.ts
│   ├── (site)/                   # Public site (cached)
│   │   ├── layout.tsx            # TopNav, Footer, Mascot
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
│   ├── auth/callback/route.ts
│   ├── api/
│   │   ├── leads/route.ts
│   │   └── revalidate/route.ts
│   └── portal/                   # Private, dynamic
│       ├── layout.tsx            # Sidebar + header + notifications bell
│       ├── page.tsx              # Role dashboard
│       ├── set-password/  profile/  notifications/  search/
│       ├── users/                # page, new, [id], actions.ts
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
│   ├── portal/                   # Sidebar, DataTable, KanbanBoard, FileUpload, ...
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
│   ├── auth.ts                   # getSessionUser, requireRole, requirePermission
│   ├── permissions.ts            # Role matrix (mirrors docs/ROLES_PERMISSIONS.md)
│   ├── site-data.ts              # Cached CMS readers with tags
│   ├── notify.ts  audit.ts
│   ├── email.ts                  # Resend templates
│   ├── turnstile.ts
│   ├── sanitize.ts
│   └── validation/               # zod schemas per feature (lead.ts, user.ts, task.ts, ...)
├── types/
│   └── database.ts               # Generated: supabase gen types typescript
├── supabase/
│   ├── config.toml
│   ├── migrations/               # 0001_init.sql, 0002_rls.sql, ... (never edit applied ones)
│   ├── seed.sql                  # Demo CMS content + first super_admin (local only)
│   ├── tests/rls.test.sql        # Role-by-role access checks
│   └── functions/send-reminders/ # Edge function for reminder emails
├── tests/
│   ├── unit/                     # Vitest
│   └── e2e/                      # Playwright
├── public/                       # Static assets (favicon fallback, fonts if self-hosted)
├── middleware.ts                 # Session refresh + /portal guard
├── next.config.ts                # Images remote pattern, security headers
├── .env.example
└── package.json
```

## Conventions
- **Feature folders:** a portal feature keeps its page, components used only there, and `actions.ts` together.
- **Server vs client:** components are server by default; client components end in the folder with `'use client'` at top and are kept small.
- **Naming:** components `PascalCase.tsx`, hooks `useThing.ts`, utilities `kebab-case.ts`, SQL `snake_case`.
- **Types:** never hand-write DB types; regenerate `types/database.ts` after each migration.
- **Imports:** absolute via `@/` alias.
- **File size:** 200–400 lines typical, 800 max.
