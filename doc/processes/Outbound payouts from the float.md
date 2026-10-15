# Outbound payouts from the operational float

Oct 10, 2026 · @Abubakar

## Summary and verdict

**Verdict.** Paying out of the operational float is safe for the platform's own obligations, and only for those. Vendors, platform-funded rewards and refunds of platform fees can go straight from the float. Member withdrawals, loans, dividends and a society's vendors cannot.

- **Why the line sits there.**
  - When a society pre-funds the platform's balance so the platform can pay on its behalf, that looks like holding customer funds and like money transmission.
  - Running an unlicensed payment business is an offence under section 57 of the Banks and Other Financial Institutions Act. The fine is at least ₦10m, plus ₦200,000 a day.
- **Fronting and recovering is not the answer either.** If it is interest-free and incidental to the software, it is probably not moneylending. But the platform carries the society's credit risk, and the money still moves outside the society's mandate.
- **The real risk is fraud, not regulation.** Nigerian payment companies have lost tens of billions of naira since 2023, through insider access, abused credentials and small transfers that slip under thresholds. A float that can pay anyone is the asset attackers want.
- **The operating model.**
  - Two people approve every payout, and nobody approves their own.
  - New bank details wait through a cooling-off period.
  - The platform's own payout identifier is reused on every retry, and a timeout is requeried rather than resent.
  - Withholding tax is deducted from vendor payments.
  - One switch stops every outbound transfer.
- **Before go-live.** Get a Nigerian counsel opinion on the pay-on-behalf flows and on the gateway's merchant terms.

## Which payouts may leave the float

The test is whose money is being paid. The float holds platform money, so it may pay out platform obligations. Paying society obligations from it either means holding society money or lending to the society.

| Payout type | Example | Whose money | From the float? | Conditions |
| --- | --- | --- | --- | --- |
| **Platform vendor** | SMS aggregator, KYC provider, contractor, hosting | Platform | **Yes** | Invoice on file, withholding tax deducted, two approvals, vendor bank details verified |
| **Platform-funded member payment** | Referral reward, refund of a platform fee, service credit | Platform | **Yes** | A budget cap and a per-member cap, the member's KYC done, a published rule that triggers it |
| **Refund of a misdirected payment** | A member paid the platform's account by mistake | The member | **Yes, back to source only** | Same amount, to the originating account only, after the credit is confirmed |
| **Pay-on-behalf, society pre-funds the float** | A society sends money to the float so the platform pays its members or vendors | Society | **No** | This is holding customer funds in a platform account, which only banks, payment service banks and mobile money operators may do |
| **Pay-on-behalf, platform fronts and recovers** | The platform pays a loan or dividend now and invoices the society later | Platform, then a receivable | **No** | This is credit to the society. If it is interest-free and incidental to the software, it is probably not moneylending, but the platform carries the society's credit risk. Society payouts leave from the society's own account |
| **Member withdrawal, loan or dividend** | Savings withdrawal, loan disbursement, year-end dividend | Society | **No** | Paid from the society's own account under its signatories' approval, as the earlier tabs set out |
| **Society's own vendor** | Auditor, estate contractor, legal fees | Society | **No** | Paid from the society's account, with the platform preparing the payment and the society approving it |

**The grey zone to avoid.** A society might ask the platform to "just send it from your side" when its own bank channel is slow. The answer stays no. One exception makes the next one easier to argue, and the platform's float soon becomes the pooled payout wallet the design forbids.

## The end-to-end payout process

Every payout from the float follows the same ten steps, whatever its type.

1. **Request.** A payout is created from a vendor invoice, a reward rule or a refund case. It carries its type, amount, beneficiary, reason and supporting document. The system refuses any type the matrix above forbids.
2. **Duplicate check.** The system blocks a request that matches an earlier one on beneficiary, amount and invoice or case reference within 30 days, unless a named person overrides it with a reason.
3. **Beneficiary validation.**
   - A name enquiry runs against the destination bank.
   - The returned name must match the vendor master or the member's KYC name.
   - A new account, or a changed one, puts the payout on hold for a 24-hour cooling-off period. During that hold, someone confirms the change with the beneficiary on a phone number already on file.
4. **Screening.** Vendors are checked against sanctions and internal blocklists. Members are checked against the internal blocklist.
5. **Tax.** For a vendor, the system computes withholding tax and pays the net amount. The gross amount, the tax withheld and the VAT are recorded.
6. **Approval.**
   - The person who prepared the payout cannot approve it.
   - Larger amounts need a second approver, and the largest need a director.
   - Approvers confirm with a second factor at the moment of approval.
7. **Funds check and reservation.** The amount plus the provider fee is reserved against the float. A payout is never sent against an unreserved balance.
8. **Dispatch.** The transfer is sent with the payout's own identifier as the provider reference. Every retry reuses that reference, so the provider can reject a duplicate.
9. **Confirmation.**
   - A payout becomes successful only after a signed webhook and a status query agree.
   - A timeout or an unclear answer moves it to pending.
   - A pending payout is requeried and never re-sent as a new transfer.
10. **Posting and reconciliation.**
    - The ledger is posted.
    - The beneficiary receives a remittance advice.
    - The daily proof matches the ledger, the provider's transfer list and the float balance.

## The payout lifecycle

&#91;embedded content: payout lifecycle · ten states\]

**How to read it.**

- **Pending.** Pending is the state that causes double payments. A pending payout leaves only through a status query, never through a second send.
- **Held.** New or changed bank details send a payout to Held until the change is confirmed outside the system.
- **Reversed.** A reversal after success reopens the case. The funds return to the float, and someone checks why before anything is re-sent.

## Failure handling

Several provider and NIBSS sites blocked automated reading on 10 October 2026, so some points below rest on mirrors and secondary sources. The official NIP response-code table is not public.

**What each provider does with an outbound transfer.**

| Provider | Idempotency | Approval gate | Bulk | Limits and notes |
| --- | --- | --- | --- | --- |
| Paystack | Your own reference of 16 to 100 characters, reused on retry | OTP, abandoned after 30 minutes, or an approval URL on your server. If the URL does not answer in time, the transfer is blocked | 100 items every 5 seconds, OTP must be off | ₦50 to ₦10m per transfer. Higher limits need app-based 2FA for all users, approvals and IP whitelisting of up to 10 addresses |
| Flutterwave | Idempotency header in version 4 | Dashboard payout rules, which do not apply to API transfers | Supported | One completion webhook. A secondary source says the retry endpoint can duplicate a successful transfer, so never automate it |
| Monnify | A duplicate reference is rejected | Email OTP on by default | Up to 5,000 items, with continue or stop on validation failures | Live mode accepts only whitelisted static IPs (secondary source). Separate success, failed and reversed webhooks |
| Embedly | None documented | None documented | No bulk endpoint | Success on initiation only means "being processed". Webhooks stop after the fifth retry, so a scheduled status sweep is needed |

**Which answers are final.**

- Only a confirmed `00` is final success.
- Codes such as 09, 96, 97, 01 and 94, and any timeout, are unknown. Query the status with the same reference, and never send again.
- A "no record" answer (code 25) counts as failed only after a grace window.
- Insufficient funds (51), and invalid-account, limit and fraud codes, are final failures.
- Banks typically wait about 40 seconds and then query up to three times. The query stays valid for about 24 hours (secondary source).

**The rules the system follows.**

1. A timeout is never a failure. The payout moves to pending.
2. A pending payout is requeried on a schedule. After 24 hours it goes to a person, not to an automatic retry.
3. A retry always reuses the original reference. A new reference is created only after a confirmed failure.
4. Webhooks are verified by signature and then confirmed by a status query before the ledger is posted.
5. A reversal can arrive after success. It reopens the payout and returns the funds to the float.

**Money sent to the wrong account.**

- Under the CBN's 2018 instant-transfer rules, a receiving bank must reverse its own error within one business day.
- When the sender gave the wrong account, the receiving bank can hold the amount for up to two weeks, but it needs the beneficiary's consent to return it.
- Bank-to-bank disputes have three working days, through the NIBSS dispute portal.
- No recall success rate is published. Recovery in reported cases ranged from about 13% to 73%.
- Double credits happen at scale. In September 2024 a NIBSS glitch credited ₦13.66bn to 176 accounts, and ₦4.19bn is still unrecovered.

**What this means.** Prevention beats recall. Validating the name before sending and the cooling-off on new accounts are worth more than any recovery process.

## Regulation

Markers follow the research notes. Some findings come from the primary text of the law. Others come from secondary sources or are inferences, and they say so. Treat this section as a brief for counsel.

**Licensing.**

- **No licence for the platform's own payments.** Paying the platform's own vendors, rewards and refunds is a business spending its own money. That needs no CBN licence. This is an inference, since no CBN text says it in terms.
- **Pre-funded pay-on-behalf is the danger.** A society pre-funding the platform's balance looks like holding customer funds, which only banks, payment service banks and mobile money operators may do. Running an unlicensed payment business is an offence under section 57 of the Banks and Other Financial Institutions Act. The fine is at least ₦10m, plus ₦200,000 a day.
- **The nearest licence still would not allow it.** That is the payment solution service provider licence, with ₦100m capital, and it cannot hold funds either. No CBN rule or law-firm opinion was found on disbursement or payroll providers specifically.
- **Fronting is probably not moneylending.** The Lagos Moneylenders Law excludes lending incidental to another business, and the Supreme Court in *Uzoukwu v Idika* (2022) supports this, provided the advance is interest-free. The FCCPC digital-lending rules cover consumer loans only. Recover from the society, never from members.

**Anti-money laundering.**

- **Reporting and records under the 2022 Act.** Suspicious transactions are reported within 24 hours. A single transfer above ₦5m for an individual or ₦10m for a company is reported within 7 days. Records are kept for 5 years. Splitting a transfer to avoid a report is an offence.
- **How the duties reach the platform.** Software platforms are not on the Act's lists of financial institutions or designated businesses. The duties arrive through the gateway's merchant contract instead. Build to them anyway.
- **Information that travels with a transfer.** The CBN's 2022 AML regulations require originator and beneficiary details to travel with a domestic transfer, or be producible within 3 business days. A sanctions hit must be frozen without delay.

**Limits and recalls.**

- **CBN ceilings from 1 July 2026.** ₦25m per transaction for individuals and ₦250m for companies (secondary source). New accounts are capped at ₦20,000 for their first 24 hours.
- **Paystack's limits.** Paystack allows ₦50 to ₦10m per transfer. Its name check can be overridden by hand, so the platform must block that override.
- **Sending-bank errors.** A transfer error caused by the sending bank, reported within 14 working days, is reversed within one business day.
- **Fraud proceeds.** NIBSS has debited receiving banks for fraud proceeds since January 2025.
- **Draft refund rules.** A draft fraud-refund framework from November 2025 proposes about 16 working days. It is not confirmed as final.

**Tax on vendor payments.**

- **Withholding tax rates.** 2% for general services and goods, 5% for professional fees and commission, and 10% for rent and interest. The rate doubles if the vendor has no tax ID.
- **Remittance and filing.** Remit by the 21st of the following month. Filing moved to the NRS Rev360 portal on 30 April 2026.
- **Small-payer exemption.** A small company need not deduct for vendors with a tax ID up to ₦2m a month. The Nigeria Tax Act now puts the small-company turnover limit at ₦50m. Whether the regulations were updated to match is unconfirmed.
- **VAT.** VAT is 7.5% on vendor invoices. The platform does not withhold it unless the tax authority appoints it to.
- **Stamp duty.** The ₦50 stamp duty on transfers of ₦10,000 or more is paid by the sender. The Act exempts instruments "executed by or on behalf of a co-operative society". Whether that covers electronic transfers is untested, so it is a question for counsel.

**Member rewards.**

- Discounts and rebates count as income under the Nigeria Tax Act. The first ₦800,000 of an individual's income is taxed at 0%, so small rewards are unlikely to bite.
- Prize draws decided by chance count as a lottery and carry 5% withholding tax, so avoid them.
- A fee refund returns the member's own money and is not income. That view is an inference.

**Data protection.**

- Under the 2025 General Application and Implementation Directive, fintechs and payment gateways are in the top tier of data controllers.
- The annual compliance audit return is due by 31 March, through a licensed audit firm.
- Financial services require a data protection impact assessment. Breaches are reported within 72 hours.
- BVN data must stay in Nigeria (secondary source).

**Enforcement seen so far.**

- The CBN fined Paystack ₦250m in April 2025, and Moniepoint and OPay ₦1bn each in December 2024, though OPay denied it.
- The data protection regulator fined Fidelity Bank ₦555.8m in August 2024.
- No sanction against an unlicensed software platform was found. The practical risk is the gateway closing the account.

## Risk register

**What Nigerian incidents show.**

- **Flutterwave.** It lost ₦2.95bn in February 2023, sent first to 28 accounts and then 107. It lost ₦19bn in October 2023 through abused merchant access. It lost ₦11bn in April 2024 in small transfers that stayed under fraud checks, spread across five institutions in four days.
- **Interswitch.** It lost about ₦30bn to insider-driven chargebacks in 2023 and recovered about ₦10bn.
- **eTranzact.** It lost ₦1.39bn in July and August 2026 through database manipulation. A court froze 69 accounts, and ₦181m was recovered.
- **First Bank.** A manager who was the sole approver diverted about ₦40bn.
- **Cooperatives and payroll.** The EFCC is prosecuting cooperative executives in Jos over ₦178.9m. An IPPIS audit found 587 suspected ghost workers.
- **The common thread.** Money is spread across dozens of mule accounts within hours. The window to freeze it is minutes to hours, not days.

| Risk | How it happens | Likelihood | Impact | Main controls |
| --- | --- | --- | --- | --- |
| **Insider payout fraud** | A staff member creates and approves a payout to an account they control | Medium | High | Maker-checker that the system enforces, no single approver at any amount, approver alerts, monthly access review |
| **API key compromise** | A leaked secret key sends transfers straight from the float | Medium | High | Keys in a vault, IP whitelisting, Paystack's approval URL as a second gate, velocity limits, a kill switch |
| **Bank-detail change fraud** | A fake email or call changes a vendor's or member's account | Medium | High | Name check, 24 to 72 hour cooling-off, confirmation on a phone number already on file, two people to approve the change |
| **Approver account takeover** | An approver's login and second factor are stolen | Low | High | App-based second factor, device binding, step-up on approval, alerts to the approver on every approval |
| **Double payment** | A timeout is retried as a new transfer | Medium | Medium | Same reference on every retry, the pending state, requery before any resend |
| **Small-amount drain** | Many transfers just under the review thresholds | Medium | High | Daily and hourly totals per beneficiary and overall, alerts on many new beneficiaries |
| **Ghost or duplicate beneficiaries** | Fake members or vendors added to receive rewards or refunds | Medium | Medium | Rewards only to KYC-verified members, BVN uniqueness, one bank account per member, duplicate detection |
| **Misdirected payout** | A wrong account number that passes name enquiry, or a near-match name | Low | Medium | Strict name match, a hold on near-matches, the recall playbook ready |
| **Provider outage or freeze** | The gateway is down or the float is frozen | Low | Medium | A second provider, a minimum float, nothing urgent depending on one rail |
| **Scope creep into society money** | Pressure to pay a society's members "just this once" | Medium | High | The type matrix enforced in code, refusals logged, board-approved policy |

**Incident playbook.**

1. Hit the kill switch. It suspends all outbound payouts across providers within one minute.
2. Call the provider's fraud desk, and the fraud desks of the receiving banks, with transfer references and session IDs.
3. File a police report and apply for a court order to place post-no-debit on the receiving accounts. Keep the templates ready in advance.
4. Rotate every key and session, and review approvals for the past 30 days.
5. Reconcile the float and report to the board within 24 hours.

## Accounting entries

| Event | Debit | Credit |
| --- | --- | --- |
| Vendor invoice approved | Expense, plus input VAT | Vendor payable |
| Payout reserved | Float reserved for payouts | Float available |
| Vendor paid, net of withholding tax | Vendor payable, gross amount | Float reserved for payouts, net amount, and withholding tax payable |
| Transfer fee charged by the provider | Bank charges expense | Float available |
| Withholding tax remitted to the tax authority | Withholding tax payable | Operating bank |
| Member reward or fee refund paid | Marketing expense or fee revenue reversal | Float reserved for payouts |
| Misdirected payment returned | Misdirected receipts suspense | Float reserved for payouts |
| Payout failed | Float available | Float reserved for payouts |
| Payout reversed after success | Float available | Vendor payable or the original expense, reopening the item |

**Rules.**

- The two float accounts, available and reserved, always sum to the provider's reported balance.
- A reversal never deletes the original journal. A new journal reverses it and names the reversal reference.
- Withholding tax is remitted on the statutory date, and the vendor gets a credit note for it.

## Controls and requirements for the FRD

These requirements extend FR-FLT-01 to FR-FLT-10 in the float tab.

**FR-PAY-01 Allowed payout types.** The system allows only platform vendor payments, platform-funded member payments and returns of misdirected receipts from the float.

- A payout whose type or source of funds is a society is refused, and the refusal is logged.
- The allowed list can only be changed through a release, not through a setting.

**FR-PAY-02 Duplicate prevention.** The system blocks a payout that matches an earlier one on beneficiary, amount and reference within 30 days.

- An override needs a named person and a written reason.

**FR-PAY-03 Beneficiary validation.** Every payout runs a name enquiry, and the name must match the vendor master or the member's KYC name.

- A near-match goes to a person. A mismatch is refused.
- The provider's manual name-check override is never used.

**FR-PAY-04 Bank-detail changes.** A new or changed beneficiary account takes two approvals and a cooling-off of at least 24 hours.

- The change is confirmed on a phone number or email already on file, never one supplied with the change.
- The beneficiary is told of the change on the old contact details.

**FR-PAY-05 Approval matrix.** Every payout needs a maker and a different checker.

- Above a set amount, a second checker is needed. Above a higher amount, a director is needed.
- Every approval needs an app-based second factor at the moment of approval.

**FR-PAY-06 Funds reservation.** The amount plus the fee is reserved before dispatch.

- A payout is never sent against an unreserved balance.
- Available and reserved balances always sum to the provider's balance.

**FR-PAY-07 Safe dispatch.** Each payout's own identifier is the provider reference on every attempt.

- A timeout or an unclear code moves the payout to pending, never to failed.
- A new reference is only created after a confirmed failure.
- The provider's own retry endpoint is never automated.

**FR-PAY-08 Confirmation.** A payout is successful only when a verified webhook and a status query agree.

- Pending payouts are requeried on a schedule, and go to a person after 24 hours.
- A reversal after success reopens the payout and posts a reversing journal.

**FR-PAY-09 Velocity limits.** The system enforces limits per transfer, per beneficiary per day, per hour overall and on new beneficiaries per day.

- Breaching a limit holds the payout and alerts the finance lead and the CTO.

**FR-PAY-10 Kill switch.** One action suspends all outbound payouts across providers within one minute.

- Only named people can use it, and its use alerts the directors.

**FR-PAY-11 Withholding tax.** Vendor payouts deduct withholding tax at the rate for the payment type, doubled where the vendor has no tax ID.

- The system produces the monthly schedule for remittance by the 21st, and a credit note for each vendor.

**FR-PAY-12 AML records.** Every payout stores the originator and beneficiary details for five years.

- Transfers above ₦5m to an individual or ₦10m to a company are flagged for reporting.
- Patterns that split one payment into several are flagged as suspicious.
- Beneficiaries are screened against sanctions lists before their first payout.

**FR-PAY-13 Key and access security.** Payout keys are stored in a vault and limited to the platform's IP addresses.

- Keys are rotated every quarter and at once after any incident.
- Paystack's approval URL is used as a second, server-side gate.
- Dashboard users with payout rights are reviewed monthly.

**FR-PAY-14 Remittance advice.** Every successful payout sends the beneficiary a remittance advice with the amount, reference and session ID.

- For vendors, the advice also shows the gross amount, the tax withheld and the net amount.

## Open questions and next steps

**For counsel.**

1. Confirm that the platform's own vendor, reward and refund payouts need no licence.
2. Confirm that pay-on-behalf in either form stays out of scope, and give the exact risk if a society insists.
3. Does the cooperative stamp-duty exemption cover electronic transfers from a society's own account?
4. Does the small-company withholding-tax exemption now follow the ₦50m turnover limit?

**For providers.**

1. **Paystack.** Confirm transfer limits, and whether the approval URL can enforce the platform's own approval matrix. Confirm the merchant terms on balances and fraud losses.
2. **Flutterwave and Monnify.** Confirm payout limits, bulk sizes and reversal timings.
3. **Embedly.** Will it add idempotent references for payouts?

**Next steps.**

1. Add FR-PAY-01 to FR-PAY-14 to the FRD tab.
2. Set the approval thresholds and velocity limits with the finance lead before the pilot.
3. Run a Paystack sandbox test of the flow:
   - approval URL;
   - duplicate reference;
   - a forced timeout;
   - a reversal after success.
4. Prepare the incident pack:
   - fraud desk contacts;
   - police and court templates;
   - the kill-switch runbook.
5. Register for the data protection audit return, and run the impact assessment before go-live.

Full notes are in the research folder, in payout\_regulation\_aml\_tax.md and payout\_mechanics\_failure\_fraud.md.
