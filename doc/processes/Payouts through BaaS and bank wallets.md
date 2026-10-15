# Payouts through BaaS and bank wallet accounts

Oct 10, 2026 · @Abubakar

## Summary

**The model.** The bank holds each society's money in an account in the society's own name, and executes every transfer. The platform runs the business process: rules, requests, approvals and notices. It also keeps the ledger that says whose money each naira is. The platform never receives or holds society money.

**The catch the research found.**

- At every Nigerian BaaS provider checked, the platform's own API key can debit any wallet it opened. That includes Embedly, ProvidusUnity, Wema, Anchor, VFD, Kuda and Lenco.
- None of them asks the society for a PIN or OTP per payout, and none supports multiple signatories on API transfers.
- So with BaaS wallets the platform does not hold the money, but it alone controls it. The society's approval rules live only in the platform's code.

**Recommendation: three routes, picked by risk.**

1. **Signatories approve in the bank's own channel.** The platform prepares payment files as the uploader on the society's corporate banking profile, and the society's officers approve with their bank tokens. The platform cannot move money on its own. Use this for loan disbursements above a threshold, dividend runs and the pilot's first months.
2. **The platform as initiator on the society's own business account.** This applies where the provider enforces approver roles on API transfers. Kuda Business and Lenco have approver roles in their apps, but whether API transfers pass through them is unconfirmed.
3. **Fully automated BaaS wallets under the platform's key.** Approvals are enforced in the platform. Use this only for routine payouts under a per-payout and daily cap. It needs a legal opinion and a three-party agreement before go-live.

**Open banking is not the answer yet.** Payment initiation under the CBN's 2023 guidelines was not live in October 2026.

## Roles and the boundary

**The principle.** The bank is the record of money: it holds every naira and moves it. The platform is the record of meaning: it knows whose savings, which loan and which dividend each naira belongs to, and it runs the rules that decide when money may move. The society decides, through its signatories.

| Activity | Society | Platform | Bank or BaaS | Member |
| --- | --- | --- | --- | --- |
| Hold the money |  |  | **Does it** |  |
| Open the society's account and check its documents | Provides documents and signs the mandate | Collects and submits documents | **Approves the account** |  |
| Set the rules: loan policy, notice periods, dividend rate | **Decides at its board or general meeting** | Configures the rules |  |  |
| Raise a payout request | Officers raise vendor and dividend payouts | Hosts the request |  | **Raises a withdrawal or loan** |
| Check the request against rules and balances |  | **Does it** |  |  |
| Approve the payout | **Signatories approve** | Enforces who may approve | Enforces the mandate where it can |  |
| Check the beneficiary's name |  | Triggers the check | **Runs the name enquiry** |  |
| Move the money |  | Sends the instruction | **Debits and transfers** |  |
| Record whose money it is |  | **Keeps the ledger** |  |  |
| Record the balance and statement |  | Mirrors it | **Is the source of truth** |  |
| Reconcile | Treasurer signs off monthly | **Proves it daily** | Provides the statement |  |
| Monitor fraud and money laundering on the account |  | Flags unusual patterns | **Monitors and reports** |  |
| Recover a wrong payout | Approves the claim | Raises and tracks the case | **Runs the recall** |  |

**What the platform never does.**

- It never receives society money into its own account.
- It never moves money between two societies.
- It never pays itself, except invoiced fees that the society has approved.
- It never holds a signatory's own banking login or token.

## Account structure at the bank

**One society, one main account in its own name.** Collections arrive through virtual account numbers that route into it. Payouts leave from it. The account title carries the society's registered name, so a court order, an NDIC claim or a freeze touches only that society.

**Purpose wallets where the bank supports them.** Some providers let a business open separate wallets that each carry their own balance. Embedly business wallets and Kuda virtual sub-ledgers are examples. Where they exist, they let real money mirror the society's funds.

| Wallet | Holds | Why separate it |
| --- | --- | --- |
| Main or collections | Contributions as they arrive | Single place for check-off and member transfers to land |
| Loan fund | Money set aside for lending | Stops lending from eating members' withdrawable savings |
| Reserve | Statutory reserve and share capital | Protects money the bylaws say must not be paid out |
| Dividend run | The approved dividend total, moved in before the run | Caps a bulk payout at exactly what the meeting approved |
| Operations | Running costs of the society | Keeps expenses away from member money |

**Members do not get bank wallets.** A member's savings, shares and loans live in the platform ledger as balances the society owes or is owed. Giving every member a bank wallet would turn each member into a bank customer with KYC tiers, limits and fees, for no gain in safety.

**Start simple.** Begin with one main account and separate the funds in the ledger. Add real purpose wallets for the loan fund and the dividend run once the bank offers free transfers between them.

## One payout, step by step

&#91;embedded content: one member withdrawal · nine steps across five parties\]

Only step 6 moves money, and only the bank does it.

- **Steps 1 to 5.** The platform checks the rules and the balance, places a hold, collects two approvals, and sends one instruction carrying its own reference.
- **Steps 7 and 8.** The platform waits for the bank's answer before it posts the ledger.
- **Step 9.** The nightly statement proves the day.

Under the signatory-channel model described next, steps 4 and 5 happen inside the bank's own channel instead.

## Ways to authorise a payout

|  | Signatories approve in the bank's channel | Platform approval, bank executes by API | Open-banking payment initiation | Pull by mandate (Mono, Remita) |
| --- | --- | --- | --- | --- |
| **Who can move money alone** | Nobody. The bank enforces the mandate | The platform's API key | The payment provider, with the society's consent | The mandate holder |
| **How approval is proven** | Bank tokens of named signatories | Platform records, with second-factor approvals | Consent captured by the provider | The mandate set up once |
| **Speed** | Hours, because officers log in to the bank | Seconds | Seconds | Seconds once the mandate is live |
| **Bulk payouts** | Yes, by file upload | Yes, by API | Unclear | Yes |
| **Works at** | Any bank with corporate internet banking | The BaaS provider only | Not live in Nigeria as of October 2026 | Mono needs each society to be its own client. Remita needs card and OTP to add an account |
| **Main risk** | Slow and manual, with officers sharing tokens | The platform is a single point of control and fraud | Not available | Fit and onboarding friction |
| **Legal position** | Clean. The bank acts on the society's mandate | Needs a legal opinion on control and licensing | Would be clean once live | Clean, but the product does not fit |

**How the routes combine.** Payouts flow through the platform the same way whichever route executes them. The rules engine picks the route per payout from its type, its amount and the society's setup:

- under the cap, automated;
- over the cap, or a dividend run, approved in the bank's channel.

**Signatory-channel details from the banks.**

- GTBank's corporate channel form of January 2026 has uploader, reviewer and approver roles, each with its own limit, and tokens for approvers.
- FirstBank's FirstDirect has a designated-person role, and approvals must follow the account mandate.
- Access, Fidelity, UBA and Zenith have similar maker-checker setups, from secondary sources.
- No bank publishes an API that lets a partner create a pending batch for the customer to approve. The route is file upload, or host-to-host for larger clients.

## Payout workflows

**Member withdrawal.**

1. The member requests an amount in the app.
2. The platform checks:
   - the notice period;
   - the minimum balance;
   - savings pledged as loan security or as a guarantee;
   - any arrears the bylaws allow it to offset.
3. The platform places a hold on the member's ledger balance.
4. The signatories approve.
5. The instruction goes to the bank.
6. On success, the hold becomes a debit and the member is notified.

**Loan disbursement.**

1. The application collects guarantors, who confirm in the app.
2. The credit committee approves the loan, and the member accepts the offer.
3. The treasury signatories approve the payment. This is a separate approval from the credit decision.
4. The platform pays the net amount after any fees or insurance the policy deducts.
5. The platform creates the repayment schedule, and adds the first instalment to the next deduction schedule sent to the employer.

**Dividend run.**

1. The general meeting approves the rate and the total.
2. The platform computes each member's dividend from shares and patronage. It applies the tax rule that counsel confirms for cooperative dividends.
3. The treasurer moves the approved total into the dividend wallet, where one exists.
4. The signatories approve the batch as a whole and see its total, count and the largest items.
5. The bank pays in batches.
6. Failed items go to a retry queue with a reason. Members with no valid account stay owed in the ledger.

**Society vendor payment.**

1. An officer uploads the invoice.
2. Approvals follow the society's spending limits.
3. The platform deducts withholding tax where the society is required to.
4. The bank pays, and the platform files the remittance advice against the invoice.

**Refund of an overpayment.**

1. The platform confirms the credit on the bank statement.
2. A signatory approves the refund.
3. The bank returns the money to the account it came from, and to no other account.

## Ledger design

**Books per society.** Each society has its own set of books in the platform. No journal ever touches two societies.

| Ledger account | Type | Granularity |
| --- | --- | --- |
| Bank: main, and each purpose wallet | Asset | One per real bank account or wallet |
| Payouts in flight | Asset, clearing | One per society |
| Member savings | Liability | One per member, with available and held balances |
| Member shares | Equity | One per member |
| Loans receivable | Asset | One per loan |
| Interest and fee income | Income | One per product |
| Dividends payable | Liability | One per member, per dividend run |
| Unmatched receipts | Liability, suspense | One per society |
| Fees owed to the platform | Liability | One per society |

**Journals for a member withdrawal.**

| Event | Debit | Credit |
| --- | --- | --- |
| Approved and sent to the bank | Member savings | Payouts in flight |
| Bank confirms success | Payouts in flight | Bank |
| Bank confirms failure | Payouts in flight, reversed by a new entry | Member savings, restored by the same entry |
| Reversal after success | Bank | Payouts in flight, reopening the payout |

A loan disbursement debits loans receivable instead of member savings. A dividend run debits dividends payable.

**Rules.**

- Entries are never edited or deleted. A correction is a new entry that names the one it corrects.
- Each entry carries the bank reference and session ID when there is one.
- Each entry is chained by hash to the one before it.
- The ledger's bank account must equal the bank's posted balance at the end of every day.
- Payouts in flight must be zero, or must equal only the transfers still awaiting a final answer.
- Holds reduce what a member can withdraw, but they are not entries. They expire or convert, and every change is logged.

## Reconciliation and sync with the bank

**Four signals, in order of trust.**

| Signal | When | Used for | Trusted for |
| --- | --- | --- | --- |
| Webhook | Seconds after the event | Fast updates to the member and the app | Nothing on its own. It is verified by signature and then confirmed |
| Status query | On demand, and on a schedule for pending items | Settling a single payout | The final state of one payout |
| Transaction history or statement | Daily, and on demand | Proving the day | The source of truth for money |
| Balance | Hourly | Early warning of drift | Cross-checking the ledger |

**The daily three-way proof.**

1. Every payout order in the platform must match exactly one bank debit, by reference or session ID. If neither is available, it is matched by amount, date and account, and a person reviews the match.
2. Every bank line must match a ledger entry.
3. The ledger's bank balance must equal the bank's posted balance.

**Breaks and how fast they are handled.**

| Break | Severity | Owner and deadline |
| --- | --- | --- |
| A bank debit with no payout order behind it | Critical | Freeze outbound payouts for that society. The CTO and the society's chair are told within one hour |
| A payout order with no bank line after 24 hours | High | Finance runs a status query and opens a bank case the same day |
| An amount differs between order and bank | High | Finance resolves it the same day |
| A credit that matches no member | Medium | It goes to unmatched receipts and is cleared within five working days |
| The ledger balance differs from the bank's | High | No new payouts until the difference is explained |

**Monthly.** The society's treasurer signs off the bank statement against the platform's reconciliation report. The signed report is kept for the auditor.

## Provider fit in Nigeria

Researched 10 October 2026. Several bank and CBN sites blocked automated reading, so some rows rest on secondary sources. Full notes are in bank\_wallet\_payout\_authorisation.md.

| Provider | Society account in its own name | What authorises an API payout | Multiple signatories on API payouts | Statements and webhooks | Deposit cover |
| --- | --- | --- | --- | --- | --- |
| Embedly (Sterling) | Yes. Corporate onboarding needs RC number, TIN, board resolution, memorandum and articles, SCUML | The platform's key | No | Paged history. The two doc portals disagree on how webhooks are signed | Commercial bank, ₦5m |
| ProvidusUnity | Yes, through its transfer services | The platform's key on a configured account | No | Collection webhooks with fee and VAT | Commercial bank, ₦5m |
| Wema | Yes | The platform's key. Its security check calls the platform's own URL, so it is a platform control | No | Available | Commercial bank, ₦5m |
| Anchor | Yes, with an explicit cooperative-society type | The platform's key | No | The richest set, including a reversal event | Partner microfinance bank, ₦2m |
| VFD | Yes. RC number plus one director's BVN | The platform's key. Consent is taken once by OTP at opening | No | Transactions by date. Webhooks are unsigned, so reversals must be found by status query | Microfinance bank, ₦2m |
| Kuda Business | The society's own business account | Approver roles in the app. Unconfirmed for API transfers | Possibly | Available | Microfinance bank, ₦2m |
| Lenco | The society's own account, held at Providus | Approver roles in the app. Unconfirmed for API transfers | Possibly | Balance and transaction webhooks | Held at a commercial bank |
| GTBank, FirstBank and other commercial banks | The society's own corporate account | Signatories' tokens in the bank channel | Yes, by mandate | Statements in the channel. No partner API | Commercial bank, ₦5m |

**Points to settle with providers.**

- **Registration documents.** No provider confirmed that it accepts a state cooperative registration certificate without a CAC number. Many societies only have the state certificate.
- **Corporate wallet limits.** No provider publishes tier limits for corporate wallets. VFD lets the integrator change limits by API, which is one more power sitting with the platform.
- **Avoid microfinance banks.** Anchor's sample account sits at a microfinance bank, and VFD is one. That fits the earlier tabs' advice to keep society money at commercial banks.

## Regulation, liability and contracts

This section is a brief for counsel, not a legal opinion.

**Control without custody.**

- The platform never holds society money, so the CBN's holding-funds rule is not triggered in the plain sense.
- On a BaaS wallet, though, the platform alone can instruct payment from the society's money. Counsel must say whether that control:
  - needs a payment solution service provider licence;
  - makes the platform a fiduciary of the society;
  - changes who is liable when a payout is wrong.
- The signatory-channel route avoids the question, because the bank acts only on the society's own mandate.

**Open banking.**

- The CBN issued its open banking operational guidelines in March 2023, with NIBSS named to run the registry and consent system.
- The August 2025 go-live slipped. The CBN's February 2026 fintech report promised a phased roll-out from mid-2026.
- No registered payment-initiation service was found live by October 2026. Open Banking Nigeria is a standards body with over 55 members, not the registry.

**Who is liable for a wrong payout.**

- **The bank's own error.** Under the CBN's 2018 instant-transfer rules, the bank reverses its own error within one business day.
- **A wrong account given by the society.** Recovery needs the beneficiary's consent, and the receiving bank can hold the funds for up to two weeks.
- **A platform system error.** The contract must name the platform as liable for its own errors, such as a duplicate send or a wrong amount computed. The platform should carry cover for this.
- **A payout the signatories properly approved.** The society carries the result.

**What a society signs.** The pieces already exist in the banks' standard documents. GTBank's society form needs a resolution and a mandate rule, and gives the bank set-off rights across the society's accounts. Standard Chartered's resolution lets signatories delegate to named persons. No public three-party template was found, so counsel drafts one.

**Clauses for the bank, platform and society agreement.**

1. **Board resolution.** The board appoints the platform as uploader or initiator only, names the signatories, and sets the approval rule and limits.
2. **Bank's acceptance.** The bank acts only on instructions that carry the platform's approval evidence. Under the signatory route, it acts only on signatory approval in its own channel.
3. **Record access.** The society and the bank get direct, continuous access to the ledger records, including after the platform fails or the contract ends. This is the main lesson of the Synapse collapse in the United States.
4. **Exit.** On termination, the account stays with the society and the platform's access is removed within one business day.
5. **Liability.** Liability and indemnities are split as set out above, and the platform carries professional indemnity cover.
6. **Data protection.** Data protection terms apply under the 2023 Act and the 2025 directive, with BVN data kept in Nigeria.

## Risks and controls

| Risk | Route most exposed | Control |
| --- | --- | --- |
| **The platform key moves society money without a real approval** | Automated BaaS | Approvals with a second factor, enforced in code. Per-payout and daily caps per society. Larger payouts routed to the bank channel. Keys in a vault with IP whitelisting. A kill switch |
| **One insider at the platform changes rules or limits** | Automated BaaS | Rule and limit changes need two people and the society's chair, and every change is logged and shown to the society |
| **Officers share bank tokens, so one person approves alone** | Signatory channel | Token custody stated in the resolution. The platform alerts the chair on every approval it sees |
| **The ledger and the bank drift apart** | All | Daily three-way proof, unmatched lines posted to suspense at once, payouts stopped while a break is open |
| **A reversal arrives after the ledger shows success** | All | A reversed-after-success state, with polling where webhooks are unsigned or missing |
| **Provider failure or freeze** | Automated BaaS | Commercial bank only. Society-titled accounts. An exportable ledger the society can take to a new bank |
| **The platform fails or exits** | All | Record-access and exit clauses. A monthly ledger export sent to each society's treasurer |
| **Wrong beneficiary** | All | Name enquiry match, cooling-off on new accounts, recovery playbook |
| **A provider changes limits by API** | Automated BaaS | Limit changes need the same two-person rule as payouts, and they are reported to the society |

**Lessons from abroad.**

- Synapse failed because its records of who owned what could not be matched to the banks' records. The shortfall was $60m to $90m.
- A proposed US rule would make banks keep owner-level records and reconcile daily. It was still not final in October 2026.
- The design here applies the same idea anyway. Each society's account is in its own name, the ledger is proven against the bank every day, and the society can always see its own records.

## Requirements for the FRD

**FR-BPO-01 Society-titled accounts.** Every society's money sits in an account or wallet titled in the society's registered name at a commercial bank.

- The system refuses to link an account whose title does not match the society's registered name.

**FR-BPO-02 Route selection.** Each payout is routed by type, amount and the society's setup.

- Below the society's cap, it goes to the automated route.
- Above the cap, and every dividend run, it goes to the bank's signatory channel.
- The route and the reason are stored with the payout.

**FR-BPO-03 Hold at request.** A member's available balance is reduced as soon as a payout is requested, including while it waits for approval.

- A rejected or failed payout releases the hold in full.

**FR-BPO-04 Approvals.** Approvals follow the society's mandate rule, such as two of three, with a second factor at approval.

- Nobody approves a payout they raised.
- Adding or changing a beneficiary needs approval too.

**FR-BPO-05 Bank-channel payment files.** For the signatory route, the system produces the bank's bulk payment file and records its hash.

- The system matches the bank's executed lines back to the file, and flags any line that was added, removed or changed.

**FR-BPO-06 Safe API instructions.** On the automated route, every instruction uses the payout's own identifier as the reference, and every retry reuses it.

- A timeout moves the payout to pending.
- A pending payout is settled by status query, never by a new send.

**FR-BPO-07 Caps and kill switch.** The system enforces per-payout, per-member daily and per-society daily caps.

- Raising a cap needs two platform staff and the society's chair.
- One action suspends the automated route for one society or for all of them.

**FR-BPO-08 Ledger.** Each society has its own double-entry books. Entries are immutable and hash-chained, and each carries the bank reference.

- Every balance can be rebuilt by replaying the entries.

**FR-BPO-09 Daily proof.** Every day the system matches payout orders, bank lines and ledger entries, and proves the ledger's bank balance against the bank.

- A bank debit with no order behind it suspends that society's payouts and alerts the chair within one hour.

**FR-BPO-10 Society access to records.** The treasurer can export the full ledger and reconciliation at any time.

- A copy is sent automatically every month.
- The export can be read without the platform.

**FR-BPO-11 Transparency.** Signatories and the chair receive a notice for every payout, every approval, and every rule or cap change.

- The notice goes to their registered contact details, not to the person who made the change.

**FR-BPO-12 Exit.** When a society leaves, the platform's access to its account is revoked and confirmed by the bank within one business day.

- A final ledger export is delivered at the same time.

## Open questions and next steps

**For providers.**

1. **Kuda Business and Lenco.** Do API-initiated transfers pass through the approver roles set in the app?
2. **Embedly, Wema and ProvidusUnity.**
   - Can a society wallet require a second credential, held by the society, for payouts above a set amount?
   - Will you sign a three-party agreement?
3. **All providers.**
   - Do you accept a state cooperative registration certificate without a CAC number?
   - What are the corporate wallet limits?
   - How are webhooks signed?
4. **GTBank, FirstBank and Access.**
   - Can the platform be set up as uploader on a society's corporate profile?
   - Is host-to-host available at a pilot's scale?

**For counsel.**

1. Does sole technical control of society-titled wallets need a licence, or create fiduciary duties?
2. Draft the three-party agreement and the board resolution, using the clause list above.
3. What tax rule applies to cooperative dividends paid through the platform?

**Next steps.**

1. Add FR-BPO-01 to FR-BPO-12 to the FRD tab.
2. Pilot with the signatory route for every payout, and turn on the automated route below a small cap after the legal opinion.
3. Build the payment-file generator for one commercial bank first, together with the daily proof.
4. Set the caps and approval rules with the first societies' executives, and record them in each society's resolution.

Full notes are in bank\_wallet\_payout\_authorisation.md and orchestrator\_ledger\_practice.md in the research folder.
