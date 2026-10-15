# Nigeria group-finance platform: regulatory, identity, payment and data enablers and constraints

Compiled 8 October 2026. Scope: what a Nigerian platform offering pooled-savings custody, cooperative ledgers, payroll check-off reconciliation, cooperative-estate escrow and member lending would need. Figures carry the date they were published/effective. Items marked **[GAP]** could not be verified against a primary or reliable secondary source in this pass; items marked **[RECALLED, unverified]** are from general knowledge and must be checked before use.

Method note: ~35 web searches plus direct fetches of CBN/SEC/NDPC/NIBSS/law-firm documents. cbn.gov.ng, paystack.com, nairametrics, thecable, techcabal and remita.net blocked automated fetches, so several items rely on law-firm explainers or secondary press.

---

## 1. CBN licences for holding or moving pooled money

### 1.1 Microfinance Bank (MFB)
Source: CBN Guidelines for the Regulation and Supervision of MFBs 2020 (effective 1 Apr 2020), as summarised by Templars (Sep 2020) and Lex Praxis; CBN circular 29 Apr 2020 extending deadlines.

| Category | Minimum paid-up capital | Application fee | Licence fee | Geography |
|---|---|---|---|---|
| Tier 1 Unit (urban) | ₦200m | ₦100,000 | ₦250,000 | HQ + up to 4 branches in 5 contiguous LGAs |
| Tier 2 Unit (rural/unbanked) | ₦50m | ₦100,000 | ₦250,000 | HQ + 1 branch in same LGA |
| State | ₦1bn | ₦200,000 | ₦500,000 | one state/FCT; max 10 branches at start |
| National | ₦5bn | ₦300,000 | ₦1,000,000 | nationwide; max 10 branches at start |

- Recapitalisation path: 2018 circular raised Unit ₦20m→₦200m, State ₦100m→₦1bn, National ₦2bn→₦5bn. COVID extension (29 Apr 2020): interim thresholds by Apr 2021 (Tier 2 ₦35m, Tier 1 ₦100m, State ₦500m, National ₦3.5bn), full by **Apr 2022**. Fees above are "non-refundable"; CBN aims to respond to an application within 90 days (AIP), then 6 months to final licence (Lex Praxis).
- Ownership: individuals/related parties capped at 49%; no controlling interest in more than one MFB; capital may not be borrowed from the Nigerian banking system; shareholder loans must be ≥7-year tenor (Templars).
- Permitted: deposits, loans, domestic transfers, e-wallets, agent banking (Templars). Unit MFBs serving digital customers nationally was a known regulatory arbitrage (Templars 2020).
- Jan 2026: CBN announced national-MFB upgrades for OPay, Moniepoint, Kuda, PalmPay and Paga (Condia, 27 Jan 2026; Economic Times NG). The ₦5bn national threshold is in force.
- **[RECALLED, unverified]** CBN revoked 179 MFB licences in May 2023 and further batches since; there were roughly 870–880 licensed MFBs in 2023. Verify against CBN's current OFI list.
- Deposit insurance: NDIC raised MFB cover to **₦2m per depositor** (from ₦200,000) and DMBs to ₦5m, decided 24–25 Apr 2024, effective immediately (Nairametrics, Vanguard 2 May 2024). Cooperative deposits have **no** NDIC cover.

Sources:
- https://www.templars-law.com/app/uploads/2020/09/CBN-Microfinance-Bank-Guidelines-2020-and-implications-for-Digital.Banks-08-09-2020-Final.pdf
- https://lexpraxisng.com/how-to-set-up-a-microfinance-bank-in-nigeria/
- https://nairametrics.com/2020/04/30/recapitalisation-cbn-extends-deadline-for-microfinance-banks/
- https://thecondia.com/cbn-national-licence-announcement-delayed-reclassification/
- https://nairametrics.com/2024/05/02/ndic-raises-deposit-insurance-coverage-for-bank-customers-in-nigeria-to-n5-million/

### 1.2 Payments licensing framework (CBN circular 9 Dec 2020)
Source: Templars newsletter 16 Dec 2020; Resolution Law / Tetra Consultants for fee figures.

| Licence | Min capital | Can hold customer funds? | Permitted |
|---|---|---|---|
| Switching & Processing | ₦2bn | No | switching, card processing, clearing/settlement agent, non-bank acquiring + all PSS activities |
| Mobile Money Operator (MMO) | ₦2bn | **Yes (only category that can)** | e-money issuance, wallet creation/management, pool-account management, super-agent activities |
| PSSP | ₦100m | No | gateways/portals, payment app development, merchant aggregation & collections |
| PTSP | ₦100m | No | POS deployment/ownership, agent/merchant support |
| Super Agent | ₦50m | No | agent network recruitment/management |
| Regulatory Sandbox | none | per CBN | testing with forbearance |

- Fees: application ₦100,000 (non-refundable), licensing ₦1,000,000 (Resolution Law; Tetra). Some sources add a refundable escrow deposit equal to the minimum capital (₦100m PSSP/PTSP; ₦2bn MMO/switch) lodged with CBN during licensing. **[GAP]** confirm current escrow-deposit rule.
- Switch + MMO must be held through a HoldCo (one activity per subsidiary). PSS licences can be combined.
- **All collaborations between licensed PSPs, banks and OFIs on products/services require CBN prior approval** (Templars) — this is the legal hook for bank-partnership ("BaaS") models.
- Existing licensees had to comply by June 2021.
- Payment Service Bank (PSB): minimum capital **₦5bn**, application ₦500,000, licence ₦2,000,000, change-of-name ₦1,000,000 (CBN revised PSB Guidelines, 27 Aug 2020). PSBs can take deposits (NDIC-insured), issue wallets/cards, but cannot lend. Five licensed (Hope, MoneyMaster, 9PSB, MoMo PSB, SmartCash).

Sources:
- https://www.templars-law.com/app/uploads/2020/12/Newsletter-on-CBN-Payments-System-Circular-v.16.12.20.pdf
- https://www.resolutionlawng.com/payments-service-provider-license/
- https://www.tetraconsultants.com/jurisdictions/nigeria-service-provider-license/
- https://www.cbn.gov.ng/Out/2020/CCD/APPROVED%20REVIEWED%20GUIDELINES%20FOR%20LICENSING%20AND%20REGULATION%20OF%20PAYMENT%20SERVICE%20BANKS%20IN%20NIGERIA-27AUG2020.pdf
- https://nairametrics.com/2020/12/11/cbn-approves-new-license-categorizations-for-payment-systems/

### 1.3 Bank-partnership ("banking-as-a-service") route
- Common pattern: unlicensed platform + PSSP (or no licence at all) + a partner bank that issues virtual/pool accounts and holds funds. Providers named in market: Providus, Wema (ALAT), Sterling (OneBank), VFD MFB (VBank), Fidelity, Polaris, SunTrust, Globus, Titan Trust (Paystack-Titan). Aggregators: OnePipe, Fincra, Monnify (Moniepoint), Paystack DVAs, Flutterwave.
- Legal constraints: only MMOs/banks may hold customer funds; PSP–bank collaborations need CBN no-objection (2020 circular). **[RECALLED, unverified]** In Apr–May 2024 CBN froze new-customer onboarding at OPay, Kuda, Moniepoint and PalmPay over KYC concerns; in Dec 2023 CBN mandated BVN/NIN on all Tier 1 wallets (effective 1 Mar 2024). CBN has not published a dedicated "BaaS" guideline **[GAP]**.
- Practical implication: pooled savings for a cooperative can sit in a bank account in the cooperative's name (with sub-ledgers kept by the platform) without the platform holding funds; the platform's exposure is as technology/collections agent. Interest on pooled balances is contractual with the partner bank.

Sources:
- https://blog.lendsqr.com/4-best-monnify-alternatives-for-virtual-accounts/
- https://www.monito.com/en/wiki/best-digital-banks-in-nigeria
- https://techpoint.africa/insight/delayed-open-banking-launch/ (notes Wema/Providus/Sterling API-revenue models)

### 1.4 Wallet, escrow and trust-account rules
- Tiered KYC (CBN 3-tier framework, as updated): Tier 1 — max balance ₦300,000, daily limit ₦50,000 (some sources ₦30,000); Tier 2 — balance ₦500,000, daily ₦500,000; Tier 3 — no limits. BVN **or** NIN mandatory on Tier 1 wallets; BVN **and** NIN on Tier 2/3 from **1 Mar 2024** (CBN circular Dec 2023; Dojah/Youverify explainers).
- Escrow: there is **no CBN escrow licence**. Escrow in practice is (a) a bank-held account under a tri-partite agreement, (b) an MMO wallet, or (c) a lawyer's client account. Fintech "escrow" products operate under PSSP + partner-bank structures. **[GAP]** no 2024–26 CBN circular specific to escrow/trust accounts was found; NIBSS Direct Debit docs list "pool account management" as an MMO activity.
- FSRCC "Advisory on Illegal Financial Operators", **4 Apr 2022**: anyone who is not a *registered cooperative society* and takes deposits/investments without a CBN licence or SEC approval is an illegal financial operator (SRJ Legal summary). This is the clearest statement that registered cooperatives may take member deposits without a CBN licence.

Sources:
- https://dojah.io/blog/navigating-nigerias-new-kyc-rules-cbn-mandates-bvn-and-nin-linkage
- https://youverify.co/blog/3-tiered-know-your-client-kyc-requirements
- https://srjlegal.com/hemmed-in-and-narrowed-activities-of-unlicensed-financial-operators/

### 1.5 Cooperative-specific CBN policy
- 2005 Microfinance Policy and 2011 Revised Microfinance Policy Framework treat cooperatives as microfinance providers outside CBN licensing. **[RECALLED, unverified]** The 2011 framework states that financial cooperatives reaching ~2,000 members or ~₦20m in deposits/assets should convert to MFBs; the CBN PDF (cbn.gov.ng/out/2011/...) returned 403 and no secondary source quoted the clause. Treat as policy intent, never enforced.
- No 2024–2026 CBN circular on "financial cooperatives" was found **[GAP]**. Regulatory movement on cooperatives in 2025–26 is coming from the Federal Ministry of Agriculture & Food Security (FMAFS), not CBN (see §3).

---

## 2. SEC perimeter

| Function | Min paid-up capital | Fees (filing / processing / registration) | Notes |
|---|---|---|---|
| Fund/Portfolio Manager | ₦150m (SEC checklist, current) | ₦100k / ₦300k / **₦10m**; director ₦100k; sponsored individual ₦100k each | fidelity bond ≥20% of min capital; ≥3 sponsored individuals (MD + Compliance Officer) |
| Collective Investment Scheme (per scheme) | n/a | ₦50k / ₦200k / ₦500k; sponsored individual ₦50k | SEC CIS Rules 2020 |
| Crowdfunding Intermediary/Portal | **₦100m** (Rules 21 Jan 2021) | ₦100k / ₦300k / ₦1m; director ₦100k | annual renewal by 31 Jan; fidelity bond 20% |

- **SEC circular 16 Jan 2026** revised minimum capital (compliance by **30 Jun 2027**): Tier-1 portfolio manager (AUM >₦20bn) ₦5bn; Tier-2 ₦2bn; PE fund manager ₦500m; VC fund manager ₦200m; broker ₦600m; dealer ₦1bn; broker-dealer ₦2bn (Mondaq, 2026). The SEC checklist page still showed ₦150m for fund managers at fetch time; assume the 2026 figures govern new applicants.
- **ISA 2025** (signed 29 Mar 2025): widens "securities"/investment-contract perimeter (digital assets, investment contracts), requires every crowdfunding offer to be pre-cleared and the securities registered; retail investor cap 10% of net annual income; fines of ≥50% of security value for unregistered offers, directors 10%/≥5 years. Ponzi/illegal investment schemes criminalised (Financial Nigeria feature).
- **Investment-club / NAV accounting:** a platform that only keeps books for a self-managed club (members decide, assets in the club's own brokerage/bank account) is not itself a fund manager; it becomes one if it exercises discretion over pooled assets or solicits the public. Cooperative *member* savings are outside SEC under the Co-operative Societies Act; marketing "returns" to non-members is the trigger. **[GAP]** no SEC guidance specific to investment clubs found; the 2022 FSRCC advisory is the operative statement.
- Crowdfunding caps: medium enterprise ₦100m, small ₦70m, micro ₦50m per 12 months (SEC 2021 Rules).

Sources:
- https://sec.gov.ng/about/resources/checklists/individual-registration-requirements-for-each-cmo/fundportfolio-managers-registration-requirements/
- https://www.sec.gov.ng/about/resources/checklists/individual-registration-requirements-for-each-cmo/crowdfunding-intermediary-registration-requirements/
- https://www.mondaq.com/nigeria/fund-management-reits/1735164/sec-revises-minimum-capital-requirements-for-regulated-capital-market-entities
- https://www.financialnigeria.com/how-the-isa-2025-reshapes-nigeria-s-crowdfunding-regulation-feature-597.html
- https://techcabal.com/2021/02/15/crowdfunding-regulations/

---

## 3. Cooperative regulation and the 2025–26 federal reform

### 3.1 Existing law
- Nigerian Co-operative Societies Act (Cap N98 LFN 2004, originally Decree 90 of 1993) + state cooperative laws (e.g., Lagos). Registration is by the **State Director of Cooperatives** (Federal Director for FCT and multi-state societies). Act parts cover registration, duties/privileges, member rights, funds, audit/inspection, cancellation, surcharge, disputes, amalgamation (LawNigeria).
- **[RECALLED, unverified]** Minimum 10 members (6 for some state laws), bye-laws, feasibility report, inaugural-meeting minutes; state fees typically ₦10,000–₦50,000; annual audited accounts to the Director. Verify per state.
- Sector size: >370,000 registered societies, >30m members, ~₦1.7tn annual turnover (FMINO/VON, May 2026).

### 3.2 RH-CRRP 2025–2030 and NCDAP
- Renewed Hope Cooperative Reform & Revamp Programme approved at the 8th National Council on Cooperative Affairs, Kaduna, **March 2026**: seven pillars incl. legal/regulatory reform, digitalisation/data, cooperative finance (Cooperative Bank), capacity, value chains, inclusion, partnerships (VON 12 May 2026; FMINO).
- **MoU 12 Aug 2026** — FMAFS (Perm Sec Marcus Ogunbiyi) + Cooperative Federation of Nigeria + **Seamfix**: National Cooperative Digital Architecture Platform (NCDAP) = National Cooperative Smart Registry (NCSR) + **Cooperative Verification Number (CVN)** per society + **CoopID** per member, both **linked to NIN**. Seamfix to "deploy, operate, maintain and support the initial phase" (registration validation, member onboarding, enrolment). USSD/feature-phone access planned. **Six-month national revalidation/re-certification** of all societies via State Departments of Cooperatives began Aug 2026 (Vanguard; Co-op News; Biometric Update; Nairametrics 17 Aug 2026).
- Stated integrations: **CoopCHECK** (described as a cooperative credit bureau) for credit profiling; the proposed Cooperative Bank for lending. **No public API, data-access policy, pricing or timeline for third-party verification has been published [GAP]** — a platform should plan to verify CVN/CoopID manually or via Seamfix commercial channels until an API exists.
- Legal reform: Minister of State Aliyu Sabi Abdullahi is "harmonising reviews" of the Co-operative Societies Act toward a modern Act covering registration, governance, reporting, supervision, audit, sanctions and data protection (Co-op News, Aug 2026). **No bill number or National Assembly stage found [GAP].**

### 3.3 Proposed Cooperative Bank of Nigeria
- Announced 12 May 2026 (VON) and launched for capital mobilisation 18 May 2026 (Lagos, South-West engagement) and 19 Jun 2026 (Leadership): target **₦200bn** share capital; ownership 65% cooperatives/unions/federations/cooperators, 30% institutional/DFI/impact investors, 5% employees and affiliated enterprises. "Government-enabled, not government-funded; no TSA funds."
- Mobilisation tiers: 1,000 societies × ₦21–50m; 3,000 × ₦16–20m; 6,000 × ₦1–15m; 10,000 societies across 36 states + FCT.
- **Licensing status: no CBN application, AIP, or bank category (commercial/national/MFB) has been reported as of Oct 2026 [GAP].** ₦200bn would clear the 2024 recapitalisation bar for a national commercial bank (₦200bn) but no timeline is public.

Sources:
- https://gazettengr.com/fg-signs-mou-on-national-cooperative-identity/
- https://www.thenews.coop/nigerian-government-signs-mou-to-strengthen-co-operative-identity/
- https://www.vanguardngr.com/2026/08/fg-begins-six-month-revalidation-of-cooperatives-unveils-digital-id-system/
- https://nairametrics.com/2026/08/17/seamfix-federal-government-and-cfn-sign-agreement-to-launch-national-digital-identity-infrastructure-for-nigerias-cooperative-sector/
- https://www.biometricupdate.com/202608/nigeria-extends-national-digital-id-into-cooperative-sector-with-seamfix
- https://von.gov.ng/nigeria-launches-cooperative-digitalisation-bank-reform-initiative/
- https://allafrica.com/stories/202605180292.html
- https://leadership.ng/federal-govt-targets-n200bn-capital-mobilisation-for-proposed-cooperative-bank/
- https://msmeafricaonline.com/fg-launches-n200bn-capital-mobilization-drive-for-proposed-cooperative-bank/
- https://lawnigeria.com/2025/03/26/nigerian-co-operative-societies-act/

---

## 4. Identity and KYC

- **BVN** (NIBSS): 67.8m at 31 Dec 2025 (+6.8% y/y; 63.5m 2024, 60.1m 2023); 70.04m at 31 Aug 2026; 70.35m at 28 Sep 2026 (NIBSS; Vanguard Jan 2026; The Sun). Access to BVN validation is via NIBSS for licensed institutions (banks/OFIs/MMOs); non-licensed platforms use NIBSS-connected aggregators (Dojah, Prembly/IdentityPass, Youverify, Mono, Smile ID). NIBSS consent layer "iGree" exists. **Per-lookup pricing from NIBSS not published [GAP]**; aggregator list prices: Dojah "as low as $0.04–0.06 per API call" (pricing page, 2026); Didit $0.08 per conclusive NIN query.
- **NIN** (NIMC): ~110m enrolled (2026). NIMC Verification Service is tiered (Level 1 minimal → Level 5 full demographics), fees "negotiable by access level"; enterprise tier pricing at wiki.nimc.gov.ng/en/pricing/enterprises (site unreachable at fetch) **[GAP]**. NIMC Act 2026 passed (Techeconomy) — monetisation of verification is an explicit aim.
- **CAC**: public search at search.cac.gov.ng free (name/RC/BN); certified extracts ₦1,000–₦15,000; Mono CAC Lookup ₦10, Lookup+ (shareholders) ₦50 per call. No official CAC API for third parties; vendors scrape/resell.
- **Cooperative registration verification: no API exists.** State registers are paper/Excel; the NCSR/CVN is the intended fix (see §3.2).
- Open-banking identity endpoints (Mono): free tier 5 accounts/month; Basic ₦50,000/month for 100 accounts; data sync ₦5/call.

Sources:
- https://nibss-plc.com.ng/nigerias-bvn-database-hits-67-8-million-registrations-by-end-of-2025-nibss/
- https://thesun.ng/nibss-bvn-database-hits-70-34m-as-banks-leverage-biometrics-to-fight-fraud/
- https://didit.me/blog/nigeria-national-id-database-validation/
- https://dojah.io/pricing ; https://mono.co/pricing
- https://www.9jadirectory.org/blog/cac-public-search-verify-company-nigeria
- https://techeconomy.ng/nimc-act-2026-turning-nigerias-identity-system-into-an-economic-asset

---

## 5. Payment rails

### 5.1 NIBSS Instant Payments (NIP)
- Volumes: 2bn (2020) → **11bn transactions (2024)**, CAGR 53%; value $457bn → **$1.1tn (2024)**; 57.7m unique users (52% of adults); H1-2024 alone 5.63bn transactions / ₦476.89tn (SIIPS 2025 NIP case study; NIBSS). **Full-year 2025 figure not retrieved [GAP]** (press reports ~₦1 quadrillion e-payments in 2024).
- Participants: 29 direct (commercial banks) + 670 indirect incl. 541 MFIs, 10 MMOs, 5 PSBs, 20 PMBs (SIIPS 2025). MFBs/MMOs connect via NIP EasyPay.
- Participant fee: **₦3.75 flat per transaction to NIBSS from 1 Jul 2023** (down from ₦5; ₦46 in 2021 per SIIPS). NIBSS has said it targets zero per-transaction fees via a subscription model "by 2026" (Nigeria Communications Week) — not yet effective **[GAP]**.
- End-user caps (CBN Guide to Charges 2020, still in force): ₦10 (≤₦5,000), ₦25 (₦5,001–₦50,000), ₦50 (>₦50,000) + 7.5% VAT; these are the same bands Flutterwave charges for payouts.

### 5.2 Direct debit / e-mandates
- NIBSS Direct Debit (NDD) / Central Mandate Management System: fixed, variable and GSI mandates; originators can be banks, MMOs, OFIs (as aggregators), fintechs/PSSPs, pension/insurance/credit companies. **NIBSS fee ₦30 per successful debit**, markup allowed; onboarding via NIBSS relationship manager, integrating Mandate Management + EasyPay APIs (NIBSS NDD page).
- e-mandate activation in practice: customer sends **₦50 verification transfer** (fixed, paid to NIBSS) then a ₦100 test debit (Lendsqr/Paystack-Titan flow); Flutterwave charges a ₦100 authorisation fee; tokens valid ≤365 days; 22 banks supported (Flutterwave docs). Adoption historically poor because of paper steps and exact-amount failure (Lendsqr).
- Remita (SystemSpecs) direct debit and payroll products: site blocked; **fees not retrieved [GAP]**. Remita USSD *347*780# is used by payroll lenders (VeendHQ).
- Global Standing Instruction (GSI): CBN guidelines 14 Jul 2020 (effective 1 Aug 2020, loans from 28 Aug 2019); Jan 2022 amendment made recovery attempts continuous/unrestricted; covers savings, current, domiciliary, investment accounts, e-wallets and joint accounts across all banks and OFIs; **individuals only**; mandate must be in loan documents and linked via BVN. Participating creditors must be CBN-licensed institutions — a cooperative or unlicensed platform cannot originate GSI directly (must lend via an MFB/bank partner).

### 5.3 Gateways, virtual accounts, splits
- Paystack (2026): local cards/transfers 1.5% + ₦100 (₦100 waived <₦2,500), cap ₦2,000; international 3.9% + ₦100; standard payouts free; settlement same/next day. **Dedicated virtual account fee not confirmed [GAP]** — **[RECALLED]** 1% capped ₦300 per DVA credit; split payments/subaccounts are free beyond the transaction fee.
- Flutterwave (official NG pricing page, 2026): 2% local (third-party comparisons cite 1.4%), cap ₦2,000; international 3.8% (+1% for FX receipt); payouts ₦10/₦25/₦50 by band; fees + 7.5% VAT; split payment and virtual-account fees not itemised.
- Interswitch: switching/processing licence holder (₦2bn class); no public price list **[GAP]**.

### 5.4 Open banking
- CBN Operational Guidelines Mar 2023 (framework Feb 2021). CBN announced (Apr 2025) go-live **1 Aug 2025**; it did not happen. CBN (Oct 2025): "hasn't gone live yet… will happen soon"; five workstreams, API standard at apis.openbanking.ng. As of **Jun 2026** industry blog posts still discuss pre-go-live issues; **no confirmed go-live date [GAP]**. Okra shut down May 2025; Mono (raised $17.6m) and Stitch are the surviving aggregators. Unlicensed entities will get anonymised data only.

### 5.5 USSD and agency banking
- USSD: end-user billing from **18 Jun 2025**: ₦6.98 per 120-second session charged from airtime by the MNO; banks may not charge for USSD (NCC/CBN; ALTON).
- Agent banking: CBN guidelines **6 Oct 2025**: agent exclusivity (one principal, one super-agent) from **1 Apr 2026**; customer limits ₦100,000/day and ₦500,000/week for cash-in/out/bills; agent cash-out ₦1.2m/day; geo-tagged terminals via PTSAs (Aug 2025 directive); agents must have clean BVN, no NPLs in 12 months. Agent commissions ~0.3% capped ₦18–20 per cash transaction (Techpoint).

Sources:
- https://www.africanenda.org/uploads/files/siips2025/siips_2025_NIP-Nigeria_CaseStudy_en.pdf
- https://www.nigeriacommunicationsweek.com.ng/nibss-targets-zero-transfer-fees-on-instant-payments-by-2026/
- https://nibss-plc.com.ng/nibss-direct-debit-ndd/
- https://docs.lendsqr.com/how-to-activate-direct-debit-with-nibss-emandate/
- https://developer.flutterwave.com/docs/direct-debit
- https://www.harlemsolicitors.com/2022/01/30/explanatory-note-on-cbns-circular-on-global-standing-instruction-gsi/
- https://www.cbn.gov.ng/Out/2020/CCD/CBN%20-%20Operational%20Guidelines%20on%20Global%20Standing%20Instructions%20GSI%20-%20Individuals.pdf
- https://kudicompass.com/paystack-vs-flutterwave-nigeria-payment-gateway-comparison-2026/ ; https://flutterwave.com/ng/pricing
- https://techcabal.com/2025/04/29/cbn-launches-open-banking/ ; https://technext24.com/2025/10/20/cbns-open-banking-system-hasnt-launched/ ; https://www.vanguardngr.com/2025/10/nigerias-open-banking-will-happen-soon-cbn/ ; https://openbanking.ng/blog/ ; https://techlabari.com/nigerian-open-banking-startup-okra-quietly-shut-in-may-2025/
- https://techpoint.africa/news/nigerians-pay-ussd-banking-service/ ; https://ncc.gov.ng/frequently-asked-questions/end-user-billing-eub-policy-ussd-banking-transactions-nigeria
- https://techpoint.africa/news/cbn-agent-banking-guidelines/ ; https://techpoint.africa/insight/agent-banking-rules-competition/

---

## 6. Payroll check-off integration

- **IPPIS (OAGF)**: explicitly remits third-party deductions "Tax, Pension, Cooperatives, Union Dues and Bank Loans". Third parties obtain a deduction code after an OAGF/IPPIS application (letter, CAC/cooperative certificate, bank details, MoU) **[procedure not published online — GAP]**. **[RECALLED, unverified]** Public-service rule caps total deductions so net pay ≥ one-third of gross (the "one-third rule"); verify in current Financial Regulations/OAGF circulars. In 2025 OAGF moved MDA cash access from Remita to the Treasury Management & Revenue Assurance System (TMRAS); Remita remains the collections gateway (Leadership; OAGF Feb 2025 circular). GIFMIS handles MDA payments, not payroll deductions.
- **State payrolls**: fragmented (Oracle/own systems; several states use Remita or SystemSpecs' HumanManager). Lender/cooperative onboarding is state-by-state via the Accountant-General **[GAP: no central list]**.
- **Remita payroll lending**: lenders integrate for salary-linked loans and recurring deductions; VeendHQ markets "Remita payroll loans" via *347*780#. Fees not public **[GAP]**.
- **Private payroll software**: SeamlessHR (API available; Dynamics 365/Sage integrations; voluntary deductions configurable), PaidHR (API available), Bento Africa (embedded salary advances/loans). None publishes a third-party "cooperative deduction" API; integration is bespoke.

Sources:
- https://oagf.gov.ng/initiative/ippis/
- https://leadership.ng/remita-not-discarded-integrated-into-treasury-management-revenue-assurance-oagf/
- https://oagf.gov.ng/wp-content/uploads/2025/02/IMPLEMENTATION-OF-TREASURY-MANAEMENT-AND-REVENUE-ASSURANCE-SYSTEM-TMRAS-UNDER-THE-TREASURY-SINGLE-ACCOUNT-TSA-POLICY-OF-THE-FEDERAL-GOVERNMENT-OF-NIGERIA.pdf
- https://veendhq.com/blog/ippis-remita-code-for-payday-loans
- https://innovation-village.com/top-7-payroll-software-solutions-for-nigerian-smes-a-2026-comparison/

---

## 7. Land, title and housing data

- **Lagos Lands Bureau e-GIS** (launched 27–31 Jan 2024): online title search/verification, Governor's Consent and C of O applications, fee payment. Search fee **₦3,750**; Certified True Copy **₦5,625**; Governor's consent ~1.5% of FMV; stamp duty ~0.5%; registration ~0.5%; CGT ~0.5% (Lands Bureau schedule as summarised; bureau site returned 522). **No API or bulk access [GAP].**
- **LASRERA** (Lagos Real Estate Regulatory Law 2021): registers developers/agents and runs a property-search service; fee chart "varies by LGA, property type and term" and is given by LASRERA agents; site unreachable at fetch **[GAP]**.
- **FCT AGIS legal search**: ₦10,000 (private) / ₦20,000 (commercial); pay via Remita, submit in person 9am–3pm; result in 24h–3 days; **no online or API access**.
- Other states: Kaduna (KADGIS), Ogun, Kano have GIS portals; none offers third-party APIs **[GAP: not surveyed in detail]**.
- **NIQS / QSRBN**: a national construction cost database is still at proposal/process stage (QSRBN paper, Mar 2026); no public dataset.
- **REDAN**: directory of financial members; full membership ₦350,000 first year (₦10k application, ₦100k registration, ₦140k levy, ₦100k subscription); REDAN registration is a precondition for SCUML; **no verification API**.

Sources:
- https://thelagostoday.com/lagos-govt-launches-e-gis-portal-for-application-verification-of-land-titles/ ; https://techpoint.africa/news/lagos-land-registration-portal/
- https://cleecdesigns.com/cost-land-registration-in-lagos/ ; https://landsbureau.lagosstate.gov.ng/schdule-of-fees/
- https://agis.fcta.gov.ng/legal-search/
- https://qsrbn.gov.ng/wp-content/uploads/2026/04/Construction-Cost-Database-the-Processes-and-pro_260329_144021.pdf
- https://www.feybay.com/procedure-on-how-to-obtain-redan-certificate-in-nigeria-a-step-by-step-guide/

---

## 8. Credit data

- Three CBN-licensed bureaus: CRC Credit Bureau, FirstCentral, CreditRegistry. Consumer self-enquiry: one free report/year by law; CRC paid ₦5,000 (self) / ₦10,000 (Profile360); FirstCentral ₦1,000–₦2,000 for scored report; CreditRegistry ₦250 premium. **Institutional per-report pricing is contractual, not published [GAP]** (market practice: a few hundred naira per enquiry, falling with volume).
- Coverage: bureaus ingest from CBN-licensed lenders under the Credit Reporting Act 2017 and CBN's mandate that licensed institutions report to at least two bureaus. **Cooperative loans are generally absent** because cooperatives are not licensed institutions; some bureaus accept voluntary data submission by cooperatives/MFIs **[GAP: no published count]**.
- GSI: see §5.2 — effective recovery tool only for CBN-licensed creditors.
- **CoopCHECK**: named in the Aug 2026 NCDAP MoU as a "co-operative credit bureau" to be integrated for credit profiling. No website resolved (coopcheck.ng / .com.ng), no operator, licence status or pricing found **[GAP]**. It is not on the CBN credit-bureau licence list.

Sources:
- https://www.kuda.com/blog/how-to-check-credit-score-nigeria-why-it-matters/ ; https://getoze.com/blog/check-credit-score-nigeria/
- https://www.thenews.coop/nigerian-government-signs-mou-to-strengthen-co-operative-identity/

---

## 9. Data protection (NDPA 2023, GAID 2025)

- NDPA 2023 s.44: data controllers/processors "of major importance" must register with NDPC within 6 months of becoming one. GAID issued **20 Mar 2025**, effective **19 Sep 2025**; repeals NDPR 2019.
- Thresholds (NDPC Guidance Notice, updated 2024; GAID Sch.10): >200 data subjects in 6 months = major importance.
  - **UHL** (>5,000 subjects, or sector-listed: commercial banks, telcos, insurers, "fintech companies", multinationals, DisCos, oil & gas, social-media apps): **₦250,000** registration, once; annual Compliance Audit Return (CAR); plus ₦5,000 per processor engaged per 12 months.
  - **EHL** (1,000–4,999 subjects; MDAs, **microfinance banks**, mortgage banks, higher institutions, hospitals): **₦100,000** once; annual CAR.
  - **OHL** (200–999 subjects; schools, PHCs, small hotels, processors of sensitive data for commercial purposes): **₦10,000**, renewed annually; no CAR.
- A group-finance platform processing BVN/NIN (biometric-linked identifiers) for thousands of cooperators will be UHL by sector ("fintech") or by scale. Mandatory consent for sensitive data, automated processing and cross-border transfer; DPO with semi-annual reports; DPIA for new tech; enforcement live (Fidelity Bank fined ₦555.8m).

Sources:
- https://ndpc.gov.ng/wp-content/uploads/2025/07/Updated-Guidance-Notice-on-Registtration-2024.pdf
- https://www.gelias.com/images/Highlights_of_The_GAID.pdf
- https://www.legal500.com/intelligence/nigeria/privacy/review-of-nigeria-data-protection-act-ndpa-2023-&-gaid-2025-data-privacy-in-nigeria

---

## 10. Taxes affecting the platform (Nigeria Tax Act 2025, No. 7; effective 1 Jan 2026)

- **VAT 7.5%** retained; applies to platform fees and gateway fees (Flutterwave adds 7.5% VAT on all fees). "Financial services" defined to include depository, custodial, investment and insurance services (NTA s.? definitions, p.120); core interest/deposit margins remain outside VAT; fee-based services are taxable. Input-VAT recovery on services and capital items now allowed.
- **Stamp duty on electronic transfers (ex-EMTL)**: ₦50 on electronic transfers/receipts of ₦10,000 or more; from **1 Jan 2026 charged to the sender** (TechCabal 22 Dec 2025; Legit). NTA Part III exemptions (p.105): transfers **below ₦10,000**, **salary payments**, and **intra-bank self-transfers** are exempt; so are "any instrument executed by or on behalf of a co-operative society registered under any Act or law" (item (e)). EMTL has applied to fintech wallets since Dec 2024.
- **Cooperative societies**: NTA s.163(1)(a)(ii) exempts profits/gains of a registered co-operative society "not being profits or gains from any trade or business carried on by that society"; s.55 exempts CGT on property held in trust for a cooperative (non-trade); stamp-duty exemption above. Member-savings interest and loan interest to members are treated as non-trade mutual income in practice **[verify with counsel]**.
- **Withholding tax on interest**: **[RECALLED, unverified]** 10% on interest paid to companies and individuals under the Deduction of Tax at Source (Withholding) Regulations 2024 (effective 1 Jan 2025), final tax for individuals; cooperatives paying interest to members are not within the deducting-agent list unless registered taxpayers — confirm. NTA itself defers rates to those regulations (s.50(4) reference).
- **Small company**: NTA defines small company as turnover ≤₦50m and fixed assets ≤₦250m (excludes professional services) → 0% CIT/CGT/Development Levy. (Some commentaries cite ₦100m; the Act text read says ₦50m.) **Development Levy 4%** of assessable profits for other companies.

Sources:
- https://irs.gm.gov.ng/docs/national/NIGERIA_TAX_ACT_2025.pdf (ss.55, 163, Part III stamp-duty exemptions; definitions)
- https://techcabal.com/2025/12/22/why-your-50000-transfer-will-cost-100-from-2026/ ; https://techcabal.com/2025/09/22/how-50-naira-becomes-796bn/
- https://www.legit.ng/business-economy/money/1690178-nigerians-pay-n50-stamp-duty-electronic-transfers-january-1/
- https://www.ey.com/en_gl/technical/tax-alerts/nigeria-tax-act-2025-has-been-signed-highlights
- https://www.mondaq.com/nigeria/tax-authorities/1705160/the-nigeria-tax-reform-act-2025-and-how-it-affects-businesses

---

## 11. What is missing (summary of structural gaps)

1. **No licence category fits a cooperative-services platform.** Holding pooled funds requires MMO (₦2bn), PSB (₦5bn, no lending) or MFB (₦200m–₦5bn, geography-limited); PSSP (₦100m) cannot hold funds. The realistic design is PSSP/none + partner-bank pool accounts in each cooperative's name + platform sub-ledgers, with CBN no-objection for the bank collaboration.
2. **No API to verify cooperative registration or membership**; NCSR/CVN/CoopID (Seamfix, Aug 2026) has no published API, pricing or timeline; six-month revalidation runs to ~Feb 2027.
3. **No credit-data coverage of cooperative loans**; CoopCHECK is unlaunched/undocumented; GSI and bureaus are only open to CBN-licensed creditors — lending to members must route through a licensed partner to use GSI/bureau rails.
4. **No escrow licence or escrow rulebook**; estate escrow must be contractual on bank/MMO accounts.
5. **No deposit insurance for cooperative savings** (NDIC covers MFB ₦2m / PSB; cooperatives none).
6. **Open banking not live** (announced Aug 2025; still pending Jun 2026); Okra gone; direct debit works but needs ₦50 mandate authorisation and suffers exact-amount failures.
7. **Payroll check-off is manual**: IPPIS deduction-code onboarding and state payroll access are paper processes without public documentation; no payroll-software vendor offers a standard third-party deduction API.
8. **Land/title data is search-by-search** (₦3,750 Lagos, ₦10–20k AGIS), in-person outside Lagos, no bulk/API; no national construction-cost dataset.
9. **Cooperative law reform and the Cooperative Bank** are at policy/mobilisation stage; no bill text or CBN licence application is public.
10. **Cost stack to budget (2026)**: NIP ₦3.75/transfer (+₦10–50 end-user cap), direct debit ₦30 + ₦50 mandate, gateway 1.4–1.5% capped ₦2,000, EMTL ₦50 per ≥₦10k transfer (sender), USSD ₦6.98/session, KYC ~$0.04–0.08 per lookup, NDPC ₦250k registration + annual audit, SEC fund-manager ₦10m registration if NAV discretion is taken, MFB licence ₦350k–₦1.3m in fees on top of capital.
