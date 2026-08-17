# Governance — sage100 plugin

## What this plugin connects as

The plugin contains no credentials and opens no connections itself. All data
access flows through a Conduit on-prem tunnel's generic `mssql` connector,
which authenticates with whatever SQL principal the customer configured. The
skills assume — and instruct customers to configure — a dedicated read-only
login scoped to the Sage company database.

## How read-only actually holds

Three independent layers, in decreasing order of authority:

1. **The SQL principal.** A login limited to `db_datareader` (or narrower)
   cannot write regardless of what any client sends. This is the boundary
   that matters. **This plugin cannot verify the customer configured it** —
   that is the honest enforcement gap.
2. **The connector's statement filter.** Conduit's SQL connector accepts a
   single SELECT or WITH statement per call, strips comments, rejects
   multi-statement batches, and enforces a 1000-row cap and 15-second
   timeout. EXEC, DDL, and DML are refused before reaching the database.
3. **The Sage ODBC driver** (Standard/Advanced path) is read-only by design;
   Sage ships no writable driver.

## What the skills will and will not do

- Read-only reporting: lookups, aging, sales/spend analysis, trial balance,
  inventory positions.
- No writes, ever. Sage 100 writes require the Business Object Interface to
  preserve business rules and audit trails; raw SQL writes to ProvideX data
  are a documented corruption risk. The skills say so explicitly.
- Instructional DDL (creating the read-only login, the linked server) is
  addressed to the customer's administrator to run in their own tooling —
  the connector cannot execute it.

## Data sensitivity

Sage 100 databases contain financial records, customer and vendor PII, and
payroll tables (`PR_`). The recommended read-only login should be scoped to
the reporting tables the org actually needs; the skills never direct queries
at payroll or stored-payment data.

## Open items

- Conduit currently classifies unknown connector tools at admin tier when
  access-grant enforcement is enabled; until the generic SQL tools are
  classified read-tier in Conduit, org admins must grant the connector slug
  at the tier their enforcement mode requires.
- Skill packs from this marketplace are not yet gated on connector access in
  Conduit (no vendor-follow linkage for generic connector slugs); the pack
  is visible to org members whether or not they can call the connector.
