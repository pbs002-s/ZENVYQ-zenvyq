# Product Requirements Document (PRD)

**Product:** ZENVYQ website + employee portal
**Status:** Draft v1 · 2026-11-08
**Owner:** ZENVYQ founders

## 1. Problem
ZENVYQ is a new software agency that builds and sells custom software. It needs:
- A credible public presence that explains what it builds, who it serves, and proof of work, and turns visitors into inquiries.
- An internal tool where staff manage their documents, projects, tasks, and company notices without paying for several SaaS products.
- A way for non-developers (admin) to update website content without a code deploy.

## 2. Goals
| # | Goal | Measure |
|---|---|---|
| G1 | Convert visitors into project inquiries | Inquiry form submissions per month |
| G2 | Present the agency as professional and detail-oriented | Lighthouse ≥ 90 on all categories; bounce rate |
| G3 | Centralise staff documents and approvals | % of staff with complete approved documents |
| G4 | Track project delivery in one place | All active projects and tasks live in the portal |
| G5 | Zero hosting cost at launch | Monthly bill = $0 |

## 3. Non-goals (v1)
- Client login portal, invoicing, payroll, attendance and leave, time tracking, chat.
- Public signup of any kind.
- Multi-language site (can be added later).
- Any public mention of AI.

## 4. Users
| User | Description | Main needs |
|---|---|---|
| Visitor / prospective client | Founders, SMB owners, product managers | Understand services, see work, contact quickly |
| Super admin | Company owner | Full control, cannot be locked out |
| Admin | Operations lead | Manage users, CMS, leads, everything except super admin |
| HR | People team | Onboard staff, review documents, post notices |
| Project manager | Runs client projects | Create projects, assign tasks, track progress |
| Employee | Developer / designer | See tasks, update status, submit documents |
| Intern | Short-term staff | Same as employee but cannot create tasks |

## 5. Features — Public site

### 5.1 Navigation
- Open (transparent) sticky top bar that becomes solid on scroll.
- Items: Logo · Services · Work · Clients · Process · About · Contact · **Start a project** (CTA).
- Mobile: full-screen menu overlay. Active section highlighted while scrolling.

### 5.2 Home page sections
Every section's content, order, and visibility come from the CMS.
1. **Hero** — headline, sub-headline, primary CTA (Start a project), secondary CTA (See our work), mascot entrance animation.
2. **Services (what we sell)** — cards: Web apps, Mobile apps, SaaS / MVP builds, UI/UX design, Business automation, Maintenance & support. Each links to `/services/[slug]`.
3. **Stats strip** — projects delivered, clients, countries, years active.
4. **Selected work** — portfolio cards linking to `/work/[slug]` case studies.
5. **Clients / who we serve** — logo wall and industries served.
6. **Process** — Discover → Design → Build → Launch → Support.
7. **Testimonials** — quote, name, role, company, photo.
8. **Engagement models** — Fixed-scope project, Monthly retainer, Dedicated team (prices optional).
9. **FAQ** — accordion.
10. **Contact / inquiry form** — name, email, company, budget range, service of interest, message.
11. **Footer** — contact details, socials, legal links.

### 5.3 Other pages
- `/work` list + `/work/[slug]` case study (client, problem, solution, stack, results, gallery).
- `/services/[slug]` service detail (what's included, process, related work, CTA).
- `/about` (story, values, team from directory where `show_on_site = true`).
- `/contact` (form + details).
- `/privacy`, `/terms`.
- Custom 404.

### 5.4 3D mascot guide
- Original small chubby low-poly panda-like bot with glowing visor eyes in brand colour.
- Sits bottom-right; idles, eyes follow the cursor.
- Changes pose and shows a speech bubble per section while scrolling.
- Click opens a quick-nav menu: jump to sections, Start a project, Book a call.
- Lines and enable/disable are editable in the CMS.
- Static image fallback for reduced motion and low-power devices.

### 5.5 SEO
- Per-page title, description, OG image from CMS.
- `sitemap.xml`, `robots.txt`, JSON-LD Organization schema.

## 6. Features — Employee portal

### 6.1 Authentication
- Login with email + password at `/login`.
- Account created by super_admin / admin / hr → invite email → user sets password.
- Forgot password via email link.
- Disabled users cannot log in.

### 6.2 Role dashboards
| Role | Dashboard widgets |
|---|---|
| super_admin / admin | New leads, pending document reviews, active projects, recent activity |
| hr | Pending document reviews, expiring documents, recent hires |
| project_manager | My projects progress, overdue tasks, team workload |
| employee / intern | My tasks by status, due soon, latest announcements, my document status |

### 6.3 Users & team directory
- Create, edit, disable users; assign role, department, designation.
- Profile: photo, name, designation, department, phone, joined date, skills, bio, `show_on_site`.
- Directory searchable by name, department, skill.

### 6.4 Documents & files
- Categories: ID proof, Contract, Certificate, NDA, Report, Other.
- Upload PDF / JPG / PNG / DOCX up to 10 MB.
- Status: pending → approved / rejected (rejection needs a reason). Rejected documents can be re-uploaded.
- Optional expiry date; reminder 14 days before expiry.
- Employees see only their own documents; HR/admin see all.
- HR/admin can request a specific document from a user (creates a pending request with a due date).

### 6.5 Projects & tasks
- Project: name, client name, description, start/end dates, status (planning, active, on_hold, completed, cancelled), members.
- Task: title, description, status (todo, in_progress, review, done), priority (low, medium, high, urgent), assignee, due date, attachments, comments.
- Kanban board with drag and drop; list view with filters.
- Progress % = done tasks / total tasks.
- Comments support @mentions (creates a notification).

### 6.6 Announcements
- Title, rich text body, audience (all / role / project), pinned flag.
- Read receipts; authors see who has read.

### 6.7 Leads (CRM-lite)
- Created from the public inquiry form.
- Status: new → contacted → proposal_sent → won / lost.
- Notes, assigned owner.
- Email to admins on every new lead.

### 6.8 Site CMS
- Editors for: site settings (brand, contact, socials, SEO, mascot lines, section order/visibility), hero, services, case studies (with image gallery), clients, testimonials, stats, process steps, FAQs, engagement models, legal pages.
- Draft vs published per item; "Publish" refreshes the public site within seconds.
- Image upload to public media bucket.

### 6.9 Supporting features
- **Notifications** — in-app bell (realtime) for task assigned, mention, document reviewed, document requested, announcement, new lead, expiry reminder.
- **Audit log** — role changes, user disable, document approve/reject, CMS publish, lead status changes.
- **Global search** — users, projects, tasks (portal only).
- **My profile** — edit own details, avatar, change password.

## 7. Acceptance criteria (v1 release)
- A visitor can read every section on mobile and desktop and submit an inquiry that appears in the portal within 5 seconds.
- Admin changes hero text in the CMS, clicks Publish, and the live site shows it on next load.
- Admin invites a new employee; the employee sets a password, logs in, uploads a document; HR approves it; the employee sees "Approved".
- PM creates a project, adds a task for an employee; the employee gets a notification and moves the task to Done; progress updates.
- An intern cannot create tasks, see other users' documents, or open the CMS (UI hidden **and** blocked by RLS).
- Monthly hosting cost is $0.

## 8. Future (post-v1)
Attendance & leave, client portal, invoices, time tracking, multi-language site, blog, careers page with applications.
