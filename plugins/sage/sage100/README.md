# Sage 100 plugin

Read-only reporting against Sage 100 ERP through [Conduit](https://conduit.wyre.ai)'s
**generic on-prem SQL connector** — no Sage-specific MCP server involved.

## Why a skill instead of a Sage MCP server

Sage 100's data is reachable with general-purpose technology: Premium edition
runs natively on Microsoft SQL Server, and Standard/Advanced expose their
ProvideX file store through a plain read-only ODBC driver. There is no
proprietary query protocol to wrap, so a vendor-specific server would add
maintenance surface without adding capability. Everything Sage-specific is
*knowledge* — table maps, composite join keys, dialect quirks — and knowledge
ships as skills. (As of August 2026 no official Sage MCP server for Sage 100
exists; the SQL/ODBC path is also the only edition-spanning one.)

## What you get

| Skill | Covers |
|---|---|
| `schema-and-conventions` | Editions, table naming, composite keys, data-type quirks, schema discovery, plus a curated table reference |
| `connecting-and-setup` | Edition detection, connector config, the ProvideX linked-server bridge, OPENQUERY dialect, verification |
| `sales-and-ar` | Customers, open invoices, AR aging, sales analysis, open orders and backlog |
| `purchasing-and-ap` | Vendors, AP aging, cash requirements, payment history, purchase orders |
| `gl-and-financials` | Chart of accounts, trial balance, GL detail drill-downs |
| `inventory-and-items` | Items, per-warehouse quantities, availability, costs, price schedules |

The skills are written against the connector's exact tool surface:
`mssql__query` (single read-only SELECT/WITH, 1000-row cap),
`mssql__list_tables`, and `mssql__describe_table`.

## Prerequisites

Customer-side setup is heavier than for cloud vendors:

- A deployed Conduit tunnel with the `mssql` connector configured.
- **Premium**: a read-only SQL login (`db_datareader` or narrower) on the
  company database (`MAS_<CompanyCode>`).
- **Standard/Advanced**: a Windows bridge host with Sage's 64-bit ODBC
  driver, a System DSN with stored credentials, a linked server over
  MSDASQL, and SQL Server (Express is fine) for the connector to reach.
  The `connecting-and-setup` skill walks through it.

## External references

Official Sage documentation used to build these skills (not linked from the
skills themselves):

- Sage 100 File Layouts and Object Reference: `https://help-sage100.na.sage.com/<year>/FLOR/index.htm`
- Sage KB 19495 (silent ODBC connections), KB 36360 (linked server), KB 34167 (64-bit ODBC driver)
- PxPlus ODBC SQL reference: `https://manual.pvxplus.com/PXPLUS/odbc/`

## Governance

See [GOVERNANCE.md](./GOVERNANCE.md) for the read-only posture and its limits.
