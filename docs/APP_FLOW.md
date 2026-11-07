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
/auth/callback          Supabase email link handler
/portal                 Role dashboard
/portal/set-password    First login / reset
/portal/profile         My profile
/portal/notifications   All notifications
/portal/search?q=       Global search
/portal/users           Users list            (super_admin, admin, hr)
/portal/users/new       Invite user           (super_admin, admin, hr)
/portal/users/[id]      User detail / edit
/portal/directory       Team directory        (all)
/portal/documents       My documents          (all)
/portal/documents/review  Review queue        (super_admin, admin, hr)
/portal/projects        Projects list
/portal/projects/new    New project           (super_admin, admin, pm)
/portal/projects/[id]   Board / list / members / files
/portal/tasks           My tasks (cross-project)
/portal/announcements   Feed + compose
/portal/leads           Leads                 (super_admin, admin; pm read)
/portal/leads/[id]      Lead detail
/portal/cms             CMS home              (super_admin, admin)
/portal/cms/[collection]  Editor per collection
/portal/audit           Audit log             (super_admin, admin read)
```

## 2. Visitor journey
```
Land on /  ──► Hero (mascot waves, "Hi! Want a tour?")
   │
   ├─ scroll ─► each section: mascot changes pose + bubble line from CMS
   ├─ click mascot ─► quick-nav menu ─► jump to section / Start a project / Book a call
   ├─ click service card ─► /services/[slug] ─► CTA ─► contact form
   ├─ click work card ─► /work/[slug] ─► "Start a similar project" ─► contact form
   └─ Start a project ─► contact form (#contact)
            │
            ▼
     Fill form ─► Turnstile check ─► submit
            │
            ├─ success: inline thank-you, mascot "celebrate", lead saved, admins emailed + notified
            └─ error: field errors inline; rate-limited ─► "Please try again in a few minutes"
```

## 3. Account lifecycle
```
Admin/HR ─► /portal/users/new ─► fill name, email, role, department, designation
   │   (HR may only pick employee or intern; admin cannot pick super_admin)
   ▼
Server action: validate ─► check permission ─► auth.admin.inviteUserByEmail
   ─► trigger creates profiles row ─► audit_log "user.invited"
   ▼
Employee receives email ─► clicks link ─► /auth/callback ─► /portal/set-password
   ─► sets password ─► /portal (dashboard)

Later:
- Forgot password: /forgot-password ─► email ─► /auth/callback ─► /portal/set-password
- Disable: admin ─► user detail ─► Disable ─► ban in auth + is_active=false ─► sessions rejected
- Role change: admin ─► user detail ─► role select ─► audit_log "user.role_changed"
```

## 4. Login & routing guard
```
Any /portal/* request
  └─ middleware: refresh session
        ├─ no session ─► redirect /login?next=<path>
        └─ session ─► page server component
              └─ requireRole(allowed) ─► not allowed ─► /portal (with toast "No access")
/login with valid session ─► redirect /portal
Inactive profile ─► sign out ─► /login?error=disabled
```

## 5. Document submission
```
Employee ─► /portal/documents ─► Upload
   ─► pick category, title, optional expiry, file (≤10 MB, allowed type)
   ─► browser uploads to employee-docs/<uid>/<doc_id>/file
   ─► server action inserts documents row (status=pending)
   ─► notify HR/admin
HR ─► /portal/documents/review ─► open ─► preview (signed URL)
   ├─ Approve ─► status=approved ─► notify owner ─► audit
   └─ Reject + reason ─► status=rejected ─► notify owner ─► audit
         └─ owner re-uploads ─► new version, old file deleted ─► status=pending

HR "Request document" ─► document_requests row (user, category, due date)
   ─► notify user ─► user's upload fulfils request automatically (same category)

Daily cron: documents expiring in 14 days ─► notify owner + HR (+ email)
```

## 6. Projects & tasks
```
PM ─► /portal/projects/new ─► name, client, dates, members ─► project created
   ─► members notified
PM/admin/employee(member) ─► board ─► "Add task" ─► title, assignee, priority, due
   ─► assignee notified
Assignee ─► drag card To do ─► In progress ─► Review
PM ─► Review ─► Done (or back to In progress with comment)
Comment with @name ─► mention notification
Attach file ─► task-files bucket ─► task_attachments row
Daily cron: tasks due tomorrow and overdue ─► notify assignee
```
Intern: can move own assigned cards and comment, cannot create or assign tasks.

## 7. Announcements
```
Admin/HR (or PM for own project) ─► compose ─► audience (all / role / project), pin?
   ─► publish ─► notifications for audience
User ─► opens announcement ─► announcement_reads row ─► author sees read count
```

## 8. Leads
```
New lead (from site) ─► status=new ─► admin opens ─► assign owner
   ─► contacted ─► proposal_sent ─► won / lost (lost needs reason)
   ─► notes timeline; each change in audit log
```

## 9. CMS publish
```
Admin ─► /portal/cms ─► choose collection (e.g. Services)
   ─► edit item (form + image upload to site-media) ─► Save draft (is_published=false)
   ─► Preview (opens /?preview=1 using draft data, admin only)
   ─► Publish ─► is_published=true ─► revalidateTag('site:<collection>') ─► audit
Section order / visibility ─► site_settings.sections (drag list) ─► Publish ─► revalidateTag('site')
```

## 10. Notifications
```
Event (task assigned, mention, doc reviewed, ...) ─► insert notifications row
   ─► Realtime channel per user ─► bell badge +1, toast
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
