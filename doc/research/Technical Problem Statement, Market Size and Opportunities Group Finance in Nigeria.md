# Technical Problem Statement, Market Size and Opportunities: Group Finance in Nigeria

Oct 8, 2026 · @Abubakar

## Executive summary

Nigerian group finance has no system of record and control. Fifty-one million adults save in informal groups, 20–30 million belong to cooperatives, and staff cooperatives hold up to ₦66bn each, yet the money sits in individuals' accounts, the ledgers are notebooks and spreadsheets, payments leave on one signature, payroll deductions vanish for years, land is bought without a title search, and nothing is reported to members or regulators. Every documented loss in the problem statement, from a market ajo to the ₦806m NNPC cooperative freeze and the ₦700m NLNG land fraud, is a failure of custody, ledger, payment control, payroll reconciliation, identity, asset verification or reporting.

The revenue pool for solving this is roughly ₦100–130bn a year (about $70–100m), of which ₦35–45bn is reachable through payroll, registered cooperatives and digital ajo, and ₦4–6bn a year is a realistic five-year capture for one platform, plus float on custodied funds. No Nigerian competitor covers the full set: bank products hold money without ledgers, SaaS ledgers hold no money, consumer apps do not pool at all, and nobody reconciles payroll deductions or escrows estate payments. Kenya has the whole stack and shows the ceiling.

The recommended opportunity is a staff-cooperative operating system: partner-bank custody in each society's name, an append-only member ledger, dual-control payments, payroll deduction reconciliation with employer-arrears ageing, and automated statements and returns, priced at about ₦200 per member per month and sold to committees and employers who now carry fiduciary liability (Oredo, Jan 2025). Estate escrow, investment-club NAV accounting, digital ajo and a regulator data layer extend it in later phases. The platform should never hold money on its own balance sheet; licensing as a PSSP under a CBN-approved bank collaboration is the route, with an MFB licence deferred to phase 3.

## Technical statement of the problem

The business failures catalogued in the problem statement reduce to eight missing system capabilities. No platform in Nigeria today provides them together for groups, and most groups run on paper, WhatsApp and a single person's bank account.

### 1. Custody: pooled money sits in an individual's account

An ajo circle, an investment club or an unregistered cooperative has no account in its own name, so funds pass through the collector's, treasurer's or chairman's personal account. The system problem is the absence of a group-owned, licensed, segregated custody account with a mandate enforced by the bank rather than by trust. Every collector-absconding, treasurer-fraud and "changed signatories" case is a custody failure.

### 2. Ledger: no authoritative record of who owns what

Member balances, loans, shares and allocations are kept in exercise books, spreadsheets or the collector's memory. Records are lost (the Police cooperative cannot pay pre-2019 balances), disputed (NNPC members cannot see accounts since 2022) or forged. The system problem is the lack of an append-only, double-entry ledger per group with member-level sub-ledgers, unit or NAV accounting for clubs, capital and current accounts for ventures, and statements members can verify independently.

### 3. Payment control: one approver, no gates

Money leaves the group on one signature, often before the thing it pays for exists. The system problem is that approval workflows (two-of-three signatures, spending limits, general-meeting thresholds, milestone certification, escrow release conditions) are not enforced by the payment rail; they are policies on paper that the payer can ignore. Land vendors were paid ₦700m (NLNG) and ₦406m (Apo-Tafyi) because nothing stood between the committee and the bank.

### 4. Payroll integration: deductions are a black box

Staff cooperatives depend on employer payroll deducting and remitting. The system problem is that there is no machine-readable deduction schedule going out, no itemised remittance file coming back, no automatic line-by-line reconciliation, no cross-lender view of a payslip against the one-third cap, and no ageing of employer arrears. Non-remittance of 17 months to three years is discovered by members, not by the system.

### 5. Identity and eligibility: membership is not verified

Who is a member, who is confirmed staff, who has consented to deductions, who is a guarantor and who is the nominee are not verifiable against an authoritative source. The system problem is the lack of BVN/NIN-anchored member identity, signed and stored consent, employer HR status checks and a guarantor registry, so deductions are taken from people who never agreed and loans are stacked across lenders invisibly.

### 6. Asset verification: land and ventures are bought blind

Cooperatives pay for land without a registry search, charting or vendor-authority check, and fund ventures nobody inspects. The system problem is that title verification, layout-approval status, developer registration and milestone evidence are manual, offline and unrecorded, so the gate "title before money" cannot be enforced by software.

### 7. Reporting and compliance: nothing is published or filed on time

Societies do not produce NPL, liquidity or dividend-on-savings figures; AGMs lapse; returns to the Director and CAC are missed; members receive no annual statement. The system problem is the absence of automated financial statements, prudential ratios, member statements, regulatory returns and compliance calendars generated from the ledger.

### 8. Sector data: regulators and lenders are blind

No default rates, no society count, no exposure to employer arrears, no transaction-price index. The system problem is that no aggregation layer exists: group-level data never reaches a registry (the national CoopID registry is only now being built), a credit bureau (CoopCheck does not exist) or a supervisor, so policy, lending and membership decisions are made without evidence.

### The problem in one sentence

Nigerian group finance lacks a trustworthy system of record and control — custody, ledger, gated payments, payroll reconciliation, verified identity, verified assets, automated reporting and shared data — and until one exists, every group is as safe as the honesty and competence of the one person holding the money.

### Non-functional dimensions of the problem

- **Trust under adversarial insiders**: the threat model is the administrator, not an outside hacker; controls must bind the people who run the system.
- **Low connectivity and literacy**: members are market traders, drivers and junior civil servants on feature phones; USSD and agent channels matter as much as apps.
- **Regulatory perimeter**: holding money requires a CBN-licensed partner or licence; investment-club accounting may touch the SEC perimeter; cooperative supervision is state-level and offline.
- **Data quality**: source registries (land, cooperatives, payroll) are incomplete, inconsistent and often offline; the system must record provenance and uncertainty, not assume clean inputs.
- **Longevity**: estates take 3–5 years and loans 30 years; records must outlive committees, vendors and the platform's own versions.

## Target users and jobs to be done

| Segment | Who | Job to be done | Today's workaround | What fails |
| --- | --- | --- | --- | --- |
| Informal savings circles (ajo, esusu, adashe) | 5–50 traders, artisans, drivers; a collector | Save daily or monthly, receive the pot in turn, get credit without collateral | Cash to a collector; WhatsApp; a notebook | Collector absconds; no record; zero or negative return |
| Investment clubs | 10–50 professionals, alumni, church or diaspora groups | Pool monthly, buy T-bills, land or a business, share returns fairly across joiners | Treasurer's personal account; spreadsheet | No unit accounting; treasurer control; unclear SEC status |
| Registered community cooperatives | 20–500 members, market or trade based | Thrift and credit, bulk purchase, land | Exercise books; one bank account; manual AGM | No audit; officers overstay; records lost |
| Staff cooperatives | 300–9,000 employees of one employer | Save by payroll, borrow at 2x–5x, buy land or goods on instalment | Payroll deduction file, Excel ledger, committee approvals | Unremitted deductions; frozen withdrawals; no NPL or liquidity view; estate illiquidity |
| Cooperative housing estate committees | A society and 50–1,000 subscribers | Buy titled land, build, allocate, title each member | Committee pays vendor directly; allocation letters | Money before title; no escrow; 27–38% delivery |
| Employers (HR and payroll) | Federal, state, LG, parastatals, private firms | Deduct lawfully, remit on time, stay within the one-third cap, avoid liability | IPPIS/Remita codes; manual schedules | Fiduciary liability; strikes; errors |
| Lenders and FMBN | Banks, MFBs, FMBN, Shelter Afrique | Lend to groups with verified ledgers and milestone control | Paper AGM reports; site visits | No standard data; approval without disbursement |
| Regulators | State Directors of Cooperatives, SEC, CBN, FCCPC | Know what exists, supervise, detect Ponzis early | Paper returns; recertification drives | 19,786 registered vs 370,000 claimed; detection after collapse |

The primary paying customers are staff cooperatives (budget, payroll, scale) and employers (liability); the largest user base is informal circles, who pay little but are the on-ramp; regulators and lenders are the data customers.

## Market size

Bottom-up, the platform revenue pool for Nigerian group finance is roughly ₦100–130bn a year (TAM), of which about ₦35–45bn is reachable through payroll, registered cooperatives and digital ajo channels (SAM), and ₦4–6bn a year is a realistic five-year capture for one platform (SOM), before float income on custodied funds. Every cooperative headline figure in Nigeria is an unaudited claim, so the ranges below carry the inputs that drive them.

### Demand-side inputs (latest, dated)

| Input | Value | Year / source type |
| --- | --- | --- |
| Adults | 111m (2023); \~119m implied by the 2026 release | EFInA A2F |
| Adults using an informal financial provider | 46% ≈ 51m (33% in 2020) | EFInA 2023 |
| Informal-only adults | 10.7m (10%) | EFInA 2023 |
| Cooperative societies | 370,000 (Minister, Jul 2025) vs 30,000 (same minister, Jan 2026) | Unaudited |
| Cooperative members | 20m (CFN 2023) vs 30m+ (CFAN 2023) | Unaudited |
| Cooperative savings / loans | ₦1.8trn / ₦1.3trn | CRASON claim, Nov 2024 |
| Lagos societies / funds | \~16,000 / \~₦80bn | State, 2022 |
| Wage employees | 12.1m (14.4% of 84.15m employed); 25.6% public | NBS Q2 2024 |
| Federal staff on IPPIS | \~720,000 | OAGF 2025–26 |
| Teachers / police / military | 2.09m / 372k / 223k | 2020–22 |
| Pension RSA holders / assets | 11.39m / ₦31.8trn | PenCom Aug 2026 |
| Bank consumer credit | ₦3.78trn (−19.9%) | CBN 2025 |
| Deducted-but-unremitted payroll loans | ₦12bn | Lenders' claim, Dec 2024 |
| NHF collections / contributors | ₦152.4bn (2025, +48%) / \~6m | FMBN |
| FMBN project and cooperative loans | ₦79bn, 6,911 units | FMBN 2025 |
| PiggyVest | 6m+ users; ₦1.3trn paid out in 2025 | Company claim |
| Bankly / LibertyPay / Alajo | 2m users; 150k ajo cards, \~₦5bn a month; ₦373m saved | Company claims 2023–25 |
| Kenya regulated SACCOs (benchmark) | 357 SACCOs, 7.9m members, KES 1.21trn assets | SASRA 2025 |

### Pricing benchmarks

| Model | Observed price | Source |
| --- | --- | --- |
| Cooperative SaaS per member | ₦200 per member per month (SmartCoopHub); ₦2,000 per member per year (SmartTeller) | Nigerian vendors, 2025–26 |
| Cooperative SaaS per society | ₦300–500k a year (Prolean); from ₦15k a month (Coopco) | Nigerian vendors |
| Chama / SACCO SaaS (Kenya) | KES 600–1,000 per group per month + KES 30 per member (Chamasoft); from KES 30k a month unlimited (Kwara, 230+ SACCOs) | Vendor pages |
| ROSCA organiser commission | One round (3–8% of contributions); India chit foreman 5% (cap 7%) | Research notes |
| Fund-manager take | 0.5% of AUM (Risevest) | Company |

### TAM: the whole revenue pool

Assumptions are stated per line; sources and conflicts are in the market notes.

| Revenue stream | Arithmetic | Annual pool |
| --- | --- | --- |
| Member SaaS on cooperative members | 20m members (lower claim) × ₦2,400 a year | ₦48bn |
| Transaction take on informal group flows | 51m adults × ₦10,000 a month (anecdotal) × 12 = ₦6.1trn × 0.5% | ₦30bn |
| Loan origination and servicing fee | ₦1.3trn cooperative loan stock, \~₦650bn originated a year × 1.5% | ₦10bn |
| Payroll deduction rail | 5m deduction lines a month × ₦50 × 12 | ₦3bn |
| Estate escrow and milestone control | ₦300bn of cooperative housing collections a year (FMBN ₦79bn plus member equity, estimate) × 1% | ₦3bn |
| Float spread on custodied funds | ₦500bn average balances × 3% net of partner share | ₦15bn (excluded from TAM total, shown separately) |
| **TAM (fees)** |  | **₦94bn, range ₦100–130bn on the higher membership claim** |

In dollars at ₦1,331 per US$ (8 Oct 2026): roughly $70–100m a year in fees, plus $10–20m float.

### SAM: what a platform can reach in five years

| Segment | Reachable members | Basis |
| --- | --- | --- |
| Staff cooperatives, public sector | \~3.4m (IPPIS 720k, teachers 2.09m, police 372k, military 223k) | Payroll-integrable |
| Staff cooperatives, private formal | \~2m of \~9m private wage employees | Employers with payroll software |
| Active registered community cooperatives | \~3m (20m members × \~15% demonstrably active, the Lagos revalidation ratio) | Societies that file accounts |
| Digital-ajo and group-savings users | \~2.5m (Bankly 2m, LibertyPay, Alajo, app group features) | Already digitised |
| **SAM members** | **\~10–11m** |  |
| SAM fees | 10.5m × ₦2,400 SaaS + transaction and loan take on their flows (\~₦1.2trn) at 0.75% | **₦34–44bn a year** |

### SOM: a defensible five-year share

| Year | Members | Fees | Basis |
| --- | --- | --- | --- |
| 1 | 50,000 | ₦0.2bn | 5–10 staff cooperatives of 3,000–9,000 members plus pilot circles |
| 3 | 400,000 | ₦1.6bn | 60–80 societies; two payroll platforms integrated |
| 5 | 1,000,000 (\~10% of SAM) | ₦4–6bn (\~$3–4.5m) plus ₦2–3bn float | 150+ societies, one state registry, FMBN estate pipeline |

### Reading the numbers

- The largest, most certain pool is **staff cooperatives**: identifiable employers, existing deductions, balance sheets of ₦0.4–66bn each, and a legal liability (Oredo) that makes employers pay for reconciliation.
- The largest **user base** is informal circles (51m adults), but it monetises at kobo per transaction and needs agents; it is the on-ramp, not the revenue.
- **Housing escrow** is small in fees but is where the catastrophic losses are (₦700m NLNG, ₦406m Apo-Tafyi); it is a trust feature that sells the platform to cooperatives, not a profit centre.
- **Float** can exceed fees: 15.9% one-year T-bills (Oct 2026) against 2–3% paid to savers leaves a spread, but it belongs to the licensed custodian and must be shared or passed to members to avoid repeating the negative-real-return problem.
- Kenya shows the ceiling: 7.9m regulated SACCO members and KES 1.21trn (\~₦12trn) of assets in a country with a quarter of Nigeria's population.

### Confidence

Low on absolute size (cooperative counts differ twelve-fold between two statements by the same minister), medium on the staff-cooperative base (payroll headcounts are official), medium on pricing (vendor pages). Treat TAM as an order of magnitude and SAM/SOM as planning ranges to be replaced by pilot data.

## Competitive landscape

No Nigerian player covers the full problem set. Custody, ledger, payroll reconciliation, escrow and title verification each exist somewhere as a separate product; nobody combines them, and the consumer apps with scale do not pool money at all.

### Who does what

| Capability | Who offers it today | Scale and price | What is missing |
| --- | --- | --- | --- |
| Pooled custody in the group's name | Wema CoopHub (bank account with initiator, verifier, approver roles); AjoCard and Rank (ex-Moni) MFBs; Sterling and Wema ajo schemes | CoopHub scale and price undisclosed; AjoCard licensed MFB | Only two bank products; no ledger, loans or estates attached |
| Cooperative ledger SaaS (non-custodial) | Coopify, Coopco, CoopSolve, Coopra, Esusu Africa, SmartCoopHub, SmartTeller, Prolean | ₦15k–₦90k a month per society; ₦200 per member per month | Money stays in the treasurer's bank; no payment control, no payroll reconciliation, no regulator returns |
| Consumer group savings | PiggyVest Group Target and Joint Accounts (Sep 2026, rules unpublished); Cowrywise Challenges and Collections; Kuda, Moniepoint, OPay, PalmPay have no group product | PiggyVest 6m+ users, ₦1.3trn paid out 2025 | Co-saving in individual accounts; no pot, no credit, no committee governance |
| Ajo digitisation | AjoCard (MFB), LibertyPay (150k cards), Thrifto, TrustAjo; Bankly absorbed by C-One (Apr 2025); Alajo, AjoMoney, Prolean apparently dormant | \~₦5bn a month projected (LibertyPay claim) | "Guaranteed payouts" marketed without disclosed custody or licence; high mortality |
| Payroll deduction rails | Remita third-party deductions; IPPIS deduction codes; Lendsqr, Configure, Indicina for lenders | Fees and file formats unpublished | IPPIS collapses all lenders into one payslip line; duplicate and ghost deductions documented (Jan 2024); no reconciliation tool exists anywhere |
| Payroll lenders | Credit Direct (₦150bn cumulative, 1.5m customers), FairMoney (₦150bn in 2025), Renmoney, Page, CREDICORP (₦37bn to 200k civil servants) | Monthly rates 2–4% | Compete with cooperatives for the same payslip; worsen the one-third-cap problem |
| Escrow | PayScrow (2% + ₦100), Vesicash (2.5%); Propcrowdy (SEC portal with custodian bank and milestone release) | Transaction-level | No CBN escrow licence category; no cooperative-estate product; developers (RevolutionPlus \~₦787m into own accounts) bypass it |
| Title verification | LandVerify API, PlotSight; Lagos e-GIS search (no API); FCT AGIS portal | Per-search fees | Not wired into any payment flow |
| Real-estate pooling | Keble (30k users), Risevest (600k users, 0.5% AUM; its cooperative was declared unlicensed by SEC in Jan 2025 before a fund-manager licence in Feb 2026), Cowa (unverifiable) | Fractional ownership | Investor products, not cooperative estate delivery |
| Regulator and registry | Seamfix national registry (MoU 12 Aug 2026: CVN, CoopID, CoopCHECK; six-month revalidation to \~Feb 2027); Cooperative Bank of Nigeria (₦200bn share drive from end-Oct 2026, CBN category unannounced) | Fees to societies unpublished | No API yet; no licensing regime for cooperative software |

### Kenya, the benchmark

Kenya shows what the stack looks like when it exists: SASRA publishes sector data and an employer-arrears league table (KSh3.92bn, 2025); the Cooperative Societies Act s.35 lets a SACCO attach an employer's bank account for unremitted check-off; Co-operative Bank runs a SACCO switch for 110 deposit-taking SACCOs; core-banking vendors (Kwara from KES 30k a month, Craft Silicon, Fintech Group, Coretec) all ship check-off reconciliation and SASRA returns; Chamasoft and ChamaPay (KES 500–5,000 a month) auto-match M-Pesa contributions. Two warnings travel with it: KUSCCO lost KSh13.3bn through unregulated apex custody, and no deposit insurance is live.

### White space

1. Pooled custody, society ledger and dual-control payments in one product.
2. Deduction-schedule generation and line-by-line remittance reconciliation for IPPIS, Remita and state payrolls, with employer-arrears ageing.
3. Developer-payment escrow for cooperative estates with release tied to title, bills of quantities and member approval.
4. Title verification wired into payment release rather than sold as a separate search.
5. Member statements and automated returns to the Director, the national registry (CVN) and the Cooperative Bank.
6. Governance tooling for staff cooperatives (terms, AGM, audit trail, related-party flags) — live pain since the NNPC Lagos freeze.
7. A ROSCA product with priced rotation risk (guarantors for early positions, default reserve) instead of "guaranteed payouts".

### Open items needing direct enquiry

IPPIS deduction-code onboarding process and file formats; Remita payroll fee schedule; Seamfix registry fee model and API timeline; Wema CoopHub pricing and society count; whether Cowa exists as a licensed entity.

## Enablers and constraints

The rails to build on exist (BVN, NIN, NIBSS, direct debit, bank pool accounts, state land portals); the licence to hold cooperative money does not, and the registry, credit bureau and payroll interfaces the platform needs are either unpublished or still being built.

### Licensing routes

| Route | Capital and fees | What it allows | Fit |
| --- | --- | --- | --- |
| Partner bank or MFB holds pooled funds; platform runs sub-ledgers, licensed as PSSP or unlicensed software | PSSP ₦100m capital; every PSP–bank collaboration needs CBN prior approval (payments framework, 9 Dec 2020) | Custody in the group's name at the bank, platform controls workflow and records | **Recommended start**: no deposit-taking risk, fastest to market |
| Own Microfinance Bank | Tier 2 unit ₦50m, Tier 1 ₦200m, state ₦1bn, national ₦5bn (2020 guidelines); application ₦100–300k, licence ₦250k–1m | Hold deposits, lend, NDIC cover to ₦2m per depositor | Later phase; converts cooperatives at the 2011 threshold (\~2,000 members or ₦20m, unverified) into a regulated home |
| Mobile Money Operator or Payment Service Bank | ₦2bn / ₦5bn | Wallets and float | Over-sized for this problem |
| SEC fund manager | ₦150m today; portfolio managers to ₦2bn–₦5bn by 30 Jun 2027 (circular 16 Jan 2026) | Pooled investing for clubs | Only if the platform takes discretion over members' investments; pure bookkeeping is outside the perimeter |
| SEC crowdfunding portal | ₦100m | Public raises for MSMEs | Not for cooperatives (on-lending excluded) |

Registered cooperative societies are expressly outside the CBN's "illegal deposit-taker" definition (FSRCC advisory, 4 Apr 2022), so a society may hold member savings itself; the platform must not.

### Identity and KYC

- BVN 70.35m (Sep 2026); NIN about 110m; both mandatory on all wallet tiers since 1 Mar 2024. Aggregator lookups cost roughly $0.04–0.08; NIMC's own pricing is tiered and unpublished.
- Tier 1 wallet limits: ₦300k balance, ₦50k daily — enough for ajo members, not for cooperative treasurers.
- CAC company search is free; extracts ₦1–15k; Mono lookups ₦10–50. **No API verifies cooperative registration**; the CoopID/CVN registry (Seamfix MoU, 12 Aug 2026; revalidation to about Feb 2027) has no published API, fee or operator.

### Payment rails and costs

- NIBSS Instant Payments: 11bn transactions in 2024; ₦3.75 per transfer to the bank since 1 Jul 2023; end-user caps ₦10/25/50 plus VAT.
- NIBSS direct debit: ₦30 per debit plus ₦50 mandate (Flutterwave ₦100); fintechs act as aggregators. This is the rail for members outside payroll.
- Collections via Paystack 1.5% + ₦100 capped at ₦2,000; Flutterwave 1.4–2% capped at ₦2,000; payouts ₦10/25/50.
- USSD ₦6.98 per session (from 18 Jun 2025); agent banking rules (6 Oct 2025) impose exclusivity from 1 Apr 2026 and ₦100k daily customer limits.
- Global Standing Instruction (loan recovery across banks) is open only to CBN-licensed creditors, so a cooperative cannot use it unless it lends through a licensed partner.
- Open banking was announced for 1 Aug 2025 and is not live as of mid-2026; Okra shut in May 2025. Account-data access is bilateral for now.

### Payroll check-off

- IPPIS remits cooperative and loan deductions but publishes no onboarding procedure, deduction-code process, file format or fee; the one-third rule is applied inconsistently; IPPIS collapses all lenders into one payslip line, so a member cannot see individual deductions. OAGF moved MDA cash access from Remita to TMRAS in 2025.
- Remita runs payroll-loan rails (USSD *347*780#) with unpublished fees; state payrolls vary; private payroll software (SeamlessHR, PaidHR, Bento) exposes generic APIs with no standard third-party check-off interface.
- Consequence: payroll integration will be built employer by employer, with file-based exchange first and APIs where they exist. This is the hardest integration and the highest-value one.

### Land and housing data

- Lagos e-GIS (Jan 2024): search ₦3,750, certified copy ₦5,625; consent about 1.5% of value. FCT AGIS: ₦10–20k in person, 1–3 days. No API or bulk access anywhere; LandVerify and PlotSight wrap manual searches.
- No national construction-cost index (NIQS/QSRBN database is still a proposal, Mar 2026); REDAN membership ₦350k in year one, no verification API.
- FMBN's cooperative loan disburses to the society or developer with no member-level escrow and no API.

### Data protection and tax

- Nigeria Data Protection Act 2023 and GAID 2025 (effective 19 Sep 2025): fintechs are Ultra-High-Level processors (₦250k registration), MFBs Extra-High (₦100k), annual compliance audit return, ₦5k per processor.
- Nigeria Tax Act 2025 (from 1 Jan 2026): VAT 7.5% on platform fees; ₦50 stamp duty on electronic transfers of ₦10k or more, paid by the sender, with instruments executed by cooperatives exempt; s.163 exempts cooperatives' non-trade profits; small-company 0% CIT (threshold reported as ₦50m by one source and ₦100m by others; see the problem statement).

### Policy tailwinds

- The 2025–2030 cooperative reform plan, the national registry and CoopID, the planned CoopCHECK credit bureau and the ₦200bn Cooperative Bank (share drive from end-Oct 2026, CBN licence status unreported) all create demand for exactly the data a platform produces, and a government buyer for registry and returns integration.
- The Oredo judgment (Jan 2025) gives employers a financial reason to pay for remittance reconciliation.

### Hard constraints

1. No licence category fits a cooperative-services platform; custody must sit with a bank or MFB partner under CBN-approved collaboration.
2. No deposit insurance covers cooperative savings; the platform cannot promise safety it does not have.
3. No registry, credit-bureau, payroll or land API exists today; the first versions are file and manual-search based.
4. Several primary texts (2011 CBN cooperative policy, NIMC and NIBSS per-lookup pricing, Remita fees, state registration fees) could not be verified online and must be confirmed directly.

## Business opportunities in Nigeria

Ranked by the product of market certainty, willingness to pay and absence of competition. The first three form one product for staff cooperatives; the rest extend it.

| Rank | Opportunity | Target segment | Product | Revenue model | Size (annual, 5-year reach) | Entry barrier | Main risk |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 1 | **Staff cooperative core: custody, ledger, loans, governance** | 5m public and private staff-cooperative members; 150–300 societies of 1,000–9,000 members | Bank-held pooled account with dual control, member ledger, loan lifecycle, surplus appropriation, AGM and audit trail, member statements over app, USSD and SMS | ₦150–250 per member per month SaaS, paid by the society; set-up fee per society | ₦2–4bn fees from 1m members; the largest certain pool | Winning the first ten societies; partner bank approval | Committee resistance to transparency; society-level sales cycle |
| 2 | **Payroll check-off reconciliation and arrears ageing** | The same societies plus their employers (federal MDAs on IPPIS, states, parastatals, oil and gas, banks, universities) | Deduction-schedule generation, remittance file matching, variance queue, one-third-cap headroom, employer arrears ledger and escalation letters | Per deduction line (₦20–50) or bundled in rank 1; employer-side licence for HR teams | ₦1–2bn; sold on the Oredo liability | No published IPPIS or Remita interface; employer-by-employer onboarding | Payroll platforms refuse data; political states ignore arrears |
| 3 | **Estate escrow and milestone control** | Cooperative housing committees, FMBN-financed schemes, developers wanting credibility | Project account, title-verification record, layout-approval status, BOQ milestone certification, member-vote release, subscriber statements | 0.5–1% of funds controlled; developer certification fee | ₦1–3bn on ₦100–300bn of collections; trust feature that sells rank 1 | No escrow licence category; must run through partner-bank trust accounts | Developers route around it; title data remains manual |
| 4 | **Digital ajo with priced risk** | 51m informal users via agents and collectors | Collector-led circles with licensed custody, rotation ledger, guarantors for early positions, default reserve, USSD-first | ₦20–50 per contribution; collector commission share; float share with partner | ₦3–8bn at 5–10m users; the user-acquisition engine | Agent network; unit economics at ₦10k contributions | High churn; collector incumbency; past failures (Alajo, AjoMoney dormant) |
| 5 | **Investment-club accounting with NAV units** | 10–50-member professional, alumni, church and diaspora clubs | Unit issuance at NAV, contribution and distribution tracking, treasurer dual control, T-bill and fund integrations via licensed partners | ₦5–15k per club per month; partner revenue share on instruments | ₦0.5–1bn | SEC perimeter clarity (no discretion, no public solicitation) | SEC reclassification (Risevest precedent) |
| 6 | **Regulator and registry data layer** | State Directors of Cooperatives, the national registry (CVN/CoopID), CoopCHECK, Cooperative Bank, FMBN | Automated annual returns, prudential ratios, revalidation data, sector dashboards, arrears league table | Government contract or per-society return fee; data licence to lenders | ₦0.5–2bn; policy-driven | Procurement; registry API not yet published | Political timelines; registry operator captures the layer |
| 7 | **Employer-paid welfare and credit rails** | Employers with staff cooperatives | Cap enforcement across lenders, consented deduction registry, salary-linked emergency credit through the cooperative at cooperative rates instead of 3–4% monthly fintech loans | Per-employee per month HR fee | ₦1–2bn | HR buy-in | Competes with payroll lenders' distribution |
| 8 | **Cooperative-to-MFB conversion service** | Societies above the 2011 threshold (\~2,000 members or ₦20m) | Licence application, core system, NDIC cover, CBN reporting | Project fee plus ongoing SaaS | Dozens of societies; high ticket | CBN process; capital ₦50m–₦200m per society | Regulatory delay |

### Why staff cooperatives first

- The buyer is identifiable (committee plus employer), funded (balance sheets of ₦0.4–66bn) and under pressure (NNPC Lagos freeze, Police withdrawals, Oredo liability).
- Credit risk is near zero on payroll (0.157% NPL at Total), so the platform's value is control and reconciliation, which software does well.
- One society of 5,000 members at ₦200 a month is ₦12m a year; fifty such societies fund the company.
- Each society brings an employer, and each employer brings every other cooperative on its payroll.

### Pricing and unit economics (planning view)

- Member SaaS ₦200 a month; partner-bank custody cost and NIP fees under ₦30 per member per month at expected transaction counts; KYC ₦60–120 one-off; USSD ₦7 per session passed through or capped.
- Gross margin above 60% once a society exceeds 1,000 members; payback on sales cost within 12 months at a ₦500k set-up fee.
- Float: with partner consent, a share of the spread between T-bills (15.9%) and member savings rates (2–3%) can fund member-facing features, but the design intent should be to pass most of it to members to fix the negative-real-return problem, not to replicate it.

### Opportunities to avoid

- Holding member money on the platform's own balance sheet without a licence.
- "Guaranteed payout" ajo products.
- Taking discretion over club investments without SEC registration.
- Lending the platform's own money into cooperatives (becomes a payroll lender, a different business).

## Acceptance criteria

The solution is accepted when every criterion below passes against the stated measure. Criteria are grouped by the eight capabilities in the technical problem statement; each is testable without judgement calls.

### AC1. Custody

- **Given** a group is onboarded, **when** its account is created, **then** the account is held at a CBN-licensed institution in the group's name (or a segregated sub-ledger of a licensed partner's pooled account), and no individual member can withdraw from it outside the approval workflow.
- **Given** any payment instruction, **when** it is submitted, **then** it executes only after the configured approvals (minimum two of three signatories), and the bank-side mandate rejects anything else. *Measure:* 100% of payments in the audit log carry the required approvals; zero single-approver executions.
- **Given** a signatory is changed, **when** the change is requested, **then** it requires the general-meeting threshold, a cooling-off period of at least 48 hours, and notification to every member.
- **Given** member funds and project funds, **when** balances are reported, **then** thrift savings and estate or venture project funds are in separate accounts or sub-ledgers that reconcile independently to the bank.

### AC2. Ledger

- **Given** any transaction, **when** it is posted, **then** it is recorded as a double-entry journal in an append-only log with a hash chain; corrections are new entries, never edits. *Measure:* ledger replays to the same balances from day one.
- **Given** a member, **when** they open their statement, **then** they see savings, shares, loans, interest, dividends, allocations and drawings to the current day, with every line traceable to a journal id.
- **Given** an investment club, **when** a member joins mid-period, **then** units are issued at the NAV of the valuation date and all later distributions are per unit. *Measure:* NAV test cases (ICLUBcentral method) reproduce to the kobo.
- **Given** a venture, **when** the year closes, **then** the system produces capital and current accounts per member, the appropriation statement in the deed's order, and a statement of partners' equity.
- **Given** a cooperative, **when** surplus is appropriated, **then** the system enforces at least 25% to reserve, education up to 10% of the remainder, honoraria at or below the bye-law cap, and blocks any payout before committee approval and AGM confirmation are recorded.
- **Given** the bank statement, **when** the day closes, **then** ledger cash equals bank cash or every difference is an open reconciling item with an owner and an age. *Measure:* unreconciled items older than 7 days are zero or escalated.

### AC3. Payment control and gates

- **Given** a payment to a land vendor, developer or contractor, **when** it is requested, **then** the system requires an attached title-verification record, layout-approval status and (for construction) a certified milestone, and routes funds through escrow or lender-controlled release. *Measure:* zero land or construction payments without the required attachments.
- **Given** a spending limit per signatory, committee and general meeting, **when** a payment exceeds a limit, **then** it escalates to the next tier and cannot be split to evade the limit (same payee, same purpose, 30 days).
- **Given** a related party (officer, their relative, their company), **when** a payment to them is requested, **then** it is flagged, requires general-meeting approval and appears in the annual report.
- **Given** an estate project, **when** collections start, **then** substantive instalments are blocked until the master title is perfected or lodged for consent and every subscriber has an executed subscription agreement on file.

### AC4. Payroll integration and remittance

- **Given** a cooperative's member schedule, **when** the payroll cut-off arrives, **then** the system generates the deduction file in the employer's format (IPPIS, Remita, state payroll, private payroll) and records what was sent.
- **Given** a remittance, **when** it is received with its itemised schedule, **then** the system reconciles line by line within one business day, posts matched lines, and lists every variance (missing member, wrong amount, duplicate, non-member).
- **Given** the one-third cap, **when** a loan is approved, **then** total deductions on the member's payslip after the loan, across all known lenders, stay within one-third of gross pay; the system records the headroom calculation.
- **Given** an employer's remittance is late, **when** 30, 60 and 90 days pass, **then** the system issues the demand letter, escalates to the advisory committee and Director, and books a provision; new loans funded from unreceived remittances are blocked. *Measure:* employer arrears ageing is visible to every member.

### AC5. Identity, consent and eligibility

- **Given** a new member, **when** they are onboarded, **then** identity is verified against BVN or NIN, employment status against the employer's HR source where available, and the nominee is recorded.
- **Given** a deduction authority, **when** it is captured, **then** it is a signed, timestamped, member-specific consent stored with the deduction schedule and retrievable on demand; no deduction is scheduled without it.
- **Given** a loan application, **when** guarantors are named, **then** the system checks each guarantor's membership, confirmation status and existing surety exposure, and locks the guarantor's exit until the bye-law release condition is met.
- **Given** a member's declaration of outside debts, **when** a credit bureau is available, **then** the declaration is checked against it and discrepancies flagged.

### AC6. Asset verification

- **Given** a parcel of land, **when** it is proposed for purchase, **then** the system holds a due-diligence record with registry search result, Surveyor-General charting status, litigation search, vendor-authority evidence and a lawyer's sign-off; purchase approval is impossible with any field empty.
- **Given** land with "excision in progress", under acquisition or unregularised Area Council papers, **when** it is entered, **then** the system marks it ineligible for member collections.
- **Given** a developer or contractor, **when** they are engaged, **then** REDAN or LASRERA status, performance bond and contract terms (fixed price or escalation clause, retention, milestone schedule) are recorded before the first payment.
- **Given** a venture asset (fleet, farm, shop), **when** it is funded, **then** an inspection schedule with named inspectors exists and missed inspections are visible to members.

### AC7. Reporting and compliance

- **Given** a cooperative, **when** any month closes, **then** the system publishes to members: NPL ratio, liquidity ratio, employer arrears, dividend-on-savings to date, portfolio composition and illiquid-asset share against the cap.
- **Given** a financial year, **when** it closes, **then** audited-ready financial statements, the surplus appropriation, member annual statements and the Director's return are generated within the bye-law dates, and the AGM notice (21 days) is issued automatically.
- **Given** a company or business-name venture, **when** CAC or tax deadlines approach, **then** the compliance calendar alerts the responsible officer 30 and 7 days before, and records filing evidence.
- **Given** an estate project, **when** a quarter closes, **then** subscribers receive collections, spend against budget, milestone, title and approval status, and arrears.
- **Given** a vendor, title, developer or revocation failure, **when** it is recorded, **then** every member and the Director are notified within 30 days.

### AC8. Sector data and interoperability

- **Given** a registered society, **when** the national registry (CoopID/CVN) exposes an interface, **then** the system verifies registration status and syncs the member roll with consent.
- **Given** aggregate data, **when** a regulator or lender requests it, **then** the system can export anonymised sector metrics (default rates, arrears, delivery rates) and a group's audited ledger in a standard format.
- **Given** a credit bureau or the planned CoopCheck, **when** available, **then** member loan performance is reported and queried under consent.

### AC9. Non-functional

- **Adversarial insider**: no role, including platform administrators, can alter posted journals, bypass approvals or change signatories without a recorded, multi-party, time-delayed process; penetration testing covers insider scenarios.
- **Channels**: every member-facing action (balance, statement, approve, consent) works over USSD and SMS as well as app and web; agents can onboard and collect.
- **Availability and durability**: 99.5% monthly availability for member channels; ledger backed up across regions with a tested restore under four hours; records retained at least ten years.
- **Data protection**: NDPA 2023 compliance, consent-based sharing, member right to export their own record.
- **Scale**: a society of 10,000 members with 30 years of history and a daily ajo circle of 50 members both operate within the same response-time targets (under 3 seconds for a statement).
- **Auditability**: an external auditor can reproduce every balance from the journal and bank statements without platform assistance.

## Recommended architecture and phasing

Build a system of record and control that never holds money itself: a partner bank or MFB holds every group's funds in the group's name, and the platform owns identity, ledger, approval workflow, integrations and reporting. Licence as a PSSP (or operate as unlicensed software under the partner's CBN-approved collaboration) and defer any MFB licence to phase 3.

### Capability layers

1. **Channels**: web and app for committees and employers; USSD, SMS and agent app for members; employer HR console.
2. **Identity and consent**: BVN/NIN verification, employer status checks, signed deduction authorities, nominee and guarantor registry, role-based access with multi-party, time-delayed changes for signatories and administrators.
3. **Group and governance**: society, club, circle and estate project models; bye-law and deed parameters (thresholds, limits, terms of office, appropriation rules); meetings, resolutions and minutes; related-party flags.
4. **Ledger**: append-only double-entry journal with hash chaining; member sub-ledgers; unit/NAV accounting; capital and current accounts; appropriation engine; daily bank reconciliation.
5. **Payments and custody integration**: partner-bank pool and sub-accounts, dual-control instruction flow, NIBSS transfers and direct debit, collections gateways, escrow and milestone release, spending limits and anti-splitting.
6. **Payroll integration**: deduction-schedule generators per employer format, remittance ingestion and matching, variance queue, one-third-cap headroom, arrears ageing and escalation.
7. **Asset verification**: due-diligence records for land (registry, charting, litigation, vendor), developer and contractor records, inspection schedules; manual-search adapters first, APIs when they appear.
8. **Reporting and compliance**: statements, prudential ratios, financial statements, Director and CAC returns, AGM pack, compliance calendar, regulator and lender exports.
9. **Data and audit**: event log, external-auditor reproduction tools, anonymised sector metrics, registry (CVN/CoopID) and bureau (CoopCHECK) connectors when published.

### Build, partner or buy

| Component | Decision | Reason |
| --- | --- | --- |
| Custody and float | Partner (bank or MFB; two partners for resilience) | Licence and deposit insurance |
| Ledger, governance, workflow, payroll reconciliation | Build | The differentiator; nothing to buy in Nigeria |
| KYC, payments, USSD, SMS | Buy (aggregators, NIBSS participants, gateways) | Commodity rails with published prices |
| Title and company verification | Partner now (LandVerify, PlotSight, manual e-GIS), build adapters | No APIs yet |
| Core banking (phase 3) | Buy or licence a proven SACCO core (Kenyan vendors) | Faster than building for an MFB licence |

### Phasing

**Phase 1 (months 0–9): staff cooperative pilot.** Partner bank approved; five to ten societies live with custody, ledger, loans, dual control, member statements over app and USSD; deduction files for two employer formats; manual remittance reconciliation with the variance queue. Exit criterion: AC1, AC2, AC4 and AC7 pass in production for at least 20,000 members.

**Phase 2 (months 9–24): payroll and estates.** IPPIS or Remita file integration for federal members and two state payrolls; employer HR console; estate project accounts with escrow release and due-diligence records; FMBN scheme support; investment-club NAV module; first state Director returns. Exit criterion: 400,000 members, two payroll platforms, five estate projects under escrow, AC3, AC5, AC6 passing.

**Phase 3 (months 24–60): scale and licence.** Digital ajo through agents; registry (CVN/CoopID) and CoopCHECK integration; sector data products for regulators and lenders; MFB licence or acquisition so large societies can convert with NDIC cover; Cooperative Bank interface. Exit criterion: one million members, AC8 and AC9 passing, independent audit of the ledger.

### Architecture principles

- The administrator is the threat model: no single role can move money, alter history or change signatories.
- Record provenance on every external fact (title search, employer status, remittance) with its source, date and confidence.
- Design for file-based integration first; every external interface is an adapter so APIs can replace files without touching the ledger.
- Member channels work without data: USSD and SMS carry balances, approvals and consents.
- Ten-year retention and auditor-reproducible balances are release requirements, not later features.

## Risks, assumptions and open questions

### Risks

| Risk | Likelihood | Impact | Mitigation |
| --- | --- | --- | --- |
| CBN treats the platform as unlicensed deposit-taking | Medium | Fatal | Partner-bank custody under approved collaboration; no platform balance sheet; legal opinion before launch |
| Partner bank fails or withdraws | Low–medium | High | Two custody partners; portable ledger; member funds always in the group's name |
| SEC reclassifies club or cooperative pooling as a collective investment scheme | Medium | High | No discretion, no public solicitation, no fixed-return promises; engage SEC early (Risevest precedent) |
| Payroll platforms refuse or delay integration | High | High | File-based exchange first; employer-side console gives HR a reason to cooperate; start with private and parastatal payrolls |
| Committees resist transparency and dual control | High | Medium | Sell to general meetings and employers, not only committees; make the Oredo liability and NNPC freeze the pitch |
| Insider or administrator fraud on the platform | Medium | Fatal to trust | Multi-party time-delayed controls, hash-chained ledger, external audit, insider penetration tests |
| Low willingness to pay in informal circles | High | Medium | Monetise societies and employers first; ajo as acquisition |
| Registry operator (Seamfix) or Cooperative Bank captures the data layer | Medium | Medium | Integrate early; position as the operating system societies use, not the registry |
| Macro: inflation, devaluation, fee caps | High | Medium | Naira pricing indexed; low fixed costs; float shared with members |
| Data protection breach of member identity data | Medium | High | NDPA/GAID compliance, encryption, minimal retention of raw identity documents |

### Assumptions to validate in the pilot

1. Societies will pay ₦150–250 per member per month, or employers will pay for them.
2. A partner bank will open sub-accounts in each society's name with platform-enforced dual control and approve the collaboration with CBN.
3. At least two employers will exchange deduction and remittance files in a machine-readable form within six months.
4. Members will use USSD for balances and approvals at ₦7 a session.
5. Committees will accept published NPL, liquidity and arrears figures.
6. The 2011 CBN conversion threshold (₦20m assets or 2,000 members) is still policy; the pilot societies are far above it and are not challenged.

### Open questions

- IPPIS deduction-code onboarding procedure, file format and fees.
- Remita and TMRAS fee schedules for third-party deductions.
- Seamfix registry API, fee model and timeline; CoopCHECK operator.
- Whether the Cooperative Bank will be licensed as a commercial, merchant or national microfinance bank, and whether it will offer custody to societies.
- Which small-company tax threshold applies (₦50m or ₦100m) and whether platform fees to cooperatives attract VAT exemptions.
- Wema CoopHub pricing and reach, the nearest existing competitor.

## Sources

This document synthesises the research notes below, each of which carries dated, URL-level citations and flags every unverified or conflicting figure. Where a figure here is a company claim or a ministerial statement, it is labelled as such in the tables above.

- *Problem Statement: Pooled Capital, Staff Cooperatives and Cooperative Housing in Nigeria* (8 Oct 2026) and the three research streams behind it: `research_notes/Pooled capital business structures/`, `research_notes/Staff cooperatives in Nigeria/`, `research_notes/Cooperative housing estates Nigeria/`.
- Market inputs: `research_notes/Group finance market Nigeria/market_size_inputs.md` — EFInA A2F 2023 and 2026 release, NBS labour force Q2 2024, PenCom (Aug 2026), CBN 2025 annual report, FMBN 2025 statements, OAGF/IPPIS, SASRA 2025, company disclosures (PiggyVest, Bankly, LibertyPay, Alajo, Risevest, Keble), vendor pricing pages (SmartCoopHub, SmartTeller, Prolean, Coopco, Chamasoft, Kwara), CBN and NIBSS statistics, Partech and Disrupt Africa funding data.
- Competitive landscape: `research_notes/Group finance market Nigeria/competitive_landscape.md` — 60-row comparison across five segments, Kenya comparators, white-space analysis.
- Enablers and constraints: `research_notes/Group finance market Nigeria/enablers_and_constraints.md` — CBN 2020 MFB guidelines and payments licensing framework, FSRCC advisory (4 Apr 2022), SEC rules and the 16 Jan 2026 capital circular, ISA 2025, NIMC and NIBSS pricing, Lagos e-GIS and FCT AGIS fees, GAID 2025, Nigeria Tax Act 2025, RH-CRRP 2025–2030 and the Seamfix MoU (12 Aug 2026).

Primary sources that could not be fetched directly and should be confirmed by hand: CBN circular PDFs (including the 2011 microfinance policy), NIMC and NIBSS per-lookup price lists, Remita and TMRAS fee schedules, IPPIS deduction-code procedures, state cooperative fee schedules, the EFInA A2F 2026 full deck.
