---
name: "Sage 100 Purchasing and AP"
description: >
  Querying the purchasing and accounts payable side of Sage 100: vendor
  lookup, open payables and aging, cash requirements and early-payment
  discounts, posted AP invoice and payment history, and open purchase
  orders with their receipt status — all through Conduit's generic SQL
  connector, with the composite keys that make AP joins correct.
when_to_use: >-
  When a question involves vendors, bills, or what the company owes and has
  ordered: who a vendor is, which AP invoices are open or overdue, what is
  due in the next week or month, where discount opportunities are, how much
  was spent with a vendor over a period, which invoices a check paid, or
  which purchase order lines are still waiting on goods. Use when: vendor,
  accounts payable, AP aging, payables, amounts due, cash requirements,
  vendor spend, check history, payment history, purchase order, PO receipts,
  AP_Vendor, AP_OpenInvoice, PO_PurchaseOrderHeader.
---

# Sage 100 Purchasing and AP

## Overview

This skill covers the AP and PO modules: vendors (`AP_Vendor`), open
payables (`AP_OpenInvoice`), posted invoice history
(`AP_InvoiceHistoryHeader`/`Detail`), payment history
(`AP_CheckHistoryHeader`/`Detail`), and purchasing
(`PO_PurchaseOrderHeader`/`Detail`, `PO_ReceiptHistoryHeader`/`Detail`).
Queries run through `mssql__query`, with `mssql__describe_table` to confirm
columns (the `mssql__` prefix follows the connector slug the org
configured; a named instance may surface as `sage100__query`).

The primary form throughout is Premium T-SQL against `dbo`. On
Standard/Advanced the same table names are reached over the ProvideX ODBC
dialect through an OPENQUERY bridge — dialect differences are noted where
they bite, and the bridge mechanics live in the connecting-and-setup skill.
The shared table map in `table-reference.md` (next to the
schema-and-conventions skill) is the source for keys and columns; this
skill applies it, it does not restate it.

## Key Concepts

**APDivisionNo + VendorNo is the vendor key — everywhere.** `VendorNo` is
only unique within its division, so every AP join, GROUP BY, and page key
carries both columns. Sites with divisions "off" still have `'00'` in
`APDivisionNo`, and it still participates in the key.

**Open vs history.** `AP_OpenInvoice` holds only unpaid balances; once an
invoice is fully paid it exists only in the history tables. "What do we owe"
questions go to the open file; "what did we spend / what did we pay"
questions go to `AP_InvoiceHistoryHeader`/`Detail` and
`AP_CheckHistoryHeader`/`Detail`. Payments key on **BankCode + CheckNo**,
and check detail rows reference the invoices each check paid.

**PO base tables are the open set.** `PO_PurchaseOrderHeader`/`Detail`
hold open purchase orders; receipt progress is on the detail line as
`QuantityOrdered` vs `QuantityReceived` (with `UnitCost` for valuation).
Received-goods history lives in `PO_ReceiptHistoryHeader`/`Detail`, keyed
by ReceiptNo. PO headers carry `APDivisionNo + VendorNo` to join to
`AP_Vendor`.

**The blank-date sentinel poisons date math.** On Premium, dates are
`datetime NOT NULL` and blank dates are stored as `1753-01-01`. An
unguarded aging query puts those rows tens of thousands of days past due.
Exclude the sentinel explicitly in every DATEDIFF, MIN, or window filter.

**Verified vs verify-first columns.** Keys and the columns named in
`table-reference.md` (`Balance`, `InvoiceDueDate`, `InvoiceAmt`,
`DiscountAmt`, `QuantityOrdered`, `QuantityReceived`, `UnitCost`, ...) are
verified. Some columns used below follow Sage's conventions but are not in
the shared map — vendor name/address fields, history-table dates and
amounts, check-detail invoice references, and any discount due-date
column. Those are flagged in place: verify with `mssql__describe_table`
first.

**Respect the row caps.** `mssql__query` returns 100 rows by default and
1000 at most. Aggregate in SQL rather than pulling invoice detail to
summarize client-side, filter history tables by date, and page big vendor
or invoice lists by key (`WHERE APDivisionNo = '00' AND VendorNo >
'LASTSEEN' ORDER BY VendorNo`).

## Common Workflows

**Find a vendor.** Name and address columns on `AP_Vendor` are not in the
shared map — `VendorName` is the conventional name; verify with
`mssql__describe_table` first, then:

```sql
SELECT v.APDivisionNo, v.VendorNo, v.VendorName
FROM dbo.AP_Vendor v
WHERE v.VendorName LIKE '%ACME%'
ORDER BY v.APDivisionNo, v.VendorNo
```

**Open payables by vendor.** The core AP snapshot — always joined on the
full composite key:

```sql
SELECT o.APDivisionNo, o.VendorNo, v.VendorName,
       COUNT(*) AS OpenInvoices, SUM(o.Balance) AS TotalBalance
FROM dbo.AP_OpenInvoice o
JOIN dbo.AP_Vendor v
  ON v.APDivisionNo = o.APDivisionNo AND v.VendorNo = o.VendorNo
WHERE o.Balance <> 0
GROUP BY o.APDivisionNo, o.VendorNo, v.VendorName
ORDER BY TotalBalance DESC
```

**Payables aging.** Bucket by days past due with DATEDIFF/CASE, guarding
the sentinel. Note the guard drops invoices with a blank due date — if
totals must tie to the vendor balance, count `InvoiceDueDate =
'1753-01-01'` rows in a separate query:

```sql
SELECT o.APDivisionNo, o.VendorNo,
  SUM(CASE WHEN DATEDIFF(day, o.InvoiceDueDate, GETDATE()) <= 0
           THEN o.Balance ELSE 0 END) AS CurrentDue,
  SUM(CASE WHEN DATEDIFF(day, o.InvoiceDueDate, GETDATE()) BETWEEN 1 AND 30
           THEN o.Balance ELSE 0 END) AS Days1To30,
  SUM(CASE WHEN DATEDIFF(day, o.InvoiceDueDate, GETDATE()) BETWEEN 31 AND 60
           THEN o.Balance ELSE 0 END) AS Days31To60,
  SUM(CASE WHEN DATEDIFF(day, o.InvoiceDueDate, GETDATE()) > 60
           THEN o.Balance ELSE 0 END) AS Over60
FROM dbo.AP_OpenInvoice o
WHERE o.Balance <> 0
  AND o.InvoiceDueDate > '1753-01-01'
GROUP BY o.APDivisionNo, o.VendorNo
ORDER BY o.APDivisionNo, o.VendorNo
```

On the ProvideX path, skip SQL-side bucketing: pull the filtered rows
(division, vendor, invoice, due date, balance) and compute the buckets
client-side.

**Cash requirements — what is due soon.** Everything due through a cutoff,
past-due included, grouped by due date for a payment-run view:

```sql
SELECT o.InvoiceDueDate, SUM(o.Balance) AS AmountDue
FROM dbo.AP_OpenInvoice o
WHERE o.Balance <> 0
  AND o.InvoiceDueDate > '1753-01-01'
  AND o.InvoiceDueDate <= DATEADD(day, 14, GETDATE())
GROUP BY o.InvoiceDueDate
ORDER BY o.InvoiceDueDate
```

**Discount opportunities.** `DiscountAmt` on the open invoice shows where
early-payment discounts exist. The discount's deadline lives in a separate
column whose name varies — verify with `mssql__describe_table` first
before filtering or sorting on it; do not guess a name:

```sql
SELECT o.APDivisionNo, o.VendorNo, o.InvoiceNo,
       o.InvoiceDueDate, o.Balance, o.DiscountAmt
FROM dbo.AP_OpenInvoice o
WHERE o.Balance <> 0 AND o.DiscountAmt <> 0
ORDER BY o.InvoiceDueDate
```

**Spend by vendor over a period.** From posted history. Only the keys of
`AP_InvoiceHistoryHeader` are in the shared map; `InvoiceDate` and
`InvoiceAmt` below are the conventional names — verify with
`mssql__describe_table` first. Line-level breakdowns join
`AP_InvoiceHistoryDetail` on the same vendor + invoice keys:

```sql
SELECT h.APDivisionNo, h.VendorNo, v.VendorName,
       COUNT(*) AS Invoices, SUM(h.InvoiceAmt) AS TotalSpend
FROM dbo.AP_InvoiceHistoryHeader h
JOIN dbo.AP_Vendor v
  ON v.APDivisionNo = h.APDivisionNo AND v.VendorNo = h.VendorNo
WHERE h.InvoiceDate >= '2026-01-01' AND h.InvoiceDate < '2026-07-01'
GROUP BY h.APDivisionNo, h.VendorNo, v.VendorName
ORDER BY TotalSpend DESC
```

History can include adjustment and credit documents — spot-check how they
are signed before presenting spend totals as authoritative.

**Trace a payment — which invoices a check paid.** Checks key on
BankCode + CheckNo; the detail rows reference the paid invoices. Beyond
those keys, column names on the check tables (invoice reference, check
date, amounts) are verify-first — run `mssql__describe_table` on
`AP_CheckHistoryHeader` and `AP_CheckHistoryDetail` before widening the
select. `InvoiceNo` below is the conventional reference name. CheckNo is a
string key — probe a few header rows first to see the stored format:

```sql
SELECT d.BankCode, d.CheckNo, d.APDivisionNo, d.VendorNo, d.InvoiceNo
FROM dbo.AP_CheckHistoryDetail d
WHERE d.BankCode = 'A' AND d.CheckNo = '012345'
```

The reverse question — a vendor's payment history — starts from
`AP_CheckHistoryHeader` filtered by `APDivisionNo + VendorNo`, joined to
detail on BankCode + CheckNo.

**Open PO lines awaiting receipt.** Receipt progress is ordered vs
received on the detail line:

```sql
SELECT h.PurchaseOrderNo, h.APDivisionNo, h.VendorNo,
       d.ItemCode, d.QuantityOrdered, d.QuantityReceived,
       d.QuantityOrdered - d.QuantityReceived AS QuantityRemaining,
       d.UnitCost,
       (d.QuantityOrdered - d.QuantityReceived) * d.UnitCost AS OpenValue
FROM dbo.PO_PurchaseOrderHeader h
JOIN dbo.PO_PurchaseOrderDetail d
  ON d.PurchaseOrderNo = h.PurchaseOrderNo
WHERE d.QuantityOrdered > d.QuantityReceived
ORDER BY h.PurchaseOrderNo
```

Aggregate the same shape (SUM of OpenValue, GROUP BY
`APDivisionNo + VendorNo` joined to `AP_Vendor`) for open-PO exposure by
vendor. For what actually arrived, `PO_ReceiptHistoryHeader`/`Detail` hold
receipt-of-goods history keyed by ReceiptNo; columns beyond that key
(receipt date, PO linkage) are verify-first via `mssql__describe_table`.

## API Patterns

Every example above is a single read-only SELECT — one statement per
`mssql__query` call, no semicolons, `max_rows` up to 1000. Aggregates and
GROUP BY keep results under the cap; when a raw list is genuinely needed,
page by the composite key rather than re-running an unbounded select.

On the ProvideX path the same logic is rewritten in the PVX dialect and
sent through the OPENQUERY bridge (see connecting-and-setup for the
mechanics): `{d'YYYY-MM-DD'}` date literals instead of quoted strings,
joins wrapped in `{ IJ ... }` braces, no subqueries or UNION, one or two
joins at most, single statement only. The open-payables snapshot in PVX
form:

```sql
SELECT o.APDivisionNo, o.VendorNo, SUM(o.Balance)
FROM { IJ AP_OpenInvoice o INNER JOIN AP_Vendor v
     ON o.APDivisionNo = v.APDivisionNo AND o.VendorNo = v.VendorNo }
WHERE o.Balance <> 0 AND o.InvoiceDueDate <= {d'2026-08-31'}
GROUP BY o.APDivisionNo, o.VendorNo
```

The `1753-01-01` sentinel is Premium-only; on the ODBC path blank dates
surface as empty or NULL depending on DSN settings — connecting-and-setup
covers the handling. `mssql__describe_table` reaches Premium tables
directly but cannot see through a linked server, so on bridge sites
verify-first columns are confirmed with a one-row probe query instead.

## Gotchas

- **Joining on VendorNo alone silently mixes divisions.** Carry
  `APDivisionNo` through every join, GROUP BY, and page key — even when
  the site's divisions are all `'00'`.
- **Unguarded aging is wrong aging.** A `1753-01-01` due date is a blank,
  not a 270-year-old bill; exclude it before DATEDIFF or MIN. And once
  excluded, remember those invoices still owe money — reconcile them
  separately when totals must tie.
- **`DiscountAmt` is the opportunity, not the deadline.** The discount
  due-date column name is not verified here; confirm it with
  `mssql__describe_table` rather than guessing, or present discounts
  sorted by `InvoiceDueDate` as the safe default.
- **"How much did we pay X" is a history question.** `AP_OpenInvoice`
  loses fully paid invoices; payment questions go to the check history
  tables, spend questions to invoice history.
- **A PO line is not received just because the PO is old.** Receipt status
  is per line (`QuantityReceived` vs `QuantityOrdered`), not per header —
  a nine-line PO can be complete on eight lines and still owe you goods.
- **History tables grow huge.** Filter `AP_InvoiceHistoryHeader` and
  `AP_CheckHistoryHeader` by date or vendor before selecting; never pull
  them raw into a 1000-row cap and call the result "total spend" — check
  the `truncated` flag on every result.
- **PVX is not T-SQL.** DATEDIFF/CASE bucketing, DATEADD windows, and
  multi-join aggregates in this skill are Premium forms; on
  Standard/Advanced keep queries to simple filtered SELECTs with `{ IJ }`
  joins and bucket client-side.

## Related Skills

- schema-and-conventions — table naming, composite keys, data-type quirks,
  and the shared `table-reference.md` map this skill builds on
- connecting-and-setup — edition detection, connector configuration, the
  ProvideX linked-server bridge, and OPENQUERY dialect rules
- sales-and-ar — the mirror image: customers, invoices, receivables
- gl-and-financials — chart of accounts, trial balance, GL detail behind
  AP postings
- inventory-and-items — items, warehouses, and quantities the PO lines
  reference
