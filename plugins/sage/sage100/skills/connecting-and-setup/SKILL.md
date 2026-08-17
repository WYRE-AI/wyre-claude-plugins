---
name: "Sage 100 Connecting and Setup"
description: >
  How to reach a Sage 100 database through Conduit's generic SQL connector:
  detecting which edition a site runs, configuring the connector for Sage
  100 Premium (native SQL Server), building the linked-server bridge for
  Standard/Advanced (ProvideX ODBC), the OPENQUERY dialect rules, and
  verifying the connection end to end.
when_to_use: >-
  When first connecting a Sage 100 site, when queries fail with driver or
  linked-server errors, when deciding between the Premium and ProvideX
  access paths, or when adding another Sage company to an existing setup.
  Use when: connect sage 100, sage odbc, providex, sotamas90, linked
  server, openquery, mas90 driver, sage 100 premium sql, or sage company
  database.
---

# Sage 100 Connecting and Setup

## Overview

Sage 100 has no proprietary query protocol — its data is reachable with
general-purpose SQL technology, which is why this plugin rides Conduit's
generic `mssql` connector instead of a Sage-specific server. The connector
exposes `mssql__query`, `mssql__list_tables`, and `mssql__describe_table`
(the prefix follows the connector slug; a named instance can surface as
`sage100__query`). Everything is read-only by design: single SELECT/WITH
statements, capped at 1000 rows per call.

There are two access paths, decided by the customer's edition:

1. **Premium** — the ERP runs natively on Microsoft SQL Server. The
   connector points straight at the company database. Full T-SQL.
2. **Standard / Advanced** — data lives in ProvideX files behind Sage's
   read-only ODBC driver. The connector points at a small SQL Server
   "bridge" that reaches Sage through a linked server; queries pass
   through OPENQUERY with a restricted SQL dialect.

## Key Concepts

**Edition detection.** Ask the site administrator, or infer: if there is a
SQL Server instance hosting databases named `MAS_<CompanyCode>` (like
`MAS_ABC`) plus `MAS_SYSTEM`, it is Premium. If the server share has a
`..\MAS90\` folder tree with `.M4T` files, it is Standard or Advanced.
Versions: Sage 100 2022 = 7.10, 2023 = 7.20, 2024 = 7.30, plus the 2025
release. Names like "Sage 100cloud" or "MAS 90/200" refer to the same
on-prem product line.

**One company = one database.** Premium: `MAS_ABC` per company. The
connector's `database` field decides which company you can see, and a
ProvideX connection is similarly fixed to one company by its DSN. For
multi-company sites, configure one named connector instance per company:
`{"sage100_abc": {"type": "mssql", "database": "MAS_ABC", ...}}` yields
`sage100_abc__query`.

**The security boundary is the SQL login.** The connector enforces
read-only statements, but the durable guarantee is a dedicated SQL
principal with `db_datareader` (or narrower) on the company database and
nothing else. On the ProvideX path the Sage ODBC driver is itself
read-only — every driver Sage ships refuses writes.

**SOTAMAS90 is reserved.** Sage creates a user DSN named `SOTAMAS90` for
its own reporting. Integrations must use their own System DSN — borrowing
SOTAMAS90 breaks as soon as a user opens Sage 100 on that machine.

## Common Workflows

**Configure the Premium path.** The site admin creates a read-only login
on the Sage SQL Server (run by the admin in their own tooling, not through
the connector):

```sql
CREATE LOGIN conduit_readonly WITH PASSWORD = '<strong password>';
```

```sql
USE MAS_ABC;
CREATE USER conduit_readonly FOR LOGIN conduit_readonly;
ALTER ROLE db_datareader ADD MEMBER conduit_readonly;
```

Then the Conduit connector config (pushed from the Conduit org tunnel
settings) is plain `mssql`:

```jsonc
{"connectors": {
  "sage100": {"type": "mssql", "host": "sage-sql.internal", "port": 1433,
               "database": "MAS_ABC", "user": "conduit_readonly",
               "password": "<from a secret store>", "encrypt": "true"}
}}
```

Windows integrated auth is available with `"auth": "integrated"` (no
user/password; the tunnel service account — typically a gMSA — must have
the datareader grant).

**Build the Standard/Advanced bridge.** On a Windows host that can reach
the Sage server share (a small SQL Server Express instance is enough):

1. Install Sage's **64-bit ODBC driver** (optional install on 2021–2024,
   default on 2025). 64-bit SQL Server cannot load the 32-bit driver.
2. Create a **System DSN** on the "MAS 90 4.0 ODBC Driver" with the
   three-part logon stored in the DSN: Company code (for example `ABC`),
   a dedicated Sage user, and its password. Stored credentials are what
   make the connection "silent" — there is no SILENT keyword, and without
   them the driver pops a logon dialog no service can answer.
3. Create a linked server over the **MSDASQL** provider against that DSN
   (Sage documents this pattern but does not support it — keep the DSN's
   Debug-tab provider string handy for driver-direct fallback).
4. Point the Conduit `mssql` connector at the bridge instance, with a
   read-only login on the bridge.

**Query through the bridge.** All Sage data flows through OPENQUERY — the
outer statement is a normal SELECT, so it passes the connector's read-only
filter:

```sql
SELECT * FROM OPENQUERY(SAGE100,
  'SELECT ARDivisionNo, CustomerNo, CustomerName FROM AR_Customer')
```

Rules for the inner ProvideX string:

- ODBC date-escape literals only: `{d''2026-01-01''}` (doubled quotes
  inside the OPENQUERY string), never a bare `'2026-01-01'`.
- Joins go inside brace escapes: `{ IJ TableA A INNER JOIN TableB B ON
  ... }`; keep to one or two joins per statement.
- No subqueries, no UNION, no square brackets, no semicolons anywhere in
  the inner string (a semicolon also violates the connector's
  single-statement rule).
- WHERE/ORDER BY on key columns uses index access; anything else scans
  whole files over the network — filter on keys wherever possible.
- Push filters INTO the inner string. `OPENQUERY(...)` runs the inner
  query remotely and only then applies the outer WHERE — an unfiltered
  inner SELECT drags the entire file across the wire first.

**Verify a new connection.** Work up in steps, each through
`mssql__query`:

1. `SELECT 1 AS ok` — connector and credentials are alive.
2. Premium: `SELECT TOP 1 CustomerName FROM dbo.AR_Customer` — right
   database, right permissions. Bridge: the OPENQUERY customer probe
   above with an inner `TOP 1` removed if the driver rejects it (TOP
   support varies by driver build — degrade to `max_rows` capping).
3. `mssql__list_tables` — on Premium you should see the `AR_`/`GL_`/`SO_`
   module prefixes; on a bridge you will only see the bridge's own local
   tables, which is expected (Sage tables are behind the linked server
   and invisible to catalog views).

## API Patterns

- `mssql__query` — one SELECT/WITH statement; `max_rows` up to 1000;
  results return as JSON rows with a `truncated` flag
- `mssql__list_tables` — Premium: enumerate Sage tables; bridge: bridge
  tables only
- `mssql__describe_table` — Premium only in practice; it takes a plain
  identifier (letters/digits/underscores), so linked-server four-part
  names can never be described — use the curated table reference in
  schema-and-conventions instead

Premium and bridge forms of the same question, side by side:

```sql
SELECT ARDivisionNo, CustomerNo, InvoiceNo, Balance
FROM dbo.AR_OpenInvoice
WHERE Balance <> 0
```

```sql
SELECT * FROM OPENQUERY(SAGE100,
  'SELECT ARDivisionNo, CustomerNo, InvoiceNo, Balance
   FROM AR_OpenInvoice WHERE Balance <> 0')
```

## Gotchas

- **Bitness**: a 64-bit SQL Server + the default 32-bit Sage driver is
  the classic dead end ("The OLE DB provider MSDASQL ... could not be
  found" or architecture-mismatch errors). Install the 64-bit driver or
  bridge through a 32-bit SQL Express instance.
- **DSN defaults trade consistency for speed**: `DirtyReads=1` may serve
  slightly stale data and `BurstMode=1` takes short file locks while
  fetching. Fine for reporting; turn both off if users report contention
  while working in Sage during business hours.
- **Advanced sites should prefer the Client/Server ODBC service** — it
  executes reads on the host instead of dragging files over the network,
  and markedly speeds up the bridge.
- **A down database fails late.** The connector connects lazily, so a
  wrong host or password surfaces on the first `mssql__query` call, not
  when the config is saved.
- **Query timeout is 15 seconds** on the connector. A ProvideX full-file
  scan of a large history table will hit it — filter on keys, or use the
  Premium-style summary tables where they exist.
- **Do not reuse SOTAMAS90** and do not create read-write ODBC paths to
  ProvideX data; third-party writable drivers exist and are a documented
  data-corruption risk. Writes belong to Sage's Business Object
  Interface, outside this plugin's scope.
- **Sage 300 is a different product** with a different schema — none of
  this plugin applies to it.

## Related Skills

- schema-and-conventions — table naming, composite keys, data quirks, and
  the curated table reference
- sales-and-ar — customers, invoices, receivables, sales analysis
- purchasing-and-ap — vendors, payables, purchase orders
- gl-and-financials — chart of accounts, trial balance, GL detail
- inventory-and-items — items, warehouses, quantities, pricing
