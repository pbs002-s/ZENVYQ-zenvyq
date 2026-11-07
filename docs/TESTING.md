# Testing

Target: 80%+ line coverage on `lib/` and server actions; every RLS policy covered; critical flows covered end to end.

## Layers
| Layer | Tool | What | Location |
|---|---|---|---|
| Unit | Vitest | `lib/permissions.ts` matrix, rank rules, zod schemas, mention parser, rate-limit logic, mascot state machine | `tests/unit/` |
| Integration | Vitest + local Supabase | Server actions against a real local DB (seeded users per role) | `tests/integration/` |
| Database / RLS | SQL script run by `supabase test db` (pgTAP) | Each role attempts select/insert/update/delete on each table and bucket; results must match the matrix | `supabase/tests/` |
| E2E | Playwright | Critical flows below, desktop + mobile viewport | `tests/e2e/` |
| Performance / a11y | Lighthouse CI, axe via Playwright | Landing ≥ 90; no serious axe violations | CI |

## Critical E2E flows
1. Visitor submits inquiry → lead visible in `/portal/leads`; mascot shows celebrate.
2. Admin edits hero headline → publish → home page shows new headline.
3. Admin invites employee → employee sets password → logs in → uploads document → HR approves → employee sees Approved.
4. PM creates project + task for employee → employee gets notification → drags to Done → progress 100%.
5. Intern tries `/portal/cms` and task creation → blocked.
6. Disabled user cannot log in.

## Seed users (local only)
`super@zenvyq.test`, `admin@`, `hr@`, `pm@`, `employee@`, `intern@` — password from `.env.test`, never used in production.

## Commands (once scaffolded)
```
npm run test           # vitest
npm run test:coverage
npm run test:e2e       # playwright (starts local supabase + next)
npx supabase test db   # RLS tests
```

## Rule
A feature is not done until its unit/integration tests and RLS tests pass. Bug fixes start with a failing test.
