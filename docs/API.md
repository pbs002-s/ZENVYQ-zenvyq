# API — Server Actions & Routes

All mutations follow one pipeline:
```
input ─► zod.parse ─► getSessionUser() ─► requireRole/can() ─► Supabase write (RLS applies)
      ─► log_audit() if sensitive ─► notify() if someone should know ─► revalidatePath/Tag
      ─► return { ok: true, data } | { ok: false, error, fieldErrors? }
```
Response envelope (all actions):
```ts
type ActionResult<T> = { ok: true; data: T } | { ok: false; error: string; fieldErrors?: Record<string, string[]> };
```
Error messages shown to users are generic; details go to server logs.

## Route handlers
| Method | Path | Auth | Purpose |
|---|---|---|---|
| POST | `/api/leads` | public | Inquiry form. Body: `{ name, email, company?, budget?, service?, message, turnstileToken, website (honeypot) }`. Verifies Turnstile, rate limit 3/10 min per IP, inserts with service role, notifies admins, emails via Resend. Returns 201 / 400 / 429. |
| GET | `/auth/callback` | public | Exchanges Supabase code for session, redirects to `next` (whitelisted to `/portal/*`). |
| POST | `/api/revalidate` | secret header | Optional: manual cache bust (`x-revalidate-secret`). |

## Server actions (by feature folder)
### auth — `app/(auth)/actions.ts`
| Action | Who | Notes |
|---|---|---|
| `signIn({ email, password })` | public | Generic "Invalid email or password" on failure |
| `signOut()` | user | |
| `requestPasswordReset({ email })` | public | Always returns ok (no account enumeration) |
| `setPassword({ password })` | user | Min 10 chars, zxcvbn score ≥ 3 |

### users — `app/portal/users/actions.ts`
| Action | Who |
|---|---|
| `inviteUser({ email, fullName, role, department?, designation? })` | users.invite + rank rule |
| `updateUser(id, patch)` | users.edit + rank rule |
| `changeRole(id, role)` | users.change_role |
| `setUserActive(id, active)` | users.disable |
| `updateMyProfile(patch)` | self (no role/is_active fields) |

### documents — `app/portal/documents/actions.ts`
| Action | Who |
|---|---|
| `createDocument({ category, title, filePath, fileSize, mimeType, expiresOn?, requestId? })` | documents.own; filePath must start with own uid |
| `reviewDocument(id, { decision: 'approved' \| 'rejected', note? })` | documents.review |
| `requestDocument({ userId, category, note?, dueOn? })` | documents.request |
| `getDocumentUrl(id)` | owner or reviewer — returns 60 s signed URL |
| `deleteDocument(id)` | owner while pending, reviewers |

### projects & tasks — `app/portal/projects/actions.ts`
| Action | Who |
|---|---|
| `createProject(input)` | projects.create |
| `updateProject(id, patch)` | admins or project lead |
| `setProjectMembers(id, userIds[])` | admins or project lead |
| `createTask(projectId, input)` | tasks.create (member, not intern) |
| `updateTask(id, patch)` | tasks.update_any or assignee (status/description only) |
| `moveTask(id, { status, position })` | same as updateTask |
| `addComment(taskId, { body })` | project member; parses @mentions |
| `addAttachment(taskId, { filePath, fileName, fileSize })` | project member |
| `deleteTask(id)` | admins or project lead |

### announcements — `app/portal/announcements/actions.ts`
`createAnnouncement`, `updateAnnouncement`, `deleteAnnouncement`, `markAnnouncementRead`.

### leads — `app/portal/leads/actions.ts`
`updateLeadStatus(id, { status, lostReason? })`, `assignLead(id, ownerId)`, `addLeadNote(id, { body })`.

### cms — `app/portal/cms/actions.ts`
`updateSiteSettings(section, data)`, `upsertItem(collection, data)`, `reorderItems(collection, ids[])`, `setPublished(collection, id, bool)`, `deleteItem(collection, id)`.
Each publishing action calls `revalidateTag('site')` or the narrower collection tag.

### notifications — `app/portal/notifications/actions.ts`
`markRead(id)`, `markAllRead()`.

## Shared server helpers (`lib/`)
| Helper | Purpose |
|---|---|
| `getSessionUser()` | Returns `{ id, email, role, isActive }` or null (cached per request) |
| `requireRole(...roles)` / `requirePermission(p)` | Throws redirect to `/portal` if not allowed |
| `notify(userIds, { type, title, link, payload })` | Inserts notifications via RPC |
| `logAudit(action, entity, entityId, diff?)` | Calls `log_audit()` RPC |
| `sanitizeHtml(html)` | DOMPurify (isomorphic) allow-list for rich text |
