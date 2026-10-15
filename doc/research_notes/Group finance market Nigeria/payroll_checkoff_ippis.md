# Federal payroll check-off in Nigeria: IPPIS, WACS, GIFMIS, TMRAS — how third-party (cooperative and loan) deductions actually work, and every documented failure

**Compiled:** 8 Oct 2026. **Method:** ~45 web searches and ~60 page fetches across oagf.gov.ng (incl. one circular PDF extracted locally), FIJ, Vanguard, Daily Trust, TheCable, SaharaReporters, Guardian, Leadership, Nairametrics, Tribune, NICN portal, PenCom/FMBN coverage, lender sites (Credit Direct, VeendHQ, Moniestack, Alpha-Vista), and two academic papers. **Labels:** **[published]** = official document or official statement; **[practice]** = how it works as described by participants (cooperative officers, lenders, borrowers, journalists), not in any official document; **[unverified]** = single or indirect source; **[snippet]** = seen only in a search-engine excerpt of a page that would not load; **[not found]** = searched for, nothing located. Several sites (ippis.gov.ng itself, Nairaland, Guardian, Blueprint, Leadership-dated pages, Remita support) blocked automated fetches; those items rely on secondary coverage.

**Headline:** There is **no published OAGF/IPPIS document** describing how a cooperative or lender obtains a deduction code, what the schedule format is, what it costs, or when remittance is due. Everything about the onboarding/schedule/remittance cycle below is reconstructed from participants' accounts. By contrast, the *failures* are abundantly documented: aggregation of all lender lines into one opaque "WACS" payslip line (Oct 2023), duplicate deductions (Jan 2024), deductions for loans never taken (2023–2025), deductions continuing after payoff (2023–2025), unremitted cooperative deductions (NBS Kwara ₦4.4m, held since Jan 2022), ₦12bn unremitted to lenders (Dec 2024), ₦14.9bn unremitted NHF (2021–22), net-only payrolls that silently "step down" all deductions when cash backing is short (from Dec 2022), and the Nov 2024 tertiary exit to Excel-based GIFMIS payroll that broke remittances for a year (Jun 2025 AGF admission; ASUU Sep 2026).

---

## 0. One-page picture of the cycle (as practised)

```
 Cooperative / lender                 MDA payroll desk                 OAGF – IPPIS Dept.                     Payment rails              Third party's bank
 ───────────────────                  ────────────────                 ──────────────────                     ─────────────              ─────────────────
 1. Member borrows; society           2. Desk officer/HR receives      3. IPPIS runs monthly payroll on       5. Salary paid to staff     6. Third-party remittance
    prepares "variation"                 variation and submits to         Oracle HCM (to Jul 2026) / new        between ~20th–25th (after   expected "first week of
    (name, IPPIS no., amount,            IPPIS "for capture"              SoftAlliance platform (from Jul       FAAC) via NIBSS/NIPS,       following month" — bulk
    months) and sends it to IPPIS        [practice; brands.ng 2026]       2026) [published]; deductions          e-Tranzact, Remita, GIFMIS   credit, no itemised
    directly or through the MDA                                           appear on payslip (cooperative by      [published, Deko Aug 2023]  schedule described
    [practice; FIJ Nov 2024]                                              name; lenders as "WACS Loan"/                                     [practice; FIJ Nov 2024]
                                                                          "WACS-<lender>") [practice; FIJ Oct 2023; Soluap]
                                                                       4. If cash backing is short, IPPIS is directed to pay NET ONLY and all deductions are "stepped down"
                                                                          (deducted on payslip, not remitted) [published, Deko to Reps, Aug 2023]
```

The cooperative's and lender's recourse when step 6 fails: physical letters to OAGF, the 0700-2754774 call centre (routinely unanswered per FIJ), bursary units, or the press.

---

## 1. IPPIS: who runs it, coverage, calendar

**Takeaway.** IPPIS is a department of the Office of the Accountant-General of the Federation (OAGF) that *processes* the federal executive payroll; it does not itself hold or move money — payment goes out via gateways (NIBSS/NIPS, e-Tranzact, Remita, GIFMIS). Coverage figures conflict wildly (600k vs. 720k vs. 1.0m+). Salaries land roughly 20th–25th. No published payroll calendar or variation cut-off exists.

**Cited findings.**
- Origin: conceived Oct 2006; pilot Apr 2007 with 7 MDAs; World Bank-financed ~USD 4.9m contract with SystemSpecs consortium; transferred to OAGF 2008 [published; SCIRP paper "Gains and Pains of IPPIS", 2021; ResearchGate 2022]. Platform was Oracle HCM/E-Business Suite until the 28 Jul 2026 go-live of a locally developed replacement by SoftAlliance and Resources Ltd, with HR modules (postings, promotions, leave, ESS) [published; Pulse, 28 Jul 2026].
- Coverage: 308 MDAs / 257,516 enrolled (Mar 2014, OAGF IPPIS page — stale) [published]; 459 agencies / 300k+ (Oct 2019, Pulse) ; 701 MDAs / 1,121,031 public servants, 9 MDAs still outside (Jul 2020, SCIRP) ; "720,000" personnel after 70,000 ghost workers removed (BPSR DG Dasuki Arabi, 24 Jun 2022, NewsVerge) [published]; "508 MDAs, 1,008,800+ employee records" (AGF at HR-module go-live, 28 Jul 2026, Pulse) [published]. The ~600k figure in blog guides is **[unverified]**. Note the Jul 2026 figure is *records*, not necessarily active paid staff, and post-dates the Nov 2024 exit of tertiary institutions.
- Scope: federal executive arm only. Self-funding/revenue agencies — CBN, FIRS, NNPC "and a few others" — are outside IPPIS because their personnel cost is not drawn from the Consolidated Revenue Fund (Channels TV, 4 Dec 2019) [published, via snippet]. Buhari directive 8 Oct 2019: any federal staff not on IPPIS by end-Oct 2019 would not be paid [published].
- Pay day: "between 20th–25th of every month" after the FAAC meeting [practice; Soluap]. Military guides say 25th–30th [unverified]. No official calendar located **[not found]**.
- Payment mechanics: IPPIS Director Emma James Deko told the House ad-hoc committee on NHF (Aug 2023) that IPPIS only *processes*; disbursement is by four designated gateways — NIPS (NIBSS), e-Tranzact, Remita and GIFMIS — and that deduction-remittance records sit with those gateways, which made collating NHF remittance history "as far back as 2011" slow [published; Leadership / TheCable / Peoples Daily, Aug 2023].
- Complaints channel: IPPIS Call Centre 0700-2754774; OAGF, Plot 1570 Samuel Ladoke Akintola Boulevard, Garki II, Abuja; payslips via MDA desk officer or the Head of Service "Self-Service" platform (Treasury Circular, IPPIS Dept, 6 Mar 2025) [published]. Payroll validation portal payrollvalidation.oagf.gov.ng introduced Jun 2024 [published; OAGF circulars list].

**Gaps.** No published monthly payroll calendar, variation cut-off date, or SLA for remittance timing. Active headcount in 2026 is unknown.

---

## 2. Third-party onboarding: how a cooperative or lender gets a "deduction code"

**Takeaway.** No public document, form, fee schedule or portal for third-party onboarding exists. Participants describe two parallel tracks: (a) **cooperatives/unions** register a deduction line with IPPIS (historically via the MDA) and then send monthly "variations"; (b) **lenders** since ~Sep 2023 operate through the IPPIS Third-Party Division's **WACS** platform and a 2023 **Service Level Agreement** signed between the Lenders Association (HDEO) and OAGF. Required documents, fees and timelines are **[not found]**.

**Cited findings.**
- OAGF's own description of IPPIS: pays salaries "with appropriate deductions and remittances of 3rd party payments such as: tax, pension, cooperatives, union dues and bank loans" [published; oagf.gov.ng/initiative/ippis]. Lists of channels elsewhere: FIRS, State IRS, NHIS, NHF, PFAs, Cooperative Societies, Trade Union/Association dues, Bank Loans [published; Zamfara AG page mirrors federal wording].
- IPPIS has a **Third Party & Other Payment Unit** handling "pension matters, PAYE and unions/cooperatives deductions" and performing "remittance of payroll deductions to third party" [practice; Soluap "WACS loan", undated].
- Cooperative track, as described by NBS Kwara staff cooperative officers: "what we do is to send lists of staff members who are co-operative members to IPPIS"; the society "prepare[s] the variation", e.g. a ₦100k–₦200k loan "divide[d] by 12 months"; IPPIS deducts and is expected to remit the lump sum to the society's bank account "during the first week of the following month"; the society communicates with IPPIS by **physical letters only** (no working email or phone) [practice; FIJ, 11 Nov 2024].
- Payslip-guide description: cooperative deductions, bank loan repayments and union dues appear if "your MDA has submitted [them] to IPPIS for capture" — i.e., routed through the MDA payroll unit [practice; brands.ng, 2026].
- Lender track: "the lender links that loan contract directly to the employee's unique IPPIS number" so repayment is "deducted at source on payday and remitted to the lender before the remaining salary balance reaches the employee's account" [practice; Credit Direct blog, 2026]. IPPIS describes WACS as "the IPPIS Third Party Settlement Management System… designed by the third-party division… to provide a single source of truth for loan data to eliminate all the lapses associated with manual processing and management of pay-day loans" [published (IPPIS website text quoted by FIJ, 10 Oct 2023)].
- Lenders Association (Humanity Development and Empowerment Organisation, HDEO; commercial banks, MFBs, finance companies): signed a **Service Level Agreement with OAGF in 2023** establishing "a framework for equitable credit access while safeguarding stakeholders' interests"; members disbursed ₦6.2bn to 2,400+ civil servants Nov 2022–Nov 2024 [published statement; Daily Trust 29 Nov 2024; Vanguard 1 Dec 2024]. SLA text **[not found]**.
- Borrower-side requirements lenders publish: IPPIS number, payslip, bank statement showing net pay, employment/confirmation letter, BVN, national ID, work ID [practice; lender sites: GTI MFB, Mutual Trust MFB, Glory MFB, Alpha-Vista]. There is no "IPPIS loan code" — USSD codes (e.g. VeendHQ *347*780#) are lenders' own [practice; Silicon Africa; VeendHQ].
- CREDICORP (est. Apr 2024) does not lend directly; it wholesales to banks/MFBs/fintechs/cooperatives; its civil-servant scheme (₦100bn, target 500k; 10,942 disbursed by Oct 2024; 25,000 by late 2024; 400k targeted by Q2 2025) repays via IPPIS deduction; MoU with Credit Direct (FCMB) [published; TheCable, Vanguard Oct 2024, Leadership].

**Gaps.** Required documents (CAC/cooperative registration certificate, MoU, bank mandate), any OAGF/IPPIS fee, approval timeline, whether cooperatives sign the same SLA as lenders, and whether a WACS lender portal exists — all **[not found]**. Treat any vendor claim of a "registered IPPIS deduction code" as unverifiable.

---

## 3. Schedule submission: channel and format

**Takeaway.** Cooperatives send a monthly "variation" list (member names, IPPIS numbers, amounts, tenor) to IPPIS, by letter/through the MDA; lenders submit via WACS. No template, column spec or file format is published.

**Cited findings.**
- NBS Kwara cooperative: monthly list of members and amounts; loans split over 12 months; sent to IPPIS [practice; FIJ 11 Nov 2024].
- Deductions enter payroll as items "your MDA has submitted to IPPIS for capture" [practice; brands.ng 2026].
- Union of Uyo study: "no clear relationship between the Unions, cooperative societies and IPPIS Office to aid in the recovery of dues and third party loans" [academic; Zenodo 5501037, 2021].
- The variation data that lenders key into WACS is the IPPIS number + amount; one FIJ victim is identified by "IPPIS number 331851" in the complaint, suggesting the IPPIS number is the key [practice; FIJ via snippet, 2025/26].
- For tertiary institutions post-Nov 2024: institutions "prepare their payroll in Excel format and submit it to IPPIS for verification and validation", payment via GIFMIS [published; SaharaReporters 4 Nov 2024; TheCable Oct 2024]. The OAGF "Guidelines" document circulates on Scribd (doc 878796996) but would not load **[snippet]**.

**Gaps.** Column specification, file type (Excel vs portal), cut-off day, start/end-date fields, and who in the MDA signs off — **[not found]**.

---

## 4. Payslip presentation: itemised vs collapsed "WACS" line

**Takeaway.** Cooperative and union deductions appear by name on the IPPIS payslip. Since Sep/Oct 2023, lender deductions have been collapsed into a line labelled "WACS Loan Account" (sometimes "WACS-<lender>"), which removed the lender name that payslips previously showed and left borrowers unable to prove which lender was paid.

**Cited findings.**
- "WACS" = **Workers Aggregated Credit Scheme**; "if a federal government staff… has a loan from an organisation connected to IPPIS then the deductions will be presented on the staff's pay slip as WACS-Bank ABC or WACS-Bank XYZ" [practice; Soluap]. FIJ quotes the IPPIS site calling it "a loan management system… to unify loan collection, management and repayment for federal workers" [published text, 10 Oct 2023]. "Wages Analysis and Control System" is **not** the expansion used by any source found.
- Collapse complaint: deductions "appeared as 'WACS Loan Account' on… September 2023 salary slips", ₦89,200 to ₦100k+, with no prior notice; a worker: "the system we had always highlighted all the third parties"; another: "IPPIS paid Credit Direct, as indicated on my payslip, confirmed by Credit Direct" — but after the switch "payments redirected to a WACS account were not acknowledged by the lender" [practice; FIJ 10 Oct 2023; FIJ 8 Nov 2023].
- IPPIS front-desk response: deductions legitimate; a mobile app to show loan breakdowns was "forthcoming" (Oct 2023) [published statement; FIJ]. No such app located by Oct 2026 **[not found]**.
- Later payslips do show "WACS to Liquid Cash ₦53,720" per month — i.e., lender name reappears in some cases [practice; FIJ via snippet, 2025/26].
- Payslip access itself fails: paramilitary staff said the payslip portal "was shut down" May 2024 and they could not identify causes of "excessive and abnormal" deductions; a Level 10 officer netted ₦25,000 [practice; SaharaReporters 2 Dec 2024].

**Gaps.** Whether WACS is a module inside the payroll (Oracle) system or a separate settlement application is not documented; the FIJ-quoted IPPIS text calls it a "settlement management system" run by the Third-Party Division, which suggests a separate ledger that posts one aggregate line to payroll **[unverified inference]**.

---

## 5. Remittance: when and how funds reach the third party; statements

**Takeaway.** Remittance is a lump-sum bank credit expected in the first week after pay day. No remittance advice/schedule file is described by any cooperative or lender; reconciliation is by the third party's own variation list against the credit received. Remittance is contingent on "cash backing": when the Treasury funds only net pay, deductions are shown on payslips but not remitted.

**Cited findings.**
- Expected timing: "first week of the following month" to the society's account [practice; FIJ 11 Nov 2024]. Lenders: remitted "before the remaining salary balance reaches the employee's account" [practice; Credit Direct] — contradicted by Kaylog: "IPPIS WACS has not remitted the money that they deducted… for November and October" (said Dec 2023) and "once IPPIS remits, we send the money out" [practice; FIJ 8 Feb 2024].
- Cash-backing rule (official): Deko told the Reps NHF committee (Aug 2023) that deductions are often not remitted "because of shortage of cash backing for approvals for wages"; "sometimes they get directives to only pay the net of the workers' salaries, while the gross is kept in view"; "since December 2022 the system paid only net salaries" and in such months "all deductions would be stepped down" [published; Leadership/TheCable/Peoples Daily Aug 2023].
- Pension comparator (works): for IPPIS staff "the OAGF deducts and directly remits their contributions into their respective RSAs"; law requires remittance within 7 working days of salary payment with ≥2% penalty [published; PenCom FAQ 2020]. PenCom's new mandatory PSSP remittance platform (deadline 1 Jun 2025; nine PSSPs: PenPay/Awabah, PayPen/Netline, PenCentral/ChamsAccess, PenSphere, PenRemit/Cyberspace, PenSol/Uniswitch, ENCO/Gemspay, PCOSS/NIBSS, Interswitch) validates PIN+PFA before payment — a worked example of schedule-with-payment that cooperatives lack [published; TheCable, Leadership, Tribune, Apr–May 2025].
- NHF comparator (fails too): FMBN MD Madu Hamman told the Reps ad-hoc committee IPPIS failed to remit ₦11.587bn (Jan–Dec 2022) and ₦3.356bn (Apr–Jul 2021) of 2.5% NHF deductions [published; Daily Times, Aug 2023; Vanguard 2023].
- Statements/reports for third parties: **[not found]**. No cooperative or lender source mentions downloading any IPPIS statement; the NBS Kwara society relied on letters.

**Gaps.** Whether a remittance advice (CSV/Excel) accompanies bank credits; whether WACS gives lenders a portal statement; T+? remittance SLA in the 2023 agreement.

---

## 6. Fees

**Takeaway.** No OAGF/IPPIS fee for deduction processing is published anywhere. **[not found]**

**Cited findings.** Lender pricing to borrowers is the only cost data: 3–4%/month flat (GTI MFB "3% monthly"; Mutual Trust "4% per month fixed") [practice; lender sites]. CREDICORP partner loans ₦30k–₦3m at "reduced" rates [published; TheCable 2024]. Remita direct-debit pricing (~₦50/successful debit) is from the companion note *competitive_landscape.md* and is not an IPPIS fee.

**Gaps.** Whether OAGF deducts any charge from third-party remittances; whether the 2023 Lenders SLA includes fees.

---

## 7. GIFMIS and TMRAS (2025): what changed

**Takeaway.** GIFMIS is the Treasury's payment/accounting system; since Nov 2024 it is also the payment rail for federal tertiary institutions' self-prepared Excel payrolls (still vetted by IPPIS). TMRAS (live 4 Mar 2025) is a revenue-collection and payments front end that absorbs Remita's role as sole gateway; its documented scope is MDA collections and vendor payments with automatic VAT/WHT/stamp duty — **not** salary deductions. Nothing found says TMRAS changed cooperative/lender remittance.

**Cited findings.**
- TMRAS: OAGF memo dated 28 Feb 2025; live 4 Mar 2025; Phase 1 naira collections and payments, automatic tax deductions on vendor/contractor payments; Phase 2 from 1 Jun 2025 (FX, MDA ERP integration, budget-control module for off-budget agencies); Remita "not discarded" but integrated as one of several CBN-licensed PSSPs; government takes over "front-end payment infrastructure previously handled by SystemSpecs" [published; Nairametrics 4 Mar 2025; Guardian; Leadership; Channels]. Phase 2 completion status **[not found]**.
- GIFMIS for tertiary: FEC approved exit (13 Dec 2023); OAGF processed Oct 2024 payroll on IPPIS then shut IPPIS for FTIs; Nov 2024 salaries via GIFMIS from Excel payrolls "submit[ted] to IPPIS for verification and validation" [published; TheCable Oct 2024; SaharaReporters 4 Nov 2024; OAGF spokesman Bawa Mokwa]. ASUU: "GIFMIS is still an appendage of IPPIS" because the payroll still goes to OAGF-IPPIS for vetting [published statement; TheCable/Vanguard Oct 2024].
- Consequences: NAAT (23 Feb 2025): "delays, disparities in salaries, and irregular remittances of statutory and third-party deductions" incl. check-off dues [published statement; PM News]. AGF Shamseldeen Ogunjimi (16 Jun 2025): "hasty and uncoordinated migration"; office "overwhelmed with petitions" from PFAs, State IRSs, "microcredit organisations" and unions over non-remittance since he took office Mar 2025; institutions skipped the inter-ministerial committee's trial payrolls (Nov–Dec 2024, IPPIS-checked, GIFMIS-paid, sign-off 31 Dec 2024); bursar training approved with the Association of Bursars [published; Daily Trust 17 Jun 2025; The Whistler; TheFact].
- ASUU NEC (5 Sep 2026): government has failed to remit "pension contributions, staff cooperative deductions and ASUU check-off dues" running to "billions of naira", a "double financial loss"; strike threat [published statement; Tribune 12 Sep 2026; PoliticsNigeria 11 Sep 2026]. (The brief's "Oct 2026 ASUU complaints" correspond to these Sep 2026 statements; nothing dated Oct 2026 found.)

**Gaps.** Who now remits cooperative deductions for FTIs (bursary via GIFMIS third-party payment vouchers, or IPPIS), and on what timeline — **[not found]**.

---

## 8. Caps, consent and the "one-third" rule

**Takeaway.** No OAGF/IPPIS circular capping third-party deductions as a share of net pay was found. The one-third rule the brief mentions is a **Kenyan** statutory rule (Employment Act 2007, enforced in TSC T-Pay); Nigeria has no federal equivalent in any located document. Lenders size loans to "net pay" by their own policies.

**Cited findings.**
- Kenya TSC T-Pay guidelines: third parties must ensure deductions leave the employee one-third of basic pay; violations lead to suspension of the third party [published; arena.co.ke]. Not Nigerian.
- Nigeria: all located lender guides say only that "licensed lenders assess your monthly income and existing deductions" [practice; Credit Direct 2026]. Evidence of no effective cap: a worker left "with approximately ₦2,000 of monthly salary" after deductions (FIJ 8 Feb 2024); Level 10 paramilitary officers netting ₦25,000 (SaharaReporters Dec 2024); a GL14 officer whose "automatic deductions for cooperative loans, commercial lending facilities, and informal salary advances had swallowed nearly every naira" (Streamlinefeed, 30 Jun 2026) [practice].
- Consent: ASUU's 2019 claim that "IPPIS will not allow for deductions… arising from legally sanctioned union and cooperative society activities" (Ogunyemi, Oct 2019) [published statement] was not borne out — IPPIS does process them; the actual consent failure is the reverse (deductions *without* consent, §9).
- OAGF circulars located (Treasury Circulars page, Oct 2026): "Complaints about Purported Shortfall in Salaries" (6 Mar 2025 — explains Jan–Feb 2025 drops as end of arrears payments and recovery of Dec 2024 overpayments; no deduction cap); "Introduction of OAGF–IPPIS Payroll Validation Portal" (Jun 2024); "IPPIS: No directive for workers to change salary accounts" (Nov 2024); "Mandatory Transfer of Contractors' Retention Fees" (Jun 2024). **No circular on third-party/loan/cooperative deductions** [published list].

**Gaps.** Any internal IPPIS affordability check (e.g., minimum net pay) and any Head-of-Service circular on loan deductions — **[not found]**.

---

## 9. Catalogue of documented failures

| # | Failure | Who / where | Amount | Period | Source (date) | Status |
|---|---|---|---|---|---|---|
| F1 | Lender lines collapsed into one "WACS Loan Account" line; no prior notice; amounts exceed scheduled repayment | Federal workers nationwide | ₦12,282–₦129,650 per person | Sep–Oct 2023 payslips | FIJ 7 Oct, 10 Oct, 8 Nov 2023 | IPPIS: "legitimate"; app promised |
| F2 | Deduction for a loan never taken ("I have never borrowed from this so-called WACS") | Ondo, Lagos workers | ₦12,282; ₦95,195 | Sep 2023 | FIJ 7 Oct 2023 | No IPPIS response |
| F3 | Double deduction of identical loan instalment | ≥8–10 staff of UCH Ibadan, NOH Kano, ABUTH Zaria, FMC Yenagoa, UCTH Calabar; mostly Credit Direct borrowers; Kaylog named | e.g., 2 × ₦29,030; ₦10,625 | Jan 2024 salaries | FIJ 8 Feb 2024 | Credit Direct denies; Kaylog blames IPPIS |
| F4 | Deducted but not remitted to lender | Kaylog Business Support | n/d | Oct–Nov 2023 | FIJ 8 Feb 2024 | Unresolved at publication |
| F5 | Deduction to unknown/dormant MFB (Crevance Credit Ltd) for non-existent loan; commenter recovered 2 of 6 months | National Theatre staff; others | ₦12,000; 6 × ₦32,393 | Oct 2024 | FIJ 21 Nov 2024 | Lender untraceable at listed addresses |
| F6 | Deductions continue after payoff, even with lender's letter of non-indebtedness forwarded to WACS; lender says Apr–Jul 2025 instalments never remitted | Rhoda Bello (Liquid Cash); Taiwo Akp | ₦53,720/month | 2023; Oct–Dec 2025 | FIJ 7 Oct 2023; FIJ 2025/26 [snippet] | Unresolved |
| F7 | Cooperative deductions withheld | NBS Kwara Staff Multipurpose Co-op | ₦4.39–4.42m | Dec 2021 deduction, unremitted since Jan 2022 (also Nov 2020 incident) | FIJ 11 Nov 2024 | No IPPIS contact reachable; members denied loans |
| F8 | ₦12bn deducted from civil servants, not remitted to lenders | Lenders Association (HDEO) members | ₦12bn+ | cumulative to Nov 2024 | Daily Trust 29 Nov 2024; Vanguard 1 Dec 2024 | Outstanding; SLA (2023) not enforced; no 2025–26 update found |
| F9 | NHF 2.5% deductions not remitted | FMBN | ₦11.587bn (2022) + ₦3.356bn (Apr–Jul 2021) | 2021–22 | Daily Times / Vanguard Aug 2023 | Reps probe; OAGF summoned |
| F10 | Net-only payroll: all deductions "stepped down" when cash backing short | All IPPIS MDAs | n/d | from Dec 2022 (Deko) | Leadership/TheCable Aug 2023 | Structural; no public rule |
| F11 | Cooperative loan deduction routed to a stranger ("one Bisi in Kano State"); cooperatives "shortchanged… it cannot be removed from a lot of people" | FUT Minna SSANU; BUK | n/d | 2020–21 | Daily Trust Oct 2021 | — |
| F12 | Members using IPPIS confusion to default on cooperative loans | FUNAAB cooperative | n/d | Aug 2021 | FUNAAB news 12 Aug 2021 | VC warning |
| F13 | Tertiary exit to Excel/GIFMIS: delayed salaries, irregular remittance of check-off, cooperative, pension, tax | 100+ FTIs; NAAT, SSANU, ASUU | "billions" (ASUU) | Nov 2024 → Sep 2026 | PM News 23 Feb 2025; Daily Trust 17 Jun 2025; Tribune 12 Sep 2026 | AGF blames "hasty" migration; unresolved |
| F14 | Paramilitary: "excessive and abnormal" deductions, payslip portal down | NSCDC, NIS, NCoS, FFS | Level 10 net ₦25,000 | May–Dec 2024 | SaharaReporters 2 Dec 2024; Daily Trust 30 Nov 2024 | OAGF: deductions "normal" (pension, PAYE, dues) |
| F15 | Police: non-remittance of loans to banks/cooperatives, union dues; unexplained deductions; pay disparity | NPF (on IPPIS; 8 commands had errors Aug–Sep 2017) | n/d | 2017–2024 | PRNigeria 2016; Blueprint 14 Oct 2022; OpinionNigeria 16 Aug 2024 | IGP: "beyond [police] influence" |
| F16 | Overpayments by "system error" then clawed back by deduction | All MDAs | n/d | Dec 2024 → 2025 | OAGF circular 6 Mar 2025 | Ongoing recovery |
| F17 | Payroll fraud via fake identities; ₦941.9m forfeited; 909 accounts in 17 banks incl. NPF MFB | Security-sector-linked suspects | ₦941.9m | order 13 Jul 2026 | Premium Times opinion Jul 2026 | Shows privileged-access risk |
| F18 | Validation portal / staff data "hijacked" (domain) | OAGF payroll validation | — | Sep 2024 | FIJ; ITEdgeNews | OAGF denied hack |
| F19 | LUTH new hires unpaid Jan–Jul then told to refund "overpayment" | LUTH | n/d | 2024 | FIJ 19 Dec 2024 | Suspected misuse of staff accounts |

**Not found:** any formal "suspension of third-party deductions" directive 2020–2023 (only the de-facto net-only months, F10). No NICN judgment specifically on IPPIS third-party deductions located; nearest are *Offoh v. IMT Enugu* (16 Jan 2023, Arowosegbe J.: arbitrary half-salary deductions "a brazen violation of the 1999 Constitution", ₦5m exemplary damages — not an IPPIS case) and the ASUU/IPPIS autonomy ruling (Kanyip PNICN, 2023: FG cannot force universities onto IPPIS) [published; nicnadr.gov.ng].

---

## 10. Other federal payrolls: military, police, self-funded agencies

**Takeaway.** Police and paramilitary are on IPPIS (police verification from 2016, pay errors in 8 commands Aug–Sep 2017); military enrolment began 30 Jul 2018 and current guides say military pay is IPPIS-administered, though 2020 reporting had the Army Finance Department handling pay — status **[unverified]**. CBN, FIRS (now NRS), NNPC and other self-funded agencies run their own payrolls; how cooperatives check off there is **[not found]**.

**Cited findings.** PRNigeria 24 May 2016 (police verification for IPPIS); Blueprint 14 Oct 2022 (IGP: salary computation "rests squarely on… IPPIS"); PM News 17 Jul 2018 (military enrolment from 30 Jul 2018); Vanguard Feb 2020 (Army rectified soldiers' pay discrepancies itself) ; Channels 4 Dec 2019 (CBN/FIRS/NNPC excluded). NPF Microfinance Bank exists as the police-linked lender (Wikipedia) — its deduction channel not documented.

---

## 11. Published vs. practice — summary table

| Element | Published (official) | Practice (participant accounts) |
|---|---|---|
| Operator | OAGF IPPIS Dept; Third-Party Division; call centre 0700-2754774 | Phones unanswered; letters only (FIJ) |
| Onboarding | None published | Cooperative sends member list to IPPIS/MDA; lenders via WACS + 2023 SLA |
| Fees | None published | None reported |
| Schedule format | None published (FTIs: "Excel" to IPPIS for vetting) | Monthly "variation": name, IPPIS no., amount, months |
| Payslip | "appropriate deductions" | Cooperatives by name; lenders as "WACS Loan"/"WACS-<lender>" |
| Remittance | Pension within 7 working days (PenCom law); NHF within 1 month (NHF Act) | Cooperative: first week after pay day, lump sum, no schedule; lenders: months late or never |
| Cap/consent | None | None effective; net pay as low as ₦2,000–₦25,000 |
| Statements | None | None |
| Failure recourse | Call centre; OAGF Garki II | FIJ, unions, Reps committees, lenders' association |

---

## Sources (all accessed 8 Oct 2026)

Official
- https://oagf.gov.ng/initiative/ippis/
- https://oagf.gov.ng/publications/treasury-circulars/
- https://oagf.gov.ng/wp-content/uploads/2025/03/TREASURY-CIRCULAR.-COMPLAINTS-ABOUT-PORPORTED-SHORTFALL-IN-SALARIES.pdf (text extracted locally)
- https://oagf.gov.ng/wp-content/uploads/2024/11/IPPIS-NO-DIRECTIVE-FOR-WORKERS-TO-CHANGE-SALARY-ACCOUNTS-OAGF.pdf
- https://www.pencom.gov.ng/wp-content/uploads/2020/06/FREQUENTLY-ASK-QUESTION-2020-ENGLISH-MAIN-VISUAL.pdf
- https://nicnadr.gov.ng/news/1597/you-cannot-impose-ippis-on-asuu-industrial-court-tells-fg
- https://www.nicnadr.gov.ng/news/1521/industrial-court-declares-unauthorized-deduction-from-offoh-s-salary-awards-n5m-damages-against-enugu-institute-of-technology

FIJ series
- https://fij.ng/article/through-ippis-wacs-loan-debiting-nigerian-workers-for-unknown-reasons/ (7 Oct 2023)
- https://fij.ng/article/more-civil-servants-complain-about-wacs-deduction-but-ippis-says-its-loan-management/ (10 Oct 2023)
- https://fij.ng/article/ippis-deducts-civil-servants-salaries-for-mysterious-loan-repayment/ (8 Nov 2023)
- https://fij.ng/article/ippis-wrongfully-deducting-loans-from-workers-salaries-but-not-remitting-them/ (8 Feb 2024)
- https://fij.ng/article/ippis-withholds-nbs-kwara-staff-cooperatives-n4m-since-2022/ (11 Nov 2024)
- https://fij.ng/article/ippis-lets-unknown-crevance-credit-deduct-non-existent-loan-from-civil-servants-salary/ (21 Nov 2024)
- https://fij.ng/article/staff-suspect-luth-uses-their-accounts-to-steal-public-funds-through-ippis/ (19 Dec 2024)
- https://fij.ng/article/revealed-ippis-payroll-validation-fg-workers-personal-data-on-oagf-website-have-been-hijacked/

Press
- https://www.vanguardngr.com/2024/12/remit-n12bn-loans-deducted-from-workers-salaries-creditors-urge-fg/
- https://dailytrust.com/n6-2bn-loan-disbursed-to-2400-civil-servants-in-2yrs-lenders/
- https://dailytrust.com/ippis-deductions-civil-defence-ncos-personnel-set-for-protest/
- https://dailytrust.com/agf-explains-salary-crisis-in-tertiary-institutions/
- https://dailytrust.com/how-ippis-breeds-irregularities-uncertainty-in-varsities/
- https://saharareporters.com/2024/11/04/nigerian-government-removes-universities-other-tertiary-institutions-ippis-system
- https://saharareporters.com/2024/12/02/nigerian-paramilitary-workers-decry-excessive-deductions-salaries-inability-access
- https://www.thecable.ng/fg-to-remove-tertiary-institutions-from-ippis-payroll-platform/
- https://www.thecable.ng/non-remittance-of-workers-contributions-to-nhf-due-to-shortage-of-cashbacking-says-ippis-boss
- https://leadership.ng/ippis-officials-dont-pay-workers-salaries-we-only-process-director/
- https://dailytimesng.com/intrigues-as-reps-launch-probe-into-national-housing-fund/
- https://nairametrics.com/2025/03/04/remita-will-remain-operational-in-nigeria-to-be-integrated-into-tmras-fg/
- https://thewhistler.ng/agf-blames-tertiary-institutions-salary-delay-on-poor-gifmis-network/
- https://thefact.ng/gifmis-fg-to-train-tertiary-institutions-bursars-on-payroll-processing-payments/
- https://pmnewsnigeria.com/2025/02/23/naat-demands-immediate-payment-of-withheld-salaries/
- https://tribuneonlineng.com/economic-hardship-worsening-poverty-eroding-workers-wages-asuu/ (12 Sep 2026)
- https://politicsnigeria.com/2026/09/11/fresh-strike-looms-as-asuu-issues-final-ultimatum-to-fg-states/
- https://www.vanguardngr.com/2024/03/ippis-exit-of-varsity-based-unions-and-the-sudden-silence/
- https://www.pulse.ng/story/fg-digital-ippis-hr-platform-2026072815030924155 (28 Jul 2026)
- https://www.pulse.ng/story/what-is-ippis-and-why-is-asuu-about-to-strike-because-of-it-pulse-explains-2024081416344414011
- https://newsverge.com/2022/06/24/ippis-trims-federal-workers-to-720000-saves-n220bn-fg/
- https://www.channelstv.com/2019/12/04/video-why-cbn-firs-nnpc-are-excluded-from-ippis/
- https://pmnewsnigeria.com/2018/07/17/nigerian-govt-begins-enrolment-of-military-personnel-on-ippis/
- https://blueprint.ng/agitations-ippis-responsible-for-salary-disparity-igp-investigation/
- https://www.opinionnigeria.com/blame-the-ippis-and-not-the-nigeria-police-by-adewole-kehinde/
- https://www.thecable.ng/credicorp-500k-civil-servants-to-benefit-from-n100bn-consumer-credit-scheme/
- https://www.vanguardngr.com/2024/10/consumer-credit-credicorp-disburses-loan-to-10942-civil-servants/
- https://www.thecable.ng/pencom-issues-june-1-deadline-for-employers-to-adopt-new-pension-remittance-system/
- https://www.premiumtimesng.com/opinion/897149-ghost-workers-ippis-fraud-and-lessons-from-the-%E2%82%A6941m-forfeiture-by-nafisat-bello.html
- https://streamlinefeed.co.ke/news/abuja-civil-servants-ippis-debt-trap-inflation (30 Jun 2026)

Practitioner / lender / academic
- https://soluap.com/wacs/ ; https://soluap.com/when-do-ippis-system-pay-salary-and-why-do-some-people-dont-get-their-salaries/
- https://brands.ng/blog/ippis-payslip-2026-how-to-download-print-and-verify/
- https://www.creditdirect.ng/blog/payday-loans-nigeria
- https://veendhq.com/products/remita-payroll-loan ; https://moniestack.com/how-to-get-remita-loans-as-federal-and-state-civil-servants/
- https://alpha-vistainvestment.com/service/federal-civil-servant-loan/ ; https://research.gti.com.ng/ippis-loan-for-federal-workers/ ; https://mutualtrustmfb.com/ippis-loan/
- https://funaab.edu.ng/repay-your-loans-stop-hiding-under-ippis-deductions-vc-charges-cooperators/
- https://zenodo.org/records/5501037 (Univ. of Uyo unions/cooperatives & IPPIS, 2021)
- https://www.scirp.org/journal/paperinformation?paperid=113850 (Gains and Pains of IPPIS)
- https://journalsglobal.com/index.php/jgeb/article/download/138/147 (IPPIS in Gombe State, Apr 2023 — state-level, limited relevance)
- https://arena.co.ke/tsc-t-pay-guidelines-for-all-third-party-deductions-revised/ (Kenya one-third rule comparator)
