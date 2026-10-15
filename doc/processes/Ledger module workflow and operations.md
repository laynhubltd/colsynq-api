# Ledger module: workflow and operations

## 1. Purpose and principles

This tab specifies how the platform's ledger module is run, step by step: who does what, when, and with which checks. It is written from a chartered accountant's point of view. It builds on the ledger design in the FRD tabs: platform-built, TigerBeetle concepts, PostgreSQL, one book per society.

**Eight principles.**

1. **Double entry, always.** Every journal balances. Total debits equal total credits in every book, at every moment.
2. **Nothing is ever edited or deleted.** A mistake is corrected by a new journal that reverses it, and the trail shows both.
3. **Every entry has a reason and a document.** Each line carries its event code, its business date and a link to the source document. That might be an invoice, a cash-up, a statement line, a delivery code or a resolution.
4. **The sub-ledger and the control account always agree.** Member savings, loans, credit owed, supplier balances and the rest are posted line by line in the sub-ledger. Their totals equal the general ledger control account every day.
5. **The bank is the record of money. The ledger is the record of meaning.** Every bank line is matched to a journal, and every journal that moves money is matched to a bank line.
6. **Automatic first, manual by exception.** Most journals come from business events through approved posting rules. Manual journals are rare, need two people, and are reviewed monthly.
7. **Closed periods stay closed.** Nothing can be posted into a locked period. Reopening needs the board, and is logged.
8. **Every naira is attributed.** Each line is tagged with its society book, business unit, and where relevant the round, venture asset, member, supplier or outlet. Any report can then be cut without separate books.

## 2. Ledger structure

**Books.** Each society has one naira book, in kobo, and one quantity book for stock in units. The database refuses any journal between two books. Business units, rounds and ventures are dimensions inside the society's book, not separate books, so money can move between a society and its units.

**Chart of accounts.** This is the standard chart every society starts with. A society can add accounts within a range, but cannot change a code's type once anything has posted to it.

| Code | Account | Type | Notes |
| --- | --- | --- | --- |
| **1100 to 1199** | **Cash and bank** | Asset |  |
| 1110 | Main account bank | Asset | One per real account |
| 1120 | Unit bank accounts | Asset | One per unit account |
| 1130 | Estate deposit account | Asset | Ring-fenced buyer deposits |
| 1140, 1150, 1160 | Cash in till, safe, petty cash | Asset | Cannot go below zero |
| 1170 | Settlements in transit | Asset | Card and wallet takings awaiting settlement |
| 1180 | Transfers between own accounts | Asset | Must be zero at day end |
| **1200 to 1299** | **Member and customer receivables** | Asset |  |
| 1210 | Member loans | Control | Sub-ledger per loan |
| 1215 | Interest receivable on loans | Control |  |
| 1220 | Member credit owed, for commodities and assets | Control | Sub-ledger per plan |
| 1225 | Staff credit owed | Control | Sub-ledger per plan, reported separately |
| 1230 | Hire-purchase receivables | Control | Sub-ledger per agreement |
| 1240 | Requester receivables, for top-ups | Control |  |
| 1250 | Employer remittances due | Control | Deducted but not yet received |
| 1290 | Allowance for expected credit losses | Contra asset |  |
| **1300 to 1399** | **Inventory** | Asset |  |
| 1310 | Stock, by unit and item | Asset | Value side of the quantity book |
| 1320 | Round cost pool | Asset | Must be zero when a round closes |
| 1330 | Estate units under construction | Asset | Held for sale |
| **1400 to 1499** | **Investments and inter-unit** | Asset |  |
| 1410 | Investment in business units | Asset | Society side, eliminated in consolidation |
| 1420 | Advances to units | Asset | Society side |
| 1430 | Due from society | Asset | Unit side |
| 1440 | Investment property | Asset | Estates held to rent |
| 1450 | Fixed deposits and treasury bills | Asset |  |
| **1500 to 1599** | **Property, plant and equipment** | Asset | Vehicles, equipment, buildings, land. Accumulated depreciation is a contra account |
| **1600 to 1699** | **Prepayments and other receivables** | Asset | Including VAT input |
| **2100 to 2199** | **Member savings** | Control liability | Sub-ledger per member per product. Cannot go below zero |
| **2200 to 2299** | **Requester prepayments** | Control liability | Per requester per round. Non-member staff in a separate account |
| **2300 to 2399** | **Suppliers payable** | Control liability | Sub-ledger per supplier |
| **2400 to 2499** | **Taxes payable** | Liability | Withholding tax, VAT output, PAYE, company tax |
| **2500 to 2599** | **Clearing and suspense** | Liability | Payouts in flight, unidentified receipts, requester overpayments, due to business unit, due to society |
| **2600 to 2699** | **Buyer deposits held** | Liability | Estate buyers, until handover |
| **2700 to 2799** | **Accruals, dividends and refunds payable** | Liability |  |
| **2800 to 2899** | **Withdrawable member shares** | Liability | Shares members can withdraw on demand are a liability (IFRIC 2) |
| **3100 to 3999** | **Capital and reserves** | Equity | Non-withdrawable share capital, statutory reserve fund, education fund, other reserves, retained surplus. Unit side: capital from society, and profit and capital returned |
| **4100 to 4999** | **Income** | Income | Loan interest, sales, round sales, credit markup, service fees, transport income, rent, income from units (eliminated), gains on disposal |
| **5100 to 5499** | **Cost of sales** | Expense | Cost of goods sold, cost of round sales, stock losses, cost of estate units sold |
| **6100 to 6999** | **Operating expenses** | Expense | Staff, rent, utilities, fuel and maintenance, depreciation, bank and card charges, impairment and expected losses, cash over and short, audit and supervision fees |
| **7100 to 7499** | **Appropriations** | Equity, at year-end | Transfer to reserve, education fund, dividends, patronage refunds |

**Dimensions.** Every journal line carries the dimensions its account requires. The ledger refuses a line that is missing one.

| Dimension | Required on |
| --- | --- |
| Business unit | Every income, cost, inventory, unit bank and inter-unit line |
| Round | Round sales, round cost pool, requester prepayments and refunds |
| Venture asset | Asset, depreciation, transport income and hire-purchase lines |
| Member or staff | Every sub-ledger line: savings, shares, loans, credit owed, prepayments |
| Supplier | Supplier payable and withholding tax lines |
| Employer | Remittance and employer arrears lines |
| Outlet or till | Sales, cash and settlement lines |

**Control accounts and sub-ledgers.**

- Control accounts can only be posted through their sub-ledger. A manual journal cannot touch a control account directly. It must post to a named member, supplier or plan, which then updates the control account.
- The daily check proves every control account equals the sum of its sub-ledger.

**Inter-unit accounts.** The society's "investment in units" and "advances to units" mirror each unit's "capital from society" and "due to society". The consolidated society report eliminates them against each other. Any difference is a break.

## 3. Journal types

| Type | Source | Approval | Examples |
| --- | --- | --- | --- |
| **Event journal** | Generated by a module when a business event happens, through a posting rule | None at posting. The business approval happened in the module | Contribution received, sale, delivery, supplier payment, remittance matched |
| **Hold** | A pending entry that reserves money or stock | As for the event | Payout requested, stock reserved for a member |
| **Hold completion or release** | Completes or releases a hold | As for the event | Payout confirmed, payout failed, reservation expired |
| **Scheduled journal** | A schedule approved once | Approved when the schedule is set up | Monthly depreciation, markup recognition, prepayment release, rent |
| **Accrual** | Raised by the accountant at month-end, and reversed automatically on the first day of the next month | Maker and checker | Audit fee accrued, utility bill not yet received |
| **Allocation** | Shares a common cost across units by an agreed basis | Maker and checker | Shared office rent spread across units by floor space |
| **Manual adjusting journal** | Entered by the accountant | Maker and checker, plus the board above a set amount | Reclassification, write-off, impairment |
| **Correction** | Reverses a wrong journal, then posts the right one | Maker and checker | Wrong unit tagged, wrong amount |
| **Period-end run** | System runs at close | Run by the accountant, reviewed by the treasurer | Expected credit losses, stock valuation, interest accrual |
| **Year-end closing and appropriation** | Closes income and costs to surplus, then appropriates | General meeting resolution | Reserve fund, education fund, dividends, patronage refunds |
| **Opening balance** | One-time migration, in a special opening period | Maker, checker, and treasurer sign-off on the trial balance | Balances brought over from the old records |

## 4. The journal lifecycle

&#91;embedded content: journal lifecycle · five steps, three exits\]

**Each step.**

1. **Draft.** A module or the accountant prepares the journal: lines, accounts, amounts, dimensions, business date, narrative and the linked document. Event journals are created already complete.
2. **Validated.** The ledger checks that:
   - the journal balances;
   - every account exists and accepts postings;
   - every required dimension is present;
   - the business date falls in an open period;
   - no control account is hit without a sub-ledger line;
   - the idempotency key is new.

   A failure returns the reason, and nothing is saved as posted.
3. **Approved.** This applies to manual, accrual, allocation, correction and opening-balance journals only. A second person, never the preparer, reviews the journal against its document and approves or rejects it. Above the board threshold, a board member must also approve.
4. **Posting.** The ledger locks the book and the affected accounts in a fixed order, and applies the balance guards. It writes the journal and its lines, updates balances, and extends the fingerprint chain, all in one transaction. If anything fails, nothing is posted, and the journal is marked Failed with the reason.
5. **Posted.** The journal is final. It appears in reports and statements, and is linked from its source document.

**Exits.**

- **Rejected.** The approver gives a reason. The preparer can raise a corrected new draft.
- **Failed.** For example, a balance guard refused it, or the period closed in the meantime. A retry is a new attempt with a new key.
- **Reversed.** A correction journal reverses it in full, with the original's reference. A journal can be reversed only once.

**Numbering.** Every posted journal gets the next number in its book's sequence, with no gaps. Failed and rejected attempts keep their own reference outside the sequence, so the posted series has no holes for an auditor to question.

## 5. Posting rules: from business event to journal

**How it works.** Each business event has one posting rule. The rule names the accounts to debit and credit, how each amount is computed (gross, net, fee or tax), which dimensions are copied from the event, and which document is linked. Modules never choose accounts themselves. They emit the event, and the rule produces the journal.

**Main posting rules.**

| Event | Debit | Credit | Dimensions |
| --- | --- | --- | --- |
| Savings contribution matched | 1110 main bank | 2100 member savings | Member, employer |
| Loan disbursed | 1210 member loan | 2510 payouts in flight, then 1110 on bank confirmation | Member, loan |
| Loan repayment matched | 1110 main bank | 1210 member loan, and 4110 interest income | Member, loan, employer |
| Round prepayment received | 1120 unit bank | 2200 requester prepayments | Unit, round, requester |
| Round purchase paid | 1320 round cost pool | 1120 unit bank, net, and 2410 withholding tax | Unit, round, supplier |
| Goods delivered, prepaid | 2200 requester prepayments | 4210 round sales, and 4400 service fees | Unit, round, requester |
| Cost of goods delivered | 5200 cost of round sales | 1320 round cost pool | Unit, round |
| Goods delivered on credit | 1220 or 1225 credit owed | 4210 round sales | Unit, round, member or staff |
| Monthly markup | 1220 or 1225 credit owed | 4300 credit markup | Unit, plan |
| Card sale | 1170 settlements in transit | 4200 sales | Unit, outlet |
| Card settlement | 1120 unit bank, and 6500 card charges | 1170 settlements in transit | Unit, outlet |
| Cash banked | 1120 unit bank | 1140 cash in till | Unit, outlet |
| Supplier invoice matched | 1310 stock, or an expense, and VAT input | 2300 suppliers payable | Unit, supplier |
| Supplier paid | 2300 suppliers payable | 1120 unit bank, and 2410 withholding tax | Unit, supplier |
| Capital tranche | 1410 investment, and on the unit side 1120 unit bank | 1110 main bank, and on the unit side 3600 capital from society | Unit |
| Driver's return | 1120 unit bank | 4500 transport income, or 1230 hire purchase and 4300 interest | Unit, asset, driver |
| Depreciation run | 6400 depreciation | Accumulated depreciation | Unit, asset |
| Profit returned | 3610 profit returned (unit side), and 1110 main bank (society side) | 1120 unit bank (unit side), and 4700 income from units (society side) | Unit |

**Changing a rule.**

- **Approval.** Rules are versioned with an effective date. A change needs the accountant to propose it, the treasurer to approve it, and a test against sample events before it goes live.
- **No backdating.** A new rule never re-posts history. Earlier journals affected by a wrong rule are fixed by approved correction journals, listed in a correction batch.

## 6. Daily and weekly operations

**The daily run sheet.** Business days are in West Africa Time. The business date cut-off is 23:59.

| When | Task | Who | Done when |
| --- | --- | --- | --- |
| Overnight | Integrity check: debits equal credits, balances equal their journals, the fingerprint chain is intact, and control accounts equal their sub-ledgers | System | Report green. A red result suspends payouts for that book and alerts the treasurer and supervisory committee |
| Overnight | Expire holds past their time limit, and run scheduled journals that fall due | System | Expired holds released. Schedules posted |
| 07:00 | Import bank statements, by feed or upload | System or finance officer | Every bank account has yesterday's statement |
| 08:00 | Bank matching: every statement line cleared or a break opened | Finance officer, with a checker for manual matches | No unmatched line without an owner |
| 09:00 | Clearing accounts check: settlements in transit, payouts in flight, transfers between own accounts, round cost pools | Finance officer | Each holds only items still inside their window |
| 09:00 | Cash-ups from yesterday matched to deposits | Finance officer | Every cash-up banked, or a break raised |
| 10:00 | Unidentified receipts worked: matched to a member, requester or employer, or returned | Finance officer | Nothing older than 5 business days |
| Through the day | Manual journals reviewed and approved | Accountant prepares, treasurer approves | Queue empty by close |
| Through the day | Exceptions queue: guard refusals, failed postings, hold expiries after bank success | Accountant | Each resolved or escalated |
| 23:59 | Business date closes, and the daily balance snapshot is taken | System | Snapshot stored |

**Weekly.**

1. **Breaks.** Review aged breaks. Anything over 5 business days goes to the board's list.
2. **Old holds.** Review holds older than half their time limit, and chase the bank for an answer.
3. **Journal sample.** The supervisory committee reviews a sample of manual journals and posting-rule exceptions.
4. **Payroll.** Prepare next month's deduction schedules from the ledger's instalment plans, and reconcile them to credit owed.
5. **Suppliers.** Check that the suppliers due for payment match approved, matched invoices.

## 7. Month-end close, day by day

**Two locks.** Each period has two locks, like the lock dates in mainstream accounting tools.

- **Soft lock**, from the last business day. Modules can no longer post into the month. Only the accountant can post adjusting journals into it, and only with approval.
- **Hard lock**, at the end of the close. Nobody can post into the month. Reopening needs the board.

**Late events.** An event that arrives after the soft lock with a business date in the closed month is posted in the next open month. It keeps its real business date for reporting, and is listed in the close pack as a late item.

**The close calendar.** BD means business day. BD0 is the last business day of the month.

| Day | Task | Owner | Evidence |
| --- | --- | --- | --- |
| BD-2 | Remind units of cut-off. Chase supplier invoices for goods received. Check the next deduction schedules have gone to employers | Accountant | Cut-off notice, list of goods received without an invoice |
| BD-1 | Review open holds and payouts in flight. List expected accruals and prepayments | Accountant, treasurer | Holds list, accruals list |
| BD0 | Business date closes at 23:59 and the soft lock applies. Monthly stock count and surprise cash counts. Final bank import | System, storekeeper, supervisory committee | Count sheets, cash-up reports |
| BD+1 | All bank accounts reconciled. Clearing accounts checked. Cash-ups matched to deposits. Remittances matched to schedules. Scheduled runs posted: depreciation, credit markup, prepayment release, loan interest accrual | Finance officer, accountant | Reconciliations R1 to R4 and R8, run reports |
| BD+2 | Accruals posted, set to reverse automatically on the first day of next month. Shared costs allocated to units. Stock valuation posted, with shrinkage and expiry write-downs. Expected credit loss allowance updated from the provision matrix. Asset register agreed to the ledger. Inter-unit accounts agreed | Accountant | Accrual schedule, valuation report, provision matrix, R9 to R11 |
| BD+3 | Every reconciliation in the catalogue signed off. Every control account certified against its sub-ledger. Trial balance reviewed. Variance analysis against budget and last month, explaining every line that moved more than 10% and more than ₦100,000 (configurable). Tax schedules prepared | Accountant prepares, treasurer reviews | Reconciliation pack, variance report, tax schedules |
| BD+4 | Unit chairs review their unit packs. Final adjusting journals approved. Treasurer signs the close checklist. Hard lock applied | Unit chairs, treasurer | Signed checklist |
| BD+5 | Monthly pack to the board and the supervisory committee. Tax payments scheduled before their due dates | Treasurer | Board pack, payment schedule |

**The expected credit loss allowance.** This uses the simplified approach for receivables, as IFRS 9 allows. Each receivable type has a provision matrix:

- **The receivable types** are member loans, member credit owed, staff credit owed, hire-purchase receivables, employer remittances due and requester top-ups.
- **The percentages** are set by days past due: current, 1 to 30, 31 to 60, 61 to 90, and over 90.
- **Where they come from.** The society's own repayment history, adjusted for current conditions. The board approves them each year.
- **Each month** the system applies the matrix to the aged balances. It posts only the change in the allowance.

**Stock at month-end.** It is valued at cost, first-in first-out or weighted average. Each item is written down to what it can be sold for, if lower. Expired and damaged goods are written down at once.

## 8. Year-end close, appropriation and audit

**Before the year ends.**

1. **Full counts.** A full stock count, a physical check of every asset in the register, and surprise cash counts.
2. **Confirmations.** Statements requested from banks, suppliers and employers. Members can see and confirm their balances in the app.
3. **Reviews.** Every loan, credit plan and hire-purchase agreement reviewed for expected losses. Impairment tests on damaged or under-earning assets.
4. **Tax.** The trading-profit tax computation for each unit and the society, including the small-company test.

**The year-end close, in order.**

1. **December close.** The December close runs as normal and hard-locks.
2. **Year-end adjustments period.** A special adjustments period opens: "period 13", dated the last day of the year. Only year-end and audit adjustments post there, each with two approvals.
3. **Closing journal.** The system closes every income and cost account to "current year surplus". Each unit's result is kept by dimension, so unit results stay visible after closing.
4. **Statutory reserve.** At least 25% of net surplus is required by the Act, unless the Director of Cooperatives varies it. It is posted as part of the close.
5. **Other appropriations wait for members.** The education fund, dividends and patronage refunds are recorded only when the general meeting approves them. A dividend declared after the year end is not a liability at the year end. It is posted when declared, to dividends payable, and disclosed in the accounts.
6. **Audit adjustments.** The external auditor works with read-only access. Agreed adjustments post in period 13 as audit adjustments, with the auditor's reference.
7. **Final lock.** Once the accounts are signed, period 13 hard-locks, and the year can never be reopened.
8. **The new year.** Opening balances are carried forward automatically. Balance sheet accounts carry their balances. Income and cost accounts start at zero. The new year's retained surplus equals last year's closing figure.

**The appropriation account.** It is presented to the general meeting in this order:

| Line | Basis |
| --- | --- |
| Net surplus for the year, after tax | From the closing journal |
| Less: statutory reserve fund | At least 25% |
| Less: education fund | Up to 10% of the remainder, with the Director's approval |
| Available for distribution |  |
| Dividend on shares | As the general meeting resolves |
| Patronage refunds | In proportion to members' recorded business |
| Retained surplus carried forward | The balance |

**Audit preparation list.** The ledger produces these for the auditor:

- trial balance and general ledger;
- journal register;
- every reconciliation with sign-offs;
- sub-ledger listings, with member, supplier and employer confirmations;
- the asset register;
- stock count sheets and valuation;
- the provision matrices;
- tax schedules and receipts;
- board and general meeting minutes;
- the audit trail of manual journals and posting-rule changes.

## 9. Reconciliations catalogue

| # | Account | Against | How often | Prepared by | Reviewed by | Tolerance |
| --- | --- | --- | --- | --- | --- | --- |
| R1 | Every bank account | Bank statement | Daily | Finance officer | Treasurer, and the supervisory committee monthly | None |
| R2 | Cash in till and safe | Cash-up counts and deposits | Daily | Finance officer | Treasurer | Till tolerance only. Bank side none |
| R3 | Settlements in transit | Provider settlement reports | Daily | Finance officer | Treasurer | Zero beyond the settlement window |
| R4 | Payouts in flight, and open holds | Open payouts list, and bank results | Daily | Finance officer | Treasurer | Zero beyond the hold time limit |
| R5 | Member savings, shares, loans, credit owed, staff credit | Sub-ledger totals | Daily, automatic | System | Accountant | None |
| R6 | Requester prepayments and round cost pool | Round settlement | At round close | Unit accountant | Unit chair and treasurer | None |
| R7 | Suppliers payable | Supplier statements | Monthly | Accountant | Treasurer | None |
| R8 | Employer remittances due | Deduction schedules and remittance advices | Monthly | Payroll officer | Treasurer | None |
| R9 | Inter-unit accounts | The unit's mirror accounts | Monthly | Accountant | Treasurer | Net zero |
| R10 | Stock value | Stock count and valuation | Monthly | Storekeeper and accountant | Supervisory committee | Shrinkage tolerance |
| R11 | Fixed assets | Asset register | Monthly | Accountant | Treasurer | None |
| R12 | Taxes payable | Tax schedules and remittance receipts | Monthly | Accountant | Treasurer | None |
| R13 | Buyer deposits held | Estate deposit account and buyer schedule | Monthly | Venture accountant | Treasurer | None |
| R14 | Unidentified receipts and requester overpayments | Ageing list | Weekly | Finance officer | Treasurer | Nothing over 5 business days |

**Every reconciliation:**

- shows the ledger balance and the outside balance, and lists every reconciling item with its age and owner;
- is signed off by a reviewer who did not prepare it;
- is locked with the period once signed.

A reconciling item older than its limit escalates automatically.

## 10. Suspense, breaks, corrections and prior-period errors

**No balancing plugs.** The ledger refuses an unbalanced journal, so there is no general suspense account for forcing journals to balance. Only two suspense-type accounts exist, and both are worked daily:

- **Unidentified receipts.** Money in that matches no member, requester or employer. It is cleared within 5 business days by matching it or returning it to the sender.
- **Requester overpayments.** Money owed back to requesters or staff. It is cleared by refund, or by applying it to another plan as the person chooses.

**Breaks.** Any difference between the ledger and an outside record opens a break with an owner, an age and an escalation path. A break is closed only by a correcting journal that names it, or by an outside correction, such as the bank reversing its own error. A break is never closed by writing the difference off.

**Correcting a wrong journal.**

1. The accountant raises a correction that reverses the wrong journal in full, with a reference to it.
2. The accountant posts the right journal.
3. Both go to the treasurer for approval as one batch.
4. If the wrong journal is in an open period, both post in that period. If the period is closed, both post in the current period, with the original business date kept for reporting.

**Write-offs.** Writing off a receivable needs the treasurer up to a set amount, and the board above it. The write-off first uses the expected-loss allowance. Recoveries after a write-off are recorded as income.

**Prior-period errors.** These are handled as IAS 8 requires.

- **Not material.** The error is corrected in the current period, as an ordinary correction.
- **Material.** The accounts are restated:
  - the correction is posted in the current period to opening retained surplus, with a prior-period flag;
  - the reports restate the comparative figures and disclose the error;
  - the board approves it.
- **Audited years** are never reopened in the ledger.

## 11. Opening balances: moving a society onto the ledger

1. **Choose the cut-over date.** Use a month end, ideally the end of an audited year, so the opening figures are already audited.
2. **Map the old accounts** to the standard chart. Every old account maps to exactly one new account. The accountant prepares the map, and the treasurer approves it.
3. **Extract and clean member-level data** from the old spreadsheets or system:
   - savings and shares per member;
   - each loan with its schedule, balance and arrears;
   - each credit plan.

   Remove duplicates, and match members to their identity records.
4. **Load the sub-ledgers** first: members, loans, plans, suppliers and assets with their accumulated depreciation.
5. **Post the opening balance journal** in a special opening period, through an opening-balance clearing account. The clearing account must end at zero. Anything left there is an unexplained difference that must be resolved before go-live.
6. **Tie out:**
   - the opening trial balance equals the old accounts at the cut-over date;
   - every control account equals its sub-ledger;
   - every bank account equals its statement at the cut-over date;
   - stock equals the count;
   - assets equal the register.
7. **Member confirmation.** Every member sees their opening balances in the app and can raise a query within 30 days. Queries are logged, investigated and corrected by approved journals.
8. **Parallel run.** Run the old and new records side by side for one month. Compare the month-end figures, and explain every difference.
9. **Sign-off and lock.** The treasurer and the supervisory committee sign the opening position, with the auditor's review where possible. The opening period is then hard-locked.

## 12. Reports the ledger produces

| Report | Content | Who uses it |
| --- | --- | --- |
| Trial balance | Every account's opening balance, debits, credits and closing balance, by period, filterable by unit | Accountant, auditor |
| General ledger detail | Every journal line in an account, with document links | Accountant, auditor |
| Journal register | Every posted journal in number order, with preparer, approver and type | Auditor, supervisory committee |
| Sub-ledger statements | Member savings, loans, credit plans, staff credit, supplier and hire-purchase statements | Members, staff, suppliers, treasurer |
| Profit and loss | By society, unit, round and venture, against budget and last period | Board, unit committees |
| Balance sheet | Society and unit, with consolidation that eliminates inter-unit accounts | Board, auditor |
| Cash flow | Direct method, from bank-tagged journals | Board |
| Ageing reports | Loans, credit owed, employer arrears, suppliers, unidentified receipts, breaks | Treasurer, board |
| Reconciliation pack | Every reconciliation in the catalogue, with sign-offs | Treasurer, auditor |
| Appropriation account | Year-end surplus and its appropriation | General meeting |
| Audit trail | Every action on the ledger and its settings, with user, time and values | Auditor, supervisory committee |
| Tax schedules | Withholding tax by supplier, VAT output and input, PAYE | Accountant, tax adviser |

Every figure in every report can be drilled down to its journals, and from a journal to its source document.

## 13. Controls, segregation of duties and journal testing

**Ledger roles.**

| Role | May | May not |
| --- | --- | --- |
| Preparer (accountant) | Draft manual, accrual, allocation and correction journals, and run period-end jobs | Approve their own journals, or change posting rules alone |
| Approver (treasurer) | Approve journals and correction batches, sign reconciliations, apply locks | Prepare the journals they approve |
| Board approver | Approve journals above the board threshold, write-offs above the limit, and period reopening | Prepare journals |
| Posting service (system) | Post validated and approved journals | Post anything that is unapproved or unbalanced |
| Reconciliation preparer (finance officer) | Prepare matches and reconciliations | Approve their own matches, or handle cash |
| Rule administrator | Propose posting-rule and chart changes | Activate them without treasurer approval and a test |
| Supervisory committee | Review samples, run journal tests, see everything | Post or approve |
| External auditor | Read everything and export | Change anything |

The system refuses role combinations that break these lines. Access is reviewed every quarter.

**Approval thresholds for manual journals.**

- The accountant prepares, and the treasurer approves, up to ₦1,000,000 (configurable).
- A board member also approves above that.
- Any manual journal touching a bank, member savings or a tax account always needs the treasurer, whatever the amount.

**Monthly journal testing.** The supervisory committee runs the system's journal tests every month. They follow the approach auditors use to look for fraud in journal entries. The tests flag:

1. manual journals posted on weekends, public holidays or outside working hours;
2. round amounts and amounts just below an approval threshold;
3. entries to accounts that are rarely used;
4. entries by users who rarely post, or who posted outside their usual units;
5. entries posted after the soft lock, and reversals early in the next month;
6. entries with no document attached, or with a weak narrative;
7. entries to suspense, clearing and control accounts;
8. repeated corrections of the same item.

Each flagged entry is reviewed and signed off, or escalated.

**Change control.** Posting rules, the chart of accounts, dimensions, thresholds and lock settings are versioned. Each change records who proposed it, who approved it, the test evidence and its effective date.

## 14. Tax, statutory returns and record retention in Nigeria

Items marked "to confirm" are being checked against the law and the tax authority's current guidance. Until then, treat them as working assumptions.

| Obligation | What the ledger produces | Deadline |
| --- | --- | --- |
| Withholding tax deducted from suppliers | Schedule by supplier with TIN, rate and amount, and a credit note for each supplier | Remit by the 21st of the next month |
| VAT, where the society or unit is registered | Output and input VAT schedules. Small businesses with turnover of ₦100m or less are exempt from registering | Return by the 21st of the next month |
| PAYE for staff of units | Payroll deduction schedule | Monthly, by the 10th of the next month (to confirm) |
| Company income tax on trading profit | Tax computation by unit, with the small-company test | Annual return, within the statutory period after year end (to confirm) |
| E-invoicing with the tax authority | Invoice data in the required format, if the society falls within the rollout | Phased by taxpayer size. Whether and when it reaches cooperatives is to be confirmed |
| Annual audit and returns to the Director of Cooperatives | Audited accounts, appropriation account and member statistics | As the state's cooperative law sets (to confirm per state) |

**Record retention.**

- **Accounting and tax records.** Keep at least six years after the end of the year they relate to (to confirm under CAMA 2020 and the tax administration law).
- **Anti-money-laundering records.** Keep for at least five years, under the Money Laundering (Prevention and Prohibition) Act 2022.
- **The platform's rule.** Ledger records, documents and audit trails are kept for at least six years after year end. They are never deleted before then.
- **Personal data.** After the retention period, personal data is anonymised in line with the Nigeria Data Protection Act. Accounting totals are kept.

**Reporting framework.** Smaller societies prepare accounts under IFRS for SMEs, as the Financial Reporting Council framework allows. Societies classed as public interest entities use full IFRS (to confirm).

## 15. Exceptions and incident handling

| Exception | What the ledger does | What people do |
| --- | --- | --- |
| A journal is refused by a balance guard | Marks it Failed with the guard's reason. Nothing posts | The module tells the requester. A retry gets a new key |
| The nightly integrity check fails | Suspends payouts for that book, and alerts the treasurer and supervisory committee | The accountant and the platform's engineers find the cause the same day. Fixes are only approved correction journals, never data edits |
| The fingerprint chain is broken | Treated as possible tampering. Payouts are suspended, and the evidence is preserved | Supervisory committee and the external auditor are told. Investigate before anything else |
| A control account differs from its sub-ledger | Opens a critical break | The accountant traces the difference. Correction journals go through the sub-ledger |
| The posting service is down | Modules queue their events. Nothing is lost, because every event has its own key | When the service returns, queued events post in order. Duplicates are ignored |
| A posting rule was wrong | The rule is disabled at once, and affected events are queued | The accountant prepares a correction batch, the treasurer approves it, and the rule is fixed and re-tested |
| A bank success arrives after a hold expired | Posts as a new journal, and flags it | The accountant checks the payout was not repeated |
| A request to reopen a closed period | Refused until the board approves | Prefer a correction in the current period. Reopen only for errors that matter, with the board's reason recorded |
| The database must be restored | Restores to the last consistent point, then the integrity check runs | Re-import bank statements since that point, and replay queued events by their keys. Reconcile before payouts resume |

## 16. Requirements and acceptance tests

**Requirements.**

| ID | Requirement | Priority |
| --- | --- | --- |
| FR-LED-01 | Standard chart of accounts per society. Code types cannot change after first posting | Must |
| FR-LED-02 | Required dimensions per account, enforced at validation | Must |
| FR-LED-03 | Control accounts post only through sub-ledgers. Daily sub-ledger to control proof | Must |
| FR-LED-04 | Journal lifecycle: draft, validated, approved, posting, posted. Rejected, failed and reversed states. Gap-free numbering of posted journals | Must |
| FR-LED-05 | Maker and checker on manual, accrual, allocation, correction and opening journals, with thresholds | Must |
| FR-LED-06 | Versioned posting rules with approval, test evidence and an effective date. No backdated re-posting | Must |
| FR-LED-07 | Scheduled journals: depreciation, markup, prepayments, interest | Must |
| FR-LED-08 | Accruals that reverse automatically on the first day of the next period | Must |
| FR-LED-09 | Soft and hard period locks. Late events post in the next open period with their real business date. Reopening needs the board | Must |
| FR-LED-10 | Close calendar with tasks, owners, evidence and sign-off, by business day | Must |
| FR-LED-11 | Reconciliation catalogue R1 to R14, with preparer and reviewer sign-off and ageing of reconciling items | Must |
| FR-LED-12 | Provision matrix for expected credit losses, posting only the change each month | Must |
| FR-LED-13 | Year-end adjustments period, closing journal, statutory reserve, and appropriations on general meeting approval | Must |
| FR-LED-14 | Opening balance migration with a clearing account that must be zero, tie-outs, member confirmation and a parallel run | Must |
| FR-LED-15 | Monthly journal-testing report for the supervisory committee | Must |
| FR-LED-16 | Reports listed in section 12, with drill-down to source documents | Must |
| FR-LED-17 | Tax schedules for withholding tax, VAT and PAYE, and supplier credit notes | Must |
| FR-LED-18 | Records kept at least six years after year end, and never deleted earlier | Must |
| FR-LED-19 | Consolidated society reports eliminating inter-unit accounts | Must |
| FR-LED-20 | Integrity checks and incident handling as in section 15 | Must |

**Acceptance tests.**

| Test | How | Expected result |
| --- | --- | --- |
| TL-01 | Post a manual journal directly to member savings with no member | Refused at validation |
| TL-02 | Post a sales line with no business unit | Refused at validation |
| TL-03 | The accountant approves their own journal | Refused |
| TL-04 | Post a ₦1,500,000 manual journal with only the treasurer's approval | Held for a board approver |
| TL-05 | Post an accrual on 31 March | It reverses automatically on 1 April |
| TL-06 | A module posts into March after the soft lock | Posted in April with a March business date, and listed as a late item |
| TL-07 | Anyone posts into March after the hard lock | Refused |
| TL-08 | Close March with an unsigned bank reconciliation | The hard lock is refused |
| TL-09 | Member savings sub-ledger differs from its control account | A critical break opens, and payouts stop for that book |
| TL-10 | Run the provision matrix twice in a month | The second run posts only any further change |
| TL-11 | Post dividends at year end before the general meeting | Refused. Only the statutory reserve posts at close |
| TL-12 | Leave ₦500 in the opening-balance clearing account | Go-live is blocked |
| TL-13 | A posting-rule change without test evidence | It cannot be activated |
| TL-14 | A manual journal posted at 23:30 on a Sunday, for a round amount just below the threshold | Flagged by the journal tests |
| TL-15 | Reverse the same journal twice | The second reversal is refused |
| TL-16 | Consolidate the society with two units | Inter-unit accounts eliminate to zero, or a break is shown |

## 17. Open issues

| # | Issue | Owner |
| --- | --- | --- |
| 1 | Confirm the PAYE remittance date, the company tax return deadline, and record retention periods under CAMA 2020 and the 2025 tax laws | Tax adviser |
| 2 | Confirm whether and when the tax authority's e-invoicing rollout reaches cooperatives and their units | Tax adviser |
| 3 | Confirm annual return and audit deadlines with the Director of Cooperatives in each pilot state | Counsel |
| 4 | Set the provision-matrix percentages from each pilot society's repayment history, and have the board approve them | Treasurer and board |
| 5 | Set the variance thresholds and the manual-journal approval thresholds per society | Board |
| 6 | Confirm the reporting framework (IFRS for SMEs, or full IFRS) for any society that may be a public interest entity | Auditor |
| 7 | Agree the migration approach and cut-over date with each pilot society | Treasurer |
