# App Flow

Related: [PRD](PRD.md) · [Roles & permissions](ROLES_PERMISSIONS.md) · [API](API.md)

## 1. Sitemap
```
/                       Home (all landing sections)
/services/[slug]        Service detail
/work                   Portfolio list
/work/[slug]            Case study
/about                  About + team
/contact                Contact form
/privacy  /terms        Legal
/login                  Email + password
/forgot-password        Request reset email
/auth/confirm           Email link handler (invite, recovery)
/api/draft              Enter/exit CMS preview (admin)
/portal                 Role dashboard
/portal/set-password    First login / reset
/portal/profile         My profile
/portal/notifications   All notifications
/portal/search?q=       Global search
/portal/users           Users list            (super_admin, admin, hr)
/portal/users/new       Invite user           (super_admin, admin, hr)
/portal/users/[id]      User detail / edit / offboarding
/portal/directory       Team directory        (all)
/portal/documents       My documents          (all)
/portal/documents/review  Review queue        (super_admin, admin, hr)
/portal/projects        Projects list
/portal/projects/new    New project           (super_admin, admin, project_manager)
/portal/projects/[id]   Board / list / members / files
/portal/tasks           My tasks (cross-project)
/portal/announcements   Feed + compose
/portal/leads           Leads                 (super_admin, admin; project_manager read-only)
/portal/leads/[id]      Lead detail
/portal/cms             CMS home              (super_admin, admin)
/portal/cms/[collection]  Editor per collection
/portal/audit           Audit log             (super_admin, admin)
```

## 2. Visitor journey
```
Land on /  ──► Hero (mascot waves, "Hi! Want a quick tour?")
   │
   ├─ scroll ─► each section: mascot changes pose + bubble line from CMS
   ├─ click mascot ─► quick-nav menu ─► jump to section / Start a project / Book a call
   ├─ click service card ─► /services/[slug] ─► CTA ─► contact form (service pre-selected)
   ├─ click work card ─► /work/[slug] ─► "Start a similar project" ─► contact form
   └─ Start a project ─► contact form (#contact)
            │
            ▼
     Fill form ─► Turnstile check ─► submit
            │
            ├─ success: inline thank-you, mascot "celebrate", lead saved, admins notified + emailed
            ├─ validation error: field errors inline, input kept
            ├─ rate-limited (429): "Please try again in a few minutes"
            └─ server unavailable (503): "Something went wrong — email us at <contact.email>"
```

## 3. Account lifecycle
```
Admin/HR ─► /portal/users/new ─► name, email, role, department, designation, joined date
   │   Role list shows only roles the actor may assign:
   │   super_admin → any · admin → hr / project_manager / employee / intern · hr → employee / intern
   ▼
Server action: zod ─► assert_can_invite(role) ─► auth.admin.inviteUserByEmail
   ─► trigger creates profile (role intern) ─► action sets real role + job fields
   ─► audit "user.invited"
   ▼
Employee opens email ─► /auth/confirm (verifyOtp) ─► /portal/set-password
   ─► sets password (≥10 chars, strength ≥3) ─► password_set_at = now ─► /portal
   (Leaves before setting a password ─► any /portal page sends them back to set-password.)
   (Expired link ─► /login?error=link_expired ─► "Ask your admin to resend the invite".)

Later:
- Resend invite: user detail ─► "Resend invite" (only while password not set)
- Forgot password: /forgot-password ─► always "If the account exists, we sent a link"
     ─► email ─► /auth/confirm ─► /portal/set-password
- Role change: user detail ─► role select ─► change_user_role RPC ─► audit "user.role_changed"
- Disable: user detail ─► Disable ─► set_user_active RPC ─► Auth ban ─► audit "user.disabled"
     ─► next request: RLS denies everything, portal layout signs them out
```

### 3a. Offboarding
```
Admin ─► user detail ─► Disable (above)
   ─► "Export documents" ─► list of 5-minute download links ─► admin saves files offline
   ─► "Delete documents" ─► type the user's email to confirm ─► all files + rows deleted
   ─► audit "user.documents_purged"
Profile row stays (inactive) so tasks, comments and audit entries keep their author.
Turn off `show_on_site` if they were on the public team page.
```

## 4. Login & routing guard
```
Any /portal/* request
  └─ proxy (middleware): refresh session
        ├─ no session ─► redirect /login?next=<path>
        └─ session ─► portal layout
              ├─ profile inactive ─► sign out ─► /login?error=disabled
              ├─ password_set_at null ─► /portal/set-password
              └─ page server component
                    └─ requirePermission(p) ─► not allowed ─► /portal (toast "No access")
/login with valid session ─► redirect /portal
```

## 5. Document submission
```
Employee ─► /portal/documents ─► Upload
   ─► pick category, title, optional expiry, file (≤10 MB, allowed type; checked in browser)
   ─► browser generates doc id ─► uploads to employee-docs/<uid>/<doc_id>/<file> (progress bar)
         └─ upload fails ─► "Upload failed, try again" (Retry; nothing saved)
   ─► createDocument action inserts row (status=pending)
         └─ insert fails ─► error shown; orphan file removed by weekly cleanup
   ─► trigger notifies HR/admin; fulfils a matching document request
HR ─► /portal/documents/review ─► open ─► preview (60 s signed URL)
   ├─ Approve ─► status=approved ─► trigger notifies owner + audit
   └─ Reject + reason ─► status=rejected ─► trigger notifies owner + audit
         └─ owner "Replace file" ─► uploads new file ─► same row back to pending,
            review fields cleared ─► old file deleted
(Reviewers cannot approve their own documents.)

HR "Request document" ─► document_requests row (user, category, due date)
   ─► trigger notifies user ─► user's next upload in that category fulfils it

Daily cron: documents expiring in 14 and 3 days ─► notify owner + HR (+ email)
```

## 6. Projects & tasks
```
PM ─► /portal/projects/new ─► name, client, dates, lead, members ─► project created
   ─► creator + lead auto-added as members ─► each new member notified
PM/admin/employee (member) ─► board ─► "Add task" ─► title, assignee, priority, due
   (employees can only assign themselves; interns cannot add tasks)
   ─► assignee notified
Assignee ─► drags card To do ─► In progress ─► Review ─► Done
   (Review is a team convention, not a gate — DECISIONS D-010)
PM can move any card, reopen Done cards, reassign, edit, delete
Drag rejected (network/permission) ─► card snaps back + toast
Comment with @name (picker lists project members only) ─► mention notification
Attach file ─► task-files bucket ─► task_attachments row
Daily cron: tasks due tomorrow and overdue ─► notify assignee
```
Intern: can move their own assigned cards, edit the description, comment and attach; cannot create, assign, or edit other fields.

## 7. Announcements
```
Admin/HR (any audience) or PM (project audience, projects they lead) ─► compose
   ─► audience (all / role / project), pin? ─► publish
   ─► trigger notifies everyone in the audience
User ─► opens announcement ─► announcement_reads row ─► author sees "Read by N of M" + names
```

## 8. Leads
```
New lead (from site) ─► status=new ─► admin opens ─► assign owner
   ─► contacted ─► proposal_sent ─► won / lost (lost needs reason)
   ─► notes timeline; status and owner changes audited
PM: read-only view of the pipeline (current default — OPEN_QUESTIONS #13)
```

## 9. CMS publish
```
Admin ─► /portal/cms ─► choose collection (e.g. Services)
   ─► edit item (form + image upload to site-media) ─► Save (is_published unchanged)
   ─► Preview ─► /api/draft?path=/services/<slug> ─► page renders drafts with "Preview mode" bar
   ─► Publish ─► is_published=true ─► cache tag invalidated ─► audit "cms.published"
Settings (hero, contact, SEO, sections order/visibility, mascot lines)
   ─► saved into site_settings.draft ─► Preview ─► Publish ─► draft copied live ─► invalidate
Unsaved changes ─► "Leave without saving?" prompt
```

## 10. Notifications
```
Event (task assigned, mention, doc reviewed, ...) ─► DB trigger inserts notifications row
   ─► Realtime (filter user_id = me) ─► bell badge +1, toast
Click ─► mark read ─► navigate to target link
"Mark all read" on /portal/notifications
```

## 11. Portal layout
```
┌───────────────────────────────────────────────────────────┐
│ Logo   Search [..........]            🔔  Avatar ▾         │
├──────────────┬────────────────────────────────────────────┤
│ Dashboard    │                                            │
│ My tasks     │   Page content                             │
│ Projects     │                                            │
│ Documents    │                                            │
│ Announcements│                                            │
│ Directory    │                                            │
│ ── Admin ──  │                                            │
│ Users        │                                            │
│ Leads        │                                            │
│ Site CMS     │                                            │
│ Audit log    │                                            │
└──────────────┴────────────────────────────────────────────┘
```
Sidebar items are filtered by `can(role, permission)`; mobile uses a drawer.

## 12. Error & edge states
| Situation | Behaviour |
|---|---|
| Supabase paused or down | Public site serves cached pages; contact form returns 503 with the fallback email. Portal shows a full-page "Service temporarily unavailable — try again shortly" (`portal/error.tsx`), no stack trace. |
| Session expired mid-use | Next server action returns unauthenticated → redirect to `/login?next=<current path>`; unsaved CMS and task-description input kept in `sessionStorage`. |
| User disabled while logged in | Next request: RLS returns nothing / RPC errors → portal layout signs out → `/login?error=disabled`. |
| Realtime disconnected | Bell shows a small "offline" dot; reconnects with backoff; on reconnect, refetches unread count. Notifications are rows, so none are lost. |
| File upload interrupted | Progress bar shows error + Retry; no row is created until the upload succeeds. |
| Denied by RLS / guard trigger | Generic toast "You don't have permission to do that"; detail logged. The UI should normally prevent this. |
| Concurrent edit (two admins, one CMS item) | On save, if `updated_at` changed since load: "Updated by <name> at <time> — overwrite or reload?" |
| Empty states | Every list has an `EmptyState` with the next action (e.g. "No documents yet — Upload"). |
| 404 / no access to a record | `not-found.tsx`; a row hidden by RLS looks the same as a missing one (no existence leak). |
| Browser offline | Toast "You're offline"; actions disabled until back online. |
