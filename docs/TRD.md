# Technical Requirements Document (TRD)

**Status:** Draft v2 · 2026-11-08
Related: [PRD](PRD.md) · [Backend schema](BACKEND_SCHEMA.md) · [Project structure](PROJECT_STRUCTURE.md)

## 1. Stack
| Layer | Technology | Notes |
|---|---|---|
| Framework | Next.js (latest stable at Phase 1, expected 16.x), App Router, React 19, TypeScript (strict) | One app serves site + portal. See §9 for version-specific names. |
| Styling | Tailwind CSS v4, CSS custom properties for tokens | Design swap = token change |
| UI primitives | shadcn/ui (copied components, Radix underneath) | Portal forms, dialogs, tables |
| Database | Supabase Postgres | RLS on every table |
| Auth | Supabase Auth (email + password), `@supabase/ssr` cookies | Public signup disabled |
| Files | Supabase Storage | 4 buckets, policies per bucket |
| Realtime | Supabase Realtime | Notifications bell only |
| Scheduled jobs | Supabase `pg_cron` + `pg_net` → Edge Functions | Daily reminders, weekly cleanup |
| 3D | three, @react-three/fiber, @react-three/drei | Mascot only, lazy loaded |
| Kanban drag & drop | @dnd-kit/core + @dnd-kit/sortable | Keyboard sensor enabled |
| Rich text | Tiptap (announcements, case studies, FAQs, legal) | Stored as sanitised HTML |
| Sanitising | isomorphic-dompurify | One allow-list in `lib/sanitize.ts` |
| Validation | zod | Shared client/server schemas |
| Password strength | @zxcvbn-ts/core + language pack | Score ≥ 3, lazy-loaded on the set-password page only |
| Image compression | browser-image-compression | Before every image upload |
| Email | Resend (API + custom SMTP for Supabase Auth) | Invites, resets, lead alerts, expiry reminders |
| Bot protection | Cloudflare Turnstile | Public forms |
| Analytics | Cloudflare Web Analytics (cookieless, free) | Needed to measure PRD goals G1/G2; no cookie banner required |
| Tests | Vitest, Playwright, pgTAP | See [TESTING](TESTING.md) |
| Hosting | Netlify (free plan, commercial use allowed) | See [DEPLOYMENT](DEPLOYMENT.md) and DECISIONS D-008 |
| Ops automation | GitHub Actions | CI, backups, keep-alive, free-tier usage check |

## 2. Architecture
```
Browser
  │
  ├── Public site (RSC, cached) ──reads──► Supabase (anon key, published rows only)
  │        └── Mascot (client component, dynamic import, no SSR)
  │
  ├── POST /api/leads ──► honeypot ─► Turnstile verify ─► rate limit (IP + global)
  │                       ─► insert lead (service role) ─► DB trigger notifies admins ─► Resend alert
  │
  └── Portal (/portal/*, dynamic, auth cookie)
           ├── proxy.ts (middleware): refresh session, redirect unauthenticated to /login
           ├── Portal layout: inactive → sign out; password not set → /portal/set-password
           ├── Server components: read via user-scoped Supabase client (RLS applies)
           ├── Server actions: zod ─► can() ─► write via user client (RLS + guard triggers)
           │        ─► DB triggers write notifications + audit ─► revalidate
           └── Auth-admin operations only (invite, ban/unban, purge files) use the service-role
               client, and only after a DB permission RPC (assert_can_invite / set_user_active) succeeds
```

Principles:
- **The database is the security boundary.** RLS decides which rows; guard triggers decide which columns and transitions; privileged RPCs hold the rank rules. UI hiding and `can()` are UX + defence in depth.
- **Side effects live next to the data.** Notifications and audit entries are written by DB triggers, so no action can forget them and no user can forge them.
- **Server actions over API routes.** Route handlers exist only where an HTTP endpoint is required: `/api/leads`, `/auth/confirm`, `/api/draft`, `/api/revalidate`.
- **No client-side service key.** `lib/supabase/admin.ts` imports `server-only`.

## 3. Rendering & caching
| Area | Strategy |
|---|---|
| Public pages | Static with tagged data cache. Tags: `site`, `site:settings`, `site:<collection>`, `site:work:<slug>`, `site:service:<slug>`. |
| CMS publish | Server action invalidates the affected tag(s) → site updates on next request (seconds) |
| Case study / service pages | `generateStaticParams` from published rows; unknown slug → 404; new slugs rendered on demand then cached |
| CMS preview | `/api/draft` enables Next.js **draft mode** after verifying an admin session. In draft mode `lib/site-data.ts` skips the cache and reads drafts with the user-scoped client. A visible "Preview mode" bar has an exit link. |
| Portal | Dynamic (`cookies()`), no caching of user data |
| Images | `next/image` with Supabase Storage remote pattern; covers/OG images pre-sized on upload |
| Supabase paused / unreachable | Public pages keep serving the last cached version. Uncached reads fall back to built-in placeholder content and log an error instead of crashing. |

Result: visitors almost never hit the database directly, keeping Supabase usage low.

## 4. Auth design
- `@supabase/ssr` with cookie session; `proxy.ts` refreshes tokens on `/portal/*` and `/login`.
- **Email links use `token_hash`, not PKCE code exchange.** Invites are sent by the server and links are often opened on another device, so the browser has no PKCE verifier. Templates link to `{{ .SiteURL }}/auth/confirm?token_hash={{ .TokenHash }}&type=<invite|recovery>&next=/portal/set-password`; the route calls `supabase.auth.verifyOtp({ token_hash, type })`, which sets the session cookie, then redirects.
- `profiles` row is created by a trigger on `auth.users` insert with the lowest role (`intern`). The invite action then sets the real role via the service role. Role is **never** taken from user metadata (user-editable).
- Role lookup in SQL: `auth_role()` (security definer, stable) reads `profiles.role` for `auth.uid()` and returns null for inactive users.
- Invite: `assert_can_invite(role)` RPC (user client) → `auth.admin.inviteUserByEmail(email, { data: { full_name }, redirectTo })` → set `profiles.role` + job fields (service role) → `log_audit('user.invited')`.
- First login: `profiles.password_set_at` is null until `setPassword` succeeds; the portal layout redirects such users to `/portal/set-password`.
- Disable: `set_user_active(id, false)` RPC (rank checks in SQL) → `auth.admin.updateUserById(id, { ban_duration: '876000h' })`. Enable: RPC then `ban_duration: 'none'`. RLS helpers also require `is_active`, so access stops at the next request even before the JWT expires.
- Supabase built-in SMTP is heavily rate-limited; Resend is configured as custom SMTP in Supabase Auth.

## 5. Files
| Bucket | Public | Path convention | Max size | Types |
|---|---|---|---|---|
| `employee-docs` | No | `<user_id>/<document_id>/<slug-filename>` | 10 MB | pdf, jpg, png, docx |
| `task-files` | No | `<project_id>/<task_id>/<uuid>-<slug-filename>` | 10 MB | pdf, images, docx, xlsx, pptx, zip, txt, csv |
| `avatars` | Yes | `<user_id>/<uuid>.webp` | 1 MB | jpg, png, webp (converted to webp client-side) |
| `site-media` | Yes | `<collection>/<uuid>.<ext>` | 5 MB images, 20 MB mp4 | jpg, png, webp, avif, svg*, mp4 |

- Size and MIME limits are configured **on the bucket**, so Storage enforces them even for direct browser uploads.
- Upload flow: browser generates the row id (`crypto.randomUUID()`), uploads with the user's session (policies enforce the path), then a server action inserts the row. Abandoned uploads are removed by the weekly `cleanup-orphans` job.
- \* **SVG** is the only type not uploaded directly: it goes through a server action that sanitises it (DOMPurify SVG profile: no scripts, foreign objects, or external references) and uploads with the service role.
- Private files are downloaded through server-created signed URLs (60 s expiry).
- Images are resized/compressed client-side before upload to protect the 1 GB quota.

## 6. 3D mascot technical spec
- Built from primitives (RoundedBox, Sphere, Capsule) in `components/mascot/MascotModel.tsx`; no GLB download.
- Loaded with `next/dynamic(() => import(...), { ssr: false })` after `requestIdleCallback` (`setTimeout` fallback for Safari).
- Canvas ~140×140 px (100 px mobile), `dpr={[1, 1.5]}`, `frameloop="demand"` when idle, pauses when the tab is hidden.
- Section awareness: `IntersectionObserver` on sections with `data-mascot-section` → state machine (`idle`, `wave`, `point`, `think`, `celebrate`, `sleep`). See [DESIGN_SYSTEM](DESIGN_SYSTEM.md).
- Quick-nav popover is plain HTML (accessible, keyboard focusable), not inside the canvas.
- Fallback: static SVG when `prefers-reduced-motion`, WebGL unavailable, or `navigator.hardwareConcurrency <= 2`. Nothing renders when `site_settings.mascot.enabled = false`.
- Budget: mascot chunk ≤ 180 KB gzipped; must not affect LCP.

## 7. Non-functional requirements
| Area | Requirement |
|---|---|
| Performance | Landing LCP < 2.5 s on 4G, CLS < 0.1, INP < 200 ms, Lighthouse ≥ 90 |
| Accessibility | WCAG 2.2 AA; keyboard nav for nav, mascot menu, Kanban (dnd-kit keyboard sensor) |
| Browser support | Last 2 versions of Chrome, Edge, Firefox, Safari; iOS 16+ |
| Responsiveness | 360 px → 1920 px |
| Security | See [SECURITY](SECURITY.md) |
| Cost | $0/month at launch; stay under free-tier limits in [DEPLOYMENT](DEPLOYMENT.md) |
| Observability | Netlify function logs + Supabase logs; `audit_log` for business events; weekly usage report (DEPLOYMENT) |
| Data retention | Notifications 90 days; audit log 1 year; leads per [SECURITY](SECURITY.md#personal-data) |
| Pagination | Portal tables: offset, 25 per page (max 100). Feeds (notifications, comments, audit, lead notes): cursor on `(created_at, id)`, 20 per page. See [API](API.md#pagination). |
| Error handling | Every portal route segment has `error.tsx` and `loading.tsx`; behaviour in [APP_FLOW §12](APP_FLOW.md#12-error--edge-states) |

## 8. Free-tier constraints and mitigations
| Limit | Mitigation |
|---|---|
| Supabase 500 MB DB | Lean schema, 90-day notification cleanup, no blobs in DB; weekly size check alerts at 80 % |
| Supabase 1 GB storage | Bucket size caps, client-side compression, replaced files deleted, orphan cleanup; weekly check alerts at 80 % |
| Supabase pauses after 7 days without API activity | Daily keep-alive GitHub Action calls the REST API (pg_cron does **not** count); real traffic helps too |
| Supabase free: no backups / PITR | Weekly encrypted `pg_dump` from Phase 1; storage backup from Phase 6 ([DEPLOYMENT](DEPLOYMENT.md#backups)) |
| Auth email rate limit | Resend as custom SMTP |
| Resend 3,000 emails/month, 100/day | Emails only for invites, resets, new leads, expiry reminders; global lead rate limit (50/h) |
| Netlify free usage allowance | Static/cached pages, compressed images; usage checked monthly |

## 9. Framework version notes
Pin the latest stable Next.js at Phase 1 and record the exact version in CHANGELOG. If it is **16.x**:
- The request-interception file is `proxy.ts` (`middleware.ts` is the pre-16 name). Docs say "proxy (middleware)".
- Tagged data cache uses `'use cache'` + `cacheTag()`; server actions use `updateTag()` for read-your-own-writes and `revalidateTag(tag, 'max')` elsewhere. On 15.x use `unstable_cache` + `revalidateTag(tag)`.
- Confirm the Netlify Next.js runtime supports the chosen version (draft mode, tag revalidation) before Phase 2; otherwise pin the newest version it supports.
- Verify these API names against the official docs at setup — they changed between 15 and 16.
