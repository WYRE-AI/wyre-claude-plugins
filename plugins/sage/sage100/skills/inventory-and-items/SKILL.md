---
name: "Sage 100 Inventory and Items"
description: >
  Querying Sage 100 inventory: the CI_Item item master, per-warehouse
  quantities in IM_ItemWarehouse, the standard availability formula,
  company-wide stock positions, cost tiers, price schedules, movement
  history and slow-mover analysis, and how items tie into open sales and
  purchase orders.
when_to_use: >-
  When a question involves items, stock levels, on-hand or available
  quantity, warehouse quantities, item costs or price schedules,
  slow-moving or dead stock, what an item has been selling, or how much of
  an item is committed to open orders. Use when: inventory, item lookup,
  on hand, available quantity, stock by warehouse, CI_Item,
  IM_ItemWarehouse, cost tiers, price code, dead stock, slow movers, item
  movement history, items on order.
---

# Sage 100 Inventory and Items

## Overview

Inventory questions in Sage 100 split across two places: the item master —
`CI_Item`, in Common Information, deliberately not an `IM_` table — and the
Inventory Management tables that hold quantities (`IM_ItemWarehouse`), cost
tiers (`IM_ItemCost`), price schedules (`IM_PriceCode`), and movements
(`IM_ItemTransactionHistory`). Every quantity in the system is kept per
`ItemCode + WarehouseCode`; there is no stored company-wide total, so
"how many do we have" always means aggregating over warehouses yourself.

Queries go through Conduit's generic SQL connector: `mssql__query` for
single read-only SELECT/WITH statements (100 rows by default, 1000 max per
call) and `mssql__describe_table` to confirm columns before relying on
them (the `mssql__` prefix follows the connector slug the org configured;
a named instance may surface as `sage100__query`). Examples below are
Premium T-SQL against `dbo`; ProvideX dialect differences are noted where
they bite, and the OPENQUERY bridge mechanics live in connecting-and-setup.

## Key Concepts

**The item master is `CI_Item`.** Primary key `ItemCode` (30 characters,
case-sensitive like all Sage key fields). Useful columns: `ItemCodeDesc`,
`ProductLine`, `ProductType` (F finished good, R raw material, D
discontinued, K kit), `ItemType`, `Valuation` (the costing method), and
`StandardUnitPrice` / `StandardUnitCost` (16.6 precision). Because it is
Common Information, it covers every item type — looking for an "IM_Item"
master table is the classic dead end.

**Quantities live in `IM_ItemWarehouse`, one row per item per warehouse.**
Primary key `ItemCode + WarehouseCode`. The columns are maintained running
totals: `QuantityOnHand`, `QuantityOnSalesOrder`, `QuantityOnBackOrder`,
`QuantityOnPurchaseOrder`, `QuantityOnWorkOrder`, `QuantityRequiredForWO`,
`QuantityInShipping`. Never report a bare `QuantityOnHand` as "the"
quantity without saying which warehouse — or summing over all of them.

**Available quantity is computed, not stored.** The standard formula:

    QuantityOnHand − QuantityOnSalesOrder − QuantityOnBackOrder
                   − QuantityRequiredForWO

Add `QuantityOnPurchaseOrder` when the question wants an
incoming-inclusive view. This is the conventional formula, but sites vary
in policy (whether backorders or incoming POs count) — state which
definition you used when reporting availability.

**Cost tiers sit in `IM_ItemCost`.** One row per tier per
`ItemCode + WarehouseCode`, carrying FIFO/LIFO/lot/serial cost layers for
valuation-tracked items. The tier-identifying and cost columns vary —
verify with `mssql__describe_table` first before naming them in a query.

**Price schedules sit in `IM_PriceCode`.** Rows are typed by
`PriceCodeRecord` (item-level, customer-price-level, and combination
records) with customer price levels and quantity-break columns. Exact
column names are version-sensitive — verify with `mssql__describe_table`
first.

**Movements are in `IM_ItemTransactionHistory`,** keyed by
`ItemCode + WarehouseCode + TransactionDate`. The last movement date per
item is the backbone of slow-mover and dead-stock analysis. What actually
*sold* is a different table: posted sales lines are
`AR_InvoiceHistoryDetail` (see sales-and-ar).

The full table map with keys is `table-reference.md` in the
schema-and-conventions skill — reach for it before guessing a table.

## Common Workflows

**Look up an item.** Search `CI_Item` by code or description (item lookup
example below). Check `ProductType` while you are there — D means
discontinued, K means kit — and `Valuation` to know whether cost-tier
queries are even meaningful for the item.

**Answer "how many do we have?"** Decide the scope first: one warehouse
(select the `IM_ItemWarehouse` row) or company-wide (SUM over warehouses).
Then decide on-hand vs available and apply the formula explicitly. The
warehouse-availability and stock-position examples below cover both.

**Find dead stock.** Compute last-movement dates from
`IM_ItemTransactionHistory`, then keep items that still have on-hand
quantity but no recent movement (dead-stock example below).

**See what is selling.** Aggregate `AR_InvoiceHistoryDetail` by `ItemCode`
over a date window (what-sold example below). For anything past a quick
ranking — credit memos, revenue by customer, margins — hand off to
sales-and-ar, which treats invoice types properly.

**Check items against open orders.** Open demand is
`SO_SalesOrderDetail` (via headers with `OrderStatus` in N/O/H, excluding
quotes); incoming supply is `PO_PurchaseOrderDetail`. The maintained
totals in `IM_ItemWarehouse` are the fast path for a single number;
line-level sums let you break commitments out by order, customer, or date
(demand-vs-supply example below).

**Respect the row caps.** A full item list at a mature site exceeds 1000
rows. Aggregate in SQL, filter by `ProductLine` or warehouse, and page
large extracts by keyed range (`WHERE ItemCode > 'LAST-SEEN' ORDER BY
ItemCode`) rather than pulling everything and clipping.

## API Patterns

**Item lookup** — code or description search on the master:

```sql
SELECT ItemCode, ItemCodeDesc, ProductLine, ProductType,
       Valuation, StandardUnitPrice, StandardUnitCost
FROM dbo.CI_Item
WHERE ItemCodeDesc LIKE '%BRACKET%'
ORDER BY ItemCode
```

**On-hand / committed / available by warehouse** for one item:

```sql
SELECT WarehouseCode,
       QuantityOnHand,
       QuantityOnSalesOrder + QuantityOnBackOrder AS QuantityCommitted,
       QuantityRequiredForWO,
       QuantityOnPurchaseOrder AS QuantityIncoming,
       QuantityOnHand - QuantityOnSalesOrder - QuantityOnBackOrder
         - QuantityRequiredForWO AS QuantityAvailable
FROM dbo.IM_ItemWarehouse
WHERE ItemCode = 'WIDGET-001'
ORDER BY WarehouseCode
```

**Company-wide stock position** — always an explicit SUM over warehouses,
joined to the master for descriptions:

```sql
SELECT i.ItemCode, i.ItemCodeDesc,
       SUM(w.QuantityOnHand) AS TotalOnHand,
       SUM(w.QuantityOnHand - w.QuantityOnSalesOrder
           - w.QuantityOnBackOrder - w.QuantityRequiredForWO)
         AS TotalAvailable
FROM dbo.CI_Item i
JOIN dbo.IM_ItemWarehouse w ON w.ItemCode = i.ItemCode
WHERE i.ProductType <> 'D'
GROUP BY i.ItemCode, i.ItemCodeDesc
HAVING SUM(w.QuantityOnHand) <> 0
ORDER BY i.ItemCode
```

On the ProvideX path the same join uses the `{ IJ ... }` brace escape and
runs through OPENQUERY (mechanics in connecting-and-setup):

```sql
SELECT i.ItemCode, i.ItemCodeDesc, SUM(w.QuantityOnHand)
FROM { IJ CI_Item i INNER JOIN IM_ItemWarehouse w
       ON i.ItemCode = w.ItemCode }
GROUP BY i.ItemCode, i.ItemCodeDesc
```

**Cost tiers** for a valuation-tracked item — run
`mssql__describe_table` on `IM_ItemCost` first to learn the tier and cost
column names on this site, then select by the known key:

```sql
SELECT *
FROM dbo.IM_ItemCost
WHERE ItemCode = 'WIDGET-001'
  AND WarehouseCode = '000'
```

**Price schedule** for an item — same verify-first approach on
`IM_PriceCode` (`PriceCodeRecord` distinguishes item-level from
customer-level records; break columns vary):

```sql
SELECT *
FROM dbo.IM_PriceCode
WHERE ItemCode = 'WIDGET-001'
```

**Dead stock** — on-hand quantity with no movement in a year:

```sql
WITH LastMove AS (
  SELECT ItemCode, MAX(TransactionDate) AS LastMovement
  FROM dbo.IM_ItemTransactionHistory
  GROUP BY ItemCode
)
SELECT w.ItemCode, m.LastMovement,
       SUM(w.QuantityOnHand) AS TotalOnHand
FROM dbo.IM_ItemWarehouse w
JOIN LastMove m ON m.ItemCode = w.ItemCode
WHERE m.LastMovement < DATEADD(year, -1, GETDATE())
GROUP BY w.ItemCode, m.LastMovement
HAVING SUM(w.QuantityOnHand) > 0
ORDER BY m.LastMovement
```

ProvideX cannot do CTEs or subqueries — run the last-movement aggregate
and the quantity check as two separate single-statement queries and
combine the results yourself, with date filters written as ODBC escape
literals (`{d'2025-08-17'}`), never bare quoted strings.

**What sold recently** — units by item from posted invoice lines:

```sql
SELECT TOP 25 d.ItemCode,
       SUM(d.QuantityShipped) AS UnitsSold,
       SUM(d.ExtensionAmt) AS SalesAmount
FROM dbo.AR_InvoiceHistoryDetail d
JOIN dbo.AR_InvoiceHistoryHeader h
  ON h.InvoiceNo = d.InvoiceNo AND h.HeaderSeqNo = d.HeaderSeqNo
WHERE h.InvoiceDate >= DATEADD(day, -90, GETDATE())
  AND d.ItemCode <> ''
GROUP BY d.ItemCode
ORDER BY UnitsSold DESC
```

This is a quick ranking only — it does not net out credit memos. For real
sales analysis (invoice types, revenue, customers) use sales-and-ar.

**Demand vs supply on open orders** — items whose open sales-order demand
exceeds on-hand stock, with incoming purchase orders alongside:

```sql
WITH Demand AS (
  SELECT d.ItemCode,
         SUM(d.QuantityOrdered - d.QuantityShipped) AS OpenSOQty
  FROM dbo.SO_SalesOrderDetail d
  JOIN dbo.SO_SalesOrderHeader h ON h.SalesOrderNo = d.SalesOrderNo
  WHERE h.OrderStatus IN ('N', 'O', 'H') AND h.OrderType <> 'Q'
  GROUP BY d.ItemCode
),
Incoming AS (
  SELECT ItemCode,
         SUM(QuantityOrdered - QuantityReceived) AS OpenPOQty
  FROM dbo.PO_PurchaseOrderDetail
  GROUP BY ItemCode
),
Stock AS (
  SELECT ItemCode, SUM(QuantityOnHand) AS OnHand
  FROM dbo.IM_ItemWarehouse
  GROUP BY ItemCode
)
SELECT s.ItemCode, s.OnHand, d.OpenSOQty, i.OpenPOQty
FROM Stock s
JOIN Demand d ON d.ItemCode = s.ItemCode
LEFT JOIN Incoming i ON i.ItemCode = s.ItemCode
WHERE d.OpenSOQty > s.OnHand
ORDER BY d.OpenSOQty - s.OnHand DESC
```

## Gotchas

- **The item master is `CI_Item`, not an `IM_` table.** Common
  Information holds the master; `IM_` tables hold quantities, costs,
  and movements keyed off it.
- **Quantities are meaningless without a warehouse.** Every
  `IM_ItemWarehouse` figure is per `ItemCode + WarehouseCode`; a
  company-wide number is always an explicit SUM over warehouses, and
  joining item to warehouse rows without grouping multiplies rows per
  warehouse.
- **"Available" is a convention, not a column.** State the formula you
  used; whether backorders or incoming POs count is site policy.
- **Discontinued items (`ProductType = 'D'`) still carry quantity and
  history.** Filter them in or out deliberately, not by accident.
- **Empty strings, not NULLs.** Filter non-item detail lines with
  `ItemCode <> ''`, never `IS NOT NULL` — Sage 100 data effectively has
  no NULLs.
- **`1753-01-01` is Premium's blank-date sentinel.** Exclude it before
  MIN/MAX date math on `TransactionDate` or anything date-shaped.
- **`IM_ItemCost` and `IM_PriceCode` column names are version-sensitive.**
  Verify with `mssql__describe_table` first; do not guess tier or
  quantity-break columns.
- **History tables need filters.** `IM_ItemTransactionHistory` and
  `AR_InvoiceHistoryDetail` grow to millions of rows — always filter by
  item, warehouse, or date, aggregate in SQL, and remember the 1000-row
  cap flags `"truncated": true` rather than failing.
- **ProvideX dialect is narrower.** Single statement, `{ IJ ... }` for
  joins, `{d'YYYY-MM-DD'}` date literals, no subqueries or UNION — split
  CTE-shaped work into multiple calls. Bridge mechanics live in
  connecting-and-setup; do not improvise them.

## Related Skills

- schema-and-conventions — table naming, composite keys, data-type
  quirks, and the shared `table-reference.md` map
- connecting-and-setup — edition detection, connector configuration, and
  the OPENQUERY bridge for ProvideX sites
- sales-and-ar — customers, invoices, and the full sales-analysis
  treatment of what sold
- purchasing-and-ap — vendors, payables, and purchase-order detail
- gl-and-financials — chart of accounts, trial balance, and GL detail
