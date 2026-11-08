# Testing

Target: 80%+ line coverage on `lib/` and server actions; every RLS policy, guard trigger and privileged RPC covered; critical flows covered end to end.

## Layers
| Layer | Tool | What | Location |
|---|---|---|---|
| Unit | Vitest | `lib/permissions.ts` matrix, `canManageRole`, zod schemas, mention parser, pagination cursors, redirect allow-list, error mapping, Kanban position maths, mascot state machine | `tests/unit/` |
| Integration | Vitest + local Supabase | Server actions against a real local DB (seeded users per role), incl. invite → role set, disable → ban, replace document file, purge | `tests/integration/` |
| Database | pgTAP via `supabase test db` | See below | `supabase/tests/` |
| E2E | Playwright | Critical flows below, desktop + mobile viewport | `tests/e2e/` |
| Performance / a11y | Lighthouse CI, axe via Playwright | Landing ≥ 90 all categories; no serious axe violations on site and portal pages | CI |

## Database tests (pgTAP)
- **RLS matrix:** each seeded role attempts select/insert/update/delete on each table and bucket; results must match [ROLES_PERMISSIONS](ROLES_PERMISSIONS.md).
- **Guard triggers:** assignee cannot change title/assignee/project; employee cannot assign others; intern cannot insert tasks; document owner cannot set `status`; reviewer cannot approve own document; user cannot change own `role` / `is_active` / `email`.
- **Privileged RPCs:** `change_user_role` / `set_user_active` / `assert_can_invite` for all 36 actor × target pairs; parity with `lib/permissions.ts`.
- **Invariants:** last super admin cannot be demoted/disabled/deleted; an inactive user gets nothing from any table.
- **Functions:** `hit_rate_limit` allows exactly N per window; `project_progress` returns 0 / 50 / 100 correctly; `rebalance_task_column` preserves order; `notify` / `log_audit` / `hit_rate_limit` not executable by `authenticated`.
- **Side-effect triggers:** assigning a task creates exactly one notification; mentioning a non-member creates none; a document review writes one audit row.
- **Hygiene:** every public table has RLS; every public view has `security_invoker`.

## Critical E2E flows
1. Visitor submits inquiry → lead visible in `/portal/leads`; mascot shows celebrate. 4th submission within 10 min → rate-limit message.
2. Admin edits hero headline → preview shows draft → publish → home page shows new headline.
3. Admin invites employee → employee opens link, sets password → uploads document → HR rejects with reason → employee replaces file → HR approves → employee sees Approved.
4. PM creates project + task for employee → employee gets notification → drags to Done → progress 100 %.
5. Intern tries `/portal/cms` and task creation → blocked (UI hidden, and a direct action call is rejected).
6. Admin disables a logged-in user → that user's next navigation lands on `/login?error=disabled`; login fails.
7. Admin does not see "admin"/"super_admin" in invite/role dropdowns; a direct action call with those roles is rejected.
8. Offboarding: disable → export → purge → files gone, audit row present.
9. Local Supabase stopped: home page still renders from cache; contact form shows the fallback email.

## Seed users (local only)
`super@zenvyq.test`, `admin@`, `hr@`, `pm@`, `employee@`, `intern@`, `disabled@` — password from `.env.test`, never used in production.

## Commands (once scaffolded)
```
npm run test           # vitest
npm run test:coverage
npm run test:e2e       # playwright (starts local supabase + next)
npx supabase test db   # pgTAP
```

## Rule
A feature is not done until its unit/integration and database tests pass in CI. Bug fixes start with a failing test.
