---
name: "Sage 100 Sales and AR"
description: >
  Querying Sage 100 sales and receivables through Conduit's SQL connector:
  customer lookup and profile, open invoices and AR aging, posted invoice
  history and sales analysis by customer or item, open sales orders and
  backlog, top customers, and payment history — with the composite keys
  and invoice-type semantics that make the numbers come out right.
when_to_use: >-
  When a question involves Sage 100 customers, receivables, invoices,
  sales analysis, or sales orders: who owes what and how old it is, what
  a customer bought over a period, which items or customers drive
  revenue, or what is ordered but not yet shipped. Use when: sage
  customers, AR aging, open invoices, receivables, invoice history,
  sales by customer, sales by item, top customers, open sales orders,
  backlog, AR_OpenInvoice, AR_InvoiceHistoryHeader, SO_SalesOrderHeader,
  customer payments, cash receipts.
---

# Sage 100 Sales and AR

## Overview

This skill covers the Accounts Receivable and Sales Order side of Sage
100: the customer master, what customers owe (open invoices, aging),
what they bought (posted invoice history), what they have on order
(open sales orders, backlog), and how they paid. Queries run through
`mssql__query`, with `mssql__describe_table` for column verification
(the `mssql__` prefix follows the connector slug the org configured; a
named instance may surface as `sage100__query`).

Every query here is a single read-only SELECT, 100 rows by default and
1000 max. Primary form is Premium T-SQL against `dbo`; where the
ProvideX (Standard/Advanced) dialect differs it is called out inline,
and the OPENQUERY bridge mechanics live in the connecting-and-setup
skill. Table shapes and join keys come from `table-reference.md` in
the schema-and-conventions skill; this skill does not repeat that map.

## Key Concepts

**The customer key is ARDivisionNo + CustomerNo.** Never join or filter
on `CustomerNo` alone — it is only unique within its division, and the
division column holds `'00'` (and still participates in every key) even
when a site has divisions disabled. Carry both columns through every
join in this domain.

**Invoices live in three populations.**

1. `AR_OpenInvoice` — open (unpaid) invoices, one row per open
   document. Key: ARDivisionNo + CustomerNo + InvoiceNo + InvoiceType.
2. `AR_InvoiceHistoryHeader` / `AR_InvoiceHistoryDetail` — posted
   invoice history for both AR and Sales Order invoices. Header key:
   **InvoiceNo + HeaderSeqNo**; details join on both columns.
3. `SO_InvoiceHeader` / `SO_InvoiceDetail` — invoices sitting in an
   unposted Sales Order invoice batch. A "complete list of invoices"
   must consider these plus posted history; they are invisible to both
   of the tables above until the batch posts.

**InvoiceType drives the sign.** In `AR_OpenInvoice` the types are IN
(invoice), CM (credit memo), DM (debit memo), FC (finance charge), PP
(prepayment), PY (payment), and BC/BF balance rows. **PP and PY rows
carry negative balances** — they are unapplied credits, and a naive
`SUM(Balance)` is only correct because of that sign. In invoice history
the types include IN, CM, DM, AD, FC, CA, XD; for sales analysis the
canonical treatment is to negate CM rows.

**Sales amount = TaxableSalesAmt + NonTaxableSalesAmt.** That is net
merchandise revenue on `AR_InvoiceHistoryHeader`; `FreightAmt` and
`SalesTaxAmt` are separate columns to add only if the question is about
gross invoice totals.

**Order status and type gate every backlog number.** On
`SO_SalesOrderHeader`, OrderStatus is N (new), O (open), C (closed), H
(hold); OrderType is S/B/M/R/Q/P, where Q is a quote. Backlog queries
take `OrderStatus IN ('N','O','H')` and exclude quotes with
`OrderType <> 'Q'`. Open quantity on a line is
`QuantityOrdered - QuantityShipped` from `SO_SalesOrderDetail`.

**Blank dates are `1753-01-01` on Premium.** Date columns are
`datetime NOT NULL`; a blank due date materializes as the sentinel and
would look centuries overdue in aging math. Exclude or special-case it
in every DATEDIFF.

## Common Workflows

**Find a customer.** Look up by name fragment, then use the full
composite key in every subsequent query:

```sql
SELECT c.ARDivisionNo, c.CustomerNo, c.CustomerName, c.City, c.State
FROM dbo.AR_Customer c
WHERE c.CustomerName LIKE '%ACME%'
ORDER BY c.CustomerName
```

The master also carries address, phone, terms, and salesperson fields;
verify their exact names with `mssql__describe_table` before selecting
them.

**What does this customer owe?** List open documents; the PP/PY rows
that appear are unapplied payments and prepayments, not receivables:

```sql
SELECT o.InvoiceNo, o.InvoiceType, o.InvoiceDate, o.InvoiceDueDate,
       o.Balance
FROM dbo.AR_OpenInvoice o
WHERE o.ARDivisionNo = '01'
  AND o.CustomerNo = 'ACME01'
  AND o.Balance <> 0
ORDER BY o.InvoiceDueDate
```

**Receivables aging (Premium).** Bucket open balances by days past
due, keeping the blank-date sentinel out of the math:

```sql
SELECT c.ARDivisionNo, c.CustomerNo, c.CustomerName,
       SUM(o.Balance) AS TotalDue,
       SUM(CASE WHEN DATEDIFF(day, o.InvoiceDueDate, GETDATE()) <= 0
                THEN o.Balance ELSE 0 END) AS NotYetDue,
       SUM(CASE WHEN DATEDIFF(day, o.InvoiceDueDate, GETDATE())
                BETWEEN 1 AND 30 THEN o.Balance ELSE 0 END) AS Age1To30,
       SUM(CASE WHEN DATEDIFF(day, o.InvoiceDueDate, GETDATE())
                BETWEEN 31 AND 60 THEN o.Balance ELSE 0 END) AS Age31To60,
       SUM(CASE WHEN DATEDIFF(day, o.InvoiceDueDate, GETDATE())
                BETWEEN 61 AND 90 THEN o.Balance ELSE 0 END) AS Age61To90,
       SUM(CASE WHEN DATEDIFF(day, o.InvoiceDueDate, GETDATE()) > 90
                THEN o.Balance ELSE 0 END) AS AgeOver90
FROM dbo.AR_OpenInvoice o
JOIN dbo.AR_Customer c
  ON c.ARDivisionNo = o.ARDivisionNo AND c.CustomerNo = o.CustomerNo
WHERE o.Balance <> 0
  AND o.InvoiceDueDate > '1753-01-01'
GROUP BY c.ARDivisionNo, c.CustomerNo, c.CustomerName
ORDER BY TotalDue DESC
```

Rows excluded by the sentinel filter have no due date — check them
separately and age them from `InvoiceDate` if any exist. On the
ProvideX path DATEDIFF does not exist: pull `InvoiceDueDate` and
`Balance` per invoice and compute the buckets client-side, or try
`{fn TIMESTAMPDIFF(SQL_TSI_DAY, ...)}` and verify it at runtime on the
site's driver build before trusting it.

**Net sales by customer over a period.** Posted history, CM negated,
half-open date range (which also excludes sentinel dates):

```sql
SELECT h.ARDivisionNo, h.CustomerNo,
       SUM(CASE WHEN h.InvoiceType = 'CM'
                THEN -(h.TaxableSalesAmt + h.NonTaxableSalesAmt)
                ELSE h.TaxableSalesAmt + h.NonTaxableSalesAmt
           END) AS NetSales
FROM dbo.AR_InvoiceHistoryHeader h
WHERE h.InvoiceDate >= '2026-01-01' AND h.InvoiceDate < '2026-07-01'
GROUP BY h.ARDivisionNo, h.CustomerNo
ORDER BY NetSales DESC
```

**Top customers by sales.** Same aggregate joined to the customer
master for names:

```sql
SELECT TOP 10 c.ARDivisionNo, c.CustomerNo, c.CustomerName,
       SUM(CASE WHEN h.InvoiceType = 'CM'
                THEN -(h.TaxableSalesAmt + h.NonTaxableSalesAmt)
                ELSE h.TaxableSalesAmt + h.NonTaxableSalesAmt
           END) AS NetSales
FROM dbo.AR_InvoiceHistoryHeader h
JOIN dbo.AR_Customer c
  ON c.ARDivisionNo = h.ARDivisionNo AND c.CustomerNo = h.CustomerNo
WHERE h.InvoiceDate >= '2026-01-01' AND h.InvoiceDate < '2027-01-01'
GROUP BY c.ARDivisionNo, c.CustomerNo, c.CustomerName
ORDER BY NetSales DESC
```

On ProvideX, `TOP` is verify-at-runtime on the bundled driver — if it
fails, drop it, keep the ORDER BY, and let the row cap truncate.

**Sales by item.** The header-detail join — always on both key columns:

```sql
SELECT d.ItemCode,
       SUM(d.QuantityShipped) AS QtyShipped,
       SUM(d.ExtensionAmt) AS Sales
FROM dbo.AR_InvoiceHistoryHeader h
JOIN dbo.AR_InvoiceHistoryDetail d
  ON d.InvoiceNo = h.InvoiceNo AND d.HeaderSeqNo = h.HeaderSeqNo
WHERE h.InvoiceDate >= '2026-01-01' AND h.InvoiceDate < '2026-07-01'
  AND h.InvoiceType <> 'CM'
GROUP BY d.ItemCode
ORDER BY Sales DESC
```

(`ExtensionAmt` is widely used but flagged verify-first in the research
backbone — confirm it with `mssql__describe_table` on a new site. Fold
CM rows back in with negated amounts when returns matter.) The PVX form
wraps the join in braces and uses date-escape literals:

```sql
SELECT D.ItemCode, SUM(D.QuantityShipped), SUM(D.ExtensionAmt)
FROM { IJ AR_InvoiceHistoryHeader H
       INNER JOIN AR_InvoiceHistoryDetail D
       ON H.InvoiceNo = D.InvoiceNo
       AND H.HeaderSeqNo = D.HeaderSeqNo }
WHERE H.InvoiceDate >= {d'2026-01-01'}
  AND H.InvoiceDate < {d'2026-07-01'}
  AND H.InvoiceType <> 'CM'
GROUP BY D.ItemCode
```

**Open sales orders.** Backlog headers exclude quotes and closed
orders:

```sql
SELECT h.SalesOrderNo, h.OrderDate, h.ARDivisionNo, h.CustomerNo,
       h.OrderStatus
FROM dbo.SO_SalesOrderHeader h
WHERE h.OrderStatus IN ('N', 'O', 'H')
  AND h.OrderType <> 'Q'
ORDER BY h.OrderDate
```

Order totals are conventionally read as TaxableAmt + NonTaxableAmt on
the header — those column names are not in the verified layout set, so
confirm them with `mssql__describe_table` before using them.

**Backlog by item.** Open quantity is ordered minus shipped, summed
from the detail:

```sql
SELECT d.ItemCode,
       SUM(d.QuantityOrdered - d.QuantityShipped) AS OpenQty
FROM dbo.SO_SalesOrderHeader h
JOIN dbo.SO_SalesOrderDetail d ON d.SalesOrderNo = h.SalesOrderNo
WHERE h.OrderStatus IN ('N', 'O', 'H')
  AND h.OrderType <> 'Q'
GROUP BY d.ItemCode
HAVING SUM(d.QuantityOrdered - d.QuantityShipped) > 0
ORDER BY OpenQty DESC
```

**The "all invoices" check.** Before declaring an invoice list
complete, check for unposted batches with
`SELECT COUNT(*) FROM dbo.SO_InvoiceHeader` — anything there has not
reached AR history yet. On Premium a single UNION ALL over
`SO_InvoiceHeader` and `AR_InvoiceHistoryHeader` works (one statement);
on ProvideX UNION is unavailable — run two queries and combine
client-side. `SO_InvoiceHeader` columns beyond `InvoiceNo` are
verify-first with `mssql__describe_table`.

**How did the customer pay?** `AR_TransactionPaymentHistory` holds
payments applied to invoices (keyed ARDivisionNo + CustomerNo +
InvoiceNo); `AR_CashReceiptsHistory` is deposit/receipt oriented.
Column detail for both is verify-first with `mssql__describe_table`:

```sql
SELECT TOP 20 *
FROM dbo.AR_TransactionPaymentHistory
WHERE ARDivisionNo = '01' AND CustomerNo = 'ACME01'
```

## API Patterns

- `mssql__query` accepts exactly one SELECT/WITH statement, no
  semicolons inside, `max_rows` capped at 1000, and marks clipped
  results `"truncated": true`. When it clips, do not re-request more
  rows — aggregate in SQL, tighten the date or key filter, or page by
  key (e.g. `WHERE CustomerNo > '01LAST' ORDER BY CustomerNo`, carrying
  ARDivisionNo in the predicate on multi-division sites).
- `mssql__describe_table` is the verify-first tool this skill leans on
  for the flagged columns (order totals, payment history detail,
  `ExtensionAmt`). It takes plain identifiers only — every `AR_`/`SO_`
  table name passes; linked-server four-part names never do.
- On ProvideX bridge sites, all of the SQL here travels inside
  OPENQUERY in the PVX dialect: `{d'YYYY-MM-DD'}` date literals,
  `{ IJ ... }` join braces, no subqueries or UNION, one or two joins
  per statement at most, single statement always. The bridge mechanics
  and dialect ground rules are in connecting-and-setup — do not guess
  them from T-SQL habits.

## Gotchas

- **PP and PY rows are negative.** A per-customer `SUM(Balance)` is
  net of unapplied prepayments and payments; if the question is "gross
  open invoices," filter to the debit types instead of summing
  everything.
- **`1753-01-01` due dates poison aging.** The sentinel lands in the
  oldest bucket as centuries past due. Filter it out and handle
  no-due-date rows deliberately.
- **InvoiceNo alone duplicates history rows.** Invoice numbers are
  reused across header sequences; every history join carries
  `HeaderSeqNo` too.
- **CustomerNo alone mixes divisions** — division + number, always.
- **Quotes inflate backlog.** OrderType Q rows sit in
  `SO_SalesOrderHeader` alongside real orders; forgetting
  `OrderType <> 'Q'` overstates open demand.
- **`SO_SalesOrderHeader` is not order history.** Completed orders
  leave it; `SO_SalesOrderHistoryHeader` keeps every order ever,
  including closed and deleted ones.
- **Posted history is incomplete while a batch is open.** Invoices in
  `SO_InvoiceHeader` have not posted to AR history — reconcile both
  when completeness matters.
- **History tables are huge.** Never query
  `AR_InvoiceHistoryHeader`/`Detail` without a date or key filter; the
  row cap will truncate silently correct-looking partial answers.

## Related Skills

- schema-and-conventions — table map (`table-reference.md`), composite
  keys, data-type quirks, schema discovery
- connecting-and-setup — edition detection, connector configuration,
  the ProvideX linked-server bridge, and OPENQUERY dialect rules
- purchasing-and-ap — vendors, payables, purchase orders
- gl-and-financials — chart of accounts, trial balance, GL detail
- inventory-and-items — items, warehouses, quantities, pricing
