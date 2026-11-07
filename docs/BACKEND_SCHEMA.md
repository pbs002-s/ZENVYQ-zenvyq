# Backend Schema (Supabase Postgres)

Status: designed, not yet migrated. When migrations exist, this doc must match `supabase/migrations/`.
Related: [Roles & permissions](ROLES_PERMISSIONS.md) · [TRD](TRD.md)

Conventions: `uuid` primary keys (`gen_random_uuid()`), `created_at timestamptz default now()`, `updated_at` maintained by a shared trigger, snake_case, RLS enabled on **every** table.

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
  id            uuid primary key references auth.users(id) on delete cascade,
  email         text not null unique,
  full_name     text not null,
  role          app_role not null default 'employee',
  designation   text,
  department    text,
  phone         text,
  avatar_url    text,
  bio           text,
  skills        text[] default '{}',
  joined_on     date,
  show_on_site  boolean not null default false,   -- appears on /about team
  is_active     boolean not null default true,
  created_at    timestamptz not null default now(),
  updated_at    timestamptz not null default now()
);
```
Trigger `on_auth_user_created` (after insert on `auth.users`) inserts the profile using `raw_app_meta_data->>'role'` and `raw_user_meta_data->>'full_name'`.
Trigger `protect_last_super_admin` (before update/delete on `profiles`) raises if the change would leave zero active super admins.
Trigger `prevent_self_role_change` raises if `new.role <> old.role and old.id = auth.uid()`.

## 3. Helper functions (security definer, `set search_path = public`)
```sql
create function auth_role() returns app_role
  language sql stable security definer as $$
  select role from profiles where id = auth.uid() and is_active $$;

create function has_role(variadic roles app_role[]) returns boolean
  language sql stable security definer as $$
  select coalesce(auth_role() = any(roles), false) $$;

create function is_project_member(p uuid) returns boolean
  language sql stable security definer as $$
  select exists(select 1 from project_members m join profiles pr on pr.id = m.user_id
                where m.project_id = p and m.user_id = auth.uid() and pr.is_active) $$;
```

## 4. Documents
```sql
create table documents (
  id           uuid primary key default gen_random_uuid(),
  owner_id     uuid not null references profiles(id) on delete cascade,
  category     doc_category not null,
  title        text not null check (char_length(title) between 1 and 120),
  file_path    text not null,                -- employee-docs/<owner_id>/<id>/<file>
  file_size    int  not null check (file_size <= 10485760),
  mime_type    text not null,
  status       doc_status not null default 'pending',
  review_note  text,                          -- required when rejected (check below)
  reviewed_by  uuid references profiles(id),
  reviewed_at  timestamptz,
  expires_on   date,
  request_id   uuid references document_requests(id),
  created_at   timestamptz not null default now(),
  updated_at   timestamptz not null default now(),
  check (status <> 'rejected' or review_note is not null)
);

create table document_requests (
  id            uuid primary key default gen_random_uuid(),
  user_id       uuid not null references profiles(id) on delete cascade,
  category      doc_category not null,
  note          text,
  due_on        date,
  requested_by  uuid not null references profiles(id),
  fulfilled_at  timestamptz,
  created_at    timestamptz not null default now()
);
```
(Create `document_requests` before `documents` in the migration.)

## 5. Projects & tasks
```sql
create table projects (
  id           uuid primary key default gen_random_uuid(),
  name         text not null,
  client_name  text,
  description  text,
  status       project_status not null default 'planning',
  start_on     date,
  end_on       date,
  lead_id      uuid references profiles(id),       -- PM in charge
  created_by   uuid not null references profiles(id),
  created_at   timestamptz not null default now(),
  updated_at   timestamptz not null default now()
);

create table project_members (
  project_id  uuid references projects(id) on delete cascade,
  user_id     uuid references profiles(id) on delete cascade,
  added_at    timestamptz not null default now(),
  primary key (project_id, user_id)
);

create table tasks (
  id           uuid primary key default gen_random_uuid(),
  project_id   uuid not null references projects(id) on delete cascade,
  title        text not null check (char_length(title) between 1 and 200),
  description  text,
  status       task_status not null default 'todo',
  priority     task_priority not null default 'medium',
  assignee_id  uuid references profiles(id) on delete set null,
  due_on       date,
  position     double precision not null default 0,   -- ordering within a column
  created_by   uuid not null references profiles(id),
  completed_at timestamptz,
  created_at   timestamptz not null default now(),
  updated_at   timestamptz not null default now()
);
create index on tasks (project_id, status, position);
create index on tasks (assignee_id, status);

create table task_comments (
  id         uuid primary key default gen_random_uuid(),
  task_id    uuid not null references tasks(id) on delete cascade,
  author_id  uuid not null references profiles(id),
  body       text not null check (char_length(body) <= 5000),
  mentions   uuid[] default '{}',
  created_at timestamptz not null default now()
);

create table task_attachments (
  id          uuid primary key default gen_random_uuid(),
  task_id     uuid not null references tasks(id) on delete cascade,
  file_path   text not null,
  file_name   text not null,
  file_size   int not null check (file_size <= 10485760),
  uploaded_by uuid not null references profiles(id),
  created_at  timestamptz not null default now()
);
```
View `project_progress` = `count(*) filter (where status='done') / nullif(count(*),0)` per project.

## 6. Announcements
```sql
create table announcements (
  id              uuid primary key default gen_random_uuid(),
  title           text not null,
  body_html       text not null,           -- sanitised on server before insert
  audience        audience_type not null default 'all',
  audience_role   app_role,                -- when audience = 'role'
  audience_project uuid references projects(id) on delete cascade, -- when 'project'
  pinned          boolean not null default false,
  author_id       uuid not null references profiles(id),
  created_at      timestamptz not null default now(),
  check ((audience='role') = (audience_role is not null)),
  check ((audience='project') = (audience_project is not null))
);

create table announcement_reads (
  announcement_id uuid references announcements(id) on delete cascade,
  user_id         uuid references profiles(id) on delete cascade,
  read_at         timestamptz not null default now(),
  primary key (announcement_id, user_id)
);
```

## 7. Leads
```sql
create table leads (
  id          uuid primary key default gen_random_uuid(),
  name        text not null,
  email       text not null,
  company     text,
  budget      text,                       -- range label from CMS options
  service     text,                       -- service slug
  message     text not null check (char_length(message) <= 5000),
  status      lead_status not null default 'new',
  owner_id    uuid references profiles(id),
  lost_reason text,
  source      text default 'website',
  ip_hash     text,                       -- sha256(ip + salt), for rate limit only
  created_at  timestamptz not null default now(),
  updated_at  timestamptz not null default now()
);

create table lead_notes (
  id         uuid primary key default gen_random_uuid(),
  lead_id    uuid not null references leads(id) on delete cascade,
  author_id  uuid not null references profiles(id),
  body       text not null,
  created_at timestamptz not null default now()
);
```
Leads are inserted only by the `/api/leads` route using the service-role client (anon has no insert policy).

## 8. Notifications & audit
```sql
create table notifications (
  id         uuid primary key default gen_random_uuid(),
  user_id    uuid not null references profiles(id) on delete cascade,
  type       text not null,     -- task.assigned, task.mention, doc.reviewed, doc.requested, doc.expiring, announcement.new, lead.new, task.due
  title      text not null,
  link       text,              -- in-portal path
  payload    jsonb default '{}',
  read_at    timestamptz,
  created_at timestamptz not null default now()
);
create index on notifications (user_id, read_at, created_at desc);

create table audit_log (
  id         bigint generated always as identity primary key,
  actor_id   uuid references profiles(id),
  action     text not null,     -- user.invited, user.role_changed, doc.approved, cms.published, ...
  entity     text not null,
  entity_id  text,
  diff       jsonb,
  created_at timestamptz not null default now()
);
```
`audit_log` is insert-only: no update/delete policies; inserts happen through a `log_audit()` security-definer function.

## 9. CMS (public site content)
```sql
create table site_settings (
  id          int primary key default 1 check (id = 1),   -- single row
  brand       jsonb not null default '{}',  -- name, tagline, logo_url, favicon_url
  contact     jsonb not null default '{}',  -- email, phone, whatsapp, address, calendly_url
  socials     jsonb not null default '{}',  -- linkedin, x, github, dribbble, instagram
  seo         jsonb not null default '{}',  -- default title, description, og_image
  hero        jsonb not null default '{}',  -- headline, subheadline, cta labels/links
  sections    jsonb not null default '[]',  -- [{key:'services', visible:true}, ...] order = array order
  mascot      jsonb not null default '{}',  -- enabled, lines per section key, quick-nav items
  form        jsonb not null default '{}',  -- budget options, success message
  legal       jsonb not null default '{}',  -- privacy_html, terms_html
  updated_by  uuid references profiles(id),
  updated_at  timestamptz not null default now()
);
```
Collection tables share these columns: `id, sort_order int, is_published boolean default false, created_at, updated_at`.

| Table | Extra columns |
|---|---|
| `services` | `slug unique, title, summary, icon, body_html, features text[], seo jsonb` |
| `case_studies` | `slug unique, title, client_name, industry, summary, problem_html, solution_html, results jsonb ([{label,value}]), stack text[], cover_url, gallery text[], service_slugs text[], featured bool, seo jsonb` |
| `clients` | `name, logo_url, website_url, industry` |
| `testimonials` | `quote, author_name, author_role, company, photo_url, case_study_id` |
| `stats` | `label, value, suffix` |
| `process_steps` | `title, description, icon` |
| `engagement_models` | `name, description, price_label, features text[], cta_label` |
| `faqs` | `question, answer_html` |

## 10. Rate limiting
```sql
create table rate_limits (
  key        text not null,          -- 'lead:<ip_hash>'
  window_start timestamptz not null,
  count      int not null default 1,
  primary key (key, window_start)
);
```
Function `hit_rate_limit(key text, max int, window interval) returns boolean` (service role only). Limit: 3 leads per IP per 10 minutes.

## 11. RLS policies (summary)
| Table | select | insert | update | delete |
|---|---|---|---|---|
| profiles | any active user (directory) | trigger only | self (non-role fields via column grants) or `has_role(super_admin,admin,hr)` with rank rule | none (disable instead) |
| documents | owner or `has_role(super_admin,admin,hr)` | owner, `owner_id = auth.uid()` | owner while pending (replace file); reviewers (status/note) | owner while pending; reviewers |
| document_requests | target user or reviewers | reviewers | reviewers | reviewers |
| projects | `has_role(super_admin,admin)` or member | `has_role(super_admin,admin,project_manager)` | admins or `lead_id = auth.uid()` | admins |
| project_members | admins or member of same project | admins or project lead | – | admins or project lead |
| tasks | admins or project member | project member and role ≠ intern | admins, project lead, PM members, or assignee | admins or project lead |
| task_comments | same as tasks | project member | author | author or admins |
| task_attachments | same as tasks | project member | – | uploader or admins |
| announcements | audience matches (all / my role / my project) or author or admins | admins/hr (any), PM (own project only) | author | author or admins |
| announcement_reads | own rows; author sees rows for own announcements | own | – | – |
| leads, lead_notes | `has_role(super_admin,admin,project_manager)` | service role only (leads); admins (notes) | admins | super_admin |
| notifications | own | security-definer `notify()` | own (`read_at` only) | own |
| audit_log | `has_role(super_admin,admin)` | `log_audit()` only | none | none |
| site_settings, CMS tables | anon + authenticated: `is_published` rows (site_settings always) ; admins: all | admins | admins | admins |
| rate_limits | none | service role | service role | service role |

## 12. Storage policies
| Bucket | Read | Write | Delete |
|---|---|---|---|
| `employee-docs` (private) | `(storage.foldername(name))[1] = auth.uid()::text` or reviewers | first folder = own uid | owner while pending, reviewers |
| `task-files` (private) | `is_project_member(first folder)` or admins | project member (interns included) | uploader or admins |
| `site-media` (public) | public | `has_role(super_admin,admin)` | `has_role(super_admin,admin)` |

## 13. Scheduled jobs (pg_cron, daily 03:00 UTC)
| Job | Action |
|---|---|
| `remind_expiring_docs` | Notify owner + HR for docs expiring in 14 and 3 days |
| `remind_due_tasks` | Notify assignees for tasks due tomorrow or overdue (not done) |
| `remind_doc_requests` | Notify users with unfulfilled requests past due |
| `cleanup` | Delete read notifications > 90 days, audit_log > 1 year, rate_limits > 1 day |

Emails for reminders are sent by a Supabase Edge Function called from pg_cron via `pg_net` (only for expiring docs; others are in-app).

## 14. Realtime
Publication `supabase_realtime` includes only `notifications` (filtered client-side by `user_id=eq.<uid>`; RLS also applies).
