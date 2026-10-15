# CoopHub competitive review

Oct 9, 2026 · @Abubakar

## Verdict

CoopHub has the right architecture and the wrong ambition. It is the only Nigerian product that keeps a cooperative's money at a licensed bank and requires three officers to release it, which is the core control this whole problem needs. But two and a half years after launch it has one unrefreshed scale claim (1,000+ cooperatives, July 2024), about a thousand Android installs, 13 app-store reviews, no update since October 2025, no published price, and no payroll check-off, statutory accounting, USSD or estate features. It is a Wema deposit-gathering product on maintenance, not a cooperative operating system.

**It is not the biggest threat.** The real incumbent is the WhatsApp group and the notebook (over 70% of cooperatives by one estimate). The faster competitor is Rank, which now holds a microfinance licence and launched payroll-deducted savings circles in July 2026. The structural force is the government's national registry and the Cooperative Bank of Nigeria, which will set identity and custody defaults from 2027.

**Recommendation:** do not build a CoopHub clone and do not compete on custody. Lead as the neutral cooperative operating system (payroll reconciliation, cooperative accounting and governance, USSD, registry returns) on top of any partner bank, and offer Wema an integration in which CoopHub keeps the deposits and gains the features it lacks. Test this in 60 days with treasurer interviews that include CoopHub users and a historical payroll reconciliation CoopHub cannot match.

## What CoopHub is

CoopHub is Wema Bank's in-house cooperative app, launched on 10 May 2024, that puts a cooperative's money in a Wema account and its members, contributions and loans in a Wema-built app, with three admin roles that must act together to release funds. It is the only Nigerian bank product that combines custody and cooperative records; it is also small, quiet and peripheral to Wema's strategy.

### Profile

| Item | What is known | Source and confidence |
| --- | --- | --- |
| Owner and builder | Wema Bank / ALAT, built in-house by the IDEAx Labs innovation team; product managers named publicly; no vendor or JV found | Press and LinkedIn; one hosting detail hints at a contractor |
| Launch | 10 May 2024 (Wema's 79th anniversary) | Press |
| Channels | Admin web portal coophub.alat.ng; member app on Android and iOS | App stores, portal |
| Account structure | The cooperative must hold a Wema account; the T&Cs also require every member to bank with Wema | T&Cs (from the portal's public JavaScript) |
| Admin roles | Initiator, Verifier, Approver; succession steps; cooperative must notify Wema of signatory changes | T&Cs. Whether the roles are enforced on the bank mandate itself or only in the app is **unknown** |
| Features | Contribution set-up and tracking; member withdrawal ("liquidation") requests; loan lifecycle with configurable terms, interest and eligibility; claimed direct-debit recovery and integration with Wema's Finacle core; transfers; member wallets with KYC tiers; "incentives" (closest thing to dividends); announcements, events, chat; reports; bulk Excel upload of contribution and loan history | Portal bundle, press |
| Not documented | Payroll check-off (IPPIS, Remita, state payrolls); USSD; AGM or voting; investments or insurance; public API. Recurring contributions "still under development". White-label advertised, no deployment named | Portal bundle, press |
| Pricing | Subscription fee renewing monthly or biannually; **no price published anywhere**; Wema also earns on captured deposits | T&Cs |
| Scale | "1,000+ cooperatives onboarded" (Daily Trust, 9 Jul 2024), a company claim never updated; FY2025 annual report names CoopHub once with no metrics | Press, annual report |
| App traction (9 Oct 2026) | Android 1K+ installs, 4.2★ from 23 ratings, last update 3 Oct 2025; iOS last version 1.1.6 (6 Oct 2025), rating count reported as 1 to 10 across checks | App stores |
| Licence and insurance | Runs under Wema's commercial banking licence (₦200bn recapitalisation met in 2025); NDIC cover is per depositor, so a cooperative account above the cover limit is only partly insured | CBN, NDIC |
| Partners | Lagos State Ministry of Commerce and Cooperatives co-hosted International Day of Cooperatives (6 Jul 2024); pitched to the Federal Minister of State for Agriculture (date unknown); no federation MoU or award | Press |
| Sales and marketing | Through branches; Instagram @coop\_hub\_ about 341 followers | Social |
| Technology | Next.js and React on Microsoft IIS | Portal headers |

### Wema in context

Wema reported FY2025 profit before tax of ₦221.85bn on ₦5.07trn of assets, ₦3.29trn of deposits and 3.75m active digital customers; H1 2026 profit before tax was ₦154.56bn with deposits of ₦3.45trn. Its strategy messaging is ALAT 2.0 and the Hackaholics programme; CoopHub appears only in the sustainability section of the annual report. For Wema, cooperatives are a deposit-gathering niche, not a core business.

## What users say

Almost nothing, and that is the finding. Across app stores, X, Facebook, Instagram, TikTok, LinkedIn, Nairaland, Reddit and the press, the only user-written content about CoopHub is 13 short app-store reviews. For a product claiming 1,000+ cooperatives, the silence suggests low active use, not satisfied use.

### App stores (checked 9 Oct 2026)

| Store | Rating | Volume | Last update | Pattern |
| --- | --- | --- | --- | --- |
| Google Play | 4.2★ | 23 ratings, 1K+ installs, 11 written reviews | 3 Oct 2025 (v1.1.6) | 6 of 11 reviews posted 10–16 May 2024, all 5-star, launch week; one reviewer shares a name with Wema's lead product designer (name match only, unconfirmed) |
| Apple App Store (Nigeria) | 4.9★ | 10 ratings, 2 written reviews | No update in \~12 months | Too few to read |

The later reviews are not about features. They are about getting in and getting out:

- How to obtain the "default password" (May 2024).
- How to create the cooperative profile (Feb 2025).
- An app "refusing to close" an account the user did not want (Sep 2026).

Wema replied to none of the 13 reviews.

### Social media, forums and press

No CoopHub complaints, praise or discussion were found on Nairaland, Reddit, X, Facebook, Instagram, TikTok or in the press. No user compares CoopHub with spreadsheets, Coopify, Coopco or any other tool in public. Nobody mentions payroll deductions, statements or data export. (Nairaland and Reddit blocked automated access, and Facebook, Instagram and LinkedIn need a login; search snippets were used and labelled.)

### The bank experience CoopHub users inherit

CoopHub users are Wema customers, so ALAT and core-banking quality is their quality:

| Signal | Evidence |
| --- | --- |
| ALAT app ratings falling | Newest 8,000 Play reviews: average 3.55 (2024) to 2.99 (2026); 1–2 star share 32% to 47% |
| Complaint themes (2,469 low-star reviews; overlapping counts) | KYC, NIN, BVN or tier upgrades \~24%; login, OTP or face capture \~20%; network failures \~20%; customer support \~10%; post-no-debit restrictions \~7%; fees \~2% |
| January 2026 ALAT relaunch | Complaint spike, February 2026 average 2.55; lost loan options |
| Core-banking incident | 16 Jan 2025: a failure let ₦2.9bn move without authorisation; Wema sued 26 institutions |
| Outages | X posts in Jan, Feb and Oct 2025 |
| Fraud losses | About ₦847.6m (snippets disagree on 2024 vs 2025) |
| Responsiveness | Replies to 2% (2024) and 16% (2026) of ALAT reviews; none on CoopHub; stopped engaging on X (7 Jun 2026) and left Telegram (29 Jun 2026) because of impersonators |

No security incident tied to CoopHub itself was found.

### What this means

1. The "1,000+ cooperatives" figure is a July 2024 launch claim, never updated; Wema's 2025 annual report lists CoopHub by name with no numbers. With 1K+ Android installs and admins working on a web dashboard, active use cannot be confirmed either way.
2. The friction users report is onboarding and exit, the two places a cooperative decides whether to stay.
3. CoopHub's trust is borrowed from Wema, and Wema's own app sentiment and incident record are deteriorating.
4. The only way to learn what cooperatives think is to ask them: a logged-in pass through Facebook cooperative groups and 10–15 treasurer interviews, starting with Lagos staff cooperatives that attended Wema's July 2024 event.

## Feature by feature against the eight capabilities

CoopHub is strong on one capability (custody with role-based release) and partial on two (ledger, identity). It has nothing on payroll reconciliation, asset verification or sector data, which are the capabilities behind the largest documented losses.

| Capability | What the design needs | CoopHub | Gap |
| --- | --- | --- | --- |
| 1. Custody | Society-named account at a licensed bank; no individual can withdraw | **Strong**: Wema account in the cooperative's name; three roles to release | Single bank only; members must also bank with Wema |
| 2. Ledger | Append-only double entry, member sub-ledgers, statements, unit accounting, appropriation | **Partial**: contributions, loans, withdrawals, "incentives", reports, Excel import | No documented double-entry ledger, dividend appropriation (25% reserve, education fund), share capital, guarantor registry, audit-reproducible journal or data export |
| 3. Payment control | Two-of-three approvals, limits, anti-splitting, evidence binding, cooling-off on signatory change | **Partial**: Initiator, Verifier, Approver | Unknown whether enforced on the bank mandate; no documented limits, evidence binding, related-party flags or signatory-change cooling-off |
| 4. Payroll check-off | Schedule generation, remittance matching, variance queue, employer arrears ageing | **None** | No IPPIS, Remita, state or corporate payroll integration; recurring contributions still in development |
| 5. Identity and consent | BVN/NIN, employer status, signed deduction authority, guarantor checks | **Partial**: Wema KYC tiers on member wallets | No deduction consent, employer verification or guarantor exposure tracking |
| 6. Asset verification | Title due diligence, developer records, milestone escrow | **None** | No estate, escrow or project-account features |
| 7. Reporting and compliance | Prudential ratios, member statements, Director returns, AGM pack, compliance calendar | **Partial**: reports and announcements | No AGM or voting tools, no Director or registry returns, no published ratios |
| 8. Sector data | Registry and bureau integration, anonymised sector metrics | **None** | No link to the national registry (CVN/CoopID) or CoopCHECK |
| Channels | App, web, USSD, agents | App and web | No USSD; members without smartphones excluded |
| Openness | Multi-bank, API, data portability | Wema only; no public API | Lock-in by design |

## CoopHub's real advantages

Be honest about these: they are structural, and a new entrant cannot copy them quickly.

1. **It holds the money legally and enforces release roles.** CoopHub is the only product in Nigeria that combines cooperative records with custody at a licensed bank and three-role release. Every neutral software vendor is a ledger beside someone else's bank account; Coopify states plainly that it is "not a registered bank". A treasurer cannot empty a CoopHub account alone. That is the single most important control in the whole problem space.
2. **No licence work for the customer.** Because the product sits inside Wema's commercial licence, a cooperative gets custody, deposit insurance up to the NDIC limit and a regulated counterparty without anyone filing for CBN approval. A new platform needs a bank partnership and a CBN collaboration approval before it can offer the same.
3. **Distribution through a branch network and a balance sheet.** Wema can sell through relationship managers, bundle CoopHub with loans to cooperatives, and price it below cost because the deposits pay for it.
4. **Core-banking integration.** Claimed integration with Finacle means loan disbursement, repayment and direct-debit recovery run inside the bank; a third party gets this only through APIs the bank chooses to expose.
5. **First-mover with government.** Lagos State's cooperatives ministry co-hosted the launch-season event and Wema has pitched the federal ministry; it has a head start on the policy relationships that will shape the registry and Cooperative Bank.
6. **Brand for the cautious buyer.** For a committee worried about fraud, "it's in a bank" is an easy sale; a startup must earn that trust.
7. **Bank scale behind it.** ₦3.45trn of deposits and a met ₦200bn recapitalisation mean Wema can fund CoopHub indefinitely if it chooses to.

## CoopHub's weaknesses and gaps

The weaknesses are as structural as the strengths: they come from being one bank's deposit-gathering product rather than a cooperative's operating system.

### Product

- **No payroll check-off at all.** The largest, most fundable segment (staff cooperatives, payroll-deducted, ₦0.4–66bn balance sheets) runs on deductions CoopHub cannot schedule, match or chase. The NNPC Lagos freeze, Police withdrawals and Oredo judgment all sit in this gap.
- **Thin cooperative accounting.** No documented statutory appropriation (25% reserve, education fund), share capital, guarantor registry, AGM, voting or Director returns; "incentives" stand in for dividends.
- **Basic gaps after two years.** Recurring contributions are "still under development"; neither app has been updated since October 2025.
- **No USSD.** Members without smartphones (most market and junior civil-service members) cannot use it.
- **No estates or escrow.** The catastrophic land and developer losses are out of scope.

### Commercial

- **Lock-in.** The cooperative and, per the T&Cs, every member must bank with Wema. Federations and bank staff cooperatives already prefer neutral software (the federation names Sterling as its bank; Sterling and Fidelity staff cooperatives use Coopify).
- **Opaque pricing.** A subscription fee exists but is not published; neutral vendors publish ₦10k–₦100k a month.
- **Not a strategic priority.** One mention in the FY2025 annual report, no metrics, no updates, about 341 Instagram followers.

### Evidence of traction

- **The only scale claim is two years old** ("1,000+ cooperatives", July 2024) and never updated.
- **Member adoption is low**: 1K+ Android installs and a handful of iOS ratings against a claim of 1,000 cooperatives.
- **Near-total silence**: 13 app-store reviews, mostly launch week, none answered by Wema; no social or forum discussion at all.
- **The visible friction is onboarding and exit**: default passwords, creating the cooperative profile, an account that would not close.

### Inherited bank risk

- ALAT app sentiment is falling (3.55 to 2.99 average, 2024 to 2026), with KYC, login and network failures leading.
- A core-banking failure on 16 January 2025 let ₦2.9bn move without authorisation; outages in 2025; fraud losses around ₦847.6m.
- Wema has stepped back from public complaint channels (X in June 2026, Telegram in June 2026) because of impersonators.
- A cooperative account above the NDIC per-depositor limit is only partly insured; the cooperative bears Wema's concentration risk entirely.

### The white space this leaves

Payroll check-off reconciliation with employer arrears; full cooperative accounting and governance (appropriation, guarantors, AGM, returns); USSD member access; multi-bank neutrality and data portability; estate escrow; registry and bureau integration. These are exactly the capabilities ranked highest in this document.

## The competitors around CoopHub

CoopHub has no direct bank rival: no other Nigerian bank or MFB offers a cooperative app with member records, loans and three-role withdrawal approval. Its competition is cheap neutral software that holds no money, a fast-moving fintech now holding a microfinance licence, and a government registry that will soon own every society's identity.

### Positioning

| Player | Type | Holds money? | Price | Traction (dated, self-reported unless stated) | Lacks |
| --- | --- | --- | --- | --- | --- |
| **CoopHub (Wema)** | Bank cooperative app | Yes, at Wema, three-role release | Monthly or biannual subscription, amount unpublished (own T&Cs, read 9 Oct 2026) | "1,000+ cooperatives" (Jul 2024, only figure); Android 1K+ downloads, 4.2★ from \~23 reviews; iOS 10 ratings; no app update since Oct 2025 | Payroll reconciliation, guarantors, dividends, AGM, recurring contributions ("under development"), registry link, multi-bank |
| Access, GTBank, Polaris, Sterling | Plain club/society accounts; Sterling OneBank retail group savings | Yes | Standard tariffs | Not disclosed | No cooperative software |
| **Rank (formerly Moni)** | Fintech with an MFB (acquired Nov 2025) | Yes | Not published | Three group products launched Jul 2026, including **payroll-deducted savings circles**; claims >$100m paid out in 12 months | Cooperative governance; registered-society ledger |
| Coopify | Neutral cooperative SaaS | No ("not a registered bank") | Free to 10 members, then paid | Used by Sterling and Fidelity staff cooperatives | Custody, payment control |
| BankCore, Coopco, SmartTeller, CoopSolve, Coopra, SmartCoopHub, Prolean | Neutral cooperative SaaS | No | BankCore from ₦10k/month; Coopco ₦15–90k/month; SmartTeller ₦50–100k/month; SmartCoopHub ₦200/member/month | 300–2,500 cooperatives claimed each (CoopLink's figures look dubious) | Custody, bank integration, payroll reconciliation |
| PiggyVest, Cowrywise, AjoCard, LibertyPay, Thrifto, TrustAjo | Group and ajo savings apps | Individual accounts or MFB custody | Free to member; float | PiggyVest 6m+ users | Do not serve registered cooperatives |
| **National cooperative registry (Seamfix)** | Government infrastructure | No | Fees to societies unpublished | MoU 12 Aug 2026; society ID (CVN), member ID linked to NIN, credit bureau (CoopCHECK); revalidation to \~Feb 2027 | Not an operating system; data layer only |
| **Cooperative Bank of Nigeria** | Planned cooperative-owned bank | Will | — | ₦200bn share drive from end-Oct 2026; licence category unannounced | Not yet operating |
| National Cooperative Financing Agency (ex-NACCUN) | Federation | — | — | Names **Sterling**, not Wema, as partner bank | — |
| The real substitute | Bank account + WhatsApp + notebook or Excel | Bank | Bank charges only | One cited study: >70% of cooperatives keep manual or cash books (snippet) | Everything |

### What this says about the market

- The incumbent's real competitor is the notebook. Any product that wins must be easier than a WhatsApp group and an Excel sheet, not just better than CoopHub.
- The neutral SaaS vendors prove cooperatives will pay ₦10k–₦100k a month for software, but none holds money or reconciles payroll, so none can stop a treasurer from emptying the account.
- Rank is the competitor to watch for staff cooperatives: it now has a deposit licence and has launched payroll-deducted circles, aimed at exactly the check-off flow this design targets.
- The registry and the Cooperative Bank will set the data and identity standards; whoever integrates first becomes the default.
- Banks' own staff cooperatives (Sterling, Fidelity) choose neutral software over bank apps, which suggests cooperatives value independence from a single bank.

### Kenya benchmarks

Co-op Bank's Sacco Link is a movement-owned switch with 619 SACCOs on front-office services (2024). Kwara raised about $7m but tied its distribution to the KUSCCO apex body, and lost it when KUSCCO collapsed in fraud and liquidation: a warning against single-channel distribution. Chamasoft stays cheap and bookkeeping-only but reconciles bank alerts automatically, which is why chamas use it.

## Our competitive advantage

Do not try to out-bank a bank. CoopHub wins on custody and brand; the platform wins on everything a cooperative actually does each month that CoopHub does not touch, starting with the payroll.

### Defensible differentiators

| Differentiator | Why CoopHub cannot easily copy it | Proof point to build |
| --- | --- | --- |
| **Payroll check-off reconciliation and employer arrears** | Requires employer-by-employer adapters, a matching engine and a neutral posture toward employers that a bank selling payroll accounts to the same employer finds awkward | Historical reconciliation of ten pilot societies showing each one's real arrears in naira |
| **Bank neutrality** | A Wema product cannot run on Sterling, Access or the coming Cooperative Bank; federations and bank staff cooperatives already prefer neutral tools | Live on two custody partners; one-click data export |
| **Full cooperative accounting and governance** | Bank product teams build banking features; statutory appropriation, guarantors, AGM, Director returns and audit reproduction are cooperative-law features | Auditor reproduces a society's year from the platform without staff help |
| **USSD and agent access** | CoopHub is app and web only; adding USSD and agent flows is a channel investment Wema has not prioritised | Members' first balance check by USSD in the pilot |
| **Registry and bureau integration** | First integrator with the national registry and CoopCHECK becomes the default operating system for revalidation and returns | Revalidation pack generated for pilot societies before Feb 2027 |
| **Estate escrow and title gates** | Out of scope for a cooperative deposit product | First estate project under release-token control |

### Positioning statement

For staff and registered cooperatives that need to know exactly what every member and every employer owes them, the platform is the cooperative operating system that reconciles payroll deductions line by line, enforces two-signature control over funds at any partner bank, and produces the statements, returns and audit trail the law requires. Unlike CoopHub, it works with any bank, chases the employer, and reaches members on USSD.

### What not to compete on

- **Custody and deposit rates.** Wema will always hold money more cheaply and with a better brand; partner with a bank for custody instead.
- **Price on basic contribution tracking.** Neutral SaaS already sells this at ₦10k–₦90k a month and free tiers exist; the value is reconciliation and governance, not a contribution ledger.
- **Consumer group savings.** PiggyVest, Cowrywise and Rank own that customer.
- **Lending from the platform's balance sheet.** That is the payroll lenders' business and it worsens the one-third-cap problem.

## Strategy options

| Option | What it means | For | Against |
| --- | --- | --- | --- |
| **A. Compete head-on** | Build a rival bank-integrated cooperative app with another bank (Sterling, Providus) and win CoopHub's customers | Neutrality and depth win against a stalled product; federation already leans to Sterling | Replicates CoopHub's strength (custody) instead of exploiting its gaps; long sales fight on Wema's ground |
| **B. Partner on CoopHub** | White-label or integrate: CoopHub keeps custody and roles at Wema; the platform supplies payroll reconciliation, accounting, governance, USSD and registry returns on top | Fastest route to custody with dual control already built; Wema gains features it lacks; one bank conversation instead of CBN approval from scratch | Dependent on Wema's API willingness and roadmap; inherits Wema lock-in and service quality; Wema could copy the features later |
| **C. Coexist as the neutral layer** | Sell reconciliation, accounting and governance to any cooperative at any bank, including CoopHub users, connecting through statement feeds and file exchange | Keeps neutrality; addresses the >70% on notebooks and the neutral-SaaS users; adds value to CoopHub customers without needing Wema's permission | Without bank-enforced control the platform's payment-control promise is weaker where the bank does not cooperate |

### Recommendation: C, with B as the opening move where Wema says yes

1. **Lead with the neutral layer.** Payroll reconciliation, cooperative accounting and governance, USSD and registry returns, sold to staff cooperatives regardless of bank, connecting to two custody partners (Providus, Sterling or similar) for bank-enforced control.
2. **Approach Wema as a partner, not an enemy.** Offer CoopHub the payroll-reconciliation and governance modules as an integration: Wema keeps the deposits and custody, its cooperative customers get the features CoopHub lacks. If Wema exposes the roles and statements by API, CoopHub accounts become one of the platform's custody options.
3. **Do not build a CoopHub clone.** Compete on the payroll and the governance, never on the deposit.

### What would change the recommendation

- **Wema refuses integration and adds payroll check-off itself:** move to A with a second bank and push neutrality hard with federations.
- **The Cooperative Bank of Nigeria launches with its own platform:** integrate with it first; it will be the default custody for many societies.
- **The registry operator builds operating features beyond identity:** become its certified operating-system partner rather than compete.
- **Rank wins staff cooperatives with payroll-deducted circles:** emphasise governance, statutory compliance and registered-society requirements Rank's consumer product does not meet.

## Honest business review

Calling CoopHub "the biggest competitor" is half right. It is the only incumbent with the right architecture, and the only one a cautious committee will compare us to. But the evidence says it is not winning the market: the real incumbent is the notebook, and the real threats are Rank and the government.

### Where our plan is exposed

1. **CoopHub's architecture is the one we recommend.** Bank custody plus role-based release is exactly what this design argues for; a committee may reasonably ask why it should not just open a CoopHub account. Our answer must be the payroll and governance features, demonstrated on their own data, not on slides.
2. **Wema can add payroll check-off.** It owns Finacle, claims direct-debit recovery and sells payroll accounts to employers. If it builds reconciliation, our lead narrows. Mitigation: move fast on the first ten staff cooperatives and lock in the employer adapters, which take months to build and are the moat.
3. **Banks control the rails.** Every custody partner, including Wema, decides whether to expose mandate enforcement by API. Without a willing bank, our payment-control promise falls back to in-platform approvals on a single credential, which is weaker than CoopHub's.
4. **Rank has a licence and payroll-deducted products already.** It is a better-funded, faster competitor for staff cooperative members than Wema, even if it does not serve the registered society itself.
5. **The registry and Cooperative Bank may standardise the market around them.** If either ships operating features, independent platforms become integrators.
6. **Cooperatives may not pay.** Neutral SaaS prices (₦10k–₦100k a month) and CoopHub's subscription show willingness to pay exists, but only at society level; ₦200 per member per month must be proven in the pilot.
7. **Our evidence about CoopHub is thin.** It rests on app-store data, terms and conditions and press; we have not spoken to one CoopHub cooperative. The silence could hide quiet, satisfied users on the web portal.

### Wema's likely responses

| If we… | Wema is likely to… | Our move |
| --- | --- | --- |
| Approach as partner | Listen if we bring deposits and features it lacks; slow decision; may want exclusivity | Offer a pilot integration, refuse exclusivity, keep a second bank |
| Win a Wema-banked staff cooperative | Offer the society loans or fee waivers to stay on CoopHub | Show the arrears number CoopHub never produced; let the general meeting decide |
| Grow with federations | Court the federation (which currently names Sterling) | Stay neutral; integrate with any bank the federation chooses |
| Do nothing visible | Leave CoopHub on maintenance as it appears to be now | Use the window; convert CoopHub users through the neutral layer |

### What to do in the next 60 days to test this review

1. Interview 10–15 cooperative treasurers, including at least five CoopHub users (Lagos staff cooperatives from the July 2024 event), on onboarding, payroll, fees and why they stay or leave.
2. Ask Wema directly for CoopHub's price, cooperative count and API position.
3. Run historical payroll reconciliation for two staff cooperatives as the demonstration CoopHub cannot match.
4. Track Rank's group products and the registry's operating scope monthly.

## Sources and confidence

This review draws on three research notes, each with dated, URL-level citations:

- `research_notes/Group finance market Nigeria/coophub_profile.md`: Wema press releases and annual reports (FY2025, H1 2026), Daily Trust (9 Jul 2024), LinkedIn, app-store listings, and CoopHub's FAQ and terms read from the portal's public JavaScript bundle (coophub.alat.ng would not render as text).
- `research_notes/Group finance market Nigeria/coophub_voice_of_customer.md`: Google Play and Apple App Store reviews (read 9 Oct 2026), 8,000 ALAT Play reviews, X, TikTok and Instagram via search snippets, press incident reports.
- `research_notes/Group finance market Nigeria/coophub_competitive_set.md`: bank product pages, cooperative-SaaS vendor sites and pricing, Rank announcements, the Seamfix registry MoU and Cooperative Bank releases, Kenyan benchmarks.

| Claim | Confidence | Why |
| --- | --- | --- |
| Product features and T&Cs | Medium–high | Read from Wema's own portal code and terms |
| "1,000+ cooperatives" | Low as a measure of use | Single 2024 company claim, never updated |
| Low member adoption | Medium | App-store install and rating counts are public; admins may use web only |
| User sentiment | Low | Only 13 reviews exist; forums and Facebook groups not reachable without login |
| Competitor prices and counts | Medium for prices, low for counts | Prices published; client counts self-reported |
| Wema financials and incidents | Medium–high | Annual reports and press; fraud-loss year conflicts across snippets |

Not yet verified: CoopHub's subscription price; any cooperative count after July 2024; whether the three roles bind the bank mandate; Wema's 2024 annual report and earnings-call mentions. Each needs a direct question to Wema or treasurer interviews.
