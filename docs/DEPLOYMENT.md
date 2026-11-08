# Deployment

## Environments
| Env | App | Database |
|---|---|---|
| Local | `npm run dev` | `npx supabase start` (Docker) |
| CI | GitHub Actions | Local Supabase in the runner, seeded |
| Preview | Netlify deploy preview per PR | Phases 1–4: seeded demo project. Phase 5+: `staging` Supabase project. **Never production data.** |
| Production | Netlify production (custom domain) | Supabase free project |

The Supabase free plan allows two projects: `production` and `staging`.

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
NEXT_PUBLIC_CF_ANALYTICS_TOKEN=     # Cloudflare Web Analytics beacon token (public)
```
App fails fast at startup if a required variable is missing (`lib/env.ts` with zod).

GitHub Actions secrets (not in the app): `SUPABASE_DB_URL` (direct connection string), `SUPABASE_URL`, `SUPABASE_ANON_KEY`, `SUPABASE_SERVICE_ROLE_KEY`, `BACKUP_AGE_PUBLIC_KEY`, `ALERT_EMAIL`, `RESEND_API_KEY`.

## One-time setup
1. **Supabase**
   - Create project (region closest to users).
   - Auth: disable signups; minimum password length 10; Site URL; redirect allow-list (`https://<domain>/auth/confirm`, staging/preview domains).
   - Auth → Email templates: invite and recovery → `{{ .SiteURL }}/auth/confirm?token_hash={{ .TokenHash }}&type=invite|recovery&next=/portal/set-password` (source in `supabase/templates/`).
   - Auth → SMTP: Resend credentials.
   - Enable extensions `pg_cron`, `pg_net`, `pg_trgm`; store Edge Function URL + secret in Vault.
   - `npx supabase db push` (migrations, including buckets with limits), `npx supabase functions deploy`.
   - **First super admin:** Auth → Users → *Invite user* (your email). Then in the SQL editor: `update profiles set role = 'super_admin' where email = '<you>';` (runs as `postgres`, so guard triggers allow it). Accept the invite and set a password.
2. **Resend:** add and verify the domain (SPF, DKIM, DMARC).
3. **Turnstile:** create a widget for the domain, copy keys.
4. **Cloudflare Web Analytics:** add the site, copy the beacon token.
5. **Netlify:** import repo, set env vars, add custom domain + HTTPS. Confirm the Next.js runtime supports the pinned Next.js version.
6. **GitHub:** private repo, branch protection on `main`, Dependabot, Actions secrets above.
7. **Smoke test:** run E2E flows 1–3 from [TESTING.md](TESTING.md) against production with a test account, then disable and purge it.

## Scheduled GitHub Actions (`.github/workflows/`)
| Workflow | Schedule | Does |
|---|---|---|
| `ci.yml` | every PR / push | Lint, typecheck, unit, integration + pgTAP on local Supabase, E2E, Lighthouse CI |
| `keepalive.yml` | daily | `GET <SUPABASE_URL>/rest/v1/site_settings?select=updated_at` with the anon key. Prevents the free-tier 7-day pause (pg_cron does **not** count as activity). Fails loudly if the project is already paused. |
| `backup-db.yml` | weekly (Sun) + manual | `pg_dump` (schema + data, excluding `rate_limits`) → `age`-encrypt with `BACKUP_AGE_PUBLIC_KEY` → artifact (retention 90 days). |
| `backup-storage.yml` | weekly (Sun) + manual, from Phase 6 | Downloads `employee-docs`, `task-files`, `site-media`, `avatars` with the service role → tar → `age`-encrypt → artifact (90 days). If the archive would exceed 1.5 GB, alert instead. |
| `usage-check.yml` | weekly (Mon) | Queries `pg_database_size(current_database())` and `sum((metadata->>'size')::bigint) from storage.objects`; emails `ALERT_EMAIL` via Resend if DB > 400 MB or storage > 800 MB (80 %). |

## Free-tier limits to monitor
| Service | Limit | How it's watched |
|---|---|---|
| Supabase DB | 500 MB | `usage-check.yml` (alert at 400 MB) |
| Supabase Storage | 1 GB | `usage-check.yml` (alert at 800 MB) |
| Supabase egress | 5 GB / month | Cached site keeps this low; Supabase Usage page monthly |
| Supabase MAU | 50,000 | Not a concern |
| Supabase inactivity | Pauses after 7 days without API requests | `keepalive.yml` daily |
| Resend | 3,000 / month, 100 / day | Only invites, resets, leads, expiry reminders; global lead limit 50/h |
| Netlify free | Usage allowance (check the current plan page at setup; limits change) | Netlify dashboard monthly |
| GitHub Actions | 2,000 min/month on private repos | Workflows are short; check monthly |
| Turnstile, CF Web Analytics | Free | — |

Monthly check (first Monday): Supabase, Netlify, Resend and GitHub usage pages. Anything above 60 % → add an item to OPEN_QUESTIONS.

## Backups
Supabase free tier has no automatic backups or point-in-time recovery.
- **Database:** weekly encrypted `pg_dump` from **Phase 1** (`backup-db.yml`). Also run it manually before every production migration.
- **Storage:** weekly encrypted archive from **Phase 6**, when employee documents start arriving (`backup-storage.yml`). `pg_dump` does not include files.
- The `age` **private** key is kept offline by a founder (password manager), never in GitHub.
- **Restore drill** once per quarter: decrypt the latest dump, restore into local Supabase, open the portal and spot-check rows. Record the date in CHANGELOG.

## Upgrade path
When budget allows or limits are hit: Supabase Pro ($25/month — daily backups, no pausing, leaked-password protection) first; then hosting (Netlify paid or Vercel Pro). No code changes required.
