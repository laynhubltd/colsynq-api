# Competitive landscape: group-finance software and services in Nigeria (Kenya as comparator)

**Compiled:** 8 Oct 2026. **Method:** five parallel desk-research passes (~130 web searches, ~325 page fetches) over company sites, pricing pages, app stores, TechCabal/Techpoint/Technext/Nairametrics/BusinessDay, FIJ, SASRA, SEC/CBN/FCCPC public lists, Kenyan press. Every claim carries a source and date in brackets. Labels: **[unverified]** = single or indirect source, could not corroborate; **[snippet]** = seen only in a search-engine excerpt of a page that would not load; **[self-reported]** = company marketing figure. Where a search budget ran out before a name could be checked, that name is listed under *Not researched* rather than guessed.

**Problem set used to score players** (from the brief): (1) custody of pooled funds, (2) auditable ledger, (3) dual-control / multi-signatory payments, (4) payroll check-off reconciliation, (5) escrow for land/developer payments, (6) title verification, (7) member statements, (8) regulator returns, plus loans.

---

## 1. Headline findings

1. **Nobody in Nigeria covers the full problem set.** The pieces exist separately — bank custody (Wema CoopHub), member ledgers (Coopify, Coopco, CoopSolve, Esusu Africa), SEC-custodied milestone disbursement (Propcrowdy), fintech escrow (PayScrow, Vesicash), title checks (LandVerify, Sytemap, Lagos e-GIS) — but no product stitches custody + ledger + dual control + check-off reconciliation + developer escrow + title verification + member statements + returns.
2. **Consumer fintechs do not pool money.** PiggyVest "Group Target" and Cowrywise "Challenges" are co-saving in individual accounts; Kuda, Moniepoint, OPay and PalmPay have no group/ajo product at all. Only Cowrywise "Collections", PiggyVest Joint Accounts (launched 26 Sep 2026, rules unpublished) and the unverifiable Cowa pool funds.
3. **Cooperative SaaS is small, undocumented and cheap** (₦15k–₦90k/month tiers; Esusu Africa ₦25k/month to ₦10m lifetime; Orbital ₦650k–₦1.8m one-off [snippet]). None publishes client counts beyond "500+ cooperatives" [self-reported]; none documents IPPIS/Remita schedule import, regulator returns, or dual-control payments except Wema CoopHub (Initiator/Verifier/Approver).
4. **Payroll check-off is a broken rail.** IPPIS aggregates all lender deductions into one opaque "WACS" payslip line (FIJ, Oct 2023); double/ghost deductions documented (FIJ, Feb 2024); tertiary institutions moved to Excel-based GIFMIS payroll (Nov 2024); ASUU reports months of unremitted cooperative deductions (Oct 2026); Kano found ₦1.8bn of arbitrary vendor deductions and barred vendors (May 2026). **No reconciliation tool for cooperatives exists** and IPPIS/Remita schedule formats are not published.
5. **The federal state is moving in.** RH-CRRP 2030 (Mar 2026), the Seamfix national cooperative registry (CVN/CoopID/CoopCHECK; MoU 12 Aug 2026; six-month revalidation to ~Feb 2027) and a Cooperative Bank of Nigeria (₦200bn share drive, subscription from end-Oct 2026) will define identity, credit-bureau and possibly custody rails — but none is a society-level ledger or payments product.
6. **Regulatory precedent against the "cooperative wrapper":** SEC publicly declared Risevest Cooperative Multipurpose Society unlicensed (Jan 2025) before Risevest obtained a fund-manager licence (Feb 2026). Any pooled-investing model styled as a cooperative faces the same risk.
7. **Housing failures are developer-side, not cooperative-side:** subscribers paid developers' own accounts with no escrow (RevolutionPlus ~₦787m, Adron 30–40% refund haircuts, 176 illegal Lagos estates Aug 2025). FMBN's cooperative loan requires an *unencumbered* title up front and disburses to the cooperative/developer with no member-level escrow.
8. **Kenya is 10+ years ahead institutionally** (SASRA, statutory check-off with employer-account attachment, Co-op Bank SACCO switch, Sacco Central liquidity co-op, SACCO Amendment Bill 2025 with digital returns) and in software (every SACCO core has a check-off reconciliation module; sub-KSh 5,000/month chama SaaS auto-matches M-Pesa STK pushes). Yet even Kenya has no live deposit insurance, no chama software that holds funds, and a 4-year member-redress backlog — so the custody + dual-control + escrow combination is white space there too.

---

## 2. Comparison table

Legend: **Y** documented; **P** partial/indirect; **N** not offered / not documented; **?** unknown. Custody column: who holds pooled money.

| Player | Segment | Custody of pooled funds | Ledger / member statements | Dual control | Loans | Payroll check-off | Escrow | Title verif. | Regulator returns | Pricing / take | Scale (date) | Funding | Licence | Status / notable gap |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| **PiggyVest** | Retail savers, SMEs, kids | N (co-saving; Joint Accounts Sep 2026 rules unpublished) | Individual only | N | N | N | N | N | N | Yield spread; 1–3.5% early-break fees | 6m+ users; ₦1.3trn paid out 2025 | $1.1m seed 2018 | SEC fund mgr (PV Capital); CBN MFB + MMO (Pocket) | Repeated withdrawal outages 2022–Oct 2026 |
| **Cowrywise** | Young professionals | P (Collections pool; mechanics undocumented) | Individual | ? | N | N | N | N | N | 0% naira plans; 1.5% on MF buys; fund TER 1.5–2.5% | ~2m registered / ~500k active [snippet] | $3m pre-A Jan 2021 | SEC fund mgr + digital sub-broker | Bot-only support, Trustpilot 2.4 (Jun 2026) |
| **Risevest** | USD retail investors | N (community = Telegram) | Individual | N | N | N | N | N | N | 0.5–1% AUM p.a.; 1.4–3.9% deposit fees | ~600–700k users (2024) | Undisclosed VC; acquired Chaka, Hisa | SEC fund mgr Feb 2026; US BD | SEC "unlicensed" warning Jan 2025 (cooperative society) |
| **Bamboo / Chaka / Trove** | Retail stock investors | N | Individual | N | N | N | N | N | N | 1–1.5% commission + FX spread | Bamboo 500k–2.3m [conflicting]; Chaka 100k; Trove 400–500k | Bamboo $17.5m; Chaka $1.5m; Trove $3.3m | SEC digital sub-broker / broker-dealer | SEC ban Apr 2021; no club product |
| **Cowa** | Church/family/coop groups | Y (via ARM/Meristem) [unverified] | ? | ? | N | N | N | N | N | 0.5% group mgmt fee [unverified] | ? | ? | "SEC-registered" [unverified; no register entry found] | Site does not resolve; treat as unproven |
| **Kuda / Moniepoint / OPay / PalmPay** | Mass-market | N — no group product | Individual | N | Overdrafts/working capital | N | N | N | N | Zero-fee accounts, float, lending | Kuda 7m; OPay 50m; PalmPay 35m; Moniepoint 14bn txns 2025 | Kuda $91m; OPay $400m; PalmPay $40m; Moniepoint >$200m Series C | National MFB / MMO (Dec 2025–Jan 2026) | CBN onboarding ban Apr–Jun 2024 (KYC) |
| **AjoCard** | Market traders via agents | Y (own MFB, NDIC) [self-reported] | Agent-side | N | Y (≤₦100k, 30 days) | N | N | N | N | Loan interest, float | Not published since 2021 | Seed Jul 2021 (Beta.Ventures) | CBN MFB (tier unpublished) | Thin capital; no metrics |
| **Bankly** | Thrift collectors, agents | Y (MFB) | P | N | Y | N | N | N | N | Agent fees, loans | 2m users, 50k agents (Apr 2025) | $2m seed | CBN MFB (+PSSP per UNDP) | Absorbed by C-One Ventures 29 Apr 2025 — dormant brand |
| **LibertyPay** | Ajo collectors | ? (custody bank undisclosed) | Card-based | N | Y (4–13%/month) | Y via Remita payroll loans ("Seeds Cooperative") | N | N | N | Loan interest, agent commission | 150k ajo cards planned Q4 2023 | None found | CBN super-agent AIP (Jan 2023) | In-app lotto; slow support |
| **Alajo App** | Unbanked, USSD | "Escrowed" (bank undisclosed) | Operator app | N | N | N | N | N | N | Subscriptions + collection fees | ₦373m saved (Apr 2023) | ~$400k | ? | Site non-functional Oct 2026 — possibly dormant |
| **AjoMoney** | Consumer ROSCAs | ? | App | N | N | N | N | N | N | ? | ? | ? | "Unlicensed" per TrustFinance | Likely dormant |
| **Thrifto** | Consumer ROSCAs | ? ("guaranteed payouts", no bank named) | Y real-time ledger | N | N | N | N | N | N | Undisclosed | Launched 13 Mar 2026 | Undisclosed | None disclosed | Custody risk flag |
| **Esusu Africa** | B2B: thrift collectors, coops, MFIs | N (client's bank) | Y (accounts, reports, portal) | N | Y | N | N | N | P (financial reports) | ₦25k/month → ₦10m lifetime (7 tiers) | 180+ institutions; 500k customers; ₦100bn saved [self-reported, undated] | Gates Foundation grant | None (software) | No payroll import, no returns |
| **Sterling OneBank Ajo** | Retail | Y (bank) | Bank statements | N | N | N | N | N | N | Free | ? | — | Commercial bank | Bank does not guarantee defaults |
| **Wema ALAT Goals / CoopHub** | Retail / cooperatives | Y (Wema accounts) | Y (contributions, loans, analytics) | **Y** Initiator/Verifier/Approver | Y | N | N | N | N | Undisclosed | Undisclosed | — | Commercial bank | Only promo coverage; no onboarding numbers |
| **Coopify / ProbityCoop** | Cooperatives | N | Y (contributions, loans, guarantors, portal, reports) | N | Y | N | N | N | N | Free ≤10; per-member/month; one-off >500 | "500+ cooperatives" [self-reported] | — | None | Payments via 3rd Quadrant |
| **Coopco** | Cooperatives | N | Y ("13 books", audit trail) | N | Y | N | N | N | P | ₦15k/₦30k/₦90k per month | ? | — | None | — |
| **Coopra** | Staff cooperatives | N | Y incl. deduction schedules | N | Y | **P** (staff deduction schedules) | N | N | N | Not retrievable | ? | — | None | Only vendor naming deduction schedules |
| **CoopSolve** | Staff cooperatives (ATBU, NNPC-PH, Aidlife) | N | Y | N | Y (instant loans) | P (auto-deductions) | N | N | N | Undisclosed | Aidlife 5,000+ members | — | None | Branded apps per coop |
| **Orbital CMS** | Cooperatives (Abuja) | N | Y (TB, BS, P&L, bank rec) | N | Y | P (payroll admin) | N | N | N | ₦650k / ₦1.8m [snippet] | ? | — | None | Housing-project module |
| **SmartCoopHub / SureThrift / BankCore** | Cooperatives / MFBs | N | Y | N | Y | N | N | N | N | BankCore ₦10k–₦1m/month + ₦100k setup | "thousands" [self-reported] | — | None | — |
| **BankOne (Qore) / Oradian / Musoni / Mambu** | MFB/MFI core banking | N (client) | Y | P | Y | N | N | N | P (CBN returns for MFBs) | Enterprise licence (undisclosed) | BankOne 500+ FIs; Oradian used by Pearl Funds MPCS | — | None | Not cooperative-specific; "rudimentary" loan origination (Lendsqr on BankOne) |
| **CFAN** (ex-NACCUN) | Apex credit unions | N | Planned Unified Co-operative Platform | — | Dev. Fund (₦5bn/yr target) | N | N | N | N | — | 5.2m members, 2,040 groups | — | Apex body | UCP vendor/cost unknown |
| **NCDAP / Seamfix registry** | All cooperatives | N | Identity only (CVN, CoopID, CoopCHECK bureau) | — | N | N | N | N | **P** (national register) | "No cost to governments" — user fees unknown | 30k–370k societies [conflicting] | PPP | Federal | Does not replace state registration; no ledger |
| **Cooperative Bank of Nigeria** | Cooperatives | Planned | — | — | Planned | — | — | — | — | ₦21–50m shares per large society | ₦200bn target, 10k societies | Subscription from end-Oct 2026 | CBN category unannounced | Plan only |
| **Remita (SystemSpecs)** | Lenders, FG payroll | N (rail) | Per-txn settlement T+1 | — | Data referencing for lenders | **Y** (salary debit; >2m salaries/month 2019) | N | N | N | ~₦50/successful DD; payroll fees unpublished | — | — | CBN PSSP | Lender onboarding undocumented |
| **IPPIS / WACS** | Federal civil servants | — | Single aggregated payslip line | — | — | **Y** (opaque) | — | — | — | Deduction-code fee unpublished | ~600k employees [unverified] | — | OAGF | Double/ghost deductions; Excel payroll for tertiary (Nov 2024) |
| **Credit Direct / Page / Zedvance / Renmoney / FairMoney / Carbon / Specta** | Salaried + civil servants | N | Borrower-level | — | Y | Y (IPPIS/Remita/CREDICORP; Zedvance integrates Remita, NIBSS, Mono) | N | N | N | 2.1–3.95% flat/month; 1% mgmt fee | Credit Direct 2m+; Renmoney 3m+; FairMoney ₦150bn disbursed 2025 | FairMoney ~$57m [unverified] | CBN finance co. / MFB | Duplicate IPPIS deductions (Credit Direct, Jan 2024) |
| **Lendsqr / Configure / Indicina / Mono** | Lender SaaS & data | N | Loan ledger | N | Y | N (no schedule import) | N | N | N | Lendsqr $0–$500/month; Mono DD 1% capped ₦1,000 | Lendsqr 3.7m borrowers | — | None | No cooperative module |
| **Keble** | Fractional property | Trustee (Meristem) | Individual | N | N | N | N | Deed of assignment | N | Undisclosed | 30k users, 70 properties (Sep 2025) | Techstars, ARM | SCUML only — no SEC licence found | Fee opacity, no liquidity |
| **Propcrowdy** | Retail → SME developers | **Y** custodian bank, milestone release | Investor-level | P (SEC-approved listings) | N | N | **Y** (milestone) | Title collateralised | SEC returns | Undisclosed | 20k investors [snippet] | $500k (2021) | SEC crowdfunding portal (Dec 2022) | ₦100m raise cap; custodian unnamed |
| **MonieWorx** | Retail → SME notes | SEC portal | Investor-level | — | — | — | — | — | SEC | Undisclosed | ₦5.23bn raised, 10k investors | — | SEC crowdfunding portal (Dec 2022) | Not real-estate specific |
| **Estate Intel** | B2B data | — | — | — | — | — | — | — | — | Subscription | — | $600k pre-seed | — | Data only |
| **FMBN e-NHF / CHDLS** | NHF contributors, coops | FMBN disburses to coop/developer | Member NHF balance (app) | BOQ-based tranche | Y (₦500m @10%, 24m) | NHF via payroll | N | Requires unencumbered title first | — | Dev fees ₦600k accred.; ₦160k legal search | ~40 coop projects / 2,570 units (2021); 6,911 units 2025 | FG | Statutory | No member-level escrow; no API |
| **LagosHOMS** | Lagos residents ₦50–250k income | State | — | — | Rent-to-own 5% down | — | — | — | — | ₦5k + ₦10k fees | 20k families since 2014 | State | Statutory | Portal only |
| **Lagos e-GIS** | Title holders | — | — | — | — | — | — | **Y** search/C of O/consent online | — | Statutory fees | >60% titles digitised [snippet] | State | — | No public API; TLS issues |
| **PayScrow / Vesicash / Escrow Africa / Escrownet** | Marketplace & property escrow | Y (partner DMB accounts) | Txn-level | Milestone release | N | N | **Y** | Escrownet: AGIS check | N | 1.5–2.5% (+₦100); Vesicash 2.5%/5% | Undisclosed | Vesicash: Ingressive | None of their own | No cooperative/estate workflow |
| **Bank / SEC trustees** (FirstBank, ARM, UTL) | Large deals | Y | — | Y | — | — | Y | — | SEC | Bespoke | — | — | SEC trustee | Not productised for coops |
| **Gate Africa / Residence / GatePass** | Estate dues | N (wallets) | Dues statements, approvals | P | N | N | N | N | N | Subscription | Gate Africa 10k users; GatePass 709 homes | — | None | No equity ledger / developer escrow |
| **LandVerify / PlotSight / Sytemap** | Title checks | N | — | — | — | — | — | **Y** (₦5k–₦49.5k; LandVerify API) | — | Per search | Sytemap 30k users, ₦5bn land | Sytemap $400k | SCUML only | Sytemap T&Cs allow 40% refund fee |
| **KENYA — Chamasoft / EazzyChama** | Chamas; bank white-label | N (group bank a/c) | **Y** auto statements, invoicing | Bank mandate | N | N | N | N | N | Free; KSh 1,000/month ≤20 members | 50k users, 10k active | Bootstrapped [unverified] | None | No custody |
| **ChamaPay / MyChama** | Chamas | N (paybill) | Y auto-match STK push, PDF statements | Role-based | Y with guarantors | N | N | N | N | KSh 500–5,000/month | 500+ chamas | — | None ("not a bank") | — |
| **Safaricom Ziidi / Mali** | 41m M-Pesa users | CMA custodian | Individual | N | N | N | N | N | CMA | Free in/out | Ziidi 2m customers, KSh 18.7bn (Mar 2026) | Safaricom | CMA CIS | Fund-manager dispute (Genghis); no chama tooling |
| **Kwara** | SACCO core + neobank | N (SACCO) | Y + USSD + guarantor flow | P | Y | **Y** (SASRA-compliant) | N | N | **Y** SASRA | Fixed annual subscription | 230+ SACCOs [self-reported] | $7m seed | None | Exclusive KUSCCO channel compromised by KUSCCO collapse |
| **Craft Silicon / Fintech Group / Coretec (Dynamics BC)** | SACCO core | N | Y | Y | Y | **Y** check-off modules | N | N | **Y** | Quote-based | Craft 200+ FIs; Fintech 500+; Coretec 37 BC + 29 CoopMIS | — | None | — |
| **Co-op Bank Kenya Sacco Link** | DT-SACCOs | Y (bank) | — | Y | — | — | — | — | — | Switch fees | 619 FOSA branches; 110 SACCOs ATM | — | CBK | Movement-owned clearing bank — no Nigerian equivalent |
| **NACHU** | Housing co-ops | Y (Cap 490) | Member savings | — | Y housing microloans | N | N | Land purchase for members | — | Donor-dependent | ~2,000 units; 355k households claimed | Donor | Co-op (not SASRA) | Discontinued 400-unit project Sep 2020 |
| **KUSCCO** (apex) | 3,000 SACCOs | Y — unregulated | — | N | Y | — | — | Housing fund | **None** | — | KSh 17.7bn liabilities | — | None (gap) | KSh 13.3bn loss; liquidation Sep 2026 |

---

## 3. Per-player findings — Nigeria consumer / group savings & investing

### 3.1 PiggyVest (Piggytech / PV Capital / Pocket)
- **Segment:** mass-market savers; SMEs (2024); kids and families (Sep 2026). [Technext, 26 Sep 2026: https://technext24.com/news/piggyvest-unveils-kids-joint-business-accounts/]
- **Group features:** *Group Target Savings* — creator sets goal, members contribute from their **own** NDIC-insured accounts; shared progress, private balances; up to 12% p.a. **No pooled custody, no admin control, no group ledger, no dual control.** [PiggyVest blog, 29 May 2026: https://blog.piggyvest.com/save/piggyvest-target-group-savings/] *Joint Accounts* launched 26 Sep 2026 at OpenHouse — first shared-balance product; approval rules unpublished [unverified]. Pocket "Split" bill-splitting. No loans, payroll, escrow, statements for pooled funds, returns.
- **Pricing:** yield spread; Target early-break 1% + interest forfeit; Piggybank off-date withdrawal 3.5%; Flex Dollar deposit 0.3%. Rates 12–18.5%. [FAQ: https://www.piggyvest.com/faq]
- **Scale:** 6m+ users; ₦1.3trn paid out 2025 (+56%); AUM +110% 2025 (absolute undisclosed). [Techpoint, Jan 2026: https://techpoint.africa/news/piggyvest-2025-payouts/]
- **Funding:** $1.1m seed 2018 (used to buy MFB licence); no later priced round disclosed. [TechCabal, 7 Oct 2026, snippet: https://techcabal.com/2026/10/07/piggyvest-10-years/]
- **Licences:** SEC Fund/Portfolio Manager (PV Capital, RC 1760152); CBN MMO (Pocket, AIP Apr 2022); CBN MFB. [FAQ]
- **Failures:** withdrawal outages 15 Aug 2022 [The Whistler], 7 May 2025 and 8 Sep 2025 [Technext], and transaction delays after Sep 2026 redesign blamed on NIBSS [Technext, c. 1–2 Oct 2026: https://technext24.com/news/piggyvest-transaction-delays-nibss-downtime/]. No CBN/SEC sanctions found.

### 3.2 Cowrywise (CFTL)
- **Group features:** *Circles* (Feb 2019) — **Challenges** (individual goals, shared progress) and **Collections** ("pool funds for a shared interest"); circles up to 6,342 members. Custody, admin roles, withdrawal authorisation and ledger mechanics for Collections **not documented anywhere reachable** [unverified]. [https://cowrywise.com/blog/cowrywise-circles-group-savings/ (7 Feb 2019); https://www.cowrywise.com/circles]
- **Pricing:** 0% on naira plans; 1.5% processing on mutual-fund buys (caps ₦2,000/₦250); fund TER 1.5–2.5%; Stash withdrawal ₦25–100. [help.cowrywise.com]
- **Scale:** ~2m registered, ~500k active, 11 staff [TechCabal, 30 May 2026, snippet]. **Funding:** $3m pre-Series A, Jan 2021 (Quona). [TechCabal, 28 Jan 2021]
- **Licences:** SEC Fund/Portfolio Manager (#1940); SEC Digital Sub-Broker (#3005). [T&Cs]
- **Complaints:** bot-only support, compliance holds on withdrawals; Trustpilot 2.4 (Jun 2026). [brands.ng]

### 3.3 Risevest (RV Fund Management)
- **Group features:** "investment club" = Telegram community, not pooled investing. [https://www.risevest.com/investment-club]
- **Pricing:** 0.5% p.a. ≥$1,000 / 1% below; deposit 1.4–3.9%; "Managed by Rise" US property 1% procurement + fee on returns or 3% procurement-only. [Glider blog; Estate Intel]
- **Scale:** ~600k users (Sep 2024). **M&A:** Chaka (Sep 2023), Hisa Kenya (Sep 2024). [Techpoint]
- **Licences / actions:** SEC named *Risevest Cooperative Multipurpose Society Ltd* and Risevest Technologies unlicensed, Jan 2025; SEC Fund & Portfolio Manager licence 19 Feb 2026 (₦150m min capital); US broker-dealer Feb 2025. [BusinessDay / Technext, 19 Feb 2026: https://technext24.com/2026/02/19/risevest-secures-sec-licence/] **Key precedent** for any cooperative-wrapped pooled investing.

### 3.4 Bamboo / Chaka / Trove
- No investment-club or pooled products. Bamboo "Pulse" is a social feed. Fees 1–1.5% + FX spread. Bamboo $17.5m raised ($15m Series A, Jan 2022, Tiger Global); claims 500k–2.3m users [conflicting]. Chaka first SEC digital sub-broker (Mar 2021), acquired by Risevest. Trove 400–500k users, own broker-dealer Innova Securities; $3.3m raised. All three hit by SEC ban Apr 2021. [BusinessDay, 19 Feb 2026: https://businessday.ng/technology/article/top-fintech-apps-powering-nigerias-investment-boom/; https://investbamboo.com/about-us; https://blog.trovefinance.com/is-trove-sec-licensed-in-nigeria/ (3 Jun 2026)]
- Complaints: Bamboo withdrawals debited-not-credited (May 2026 Play reviews) [brands.ng].

### 3.5 Cowa / Cowa Africa
- Only source is a listicle: digital esusu with ARM/Meristem fund managers, groups ≤50, ₦500 minimum, **0.5% group management fee**, 10–15% expected, "SEC-registered". [Nairatechie, 23 May 2026: https://nairatechie.com/best-investment-apps-in-nigeria/] Official site does not resolve; no SEC register entry, founder, users, funding found. **All claims [unverified]**; "SEC-registered" may refer to partner fund managers.

### 3.6 Kuda, Moniepoint, OPay, PalmPay
- **No group/ajo/joint features** in any public documentation. Savings are individual (Kuda Spend+Save/Fixed to 20%; Moniepoint Flexible/Locked/Target Oct 2025; OPay OWealth; PalmPay Cashbox/Smart Earn to 22%). [kuda.com/en-ng/save; Arbiterz Sep 2026: https://arbiterz.com/nigeria-fintech-savings-rates-opay-palmpay-kuda-moniepoint]
- Scale: Kuda 7m customers; OPay 50m+; PalmPay 35m; Moniepoint 14bn txns/₦412trn 2025. Funding: Kuda $91m to Series B ($500m val.); OPay $400m SoftBank 2021; Moniepoint $110m Series C Oct 2024 + ~$90m ext. (>$1bn val.); PalmPay $40m seed 2019. [Innovation Village, 14 May 2026: https://innovation-village.com/nigerias-fintech-six-...]
- Licences upgraded to **national** MFB/MMO Dec 2025–Jan 2026. **Action:** CBN onboarding ban 29 Apr–Jun 2024 (KYC/forex). [Arise TV, 29 Apr 2024]

---

## 4. Per-player findings — Ajo/esusu digitisation

### 4.1 AjoCard — agent-led daily savings for market traders; 30-day loans ≤₦100k; "licensed by CBN", lists an MD-MFB; NDIC-insured claim [self-reported]. Seed Jul 2021 (Beta.Ventures), no later rounds. No scale metrics since 2021. No ledger/returns/dual-control features. [https://ajocard.com/about-us; https://www.cbinsights.com/company/ajocard/financials]

### 4.2 Bankly — 2m users, 50k agents; CBN MFB (+PSSP per UNDP); $2m seed (date conflicting 2021 vs Mar 2024). **Acquired outright by C-One Ventures 29 Apr 2025**; team absorbed, CEO to advisory — treat as dormant independent brand. [Nairametrics, 29 Apr 2025: https://nairametrics.com/2025/04/29/c-one-ventures-acquires-nigerian-fintech-bankly/]

### 4.3 LibertyPay (Liberty Assured) — ajo cards, >1,000 POS, USSD; micro-credit ₦10k–₦300k at 4–13%/month (APR 48–156%); CBN super-agent AIP Jan 2023; 150k cards planned Q4 2023, ₦5bn/month projected [unverified achieved]. Also Remita payroll loans "powered by Seeds Cooperative" (LSCS 19744). App Store 4.1 (19 ratings), in-app lotto → 18+. [BusinessDay, 16 Sep 2023; https://www.libertyng.com/]

### 4.4 Alajo App — USSD/IVR "escrowed digital piggy bank" via PoS agents; subscription + collection fees; ₦373m saved by Apr 2023; ~$400k raised; escrow bank undisclosed. **Site non-functional Oct 2026 — possibly dormant** [unverified]. [Disrupt Africa, 18 Apr 2023]

### 4.5 AjoMoney — consumer ROSCA app (2021); TrustFinance "unlicensed", no recent activity; **likely dormant** [unverified]. [https://www.trustfinance.com/companies/ajo-money]

### 4.6 Thrifto — launched 13 Mar 2026; web app, real-time ledger, payout scheduling, Trust Score; markets "guaranteed payouts" with **no disclosed custody bank or licence**. Pricing, users, funding undisclosed. [theouut.com, 13 Mar 2026; itedgenews.africa, 12 Mar 2026]

### 4.7 Esusu Africa — B2B2C software for thrift collectors/cooperatives/MFIs: member accounts, loans, automated repayment, reports, self-service portal, POS/cards/APIs. **Published pricing:** 7 tiers ₦25,000/month (500 members) to ₦9,999,999 lifetime. 180+ institutions, 500k customers, ₦100bn saved [self-reported, undated]. Gates Foundation-funded; no licence (custody with client's bank). [https://esusu.africa/; https://www.gatesfoundation.org/ideas/articles/digital-finance-nigeria-market-sellers]

### 4.8 Sterling OneBank "Ajo scheme" (2 Mar 2021) — rotating contributions, auto-debit, payout 28th; **bank explicitly does not guarantee payments or arbitrate unfunded accounts**. [BusinessDay, 2 Mar 2021] Alt bank group features: none found.

### 4.9 Wema ALAT Goals — personal + group target and rotating savings up to 17%. [TechCabal, 22 Aug 2025] See CoopHub §5.1.

### 4.10 Others — TrustAjo (escrow-locked, member vote for emergency withdrawal; no bank/licence); My Ajo App (non-custodial via Paystack; 2%/contribution, ₦1,000 KYC, admin ₦2–5k/month; **40+ users**); Ajo (ajo-app.com) free tracker; CircleFunds. Rank (ex-Moni, rebranded Nov 2025; Rank MFB) now offers Circles/Ajo community savings. [myajoapp.app; trustajo.com; userank.com]

---

## 5. Per-player findings — Cooperative management SaaS, core systems, institutions

### 5.1 Wema CoopHub (coophub.alat.ng) — the only Nigerian product documenting **three-role withdrawal control (Initiator/Verifier/Approver)**, plus member registration, contributions, loan requests/disbursement, bulk transfers, analytics; custody in Wema accounts. Pricing and societies onboarded undisclosed; payroll check-off, statements, returns not described. [Vanguard, 29 May 2024 (promotional): https://www.vanguardngr.com/2024/05/a-review-of-coophub-wema-banks-digital-solution-for-cooperative-societies/]

### 5.2 Coopify / ProbityCoop — cloud/offline/self-hosted; membership, contributions, loans, guarantor verification, SMS, portal, reports. Free ≤10 members; per-member/month Standard; one-off Premium >500. "500+ cooperatives" incl. Sterling Bank Staff Cooperative [self-reported]. Payments via 3rd Quadrant. No payroll deduction mention. [https://coopify.africa/; https://probitysoftware.com/coop.html]

### 5.3 Coopco — ₦15k/₦30k/₦90k per month; "13 books of accounts", financial statements, audit trail, member portal. [https://coopco.com.ng/]

### 5.4 Coopra — staff-cooperative focus; **staff deduction schedules**, savings, loans, dividends, reports; pricing not retrievable. [https://getcoopra.com/solutions/employee-cooperative/]

### 5.5 CoopSolve — clients ATBU Staff Coop (auto-deductions, portals), Aidlife (5,000+ members), NNPC-PH Refinery thrift app with instant loans. [https://coopsolve.com/; Play: com.coopsolve.phrc]

### 5.6 Orbital CMS (Abuja) — savings/loan managers, appropriation account, TB/BS/P&L, bank reconciliation, payroll administration, **housing projects module**; ₦650k Standard / ₦1.8m Professional [snippet, unverified]. [https://www.orbital.com.ng/coop.html]

### 5.7 SmartCoopHub, SureThrift, BankCore Africa (₦10k–₦1m/month + ₦100k setup; "5k+ businesses" [self-reported]); Prolean (₦300k/₦500k; **domain does not resolve — possibly defunct**); CoopEngine-core (open-source, 0 stars). [sites as named]

### 5.8 Names with no Nigerian product found — Coopnet, CoopManager, Thrift Pro, Cooperative.ng, Koopmart, Sunrise, Mycoop, CoopRunner, CoopPlus. Treat as non-existent or unindexed [unverified].

### 5.9 Core banking used by MFBs/some cooperatives — **BankOne (Qore):** 500+ FIs, enterprise licence, loan origination "rudimentary" per Lendsqr [blog.lendsqr.com, upd. 21 Jul 2026]. **Oradian Instafin:** LARDI, Pearl Funds & Asset Mgmt MPCS (a cooperative), ANMFIN partnership [BusinessDay, 28 Mar 2019]. **Musoni:** Nigeria launch, 39+ FIs [unverified]. **Mambu:** Renmoney, Carbon — not cooperative. None documents IPPIS/Remita schedule import.

### 5.10 Large staff cooperatives — **CBN Staff MPCS ("CoinCoop")**: 9,000+ members, member/backend portals, vendor unidentified [cbn.coop.ng]. **NNPC Staff Cooperative (Lagos, est. 1979)**: ₦806m loss 2022 vs ₦96m profit 2021; no AGM for six years; retirees unable to withdraw; forensic audit demanded (May 2025) [Vanguard, 30 May 2025: https://www.vanguardngr.com/2025/05/nnpc-retirees-cry-out-over-possible-loss-of-savings-with-staff-cooperative/] — a documented **governance/dual-control failure in a flagship staff coop**. University coops (UNN Lion MPCS, ATBU) on bespoke/CoopSolve systems.

### 5.11 CFAN (ex-NACCUN; rebranded 7 Apr 2015) — 5.2m members, 2,040 groups, 774 LGAs; building a cloud **Unified Co-operative Platform (UCP)** for MIS; Cooperative Development Fund (Nov 2019, ₦5bn/yr target). UCP vendor/cost unknown. [https://www.cfan.coop/about-cfan/]

### 5.12 Federal programme — RH-CRRP 2030, NCDAP, Seamfix registry
- RH-CRRP 2030 approved at 8th National Council on Cooperative Affairs (Kaduna, Mar 2026); 7 pillars incl. Cooperative Bank and digitalisation; assets ₦1.7trn → ₦5trn target. [agronigeria.ng, 22 May 2026; fmino.gov.ng, 7 Oct 2026]
- NCDAP = National Cooperative Societies Register (NCSR) + Cooperative Verification Number (CVN) + CoopID (NIN-linked) + **CoopCHECK credit bureau** ("powered by CreditRegistry" [unverified]) + USSD. [thecooperator.news, 6 Sep 2026]
- **MoU 12 Aug 2026** FMAFS + Cooperative Federation of Nigeria + **Seamfix** (deploy/operate/maintain); six-month revalidation of all cooperatives via State Directors → NCSR → CVN; pilot in six zones. [agriculture.gov.ng; Nairametrics, 17 Aug 2026: https://nairametrics.com/2026/08/17/seamfix-federal-government-and-cfn-sign-agreement-...] "No cost to Federal or State Governments" (7 Oct 2026) → vendor-funded; fees to societies **unpublished**. Registry does **not** replace state registration. Scale cited: 370k societies (Nairametrics) vs 30k (Technology Times) [conflicting]. Kogi mandated digital revalidation 4 Sep 2026.
- **Cooperative Bank of Nigeria:** 65% cooperatives (via "CoopTrust") / 30% institutional / 5% ESOP; "government-enabled, not government-funded"; ₦200bn share drive launched 18–19 Jun 2026 for 10,000 societies (₦21–50m per large society); **subscription from end-Oct 2026**; CBN licence category unannounced. [msmeafricaonline.com, 19 Jun 2026; allafrica.com, 18 May 2026]
- **Legislation:** still Nigerian Co-operative Societies Act 1993 (Cap N98). Minister directed a repeal-and-re-enactment Cooperative Societies Bill (Aug 2026) recognising e-records/digital ID/fintech; no bill text or NASS reading found [unverified]. **No licensing regime for cooperative software vendors exists**; the de-facto gate is state revalidation → CVN.

---

## 6. Per-player findings — Payroll-deduction rails and lenders

### 6.1 IPPIS / WACS (OAGF)
- Pays ~600k federal employees [unverified] with third-party deductions (tax, pension, cooperatives, unions, bank loans). Official page last updated with 2014 data. [https://oagf.gov.ng/initiative/ippis/]
- **WACS** ("Workers Aggregated Credit Scheme") aggregates **all** lender deductions into one payslip line "WACS-Bank ABC"; IPPIS promised a per-lender breakdown app (Oct 2023) — no evidence it shipped. [FIJ, 10 Oct 2023: https://fij.ng/article/more-civil-servants-complain-about-wacs-deduction-but-ippis-says-its-loan-management/]
- **Deduction-code application process, OAGF approval workflow and fees: not published anywhere** [unverified]. One-third-of-salary cap cited by Kano audit; federal circular not located.
- **Failures:** duplicate ₦29,030 Credit Direct deductions and deductions for non-existent "Kaylog"/"Crevance Credit" loans at UCH, ABUTH, FMC Yenagoa etc., Jan 2024 [FIJ, 8 Feb 2024]; paramilitary protests Nov 2024 [Daily Trust]; University of Uyo study: no formal IPPIS–cooperative recovery relationship [Zenodo, 2021].
- **GIFMIS migration:** federal tertiary institutions removed from IPPIS 4 Nov 2024; payroll now prepared **in Excel** and submitted for GIFMIS processing [West African Pilot, 4 Nov 2024]. ASUU (Sep–Oct 2026): cooperative/union/pension deductions "running into billions" unremitted for months [Leadership, 8 Oct 2026].
- No evidence of "IPPIS outsourced to Remita" or "HR/Payroll by Treasury"; only a June 2026 HR self-service upgrade found.

### 6.2 Remita (SystemSpecs)
- Data Referencing API for lenders (net salary, history, existing loans; debit next salary); >2m salaries/month (2019). Partner lenders: Access, Carbon, Ibile MFB, etc. [BusinessDay, 27 Sep 2019]
- Remita payroll-loan lenders: VeendHQ, Funds Home, Liberty Assured, Glory MFB. Remita **Direct Debit** (bank mandate): ~₦50 or small % per successful debit, T+1, 3–4 retries; excludes First Bank, Stanbic, StanChart [Lendsqr, Nov 2025: https://blog.lendsqr.com/frequently-asked-questions-on-loans-powered-by-remita/]. Official payroll fee schedule and lender onboarding: **not retrievable** [unverified]. TMRAS (Mar 2025) concerns revenue collection, not payroll.

### 6.3 CREDICORP — FG DFI (Apr 2024); 1.6m applications; partners Credit Direct, Page, Accion, Ecobank, Wema, Letshego etc.; Credit Direct CREDICORP loans 3.8–3.95% flat/month, ≤₦5m, ≤36 months, salary-deducted. [https://www.credicorp.ng/credit; creditdirect.ng, 18 Sep 2024]

### 6.4 State payroll — **Kano (8 May 2026):** vendors deducted ₦1.8bn arbitrarily, breached 1/3 cap; ₦13bn outstanding; state barred all vendors from payroll [Ripples: https://www.ripplesnigeria.com/kano-govt-fumes-...]. Lagos: Page Financials/Addosser LSCS products; Oracle payroll interface details not found. Kaduna: nothing found.

### 6.5 NIBSS direct debit / e-mandate — e-mandate via ₦50 verification transfer, ~2h activation; neobanks (OPay, PalmPay, Moniepoint, Kuda) don't support DD; Mono DD 1% capped ₦1,000. Incidents: ₦13.66bn NIP dry-posting Sep 2024; NIBSS downtimes May 2026. FCCPC DEON pushes lenders to GSI. [Lendsqr, 31 Aug 2024; mono.co, 5 Mar 2025; BusinessDay 2026]

### 6.6 FCCPC DEON Regulations 2025 — effective 21 Jul 2025, full enforcement Jan 2026; 560 approved / 35 conditional / 112 watchlist / 54 delisted (Oct 2026); fines to ₦100m or 1% turnover. [https://fccpc.gov.ng/registration-of-digital-money-lenders/; Legit, 17 Apr & 22 Jul 2026]

### 6.7 Lenders (summary)
| Lender | Licence | Payroll rail | Pricing | Scale | Notes |
|---|---|---|---|---|---|
| Credit Direct (FCMB) | CBN finance co. | IPPIS, state, CREDICORP | 3.8–3.95% flat/mo (CREDICORP) | 2m+ customers | duplicate IPPIS deductions Jan 2024 |
| Page Financials | CBN finance co. | IPPIS, Lagos State | n/p | n/p | CREDICORP partner |
| Zedvance | CBN finance co. | Remita, NIBSS, Mono, CreditRegistry | n/p | 500k+ customers, ₦40bn+/18 months [self-reported] | public & private payroll |
| Renmoney | CBN MFB | card/DD (no payroll evidence) | 2.12–2.65%/mo + 1% fee | 3m+ customers | Mambu core |
| FairMoney | CBN MFB | DD | 2.5–30%/mo [unverified] | ₦150bn disbursed 2025 | ~$57m funding [unverified] |
| Carbon | CBN MFB | Remita data ref. | from 3.99%/mo | — | — |
| Specta (Sterling) | bank | IPPIS [unverified] | n/p | — | JS-only site |
| Branch, QuickCheck, Aella, Umba, Palmcredit | MFB/FCCPC | DD | n/p | QuickCheck 3.5m; Umba 50k | — |
| Sycamore | SEC | — | "Loan Friends" P2P ≤50 friends | — | no coop/payroll |

### 6.8 Lender SaaS — Lendsqr ($0/$200/$500/month; NIBSS + Remita DD; 3.7m borrowers; **no cooperative module, no schedule import**); Evolve Credit/Configure (credit unions, MFBs; no pricing); Indicina; Mono Connect ₦50k/month; Okra likely defunct [unverified]; Periculum ($620k pre-seed). [https://lendsqr.com/pricing; https://www.evolvecredit.co/; https://mono.co/pricing]

### 6.9 Reconciliation tools for cooperatives — **none found.** IPPIS schedule and Remita payroll file formats unpublished; deduction schedules move as spreadsheets; only Coopra/CoopSolve/Orbital mention deduction schedules at all.

---

## 7. Per-player findings — Housing, proptech, escrow, title

### 7.1 Keble — fractional/invest-to-own, $10 min, NGN/USD/GBP; trustee Meristem, audits Deloitte/EY, AXA cover; 30k users, 70+ properties (Sep 2025); Techstars/ARM-backed; **SCUML AML registration only — no SEC licence found**; fees undisclosed; no secondary market. [Nairametrics, 25 Sep 2025; keble.co]

### 7.2 Propcrowdy — **SEC Crowdfunding Intermediary (Dec 2022)**, one of 5 licensed portals (Sep 2025); investor funds held with **custodian bank, disbursed to developers by milestone**; title collateralised; SEC approves listings; ₦100m cap per raise; ~20k investors [snippet]; $500k (2021). Take rate undisclosed; custodian unnamed. [Nairametrics, 27 Dec 2022; BusinessDay, 28 Sep 2025] **Closest existing analogue to developer escrow for retail pools.**

### 7.3 MonieWorx — SEC portal (Dec 2022); ₦5.23bn raised, 10k investors, ₦1,000 tickets, 12% 90-day notes; not real-estate specific. [monieworx.com]

### 7.4 Estate Intel — B2B data; $500k MetaProp pre-seed + $100k Google BFF (2022). Benchmark source only.

### 7.5 Spleet — rent-tech; $2.6m seed Oct 2022; **CEO removed Mar 2024 after audit, ~$1.5m alleged misappropriation** [snippet, unverified]; no 2025–26 updates. Governance lesson: founder-controlled funds without dual control.

### 7.6 Kwaba — rent-now-pay-later 6%, savings 7–8%; scale/licence unverified; TLS expired on both domains.

### 7.7 HouseAfrica / Sytemap — satellite-verified listings, NFT plot allocations, "10-year encumbrance guarantee"; 30k users, ₦5bn land, 100+ developers; $400k (Apr 2023); SCUML only; **own T&Cs allow up to 40% admin fee on refunds**; no open verification API. [sytemap.com; Nairametrics, 18 Apr 2023]

### 7.8 Seso Global, Landproof, Buyletlive — active but no registry adoption, no pricing, no escrow. Flexcrow, Trusted Transfers, PayEscrow Africa, Trustcrow — domains parked/unresolved: **defunct or misnamed**.

### 7.9 FMBN — FMBN Mobile (NHF balance, calculator); 2025: Rent-to-Own 367 homes/₦7.1bn, 6,911 units financed, ₦15.6bn refunds [NAN, 16 Jul 2026]. **Cooperative Housing Development Loan:** registered coop with **unencumbered title**, members NHF contributors, ≤₦500m at 10%, 24-month tenor/12-month moratorium; ~40 projects/2,570 units (2021) [ThisDay, 11 Aug 2021]. Developer fees ₦600k accreditation, ₦20k/₦50m application, ₦160k legal search; title perfected before first tranche; BOQ-based disbursement [FMBN PDF, May 2026]. **No API, no member-level escrow.**

### 7.10 Family Homes Funds / MREIF — ₦128bn mortgages to 1,859 families at 9.75%/20 yrs (Jun 2026); 38,400 student beds PPP. [BrandSpur, 10 Jun 2026]

### 7.11 LagosHOMS — Rent-to-Own 5% down/10 yrs; mortgage 6%/30% equity; portal 15 Jun 2025; ₦5k + ₦10k fees; 20k families since 2014. [lagoshoms.gov.ng; HousingTV, 3 Sep 2026]

### 7.12 Lagos Lands Bureau e-GIS (27 Jan 2024) — online title search, C of O and consent applications; **no public API**; TLS issues; Assembly summoned Bureau May 2025. KANGIS/KADGIS domains unresolved. [Nairametrics, 27 Jan 2024]

### 7.13 Escrow providers — **PayScrow** 2% + ₦100, segregated DMB accounts, milestone release, real estate supported; **Vesicash** 2.5%/5%, milestone & split, API, Ingressive-backed; **Escrow Africa** 1.5% (Accra); **Escrownig** fixed + 2%; **Escrownet.ng** "escrow banking layer for real-estate buyback", AGIS title check before release. None holds its own CBN licence; **no CBN escrow licence exists** — escrow runs under PSP framework or banks/SEC trustees [Mondaq, 2024]. Trustees: FirstBank Trustees, ARM Trustees (escrow listed as *unregulated* offering), UTL Trustees (SEC, since 1966). Paystack/Flutterwave: split payments only, **no hold-and-release**.

### 7.14 Estate software — Gate Africa (dues, statements, approval workflows; 10k users), Residence.ng (16+ estates), GatePass (709 homes), AreaConnect, VENCO. **None offers cooperative equity ledgers, developer escrow or dual-control releases.** Veegil, Mycommunity, Estate Manager NG, Tenantr — no live sites.

### 7.15 Title verification — LandVerify ₦25.5k/₦49.5k, 4 states, **API offered**; PlotSight ₦5k instant / ₦20k full; LandSafe (May 2025) [unverified]; Youverify/Smile ID have **no land-title product**. Governor's consent: Lagos 30-day target vs 3–6+ months real; all-in 10–15% of price [WithinNigeria, 8 Jul 2026]. *Olabanji v Minister FCT* (25 Feb 2025): first-in-time prevails on double allocation.

### 7.16 Documented housing failures
| Case | Facts | Source |
|---|---|---|
| RevolutionPlus — Flourish Apartments Lekki | ~50 subscribers paid ~₦787m into company/Providus accounts, no escrow; stalled; ₦32.2m "variation" demanded; EFCC probe Aug 2024; settlement May 2025; Interpol notice sought May 2026 | Sahara Reporters 19 May 2026; ThisDay 28 Jan 2026 |
| RevolutionPlus land-banking / LASRERA | 40% p.a. notes unpaid; refunds minus 40% | FIJ 13 Sep 2025; Tribune 22 Jan 2024 |
| Adron Homes | 30% admin charge on refunds; ₦11.5m undisclosed development fees; 5-year refund delays | FIJ Aug 2025 |
| Lagos illegal estates | 176 estates (Ibeju-Lekki 111) without layout approval; 21-day ultimatum | Supreme Magazine, 4–5 Aug 2025 |
| FCCPC sealing | Ochacho Group, Ti'Bilon (Abuja) for non-delivery | NigeriaHousingMarket 21 May 2026 |
| Omo-onile | 7,500 petitions, 205 arrests, 60 prosecutions (Lagos, Feb 2025) | Tribune 22 Feb 2025 |
| Sector losses | ₦16.2bn lost Jan–Jul 2025 | [snippet, unverified] |
| Registered housing cooperative losing pooled funds | **No named court case surfaced** — failures are developer-side | gap: check EFCC/LASRERA records |

---

## 8. Kenya comparators

### 8.1 Sector and regulation
- SASRA Supervision Report 2025: assets KSh 1.21trn (+12.5%), deposits KSh 832.7bn, loans KSh 948.7bn, 7.87m members, 357 regulated SACCOs (177 DT + 178 NWDT, 2024); NPL 6.36% vs banks 15.56%; 60 large SACCOs hold 77% of assets. Digital: 407 approved digital products, 250 USSD SACCOs, 184 app/web SACCOs. [Launch Base Africa, 8 Oct 2026: https://launchbaseafrica.com/2026/10/08/sacco-sector-kenya/; saccoreview.co.ke]
- **Check-off:** s.35 Co-operative Societies Act — Commissioner can attach employer bank accounts (KSh 200m recovered from UoN). Non-remittance KSh 2.7bn (2022) → **KSh 3.92bn (2025, 104,331 members, 89 SACCOs)**; county governments KSh 1.88bn. Treasury reform to pay SACCOs **directly from central payroll** (expected Jul 2026, still pending 30 Sep 2026). [Kenyans.co.ke, 28 Sep 2026; Business Daily, 30 Sep 2026]
- **Sacco Central** (liquidity + shared services co-op, 2021; 52–55 DT-SACCOs; World Bank SAFER) and **SACCO Societies (Amendment) Bill 2025** (CLF licence, Deposit Guarantee Fund operationalisation, mandatory digital returns) — still before Parliament (Jul 2026). **No operating SACCO deposit insurance yet.** [ckadvocates.co.ke, 20 Jul 2026]
- **KUSCCO scandal:** unregulated apex deposit-taking from 247 SACCOs; PwC audit: KSh 13.3bn lost, insolvent by KSh 12.5bn, forged auditor signature; 201 SACCOs lost ~10% core capital; liquidation Sep 2026. SASRA had no mandate over secondary co-ops. [Capital FM, 13 May 2026; Kenyans.co.ke, 6 Mar 2025]
- Member redress: Co-operative Tribunal ~10,959 pending; refund disputes ~4 years.

### 8.2 Players
- **Chamasoft (Digital Vision EA):** invoicing, auto statements, reminders, bank-alert reconciliation, M-Pesa/Jenga; Free; KSh 1,000/month ≤20 members (+KSh 30/member); 50k users; white-labels Equity **EazzyChama**. No custody/licence. [https://chamasoft.com/pricing/]
- **ChamaPay / MyChama:** STK push, C2B auto-match, guarantor loans, PDF statements, role-based access; KSh 500–5,000/month; ChamaPay 500+ chamas, KSh 100m+ loans; explicitly "not a bank". [chamapay.co.ke; mychama.app]
- **Postbank M-Chama** (FSD Kenya): group account + SMS, paybill 200999; free subscription; requires constitution and minutes. [postbank.co.ke]
- **Safaricom Mali/Ziidi:** Ziidi MMF (CMA Nov 2024) 2m customers, KSh 18.7bn AUM (Mar 2026); Ziidi Shari'ah launched 6–7 Oct 2026; Mali fund-manager dispute with Genghis (Dec 2024). No chama governance tooling. [Business Daily; TechTrends, 7 Oct 2026]
- **Kwara:** SACCO core + member app/USSD, digital guarantors, SASRA-compliant reporting; fixed annual subscription; 230+ SACCOs [self-reported]; $7m seed; acquired IRNET (KUSCCO software arm) Jan 2023 with exclusive 4,000-SACCO channel — now compromised. [TechCrunch, 16 Jan 2023; kwara.com]
- **Craft Silicon** (Bankers Realm/Elma; 200+ FIs), **Fintech Group** (500+ FIs since 1993), **Coretec** (CoopMIS 29 SACCOs; Dynamics 365 BC 37 implementations; SASRA automated reporting, bank rec, CRB) — all ship **check-off reconciliation**. Sky World, Finsoft, Neptune [unverified]. [coretecafrica.com]
- **Co-op Bank Kenya:** Sacco Link switch — 619 FOSA branches (2024); serves 110 DT-SACCOs for ATMs, 92 cheque clearing, 21 PesaLink; MCo-opCash. KCB–Stima Sacco integration 6 Oct 2026. [Business Daily, 9 May 2025; nyongesasande, 8 Oct 2026]
- **Bank chama products:** KCB Chama (loan approve/decline, reconciliation from alerts, TB/BS reports), Family Bank Chama (4.5%, loans 3x savings), Equity EazzyChama, Stanbic Chama app. Dual signatories enforced at account-mandate level.
- **NACHU:** housing-co-op apex (1979); ~2,000 units; housing microloans; donor-dependent; **discontinued 400-unit project Sep 2020** as contributions fell, KSh 500m rescheduled. [CAHF case study; commercialpropertykenya, 22 Sep 2020]
- **Hustler Fund group loans:** 51k groups, 1.1m members; 68.3% default (KHRC Aug 2025); 386,735 accounts closed unpaid. [Capital FM, 4 Aug 2025; Nation]

### 8.3 What Kenya has that Nigeria lacks
Institutional: prudential SACCO regulator with public annual data; statutory check-off with enforcement; movement-owned Tier-1 bank running a SACCO switch; legislated liquidity facility + deposit guarantee pipeline; instant rails (PesaLink, STK push) wired into SACCO/chama software; a housing-co-op apex. Software: native check-off modules and SASRA return generators in every core; bank white-label chama ledgers; sub-KSh 5,000/month chama SaaS with auto-matching; member USSD/app self-service at most SACCOs. **Kenya's own gaps:** no live deposit insurance, chama software never holds funds or offers escrow/dual control natively, analog member redress, apex liquidity unlicensed.

---

## 9. White space (where there is clearly nobody)

| # | Gap | Evidence | Nearest incumbent |
|---|---|---|---|
| 1 | **Pooled-fund custody + society ledger + dual-control payments in one product** | Only Wema CoopHub has 3-role control (undocumented scale); SaaS vendors are non-custodial; fintechs don't pool | Wema CoopHub; Propcrowdy (for crowdfunding only) |
| 2 | **Payroll check-off reconciliation** (import IPPIS/Remita/state schedules, match to member ledgers, age unremitted balances, flag double deductions) | WACS single-line payslips; Excel GIFMIS payroll; ASUU unremitted billions; Kano ₦1.8bn; Lendsqr/Configure/Indicina have no module; formats unpublished | Coopra, CoopSolve, Orbital (schedule entry only); every Kenyan SACCO core |
| 3 | **Developer-payment escrow for cooperative estates** (milestone release tied to BOQ + title + member votes) | RevolutionPlus/Adron pattern; FMBN disburses to coop/developer; escrow fintechs are marketplace-oriented; no CBN escrow licence | Propcrowdy custodian model; PayScrow/Vesicash milestone APIs; SEC trustees |
| 4 | **Title-verification API wired into payment release** | Lagos e-GIS has no API; LandVerify API exists but no one integrates it into escrow release except Escrownet's AGIS check | LandVerify, Escrownet.ng |
| 5 | **Member statements and regulator returns for registered cooperatives** (state Director returns, future CVN/CoopCHECK feeds, Cooperative Bank KYC) | No vendor documents returns; registry just launching; Cooperative Bank will need society-level data | Coopco "13 books"; Kenyan SASRA return generators |
| 6 | **Staff/workplace cooperative governance** (AGM, audit trail, trustee rotation, withdrawal SLAs) | NNPC coop ₦806m loss, 6 years without AGM; CBN coop vendor unidentified | CoopSolve, Coopra |
| 7 | **Licensed custody model for ROSCA/ajo apps** | Thrifto, TrustAjo, AjoMoney, Alajo all undisclosed custody; Sterling disclaims guarantees; Rank MFB is the only licensed ajo re-entrant | Rank (ex-Moni), AjoCard |
| 8 | **Movement-level clearing / liquidity** (Co-op Bank Kenya / Sacco Central analogue) | Cooperative Bank of Nigeria is a plan; CFAN UCP unbuilt | CFAN, Cooperative Bank of Nigeria (planned) |
| 9 | **Cooperative-structured pooled investing done legally** | SEC struck Risevest's cooperative wrapper; Cowa unverifiable | Cowa [unverified], PiggyVest Joint Accounts |
| 10 | **Housing-cooperative apex with technical/finance support** (NACHU analogue) | FMBN CHDLS requires title first; no apex, no member-level escrow | FMBN, Family Homes Funds |

**Pricing reference points for positioning:** cooperative SaaS ₦15k–₦90k/month or ₦300k–₦1.8m one-off; Esusu Africa ₦25k/month entry; Lendsqr $200–$500/month; escrow 1.5–2.5% of value; crowdfunding portals take rate undisclosed; Kenyan chama SaaS KSh 500–5,000/month; Kwara fixed annual subscription; Mono DD 1% capped ₦1,000; Remita DD ~₦50/debit.

---

## 10. Open questions / not researched
- IPPIS deduction-code application process, fees and file formats; Remita official payroll-loan fee schedule — require direct enquiry.
- Seamfix registry fee model to societies; CoopCHECK/CreditRegistry contract; Cooperative Bank CBN licence category.
- Cowrywise Collections custody rules; PiggyVest Joint Account control rules; Cowa's existence/SEC status.
- Wema CoopHub societies onboarded and pricing; CBN Staff Coop vendor; FIRS/NDIC coop systems.
- Specta, Palmcredit, Lendha, Kwaba, Okra, Onepipe payroll products; loan-book sizes beyond marketing.
- Bankly post-acquisition status; Alajo/AjoMoney/Prolean liveness.
- Gidimo, Fundall, Ajo.ng, Esusu Mobile, M2P; Kenyan Tanda, Stawi, Jijenge, Sky World, Finsoft, Neptune.
- Named court cases of Nigerian registered housing cooperatives losing pooled funds (EFCC/LASRERA records).

## 11. Source index (abbreviated; full URLs inline above)
Nigeria consumer fintech: piggyvest.com/faq; blog.piggyvest.com (29 May 2026); technext24.com (26 Sep 2026, 9 Jan 2026, 8 Sep 2025, 7 May 2025, 19 Feb 2026); techpoint.africa (Jan 2026, 11 Sep 2024, 26 Sep 2023); techcabal.com (7 Oct 2026, 30 May 2026, 28 Jan 2021); cowrywise.com/circles; help.cowrywise.com; businessday.ng (19 Feb 2026); innovation-village.com (14 May 2026); arbiterz.com (Sep 2026); arise.tv (29 Apr 2024); nairatechie.com (23 May 2026); investbamboo.com; blog.trovefinance.com (3 Jun 2026); legit.ng (Apr 2021); Wikipedia (Kuda, Moniepoint, OPay, PalmPay).
Ajo & coop SaaS: ajocard.com; cbinsights.com; nairametrics.com (29 Apr 2025, 17 Aug 2026); businessday.ng (16 Sep 2023, 2 Mar 2021); disruptafrica.com (18 Apr 2023); technext24.com (29 Jul 2025); trustfinance.com; theouut.com (13 Mar 2026); esusu.africa; gatesfoundation.org; vanguardngr.com (29 May 2024, 30 May 2025, 13 Aug 2026); coopify.africa; coopco.com.ng; getcoopra.com; coopsolve.com; orbital.com.ng; smartcoophub.com; surethrift.com; bankcore.africa; qore.inc; blog.lendsqr.com (21 Jul 2026); cfan.coop; agriculture.gov.ng; fmino.gov.ng (7 Oct 2026); thecooperator.news (6 Sep 2026); msmeafricaonline.com (19 Jun 2026); allafrica.com (18 May 2026); agronigeria.ng (22 May 2026); legal500.com (Jul 2026).
Payroll rails: oagf.gov.ng; soluap.com/wacs; fij.ng (10 Oct 2023, 8 Feb 2024); dailytrust.com (30 Nov 2024); zenodo.org (2021); westafricanpilotnews.com (4 Nov 2024); leadership.ng (8 Oct 2026); businessday.ng (27 Sep 2019); blog.lendsqr.com (Nov 2025, 31 Aug 2024); docs.lendsqr.com; credicorp.ng; creditdirect.ng (18 Sep 2024); ripplesnigeria.com (8 May 2026); mono.co (5 Mar 2025); fccpc.gov.ng; legit.ng (17 Apr, 22 Jul 2026); nairametrics.com (25 Apr 2026); zedvance.com; renmoney.com; lendsqr.com/pricing; evolvecredit.co.
Housing/escrow: keble.co; nairametrics.com (25 Sep 2025, 27 Dec 2022, 18 Apr 2023, 27 Jan 2024, 24 Sep 2026); businessday.ng (28 Sep 2025); monieworx.com; techcrunch.com (4 Oct 2022); sytemap.com; thisdaylive.com (11 Aug 2021, 28 Jan 2026); fmbn.gov.ng (PDF May 2026); nannews.ng (16 Jul 2026); brandspurng.com (10 Jun 2026); lagoshoms.gov.ng; housingtvafrica.com (3 Sep 2026); payscrow.net; vesicash.com; escrowafrica.net; escrownet.ng; arm.com.ng; utltrustees.com; mondaq.com (2024); developer.flutterwave.com; gate.africa; residence.ng; gatepass.io; app.landverify.ng; plotsight.tech; withinnigeria.com (8 Jul 2026); legal500.com (Olabanji, 25 Feb 2025); saharareporters.com (19 May 2026); fij.ng (13 Sep & 25 Aug 2025); tribuneonlineng.com (22 Jan 2024, 22 Feb 2025); suprememagazine.news (Aug 2025); nigeriahousingmarket.com (21 May 2026).
Kenya: launchbaseafrica.com (8 Oct 2026); saccoreview.co.ke; sasra.go.ke; kenyans.co.ke (28 Sep 2026, 6 & 13 Mar 2025); businessdailyafrica.com (30 Sep 2026, 9 May 2025, 4 Mar 2025); ckadvocates.co.ke (20 Jul 2026); capitalfm.co.ke (13 May 2026); techtrendske.co.ke (30 Sep & 7 Oct 2026); chamasoft.com; chamapay.co.ke; mychama.app; postbank.co.ke; kwara.com; techcrunch.com (16 Jan 2023); coretecafrica.com; craftsilicon.com; fintech-group.com; kcbchama.kcbgroup.com; housingfinanceafrica.org (NACHU case study); commercialpropertykenya.com (22 Sep 2020); capitalfm.africa (4 Aug 2025); nation.africa.
