# Operational float and BaaS settlement wallet

Oct 10, 2026 · @Abubakar

## Summary

**The answer in one line.** Keep one small operating float of the platform's own money, and never build a settlement wallet that holds society money.

- **Why a float is needed at all.** Gateways and vendors charge per event: SMS, BVN and NIN checks, transfer fees. Almost every Nigerian gateway pays those from a pre-funded balance, not from a linked bank account. So the platform needs prepaid money somewhere.
- **What the float is.** It is the platform's own working capital, sized from the busiest days of the check-off cycle. For a 20-society pilot it is in the low hundreds of thousands of naira, on the assumptions in the sizing section.
- **What it is not.** It is not a place where contributions wait, and not a pot that funds member payouts. Only banks, payment service banks and mobile money operators may hold customer funds in Nigeria. The CBN fined Paystack ₦250m in April 2025 for a wallet that held funds under the wrong licence.
- **How society money moves instead.**
  - Collections settle straight to each society's own account by sub-account or split settlement. Paystack, Monnify, Flutterwave, Interswitch, Remita and Mono all support this.
  - Payouts leave from the society's own account. Either the society's wallet at the custodian bank is debited directly, or Mono Disburse or Remita pull from the society's account under a mandate.
- **Recommended setup.**
  - The operating float sits in a Paystack balance. It is topped up through the Paystack bank account, and alerted by Paystack's warning and critical webhooks.
  - A second gateway is kept warm for failover.
  - Society accounts stay at a commercial bank, never a microfinance bank.

## The four balances and whose money each one is

People say "the float" for four different balances. They carry different owners, different legal risk and different design answers. Keeping them apart is the whole design.

| Balance | What sits in it | Whose money | Where it lives | Should it exist? |
| --- | --- | --- | --- | --- |
| **Platform operating float** | Prepaid money that pays the platform's own running costs: SMS and WhatsApp, BVN and NIN lookups, gateway transfer fees, USSD sessions | The platform company | A balance at the gateway or a platform-named bank account | Yes. It is the only true float, and it is small |
| **Society payout float** | Money pre-loaded so a gateway can pay withdrawals, loans and dividends | The society | A wallet at the gateway | Avoid it. Pay straight from the society's own bank account. If a provider forces it, use one wallet per society in the society's name, never a shared one |
| **Collection settlement balance** | Card, direct-debit and transfer collections waiting for the gateway to settle | The society, from the moment the member pays | The gateway's merchant balance until settlement | Only as transit. Settle straight to the society's account with split or sub-account settlement, and target a zero balance after each settlement |
| **Suspense** | Credits that cannot be matched to a member or a deduction line | The society | A ledger account inside the society's own bank account | Yes, but as a ledger tag only. Money never moves to a platform account to wait for matching |

**The rule that follows.** The platform may hold its own money in a float. It may not hold society money for longer than a provider's settlement cycle, and never in a balance that mixes societies. That keeps the design consistent with the "platform never holds member money" principle in the BaaS tab.

**Why the payout float is the dangerous one.** A pooled payout float is a shared pot of many societies' money in a company account. It is exactly the structure that failed at Synapse in the United States. It also exposes every society to one leaked API key, one court freeze on the platform's account, or one provider failure.

## How Nigerian providers handle balances and settlement

Compiled 10 October 2026 from provider docs, terms and press. Several provider sites block automated reading, so some rows rest on mirrors or search snippets. Confirm each before signing.

| Provider | Payouts paid from | Collections settle | Straight to a society's own bank? | Reserve or set-off terms | Low-balance alert |
| --- | --- | --- | --- | --- | --- |
| Paystack | Pre-funded balance. Top-up by checkout, free in Nigeria, or by a dedicated bank account that auto-tops up | Next working day, no fee | Yes, sub-accounts and multi-split to any bank | None published | Yes: warning and critical webhooks, email, SMS |
| Monnify | Pre-funded disbursement wallet at Moniepoint MFB | Same day at 10 PM, to bank or wallet (secondary source) | Yes, sub-accounts with income split, including on reserved accounts | 15% reserve for 90 days on USD (secondary source) | None found |
| Flutterwave | Pre-funded payout balance, topped up by virtual account | Next working day | Yes, split sub-accounts | 10% rolling reserve for 180 days, and direct debits of the merchant's bank | None found |
| Squad (GTCO) | Squad wallet | Next day, or instant to a GTBank account | Sub-merchants. Instant only to GTBank | None published | None found |
| Interswitch | Payouts wallet, topped up by virtual account | Next working day, straight to the nominated bank | Yes, split settlement | 10% of three months' settlement. Funds held up to 180 business days after exit | None found |
| Remita | No float. Bulk payouts debit the payer's own bank account after card and OTP activation | Same day for most channels, next day for cards (secondary source) | Yes, line-item split to any bank | Not found | Not applicable |
| Mono Disburse | No float. Debits the client's own bank account under a NIBSS e-mandate | Collections next day at 7 PM | Yes, split sub-accounts at any bank | Not applicable | Not applicable |
| Embedly (Sterling) | Pre-funded wallet with its own account number | Into the wallet | Not documented | Not published | None found |
| ProvidusUnity | A configured Providus settlement account | Swept net of fees once the webhook is acknowledged | To a settlement account, probably at Providus | Not published | None found |
| Kuda Business | Main account or virtual sub-ledger | Default probably not instant. Instant settlement costs extra | No. A payout is needed | Not published | None found |
| VFD (MFB) | Pool account or one wallet per customer | Swept to the pool in the pool model | No | Not published | None found |
| Lenco, Anchor | Pre-funded account at the partner bank | Into the provider account | No | Anchor auto-debits the master account for disputes and fees | None found |

**What this means for the design.**

- **Collections never need a platform balance.** Six providers can settle each society's share straight to the society's own account.
- **Payouts can avoid a float.** Paying from the society's own wallet at the custodian bank needs no float. Neither does Mono Disburse or Remita pulling from the society's account under a mandate. Mono charges ₦35 per payout plus a ₦25 to ₦35 transfer charge, and setting up the mandate takes a ₦50 authorisation transfer and up to 48 hours.
- **The platform float fits Paystack best.** It is the only provider with documented low-balance webhooks. It also has a dedicated bank account that tops up the balance automatically.
- **Set the platform's split share to zero.** On society sub-accounts, the platform takes no percentage of contributions. Fees are recovered by invoice, as the accounting section describes.
- **Read the reserve clauses.** Flutterwave and Interswitch can hold back ten per cent and debit the merchant's bank. If a society is the merchant of record, that clause reaches society money.

Full notes are in the research folder, in float\_provider\_mechanics.md.

## How money moves through the float and settlement balances

&#91;embedded content: money flow · platform float above, society money below\]

The top row is the only float, and it holds platform money. Society money moves from the payer to the society's own account, either through a gateway that settles per society or by a direct transfer to the society's virtual account. Payouts leave from the society's account after its signatories approve. The dashed line is the only flow from society money to the platform: a fee invoice the society pays.

## Sizing the float

**Formula.** Size the operating float to survive the busiest stretch without a top-up, plus a buffer.

```
target float = max(peak-day spend x peak days covered, average daily spend x normal days covered) x (1 + buffer)
reorder point = peak-day spend x days needed to top up x 1.5
```

- **Peak day.** Check-off money arrives in a few days around payday, so costs bunch. Assume one quarter of the month's cost falls on the single busiest day until real data replaces it.
- **Days covered.** Two peak days, or seven normal days, whichever is larger.
- **Buffer.** Twenty per cent while the pilot runs, cut to ten per cent after three months of data.
- **Days to top up.** One working day if a bank transfer funds the float. Weekends and public holidays need the reorder point raised before Friday.

**Worked example: a 20-society pilot.** Every unit price below is an assumption for illustration. Replace each one with the signed provider rate.

| Cost line | Monthly volume | Assumed unit cost | Monthly cost |
| --- | --- | --- | --- |
| Member SMS and WhatsApp notices | 20,000 | ₦5 | ₦100,000 |
| Gateway transfer fees on payouts | 2,000 | ₦50 | ₦100,000 |
| BVN and NIN checks for new members | 500 | ₦100 | ₦50,000 |
| **Total** |  |  | **₦250,000** |

- Peak-day spend is about ₦62,500.
- Two peak days come to ₦125,000. Seven normal days come to about ₦58,000. The larger is ₦125,000.
- With a twenty per cent buffer the target float is **₦150,000**.
- With one day to top up, the reorder point is about **₦94,000**.

**What this shows.** The platform's own float is a small working-capital line, not a treasury problem. The large sums only appear if society payouts are pre-funded. One society of 500 members disbursing ₦20m of loans a month would need several million naira sitting idle at the gateway. That is the strongest practical argument for paying out from the society's own account.

## Funding and sweeping workflows

**1. Top-up of the operating float.**

1. The system compares the live provider balance with the reorder point every hour, and after every debit webhook.
2. At the reorder point it raises a top-up request for the target float minus the current balance.
3. A finance officer prepares the transfer from the platform's operating bank account. A second officer approves it.
4. The transfer goes to the provider's funding account or the float's dedicated virtual account.
5. The credit webhook is verified, matched to the request and posted to the ledger. Unmatched credits go to the platform's own suspense, never to a society.

**2. Low-balance alerts and spend priority.**

| Level | Trigger | Action |
| --- | --- | --- |
| Amber | Below the reorder point | Raise the top-up request and notify finance |
| Red | Below one peak day | Notify the finance lead and the CTO. Pause bulk marketing messages |
| Critical | Below a quarter of a peak day | Keep only remittance notices, payout fees and KYC running. Queue the rest |
| Empty | Provider rejects for insufficient balance | Retry queued items after top-up in order of priority. Never fail a member payout silently |

**3. Sweep of collections to society accounts.**

1. Each society's collections are tagged with its sub-account or split code when the payment is created.
2. The gateway settles each society's share straight to the society's own account on its settlement cycle.
3. After the cycle, the system pulls the settlement report and checks that every collection appears in exactly one settlement.
4. Any society-tagged amount still sitting in the gateway balance after the cycle raises a break. The finance lead clears it the same day.

**4. End-of-day proof.**

1. Pull every provider balance and the day's transaction list through the API.
2. Prove each balance: opening plus top-ups minus fees and payouts equals closing.
3. Prove that society-tagged money in transit is zero, or equals only the collections still inside their settlement window.
4. Sign off, or open a break with an owner and a due date.

## Accounting and reconciliation

**Two books, never one.** The platform's float is a platform asset in the platform's books. Society money in transit is not a platform asset. It is tracked in a memorandum ledger and in each society's own books.

| Ledger account | Book | Type |
| --- | --- | --- |
| Float at provider, one per provider | Platform | Asset |
| Messaging, KYC and transfer-fee expense | Platform | Expense |
| Fees recoverable from societies, one per society | Platform | Receivable |
| Collections in transit, one per society | Memorandum | Off balance sheet |
| Collections in transit | Society | Asset, cleared on settlement |
| Unmatched receipts suspense | Society | Liability, cleared on matching |

**Standard journals in the platform's books.**

| Event | Debit | Credit |
| --- | --- | --- |
| Top-up of the float | Float at provider | Operating bank |
| Cost the platform absorbs, such as SMS | Messaging expense | Float at provider |
| Cost passed through to a society, such as a payout fee | Fees recoverable from that society | Float at provider |
| Society pays its fee invoice | Operating bank | Fees recoverable from that society |
| Interest or rebate credited by the provider, if any | Float at provider | Interest income |

**Fee recovery.** Recover pass-through costs monthly by invoice, or by a disclosed deduction the society's executive approves once in its service agreement. Do not net fees off members' contributions. That would put the platform's hand into society money and blur the audit trail.

**Daily proof.** The ledger balance for each provider float must equal the provider's reported balance at the same cut-off. A difference opens a break, keeps the hash-chained ledger untouched, and is fixed only with a correcting journal that names the break.

## Regulation and risk

Most findings below come from secondary sources and law-firm summaries, not the primary CBN circulars. Treat them as a brief for Nigerian counsel, not as a legal opinion.

**Holding funds.**

- Under the CBN licence categories of December 2020, only mobile money operators, banks and payment service banks may hold customer funds. Payment solution service providers, switches and super-agents may not.
- Pre-paying the platform's own money into a vendor or gateway balance for its own SMS, KYC and transfer fees is very likely not "holding customer funds". No CBN text says so in terms. The view rests on inference and on the UK e-money test, so it needs a counsel opinion.
- The risky structure is member or society money pooled in an account or gateway balance in the platform's name. That is the line this design does not cross.

**Settlement timing.**

- The CBN e-payment channel guidelines of 2020 require next-day (T+1) settlement to merchants for POS and web payments.
- From 1 July 2026, new accounts face instant-payment limits, including ₦20,000 in the first 24 hours. New society accounts may hit this during onboarding, so plan the first payout run after day one.
- Payment data must be stored in Nigeria by 1 January 2027. Ask each provider where settlement and balance data sit.

**Deposit insurance and provider failure.**

- NDIC pass-through cover is ₦5m per depositor at a commercial bank and ₦2m at a microfinance bank. A gateway balance is not a bank deposit, so it may carry no cover at all.
- The CBN revoked 46 microfinance bank licences effective 1 July 2026. Keep treasury and society accounts away from microfinance banks.
- Synapse in the United States lost an estimated $65m to $95m because its pooled ledger could not be reconciled to the bank. Per-society accounts and a daily proof are the defence.

**Freezes.**

- In April 2024 an EFCC order froze 1,146 accounts for 90 days across several fintechs, and four fintechs paused onboarding.
- In August 2026 a Federal High Court order put 69 accounts at 12 banks and 6 fintechs on post-no-debit after the eTranzact breach.
- A pooled float would freeze every society at once. Separate society accounts limit a freeze to the society named in the order.

**Payout fraud.**

- Flutterwave lost about ₦11bn in April 2024 through many transfers kept under review thresholds, and about ₦19bn in October 2023 through unauthorised merchant transfers.
- eTranzact lost about ₦1.39bn in July and August 2026.
- No public Nigerian case was found of a stolen API key draining a payout balance. Leaked Stripe keys elsewhere show that one key can both charge and pay out, so key scope matters.

**Interest and tax.**

- Gateways do not normally pay interest on balances. Any interest on a society account belongs to the society, and the platform must not sweep it.
- From 1 January 2026, the ₦50 electronic money transfer levy is replaced by a ₦50 stamp duty, paid by the sender on transfers of ₦10,000 or more. Transfers between a person's own accounts are exempt. Whether a top-up to the platform's own gateway account counts as an own-account transfer is untested.
- Vendor fees carry 7.5% VAT, and the platform can recover it as input VAT. Withholding tax on interest is 10%.

**What other markets require.** UK safeguarding rules in force from 7 May 2026 put client money in a statutory trust and require daily internal and external reconciliation, monthly returns, an annual audit and a resolution pack within 48 hours. That is a good benchmark for the society-money side, even though Nigerian law does not yet require it of this platform.

## Controls and requirements for the FRD

These requirements extend the FRD tab. Each one lists its acceptance criteria.

**FR-FLT-01 Separate balances.** The system keeps the operating float and each society's money in separate ledgers and separate provider accounts.

- A society payment can never credit the operating float. The posting is rejected and an alert is raised.
- No provider account holds money for more than one society.

**FR-FLT-02 Float monitoring.** The system reads every provider balance hourly and after every debit webhook.

- An amber, red or critical alert fires within five minutes of the balance crossing its threshold.
- Thresholds are calculated from the sizing formula and shown with the inputs used.

**FR-FLT-03 Maker-checker top-ups.** A top-up needs one person to prepare it and a different person to approve it.

- The same user cannot prepare and approve one request.
- Top-ups can only go from a whitelisted platform operating account.

**FR-FLT-04 Withdrawals from the float.** Money can leave the float only to a whitelisted platform account, with two approvals.

- An attempt to pay anyone else from the float is blocked and logged.

**FR-FLT-05 Spend priority.** When the float is critical, the system runs only remittance notices, payout fees and KYC.

- Lower-priority jobs queue, and they run in priority order after a top-up.
- No member payout fails silently. Every rejected payout is shown to its society with a reason.

**FR-FLT-06 Settlement proof.** After each settlement cycle, the system matches every collection to exactly one settlement to the right society account.

- Society-tagged money left at the gateway after the cycle opens a break with an owner.
- The proof report can be exported for the society's auditor.

**FR-FLT-07 Daily float proof.** Each day, opening balance plus top-ups minus debits equals the provider's closing balance.

- A difference opens a break, and a correcting journal must name that break.
- The ledger stays append-only and hash-chained.

**FR-FLT-08 Fee recovery.** Pass-through costs are charged to the right society's receivable and invoiced monthly.

- No fee is netted off a member's contribution.
- The invoice lists every charge with its date, its type and the event that caused it.

**FR-FLT-09 API key security.** Payout keys are stored in a vault, rotated on a schedule, and limited to the platform's IP addresses.

- Each key has per-transfer and daily velocity limits.
- One switch suspends all outbound payouts across providers within one minute.

**FR-FLT-10 Second provider.** The system can route fees and payouts to a second bank or gateway.

- A failover test runs every quarter and its result is recorded.

## Open questions and next steps

**Questions for providers.**

1. **Paystack.** Who keeps the main-account share on a sub-account split, and can it be set to zero? Who bears refunds after a society sub-account has settled? Is there a minimum balance, a reserve, or a 90-day transfer rule on balances?
2. **Monnify.** Confirm same-day settlement to a society's account at another bank. Confirm the sub-account settlement time and any reserve on naira collections.
3. **Custodian bank, Sterling or ProvidusUnity.**
   - Can each society wallet pay out directly under the society's own mandate?
   - Can it settle to an account at another bank?
   - What are the per-transfer fees and limits?
4. **Mono.** Which banks fail the ₦50 mandate authorisation? What is the payout limit per mandate, and how long does mandate approval take in practice?
5. **All providers.**
   - Where are balance and settlement data stored ahead of the January 2027 localisation deadline?
   - What happens to balances on suspension or a court order?

**Questions for counsel.**

1. Confirm that a platform-funded fee float is not "holding customer funds".
2. Confirm the position when a society is merchant of record and a gateway reserve clause applies.
3. Confirm that stamp duty is not charged on top-ups between the platform's own accounts.

**Next steps.**

1. Add FR-FLT-01 to FR-FLT-10 to the FRD tab.
2. Run a sandbox test on Paystack:
   - a society sub-account with a zero platform share;
   - balance alerts;
   - a refund after settlement.
3. Replace every assumed unit price in the sizing example with signed rates, then reset the target float and reorder point.
4. Build the daily float proof and the settlement proof before the pilot goes live.
