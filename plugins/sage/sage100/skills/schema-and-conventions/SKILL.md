---
name: "Sage 100 Schema and Conventions"
description: >
  The Sage 100 database landscape: editions and their storage backends,
  module-prefixed table naming, composite keys, data-type quirks, and how to
  discover schema through Conduit's generic SQL connector tools. The
  foundation every other Sage 100 skill builds on.
when_to_use: >-
  When you are about to query a Sage 100 database for the first time in a
  session, when a join returns duplicate or missing rows, when dates or
  times come back looking wrong, or when you need to find which table holds
  a piece of Sage 100 data. Use when: sage 100 schema, sage tables, mas 90,
  mas 200, providex, AR_Customer, table layout, sage 100 join keys, or sage
  100 data types.
---

# Sage 100 Schema and Conventions

## Overview

Sage 100 is queried through Conduit's generic SQL connector — there is no
Sage-specific server. The model-facing tools are `mssql__query`,
`mssql__list_tables`, and `mssql__describe_table` (the `mssql__` prefix
follows the connector slug the org configured; a named instance may surface
as `sage100__query`). All access is read-only by design: single SELECT/WITH
statements, 100 rows by default, 1000 max per call.

Which edition the customer runs determines what you are actually talking to:

| Edition | Backend | What the connector reaches |
|---|---|---|
| Premium | Microsoft SQL Server | The company database directly (`MAS_ABC`) |
| Standard / Advanced | ProvideX files + read-only ODBC | A bridge SQL Server with a linked server; queries go through OPENQUERY |

Table names are identical across editions (`AR_Customer` is `AR_Customer`
everywhere), so the schema knowledge here applies to both; only the SQL
dialect differs. See the connecting-and-setup skill for which path a given
site uses.

## Key Concepts

**Module prefixes.** Every table is `<Module>_<Entity>`: `GL_` general
ledger, `AR_` receivables, `AP_` payables, `SO_` sales orders, `PO_`
purchasing, `IM_` inventory, `CI_` common information (item master), `BM_`
bill of materials, `JC_`/`JT_` job cost, `PR_` payroll, `SY_` system.
Suffixes follow a pattern: `...Header`/`...Detail` for document pairs,
`...History` for posted records. The full curated map with join keys is in
`table-reference.md` next to this skill.

**One company = one database.** Premium stores each company in its own
database, `MAS_<CompanyCode>` (e.g. `MAS_ABC`), plus `MAS_SYSTEM` for
users, roles, and the company list (`SY_Company`). Standard/Advanced mirror
this as folders of ProvideX files. Consequently **no table has a
CompanyCode column** — company separation is physical. A connector instance
is scoped to one company; multi-company orgs typically configure one named
instance per company (`sage100_abc`, `sage100_xyz`).

**Composite keys are the #1 source of wrong results.** Customer and vendor
numbers are only unique within their division:

- Customers: `ARDivisionNo + CustomerNo` (never `CustomerNo` alone)
- Vendors: `APDivisionNo + VendorNo`
- Inventory quantities: `ItemCode + WarehouseCode`
- AR invoice history: `InvoiceNo + HeaderSeqNo`

Even when a site has divisions "turned off," the division column contains
`'00'` and still participates in every key. GL tables join on `AccountKey`
(a 9-digit surrogate), never on the formatted `Account` string.

**Effectively no NULLs.** Sage 100 data uses empty strings and zeros, not
NULLs. Filter with `WHERE SomeColumn <> ''`, not `IS NOT NULL`. On Premium,
date columns are `datetime NOT NULL` and blank dates appear as the sentinel
`1753-01-01` — exclude it explicitly in date math.

**Times are decimal hours.** `TimeCreated`/`TimeUpdated` hold a fraction of
the 24-hour clock as a string: `15.5` means 3:30 PM, `8.49238` is about
8:29:32 AM. hours = floor(value); minutes = (value − floor(value)) × 60.

**Character fields pad with trailing spaces** on the ProvideX path; compare
with `RTRIM` or equality on trimmed values when results look off.

## Common Workflows

**Orient in a new database (Premium).** Start with the connector's
discovery tools:

1. `mssql__list_tables` with schema `dbo` — confirm you see the module
   prefixes and note which modules the site actually uses (no `JC_` tables
   means no Job Cost data to query).
2. `mssql__describe_table` with table `AR_Customer` — verify columns before
   writing a join. Every Sage table name passes the tool's identifier rules.
3. Check the company: the database name in the connector config is the
   company (`MAS_ABC` = company ABC). To enumerate companies, query
   `SY_Company` — but note it lives in `MAS_SYSTEM`, which is usually a
   separate connector instance.

**Orient on a ProvideX bridge site.** `mssql__list_tables` shows the bridge
server's own tables, not Sage's, and `mssql__describe_table` cannot reach
through a linked server. Rely on `table-reference.md` for the schema, and
verify a table exists by selecting one row through OPENQUERY (see
connecting-and-setup).

**Find the right table for a question.** Reach for `table-reference.md`
first — it maps the ~25 tables that answer most business questions. General
rules: "open" documents live in the base table (`AR_OpenInvoice`,
`SO_SalesOrderHeader`), posted documents live in `...History` tables, and
the item master is `CI_Item` (common information), not an `IM_` table.

**Respect the row cap.** `mssql__query` returns at most 1000 rows and
flags `"truncated": true` when it clipped. Aggregate in SQL (GROUP BY,
SUM) instead of pulling detail rows to summarize client-side, and page
large extracts by a keyed range (e.g. `WHERE CustomerNo > '01-LASTSEEN'
ORDER BY CustomerNo`).

## API Patterns

The tool family this plugin is written against:

- `mssql__query` — one read-only SELECT or WITH statement per call;
  `max_rows` up to 1000; result is JSON `{rows, rowCount, truncated}`
- `mssql__list_tables` — base tables, optionally filtered by schema
- `mssql__describe_table` — columns, types, nullability for one table

Premium example — always qualify with `dbo` and remember composite keys:

```sql
SELECT c.ARDivisionNo, c.CustomerNo, c.CustomerName, c.City, c.State
FROM dbo.AR_Customer c
WHERE c.CustomerName LIKE '%ACME%'
```

Date handling on Premium (sentinel-aware):

```sql
SELECT InvoiceNo, InvoiceDate, InvoiceDueDate
FROM dbo.AR_OpenInvoice
WHERE InvoiceDueDate > '1753-01-01'
  AND InvoiceDueDate < GETDATE()
```

On the ProvideX path the same logic must use ODBC date-escape literals
inside OPENQUERY — `{d'2026-01-01'}`, never a bare quoted date string. The
dialect differences are covered in connecting-and-setup.

## Gotchas

- **Joining on CustomerNo alone** silently mixes divisions. Always carry
  the division column through every AR/AP join.
- **`Account` vs `AccountKey`**: the human-readable account number is a
  formatted string; all GL joins use the surrogate `AccountKey`.
- **`1753-01-01` is not real data** — it is Premium's blank-date sentinel
  and will dominate MIN(date) and aging calculations if not excluded.
- **`SY_Company` is not in the company database.** It lives in
  `MAS_SYSTEM`; querying it needs a connector instance pointed there.
- **`describe_table` accepts plain identifiers only** (letters, digits,
  underscores). For anything else — or any linked-server object — query
  `INFORMATION_SCHEMA.COLUMNS` through `mssql__query`, or fall back to the
  curated reference.
- **History tables grow huge.** `GL_DetailPosting` and
  `AR_InvoiceHistoryDetail` at a mature site hold millions of rows; never
  select from them without a date or key filter, and prefer the
  period-summary tables (`GL_PeriodPostingHistory`) when totals suffice.
- **The model must not attempt writes.** The connector rejects anything
  but SELECT/WITH, the Sage ODBC driver is read-only, and writing to Sage
  tables outside the application corrupts business data. Writes go through
  Sage's Business Object Interface, which is out of scope for this plugin.

## Related Skills

- connecting-and-setup — edition detection, connector configuration, the
  ProvideX linked-server bridge, and OPENQUERY dialect rules
- sales-and-ar — customers, invoices, receivables, sales analysis
- purchasing-and-ap — vendors, payables, purchase orders
- gl-and-financials — chart of accounts, trial balance, GL detail
- inventory-and-items — items, warehouses, quantities, pricing
