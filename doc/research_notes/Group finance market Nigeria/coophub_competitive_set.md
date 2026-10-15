# CoopHub competitive set: who competes with Wema Bank's cooperative product in Nigeria (as of 9 Oct 2026)

**Compiled:** 9 Oct 2026. **Method:** desk research (about 150 web searches and page fetches across four parallel passes), plus direct extraction of CoopHub's own FAQ and Terms text from the coophub.alat.ng JavaScript bundle and Google Play / App Store metadata pulled on 9 Oct 2026. This builds on `competitive_landscape.md` and `bank_integration_options.md` in the same folder, and corrects them where noted.

**Labels:** **[self-reported]** = company or government claim; **[snippet]** = seen only in search-engine text because the page would not load; **[unverified]** = single weak source or our inference; **[primary]** = read directly from the product's own site, terms or store listing. Dates are publication dates unless marked "accessed". Several sites blocked fetching (TechCabal, Guardian, Altbank, Nairaland, getcoopra.com), so some facts rest on snippets.

---

## 1. Headline findings

1. **CoopHub has no direct bank rival.** No other Nigerian commercial bank or MFB was found offering a cooperative-management app with member records, loans and maker-checker withdrawals. That covers 2024, 2025 and 2026 to date. Other banks offer branch-opened "clubs/societies/associations" accounts (Access Community Current, Polaris Community Savings, GTBank Unincorporated Societies form). Sterling OneBank's "Community" and "Ajo" features are retail group savings, not cooperative governance. CoopHub's real rivals are neutral SaaS vendors, a microfinance bank-backed fintech (Rank), and the federal government's own platform.
2. **CoopHub is not free and it locks cooperatives to Wema.** Its Terms [primary, accessed 9 Oct 2026] require the cooperative to "operate an account with Wema Bank" and to "pay the subscription fee for access to the platform". Subscriptions auto-renew "monthly or biannual". The FAQ says a non-Wema account cannot be used. The amount is not published anywhere. Earlier notes (and press coverage) left the impression it was free or undisclosed; it is **paid, with an unpublished price**.
3. **CoopHub traction is stale and thin on public evidence.** The only figure is "over 1,000 cooperatives" (9 Jul 2024) [self-reported, Daily Trust]. No 2025–26 update was found. The member app has **1K+ Android downloads (4.2★, ~23 reviews; last updated 3 Oct 2025)** and **4.9★ from 10 iOS ratings (v1.1.6, 6 Oct 2025)** [primary]. Downloads at that level are hard to square with 1,000 active cooperatives unless most members never install the app. Neither store listing has been updated in about a year. Its FAQ still lists recurring contributions as "still under development" [primary].
4. **CoopHub's moat is bank custody plus three-role withdrawal control** (Initiator → Verifier → Approver; "No single person can make a withdrawal"). No neutral SaaS vendor documents an equivalent control enforced over real money. Coopify explicitly says it is "not a registered bank".
5. **Neutral SaaS is cheap and publishes prices. CoopHub does not.** Coopify is free for up to 10 members. BankCore starts at ₦10k/month (+₦100k setup). Coopco charges ₦15k/₦30k/₦90k per month. SmartTeller charges ₦50k/₦100k per month. Esusu Africa ranges from ₦25k/month to ₦10m lifetime. Vendors self-report 300–2,500 cooperatives each, but none of these counts is independently verified.
6. **The fastest-moving competitor is Rank (ex-Moni).** It rebranded in Nov 2025 and bought AjoMoney and Zazzau MFB (now Rank MFB). In Jul 2026 it launched Money Circles, Tribe (private groups) and Rank Perks (payroll-deducted workplace savings circles with salary-backed credit). It claims >$100m paid out in the last 12 months [self-reported]. Rank owns its licence and targets the informal and workplace groups that become or behave like staff cooperatives.
7. **The biggest structural threat is the state, not a bank.** RH-CRRP 2030 (approved Mar 2026) includes several pieces:
   - the **Seamfix national registry**: NCSR, a CVN per society, a CoopID per member and the CoopCHECK credit bureau. The MoU was signed 12 Aug 2026, and a six-month revalidation runs to about Feb 2027.
   - a planned **Cooperative Bank of Nigeria**: 65% cooperative-owned, with a ₦200bn share drive and subscription targeted from end-Oct 2026.

   Whoever integrates CVN/CoopID first gets an onboarding edge. A cooperative-owned bank would compete for exactly CoopHub's custody role.
8. **Federation alignment favours Sterling, not Wema.** CFAN (ex-NACCUN) lists Sterling Bank, CBN, FMBN and IFC as partners and runs its own Unified Co-operative Platform. Wema's visible institutional tie is the Lagos State cooperative ministry (Jul 2024 event).
9. **No public evidence was found of cooperatives switching providers, or of their stated reasons.** The only signals are:
   - vendor testimonials;
   - a reported study finding that >70% of cooperatives still use handwritten ledgers or cash [snippet];
   - a governance failure at the NNPC Staff Cooperative (May 2025);
   - general complaints about bank charges.

   The "why switch" question needs primary interviews (see §8).
10. **Kenya shows what "mature" looks like.**
    - **Co-op Bank Sacco Link:** a movement-owned bank acts as the switch and channel provider (619 SACCOs on FOSA services in 2024; 110 DT-SACCOs on its ATM platform).
    - **Kwara:** a VC-funded core with a member app (≈$7m raised; 200+ SACCOs).
    - **Chamasoft:** cheap ledger-only group software (10K+ Android downloads), with Co-op Bank as an investor.

    Nigeria lacks the cooperative-owned switch. The Cooperative Bank of Nigeria is the proposed equivalent.

---

## 2. Positioning map (relative to CoopHub)

| Axis | CoopHub | Who sits elsewhere |
|---|---|---|
| **Bank-backed vs neutral** | Bank-backed, single-bank (Wema account mandatory) | Neutral: Coopify, Coopco, CoopSolve, Coopra, SmartTeller, CoopLink, Hyves, Esusu Africa, BankCore. Own-licence fintech: Rank (Rank MFB), AjoCard (Save Simple MFB). State/federation: NCDAP/Seamfix, CFAN UCP, planned Cooperative Bank. |
| **Free vs paid** | Paid subscription (monthly/biannual), price unpublished | Published prices: Coopify (free ≤10), BankCore ₦10k+, Coopco ₦15–90k, SmartTeller ₦50–100k, Esusu Africa ₦25k→₦10m. Free to user: bank community accounts (fees via COT/charges), PiggyVest/Cowrywise group features, Rank Circles (fixed fee per slot). |
| **Custody vs ledger-only** | Custody (cooperative's Wema account + member virtual wallets, KYC-tiered limits) | Ledger-only: almost every SaaS vendor. Custody: Rank MFB, AjoCard/Save Simple MFB, PiggyVest/Cowrywise (individual accounts, not pooled), banks' plain accounts. Payment-rail integration without custody: CoopLink (Paystack, Flutterwave). |
| **Governance controls** | Initiator/Verifier/Approver in-app over bank money | Others: role-based admin only (SmartTeller, CoopSolve); member-vote unlock (TrustAjo, pre-launch); bank portal mandates (GTBank GAPS, Access PrimusPlus), which are generic corporate tools. |
| **Payroll check-off** | Not offered | Coopra (staff deduction schedules) [snippet]; Rank Perks (payroll-deducted circles); CoopSolve (auto-deductions); Orbital (payroll/NHF). None documents IPPIS/Remita schedule import. |

---

## 3. Comparison table

Legend: **Y** documented; **P** partial; **N** not offered / not documented; **?** unknown.

| Player | Type | Bank-backed? | Custody | Price | Ledger / reports | Loans | Dual control | Payroll check-off | Scale / traction (date) | Funding | What it lacks vs CoopHub / what CoopHub lacks vs it |
|---|---|---|---|---|---|---|---|---|---|---|---|
| **Wema CoopHub** | Bank coop platform | Y (Wema only) | Y (coop Wema a/c + member virtual wallets) | Paid subscription, amount unpublished [primary] | Y (contributions, statements export, analytics) | Y (configurable products) | **Y** I/V/A | N | 1,000+ coops (Jul 2024) [self-reported]; Android 1K+ dl, iOS 10 ratings (Oct 2026) | Wema balance sheet | Lacks: price transparency, multi-bank, payroll import, guarantor flow, dividend/AGM, recurring contributions ("under development"), CVN integration |
| Sterling (OneBank, Alt Bank) | Bank | Y | Y (retail) | Free retail | Individual | Alt Bank agri/coop finance; ₦250k "Social Mobilization Facility" 9.5% [snippet] | N | N | — | — | No coop governance tool; but **CFAN partner bank** |
| Access Bank | Bank | Y | Y (Community Current Account) | ₦10k opening/operating bal.; ₦3/mille >₦1m debit turnover | Bank statements | Not coop-specific | Portal mandates (PrimusPlus) | N | — | — | Account only, no member layer |
| GTBank | Bank | Y | Y (Unincorporated Societies account) | Standard | Statements | — | GAPS portal | N | — | — | Account only (Uganda has "GTSACCO"; Nigeria doesn't) |
| Polaris | Bank | Y | Y (Community Savings; ₦5k min; ≤4 withdrawals/mo) | Standard | Statements | — | — | N | — | — | Account only |
| ProvidusUnity (Unity + Providus merged, 2026) | Bank | Y | Y | — | — | Unity Farmers Co-op Finance Scheme: 20% cash collateral, up to 5× savings, 12–24m [snippet] | — | N | — | — | Scheme's survival post-merger unknown |
| First Bank / Union / Fidelity / FCMB / Zenith / Keystone / Ecobank / UBA / Stanbic | Banks | Y | Generic society accounts | — | — | — | Portal | N | — | — | **No coop-specific product found** |
| Lotus / TAJ / Jaiz | Non-interest banks | Y | Lotus "Cluster Account with Qard" for coops [snippet] | — | — | TAJ Sukuk targets coops | — | N | — | — | Account/product only |
| LAPO / NIRSAL MFB / Bank of Agriculture | MFB / DFI | Y | — | ~9% NIRSAL [snippet] | — | Group/coop lending | — | N | BOA ">₦250bn, >2m farmers" [self-reported, undated] | State | Credit only, no software |
| **Rank (ex-Moni)** | Fintech + own MFB | Own MFB | Y | Fixed fee by payout slot; zero interest on Circles | App | Salary-backed credit (Perks) | N documented | **Y** (Rank Perks payroll-deducted circles) | >$100m paid out in 12 months (Jul 2026); pilot 10k participants/₦16bn [self-reported] | YC-backed | Not a registered-coop governance tool; but owns licence and workplace channel |
| AjoCard | Agent ajo + MFB | Save Simple MFB | Y (NDIC claimed) | Loan interest | Agent-side | ≤₦100k | N | N | No metrics since 2021; app updated Jul 2026 | $1m seed 2021 | Market traders, not coops |
| LibertyPay | Ajo-collector rails | Super-agent | ? | Loans 4–13%/mo | Card | ₦10k–₦300k | N | Via Remita payroll loans | 150k cards planned (2023) | — | No 2025–26 news |
| PiggyVest (Group Target, Joint Accounts) | Consumer savings | Own MFB/MMO | Individual accounts; joint accts Sep 2026 | Free; 1% early-break | Individual | N | N (rules unpublished) | N | 6m+ users; ₦1.3trn paid out 2025 [self-reported] | $1.1m seed | Co-saving, not pooling; informal groups only |
| Cowrywise (Circles: Challenges/Collections) | Consumer savings/investing | SEC fund mgr | Collections pools [snippet] | 0% plans | Individual | N | ? | N | ~500k active [unverified] | ~$3.7m | No coop governance |
| Thrifto / TrustAjo | ROSCA apps | Unnamed partner bank | TrustAjo escrow (pre-launch: "no real member money has moved") | Free escrow claimed | Y | N | Member vote | N | Thrifto launched Mar 2026; TrustAjo waitlist | — | Unproven |
| **Coopify / ProbityCoop** | Neutral SaaS | N | N ("not a registered bank"; 3rd Quadrant payments) | Free ≤10; per-member/mo; one-off >500 | Y | Y + guarantors | N | N | "500+ coops" [self-reported]; Android 1K+ dl, 5.0★, updated 9 Sep 2026 [primary] | Unfunded (Tracxn) [snippet] | No custody/dual control; clients incl. Sterling & Fidelity staff coops |
| **Coopco** | Neutral SaaS | N | N | ₦15k / ₦30k / ₦90k per month | Y "13 books", audit trail | Y + e-sign guarantor | N | N | "300+ coops" [self-reported] | — | No custody |
| CoopSolve (FirstLincoln) | SaaS + white-label apps | N | N (e-wallet, auto-debit) | Undisclosed | Y double-entry | Y instant | N | P auto-deductions | Aidlife 5,000+ members; PHRC app 100+ dl [primary] | Unfunded [snippet] | Per-coop apps = small reach |
| Coopra | SaaS (staff coops) | N | N | Undisclosed | Y | Y | N | **P** staff deduction schedules [snippet] | ? | — | Unverifiable |
| SmartTeller | SaaS + USSD/POS | N | ? | ₦50k (≤1,000 customers) / ₦100k per month | Y | Y auto-repay | Role-based | N | 1,056 coops, 388 MFBs, 245 thrift collectors [self-reported, undated] | — | Custody unstated |
| BankCore Africa | White-label core/BaaS | N | ? | ₦10k/mo + ₦100k setup | Y | Y | ? | N | "5K+ businesses", "₦500bn+ txns" [self-reported] | — | Generic core |
| CoopLink | SaaS | N | N (Paystack, Flutterwave) | Hidden; annual −35% | Y | Y | N | N | "2,500+ orgs, 1M+ members" [self-reported, **dubious**: generic testimonials, no company named] | — | Credibility |
| SmartCoopHub / Hyves / Clé / Cooperativ / CoopOS / TrustCoop | SaaS | N | N/? | Mostly undisclosed | Y | Y | ? | N | "Thousands" [vague self-reported]; Clé launched 7 Apr 2024 | — | Small/unproven |
| Esusu Africa | BaaS for thrift/coops/MFIs | N | Client's bank | ₦25k/mo → ₦10m lifetime | Y | Y | N | N | 150–180+ institutions, 400–500k savers [self-reported, undated, conflicting] | Gates grant | Not coop-governance |
| Riby | Coop banking app + agents | N | Partner | ? | Y | Y | ? | ? | 350k members onboarded; GEEP delivery 4m+ [self-reported; Condia Mar 2020; Vanguard Jan 2026 snippet] | ~$1m (last 2017) | Quiet since 2020 |
| **NCDAP / Seamfix registry** | Federal identity rail | — | N | "No cost to governments"; user fees unpublished | Identity + CoopCHECK bureau | N | — | N | 370k+ coops, 30m+ members targeted [self-reported, FMINO 7 Oct 2026]; enrolment numbers unpublished | PPP (Seamfix) | Not a ledger; becomes a gate |
| **Cooperative Bank of Nigeria** | Planned bank | Planned | Planned | Shares ₦1m–₦50m per society | — | Planned | — | — | ₦200bn target, 10k societies; subscription from end-Oct 2026 [snippet] | Coops 65% / investors 30% / staff 5% | Plan only; licence category unannounced |
| CFAN Unified Co-operative Platform | Federation MIS | Sterling partner | N | ? | Planned | Coop Dev Fund | — | — | 5.2m members, 2,040 groups [self-reported] | — | Vendor/status unknown |
| KENYA: Co-op Bank Sacco Link | Movement bank switch | Y | Y | Switch fees | — | — | Y | Statutory check-off | 619 SACCOs on FOSA (2024); 110 DT-SACCOs ATM (2025) | — | Benchmark |
| KENYA: Kwara | SACCO core + app | N | N | Annual subscription | Y | Y + online guarantors | P | Y (SASRA) | 200+ SACCOs (KE/ZA/PH) [self-reported] | ≈$7m (seed ext. $3m Jan 2023) | KUSCCO channel collapsed (liquidation 2025–26) |
| KENYA: Chamasoft | Chama ledger | N (bank-linked) | N | Free → KSh 1,000/mo | Y auto-reconciles Equity alerts | Y | Bank mandate | N | Android 10K+ dl (Oct 2026); 13 staff (Apr 2026) | Co-op Bank Kenya, Village Capital | Benchmark |

---

## 4. Per-player notes

### 4.1 Wema CoopHub (baseline)
- **Launch:** 10 May 2024, Wema's 79th anniversary. CEO Moruf Oseni pitched "100 percent transparency" [BusinessDay, 16 May 2024: https://businessday.ng/companies/article/wema-bank-targets-cooperative-societies-growth-with-coophub/; Nairametrics, 11 May 2024].
- **Traction:** "over 1,000 cooperative societies" onboarded, announced at the Lagos State Ministry of Commerce, Cooperatives, Trade & Investment's International Day of Cooperatives event [Daily Trust, 9 Jul 2024: https://dailytrust.com/1000-cooperatives-onboarded-on-wema-banks-coophub/] [self-reported]. **No later figure found** in 2025–26 searches.
- **Terms [primary, coophub.alat.ng/terms-and-condition, accessed 9 Oct 2026]:**
  - eligibility requires the society's registration documents and bye-laws, an account with Wema Bank, and payment of "the subscription fee for access to the platform";
  - the account must be operated with an initiator, verifier and approver: "multiple signatory approvals (Initiator, Verifier, Approver) before transactions are honoured";
  - subscriptions auto-renew monthly or biannually; price changes need notice.
- **FAQ [primary]:**
  - a non-Wema account cannot be used;
  - members join only by invite link;
  - members hold a "virtual wallet" with withdrawal limits tied to KYC level;
  - daily transfer limits follow the cooperative's Wema account;
  - "Incentives" can be withdrawn or reinvested;
  - members can request liquidation of contributions (admin can decline);
  - statements of account and transaction history can be exported;
  - loans are configured by the cooperative;
  - recurring contributions are "still under development";
  - the dashboard cannot be customised.
- **Apps [primary, 9 Oct 2026]:**
  - Android `alat.ng.coophub`: 1K+ downloads, 4.2★, ~23 reviews, released 9 May 2024, updated 3 Oct 2025.
  - iOS id6502437864: 4.9★ from 10 ratings, v1.1.6 (6 Oct 2025).
- **Gaps:** no payroll/IPPIS/Remita import, no guarantor workflow documented, no dividend/AGM module, no public API, no published price, single-bank lock-in, no CVN/CoopID integration announced.

### 4.2 Other banks
- **Sterling.**
  - OneBank FAQ: "Community banking" (create or join a community) and "Ajo" group savings. These are retail tools [snippet: https://sterling.ng/onebank/faqs/].
  - Alt Bank (non-interest): agri finance backing "cooperatives and registered farmers", plus a ₦250k Social Mobilization Facility at 9.5% p.a. markup [snippet, undated: https://altbank.ng/business/].
  - Embedly (Sterling BaaS): 246+ clients, 79k+ wallets [self-reported; see bank_integration_options.md].
  - **Named partner of CFAN** [https://www.cfan.coop/about-cfan/]. Sterling Bank Staff Cooperative uses Coopify, not a Sterling product.
  - Position: the best-placed bank to copy CoopHub (federation tie plus BaaS stack), but no product yet.
- **Access Bank.** Community Current Account for registered clubs/societies/associations/churches:
  - ₦10,000 opening/operating balance, no interest;
  - "no COT" headline, but ₦3/mille on debit turnover above ₦1m;
  - needs constitution, signatory mandate, CAC certificate and TIN [https://www.accessbankplc.com/personal/everyday-accounts/community-current-account, accessed Oct 2026].
  - Community Basic Account for unregistered groups. PrimusPlus corporate portal has maker-checker. No coop app.
- **GTBank.** Unincorporated Societies/Clubs/Associations account form (registration certificate, constitution, references, TIN, SCUML) [https://gtbank-plc.files.svdcdn.com/production/download-forms/Account-Opening-Form-Unincorporated-Societies.pdf]. GAPS portal gives Uploader/Reviewer/Approver. GTBank Uganda's "GTSACCO" shows the group could productise this but has not in Nigeria.
- **Polaris.** Community Savings Account: ₦5,000 minimum, ≤4 withdrawals a month, constitution or minutes required [https://www.polarisbanklimited.com/polaris-community-savings-account/]. SurePadi agency banking reportedly serves cooperatives [snippet]. VULTe API banking offers direct debits and lien accounts.
- **ProvidusUnity.** Unity Bank's "Unity Farmers Co-operative Finance Scheme":
  - 20% savings as cash collateral, borrow up to 5× savings;
  - members counter-guarantee each other;
  - cooperative must be registered ≥6 months; 12–24 month tenor [snippet].
  - The Unity–Providus merger cleared its final court approval in 2026 [BrandSpur, 2 Jun 2026: https://brandspurng.com/2026/06/02/supreme-court-approves-unity-bank-providus-bank-merger-clears-final-legal-hurdles/]. Whether the scheme survives is unknown.
- **First Bank, Union, Fidelity, FCMB, Zenith, Keystone, Ecobank, UBA, Stanbic.** Nothing cooperative-specific found.
  - Firstmonie is agent banking (Agent Credit ₦571bn cumulative) [InvestorsKing, 2 Jun 2025].
  - Fidelity YEIDEP mentions cooperatives [snippet, page 404].
  - Escrow/trustee arms (FirstBank Trustees, FCMB Trustees) are relevant for cooperative housing, not management.
- **Non-interest banks.** Lotus "Cluster Account with Qard" for cooperatives and associations [snippet]. TAJBank Sukuk lists cooperatives as target investors [snippet]. Jaiz: nothing specific.
- **"Cooperative banking desks":** no evidence of a dedicated desk at any commercial bank.

### 4.3 Microfinance banks and DFIs
- **LAPO MfB:** group lending, mostly to women micro-entrepreneurs; no digital coop tooling.
- **Accion MfB, AB MfB:** nothing for client cooperatives.
- **NIRSAL MFB:** lends via cooperatives; collateral-free women's group loans (5–10 members), ~9% [snippet, nairacompare 2026].
- **Bank of Agriculture:** cooperative lending, ">₦250bn disbursed, >2m farmers" [self-reported, undated].
- These are credit sources that a CoopHub cooperative might borrow from. They do not compete on management software.

### 4.4 Rank (ex-Moni): closest fintech competitor
- **Nov 2025:** rebranded, acquired AjoMoney and Zazzau MFB (→ Rank MFB) [Disrupt Africa, 12 Nov 2025: https://disruptafrica.com/2025/11/12/nigerias-moni-rebrands-as-rank-acquires-ajomoney-and-zazzau-mfb/; Ecofin, 12 Nov 2025]. Pilot: 10k participants, ₦16bn payouts [self-reported].
- **Jul 2026:** launched three products [Disrupt Africa, 27 Jul 2026: https://disruptafrica.com/2026/07/27/nigerias-rank-launches-3-new-community-powered-financial-services-products/]:
  - **Money Circles:** digital ROSCA, fixed fee by slot, zero interest;
  - **Tribe:** private groups for family/friends/colleagues;
  - **Rank Perks:** employer payroll-deducted savings circles plus salary-backed credit.
  - Claims >$100m paid out in 12 months [self-reported]. YC-backed.
- **Why it matters:** it owns the licence (custody), sells into workplaces via payroll (CoopHub's staff-cooperative heartland) and has momentum. It lacks registered-cooperative features: bye-laws, AGM, dividends, multi-signatory governance, CVN.

### 4.5 Other ajo / group-savings apps
- **PiggyVest:**
  - Group Target is co-saving in individual accounts (up to 12% p.a.; 1% early-break fee).
  - Joint Accounts and Children's Accounts launched 26 Sep 2026; approval/withdrawal rules unpublished [BusinessDay, 28 Sep 2026].
  - 6m+ users; ₦1.3trn paid out in 2025 vs ₦835bn in 2024 [Techpoint, Jan 2026] [self-reported].
- **Cowrywise:** Circles: Challenges (individual) and Collections (pooled) [snippet]. ~$3.7m raised (Quona-led seed, Jan 2021) [Tracxn snippet].
- **AjoCard:** agent ajo for traders; Save Simple MFB, NDIC claimed; $1m seed (Jul 2021); app updated Jul 2026; no metrics.
- **LibertyPay:** 1,000+ POS and USSD ajo collection; loans ₦10k–₦300k; no 2025–26 news [TechCabal, 19 Sep 2023].
- **Thrifto:** "bank-integrated" ROSCA web app with Trust Rating Score; launched Mar 2026; custody partner unnamed [Newsmart snippet].
- **TrustAjo:** escrow with an unnamed partner bank; still waitlist ("No real member money has moved yet") [primary, trustajo.com, Oct 2026].
- **Kuda / OPay / Moniepoint:** no group product found [unverified].
- **These compete for informal groups that never register.** They do not serve registered cooperatives' governance needs. The real "instead" product is a **personal or plain bank account plus WhatsApp plus a notebook or Excel**. A study cited by TechEconomy says >70% of cooperatives use manual ledgers and cash [snippet, date unknown]. WhatsApp is on 95% of Nigerian smartphones [Orange/KPMG via BrandSpur, 26 Sep 2026].

### 4.6 Neutral cooperative SaaS
- **Coopify / ProbityCoop (Probity Software, Lagos):**
  - Cloud, LAN or embedded deployment; guarantor verification, SMS, portal, audited reports.
  - Free for ≤10 members; per-member/month Standard; one-off Premium >500.
  - "Trusted by over 500 cooperatives" [self-reported]; clients Sterling Bank Staff Coop, Fidelity Bank Credit MPCS, Irorun MPCS.
  - "not a registered bank"; payments via 3rd Quadrant; unfunded [Tracxn snippet].
  - Android 1K+, 5.0★, updated 9 Sep 2026 [primary].
  - **The most visibly maintained neutral competitor, with bank-staff cooperatives as clients**, which shows banks' own staff coops don't default to bank products. [https://coopify.africa; https://probitysoftware.com/coop.html]
- **Coopco (Ayobo, Lagos):** ₦15k/₦30k/₦90k per month; "13 books of accounts", inventory, audit trail, e-signature guarantor approval; "300+ cooperatives" [self-reported]. [https://coopco.com.ng]
- **CoopSolve (FirstLincoln Technologies):**
  - Double-entry accounting, auto-debit savings, e-wallet; builds white-label apps per society (PHRC Cooperative, NNPC PH refinery thrift: 100+ downloads, updated 8 Oct 2025 [primary]).
  - Aidlife 5,000+ members, "60% faster loan processing" [self-reported]; ATBU Staff Coop. Unfunded [snippet]. [https://coopsolve.com]
- **Coopra:** staff-cooperative focus incl. deduction schedules and dividends [snippet; site JS-only].
- **SmartTeller (smartteller.tech):** web, apps, USSD and POS "Box"; ₦50k/month (≤1,000 customers), ₦100k unlimited; claims 1,056 cooperatives, 388 MFBs, 245 thrift collectors [self-reported, undated]. Custody unstated.
- **BankCore Africa:** white-label core; ₦10k/month + ₦100k setup; "5K+ businesses" [self-reported].
- **CoopLink (cooplink.io):** Paystack/Flutterwave integration, white-label subdomain, PWA; claims 2,500+ orgs and 1M+ members. **Treat as dubious:** no company or founders named, generic testimonials.
- **Others:** SmartCoopHub (Abuja, "thousands" [vague]), Hyves (cooperative ERP with credit scoring), Clé (launched 7 Apr 2024, web + USSD, dividends [BusinessDay]), Cooperativ (early access, multilingual), CoopOS (WhatsApp-native), TrustCoop, Coopvest/Coopmoni (Ibadan).
- **Riby:** ~$1m raised (last 2017); 350k coop members onboarded; ran GEEP delivery [Condia, 2 Mar 2020; Vanguard Jan 2026 snippet]. Quiet as a coop product.
- **Esusu Africa:** BaaS for thrift/coops/MFIs; ₦25k/month → ₦10m lifetime; 150–180+ institutions [self-reported, conflicting].
- **Dead or not found:** Prolean (DNS dead), Armony/MEMCOS (dead), CoopManager (Lainos, desktop legacy), Thrift Pro, MyCoop (Indonesian app), Cooperative Hub, Kooperatives, Jamii, Gloopay, Tricoop, Koopfund, CoopAfrica.
- **What SaaS lacks vs CoopHub:** custody, bank-enforced dual control, instant transfers. **What CoopHub lacks vs SaaS:** published price, bank neutrality, full double-entry books (Coopco, CoopSolve), guarantor e-sign (Coopco, Coopify), inventory/store (Coopco, Coopify), offline/LAN (Coopify), USSD (SmartTeller, Clé), payroll deduction schedules (Coopra).

### 4.7 Government and federation
- **Ownership of cooperatives:** the Federal Ministry of Agriculture & Food Security; the Minister of State is "Supervising Minister of Cooperative Affairs" [agriculture.gov.ng, Aug 2026].
- **RH-CRRP 2025–2030:** approved Mar 2026; seven pillars incl. digitalisation, cooperative finance, SACCO model [AllAfrica, 18 May 2026; VON, 12 May 2026].
- **Seamfix / NCDAP:**
  - MoU signed 12 Aug 2026 by the Ministry, Cooperative Federation of Nigeria (CFN) and Seamfix [Nairametrics, 17 Aug 2026: https://nairametrics.com/2026/08/17/seamfix-federal-government-and-cfn-sign-agreement-to-launch-national-digital-identity-infrastructure-for-nigerias-cooperative-sector/].
  - Components: NCSR (registry), CVN (society), CoopID (member, NIN-linked), CoopCHECK (credit bureau with CreditRegistry). USSD planned.
  - Six-month revalidation from 12–13 Aug 2026 (≈ mid-Feb 2027) [Vanguard, Aug 2026].
  - "4Ps" PPP at "no cost" to governments; user fees unpublished [FMINO, 7 Oct 2026].
  - Sector claim: 370k+ cooperatives, 30m+ members, ₦1.7trn turnover [self-reported].
  - States: Kogi digitising (4 Sep 2026); Oyo joined; FG charged all 36 states (Aug 2026). Lagos has 16k+ societies but no e-portal found.
- **Cooperative Bank of Nigeria:**
  - Flag-off 11 May 2026; "government-enabled, not government-funded".
  - Ownership 65% coops (via CoopTrust) / 30% investors / 5% staff.
  - ₦200bn drive launched 18 Jun 2026, with three subscription tiers:
    - 1,000 societies at ₦21–50m;
    - 3,000 at ₦16–20m;
    - 6,000 at ₦1–15m.
  - Subscription targeted from end-Oct 2026 [Authority, 18 Jun 2026; Leadership snippet]. CBN licence category not announced.
- **CFAN:** National Cooperative Financing Agency (ex-NACCUN, 1978; rebranded 2015); Unified Co-operative Platform; partners Sterling, CBN, FMBN, IFC [https://www.cfan.coop/about-cfan/]. **CFN** (apex federation; president Hannatu Mershak) is the Seamfix MoU signatory.
- **Implication for CoopHub [inference]:**
  - The registry is a gate it must integrate with: a CVN may become the KYC anchor for opening cooperative accounts.
  - The Cooperative Bank is a direct long-run competitor for custody.
  - CoopCHECK could commoditise the member credit data CoopHub's loan module would otherwise own.

### 4.8 Kenya benchmarks (brief)
- **Co-op Bank Kenya (Sacco Link):** switch giving SACCOs ATM/POS/mobile/internet channels; 619 SACCOs linked for FOSA (2024) [Business Daily]; largest DT-SACCO ATM provider with 110 (2025); Saccolink card campaigns Dec 2025–Jan 2026 [co-opbank.co.ke]. It is a *movement-owned* bank, which is what the Cooperative Bank of Nigeria aspires to be.
- **Kwara:**
  - SACCO core banking plus member app (Kwara Connect: deposits, repayments, statements, mobile loans, online guarantors).
  - ≈$7m raised; $3m seed extension (Jan 2023) with an exclusive KUSCCO distribution deal to 4,000+ SACCOs [TechCrunch, 16 Jan 2023]; 200+ SACCOs in Kenya, South Africa and the Philippines [Tracxn].
  - KUSCCO's ~KSh 12bn fraud and liquidation vote (2025) undermined that channel [Capital FM, Feb 2025; Kenyan Wall Street].
  - Lesson: distribution via an apex body is fragile if the apex is weak.
- **Chamasoft:** chama bookkeeping since 2013; auto-reconciles Equity Bank alerts; Co-op Bank Kenya and Village Capital investors; ~13 staff (Apr 2026); Android 10K+ downloads, updated 22 Sep 2026 [primary; Tracxn]. Shows ledger-only software can scale cheaply when bank alerts are machine-readable.
- **What "mature" means:** statutory payroll check-off, regulator returns (SASRA), a movement-owned switch bank, and member apps with online guarantors. CoopHub has only the bank-custody and member-app parts.

---

## 5. Who is gaining traction (dated evidence only)

| Player | Evidence | Date | Strength |
|---|---|---|---|
| Rank | Rebrand + MFB acquisition; 3 group products incl. payroll circles; >$100m paid out/12m | Nov 2025, Jul 2026 | Strong momentum [self-reported] |
| Federal NCDAP/Seamfix | MoU, revalidation launched, states onboarding (Kogi, Oyo) | Aug–Oct 2026 | Strong (policy-mandated) |
| Cooperative Bank of Nigeria | ₦200bn drive; subscription end-Oct 2026 | Jun–Oct 2026 | Plan; execution unproven |
| PiggyVest | Joint Accounts; ₦1.3trn payouts 2025 | Jan & Sep 2026 | Strong, but informal groups only |
| Coopify | App updated Sep 2026; bank staff-coop clients | Sep 2026 | Steady, small |
| Wema CoopHub | 1,000+ coops; apps last updated Oct 2025 | Jul 2024 / Oct 2025 | **Stalled publicly** |
| SmartTeller, CoopLink, Coopco, SmartCoopHub, Hyves | Undated self-reported counts | — | Unverifiable |
| Kwara (KE) | Funding 2023; KUSCCO channel collapsed | 2023–2025 | Weakened channel |

---

## 6. What each lacks (summary)
- **CoopHub:** price transparency; works with Wema only; no payroll/IPPIS/Remita reconciliation; no guarantor e-sign or dividend/AGM tooling documented; recurring contributions unfinished; no CVN/CoopID link; small app footprint; no update since Oct 2025.
- **Other banks:** any member layer at all. They sell accounts, not cooperative operations.
- **Neutral SaaS:** custody, enforced dual control over real money, credible scale data, funding (most unfunded), payroll-rail integration.
- **Rank / ajo apps:** registered-cooperative governance (bye-laws, multi-signatory, dividends, regulator returns); PiggyVest/Cowrywise don't pool money.
- **State platforms:** a ledger, payments and member services. They are identity and credit rails, and the bank is only a plan.

## 7. Where cooperatives say they would switch, and why
- **No direct, dated evidence was found** of cooperatives publicly comparing or switching between CoopHub and alternatives (Nairaland/X/LinkedIn searches returned nothing usable; Nairaland blocked).
- **Indirect signals:**
  1. **Trust and governance failures drive demand for controls.** NNPC Staff Cooperative: ₦806m loss in 2022, no AGM for six years, retirees unable to withdraw [Vanguard, 30 May 2025]. This argues for CoopHub-style dual control.
  2. **Bank-staff cooperatives choose neutral SaaS.** Sterling and Fidelity staff coops are on Coopify [self-reported], suggesting price and bank-neutrality matter more than custody for staff coops whose money is payroll-deducted.
  3. **Charges sensitivity.** General complaints about unexplained bank deductions and cash/ATM fees [Techpoint digest] are a reason not to be locked to one bank.
  4. **Policy push.** Mandatory revalidation and CVN may force digitisation in 2026–27, and cooperatives will likely pick tools that integrate it [inference].
  5. **Workplace channel.** Rank Perks' payroll-deducted circles target the same staff-cooperative use case with zero interest and instant payout [inference from product design].
- **Recommended primary research:** 10–15 interviews with cooperative secretaries/treasurers (staff coops on IPPIS, Lagos thrift coops, farmer coops). Ask for:
  - current tool and bank;
  - CoopHub awareness and price quoted;
  - deal-breakers (single-bank lock-in, fees, payroll reconciliation, revalidation burden).

## 8. Open questions
- CoopHub subscription price, active-cooperative count in 2025–26, white-label clients. Ask Wema.
- Whether CoopHub roles are enforced in core banking or only in the app (see bank_integration_options.md §A).
- Seamfix/NCDAP user fees; whether CVN will be required for bank account opening.
- Cooperative Bank licence type and launch date.
- Rank Perks employer count and whether it targets registered staff cooperatives.
- Install counts for Rank, SmartTeller, Coopco apps (package IDs not located).

## 9. Corrections to earlier notes in this folder
- `competitive_landscape.md` lists CoopHub pricing as "Undisclosed". It is a **paid subscription** (monthly/biannual) per its Terms, with the amount unpublished.
- `competitive_landscape.md` lists scale as "Undisclosed". The public figure is 1,000+ cooperatives (Jul 2024), as already noted in bank_integration_options.md.
- Coopco's self-reported scale is "300+ cooperatives" (new).
- Unity Bank is now ProvidusUnity (merger final in 2026).
- The Seamfix MoU co-signatory is **CFN** (Cooperative Federation of Nigeria), not CFAN.
