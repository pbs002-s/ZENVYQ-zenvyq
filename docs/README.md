# ZENVYQ Documentation

ZENVYQ is one Next.js app with two parts: a CMS-driven public agency site (with a 3D guide mascot) and a private employee portal (roles, documents, projects and tasks, announcements, leads). Current progress lives in [ROADMAP.md](ROADMAP.md).

## Keeping docs current
After any change that affects behaviour, structure, schema, or scope:
1. Update the affected doc(s) so they describe what the code does.
2. Tick or add tasks in [ROADMAP.md](ROADMAP.md).
3. Add a dated entry to [CHANGELOG.md](CHANGELOG.md).
4. Log real trade-offs in [DECISIONS.md](DECISIONS.md).
5. Move answered items out of [OPEN_QUESTIONS.md](OPEN_QUESTIONS.md).
6. Schema change: update [BACKEND_SCHEMA.md](BACKEND_SCHEMA.md) and add a new migration. Permission change: update [ROLES_PERMISSIONS.md](ROLES_PERMISSIONS.md), `lib/permissions.ts`, and RLS together.

If a doc and the code disagree, the code wins; fix the doc.

| Doc | Purpose |
|---|---|
| [PRD.md](PRD.md) | Product requirements: goals, users, features, scope |
| [TRD.md](TRD.md) | Technical requirements: stack, architecture, non-functional needs |
| [APP_FLOW.md](APP_FLOW.md) | User journeys and screen flows |
| [BACKEND_SCHEMA.md](BACKEND_SCHEMA.md) | Database schema, RLS, storage, cron |
| [ROLES_PERMISSIONS.md](ROLES_PERMISSIONS.md) | Who can do what |
| [API.md](API.md) | Server actions and HTTP routes |
| [PROJECT_STRUCTURE.md](PROJECT_STRUCTURE.md) | Folder layout and conventions |
| [DESIGN_SYSTEM.md](DESIGN_SYSTEM.md) | Design tokens, components, 3D mascot |
| [SECURITY.md](SECURITY.md) | Security requirements and checklist |
| [TESTING.md](TESTING.md) | Test strategy |
| [DEPLOYMENT.md](DEPLOYMENT.md) | Environments, env vars, free-tier limits |
| [ROADMAP.md](ROADMAP.md) | Build phases and progress |
| [DECISIONS.md](DECISIONS.md) | Decision log |
| [OPEN_QUESTIONS.md](OPEN_QUESTIONS.md) | Items waiting on the product owner |
| [CHANGELOG.md](CHANGELOG.md) | Dated change history |
