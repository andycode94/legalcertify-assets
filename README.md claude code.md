# LegalCertify

Case management and CRM for LegalCertify — certified translation and
immigration document support, Fort Mill SC and Charlotte NC.

Staff-only web application. Clients do not log in; they receive
single-use secure links.

## Contents

    CLAUDE.md              context for Claude Code — read this first
    supabase/migrations/   database, five files, already applied
    supabase/functions/    scheduled jobs
    src/                   the application
    docs/SETUP.md          setup walkthrough
    docs/HANDOFF.md        what to paste into Claude Code
    docs/BLUEPRINT.md      architecture notes
    docs/dashboard-prototype.html   visual direction

## Stack

Next.js on Vercel · Supabase (Postgres, Auth, Storage, Edge Functions)

The browser does not query the database directly. Reads and writes go
through server functions holding the service role key, so the public key
on its own does not reach any data.

## How configuration works

Wording and timing live in the database rather than in code, so they can
be changed from inside the application without a developer:

| Table | Holds |
|---|---|
| `settings` | Retention windows, reminder timing, feed options, link lifetimes |
| `message_templates` | Every client email, per language, with edit history |
| `ui_strings` | On-screen labels, disclaimers, certificate wording |
| `intel_rules` | Keywords the regulatory watcher grades on |

Each carries its shipped default alongside the current value, so any
change can be reverted from the interface.

## How permissions work

A person holds any number of roles; roles grant capabilities; the
application checks capabilities. Roles and their capabilities are
editable, and new roles can be created.

Olha is a superuser: `profiles.is_superuser`. That holds every
capability in the system including any added in future migrations,
reaches every case regardless of assignment, and overrides every
validation trigger in the schema. Nothing in the code sits above it.

## Build order

- [x] Phase 0 — repo, database, migrations
- [ ] Phase 1 — auth, roles, contacts, matters, documents, upload links
- [ ] Phase 2 — task templates, the clock, localised notifications
- [ ] Phase 3 — packet assembly, print path
- [ ] Phase 4 — filing records, USCIS case status polling
- [ ] Phase 5 — FOIA API
- [ ] Phase 6 — invoicing
- [ ] Phase 7 — retention state machine
