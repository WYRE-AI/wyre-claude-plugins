---
name: "Sage 100 GL and Financials"
description: >
  General Ledger reporting in Sage 100: the chart of accounts and its
  AccountKey surrogate, trial balances and ending balances from period
  posting history, transaction-level drill-down into GL detail postings,
  source-journal analysis, and reconciling period summaries against summed
  detail — all through Conduit's read-only SQL connector.
when_to_use: >-
  When the user asks for a trial balance, an account balance, GL activity
  for a period, journal or register detail, or any financial-statement-
  shaped number from Sage 100, or when a GL figure needs to be traced back
  to the postings behind it. Use when: trial balance, general ledger, GL,
  chart of accounts, account balance, GL_Account, GL_DetailPosting,
  GL_PeriodPostingHistory, fiscal period, source journal, journal register,
  debits and credits, drill down.
---

# Sage 100 GL and Financials

## Overview

Three tables carry the entire General Ledger story. `GL_Account` is the
chart of accounts. `GL_PeriodPostingHistory` holds one summary row per
account per fiscal period — the trial-balance source. `GL_DetailPosting`
holds every individual posting — the drill-down source, and usually the
largest table at a site. Their keys and columns are mapped in
`table-reference.md` next to the schema-and-conventions skill.

Queries run through Conduit's generic SQL connector: `mssql__query` for
reading, `mssql__describe_table` for column discovery (the `mssql__`
prefix follows the connector slug the org configured; a named instance may
surface as `sage100__query`). Access is read-only — one SELECT or WITH
statement per call, 100 rows by default, 1000 max.

Examples below are Premium T-SQL against `dbo`. Standard/Advanced sites
reach the same tables through a linked-server bridge in the ProvideX
dialect — the differences are noted under API Patterns, and the bridge
mechanics (OPENQUERY, DSNs) live in the connecting-and-setup skill.

## Key Concepts

**AccountKey is the only GL join key.** `GL_Account` carries a 9-digit
surrogate `AccountKey`, the formatted display number `Account`, and the
name `AccountDesc`. Every GL join — to period history, to detail postings
— uses `AccountKey`. Never join on the formatted `Account` string; it is
for display and lookup only. Those three columns are the verified surface
of `GL_Account`; anything further (account type, category, or grouping
columns for building a P&L versus a balance sheet) varies by version —
discover with `mssql__describe_table` on `GL_Account` first.

**Period arithmetic.** `GL_PeriodPostingHistory` is keyed
`AccountKey + FiscalYear + FiscalPeriod`. Each row carries the period's
`DebitAmount` and `CreditAmount`; `BeginningBalance` is populated only on
the period-1 row, where it holds the account's balance entering the year.
So for any target period:

    ending balance = period-1 BeginningBalance
                   + sum of DebitAmount over periods <= target
                   - sum of CreditAmount over periods <= target

**FiscalYear and FiscalPeriod are strings.** Quote them: `'2026'`, `'06'`
— not `2026` or `6`. Periods are zero-padded two-character values, so a
lexicographic `<= '06'` compares the same as numeric and the ending-
balance aggregation works with plain string comparison.

**Detail postings are keyed for filtered access.** `GL_DetailPosting` is
keyed `AccountKey + PostingDate + SourceJournal + JournalRegisterNo +
SequenceNo`. Always filter by a `PostingDate` range and/or `AccountKey` —
never scan it open-ended. `SourceModule` and `SourceJournal` identify
where a posting came from (which module, which journal run), and
`JournalRegisterNo` ties postings to their register; `PostingComment` is
the human-readable line description.

**AccountKey values are quoted in the examples.** The surrogate is a
9-digit key and the examples treat it as a string (`'004100000'`). If a
comparison misbehaves on a specific site, confirm the column type with
`mssql__describe_table` on `GL_Account`.

## Common Workflows

**Orient in the chart of accounts.** Find the accounts in play before
touching balances — this also captures the `AccountKey` values the other
queries need:

```sql
SELECT AccountKey, Account, AccountDesc
FROM dbo.GL_Account
WHERE AccountDesc LIKE '%cash%'
ORDER BY Account
```

A full chart can exceed the row cap; list it in keyed pages
(`WHERE AccountKey > 'lastseen' ORDER BY AccountKey`) rather than one
bulk pull.

**Trial balance — single-period snapshot.** One row per account for one
period, straight from the summary table:

```sql
SELECT a.Account, a.AccountDesc,
       p.BeginningBalance, p.DebitAmount, p.CreditAmount
FROM dbo.GL_PeriodPostingHistory p
INNER JOIN dbo.GL_Account a ON a.AccountKey = p.AccountKey
WHERE p.FiscalYear = '2026'
  AND p.FiscalPeriod = '06'
ORDER BY a.Account
```

Expect `BeginningBalance` to be zero here for any period other than
`'01'` — that column only carries the year-opening balance on the
period-1 row.

**Trial balance — ending balance through a period.** Aggregate all
periods up to and including the target. The period-1 row is the only one
with a nonzero `BeginningBalance`, so summing it across periods
contributes the opening balance exactly once:

```sql
SELECT a.Account, a.AccountDesc,
       SUM(p.BeginningBalance)
         + SUM(p.DebitAmount) - SUM(p.CreditAmount) AS EndingBalance
FROM dbo.GL_PeriodPostingHistory p
INNER JOIN dbo.GL_Account a ON a.AccountKey = p.AccountKey
WHERE p.FiscalYear = '2026'
  AND p.FiscalPeriod <= '06'
GROUP BY a.Account, a.AccountDesc
ORDER BY a.Account
```

**Profile a year's activity.** Total debits and credits per period across
the whole ledger — a quick health check, since a balanced ledger posts
equal debits and credits each period:

```sql
SELECT FiscalPeriod,
       SUM(DebitAmount) AS TotalDebits,
       SUM(CreditAmount) AS TotalCredits
FROM dbo.GL_PeriodPostingHistory
WHERE FiscalYear = '2026'
GROUP BY FiscalPeriod
ORDER BY FiscalPeriod
```

**Drill into one account's activity.** The detail behind a balance, always
bounded by account and date range:

```sql
SELECT PostingDate, SourceModule, SourceJournal, JournalRegisterNo,
       DebitAmount, CreditAmount, PostingComment
FROM dbo.GL_DetailPosting
WHERE AccountKey = '004100000'
  AND PostingDate >= '2026-06-01'
  AND PostingDate < '2026-07-01'
ORDER BY PostingDate, JournalRegisterNo
```

A busy account can exceed the row cap even inside one month — page by
advancing the `PostingDate` lower bound past the last row returned, or
narrow to one `SourceJournal`.

**Analyze activity by source journal.** Where did a period's postings come
from — sales, payables, cash receipts, manual entries:

```sql
SELECT SourceModule, SourceJournal,
       COUNT(*) AS PostingCount,
       SUM(DebitAmount) AS TotalDebits,
       SUM(CreditAmount) AS TotalCredits
FROM dbo.GL_DetailPosting
WHERE PostingDate >= '2026-06-01'
  AND PostingDate < '2026-07-01'
GROUP BY SourceModule, SourceJournal
ORDER BY TotalDebits DESC
```

**Reconcile summary against detail.** Cross-check that a period's summary
row agrees with the summed detail postings for the same account —
the standard "is this balance real" verification:

```sql
WITH detail AS (
  SELECT SUM(DebitAmount) AS DetailDebits,
         SUM(CreditAmount) AS DetailCredits
  FROM dbo.GL_DetailPosting
  WHERE AccountKey = '004100000'
    AND PostingDate >= '2026-06-01'
    AND PostingDate < '2026-07-01'
)
SELECT p.DebitAmount AS SummaryDebits,
       d.DetailDebits,
       p.CreditAmount AS SummaryCredits,
       d.DetailCredits
FROM dbo.GL_PeriodPostingHistory p
CROSS JOIN detail d
WHERE p.AccountKey = '004100000'
  AND p.FiscalYear = '2026'
  AND p.FiscalPeriod = '06'
```

The date range must match the site's fiscal calendar — fiscal periods are
not guaranteed to be calendar months. Confirm the period boundaries with
the customer before treating a mismatch as missing data.

## API Patterns

The tools this skill is written against:

- `mssql__query` — one read-only SELECT or WITH statement per call;
  `max_rows` up to 1000; results carry `truncated: true` when clipped
- `mssql__describe_table` — verify `GL_Account` extras (account type or
  category columns) before using them; works on Premium, not through a
  linked server

Respect the caps by shape: aggregate in SQL (the trial-balance and
source-journal queries return one row per account or journal, not one per
posting), filter before you fetch, and page big extracts by a keyed range
rather than raising `max_rows` and hoping.

**ProvideX (Standard/Advanced) differences.** The same logic runs through
the bridge's OPENQUERY with the inner query in ProvideX dialect: date
literals use the ODBC escape form, table names are unqualified (no
`dbo.`), and the statement must stay simple — single statement, no
subqueries, no UNION, joins written inside `{ IJ ... }` brace escapes.
The drill-down above becomes:

```sql
SELECT PostingDate, SourceModule, SourceJournal, JournalRegisterNo,
       DebitAmount, CreditAmount, PostingComment
FROM GL_DetailPosting
WHERE AccountKey = '004100000'
  AND PostingDate >= {d'2026-06-01'}
  AND PostingDate <= {d'2026-06-30'}
```

The WITH-based reconciliation cannot be expressed on this path — run the
summary query and the detail sum as two separate calls and compare the
results yourself. Bridge setup and OPENQUERY quoting rules are covered in
connecting-and-setup; do not improvise them here.

## Gotchas

- **Joining GL tables on `Account`** (the formatted string) instead of
  `AccountKey` is the classic GL bug — it can appear to work on one site
  and silently miss rows on another. `AccountKey` only, always.
- **Unquoted fiscal filters**: `FiscalYear = 2026` or `FiscalPeriod = 6`
  compares a string column against a number. Quote and zero-pad:
  `'2026'`, `'06'`.
- **`BeginningBalance` is only on period 1.** A period-6 snapshot showing
  zero beginning balances is normal; compute running balances with the
  aggregation pattern, not by reading `BeginningBalance` per period.
- **Credit-normal accounts come out negative.** Ending balance is
  beginning + debits − credits, so revenue and liability accounts show
  negative values under this formula. That is the arithmetic sign
  convention, not bad data.
- **`GL_DetailPosting` is often the largest table at a site** — millions
  of rows at a mature installation. Never query it without a
  `PostingDate` range and/or `AccountKey` filter, and prefer
  `GL_PeriodPostingHistory` whenever period totals are enough.
- **`1753-01-01` is Premium's blank-date sentinel.** Exclude it before
  taking MIN(PostingDate) or doing date arithmetic.
- **Account type and category columns are unverified.** Building a P&L
  or balance sheet needs a way to classify accounts; do not guess column
  names — discover what the site's `GL_Account` actually has with
  `mssql__describe_table` first.
- **Fiscal periods are not calendar months everywhere.** Reconciliation
  between period summaries and date-filtered detail only balances when
  the date range matches the site's fiscal calendar.

## Related Skills

- schema-and-conventions — table naming, composite keys, data-type
  quirks, and the shared `table-reference.md` map
- connecting-and-setup — edition detection, connector configuration, the
  ProvideX linked-server bridge, and OPENQUERY dialect rules
- sales-and-ar — customers, invoices, receivables, sales analysis
- purchasing-and-ap — vendors, payables, purchase orders
- inventory-and-items — items, warehouses, quantities, pricing
