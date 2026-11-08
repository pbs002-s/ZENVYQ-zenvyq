# Backend Schema (Supabase Postgres)

Status: designed, not yet migrated. When migrations exist, this doc must match `supabase/migrations/`.
Related: [Roles & permissions](ROLES_PERMISSIONS.md) · [TRD](TRD.md) · [API](API.md)

Conventions: `uuid` primary keys (`gen_random_uuid()`), `created_at timestamptz default now()`, snake_case, RLS enabled on **every** table. Every `security definer` function has `set search_path = public`, `revoke execute ... from public`, and then the minimum `grant`.

Migration order (tables below are listed in this order):
`0001_extensions_enums` → `0002_people` → `0003_documents` → `0004_projects_tasks` → `0005_announcements` → `0006_leads` → `0007_notifications_audit` → `0008_cms` → `0009_rls` → `0010_storage` → `0011_cron`.
Rule: **notification and audit triggers are attached in `0007`**, after the `notify()` / `log_audit()` functions exist — earlier migrations only create tables.

## 0. Extensions & shared trigger
```sql
create extension if not exists pg_trgm;    -- portal search
create extension if not exists pg_cron;
create extension if not exists pg_net;     -- cron → Edge Functions

create function set_updated_at() returns trigger language plpgsql as $$
begin new.updated_at = now(); return new; end $$;
```
`set_updated_at` is attached (`before update for each row`) to every table with an `updated_at` column: `profiles, documents, projects, tasks, task_comments, announcements, leads, site_settings` and all CMS collection tables.

## 1. Enums
```sql
create type app_role        as enum ('super_admin','admin','hr','project_manager','employee','intern');
create type doc_category    as enum ('id_proof','contract','certificate','nda','report','other');
create type doc_status      as enum ('pending','approved','rejected');
create type project_status  as enum ('planning','active','on_hold','completed','cancelled');
create type task_status     as enum ('todo','in_progress','review','done');
create type task_priority   as enum ('low','medium','high','urgent');
create type audience_type   as enum ('all','role','project');
create type lead_status     as enum ('new','contacted','proposal_sent','won','lost');
```

## 2. People
```sql
create table profiles (
  id              uuid primary key references auth.users(id) on delete cascade,
  email           text not null unique,
  full_name       text not null check (char_length(full_name) between 1 and 120),
  role            app_role not null default 'intern',   -- lowest; the invite action sets the real role
  designation     text,
  department      text,
  phone           text,
  avatar_url      text,
  bio             text check (char_length(bio) <= 1000),
  skills          text[] not null default '{}',
  joined_on       date,
  show_on_site    boolean not null default false,       -- appears on /about team
  is_active       boolean not null default true,
  password_set_at timestamptz,                          -- null = invite not completed
  created_at      timestamptz not null default now(),
  updated_at      timestamptz not null default now()
);
create index profiles_name_trgm on profiles using gin (full_name gin_trgm_ops);
```

**Triggers on `profiles`**
| Trigger | When | Does |
|---|---|---|
| `on_auth_user_created` | after insert on `auth.users` | Inserts profile with `email`, `full_name = raw_user_meta_data->>'full_name'`, role `intern`. Role is **never** read from metadata: `inviteUserByEmail({ data })` writes *user* metadata, which the user can edit. |
| `guard_profile_update` | before update | Rules below. |
| `protect_last_super_admin` | before update/delete | Raises if the change leaves zero active `super_admin` rows. |
| `audit_profile_change` | after update | Writes `user.role_changed` / `user.disabled` / `user.enabled`. |

`guard_profile_update` rules:
1. `auth.uid() is null` (service role / SQL editor) → allowed.
2. `id` and `email` never change through a user session.
3. `role` and `is_active` change only when the transaction-local flag `app.privileged_change = 'on'` is set, i.e. only inside `change_user_role()` / `set_user_active()`.
4. A user editing **their own** row may change only `full_name, phone, avatar_url, bio, skills` (managers too — their job fields are edited by another manager).
5. Managers editing others (RLS rank-checked) may change only `full_name, designation, department, phone, avatar_url, bio, skills, joined_on, show_on_site`.

## 3. Helper & privileged functions (`security definer`, `set search_path = public`)
```sql
create function auth_role() returns app_role language sql stable security definer as $$
  select role from profiles where id = auth.uid() and is_active $$;

create function has_role(variadic roles app_role[]) returns boolean language sql stable security definer as $$
  select coalesce(auth_role() = any(roles), false) $$;

-- Who may invite / edit / change the role of / disable a user holding `target`
create function can_manage_role(actor app_role, target app_role) returns boolean language sql immutable as $$
  select case actor
    when 'super_admin' then true
    when 'admin'       then target in ('hr','project_manager','employee','intern')
    when 'hr'          then target in ('employee','intern')
    else false end $$;

create function is_project_member(p uuid) returns boolean language sql stable security definer as $$
  select auth_role() is not null and exists(
    select 1 from project_members where project_id = p and user_id = auth.uid()) $$;

create function is_project_lead(p uuid) returns boolean language sql stable security definer as $$
  select auth_role() is not null and exists(
    select 1 from projects where id = p and lead_id = auth.uid()) $$;

-- Full task control in a project: admins, the project lead, or a PM who is a member
create function can_manage_tasks(p uuid) returns boolean language sql stable security definer as $$
  select has_role('super_admin','admin') or is_project_lead(p)
      or (has_role('project_manager') and is_project_member(p)) $$;
```
`can_manage_role` is the single definition of the rank rule. `lib/permissions.ts → canManageRole` mirrors it, and a test compares the two across all 36 role pairs.

**Privileged RPCs** — callable by `authenticated`; every check lives in SQL, so the UI cannot bypass it:
| Function | Checks | Effect |
|---|---|---|
| `change_user_role(target uuid, new_role app_role)` | caller active; `target <> auth.uid()`; `can_manage_role(caller, old_role)` **and** `can_manage_role(caller, new_role)`; last-super-admin trigger | Sets `app.privileged_change`, updates `role` (audited by trigger) |
| `set_user_active(target uuid, active boolean)` | `has_role('super_admin','admin')`; not self; `can_manage_role`; last-super-admin trigger | Updates `is_active` (audited). The server action then bans/unbans in Auth with the service role. |
| `assert_can_invite(new_role app_role)` | `has_role('super_admin','admin','hr')` and `can_manage_role(caller, new_role)` | Raises if not allowed. Called by `inviteUser` **before** the service-role invite. |
| `get_public_team()` | none (public columns only) | Returns `id, full_name, designation, department, avatar_url, bio, skills` where `show_on_site and is_active`, ordered by `joined_on`. Granted to `anon`. Email/phone never exposed. |
| `rebalance_task_column(p uuid, s task_status)` | `is_project_member(p)` or admin | Renumbers positions in that column to 1024, 2048, … |

## 4. Documents
```sql
create table document_requests (
  id            uuid primary key default gen_random_uuid(),
  user_id       uuid not null references profiles(id) on delete cascade,
  category      doc_category not null,
  note          text check (char_length(note) <= 1000),
  due_on        date,
  requested_by  uuid not null references profiles(id) default auth.uid(),
  fulfilled_at  timestamptz,
  created_at    timestamptz not null default now()
);
create index on document_requests (user_id) where fulfilled_at is null;

create table documents (
  id           uuid primary key,                -- generated in the browser so the storage path can contain it
  owner_id     uuid not null references profiles(id) on delete cascade,
  category     doc_category not null,
  title        text not null check (char_length(title) between 1 and 120),
  file_path    text not null,                   -- <owner_id>/<id>/<slugified-filename> in bucket employee-docs
  file_size    int  not null check (file_size between 1 and 10485760),
  mime_type    text not null check (mime_type in ('application/pdf','image/jpeg','image/png',
                 'application/vnd.openxmlformats-officedocument.wordprocessingml.document')),
  status       doc_status not null default 'pending',
  review_note  text,
  reviewed_by  uuid references profiles(id),
  reviewed_at  timestamptz,
  expires_on   date,
  request_id   uuid references document_requests(id) on delete set null,
  created_at   timestamptz not null default now(),
  updated_at   timestamptz not null default now(),
  check (status <> 'rejected' or review_note is not null),
  check (file_path like owner_id::text || '/' || id::text || '/%')
);
create index on documents (owner_id);
create index on documents (status) where status = 'pending';
create index on documents (expires_on) where expires_on is not null;
```

**Triggers on `documents`**
| Trigger | Does |
|---|---|
| `guard_document_update` (before update) | **Owner**: may change only `title, category, file_path, file_size, mime_type, expires_on`, only while `old.status in ('pending','rejected')`; the trigger forces `status = 'pending'` and clears `review_note, reviewed_by, reviewed_at` (= re-upload). **Reviewer**: may change only `status, review_note`; trigger sets `reviewed_by = auth.uid()`, `reviewed_at = now()`. Nobody can review **their own** document. |
| `fulfil_document_request` (after insert) | If `request_id` is set and belongs to `owner_id` with the same category → sets `fulfilled_at`. If null → fulfils the oldest open request of that category for the owner. |
| `notify_document_events` (after insert/update) | Insert → `doc.submitted` to reviewers. Status → approved/rejected → `doc.reviewed` to owner. |
| `audit_document_review` (after update of status) | `doc.approved` / `doc.rejected` with note. |
| `notify_document_request` (after insert on `document_requests`) | `doc.requested` to the user. |

After a re-upload, the **server action** deletes the previous storage object with the service role (it reads the old `file_path` before updating).

## 5. Projects & tasks
```sql
create table projects (
  id           uuid primary key default gen_random_uuid(),
  name         text not null check (char_length(name) between 1 and 120),
  client_name  text,
  description  text,
  status       project_status not null default 'planning',
  start_on     date,
  end_on       date check (end_on is null or start_on is null or end_on >= start_on),
  lead_id      uuid references profiles(id),       -- PM in charge
  created_by   uuid not null references profiles(id) default auth.uid(),
  created_at   timestamptz not null default now(),
  updated_at   timestamptz not null default now()
);
create index projects_name_trgm on projects using gin (name gin_trgm_ops);

create table project_members (
  project_id  uuid references projects(id) on delete cascade,
  user_id     uuid references profiles(id) on delete cascade,
  added_at    timestamptz not null default now(),
  primary key (project_id, user_id)
);
create index on project_members (user_id);
```
Trigger `add_project_owners` (after insert, and after update of `lead_id`; security definer): adds `created_by` and `lead_id` to `project_members`. Trigger `notify_project_member_added` (after insert on `project_members`): `project.added` to the new member (not the actor).

```sql
create table tasks (
  id           uuid primary key default gen_random_uuid(),
  project_id   uuid not null references projects(id) on delete cascade,
  title        text not null check (char_length(title) between 1 and 200),
  description  text check (char_length(description) <= 20000),
  status       task_status not null default 'todo',
  priority     task_priority not null default 'medium',
  assignee_id  uuid references profiles(id) on delete set null,
  due_on       date,
  position     double precision not null default 0,   -- ordering within (project_id, status)
  created_by   uuid not null references profiles(id) default auth.uid(),
  completed_at timestamptz,
  created_at   timestamptz not null default now(),
  updated_at   timestamptz not null default now()
);
create index on tasks (project_id, status, position);
create index on tasks (assignee_id, status);
create index on tasks (due_on) where status <> 'done';
create index tasks_title_trgm on tasks using gin (title gin_trgm_ops);
```

**Triggers on `tasks`**
| Trigger | Does |
|---|---|
| `guard_task_write` (before insert/update) | Assignee must be a project member. Insert: if not `can_manage_tasks(project_id)`, `assignee_id` must be null or `auth.uid()` (employees self-assign only). Update: `project_id` never changes; if not `can_manage_tasks`, caller must be the assignee and may change only `status, position, description`. |
| `set_completed_at` (before update) | `completed_at = now()` when status becomes `done`; null when it leaves `done`. |
| `notify_task_events` (after insert/update) | New/changed `assignee_id` (not the actor) → `task.assigned`. |

**Kanban ordering.** New card = last position in the column + 1024. Moving between neighbours A and B = `(A + B) / 2`. If `|A − B| < 1e-6`, `moveTask` calls `rebalance_task_column(project_id, status)` and retries. This prevents float-precision collisions after many moves.

```sql
create table task_comments (
  id         uuid primary key default gen_random_uuid(),
  task_id    uuid not null references tasks(id) on delete cascade,
  author_id  uuid not null references profiles(id) default auth.uid(),
  body       text not null check (char_length(body) between 1 and 5000),
  mentions   uuid[] not null default '{}',
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);
create index on task_comments (task_id, created_at);

create table task_attachments (
  id          uuid primary key default gen_random_uuid(),
  task_id     uuid not null references tasks(id) on delete cascade,
  file_path   text not null,                    -- <project_id>/<task_id>/<uuid>-<filename> in task-files
  file_name   text not null,
  file_size   int not null check (file_size between 1 and 10485760),
  uploaded_by uuid not null references profiles(id) default auth.uid(),
  created_at  timestamptz not null default now()
);
create index on task_attachments (task_id);
```
Trigger `notify_mentions` (after insert on `task_comments`): `task.mention` to each id in `mentions` **that is a member of the task's project**, excluding the author; others are dropped silently.

```sql
create view project_progress with (security_invoker = true) as
select p.id as project_id,
       count(t.id)                                   as total_tasks,
       count(t.id) filter (where t.status = 'done')  as done_tasks,
       coalesce(round(100.0 * count(t.id) filter (where t.status = 'done')
                / nullif(count(t.id), 0)), 0)::int   as percent
from projects p left join tasks t on t.project_id = p.id
group by p.id;
```
- `security_invoker = true` is required; without it the view runs as its owner and **bypasses RLS**.
- `100.0 *` avoids integer division (plain `count / count` returns 0 for anything below 100%).

## 6. Announcements
```sql
create table announcements (
  id               uuid primary key default gen_random_uuid(),
  title            text not null check (char_length(title) between 1 and 200),
  body_html        text not null,           -- sanitised on server before insert
  audience         audience_type not null default 'all',
  audience_role    app_role,                -- when audience = 'role'
  audience_project uuid references projects(id) on delete cascade, -- when 'project'
  pinned           boolean not null default false,
  author_id        uuid not null references profiles(id) default auth.uid(),
  created_at       timestamptz not null default now(),
  updated_at       timestamptz not null default now(),
  check ((audience = 'role')    = (audience_role is not null)),
  check ((audience = 'project') = (audience_project is not null))
);
create index on announcements (pinned desc, created_at desc);

create table announcement_reads (
  announcement_id uuid references announcements(id) on delete cascade,
  user_id         uuid references profiles(id) on delete cascade default auth.uid(),
  read_at         timestamptz not null default now(),
  primary key (announcement_id, user_id)
);
```
Function `is_announcement_audience(a_id uuid)` (security definer): `audience='all'`, or `audience_role = auth_role()`, or `is_project_member(audience_project)`.
Trigger `notify_announcement` (after insert): `announcement.new` to every active user in the audience except the author.

## 7. Leads
```sql
create table leads (
  id          uuid primary key default gen_random_uuid(),
  name        text not null check (char_length(name) between 1 and 120),
  email       text not null check (char_length(email) <= 254),
  company     text,
  budget      text,                       -- range label from CMS options
  service     text,                       -- service slug
  message     text not null check (char_length(message) between 1 and 5000),
  status      lead_status not null default 'new',
  owner_id    uuid references profiles(id) on delete set null,
  lost_reason text,
  source      text not null default 'website',
  ip_hash     text,                       -- sha256(ip + IP_HASH_SALT), rate limit only
  created_at  timestamptz not null default now(),
  updated_at  timestamptz not null default now(),
  check (status <> 'lost' or lost_reason is not null)
);
create index on leads (status, created_at desc);

create table lead_notes (
  id         uuid primary key default gen_random_uuid(),
  lead_id    uuid not null references leads(id) on delete cascade,
  author_id  uuid not null references profiles(id) default auth.uid(),
  body       text not null check (char_length(body) between 1 and 5000),
  created_at timestamptz not null default now()
);
create index on lead_notes (lead_id, created_at);
```
Leads are inserted only by `/api/leads` with the service-role client (no anon insert policy).
Triggers: `notify_new_lead` (after insert → `lead.new` to active super_admin/admin), `audit_lead_change` (after update of `status` or `owner_id` → `lead.status_changed` / `lead.assigned`).

## 8. Notifications & audit
```sql
create table notifications (
  id         uuid primary key default gen_random_uuid(),
  user_id    uuid not null references profiles(id) on delete cascade,
  type       text not null,
  title      text not null,
  link       text check (link is null or link like '/portal/%'),
  payload    jsonb not null default '{}',
  read_at    timestamptz,
  created_at timestamptz not null default now()
);
create index on notifications (user_id, created_at desc);
create index on notifications (user_id) where read_at is null;

create table audit_log (
  id         bigint generated always as identity primary key,
  actor_id   uuid references profiles(id) on delete set null,
  action     text not null,
  entity     text not null,
  entity_id  text,
  diff       jsonb,
  created_at timestamptz not null default now()
);
create index on audit_log (created_at desc);
create index on audit_log (entity, entity_id);
```
Notification types: `task.assigned, task.mention, task.due, project.added, doc.submitted, doc.reviewed, doc.requested, doc.expiring, announcement.new, lead.new`.
Audit actions: `user.invited, user.role_changed, user.disabled, user.enabled, user.documents_purged, doc.approved, doc.rejected, cms.published, cms.unpublished, lead.status_changed, lead.assigned`.

- `notify(user_ids uuid[], type text, title text, link text, payload jsonb)` — security definer, `execute` granted to **`service_role` only**. Users cannot create notifications for others. Nearly all notifications are created by the DB triggers above, so an action cannot forget one.
- `log_audit(actor uuid, action text, entity text, entity_id text, diff jsonb)` — security definer, `execute` granted to **`service_role` only**. Sensitive changes are audited by triggers (actor = `auth.uid()`); the service role uses it for events outside the DB (`user.invited`, `user.documents_purged`).
- `audit_log` has no update/delete policies; only the `cleanup` cron job (runs as `postgres`) deletes rows older than 1 year.

## 9. CMS (public site content)
```sql
create table site_settings (
  id          int primary key default 1 check (id = 1),   -- single row, seeded
  brand       jsonb not null default '{}',  -- name, tagline, logo_url, favicon_url
  contact     jsonb not null default '{}',  -- email, phone, whatsapp, address, booking_url
  socials     jsonb not null default '{}',  -- linkedin, x, github, dribbble, instagram
  seo         jsonb not null default '{}',  -- default title, description, og_image
  hero        jsonb not null default '{}',  -- headline, subheadline, cta labels/links
  sections    jsonb not null default '[]',  -- [{key:'services', visible:true}, ...] order = array order
  mascot      jsonb not null default '{}',  -- enabled, name, lines per section key, quick-nav items
  form        jsonb not null default '{}',  -- budget options, success message
  legal       jsonb not null default '{}',  -- privacy_html, terms_html
  draft       jsonb,                        -- unpublished edits, same keys; null = nothing pending
  updated_by  uuid references profiles(id),
  updated_at  timestamptz not null default now()
);
```
- Settings use draft/publish like collections: editors write to `draft`; **Publish** copies `draft` into the live columns and sets `draft = null`.
- Anon/authenticated get a column-level grant on everything **except** `draft`; admins read `draft` through a security-definer `get_site_settings_draft()`.
- Every JSON column is validated by a zod schema in `lib/validation/cms.ts` before write.

Collection tables share: `id uuid pk, sort_order int not null default 0, is_published boolean not null default false, created_at, updated_at`.

| Table | Extra columns |
|---|---|
| `services` | `slug text unique not null, title, summary, icon, body_html, features text[], seo jsonb` |
| `case_studies` | `slug text unique not null, title, client_name, industry, summary, problem_html, solution_html, results jsonb ([{label,value}]), stack text[], cover_url, gallery text[], service_slugs text[], featured bool, seo jsonb` |
| `clients` | `name, logo_url, website_url, industry` |
| `testimonials` | `quote, author_name, author_role, company, photo_url, case_study_id uuid references case_studies on delete set null` |
| `stats` | `label, value, suffix` |
| `process_steps` | `title, description, icon` |
| `engagement_models` | `name, description, price_label, features text[], cta_label` |
| `faqs` | `question, answer_html` |

Slugs: `check (slug ~ '^[a-z0-9]+(-[a-z0-9]+)*$')`. Trigger `audit_cms_publish` on every collection (after update of `is_published`) and on `site_settings` publish writes `cms.published` / `cms.unpublished`.

## 10. Rate limiting
```sql
create table rate_limits (
  key          text not null,          -- 'lead:<ip_hash>' or 'lead:global'
  window_start timestamptz not null,
  count        int not null default 1,
  primary key (key, window_start)
);
create index on rate_limits (window_start);

create function hit_rate_limit(p_key text, p_max int, p_window interval) returns boolean
language plpgsql security definer set search_path = public as $$
declare
  secs double precision := extract(epoch from p_window);
  w    timestamptz := to_timestamp(floor(extract(epoch from now()) / secs) * secs);
  c    int;
begin
  insert into rate_limits(key, window_start) values (p_key, w)
  on conflict (key, window_start) do update set count = rate_limits.count + 1
  returning count into c;
  return c <= p_max;            -- true = allowed
end $$;
revoke execute on function hit_rate_limit(text, int, interval) from public, anon, authenticated;
grant  execute on function hit_rate_limit(text, int, interval) to service_role;
```
Atomic (one upsert), fixed window. The lead route calls it twice: `('lead:'||ip_hash, 3, '10 minutes')` and a global `('lead:global', 50, '1 hour')`, so a distributed flood cannot exhaust the Resend quota.

## 11. RLS policies (summary)
"Admins" = `has_role('super_admin','admin')`. "Reviewers" = `has_role('super_admin','admin','hr')`. Every helper already requires `is_active`.

| Table | select | insert | update | delete |
|---|---|---|---|---|
| profiles | own row always (needed for the disabled check); any active user sees active profiles; reviewers see all | trigger only | own row (guard limits columns); reviewers where `can_manage_role(auth_role(), role)` (guard limits columns); `role`/`is_active` only via RPCs | none (disable instead) |
| document_requests | target user or reviewers | reviewers | reviewers | reviewers |
| documents | owner or reviewers | owner (`owner_id = auth.uid()`) | owner while `pending`/`rejected`; reviewers (guard limits columns) | owner while `pending`; reviewers |
| projects | admins, member, `created_by = auth.uid()`, or `lead_id = auth.uid()` | `has_role('super_admin','admin','project_manager')` and `created_by = auth.uid()` | admins or project lead | admins |
| project_members | admins or `is_project_member(project_id)` | admins or `is_project_lead(project_id)` | – | admins or project lead (lead row cannot be removed) |
| tasks | admins or `is_project_member(project_id)` | member and `auth_role() <> 'intern'` (guard limits assignee) | `can_manage_tasks(project_id)` or `assignee_id = auth.uid()` (guard limits columns) | `can_manage_tasks(project_id)` |
| task_comments | same as parent task | member of task's project and `author_id = auth.uid()` | author (body only) | author or admins |
| task_attachments | same as parent task | member and `uploaded_by = auth.uid()` | – | uploader or `can_manage_tasks` |
| announcements | `is_announcement_audience(id)`, author, or admins | reviewers (any audience); PM only with `audience='project'` and `is_project_lead(audience_project)` | author | author or admins |
| announcement_reads | own rows; author of the announcement; admins | own, and in audience | – | – |
| leads | `has_role('super_admin','admin','project_manager')` | service role only | admins | super_admin |
| lead_notes | same as leads | admins | – | author or super_admin |
| notifications | own | triggers / `notify()` only | own, `read_at` only (column grant) | own |
| audit_log | admins | triggers / `log_audit()` only | none | none |
| site_settings | anon + authenticated (all columns except `draft`) | none (row seeded) | admins | none |
| CMS collections | anon + authenticated: `is_published`; admins: all | admins | admins | admins |
| rate_limits | none | service role | service role | service role |

Policy notes:
- Helpers are `security definer`, so `project_members` policies that call `is_project_member` do not recurse.
- `projects` select includes `created_by`/`lead_id` so `insert … returning` works before the membership trigger's row is visible.
- `notifications`: `revoke update on notifications from authenticated; grant update (read_at) on notifications to authenticated;`.

## 12. Storage
Bucket-level limits (`file_size_limit`, `allowed_mime_types`) are set in bucket config, so they are enforced even though uploads go straight from the browser.

| Bucket | Public | Size limit | Allowed MIME |
|---|---|---|---|
| `employee-docs` | No | 10 MB | pdf, jpeg, png, docx |
| `task-files` | No | 10 MB | pdf, jpeg, png, webp, gif, docx, xlsx, pptx, zip, txt, csv |
| `avatars` | Yes | 1 MB | jpeg, png, webp |
| `site-media` | Yes | 20 MB | jpeg, png, webp, avif, mp4 (SVG only via server action — see TRD §5) |

| Bucket | Read | Write | Delete |
|---|---|---|---|
| `employee-docs` | first folder = `auth.uid()::text`, or reviewers | first folder = own uid | owner for objects not referenced by an approved document; reviewers. Replaced files are removed by the server action (service role). |
| `task-files` | `is_project_member(first folder::uuid)` or admins | project member (interns included) | uploader (`owner_id = auth.uid()`) or admins |
| `avatars` | public | first folder = own uid | first folder = own uid |
| `site-media` | public | admins | admins |

## 13. Scheduled jobs (pg_cron, 03:00 UTC daily unless noted)
| Job | Action |
|---|---|
| `remind_expiring_docs` | `doc.expiring` to owner + HR for docs expiring in 14 and 3 days; calls Edge Function `send-reminders` for the email |
| `remind_due_tasks` | `task.due` to assignees for tasks due tomorrow or overdue (not done) |
| `remind_doc_requests` | Notify users with unfulfilled requests past due |
| `cleanup` | Delete notifications > 90 days, audit_log > 1 year, rate_limits > 1 day |
| `cleanup-orphans` (Sundays) | Calls Edge Function `cleanup-orphans`: removes objects in `employee-docs`/`task-files` with no DB row and older than 24 h (failed uploads). Uses the Storage API — never `delete from storage.objects`. |

Edge Functions are called with `pg_net`; their URL and shared secret live in **Supabase Vault**, not in cron SQL.
pg_cron runs inside the database and does **not** count as project activity for the free-tier pause — see the external keep-alive in [DEPLOYMENT](DEPLOYMENT.md).

## 14. Realtime
Publication `supabase_realtime` includes only `notifications`. The client subscribes with the server-side filter `user_id=eq.<uid>`; Realtime also enforces RLS.

## 15. Search
`search_portal(q text, lim int default 20)` — security **invoker** (RLS applies). Trigram similarity over `profiles.full_name`, `projects.name`, `tasks.title`; returns `{kind, id, label, link}`. Minimum query length 2.
