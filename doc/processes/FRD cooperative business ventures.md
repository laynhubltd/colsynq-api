# Functional Requirements Document: Cooperative Business Ventures

Oct 11, 2026 · @Abubakar

## Purpose, scope and conventions

This document specifies what the platform must do so that a registered cooperative can run a physical business without mixing it with member savings. Typical businesses are a commodity scheme, a shop, a filling station, a transport fleet or a water factory. The platform tracks every naira from the capital the society releases to the profit and capital it gets back. It turns the "Cooperative trading business" tab into testable requirements, user stories and acceptance criteria. It extends the core FRD and reuses its modules:

- M3 ledger;
- M6 deduction schedules;
- M7 remittance reconciliation;
- M8 payments and approvals;
- M9 bank integration;
- M11 year end.

**Product in one sentence.** A business-unit layer for cooperatives:

- approved capital and its own society-titled account;
- purchase-to-pay;
- sales capture across cash, POS, transfer and member credit;
- daily cash-up and banking;
- stock control;
- business books;
- monthly profit, and the return of capital and surplus to the society.

The platform never holds the money.

**In scope.**

| Release | Scope |
| --- | --- |
| **B-R1, commodity scheme and one shop** | B1 venture set-up and capital; B2 roles and separation; B3 suppliers and purchase-to-pay; B4 expenses and petty cash; B5 sales capture; B6 cash handling; B7 settlement and bank reconciliation; B8 stock; B9 member credit sales; B10 business ledger; B11 profit and returns; B12 reporting; B13 audit and fraud controls; B15 member commodity and asset requests |
| **B-R2, more ventures** | Transport returns and hire purchase; multi-outlet shops; filling-station meter readings; production costing for a water factory; subsidiary-company books; B14 integrations with shop systems and terminal providers beyond the first two |

**Out of scope.**

- Holding any business money in a platform account.
- Tax filing on behalf of societies. The platform produces the schedules, and the society or its adviser files.
- Real estate development, which the estates module covers.
- Payroll for business staff beyond recording wages as an expense.

**Conventions.**

- **Requirement IDs.** FR-B{module}-{nn}, for example FR-B6-03. Non-functional requirements are NFR-B{nn}.
- **User story IDs.** US-B{module}-{nn}, written as "As a {role}, I want {capability}, so that {outcome}".
- **Acceptance criteria.** Written as Given, When, Then. Each one can be tested on its own, and they are numbered AC-1, AC-2 within a story.
- **Priority.** Must is a release blocker. Should goes in the release if possible. Could comes later.
- **Configurable.** A configurable value is a business-unit parameter, set in B1 with the stated default.
- **Rule references.** "Act" means the Nigerian Co-operative Societies Act. "NTA" means the Nigeria Tax Act 2025.

## Roles and permissions

Permissions are granted per business unit. One person may hold several roles, but the system blocks any combination the separation rules forbid.

| Capability | Member | Business manager | Cashier | Storekeeper | Treasurer | Signatory | Business committee | Board | Supervisory committee | Auditor |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Propose a venture and business plan | — | ✓ | — | — | ✓ | — | ✓ | ✓ | — | — |
| Approve venture and capital ceiling | General meeting only |  |  |  |  |  |  | Recommends |  |  |
| Release a capital tranche | — | — | — | — | Initiates | Approves | — | Approves the tranche plan | — | — |
| Raise a purchase request | — | ✓ | — | ✓ | — | — | — | — | — | — |
| Approve a purchase | — | Within limit | — | — | — | — | Above manager limit | Above committee limit | — | — |
| Record goods received | — | — | — | ✓ | — | — | — | — | — | — |
| Pay a supplier or expense | — | — | — | — | Initiates | Approves | — | — | — | — |
| Record a sale | — | ✓ | ✓ | — | — | — | — | — | — | — |
| Buy on member credit | ✓ | — | — | — | — | — | — | — | — | — |
| Count a till | — | Witness | Own till, blind | — | — | — | — | — | Surprise counts | — |
| Count stock | — | — | — | ✓ | — | — | — | — | ✓ | Observes |
| Reconcile the business account | — | — | — | — | ✓ | — | — | — | Reviews | Read |
| Change prices | — | Within margin policy | — | — | — | — | Margin policy | — | — | — |
| Return profit or capital to the society | — | — | — | — | Initiates | Approves | — | Sets working capital | — | — |
| View business reports | Annual summary | ✓ | Own till | Stock | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |

**Separation rules.**

| Rule | The system blocks |
| --- | --- |
| SB-1 | The same person both recording sales on a till and recording goods received |
| SB-2 | The same person both raising a purchase request and approving or paying it |
| SB-3 | A cashier counting their own till without a second counter |
| SB-4 | A cashier or storekeeper reconciling the business account |
| SB-5 | A business committee member approving a purchase from a supplier they are related to |
| SB-6 | The same person approving a supplier and paying that supplier's first invoice |
| SB-7 | Anyone counting stock they issued on the same day, unless a supervisory committee member co-signs |

## B1. Venture set-up and capital

**Requirements.**

| ID | Requirement | Priority |
| --- | --- | --- |
| FR-B1-01 | A business unit is created with a type, a name, an outlet list, its business plan, and the general meeting resolution approving it with a capital ceiling | Must |
| FR-B1-02 | The business plan records start-up costs, working capital, capital tranches with dates, expected gross margin, monthly expenses, payback period, and the return expected compared with member lending | Must |
| FR-B1-03 | Each business unit links to exactly one bank account titled in the society's registered name, which is separate from the main account holding member savings. A title mismatch is refused | Must |
| FR-B1-04 | Capital is released in tranches through M8 payments, from the main account to the business account. A release that would exceed the approved ceiling is refused | Must |
| FR-B1-05 | Each release posts "investment in business unit" in the society's books and "capital from the society" in the business books, only on bank confirmation | Must |
| FR-B1-06 | Configurable parameters, with defaults: manager purchase limit ₦200,000; committee limit ₦2,000,000; quotes required above ₦500,000; petty cash float ₦30,000; cash tolerance ₦1,000 per till per day; banking deadline next business day; shrinkage tolerance 1.5% of sales; working capital to keep set by the board; report deadline the fifth working day | Must |
| FR-B1-07 | Raising the capital ceiling needs a new general meeting resolution. Changing other parameters needs the board's approval, and every change is logged and shown to the supervisory committee | Must |
| FR-B1-08 | A business unit can be paused or wound up. On wind-up, stock is sold or written off, suppliers are paid, and remaining cash and capital are returned to the main account before the unit closes | Should |

**User stories.**

**US-B1-01** As a board, we want to set up a venture only after members approve it, so that the society's money is never committed without consent.

- AC-1: *Given* a new business unit, *when* it is submitted, *then* the system requires a recorded general meeting resolution with quorum and a capital ceiling.
- AC-2: *Given* no resolution, *when* anyone tries to release capital, *then* the release is refused.

**US-B1-02** As a treasurer, I want to release capital in tranches against the plan, so that money leaves the main account only when the business needs it.

- AC-1: *Given* an approved plan, *when* the treasurer initiates a tranche, *then* the system shows the tranche against the plan, the capital already released and the ceiling remaining.
- AC-2: *Given* a tranche that would exceed the ceiling, *when* it is initiated, *then* it is refused.
- AC-3: *Given* signatory approval, *when* the bank confirms the transfer, *then* both sets of books post, and the business account balance matches the ledger.

**US-B1-03** As a member, I want the shop's takings kept apart from my savings, so that a loss in the shop cannot quietly eat my savings.

- AC-1: *Given* a business unit, *when* its bank account is linked, *then* the system refuses the society's main account or any account whose title differs from the registered name.
- AC-2: *Given* a sale, *when* it is recorded, *then* it can only post to the business unit's books.

## B2. Roles and separation

**Requirements.**

| ID | Requirement | Priority |
| --- | --- | --- |
| FR-B2-01 | Roles from the table above are assigned per business unit by the board, with start and end dates | Must |
| FR-B2-02 | Separation rules SB-1 to SB-7 are enforced at assignment and at each action | Must |
| FR-B2-03 | Related-party declarations are recorded for every committee member, manager, treasurer and signatory, and are checked against suppliers and credit customers | Must |
| FR-B2-04 | Staff with cash or stock duties must take at least one block of leave a year (configurable, default 5 working days), during which another person covers. The system flags anyone who has not | Should |
| FR-B2-05 | A handover between managers, cashiers or storekeepers triggers a cash count and a stock count, both witnessed | Must |

**User stories.**

**US-B2-01** As a supervisory committee, we want the system to stop risky role combinations, so that no single person can take money and hide it.

- AC-1: *Given* a person assigned as cashier, *when* the board tries to make them storekeeper for the same outlet, *then* the assignment is refused, quoting SB-1.
- AC-2: *Given* a committee member related to a supplier, *when* they open that supplier's purchase for approval, *then* approval is refused, quoting SB-5.

## B3. Suppliers and purchase-to-pay

**Requirements.**

| ID | Requirement | Priority |
| --- | --- | --- |
| FR-B3-01 | Supplier master: name, registration or ID, TIN, bank account verified by name enquiry, contacts, related-party flag and status. Only approved suppliers can receive purchase orders | Must |
| FR-B3-02 | A new supplier or a change of supplier bank details needs business committee approval, confirmation with the supplier on contact details already on file, and a cooling-off period before the first payment (configurable, default 24 hours) | Must |
| FR-B3-03 | Purchase request: items, quantities, estimated cost, reason, and the stock level that triggered it | Must |
| FR-B3-04 | Above the quotes threshold, three quotes are attached, and the chosen quote is justified in writing | Must |
| FR-B3-05 | An approved request becomes a purchase order with agreed prices and delivery terms. Approval follows the limits in B1 | Must |
| FR-B3-06 | Goods received: the storekeeper records quantity, condition, batch and expiry per line against the order. Partial deliveries are supported | Must |
| FR-B3-07 | Supplier invoice entry, and a three-way match of order, goods received and invoice. Differences beyond tolerance block payment (configurable: price 2%, quantity 0) | Must |
| FR-B3-08 | A matched invoice becomes a payment through M8, from the business account. Withholding tax is deducted where it applies, and the supplier receives a remittance advice | Must |
| FR-B3-09 | Credit notes and supplier returns reverse stock and payables, with the storekeeper's record of goods returned | Should |
| FR-B3-10 | Anti-splitting: requests to the same supplier for the same items within 30 days are added together and checked against the limits | Must |

**User stories.**

**US-B3-01** As a business manager, I want to raise a purchase request when stock runs low, so that the shop never runs empty and every purchase has a reason.

- AC-1: *Given* an item below its reorder level, *when* the manager opens purchasing, *then* the system suggests a request with the reorder quantity.
- AC-2: *Given* a request above the quotes threshold, *when* it is submitted without three quotes, *then* submission is blocked.

**US-B3-02** As a storekeeper, I want to record exactly what arrived, so that we pay only for what we received.

- AC-1: *Given* a purchase order, *when* goods arrive, *then* the storekeeper records the quantity, condition, batch and expiry for each line, and stock increases by the received quantity only.
- AC-2: *Given* a short delivery, *when* it is recorded, *then* the order stays open for the balance.

**US-B3-03** As a treasurer, I want to pay only matched invoices, so that ghost suppliers and inflated invoices cannot get paid.

- AC-1: *Given* an invoice, *when* its price or quantity differs from the order or goods received beyond tolerance, *then* payment is blocked and the difference is shown.
- AC-2: *Given* a supplier whose bank details changed within the cooling-off period, *when* payment is initiated, *then* it is held until the period ends and the change is confirmed.
- AC-3: *Given* a matched invoice, *when* the signatories approve, *then* the payment leaves the business account net of withholding tax, and payables, the bank and withholding tax payable post on bank confirmation.

## B4. Expenses and petty cash

**Requirements.**

| ID | Requirement | Priority |
| --- | --- | --- |
| FR-B4-01 | Expense types: rent, wages, power, fuel, transport, repairs, levies, licences, bank charges and others, each mapped to a ledger account | Must |
| FR-B4-02 | One-off expenses follow request, approval and payment, with a receipt or invoice attached | Must |
| FR-B4-03 | Recurring expenses, such as rent and wages, are approved once as a schedule with amounts and dates, and paid on their dates after a check against the schedule | Must |
| FR-B4-04 | Petty cash runs on a fixed float. Each spend needs a receipt or signed voucher, and the float is topped up by transfer from the business account only against receipts, back to the fixed amount | Must |
| FR-B4-05 | Petty cash cannot be topped up from till cash. The system rejects any cash-up that records a petty cash withdrawal | Must |
| FR-B4-06 | A budget is set for each expense type. Spend above budget needs board approval | Should |

**User stories.**

**US-B4-01** As a cashier holding petty cash, I want to top it up by showing receipts, so that small spends are covered and every naira is explained.

- AC-1: *Given* receipts totalling ₦18,000 against a ₦30,000 float, *when* a top-up is requested, *then* the system proposes ₦18,000 and the vouchers must match.
- AC-2: *Given* missing receipts, *when* a top-up is requested, *then* only the receipted amount can be approved.

## B5. Sales capture

**Requirements.**

| ID | Requirement | Priority |
| --- | --- | --- |
| FR-B5-01 | Every sale is recorded with outlet, till, user, items, quantities, prices, discounts and payment method, whatever the method | Must |
| FR-B5-02 | Payment methods: cash, bank POS, gateway terminal, transfer to the outlet's reserved account, and member credit. A split payment across methods is supported | Must |
| FR-B5-03 | A transfer sale is complete only when the bank or gateway confirms the credit through a verified webhook and a status query. A screenshot or SMS alert is never accepted | Must |
| FR-B5-04 | Each outlet or till has its own reserved account number, so that transfer sales are attributed without manual entry | Must |
| FR-B5-05 | Prices come from the price list. A sale below cost, or a discount above the manager's limit, needs the manager's approval at the till | Must |
| FR-B5-06 | Returns, refunds and voids need the manager's approval and the original receipt. A refund goes back by the original method only | Must |
| FR-B5-07 | A printed or SMS receipt with a unique number is issued for every sale | Must |
| FR-B5-08 | Sales can be captured offline and synchronised later. Receipt numbers stay unique, and offline sales are flagged in the cash-up | Should |
| FR-B5-09 | VAT is applied by item where the business is VAT-registered. Zero-rated and exempt items are coded | Should |
| FR-B5-10 | Transport returns: each driver has a reserved account and a due amount per day or week. Shortfalls are aged per driver | Could, B-R2 |

**User stories.**

**US-B5-01** As a cashier, I want to ring up every sale quickly whatever the payment method, so that the till always matches the day's takings.

- AC-1: *Given* a basket, *when* the cashier completes a cash sale, *then* stock falls, the sale posts to cash in till, and a numbered receipt prints.
- AC-2: *Given* a transfer sale, *when* the customer pays the outlet's account number, *then* the sale completes only after the bank confirms the credit, and the cashier sees the confirmation on screen.
- AC-3: *Given* a customer showing a payment screenshot with no confirmation, *when* the cashier tries to complete the sale, *then* the system keeps it open as awaiting payment.

**US-B5-02** As a business manager, I want refunds and voids to need my approval, so that staff cannot reverse real sales and keep the money.

- AC-1: *Given* a void or refund, *when* a cashier starts it, *then* the manager's approval and the original receipt number are required.
- AC-2: *Given* an approved refund, *when* it is processed, *then* it returns by the original method only, and stock is restored if the goods came back.

## B6. Cash handling

**Requirements.**

| ID | Requirement | Priority |
| --- | --- | --- |
| FR-B6-01 | Each till opens with a fixed float, issued and signed for by the cashier | Must |
| FR-B6-02 | Cash above a set amount during the day is dropped to the safe and logged with amount, time and witness (configurable, default ₦100,000) | Should |
| FR-B6-03 | At close, the system produces the day's sales report by method. The cashier enters a blind count by note value without seeing the expected figure, and a second person recounts | Must |
| FR-B6-04 | The system computes counted cash minus float against recorded cash sales, and records any over or short against the cashier | Must |
| FR-B6-05 | A variance within tolerance is logged. A variance above tolerance alerts the treasurer and the business manager the same evening. Overs and shorts are treated alike | Must |
| FR-B6-06 | A cash-up locks the day's sales for that till. Later changes need the manager's approval and leave a trail | Must |
| FR-B6-07 | Cash is banked by the banking deadline. The deposit slip is photographed and attached to the cash-up | Must |
| FR-B6-08 | No expense, supplier payment, loan or cash-out may be paid from till cash. The cash-up form has no field for it | Must |
| FR-B6-09 | The supervisory committee can run a surprise count on any till at any time. The result is compared with sales recorded up to that minute | Must |

**User stories.**

**US-B6-01** As a cashier, I want a clear cash-up at the end of my shift, so that I can prove I handed over every naira.

- AC-1: *Given* a till at close, *when* the cashier enters a blind count, *then* the expected figure stays hidden until a second person has recounted.
- AC-2: *Given* both counts, *when* they agree, *then* the system shows the over or short and asks both people to sign.
- AC-3: *Given* counts that disagree, *when* they are submitted, *then* a third count is required.

**US-B6-02** As a treasurer, I want to know the same evening when a till is out, so that problems are caught before the trail goes cold.

- AC-1: *Given* a variance above tolerance, *when* the cash-up is signed, *then* the treasurer and manager are alerted within 15 minutes, with the till, cashier and amount.
- AC-2: *Given* repeated variances by one cashier, *when* three occur within 30 days, *then* the supervisory committee is alerted.

## B7. Settlement and bank reconciliation

**Requirements.**

| ID | Requirement | Priority |
| --- | --- | --- |
| FR-B7-01 | Every cash-up is matched to its bank deposit on the business account statement. An unmatched cash-up after two business days opens a break | Must |
| FR-B7-02 | Every POS and gateway sale is matched to its settlement line by terminal, reference and amount, net of charges. A missing settlement after the provider's settlement time opens a break | Must |
| FR-B7-03 | For terminals that settle into a provider wallet, the daily sweep to the business account is recorded and matched to the day's terminal sales | Must |
| FR-B7-04 | Transfer sales are matched to their credit lines by reserved account and reference | Must |
| FR-B7-05 | The daily proof: the business account's ledger balance equals the bank's posted balance, and cash in till is zero after banking apart from floats | Must |
| FR-B7-06 | Breaks are aged with owners and deadlines. A break older than five business days goes to the board | Must |
| FR-B7-07 | Bank charges and POS fees in settlement lines are posted to bank charges automatically | Must |

**User stories.**

**US-B7-01** As a treasurer, I want every sale tied to money in the bank, so that I know the takings arrived.

- AC-1: *Given* yesterday's cash-up of ₦185,000, *when* today's statement shows a ₦185,000 deposit, *then* the two are matched and the cash in transit clears.
- AC-2: *Given* POS sales of ₦420,000 for a terminal, *when* the settlement arrives net of fees, *then* the sales clear and the fees post to bank charges.
- AC-3: *Given* no settlement by the provider's settlement time, *when* the nightly sync runs, *then* a break opens with the terminal, the amount and the provider.

## B8. Stock

**Requirements.**

| ID | Requirement | Priority |
| --- | --- | --- |
| FR-B8-01 | An item master holds code, name, unit, cost method, selling price, VAT code, reorder level, reorder quantity and expiry tracking | Must |
| FR-B8-02 | A running stock record by item, batch and outlet, updated by receipts, sales, transfers, returns and write-offs | Must |
| FR-B8-03 | Valuation at cost, first-in first-out or weighted average, with lower of cost and net realisable value at month-end, as IAS 2 requires | Must |
| FR-B8-04 | Expiry alerts a set number of days before expiry (configurable, default 30). Sales follow first-expired first-out | Must |
| FR-B8-05 | A monthly full count, blind to the record, by the storekeeper and a supervisory committee member | Must |
| FR-B8-06 | Weekly unannounced spot counts of high-value lines, selected by the system | Must |
| FR-B8-07 | Count differences post to stock losses by item, recording who was on duty. Shrinkage above tolerance alerts the supervisory committee | Must |
| FR-B8-08 | Transfers between outlets need a sender and a receiver, and are in transit until received | Should |
| FR-B8-09 | Every price change is logged with who made it and when. The system flags any price that gives a margin below the margin policy | Must |

**User stories.**

**US-B8-01** As a supervisory committee member, I want surprise spot counts on the lines that matter, so that stock theft shows up quickly.

- AC-1: *Given* the weekly schedule, *when* the system picks the lines, *then* the storekeeper is not told in advance.
- AC-2: *Given* a count below the record, *when* it is submitted, *then* the loss posts by item with the person on duty, and the shrinkage figure updates.

**US-B8-02** As a business manager, I want warnings before stock expires, so that we sell or return it instead of writing it off.

- AC-1: *Given* a batch 30 days from expiry, *when* the daily check runs, *then* the manager sees it with quantity and value.

## B9. Member credit sales and commodity schemes

**Requirements.**

| ID | Requirement | Priority |
| --- | --- | --- |
| FR-B9-01 | A member can buy on credit only up to their credit headroom. Headroom accounts for savings, loans, guarantees and the deduction cap in the core FRD | Must |
| FR-B9-02 | The member confirms each credit purchase in the app or by USSD, and a written deduction consent is held on file | Must |
| FR-B9-03 | The sale posts at the cash price. Any markup is set by the scheme terms (configurable, default 5% flat) and recognised as interest income over the repayment term | Must |
| FR-B9-04 | Instalments are added to the member's lines in the next M6 deduction schedule | Must |
| FR-B9-05 | Repayments matched in M7 reduce the member's credit receivable. The monthly total moves from the main account to the business account through M8 | Must |
| FR-B9-06 | Arrears are aged per member and per employer. An employer that deducts but does not remit is escalated as in M7 | Must |
| FR-B9-07 | A member who leaves with credit outstanding has it offset against their savings at exit, as in M12, within the bye-law rules | Must |
| FR-B9-08 | Scheme windows: a commodity scheme can run as a bulk order with a closing date, a member order list, one purchase order and delivery collection by member | Should |

**User stories.**

**US-B9-01** As a member, I want to buy foodstuff on credit and pay through my salary, so that I get goods now at a fair price.

- AC-1: *Given* my headroom is ₦150,000, *when* I order ₦120,000 of goods on a six-month plan, *then* the system shows the instalments and the markup before I confirm.
- AC-2: *Given* an order above my headroom, *when* I submit it, *then* it is refused with my available headroom.
- AC-3: *Given* a confirmed order, *when* the next schedule is built, *then* my first instalment appears on it.

**US-B9-02** As a treasurer, I want credit repayments to move to the business only when the employer has actually paid, so that the shop's books show real money.

- AC-1: *Given* matched repayments in M7, *when* the month closes, *then* the system proposes one transfer to the business account for the matched total, and lists the members.
- AC-2: *Given* unmatched lines, *when* the month closes, *then* they stay as arrears, and the business's receivable stays open for them.

## B10. Business ledger and accounting

**Requirements.**

| ID | Requirement | Priority |
| --- | --- | --- |
| FR-B10-01 | Each business unit has its own double-entry books inside the society, using the M3 ledger engine. Entries are immutable and hash-chained | Must |
| FR-B10-02 | A default chart of accounts for business units: bank, cash in till, petty cash, settlements in transit, stock, member credit receivable, trade receivables, fixed assets, suppliers payable, tax payable, capital from the society, sales by line, cost of goods sold, stock losses, cash over and short, operating expenses by type, and profit returned | Must |
| FR-B10-03 | Every operational event posts its journal automatically, as set out in the "Cooperative trading business" tab. Each journal links to its source document | Must |
| FR-B10-04 | Manual journals need a reason, a document, and approval by the treasurer and a business committee member. They are reported monthly to the supervisory committee | Must |
| FR-B10-05 | Corrections are reversing entries that name the entry they correct. Nothing is edited or deleted | Must |
| FR-B10-06 | Fixed assets register with depreciation posted monthly, by asset class and method (configurable, default straight line) | Should |
| FR-B10-07 | Tax schedules: withholding tax deducted per supplier and month for remittance by the 21st, VAT output and input where registered, and PAYE for business staff | Must |
| FR-B10-08 | The business books consolidate into the society's accounts at period end. Departmental accounts keep member income apart from trading income, as the NTA requires | Must |
| FR-B10-09 | Month-end close: all breaks cleared or carried with owners, the stock count posted, depreciation run, and the period locked. Reopening needs board approval | Must |

**User stories.**

**US-B10-01** As an auditor, I want every figure in the business accounts to trace to a document and an approval, so that I can sign the accounts.

- AC-1: *Given* any line in the profit and loss, *when* the auditor drills down, *then* they reach the journals and, for each, the source document, the user and the approvals.
- AC-2: *Given* a manual journal, *when* it is viewed, *then* its reason, document and both approvals are shown.

**US-B10-02** As a treasurer, I want the month closed in a fixed set of steps, so that the board gets reliable figures on time.

- AC-1: *Given* the close checklist, *when* a step is incomplete, *then* the period cannot be locked.
- AC-2: *Given* a locked period, *when* someone tries to post into it, *then* the posting is refused unless the board has reopened it.

## B11. Profit, returns to the society and year-end

**Requirements.**

| ID | Requirement | Priority |
| --- | --- | --- |
| FR-B11-01 | A monthly profit and loss, balance sheet and cash flow for each business unit, with gross margin, net profit and return on capital employed | Must |
| FR-B11-02 | The monthly report compares the venture's return on capital with the society's return on member lending | Must |
| FR-B11-03 | Monthly return: the system shows cash above the approved working capital. The treasurer returns it to the main account through M8, recorded as profit returned up to the month's profit, and capital returned beyond | Must |
| FR-B11-04 | The capital outstanding in each business unit is always visible: released minus returned | Must |
| FR-B11-05 | At year-end, the business unit's profit flows into the society's net profit in M11. The appropriation applies at least 25% to the reserve fund (Act; varied only if the Director's decision is recorded), and up to 10% of the remainder to an education fund with the Director's approval | Must |
| FR-B11-06 | Patronage refunds: where the general meeting approves one, each member's share is computed from their recorded purchases. It is posted as an appropriation and paid through M8 bulk payments | Should |
| FR-B11-07 | Tax on trading profit, where due under the NTA, is computed on the business unit's taxable profit before appropriation. The small-company test uses the business's turnover and fixed assets | Must |
| FR-B11-08 | A venture that misses its plan's return for two quarters in a row is flagged to the board with a review prompt | Should |

**User stories.**

**US-B11-01** As a board, we want to see each month whether the venture is worth the members' money, so that we can fix or close it early.

- AC-1: *Given* a closed month, *when* the report is produced by the fifth working day, *then* it shows sales, gross margin, expenses, net profit, return on capital, and the member-lending comparison.
- AC-2: *Given* two quarters below plan, *when* the second quarter closes, *then* the board sees a review flag on its dashboard.

**US-B11-02** As a treasurer, I want to return surplus cash to the society every month, so that members' money is not left idle in the shop's account.

- AC-1: *Given* a month-end business balance above working capital, *when* I open returns, *then* the system proposes the excess, split into profit returned and capital returned.
- AC-2: *Given* signatory approval, *when* the bank confirms, *then* the society's investment and income, and the business's capital and profit returned, post together.

**US-B11-03** As a member at the general meeting, I want to see what the venture earned and where it went, so that I can vote on the dividend and the venture's future with the facts.

- AC-1: *Given* the year-end, *when* the annual report is produced, *then* it shows:
  - capital released and returned;
  - profit by year;
  - the share that went to the reserve and education funds;
  - my own patronage, if a refund is proposed.

## B12. Reporting and transparency

**Requirements.**

| ID | Requirement | Priority |
| --- | --- | --- |
| FR-B12-01 | Daily dashboard for the manager and treasurer: sales by method and outlet, cash banked against cash sales, overs and shorts, settlements due and received, low stock and near-expiry items | Must |
| FR-B12-02 | Monthly board pack: the results in B11, stock turnover, shrinkage, member credit by age, overs and shorts by cashier, breaks by age, and capital outstanding | Must |
| FR-B12-03 | Annual member summary in the app and the general meeting pack | Must |
| FR-B12-04 | All reports can be exported as PDF and spreadsheet, and the auditor can download the full ledger | Must |

## B13. Audit and fraud controls

**Requirements.**

| ID | Requirement | Priority |
| --- | --- | --- |
| FR-B13-01 | Alerts: variance above tolerance; three variances by one cashier in 30 days; voids or refunds above a set share of a cashier's sales; sales below cost; shrinkage above tolerance; unbanked cash after the deadline; a supplier bank change; a payment to a related party; a manual journal | Must |
| FR-B13-02 | Alerts go to the named role, and to the supervisory committee for anything involving a manager or treasurer. Alerts cannot be dismissed without a recorded reason | Must |
| FR-B13-03 | The audit trail records every action with user, time, device and before-and-after values, and is kept for at least five years | Must |
| FR-B13-04 | A whistle-blowing channel lets members and staff report concerns to the supervisory committee. Reporters can stay anonymous | Should |
| FR-B13-05 | The supervisory committee can freeze outbound payments from a business unit, pending a board decision within 48 hours | Must |

**User stories.**

**US-B13-01** As a supervisory committee, we want to stop payments from a business we suspect, so that losses stop while we investigate.

- AC-1: *Given* a freeze, *when* it is applied, *then* no new payment from the business account can be approved, and the board is notified at once.
- AC-2: *Given* 48 hours without a board decision, *when* the time passes, *then* the freeze stays and the chair is reminded every 24 hours.

## B14. Integrations

**Requirements.**

| ID | Requirement | Priority |
| --- | --- | --- |
| FR-B14-01 | Reserved accounts for outlets, tills and drivers through the M9 bank adapter or a collections gateway, settling to the business account | Must |
| FR-B14-02 | Terminal transaction feeds by webhook or report from at least one bank POS provider and one gateway terminal in B-R1 | Must |
| FR-B14-03 | Wallet-settling terminals: daily sweep detection and matching. An automated sweep is used where the provider supports it | Should |
| FR-B14-04 | Import of end-of-day sales reports from a shop system with an API, such as Odoo or Lightspeed, where a society already uses one. The payment provider stays the source of truth for non-cash sales | Could |
| FR-B14-05 | All integrations follow the adapter contract, the webhook rules and the security requirements in the API integration tab | Must |

## B15. Member commodity and asset requests

The full design is in the "Member commodity and asset requests" tab. This module turns it into requirements.

**Requirements.**

| ID | Requirement | Priority |
| --- | --- | --- |
| FR-B15-01 | A member catalogue lists each item with its cash price, the available tenors, the markup per tenor, the monthly instalment, the quantity limit and the stock status. Items are marked as commodity or asset | Must |
| FR-B15-02 | A member builds a request in the app or by USSD, sees the total, markup and instalments, and confirms with an OTP that also records the deduction consent | Must |
| FR-B15-03 | Eligibility checks run on submission: membership period, arrears, credit headroom, deduction cap, item limits, active payroll and, for assets above the threshold, a guarantor. Each failure states the rule and the member's figures | Must |
| FR-B15-04 | Guarantors confirm in the app. Their own headroom falls by the guaranteed amount until the request closes | Must |
| FR-B15-05 | Commodity requests within all limits are approved automatically. Asset requests go to the business committee, and those above the board threshold go to the board | Must |
| FR-B15-06 | Prices lock at approval for the lock period. A supplier price rise above the re-confirm threshold before purchase asks the member to re-confirm or cancel | Must |
| FR-B15-07 | Approved commodity requests reserve available stock. Reservations expire after the reservation period and free the stock | Must |
| FR-B15-08 | Requests not met from stock are grouped by item and supplier into purchase orders in the buying window. Each goods-received line links to the requests it serves | Must |
| FR-B15-09 | When demand for an item exceeds supply, it is rationed by approval order and the allocation is published to members | Should |
| FR-B15-10 | Delivery needs the code sent to the member's or proxy's phone. The delivery officer must not be the storekeeper who picked that order. Assets record serial numbers and photos | Must |
| FR-B15-11 | On a confirmed delivery, the system posts the sale at the cash price, the cost of goods sold and the member credit receivable, and builds the instalment plan. Nothing posts at request or approval | Must |
| FR-B15-12 | Markup is recognised monthly over the tenor, straight-line by default or by effective interest if configured. Instalments are added to the next M6 deduction schedule | Must |
| FR-B15-13 | Matched repayments post to the clearing accounts between the society's and the business's books. The monthly transfer clears them to zero | Must |
| FR-B15-14 | Returns within the return window reverse the sale, the cost and any markup earned, and put the goods back into stock or to the supplier | Must |
| FR-B15-15 | Early settlement cancels the unearned markup, less any fee the scheme sets | Should |
| FR-B15-16 | The society keeps title to assets until the last instalment. Its ownership and the serial number show on the member's statement | Should |
| FR-B15-17 | Daily checks: the member sub-ledger equals the control account, available plus reserved stock equals the stock record, and every sale has a delivery code | Must |
| FR-B15-18 | Defaults, all configurable: 6 months' membership; headroom two times savings; asset guarantor above ₦250,000; board approval above ₦1,000,000; price lock 14 days; re-confirm above 5%; reservation 14 days; return window 7 days; commodity markup 5% flat; asset markup 10% flat over 12 months | Must |

**User stories.**

**US-B15-01** As a member, I want to order rice, oil and seasoning from my phone and pay through my salary, so that I get essentials now without cash.

- AC-1: *Given* the catalogue, *when* I add one bag of rice, one gallon of oil and one carton of seasoning on a 3-month plan, *then* I see the cash price, the markup and the monthly instalment before I confirm.
- AC-2: *Given* my headroom covers the order and the items are within limits, *when* I confirm with my OTP, *then* the request is approved at once and the stock is reserved for me.
- AC-3: *Given* I already took two bags of rice this month, *when* I add a third, *then* the system refuses it and shows the limit.

**US-B15-02** As a member, I want to buy an inverter over a year with a guarantor, so that I can afford a durable asset.

- AC-1: *Given* an asset above the guarantor threshold, *when* I submit, *then* the system asks me to name a guarantor, and the request waits until the guarantor confirms.
- AC-2: *Given* the guarantor confirms, *when* the committee approves, *then* the price is locked and the purchase order is raised.

**US-B15-03** As a storekeeper and delivery officer, we want deliveries proven by the member, so that nobody can sign goods out in a member's name.

- AC-1: *Given* a packed order, *when* the delivery officer hands it over, *then* the request cannot move to Delivered without the code from the member's or proxy's phone.
- AC-2: *Given* the delivery officer is the storekeeper who picked the order, *when* they try to record the delivery, *then* it is refused.
- AC-3: *Given* a short delivery, *when* it is recorded, *then* only the delivered items are posted, and the rest stays open.

**US-B15-04** As a treasurer, I want the books to post themselves at the right moments, so that the member balances, the stock and the business accounts always agree.

- AC-1: *Given* an approved request, *when* I look at the ledger, *then* nothing has posted.
- AC-2: *Given* a confirmed delivery, *when* it is saved, *then* the sale, the cost of goods sold and the member receivable post together, and the instalment plan is created.
- AC-3: *Given* month-end, *when* the markup run completes, *then* each open plan earns its month's markup, and the member sub-ledger still equals the control account.

**US-B15-05** As a supervisory committee, we want scarce goods shared fairly and prices checked, so that officers cannot favour friends or overcharge members.

- AC-1: *Given* demand above supply for an item, *when* stock is allocated, *then* allocation follows approval order and the list is published to members.
- AC-2: *Given* the monthly market price check, *when* a catalogue price is more than the set margin above the surveyed price, *then* the committee is alerted.

## Non-functional requirements

| ID | Requirement |
| --- | --- |
| NFR-B01 | The till screen completes a sale in under 2 seconds online. Offline capture works for at least 24 hours |
| NFR-B02 | The cash-up and settlement matching for a day completes within 30 minutes of the statement arriving |
| NFR-B03 | The monthly reports for a business unit with 50,000 sales lines are produced in under 5 minutes |
| NFR-B04 | Till and stock screens work on a low-cost Android phone or tablet over a 3G connection |
| NFR-B05 | Personal data of members and staff is protected as in the core FRD, and kept in Nigeria |
| NFR-B06 | Every money-moving action needs a second factor at the moment of approval |

## Main data entities

- **Venture set-up.** Business unit, business plan, capital tranche, parameter set.
- **Outlets and staff.** Outlet, till, role assignment.
- **Purchasing.** Supplier, purchase request, quote, purchase order, goods-received note, invoice, credit note.
- **Spending.** Expense schedule, petty cash voucher.
- **Sales and cash.** Sale, sale line, payment, refund, cash-up, safe drop, deposit.
- **Settlement.** Settlement line, sweep, break.
- **Stock.** Item, batch, stock movement, count, count line.
- **Member credit.** Member credit plan, instalment.
- **Accounting.** Fixed asset, journal, period.
- **Returns and oversight.** Return to society, appropriation, alert, freeze.

## Assumptions

1. Each venture's business account is opened at a commercial bank in the society's name, alongside the main account.
2. Terminal providers and gateways supply transaction feeds or daily reports that can be matched.
3. Societies run member credit schemes through the existing payroll deduction flow.
4. Trading profits are taxable under the NTA unless the small-company relief applies. The exact turnover threshold is confirmed before B-R1.
5. Staff have Android phones or tablets at each till.

## Dependencies

- Core modules M3, M6, M7, M8, M9 and M11, live for the society.
- A bank or gateway that issues reserved accounts settling to the business account.
- A POS provider with a transaction feed.
- A tax adviser's sign-off on the tax rules.
- Counsel's view on when a venture must move into a subsidiary company.

## Open issues

| # | Issue | Owner | Needed by |
| --- | --- | --- | --- |
| 1 | The signed NTA small-company threshold, ₦50m or ₦100m | Finance and tax adviser | B-R1 design freeze |
| 2 | Whether trading with members counts as taxable trade or mutual dealing | Tax adviser | B-R1 |
| 3 | Which POS providers to integrate first, and whether wallet-settling terminals are allowed at all | Partnerships | B-R1 |
| 4 | Default tolerances for cash variance and shrinkage, to be set with the pilot societies | Product and pilot boards | B-R1 pilot |
| 5 | State law differences on reserves, audit and investment approval, starting with Lagos 2022 | Legal | B-R1 |
| 6 | When a venture must become a subsidiary company, and how the books split | Legal and finance | B-R2 |
| 7 | Transport and filling-station specifics: driver returns, meter readings, product losses | Product | B-R2 |
