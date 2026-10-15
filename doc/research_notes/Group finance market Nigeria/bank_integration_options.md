# Bank integration options for a Nigerian cooperative custody + payroll check-off platform

Compiled 9 October 2026. Scope: how a software platform (not itself a CBN licensee) can connect to Nigerian banks and BaaS/payment providers to run (a) accounts held in a third party's name with bank-enforced maker-checker, (b) virtual/dedicated accounts per employer or member with webhooks, (c) outbound transfers with multi-approver workflows via API, (d) statement and transaction feeds, (e) NIBSS direct-debit e-mandates, (f) escrow/trust accounts, (g) float/interest on pooled balances.

Method note: ~75 web searches plus direct fetches of provider docs and press. Several primary pages blocked automated fetches (coophub.alat.ng T&C, paystack.com docs, developers.monnify.com, monnify.com/pricing, kuda.com FAQ, ubagroup.com developer zone, firstbanknigeria.com, techcabal.com, thecable.ng, hydrogenpay support, vbaas FAQ). For those, figures come from search-engine extracts of the same pages or from secondary press and are marked **[SECONDARY]**. Items marked **[GAP]** could not be verified. Items marked **[CONFIRM WITH BANK]** are design-critical and must be checked in a commercial conversation before architecture is frozen. Pricing excludes 7.5% VAT unless stated.

Related notes in this folder: `enablers_and_constraints.md` (licensing, KYC tiers, NDPA), `payroll_checkoff_*.md` (IPPIS, Remita, states).

---

## 0. Executive takeaway

1. **No Nigerian bank publicly exposes corporate maker-checker (initiator/verifier/approver) as an API.** Multi-approver controls exist on every corporate internet-banking portal (GTBank GAPS, Access PrimusPlus, UBA CIB, Wema ALAT for Business) and on Wema's CoopHub, but the public developer portals (Providus, VFD, Kuda, UBA, GTBank, Zenith, ALAT playground, Anchor, Embedly) expose single-credential transfer APIs where the *platform* is the sole approver. If the bank must enforce dual control, the realistic patterns are: (i) bank portal does approvals and the platform only prepares/uploads batches and reads statements (semi-manual), (ii) platform enforces approvals internally and the bank API executes (platform is trusted), or (iii) a negotiated, non-public bank arrangement (Wema CoopHub-style) where roles live in the bank's system. **[CONFIRM WITH BANK]** for Wema (CoopHub white-label/API) and Providus/Sterling (enterprise BaaS contracts).
2. **Virtual accounts with webhooks are commoditised.** Providus, Wema/ALAT, VFD, Kuda, UBA, Sterling/Embedly, Anchor, Monnify, Paystack (Wema + Titan), Flutterwave, Hydrogen (Access), Squad (GTBank) all offer per-customer static NUBANs with HTTP callbacks. Published fees range from ₦10 flat (Maplerad) and 0.2% capped ₦200 (Hydrogen) to 0.3% capped ₦80 (Kuda) to 1% capped ₦300 (Paystack) to 2% uncapped (Flutterwave). Bank-direct pricing (Providus, Wema, VFD, Sterling) is negotiated.
3. **Outbound NIP via API** is widely available; published per-transfer costs: ₦9–15 (Kuda API), ₦10 flat (Maplerad, VFD), ₦10/₦25/₦50 bands (Paystack, Flutterwave, Providus RIB). Add ₦50 stamp duty on the *sender* for ≥₦10,000 since 1 Jan 2026 (salary payments exempt). The CBN draft 2026 Guide to Charges (consultation closed 8 May 2026) would make transfers ≤₦5,000 free, ₦10 for ₦5,001–50,000, ₦50 above.
4. **Direct debit** is NIBSS NDD; the ₦50 e-mandate transfer is the standard activation. Aggregators: Mono (1% capped ₦1,000, min ₦55), Paystack (1.5% + ₦100 capped ₦2,000), Monnify (1.5% capped ₦2,000 **[SECONDARY]**), Remita (₦100 per debit **[SECONDARY]**), Zenith Z-Mandate, UBA (needs PSSP), Interswitch. Direct NIBSS integration requires a bank/OFI/PSSP relationship and ₦30 per successful debit.
5. **Statements**: no Nigerian bank publishes a webhook-plus-MT940 package for small corporates; Zenith offers SWIFT FIN/FileAct MT940/MT101 for large corporates; Providus, UBA, Access, VFD and Polaris VULTe expose statement REST APIs; Mono sells 1–12 month statement pulls (from ₦15,000/month plan). Open banking (CBN framework) is still not live as of Oct 2026.
6. **Escrow** is a bank/trustee product (Stanbic IBTC, FirstBank Trustees, FCMB Trustees) or a licensed fintech (Vesicash). There is no self-serve bank escrow API.
7. **Float**: current-account interest is "negotiable" under the CBN Guide to Charges; savings minimum is 30% of MPR (MPR 23% from 22 Sep 2026 → 6.9%). Bank fixed deposits quoted 15–16% (GTBank/Access/Sterling), Providus up to 23.66%; 364-day T-bills 15.89% at the 7 Oct 2026 auction. A cooperative platform cannot itself hold member money (PSSP holding-funds rule); interest must accrue to the cooperative's own account.
8. **Regulatory overhangs**: April 2024 onboarding freeze on OPay/Moniepoint/Kuda/PalmPay/Paga (KYC + FX), 46 MFB licence revocations 1 Jul 2026 (partner-bank risk), CBN circular PSS/DIR/PUB/CIR/001/004 of 15 Jun 2026 (data localisation by 1 Jan 2027, UBO disclosure, market-share caps), AML automation roadmap due 10 Jun 2026, NDIC pass-through cover only for properly documented bare-trust pool accounts.

---

## A. Accounts in a third party's name with bank-enforced maker-checker

### Takeaway
Banks enforce signing mandates ("any two to sign") on the account itself and replicate them as role-based approvals in their corporate portals. The only *cooperative-specific* bank product found is Wema CoopHub, which hard-codes Initiator → Verifier → Approver and is marketed as white-labelable. No bank advertises this control surface as an API. A platform wanting bank-enforced dual control today must either live inside a bank portal/white-label product or negotiate a bespoke integration.

### Cited findings
- **Wema Bank CoopHub** (launched 10 May 2024 for the bank's 79th anniversary; Nairametrics 11 May 2024; Vanguard review May 2024; Daily Trust 9 Jul 2024). Roles per the CoopHub T&C (as extracted by search): "The Cooperative admin comprises of the Initiator, Verifier and Approver"; Verifier is "the second line of defence"; Approver is "responsible for the final disbursement"; "No single person can make a withdrawal on their own without the concurrence of a Verifier and Approver." Features: member onboarding, contributions ("make, track and even withdraw contributions"), custom loan products, analytics, announcements, admin web portal at coophub.alat.ng, member apps on Play/App Store, "option of white labelling for a customised experience" (Google Play listing, as indexed). 1,000+ cooperatives onboarded by 9 Jul 2024 (Daily Trust). **No public API, pricing, or employer-deduction feature found.** [GAP: whether the cooperative's funds sit in a Wema corporate account in the cooperative's name, and whether the roles are enforced at core-banking level or only in the CoopHub app — CONFIRM WITH BANK]. Sources: https://nairametrics.com/2024/05/11/wema-bank-launches-coophub-digital-solution-for-cooperative-societies/ ; https://www.vanguardngr.com/2024/05/a-review-of-coophub-wema-banks-digital-solution-for-cooperative-societies/ ; https://dailytrust.com/1000-cooperatives-onboarded-on-wema-banks-coophub/ ; https://coophub.alat.ng/terms-and-condition (blocked to fetch).
- **GTBank GAPS** (corporate bulk payments): roles Uploader/Reviewer/Approver; "no single user can commence and conclude a transaction except otherwise stated on the account mandate"; approver limits; hardware tokens for approvers; local transfer limit ₦250m per transaction (GT regional pages/GAPS brochure). Source: https://www.gtbank.com/business-banking/services/e-business-services/gaps-lite ; https://www.scribd.com/document/636942531/GAPS-Brochure
- **Access Bank PrimusPlus**: "maker checker, transaction authorization", roles Initiator/Authorizer/Viewer, "customizable approval flows and flexible authorization levels". Source: https://www.accessbankplc.com/primus/ ; https://baseontechs.com/primusplus-access-bank/
- **UBA Corporate Internet Banking**: initiator → verifier → authorizer; "multilevel approvals where you can specify the number of approvals". Source: https://www.ubagroup.com/nigeria/personal-banking/digital-banking/internet-banking/
- **Signing mandates**: GTBusiness corporate account opening form (Jan 2026) and Fidelity corporate form record "single signatory / two to sign / combinations"; a board resolution fixes signatories and mandate. Source: https://gtbank-plc.files.svdcdn.com/production/general/GTBusiness-Form-Corporate_Jan-2026.pdf
- **Cooperative account opening**: banks "no longer permit cooperative activities to run on personal savings accounts"; a registration certificate (state Director of Cooperatives under the Nigerian Co-operative Societies Act, or CAC for incorporated trustees) plus TIN, signatory IDs/BVNs is required. Source: https://cacregister.com.ng/blog/how-to-register-a-cooperative-society-or-thrift-group-under-cac
- **Why a third party cannot simply hold the money**: CBN prohibits PSSPs from holding funds; once a provider lets users "hold a balance, use the balance for multiple future payments, or receive payments into that balance" the activity is e-money/banking and needs an MMO/PSB/MFB/bank licence (Balogun Harold, analysis of Zap by Paystack). Lower-risk structure: "settlement bank controls accounts independently, with PSSP having no legal title." Source: https://www.balogunharold.com/insights/paystack-and-the-holding-funds-rule-implications-for-a-pssp

### Gaps
- No evidence that any bank exposes approval-matrix state (pending/approved/rejected) via API to a third-party platform. **[CONFIRM WITH BANK]** for Wema (CoopHub white-label), Providus (enterprise), Sterling (SeaBaaS/Embedly) and Access (Hydrogen) whether a "platform initiates, bank-side officers approve in bank portal" flow can be contracted.
- CoopHub pricing, limits, interest on cooperative balances, and whether it supports employer-level virtual accounts: [GAP].

---

## B. Virtual / dedicated accounts per employer or per member, with webhooks

### Takeaway
Every bank BaaS desk and every PSSP offers static (reserved) NUBANs with HTTPS webhooks; some add dynamic (single-use, ~30-minute) accounts. The design choice is bank-direct (Providus, Wema/ALAT, VFD, Sterling/Embedly, UBA, GTBank/Squad, Access/Hydrogen; negotiated price, PSSP/MMO licence often required) versus aggregator (Paystack, Flutterwave, Monnify, Anchor, Maplerad, Kuda API; self-serve, published price, no licence needed for a registered business but settlement goes to the aggregator wallet first).

### Cited findings (per provider)
- **Providus Bank** — Banking APIs page lists Transfer API, Balance API, Account Statement API, Transaction Notification API, Digital Collection Service (virtual accounts "with instant settlement and easy reconciliation"); developer portal developer.providusbank.com (token auth; endpoints for NIP account details, bank list, Providus account, account statement, BVN). Pricing not published; charges page shows RIB/CIB interbank ₦10/₦25/₦50 and VPS "Pay With Transfer" ₦50 MSC + ₦50 stamp duty per ₦10,000+ transaction. History: Feb 2021 Providus deactivated PiggyVest/Monnify virtual accounts after a CBN intervention (Nairametrics 7 Feb 2021) — a reminder that bank-direct virtual accounts carry regulator-driven switch-off risk. Sources: https://www.providusbank.com/banking-apis ; https://developer.providusbank.com/digital-collection-services ; https://providusbank.com/charges ; https://nairametrics.com/2021/02/07/providus-bank-deactivates-virtual-account-numbers/
- **Wema / ALAT developer portal** (playground.alat.ng; Azure API Management) — Wallet Services product: Wallet Creation, Credit Wallet, Debit Wallet, Bills Payment, Transaction Notification, Account Management APIs; "these wallets operate, hold funds and transact like tier 1 accounts" (so Tier-1 limits apply: ₦30,000/day, ₦300,000 balance per CBN KYC tiers); NIN required; separate Virtual Accounts product; "Pay with Bank Account" API; go-live requires applying for production credentials and a partner unique ID. Pricing not published. Wema is also Paystack's DVA bank. Sources: https://playground.alat.ng/product-wallet-services ; https://alat.ng/developer/ ; https://playground.alat.ng/api-pay-with-bank-account
- **VFD Microfinance Bank (VBaaS)** — docs at vbaas-docs.vfdtech.ng; portal vbaas.vfdtech.ng; sandbox vbaas-dev.vfdtech.ng; Wallets API (individual and corporate wallets, each "backed by a VFD Microfinance Bank account"), Transfer API (intra/inter-bank, credit simulation), virtual accounts (fixed-amount or any-amount funding), QR, loans, biller APIs; NDIC-insured. Fees per VBaaS FAQ (as indexed): account creation ₦10, outward transfer ₦10 flat, inward 0.3% capped ₦500, BVN ₦15–20, liveness ₦60 **[SECONDARY]**; "rates are transaction-based and negotiated". Corporate wallet KYB: CAC docs, TIN, signatory IDs, utility bill, board resolution, 2 reference forms, BVNs, ₦10,000 corporate search fee. Sources: https://vbaas-docs.vfdtech.ng/docs/intro/ ; https://vbank.ng/open-banking/ ; https://www.buildstudio.com.ng/apis/vfd-api
- **Sterling Bank** — Embedly (embedly.sterling.ng; docs-embeddedfinance.embedly.ng; © Sterling Bank Ltd 2026): organisation and business virtual wallets, customer wallets with KYC, payouts (inter/intra-bank "for loan disbursements, vendor payments, salary payouts"), cards, overdraft; claims 246+ clients, 79k+ wallets, ₦251bn+ processed; named clients include Seedfi, EduBanc, HealthBanc. SeaBaaS (Peerless; Sterling migrated Sep 2024) is the core-banking layer with "open APIs" — no public third-party product sheet. OneBank "Ajo Scheme" is a consumer rotating-savings feature, not a cooperative product. Pricing: /pricing link exists, no figures captured [GAP]. Sources: https://embedly.sterling.ng/ ; https://docs-embeddedfinance.embedly.ng/api-reference/wallets ; https://sterling.ng/seabaas/ ; https://dailytrust.com/sterling-adds-ajo-scheme-to-savings-platform/
- **UBA** — Virtual Account product ("multiple virtual accounts for customers, vendors, or projects, all linked to your main account"); Developer Zone (developer.ubaopenbanking.com) with Gold/Silver/Bronze/Unlimited tiers; **Direct Debit, Transfer, Statement and Balance Enquiry APIs require a CBN PSSP or MMO licence plus a UBA corporate account** (some also require collateral) **[SECONDARY]**. Sources: https://www.ubagroup.com/nigeria/personal-banking/accounts/virtual-account/ ; https://www.ubagroup.com/nigeria/about-us/initiatives/developer-zone/
- **GTBank / HabariPay Squad** — GTBank developer portal (developer.gtbank.com; IBM API Connect) offers Identity (NUBAN auth, BVN, NIP name enquiry), Payments (GT and other-bank transfers, bills), Virtual Cards; no virtual-account product on the bank portal. Squad (HabariPay) sells static and dynamic virtual accounts; dynamic "pool" accounts are "a restricted service for selected businesses" needing profiling via help@squadco.com; merchant KYC must be a GTBank account mapped to BVN; fees quoted 1% + ₦50 (local) or 1.2% in some listings **[SECONDARY]**. "Habari" itself is a consumer lifestyle app (2018). Sources: https://developer.gtbank.com/ ; https://squadinc.gitbook.io/virtual-accounts/reference/dynamic-virtual-account-v2 ; https://businessday.ng/technology/article/how-gtcos-fintech-firm-squares-up-to-competition/
- **Access Bank / Hydrogen** — Access developer portal (developer.accessbankplc.com) offers balance, statements, transactions, payment evidence; virtual accounts are sold through Hydrogen (Access Holdings' PSSP, which absorbed Woven Finance's business Q1 2024). Hydrogen Instant Pay API: static and dynamic virtual accounts on Access Bank, **0.2% capped ₦200 per transaction**; gateway 1% capped ₦1,100 local **[SECONDARY, support.hydrogenpay.com]**. Access Closa is an agent-banking app, not an API product. Sources: https://developer.accessbankplc.com/ ; https://support.hydrogenpay.com/hc/en-us/articles/13490674860829-FAQs-About-Instant-Pay-API ; https://techcabal.com/2024/01/10/woven-finance-shuts-down/
- **Zenith** — GlobalPAY "Managed Virtual Account Service" (dynamic account at checkout); sandbox at apidocs.zenithbank.com; integrations@zenithbank.com. No reserved-per-customer virtual account product found for third parties [GAP]. Source: https://www.zenithbank.com/corporate-banking/e-business-solutions/globalpay/
- **Polaris VULTe** — "API Banking allows merchants and businesses to integrate… risk assessment, bank statement requests, lien accounts, direct debits". No public docs/pricing [GAP]. Source: https://www.vulte.ng/business/
- **Kuda Business API** — self-serve from the Kuda Business dashboard; docs.kuda.com; Pay with Transfer (virtual accounts) **0.3% capped ₦80**; Pay with Kuda 0.5% capped ₦300; webhooks for account events; instant settlement 0.2–0.4% capped ₦100. Sources: https://business-support.kuda.com/en/articles/11090215-kuda-business-api-pricing ; https://business-support.kuda.com/en/articles/9757994-getting-started-with-the-kuda-business-api
- **Moniepoint / Monnify** — Moniepoint's API surface is Monnify (customer reserved accounts on Moniepoint MFB by default, plus Wema/Sterling options historically); webhooks signed HMAC-SHA256 over `moniepoint-webhook-id__timestamp__body`; bank-transfer/reserved-account fee 1.5% capped ₦2,000 or flat ₦500 **[SECONDARY]**. Sources: https://developers.monnify.com/docs/collections/customer-reserved-account ; https://monnify.com/pricing (blocked)
- **Paystack Dedicated Virtual Accounts** — Nigeria-registered businesses only; providers Wema Bank and Paystack-Titan; up to 1,000 DVAs by default (more on request); **1% capped ₦300** per inflow; customer needs phone, full name, email; financial-services/betting merchants must create via API with customer validation; webhooks on `charge.success`. Source: https://support.paystack.com/en/articles/2124866
- **Flutterwave** — static (reusable, per customer) and dynamic virtual accounts, NGN only; collections now **2% (1.4% + 0.6% platform fee), no cap stated on the 2026 pricing page** (earlier cap ₦2,000 — "confirm cap treatment in your merchant agreement"). Sources: https://flutterwave.com/ng/pricing ; https://flutterwave.com/eu/support/pricing/fee-increase-for-local-ngn-payments
- **Anchor (getanchor.co)** — BaaS over CBN-licensed partner banks (unnamed on site; docs reference "9 Payment Service Bank" as one provider); Reserved Accounts (permanent NUBAN per customer), Dynamic accounts "coming soon"; `payin.received` webhook; KYC Tier 0 (name/address/email/phone) auto-upgrades to Tier 1 with BVN/DOB/gender; go-live = sandbox → onboarding questionnaire + SLA → pre-go-live call → KYB approval, "organisations go live in 24 hours after their integration". Pricing by sales (hello@getanchor.co). Clients cited: Bujeti, Outpost Health. Sources: https://docs.getanchor.co/docs/going-live ; https://docs.getanchor.co/docs/pay-with-transfer ; https://help.getanchor.co/portal/en/kb/articles/payment-collections-receive-money
- **Maplerad** — NGN static virtual accounts; "no fees for funding your NGN wallet via bank transfers"; NGN payout ₦10 flat; no API fees. Sources: https://maplerad.com/faq ; https://maplerad.dev/reference/create-a-virtual-account
- **Embedly** — see Sterling above (Embedly is Sterling's embedded-finance brand).
- **Blnk** — open-source double-entry ledger (Blnk Core OSS; Blnk Cloud managed), not a bank connector; relevant as the internal ledger for member sub-balances behind a single bank account. Source: https://github.com/blnkfinance/blnk
- **Woven Finance** — ceased operations Q1 2024, business transferred to Hydrogen (TechCabal 10 Jan 2024). Do not plan on it.
- **Interswitch** — Quickteller SVA (virtual accounts with float card), direct debit API, transfers; enterprise onboarding. Source: https://developer.interswitch.com/quickteller-SVA/

### Gaps
- Bank-direct virtual-account fees (Providus, Wema, Sterling/Embedly, UBA, VFD) are negotiated — [CONFIRM WITH BANK].
- Whether a per-*employer* virtual account can be opened in the *cooperative's* name (not the platform's) with the webhook delivered to the platform: technically yes at every provider that supports sub-accounts (Anchor "Deposit Accounts or Sub accounts", UBA "linked to your main account", Embedly organisation/business wallets), but the account-title/KYC owner must be the cooperative. [CONFIRM WITH BANK].

---

## C. Outbound transfers with multi-approver workflows via API

### Takeaway
All providers above offer NIP payouts via API (single and bulk). The only approval controls exposed programmatically are: OTP to the initiating user (Paystack, Monnify), optional "transfer approval" callback where the merchant confirms the request originated from its system (Paystack), and dashboard roles/limits (Kuda "set each member's role and define their access limits"). None implements an n-of-m bank-side approval matrix over API. Approval logic therefore must live in the platform, with the bank/aggregator trusting a single service credential — or payouts must be executed through the bank portal by cooperative officers.

### Cited findings
- **Paystack Transfers (Nigeria)**: ₦10 (≤₦5,000), ₦25 (₦5,001–50,000), ₦50 (>₦50,000) + ₦50 stamp duty ≥₦10,000; min ₦50, max ₦10,000,000 per transfer; OTP "sent to the email and phone number of the user who initiated the transfer"; "Transfer Approval" lets a business verify requests before processing; bulk CSV up to 100 per batch. Source: https://support.paystack.com/en/articles/2132866
- **Flutterwave payouts**: ₦10/₦25/₦50 bands; 7.5% VAT on fees. Source: https://flutterwave.com/ng/pricing
- **Kuda Business API**: interbank ₦15 (≤₦10k), ₦12 (₦10,001–100k), ₦10 (₦100,001–400k), ₦9 (>₦400k); Kuda-to-Kuda free; single and bulk transfers; team roles with access limits on the dashboard. Source: https://business-support.kuda.com/en/articles/11090215-kuda-business-api-pricing
- **Monnify Disbursements**: single and bulk NIP; two-factor OTP authorisation; from April 2026 requires name inquiry before disbursement **[SECONDARY, api-evangelist profile]**. Source: https://github.com/api-evangelist/moniepoint
- **Anchor**: Book Transfer (intra-org, free) and NIP Transfer; custom fee endpoint to charge platform fees to a customer account. Source: https://docs.getanchor.co/docs/overview-5
- **Maplerad**: NGN bank transfer ₦10 flat. Source: https://maplerad.com/faq
- **VFD**: outward transfer ₦10 flat **[SECONDARY]**.
- **Providus**: Transfer API for "business disbursements, payroll systems"; RIB/CIB bands ₦10/₦25/₦50. Source: https://www.providusbank.com/banking-apis ; https://providusbank.com/charges
- **NIP limits**: CBN 2022 circular — banks accept indemnity for "Highly Secured Online Funds Transfer" above ₦1m (individual) / ₦10m (corporate) up to ₦25m / ₦250m; Stanbic publishes a company NIP-limit indemnity form. Source: https://nairametrics.com/2022/06/03/cbn-raises-online-transfer-limit-to-n25-million-and-n250million-for-individuals-and-companies-respectively/
- **Fraud liability**: from Jan 2025 CBN directed NIBSS to debit settlement accounts of banks receiving fraud proceeds; Dec 2025 framework requires refunds within 16 working days; APP-fraud draft guidelines dated 26 Nov 2025. Payout APIs will increasingly require name-enquiry match and device/liveness checks on the initiating user. Sources: https://techcabal.com/2025/01/28/cbn-nibss-fraud/ ; https://techcabal.com/2025/12/02/nigerian-banks-16-days-fraud-refund-cbn/

### Gaps
- No provider documents "transfer requires approval by N named users before release" as an API object. [CONFIRM WITH BANK] whether Wema/Providus/Sterling can place the platform's API user under the cooperative's account mandate so that bank-side approvers must release each batch (some banks do this for corporate H2H via "approve in portal" of API-initiated batches — not evidenced publicly for Nigeria).

---

## D. Statement and transaction feeds (webhooks, MT940, APIs)

### Takeaway
Real-time credit webhooks exist for virtual accounts everywhere. Full-account statement pulls exist as REST at Providus, UBA (PSSP required), Access, VFD, VULTe and via Mono (consumer-consent model). MT940/MT101 over SWIFT is a large-corporate product (Zenith documented). Open banking as a standard feed is not yet live.

### Cited findings
- **Providus**: Account Statement API; Transaction Notification API "real-time alerts… for merchants, Fintech, MFBs". Source: https://www.providusbank.com/banking-apis
- **Zenith**: MT940 statements via SWIFT FIN; MT101 payment initiation from ERP; FileAct periodic MT940 files; requires RMA key exchange — i.e., a corporate SWIFT membership/service bureau. Source: https://www.zenithbank.com/corporate-banking/swift
- **Access**: developer portal supports balance, self-generated statements, transactions. Source: https://developer.accessbankplc.com/
- **UBA**: Statement and Balance Enquiry APIs gated on PSSP/MMO licence + corporate account **[SECONDARY]**.
- **Mono**: Statement endpoint returns 1–12 months per call for a linked account; pricing Basic ₦15,000/month (100 unique accounts), Enterprise custom; 30-day trial capped at 5 accounts. Open Banking Nigeria has argued Mono/Stitch are screen-scraping rather than true open banking. Sources: https://docs.mono.co/api/bank-data/bank-statement ; https://mono.co/pricing ; https://openbanking.ng/are-mono-and-stitch-truly-open-banking/
- **Okra**: shut down May 2025 (assets to Kernel UK). **Stitch**: entered Nigeria 2021 with pay-by-bank; no 2026 Nigerian direct-debit/statement product evidenced [GAP]. Sources: https://techlabari.com/nigerian-open-banking-startup-okra-quietly-shut-in-may-2025/ ; https://businessday.ng/news/article/south-africas-stitch-expands-into-nigeria/
- **Open banking status**: CBN approved launch for Aug 2025 (TechCabal 29 Apr 2025); slipped; Oct 2025 CBN said "not live yet"; Feb 2026 CBN Fintech Report commits to a roadmap (Phase 1 0–3 months: roadmap; Phase 2 3–9 months: data-portability guidance); NIBSS nominated as Open Banking Registry; participants must be CBN-licensed and ISO 27001-compliant (Operational Guidelines 2023). As of Oct 2026 no production go-live confirmed. Sources: https://techcabal.com/2025/04/29/cbn-launches-open-banking/ ; https://technext24.com/2025/10/20/cbns-open-banking-system-hasnt-launched/ ; https://www.mondaq.com/nigeria/fintech/1747436/the-2026-cbn-fintech-report-defining-the-future-of-fintech-in-nigeria ; https://www.cbn.gov.ng/Out/2023/CCD/Operational%20Guidelines%20for%20Open%20Banking%20in%20Nigeria.pdf

### Gaps
- Debit-side webhooks on an ordinary corporate current account (not a virtual account) are not documented by any bank; Providus's Transaction Notification API is the closest. [CONFIRM WITH BANK].
- MT940 availability at Wema/Providus/Sterling/Access for mid-size corporates: [GAP].

---

## E. NIBSS direct-debit e-mandates through the bank

### Takeaway
NIBSS Direct Debit (NDD) is the single rail. Mandate types: Fixed, Variable, Global Standing Instruction (GSI, BVN-wide recovery). E-mandate = payer transfers exactly ₦50 to a designated NIBSS account (Paystack-Titan 9880218357 in Lendsqr's flow); activation typically 5 min–24 h, sometimes 48 h+. Direct NIBSS access needs a bank/OFI/PSSP relationship ("contact your NIBSS Relationship Manager"), a biller front-end, a scheduler and the Mandate Management + EasyPay APIs; NIBSS charges ₦30 per successful debit and allows mark-up. Aggregators remove the licence requirement at a price.

### Cited findings
- **NIBSS NDD page**: users = banks, OFIs, PSSPs (as aggregators), billers; ₦30 per successful debit; partial debits configurable; GSI mandate type. Source: https://nibss-plc.com.ng/nibss-direct-debit-ndd/
- **Mono Direct Debit** (pricing effective 19 May 2026): 1% capped ₦1,000 + 7.5% VAT per debit; min ₦55 for ₦200–₦20,000; balance enquiry ₦50/call; e-mandate via transfer of ₦50 (non-negotiable NIBSS fee) to Flutterwave MFB / Parallex / Paystack-Titan accounts within 1 hour; some banks (Unity, Union, StanChart, Ecobank, OPay) enforce ₦100 minimum transfer so ₦50 e-mandate fails — signature mandate (1–72 h) as fallback; "ready-to-debit" 5 min–24 h. Sources: https://support.mono.co/en/articles/8698485-mono-direct-debit-pricing ; https://docs.mono.co/docs/payments/direct-debit/mandate-setup-explained
- **Paystack Direct Debit** (launched Nov 2023 with NIBSS; ~two dozen banks): 1.5% + ₦100, capped ₦2,000, ₦100 waived under ₦2,500; mandate via ₦50 transfer to NIBSS account at Paystack-Titan; webhook returns authorisation code. Sources: https://support.paystack.com/en/articles/2124930 ; https://techcabal.com/2023/11/09/paystack-direct-debit/
- **Lendsqr NIBSS e-mandate how-to**: exact ₦50, within 7 days of initiation, ₦150 minimum balance, name must match BVN, confirmation usually within 2 h. Source: https://docs.lendsqr.com/how-to-activate-direct-debit-with-nibss-emandate/
- **Monnify**: NIBSS mandates authorised via URL for banks not on TeamApt rails; DD fee 1.5% capped ₦2,000 or ₦500 flat **[SECONDARY]**. Source: https://developers.monnify.com/docs/collections/recurring-payments/direct-debit (blocked)
- **Zenith Z-Mandate / E-Mandate Direct Debit Service**: SOAP-style methods CreateMandate, ValidateToken, MakePayment, CancelMandate, ResendOTP, QueryPayment, Ping; OTP-based customer consent; corporate product. Source: https://apidocs.zenithbank.com/EmandateDocumentation/index.html
- **UBA Direct Debit API**: PSSP/MMO licence + corporate account **[SECONDARY]**.
- **Remita** (SystemSpecs): direct-debit/standing-order ₦100 per transaction; collections 2% capped ₦2,500; payroll ₦500 per employee setup + ₦250 per transaction; handles cooperative deductions and remits to "Staff Cooperatives" with electronic schedules; pricing negotiated per merchant **[SECONDARY]**. Sources: https://support.remita.net/portal/en/kb/articles/remita-payroll ; https://paymentproviders.io/providers/remita
- **Interswitch**: Direct Debit API "enables merchants to initiate direct debit on the customer's account". Source: https://www.interswitchgroup.com/api/payment_services/docs.html

### Gaps
- NIBSS mandate-management API documentation is not public; onboarding timeline [GAP].
- Whether a cooperative (registered under a state cooperative law, not CAC) can be a biller at NIBSS directly, or only via a bank sponsor: [CONFIRM WITH BANK/NIBSS].

---

## F. Escrow and trust accounts

### Takeaway
Escrow is offered as a relationship product by banks' transaction-banking or trustee arms (Stanbic IBTC Escrow Services; FirstBank/FBNQuest Trustees; FCMB Trustees, SEC-licensed) and by Vesicash (licensed digital escrow, Merchant-of-Record). None exposes a self-serve API. For a cooperative-estate or group-purchase use case the practical path is a bank escrow agreement with the cooperative and vendor as parties, or Vesicash's API for smaller tickets.

### Cited findings
- Stanbic IBTC: bank "acts as escrow agent", "holds funds in safe custody in Nigeria", executes per agreed instructions; fees/interest not published. Source: https://www.stanbicibtcbank.com/nigeriabank/corporate-and-investment/transaction-banking/Escrow-services
- FirstBank Escrow Services (Trustee page): independent escrow agent holds asset in trust until conditions met. Source: https://www.firstbanknigeria.com/business/trustee/escrow-services/
- FCMB Trustees: SEC-licensed trustee offering escrow agent services. Source: https://www.fcmbtrustees.com/page.php?a=escrow-agent
- Vesicash: "instant digital escrow for B2B, P2P and marketplace"; for law firms "hold and manage client monies in a single, transparent, central, safeguarded transactional account". Source: https://techpoint.africa/2021/03/23/vesicash-escrow-services/
- Trustee licensing: trustee services are SEC-licensed and open only to CAC-registered corporates. Source: https://mandelsolicitors.com/registrations-and-licensing-with-the-nigerian-securities-and-exchange-commission-sec/
- CBN's own "escrow" for PSP licence applications (refundable capital deposit) is unrelated to customer escrow. Source: https://thecondia.com/cbn-regulations-require-fintech-pay-5million-escrow-account/

### Gaps
- Escrow fees, minimum ticket, interest treatment on escrow balances at each bank: [GAP, CONFIRM WITH BANK].
- Whether a cooperative can be the escrow beneficiary without CAC incorporation: [GAP].

---

## G. Float / interest on pooled balances

### Takeaway
Interest on a cooperative's pooled balance depends on account type. Current accounts: interest "negotiable" (CBN Guide to Charges 2020; Stanbic 2026 pricing guide says the same). Savings: minimum 30% of MPR → 6.9% from 22 Sep 2026 (MPR 23%), forfeited if >4 withdrawals a month. Deposits held as collateral: negotiable, min 30% MPR. Fixed deposits at tier-1 banks 15–16% (12–24 months), Providus up to 23.66%, high-yield savings (Access HIDA up to 13%, Fidelity 10.45%); 364-day T-bills 15.89% (7 Oct 2026 auction). Because a non-licensed platform cannot hold funds, float economics must sit with the cooperative (or with a licensed partner under a bare-trust pool with NDIC pass-through).

### Cited findings
- CBN Guide to Charges (eff. 1 Jan 2020): current account interest "negotiable"; savings ≥30% MPR; collateral deposits negotiable ≥30% MPR. Source: https://www.cbn.gov.ng/Out/2022/CCD/Guide%20to%20Charges%20by%20Banks%20Other%20Financial%20and%20Non-Financial%20Institutions%20eff%20Jan%201%202020.pdf
- Draft 2026 Guide to Charges (Nairametrics 23 Apr 2026): transfers ≤₦5,000 free, ₦10 for ₦5,001–50,000, ₦50 above; MSC capped 0.5% max ₦10,000; APR disclosure; consultation to 8 May 2026; "may take effect from May 1, 2026" — final status [GAP]. Source: https://nairametrics.com/2026/04/23/cbn-caps-bank-fees-mandates-transparency-in-proposed-guidelines/
- MPC 21–22 Sep 2026: MPR 26.5% → 23%; minimum savings rate ≈6.9% (BusinessDay 23 Sep 2026). Source: https://businessday.ng/news/article/cbn-rate-cut-lowers-returns-for-savers-as-deposit-rate-falls-to-6-9/
- T-bills 7 Oct 2026: 91-day 15.50%, 182-day 15.80%, 364-day 15.89%; ₦900bn offered. Source: https://nairametrics.com/2026/10/06/cbn-offers-n900-billion-at-first-treasury-bills-auction-of-fourth-quarter-2026/
- Fixed deposit quotes 2026: GTBank 15.5% (12m), Access 15%, Sterling 16% (24m), Providus 23.66% (comparison sites; verify with bank). Source: https://nairacompare.ng/blogs/top-5-fixed-deposit-accounts-with-the-best-rates-in-nigeria
- NDIC pass-through: MMO subscribers insured up to ₦5m per subscriber per DMB only where a documented bare trust exists, pool is at an NDIC-insured DMB, records identify each subscriber, and KYC is complete. Source: https://ndic.gov.ng/ndic-releases-deposit-insurance-guidelines-on-mobile-payments-system/ ; https://www.legal500.com/intelligence/nigeria/finance-and-banking/when-a-banking-partner-loses-its-licence-what-the-cbns-revocation-of-46-mfb-licences-means-for-fintech-resilience-and-customer-funds
- Cooperative deposits have no NDIC cover (cooperatives are state-regulated, not CBN-licensed). Source: https://kudicompass.com/how-to-join-cooperative-society-nigeria/

### Gaps
- Actual negotiated current-account/call rates for ₦100m–₦1bn cooperative balances: [CONFIRM WITH BANK]. Expect banks to offer a "call account" or sweep into fixed deposit; none publish it.

---

## H. Cross-cutting: licensing, onboarding, charges, CBN constraints

### Takeaway
A platform without a CBN licence can use aggregators (Paystack, Flutterwave, Monnify, Anchor, Maplerad, Kuda API, Mono) with only business registration. Bank-direct APIs at UBA explicitly require PSSP/MMO; Providus, Wema, Sterling, VFD and Anchor onboard on contract plus KYB. Open-banking participation will require a CBN licence and ISO 27001. Transaction costs in 2026: NIP ₦10–50 + ₦50 stamp duty (sender, ≥₦10k, salary exempt) + 7.5% VAT on fees. Watch: data localisation by 1 Jan 2027, UBO disclosure, AML automation, partner-bank failure risk.

### Cited findings
- **April 2024 freeze**: CBN directed OPay, Moniepoint, Kuda, PalmPay (and Paga) to pause new onboarding on 29 Apr 2024 amid an EFCC freeze of 1,146 accounts for illegal FX trading and a KYC audit (Tier-1 limits disregarded). Lifted later in 2024; the five were upgraded to national MFB licences in Jan 2026. Sources: https://techcabal.com/2024/04/29/exclusive-cbn-directs-four-fintechs-to-stop-onboarding/ ; https://nairametrics.com/2024/04/29/forex-deals-cbn-stops-4-fintechs-from-onboarding-new-customers/ ; https://businessday.ng/technology/article/cbn-upgrades-licences-of-opay-moniepoint-kuda-other-fintechs-to-national-status/
- **46 MFB licences revoked 1 Jul 2026** (BOFIA 2020 ss.12–13); NDIC liquidator; MFB cover ₦2m, DMB ₦5m; fintechs told to map "regulatory responsibility for each product feature", keep "independent reconciliation layers", pre-arrange data portability. Source: https://www.legal500.com/intelligence/nigeria/finance-and-banking/when-a-banking-partner-loses-its-licence-what-the-cbns-revocation-of-46-mfb-licences-means-for-fintech-resilience-and-customer-funds
- **CBN circular PSS/DIR/PUB/CIR/001/004, 15 Jun 2026** (signed Rakiya O. Yusuf): payment transaction data to be stored/managed in Nigeria by 1 Jan 2027 (foreign staff login = cross-border transfer); card-issuing ≤25% and merchant-acquiring ≤15% market share at group level by 31 Dec 2026; ongoing UBO disclosure for DMBs/PSPs. Source: https://www.kietlaw.com/blog/the-central-bank-of-nigeria-payments-system-circular-market-structure-data-localisationbeneficial-ownership-amp-systemic-oversight
- **AML automation**: circular BSD/DIR/PUB/LAB/019/002 — automated AML roadmap due 10 Jun 2026; real-time screening at onboarding and each transaction. Source: https://youverify.co/en/blogs/cbn-kyc-aml-requirements-2026
- **Agent banking guidelines** 6 Oct 2025 (exclusivity/location rules from 1 Apr 2026) — relevant only if cash-in via agents is contemplated. Source: https://www.kietlaw.com/ (same post) ; https://complyan.com/nigerias-cbn-data-localisation-directive-what-banks-and-fintechs-must-do-before-2027/
- **KYC tiers**: Tier 1 ₦30,000/day, ₦300,000 balance; Tier 2 ₦500,000/day; Tier 3 unlimited; BVN/NIN mandatory from 1 Mar 2024. Source: https://youverify.co/en/blogs/cbn-kyc-aml-requirements-2026
- **PSSP**: ₦250m capital (per legal explainers; see enablers note for full table); cannot hold funds; must evidence bank/processor partnership. Source: https://www.618bees.com/article/1276-understanding-the-cbn-payment-solution-service-provider-pssp-license-in-nigeria
- **Open banking requirements**: ISO 27001, CBN risk-based cyber framework, CBN approval before customer data goes to outsourced/foreign providers (Operational Guidelines 2023). Source: https://www.cbn.gov.ng/Out/2023/CCD/Operational%20Guidelines%20for%20Open%20Banking%20in%20Nigeria.pdf
- **Onboarding timelines**: Anchor "24 hours after integration" (KYB permitting); Kuda self-serve dashboard; Paystack DVA immediate for registered businesses; CBN sandbox decisions 45–60 working days; bank BaaS contracts (Providus/Wema/Sterling) — [GAP], industry anecdote is 4–12 weeks **[RECALLED, unverified]**.
- **Stamp duty / EMTL from 1 Jan 2026**: ₦50 on electronic transfers ≥₦10,000 now debited to the *sender*; salary payments and intra-bank self-transfers exempt (Nigeria Tax Act 2025). Press reported ₦75 all-in for some banks (₦50 duty + ₦25 NIP). Sources: https://techcabal.com/2025/12/22/why-your-50000-transfer-will-cost-100-from-2026/ ; https://www.thecable.ng/banks-to-start-charging-senders-n50-stamp-duty-on-transfers-above-n10k-from-january/ ; https://www.legit.ng/business-economy/money/1690443-banks-charge-n50-stamp-duty-electronic-transfers-january-1-2026/
- **VAT**: 7.5% on all provider fees (Flutterwave, Mono state explicitly).
- **NIBSS zero-fee ambition**: NIBSS said it targets eliminating NIP fees in favour of subscription pricing by 2026 (Oct 2025) — not implemented as of this note. Source: https://www.nigeriacommunicationsweek.com.ng/nibss-targets-zero-transfer-fees-on-instant-payments-by-2026/

### Gaps
- No public "fintech–bank collaboration approval" circular text was located for 2024–26 beyond the open-banking guidelines and the 2020 PSP framework; the CBN's Feb 2026 Fintech Report does not address BaaS/partnership rules. Treat the bank's own compliance sign-off as the gate. [GAP]
- Specific PCI-DSS requirement: only relevant if cards are issued; Embedly cites PCI DSS for its card product.

---

## I. Documented cooperative / fintech usage examples

- Wema CoopHub: 1,000+ cooperatives by Jul 2024 (Daily Trust). No named cooperatives.
- Paystack DVAs: Wema Bank and Titan Trust as providers; used by thousands of Nigerian merchants (Paystack support).
- Providus: historically powered PiggyVest (via TeamApt/Monnify), Cowrywise "Stash", Monnify reserved accounts (2019–21 press); TechPoint Aug 2025 names Wema, Providus and Sterling as the banks "generating revenue from digital services like virtual accounts" through fintech partnerships.
- Sterling/Embedly: Seedfi, EduBanc, HealthBanc, Cafe One, Foodcourt (site).
- Anchor: Bujeti, Outpost Health (Frontier Fintech substack).
- Lendsqr lenders use NIBSS e-mandate via Paystack-Titan; Mono DD used by lenders/insurers (Mono blog).
- Remita remits cooperative deductions for IPPIS/MDA payrolls (OAGF; Remita Payroll page) — see `payroll_checkoff_ippis.md`.
- Cooperative software vendors in Nigeria (ProbityCoop, VerifyGo, SmartTeller, CoopSolve) advertise bank integration and bulk deductions but none documents which bank API they use [GAP].

---

## J. Comparison table

| Provider | Type / licence | (a) Third-party account + bank maker-checker | (b) Virtual accounts + webhook | (c) Payout API / approvals | (d) Statements | (e) Direct debit | (f) Escrow | Published pricing (ex-VAT) | Onboarding gate |
|---|---|---|---|---|---|---|---|---|---|
| Wema (CoopHub / ALAT API) | DMB | CoopHub: Initiator/Verifier/Approver in-app; white-label offered; API not public | ALAT wallets (Tier-1 limits, NIN) + virtual accounts; notification API | ALAT transfer APIs; no approval matrix | Account mgmt API; MT940 [GAP] | Via Paystack-Titan/NIBSS; none public | No | Not published | Partner credentials via playground.alat.ng; CoopHub via bank |
| Providus | DMB | Portal only | Digital Collection Service (dynamic + reserved); Transaction Notification API | Transfer API; single credential | Account Statement API | Not public | No | RIB ₦10/25/50; VA fee negotiated | Business concierge; contract |
| Sterling (Embedly / SeaBaaS) | DMB | Portal only | Org/business/customer wallets; webhooks | Payout API | Dashboard; API [GAP] | [GAP] | No | Not published | KYB; "minutes" per site |
| VFD (VBaaS) | MFB (NDIC ₦2m) | Portal only | Wallets backed by VFD accounts; corporate wallets | Transfer API ₦10 flat [SECONDARY] | Transaction history API | [GAP] | No | Create ₦10; in 0.3% cap ₦500; out ₦10 [SECONDARY] | KYB; negotiated |
| Access (Hydrogen) | DMB + PSSP | PrimusPlus portal only | Static/dynamic on Access; 0.2% cap ₦200 | Hydrogen payouts | Access dev portal statements | [GAP] | No | 0.2% cap ₦200 (VA); 1% cap ₦1,100 (gateway) | Hydrogen merchant KYB |
| Zenith | DMB | Portal only | GlobalPAY managed VA (checkout) | Sandbox APIs | SWIFT MT940/MT101 FileAct | Z-Mandate e-mandate API | No | Not published | integrations@zenithbank.com; SWIFT RMA |
| GTBank / Squad | DMB + PSSP | GAPS portal only | Squad static/dynamic (dynamic restricted) | GTBank transfer APIs; Squad payouts | GTBank dev portal | [GAP] | No | Squad ~1% + ₦50 [SECONDARY] | GTBank account mapped to BVN |
| UBA | DMB | CIB portal only | Virtual accounts linked to main account | Transfer API (PSSP/MMO required) | Statement API (PSSP/MMO required) | Direct Debit API (PSSP/MMO) | No | Not published | PSSP/MMO licence + UBA corporate account |
| First Bank | DMB | Portal only | [GAP] | [GAP] | [GAP] | [GAP] | Trustees escrow | Not published | [GAP] |
| Polaris VULTe | DMB | [GAP] | [GAP] | API banking (direct debits, lien, statements) | Statement API | Yes (per site) | No | Not published | [GAP] |
| Kuda Business API | MFB (national) | Dashboard roles/limits; no bank-side matrix | Pay with Transfer 0.3% cap ₦80; webhooks | ₦9–15 per NIP; bulk | Transaction history API | No | No | Published | Self-serve KYB |
| Moniepoint / Monnify | MFB (national) + PSSP | PIN-based; no matrix | Reserved accounts; HMAC webhooks | Disbursements with OTP; bulk | API | NIBSS mandates | No | 1.5% cap ₦2,000 or ₦500 [SECONDARY] | Merchant KYB |
| Paystack | PSSP (Stripe) | Transfer Approval callback + OTP | DVA (Wema/Titan) 1% cap ₦300; 1,000 default | ₦10/25/50; max ₦10m; bulk 100 | Dashboard/API | 1.5% + ₦100 cap ₦2,000 | No | Published | Registered NG business |
| Flutterwave | PSSP + MFB | OTP | Static/dynamic VA; 2% (1.4% + 0.6%) | ₦10/25/50 | API | [GAP] | No | Published | Registered business |
| Anchor | BaaS over partner banks (incl. 9PSB) | Platform-side only | Reserved NUBAN; `payin.received` | Book (free) + NIP | API | [GAP] | No | Sales | SLA + KYB; ~24 h post-integration |
| Maplerad | BaaS | Platform-side only | NGN static VA, free inbound | ₦10 flat | API | No | No | Published | KYB |
| Embedly | = Sterling | see Sterling | | | | | | | |
| Blnk | OSS ledger | n/a (internal ledger) | n/a | n/a | n/a | n/a | n/a | OSS / cloud | n/a |
| Mono | Open-finance aggregator | n/a | n/a | n/a | Statements 1–12 months; from ₦15k/month | 1% cap ₦1,000, min ₦55 (May 2026) | No | Published | Registered business |
| Okra / Stitch | Aggregators | — | — | — | Okra closed May 2025; Stitch NG status [GAP] | — | — | — | — |
| NIBSS NDD (direct) | Infrastructure | n/a | n/a | n/a | n/a | ₦30 per successful debit; Fixed/Variable/GSI | n/a | Published | Bank/OFI/PSSP via NIBSS RM |
| Interswitch | Switch + PSSP | n/a | Quickteller SVA | Transfers | Transaction search | Direct Debit API | No | Negotiated | Enterprise |
| Remita | PSSP (SystemSpecs) | n/a | Referenced collections | Payroll ₦250/txn | Reports | ₦100 per DD [SECONDARY] | No | Partly published | Merchant profile |
| Stanbic IBTC / FirstBank Trustees / FCMB Trustees | Bank / SEC trustee | n/a | n/a | n/a | n/a | n/a | Escrow agent (manual) | Not published | Escrow agreement |
| Vesicash | Licensed escrow fintech | n/a | n/a | n/a | n/a | n/a | Digital escrow API | Not captured | Business KYB |

---

## K. Recommended architecture framing (for the solution architect; derived, not sourced)

1. **Custody**: cooperative holds its own DMB corporate account(s) with an "any two to sign" mandate; platform never takes title. Optional: bank sweeps to call/fixed deposit for float. Confirm interest terms per bank.
2. **Collections**: per-employer and per-member reserved NUBANs issued by a bank BaaS desk (Providus/Wema/Sterling) *in the cooperative's name* with webhooks to the platform, or by an aggregator (Paystack DVA on Wema/Titan, Kuda API, Anchor) if speed matters more than title. Pair with Remita/IPPIS schedules for check-off reconciliation.
3. **Approvals**: platform enforces initiator/verifier/approver internally and records evidence; execute payouts either (i) via API under a single service credential (bank trusts platform) or (ii) by exporting bank-portal batches that cooperative officers approve in GAPS/PrimusPlus/ALAT for Business (bank-enforced). Ask Wema whether CoopHub's roles can be exposed or white-labelled.
4. **Direct debit**: Mono or Paystack for fast go-live; migrate to a bank-sponsored NIBSS NDD biller profile at scale (₦30/debit).
5. **Ledger**: internal double-entry (e.g. Blnk) keyed to statement feeds; daily reconciliation against bank statement API/MT940; keep data in Nigeria (1 Jan 2027).
6. **Risk**: dual partner banks; avoid single-MFB dependency; document bare-trust if any pooled structure is used; monitor CBN circulars monthly.

All items under A–H marked [CONFIRM WITH BANK] should be raised in the first commercial meetings with Wema, Providus and Sterling.
