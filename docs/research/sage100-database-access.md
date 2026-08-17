# Sage 100 database access — research backbone

Verified 2026-08-17. This document is the factual source for the `sage100` plugin's
skills. Skills must not assert facts that are not in this document (or the
companion `conduit-connector-contract.md`) without flagging them as unverified.

## 1. Editions and storage backends

| Edition | Lineage | Storage | External access |
|---|---|---|---|
| **Standard** (fka MAS 90) | ProvideX (PVX) file-based ISAM; `.M4T` files opened directly over a file share | Sage 100 ODBC driver ("MAS 90 4.0 ODBC Driver", SOTAMAS90 DSN), **read-only** |
| **Advanced** (fka MAS 200) | Same ProvideX files, client/server application engine on the host | Same ODBC driver; optional **Client/Server ODBC driver** (`pvxiosvr`) executes reads server-side |
| **Premium** | Business Framework on **Microsoft SQL Server** | Native SQL Server (any SQL client), `dbo` schema |

- "M4T" = MAS version-4 tables (4.x Business Framework); `AR_Customer.M4T` is exposed as table `AR_Customer`.
- Naming: Sage 100c → "Sage 100cloud" (2018) → plain **"Sage 100"** again effective April 2024 ("Sage 100 SPC" for Sage Partner Cloud hosting). Never a SaaS product — same on-prem code.
- Versions: 2022 = 7.10, 2023 = 7.20, 2024 = 7.30; Sage 100 2025 shipped April 2025. Support = current + two prior versions.
- **Sage 100 2025 installs both 32-bit and 64-bit ODBC drivers by default**; 2021–2024 offered 64-bit as an optional install (Sage KB 34167).

## 2. ODBC access (Standard/Advanced)

**DSN and driver**
- Install creates a **User DSN `SOTAMAS90`** on the **"MAS 90 4.0 ODBC Driver"**. SOTAMAS90 is reserved for Sage 100's own use (Crystal rendering) — integrations must **create their own DSN** (System DSN for services/linked servers). (Sage KB 19495, KB 36360.)
- 32-bit is the default driver through v2024; manage 32-bit DSNs via `C:\Windows\SysWOW64\odbcad32.exe`.

**Authentication — three-part login**
- Company Code (3 chars, uppercase, e.g. `ABC`) + Sage 100 user + password. A connection is **scoped to exactly one company**; another company = another DSN/connection with a different `Company=`.
- Role Maintenance can restrict ODBC table/field access per user.

**Silent (UI-less) connections**
- Without stored credentials the driver pops the Sage logon dialog (fatal for services). A "silent connection" = a custom DSN with Company/User/Password pre-filled on the Logon tab. **There is no literal `SILENT=` connection-string keyword — that is folklore**; the mechanism is stored DSN credentials (Sage KB 19495).
- DSN-less strings work: `DSN=MyDSN;UID=USER;PWD=pass;Company=ABC;` or driver-direct with `Driver={MAS 90 4.0 ODBC Driver}`, `Directory=...\MAS90`, `Prefix=...\MAS90\SY\, ...\MAS90\==\`, `ViewDLL=...\MAS90\HOME`, `Company=ABC`, `DirtyReads=1`, `BurstMode=1`, `StripTrailingSpaces=1`. The exact provider string for an install is shown on the DSN's **Debug tab**.

**Read-only**
- **All Sage-shipped ODBC drivers are read-only.** Writes must go through the Business Object Interface (BOI). A generic PVX read-write driver exists from PVX Plus; partners report it corrupts Sage 100 data — never write via ODBC.

**Performance and consistency**
- Standard (and Advanced without C/S ODBC) has **no server engine**: non-keyed filters pull whole files over the network. Keyed WHERE/ORDER BY uses index access; everything else scans.
- Advanced's Client/Server ODBC service executes reads host-side — markedly faster.
- DSN defaults `DirtyReads=1` (may miss in-flight updates) and `BurstMode=1` (short file locks during fetches). Disable for consistency-sensitive interactive use; leave on for nightly ETL.

**Bridging to SQL Server (the pattern Conduit uses)**
- **Linked server**: officially documented by Sage — provider **MSDASQL** over a System DSN on the MAS 90 driver; query via `SELECT * FROM OPENQUERY(SAGE100, 'SELECT ... FROM GL_Account')`. Sage support does not assist with linked-server config (KB 36360).
- **Bitness trap**: 64-bit SQL Server loads only 64-bit ODBC drivers through MSDASQL. Options: Sage's 64-bit driver (2021+; default 2025), driver-direct provider string, or a legacy 32-bit SQL Express bridge instance.
- Scheduled ETL/replication to a SQL mirror is the dominant production BI pattern (DataSelf ETL+, ROI IN-SYNCH real-time mirroring + REST "Connector API").

## 3. Premium / MSSQL specifics

- **One database per company: `MAS_XXX`** (XXX = company code, e.g. `MAS_ABC`) plus **`MAS_SYSTEM`** (users, companies, roles). On Standard/Advanced the same names exist as folders: `..\MAS90\MAS_ABC\*.M4T`.
- Schema `dbo`; **same logical table names as the ODBC view** (`dbo.AR_Customer`), so reporting SQL ports between editions with dialect adjustments only.
- External tools use the standard SQL Server driver, not SOTAMAS90.
- **Date columns are `datetime NOT NULL`; empty dates materialize as `1753-01-01`** — filter the sentinel when porting queries.
- Cross-company queries possible on Premium (`MAS_ABC.dbo.AR_Customer` vs `MAS_XYZ.dbo.AR_Customer`); impossible in one ProvideX connection.

## 4. Table map (verified against the official File Layouts and Object Reference)

Naming = `<ModulePrefix>_<Entity>[Header|Detail|History...]`. Prefixes: GL, AR, AP, SO, PO, IM (Inventory), CI (Common Information), SY (system), BM (Bill of Materials), JC/JT (Job Cost), PR (Payroll), RA (RMA).

**Critical structural fact: tables have NO CompanyCode column.** Company separation is physical (folder per company / database per company). Any source claiming "join on CompanyCode" is wrong for Sage 100.

| Table | Purpose | Primary key / join notes |
|---|---|---|
| `AR_Customer` | Customer master | **ARDivisionNo + CustomerNo** (composite) |
| `AR_OpenInvoice` | Open (unpaid) AR invoices | **ARDivisionNo + CustomerNo + InvoiceNo + InvoiceType**; `Balance` (14.2), `InvoiceDueDate`; InvoiceType ∈ IN, CM, DM, FC, PP, PY, BC, BF |
| `AR_InvoiceHistoryHeader` | Posted AR/SO invoice history | **InvoiceNo + HeaderSeqNo**; ARDivisionNo, CustomerNo, InvoiceDate, InvoiceType (IN, CM, DM, AD, FC, CA, XD), TaxableSalesAmt, NonTaxableSalesAmt, FreightAmt, SalesTaxAmt |
| `AR_InvoiceHistoryDetail` | Invoice history lines | Join **InvoiceNo + HeaderSeqNo** (+ DetailSeqNo); ItemCode, QuantityShipped, UnitPrice, ExtensionAmt |
| `AR_TransactionPaymentHistory` | AR payments/applications | ARDivisionNo + CustomerNo + InvoiceNo |
| `AR_CashReceiptsHistory` | Cash receipt history | — |
| `AP_Vendor` | Vendor master | **APDivisionNo + VendorNo** (composite) |
| `AP_OpenInvoice` | Open AP invoices | **APDivisionNo + VendorNo + InvoiceNo**; `Balance` (13.2), `InvoiceDueDate`, InvoiceAmt, DiscountAmt |
| `AP_InvoiceHistoryHeader`/`Detail` | Posted AP invoice history | APDivisionNo + VendorNo + InvoiceNo (+ header seq) |
| `AP_CheckHistoryHeader`/`Detail` | AP payment history | CheckNo/BankCode |
| `GL_Account` | Chart of accounts | **AccountKey** (9-digit surrogate) is the join key everywhere; `Account` is the formatted display number |
| `GL_DetailPosting` | GL transaction detail | **AccountKey + PostingDate + SourceJournal + JournalRegisterNo + SequenceNo**; DebitAmount/CreditAmount (16.2), SourceModule |
| `GL_PeriodPostingHistory` | GL period summary (trial balance source) | **AccountKey + FiscalYear + FiscalPeriod**; BeginningBalance (period 1 only), DebitAmount, CreditAmount; ending = beginning + debits − credits |
| `SO_SalesOrderHeader` | Open sales orders | **SalesOrderNo**; OrderStatus: N=New, O=Open, C=Closed, H=Hold; OrderType: S/B/M/R/Q/P; ARDivisionNo+CustomerNo → AR_Customer |
| `SO_SalesOrderDetail` | Order lines | Join **SalesOrderNo**; ItemCode, QuantityOrdered, QuantityShipped, UnitPrice, ExtensionAmt |
| `SO_InvoiceHeader`/`Detail` | Unposted (in-batch) SO invoices | InvoiceNo; union with AR history for "all invoices" |
| `SO_SalesOrderHistoryHeader` | All orders incl. closed/deleted | SalesOrderNo |
| `PO_PurchaseOrderHeader`/`Detail` | Open POs | **PurchaseOrderNo**; APDivisionNo+VendorNo |
| `PO_ReceiptHistoryHeader`/`Detail` | PO receipt history | ReceiptNo |
| `CI_Item` | Item master (all item types) | **ItemCode** (30 chars); ProductLine, ProductType (F/R/D/K), ItemType, Valuation, StandardUnitPrice/Cost (16.6) |
| `IM_ItemWarehouse` | Per-warehouse quantities | **ItemCode + WarehouseCode** (composite); QuantityOnHand/OnSalesOrder/OnPurchaseOrder/OnBackOrder/OnWorkOrder/RequiredForWO/InShipping — 16.6, maintained totals |
| `IM_ItemCost` | Cost tiers (FIFO/LIFO/lot/serial) | ItemCode + WarehouseCode + tier |
| `IM_ItemTransactionHistory` | Inventory movements | ItemCode + WarehouseCode + TransactionDate |
| `IM_PriceCode` | Price schedules / price lists | PriceCodeRecord + ItemCode/PriceCode + CustomerPriceLevel |
| `SY_Company` (MAS_SYSTEM) | Company list | CompanyCode — enumerate companies here |

**Composite-key gotchas** (the classic wrong-join bugs): `CustomerNo` is unique only within `ARDivisionNo`; `VendorNo` within `APDivisionNo`; quantities within `ItemCode + WarehouseCode`; AR invoice history within `InvoiceNo + HeaderSeqNo`. With divisions disabled the division is literally `'00'` and still participates in keys. GL joins must use `AccountKey`, never the formatted `Account` string.

## 5. Data-type quirks

- **Dates (PVX/ODBC)**: stored as 8-char `YYYYMMDD` strings but dictionary-typed DATE; the driver requires ODBC escape literals: `WHERE InvoiceDate >= {d'2026-01-01'}` — plain `'2026-01-01'` does not work. Bad/blank legacy dates can crash extracts; the DSN "NULL Date" option returns invalid dates as NULL, or `{fn CONVERT(OrderDate, SQL_VARCHAR)}` pulls raw. **Premium**: `datetime NOT NULL`, blank = `1753-01-01` sentinel.
- **TimeCreated/TimeUpdated/TimeEntered**: decimal fraction-of-hour on a 24h clock, returned as string — `15.5` = 3:30 PM, `8.49238` ≈ 8:29:32 AM. hours = floor(v); minutes = (v − floor(v)) × 60.
- **Numerics**: dictionary precisions 13.2/14.2/16.2 (amounts), 16.6 (quantities, unit costs/prices); signed.
- **NULLs**: ProvideX data effectively has none — empty strings and zeros instead (`WHERE X <> ''`, not `IS NOT NULL`). Premium: non-null empty strings; 1753 date sentinel.
- **Trailing spaces**: CHAR-style fields pad; `StripTrailingSpaces=1` handles it.
- Key fields are case-sensitive; company code and user ID uppercase.

## 6. Schema discovery

- PVX ODBC answers `SQLTables`/`SQLColumns` catalog functions (driven by the data dictionaries at the DSN's `Prefix`). Premium: `INFORMATION_SCHEMA.TABLES/COLUMNS`.
- In-product: Resources (pre-2025) / Help tab (2025+) → "File Layouts and Program Information"; also `..\MAS90\Home\FileLayouts.chm`.
- Online FLOR: `https://help-sage100.na.sage.com/<year>/FLOR/index.htm` (verified 2018–2024); direct pages `https://help-sage100.na.sage.com/2018/FLOR/Content/File_Layouts/<Module>/<Table>.htm` (e.g. `Sales_Order/SO_SalesOrderHeader.htm`) — fetch without login.
- `SY_ReportMaster` is report definitions, not a schema catalog.

## 7. ProvideX SQL dialect ground rules

Supported (per PxPlus docs): SELECT with WHERE / GROUP BY / HAVING / ORDER BY / DISTINCT, aggregates (COUNT/SUM/AVG/MIN/MAX), INNER/LEFT/RIGHT joins incl. nesting via the **`{ IJ ... }` brace escape**, `TOP n` / `LIMIT n [OFFSET m]`, `{fn ...}` scalar functions, `{d'YYYY-MM-DD'}` literals, CASE/CAST/IFNULL.

**However, Sage bundles an old build**: UNION / subqueries / FULL OUTER JOIN arrived in PxPlus 2016/2018 and are unreliable or absent in the bundled "MAS 90 4.0" driver. Practitioners report failures with OR-heavy predicates, semicolons, square brackets, and more than 1–2 joins per statement.

**Skill guidance**: keep PVX queries to single-purpose SELECTs, one or two joins inside `{ IJ ... }`, no subqueries/UNION, treat `TOP`/`LIMIT` as verify-at-runtime, and treat T-SQL features as Premium-only.

## 8. Canonical queries (validated forms)

See the query skills for the full set. Canonical patterns confirmed by research:

- Open AR: `SELECT ... FROM AR_OpenInvoice WHERE Balance <> 0` (watch PP/PY negative credit rows).
- Sales by customer: SUM(TaxableSalesAmt + NonTaxableSalesAmt) from `AR_InvoiceHistoryHeader`, treating CM as negative.
- Sales by item: `{ IJ AR_InvoiceHistoryHeader H INNER JOIN AR_InvoiceHistoryDetail D ON H.InvoiceNo = D.InvoiceNo AND H.HeaderSeqNo = D.HeaderSeqNo }` (PVX) / plain JOIN (Premium).
- Open orders: `SO_SalesOrderHeader.OrderStatus IN ('N','O','H') AND OrderType <> 'Q'`.
- Inventory availability: `QuantityOnHand − QuantityOnSalesOrder − QuantityOnBackOrder − QuantityRequiredForWO (+ QuantityOnPurchaseOrder if incoming counts)` from `IM_ItemWarehouse`.
- Trial balance: `GL_PeriodPostingHistory` joined to `GL_Account` on `AccountKey`; ending balance = period-1 BeginningBalance + Σ debits − Σ credits across periods ≤ target.
- AP aging: `AP_OpenInvoice` joined to `AP_Vendor` on APDivisionNo + VendorNo.

## 9. Adjacent approaches (positioning)

- **eBusiness Web Services** (SOAP/WCF, IIS): official web API, ships in 2024/2025; narrow scope (Sales Orders, Customers, Contacts, card vaulting). No GL/AP/inventory reporting surface.
- **Business Object Interface (BOI)**: the official COM read/write API; the only supported write path.
- **No native REST API** from Sage for Sage 100; REST comes from third parties (ROI IN-SYNCH Connector API, middleware).
- Microsoft 365 Connector retired with Sage 100 2025.0.
- **No official Sage MCP server for Sage 100 exists as of Aug 2026** — the durable, edition-spanning access path is SQL/ODBC plus schema knowledge, which is exactly what this plugin curates.

## 10. Flagged as unverified

1. Any literal `SILENT` connection-string keyword (folklore; mechanism is stored DSN credentials).
2. Exact 64-bit driver display-name string.
3. `TOP`/`LIMIT`, UNION, subquery behavior on the specific bundled driver build — probe at runtime, degrade to client-side.
4. AR_Customer aging-bucket column names by version; `ExtensionAmt` presence in AR_InvoiceHistoryDetail (widely used, not re-checked in FLOR).
5. StarShip/BizNet ODBC bridging depth; "2025 = v7.40" numbering; 2026 release specifics.

Primary sources: Sage KB 19495 (silent ODBC), KB 36360 (linked server), KB 34167 (64-bit ODBC), KB 19912 (File Layouts access); official FLOR pages under `help-sage100.na.sage.com/<year>/FLOR/Content/File_Layouts/`; PxPlus ODBC manual (`manual.pvxplus.com`); DataSelf KB; Schulz Consulting; Sage Community Hub threads.
