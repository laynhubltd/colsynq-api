# Bank role and system automation for check-off and custody

Oct 9, 2026 · @Abubakar

## The problem this tab solves

A cooperative's money and its records live in different places, controlled by different people, with no system binding them. The money sits in a bank account that one or two officers can operate; the records sit in a spreadsheet the treasurer keeps; the deductions that feed both are decided by an employer's payroll that reports nothing back. The result is the failure list in the problem statement: lump sums with no schedule, withdrawals frozen for years, duplicate deductions, land paid for on one signature, and arrears nobody can quantify.

The design in this tab puts the bank in the loop as the enforcer and the platform as the system of record, so that:

- **Money cannot move without the record.** Every outbound payment from the society's account is initiated in the platform, approved by the people the bye-laws name, and executed by the bank only on that approved instruction. The bank's mandate, not the committee's goodwill, is what stops a single-signatory withdrawal.
- **Every credit arrives with its identity.** Each employer remits into a dedicated virtual account belonging to the society, so the platform knows which employer paid, when, and how much, the moment the credit lands, and can hold it unapplied until the schedule arrives.
- **The ledger and the bank agree every day.** Statement feeds reconcile automatically; the society's balance in the platform is the bank's balance, proven, not asserted.
- **Members can see and act without the treasurer.** Balances, expected deductions, approvals and consents work over USSD and app, against the same ledger the bank settles.
- **The employer is measured.** Because remittances are identified at the bank, arrears age from the day the payroll ran, and the employer's statement is produced by the system, not typed by an aggrieved secretary.

The platform never holds money. The bank holds it, in the society's name, with NDIC cover where the bank is an insured institution; the platform holds the truth about whose money it is and what it may be used for.

## Actors

| Actor | Belongs to | Role in the process | Rights | Obligations |
| --- | --- | --- | --- | --- |
| Member | Society | Saves and repays by payroll deduction; borrows; approves where bye-laws require | See own ledger and expected deductions on USSD or app; receive statements; consent to or revoke deductions; nominate | Sign a deduction authority; declare outside debts; repay |
| Society general meeting | Society | Highest authority | Approve bye-law parameters, signatories, borrowing, investments, budget | Quorum and notice rules; minute decisions |
| Management committee | Society | Approves loans, schedules and payments within limits | Initiate and approve within bye-law thresholds | Fiduciary duty; related-party disclosure; term limits |
| Signatories / trustees (three) | Society | Approve outbound payments in the platform | Any two approve; none can approve alone | Cannot be changed without general-meeting resolution and bank re-mandate |
| Treasurer / secretary | Society | Operates the platform day to day | Prepare schedules; initiate payments; resolve variances; cannot approve own initiations | Reconcile monthly; issue statements |
| Supervisory committee | Society | Inspects | Read-only access to everything; inspection reports | Quarterly inspection; report to AGM |
| External auditor | Society | Audits | Reproduce balances from stored artefacts | Annual audit (Act s.36) |
| Employer HR / payroll officer | Employer | Applies the schedule, runs payroll, remits | Employer console: own employees only; upload remittance schedule; see arrears | Deduct only on consent; remit by the agreed date with a schedule |
| Payroll platform (IPPIS, Remita, state HRMIS, SAP, HRIS) | Employer | Computes deductions and pays | — | Accept the schedule in its layout; produce payslips |
| Platform operator | Platform | Runs the system of record | Operate adapters, run reconciliation, support | Never hold money; never approve payments; multi-party controls on its own admins |
| **Bank: relationship manager** | Bank | Onboards the society; owns the partnership | Approve account opening and products | KYC the society and signatories; set up mandates |
| **Bank: operations / settlement** | Bank | Executes approved instructions; handles exceptions | Reject instructions that fail mandate, limits or sanctions checks | Execute within SLA; return failed items with reason codes |
| **Bank: API / digital channels team** | Bank | Exposes accounts, transfers, statements and webhooks to the platform | Issue credentials; set rate limits | Maintain the API; notify changes |
| **Bank: compliance** | Bank | Ensures the collaboration is CBN-approved and AML-compliant | Suspend on suspicion | Obtain CBN approval for the partnership; transaction monitoring |
| **Bank: treasury** | Bank | Prices float on pooled balances | Set interest on society accounts | Pay the agreed rate; report balances |
| NIBSS | Rail | Instant payments, direct-debit mandates, BVN | — | Settle; validate mandates |
| Payment service provider (PSSP) | Rail | Direct-debit aggregation, collections, USSD | Charge per transaction | Hold a CBN licence; no custody |
| State Director of Cooperatives | Regulator | Supervises the society | Inquiry, audit, caretaker, arbitration | Receive returns; publish arrears if agreed |
| CBN | Regulator | Approves the bank–platform collaboration; supervises the bank | Approve or refuse; set limits | — |
| NDIC | Regulator | Insures deposits at the bank | — | Pay insured depositors on bank failure (society accounts are insured as depositors, not members individually) |
| Credit bureau / CoopCHECK | Data | Loan performance sharing | — | Under member consent; when available |

The three roles that must never collapse into one person: the initiator of a payment, its approvers, and the operator of the platform. The bank enforces the first two through the mandate; the platform enforces the third through its own multi-party administration.

## The end-to-end process, explained

Eight stages. The first two happen once per society and per employer; the rest repeat every month. At every stage the rule is the same: the platform decides and records, the bank moves and enforces, and no money moves on anyone's word alone.

### Stage 1. Society onboarding (once)

1. The society registers on the platform with its certificate, bye-laws and officers' list. The platform captures the bye-law parameters that will drive automation: minimum savings, loan multiples and rates, waiting period, signatory rule (two of three), spending limits per tier, appropriation rules, exit charges.
2. The bank opens a **current account in the society's name** (the "operating account") after KYC on the society and its three signatories, and records the **mandate**: payments may be executed only on instructions delivered through the platform carrying two signatory approvals; no cheque book; no branch withdrawals; card and portal access disabled or read-only.
3. The bank opens a second account, the **project or escrow account**, for estate or venture funds if the society has them, under the same mandate plus a release condition.
4. The platform connects: statement feed and webhooks on both accounts, outbound-transfer API credentials scoped to the society, and the bank's reference for each account.
5. Members are loaded: identity verified (BVN or NIN), employer and payroll identifier recorded, nominee captured, each member's **signed deduction authority** stored and referenced.
6. Historical reconciliation: the last three to six months of schedules and bank credits are loaded and matched, producing the society's opening ledger and its real arrears position.

### Stage 2. Employer onboarding (once per employer)

1. The society, employer and platform sign or confirm the **tripartite deduction agreement**: schedule format, remittance deadline, interest on late remittance, reconciliation, the one-third cap, termination, and the fallback to direct debit.
2. The bank issues a **dedicated virtual account** under the society's operating account for this employer (one per employer; optionally one per employer per month). The employer remits only to that account, so every credit identifies its payer.
3. The platform's schedule adapter is configured to the employer's layout; the payroll officer confirms the identifier mapping once; the employer console is provisioned.

### Stage 3. Monthly schedule (around the 10th–15th)

1. The platform computes each member's expected deductions from the ledger: savings line, loan instalments from amortisation schedules, approved changes.
2. Every line is checked: consent on file; headroom within the cap; no loan line for a member with an unresolved variance over 60 days (configurable).
3. The schedule is frozen as a version with a hash, exported in the employer's layout, and sent (portal download, email, or API where one exists). Members receive their expected lines by SMS or USSD.

### Stage 4. Deduction and remittance (employer, around the 20th–30th)

1. Payroll applies the schedule and pays salaries. The member's payslip shows the deduction.
2. The employer transfers the total to the society's **virtual account for that employer** and uploads or sends the itemised remittance schedule.
3. The bank credits the account and fires a **webhook** to the platform with the credit details (amount, time, payer, NIP reference, narration).

### Stage 5. Reconciliation (platform, within one business day of the credit)

1. The credit is posted to **unapplied remittances** for that employer.
2. When the remittance schedule arrives, the matching engine runs: tier 1 exact matches post to member ledgers; partial, missing, unexpected, duplicate and period-shift lines open variance cases.
3. Three-way balance: schedule total, remittance-file total, bank credit. Any file-to-credit shortfall becomes an **employer short-remittance** receivable.
4. Members are notified: "credited" or "deducted, awaiting remittance" with the amount.

### Stage 6. Posting and arrears (platform, automatic)

1. Matched lines are posted; loan schedules advance; interest accrues; guarantor exposure updates.
2. Unmatched expected lines age from the employer's pay date; at 15, 30, 60, 90 days the escalation steps fire (reminder, demand letter, Director escalation and interest, provision and loan override).
3. The employer statement and scorecard are generated for HR; the society's dashboard shows liquidity, arrears and the unapplied balance.

### Stage 7. Outbound payments (platform initiates, bank executes)

1. A treasurer initiates a payment in the platform: loan disbursement, savings withdrawal, refund, supplier, dividend. The platform attaches the purpose, the ledger entry it will create, and the evidence required for that payment type (loan approval id; withdrawal eligibility; invoice; for land, the due-diligence record; for construction, the certified milestone).
2. The platform checks limits and anti-splitting, flags related parties, and routes to the approvers the bye-laws require (two signatories; the general meeting for reserved matters).
3. On approval, the platform sends the signed instruction to the bank's transfer API. The bank verifies the mandate (instruction origin, approver signatures, limits, sanctions) and executes by NIP or internal transfer, returning a reference or a reason code.
4. The ledger entry is posted on the bank's confirmation, never before; the statement feed confirms it the same day.

### Stage 8. Exceptions and member access

- **Failed transfers**: returned with the bank's reason code; the initiation reopens; nothing is posted.
- **Employer default**: members on the missing lines may be switched to NIBSS direct debit under the agreement; loans are not marked delinquent while the employer owes the money.
- **Signatory change**: general-meeting resolution uploaded, 48-hour cooling-off, bank re-mandates, all members notified.
- **Members**: USSD and app give balance, expected deductions, loan status, approval of items that need member consent, and the status of every deduction line.
- **Audit**: any month reproducible from the frozen schedule, the remittance file, the bank credit, the match results and the approved payment instructions.

## The bank's role

The bank does seven things the platform cannot and should not do. Each one converts a trust problem into an enforced rule, and each one gives the bank a reason to partner: deposits, fee income, float, and a regulated route to a customer segment it cannot otherwise reach.

### What the bank does

| Function | What it means in practice | Why the bank wants it |
| --- | --- | --- |
| **Custody in the society's name** | Operating and project accounts opened for the society as a registered body; the platform is a channel, never the account holder | Deposits from cooperatives with ₦0.4–66bn balance sheets; NDIC-insured institution status is a selling point |
| **Mandate enforcement** | The account mandate says: instructions only through the platform, two of three named signatories, limits per instruction and per day, no cheque book, no branch cash withdrawals. The bank's core system rejects anything else | Removes the bank's own fraud and dispute exposure on cooperative accounts |
| **Identified inflows** | A virtual account per employer (and optionally per month) under the operating account; credits carry the payer identity and fire webhooks | Virtual-account fees; data that makes the bank the employer's natural payroll bank too |
| **Executed outflows** | Transfers by API on approved instructions, with reason codes on failure and same-day confirmation | NIP and transfer fee income; volume |
| **Statement and event feeds** | Daily statements and real-time webhooks so the ledger reconciles to the bank every day | Fewer manual statement requests; cleaner audits |
| **Direct-debit mandates** | The society, through the bank or a PSSP, becomes a biller so members outside payroll or during employer default pay by NIBSS direct debit | Mandate fees; retention of member accounts |
| **Escrow and milestone release** | Project accounts whose outflows require the platform's release condition (title record, certified milestone, member vote) in addition to signatories | Project-finance relationships (FMBN schemes, developers); trust-account fees |
| **Float and interest** | Interest on pooled balances at an agreed rate, reported monthly; with the design intent of passing most of it to members | Cheap, sticky deposits; cross-sell (loans to the society, salary accounts for members) |

### What the bank must not do

- Accept instructions on the society's account from any channel other than the platform (including its own portal) once the mandate is live, except a documented emergency procedure requiring a general-meeting resolution and a cooling-off period.
- Net off the society's balances against any other obligation without the mandate's approval route.
- Change signatories or limits on a letter from a committee member.

### Why a bank rather than the platform holding money

- Only banks, MFBs, mobile-money operators and payment service banks may hold customer funds under the CBN payments framework; a software platform holding pooled funds is unlicensed deposit-taking.
- Deposit insurance attaches to the bank, not to the platform.
- Banks already run maker-checker, sanctions screening, limits and settlement; the platform adds the business rules on top rather than rebuilding them.
- A society can switch platforms without moving its money, which is the right dependency: the platform earns its place by being useful, not by holding the funds hostage.

### The commercial shape

A partnership agreement under which the bank (a) opens accounts and issues virtual accounts for societies the platform onboards, (b) enforces the platform-channel mandate, (c) exposes transfer, statement and webhook APIs, (d) pays an agreed float rate, and (e) shares fee income on transfers, mandates and virtual accounts. CBN prior approval of the collaboration is the bank's obligation. The platform's obligation is to deliver KYC-complete societies, enforce the business rules, and never touch the money.

## Connecting the system to the bank: what exists in Nigeria today

Virtual accounts, webhooks, payout APIs, statement APIs and direct-debit mandates are all commodity products from several banks and providers. The one thing the design needs that no Nigerian bank exposes as a public API is bank-enforced dual control (initiator, verifier, approver): it exists in corporate portals and inside Wema's CoopHub, so it must be negotiated, white-labelled, or approximated. Everything below is dated October 2026 and marked where it must be confirmed with the bank.

### Provider map

| Need | Who offers it | Published terms | Status for this design |
| --- | --- | --- | --- |
| Bank-enforced dual control on a corporate account | Corporate portals: GTBank GAPS, Access PrimusPlus, UBA CIB; Wema CoopHub (in-app initiator/verifier/approver roles, white-label offered, 1,000+ cooperatives by Jul 2024) | No public API or pricing anywhere | **Confirm with bank**: bespoke mandate with Wema, Providus or Sterling, or white-label CoopHub |
| Virtual accounts and credit webhooks | Providus, Wema/ALAT, VFD (VBaaS), Sterling/Embedly, UBA (licence required), GTBank Squad, Access Hydrogen, Paystack dedicated accounts (on Wema/Titan), Kuda API, Flutterwave, Anchor, Maplerad | Access 0.2% capped ₦200; Paystack 1% capped ₦300 (1,000 accounts default); Kuda 0.3% capped ₦80; Maplerad free inbound | Ready; choose the bank that will also hold the mandate |
| Payout and transfer APIs | Kuda (₦9–15), Maplerad and VFD (₦10 flat), Paystack, Flutterwave, Providus (₦10/25/50 by band) | Corporate NIP limit ₦10m default, ₦250m with indemnity; only OTP and Paystack's transfer-approval callback exist as programmatic controls | Ready; approvals enforced in-platform, executed by one scoped credential, unless the bank agrees mandate-side checks |
| Statement feeds | REST statement APIs at Providus, Access, UBA (licensed), VFD, Polaris VULTe; Zenith MT940/MT101 for large corporates; Mono statement pulls from ₦15,000 a month; Okra closed May 2025; CBN open banking not live (Oct 2026) | Per bank | Ready with the chosen bank; MT940 availability for mid-size corporates to confirm |
| Direct debit | NIBSS NDD through a sponsoring bank, OFI or PSSP: fixed, variable and GSI mandates; aggregators Mono, Paystack, Monnify, Remita, Zenith Z-Mandate | ₦30 per successful debit; e-mandate by an exact ₦50 transfer, active in 5 minutes to 48 hours; Mono 1% capped ₦1,000 (min ₦55), Paystack 1.5% + ₦100 capped ₦2,000, Remita ₦100 | Ready as the employer-default fallback |
| Escrow and trust accounts | Manual bank or trustee products (Stanbic IBTC, FirstBank Trustees, FCMB Trustees); Vesicash | No self-serve escrow API | Project accounts under the society mandate with the platform's release token; trustee product for large estates |
| Float on balances | Current-account interest "negotiable" (CBN Guide to Charges); savings minimum 30% of MPR = 6.9% after the 22 Sep 2026 cut to 23%; fixed deposits 15–16% (Providus up to 23.66%); 364-day T-bill 15.89% (7 Oct 2026) | Negotiated | Call-account rate per society to negotiate; pass-through to members disclosed |

### Four ways to get dual control, in order of preference

1. **Bespoke mandate with the partner bank.** The bank restricts the society's account to the platform channel and verifies that each API instruction carries the signatures of two registered signatories (the platform signs over the approvals; the bank verifies the platform's key and the approver identities it registered at onboarding). No bank publishes this; Wema, Providus and Sterling are the candidates because they already run bank-as-a-service and cooperative products.
2. **White-label Wema CoopHub.** The platform rides the bank's existing cooperative account product, whose roles are enforced in the bank's own app; the platform supplies the ledger, payroll reconciliation and member channels around it. Fastest regulatory path; least control over the bank-side roadmap; API, pricing and account-title terms unpublished.
3. **Bank-side approval routing.** The platform submits batches as initiator and the bank routes them to approvers registered on its corporate portal. Works with GTBank, Access or UBA portals today if the bank will accept API-initiated batches; to confirm.
4. **In-platform approvals, single execution credential.** The platform enforces two-of-three approvals and signs the instruction; the bank executes on one scoped credential with limits. This is what every fintech payout integration does today. It is acceptable for a pilot only with strict key custody, limits and daily reconciliation, because the control is no longer enforced by the bank.

### What the platform may and may not hold

A platform without a licence cannot hold customer funds (payments framework rule on PSSPs). Money sits in the society's own deposit-money-bank account, or in a documented bare-trust pool where NDIC pass-through cover (₦5m per beneficiary) applies only if KYC and records meet NDIC's conditions; cooperative deposits are otherwise uninsured. The design keeps every society as the named account holder, which is the simplest way to stay on the right side of this.

### Costs the design must carry

- From 1 January 2026 the ₦50 stamp duty on transfers of ₦10,000 and above is charged to the sender (salaries exempt; instruments executed by cooperatives exempt).
- The draft 2026 CBN Guide to Charges (consultation closed 8 May 2026) would make transfers up to ₦5,000 free, ₦10 to ₦50,000 and ₦50 above; 7.5% VAT applies to provider fees.
- Direct debit at ₦30 plus the aggregator mark-up makes member-level fallback viable for monthly amounts above about ₦3,000.

### Regulatory context for the bank conversation

The April 2024 CBN onboarding freeze on OPay, Moniepoint, Kuda, PalmPay and Paga, the revocation of 46 MFB licences on 1 July 2026, circular PSS/DIR/PUB/CIR/001/004 (15 June 2026: data localisation by 1 January 2027, beneficial-ownership disclosure, market-share caps), the AML automation roadmap, and bank liability for fraud proceeds since January 2025 all make banks cautious about new fintech partners. The pitch that works is the one this design makes: the bank keeps custody, enforces the mandate, and gains deposits and fee income; the platform brings KYC-complete societies and takes no money.

### Working examples to point to

Wema CoopHub (cooperative accounts with roles), Providus and VFD virtual-account programmes behind most Nigerian fintechs, Paystack dedicated accounts on Wema and Titan, PenCom's remittance validation platform (schedule validated before payment), and Kenya's Co-operative Bank Sacco Link as the channel model.

## Architecture at a glance

&#91;embedded content: architecture · 3 systems, 7 flows\]

Nothing in the platform holds money and nothing in the bank holds the business rules. Flows 1 and 2 are files until payroll APIs exist; flows 3 to 6 are bank APIs and webhooks from day one; flow 7 runs over USSD as well as the app.

## The automated workflow in detail

The monthly run is a set of scheduled jobs and event handlers. Human approvals are inserted only where a bye-law or the bank mandate requires a person; everything else is automatic and logged. Times are indicative for a society whose employer pays around the 25th.

### Timeline of one cycle

| When | Trigger | What runs | Human step |
| --- | --- | --- | --- |
| Day 1 of month | Scheduler | Interest accrual on loans and savings; dormancy checks; age all open variances and employer receivables; fire 15/30/60/90-day escalation actions due today | None (letters are generated; the secretary may edit before sending if configured) |
| Day 8–10 | Scheduler (employer cut-off minus 5 days) | **Schedule build**: compute expected lines from the ledger; validate consent, headroom, variance holds; freeze version; export in employer layout; publish expected lines to members (SMS/USSD) | Treasurer reviews exceptions (lines blocked by validation); committee approves the schedule if bye-laws require |
| Day 10 | Approval event | Schedule sent to employer (portal, email, or API); delivery logged | Payroll officer acknowledges in the console |
| Day 20–30 | Employer pays | Bank credit lands in the employer's virtual account | None |
| On credit | **Bank webhook** | Credit recorded; posted to unapplied remittances for that employer; expected-credit check (is this the scheduled amount? early, late, short?) | None |
| On schedule file | Upload or email ingestion | Adapter parses; mapping applied; **matching engine** runs; member postings for tier-1 matches; variance cases opened; three-way balance computed; employer statement drafted | Treasurer works the variance queue; disputes assigned |
| Credit + 1 business day | Scheduler | If no schedule yet: reminder to employer; credit stays unapplied; members informed | None |
| Credit + 5 business days | Scheduler | If still no schedule: escalation to HR and the committee; unapplied balance highlighted on dashboards | Committee may authorise provisional posting against the sent schedule only if bye-laws allow (not recommended) |
| Any day | Treasurer action | **Outbound payment**: initiation with evidence; limit and anti-splitting checks; related-party check; routing to approvers | Two signatories approve in the app (push notification, OTP or passkey) |
| On approval | Approval event | Signed instruction to bank transfer API; bank executes; confirmation received; ledger posted on confirmation; member notified | None |
| On failure | Bank reason code | Instruction reopened; initiator and approvers notified; nothing posted | Treasurer corrects and re-initiates |
| Daily, end of day | Statement feed | **Bank reconciliation**: every statement line matched to a ledger entry or an unapplied item; unmatched lines raised | Treasurer resolves; supervisory committee sees aged items |
| Month end | Scheduler | Member statements; employer scorecard; liquidity, NPL, arrears and dividend-on-savings figures; Director return data | Committee reviews the pack |

### Event handlers

- **CreditReceived(account, amount, payer, reference, narration)**: identify employer from the virtual account; post unapplied; emit ExpectedCreditEvaluated.
- **ScheduleReceived(employer, period, file)**: parse; map; emit MatchRun.
- **MatchRun(employer, period)**: classify every line; post matches; open variances; compute balances; emit EmployerStatementReady.
- **PaymentInitiated(id)**: validate evidence, limits, related party; emit ApprovalRequested to the right approvers.
- **PaymentApproved(id, approver)**: when the quorum is met, sign and send to the bank; emit InstructionSent.
- **BankConfirmation(id, status, reference)**: post or reopen; notify.
- **StatementLine(account, line)**: reconcile to ledger; emit UnmatchedStatementLine if none.
- **Clock(day)**: ageing, escalations, accruals, dormancy, schedule build.

### Idempotency and ordering

Every bank webhook and statement line is keyed on the bank's reference and processed once; replays are ignored and logged. Schedule versions are immutable; a corrected schedule is a new version. Postings are append-only; a reversal is a new entry referencing the original. Jobs are safe to re-run.

### What is never automated

Approving a payment, changing a signatory, changing a bye-law parameter, posting a credit without a schedule, and writing off an employer receivable. Each requires the named humans, and the system records who, when and on what evidence.

## The monthly run as a sequence

&#91;embedded content: monthly run · 4 parties, 12 messages\]

The money (step 3) and its description (step 5) arrive separately; step 6 is where the platform binds them, and step 7 is the first time a member learns the truth about their deduction without asking the treasurer.

## The bank-side workflow

From the bank's seat the platform is a corporate channel with a strict mandate. Its operations, systems and compliance teams touch the process at seven points.

### 1. Onboarding a society

1. Relationship manager receives the society's KYC pack from the platform: registration certificate, bye-laws, officers' resolution naming three signatories, BVN and ID for each, proof of address, the platform-channel mandate letter.
2. Account opening in the core banking system: current account (operating), optional project account, interest terms per the partnership.
3. **Mandate configuration**: channel restricted to the platform's API credentials; signing rule two of three; per-instruction and daily limits; cheque book and branch withdrawal disabled; portal access read-only. Captured as a mandate record the core system enforces.
4. Virtual accounts issued under the operating account on the platform's request, one per employer; each mapped to the society in the bank's virtual-account service.
5. API credentials scoped to the society's accounts; webhook endpoints registered; a test credit and a test instruction run end to end.

### 2. Inbound credits

1. An employer's transfer arrives by NIP into the virtual account; the bank's virtual-account service credits the parent operating account and records the virtual-account identifier.
2. The bank fires the credit webhook to the platform within seconds and includes the NIP session reference, payer name and account, amount, narration and timestamp.
3. The line appears on the daily statement feed with the same reference, so the platform can reconcile webhook to statement.

### 3. Outbound instructions

1. The platform submits a signed transfer instruction: society account, beneficiary, amount, narration (with the platform's payment id), and the signatures of the approving signatories (the platform's own signing key plus the approvers' authenticated approvals).
2. The bank's API gateway authenticates the platform, then the **mandate check** runs: channel, signatory quorum, limits, beneficiary status.
3. Sanctions and AML screening; velocity rules.
4. Execution by NIP or book transfer; the bank returns the session reference and status synchronously or by callback; failures carry reason codes (insufficient funds, beneficiary invalid, limit exceeded, mandate failed).
5. The debit appears on the statement feed with the reference.

### 4. Direct debit

1. The society (or the platform's PSSP on its behalf) is enrolled as a biller; members' e-mandates are created with the ₦50 verification transfer and activated within seven days.
2. On the platform's instruction, debits run on the member's bank account for the scheduled amount; successes and failures are reported; ₦30 per successful debit plus the PSSP mark-up.

### 5. Escrow and project accounts

1. The project account mandate adds a **release condition**: the platform's instruction must carry a release token generated only when the evidence rule is satisfied (due-diligence record complete, milestone certified, member vote recorded).
2. The bank verifies the token's signature as part of the mandate check; no token, no payment, regardless of signatories.

### 6. Exceptions

| Event | Bank action |
| --- | --- |
| Instruction fails mandate | Reject with reason; log; no retry without a new approved instruction |
| Suspected fraud or sanctions hit | Hold; compliance review; notify the platform's compliance contact; CBN/NFIU reporting as required |
| Signatory change request | Accept only a general-meeting resolution via the platform's signatory-change flow with the cooling-off period; re-mandate; confirm to all signatories |
| Emergency access (platform outage) | Documented procedure: resolution by the signatories plus the Director's notification, executed at a branch with dual officer approval, reported to the platform for posting |
| Account freeze (court or regulator) | Freeze; notify the platform so member channels show the status |
| Bank system change (API version, webhook format) | Notice period in the agreement; sandbox; parallel run |

### 7. Reporting to the partnership

Monthly: balances per society, interest paid, transaction counts and fees, failed-instruction statistics, direct-debit success rates, virtual-account inventory. Quarterly: compliance review of the collaboration; reconciliation of the bank's and the platform's society-level balances to the kobo.

## Controls, security and compliance in the bank connection

The design assumes the adversary is an insider: a treasurer, a committee member, a platform administrator or a bank officer. Every control below binds one of them.

### Mandate and approval controls

- **Two-of-three signatories** enforced by the bank's mandate, not only by the platform; the platform cannot be configured to send an instruction with fewer approvals than the mandate requires.
- **Approver authentication**: each approval is a separate authenticated act (device-bound passkey or OTP to the signatory's registered number), stored with timestamp, device and IP; approvers cannot approve their own initiations.
- **Limits**: per instruction, per day, per beneficiary, per payment type; general-meeting threshold for reserved matters; anti-splitting rule (same payee and purpose within 30 days aggregates).
- **Evidence binding**: the payment instruction references the evidence record (loan approval, withdrawal eligibility, invoice, due-diligence record, milestone certificate); the release token for project accounts is signed over the evidence hash.
- **Signatory change**: general-meeting resolution, 48-hour cooling-off, notification to all members, bank re-mandate; the platform shows a visible "signatories changing" state during the window.

### Platform administration controls

- No platform administrator can initiate, approve or sign a society payment; administrative access to ledgers is read-only; writes happen only through the recorded business flows.
- Multi-party, time-delayed administration for key rotation, adapter changes and configuration of bye-law parameters, with every change logged and visible to the society.
- The platform's signing keys live in a hardware security module or cloud key service; instructions are signed per society; keys are rotated on a schedule and on any suspected compromise.

### Integrity of the record

- Append-only double-entry ledger with a hash chain; reversals are new entries.
- Immutable schedule versions and stored remittance files; webhook and statement lines stored raw with the bank's references.
- Daily three-way reconciliation (ledger, statements, webhooks) with any difference raised as an item owned by a person.
- External auditor read access to reproduce any month without platform staff.

### Data protection and AML

- NDPA 2023 and GAID 2025 compliance (registration at the applicable level, annual audit return, processor agreements with the bank and PSSPs); member identity data minimised and encrypted; consent captured for any sharing with bureaus or regulators.
- KYC on the society and signatories by the bank; BVN/NIN on members by the platform; transaction monitoring by the bank; suspicious-activity escalation defined in the agreement.
- Tiered limits respected for any member-facing wallet feature; the design avoids member wallets by keeping member value in the society's ledger rather than in individual bank products.

### Regulatory alignment

- The bank obtains CBN prior approval for the collaboration under the payments framework; the platform operates as a PSSP or as the bank's technology provider, never as a deposit taker.
- Cooperative societies remain outside the CBN deposit-taking perimeter as registered societies (FSRCC advisory, 4 Apr 2022); the platform does not change that status, because the society, not the platform, is the account holder.
- Interest paid on society balances is the bank's, under the partnership; any share passed through is disclosed to members.
- Stamp duty (₦50 on transfers of ₦10,000 and above, sender-paid; cooperative-executed instruments exempt) and VAT on fees are applied as the law provides and shown on statements.

### Operational resilience

- Two custody partners where possible; the ledger is portable; a society can move banks without losing history.
- Webhook loss tolerated: statements are the source of truth; webhooks accelerate.
- Sandbox and parallel run for any bank API change; incident procedure with named contacts on both sides.
- Backups across regions; restore tested; retention of ten years.

## What to agree with the bank: the partnership term sheet

| Term | Position to seek | Fallback |
| --- | --- | --- |
| Account structure | Operating and project current accounts per society, in the society's name; virtual accounts per employer at no set-up cost | Virtual accounts at a small monthly fee |
| Mandate | Platform-only channel; two of three signatories; limits per instruction and day; no cheque book; read-only portal; emergency procedure documented | Portal maker-checker mirrored to the platform's approvals as an interim |
| APIs | Transfers with synchronous status and callbacks; account balance; statement feed (daily, with intraday on request); credit and debit webhooks with NIP references; virtual-account management; beneficiary validation | MT940 daily file plus statement API via an aggregator |
| Direct debit | Society enrolled as biller through the bank; mandate creation and debit APIs | Through a licensed PSSP |
| Escrow | Project accounts with release-token verification in the mandate check | Signatory-only mandate on project accounts with platform-side evidence gate |
| Float | Interest on aggregate society balances at an agreed benchmark (for example a margin below the one-year T-bill) paid monthly per society; disclosed to members | Flat savings rate per account |
| Fees | Transfer fees at corporate rates; virtual-account and mandate fees capped; revenue share on fee income above a volume threshold | Standard tariff |
| Service levels | Webhooks within 60 seconds; transfer execution within 5 minutes during NIP hours; API availability 99.9%; incident contacts and a 30-day notice for API changes | Best efforts with named escalation |
| Compliance | Bank obtains CBN approval for the collaboration; bank performs KYC on societies and signatories; platform provides member KYC data under NDPA agreements; joint AML procedures | — |
| Data | The bank shares account-level data only for the society's accounts; the platform shares reconciliation outputs the bank needs for its own audit; no sale of member data | — |
| Exclusivity | None on the platform's side (second custody partner permitted); the bank may be the preferred salary bank for members | Limited exclusivity for a defined pilot period |
| Exit | Societies may move accounts on 60 days' notice; the bank returns balances and history; the platform supports migration | — |
| Pilot | Ten societies, six months, agreed success metrics (auto-match rate, instruction success rate, reconciliation breaks, member channel usage), then general rollout | — |

The partnership is more valuable to the bank than a vendor contract: it brings cooperative deposits the bank cannot otherwise reach, with controls that cut the bank's own fraud and dispute costs, and it positions the bank for the Cooperative Bank of Nigeria and national registry initiatives now under way.

## Open questions and next steps

### To close with candidate banks (Wema, Providus, Sterling first)

1. Will the bank enforce a platform-only, two-of-three mandate on society accounts and verify approver signatures on API instructions? If not, will it route API-initiated batches to approvers on its portal?
2. CoopHub: is a white-label or API available, in whose name are accounts titled, what does it cost, and can the platform's ledger sit around it?
3. Virtual-account pricing direct from the bank (versus Paystack or Kuda intermediaries), and the number of virtual accounts permitted per society.
4. Statement access: REST API or MT940 for mid-size corporate accounts; webhook delivery guarantees and references.
5. Direct debit: can the society be enrolled as a biller directly, or must a PSSP sponsor?
6. Escrow: trustee-product fees and whether a release-token check can be added to a project-account mandate.
7. Float: negotiated call-account rate on aggregate society balances and the reporting cadence.
8. The CBN collaboration approval: who files, what it covers, how long it takes.

### To close with employers and payroll platforms

- IPPIS: onboarding, deduction code, schedule template, remittance advice, contact, fee (nothing is published).
- SystemSpecs: cooperative eligibility for Remita Inflight Collections and per-beneficiary payroll remittance reports.
- Lagos, Kaduna, Ogun payroll units: schedule layout and remittance practice.
- HRIS vendors: third-party deduction export and partner API roadmaps.

### Legal

- Verify the Labour Act deduction-cap text and encode the cap as an agreement term.
- Draft the tripartite deduction agreement with a remittance deadline, interest, schedule obligation and direct-debit fallback.
- NDPA registration level and processor agreements with the bank and PSSPs.

### Next steps, in order

1. Select two candidate banks; present this tab; obtain written positions on the four dual-control options and on float.
2. Choose the pilot ten societies (oil and gas, banks, parastatals) and run historical reconciliation on their last six months to quantify arrears before go-live.
3. Build the minimum loop: virtual accounts, credit webhooks, statement feed, schedule adapters for the pilot employers, the matching engine, the variance queue, in-platform two-of-three approvals with a scoped execution credential, USSD balances.
4. Sign the bank partnership and file for the CBN collaboration approval in parallel with the pilot build.
5. Go live with one society per employer; measure auto-match rate, instruction success, reconciliation breaks and member channel use for three cycles before scaling.
6. Upgrade from in-platform approvals to bank-enforced mandates as soon as a bank agrees; add escrow release tokens for the first estate project.
