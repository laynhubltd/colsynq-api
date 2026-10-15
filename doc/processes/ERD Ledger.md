# ERD: Ledger

Oct 11, 2026 · @Abubakar

## 1. The idea

This model follows "The ledger explained, end to end" directly, and replaces the earlier, more fragmented version.

**One rule.** Anything that owns or owes money has accounts. That includes:

- the society;
- each member;
- each non-member staff member;
- each business unit;
- each supplier, employer and driver.

Every account is one of the five types: asset, liability, equity, income or expense. Every money event is a journal. A journal is a set of entries, each debiting or crediting one account, and the debits always equal the credits.

**What that gives.**

- **Members.** Each member's savings, shares, loans and credit plans are their own accounts. A member statement is simply the entries on that member's accounts.
- **Business units.** Each unit has its own bank, stock, sales, cost and expense accounts. A unit's profit and loss is simply its own income and expense accounts.
- **Control totals.** "Member savings" is a parent account whose balance is the sum of every member's savings account. It can never disagree with them.

&#91;embedded content: the ledger · 8 tables, with owners from the business model\]

## 2. The eight tables

| Table | Columns | What it holds |
| --- | --- | --- |
| `ledger_books` | id, society\_id, unit (naira in kobo, or stock in units), last\_journal\_number, last\_hash | One naira book and one stock book per society. Everything below belongs to one book |
| `account_types` | code (A, L, E, I, X), name, normal\_side (debit or credit), statement (balance sheet, or profit and loss) | Exactly five fixed rows: asset, liability, equity, income, expense |
| `accounts` | id, book\_id, account\_type, code, name, owner\_type, owner\_id, parent\_account\_id, is\_parent, allow\_negative, debit\_posted, credit\_posted, debit\_pending, credit\_pending, status (active, no\_debit, closed), opened\_at | One row for every account of every owner. A parent account takes no entries and sums its children |
| `journals` | id, book\_id, period\_id, number, journal\_date, posted\_at, description, kind (event, manual, accrual, correction, closing, opening), source\_type, source\_id, status (draft, approved, posted, rejected, failed), prepared\_by, approved\_by, reverses\_journal\_id, idempotency\_key, previous\_hash, hash | One row per money event. The source points to the business record that caused it |
| `journal_entries` | id, journal\_id, account\_id, debit\_kobo, credit\_kobo, status (pending, posted, released), expires\_at, value\_date, memo | The debits and credits. Exactly one of debit or credit is above zero |
| `periods` | id, book\_id, starts\_on, ends\_on, status (open, soft\_locked, closed), closed\_by, closed\_at | Months, plus a year-end adjustments period |
| `account_balances` | account\_id, period\_id, opening\_kobo, debits\_kobo, credits\_kobo, closing\_kobo | Each account's movement per period. The trial balance and statements are read from here |
| `statement_lines` | id, bank\_account\_id, value\_date, amount\_kobo, direction, reference, matched\_entry\_id, matched\_by, checked\_by, status (unmatched, matched, break) | Lines from the bank statement, each matched to an entry on that bank account, or left as a break |

**Owners** are the business tables, not ledger tables: `societies`, `persons`, `business_units`, `suppliers`, `employers` and `counterparties`. An account points to its owner by `owner_type` and `owner_id`.

## 3. Who gets which accounts

| Owner | Asset accounts | Liability accounts | Equity accounts | Income accounts | Expense accounts |
| --- | --- | --- | --- | --- | --- |
| **Society** | Main bank, cash, investment in units, fixed deposits | Unidentified receipts, payouts in flight, tax payable, dividends payable | Share capital, reserve fund, education fund, retained surplus | Loan interest, fees, income from units | Staff, rent, audit, bank charges, impairment |
| **Member** | Loan, credit owed for goods | Savings (one per product), withdrawable shares, prepayments |  |  |  |
| **Non-member staff** | Credit owed for goods | Prepayments, overpayments |  |  |  |
| **Business unit** | Unit bank, cash in till, stock, settlements in transit, vehicles and equipment | Payables, buyer deposits, due to society | Capital from society | Sales, markup, transport income, rent | Cost of goods sold, stock losses, fuel, wages, depreciation |
| **Supplier** |  | Amount owed to the supplier |  |  |  |
| **Employer** | Deductions due, not yet remitted |  |  |  |  |
| **Driver or tenant** | Hire purchase or rent owed | Deposits held |  |  |  |

**Parent accounts.** Parent accounts give the society-level totals:

- "Member savings" is the parent of every member's savings account.
- "Member loans" is the parent of every loan account.
- "Suppliers payable" is the parent of every supplier account.

The balance sheet shows the parent. Each person's statement shows their own account.

**Rounds and ventures.** These are not owners. Their entries go to the unit's accounts, and the journal's source points to the round, the venture or the asset, so they can be reported on their own.

## 4. How one event is recorded

Take the payroll remittance from the explainer, cut down to two members. Musa saves ₦5,000 and repays ₦10,000 of his loan plus ₦500 interest. Aisha saves ₦7,000.

**The journal.** Journal 1045, dated 5 October, kind event. Its source is remittance R-2026-10, and its status is posted.

**Its entries.**

| Account | Owner | Type | Debit | Credit |
| --- | --- | --- | --- | --- |
| 1110 Main bank | Society | Asset | ₦22,500 |  |
| 2100 Savings | Musa | Liability |  | ₦5,000 |
| 2100 Savings | Aisha | Liability |  | ₦7,000 |
| 1210 Loan | Musa | Asset |  | ₦10,000 |
| 4110 Loan interest | Society | Income |  | ₦500 |
| **Total** |  |  | **₦22,500** | **₦22,500** |

**Where each balance moves.**

- The society's bank goes up.
- Musa's and Aisha's savings go up.
- Musa's loan goes down.
- The society earns ₦500.
- The parent "Member savings" rises by ₦12,000 without being posted to directly.

**A held payment.** Musa asks to withdraw ₦3,000. A journal is created with two pending entries: a debit to Musa's savings and a credit to the main bank, each ₦3,000, expiring in 7 days.

- **Posted balances** do not change yet.
- **Available balances** drop at once, so the money cannot be withdrawn twice.
- **When the bank confirms,** both entries become posted.
- **If the bank fails,** both are released.

**A unit event.** The foodstuff unit pays ₦800,000 for rice. The unit's stock account is debited ₦800,000, and the unit's bank account is credited ₦800,000. Both accounts are owned by the unit.

## 5. How balances are worked out

| Account type | Posted balance | Available balance |
| --- | --- | --- |
| Asset and expense (debit side) | debit\_posted − credit\_posted | posted balance − credit\_pending |
| Liability, equity and income (credit side) | credit\_posted − debit\_posted | posted balance − debit\_pending |

- **Holds.** Pending entries always reduce what is available, never increase it, so a hold can only make money harder to spend.
- **Parent accounts.** A parent's balances are the sums of its children's.
- **Accounts that cannot go negative.** Where allow\_negative is false (savings, bank, cash and stock), the available balance can never go below zero.

## 6. What every report reads

| Report | Read from |
| --- | --- |
| Member statement | The entries on that member's accounts, with running balance |
| Trial balance | `account_balances` for the period, top-level accounts only. Total debits equal total credits |
| Profit and loss | Income and expense accounts. For a unit, only that unit's accounts |
| Balance sheet | Asset, liability and equity accounts, with surplus for the year |
| Bank reconciliation | Entries on a bank account, against `statement_lines` |
| General ledger detail | Entries on any account, linked to their journal and source document |
| Audit trail | Journals in number order, with preparer, approver and hash |

## 7. Rules the database enforces

1. **Balanced journals.** Every journal's total debits equal its total credits before it can be posted.
2. **One side per entry.** Each entry has exactly one of debit or credit above zero.
3. **One book.** All entries in a journal belong to accounts in the journal's book. Money never crosses societies.
4. **Parents take no entries.** Parent accounts cannot receive entries. Their balance is always the sum of their children.
5. **No overdrawing.** Accounts with allow\_negative false refuse any entry that would take their available balance below zero.
6. **Restricted accounts.** Accounts with status no\_debit refuse debits, as for a freeze or a court order. Closed accounts refuse everything.
7. **No edits after posting.** Posted journals and entries are never edited or deleted. A correction is a new journal that reverses the original, and a journal can be reversed only once.
8. **Holds.** A pending entry can change status only once, to posted or released, and it is released automatically after expires\_at.
9. **Closed periods.** A closed period refuses journals. A soft-locked period accepts only approved adjustments.
10. **No gaps or duplicates.** Journal numbers are gap-free within a book, and each idempotency key is used once.
11. **Tamper evidence.** Each journal's hash includes the previous journal's hash, and a nightly check proves the chain.
12. **Bank agreement.** Each bank account's posted balance must agree with the bank statement. Every statement line ends matched, or as a break with an owner.

## 8. What changed from the previous version

| Before | Now |
| --- | --- |
| `ledger_accounts` plus `account_codes` | One `accounts` table, with the type and owner on each account |
| `transfers` with debit and credit account pairs | `journal_entries`, one line per account, as in the explainer |
| Dimension columns on every transfer | Not needed. The owner of the account tells you whose it is. The journal's source tells you which round, venture or asset |
| `control_proofs` | Not needed. Parent accounts sum their children by construction |
| `daily_balances`, `period_balances`, `balance_snapshots` | One `account_balances` table per period. Daily views can be computed |
| Separate tables for reconciliation runs, items, matches and breaks | `statement_lines`, with the matched entry and its status |
| `account_restrictions` | The account's status: active, no\_debit or closed |
| `posting_rules`, `journal_approvals`, `close_tasks`, `business_days`, `settlement_batches` and others | Kept in the application, outside the core ledger. They can be added back later if needed, without changing these eight tables |
