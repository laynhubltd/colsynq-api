# Functional Requirements Document: Cooperative Operating Platform

Oct 9, 2026 · @Abubakar

## Purpose, scope and conventions

This document specifies what the Cooperative Operating Platform must do for release 1 (staff cooperative pilot) and release 2 (payroll scale-out and estates). It turns the problem statement, the business rules, the payroll check-off design, the bank automation design and the CoopHub review into testable requirements, user stories and acceptance criteria.

### Product in one sentence

A bank-neutral system of record and control for registered cooperatives: member ledger, loans, payroll deduction schedules and line-by-line remittance reconciliation, two-signature payments executed by a partner bank, statutory accounting and governance, and member access over USSD and app. The platform never holds money.

### In scope

| Release | Scope |
| --- | --- |
| **R1 — Staff cooperative pilot** (months 0–9) | M1 society onboarding; M2 members, identity, consent; M3 ledger; M4 savings; M5 loans and guarantors; M6 deduction schedules (file export); M7 remittance reconciliation and employer arrears; M8 payments and approvals; M9 bank integration (virtual accounts, webhooks, statements, transfers); M10 governance (core); M11 year end; M12 exit; M13 statements and reports; M14 USSD and app; M15 employer console (basic); M17 administration and audit |
| **R2 — Payroll scale and estates** (months 9–24) | IPPIS, Remita and state payroll adapters; direct-debit fallback; employer console (full); M16 estates and escrow; investment-club NAV accounting; Director returns; registry (CVN/CoopID) integration when published |

### Out of scope

Holding customer funds; lending from the platform's own balance sheet; consumer group-savings products; ajo collector networks (release 3); SEC-regulated discretionary investment management; tax filing on behalf of societies.

### Conventions

- **Requirement IDs**: FR-M{module}-{nn}, e.g. FR-M7-04. Non-functional: NFR-{nn}.
- **User story IDs**: US-M{module}-{nn}, written as "As a {role}, I want {capability}, so that {outcome}".
- **Acceptance criteria**: Given / When / Then, each independently testable; numbered AC-{story}-{n}.
- **Priority**: **Must** (release blocker), **Should** (in release if possible), **Could** (later).
- **Rule references**: business rules are cited by source, e.g. Act s.34 (Nigerian Co-operative Societies Act), Labour Act s.5, bye-law parameter.
- **Configurable**: any value marked *configurable* is a society-level bye-law parameter set in M1, with the stated default.

## Roles and permissions

Permissions are granted per society. A person may hold several roles, but the system blocks any combination the rules forbid (for example, approving a payment one initiated).

| Capability | Member | Treasurer / secretary | Signatory (3) | Committee | Supervisory committee | Auditor | Employer payroll officer | Platform support | Platform admin |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| View own ledger, loans, expected deductions | ✓ | — | — | — | — | — | — | — | — |
| View all members' ledgers | — | ✓ | ✓ | ✓ | ✓ (read) | ✓ (read) | — | Masked | — |
| Apply for loan, withdrawal, exit | ✓ | — | — | — | — | — | — | — | — |
| Approve loan within limits | — | — | — | ✓ | — | — | — | — | — |
| Build and send deduction schedule | — | ✓ | — | Approve if configured | — | — | — | — | — |
| Work variance queue | — | ✓ | — | ✓ | Read | Read | Own employees' lines | — | — |
| Initiate payment | — | ✓ | — | — | — | — | — | — | — |
| Approve payment | — | — | ✓ (not own initiation) | — | — | — | — | — | — |
| Change bye-law parameters | — | Propose | — | Propose | — | — | — | — | — |
| Record general-meeting resolution | — | ✓ | — | ✓ | — | — | — | — | — |
| Change signatories | — | Initiate with resolution | — | — | — | — | — | — | — |
| Upload remittance schedule | — | ✓ | — | — | — | — | ✓ | — | — |
| Export audit pack | — | ✓ | — | ✓ | ✓ | ✓ | — | — | — |
| Configure adapters, bank connections | — | — | — | — | — | — | — | — | Two-person, time-delayed |
| Post, edit or delete journals directly | — | — | — | — | — | — | — | — | **Never** |

**Separation rules (Must):**

- SR-1: The initiator of a payment cannot approve it.
- SR-2: A signatory cannot approve a payment to themselves, a relative registered in the related-party register, or an entity they control.
- SR-3: No platform role can initiate, approve or sign a society payment.
- SR-4: A member who is a guarantor on a loan cannot approve that loan as a committee member.
- SR-5: Platform administrative actions require two administrators and a 24-hour delay, and are visible in the society's audit log.

## Glossary

| Term | Meaning in this document |
| --- | --- |
| Society | A registered cooperative society (Nigerian Co-operative Societies Act or state law) using the platform |
| Common-bond employer | The employer whose staff form a staff cooperative and whose payroll deducts for it |
| Operating account | The society's current account at the partner bank, under the platform-channel mandate |
| Project account | A separate society account for estate or venture funds, with a release condition |
| Virtual account | A bank-issued account number mapped to the operating account, one per employer, identifying the payer of each credit |
| Mandate | The bank's record of who may instruct the account and how (platform channel, two of three signatories, limits) |
| Deduction authority | A member's signed, specific consent to payroll deductions (Labour Act s.5(2)) |
| Expected line | One member's scheduled deduction for one period and one purpose (savings, loan, share) |
| Schedule version | An immutable, hashed set of expected lines sent to an employer for a period |
| Remittance | The money an employer pays the society for a period, plus the schedule file describing it |
| Unapplied remittance | Money received but not yet matched to member lines |
| Variance | Any expected or remitted line that does not match exactly (partial, missing, unexpected, duplicate, period shift, short remittance) |
| Employer receivable | Deducted-but-unremitted money owed by an employer, aged in buckets |
| Headroom | One-third of gross pay minus all known non-statutory deductions (configurable cap) |
| Appropriation | The year-end split of surplus: reserve (≥25%), education (≤10% of remainder), honoraria, dividend, bonus (Act s.34) |
| Release token | A platform-signed proof that a project-account payment's evidence rule is satisfied |
| CVN / CoopID | Cooperative Verification Number (society) and member ID linked to NIN, from the national registry |
| USSD | Unstructured Supplementary Service Data: menu access from any mobile phone without data |

## M1. Society onboarding and configuration

### Requirements

| ID | Requirement | Priority |
| --- | --- | --- |
| FR-M1-01 | Register a society with registration certificate, registering authority (state or FCT), registration number, date, bye-laws, officers' list and registered address | Must |
| FR-M1-02 | Capture bye-law parameters: minimum monthly savings by grade; loan multiples by product; interest rates and method (flat, reducing); waiting period; guarantors per loan; deduction cap (default one-third of gross pay); signatory rule (default two of three); spending limits per tier; withdrawal limits; exit charges; appropriation percentages; committee size and term limits | Must |
| FR-M1-03 | Changes to bye-law parameters take effect only after a recorded general-meeting resolution, with an effective date; prior values are retained with history | Must |
| FR-M1-04 | Register three signatories with BVN-verified identity, phone and device binding | Must |
| FR-M1-05 | Link the society to a partner bank: operating account, optional project account, mandate reference, API credentials, webhook endpoints | Must |
| FR-M1-06 | Register common-bond employers with payroll platform type, payroll identifier format, cut-off day, pay day and contact | Must |
| FR-M1-07 | Import opening balances (member savings, shares, loans) from a template with validation and a reconciliation to the society's last audited accounts | Must |
| FR-M1-08 | Run historical reconciliation on up to 12 months of past schedules and bank credits to produce opening employer receivables | Should |
| FR-M1-09 | Record the society's CVN when the national registry issues it | Could |

### User stories

**US-M1-01** As a society secretary, I want to register my cooperative and its bye-law rules, so that the system enforces our own rules instead of our memory.

- AC-1: **Given** a secretary with the certificate and bye-laws, **when** they complete registration, **then** the society is created in "pending activation" with every mandatory parameter set or defaulted, and each default is listed for confirmation.
- AC-2: **Given** a parameter outside its permitted range (for example deduction cap above 50% or fewer than two required signatories), **when** it is saved, **then** it is rejected with the permitted range shown.
- AC-3: **Given** the society is pending, **when** the committee confirms parameters and the bank link is verified, **then** the society becomes "active" and the activation is logged with both events.

**US-M1-02** As a committee member, I want parameter changes to require a recorded resolution, so that no officer can quietly change loan or withdrawal rules.

- AC-1: **Given** an active society, **when** a treasurer proposes a new loan multiple, **then** it is saved as "proposed" and has no effect.
- AC-2: **Given** a proposed change, **when** a general-meeting resolution with quorum is recorded referencing it, **then** the change takes effect on the stated date and the prior value remains visible in history.
- AC-3: **Given** a change took effect, **when** any member views the bye-law parameters, **then** they see the current value, the effective date and the resolution reference.

**US-M1-03** As a treasurer, I want to load our existing balances and see last year's arrears, so that we start from a true position.

- AC-1: **Given** an opening-balance file, **when** it is imported, **then** every row is validated (member exists, amounts numeric, loans have schedules) and rejected rows are listed with reasons; nothing posts until all rows pass or are excluded.
- AC-2: **Given** imported balances, **when** the import is confirmed, **then** totals equal the society's last audited balance sheet within ₦1, or the difference is posted to a suspense account requiring committee sign-off.
- AC-3: **Given** past schedules and statements, **when** historical reconciliation runs, **then** an opening employer-receivable balance per employer is produced with line-level detail.

## M2. Members, identity and consent

### Requirements

| ID | Requirement | Priority |
| --- | --- | --- |
| FR-M2-01 | Enrol members with name, BVN or NIN (verified), phone, employer, payroll identifier, grade, confirmation status, date joined, nominee(s) with shares | Must |
| FR-M2-02 | Capture a signed deduction authority per member (e-signature, USSD PIN confirmation or scanned paper), stored immutably with timestamp and referenced by every expected line | Must |
| FR-M2-03 | Allow a member to revoke the deduction authority with a notice period (configurable, default one payroll cycle) | Must |
| FR-M2-04 | Maintain a related-party register: officers' declared relatives and controlled entities | Must |
| FR-M2-05 | Store a deduction profile per member: gross pay (from payslip upload or employer data), statutory deductions, and declared outside deductions | Must |
| FR-M2-06 | Flag duplicate members by BVN, NIN or payroll identifier | Must |
| FR-M2-07 | Support associate members (contract or unconfirmed staff) with restricted product eligibility | Should |
| FR-M2-08 | Verify employment status against an employer file or console where available | Should |

### User stories

**US-M2-01** As a member, I want to consent to deductions once, clearly and provably, so that nobody can deduct from my salary without my agreement.

- AC-1: **Given** a new member, **when** they consent by e-signature or USSD PIN, **then** a consent record is stored with the amount type, the start date, the method and a hash, and the member receives an SMS confirmation.
- AC-2: **Given** no consent on file, **when** a schedule is built, **then** no expected line is created for that member and the member appears in the schedule exceptions list.
- AC-3: **Given** a member revokes consent, **when** the next schedule is built after the notice period, **then** no savings or new-loan lines are created; existing loan lines continue only if the loan agreement permits and that permission is shown.

**US-M2-02** As a treasurer, I want each member's gross pay and other deductions on file, so that I never approve a loan that breaks the one-third rule.

- AC-1: **Given** a payslip upload, **when** it is processed, **then** gross pay, statutory deductions and third-party deductions are recorded with the payslip date.
- AC-2: **Given** a deduction profile older than 3 months (configurable), **when** a loan is applied for, **then** the system requires a newer payslip before approval.

**US-M2-03** As a supervisory committee member, I want officers' relatives and companies registered, so that related-party payments are caught.

- AC-1: **Given** an officer, **when** they take office, **then** they must complete a related-party declaration before receiving approval rights.
- AC-2: **Given** a payment to a registered related party, **when** it is initiated, **then** it is flagged and routed to the general-meeting approval tier.

## M3. Ledger and accounting

### Requirements

| ID | Requirement | Priority |
| --- | --- | --- |
| FR-M3-01 | Maintain a double-entry general ledger per society with a configurable chart of accounts seeded for Nigerian cooperatives (savings, shares, loans, interest income, reserve fund, education fund, unapplied remittances, employer receivables, suspense, bank) | Must |
| FR-M3-02 | Journals are append-only; every journal has an id, timestamp, source event, user or system actor and a hash chained to the previous journal | Must |
| FR-M3-03 | Corrections are made only by reversing journals that reference the original; no journal can be edited or deleted by any role | Must |
| FR-M3-04 | Member sub-ledgers for savings (by type), shares, each loan (principal, interest, penalties) and guarantor exposure | Must |
| FR-M3-05 | Daily bank reconciliation: every bank statement line matched to a journal or raised as an unmatched item with an owner | Must |
| FR-M3-06 | Period close (monthly) locks the period; postings after close go to the next open period with a reference | Must |
| FR-M3-07 | Trial balance, balance sheet and income statement on demand for any closed period | Must |
| FR-M3-08 | Replay: rebuild all balances from journal one and compare to stored balances; any difference raises a critical alert | Must |
| FR-M3-09 | Unit (NAV) accounting for investment-club ledgers | Could (R2) |

### User stories

**US-M3-01** As an external auditor, I want to reproduce any balance from the journals and bank statements without platform help, so that I can certify the accounts independently.

- AC-1: **Given** auditor access, **when** they export a period, **then** the export contains every journal, the hash chain, the bank statements, the schedule versions, remittance files and match results for the period.
- AC-2: **Given** the export, **when** journals are summed by account, **then** they equal the trial balance to the kobo.
- AC-3: **Given** any journal in the chain is altered in the export, **when** the hash chain is verified, **then** verification fails at that journal.

**US-M3-02** As a treasurer, I want the ledger and the bank to agree every day, so that a missing naira is found the day it goes missing.

- AC-1: **Given** the day's statement feed, **when** reconciliation runs at end of day, **then** ledger cash equals bank cash, or every difference appears as an unmatched item with amount, date and owner.
- AC-2: **Given** an unmatched item older than 7 days (configurable), **when** the daily job runs, **then** it is escalated to the committee and the supervisory committee.

**US-M3-03** As a committee member, I want a mistaken posting corrected without erasing history, so that the audit trail stays intact.

- AC-1: **Given** a posted journal, **when** a correction is requested, **then** the only available action is a reversal plus a new journal, both referencing the original and a reason.
- AC-2: **Given** any user including platform administrators, **when** they attempt to edit or delete a posted journal through any interface or API, **then** the action is refused and the attempt is logged.

## M4. Savings and contributions

### Requirements

| ID | Requirement | Priority |
| --- | --- | --- |
| FR-M4-01 | Savings products per society: ordinary, special, target (education, festive, retirement) with interest rate, lock period and withdrawal rules | Must |
| FR-M4-02 | Standing monthly savings per member, at or above the grade minimum (configurable), feeding the deduction schedule | Must |
| FR-M4-03 | Share capital purchases with the one-fifth cap per member (Act s.27) enforced | Must |
| FR-M4-04 | Withdrawal requests subject to rules: not below lien for loans or guarantees; withdrawal limit (default 60% once a year, configurable); notice period; charges | Must |
| FR-M4-05 | Dormancy: account marked dormant after N months without contribution (configurable, default 6) | Must |
| FR-M4-06 | Interest on savings accrued monthly and credited per product rules | Should |
| FR-M4-07 | Voluntary contributions by bank transfer to a member-specific reference or virtual account, matched automatically | Should |

### User stories

**US-M4-01** As a member, I want to change my monthly savings amount, so that my contribution fits my pay.

- AC-1: **Given** a member, **when** they request a new amount via app or USSD, **then** the amount is accepted only if it is at or above their grade minimum and within headroom, and takes effect from the next schedule build.
- AC-2: **Given** a change after the schedule is frozen, **when** it is submitted, **then** the member is told it applies from the following month.

**US-M4-02** As a member, I want to withdraw part of my savings when I need it, so that my money is not trapped.

- AC-1: **Given** a withdrawal request, **when** it is submitted, **then** the available amount shown equals balance minus loan lien, guarantee lien and minimum balance, capped by the configured withdrawal limit.
- AC-2: **Given** an eligible request, **when** it is approved under the payments workflow (M8), **then** funds are paid to the member's verified bank account and the ledger posts only on bank confirmation.
- AC-3: **Given** an ineligible request, **when** it is submitted, **then** the reason (lien, limit, notice) is shown and nothing is created.

**US-M4-03** As a committee member, I want share purchases capped by law, so that no member exceeds one-fifth of share capital.

- AC-1: **Given** a share purchase that would take a member above 20% of total share capital, **when** it is submitted, **then** it is rejected with the maximum permitted amount shown.

## M5. Loans and guarantors

### Requirements

| ID | Requirement | Priority |
| --- | --- | --- |
| FR-M5-01 | Loan products per society: normal, special, emergency, asset, housing; each with multiple of savings, maximum amount, tenor, rate, method (flat, reducing), up-front charges, penal rate | Must |
| FR-M5-02 | Eligibility checks at application: waiting period met; no loan in arrears; amount within multiple and single-obligor limit; headroom within cap after the new instalment; guarantors meet rules | Must |
| FR-M5-03 | Guarantor workflow: each named guarantor consents by app or USSD; system checks membership, confirmation status, years to retirement (configurable) and total guarantee exposure | Must |
| FR-M5-04 | Approval by committee within limits; above limits escalates to the configured tier; guarantors and the applicant cannot approve | Must |
| FR-M5-05 | Disbursement through the payments workflow (M8); loan is active only on bank confirmation | Must |
| FR-M5-06 | Amortisation schedule generated and stored; instalments feed the deduction schedule (M6) | Must |
| FR-M5-07 | Arrears classification: instalment due and unpaid because of the member vs because of employer non-remittance (employer-caused arrears do not age the member's loan) | Must |
| FR-M5-08 | Default recovery order: set-off against savings, shares and dividends (Act s.16); then guarantors pro rata; then terminal benefits where bye-laws allow | Must |
| FR-M5-09 | Top-up and refinance with the same eligibility checks | Should |
| FR-M5-10 | Loan agreement generated and e-signed with the amortisation schedule attached | Should |

### User stories

**US-M5-01** As a member, I want to apply for a loan from my phone and know at once whether I qualify, so that I don't wait weeks for a no.

- AC-1: **Given** a member, **when** they open a loan application, **then** the maximum eligible amount is shown with the limiting factor (multiple, headroom, single-obligor limit, waiting period).
- AC-2: **Given** an application above the maximum, **when** submitted, **then** it is rejected with the limiting factor named.
- AC-3: **Given** an eligible application, **when** submitted, **then** each named guarantor receives an SMS and app request to consent, and the application waits for all consents.

**US-M5-02** As a guarantor, I want to see and accept exactly what I am guaranteeing, so that I am not exposed without knowing.

- AC-1: **Given** a guarantee request, **when** the guarantor opens it, **then** they see borrower, amount, tenor, their share of exposure and their total current guarantee exposure.
- AC-2: **Given** a guarantor's total exposure would exceed the configured limit, **when** they try to accept, **then** acceptance is blocked with the limit shown.
- AC-3: **Given** a guarantor accepts, **when** the loan is disbursed, **then** their savings carry a guarantee lien that blocks withdrawal and exit until the borrower has repaid the configured share.

**US-M5-03** As a committee member, I want to approve loans with every check already done, so that I decide on judgement, not arithmetic.

- AC-1: **Given** an application with consents, **when** it reaches the committee, **then** the view shows eligibility results, headroom calculation with inputs, guarantor checks, related-party flags and the member's repayment history.
- AC-2: **Given** an approver who is the applicant, a guarantor or a related party, **when** they try to approve, **then** approval is refused.

**US-M5-04** As a member, I don't want my loan marked late when my employer kept the money, so that I am not penalised for their default.

- AC-1: **Given** an instalment line marked "deducted, awaiting remittance" in M7, **when** the instalment falls due, **then** the loan status is "current — employer arrears" and no penal interest accrues.
- AC-2: **Given** the employer later remits, **when** the line is matched, **then** the instalment is posted as paid on the original due date.

**US-M5-05** As a treasurer, I want defaulted loans recovered in the legal order, so that recovery is fair and defensible.

- AC-1: **Given** a loan in member-caused default beyond the configured days, **when** recovery is triggered, **then** the system proposes set-off from the borrower's savings, shares and dividends first, then guarantors pro rata to their guarantee, and shows each amount.
- AC-2: **Given** the proposal, **when** approved by the committee, **then** journals post with references to the loan and the bye-law clause, and each affected member is notified.

## M6. Payroll deduction schedules

### Requirements

| ID | Requirement | Priority |
| --- | --- | --- |
| FR-M6-01 | Build expected lines per member per period from the ledger: standing savings, share purchases, loan instalments from amortisation schedules, approved changes | Must |
| FR-M6-02 | Validate every line: consent on file; headroom within cap; member active and confirmed for the product; no line for a member under a variance hold (configurable) | Must |
| FR-M6-03 | Freeze the schedule as an immutable version with a hash, period, employer, line count and total | Must |
| FR-M6-04 | Export the version in the employer's layout through a versioned adapter (R1: generic Excel/CSV and two pilot employer layouts; R2: IPPIS, Remita, state layouts) | Must |
| FR-M6-05 | Record delivery: channel, recipient, timestamp; employer acknowledgement in the console | Must |
| FR-M6-06 | Optional committee approval of the schedule before sending (configurable) | Should |
| FR-M6-07 | Publish each member's expected lines to USSD and app; SMS summary on freeze | Must |
| FR-M6-08 | Schedule exceptions report: members excluded and why | Must |
| FR-M6-09 | Scheduler builds the schedule automatically N days before the employer's cut-off (configurable, default 5) | Must |

### User stories

**US-M6-01** As a treasurer, I want the monthly deduction schedule built from our records, so that I stop typing it into Excel.

- AC-1: **Given** an employer with cut-off day D, **when** day D minus 5 arrives, **then** a draft schedule is built containing one line per member per deduction type, each with payroll identifier, amount, code, period and consent reference.
- AC-2: **Given** the draft, **when** the treasurer opens it, **then** totals by type, line count, changes from last month and the exceptions list are shown.
- AC-3: **Given** the treasurer (and committee if configured) approves, **when** the schedule is frozen, **then** a version with hash is stored and cannot be changed; corrections create a new version.

**US-M6-02** As a payroll officer, I want the schedule in my payroll's own format, so that I can load it without retyping.

- AC-1: **Given** a frozen version and an employer layout, **when** it is exported, **then** the file validates against the layout's column and type rules and its total equals the version total.
- AC-2: **Given** the export, **when** the officer downloads it from the console, **then** the download is logged and the society sees "delivered".

**US-M6-03** As a member, I want to see what will be deducted before pay day, so that I can query it in time.

- AC-1: **Given** a frozen schedule, **when** the member dials USSD or opens the app, **then** they see each expected line with type and amount for the period.
- AC-2: **Given** the member disputes a line, **when** they raise a query, **then** a case is created for the treasurer with the line reference.

## M7. Remittance reconciliation and employer arrears

### Requirements

| ID | Requirement | Priority |
| --- | --- | --- |
| FR-M7-01 | Record each bank credit to an employer's virtual account in unapplied remittances for that employer | Must |
| FR-M7-02 | Ingest remittance schedules by console upload, email ingestion or API; parse through the employer's adapter; store the raw file | Must |
| FR-M7-03 | Identifier mapping: propose matches for unknown identifiers (fuzzy name, BVN); require one human confirmation; store the mapping for future periods | Must |
| FR-M7-04 | Matching engine classifies every line: matched, partial, missing, unexpected, duplicate, period shift | Must |
| FR-M7-05 | Post matched lines to member sub-ledgers; never post any line without a schedule | Must |
| FR-M7-06 | Three-way balance: schedule total, remittance-file total, bank credit; file-to-credit difference becomes a short-remittance receivable | Must |
| FR-M7-07 | Variance cases with type, amount, member, employer, period, owner, due date, status and resolution path | Must |
| FR-M7-08 | Employer receivable sub-ledger per employer, aged in buckets (0, 30, 60, 90, 180, 365+ days from pay date) | Must |
| FR-M7-09 | Escalation actions on ageing thresholds (configurable): reminder, demand letter, Director escalation, provision, loan override, pre-action notice | Must |
| FR-M7-10 | Employer statement and scorecard: lines received, lines missing, arrears, on-time rate, average days late | Must |
| FR-M7-11 | Member status per line: scheduled, credited, deducted-awaiting-remittance, variance | Must |
| FR-M7-12 | Refund workflow for unexpected and duplicate lines within the bye-law period | Must |
| FR-M7-13 | Switch selected members to direct debit after employer default (M9) | Should (R2) |

### User stories

**US-M7-01** As a treasurer, I want each employer credit matched to members automatically, so that posting takes minutes, not days.

- AC-1: **Given** a bank credit and a remittance file for the same employer and period, **when** reconciliation runs, **then** every schedule line and every file line ends in exactly one classification within one business day.
- AC-2: **Given** tier-1 matches, **when** reconciliation completes, **then** they are posted to member sub-ledgers and each member's line status becomes "credited".
- AC-3: **Given** schedule total, file total and credit, **when** reconciliation completes, **then** all three are shown; any file-to-credit gap is posted as a short-remittance receivable.

**US-M7-02** As a treasurer, I want money that arrives without a schedule held aside, so that we never guess whose money it is.

- AC-1: **Given** a credit with no schedule, **when** it lands, **then** it is posted to unapplied remittances and no member balance changes.
- AC-2: **Given** no schedule after 1 business day, **when** the daily job runs, **then** a reminder goes to the employer contact; after 5 business days the committee is alerted.

**US-M7-03** As a committee member, I want to know exactly what each employer owes us and for how long, so that we can act before liquidity runs out.

- AC-1: **Given** confirmed deducted-not-remitted lines, **when** the arrears report is opened, **then** each employer's receivable is shown by age bucket with drill-down to member lines.
- AC-2: **Given** an employer receivable crosses 30, 60 or 90 days, **when** the daily job runs, **then** the configured action is generated, logged and shown on the employer's statement.
- AC-3: **Given** arrears above 90 days for an employer, **when** a member of that employer applies for a loan, **then** committee override is required and recorded.

**US-M7-04** As a member, I want to know whether my deduction reached the cooperative, so that I can see if my employer is holding my money.

- AC-1: **Given** a member line, **when** the member checks USSD or app, **then** the status shows scheduled, credited (with date), deducted-awaiting-remittance (with days outstanding) or variance (with case number).

**US-M7-05** As a treasurer, I want unexpected or duplicate money refunded properly, so that we never keep money that isn't ours.

- AC-1: **Given** an unexpected or duplicate line, **when** it is confirmed, **then** a refund payment is created to the employer or member through M8 within the bye-law period, and the line closes only on bank confirmation.

## M8. Payments and approvals

### Requirements

| ID | Requirement | Priority |
| --- | --- | --- |
| FR-M8-01 | Payment types: loan disbursement, savings withdrawal, exit payout, refund, dividend, supplier, project (estate) | Must |
| FR-M8-02 | Initiation requires purpose, beneficiary (validated by name enquiry), amount, and the evidence record for the type (loan approval id; withdrawal eligibility; invoice; due-diligence record; milestone certificate) | Must |
| FR-M8-03 | Limits per instruction, per day, per beneficiary and per type; above-limit payments escalate to the next approval tier | Must |
| FR-M8-04 | Anti-splitting: payments to the same beneficiary and purpose within 30 days (configurable) are aggregated against limits | Must |
| FR-M8-05 | Approval by two of three signatories (configurable, never fewer than two); each approval is a separate authenticated act (device-bound passkey or OTP) | Must |
| FR-M8-06 | Separation rules SR-1 to SR-4 enforced | Must |
| FR-M8-07 | On quorum, the platform signs and sends the instruction to the bank; ledger posts only on bank confirmation | Must |
| FR-M8-08 | Failed instructions reopen with the bank reason code; nothing posts | Must |
| FR-M8-09 | Bulk payments (dividends, refunds) as one approval over a frozen list | Must |
| FR-M8-10 | Signatory change: general-meeting resolution, 48-hour cooling-off (configurable, minimum 24), notification to all members, bank re-mandate | Must |
| FR-M8-11 | Payment expires if not approved within N days (configurable, default 7) | Should |

### User stories

**US-M8-01** As a treasurer, I want to initiate payments with the evidence attached, so that approvers can decide quickly and safely.

- AC-1: **Given** a payment type, **when** the treasurer initiates it, **then** the system requires the evidence record for that type and blocks submission without it.
- AC-2: **Given** the beneficiary account, **when** it is entered, **then** a name enquiry runs and the returned name is shown; a mismatch with the member or vendor name requires a reason.

**US-M8-02** As a signatory, I want to approve from my phone with everything I need to see, so that I never sign blind.

- AC-1: **Given** a pending payment, **when** the signatory opens it, **then** they see purpose, beneficiary, verified name, amount, evidence, related-party flags and the ledger effect.
- AC-2: **Given** the signatory initiated the payment, is the beneficiary, or is a related party, **when** they try to approve, **then** approval is refused.
- AC-3: **Given** the second approval, **when** it is recorded, **then** the instruction is sent to the bank within 60 seconds and both approvers are notified.

**US-M8-03** As a general meeting, we want a signatory change to be slow and visible, so that nobody can capture the account overnight.

- AC-1: **Given** a request to change signatories, **when** it is initiated, **then** a recorded general-meeting resolution with quorum is required.
- AC-2: **Given** the resolution, **when** the change is submitted, **then** all members are notified and a cooling-off timer starts; during the window the society shows "signatories changing" and no new payment can be approved by an incoming signatory.
- AC-3: **Given** the window ends, **when** the bank confirms the re-mandate, **then** the new signatories become active and the old ones lose approval rights.

**US-M8-04** As an auditor, I want every payment traceable from request to bank, so that I can test controls.

- AC-1: **Given** any payment, **when** viewed, **then** initiator, approvers with timestamps and device, evidence, instruction, bank reference and journal are linked.

## M9. Bank integration

### Requirements

| ID | Requirement | Priority |
| --- | --- | --- |
| FR-M9-01 | Adapter per partner bank for: balance; transfer initiation and status; statement retrieval; credit and debit webhooks; virtual-account creation and listing; name enquiry | Must |
| FR-M9-02 | One virtual account per employer per society, created on employer onboarding | Must |
| FR-M9-03 | Webhooks are idempotent on the bank reference; replays ignored and logged | Must |
| FR-M9-04 | Statements are the source of truth; webhooks accelerate; daily three-way check of ledger, statements and webhooks | Must |
| FR-M9-05 | Instructions signed per society with keys in an HSM or cloud KMS; key rotation scheduled | Must |
| FR-M9-06 | Support at least two partner banks; a society can migrate between banks with history intact | Should |
| FR-M9-07 | NIBSS direct-debit mandates via the bank or a PSSP: create, activate, debit, cancel; results reconciled like remittances | Should (R2) |
| FR-M9-08 | Project accounts with release-token verification where the bank supports it; otherwise platform-side evidence gate | Should (R2) |
| FR-M9-09 | Bank outage handling: queue instructions, retry with backoff, alert; never execute twice | Must |

### User stories

**US-M9-01** As a treasurer, I want each employer to pay into its own account number, so that I always know who paid.

- AC-1: **Given** a new employer, **when** it is onboarded, **then** a virtual account is created at the partner bank and shown on the employer's console and statement.
- AC-2: **Given** a credit to that virtual account, **when** the webhook arrives, **then** the credit is attributed to that employer within 60 seconds.

**US-M9-02** As a treasurer, I want the ledger to reconcile itself to the bank, so that I don't download statements.

- AC-1: **Given** the end of day, **when** the statement job runs, **then** every statement line is matched to a journal or raised as unmatched.
- AC-2: **Given** a webhook that was never received, **when** the statement line appears, **then** the credit is recorded from the statement and flagged "webhook missing".

**US-M9-03** As a platform engineer, I want bank failures to be safe, so that no payment is ever sent twice or lost.

- AC-1: **Given** a transfer call that times out, **when** retried, **then** the same idempotency key is used and the bank status is queried before any resend.
- AC-2: **Given** the bank reports success after a timeout, **when** reconciled, **then** exactly one journal posts.

**US-M9-04** As a member whose employer stopped remitting, I want to pay by direct debit, so that my loan stays current. *(R2)*

- AC-1: **Given** consent and a bank account, **when** a mandate is created, **then** the member is told to make the ₦50 verification transfer and the mandate status is tracked to active.
- AC-2: **Given** an active mandate, **when** the instalment falls due, **then** the debit runs, and success or failure is reconciled to the member's loan.

## M10. Governance and meetings

### Requirements

| ID | Requirement | Priority |
| --- | --- | --- |
| FR-M10-01 | Officers register: role, start date, term length, consecutive-term limit (configurable, default two 2-year terms); alerts before term end | Must |
| FR-M10-02 | Meetings: AGM, general and committee meetings with notice (default 21 days for general meetings), agenda, attendance, quorum (default one-quarter of members), minutes | Must |
| FR-M10-03 | Resolutions recorded with type (ordinary, special at 75%), votes, and links to the actions they authorise | Must |
| FR-M10-04 | Remote voting by app or USSD for resolutions where bye-laws permit | Should |
| FR-M10-05 | Related-party transactions require general-meeting approval and are listed in the annual report | Must |
| FR-M10-06 | Members may inspect books (request logged, fulfilled within 7 days) | Should |

### User stories

**US-M10-01** As a member, I want the AGM to happen on time with the accounts in front of me, so that the committee answers to us.

- AC-1: **Given** the financial year closed, **when** audited accounts are uploaded, **then** the system proposes an AGM date and generates the notice at least 21 days ahead, sent to every member by SMS and app.
- AC-2: **Given** the AGM date passes without a recorded meeting, **when** 30 days elapse, **then** the supervisory committee and (if configured) the Director are notified.

**US-M10-02** As a committee member, I want officers' terms tracked, so that nobody overstays.

- AC-1: **Given** an officer within 60 days of term end, **when** the daily job runs, **then** the committee and the officer are notified.
- AC-2: **Given** an officer at the term limit, **when** re-election is recorded, **then** the system blocks it unless a resolution waiving the limit (where bye-laws allow) is attached.

## M11. Year end and surplus appropriation

### Requirements

| ID | Requirement | Priority |
| --- | --- | --- |
| FR-M11-01 | Year-end close producing financial statements and the surplus for appropriation | Must |
| FR-M11-02 | Appropriation engine enforcing: reserve ≥ 25% of net surplus; education fund ≤ 10% of the remainder (with Director approval flag); honoraria ≤ bye-law cap (default 7.5%); dividend on shares (cap configurable); bonus by patronage | Must |
| FR-M11-03 | No dividend or bonus payable until committee approval and AGM confirmation are recorded | Must |
| FR-M11-04 | Dividend and bonus calculated per member, set off against arrears where bye-laws allow, paid as a bulk payment (M8) | Must |
| FR-M11-05 | Dividend reported in naira and as a percentage of average savings | Must |
| FR-M11-06 | Member annual statement: opening, contributions, interest, dividends, loans, closing | Must |

### User stories

**US-M11-01** As a treasurer, I want the surplus split by the law automatically, so that our accounts are compliant without a consultant.

- AC-1: **Given** a net surplus, **when** appropriation is proposed, **then** the reserve line is at least 25%, the education line at most 10% of the remainder, honoraria within the cap, and the remainder split between dividend and bonus per the committee's proposal.
- AC-2: **Given** a proposal breaching any rule, **when** saved, **then** it is rejected with the rule and the permitted value shown.
- AC-3: **Given** a valid proposal, **when** committee approval and AGM confirmation are recorded, **then** member-level dividend and bonus amounts are computed and a bulk payment list is created for approval.

**US-M11-02** As a member, I want to see what my savings actually earned this year, so that I can judge the cooperative's performance.

- AC-1: **Given** dividends paid, **when** the member opens their annual statement, **then** dividend and bonus are shown in naira and as a percentage of their average savings.

## M12. Exit, retirement and death

### Requirements

| ID | Requirement | Priority |
| --- | --- | --- |
| FR-M12-01 | Exit request with notice period (configurable, default 1–3 months) | Must |
| FR-M12-02 | Exit eligibility: no outstanding loan, no active guarantee lien above the release threshold | Must |
| FR-M12-03 | Exit computation: savings + shares + accrued dividends − sums due − exit charge; statement to the member | Must |
| FR-M12-04 | Retirement: automatic flag on employer-file retirement date; deduction lines stop; retiree status per bye-laws | Must |
| FR-M12-05 | Death: nominee claim workflow with documents; payout of balance less sums due; guarantor or write-off rule for outstanding loans | Must |
| FR-M12-06 | Past-member liability period (Act s.29, default 2 years) tracked | Should |
| FR-M12-07 | Payout within the bye-law period; overdue exits escalate to the supervisory committee | Must |

### User stories

**US-M12-01** As a retiring member, I want my money paid on time with a clear statement, so that I am not chasing the cooperative after 35 years.

- AC-1: **Given** a retirement date in the employer file, **when** it is 60 days away, **then** the member is notified with an estimated exit statement.
- AC-2: **Given** the exit date, **when** eligibility passes, **then** a payout is created for approval with the itemised computation, and paid within the bye-law period.
- AC-3: **Given** a payout not made within the bye-law period, **when** the daily job runs, **then** the supervisory committee is alerted and the member sees the delay and its reason.

**US-M12-02** As a nominee, I want to claim a deceased member's balance simply, so that the family is not delayed.

- AC-1: **Given** a registered nominee, **when** they submit a claim with death certificate, **then** the claim is verified against the nominee register and the computation is shown.
- AC-2: **Given** an outstanding loan, **when** the claim is computed, **then** the bye-law rule (guarantor, insurance, write-off) is applied and explained.

## M13. Reporting, statements and regulatory returns

### Requirements

| ID | Requirement | Priority |
| --- | --- | --- |
| FR-M13-01 | Monthly society dashboard: cash, unapplied remittances, employer arrears by bucket, loan book, NPL ratio (member-caused only), liquidity ratio, illiquid-asset share against cap, dividend-on-savings to date | Must |
| FR-M13-02 | Member monthly statement (app, USSD summary, PDF on request) and annual statement | Must |
| FR-M13-03 | Financial statements and appropriation statement for any closed year | Must |
| FR-M13-04 | Audit pack export for any period (journals, hash chain, statements, schedules, remittances, matches, payments, approvals) | Must |
| FR-M13-05 | Director of Cooperatives annual return pack per state template | Should (R2) |
| FR-M13-06 | Registry integration (CVN, CoopID) and CoopCHECK reporting under consent, when published | Could (R2) |
| FR-M13-07 | Compliance calendar with alerts 30 and 7 days before: AGM, audit, Director return, CAC (if applicable), tax | Must |
| FR-M13-08 | Anonymised cross-society metrics (arrears, default, on-time rates) for authorised regulators and lenders | Could (R2) |

### User stories

**US-M13-01** As a committee member, I want one monthly page with our health figures, so that we see problems before members do.

- AC-1: **Given** a closed month, **when** the dashboard is opened, **then** every figure in FR-M13-01 is shown with its definition and the prior three months.
- AC-2: **Given** liquidity below the configured floor or illiquid assets above the cap, **when** the month closes, **then** the committee and supervisory committee are alerted.

**US-M13-02** As a supervisory committee member, I want to export everything for a quarter, so that our inspection uses evidence, not assurances.

- AC-1: **Given** a period, **when** the audit pack is exported, **then** it contains every artefact in FR-M13-04 and a manifest with counts and totals that tie to the trial balance.

## M14. Member channels (USSD, SMS, app)

### Requirements

| ID | Requirement | Priority |
| --- | --- | --- |
| FR-M14-01 | USSD menu: balance (savings, shares, loans); expected deductions this month with status; last 5 transactions; loan eligibility; consent to deduction; guarantor consent; vote (if enabled); query | Must |
| FR-M14-02 | USSD authentication by PIN set at enrolment; PIN reset via verified phone and BVN challenge | Must |
| FR-M14-03 | SMS notifications: schedule frozen, credit posted, deducted-awaiting-remittance, loan status, approvals needed, meetings, signatory changes | Must |
| FR-M14-04 | Mobile app and web with full member functions and committee functions | Must |
| FR-M14-05 | Languages: English first; Yoruba, Hausa, Igbo, Pidgin | Should |
| FR-M14-06 | Agent-assisted enrolment and queries | Could (R3) |

### User stories

**US-M14-01** As a member with a basic phone, I want my balance and deductions by USSD, so that I am not excluded for lacking a smartphone.

- AC-1: **Given** a member dials the code and enters their PIN, **when** they choose "Balance", **then** savings, shares and loan balances as of the last posting are shown within one USSD session.
- AC-2: **Given** "This month's deductions", **when** chosen, **then** each expected line with its status is shown.
- AC-3: **Given** three wrong PINs, **when** the third is entered, **then** USSD access is locked and an SMS explains how to reset.

**US-M14-02** As a member, I want to be told when my deduction did not reach the cooperative, so that I can act.

- AC-1: **Given** a member line becomes deducted-awaiting-remittance, **when** reconciliation completes, **then** the member receives an SMS with the amount and the employer.

## M15. Employer console

### Requirements

| ID | Requirement | Priority |
| --- | --- | --- |
| FR-M15-01 | Payroll officer login scoped to their employer and societies | Must |
| FR-M15-02 | Download frozen schedules in the employer's layout; acknowledge receipt | Must |
| FR-M15-03 | Upload remittance schedules; see parse results and mapping prompts | Must |
| FR-M15-04 | Employer statement: lines received, missing, arrears by bucket, virtual account details | Must |
| FR-M15-05 | Per-employee headroom view across all known deductions (with member consent) | Should (R2) |
| FR-M15-06 | Multi-society view for employers with several cooperatives | Should |

### User stories

**US-M15-01** As a payroll officer, I want to collect the schedule and send back the remittance in one place, so that I stop emailing spreadsheets.

- AC-1: **Given** a frozen schedule, **when** the officer logs in, **then** it is listed with period, total and a download in their layout.
- AC-2: **Given** the officer uploads a remittance file, **when** it is parsed, **then** line count, total and any unknown identifiers are shown before submission.

**US-M15-02** As an HR director, I want proof that we remitted everything we deducted, so that we are protected from liability.

- AC-1: **Given** a period, **when** the employer statement is opened, **then** it shows deducted per the schedule, remitted per the bank credit, and any difference, with line detail.
- AC-2: **Given** all lines matched for a period, **when** the statement is downloaded, **then** it carries a "fully remitted" confirmation with the society's reconciliation reference.

## M16. Estates and escrow (release 2)

### Requirements

| ID | Requirement | Priority |
| --- | --- | --- |
| FR-M16-01 | Estate project with budget (land, survey, legal, consent, infrastructure, construction, fees, finance, contingency ≥ 10%, margin), project account and subscriber list | Must (R2) |
| FR-M16-02 | Land due-diligence record: registry search, charting status, litigation search, vendor authority, lawyer sign-off; land marked ineligible if under acquisition, "excision in progress" or unregularised | Must (R2) |
| FR-M16-03 | Collection gates: substantive instalments blocked until master title perfected or consent lodged and each subscriber's agreement signed | Must (R2) |
| FR-M16-04 | Developer and contractor records: registration (REDAN, LASRERA), performance bond, contract type, retention, milestone schedule | Must (R2) |
| FR-M16-05 | Milestone certification by named QS or engineer; release token generated only when the evidence rule passes | Must (R2) |
| FR-M16-06 | Allocation by the approved method (ballot, seniority, payment order) with witnessed record; allocation letters | Must (R2) |
| FR-M16-07 | Title tracking per allottee to registered deed | Should (R2) |
| FR-M16-08 | Carrying-cost calendar: ground rent, Land Use Charge, consent fees with alerts | Must (R2) |
| FR-M16-09 | Quarterly subscriber report: collections, spend vs budget, milestones, title and approval status, arrears | Must (R2) |

### User stories

**US-M16-01** As an estate committee, we want the system to stop us paying for land without due diligence, so that we never repeat the Idu-Sabo loss.

- AC-1: **Given** a land purchase payment, **when** it is initiated, **then** it is blocked unless a complete due-diligence record with lawyer sign-off is attached.
- AC-2: **Given** land marked "excision in progress", **when** a collection schedule is created, **then** substantive instalments are blocked and only refundable expressions of interest are allowed.

**US-M16-02** As a subscriber, I want to know where my money is and how the estate is progressing, so that I can hold the committee to account.

- AC-1: **Given** a quarter ends, **when** reports run, **then** each subscriber receives collections, spend against budget, milestone status, title status and their own position.

**US-M16-03** As a signatory, I want construction payments to need a certified milestone, so that we pay only for work done.

- AC-1: **Given** a construction payment, **when** initiated, **then** a milestone certificate by a registered QS or engineer must be attached and the release token generated before approval is possible.

## M17. Platform administration, audit and security

### Requirements

| ID | Requirement | Priority |
| --- | --- | --- |
| FR-M17-01 | Society audit log of every user action, system job, configuration change and administrative action, viewable by the committee, supervisory committee and auditor | Must |
| FR-M17-02 | Platform administrators cannot initiate, approve or sign payments, or write to ledgers outside business flows (SR-3) | Must |
| FR-M17-03 | Administrative changes (adapter deploys, bank connection changes, key rotation) require two administrators and a 24-hour delay, and appear in affected societies' logs | Must |
| FR-M17-04 | Support access to a society is time-bound, consented by the society and masked for personal data | Must |
| FR-M17-05 | Adapter management: versioned employer and bank adapters with test fixtures and rollback | Must |
| FR-M17-06 | Data export: a society can export all its data in open formats at any time | Must |
| FR-M17-07 | Incident management and notification to societies within 24 hours of a material incident | Must |

### User stories

**US-M17-01** As a committee, we want to see every action the platform's own staff take on our society, so that the platform cannot become the insider threat.

- AC-1: **Given** any administrative action affecting the society, **when** it is executed, **then** it appears in the society's audit log with both administrators' identities and the delay.
- AC-2: **Given** a support request, **when** support accesses the society, **then** access requires the society's consent, expires after the agreed time and is logged.

**US-M17-02** As a society, we want to leave the platform with all our data, so that we are never locked in.

- AC-1: **Given** an export request, **when** it completes, **then** members, ledgers, journals, schedules, remittances, payments and documents are delivered in CSV, JSON and PDF with a manifest that ties to the trial balance.

## Non-functional requirements

| ID | Area | Requirement | Measure |
| --- | --- | --- | --- |
| NFR-01 | Availability | Member channels (USSD, app) and webhook ingestion | 99.5% monthly; webhook ingestion 99.9% |
| NFR-02 | Performance | Statement or balance view | < 3 s at p95 for a society of 10,000 members with 30 years of history |
| NFR-03 | Performance | Reconciliation run | 10,000 lines matched and posted in < 5 minutes |
| NFR-04 | Performance | Instruction to bank after final approval | < 60 s |
| NFR-05 | Durability | Ledger and artefacts | Multi-region replicated; restore tested quarterly; RPO ≤ 15 min, RTO ≤ 4 h |
| NFR-06 | Retention | Journals, schedules, remittances, payments, consents | ≥ 10 years |
| NFR-07 | Integrity | Journal hash chain | Verified daily; any break raises a critical incident |
| NFR-08 | Security | Authentication | Passkey or OTP for approvals; device binding for signatories; MFA for all staff |
| NFR-09 | Security | Keys | Instruction-signing keys in HSM or cloud KMS; rotation ≤ 12 months |
| NFR-10 | Security | Insider threat | Penetration test annually including insider scenarios (admin, treasurer, signatory collusion of one) |
| NFR-11 | Security | Data at rest and in transit | AES-256 at rest; TLS 1.2+ in transit; personal data field-level encrypted |
| NFR-12 | Privacy | NDPA 2023 and GAID 2025 | Registered at the applicable level; annual compliance audit return; DPIA before release |
| NFR-13 | Regulatory | CBN payments framework | Platform never holds funds; bank collaboration approved before go-live |
| NFR-14 | Data residency | CBN circular (Jun 2026) | Personal and transaction data stored in Nigeria by 1 Jan 2027 |
| NFR-15 | USSD | Session design | Every journey completes within one 180-second session; ≤ 5 menu levels |
| NFR-16 | Accessibility | Web and app | WCAG 2.1 AA |
| NFR-17 | Scalability | Platform | 500 societies, 1 million members, 50 employers' adapters without architectural change |
| NFR-18 | Observability | Jobs and integrations | Every scheduled job and webhook has a success metric and alert; dashboards per bank and employer adapter |
| NFR-19 | Idempotency | All external inputs | Webhooks, statements, files and API retries processed exactly once |
| NFR-20 | Portability | Society data | Full export available at any time in open formats (FR-M17-06) |

## Data, interfaces, assumptions and open issues

### Core data entities

| Entity | Key fields | Notes |
| --- | --- | --- |
| Society | id, registration no., authority, CVN, bye-law parameter set (versioned), status | Parameters change only by resolution |
| Member | id, BVN/NIN, phone, employer, payroll id, grade, confirmation status, nominees, status | Duplicate checks on BVN, NIN, payroll id |
| Consent | member, type, scope, method, timestamp, hash, revocation | Referenced by every expected line |
| Deduction profile | member, gross pay, statutory deductions, third-party deductions, source, date | Drives headroom |
| Account (ledger) | society, code, type, member (if sub-ledger) | Chart seeded for cooperatives |
| Journal | id, society, period, lines, source event, actor, prev hash, hash | Append-only |
| Loan | member, product, amount, rate, method, tenor, schedule, guarantors, status, arrears cause | Employer-caused vs member-caused |
| Employer | id, payroll platform, id format, cut-off, pay day, contacts, virtual account, adapter version | One virtual account per society |
| Schedule version | employer, period, lines, total, hash, delivery log | Immutable |
| Remittance | employer, period, bank credits, file (raw), parsed lines | Raw file kept |
| Match result | line, classification, posting journal, variance case | One classification per line |
| Variance case | type, amount, member, employer, owner, due, status, resolution | Visible to auditor |
| Employer receivable | employer, amount, age bucket, actions taken | Ages from pay date |
| Payment | type, evidence, beneficiary, amount, initiator, approvals, instruction, bank ref, journal | Posts on confirmation |
| Meeting / resolution | type, notice, quorum, votes, linked actions | Authorises parameter and signatory changes |
| Estate project (R2) | budget, title record, gates, milestones, subscribers, allocations | Release tokens |

### External interfaces

| Interface | Direction | R1 | R2 |
| --- | --- | --- | --- |
| Partner bank | Both | Transfers, status, statements, webhooks, virtual accounts, name enquiry | Mandate verification, release tokens, second bank |
| Employer payroll | Both | File exchange (generic + two pilot layouts), console upload | IPPIS, Remita, state layouts; HRIS APIs where available |
| Identity | Out | BVN/NIN verification via aggregator | Employer HR verification |
| SMS and USSD | Out | Aggregator with short code | Additional languages |
| Direct debit | Both | — | NIBSS NDD via bank or PSSP |
| Registry and bureau | Out | — | CVN/CoopID, CoopCHECK when published |
| Title verification | In | — | LandVerify or manual e-GIS adapter |

### Assumptions

1. A partner bank agrees to virtual accounts, transfer and statement APIs and webhooks for society accounts in R1; bank-enforced mandate verification may arrive in R2 (until then, in-platform approvals with a scoped execution credential, limits and daily reconciliation).
2. Pilot employers (oil and gas, banks, parastatals) will exchange schedule and remittance files in a stable layout.
3. Societies accept that no credit posts without a schedule.
4. The one-third deduction cap is a configurable agreement term pending verification of the Labour Act text.
5. Members have mobile phones; at least half will use USSD rather than the app.

### Dependencies

Partner-bank agreement and CBN collaboration approval; SMS and USSD aggregator; BVN/NIN aggregator; legal templates (tripartite deduction agreement, subscription agreement, platform terms); NDPA registration.

### Open issues

| # | Issue | Owner | Needed by |
| --- | --- | --- | --- |
| 1 | Which bank enforces the mandate and how (option 1–4 in the bank tab) | Partnerships | R1 design freeze |
| 2 | IPPIS onboarding, schedule template and remittance advice | Partnerships | R2 |
| 3 | Pricing model: per member per month vs per deduction line vs employer licence | Product | R1 pilot |
| 4 | Whether credits may post provisionally against a sent schedule in any circumstance | Product + pilot committees | R1 |
| 5 | Director of Cooperatives return templates per state | Product | R2 |
| 6 | Treatment of interest on float and its pass-through to members | Finance + bank | R1 |
| 7 | Registry (CVN/CoopID) API availability | Partnerships | R2 |
