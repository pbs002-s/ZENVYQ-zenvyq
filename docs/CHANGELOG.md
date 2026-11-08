# Changelog

Newest first. One entry per meaningful change: date, type, summary, docs touched.

## 2026-11-08 — docs v2 (review fixes)
- **fix (auth):** Invite/reset links switched from PKCE `?code=` to `token_hash` + `verifyOtp` at `/auth/confirm` — PKCE fails for server-sent invites (D-011). Role no longer read from invite metadata (that is user metadata, i.e. user-editable); the profile starts as `intern` and the action sets the role. First-super-admin setup corrected. Added `password_set_at` and the set-password guard. — TRD, API, APP_FLOW, BACKEND_SCHEMA, DEPLOYMENT
- **fix (security):** Added guard triggers (task, document, profile column limits) and privileged RPCs (`change_user_role`, `set_user_active`, `assert_can_invite`) — RLS alone let assignees edit any task field and owners approve their own documents (D-012). `notify()` / `log_audit()` restricted to the service role; notifications and audit now written by triggers (D-013). No self-review of documents. — BACKEND_SCHEMA, SECURITY, ROLES_PERMISSIONS
- **fix (schema):** `project_progress` view is now `security_invoker` (was bypassing RLS) and uses numeric division (was always 0). Defined `set_updated_at`, `hit_rate_limit`, Kanban rebalance, document-request fulfilment, project-owner membership. Table order matches migration order. Added `avatars` bucket, bucket-level size/MIME limits, `get_public_team()` (anon could not read the /about team), settings draft column, indexes, check constraints. — BACKEND_SCHEMA, TRD
- **fix (roles):** Resolved rank-rule contradiction: only super admins manage admins (D-009). HR behaves as an employee inside projects. — ROLES_PERMISSIONS, APP_FLOW
- **fix (flow):** Resolved Review → Done contradiction with the acceptance criteria (D-010). — APP_FLOW, DECISIONS
- **change (hosting):** Vercel Hobby → Netlify free (commercial use allowed) (D-008 supersedes D-002). — TRD, DEPLOYMENT, DECISIONS
- **change (ops):** Keep-alive moved to a GitHub Action (pg_cron doesn't prevent pausing). Backups moved to Phase 1, encrypted; storage backup and restore drill added. Weekly free-tier usage alert. Removed leaked-password protection (paid feature). Staging project for previews. — DEPLOYMENT, SECURITY, ROADMAP
- **add:** Pagination strategy, error & edge states, offboarding flow, CMS preview via draft mode, SVG sanitising path, analytics (Cloudflare Web Analytics) for PRD goals, zxcvbn and other missing dependencies, Next.js version notes. — API, APP_FLOW, TRD, PRD
- **add:** Database test plan (guards, RPCs, invariants, hygiene), more E2E flows. — TESTING
- **add:** Open questions for defaults chosen in this revision (#13, #15, #17, #18) and legal pages (#9). — OPEN_QUESTIONS

## 2026-11-08
- **docs:** Initial documentation set created — PRD, TRD, app flow, backend schema, roles & permissions, API, project structure, design system, security, testing, deployment, roadmap, decisions, open questions.
- **docs:** Doc maintenance rules added to `docs/README.md`.
