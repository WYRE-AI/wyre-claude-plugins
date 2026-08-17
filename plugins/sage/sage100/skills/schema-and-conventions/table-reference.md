# Sage 100 table reference

Curated map of the tables that answer most business questions, with the keys
that make joins correct. Names are identical on Premium (`dbo.` schema) and
the ProvideX/ODBC path. Verified against Sage's official File Layouts and
Object Reference; when in doubt on a specific site, confirm columns with
`mssql__describe_table` (Premium) or a one-row probe query.

## Accounts Receivable / Sales

| Table | Purpose | Keys and joins |
|---|---|---|
| `AR_Customer` | Customer master | PK **ARDivisionNo + CustomerNo**. Name, address, phone, terms, salesperson. |
| `AR_OpenInvoice` | Open (unpaid) invoices | PK **ARDivisionNo + CustomerNo + InvoiceNo + InvoiceType**. `Balance`, `InvoiceDate`, `InvoiceDueDate`. InvoiceType: IN invoice, CM credit memo, DM debit memo, FC finance charge, PP prepayment, PY payment, BC/BF balance rows. PP/PY carry negative balances. |
| `AR_InvoiceHistoryHeader` | Posted invoice history (AR and Sales Order invoices) | PK **InvoiceNo + HeaderSeqNo**. ARDivisionNo, CustomerNo, InvoiceDate, InvoiceType (IN, CM, DM, AD, FC, CA, XD), TaxableSalesAmt, NonTaxableSalesAmt, FreightAmt, SalesTaxAmt. |
| `AR_InvoiceHistoryDetail` | Posted invoice lines | Join **InvoiceNo + HeaderSeqNo** (+ DetailSeqNo). ItemCode, QuantityShipped, UnitPrice, ExtensionAmt. |
| `AR_TransactionPaymentHistory` | Payments applied to invoices | ARDivisionNo + CustomerNo + InvoiceNo. |
| `AR_CashReceiptsHistory` | Cash receipts | Deposit/receipt oriented; join to customer via division + number. |

## Sales Orders

| Table | Purpose | Keys and joins |
|---|---|---|
| `SO_SalesOrderHeader` | Open sales orders | PK **SalesOrderNo**. OrderStatus: N new, O open, C closed, H hold. OrderType: S standard, B backorder, M master, R repeating, Q quote, P price quote. ARDivisionNo + CustomerNo → `AR_Customer`. |
| `SO_SalesOrderDetail` | Order lines | Join **SalesOrderNo**. ItemCode, QuantityOrdered, QuantityShipped, UnitPrice, ExtensionAmt. |
| `SO_InvoiceHeader` / `SO_InvoiceDetail` | In-batch (not yet posted) SO invoices | InvoiceNo. Union with AR history for an "all invoices" view. |
| `SO_SalesOrderHistoryHeader` | All orders ever, incl. closed/deleted | SalesOrderNo. |

## Accounts Payable / Purchasing

| Table | Purpose | Keys and joins |
|---|---|---|
| `AP_Vendor` | Vendor master | PK **APDivisionNo + VendorNo**. |
| `AP_OpenInvoice` | Open AP invoices | PK **APDivisionNo + VendorNo + InvoiceNo**. `Balance`, `InvoiceDate`, `InvoiceDueDate`, InvoiceAmt, DiscountAmt. |
| `AP_InvoiceHistoryHeader` / `Detail` | Posted AP invoices | APDivisionNo + VendorNo + InvoiceNo (+ header sequence). |
| `AP_CheckHistoryHeader` / `Detail` | Payment (check/EFT) history | BankCode + CheckNo; details reference invoices paid. |
| `PO_PurchaseOrderHeader` | Open purchase orders | PK **PurchaseOrderNo**. APDivisionNo + VendorNo → `AP_Vendor`. |
| `PO_PurchaseOrderDetail` | PO lines | Join **PurchaseOrderNo**. ItemCode, QuantityOrdered, QuantityReceived, UnitCost. |
| `PO_ReceiptHistoryHeader` / `Detail` | Receipt of goods history | ReceiptNo. |

## General Ledger

| Table | Purpose | Keys and joins |
|---|---|---|
| `GL_Account` | Chart of accounts | PK **AccountKey** (9-digit surrogate — the universal GL join key). `Account` is the formatted display number; `AccountDesc` the name. |
| `GL_DetailPosting` | Every GL posting | PK **AccountKey + PostingDate + SourceJournal + JournalRegisterNo + SequenceNo**. DebitAmount, CreditAmount, PostingComment, SourceModule. Large — always filter by date or account. |
| `GL_PeriodPostingHistory` | Period summaries (trial balance source) | PK **AccountKey + FiscalYear + FiscalPeriod**. BeginningBalance (populated on period 1), DebitAmount, CreditAmount. Ending balance = beginning + Σ debits − Σ credits across periods ≤ target. |

## Inventory / Items

| Table | Purpose | Keys and joins |
|---|---|---|
| `CI_Item` | Item master (all item types) | PK **ItemCode** (30 chars). ItemCodeDesc, ProductLine, ProductType (F finished, R raw, D discontinued, K kit), Valuation, StandardUnitPrice, StandardUnitCost. |
| `IM_ItemWarehouse` | Per-warehouse quantities | PK **ItemCode + WarehouseCode**. QuantityOnHand, QuantityOnSalesOrder, QuantityOnBackOrder, QuantityOnPurchaseOrder, QuantityOnWorkOrder, QuantityRequiredForWO, QuantityInShipping — maintained running totals. |
| `IM_ItemCost` | Cost tiers (FIFO/LIFO/lot/serial) | ItemCode + WarehouseCode + tier fields. |
| `IM_ItemTransactionHistory` | Inventory movements | ItemCode + WarehouseCode + TransactionDate. |
| `IM_PriceCode` | Price schedules | PriceCodeRecord type + ItemCode/PriceCode + CustomerPriceLevel; quantity break columns. |

## System

| Table | Purpose | Keys and joins |
|---|---|---|
| `SY_Company` | Company list | CompanyCode. Lives in `MAS_SYSTEM`, not the company database. |

## Cross-cutting key rules

- AR joins carry **ARDivisionNo + CustomerNo** end to end; AP joins carry
  **APDivisionNo + VendorNo**. Divisions disabled ⇒ the column holds `'00'`
  but is still part of the key.
- AR invoice history joins on **InvoiceNo + HeaderSeqNo** — InvoiceNo alone
  duplicates rows when an invoice number was reused across sequences.
- Inventory quantity questions are per **ItemCode + WarehouseCode**; sum
  over warehouses for a company-wide figure.
- GL joins use **AccountKey** only.
- There is **no CompanyCode column anywhere** — company = database (Premium)
  or file folder (ProvideX). Cross-company reporting means querying each
  company's connector instance and combining results client-side.
