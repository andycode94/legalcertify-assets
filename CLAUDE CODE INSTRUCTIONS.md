# CLAUDE.md

Context for Claude Code working in this repository.

---

## The product

Case management and CRM for **LegalCertify** — certified translation and
immigration document support serving Ukrainian and Russian speaking
clients in Fort Mill SC and Charlotte NC.

Owner: Olha Mykhno. Project owner: Anthony.

Staff-only web application, 3–4 seats. **Clients never log in.** They
receive single-use tokenised links to upload documents. Olha handles
sensitive conversations in person.

Marketing site is separate: legalcertify.org, built in Webflow. Not in
this repo.

---

## Working agreement

Anthony directs; implement to his instruction. Two things he has stated
explicitly, both of which have already caused a correction:

1. **Do not hardcode anything the owner might want to change.** Wording,
   timing, thresholds, keyword lists, disclaimers all live in database
   tables and are editable in the app. Ship defaults, never locks.
2. **Do not write in a directive register.** No "ground rules", no
   telling them what is or isn't correct in their own product. Describe
   what exists and what the options are.

Olha is a **superuser** — every capability, every case, override on
every validation. Nothing in the code sits above her.

---

## Stack

- Next.js (App Router) on Vercel
- Supabase — Postgres 17, Auth, Storage, Edge Functions
- Stripe (payments), Dropbox Sign (e-signature), transactional mail TBD

**Supabase project:** `LegalCertify Thunderbolt`
**Project ref:** `cmrxfglekiyvvputzvky` · region `us-east-1`

The browser never queries Postgres directly. Every read and write goes
through a server function holding the service role key. RLS is enabled
and closed by default on all tables as defence in depth.

---

## Database — already deployed

All five migrations in `supabase/migrations/` have been applied to the
live project. Do not re-run them. Current state: 38 tables, 6 views, 20
seeded case types, 25 capabilities, 4 roles, 19 settings.

| Migration | Contents |
|---|---|
| `..._core.sql` | contacts, households, matters, tasks, templates, documents, credentials, notifications, audit |
| `..._regulatory_intel.sql` | intel_sources, intel_signals, intel_impacts, form_editions |
| `..._configuration.sql` | settings, message_templates, ui_strings, intel_rules |
| `..._authorization.sql` | capabilities, roles, profile_roles, olha superuser |
| `..._owner_authority.sql` | `profiles.is_superuser`, overridable checks |

### Concepts that are easy to get wrong

**contacts, not clients.** One person record spans CRM and case
management. `contacts.stage` moves inquiry → consult_booked →
consult_held → engaged → active_client → past_client. There is no
separate CRM.

**matters** are one case per contact per case type. `matters.route` is
`electronic` or `mail` and is changeable at any point — it only affects
the final filing step, so no upstream task branches on it.

**Task offsets are relative.** `template_tasks.offset_days` counts from
when the task opens, either at matter open or after a predecessor
completes. Never absolute dates.

**credentials is not documents.** `credentials` holds expiry facts only
(kind, expiry date, redacted reference) so the retention purge can
delete a passport scan while keeping "this passport expires 2029-04-11".
That split is what makes renewal reminders possible. Don't merge them.

**Permissions are capabilities, not roles.** Check
`has_capability('matters.edit')`, never a role name. A person holds many
roles. `is_superuser()` short-circuits everything — this is deliberate,
so Olha can hold owner and attorney simultaneously.

---

## Operating model — affects behaviour, not just documentation

An SC-licensed attorney is **attorney of record**. The client engages
the attorney, the attorney files the G-28, LegalCertify operates as the
attorney's support staff.

Consequences that show up in code:

- `g28s` are **per matter**, not per client. USCIS recognises one entry
  of appearance at a time, so supersession is modelled, not overwritten.
- Immigration practice before USCIS is **federal**. One SC attorney
  covers NC clients. Per-state divergence applies only to the notary and
  apostille lane (SC vs NC notarial certificate wording).
- **There is no USCIS filing API.** The only public endpoints are FOIA
  and Case Status. "Electronic filing" means a human uploads to the
  client's USCIS account. DS-260 is State Department via CEAC — a second
  destination entirely.
- Olha's approval gates every filing packet.

---

## Phase 0 — done

Repo created, migrations applied, catalogue seeded.

## Phase 1 — next

Auth, roles, contacts, matters, documents, upload links.

Suggested order:

1. Supabase Auth + `profiles` row creation on first sign-in
2. Set Olha's superuser flag — *blocked until her profile row exists*:
   `update profiles set is_superuser = true where email = '<hers>';`
3. Capability-aware layout shell and navigation
4. Contacts list, contact detail, stage transitions
5. Matter creation from a contact, task instantiation from template
6. Document upload via tokenised link, virus scan, hash on ingest
7. The dashboard — see `docs/dashboard-prototype.html`

### Visual direction — approved, follow it

`docs/dashboard-prototype.html` is the reference. Cool pale grey canvas
`#E8ECEE`, deep teal ink `#102F3B`, marigold `#E0A008`, sage `#4A8F6C`
for complete, dusty rose `#C25F6B` for late. Bricolage Grotesque
display, Inter body, IBM Plex Mono for data.

Principle: **calm surface, lively moments.** Energy belongs to progress
events — a stamp landing, a counter climbing, ink spreading when a task
clears. Everything else stays quiet. The brief was "not visually
difficult to look at for hours" *and* "fun and lively", and this is how
those reconcile.

The signature element is the passport-stamp progress track per matter.
It works up to about eight stages; a longer case type needs a different
treatment.

---

## Blocked on Anthony

- **Email platform** — Google Workspace Business Plus recommended,
  unless the attorney's firm runs Microsoft
- **Task templates** — Olha fills in
  `docs/LegalCertify-Case-Templates-Worksheet.xlsx`. Two worked examples
  are in it (I-765 and N-400). Load her answers into `task_templates`
  and `template_tasks`.

---

## Deferred to later phases

Packets, filings, certificates, translations, invoices, retention state
machine. Their tables are not created yet — design them when the phase
arrives rather than guessing now.

The regulatory watcher in
`supabase/functions/watch-federal-register/index.ts` is written but
never run against the live endpoint. Smoke-test before scheduling. Its
hardcoded keyword arrays should move to the `intel_rules` table, which
already exists and is seeded.

---

## Reference

- `docs/BLUEPRINT.md` — architecture and the reasoning behind it
- `docs/SETUP.md` — GitHub and Supabase setup
- `docs/dashboard-prototype.html` — visual direction, open in a browser
