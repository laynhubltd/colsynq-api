# Member requests: commodities and fixed assets on credit

Oct 11, 2026 · @Abubakar

## Summary

**What this covers.** A member orders goods from the cooperative's business unit and pays through payroll deductions. The goods are either essential commodities, such as a bag of rice, a gallon of oil or a carton of seasoning cubes, or a fixed asset, such as an inverter, a fridge or a laptop.

**How it works.**

1. **Request.** The member picks items from a catalogue, chooses a repayment period, sees the instalments and confirms with an OTP.
2. **Check and approve.** The system checks the member's credit headroom and the item limits. Commodities within limits are approved automatically. Assets go to the committee.
3. **Fulfil.** The business unit either reserves the goods from stock, or groups requests into one purchase order and buys them.
4. **Deliver.** The member confirms receipt with a one-time code. Only then does the sale happen and the member's debt begin.
5. **Repay.** Instalments go on the next deduction schedule. Repayments are matched line by line, then moved from the society's account to the business account.

**The accounting rule that holds it together.** No entry is made at request or approval. Stock moves into the ledger when it is received, and the sale and the member's debt are recognised when the member confirms delivery. The markup is earned month by month as interest, not on the day of sale. Every member balance sits in a sub-ledger that must add up to the control account every day.

## Two kinds of request

|  | Essential commodity | Fixed asset |
| --- | --- | --- |
| **Examples** | Rice, oil, seasoning cubes, sugar, beans, semovita, detergent | Inverter and batteries, fridge, freezer, laptop, phone, motorcycle, furniture |
| **Typical value** | Tens of thousands of naira | Hundreds of thousands of naira |
| **Repayment period** | 1 to 6 months (configurable) | 6 to 24 months (configurable) |
| **Markup** | Set by the scheme, for example 5% flat (configurable) | Set per tenor, for example 10% flat over 12 months (configurable) |
| **Approval** | Automatic within headroom and item limits | Business committee. Board above a set value |
| **Security** | The member's savings and deduction consent | Savings, a guarantor above a set value, and the society keeps title until paid |
| **Fulfilment** | From stock, or a monthly bulk order window | Bought to order, often delivered by the supplier |
| **Quantity limits** | Per item per month, for example 2 bags of rice | One of each asset type at a time |

## Who does what

| Actor | Duties |
| --- | --- |
| Member | Builds the request, confirms the terms and the deduction consent, receives the goods and confirms delivery |
| Guarantor | Confirms the guarantee in the app for asset requests above the threshold |
| Business manager | Manages the catalogue and prices within the margin policy, groups requests into purchase orders, and plans deliveries |
| Business committee | Approves asset requests and bulk purchase orders above the manager's limit, and settles disputes |
| Storekeeper | Receives goods, reserves and issues stock, and prepares pick lists and delivery notes |
| Delivery officer | Hands over the goods and collects the member's delivery code. Never the same person as the storekeeper for that delivery |
| Treasurer | Pays suppliers, and moves matched repayments to the business account |
| Payroll officer at the employer | Applies the deductions and remits them |
| Supervisory committee | Reviews prices against the market, delivery records and arrears |

## The request lifecycle

&#91;embedded content: member request lifecycle · eleven states\]

**What happens in each state.**

- **Submitted.** The member has confirmed items, quantities, tenor and instalments with an OTP. The deduction consent is attached.
- **Checked.** The system tests every rule in the next section. A failure goes to Rejected with the reason shown to the member.
- **Approved.** Prices are locked for a set period (configurable, default 14 days). Stock is reserved if available. The member can still cancel at no cost until purchase.
- **Fulfilled.** The goods are either reserved from stock and picked, or bought on a purchase order and received.
- **Delivered.** The member enters the delivery code sent to their phone. The sale, the cost of sale and the member's debt are posted at this moment.
- **Repaying.** Instalments appear on each deduction schedule until the balance is zero, then the request moves to Closed.
- **In arrears.** A missed or short deduction ages the arrears and escalates it. It returns to Repaying when cleared. At exit, arrears are offset from savings.
- **Returned.** Faulty or wrong goods reported within the return window reverse the sale. The goods go back to stock or to the supplier.

## Eligibility and approval rules

| Rule | Default (all configurable) |
| --- | --- |
| Minimum membership | 6 months as a contributing member |
| No arrears | No unpaid deductions or loan arrears on the member's account |
| Credit headroom | Commodity and asset balances plus loans must not exceed a multiple of savings, for example two times |
| Deduction cap | All of the member's monthly deductions together must stay within the cap set in the core FRD |
| Item limits | Per item per month, such as 2 bags of rice and 2 gallons of oil. One asset of each type at a time |
| Guarantor | Required for asset requests above ₦250,000. The guarantor's own headroom is reduced by the guaranteed amount |
| Employment | The member is on an active payroll with a remitting employer |
| Approval | Commodities within limits are approved automatically. Assets go to the business committee, and anything above ₦1,000,000 goes to the board |

**When demand exceeds supply.** If a scarce item is over-requested, the system rations it per member in the order requests were approved. It publishes the allocation to all members, so no officer can favour friends.

**When prices change.** Prices are locked at approval for the lock period. If the supplier's price rises by more than a set share before purchase (configurable, default 5%), the member must re-confirm the new instalments or cancel.

## Fulfilment

**From stock.**

1. At approval, the requested quantity is reserved against available stock. Reserved stock cannot be sold to anyone else.
2. On delivery day, the storekeeper prints a pick list by member and packs each order with its delivery note.
3. An unclaimed reservation expires after a set period (configurable, default 14 days). The request is cancelled or rescheduled, and the stock is freed.

**Bought to order.**

1. Approved requests are grouped by item and supplier into one purchase order per buying window. Commodities use a monthly window. Assets are bought as approved.
2. The purchase follows the business purchase-to-pay process: quotes, purchase order, goods received and three-way match.
3. The goods-received note records which member requests each line serves.
4. For assets delivered by the supplier straight to the member, the storekeeper inspects the item on delivery. The member's delivery code stands in for the goods-received note.

**Delivery.**

1. Delivery is by collection at a distribution point on a set day, at the member's office, or at home for assets.
2. The delivery officer checks the member's identity, hands over the goods and asks for the code sent to the member's phone. Photos are attached for assets.
3. A member can name a proxy in the app. The proxy gets the code instead.
4. Shortages or damage noticed at handover are recorded on the delivery note. Only what was delivered is posted.
5. Assets carry a serial number on the delivery note, and the society's ownership is recorded until the last instalment.

## The ledger, entry by entry

The illustrative prices below are assumptions, not market quotes.

**A commodity request.** A member orders:

- one 50 kg bag of rice at a cash price of ₦82,000, costing the business ₦76,000;
- one 25-litre gallon of vegetable oil at ₦58,000, costing ₦54,000;
- one carton of seasoning cubes at ₦14,000, costing ₦12,500.

The cash price is ₦154,000 and the cost is ₦142,500. The member takes 3 months at 5% flat, so the markup is ₦7,700, the total repayable is ₦161,700, and the monthly instalment is ₦53,900.

| Step | Book | Debit | Credit | Amount |
| --- | --- | --- | --- | --- |
| Request and approval | None. A commitment and a stock reservation only |  |  |  |
| Goods received, if bought to order | Business | Stock | Suppliers payable | ₦142,500 |
| Supplier paid, with 2% withholding tax where it applies | Business | Suppliers payable | Business bank ₦139,650 and withholding tax payable ₦2,850 | ₦142,500 |
| Delivery confirmed: the sale | Business | Member credit receivable | Sales | ₦154,000 |
| Delivery confirmed: the cost | Business | Cost of goods sold | Stock | ₦142,500 |
| Month 1 markup earned | Business | Member credit receivable | Interest income | ₦2,567 |
| Month 1 deduction lands in the society's account | Society | Main bank | Due to business unit | ₦53,900 |
| The same deduction, in the business books | Business | Due from society | Member credit receivable | ₦53,900 |
| Monthly transfer to the business | Society | Due to business unit | Main bank | ₦53,900 |
| The same transfer, in the business books | Business | Business bank | Due from society | ₦53,900 |

**The member's balance.**

| Point | Markup added | Repayment | Balance |
| --- | --- | --- | --- |
| After delivery |  |  | ₦154,000 |
| Month 1 | ₦2,567 | ₦53,900 | ₦102,667 |
| Month 2 | ₦2,567 | ₦53,900 | ₦51,334 |
| Month 3 | ₦2,566 | ₦53,900 | ₦0 |

The business earns ₦11,500 of gross margin at delivery and ₦7,700 of interest over three months.

**An asset request.** A member orders an inverter set at a cash price of ₦480,000, costing ₦440,000, over 12 months at 10% flat.

- The markup is ₦48,000 and the total repayable is ₦528,000.
- The instalment is ₦44,000 a month, and ₦4,000 of markup is earned each month.
- A guarantor is required because the value is above ₦250,000.
- The supplier delivers to the member's home. On the member's code and the storekeeper's inspection, stock is recorded and immediately issued: debit stock and credit suppliers payable, then the sale and cost entries as above.

**Accounting notes for the auditor.**

- **Revenue.** It is recognised at delivery, when the member takes control of the goods. Keeping title is security, not a reason to delay the sale.
- **Markup.** It is a financing component, earned over the term. Straight-line recognition is shown here. The system also supports the effective interest method if the auditor requires it.
- **Expected credit losses.** An allowance is held against member credit receivable by ageing band, and reviewed monthly.
- **Early settlement.** It cancels the unearned markup, with any discount the scheme allows.
- **A return within the window.** It reverses the sale, the cost and any markup earned. The goods go back into stock at cost, or to the supplier against a credit note.

## Balances and checks

| Check | Must hold | When |
| --- | --- | --- |
| Member sub-ledger | The sum of all members' credit balances equals the member credit receivable control account | Daily |
| Stock | Available plus reserved stock by item equals the stock record. The stock record equals the stock account at cost | Daily, and at each count |
| Clearing between books | "Due to business unit" in the society's books equals "due from society" in the business books, and both are zero after the monthly transfer | Monthly |
| Deductions | Every instalment on the schedule is either matched to a remittance line or sits in arrears | Monthly |
| Deliveries | Every delivered request has a sale, a cost entry and a delivery code. No sale exists without a delivery | Daily |
| Approved but undelivered | Requests approved more than the lock period ago without delivery are listed for action | Weekly |

## What the member sees

- **Ordering.** A catalogue with cash prices, the markup for each tenor, and the monthly instalment before confirming.
- **Tracking.** Status updates by SMS and in the app at each state, with the delivery date and place.
- **Collection.** The delivery code arrives by SMS on delivery day.
- **Statement.** For each request: delivered items, the balance, instalments paid and due, and the markup earned to date.
- **USSD.** Members on USSD can check headroom, place a repeat commodity order and see their balance.

## Risks and controls

| Risk | Control |
| --- | --- |
| Officers inflate prices | Three quotes on bulk orders, a monthly market price check by the supervisory committee, and margins visible to members |
| Requests made in a member's name without their knowledge | OTP at submission and a delivery code at handover, both sent to the member's registered phone |
| Goods signed out but never delivered | No sale without a delivery code, and the storekeeper never delivers their own picks |
| Favouritism when goods are scarce | Rationing by approval order, published to all members |
| Prices rise after approval | Price lock with a re-confirmation trigger |
| Stock reserved and never collected | Reservation expiry, and a weekly list of approved but undelivered requests |
| The employer deducts but does not remit | Line-by-line matching, arrears ageing and escalation from the core FRD |
| A member leaves owing | Offset from savings at exit, and a guarantor for assets |
| An asset is sold on before it is paid off | Title kept by the society, serial number recorded, and the guarantor notified |
