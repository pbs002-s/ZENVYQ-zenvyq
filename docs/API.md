# API — Server Actions & Routes

All mutations follow one pipeline:
```
input ─► zod.parse ─► getSessionUser() ─► can()/requirePermission (UX + defence in depth)
      ─► Supabase write via user-scoped client  (RLS + guard triggers are the real check)
      ─► DB triggers write notifications + audit_log
      ─► invalidate cache tag / revalidatePath
      ─► return { ok: true, data } | { ok: false, error, fieldErrors? }
```
Only auth-admin operations (invite, ban/unban, deleting storage objects owned by others) use the service-role client, and only **after** a DB permission check (RPC) has succeeded with the user's own client.

Response envelope (all actions):
```ts
type ActionResult<T> = { ok: true; data: T } | { ok: false; error: string; fieldErrors?: Record<string, string[]> };
```
Error messages shown to users are generic; details go to server logs. Guard triggers raise with `errcode = 'P0001'` and a short machine code in the message (`forbidden_column`, `last_super_admin`, `self_review`, …), which `lib/errors.ts` maps to friendly text.

## Route handlers
| Method | Path | Auth | Purpose |
|---|---|---|---|
| POST | `/api/leads` | public | Inquiry form. Body: `{ name, email, company?, budget?, service?, message, turnstileToken, website }` (`website` = honeypot; if filled, return 201 and drop). Order: zod → honeypot → Turnstile verify → `hit_rate_limit` per IP (3/10 min) and global (50/h) → insert (service role; trigger notifies admins) → Resend email to `LEAD_NOTIFY_EMAILS`. An email failure does **not** fail the request (lead is already saved). Returns 201 / 400 / 429 / 503 (DB unreachable → client shows the fallback email address). |
| GET | `/auth/confirm` | public | `verifyOtp({ token_hash, type })` for invite and recovery links, then redirects to `next` (allow-list: relative paths starting with `/portal/`). Invalid/expired → `/login?error=link_expired`. |
| GET | `/api/draft` | admin session | Enables draft mode and redirects to `?path=` (relative only). `?disable=1` exits. |
| POST | `/api/revalidate` | `x-revalidate-secret` header | Optional manual cache bust for a tag. Constant-time secret compare. |

## Server actions (by feature folder)
### auth — `app/(auth)/actions.ts`
| Action | Who | Notes |
|---|---|---|
| `signIn({ email, password })` | public | Generic "Invalid email or password". After sign-in, inactive profile → sign out + `error=disabled`. |
| `signOut()` | user | |
| `requestPasswordReset({ email })` | public | Always returns ok (no account enumeration). Rate-limited by Supabase Auth. |
| `setPassword({ password })` | user | Min 10 chars, zxcvbn score ≥ 3; on success sets `profiles.password_set_at` (service role). |

### users — `app/portal/users/actions.ts`
| Action | Who | Notes |
|---|---|---|
| `inviteUser({ email, fullName, role, department?, designation?, joinedOn? })` | users.invite | `assert_can_invite(role)` RPC → service-role invite → set role/job fields → `log_audit('user.invited')`. Duplicate email → field error. |
| `resendInvite(id)` | users.invite | Only while `password_set_at is null`. |
| `updateUser(id, patch)` | users.edit | User client; RLS rank check + guard limits columns. |
| `changeRole(id, role)` | users.change_role | `change_user_role` RPC. |
| `setUserActive(id, active)` | users.disable | `set_user_active` RPC → service-role ban/unban. If the ban call fails, the RPC change is reverted and an error is shown. |
| `updateMyProfile(patch)` | self | `full_name, phone, avatar_url, bio, skills` only. |
| `exportUserDocuments(id)` | users.offboard | Signed URLs (5 min) for all of the user's documents, for manual download. |
| `purgeUserDocuments(id)` | users.offboard | Only when the user is inactive. Deletes all their `employee-docs` objects and document rows (service role) → `log_audit('user.documents_purged', count)`. Requires typing the user's email to confirm. |

### documents — `app/portal/documents/actions.ts`
| Action | Who |
|---|---|
| `createDocument({ id, category, title, filePath, fileSize, mimeType, expiresOn?, requestId? })` | documents.own; `filePath` must equal `<uid>/<id>/…` (also a DB check) |
| `replaceDocumentFile(id, { filePath, fileSize, mimeType, title? })` | owner while pending/rejected; deletes old object after the row update succeeds |
| `reviewDocument(id, { decision: 'approved' \| 'rejected', note? })` | documents.review; `note` required for rejected; not own document |
| `requestDocument({ userId, category, note?, dueOn? })` | documents.request |
| `getDocumentUrl(id)` | owner or reviewer — 60 s signed URL |
| `deleteDocument(id)` | owner while pending, reviewers |

### projects & tasks — `app/portal/projects/actions.ts`
| Action | Who |
|---|---|
| `createProject(input)` | projects.create (creator + lead auto-added as members by trigger) |
| `updateProject(id, patch)` | admins or project lead |
| `setProjectMembers(id, userIds[])` | admins or project lead (lead cannot be removed) |
| `createTask(projectId, input)` | tasks.create (member, not intern; employees may assign only themselves) |
| `updateTask(id, patch)` | `can_manage_tasks` → any field; assignee → status/description only |
| `moveTask(id, { status, beforeId?, afterId? })` | same as updateTask; server computes the position and rebalances if needed |
| `addComment(taskId, { body })` | project member; mentions parsed from `@[Name](uuid)` tokens produced by the editor |
| `editComment(id, { body })` / `deleteComment(id)` | author (delete: author or admins) |
| `addAttachment(taskId, { filePath, fileName, fileSize })` | project member |
| `deleteAttachment(id)` | uploader or task manager |
| `deleteTask(id)` | `can_manage_tasks` |

### announcements — `app/portal/announcements/actions.ts`
`createAnnouncement`, `updateAnnouncement`, `deleteAnnouncement`, `markAnnouncementRead` (idempotent upsert).

### leads — `app/portal/leads/actions.ts`
`updateLeadStatus(id, { status, lostReason? })`, `assignLead(id, ownerId)`, `addLeadNote(id, { body })`, `deleteLead(id)` (super_admin; for data-removal requests).

### cms — `app/portal/cms/actions.ts`
| Action | Notes |
|---|---|
| `saveSettingsDraft(section, data)` | Writes `site_settings.draft[section]` |
| `publishSettings()` | Copies draft → live, invalidates `site:settings` (and `site`) |
| `upsertItem(collection, data)` | Saves; `is_published` unchanged |
| `setPublished(collection, id, bool)` | Invalidates `site:<collection>` and the slug tag |
| `reorderItems(collection, ids[])` | Rewrites `sort_order`; invalidates the collection tag |
| `deleteItem(collection, id)` | Also deletes its `site-media` files |
| `uploadSvg(collection, file)` | Server-side sanitise + upload (TRD §5) |

`collection` is validated against a fixed allow-list (`services, case_studies, clients, testimonials, stats, process_steps, engagement_models, faqs`) — never interpolated from user input.

### notifications — `app/portal/notifications/actions.ts`
`markRead(id)`, `markAllRead()`.

## Pagination
| Kind | Method | Default / max | Params |
|---|---|---|---|
| Tables (users, document review, projects, leads, CMS items) | Offset | 25 / 100 | `?page=&size=&sort=&dir=&q=` — `sort` validated against a per-table allow-list |
| Feeds (notifications, comments, lead notes, audit log, announcements) | Cursor on `(created_at, id)` desc | 20 / 50 | `?cursor=<created_at>_<id>` |
| Kanban board | None — whole project loaded | Soft cap 500 tasks; above that, list view only | — |

`lib/pagination.ts` returns `{ items, nextCursor }` or `{ items, total, page, size }`.

## Shared server helpers (`lib/`)
| Helper | Purpose |
|---|---|
| `getSessionUser()` | `{ id, email, role, isActive, passwordSet }` or null (React `cache()` per request) |
| `requireRole(...roles)` / `requirePermission(p)` | Redirect to `/portal` with "No access" if not allowed |
| `logAudit(action, entity, entityId, diff?)` | Service role only, for non-DB events |
| `sanitizeHtml(html)` / `sanitizeSvg(svg)` | DOMPurify allow-lists |
| `mapDbError(e)` | Postgres / trigger error → generic user message + log |
| `safeRedirect(path, prefix)` | Relative-path allow-list for `next` / `path` params |
