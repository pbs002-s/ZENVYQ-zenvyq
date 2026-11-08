# Roles & Permissions

This file is the source of truth for `lib/permissions.ts` and the RLS policies/functions in [BACKEND_SCHEMA.md](BACKEND_SCHEMA.md). Change all three together.

## Roles
| Role | Key | Rank | Summary |
|---|---|---|---|
| Super admin | `super_admin` | 6 | Owner. Everything. At least one active super admin must always exist. |
| Admin | `admin` | 5 | Operations. Everything except managing super admins and other admins. |
| HR | `hr` | 4 | People: invite employees/interns, review documents, company announcements. |
| Project manager | `project_manager` | 3 | Projects and tasks, project announcements, read leads. |
| Employee | `employee` | 2 | Own tasks and documents, create tasks in projects they belong to. |
| Intern | `intern` | 1 | Like employee, but cannot create or assign tasks. |

## Permission keys
```
users.view            users.invite          users.edit           users.disable      users.change_role   users.offboard
directory.view
documents.own         documents.review      documents.request
projects.view_all     projects.create       projects.manage_own
tasks.create          tasks.assign          tasks.update_assigned tasks.update_any
announcements.view    announcements.post_all announcements.post_project
leads.view            leads.manage
cms.edit
audit.view
```

## Matrix
| Permission | super_admin | admin | hr | project_manager | employee | intern |
|---|:-:|:-:|:-:|:-:|:-:|:-:|
| users.view (incl. inactive, admin fields) | ✓ | ✓ | ✓ | – | – | – |
| users.invite | ✓ any role | ✓ hr / pm / employee / intern | ✓ employee / intern | – | – | – |
| users.edit (job fields) | ✓ | ✓ same as invite | ✓ employee / intern | – | – | – |
| users.disable | ✓ | ✓ same as invite | – | – | – | – |
| users.change_role | ✓ | ✓ among hr / pm / employee / intern | – | – | – | – |
| users.offboard (export/purge documents) | ✓ | ✓ same as invite | – | – | – | – |
| directory.view | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| documents.own (upload/view own) | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| documents.review (view all, approve/reject — not own) | ✓ | ✓ | ✓ | – | – | – |
| documents.request | ✓ | ✓ | ✓ | – | – | – |
| projects.view_all | ✓ | ✓ | – | – | – | – |
| projects.create | ✓ | ✓ | – | ✓ | – | – |
| projects.manage_own (edit, members) | ✓ | ✓ | – | ✓ if project lead | – | – |
| View a project | all | all | member | member | member | member |
| tasks.create (in member project) | ✓ | ✓ | ✓ | ✓ | ✓ | – |
| tasks.assign | ✓ | ✓ | self only | ✓ (projects they manage) | self only | – |
| tasks.update_assigned (status, description, comment) | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| tasks.update_any (in project) | ✓ | ✓ | – | ✓ lead, or PM member | – | – |
| announcements.view (targeted to me) | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| announcements.post_all | ✓ | ✓ | ✓ | – | – | – |
| announcements.post_project | ✓ | ✓ | ✓ | ✓ projects they lead | – | – |
| leads.view | ✓ | ✓ | – | ✓ read-only | – | – |
| leads.manage | ✓ | ✓ | – | – | – | – |
| cms.edit | ✓ | ✓ | – | – | – | – |
| audit.view | ✓ | ✓ | – | – | – | – |

HR has no project role of its own; an HR user added to a project behaves like an employee there.

## Rules that are not a simple yes/no
1. **No self role change, no self-disable.** Enforced in `change_user_role()` / `set_user_active()`.
2. **Last super admin** cannot be disabled, demoted, or deleted (`protect_last_super_admin` trigger).
3. **Manage ceiling** — one function, `can_manage_role(actor, target)`, decides invite / edit / disable / role change / offboard:
   - super_admin → any role (subject to rules 1–2)
   - admin → `hr`, `project_manager`, `employee`, `intern` (only a super admin creates or manages admins — D-009)
   - hr → `employee`, `intern`
   - everyone else → nobody

   A role change must pass the check for **both** the old and the new role.
4. **Inactive users** (`is_active = false`) fail every RLS helper, even with a valid token.
5. **Project membership** is checked through `project_members`; admins bypass it. Creator and lead are auto-added.
6. **Column limits** are enforced by guard triggers, not just the UI: an assignee can change only `status / position / description`; a document owner cannot change `status`; nobody edits their own `role` / `is_active` / `email`.
7. **No self-review:** reviewers cannot approve/reject their own documents.

## Implementation
```ts
// lib/permissions.ts (shape)
export const ROLES = ['super_admin','admin','hr','project_manager','employee','intern'] as const;
export type Role = typeof ROLES[number];
export const PERMISSIONS: Record<Permission, readonly Role[]> = { /* mirrors matrix */ };
export const can = (role: Role, p: Permission) => PERMISSIONS[p].includes(role);
export const MANAGEABLE: Record<Role, readonly Role[]> = {
  super_admin: ROLES, admin: ['hr','project_manager','employee','intern'],
  hr: ['employee','intern'], project_manager: [], employee: [], intern: [],
};
export const canManageRole = (actor: Role, target: Role) => MANAGEABLE[actor].includes(target);
```
SQL (see schema): `auth_role()`, `has_role(variadic)`, `can_manage_role()`, `is_project_member()`, `is_project_lead()`, `can_manage_tasks()`.
A unit + pgTAP test compares `MANAGEABLE` with `can_manage_role()` for all 36 pairs so they cannot drift.
