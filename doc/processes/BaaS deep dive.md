# BaaS deep dive: integrations, transfers and money movement

Oct 10, 2026 · @Abubakar

## Summary

Banking-as-a-Service in Nigeria is mature for collections and payouts and immature for custody and control. Around 40 banks, MFBs and middleware providers offer virtual accounts, transfers and webhooks; almost none offers real accounts in a third party's name with bank-enforced multi-approver release, and none publishes an uptime SLA.

- **Money moves on NIP**: credited in seconds, settled between banks later through CBN RTGS. Fees are ₦10–50 per transfer plus a ₦50 sender-paid duty on transfers of ₦10,000 or more since 1 January 2026; same-bank book transfers avoid both. Unknown outcomes (09, 96, timeouts) must be status-queried, never re-sent; reversals can take 24–72 hours.
- **Integrations needed**: corporate accounts, virtual accounts, credit webhooks, name enquiry, single and bulk transfers, status query, statements, direct debit and KYC lookups. Cards, bills and FX are not needed.
- **Structure matters most**: a dedicated account in each society's name at a commercial bank keeps the bank as the ledger of record and gives ₦5m of deposit cover; pooled middleware accounts make a sub-ledger the only record, which is how Synapse lost $65–96m of customer money in 2024.
- **The market is shifting**: payment companies (Paystack, Flutterwave, Nomba, Moniepoint, Kuda, Paga) now own MFBs, Providus has become ProvidusUnity, Titan was absorbed into Union, Mono was bought by Flutterwave, Okra closed and Bloc is dormant. Always ask which legal entity holds the funds.

**Recommendation:** custody at a commercial bank (Sterling via Embedly or ProvidusUnity first, Wema as the partnership alternative) with each society's accounts in its own name; employer virtual accounts at the same bank; a gateway (Paystack or Monnify) for direct debit; a second bank contracted for failover. Enforce two-of-three approvals in the platform and negotiate bank-side mandate checks, because no Nigerian API offers them today.

## BaaS models explained

Banking-as-a-Service is a licensed institution renting its licence, accounts and payment rails to a non-bank through APIs. In Nigeria four models are in use. They differ in one question that matters more than any feature: who holds the licence, who holds the money, and whose ledger is the truth.

| Model | How it works | Licence holder | Money sits in | Ledger of record | Examples (Nigeria) |
| --- | --- | --- | --- | --- | --- |
| **1. Bank-direct API** | The platform integrates straight with a deposit-money bank's API: accounts, virtual accounts, transfers, statements, webhooks | The bank | Accounts at the bank, in the customer's or platform's name | The bank's core banking system | Providus, Wema, Sterling, Access, GTBank, UBA, Polaris |
| **2. MFB as sponsor** | A microfinance bank (or MFB-backed fintech) provides the same APIs, usually with wallets and tiered KYC | The MFB | Wallets or accounts at the MFB | The MFB's core | VFD MFB, Kuda, Moniepoint MFB, Rubies, 9PSB |
| **3. BaaS middleware** | A technology company aggregates one or more sponsor banks behind a single API and adds ledgering, KYC orchestration and dashboards | The sponsor bank(s) | Usually a pooled account at the sponsor in the middleware's or platform's name, with sub-accounts or virtual accounts | Often the **middleware's** ledger, reconciled to the bank's pooled balance | Anchor, Embedly, Maplerad, Blusalt |
| **4. Payment gateway as BaaS-lite** | A licensed payment processor offers collections (dedicated virtual accounts), payouts and transfers; not full accounts | The PSP plus its settlement bank | The PSP's settlement account until paid out | The PSP's ledger | Paystack, Flutterwave, Monnify, Fincra, Budpay |

### Why the model matters for this platform

- **Customer funds must be in the society's name.** Model 1 can open a corporate account for each society. Models 3 and 4 typically pool money in one account and keep the per-customer split in their own ledger, which reintroduces the very risk the platform exists to remove: if that ledger is wrong or the middleware fails, the society's balance depends on someone else's records (the Synapse failure in the US, 2024, is the cautionary case).
- **Mandate enforcement lives only in model 1.** A bank can restrict a society's account to two-of-three approvals; a middleware's pooled account cannot enforce one society's signatory rule at the bank.
- **Virtual accounts and payouts are commodities in all four.** Collections (an employer paying into a virtual account) and transfers out work in every model, at different prices.
- **Deposit insurance follows the account holder.** A society's own account at an insured bank is covered up to the NDIC per-depositor limit; a pooled account is covered per beneficiary only if NDIC's pass-through conditions on records are met.

The working conclusion before the provider research: use model 1 for custody (each society's operating and project accounts at a bank) and use models 1 or 4 for virtual-account collections and direct debit, choosing on price and API quality. Avoid any design where the platform's or a middleware's ledger is the only record of whose money is whose.

## Provider catalogue

About 40 providers expose banking by API in Nigeria. Virtual accounts and payouts are available from almost all of them; real accounts in a third party's name, statement APIs, strong webhook signing and published prices are much rarer. Prices are ex-VAT and as published in October 2026; "Nego." means negotiated.

### Banks, MFBs and payment service banks (bank-direct)

| Provider | Licence | Key products | Webhook signing | Published pricing | Client licence needed? | Status (Oct 2026) |
| --- | --- | --- | --- | --- | --- | --- |
| **ProvidusUnity** (ex-Providus, renamed 26 Jun 2026) | Commercial bank | Static and dynamic virtual accounts, Xpress Wallet, payouts, name enquiry, statements | **Static** hash of client credentials, not over the payload | VA negotiated; NIP ₦10/25/50 | Sales call | Operating; post-merger integration |
| **Wema** (ALAT API, CoopHub) | Commercial bank | Wallets and account creation, static VAs, payouts, statements; CoopHub cooperative app (no API) | None documented | Negotiated | Sales call | Operating; reported ₦8.56bn hack in 2025 |
| **Sterling / Embedly** | Commercial bank | Organisation, business and customer wallets, VAs, single and bulk payouts, bulk name enquiry, cards, overdraft | **Svix HMAC-SHA256** with timestamp | Payout ₦15/35/55; funding 0.5% capped ₦250; BVN ₦65, NIN ₦55 | No (business due diligence) | Operating |
| **VFD (VBaaS)** | MFB (tier unverified) | Individual and **corporate accounts** (RC number + director BVN), VAs, payouts, statements, loans | **None**; IP whitelist | ₦10 out; 0.3% capped ₦500 in (earlier note) | No (due diligence + certificate) | Operating; not among the 2026 revocations |
| Access / Hydrogen | Commercial bank / processor | Transfers (incl. cross-border), statements; VAs via Hydrogen | Not documented | Hydrogen VA 0.2% capped ₦200 | No | Operating |
| GTBank / Squad | Processor + PSSP | Static and dynamic VAs, payouts, card and account mandates | **HMAC-SHA512** | VA 0.25% capped ₦1,000; transfer ₦8–40 | "CBN profiling approval" for business VAs | Operating |
| Zenith | Commercial bank | MT940/SWIFT statements, Z-Mandate direct debit, GlobalPAY checkout | Not documented | Negotiated | Sales call | Operating |
| UBA Developer Zone | Commercial bank | Linked VAs, payouts, statements, direct debit | Not documented | Tiered (Gold/Silver/Bronze) | **Yes: PSSP or MMO licence plus UBA corporate account** | Operating |
| FCMB / ROVA | **FCMB MFB** (so MFB cover and limits) | Wallets, VAs, payouts, KYC | Not documented | Negotiated | Sales call | Operating |
| Fidelity PayGate+ / Polaris VULTe | Commercial banks on **OnePipe** | VAs (Pay with Transfer), payouts, statements, direct debit | MD5 request signature | Negotiated | Invite after approval | Operating; Polaris recapitalisation unresolved |
| Kuda Business API | National MFB | Collection accounts, VAs, single and bulk payouts, statements | Not documented | VA 0.3% capped ₦80; NIP ₦9–15 | No | Operating; layoffs Mar 2026 |
| Safe Haven MFB | MFB | Static and time-bound sub-accounts, payouts, BVN/NIN/CAC checks | OAuth2 JWT client assertion | ₦9,000 corporate onboarding (secondary) | No | Operating |
| 9PSB | Payment service bank | Wallet-as-a-service, VAs, payouts (no lending allowed) | Not documented | Negotiated | Unknown | Operating; underlies Anchor |
| Rubies, Globus, Parallex, Union (ex-Titan) | Various | Wallets, VAs, transfers via partners | Mostly not documented | Mostly negotiated | Mixed | Operating; Rubies removed from the CBN switch in 2021; Union recapitalisation extension; Parallex runs on SeerBit |
| Paystack MFB, Flutterwave MFB, Nombank, Moniepoint, Paga | New or upgraded MFBs (2025–2026) | Business banking, VAs planned or live | — | — | — | **Payment companies are becoming banks** |

### Middleware and payment gateways

| Provider | Account bank | Key products | Webhook signing | Published pricing | Status |
| --- | --- | --- | --- | --- | --- |
| **Paystack** | Wema; Titan (now Union, legal holder to confirm) | Dedicated VAs, transfers single and bulk, name enquiry, direct debit, split subaccounts, transfer-approval callback | **HMAC-SHA512**, 3 fixed source IPs | DVA 1% capped ₦300; payouts ₦10/25/50 + ₦50 duty | Operating; NIBSS-driven outages Jan, Feb, Jun 2026 |
| **Monnify** (Moniepoint) | Moniepoint, Wema, Sterling, Rolez MFB | Reserved accounts, payouts up to 5,000 per batch, mandatory name enquiry (from Apr 2026), NIBSS mandates, income split | HMAC-SHA512 (conflicting reports) | Collect 1.5% capped ₦2,000; payout ₦10/20/40 | Operating; live payouts need emailed OTP and whitelisted IP |
| Flutterwave | Not disclosed; own MFB from Apr 2026 | VAs, payouts, sub-accounts, FX | v3 **static** verif-hash; v4 OAuth2 | Collections 2% uncapped; transfer ₦10/25/50 | Operating; NIBSS payout incident 8 Oct 2026 |
| **Mono** (Flutterwave-owned since Jan 2026) | Client's own bank account (Disburse) | Account data 1–12 months, DirectPay, direct debit, **Disburse** (pays from the client's own account, Mono never holds funds) | Shared secret, not HMAC | Data ₦50k/month; Disburse ₦25/30/35 + ₦35 | Operating |
| Anchor | 9PSB and unnamed partners | Deposit and sub-accounts, VAs, payouts (book transfers free), cards | HMAC-SHA1, no timestamp (weak) | Sales | Operating |
| Lenco | Providus | Accounts, sub-accounts, VAs, payouts, statements, bills | HMAC-SHA512 | In free; out ₦43.50; VA 0.5% capped ₦150 | Operating |
| Maplerad | Unnamed | Wallets, NGN VAs, payouts, USD cards, FX | Svix HMAC-SHA256 | Free in, ₦10 out | Operating; card outage 9 Oct 2026 |
| Interswitch | Wema (VA option) | VAs, payouts, push-to-card, bills | HMAC-SHA512 | Quickteller Business 1.5% capped ₦2,000 | Operating |
| Remita (SystemSpecs) | Any bank | Payroll, bulk, mandates, salary data referencing | Per-request SHA512 hash | Not current | Operating; ₦182.77bn PAC refund demand (Apr 2025) |
| Fincra, BudPay, Nomba, Payaza, Juicyway, Woven, Blusalt | Various | VAs, payouts, FX | Mixed | Mostly sales | Operating (Woven's shutdown was retracted; partners VFD and Globus) |
| Bloc, Okra, Brass | — | — | — | — | **Bloc dormant; Okra closed May 2025; Brass absorbed into Paystack MFB** |

### Cheapest published rails

- **Inbound to a virtual account**: Kuda 0.3% capped ₦80; Lenco 0.5% capped ₦150; Hydrogen 0.2% capped ₦200; Embedly 0.5% capped ₦250; Paystack 1% capped ₦300; Squad 0.25% capped ₦1,000; Monnify 1.5% capped ₦2,000; Flutterwave 2% uncapped.
- **Payout**: Squad from ₦8; Kuda ₦9–15; VFD and Maplerad ₦10; Paystack, Flutterwave, Monnify ₦10–50; Embedly ₦15–55; Lenco ₦43.50; Mono Disburse ₦60–70 all-in. Add the ₦50 duty on transfers of ₦10,000 or more.

### Five things the catalogue shows

1. **Ask which legal entity holds the funds.** Payment companies now own MFBs; balances moved onto them drop from ₦5m to ₦2m of deposit cover and carry licence-revocation risk.
2. **No provider publishes an uptime SLA.** Recent Paystack and Flutterwave outages traced to the NIBSS central switch; switching aggregator does not remove that risk, only dual-provider failover reduces it.
3. **No API offers bank-side n-of-m approval.** The nearest are Paystack's transfer-approval callback, Monnify's OTP and IP whitelist, and Duplo's portal-based approval layer on Wema ALAT Business (Jul 2026).
4. **Webhook security is uneven.** Only Svix-signed (Embedly, Maplerad) and HMAC-SHA512 providers bind the signature to the payload; for all others, re-query the transaction before crediting the ledger.
5. **Hidden processors.** OnePipe sits behind Fidelity and Polaris, SeerBit behind Parallex; due diligence must cover them.

## Integrations: the API surface the platform consumes

The platform needs eleven integration capabilities. The table names each, what the platform does with it, and which providers offer it today.

| Capability | What the platform does with it | Typical API calls | Providers offering it | Notes |
| --- | --- | --- | --- | --- |
| **Corporate account opening** | Open each society's operating and project accounts in the society's name | Create corporate client; attach signatories; KYC consent | VFD (corporate endpoint, RC number + director BVN; cooperative number unverified); bank-direct desks (Providus, Wema, Sterling) by arrangement | The custody foundation; post-no-debit until BVN consent on VFD |
| **Virtual accounts** | One per employer per society; identifies who paid | Create static VA linked to parent; list; close | Almost all (Providus, Wema, VFD, Kuda, Squad, Embedly, Paystack DVA, Monnify reserved accounts, Lenco) | Price 0.2–2% per inbound; caps ₦80–₦2,000 |
| **Inbound credit webhooks** | Post employer remittances to unapplied | Webhook with session ID, settlement ID, payer, amount, narration | All VA providers | Signing strength varies; acknowledge to release settlement (Providus) |
| **Name enquiry** | Verify beneficiary before payout | Name enquiry by account and bank code | Most; mandatory on Monnify from Apr 2026 | Show verified name to approvers |
| **Single transfers** | Loan disbursements, withdrawals, refunds | Initiate with idempotency key; status query | All | Handle 00 / 09 / 96 / 25 / 51 / 91 |
| **Bulk transfers** | Dividends, bulk refunds | Bulk upload; per-item status | Paystack, Monnify (5,000 per batch), Kuda, Embedly, Mono Disburse (1,000 per batch), Anchor | Approve once over a frozen list |
| **Transaction status query** | Resolve unknown outcomes | Query by session ID or reference | All | Never re-send on unknown |
| **Statements** | Daily reconciliation | Transaction history by date; MT940 | Providus, VFD, Access, UBA, Polaris, Kuda, Lenco; Zenith MT940; Mono for third-party accounts (₦50k/month) | Source of truth over webhooks |
| **Direct debit** | Member fallback during employer default | Create mandate; activate; debit; cancel | NIBSS via Paystack, Monnify, Mono, Squad, Zenith Z-Mandate, UBA, Fidelity, Polaris | ₦50 activation transfer; ₦30 per debit plus mark-up |
| **KYC lookups** | Verify members and signatories | BVN, NIN, CAC lookups | Embedly (BVN ₦65, NIN ₦55), Safe Haven, VFD, Fidelity, Kuda | Required for every signatory under the Dec 2023 circular |
| **Approval and control** | Two-of-three release | Transfer approval callback; OTP; IP whitelist | Paystack (approval callback), Monnify (OTP + IP), Duplo on Wema (portal) | No bank-side n-of-m API exists; enforce in platform plus negotiate mandate checks |

### Not needed for this platform

Cards, bills and airtime, FX and cross-border, lending APIs. They exist at many providers but add licence, fraud and support burden without serving the cooperative use case.

### Integration rules

- One adapter per provider behind a common interface (accounts, VAs, transfers, status, statements, webhooks, direct debit), so a second provider can be added without touching the ledger.
- Verify every webhook: HMAC where offered; otherwise source-IP allow-list plus a status re-query before posting.
- Deduplicate every inbound event on the provider's settlement or session ID.
- Idempotency key on every outbound call; status query before any retry.
- Statements win over webhooks; reconcile daily.

## Transfer capabilities

Almost everything this platform moves goes over NIBSS Instant Payment (NIP): 24/7, credited in seconds, settled between banks later in batches. Intra-bank book transfers are faster and cheaper still. Everything else is either for high values (RTGS), for batches that can wait (NEFT), or for pulling money (direct debit).

### Rails

| Rail | What it is | Speed | Use in this platform |
| --- | --- | --- | --- |
| **NIP** (NIBSS Instant Payment) | Name enquiry, then funds transfer, then status query on the same 30-character session ID | Real-time credit, 24/7 | Loan disbursements, withdrawals, refunds, dividends to other banks; employers' remittances in |
| **NPS** (NIP's ISO 20022 successor) | Adds Confirmation of Payee, Request-to-Pay, purpose codes; first live 7 Nov 2025; 26.55m transactions worth ₦1.4tn across 48 institutions by Aug 2026; no NIP retirement date | Real-time | Watch: Confirmation of Payee and structured remittance would help reconciliation |
| **Intra-bank book transfer** | Debit and credit inside one bank | Instant | Member and society at the same bank: no NIP fee, and per bank notices no stamp duty |
| **NEFT** | NIBSS batch credit transfer | Next business day | Not needed |
| **RTGS** (CBN, replaced CIFTS Dec 2013) | High-value, with a minimum and no maximum | Same day, in operating hours | Large estate or land payments above NIP limits |
| **NIBSS Direct Debit** | Fixed, variable or global standing instruction (GSI, loan recovery, licensed creditors only) mandates; activated by a ₦50 transfer from the payer, usually within minutes | Per mandate | Member fallback when an employer stops remitting |
| **Bulk transfers** | Bank or NIBSS bulk upload; NIBSS Pay ₦15 per record, up to 100k items | Minutes to hours | Dividends and refunds |

### NIP response codes the platform must handle

| Code | Meaning | Platform action |
| --- | --- | --- |
| 00 | Success | Post the journal |
| 09, 96, timeout | Unknown or pending | Query status on the same session ID; **never re-send** |
| 25 (on status query) | NIBSS has no record | Treat as not sent; safe to retry with a new instruction |
| 51 | Insufficient funds | Fail; reopen the payment |
| 91 | Beneficiary bank unavailable | Fail; retry later as a new instruction |

A transfer can report success and later be reversed; the ledger must allow a successful-then-reversed state.

### Limits

| Limit | Value |
| --- | --- |
| E-channel cap, companies | ₦250m per day (customer indemnity required above ₦10m) |
| E-channel cap, individuals | ₦25m per day (indemnity above ₦1m) |
| Default corporate NIP per transaction | ₦10m, ₦250m with indemnity (bank-dependent) |
| KYC Tier 1 | ₦50k per day; ₦300k balance cap |
| KYC Tier 2 | ₦200k per day; ₦500k balance cap (bank-account figure unverified) |
| Tier 3 mobile-money wallets | ₦5m per day |
| Accounts without BVN or NIN | Blocked for debits and credits since Mar 2024 |

### Fees

| Fee | Amount | Who pays |
| --- | --- | --- |
| NIP, 2020 caps | ₦10 (≤₦5k), ₦25 (≤₦50k), ₦50 (>₦50k) | Sender |
| NIP, 2026 draft guide | Free under ₦5k; ₦10 for ₦5k–50k; ₦50 above; effective 1 May 2026 if final (press conflicts) | Sender |
| NIBSS charge to banks | About ₦3.75 per transfer | Bank |
| Electronic money transfer levy (stamp duty) | ₦50 on transfers of ₦10,000 or more; exempt below ₦10k, salary payments, same-bank transfers | **Sender, from 1 Jan 2026** |
| Gateway payouts (e.g. Paystack) | ₦10 / ₦25 / ₦50 plus the ₦50 duty | Platform or society |
| VAT | 7.5% on provider fees | Payer of the fee |

### Reversals and disputes

- Industry target: instant or within 24 hours; regulatory maximum 72 hours; NIBSS's dispute system gives banks 72 hours. Providers quote 24–72 hours in practice. A 2018 CBN fine of ₦10k per failed NIP transfer not reversed within 24 hours may no longer apply (unverified).
- The February 2026 30-second refund rule covers failed airtime and data purchases only, not bank transfers, and was a draft.
- Fraud (CBN, Nov–Dec 2025): customers report within 72 hours; investigation up to 14 working days; refund within 48 hours after. A negligent bank pays in full; otherwise the sending and receiving banks split 50/50; the CBN can have NIBSS hold settlement or debit receiving banks.

### Cost design

Keep societies and, where possible, members at the same bank: book transfers avoid both the NIP fee and the ₦50 duty. For a ₦100,000 loan disbursement to another bank, budget about ₦50 NIP fee plus ₦50 duty plus VAT on any provider fee.

## Money movement

&#91;embedded content: money movement · inbound and outbound, 7 steps\]

A customer is credited within seconds on NIP, but banks settle with each other afterwards, in batches through CBN RTGS, backed by T-bill collateral of 110% of each bank's largest past net debit (the number of daily windows is reported as 2, 4 or 12; unverified). For this platform that means three things:

- **Inbound** (steps 1–3): the society's balance at the BaaS bank rises when NIP credits it; the platform learns from the bank's webhook, deduplicates on the settlement ID, and posts to unapplied remittances. If the webhook never arrives, the next statement catches it.
- **Outbound** (steps 4–6): money leaves the society's own account only on a signed, two-approval instruction; the bank runs name enquiry, the mandate check and NIP; the ledger posts on the bank's confirmation.
- **Settlement** (step 7) is the banks' risk, not the society's: once NIP returns 00, the beneficiary is credited even though inter-bank settlement follows later. The platform never sits in the flow of funds, so a platform outage delays records, not money.

## Account structures

The account structure decides who owns the money if something breaks. A dedicated account in the society's own name makes the bank's core system the record of truth and gives the society its own deposit insurance; a pooled or virtual-account structure makes the fintech the account holder and its sub-ledger the only record of who owns what. Synapse failed in exactly that second structure.

| Structure | Account holder | Ledger of record | NDIC cover | Mandate (two of three) enforceable at bank | Fit |
| --- | --- | --- | --- | --- | --- |
| **Dedicated corporate account per society** | The society | The bank's core | Society as depositor, ₦5m at a commercial bank (from 2 May 2024) | Yes, if the bank agrees | **Custody: recommended** |
| Virtual accounts mapped to a society's parent account | The society (parent) | The bank's core plus the virtual-account mapping | As the parent account | Yes, on the parent | **Employer collections: recommended** |
| Pooled "for benefit of" account with platform sub-ledger | The platform or middleware | The platform's or middleware's sub-ledger | Per beneficiary only under NDIC pass-through conditions (bare trust, per-subscriber records, KYC); no NDIC fintech pass-through guideline was found | No: one mandate for all societies | **Avoid for custody** |
| Wallets at an MFB | The member or society at the MFB | MFB core | ₦2m at an MFB | Limited | Avoid for society funds |

### How Nigerian banks implement it

- **VFD (VBaaS)** has a corporate-account endpoint that opens a real account: it requires an RC number and a director's BVN, and the account stays on post-no-debit until BVN consent is given. Whether a cooperative registration number is accepted instead of an RC number is **unverified** and must be asked.
- **Providus, Wema and VFD** offer virtual accounts that settle into a parent account; these suit the employer-per-virtual-account design.
- **Sterling** publishes no BaaS documentation; terms only through a sales conversation.
- Every society signatory will need BVN and NIN under the 1 December 2023 KYC circular; funded accounts without them have been frozen since 1 March 2024.

### The records the platform must keep anyway

Even though the bank is the ledger of record, the platform keeps a complete but non-authoritative copy per society: the mandate, every transaction with bank and NIP references, daily balances, and member-level allocations. If the bank, a middleware or the platform fails, the society can prove its position from its own bank statements and the platform's export, and members can prove theirs from the platform's ledger.

### Resilience

- Reconcile three ways every day: bank statement, platform event log, society ledger.
- Contract and test a second bank; prefer well-capitalised commercial banks over MFBs (46 MFB licences were revoked on 1 July 2026, with NDIC cover capped at ₦2m).
- Contract for data escrow, exit assistance, step-in rights and incident notice.
- Keep societies' direct access to their bank channel for emergencies, under the documented emergency procedure.

## Reconciliation, failure handling and operations

The rails are fast but not certain. A transfer can be pending, unknown, successful-then-reversed, or duplicated by a webhook retry. The platform's job is to make every one of those states explicit and to post to the ledger only on a confirmed, deduplicated bank event.

### Outbound transfer state machine

| State | Entered when | Next |
| --- | --- | --- |
| Approved | Two signatories approve | Sent |
| Sent | Instruction accepted by the bank API with an idempotency key | Succeeded, Failed or Unknown |
| Unknown | Response 09, 96 or timeout | Status query on the same session ID at 1, 5, 15 and 60 minutes, then hourly; never re-send |
| Succeeded | Response or status query 00 | Journal posted; may become Reversed |
| Failed | 51, 91, or status query 25 (no record) | Payment reopened; nothing posted; a retry is a new instruction |
| Reversed | Credit back to the society account matching the original session ID | Reversal journal referencing the original; payment reopened; signatories notified |

### Inbound credit handling

1. The BaaS bank (not NIBSS) sends the webhook. Providus, for example, sends sessionId, settlementId, source account, name and bank, amount, fee, settled amount and remarks, and moves funds to the settlement account only after the platform acknowledges.
2. Deduplicate on the settlement or session ID; a repeated ID is acknowledged and ignored.
3. Attribute the credit to the employer by the virtual account it hit; post to unapplied remittances.
4. Confirm against the daily statement; a statement line with no webhook is recorded and flagged "webhook missing".

### Daily reconciliation

- **Three-way**: bank statement, platform event log (webhooks and instruction responses), society ledger. Every difference becomes an item with an owner the same day.
- **Unknown transfers older than 24 hours**: escalated to the bank's operations contact with the session ID.
- **Reversals**: matched to the original by session ID; a reversal without a match is a suspense item.
- **Fees and duty**: posted from the bank's charge lines, not estimated.

### Operational design rules

- Key every ledger entry on the bank's session or settlement ID.
- Run name enquiry before every payout and show the verified name, fee and ₦50 duty to approvers before they approve.
- Use idempotency keys on every API call; query before any retry.
- Allow successful-then-reversed and unknown-for-hours as normal states, not errors.
- Hold webhooks to a 60-second acknowledgement; if the platform is down, rely on the statement to catch up.
- Keep a bank operations contact and the NIBSS dispute route for each partner bank in the runbook.

### Unverified and to confirm with the bank or NIBSS

The NIP interface specification is not public (only one provider's 100-character narration limit was found); the number of daily inter-bank settlement windows (sources say 2, 4 or 12); the final status of the 2026 fee guide; the Tier 2 daily limit for bank accounts; whether the ₦10k-per-failed-transfer fine is still enforced; NEFT, RTGS and direct-debit settlement timings.

## Regulation and risk

The Nigerian rules say one thing clearly: a platform that is not a bank or mobile money operator may not hold customer funds, and any product built with a bank needs the CBN's prior approval. Everything else is about KYC, data and the bank's own exposure, which the bank will push onto the platform through its due diligence.

### Nigerian rules that bind a BaaS integration

| Rule | What it requires | Date |
| --- | --- | --- |
| CBN payments licensing framework | Only mobile money operators (and banks) may hold customer funds; every bank–fintech product collaboration needs CBN prior approval | 9 Dec 2020 |
| KYC circular | BVN and/or NIN by account tier; non-compliant funded accounts frozen | 1 Dec 2023; enforced from 1 Mar 2024 |
| NDIC limits | ₦5m per depositor at commercial banks; ₦5m per subscriber via mobile money pass-through; ₦2m at MFBs and payment service banks | 2 May 2024 |
| Circular PSS/DIR/PUB/CIR/001/004 | Payments data generated in Nigeria stored in Nigeria from 1 Jan 2027; beneficial-ownership disclosure; market-share caps (wording from law-firm summaries) | 15 Jun 2026 |
| Open banking guidelines | Issued; the 1 Aug 2025 go-live slipped; current status unconfirmed | Mar 2023 |
| Automated AML standards | Banks by Sep 2027, other institutions by Mar 2028 (sources disagree on dates) | 10 Mar 2026 |
| Consumer protection regulations | Still in force; a revision draft exists | 2019 |
| Cybersecurity framework | Basis on which the bank demands SOC 2 or ISO 27001 evidence, penetration tests and audit rights from the platform | 31 May 2024 |

### What happened in Nigeria

- **8 February 2021**: Providus cut off the virtual accounts used by PiggyVest, Cowrywise and Monnify. Users could not fund accounts until alternatives were found. Single-bank dependency is a real outage risk.
- **Late April to early June 2024**: the CBN barred OPay, Moniepoint, Kuda, PalmPay and Paga from onboarding new customers; on 24 April 2024 a court froze 105 fintech-held accounts at the EFCC's request out of 1,146 under investigation; OPay and Moniepoint were each fined ₦1bn in December 2024.
- **1 July 2026**: the CBN revoked 46 microfinance bank licences; NDIC became liquidator with cover capped at ₦2m.

### What happened elsewhere

- **Synapse (US)**: its records drifted from its partner banks' from September 2023; Evolve froze all accounts on 11 May 2024; the shortfall was $65–96m (commonly $85m) between about $180–219m held and about $265m owed. The CFPB sued (21 Aug 2025) and a judgment (12 Sep 2025) lets its penalty fund compensate users.
- **FDIC response**: a proposed rule (17 Sep 2024) would require banks to keep beneficial-owner records for custodial accounts and reconcile daily; unfinished as of 2025.
- **US consent orders** against Blue Ridge, Cross River, Lineage, Sutton, Piermont and Choice (2022–2024) typically required regulator approval before any new fintech partner; Evolve also suffered a ransomware breach affecting 7.6m people.
- **UK**: Railsr entered administration in March 2023; its Payrnet unit later showed safeguarding shortfalls.

### Lessons for this design

1. Keep each society's money in the society's own bank account; never let a pooled sub-ledger be the only record.
2. Reconcile daily, three ways, and investigate any difference the same day.
3. Run two banks; expect one to cut off or freeze at some point.
4. Expect the bank to require CBN approval, SOC 2 or ISO 27001 evidence, penetration tests, audit rights and data residency in Nigeria from 2027.
5. Get a Nigerian law-firm opinion on the platform's licence position before launch.

## Fit for this platform

Use two layers: a commercial bank for custody (each society's own accounts, mandate, statements) and a well-signed collections and payouts provider for virtual accounts and direct debit, with a second of each for failover. Avoid MFB-held and pooled custody for society money.

### Shortlist

| Role | First choice | Second choice | Why |
| --- | --- | --- | --- |
| **Custody** (society operating and project accounts) | **Sterling via Embedly**: commercial bank, organisation and business wallets, bulk payouts and bulk name enquiry, Svix-signed webhooks, published prices | **ProvidusUnity**: commercial bank, mature VA and statement APIs, used by most Nigerian fintechs; webhook signing weak | Commercial-bank deposit cover (₦5m); real accounts; statements; confirm accounts can be titled in a cooperative's name |
| Custody alternative to test | **Wema** (ALAT API + CoopHub relationship) | VFD (has a documented corporate-account endpoint, but MFB cover ₦2m and no webhook signing) | Wema is the incumbent cooperative bank; a partnership is option B in the CoopHub review |
| **Employer virtual accounts and collections** | Same bank as custody, so credits land directly in the society's account | Squad (HMAC-SHA512, 0.25% capped ₦1,000) or Kuda (0.3% capped ₦80) | Keep collections at the custody bank to avoid moving money between providers |
| **Payouts** | Custody bank's transfer API | Paystack (HMAC-SHA512, approval callback, bulk) | Book transfers within the custody bank avoid NIP fees and duty |
| **Direct debit** (employer-default fallback) | Paystack or Monnify | Mono | NIBSS mandates; ₦50 activation, ₦30 per debit plus mark-up |
| **Statements for third-party accounts** | Mono (₦50k/month) | Bank MT940 | Only for societies that keep an account outside the custody bank |

### Avoid for society funds

Pooled middleware custody (Anchor, Maplerad), MFB-held balances above ₦2m, providers with no documented webhook signing for credit events (unless every credit is re-queried), and dormant or closed providers (Bloc, Okra, Brass).

### Questions for every shortlisted provider

1. Which legal entity holds the funds, and what is the deposit cover?
2. Can accounts and virtual accounts be titled in a cooperative society's name with its registration number (not an RC number), and at what KYC tier and balance cap?
3. Will the bank enforce a platform-only, two-of-three mandate, or route API-initiated payments to bank-side approvers?
4. Does the client need a PSSP or other licence, PCI DSS or ISO 27001, minimum volumes or collateral?
5. Webhook integrity: HMAC over the body, timestamp, source IPs, mTLS?
6. Statement API or MT940 for corporate accounts; debit-side webhooks?
7. Bulk payouts, batch size, name-enquiry enforcement?
8. Negotiated pricing for virtual-account inflows, payouts and KYC calls at expected volumes; monthly minimums?
9. Uptime targets, incident notification and change-notice periods?
10. Exit: account portability, data export, notice period?
11. Counterparty soundness (recapitalisation status, regulatory actions)?

### Integration design, in one view

- Custody adapter (bank A) and failover adapter (bank B), each implementing accounts, virtual accounts, transfers, status, statements and webhooks.
- Collections and direct-debit adapter (gateway) used only for direct debit and for societies whose employer cannot pay into the custody bank.
- Every adapter behind the same interface; the ledger, matching engine and approval workflow never call a provider directly.
- Daily three-way reconciliation per society; unknown transfers queried by session ID; reversals matched to originals.
