# Technical Requirements Document (TRD)

**Status:** Draft v1 · 2026-11-08
Related: [PRD](PRD.md) · [Backend schema](BACKEND_SCHEMA.md) · [Project structure](PROJECT_STRUCTURE.md)

## 1. Stack
| Layer | Technology | Notes |
|---|---|---|
| Framework | Next.js 15, App Router, React 19, TypeScript (strict) | One app serves site + portal |
| Styling | Tailwind CSS v4, CSS custom properties for tokens | Design swap = token change |
| UI primitives | shadcn/ui (copied components, Radix underneath) | Portal forms, dialogs, tables |
| Database | Supabase Postgres | RLS on every table |
| Auth | Supabase Auth (email + password), `@supabase/ssr` cookies | Public signup disabled |
| Files | Supabase Storage | 3 buckets, policies per bucket |
| Realtime | Supabase Realtime | Notifications bell only |
| Scheduled jobs | Supabase `pg_cron` | Daily reminders |
| 3D | three, @react-three/fiber, @react-three/drei | Mascot only, lazy loaded |
| Kanban drag & drop | @dnd-kit/core | |
| Rich text | Tiptap (announcements, case studies) | Stored as sanitised HTML |
| Validation | zod | Shared client/server schemas |
| Email | Resend | Invite + lead alerts + reminders |
| Bot protection | Cloudflare Turnstile | Public forms |
| Tests | Vitest, Playwright, SQL RLS tests | See [TESTING](TESTING.md) |
| Hosting | Vercel (Hobby at start) | See [DEPLOYMENT](DEPLOYMENT.md) |

## 2. Architecture
```
Browser
  │
  ├── Public site (RSC, cached) ──reads──► Supabase (anon key, published rows only)
  │        └── Mascot (client component, dynamic import, no SSR)
  │
  ├── POST /api/leads ──► Turnstile verify ─► rate limit ─► insert lead ─► Resend alert
  │
  └── Portal (/portal/*, dynamic, auth cookie)
           ├── middleware.ts: refresh session, redirect unauthenticated to /login
           ├── Server components: read via user-scoped Supabase client (RLS applies)
           ├── Server actions: zod validate ─► requireRole() ─► write ─► audit log ─► revalidate
           └── Admin-only actions (create user, disable user) use service-role client
```

Principles:
- **RLS is the real security boundary.** UI hiding and `requireRole()` are convenience + defence in depth.
- **Server actions over API routes.** Only `/api/leads` (public, needs Turnstile) and `/api/revalidate` (optional webhook) are route handlers.
- **No client-side service key.** `lib/supabase/admin.ts` imports `server-only`.

## 3. Rendering & caching
| Area | Strategy |
|---|---|
| Public pages | Static/ISR. Data fetched with `unstable_cache` / `fetch` tagged `site`, `site:services`, `site:work:<slug>` etc. |
| CMS publish | Server action calls `revalidateTag()` for the affected tag → site updates on next request |
| Case study / service pages | `generateStaticParams` from published rows + on-demand revalidation |
| Portal | Dynamic (`cookies()`), no caching of user data |
| Images | `next/image` with Supabase Storage remote pattern |

Result: visitors almost never hit the database directly, keeping Supabase free-tier usage low.

## 4. Auth design
- `@supabase/ssr` with cookie session; `middleware.ts` refreshes tokens on `/portal/*` and `/login`.
- `profiles` row created by a trigger on `auth.users` insert; role passed via `raw_app_meta_data.role` at invite time (app metadata cannot be changed by the user).
- Role lookup in SQL: `public.auth_role()` (security definer, stable) reads `profiles.role` for `auth.uid()`.
- Disabled user: `auth.admin.updateUserById(id, { ban_duration: 'none' | '876000h' })` + `profiles.is_active = false`; RLS helpers also require `is_active`.
- Invite: `auth.admin.inviteUserByEmail(email, { data, redirectTo: /auth/callback?next=/portal/set-password })`.
- Supabase built-in SMTP is rate-limited; configure Resend as custom SMTP in Supabase Auth.

## 5. Files
| Bucket | Public | Path convention | Max size | Types |
|---|---|---|---|---|
| `employee-docs` | No | `<user_id>/<document_id>/<filename>` | 10 MB | pdf, jpg, png, docx |
| `task-files` | No | `<project_id>/<task_id>/<uuid>-<filename>` | 10 MB | pdf, images, docx, xlsx, zip |
| `site-media` | Yes | `<collection>/<uuid>.<ext>` | 5 MB | jpg, png, webp, svg (svg sanitised), mp4 ≤ 20 MB |

- Upload from browser directly to Storage with the user's session (policies enforce path ownership), then a server action records the row.
- Downloads of private files use signed URLs (60 s expiry) created server-side.
- Images resized/compressed client-side before upload (browser-image-compression) to save the 1 GB quota.

## 6. 3D mascot technical spec
- Built from primitives (RoundedBox, Sphere, Capsule) in `components/mascot/MascotModel.tsx`; no GLB download.
- Loaded with `next/dynamic(() => import(...), { ssr: false })` after `requestIdleCallback`.
- Canvas ~140×140 px, `dpr={[1, 1.5]}`, `frameloop="demand"` when idle, pauses when tab hidden.
- Section awareness: `IntersectionObserver` on sections with `data-mascot-section` → state machine (`idle`, `wave`, `point`, `think`, `celebrate`).
- Quick-nav popover is plain HTML (accessible, keyboard focusable), not inside the canvas.
- Fallback: static SVG when `prefers-reduced-motion`, WebGL unavailable, or `navigator.hardwareConcurrency <= 2`.
- Budget: mascot chunk ≤ 180 KB gzipped; must not affect LCP.

## 7. Non-functional requirements
| Area | Requirement |
|---|---|
| Performance | Landing LCP < 2.5 s on 4G, CLS < 0.1, Lighthouse ≥ 90 |
| Accessibility | WCAG 2.2 AA; keyboard nav for nav, mascot menu, Kanban (dnd-kit keyboard sensor) |
| Browser support | Last 2 versions of Chrome, Edge, Firefox, Safari; iOS 16+ |
| Responsiveness | 360 px → 1920 px |
| Security | See [SECURITY](SECURITY.md) |
| Cost | $0/month at launch; stay under free-tier limits in [DEPLOYMENT](DEPLOYMENT.md) |
| Observability | Vercel logs + Supabase logs; audit_log table for business events |
| Data retention | Notifications older than 90 days deleted by cron; audit log kept 1 year |

## 8. Free-tier constraints and mitigations
| Limit | Mitigation |
|---|---|
| Supabase 500 MB DB | Lean schema, delete old notifications, no blobs in DB |
| Supabase 1 GB storage | 10 MB cap, client-side image compression, delete rejected doc files after re-upload |
| Supabase pauses after 7 days of no activity | Site traffic + daily pg_cron job keep it active; uptime ping as backup |
| Auth email rate limit | Resend as custom SMTP |
| Resend 3,000 emails/month, 100/day | Emails only for invites, new leads, reminders; everything else in-app |
| Vercel Hobby non-commercial | Move to Netlify free or Vercel Pro once billing clients (see [DECISIONS](DECISIONS.md)) |
