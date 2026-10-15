# Third-party (cooperative and loan) deductions outside federal IPPIS: Remita, state/LG payrolls, private-sector payroll software, and the bank rails that move the money

Compiled 2026-10-08 from public web sources. Every finding carries a date and a URL. Items marked **[UNVERIFIED]** were seen only in secondary or aggregator sources, or could not be re-fetched. Items marked **[CONFIRM WITH VENDOR]** must be checked directly before being relied on in product or commercial decisions. Several primary pages (remita.net/pricing, support.remita.net KB, api.remita.net, devs.remita.net, lagosstate.gov.ng civil-service page, medium.com/bento) blocked fetching (HTTP 402/403/522) during this research; their content is cited from search snippets and flagged.

Scope note: federal IPPIS mechanics are covered elsewhere; this note deliberately covers everything *outside* IPPIS, but cites IPPIS incidents where they illuminate the same failure modes.

---

## 1. Remita (SystemSpecs) payroll and loan-deduction rails

### Takeaway
Remita is three distinct rails that are often conflated: (a) **Remita Payroll & HR** (the employer's payroll engine, which computes cooperative/union/loan deductions and pays salaries); (b) **Remita "Inflight Collections" / payroll lending** (a lender posts a *Loan Disbursement Notification* via API and Remita intercepts the repayment from any salary it processes before the salary lands — this is what the market calls "deduction at source" or "salary domiciliation" and is the engine behind *347*780#-type payday loans); and (c) **Remita Direct Debit** (a NIBSS/CBN direct-debit mandate on the borrower's bank account, independent of payroll). A cooperative can realistically plug into (b) only if it is a licensed lender accepted by Remita, and into (c) as any biller. Remita's federal role shrank with TMRAS in March 2025, but TMRAS is a revenue/collections platform; payroll-side deductions remain an IPPIS/OAGF matter, and Remita's state and private-employer payroll footprint is unaffected by TMRAS.

### Cited Findings

**Products and who uses them**
- Remita Payroll & HR "computes all staff allowances, loans, pensions, taxes, overtime allowances, cooperatives, union and other deductions" (Remita support KB, undated; page returned HTTP 402 on fetch, text from search snippet) — https://support.remita.net/portal/en/kb/articles/remita-payroll **[UNVERIFIED — page blocked]**
- Remita's own timeline states Kano State was "the first state government to adopt Remita Payroll and Payment System" in **2011** (remita.net/about-us, fetched via search snippet; page returned 402) — https://www.remita.net/about-us **[UNVERIFIED]**
- Benue State engaged Remita for salary payment; Vanguard, March 2018: "SystemSpecs' Remita cleaned up fraud in Benue's Payroll — Commissioner" — https://www.vanguardngr.com/2018/03/957312/
- BusinessDay, 27 Sep 2019: "over 2 million salaries process monthly through the platform"; lender partners named: Access Bank, Carbon, PettyCash, Coolbucks, Ibile MFB, Arvo Finance, Credit Wallet, PayDayhub Online, Free Cash Solution, CIT MFB, Devonsley, Happy Partners, Bridge Credit, Motion Yield, Consumer MFB — https://businessday.ng/financial-inclusion/article/payday-loans-how-remita-is-stimulating-consumer-sme-credit/
- Secondary aggregators list Remita payroll loans as available in Benue, Nasarawa and Kano (webguru.com.ng, moniestack.com) — https://www.webguru.com.ng/remita-loans **[UNVERIFIED — affiliate sites]**

**Lender onboarding and instruction mechanics (API)**
- Remita's API documentation (api.remita.net, via search snippets; the page body did not render on fetch) exposes an **Inflight Collections** family with three services: *Loan Disbursement Notification*, *Loan Repayment History*, *Stop Loan Collection*. Loan Disbursement Notification fields: `customerId, authorisationCode, authorisationChannel, phoneNumber, accountNumber, currency, loanAmount, collectionAmount, dateOfDisbursement, dateOfCollection, totalCollectionAmount, numberOfRepayments, bankCode`. Loan Repayment History and Stop Loan Collection take `authorisationCode, mandateRef/mandateReference, customerId`. A separate **Reference Data / Salary History** service returns salary history by phone number or account number (params: authorization code, phone/account, bank code, authorization channel) — https://api.remita.net/ and https://devs.remita.net/ **[CONFIRM WITH VENDOR — field list from snippets, not a fetched spec]**
- SystemSpecs (David Okeme, BusinessDay 2019): "With our APIs, as a lender, you have details such as customer's net salary, payment history, employment history" and "Upon notification of a loan disbursement, Remita would debit the amount from the customer's next salary. Loan repayment is assured as long as the customer is still being paid a salary" — https://businessday.ng/financial-inclusion/article/payday-loans-how-remita-is-stimulating-consumer-sme-credit/
- Onboarding as described by Lendsqr (a loan-management SaaS that fronts Remita for small lenders): lender must "create a Remita profile", "meet country requirements and maintain current lending license", then select Remita/direct debit as collection method per loan product; Lendsqr warns lenders to "take note of the fees imposed by Remita" without publishing them — https://blog.lendsqr.com/how-to-use-remita-for-your-loan-repayments-on-lendsqr/
- Access Bank PayDay Loan uses Remita data referencing; short code *901*11# (Access) and Remita's generic *347*780# (operated by VeendHQ for multiple lenders) — https://www.accessbankplc.com/help/frequently-asked-questions/payday-loan ; https://dailytrust.com/access-bank-introduces-short-code-for-payday-loan-product/

**USSD *347*780# flow (VeendHQ)**
- Steps: dial *347*780# from the phone linked to the salary account; enter BVN, salary-account phone number, first three letters of bank name; "If you are a Federal Government worker, enter your IPPIS number, or if you are not a Federal Government worker, simply enter 0"; disbursement is instant; "the loan amount will be deducted at source from your next salary payment ... before the salary hits your account" — https://newaccount1641744218590.freshdesk.com/support/solutions/articles/101000577889-how-to-apply-for-a-loan-via-ussd-code-347-780-
- VeendHQ product page: targets "individuals under the remita payroll ecosystem"; up to ₦1m, 12-month plan, disbursement "within 2 minutes"; ineligibility if "not within the ecosystem we currently serve" — https://veendhq.com/products/remita-payroll-loan

**Remita Direct Debit (bank-account mandate, not payroll)**
- "Remita Direct Debit is an end-to-end solution that empowers bank and non-bank lenders to automatically receive loan repayment from any commercial or microfinance bank account of borrowers on the due date" (Techeconomy; page returned 403 on fetch, text from snippet) — https://techeconomy.ng/remita-direct-debit-and-its-impact-on-nigerian-economy **[UNVERIFIED]**
- Lendsqr FAQ (undated): Remita "cycles through three to four attempts on the due date"; settlement "typically ... T+1"; "Remita charges a small processing fee for each successful loan repayment transaction. Typically, this fee is a fixed amount (e.g., ₦50) or a nominal percentage" — https://blog.lendsqr.com/frequently-asked-questions-on-loans-powered-by-remita/ **[figure is illustrative in source; CONFIRM WITH VENDOR]**
- The underlying scheme rules are CBN's *Guidelines on Nigeria Direct Debit Scheme* (2010) — https://www.cbn.gov.ng/out/2010/circulars/bod/guidelines%20on%20nigeria%20direct%20debit%20scheme.pdf

**Fees (published or reported)**
- Remita Collections: 2.0% of value, max ₦2,500 per transaction; educational institutions ₦300 flat (brands.ng summary of Remita pricing, 2026; nuggetsmfb.com reproduces the same) — https://brands.ng/blog/remita-pay-a-biller-2026/ ; https://nuggetsmfb.com/credit_card/remita-collections/ **[UNVERIFIED — remita.net/pricing returned 402]**
- Remita Payroll: ₦500 per employee one-off setup, ₦250 per payroll transaction (search snippet attributed to Remita pricing) **[UNVERIFIED; CONFIRM WITH VENDOR]**
- Historical TSA commission: 1% shared SystemSpecs:banks:CBN 50:40:10 (Vanguard, Nov 2015) — https://www.vanguardngr.com/2015/11/tsa-1-transaction-charges-is-revenue-for-remita-cbn-banks/
- No published Remita fee schedule for *Inflight Collections* / payroll-lending was found. Lendsqr explicitly declines to publish it.

**TMRAS (March 2025) and what it changes**
- TMRAS went live 4 March 2025 as the FG's collections/payments platform; Phase 1 covered naira collections, automatic tax deductions on vendor payments, bank statements for MDAs; Phase 2 (1 June 2025) adds FX, ERP integration and budget module (Nairametrics, Channels, Regtech Africa, 4 Mar 2025) — https://nairametrics.com/2025/03/04/fg-dumps-remita-unveils-new-revenue-collection-platform-on-march-4/ ; https://www.channelstv.com/2025/03/04/fg-launches-new-revenue-platform-says-remita-will-be-integrated/ ; https://regtechafrica.com/nigeria-to-roll-out-treasury-management-revenue-assurance-system-tmras-for-enhanced-revenue-collection/
- OAGF: "Remita not ditched, to be integrated into TMRAS" alongside other PSSPs (Daily Trust, Tribune, Blueprint, Mar 2025) — https://dailytrust.com/remita-not-ditched-to-be-integrated-into-tmras-oagf/ ; https://tribuneonlineng.com/remitta-not-ditched-to-be-integrated-into-tmras-fg/
- The "third-party deductions" TMRAS automates are *tax* deductions on vendor payments (VAT, WHT, stamp duty), not payroll check-offs. Nothing in the TMRAS coverage touches cooperative or loan deductions on salaries; those remain in IPPIS/OAGF. Implication for Remita payroll lending: federal salaries never ran through Remita Payroll (they run through IPPIS and are paid via GIFMIS/CBN), so TMRAS does not alter the state/private salary base Remita's inflight collection relies on. **[Analyst inference — confirm with SystemSpecs whether Remita still receives any federal salary flow usable for inflight collection post-TMRAS.]**

**Documented failures on the IPPIS side (for contrast)**
- Lenders Association (HDEO: banks, MFBs, finance companies) said ₦12bn deducted from federal workers' payslips via IPPIS had not been credited to lenders; an SLA was signed 2023 (Vanguard, 1 Dec 2024) — https://www.vanguardngr.com/2024/12/remit-n12bn-loans-deducted-from-workers-salaries-creditors-urge-fg/

### Gaps
- No fetched primary spec for Inflight Collections (auth scheme — merchantId/apiKey/SHA-512 hash is the Remita norm for other APIs — sandbox, webhook callbacks, settlement account rules). **[CONFIRM WITH VENDOR]**
- Which employers' salaries are actually "in the ecosystem" today (states, parastatals, private firms) — Remita publishes no list. Kano (2011), Benue (2018) are the only states confirmed by press.
- Whether a *cooperative society* (not a CBN/state-licensed money lender) can be onboarded to Inflight Collections at all; VeendHQ's eligibility language implies lenders only. **[CONFIRM WITH VENDOR]**
- Fee to lenders for inflight collection and for Remita Direct Debit (fixed vs % of repayment). **[CONFIRM WITH VENDOR]**
- Payslip presentation: whether an inflight-collected repayment appears on the employee's payslip at all (by design it is intercepted *after* payroll computes net pay, so it will typically appear only on the bank statement, not the payslip). **[UNVERIFIED]**

---

## 2. State and local-government payrolls

### Takeaway
Every state runs its own payroll (there is no state IPPIS); the dominant systems are Oracle-based (Lagos), HRMIS/biometric builds funded by PFM reforms (Kaduna), Remita Payroll (Kano since 2011, Benue), and various in-house or vendor systems elsewhere. Cooperatives and lenders get on the schedule through the Office of the Head of Service / Accountant-General (circulars govern suspension and resumption, e.g. Niger's 17 Mar 2025 circular), and the state remits — or does not. Non-remittance is chronic and well documented (Ogun 17–21 months 2019–22; Oyo Water ₦200m 2021; Oyo state-wide 6+ months 2023 then a decision to **stop deducting for cooperatives at source**; Kwara Water ₦40.2m 2015; Kaduna ₦150bn hidden liabilities including cooperative dues; Edo LGs 2024; Kano ₦1.8bn *over*-deduction by lenders and a 3-month suspension in 2025, then withdrawal as third party in 2026). The policy direction in at least two large states (Oyo 2023, Kano 2026) is for government to *exit* the third-party role entirely, which pushes cooperatives toward direct debit.

### Cited Findings

**Platforms by state**
- **Lagos**: Civil Service Commission page describes an HRMIS division that handles "creation of new employees ... on the Oracle Payroll" and "termination of all exits from the Oracle HR/payroll Database" at Alausa — https://civilservice.lagosstate.gov.ng/responsibilities-2/ (page returned 522 on direct fetch; text from snippet) **[UNVERIFIED]**. Lagos payslips circulate publicly showing itemised deductions — https://www.scribd.com/document/598441011/
- **Kaduna**: PEFA PFM assessment (Feb 2012) notes "a new computerised personnel records system on the new HRMIS system, which when completed will be linked to payroll" and biometric capture extended to LGs — https://www.pefa.org/sites/pefa/files/assessments/reports/NG-Kaduna-Feb12-PFMPR-SN-Public.pdf (PDF fetched as binary; summary from snippet). No vendor named. **[Vendor UNVERIFIED]**
- **Kano**: Remita Payroll since 2011 (see §1). In 2025 the state "integrated workers' NIN, BVN and biodata through a technological application to track deductions" (BusinessDay, 29 Apr 2025) — https://businessday.ng/news/article/21000-kano-civil-servants-indebted-to-private-loan-firms/
- **Benue**: Remita (Vanguard 2018).
- **Ogun, Oyo, Rivers, Enugu**: no public source identifying the payroll software was found. **[GAP]**

**How cooperatives/lenders get on the schedule**
- Niger State: "On March 17, 2025, the Niger State Government, through the Office of the Head of Service, issued a circular stating that third-party deductions would resume in May 2025"; deductions covered CPS (raised 7.5%→10%), cooperative repayments and "Sharp Sharp loans" (SaharaReporters, 28 May 2025; Nigerian Daily Post, 28 May 2025) — https://saharareporters.com/2025/05/28/niger-state-workers-lament-10-deductions-may-salary-third-party-charges ; https://nigeriadailypost.com.ng/2025/05/28/niger-state-organized-labour-addresses-salary-deductions-for-civil-servants/
- Kano: lenders' loans were deducted through the state payroll against a "one-third salary deduction limit approved by law"; audit found "vendors granted multiple loans against the one-third salary deduction limit" (Daily Trust, 8 May 2026) — https://dailytrust.com/kano-moves-to-recover-n1-8bn-illegally-deducted-from-workers-salaries/
- No state publishes a "deduction code" application procedure online; the mechanics (HoS/AG approval letter, cooperative registration certificate, member consent forms, monthly schedule upload by the cooperative) are described only anecdotally. **[GAP — confirm with 2–3 state Treasury/HoS offices]**

**Documented non-remittance and suspensions**
- **Ogun**: CDWR, 17 Aug 2021: 17 months of cooperative deductions unremitted to "Ogun State Salary Earners' Co-operative Thrift and Credit Union Limited", "running into billions", accumulating since 2019 — https://www.socialistnigeria.org/5450/2021/08/17/ogun-workers-cdwr-demands-immediate-payments-of-all-outstanding-salaries-and-deductions-to-workers-and-reinstatement-of-all-victimized-workers/ ; ThisDay, 28 Jun 2022: indefinite strike over "21 months' salary deductions, including cooperative deductions" — https://www.pressreader.com/nigeria/thisday/20220628/282183654738790 ; July 2025 strike primarily over pensions (only 34 of 204 months remitted to PFAs) — https://saharareporters.com/2025/07/15/nlc-organised-labour-direct-all-ogun-state-workers-begin-indefinite-strike-over-pensions
- **Oyo Water Corporation**: 15 Feb 2021 protest over ~₦200m (₦122m as at Sep 2020) unremitted cooperative, car-loan, housing-loan and bonus deductions (Tribune) — https://tribuneonlineng.com/oyo-workers-protest-govt-failure-to-remit-n200-million-cooperative-deductions/
- **Oyo state-wide 2023**: July–Aug 2023 NLC/TUC protests over 6+ months of withheld cooperative, housing and union deductions (Vanguard, Jul/Aug 2023) — https://www.vanguardngr.com/2023/07/update-oyo-workers-barricade-govt-secretariat-over-deductions-palliatives-allowances/ ; Governor Makinde broadcast, 5 Aug 2023: two months' cooperative deductions to be paid with August salary and "Going forward ... the Oyo State Government will no longer be making any deductions at source for these cooperatives" — https://seyimakinde.com/governor-seyi-makindes-broadcast/ ; ₦1.3bn released for June–Aug deductions, ~71,000 workers (Starconnect, 1 Sep 2023) — https://starconnectmedia.com/oyo-govt-clears-n1-3-bn-workers-deduction-arrears/
- **Kwara Water Corporation**: AUPCTRE strike, 27 Jan 2016: ₦40.2m cooperative dues (Feb–Jul 2015), union dues (Nov 2014–Dec 2015) and NHF deductions unremitted — https://www.bellanaija.com/2016/01/kwara-state-water-corporation-workers-begin-indefinite-strike/ (the ₦70m figure in the brief was not found; ₦40.2m is the sourced figure)
- **Kano**: Feb 2025 complaints → audit → 29 Apr 2025: 21,000 of 61,000 workers indebted; "suspend loan deductions for the next three months while these debts are verified"; statutory deductions (pensions, NHIS, union dues) continued (BusinessDay) — https://businessday.ng/news/article/21000-kano-civil-servants-indebted-to-private-loan-firms/ ; 8 May 2026: ₦1.8bn found illegally deducted, ₦13bn total loan liabilities, 12-member recovery committee, government "will no longer serve as a third party in such contractual arrangements" (Daily Trust, Ripples, PM News) — https://dailytrust.com/kano-moves-to-recover-n1-8bn-illegally-deducted-from-workers-salaries/ ; https://www.ripplesnigeria.com/kano-govt-fumes-as-loan-vendors-deduct-n1-8bn-from-civil-servants-salaries/ ; https://pmnewsnigeria.com/2026/05/08/kano-govt-inaugurates-committee-to-recover-alleged-n1-8bn-illegal-salary-deduction/
- **Kaduna**: ASUSS says teachers' check-off (up to ₦2,000/month, ~10,000 members) has been diverted to NUT for ~3 years (NAN, 6 Oct 2026) — https://nannews.ng/asuss-seeks-gov-sanis-intervention-over-alledge-diversion-of-union-dues/ ; a forensic audit cited ₦150bn hidden liabilities including "unremitted cooperative and union dues" (search snippet, naijametro.com.ng, Aug 2026) **[UNVERIFIED — partisan source]**
- **Edo LGs**: NULGE, 14 Oct 2024: salary arrears 2017–2023 and "unremitted statutory deductions ... several millions of naira" across 18 councils (Vanguard) — https://www.vanguardngr.com/2024/10/nulge-threatens-strike-over-edo-lg-workers-salary-arrears-deductions/
- **Delta LG retirees / ALGON**: Sep 2024 petition alleging ALGON changed the gratuity computation template, cutting ₦2m–₦8m per retiree (SaharaReporters) — https://saharareporters.com/2024/09/05/delta-state-retirees-cry-out-accuse-algon-fraudulent-n8million-deductions-gratuities (gratuity, not cooperative deduction, but illustrates ALGON's payroll leverage)
- **Universities**: ASUU (Sep 2026) says pension, cooperative and check-off deductions "had not been appropriately remitted for several months", "billions of naira", for federal (IPPIS) and some state universities — https://www.premiumtimesng.com/news/top-news/908790-asuu-threatens-nationwide-strike-over-unpaid-salaries-agreement-implementation.html ; https://nairametrics.com/2026/09/11/asuu-threatens-strike-over-unpaid-salaries-2025-agreement-implementation/

**Local government**
- LG allocations flow through the State Joint Local Government Account (JAAC); despite the July 2024 Supreme Court autonomy ruling allocations were still paid to state accounts (ALGON/NULGE objections) — https://www.reubenabati.com.ng/news/council-autonomy-algon-nulge-kick-as-govs-get-n2tn-lg-allocations ; Labour asked FG not to route LG workers' salaries to councils (Vanguard, Oct 2024) — https://www.vanguardngr.com/2024/10/autonomy-labour-begs-fg-not-to-give-workers-salary-to-lgs/
- Premium Times (2017): 23 states owed LG workers, some 14 months — https://www.premiumtimesng.com/news/headlines/235150-23-nigerian-states-owe-local-govt-workers-salaries-14-months-nulge.html
- No source describing a standard LG cooperative-deduction process was found; in practice LG payroll is run at state level (Local Government Service Commission / ALGON-engaged consultants, e.g. Delta's B.A. Adigun & Associates 2011–2019 and Logic Forensic Consultant). **[GAP]**

### Gaps
- Payroll vendor/platform for Ogun, Oyo, Rivers, Enugu, Kaduna (current), Lagos (confirm Oracle EBS vs Fusion). **[CONFIRM with state Treasury/HoS]**
- Formal onboarding steps and documents for a cooperative "deduction code" in any state; whether states charge an administrative fee (none found published).
- Remittance *evidence* practices (do states send schedules with the transfer? by email? portal?). Nothing published.
- Whether Kano's "withdrawal as third party" (May 2026) also covers cooperative societies or only private loan vendors. Reports say "private loan vendors"; cooperatives not mentioned. **[CONFIRM]**

---

## 3. Private-sector payroll software (banks, oil, telcos, universities, SMEs)

### Takeaway
Private employers configure cooperative/union/loan check-offs as *voluntary deductions* inside their HRIS/payroll (SAP, Oracle, Sage, HumanManager, SeamlessHR, PaidHR, Bento, Workpay). All of these support per-employee recurring deductions and payslip line items; none of the Nigerian vendors publishes a standard "third-party deduction export" or a public API for an external cooperative to push a schedule or pull remittance data. Remittance to the cooperative is a separate bulk transfer from the employer's bank. Bento and SeamlessHR embed *their own* lending partners into payroll (deduct-at-source for fintech loans), which is the model closest to what a cooperative platform would want to replicate — but it is vendor-controlled, not an open integration.

### Cited Findings
- **SeamlessHR**: supports "setup of loan deductions on the system"; salary deduction management covers statutory, voluntary (health, savings, union dues) and "financial services like salary advances or loans"; embedded finance "in partnership with fintech providers ... repayment is automatically deducted from their monthly salary" — https://seamlesshr.com/blog/glossary/salary-deduction-management/ ; https://seamlesshr.com/blog/glossary/employee-loans/ . The support article on deductions covers only *mandatory* deductions (ITF, pension, NHF, NSITF) — https://support.seamlesshr.com/mandatory-and-voluntary-payroll-deductions . No public API doc for third-party deductions found.
- **PaidHR** (ex-Pade): "implementing any number of deductions before processing payroll ... automatic remittance of taxes, pensions, and other contributions"; NHF/NSITF remittance added — https://www.paidhr.com/blog/nhf-nsitf ; https://www.paidhr.com/blog/how-paidhr-delivers-payroll-precision-and-timeliness . Common cooperative/loan deduction errors catalogued (not stopped after loan repaid; double deductions) — https://www.paidhr.com/blog/payroll-discrepancy-in-nigeria
- **Bento Africa**: 900+ Nigerian businesses; credit engine with Tarya; ₦1bn loans processed, 2–5%/month, ~1% default (TechCrunch via search; 2021–22) — https://techcrunch.com/?p=2244703 ; https://techcabal.com/2022/02/25/payroll-fintech-takes-off-in-africa/ ; salary advance flow: employee requests in member portal, approver grants, "the money is deducted from each pay check till it's paid back" (Bento Medium post; page 403 on fetch, text from snippet) — https://medium.com/bento/neat-new-features-on-our-payroll-product-f862ca73d52f **[UNVERIFIED]**
- **HumanManager** (SystemSpecs): "widely used in banking, oil and gas, and public sector"; Basic/Premium/Enterprise modules; PFA/NHF/NSITF/PAYE integration (FinancialContent press release, 16 May 2023) — https://markets.financialcontent.com/stocks/article/getnews-2023-5-16-humanmanager-introduces-exciting-features-deepens-strategic-partnerships-to-improve-customer-satisfaction . No published third-party deduction export.
- **Workpay**: statutory compliance guide (PAYE, PenCom, NHIS, ITF) — https://www.myworkpay.com/blogs/statutory-compliance-a-detailed-guide-to-statutory-deductions-in-nigeria
- **SAP**: generic "Third-Party Remittance" functionality exists in SAP HCM/S4 payroll (produces remittance postings per third party) — https://help.sap.com/docs/SAP_S4HANA_ON-PREMISE/eb65ca26f8f146c58f1fbdb013a64690/ca26e153038a424de10000000a174cb4.html . Large banks/oil companies on SAP/Oracle can therefore produce a per-cooperative remittance file, but the format is employer-specific.
- Payslip legal content (2026 employer guide) lists voluntary deductions including "loan repayments, cooperative or union dues" as standard line items — https://anoorehr.com/blog/nigeria-payslip-legal-requirements-2026

### Gaps
- No vendor publishes: a third-party-deduction schedule import (CSV spec), a per-beneficiary remittance export, webhooks on payroll run, or a partner API for external cooperatives. **[CONFIRM WITH SeamlessHR, PaidHR, Bento, HumanManager partnerships teams]**
- Bento's and SeamlessHR's embedded-lending partner terms (fee split, exclusivity). **[CONFIRM]**
- How bank/oil/telco staff cooperatives actually receive remittance (single NIP with emailed Excel schedule is the anecdotal norm). **[UNVERIFIED — interview 3–5 staff cooperatives]**

---

## 4. Bank and payment-rail side: how the remitted money moves and what evidence travels with it

### Takeaway
A remittance arrives either as one NIP credit (schedule emailed/none) or as many NIP credits from a bulk upload (NIBSS Pay or the bank's corporate portal, CSV schedule with ≤100-char narration per item). No structured remittance advice travels on NIP; the only machine-readable fields are amount, sender name, narration (≤100 chars), session/transaction reference. Cooperatives can read their accounts via the bank's corporate portal (CSV/PDF/Excel; MT940 only on request from larger banks) or via aggregators (Mono; Okra ceased trading May 2025). When the employer fails to deduct, the fallback is a NIBSS direct-debit e-mandate on the member's account (₦50 verification transfer, ₦30 NIBSS fee per successful debit, variable or fixed mandates, 1–365 days) — available to cooperatives as "billers" via a bank or PSSP; GSI is **not** available to cooperatives (CBN-licensed institutions only).

### Cited Findings

**Bulk payment schedules**
- NIBSS Pay: CSV schedule with "beneficiary account name, beneficiary account number, beneficiary bank, bank code, amount to be paid, and narration"; up to 100,000 items per file; Operator/Approver/Auditor roles; name-enquiry validation; "N15 per transaction record"; banks integrate it for corporates, MDAs can onboard directly — https://nibss-plc.com.ng/nibss-pay/
- Bank corporate portals (e.g. GTBank GAPS) upload Excel→CSV templates per bank spec; GAPS "integrates ... via Webservice - API and SWIFT" — https://www.gtbank.com/business-banking/services/e-business-services/gaps ; https://www.scribd.com/document/636942531/GAPS-Brochure
- NIP narration: "reason field ... should be less than 100 characters" (Anchor BaaS docs reflecting NIP) — https://docs.getanchor.co/docs/bank-transfer
- No ISO 20022 remittance-advice block is carried on NIP; reconciliation relies on the NIP transaction reference captured at payment time — https://paymentbrief.com/articles/nigeria-payments-operator-guide/ ; truncated/inconsistent narrations are a recognised reconciliation problem — https://tryduplo.com/blog/automate-payment-reconciliation-nigeria

**Statement feeds**
- Mono Bank Statement API: `GET /v2/accounts/{id}/statement`, 1–12 months in one call, `x-realtime: true` for latest transactions — https://docs.mono.co/api/bank-data/bank-statement . Business-account coverage not stated in the fetched page. **[CONFIRM]**
- Okra "ceased trading in May 2025" (Techpoint via search) — https://okra.ng/ **[UNVERIFIED]**
- MT940: no Nigerian bank publishes MT940 availability for SME/cooperative accounts; typically a corporate-banking request. **[GAP]**

**NIBSS Direct Debit as fallback**
- NDD mandate types: Fixed, Variable, and GSI; paper or e-mandate; billers include "Pension, insurance, and credit companies", fintechs/PSSPs; "NIBSS charges N30 per successful direct debit transaction" and institutions "can mark-up fees" — https://nibss-plc.com.ng/nibss-direct-debit-ndd/
- E-mandate activation: borrower transfers ₦50 (₦100 at banks with ₦100 minimum) to a NIBSS-designated account within 168 hours; fee "cannot be modified, as the fee is paid to NIBSS" (Mono, Fincra, Adjutor docs) — https://docs.mono.co/docs/payments/direct-debit/mandate-setup-explained ; https://docs.fincra.com/docs/initiate-mandate ; https://docs.adjutor.io/adjutor-api-endpoints/direct-debit/the-direct-debit-process
- Lendsqr variant: ₦50 to Paystack-Titan within 7 days, then a ₦100 *test* debit, activation "within 2 hours"; card tried first, mandate second; mandate reusable across loans; buffer multiplier for penalties — https://docs.lendsqr.com/how-to-activate-direct-debit-with-nibss-emandate/ ; https://docs.lendsqr.com/receiving-payments-with-nibss-direct-debit/
- Flutterwave e-mandate: ₦100 authorisation transfer within 10 minutes of token generation; `end_date` ≤365 days; charge ≤ tokenised amount; activation typically <3 hours — https://developer.flutterwave.com/docs/direct-debit
- Mono Direct Debit pricing: 1% capped ₦1,000 + 7.5% VAT per debit; minimum ₦55 for ₦200–₦20,000; balance inquiry ₦50/call; Mono Sweep setup ₦500 + ₦100 per linked account — https://support.mono.co/en/articles/8698485-mono-direct-debit-pricing
- Why direct debit under-performs in Nigeria (Lendsqr): bank-side mandate approval friction, customer distrust, insufficient-funds retries — https://blog.lendsqr.com/3-reasons-why-direct-debit-hasnt-become-a-hit-in-nigeria/

**GSI limits**
- CBN GSI Guidelines (13 Jul 2020, loans from 28 Aug 2019): individuals only; principal + interest only, no penal charges; Jan 2022 amendment made recovery "continuous and unrestricted" until repaid; only CBN-licensed banks, MFBs, finance companies, mortgage/investment banks may trigger — money lenders (state licence) and cooperatives excluded; penalties ₦500,000 per erroneous trigger — https://blog.lendsqr.com/cbn-gsi-global-standing-instruction-explained/ ; https://www.thecable.ng/gsi-loan-recovery-feature-will-now-be-continuous-until-debts-are-fully-paid-says-cbn/ ; https://www.dentonsacaslaw.com/en/insights/articles/2022/february/10/cbns-amendment-to-the-global-standing-instruction-guidelines-individuals

### Gaps
- Whether a registered cooperative society (not a licensed lender) can be onboarded as an NDD biller directly by its bank, or must use a PSSP (Mono/Paystack/Flutterwave/Remita). **[CONFIRM with 2 banks + 1 PSSP]**
- Daily/monthly NDD debit caps per bank; partial-debit behaviour. Not published.
- Bank CSV statement column formats (narration/reference lengths) across top banks — collect samples.

---

## 5. Fees: consolidated table of published or reported charges

| Rail / party | Fee | Source / status |
|---|---|---|
| Remita Collections | 2.0% capped ₦2,500; ₦300 flat education | brands.ng 2026, nuggetsmfb **[UNVERIFIED — remita.net/pricing blocked]** |
| Remita Payroll | ₦500/employee setup; ₦250/transaction | search snippet **[UNVERIFIED]** |
| Remita loan repayment (direct debit/inflight) | "fixed amount (e.g., ₦50) or nominal %" | Lendsqr FAQ — illustrative **[CONFIRM]** |
| NIBSS Pay bulk payments | ₦15 per record | nibss-plc.com.ng |
| NIBSS Direct Debit | ₦30 per successful debit; ₦50 e-mandate verification (₦100 at some banks) | nibss-plc.com.ng; Mono/Fincra/Adjutor docs |
| Flutterwave e-mandate | ₦100 authorisation; app fee shown as ₦104 on ₦200 sample | developer.flutterwave.com |
| Mono Direct Debit | 1% capped ₦1,000 + VAT; min ₦55 (₦200–₦20k); ₦50 balance inquiry | support.mono.co |
| Paystack / Flutterwave card & transfer (for comparison) | 1.5%+₦100 cap ₦2,500 / 1.4% cap ₦2,000 | Lendsqr comparison |
| State governments | No published admin fee for third-party deductions found | **[GAP]** |
| TSA historical | 1% split 50:40:10 SystemSpecs:banks:CBN | Vanguard 2015 |
| Lender "collection fee" % | Not published anywhere; only Kano audit evidence of over-charging | **[GAP — interview lenders]** |

---

## 6. Integration options for a cooperative-finance platform (synthesis)

1. **State/LG payroll (where the cooperative is already on the schedule)**: no API anywhere. Options are (a) generate the monthly deduction schedule in the state's Excel layout for the cooperative to submit to HoS/AG; (b) ingest the state's remittance (single NIP + emailed schedule, or no schedule) and reconcile member-by-member; (c) track arrears months per state, since 6–21-month gaps are documented. Expect states to exit the role (Oyo 2023, Kano 2026).
2. **Remita**: for members whose employer pays salaries through Remita Payroll, a licensed-lender partner (or the cooperative if it qualifies) can use Inflight Collections (Loan Disbursement Notification → interception at source; Stop Loan Collection on payoff). Fees and eligibility unpublished. Remita Direct Debit is a separate bank-account mandate usable by any biller.
3. **Private employer HRIS**: configure the recurring voluntary deduction in the employer's payroll (SAP/Oracle/SeamlessHR/PaidHR/HumanManager); the platform's role is to supply the per-member schedule and consume the employer's remittance file. No open APIs; Bento/SeamlessHR embed only their own lenders.
4. **Fallback — NIBSS direct debit** via a PSSP (Mono, Paystack, Flutterwave, Remita, Fincra) with e-mandate (₦50 verification, ₦30/debit + PSSP markup) for members whose employer does not deduct or stops remitting. GSI is unavailable to cooperatives.
5. **Reconciliation**: bank feed via corporate portal CSV or Mono statement API; match on NIP reference + ≤100-char narration + amount; design for schedule-less single credits.

---

## 7. Items to confirm directly with vendors / institutions
- SystemSpecs/Remita: Inflight Collections eligibility for cooperative societies; fee schedule; list of states/employers whose payroll runs on Remita Payroll; post-TMRAS federal salary flow; payslip visibility of inflight collections.
- Two or three state Treasury/HoS offices (Lagos, Kaduna, Ogun): deduction-code onboarding documents, schedule format, remittance advice practice, any admin fee.
- SeamlessHR, PaidHR, Bento, HumanManager: third-party deduction import/export specs, partner API, embedded-lender terms.
- Two commercial banks + NIBSS: can a cooperative be an NDD biller directly; MT940/CSV availability; narration/reference lengths on inbound NIP.
- Mono: business-account statement coverage; Okra status.
