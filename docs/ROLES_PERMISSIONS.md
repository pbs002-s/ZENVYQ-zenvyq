# Roles & Permissions

This file is the source of truth for `lib/permissions.ts` and the RLS policies in [BACKEND_SCHEMA.md](BACKEND_SCHEMA.md). Change all three together.

## Roles
| Role | Key | Rank | Summary |
|---|---|---|---|
| Super admin | `super_admin` | 6 | Owner. Everything. At least one must always exist. |
| Admin | `admin` | 5 | Everything except managing super admins. |
| HR | `hr` | 4 | People: invite employees/interns, review documents, announcements. |
| Project manager | `project_manager` | 3 | Projects and tasks, project announcements, read leads. |
| Employee | `employee` | 2 | Own tasks and documents, create tasks in projects they belong to. |
| Intern | `intern` | 1 | Like employee, but cannot create or assign tasks. |

## Permission keys
```
users.view            users.invite          users.edit           users.disable      users.change_role
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
| users.view | ✓ | ✓ | ✓ | – | – | – |
| users.invite | ✓ any role | ✓ not super_admin | ✓ employee/intern | – | – | – |
| users.edit | ✓ | ✓ not super_admin | ✓ employee/intern | – | – | – |
| users.disable | ✓ | ✓ not super_admin | – | – | – | – |
| users.change_role | ✓ | ✓ not to/from super_admin | – | – | – | – |
| directory.view | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| documents.own (upload/view own) | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| documents.review (view all, approve/reject) | ✓ | ✓ | ✓ | – | – | – |
| documents.request | ✓ | ✓ | ✓ | – | – | – |
| projects.view_all | ✓ | ✓ | – | – | – | – |
| projects.create | ✓ | ✓ | – | ✓ | – | – |
| projects.manage_own (edit, members) | ✓ | ✓ | – | ✓ if creator/project lead | – | – |
| View a project | all | all | – | member | member | member |
| tasks.create (in member project) | ✓ | ✓ | – | ✓ | ✓ | – |
| tasks.assign | ✓ | ✓ | – | ✓ | self only | – |
| tasks.update_assigned (status, comment) | ✓ | ✓ | – | ✓ | ✓ | ✓ |
| tasks.update_any (in project) | ✓ | ✓ | – | ✓ | – | – |
| announcements.view (targeted to me) | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| announcements.post_all | ✓ | ✓ | ✓ | – | – | – |
| announcements.post_project | ✓ | ✓ | – | ✓ own projects | – | – |
| leads.view | ✓ | ✓ | – | ✓ | – | – |
| leads.manage | ✓ | ✓ | – | – | – | – |
| cms.edit | ✓ | ✓ | – | – | – | – |
| audit.view | ✓ | ✓ | – | – | – | – |

## Rules that are not a simple yes/no
1. **No self-escalation:** nobody can change their own role.
2. **Last super admin:** cannot be disabled, demoted, or deleted (DB trigger enforces).
3. **Invite ceiling:** a user can only invite/edit roles with lower rank than their own (super_admin can invite super_admin).
4. **Inactive users** (`is_active = false`) fail every RLS helper, even with a valid token.
5. **Project membership** is checked through `project_members`; admins bypass it.

## Implementation
```ts
// lib/permissions.ts (shape)
export const ROLES = ['super_admin','admin','hr','project_manager','employee','intern'] as const;
export type Role = typeof ROLES[number];
export const PERMISSIONS: Record<Permission, readonly Role[]> = { /* mirrors matrix */ };
export const can = (role: Role, p: Permission) => PERMISSIONS[p].includes(role);
export const canManageRole = (actor: Role, target: Role) => ...; // rank rule
```
SQL helpers (see schema): `auth_role()`, `has_role(variadic roles)`, `is_project_member(project_id)`.
