# Deployment

## Environments
| Env | App | Database |
|---|---|---|
| Local | `npm run dev` | `npx supabase start` (Docker) |
| Preview | Vercel preview per branch | Supabase project (same as prod at first; split when needed) |
| Production | Vercel production (custom domain) | Supabase free project |

## Environment variables (`.env.example`)
```
NEXT_PUBLIC_SITE_URL=
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=          # server only
NEXT_PUBLIC_TURNSTILE_SITE_KEY=
TURNSTILE_SECRET_KEY=               # server only
RESEND_API_KEY=                     # server only
LEAD_NOTIFY_EMAILS=                 # comma separated
EMAIL_FROM=                         # e.g. "ZENVYQ <hello@yourdomain>"
IP_HASH_SALT=                       # server only, random 32+ chars
REVALIDATE_SECRET=                  # server only
```
App fails fast at startup if a required server variable is missing (`lib/env.ts` with zod).

## One-time setup
1. **Supabase:** create project → Auth: disable signups, enable leaked-password protection, set Site URL + redirect URLs (`/auth/callback`) → SMTP: Resend credentials → run migrations (`npx supabase db push`) → create first super admin (invite from dashboard with `app_metadata.role = super_admin`) → enable pg_cron + pg_net extensions.
2. **Resend:** add and verify domain (SPF/DKIM DNS records).
3. **Turnstile:** create site for the domain, copy keys.
4. **Vercel:** import repo, set env vars, add domain.
5. Smoke test: run E2E flows 1–3 from [TESTING.md](TESTING.md) against production with a test account, then remove it.

## Free-tier limits to monitor
| Service | Limit | Watch |
|---|---|---|
| Supabase DB | 500 MB | Dashboard → Database size |
| Supabase Storage | 1 GB, 50 MB per file max | Dashboard → Storage |
| Supabase egress | 5 GB / month | Cached site keeps this low |
| Supabase MAU | 50,000 | Not a concern |
| Supabase inactivity | Pauses after 7 days with no requests | Daily cron + site traffic |
| Resend | 3,000 / month, 100 / day | Only invites, leads, expiry reminders |
| Vercel Hobby | 100 GB bandwidth; **non-commercial use only** | Upgrade path in DECISIONS |
| Turnstile | Free, unlimited | — |

## Backups
Supabase free tier has no point-in-time recovery. Weekly `pg_dump` via a scheduled GitHub Action to a private artifact (free) — set up in phase 9.
