# FRD (Plain English): Cooperative Business Ventures, version 2.2

Oct 11, 2026 · @Abubakar

**Audience.** Cooperative executives, board members, supervisory committees, business managers, accountants and auditors.

**What this version is.** Version 2.0 replaces version 1.0, "FRD Plain English Cooperative Business Ventures". It keeps version 1.0's structure and plain language. It corrects the points that conflicted with the designs agreed in this document: payroll check-off, bank automation, payouts, the float, API integration, the trading business and member requests. It also adds the missing modules and flows. Version 2.2 adds three things. A module map shows member requests and inventory as separate entities. Integrations become an optional hub, with a manual mode for every capability. Module 14 is a platform-built ledger that copies TigerBeetle's concepts without using TigerBeetle. Plain English throughout, with no code.

## What changed from version 1.0

| # | Section in v1.0 | Version 1.0 said | Version 2.0 says | Why |
| --- | --- | --- | --- | --- |
| 1 | 1.1 Cardinal rule | Money can only enter the business account as capital tranches | Two kinds of money enter: approved capital tranches, and members' credit repayments once the employer's remittance is matched. One kind goes back: profit and capital returns. Each move needs two signatories | Commodity-scheme repayments arrive in the main account through payroll, and must reach the business |
| 2 | 2.1 Treasurer | Makes approved bank transfers | Prepares payments. Two of three signatories approve. Nobody pays alone | Matches the payments and approvals design |
| 3 | 2.1 Roles | Nine roles | Adds member, guarantor, signatories, delivery officer, employer payroll officer and the platform | Each one acts in a flow |
| 4 | 2.2 Separation rules | Seven rules | Nine rules. Adds: the delivery officer is never the picker, and the reconciliation maker is never the checker | Closes two gaps used in delivery and reconciliation fraud |
| 5 | 3.1 Settings | 14 settings | 38 settings, grouped by area | Member credit, petty cash, freeze and tolerances were missing |
| 6 | 3.1 Reconciliation cadence | Cadence decides how often cash and bank are proved | Matching runs every day as data arrives. Cadence decides when a period is proved, closed and reported. Banking deadlines and break alerts apply whatever the cadence | A weekly cadence must not let a missing deposit sit for a week |
| 7 | 4.2 Proposal | The business committee submits the plan, and the general meeting approves | The manager and treasurer prepare it. The board reviews and recommends it. The general meeting approves the venture and the ceiling. Tranches are paid by two signatories, and posted only on bank confirmation. A higher ceiling needs a new general meeting vote | Clear ownership, and the money-movement rules |
| 8 | 5.4 Spoilage | Logged as an operating expense | Posted to "stock losses and write-offs", reported separately, and counted in shrinkage | Spoilage hidden in expenses hides theft |
| 9 | 5.5 Rotation and valuation | FEFO and FIFO used loosely | Shelves rotate first-expired, first-out. Stock is valued first-in first-out or at weighted average, at the lower of cost and net realisable value | Rotation and valuation are different rules |
| 10 | 6.2 Three-way match | Records must match perfectly | Quantity must match exactly. Price may differ within a tolerance (default 2%) | Small rounding should not block every invoice |
| 11 | 6.3 Anti-splitting example | Three ₦180,000 orders avoid the ₦200,000 committee limit | ₦200,000 is the manager's limit. Three ₦180,000 orders total ₦540,000. Added together, they need committee approval and three quotes | The example used the wrong limit |
| 12 | 6.3 Withholding tax | 2% for goods, 5% for services | 2% for goods and general services. 5% for professional fees and commission. 10% for rent and interest. Doubled if the supplier has no TIN. Small-company exemption up to ₦2m a month. Remit by the 21st | The 2024 withholding tax regulations |
| 13 | 7.2 Card machines | Bank or fintech card machines | Bank POS and Paystack terminals settle to the business account. Moniepoint, OPay and PalmPay settle into the provider's own wallet, so a daily sweep must be made and matched. No cash-out agent terminal in a shop | Where the money actually lands |
| 14 | 7.3 and 9.2 Bank confirmation | Sale complete when the bank notifies | A notice counts only after a status check confirms it. The bank statement is the final proof. Unsigned notices are always checked | Forged and duplicate notices |
| 15 | 9.2 Break timing | A break after 24 to 48 hours | A break after each channel's settlement time plus a grace period | Bank POS settles next day. Wallet terminals settle the same day |
| 16 | 9.3 Manual track | The maker pairs selected transactions with statement lines | Every line on the uploaded statement for the whole period must be matched or classified. The opening and closing balances must agree with the ledger | Matching only chosen lines lets unexplained debits slip by |
| 17 | 10.3 Proof of cash | One formula for cash and bank, with small differences written off as cash over or short | Two separate proofs: one for the bank and one for till cash. Tolerance applies only to till counts. Bank differences are never written off. They are always investigated | Writing off bank differences hides fraud |
| 18 | 10.4 Financial pack | Five items | Adds the cash flow, stock turnover, shrinkage, member credit by age, overs and shorts by cashier, and capital outstanding | The board needs them to judge the venture |
| 19 | 11.2 Deduction cap | The maximum deduction allowed by law | The cap in the employer deduction agreement. The legal one-third cap is pending verification of the Labour Act text | Not yet verified |
| 20 | 11.3 Day-one profit | No profit at delivery | The trading margin is earned at delivery. Only the markup is spread month by month | Revenue is earned when the member takes the goods |
| 21 | 11 Member schemes | Headroom, markup, check-off and OTP only | Adds the full request lifecycle, approval routing, price lock, reservation expiry, rationing, guarantors, title to assets, returns and early settlement | Needed to run the scheme fairly |
| 22 | 12.1 Profit or capital returned | Returned up to the month's profit counts as profit | Returned up to profit earned and not yet returned counts as profit. Only the rest is capital | Profit kept in earlier months was being called capital |
| 23 | 12.2 Year-end | 25% reserve, up to 10% education, the rest by vote | The same, plus: the Director may vary the reserve, the education fund needs the Director's approval, tax on trading profit comes first, refunds need a general meeting vote, and dividends have no cap in normal years | Matches the Act and the Nigeria Tax Act 2025 |
| 24 | Not covered |  | New modules: expenses and petty cash, business books and tax, and bank and payment connections. Also an audit trail, approval of manual journals, and whistle-blowing | Every naira must be traceable |
| 25 | 13.2 Freeze | Locked for 48 hours, lifted only by a board resolution | Stays until a recorded board resolution lifts it. The board must meet within 48 hours, and the chair is reminded daily | The freeze should not lapse by time |

## 1. Summary and the cardinal rule

**The cardinal rule.** A cooperative's business must never use, risk or mix with members' withdrawable savings. Most cooperative collapses start with savings quietly moved into stock, property or shop losses.

**How the system enforces it.**

- **Ring-fencing.** Each venture is a separate business unit. It has an approved capital ceiling, its own bank account in the society's name, and its own books.
- **Takings.** Shop, filling-station or transport takings can never be paid into the main account where savings are kept.
- **Money in.** Only two kinds of money may enter the business account from the society. They are approved capital tranches, and members' credit repayments once the employer's remittance is matched line by line.
- **Money out.** Only one kind may go back: profit and capital returns.
- **Approval.** Each of these moves needs two of three signatories.
- **Tracing.** Every naira is traced from the moment capital leaves the main account until it comes back.

&#91;embedded content: money map · what may enter and leave the business account\]

The dashed line carries goods, not money.

**What this specification covers.**

- **Venture set-up and capital**, with the ceiling guard.
- **Stock**, with four live stock levels, and stock in and out.
- **Purchase-to-pay**, with a three-way match.
- **Expenses and petty cash.**
- **Sales**, across five payment channels.
- **Daily cash handling.**
- **Two-track reconciliation**, automatic or maker-checker.
- **Periodic proof and reporting** on a daily, weekly or monthly cadence.
- **Member commodity and asset requests.**
- **Business books and tax.**
- **The surplus sweep and year-end sharing.**
- **Governance, alerts and the emergency freeze.**
- **Optional integrations hub.**

Every limit, threshold, schedule and tolerance can be configured per venture.

## 2. Who does what

### 2.1 Roles

| Role | What they must do | What they must never do |
| --- | --- | --- |
| **General meeting** | Approves each venture and its capital ceiling, raises the ceiling, and decides how year-end surplus is shared | Take part in daily buying and selling |
| **Board** | Reviews and recommends the business plan, sets budgets and limits, appoints the business committee and staff, sets working capital, and reviews the monthly pack | Run tills or handle cash |
| **Business committee** | Oversees operations, vets suppliers, compares quotes, and approves purchases and asset requests above the manager's limit | Approve anything for themselves, their relatives or their business partners |
| **Business manager** | Runs daily operations, raises purchase requests, sets prices within the board's margin policy, plans deliveries, and approves voids and refunds | Hold a till, count stock alone, or approve payment of their own requests |
| **Cashier** | Rings up every sale, collects payment, and does the blind count at close with a witness | Receive stock, count stock alone, approve refunds, or reconcile the bank |
| **Storekeeper** | Receives and inspects deliveries, records batches and expiry dates, reserves and picks stock, and keeps stock records | Operate a till, collect money, or deliver orders they picked |
| **Delivery officer** | Hands goods to members and collects the member's delivery code | Deliver an order they picked themselves |
| **Treasurer** | Prepares capital releases, supplier and expense payments, transfers of credit repayments and returns to the society, and checks the bank daily | Pay alone, or pay orders they raised |
| **Signatories** (two of three) | Approve every payment from the main or business account, after seeing the evidence | Approve a payment they started, or one to themselves or a related party |
| **Supervisory committee** | Surprise cash and stock counts, monthly price checks, reconciliation review, investigations, and the emergency freeze | Take part in buying, selling or paying |
| **Member** | Orders goods, confirms terms and deduction consent with an OTP, and confirms delivery with a code | Collect goods without giving their code |
| **Guarantor** | Confirms guarantees for asset requests above the threshold |  |
| **Employer payroll officer** | Applies the deductions and remits them with an advice |  |
| **External auditor** | Audits the year with read-only access to all records | Change any record |
| **The platform** | Records every request, approval, sale, count and journal, runs the checks and produces the reports | Hold any money |

### 2.2 Separation rules

The system refuses any action that breaks one of these rules.

1. **No selling and receiving.** The same person cannot ring up sales and sign for supplier deliveries at the same outlet.
2. **No self-approval.** Whoever raises a purchase cannot approve it or pay it. Whoever starts a payment cannot approve it.
3. **No solo cash counts.** A second person always recounts a till.
4. **Cash handlers do not reconcile.** No cashier or storekeeper may reconcile the business account.
5. **Related parties.** No committee member may approve a purchase from a relative or business partner.
6. **Supplier details.** Whoever adds a supplier or changes their bank details cannot approve that supplier's first payment.
7. **Count independence.** Nobody counts stock they received or issued that day without a supervisory committee member co-signing.
8. **Delivery independence.** The delivery officer who collects a member's code cannot be the storekeeper who picked the order.
9. **Reconciliation independence.** The person who uploads a statement or proposes a match cannot approve it.

## 3. How a naira travels: six complete flows

**Flow A. Starting a venture and releasing capital.**

1. The business manager and treasurer prepare the business plan. It covers what will be sold, start-up costs, working capital, expected margin, monthly expenses, payback and the return compared with member lending.
2. The board reviews the plan and recommends it.
3. The general meeting votes on the venture and a capital ceiling, for example ₦10,000,000. The date, quorum and vote count are recorded.
4. A business account is opened in the society's name, for example "Harmony Cooperative Society, Supermarket Account". The system refuses an account in a person's name, or the main savings account.
5. The treasurer prepares the first tranche against the plan. Two signatories approve it, and the bank sends it from the main account to the business account.
6. When the bank confirms it, the society's books record "investment in business unit". The business books record "capital from the society".

**Flow B. Buying stock.**

1. Stock of an item falls to its reorder level, and the system suggests a purchase request.
2. The manager raises the request. Above ₦500,000, three quotes are attached and the choice is explained.
3. The request is approved within the limits: the manager up to ₦200,000, the committee up to ₦2,000,000, and the board above that. Orders to the same supplier within 30 days are added together for this test.
4. The approved request becomes a purchase order to an approved supplier.
5. The storekeeper counts the delivery and records a goods-received note with batch and expiry. Stock goes up by what arrived, not what was ordered.
6. The supplier's invoice is entered. The system matches order, goods received and invoice.
7. The treasurer prepares payment of the matched amount less withholding tax. Two signatories approve, and the bank pays from the business account.
8. When the bank confirms, the supplier's balance clears and the supplier gets a payment advice showing the tax withheld.

**Flow C. A day at the shop.**

1. Each cashier signs for a fixed opening float, for example ₦20,000.
2. Every sale is rung up, and stock falls at once. A transfer sale completes only when the bank's notice is confirmed by a status check. A POS sale completes on the terminal's approval.
3. When a drawer passes ₦100,000, the excess goes to the safe with a witness.
4. At close, the cashier counts blind, and a second person recounts. The system shows the over or short. Above ₦1,000, the treasurer and manager are alerted that evening.
5. The next business day, all cash is banked and the stamped deposit slip is photographed into the system.
6. Wallet-terminal takings are swept to the business account.
7. The following morning, someone who handles no cash matches the deposit, the POS settlements and the sweeps to the bank statement.

**Flow D. A member's commodity order.**

1. The member orders, for example, a bag of rice, a gallon of oil and a carton of seasoning cubes on a 3-month plan. They confirm the instalments with an OTP.
2. The system checks the member's headroom and limits, and approves automatically. Stock is reserved, and the price is locked for 14 days.
3. On delivery day, the storekeeper picks the order and a different person delivers it. The member gives the code sent to their phone.
4. The sale, the cost of goods and the member's debt are recorded at that moment. The first instalment goes on the next deduction schedule.
5. The employer deducts and remits to the main account. Each line is matched to the member.
6. At month-end, the matched total moves from the main account to the business account with two signatures.

**Flow E. Month-end.**

1. The full stock count is done blind, and losses are posted.
2. Every bank and cash break is cleared, or carried with an owner.
3. The month's markup on member credit is recognised, and depreciation runs.
4. The bank proof and the cash proof are run.
5. The period is locked, and the financial pack is ready by the fifth working day.
6. Cash above working capital is swept back to the main account. It is recorded as profit returned up to profit not yet returned, and the rest as capital returned.

**Flow F. Year-end.**

1. The business's profit after any trading tax joins the society's surplus.
2. At least 25% goes to the reserve fund, and up to 10% of the remainder to an education fund with the Director's approval.
3. The general meeting decides the rest: dividends on shares, patronage refunds on purchases, and retained surplus.
4. Approved refunds and dividends are paid as one bulk payment with two signatures.

### The module map: separate parts and how they connect

This section is new in version 2.2.

&#91;embedded content: module map · member requests and inventory are separate entities, integrations are optional\]

**Each module is a separate entity.**

- Each module owns its own records and rules.
- It talks to other modules only through the hand-offs listed below, never by changing another module's records.
- A module can be switched on or off for a society without breaking the others. A society can run member requests without a shop, or a shop without member requests.

**Member requests and inventory are separate entities.**

|  | Member requests | Inventory | The trading business unit |
| --- | --- | --- | --- |
| **What it is about** | A member's demand for goods on credit | The physical stock: what exists, where it is, and what it is worth | The shop or scheme as a business: prices, buying decisions, profit |
| **What it owns** | The request, eligibility checks, approval, instalment plan, delivery confirmation, and the member's credit balance | Items, locations, quantities on hand, reserved, available and in transit, every stock movement, counts, and valuation | Price lists, margin policy, purchase decisions, sales, and the business's books and profit |
| **What it never does** | Hold stock, set prices, or buy from suppliers | Decide who gets credit, set prices, or take payment | Approve a member's credit, or change stock records except through Inventory |
| **Who works in it** | Members, guarantors, the business committee for assets | Storekeepers, delivery officers, the supervisory committee for counts | Business manager, cashiers, treasurer |

**Hand-offs between modules.**

| From | To | What passes | When |
| --- | --- | --- | --- |
| Members and savings | Member requests | The member's credit headroom, arrears status and deduction capacity | When a request is checked |
| Member requests | Inventory | A fulfilment order: items, quantities, member and delivery point | On approval |
| Inventory | Member requests | Reserved, picked, delivered or expired status, with the delivery code result | As each step happens |
| Inventory | Purchasing | A shortage: items and quantities that cannot be met from stock | When a fulfilment order or the reorder level cannot be met |
| Purchasing | Inventory | Goods received, with batches and expiry dates | On the goods-received note |
| Sales and till | Inventory | Items sold or returned | On each sale or refund |
| Member requests | Payroll check-off | The instalment plan | On delivery |
| Payroll check-off | Member requests | Matched or missed instalments | After each remittance is matched |
| Purchasing, member requests, business units | Payments | A payment request with its evidence | When a payment is due |
| Every module | Ledger | A journal: one balanced bundle of movements, or a hold | At the moment the business event happens |
| Integrations hub | Payments, reconciliation, sales, payroll, members | Bank confirmations, statements, terminal reports, remittances and KYC results | Only where a connector is switched on |

**Integrations are always a choice, never a requirement.** The integrations hub is a separate module of optional connectors. Every process works fully in manual mode, and a society can switch a connector on later without changing its processes. Module 13 sets out each connector and its manual alternative.

## 4. Settings the board controls

The board sets these for each venture when it is created. Changes need board approval and are logged for the supervisory committee. Capital ceilings change only by a general meeting vote.

| # | Area | Setting | Default |
| --- | --- | --- | --- |
| 1 | Capital | Capital ceiling | Set by the general meeting |
| 2 | Capital | Tranche plan | Amounts and dates from the business plan |
| 3 | Capital | Working capital the business may keep | Stock value plus one month's expenses |
| 4 | Reporting | Monthly pack deadline | 5th working day |
| 5 | Governance | Time for the board to decide on a freeze | 48 hours |
| 6 | Buying | Manager's purchase limit | ₦200,000 |
| 7 | Buying | Committee's purchase limit | ₦2,000,000 |
| 8 | Buying | Three-quote threshold | ₦500,000 |
| 9 | Buying | Anti-splitting look-back | 30 days |
| 10 | Buying | Price tolerance in the three-way match | 2% |
| 11 | Buying | Quantity tolerance in the three-way match | None |
| 12 | Buying | Supplier bank-change cooling-off | 24 hours (72 hours for high-risk ventures) |
| 13 | Expenses | Petty cash float | ₦30,000 |
| 14 | Expenses | Budget per expense type | Set by the board |
| 15 | Cash | Opening till float | ₦20,000 |
| 16 | Cash | Safe-drop level | ₦100,000 in a drawer |
| 17 | Cash | Over or short tolerance | ₦1,000 per till per day |
| 18 | Cash | Banking deadline | Next business day |
| 19 | Cash | Repeated variance alert | 3 variances in 30 days |
| 20 | Cash | Voids and refunds alert | 5% of a till's sales |
| 21 | Reconciliation | Mode per payment channel | Automatic, manual or hybrid |
| 22 | Reconciliation | Period cadence | Daily for shops and filling stations, weekly for transport and water factories, monthly for commodity schemes |
| 23 | Reconciliation | Grace after a channel's settlement time | 1 business day |
| 24 | Reconciliation | Break escalation to the board | 5 business days |
| 25 | Stock | Valuation method | First-in first-out (weighted average allowed) |
| 26 | Stock | Expiry warning | 30 days before expiry |
| 27 | Stock | Shrinkage tolerance | 1.5% of monthly sales |
| 28 | Stock | Spot counts | Weekly, unannounced |
| 29 | Member credit | Minimum membership | 6 months |
| 30 | Member credit | Credit headroom | Two times savings, within the deduction cap |
| 31 | Member credit | Item limits | For example 2 bags of rice a month. One of each asset at a time |
| 32 | Member credit | Commodity markup and tenors | 5% flat, 1 to 6 months |
| 33 | Member credit | Asset markup and tenors | 10% flat over 12 months, 6 to 24 months |
| 34 | Member credit | Guarantor needed above | ₦250,000 |
| 35 | Member credit | Board approval above | ₦1,000,000 |
| 36 | Member credit | Price lock, and the price rise that triggers re-confirmation | 14 days, 5% |
| 37 | Member credit | Reservation expiry | 14 days |
| 38 | Member credit | Return window | 7 days |

**About cadence.** Matching runs every day, as bank and terminal data arrive, whatever the cadence. The cadence decides when a period is proved, closed and reported. The banking deadline and break alerts always apply, so a missing deposit is flagged in days, not at the end of a week or a month.

## 5. Module 1: Venture set-up and capital

**Purpose.** No venture starts, and no society money is spent on it, without a members' vote, a ring-fenced account and a capital ceiling.

**How it works.** This module follows Flow A.

**Further rules.**

- **Account check.** The business account's title must match the society's registered name. A personal account, or the main savings account, is refused.
- **Tranches only.** Capital is released in tranches against the plan, never the whole ceiling at once.
- **Ceiling guard.** A tranche that would push total releases above the ceiling is blocked. The screen shows how much of the ceiling remains.
- **Raising the ceiling** needs a new general meeting vote.
- **Recording.** A release is recorded in the books only when the bank confirms it.
- **Pause and wind-up.** A venture can be paused or wound up. On wind-up:
  1. stock is sold or written off;
  2. suppliers are paid;
  3. all remaining cash goes back to the main account before the unit closes.

**Story.** As a member, I want spending on a venture blocked unless members approved it, so that my savings are never gambled.

- **Test 1.** Given a venture with no recorded general meeting vote, when anyone tries to release money, the release is blocked.
- **Test 2.** Given a ₦10,000,000 ceiling with ₦8,000,000 released, when the treasurer tries to release ₦3,000,000, it is blocked and the screen shows ₦2,000,000 remaining.
- **Test 3.** Given an approved tranche, when the bank confirms it, the society's investment and the business's capital are recorded together, and the business account balance matches the books.

## 6. Module 2: Stock

**Purpose.** Stock is cash on shelves. Every item coming in, sitting on a shelf or going out is recorded, so losses show up quickly.

**Inventory is a separate entity.**

- It keeps the stock records for every business unit and outlet of the society.
- It serves counter sales, member requests and transfers alike, but owns none of them.
- It does not set prices, decide credit or take payment.
- Other modules ask it to reserve, issue or receive stock through the hand-offs in the module map.
- Its value is recorded in the ledger in the books of the business unit that owns the stock.

**Four stock levels, kept live for every item.**

1. **On hand.** What is physically in the store.
2. **Reserved.** What members have ordered and had approved, waiting for delivery. It cannot be sold to anyone else.
3. **Available.** On hand minus reserved. This is all a cashier can sell.
4. **In transit.** Stock moving between outlets. It belongs to neither outlet until it is received.

**Stock in.**

- **Supplier deliveries.** The storekeeper counts and inspects, and records a goods-received note with batch and expiry. Only what arrived is added. A short delivery leaves the order open for the rest.
- **Deliveries straight to a member.** An asset can go from the supplier to the member's home. The storekeeper inspects it on delivery, and the member's delivery code stands in for the goods-received note.
- **Customer returns.** These need the manager's approval and the original receipt. Resaleable goods go back to stock at cost. Others go to damaged goods.
- **Transfers in.** Goods sent between outlets are received by the receiving storekeeper.
- **Count gains.** Approved count gains are recorded as adjustments, with a reason.

**Stock out.**

- **Counter sales.** Stock falls the moment a sale completes.
- **Member orders.** Stock is reserved at approval. It leaves only when the member's delivery code is entered.
- **Spoilage, expiry and breakage.** These need a spoilage form with product, batch, reason and photo, and the manager's approval. They are posted to stock losses and write-offs, never to general expenses.
- **Unexplained shortages.** These are posted to stock losses, with the person on duty recorded.
- **Transfers out.** Goods sent to another outlet stay in transit until received.

**Rotation and valuation.**

- **On the shelf.** Goods go out first-expired, first-out. The manager is alerted every morning about items within 30 days of expiry, so they can be marked down or returned.
- **In the books.** Stock is valued at cost, first-in first-out or weighted average. At month-end it is held at the lower of cost and what it can be sold for.

**Counts.**

- **Monthly full count.** Done after closing on the last day, by the storekeeper and a supervisory committee member, on blank sheets that do not show the expected figures.
- **Weekly spot counts.** High-value lines, such as rice, oil and fuel, are chosen at random by the system and counted without notice.
- **Handover counts.** Whenever the storekeeper or manager changes, a count is done with a witness.
- **Shrinkage.** Losses above 1.5% of sales alert the supervisory committee and the board.

## 7. Module 3: Purchase-to-pay

**Purpose.** The business never pays for goods it did not receive, never buys from fake or inflated suppliers, and never pays without approval.

**How it works.** This module follows Flow B.

**The three-way match.** The purchase order, the goods-received note and the supplier's invoice must agree.

- **Quantity** must match exactly.
- **Price** may differ by up to 2%.
- **Anything beyond that** blocks payment and shows the difference.

**Safeguards.**

- **Approved suppliers only.** Each supplier is vetted by the committee: identity, TIN, a bank account checked by name enquiry, and any related-party links.
- **Bank-detail changes.** These put the supplier on hold for 24 hours, or 72 for high-risk ventures. An officer confirms the change on the phone number already on file. The person who made the change cannot approve the first payment.
- **Three quotes** above ₦500,000, with a written reason for the choice.
- **Anti-splitting.** Orders to the same supplier for the same items within 30 days are added together before limits apply. For example, three orders of ₦180,000 total ₦540,000. That is above the manager's ₦200,000 limit and the ₦500,000 three-quote threshold, so it needs committee approval and three quotes.
- **Withholding tax** is deducted on payment:
  - 2% for goods and general services;
  - 5% for professional fees and commission;
  - 10% for rent and interest.

  The rate doubles if the supplier has no TIN. A small company is exempt for payments up to ₦2m a month. Tax withheld is remitted by the 21st of the next month, and the supplier gets a credit note.
- **Credit notes and returns to suppliers** reduce stock and the amount owed, with the storekeeper's record of what went back.

## 8. Module 4: Expenses and petty cash

This module is new in version 2.0.

**Purpose.** Every running cost is approved and paid from the bank, and small cash spends stay small and receipted.

**One-off expenses.** Repairs, levies, licences and transport follow request, approval within the limits, and payment by transfer, with the invoice or receipt attached.

**Recurring expenses.** Rent, wages and power are approved once as a schedule of amounts and dates. They are then paid on their dates after a check against the schedule.

**Petty cash.**

- **A fixed float.** Petty cash runs on a fixed float, for example ₦30,000.
- **Proof for each spend.** Each spend needs a receipt or a signed voucher.
- **Top-ups.** The float is topped up only against receipts, back to the fixed amount, by transfer from the business account.
- **Never from the till.** Petty cash is never taken from till cash, and the cash-up screen has no field for it.

**Budgets.** Each expense type has a budget. Spending above it needs the board's approval.

## 9. Module 5: Sales

**Purpose.** Every sale is captured on every channel. Every customer gets a receipt. No fake alert or false reversal can pass.

**Five payment channels, and where the money lands.**

| Channel | Where the money lands | When it counts as paid |
| --- | --- | --- |
| Cash | The till, then the business account at banking | When rung up. It is proved at cash-up and at banking |
| Transfer to the till's own account number | The business account | Only when the bank's notice is confirmed by a status check |
| Bank POS or Paystack terminal | The business account, net of fees | On the terminal's approval. It is proved at settlement |
| Moniepoint, OPay or PalmPay terminal | The provider's own wallet, then a daily sweep to the business account | On the terminal's approval. It is proved when the sweep reaches the bank |
| Member credit | No money at the till. The member's debt is recorded, and repaid through payroll | When the member confirms with an OTP, within their headroom |

A sale can be split across channels.

**Rules.**

- **No screenshots.** No goods leave on an SMS alert or a screenshot. The sale stays open as "awaiting payment".
- **Numbered receipts.** Every sale gets a numbered receipt, printed or by SMS.
- **Prices.** Prices come from the price list. A sale below cost, or a discount above the manager's limit, needs the manager's approval at the till.
- **Refunds and voids.** These need the manager's approval and the original receipt. A refund goes back by the original method only.
- **No cash-out in the shop.** No cash-out agent terminal is allowed inside a shop. Agent terminals are locked to one location and one provider, and cash-out blurs the takings.
- **VAT.** Where the business is VAT-registered, each item carries its VAT code. Basic foods are zero-rated.
- **Offline sales.** If the connection drops, sales are captured offline and synchronised later. They are flagged at cash-up.

## 10. Module 6: Daily cash handling

**Purpose.** Cash is the easiest thing to lose. This module enforces one routine every day.

**How it works.** This module follows Flow C.

**Detailed rules.**

1. **Opening float.** The same fixed float every day, signed for.
2. **Safe drops.** Above the safe-drop level, the excess goes to the safe with a witness and a log entry. The safe is counted at close with the tills.
3. **Blind count.** The system keeps the expected figure hidden. The cashier counts by note value. A second person recounts. If the two counts differ, a third count is required.
4. **Over or short.** Counted cash, minus the float, minus recorded cash sales. Overs are as serious as shorts, because an over means a sale was not rung up.
5. **Alerts.** A variance above ₦1,000 alerts the treasurer and manager within 15 minutes. Three variances by one cashier in 30 days alert the supervisory committee.
6. **Lock.** A signed cash-up locks the day's sales for that till. Later changes need the manager's approval and leave a trail.
7. **Nothing paid from the till.** There is no field for expenses, petty cash, loans or cash-out.
8. **Banking.** All cash is banked by the next business day, with a photo of the stamped slip. Cash deposits currently carry no CBN fee (secondary source). Corporate withdrawals above ₦5m a week are charged 5% from 2026 (secondary source), so cash only ever flows from till to bank.
9. **Surprise counts.** The supervisory committee can count any till at any time against sales recorded up to that minute.

## 11. Module 7: Two-track reconciliation

**Purpose.** This proves that every naira recorded actually reached or left the bank, and that every bank line has a reason in the books.

&#91;embedded content: two-track reconciliation · every statement line ends cleared or as a break\]

**The rule for both tracks.** Reconciliation starts from the bank statement, not from the system's own records. Every line on the statement for the period must end in one of two states:

- **cleared**, matched to a sale, deposit, settlement, sweep, transfer or payment;
- **a break**, with an owner.

The statement's opening and closing balances must agree with the books.

**Track 1: automatic.** For banks, gateways and terminals connected by API.

- **Notices are only a first sign.** The bank or gateway sends a notice when money arrives or leaves. The system then checks the transaction's status with the provider before treating it as real. A notice that is not signed is always checked.
- **Matching.** Statement lines are matched by reference first, then by amount, date and account. A match made on amount and date alone is marked tentative, for a person to confirm.
- **Fees.** Card settlements arrive net of fees. The sale is cleared, the net amount goes to the bank, and the difference goes to bank and card charges.
- **Breaks.** A sale whose settlement has not arrived within the channel's settlement time plus one business day opens a break. That is the next day for bank POS, and the same day for wallet sweeps.

**Track 2: maker and checker.** For branches with paper statements, deposit tellers, drafts, or no connection yet.

1. **The maker.** A finance officer uploads the official statement for the whole period. The system reads or records every line, with the opening and closing balances.
2. **Pairing.** For each line, the maker pairs it with the system record, enters the teller or reference number, and attaches proof.
3. **The checker.** A different officer, such as the treasurer or an internal auditor, compares each pair with the evidence and approves or rejects it. The system refuses to let the maker act as checker.
4. **Closing the statement.** It closes only when every line is cleared or opened as a break, and the balances agree.

**Breaks, by seriousness.**

| Break | Seriousness | What happens |
| --- | --- | --- |
| Money left the bank with no approved payment behind it | Critical | Outbound payments for the business are suspended at once. The chair, the treasurer and the supervisory committee are told within one hour |
| A recorded deposit or settlement is not on the statement after its deadline | High | The treasurer chases it the same day. After 5 business days it goes to the board |
| An amount differs between the record and the bank | High | Resolved the same day, with evidence |
| Money arrived that matches no record | Medium | Held as unidentified receipts, and cleared within 5 business days |

## 12. Module 8: Periodic proof and reporting

**Purpose.** This proves the bank and the cash for each period, closes the period, and gives the board a full pack.

**Two proofs, never one.** Version 1.0 combined cash and bank in one formula and wrote small differences off as cash over or short. Version 2.0 separates them, because a bank difference is never a counting error.

**The bank proof.** Run for each period:

- start with the opening bank balance;
- add capital tranches, transfers of credit repayments, cash deposits, POS and gateway settlements, transfer sales and wallet sweeps;
- take away supplier payments, expense payments, petty cash top-ups, returns to the society and bank charges.

The result must equal the closing balance on the statement exactly. There is no tolerance. Any difference is a break and is investigated, never written off.

**The cash proof.** For each till and the safe:

- **Drawer.** The opening float plus cash sales, minus approved cash refunds and safe drops, must equal the counted cash. Differences within ₦1,000 are recorded as over or short against the cashier. Anything above is escalated.
- **Safe.** The safe at opening plus drops, minus cash banked, must equal the counted safe.
- **Banking.** Cash banked must equal the deposit on the bank statement.

**Closing a period.** A period closes only when:

1. both proofs pass, or every difference is open as a break with an owner;
2. the stock count is posted;
3. markup and depreciation have run.

Reopening a closed period needs the board's approval.

**The monthly pack**, ready by the 5th working day:

1. **Profit and loss.** Sales, cost of goods sold, gross margin, expenses by type, net profit before tax.
2. **Balance sheet.** Bank, cash, stock, member credit owed, amounts owed to suppliers, tax payable, and capital invested.
3. **Cash flow**, linking profit to the change in cash.
4. **Return on capital employed**, and the comparison with member lending. If the venture earns less than member loans would, the report says so.
5. **Stock.** Stock turnover in days, and shrinkage against tolerance.
6. **Member credit owed by age**, and employer arrears.
7. **Overs and shorts by cashier.**
8. **Breaks** by age and owner.
9. **Capital.** Capital released, profit returned and capital returned, to date.

A daily view for the manager and treasurer shows:

- sales by channel and outlet;
- cash banked against cash sales;
- settlements due against settlements received;
- low stock and items near expiry.

## 13. Module 9: Member commodity and asset requests

**Purpose.** Members get essentials and durable assets on salary credit, safely and fairly. The cooperative's money is protected, and every step is recorded.

**Member requests are a separate entity from inventory and from the shop.**

- **What this module owns.** The member's side: the request, the eligibility checks, approval, the instalment plan, delivery confirmation and the member's credit balance.
- **What it does not own.** It never holds stock. When a request is approved, it sends a fulfilment order to Inventory. Inventory reserves the goods, or passes a shortage to Purchasing, and reports back each step.
- **Who earns the profit.** The goods are sold by the trading business unit, which earns the margin and the markup.
- **The member's debt.** It sits in the member's records, and repayments come through payroll check-off.

**Two kinds of request.**

- **Commodities.** For example rice, oil, seasoning cubes and detergent. 1 to 6 months, 5% flat, approved automatically within limits, supplied from stock or a monthly bulk order.
- **Assets.** For example an inverter, fridge, laptop or motorcycle. 6 to 24 months, 10% flat over 12 months, approved by the committee (by the board above ₦1,000,000), bought to order, with a guarantor above ₦250,000. The society keeps title until the last instalment.

**The request, step by step.**

1. **Order.** The member picks items in the app or by USSD. They see the cash price, the markup, the tenor and the monthly deduction, and confirm with an OTP. The OTP also records the deduction consent.
2. **Checks.** The system checks:
   - six months' membership;
   - no arrears;
   - credit headroom: savings, loans and guarantees within two times savings;
   - the deduction cap in the employer agreement;
   - item limits;
   - an active payroll;
   - a guarantor where needed.

   A failure is shown with the rule and the member's own figures.
3. **Approval.** Commodities within all limits are approved at once. Assets go to the committee or the board. On approval, the price is locked for 14 days. If the supplier's price rises more than 5% before purchase, the member must re-confirm or cancel.
4. **Fulfilment.** Stock is reserved for the member. What is not in stock is grouped with other requests into one purchase order per buying window, through Module 3. Unclaimed reservations expire after 14 days.
5. **Fair shares.** If demand for an item is more than the supply, it is shared in approval order, and the list is published to members.
6. **Delivery.** The storekeeper picks the order, and a different person delivers it. The member, or a proxy named in the app, gives the code sent by SMS. Assets carry a serial number and photos. Short or damaged items are noted, and only what was delivered is recorded.
7. **Repayment.** Instalments go on the next payroll deduction schedule. The employer's remittance is matched line by line. Missing lines become arrears and are escalated to the employer. At month-end, the matched total moves from the main account to the business account with two signatures.
8. **Returns and early settlement.**
   - **Returns.** Faulty goods reported within 7 days reverse the sale.
   - **Early settlement.** It cancels the markup not yet earned.
   - **Leaving the society.** A member who leaves has their remaining balance offset from savings, within the bye-laws.

**How the books record it.**

- **Order and approval.** Nothing is recorded. There is only a reservation.
- **Delivery.** The sale at the cash price, the cost of goods and the member's debt are recorded together. The trading margin is earned now.
- **Markup.** It is earned month by month over the tenor, as interest.
- **Remittances.** They pass through clearing accounts between the society's books and the business's books. Both clearing accounts are zero after the monthly transfer.

**Worked example.** A bag of rice, a gallon of oil and a carton of seasoning cubes cost ₦154,000 in cash and ₦142,500 to buy (illustrative prices). Over 3 months at 5%, the markup is ₦7,700 and the monthly deduction is ₦53,900. The business earns ₦11,500 of margin at delivery and ₦7,700 of interest over three months. The member's balance reaches zero after the third deduction. The full entry-by-entry table is in the "Member commodity and asset requests" tab.

## 14. Module 10: Business books and tax

This module is new in version 2.0.

**Separate books.** Each venture keeps its own double-entry books inside the society's, held in the platform ledger described in Module 14. Every event records itself automatically and links to its document: purchase order, goods-received note, invoice, cash-up, settlement, count sheet or delivery code. Entries are never edited. A correction is a new entry that names the one it corrects.

**Main accounts.**

- **Assets.** Bank, cash in till and safe, petty cash, settlements in transit, stock, member credit owed, and fixed assets.
- **Liabilities.** Amounts owed to suppliers, and tax payable.
- **Capital.** Capital from the society.
- **Income.** Sales by product line, and interest on member credit.
- **Costs.** Cost of goods sold, stock losses, cash over and short, and expenses by type.
- **Returns.** Profit returned to the society.

**Manual entries.** Any entry not made by the system needs a reason, a document, and approval by the treasurer and a committee member. All are listed monthly for the supervisory committee.

**Expected losses on member credit.** An allowance is held by age band and reviewed monthly.

**Tax.**

- **Trading profit.** Under the Nigeria Tax Act 2025, cooperative profit from "any trade or business" is taxable, even when customers are members.
- **Small-company relief.** A small company pays 0%. The turnover limit is either ₦50m or ₦100m, pending the signed text, and fixed assets must not exceed ₦250m.
- **VAT.**
  - The standard rate is 7.5%.
  - Businesses with ₦100m turnover or less are exempt from registering and charging.
  - Basic foods are zero-rated.
  - Shared passenger transport is exempt.
- **Withholding tax and PAYE.** Withholding tax is handled as in Module 3. PAYE applies to business staff.
- **Separate accounts.** Each business keeps departmental accounts, so the auditor can separate taxable trading income from members' income.

**Registration.** A cooperative running a supermarket, hotel or estate needs SCUML registration.

## 15. Module 11: Surplus sweep and year-end sharing

**The monthly sweep.**

1. The board sets working capital: by default, stock value plus one month's expenses.
2. At month-end, the system shows cash in the business account above that level.
3. The treasurer prepares a transfer of the excess to the main account, and two signatories approve.
4. The amount up to profit earned and not yet returned is recorded as profit returned. Only the rest is recorded as capital returned, reducing the society's investment.

**Example.** A business earned ₦260,000 in each of two months and returned nothing in the first. In the second month, ₦600,000 is swept. Of that, ₦520,000 is profit returned and ₦80,000 is capital returned. Under version 1.0, ₦340,000 would wrongly have been called capital.

**Year-end, under the Co-operative Societies Act.**

1. **Tax first.** Tax on trading profit, if due, comes off first.
2. **Reserve fund.** At least 25% of net profit goes to the reserve fund, unless the Director of Cooperatives varies it. It is never shared out.
3. **Education fund.** Up to 10% of the remainder may go to an education fund, with the Director's approval.
4. **The general meeting** decides the rest, on the committee's recommendation:
   - **dividends on shares**, which have no cap in normal years;
   - **patronage refunds** in proportion to each member's recorded purchases, only if the meeting approves them;
   - **retained surplus.**
5. **Paying out.** Refunds and dividends are recorded as shares of surplus once approved, and paid as one bulk payment with two signatures.

## 16. Module 12: Governance, alerts and the emergency freeze

**Alerts.** These go to the named role, and to the supervisory committee whenever a manager or treasurer is involved:

- a till variance above tolerance;
- three variances by one cashier in 30 days;
- voids or refunds above 5% of a till's sales;
- a sale below cost;
- shrinkage above tolerance;
- cash not banked after two business days;
- a supplier bank change;
- a purchase or payment involving a related party;
- a manual journal;
- a critical reconciliation break.

An alert cannot be dismissed without a recorded reason.

**Emergency freeze.**

- **Who can freeze.** The supervisory committee or internal auditor can freeze all outbound payments from a business account.
- **On freezing.** The board is told at once and must meet within 48 hours. The chair is reminded every day until it does.
- **Lifting.** The freeze stays until a recorded board resolution lifts it.

**Audit trail.** Every action is kept for at least five years, with the user, time, device and before-and-after values.

**Whistle-blowing.** Members and staff can report concerns to the supervisory committee, anonymously if they choose.

## 17. Module 13: Optional integrations hub

Version 2.2 rewrites this module. Integrations are now a separate, optional hub.

**The principle.** No process depends on an integration. Every process runs fully in manual mode. Each connector is a separate plug-in that a society can switch on, switch off or replace without changing its processes, approvals or records. A connector only speeds things up and removes typing. It never changes who approves what.

**How the hub works.**

- **One layer.** The hub sits between the platform's modules and outside services. Modules ask the hub for a capability, such as "send this payment" or "get today's statement". The hub uses a connector if one is switched on, or opens the manual task if not.
- **Several providers.** Each capability can have more than one provider, for example two banks, so that one can take over from the other.
- **Per-society settings.** Each society chooses its connectors in its settings. The choice is logged, and switching needs board approval.
- **Same evidence either way.** A connector's result and a manual entry produce the same record and the same ledger journal.

**Each capability, with and without a connector.**

| Capability | With a connector | Manual mode |
| --- | --- | --- |
| **Collections by transfer** | A reserved account number per till, member or employer, with notices confirmed by a status check | The business account number is shown. A sale stays "awaiting payment" until a second person confirms the credit in the bank app or on the statement |
| **Payouts** | Sent to the bank or BaaS provider after two signatures, with status checks | Approved payments are exported as the bank's bulk file, or paid by the signatories in the bank's own channel. The record is completed from the statement |
| **Bank statements** | A daily feed for Track 1 reconciliation | Statement upload for Track 2, with a maker and a checker |
| **Card terminals** | Transaction and settlement feeds from the terminal provider | The terminal's end-of-day report is entered at cash-up, and the settlement is matched on the statement |
| **Payroll check-off** | Remita or employer file exchange | The deduction schedule is downloaded as a file, and the remittance advice is uploaded |
| **Name check** | Name enquiry before each payout | A second person confirms the account name from the supplier's or member's bank document |
| **Identity checks** | BVN and NIN checks | An officer reviews and records the identity documents |
| **SMS and OTP** | Codes and notices sent by SMS | Codes in the member app. Where a member has no phone, a delivery note signed by the member and witnessed by two staff |

**Rules for every connector.**

1. A connector can be switched off at any time, and pending work moves to the manual queue without loss.
2. Notices from outside are verified, and confirmed with a status check, before anything is recorded.
3. Keys and passwords are kept in a secure vault, never in settings screens.
4. The platform never holds society money through any connector. Each society's money stays in its own bank account.
5. Each connector's health is shown on the dashboard. A failing connector alerts the treasurer and falls back to manual mode.

## 18. Module 14: The ledger, accounts and settlement

Version 2.2 rewrites this module.

**The decision.** The platform does not use TigerBeetle. It copies TigerBeetle's proven ledger rules and builds them as its own ledger module inside the platform. A review of PHP and Laravel ledger packages on 11 October 2026 found none that meets these rules:

- **bavix/laravel-wallet**, the most mature, is a wallet, not double-entry books.
- **eloquent-ifrs** has good reports, but stores amounts as decimals and has no protection against two postings racing.
- **Akaunting's licence** forbids offering it as an accounting service.
- **The rest** are unmaintained or barely used.

The module uses the brick/money library for safe naira arithmetic. The module is designed so it could later move to a dedicated ledger database without changing any process.

**What the ledger guarantees.**

- Every movement takes from one account and gives to another, and total debits always equal total credits.
- Nothing can be edited or deleted. The database itself refuses, not just the screens.
- A correction is always a new, opposite movement.
- Every balance can be rebuilt from its movements at any time.

### 18.1 How accounts are set up

1. **One book per society.** All of a society's accounts sit in its own book: the main account, its business units, its members and its suppliers. The database refuses any movement between two books, so money can never move between societies by mistake. Business units are a tag on each account, so their books stay separate in reports while capital can still move between the society and its units.
2. **A separate quantity book for stock.** Inventory counts units, not naira, in its own book for each society. Stock value is kept in the naira book of the business unit that owns the stock.
3. **Naira in kobo, whole numbers only.** Markup and interest are rounded to the kobo each month, and the last instalment absorbs the rounding. This is fixed before go-live and never changed.
4. **A fixed list of account types.**

| Code range | Type | Examples | Balance grows with |
| --- | --- | --- | --- |
| 1000 to 1999 | Assets | Bank accounts, cash in till, settlements in transit, stock, member credit owed | Debits |
| 2000 to 2999 | Liabilities | Member savings, amounts owed to suppliers, tax payable, payouts in flight, unidentified receipts | Credits |
| 3000 to 3999 | Capital and reserves | Share capital, capital from the society, reserve fund, surplus | Credits |
| 4000 to 4999 | Income | Sales, interest on member credit | Credits |
| 5000 to 6999 | Costs | Cost of goods sold, stock losses, cash over and short, bank charges, expenses | Debits |

5. **Accounts that cannot go below zero.** The database refuses any movement that would overdraw:
   - a member's savings;
   - a bank account in the books;
   - petty cash;
   - a stock line.

   It counts money already on hold. Two payouts racing each other cannot both pass, because the ledger locks the accounts involved, always in the same order, while it posts.
6. **Every account tagged.** Each account records whose it is, and which business unit and outlet it belongs to.
7. **History.** A balance is kept after every movement, so statements and month-end figures come straight from the ledger.

### 18.2 How money moves

1. **One business event, one bundle.** Each event is a bundle of movements that succeed or fail together. Examples:
   - a sale and its cost;
   - a card settlement with its fee;
   - a supplier payment with its withholding tax.

   The bundle carries one journal number and the document behind it.
2. **Numbered before sending.** Every journal gets its own number before it is posted.
   - **The same journal twice.** Posting the same journal again returns the first result and changes nothing.
   - **The same number, different contents.** It is refused and raises an alert.
   - **A new attempt.** After a refusal, such as lack of funds, any new attempt gets a new number.
3. **Holds for anything that waits.**
   - **Placing a hold.** A payout request places a hold at once. The money stays in the account, but cannot be spent twice.
   - **Completing it.** The bank's confirmation completes the hold, in full or for a smaller amount, with the rest released.
   - **Releasing it.** A failure, rejection or cancellation releases it.
   - **Expiry.** Every hold has an expiry date: 7 days for payouts. An expired hold can never be completed, even if the release job runs late. A bank success after expiry is posted as a new movement and flagged.
4. **Stock reservations are holds in the quantity book.**
   - **Approval.** When a member's request is approved, a hold is placed on the stock.
   - **Delivery.** The delivery code completes it.
   - **Cancellation.** A cancellation releases it.
   - **Expiry.** After 14 days an uncollected hold expires, and the stock is available again.
   - **Overselling.** The ledger refuses any sale or hold that would take available stock below zero.
5. **Corrections.** An opposite movement with a correction code, linked to the original journal. A journal can be reversed only once.
6. **Periods.** Each movement carries its business date. The database refuses any movement dated in a closed period. Reopening needs the board.
7. **Year-end closing.** Income and cost balances are moved to the surplus account by the ledger itself, so nobody types the figures.
8. **Daily limits.** Caps such as a member's daily withdrawal limit are checked against the day's completed and held withdrawals.
9. **The emergency freeze** blocks outgoing movements for the frozen business. Incoming money is still recorded.

### 18.3 Settlement, movement by movement

| Event | What the ledger does |
| --- | --- |
| **Transfer received** (confirmed by a status check, or by a second person in manual mode) | Bank account up, and the sale, member or unidentified receipts credited. The journal number comes from the bank reference, so a duplicate notice changes nothing |
| **Card sale** | Settlements in transit up and sales credited, with the cost of goods in the same bundle. Stock is issued in the quantity book |
| **Card settlement** | One bundle: bank up by the net amount, bank charges up by the fee, and settlements in transit down by the gross amount |
| **Wallet-terminal sweep** | Bank up, and the terminal's settlements in transit down |
| **Cash sale, then banking** | Cash in till up at the sale. At banking, bank up and cash in till down, matched to the deposit on the statement |
| **Payout requested** | A hold from the payer's account towards payouts in flight |
| **Payout confirmed**, by a connector or from the statement in manual mode | One bundle: the hold is completed, then payouts in flight go down and the bank goes down |
| **Payout failed or rejected** | The hold is released. Nothing else moves |
| **Check-off remittance** | Main bank up and "due to business unit" credited. In the same bundle, "due from society" up and member credit owed down. Both go to zero at the monthly transfer |
| **Capital tranche, profit or capital return** | One bank account down and the other up, with the society's investment and the business's capital adjusted in the same bundle |

**Reconciliation with the bank.** The ledger's balance for each bank account is compared with the statement, daily or statement by statement. The ledger lists the movements behind any figure by journal, settlement batch or business date. A difference is never adjusted away. It becomes a break and, once explained, is fixed by a correction.

### 18.4 How the ledger protects itself

- **No changes.** Records cannot be changed or deleted. The database refuses, whatever the screen or user.
- **A chain of fingerprints.** Each journal carries a fingerprint of the one before it in the same book. Any tampering breaks the chain.
- **A nightly self-check** confirms that:
  - total debits equal total credits;
  - every balance equals the sum of its movements;
  - the fingerprint chain is unbroken.

  A failure stops payouts for that book and alerts the treasurer and the supervisory committee.
- **Protected tables.** The ledger's records are written only by the ledger module. Other modules send it journals and holds through the hand-offs in the module map.
- **PostgreSQL** is the recommended database, because the protections rely on its constraints and locks.

## 19. Acceptance test checklist

The system is ready for release only when it passes every test below.

| Test | What is tested | How | Expected result |
| --- | --- | --- | --- |
| TC-01 | Capital ceiling | Release ₦12,000,000 against a ₦10,000,000 ceiling | Blocked, with the remaining ceiling shown |
| TC-02 | Account separation | Link the main savings account as the business account | Refused on the title check |
| TC-03 | Three-way match, quantity | Invoice for 100 bags when 80 were received | Payment blocked, with the 20-bag difference shown |
| TC-04 | Three-way match, price | Invoice 3% above the order price | Payment blocked. At 1.5% it passes |
| TC-05 | Anti-splitting | Three ₦180,000 orders to one supplier in 30 days | Totalled at ₦540,000. Committee approval and three quotes required |
| TC-06 | Cooling-off | Change a supplier's bank details, then pay at once | Payment held until the cooling-off ends and the change is confirmed. The changer cannot approve |
| TC-07 | Withholding tax | Pay a supplier with no TIN for goods | 4% withheld, not 2% |
| TC-08 | Blind count | Evening cash-up | Expected total hidden until both counts are in. Differing counts force a third count |
| TC-09 | Over or short | A drawer ₦2,500 over | Recorded against the cashier, and an alert sent within 15 minutes |
| TC-10 | Fake alert | Transfer sale with a notice but no status confirmation | Sale stays "awaiting payment" with no receipt |
| TC-11 | No till payouts | Look for any way to pay an expense from the till | None exists |
| TC-12 | Petty cash | Top up a ₦30,000 float with ₦18,000 of receipts | Only ₦18,000 can be approved |
| TC-13 | Card settlement | Bank POS settlement arrives net of fees | Sales cleared, fees posted to charges |
| TC-14 | Wallet terminal | Moniepoint takings with no sweep by the deadline | Break opened for the terminal and amount |
| TC-15 | Manual reconciliation independence | The maker tries to approve their own match | Refused. A different checker is required |
| TC-16 | Whole statement | Close a statement with one unmatched debit line | Cannot close. A critical break is opened, and payments are suspended |
| TC-17 | Bank proof | A ₦800 bank difference | Never written off. It stays a break until explained |
| TC-18 | Stock levels | Approve a member order for the last 2 bags of rice | Available falls to zero. The till cannot sell them |
| TC-19 | Spoilage | Write off expired oil without a photo | Refused. With approval, it posts to stock losses |
| TC-20 | Delivery code | Mark a member order delivered without the code | Refused. Stock stays reserved, and no deduction starts |
| TC-21 | Delivery independence | The picker tries to record the delivery | Refused |
| TC-22 | Price lock | Supplier price rises 6% before purchase | The member must re-confirm or cancel |
| TC-23 | Markup | Month-end on a 3-month plan | One third of the markup is recognised. The member ledger still equals the control account |
| TC-24 | Credit repayments | Employer remits all but one member's line | Matched lines move to the business. The missing line shows in arrears |
| TC-25 | Profit or capital | Sweep ₦600,000 after two ₦260,000 months with nothing returned | ₦520,000 recorded as profit returned and ₦80,000 as capital returned |
| TC-26 | Year-end | Appropriate ₦3,120,000 of profit | At least ₦780,000 to the reserve fund. The education fund is limited to 10% of the rest, with approval recorded |
| TC-27 | Cadence | Weekly cadence with a missing cash deposit on day 2 | Break raised on day 3, not at week-end |
| TC-28 | Emergency freeze | The committee freezes a venture | All outbound payments blocked. The board is told. The freeze holds until a board resolution |
| TC-29 | Ceiling increase | Raise the ceiling without a general meeting vote | Refused |
| TC-30 | Manual journal | Post a manual entry with one approval | Refused until the second approval |

**Additional tests for the ledger and integrations (version 2.2).**

| Test | What is tested | How | Expected result |
| --- | --- | --- | --- |
| TC-31 | Cross-society guard | Post a movement between accounts of two societies | Refused by the engine |
| TC-32 | Overdraft guard | Two payouts at once that together exceed a member's savings | One passes and the other is refused. The savings never go below zero |
| TC-33 | Hold and release | Request a payout, then have the bank reject it | The hold is placed at request and released on rejection. Posted balances never change |
| TC-34 | Hold expiry | Leave a stock reservation uncollected for 14 days | Released by itself. The stock is available again |
| TC-35 | Duplicate notice | Send the same bank notice twice | The second is answered "already exists". Balances unchanged |
| TC-36 | Bundle failure | A card settlement bundle where one movement is refused | Nothing in the bundle is recorded |
| TC-37 | Closed period | Post a movement dated in a locked month | Refused by the platform |
| TC-38 | Freeze | Freeze a business, then receive a deposit and attempt a payout | The deposit is recorded. The payout is blocked |

## 20. Open issues

| # | Issue | Owner | Needed by |
| --- | --- | --- | --- |
| 1 | The signed tax Act's small-company limit, ₦50m or ₦100m | Tax adviser | Before design freeze |
| 2 | Whether trading with members is taxable or mutual | Tax adviser | Before design freeze |
| 3 | The legal deduction cap in the Labour Act | Counsel | Before the pilot |
| 4 | Which POS providers to connect first, and whether wallet terminals are allowed | Partnerships | Before the pilot |
| 5 | Pilot values for every setting in section 4 | Pilot boards | Before the pilot |
| 6 | State law differences on reserves, audits and investment approval, starting with Lagos 2022 | Counsel | Before the pilot |
| 7 | When a venture must move into a subsidiary company | Counsel and finance | Release 2 |

**Ledger open issues (version 2.2).**

1. Confirm PostgreSQL for the platform database. MySQL would give weaker protections.
2. Freeze the chart-of-accounts codes, movement codes and kobo rule before the first live posting.
3. Load-test posting at payroll peak, when thousands of remittance lines post in minutes.
4. Have the ledger module independently reviewed before go-live, including the database protections and the nightly self-check.

## 21. Sign-off

Any change to these rules needs the written approval of the board and the supervisory committee.

| Name and designation | Role | Signature | Date |
| --- | --- | --- | --- |
|  | President or board chair |  |  |
|  | Treasurer |  |  |
|  | Supervisory committee chair |  |  |
|  | Lead architect |  |  |
