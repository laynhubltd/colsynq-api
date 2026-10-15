# Payroll check-off reconciliation: how it works and why it matters

Oct 8, 2026 · @Abubakar

## Why this is the most valuable piece

Payroll check-off is the engine of every staff cooperative: members never touch the money, so loan default is near zero (0.157% at TotalEnergies' society) and savings are compulsory. The same mechanism is also the single largest point of loss, because the money passes through an employer who deducts it and may not remit it, and because no system reconciles what was deducted against what arrived. Fixing reconciliation protects the asset base of the most fundable segment in the market and gives the platform a paying customer on both sides of the transaction.

**The money at stake.** Roughly 3.4 million public employees (720,000 on IPPIS, 2.09m teachers, 372,000 police, 223,000 military) and about 9 million private wage earners are the staff-cooperative base. At ₦5,000–₦20,000 of savings plus loan repayments per member per month, the check-off flow across staff cooperatives is on the order of ₦50–100bn a month. Documented arrears show how much of that goes astray: Ogun State ran 17–21 months behind (2021–22, "billions"); Oyo Water Corporation reached over ₦200m; lenders put IPPIS-held deductions at ₦12bn across 2,400 civil servants (Dec 2024); one small society in Kwara waited three years for ₦4.4m. Kenya, which measures this, reports KSh3.92bn (about ₦40bn) withheld from 104,000 SACCO members in 2025.

**Who pays for it, and why they will.**

- *Societies* lose liquidity, cannot lend, freeze withdrawals and face members who were deducted but not credited. The NNPC Lagos and Police cases show what a liquidity crisis does to a committee.
- *Employers* now carry fiduciary liability: in Oredo LGC (NICN, 14 Jan 2025) the council was ordered to pay ₦112.46m plus 10% interest for deductions it took and kept. Every HR director with a cooperative on the payroll has a reason to want proof of remittance.
- *Members* are deducted for loans they did not take (duplicate and ghost IPPIS deductions, Jan 2024), see one collapsed "third party" line on the payslip, and cannot tell what went where. Kano found 21,000 of 61,000 public servants with little or no take-home pay after stacked deductions (April 2025).
- *Regulators and lenders* have no view at all; Nigeria publishes no employer-arrears figure.

**What breaks without it.** Loans are funded from remittances that never arrive; members are told they owe money they already paid; committees discover arrears from members rather than from the books; disputes take six years to reach judgment; and the cooperative's credit quality, its one real asset, cannot be shown to any lender.

**Why it is buildable now.** Deduction schedules and remittance files already exist as spreadsheets and bank transfers; the gap is the reconciliation, ageing and escalation layer between them. That layer needs no licence, holds no money, and can start with file exchange on day one. It is the capability no Nigerian vendor offers and every Kenyan SACCO core system ships as standard.

## How check-off works today in Nigeria

Check-off is a monthly cycle between four parties: the member who signs a deduction authority, the cooperative that prepares a schedule, the employer's payroll that deducts and pays, and the bank that carries the money. The cycle is the same everywhere; what differs is who runs the payroll and how much of the evidence survives the trip.

### The parties

| Party | Role in the cycle | Holds |
| --- | --- | --- |
| Member | Signs a deduction authority (Labour Act s.5(2) consent); sees the result on a payslip | Payslip, loan agreement |
| Cooperative (secretary or treasurer) | Prepares the monthly deduction list; receives the credit; posts to member ledgers | Member register, schedule, bank statement |
| Employer payroll unit | Applies the list, computes net pay, instructs payment | Payroll master data, deduction codes, payment files |
| Payroll platform | IPPIS (federal), Remita Payroll, state HRMIS, corporate SAP or Oracle, private HRIS | The deduction engine and reports |
| Treasury or finance | Releases cash to fund the payroll and deductions | Cash-backing decisions |
| Banks and NIBSS | Move the lump sum; carry a 100-character narration | NIP references, statements |

### The monthly cycle (what actually happens)

1. **Variation.** Around the middle of the month the cooperative sends the payroll unit a list of changes and standing deductions: name, payroll or IPPIS number, amount, deduction code, months. On IPPIS this is a letter or an Excel "variation" to the MDA or the IPPIS Third-Party Division; on corporate payrolls an email to the payroll officer; on Remita Payroll the employer configures it inside the system.
2. **Payroll run.** The platform computes gross pay, statutory deductions (tax, pension, NHF) and third-party deductions, and produces net pay. IPPIS pays around the 20th–25th; no published calendar or variation cut-off exists.
3. **Cash backing.** Treasury releases funds. On the federal payroll, when cash is short IPPIS has been directed to pay net salaries only and "step down" all deductions: they appear on the payslip but are not remitted (an official position in force since December 2022). States do the same informally.
4. **Payment.** Net pay goes to employees. Third-party totals go out as NIP credits through NIBSS, e-Tranzact, Remita or GIFMIS: one lump sum per beneficiary, expected "in the first week of the following month" on IPPIS, with no schedule file attached and no statement issued to the third party.
5. **Evidence.** The cooperative sees a bank credit and, if lucky, an emailed Excel schedule. The member sees a payslip line. On IPPIS since September 2023, all lender deductions collapse into a single "WACS Loan Account" line (Workers Aggregated Credit Scheme, the IPPIS Third Party Settlement Management System), so the payslip no longer names the beneficiary.
6. **Posting.** The treasurer allocates the credit to members from the previous month's list, by hand.

### The platforms

- **IPPIS (OAGF).** Processes federal executive payroll only; disbursement runs through NIBSS, e-Tranzact, Remita and GIFMIS. Coverage figures conflict (720,000 staff per BPSR 2022; "508 MDAs, 1,008,800+ records" at the July 2026 go-live of a SoftAlliance platform replacing Oracle). CBN, FIRS and NNPC run their own payrolls; police and paramilitary are on IPPIS; the military's status is unverified. There is no published onboarding document, form, fee or portal for third parties; lenders operate through the WACS division under a 2023 OAGF–lenders SLA whose text is unpublished.
- **GIFMIS.** Tertiary institutions left IPPIS in November 2024 for Excel payrolls paid through GIFMIS; the Accountant-General (June 2025) blamed a "hasty" migration for unremitted pension, cooperative, tax and union deductions.
- **TMRAS** (live 4 March 2025) replaced Remita for federal collections and vendor payments; nothing shows it touches salary deductions.
- **Remita.** Three rails: Remita Payroll and HR (the employer's payroll computes cooperative, union and loan deductions; Kano since 2011, Benue 2018, about 2m salaries a month in 2019); Inflight Collections (a licensed lender posts a Loan Disbursement Notification by API and Remita intercepts repayment from any salary it pays, the engine behind *347*780# and bank payday loans); and Remita Direct Debit (a NIBSS bank mandate independent of payroll).
- **State payrolls.** Lagos runs Oracle HR through the Alausa HRMIS; Kaduna a biometric HRMIS; Kano Remita plus a 2025 NIN/BVN deduction tracker; most other states publish nothing. Third parties get on by Head of Service or Accountant-General circular; schedules go to the payroll unit as Excel.
- **Corporate and private payrolls.** SAP, Oracle, HumanManager, SeamlessHR, PaidHR, Bento and Workpay all support recurring voluntary deductions and payslip lines; none publishes a third-party schedule import, a per-beneficiary remittance export or a partner API.

### What works, for contrast

Pension remittance works because the schedule travels with the money: OAGF remits to each RSA, and from 1 June 2025 PenCom's payment platform validates PIN and PFA before paying. NHF remittance does not (₦14.9bn unremitted in 2021–22). Cooperative check-off has the NHF pattern: money without a schedule, and no validation before payment.

### On the one-third cap

Earlier research cites the Labour Act (s.5(7)) as capping total deductions at one-third of wages; the IPPIS research found no OAGF circular applying any cap and documented net pays of ₦2,000 and ₦25,000 after deductions. Treat the cap as a statutory rule that the federal payroll does not enforce, and verify the Labour Act text directly before relying on it in an agreement.

## The monthly cycle at a glance

&#91;embedded content: monthly check-off cycle · 6 steps, 4 loss points\]

The schedule goes out, the money comes back as a lump sum, and the society re-creates the link between them by hand. The reconciliation layer described below sits between steps 4 and 6 and closes the loop with a record at each numbered point.

## Where it breaks: documented failure modes

Every failure below is a case in the research notes with a date and source. They sort into seven modes, and each maps to a specific control in the mechanics section.

| Mode | What happens | Cases | Control that catches it |
| --- | --- | --- | --- |
| **Deducted, not remitted (employer holds the money)** | Payslip shows the deduction; the cooperative's account receives nothing or less | Ogun 17 months (Aug 2021) to 21 months and an indefinite strike (Jun 2022); Oyo Water Corporation ₦200m (Feb 2021); Oyo state-wide 6+ months until ₦1.3bn was paid (Sep 2023); Kwara Water ₦40.2m (2015); NBS Kwara staff cooperative ₦4.4m unremitted since Jan 2022; Edo's 18 LGs (Oct 2024); ASUU "billions" (Sep 2026); lenders' ₦12bn on IPPIS (Dec 2024); Oredo LGC ₦127.46m (2018–22) | Missing-line variance; employer receivable; ageing and escalation |
| **Stepped-down deductions (treasury shortfall)** | Payroll pays net only; all deductions are shown but not paid | Official IPPIS position since Dec 2022 when cash backing is short | Short-remittance variance at employer level; member status "deducted, awaiting remittance" |
| **Duplicate and ghost deductions** | Members deducted twice, or for loans they never took, or after full repayment | FIJ (8 Feb 2024): duplicate deductions across five federal hospitals; deductions for loans never taken (2023; Nov 2024); deductions continuing after non-indebtedness letters (2023, 2025); Kano audit (May 2026): ₦1.8bn over-deducted by loan vendors | Unexpected and duplicate variances; consent reference on every expected line; headroom profile |
| **Opaque payslip** | One collapsed "WACS Loan Account" line since Sep 2023; member cannot see which beneficiary received what | IPPIS lender deductions; promised breakdown app never located | Member view of expected lines on USSD; itemised statement from the cooperative side |
| **Money without a schedule** | Lump-sum credit arrives with no file; treasurer allocates from memory or last month's list | IPPIS remittance practice; most state and corporate payrolls | Unapplied-remittance holding account; no posting without a schedule; schedule adapters |
| **Platform migration and lost records** | Deductions stop or go astray when the payroll moves systems | Tertiary institutions moved from IPPIS to Excel/GIFMIS (Nov 2024) with unremitted cooperative deductions acknowledged by the AGF (Jun 2025); IPPIS Oracle to SoftAlliance (Jul 2026); Police cooperative lost pre-2019 records when it computerised | Versioned schedules and stored remittance files outside the payroll; identifier mapping history |
| **Employer exits the role or caps abruptly** | Government stops acting as third party; all loan deductions suspended | Oyo (Aug 2023) announced it would no longer deduct for cooperatives at source; Kano suspended all loan deductions for three months (Apr 2025) and said it will no longer serve as a third party (May 2026); Niger protests (May 2025) | Direct-debit fallback per member; employer scorecard; liquidity provisioning |

### Two further patterns

- **Consent disputes.** The NICN ordered a ₦455,000 refund where an employee's consent to cooperative deductions was unproven; Kaduna teachers' check-off has gone to a rival union for over three years. The control is a stored, member-specific, signed authority referenced on every line.
- **Stacking across lenders.** Payroll lenders (Credit Direct, FairMoney, Renmoney, CREDICORP) share the same payslip as the cooperative; with no cross-lender view, members end up with near-zero take-home (Kano: 21,000 of 61,000 workers). The control is the headroom profile, fed by payslip data and, when it exists, a bureau.

### What the cases have in common

In every case the cooperative learned of the problem from members or from an empty bank account, never from a system. The remittance had no schedule, the schedule had no version, the payslip had no beneficiary, and nobody owned the gap. Reconciliation turns each of these into a dated, owned, visible record.

## Legal basis and leverage

Nigerian check-off rests on consent and contract, not on a cooperative statute, so the system's job is to make the contract enforceable by producing the evidence a court or a Director needs. Kenya shows what a statutory duty with teeth looks like; the design should be ready for Nigeria to adopt it.

### Nigeria today

| Instrument | What it provides | What it does not |
| --- | --- | --- |
| Labour Act s.5 | Deductions from wages are lawful only where the worker has agreed in writing or the law permits (s.5(2)); earlier research cites a one-third cap on total deductions (s.5(7)), to be verified against the statute text | No remittance deadline, no interest, no cooperative-specific provision |
| Nigerian Co-operative Societies Act | Society's first charge and set-off over a member's savings, shares and dividends (s.16); arbitration of disputes between the society and members before the Director (s.49) | Nothing on employers; arbitration cannot bind a non-member employer |
| Tripartite deduction agreement (member, society, employer) | The only document that fixes the employer's duty: schedule format, remittance deadline, interest, reconciliation, termination | Rarely written; no published model |
| Oredo LGC Staff Cooperative v Oredo LGC (NICN, 14 Jan 2025) | Employer that deducted from Sept 2018 and stopped remitting owed ₦127.46m; held in breach of the tripartite agreement and of fiduciary duty; judgment ₦112.46m plus 10% pre- and post-judgment interest and ₦250,000 costs | Took over six years from first default; full judgment text not located |
| NICN consent cases | Refund of ₦455,000 ordered where consent to cooperative deductions was unproven | Confirms consent must be provable per member |
| Pension Reform Act comparator | Statutory remittance deadline and penalties; PenCom validates PIN and PFA before payment (from 1 Jun 2025) | Not applicable to cooperatives, but the model exists in Nigerian law |

### What an employer can be made to do, and how the system helps

1. **Deduct only on proof of consent.** The system stores each member's signed authority and references it on every schedule line; an employer that deducts without it is exposed (consent cases), and one that refuses a consented deduction is in breach of the agreement.
2. **Remit by a date, with a schedule.** The tripartite agreement should fix a remittance deadline (Kenya's statute uses seven days), require an itemised schedule with every payment, and set interest on late remittance. The system's expected-deduction file is the schedule the agreement names, and its ageing report is the interest calculation.
3. **Answer for the gap.** Oredo establishes fiduciary liability for deducted-but-unremitted sums. The system's monthly employer statement (lines received, lines missing, amount outstanding) is the demand letter and, eventually, the exhibit. A six-year case becomes a reconciled receivable from day one.
4. **Report to the Director.** Although the Director cannot arbitrate against an employer, the society's return can carry employer arrears, and a state-level league table creates administrative pressure where legal pressure is slow.
5. **Allow a fallback.** The agreement should permit members to switch to direct debit when the employer defaults, so the member's loan does not fall into arrears because the employer did.

### The Kenya model to design for

Kenya's Co-operative Societies Act s.35 requires the employer to remit within seven days of deduction or pay compound interest of at least 5% a month; the Commissioner may sue summarily and may appoint any bank or institution holding the employer's funds as a collection agent for twelve months, with the agent liable as the employer after 30 days. SASRA publishes employer non-remittance by employer category each year (KSh3.92bn across 104,331 members in 2025; counties KSh1.88bn, universities KSh726m). Courts treat unremitted deductions as a civil debt, and the Commissioner calls bank-account attachment "the most effective method" (KSh200m recovered from the University of Nairobi). The Cooperatives Bill 2024 re-enacts the duty, and Treasury (Sep 2026) is designing a centralised government payroll that remits SACCO deductions directly, removing the employer from the chain.

Nigeria's Co-operative Societies Act is under review (2025–2030 reform plan, no bill text yet). The three provisions to argue for are Kenya's: a statutory remittance period with interest, an attachment remedy, and mandatory publication of employer arrears. A platform that already produces the schedule, the ageing and the league table is the evidence base for that argument, and the compliance tool once the law passes.

### Elsewhere, briefly

- **Ghana**: the Controller and Accountant-General's third-party system runs an affordability check on net pay and sends a mandate PIN to the employee's phone before a deduction starts; its documented failure is late remittance by the government itself (three months outstanding, Oct 2026).
- **South Africa**: PERSAL banned discretionary deductions after 497,000 state employees were found carrying microloan deductions; Q LINK operates deduction codes, affordability enquiries and electronic mandates, with a 40% cap on discretionary deductions.
- **India**: state cooperative laws make the employer's duty statutory on written requisition and let the society recover from a defaulting employer as arrears of land revenue, ranking with wages.

## What the system does: the mechanics

The capability is a monthly closed loop: the society tells payroll what to deduct, payroll deducts and remits, the system proves what came back line by line, and anything missing becomes an aged, owned, escalating claim on the employer. Six components, each with its data and rules.

### 1. Expected deductions (the schedule)

Every month, before the employer's payroll cut-off, the system computes an **expected deduction** for each member from the ledger, not from a spreadsheet:

- Savings line: the member's standing monthly savings (and any share purchase or special savings).
- Loan lines: the instalment due on each open loan, from the amortisation schedule, including penal interest where in arrears.
- Adjustments: approved changes effective this month (new loan, increase, suspension, exit).

Each expected line carries: society code, employer and payroll identifier (IPPIS number, staff ID or payroll number), member BVN-linked ID, deduction code, line type, amount, period (YYYY-MM), effective dates, and the consent reference proving the member signed a deduction authority. The set is frozen as a **schedule version** with a hash, exported in the employer's format (IPPIS template, Remita, state Excel, SAP or Oracle third-party file), and logged as sent with a timestamp and recipient. A member can see their own expected lines on USSD before pay day.

**Rule:** no expected line without a stored, member-specific, signed consent; no loan line whose addition would take the member's total known deductions above one-third of gross pay (the headroom test below).

### 2. Remittance ingestion

When money arrives the system captures two things: the **bank credit** (from the society's statement feed or a daily statement upload) and the **remittance schedule** the employer sends (Excel, CSV, PDF or portal download). Where an employer sends money without a schedule, the credit is held in an **unapplied remittance** account until the schedule arrives; nothing is posted to members on guesswork.

Parsers normalise each employer's file into the common line shape (payroll identifier, name, amount, code, period). Fuzzy name and identifier matching is allowed only to *propose* a mapping, which a human confirms once; the confirmed mapping is stored so the same employee matches automatically next month.

### 3. Matching

Matching runs in tiers, and every line ends in exactly one state:

| Tier | Match rule | Result |
| --- | --- | --- |
| 1 | Same payroll identifier, same deduction code, same period, same amount | **Matched**: post to the member ledger |
| 2 | Same identifier and period, amount differs | **Partial**: post the amount received; raise an under- or over-deduction variance |
| 3 | Identifier on the schedule, absent from the remittance | **Missing**: expected but not remitted; raise a variance against the employer |
| 4 | Identifier on the remittance, absent from the schedule | **Unexpected**: hold in suspense; raise a variance (non-member, leaver, duplicate, wrong society) |
| 5 | Two remittance lines for one identifier and code in one period | **Duplicate**: post one, hold one, raise a variance |
| 6 | Remittance period differs from schedule period | **Period shift**: match to the open prior period first, then flag |

Totals reconcile three ways every run: schedule total, remittance-file total and bank credit. Any difference between the file and the bank credit is an employer-level variance (short remittance), distinct from member-level variances.

### 4. The variance queue

Every non-matched line becomes a **variance case** with a type, amount, member, employer, period, owner and due date. Cases follow fixed resolution paths:

- *Missing* → confirm with employer: not deducted (member on leave, exited, salary not paid) or deducted-not-remitted (becomes employer arrears).
- *Partial* → confirm cap application or payroll error; adjust the member's loan schedule or claim the shortfall.
- *Unexpected* or *duplicate* → refund to the employer or member within the bye-law period; never silently kept.
- *Short remittance* → employer arrears from day one.

The queue is visible to the committee, the supervisory committee and the auditor; members see the status of their own lines ("deducted, awaiting remittance" is a real status, not an apology).

### 5. One-third-cap headroom

For each member the system keeps a **deduction profile**: gross pay (from payslip upload or employer data), statutory deductions (tax, pension, NHF), the society's own lines, and declared or bureau-reported third-party deductions. Headroom = one-third of gross pay minus all known non-statutory deductions. Loan approval, loan top-up and savings increases are blocked when headroom is insufficient; the calculation and its inputs are stored on the application. Where the employer's payroll applies its own cap and remits less than scheduled, the partial-match variance records that the cap, not default, caused the shortfall, so the member is not penalised.

### 6. Employer-arrears ageing and escalation

Deducted-but-unremitted amounts (confirmed missing lines and short remittances) are posted to an **employer receivable** sub-ledger per employer and aged in buckets: current, 30, 60, 90, 180, 365+ days. The ageing drives automatic actions defined by the society's bye-laws and the tripartite agreement:

| Age | Action |
| --- | --- |
| Day 0 | Remittance reconciled; employer statement issued showing lines received and lines missing |
| Day 15 | Reminder to the employer's payroll contact with the missing-lines schedule |
| Day 30 | Formal demand letter; members on the missing lines notified |
| Day 60 | Escalation to the employer's advisory committee and the State Director of Cooperatives; interest at the agreement rate begins |
| Day 90 | Provision in the society's accounts; new loans to members of that employer require committee override; lenders and auditors see the exposure |
| Day 180 | Legal pre-action notice; option to switch affected members to NIBSS direct debit |
| Day 365 | Litigation decision at the general meeting; receivable reported as impaired |

An **employer scorecard** (on-time rate, average days late, open arrears, schedule quality) is published to the society and, in aggregate, can feed a sector league table of the kind SASRA publishes in Kenya.

### Ledger entries, in brief

- Schedule frozen: no posting (memo only).
- Bank credit received, schedule pending: Dr Bank, Cr Unapplied remittances.
- Line matched: Dr Unapplied remittances, Cr Member savings or Cr Loan principal and interest.
- Line confirmed deducted-not-remitted: Dr Employer receivable, Cr Member savings or loan (the member is credited; the claim moves to the employer) — only where the bye-laws adopt this policy; otherwise the member line stays "awaiting remittance" and nothing posts.
- Unexpected line refunded: Dr Suspense, Cr Bank.

### What this is not

It does not move money, lend, or replace the employer's payroll. It is the system of record that makes every naira deducted traceable to a member and every missing naira traceable to an employer, with a clock running.

## Where the data comes from and how to connect

No Nigerian payroll exposes a published third-party deduction API. Every connection in the first two years is a file, a portal download or a bank feed, so the system is designed around adapters that accept files today and APIs when they appear. The table lists each channel, what is published, and what must be negotiated.

| Channel | Who is on it | Schedule in | Remittance evidence out | Published | Must be negotiated |
| --- | --- | --- | --- | --- | --- |
| IPPIS (OAGF) | \~720,000 federal MDA staff | Excel template uploaded or sent to IPPIS third-party desk under a deduction code | Bulk NIP credit; schedule by email or portal download; single "third party" payslip line | Almost nothing: no onboarding guide, format or fee page online | Deduction-code onboarding, file layout, remittance-advice format, contact route (see the IPPIS section) |
| Remita Payroll and HR (SystemSpecs) | Employers running payroll on Remita (Kano since 2011, Benue 2018, \~2m salaries a month in 2019) | Deductions configured in the employer's payroll; Remita computes them | Remita payroll reports to the employer; credits to beneficiaries | Product pages; pricing page blocked (reported ₦500 per employee set-up, ₦250 per transaction) | Whether a cooperative can receive a per-beneficiary remittance report directly |
| Remita Inflight Collections | Licensed lenders repaying from any Remita-paid salary | Loan Disbursement Notification via API (customer id, authorisation code, account, amount, repayments, dates) | Loan Repayment History API; Stop Loan Collection | API names from snippets; spec blocked | Cooperative eligibility (appears lender-only), fees, employer coverage |
| TMRAS (OAGF, live 4 Mar 2025) | Federal collections | Not payroll | Not payroll | Replaces Remita for FG collections only | Nothing: check-offs stay in IPPIS |
| State payrolls | Lagos (Oracle HR via Alausa HRMIS), Kaduna (HRMIS, biometric), Kano (Remita plus a 2025 NIN/BVN deduction tracker), others unpublished | Approved by Head of Service or Accountant-General circular; Excel schedules to the payroll unit | NIP credit, often without a schedule; arrears common | No state publishes a deduction-code procedure | Everything, state by state; note Oyo (Aug 2023) and Kano (May 2026) have said they will stop deducting for third parties |
| Local governments | LG staff via JAAC payroll | Through the LG treasury | Irregular | Nothing | Edo's 18 LGs had unremitted deductions (Oct 2024) |
| Parastatal and corporate payrolls (NNPC, CBN, FIRS, oil companies, banks, universities) | In-house SAP, Oracle, HumanManager or HRIS | Payroll officer configures recurring deductions; schedule by email | Bulk credit with Excel schedule, sometimes itemised payslips | HRIS vendors (SeamlessHR, PaidHR, Bento, Workpay) support recurring deductions; none publishes a third-party import, per-beneficiary export or partner API | Export format per employer; access to payslip gross pay for the cap test |
| Bank statement feeds | Every society's account | — | Corporate portal CSV or PDF; MT940 on request; Mono statement API (1–12 months, real-time flag); Okra ceased May 2025 | Partially | MT940 or API access per bank; narration content |
| NIBSS bulk payments | Employers paying out | — | CSV with name, account, bank code, amount, narration (≤100 characters); ₦15 per record on NIBSS Pay | Yes | Nothing; but no remittance block travels with the money |
| NIBSS direct debit (fallback) | Members whose employer will not deduct | e-mandate: ₦50 verification transfer within 7 days (₦100 at some banks); fixed or variable mandates up to 365 days | ₦30 per successful debit plus PSSP mark-up (Mono 1% capped ₦1,000 plus VAT, minimum ₦55) | Yes | Whether the society can be a direct biller through its bank |
| Global Standing Instruction | CBN-licensed creditors only | — | — | Yes | Not available to cooperatives |

### How the money itself arrives

Remittances are NIP credits: a single transfer with the schedule emailed separately or not at all, or a bulk upload through a bank portal or NIBSS Pay. No ISO 20022 remittance information travels with Nigerian transfers; the only machine-readable fields are the NIP reference, the narration (100 characters) and the amount. Reconciliation therefore matches the bank credit to the employer's schedule file, never to the payment message alone.

### Connection pattern

1. **Statement adapter** per bank: daily CSV or MT940 pull, or Mono API where the bank is supported; credits tagged by counterparty and narration.
2. **Schedule adapters** per employer format: IPPIS template, Remita payroll report, state Excel layouts, SAP or Oracle exports, HRIS CSVs; each adapter maps columns to the common line shape and is versioned, because employers change layouts without notice.
3. **Employer portal** for payroll officers who have no export: upload the schedule, confirm the mapping once, download the next month's expected-deduction file in their own layout.
4. **Direct-debit adapter** through a PSSP for members outside payroll or during employer arrears.
5. **API adapters** reserved for Remita Inflight Collections (if cooperatives are admitted), HRIS partner APIs as they publish them, and IPPIS if OAGF ever exposes one.

### What to confirm with vendors before building

- SystemSpecs: cooperative eligibility for Inflight Collections, fees, per-beneficiary remittance reports from Remita Payroll, employer list.
- OAGF/IPPIS: onboarding steps, schedule template, remittance-advice format, fee, named contact (see the IPPIS section for what is already known).
- Lagos, Kaduna, Ogun payroll units: schedule layout, remittance practice, circular governing third-party deductions.
- SeamlessHR, PaidHR, HumanManager: third-party deduction export and any partner API roadmap.
- Partner bank: MT940 or API statement access and whether the society can act as a direct-debit biller.

## How Kenya and others do it

The mature systems agree on one pattern: a schedule travels with the money, matching is keyed on a payroll identifier, every unmatched line has a named variance type, partial remittances sit on an employer ledger rather than on members' loans, and a regulator publishes who is late. The Nigerian design borrows each element.

### Kenya: SACCO core systems

- **Vendors** (Kwara, Craft Silicon, Fintech Group, Coretec, Fern) ship check-off as standard: "bulk uploads" of employer schedules, "one receipt distributed to many members", employer ledgers and bank reconciliation. None publishes a file specification; formats are agreed per employer.
- **The explicit model** (documented by the open-source Sawazi system): import the employer's schedule and the remittance per employer and period; match on payroll number, then member number, then national ID, then name; classify variances as short, over, missing, unscheduled or unidentifiable; park unidentifiable lines in suspense with a suggested member; report a collection rate per employer.
- **Infrastructure**: Co-operative Bank's Sacco Link connects 619 SACCOs as a payment channel (not a check-off clearing house); Sacco Central is at pilot stage for 2026; the SACCO Societies (Amendment) Bill 2025 adds a Central Liquidity Facility with daily electronic reporting to SASRA.
- **Mitigation**: many SACCOs route members' whole salaries through their own front-office accounts, so the SACCO deducts itself and the employer problem disappears.

### Kenya: the regulator

SASRA's annual report publishes employer non-remittance as a league table by employer category (KSh2.59bn in 2023, KSh3.49bn in 2024, KSh3.92bn in 2025; 77.55% of it loan repayments), which is the single most effective pressure on counties and universities. Nigeria publishes nothing comparable; the platform can produce it from day one for its own societies.

### Ghana, South Africa, India

- **Ghana**: the Controller and Accountant-General's Third Party Reference System centralises credit-union, welfare and loan deductions with an affordability engine on net pay, a mandate PIN sent to the employee's phone and photo identification on the e-payslip. The government itself remits late (three months outstanding, Oct 2026).
- **South Africa**: Q LINK runs deduction codes, affordability enquiries and reservations, voice and electronic mandates, and agent identifiers in the deduction file; PERSAL bans discretionary deductions; emolument attachment orders are judicially issued and capped at 25%; 8,808 educators were still deducted without consent in 2024.
- **India**: deductions for approved cooperative societies are a permitted wage deduction (Payment of Wages Act s.7), with the cap lifted to 75% when they are included; employers that default can be pursued as for land-revenue arrears.

### UK and US: the file-and-payment pattern

- UK credit-union payroll partners run on "one monthly data file plus one BACS payment" (Leeds Credit Union); Payroll Giving sets a chain of deadlines (employer to agency within 14 days of the tax-month end; agency to charity within 35 days).
- US credit-union cores (CU\*BASE) distinguish payroll deduction, where the file must balance to the funds before anything posts, from ACH, where whatever arrives is posted. The Nigerian design adopts the first rule.

### Remittance data standards worth copying

| Standard | What it carries | Use for Nigeria |
| --- | --- | --- |
| NACHA CCD+ with DED addenda | Case id, pay date, amount, employee id, name, employment-termination indicator with a zero amount | The closest analogue to a check-off line; the termination flag solves the leaver problem |
| NACHA CTX with X12 820 | Up to 9,999 remittance lines in one payment | A bulk remittance with its schedule attached |
| ISO 20022 pain.001 to pacs.008 to camt.054 | Structured remittance information preserved end to end; ISO 11649 creditor reference | The target when NIBSS adopts structured remittance; today only a 100-character narration survives |

Until Nigerian rails carry remittance data, the schedule file is the remittance information, and the system's job is to bind it to the bank credit.

### What Nigeria lacks, in one line each

No statutory remittance period or interest; no attachment remedy; no regulator league table; no payroll file standard; no vendor shipping check-off reconciliation; no central payroll remitting to societies directly. Each is a product feature or a policy ask, and the first five can be delivered by the platform without waiting for the law.

## Acceptance criteria and KPIs

### Acceptance criteria

- **Given** a society's ledger at the payroll cut-off, **when** the schedule is generated, **then** every line derives from a ledger balance or amortisation schedule, carries a consent reference, and the frozen schedule's total equals the sum of its lines; the export validates against the employer's template without manual editing.
- **Given** a remittance file and a bank credit for the same period, **when** reconciliation runs, **then** every schedule line and every remittance line ends in exactly one state (matched, partial, missing, unexpected, duplicate, period shift) within one business day, and schedule total, file total and bank credit are reconciled with every difference assigned to a variance case.
- **Given** a bank credit with no schedule, **when** it is received, **then** it posts to unapplied remittances and no member ledger is credited until the schedule arrives.
- **Given** a remittance line for an identifier not on the schedule, **when** it is ingested, **then** it is held in suspense, a variance case is opened, and no member balance changes.
- **Given** a loan application, **when** it is assessed, **then** the stored headroom calculation shows gross pay, each known deduction and the resulting one-third headroom, and approval is impossible when headroom is below the instalment.
- **Given** confirmed deducted-but-unremitted lines, **when** 15, 30, 60, 90, 180 and 365 days pass, **then** the defined action fires automatically, is logged, and the employer receivable appears in the ageing report in the correct bucket.
- **Given** an employer with arrears over 90 days, **when** a member of that employer applies for a loan, **then** the system requires a committee override and records it.
- **Given** any member, **when** they query USSD or the app, **then** they see their expected lines for the current period and the status of each (scheduled, remitted and credited, awaiting remittance, variance).
- **Given** an auditor, **when** they request it, **then** the system produces for any period the schedule version, the remittance file, the bank credit, the match results and the variance cases, sufficient to reproduce every member posting.
- **Given** an employer HR console, **when** payroll logs in, **then** they see only their own employees' lines, the schedule to apply, the remittance they sent, and their arrears position.

### KPIs

| KPI | Target | Why |
| --- | --- | --- |
| Auto-match rate (tier 1) | ≥ 95% of lines by month 3 per employer | Measures parser and mapping quality |
| Time from bank credit to member posting | ≤ 1 business day where the schedule is present | Members see credits before the next pay day |
| Unapplied remittances older than 10 days | 0 | No money sits unposted |
| Variance cases closed within 30 days | ≥ 90% | Queue does not become a backlog |
| Employer on-time remittance rate | Published monthly per employer | The scorecard that drives behaviour |
| Employer arrears as % of monthly schedule | Tracked; alert above 10% | Liquidity early warning |
| Members with deductions above one-third cap | 0 approved by the society; count of externally caused breaches reported | Legal and welfare compliance |
| Duplicate or unexpected deductions refunded within the bye-law period | 100% | Trust |
| Audit reproduction | Any month reproducible from stored artefacts | Auditability |

## Go-to-market and sequencing

Start where the payroll is reliable and the cooperative is large, then move toward the federal payroll, and treat state governments as a later, policy-dependent market, because two of the largest states have announced they are leaving the check-off role altogether.

### Employer order

1. **Oil and gas, banks, telcos and large parastatals** (NNPC, NLNG, Total, Shell, Chevron, CBN, FIRS, NPA, Customs, NMDPRA): in-house payrolls that pay on time, cooperatives of 1,300–9,000 members with ₦0.4–66bn balance sheets, committees under pressure after the NNPC Lagos freeze. Integration is an Excel export and a payroll officer. Ten of these are the pilot.
2. **Universities and research institutes**: large unionised cooperatives (ASUU, NASU, SSANU chapters) with documented unremitted deductions (Sep–Oct 2026) and a move off IPPIS to GIFMIS and Excel (Nov 2024) that makes reconciliation harder and more valuable.
3. **Federal MDAs on IPPIS**: the biggest single payroll (720,000) but the hardest interface; enter once the IPPIS deduction-code route and file formats are confirmed, using the employer portal and file exchange.
4. **State governments**: only where the state still deducts and remits with a schedule (Lagos, Kaduna); Oyo (Aug 2023) and Kano (May 2026) have said they will stop acting as third party, which converts their members into direct-debit users rather than check-off users.
5. **Private SMEs with HRIS**: through HRIS vendor partnerships once a third-party export or API exists.

### What to sell, to whom

| Buyer | Pain | Pitch | Price |
| --- | --- | --- | --- |
| Cooperative committee and general meeting | Frozen liquidity, member complaints, no proof of what the employer owes | Every deduction traced; employer arrears on one page; members see their lines on USSD | Bundled in ₦150–250 per member per month, or ₦20–50 per deduction line |
| Employer HR and payroll | Fiduciary liability (Oredo: ₦112.46m plus 10% interest), strikes, errors, payslip complaints | Proof of remittance, one-third-cap enforcement across lenders, itemised employee view, fewer disputes | Employer licence per employee per month; free for the first employers that onboard their cooperative |
| Supervisory committee and auditor | Cannot reproduce balances | Month reproducible from stored schedule, file, credit and matches | Included |
| State Director of Cooperatives | No view of arrears or society health | Aggregate arrears and on-time league table across societies in the state | Data agreement, later |
| Lenders to cooperatives (banks, FMBN) | No standard data on a society's cash flow | Verified deduction history and arrears as credit evidence | Data licence, later |

### Sequencing inside an employer

1. Load the member roll and consents; map identifiers to payroll numbers.
2. Run reconciliation on the last three months of historical schedules and credits to show the committee its real arrears position before go-live.
3. Generate the first forward schedule in the employer's layout; payroll officer confirms the mapping.
4. Monthly loop live; employer scorecard shared with HR after month two.
5. Add the HR console once two cycles have matched above 95%.

### Why not start with states or IPPIS

Their arrears make the product most needed there, but their interfaces are unpublished, their payment behaviour is political, and the pilot must prove the matching engine on clean data first. The historical-reconciliation step is the sales tool for those markets later: it quantifies what each government owes.

## Risks, unknowns and the questions to ask

### Risks

| Risk | Likelihood | Impact | Mitigation |
| --- | --- | --- | --- |
| Employers refuse to send schedules or change layouts without notice | High | High | Employer portal with upload and mapping; versioned adapters; make the schedule a term of the tripartite agreement |
| Governments exit the check-off role (Oyo 2023, Kano 2026) | Medium–high in states | High for those societies | Direct-debit fallback per member; treat states as a later market |
| IPPIS never publishes an interface and WACS stays opaque | High | Medium | File exchange through MDA payroll units; member-side payslip capture; lobby through CFAN and the reform process |
| Reconciliation is read as blaming the employer; HR disengages | Medium | Medium | Frame the employer statement as proof of compliance; offer the HR console free to early employers |
| Member identifiers do not match across payroll, BVN and society records | High at onboarding | Medium | Confirmed mapping stored once; historical reconciliation before go-live surfaces mismatches |
| Societies post credits without schedules to keep members happy | Medium | High (silent errors) | Hard rule: no posting without a schedule; unapplied account visible to the supervisory committee |
| Legal uncertainty on the one-third cap | Medium | Medium | Verify the Labour Act text; encode the cap as a configurable agreement term |
| Data protection exposure (payslips, BVN, loans) | Medium | High | NDPA and GAID compliance; minimal payslip retention; member consent for bureau sharing |
| Payroll lenders see the headroom profile as a threat | Medium | Low | Cap enforcement is the employer's interest; sell it to HR |

### Unknowns the research could not close

1. IPPIS third-party onboarding: documents, deduction-code issuance, fee, schedule template, remittance advice, named contact. Nothing is published; lenders operate under an unpublished 2023 SLA.
2. WACS architecture: whether it is a module of the new SoftAlliance platform or a separate settlement ledger, and whether the promised payslip-breakdown app exists.
3. Who remits cooperative deductions for tertiary institutions now paid through GIFMIS, and the current status of the ₦12bn lender arrears.
4. Remita Inflight Collections: whether cooperatives are eligible, fee schedule, employer coverage, and whether Remita Payroll can issue per-beneficiary remittance reports directly to a cooperative.
5. State practice in Lagos, Kaduna and Ogun: schedule layout, remittance advice, governing circular.
6. HRIS vendors' third-party export or API roadmaps (SeamlessHR, PaidHR, HumanManager, Workpay).
7. Bank access: MT940 or API statement feeds and direct-biller status for a cooperative under NIBSS direct debit.
8. The exact Labour Act cap provision and whether any OAGF or state circular applies a cap.

### Questions for the first conversations

**To OAGF / IPPIS Third-Party Division**

- What does a cooperative submit to obtain a deduction code, and in what format is the monthly schedule accepted?
- When and how is the remittance paid, and can a schedule accompany it?
- Does WACS cover cooperatives or only lenders, and can the payslip itemise beneficiaries?
- Is there a fee, and who holds the remittance SLA?

**To SystemSpecs (Remita)**

- Can a registered cooperative use Inflight Collections, and at what fee?
- Can an employer on Remita Payroll authorise per-beneficiary remittance reports to the cooperative?
- Which employers currently run payroll on Remita?

**To the first employers (oil and gas, banks, parastatals)**

- Which payroll system, which export formats, and who owns the deduction file?
- Will payroll accept a generated schedule in its own layout, and send the remittance schedule with the payment?
- Will HR use a console that shows deductions, remittances and the one-third headroom per employee?
- Does a tripartite agreement exist, and does it set a remittance deadline and interest?

**To the partner bank**

- MT940 or API statement feed for society accounts; bulk-credit narration content; direct-biller enrolment for direct debit.

**To the State Director of Cooperatives**

- Will the Director accept employer-arrears figures in the annual return and publish an aggregate?
- Can the Director's office intervene with a state payroll on a society's behalf?
