# Open Questions

Items waiting on the product owner. When answered, move the answer into the doc it affects, note it in CHANGELOG, and delete the item here. Items marked **Default** already have a working answer in the docs that will be used unless you say otherwise.

## Blocking phase 1–2
1. **Brand name & domain** — Is the brand "ZENVYQ"? What domain will the site use? (Needed for Resend, Turnstile, Supabase redirect URLs.)
2. **Design** — When will the design be uploaded, and in what format (Figma link, images, PDF)?
3. **Logo & colours** — Do you have a logo, brand colours, and fonts already?

## Content (placeholders meanwhile)
4. Final list of services you sell (current draft: web apps, mobile apps, SaaS/MVP, UI/UX, automation, maintenance).
5. Real portfolio projects / case studies and client logos you are allowed to show.
6. Show prices on the engagement models section, or "Contact for quote" only?
7. Contact details: email, phone, WhatsApp, address, booking link (Calendly or similar)?
8. Target markets/countries (affects copy, currency, legal pages, and whether a second language is needed later).
9. Who writes the privacy policy and terms (template vs lawyer)? Needed before launch (Phase 9).

## Mascot
10. Mascot name (working name "Zen")? Keep the panda-bot look or base it on the logo once it exists?

## Portal
11. Departments list (e.g. Engineering, Design, Marketing, HR, Operations)?
12. Required document types for every employee (so HR can track "complete" profiles — PRD goal G3)?
13. **Default: PMs see all leads, read-only.** Should they see only leads assigned to them instead?
14. Who receives new-lead emails (addresses for `LEAD_NOTIFY_EMAILS`)?
15. **Default: assignees may move their own tasks to Done** (D-010). Do you want a hard review gate instead?
16. Any need for attendance/leave soon? (Currently post-v1.)

## Ops
17. **Default: Netlify free hosting** (D-008). OK, or is $20/month for Vercel Pro acceptable?
18. Who keeps the backup decryption key (`age` private key)? At least one founder, stored in a password manager.
