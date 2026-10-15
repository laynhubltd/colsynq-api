# IAM v2: security review against banking-integration requirements

Reviewed 2026-10-13 against `iam_v2_snapshot.md` (spec v2, all 26 sections). Context read: `auth/*`, `tenancy/*`, `data_model/iam_tab_architect_review.md`, and the IDEC notes `bank_wallet_payout_authorisation.md`, `payout_mechanics_failure_fraud.md`, `integration_onboarding_golive.md` and `baas_*`.

Reviewer stance: a senior security engineer for Nigerian banking and payments integrations. The question is whether this IAM/tenancy design is good enough for a multi-tenant platform that moves money through BaaS wallets and bank channels, takes webhooks from banks and gateways, and exposes machine credentials to partners.

## How to read this

- **Severity.**
  - **Critical:** money can move, or be credited, without the control a bank partner or regulator expects. Fix before any live payout.
  - **High:** a material gap that a bank's vendor due-diligence, a CBN-supervised partner or an NDPC audit would flag. Fix before go-live.
  - **Medium:** fix before scale or before the first partner audit.
  - **Low:** hygiene.
- **Section** is the spec section in `iam_v2_snapshot.md`.
- **Reference** gives the requirement. The URL is a primary source where it could be fetched. CBN PDFs on cbn.gov.ng returned HTTP 403 to automated fetches today, so some CBN points rest on law-firm summaries. Those are marked **[S]**. Confirm them against the circular before quoting them to a bank.

## Applicability: what binds Colsynq, and how

| Regime | Binds Colsynq directly? | How it reaches the platform |
|---|---|---|
| NDPA 2023 + GAID 2025 (in force; replaced the NDPR on 19 Sep 2025) | **Yes.** Colsynq is a controller or processor of members' personal and financial data. It is likely a "data controller of major importance": GAID lists "data-driven financial assets entrusted by data subjects" | Directly: 72 h breach notice, mandatory DPIA for "financial services involving the processing of personal data through digital devices", retention policy, cross-border rules |
| CBN Risk-Based Cybersecurity Framework for DMBs and PSBs (issued 31 May 2024, effective 1 Jul 2024) | No. Colsynq is not a licensee (per `integration_onboarding_golive.md`) | Through the bank or BaaS partner's third-party risk management. The framework requires SFIs to run vendor due diligence, keep audit rights, and require third parties to meet "PCI DSS, NDPR, ISO 27001, ISO 8583" as relevant [S] |
| CBN circular on Additional Functionalities for Instant Payment (12 Mar 2026, effective 1 Jul 2026) | No | Through partners and product parity. Requirements: two independent authentication factors, one device at a time (device binding), ₦20,000 cap for 24 h after new-device activation, enterprise fraud monitoring of inflows and outflows, liveness + BVN/NIN validation on online onboarding and reactivation [S] |
| CBN Addendum to the BVN Operations and Watch-List Framework (12 Mar 2026, effective 1 May 2026) | Yes, indirectly | Only CBN-licensed FIs may access the BVN database. A BVN-linked phone number can be changed once [S] |
| CBN circular on data localisation, market structure and UBO (15 Jun 2026; localisation effective **1 Jan 2027**) | No, but it reaches non-licensed "participants facilitating payments" by contract | Payment and financial transaction data generated in Nigeria must be processed, stored and managed in Nigeria [S] |
| CBN Open Banking Regulatory Framework (2021) + Operational Guidelines (Mar 2023) | Only if Colsynq becomes an API Consumer or Provider | Risk-tiered data, proof of consent, annual re-consent or after 180 days unused, information-security policy, strong authentication, CBN approval for disclosure to non-Nigerian participants [S] |
| CBN AML/CFT/CPF Regulations 2022, CDD Regulations 2023, Baseline Standards for Automated AML Solutions (10 Mar 2026) + MLPPA 2022 | Partners directly; Colsynq by contract and as the record-keeper for societies | KYC tiers, BVN/NIN, transaction monitoring, record retention of at least 5 years after the relationship ends [S] |
| PCI DSS v4.0.1 | **Out of scope today** if card data only ever touches gateway-hosted pages | Banks' vendor questionnaires use PCI DSS Req 3, 7, 8 and 10 as the benchmark, so this review uses them as the bar |
| ISO/IEC 27001:2022, SOC 2 (TSC 2017, rev. 2022) | Voluntary | Bank vendor packs ask for ISO 27001 alignment and a pen-test summary (`integration_onboarding_golive.md`) |
| FAPI 2.0 Security Profile (Final, Feb 2025), FAPI 2.0 Message Signing (Final, Sep 2025) | Voluntary | The best-practice bar for the "partners and banks" OAuth phase (sections 26.8 and 26.9) |

---

## Findings

### Critical

**1. Critical. Sections 2, 10, 11, 14 and 21: one person can move money until an approval module exists.**

- **Problem.**
  - FINANCE_OFFICER holds `payments:execute`, and `supplier-invoices:execute` is a single `execute` action.
  - Section 21 defers the approval workflow, and nothing in the spec stops a money permission from being used before it exists.
  - The permission-level conflict table cannot stop one Treasurer from initiating and releasing the same payment. A custom role can hold both, and conflicts are checked per profile, not per record.
  - Every bank corporate channel reviewed separates the roles. GTBank GAPS has Uploader, Reviewer and Approver with tokens; UBA has initiator, verifier and authoriser; Fidelity has Initiator and Authorizer. Paystack makes approvals a precondition for higher transfer limits (`bank_wallet_payout_authorisation.md`, `payout_mechanics_failure_fraud.md`).
  - BaaS keys can debit any wallet (Embedly, VFD, Anchor), so Colsynq's own maker-checker is the only maker-checker.
- **Reference.**
  - ISO/IEC 27001:2022 A.5.3 (segregation of duties): https://www.iso.org/standard/27001
  - SOC 2 CC5.1/CC6.1: https://www.aicpa-cima.com/resources/download/2017-trust-services-criteria-with-revised-points-of-focus-2022
  - CBN Risk-Based Cybersecurity Framework 2024, Part 3 access control [S]: https://www.cbn.gov.ng/Out/2024/BSD/CBN%20Risk-Based%20Cybersecurity%20Framework%20for%20DMBs%20and%20PSBs_2024.pdf
  - The provider evidence in `payout_mechanics_failure_fraud.md`.
- **Fix.**
  1. Section 10: replace the action list with `list, read, create, update, delete, initiate, approve, release, export, manage`. Remove `execute` for `risk = money`; the catalogue sync rejects `money` slugs with `execute`.
  2. Section 14: add a shared-kernel assertion `DualControlAssertion` (owner IAM), registered for every `money` slug with action `approve` or `release`. It denies with `same_actor` when the acting profile, or the same `app_user` through another profile, created or approved an earlier step of the same record. It reads the record's actor history through an `ActorHistoryProvider` the owning module implements.
  3. Section 11: add default conflicts `*:initiate` ⟂ `*:release` (same_scope_only = yes, severity = block) for payments, payouts, loan disbursements and refunds. Add `beneficiaries:manage` ⟂ `payouts:approve` (block). A beneficiary or bank-detail change is the main payout-fraud vector in `payout_mechanics_failure_fraud.md`.
  4. Section 21, Approval workflow row, replacement text: "**Blocking dependency.** No permission with `risk = money` and action `approve` or `release` may be granted, and no money endpoint may be routed, until the approval module is live. The catalogue sync fails the deploy if a money slug with `approve`/`release` exists while the feature flag `approval_module` is off. The approval module must enforce: maker ≠ checker; N approvers above a society threshold; two approvers for any beneficiary or bank-account change; rejection on timeout (fail closed)."

**2. Critical. Section 14 (StepUpAssertion) and section 26.2: there is no transaction signing or dynamic linking for payouts.**

- **Problem.**
  - Step-up checks `amr` and `auth_time` on the session JWT. Any request in the next N minutes can then pay any amount to any beneficiary.
  - A session hijacked by XSS, a malicious extension or a stolen phone can redirect a payout the officer believes they approved.
  - CBN has required MFA for "Highly Secured Online Funds Transfer" since 2022 (circular PSM/DIR/PUB/CIR/01/006). The March 2026 instant-payment circular requires two independent factors for instant payments.
  - The best-practice pattern is PSD2 RTS Art. 5: the authentication code is bound to the amount and the payee, and any change invalidates it.
- **Reference.**
  - Commission Delegated Regulation (EU) 2018/389, Art. 5: https://eur-lex.europa.eu/eli/reg_del/2018/389/oj
  - CBN circular PSM/DIR/PUB/CIR/01/006 (26 May 2022) [S]: https://regalert.today/document/d706c17d-8187-4b16-9a2f-f9c8172a1cd0/text
  - CBN Instant Payment circular (12 Mar 2026) [S]: https://openafricapod.substack.com/p/cbn-releases-new-circulars-on-instant
  - W3C WebAuthn L3 (challenge binding): https://www.w3.org/TR/webauthn-3/
- **Fix.**
  - Add the table `iam.transaction_authorisation`:

    | Column | Type | Notes |
    |---|---|---|
    | id | uuid | PK v7 |
    | tenant_id, user_profile_id | uuid | FK (user_profile_id, tenant_id) → user_profile |
    | action | text | permission slug, e.g. `payouts:approve` |
    | subject_type, subject_id | text, uuid | the batch, payout or loan |
    | amount_minor | bigint | total, kobo |
    | currency | text | NGN |
    | payee_digest | bytea | SHA-256 over the sorted (bank_code, account_number, resolved_name, amount_minor) lines; batches hash the line set |
    | payload_hash | bytea | SHA-256 of the canonical record the server will execute |
    | challenge | bytea | 32 random bytes; WebAuthn challenge = SHA-256(challenge ‖ payload_hash) |
    | method | text | `passkey`, `device_key` |
    | authenticator_id | uuid | FK user_authenticator |
    | signature | bytea | assertion or device signature |
    | status | text | `pending`, `verified`, `used`, `expired`, `rejected` |
    | created_at, expires_at, used_at | timestamptz | CHECK expires_at ≤ created_at + 5 min |

  - Unique (id) WHERE status = 'verified'. Single use.
  - New assertion `TransactionAuthorisationAssertion` (IAM): for `money` slugs with action `approve` or `release`, and for a member's own withdrawal, it requires a `verified`, unexpired row. The row's `payload_hash` must equal the hash recomputed from the live record, and the row's profile must be the acting profile. It sets the row to `used` in the same transaction as the money command. If the record changed after signing, it denies `transaction_changed`.
  - The UI must render amount, payee count, payee names and the total from the signed payload, not from client state.
  - Backup codes, email OTP and SMS OTP never satisfy this assertion.

**3. Critical. Sections 8.5, 11, 15.1 and 15.4: a single TENANT_ADMIN can give themselves money power through a proxy.**

- **Problem.**
  - TENANT_ADMIN has `can_assign_all` and "no money permissions", but can:
    - create a profile for any email they control (section 13.1, "Created by an administrator");
    - grant that profile FINANCE_OFFICER.
  - The only conflict that touches this, `payments:execute` ⟂ `user-roles:create`, is `warn`.
  - Two colluding admins can also grant each other money roles. Gate 0 only stops self-grants.
  - The two-administrator rule protects availability, not integrity.
- **Reference.**
  - ISO/IEC 27001:2022 A.8.2 (privileged access rights) and A.5.18 (access rights): https://www.iso.org/standard/27001
  - PCI DSS v4.0.1 Req. 7.2.1–7.2.2 (access assigned on least privilege and approved by authorised personnel): https://www.pcisecuritystandards.org/document_library/
  - SOC 2 CC6.2/CC6.3.
- **Fix.**
  - Changes to `user_role` (8.4): add
    - `status text` (`pending_approval`, `active`, `rejected`, `revoked`) — `revoked_at` stays;
    - `requested_by_profile_id uuid`;
    - `approved_by_profile_id uuid`;
    - `approved_at timestamptz`;
    - `effective_from timestamptz`;
    - CHECK `approved_by_profile_id IS DISTINCT FROM requested_by_profile_id AND approved_by_profile_id IS DISTINCT FROM user_profile_id`;
    - trigger `user_role_dual_control_ck`: if the role contains any `money` or `high` permission, status `active` needs `approved_by_profile_id`, and the approver's `app_user` differs from the requester's and the target's.
  - Section 15.1, new **gate 5, dual control**: "A grant of a role containing `money` or `high` permissions is created `pending_approval`. A second holder of `user-roles:approve` (TENANT_ADMIN, or a custom role flagged as governance) approves it with step-up. For `money` roles, `effective_from = approved_at + 24 h`. Every TENANT_ADMIN and every holder of a money role is notified at request and at approval."
  - Section 11: make `payments:*` ⟂ `user-roles:create` a **block**. Add `profiles:create` ⟂ `payouts:approve` (block).
  - Section 13.1: a profile created by an administrator cannot receive a `money` role until its user has verified the email *and* enrolled a phishing-resistant authenticator. The user must do both from their own session, not the admin's.

**4. Critical. Sections 12.2 and 14 ("Workers and webhooks"): inbound webhooks from banks and gateways are unspecified.**

- **Problem.**
  - The resolver resolves tenants from `X-Tenant` or the host. A webhook carries neither, and an attacker can set both.
  - The spec defines no webhook authenticator, no signature-secret storage, no replay defence, no dedupe and no tenant mapping.
  - Provider schemes are weak or inconsistent:
    - Paystack: `x-paystack-signature` = HMAC-SHA512(body, *secret API key*); no timestamp.
    - Monnify: `monnify-signature` = SHA-512/HMAC-SHA512 with the client secret, from a single IP 35.242.133.146; no timestamp, and absent in sandbox.
    - Flutterwave: v3 `verif-hash` is a static shared secret compared verbatim; v4 adds `flutterwave-signature` HMAC-SHA256.
    - Embedly: two doc portals with conflicting schemes. `x-embedly-signature` is HMAC-SHA512 keyed with the API key.
  - None of these signs a timestamp, so a captured, valid webhook can be replayed. Several retry for up to 72 h, so duplicates are normal.
  - A forged or replayed "credit" webhook posts money into a member's savings ledger.
- **Reference.**
  - Paystack webhooks: https://paystack.com/docs/payments/webhooks/
  - Monnify webhooks: https://developers.monnify.com/docs/webhooks
  - Embedly webhooks: https://developer.embedly.ng/api-reference/webhooks.md
  - Standard Webhooks (signed `webhook-id`, `webhook-timestamp`, 5-minute tolerance): https://www.standardwebhooks.com/
  - OWASP ASVS 5.0: https://owasp.org/www-project-application-security-verification-standard/
- **Fix.**
  - Section 12.2: add `TenantRequirement::Webhook`. A webhook route never reads `X-Tenant` or the host. Its authenticator (one per provider) verifies the signature over the **raw bytes** with constant-time compare, checks the source IP against the provider list as a second layer, and only then resolves the tenant. The tenant comes from a provider-side identifier: virtual account number, provider customer or wallet ID, or the transfer reference Colsynq generated. It maps through `integration.provider_account_link (provider, external_ref, tenant_id, kind, created_at, unique (provider, external_ref))`. An unknown external_ref gives 202 + quarantine, never a default tenant.
  - New table `integration.inbound_webhook`:
    - `id`
    - `provider`
    - `provider_event_id`, unique (provider, provider_event_id); for providers without an event ID, use SHA-256 of the raw body
    - `received_at`
    - `source_ip inet`
    - `signature_scheme`
    - `signature_valid boolean`
    - `raw_body_sha256 bytea`
    - `raw_body_encrypted bytea`
    - `tenant_id`
    - `status` (`received`, `verified`, `requeried`, `applied`, `duplicate`, `rejected`, `quarantined`)
    - `applied_at`
    - `error`
  - Rule: **no ledger posting from a webhook body.** The worker re-queries the provider's transaction-status API by the Colsynq reference. It posts only the provider's answer, idempotently on (provider, provider_txn_ref).
  - Respond 2xx within 2 s and process asynchronously.
  - Webhook secrets live in the vault per provider account, with overlap rotation (old secret valid 24 h).
  - Signature failures write `audit_event` `webhook.signature_failed` and alert at more than 5 per 10 minutes.
  - Outbound webhooks Colsynq sends later use the Standard Webhooks scheme (id + timestamp + HMAC-SHA256, 5-minute tolerance).

### High

**5. High. Sections 7.3 and 25.13 (api_key), 13.1 (service account) and 14 (StepUpAssertion): service accounts contradict step-up, and API keys are weak bearer credentials for money endpoints.**

- **Problem.**
  - Service users have "no password, no MFA", while StepUpAssertion requires MFA for `money` and `high` permissions. Payroll check-off integrations either cannot work, or someone adds a bypass.
  - API keys are plain bearer secrets valid up to 1 year. `ip_allow_list` is optional, there are no per-key scopes, and there is no record of who created or approved a key.
  - `lookup_hash` uses a single pepper with no version, so rotating the pepper breaks every key.
- **Reference.**
  - PCI DSS v4.0.1 Req. 8.6.1–8.6.3 (system and application accounts: interactive use controlled, no hard-coded credentials, periodic change per targeted risk analysis) and 8.3.2 (strong cryptography): https://www.pcisecuritystandards.org/document_library/
  - ISO A.5.17.
  - RFC 8705 (mTLS-bound tokens): https://www.rfc-editor.org/rfc/rfc8705
  - RFC 9421 (HTTP Message Signatures): https://www.rfc-editor.org/rfc/rfc9421
  - Paystack and Flutterwave IP whitelisting as a limit precondition (`payout_mechanics_failure_fraud.md`).
- **Fix.**
  - `api_key` adds:
    - `environment text` (`live`, `test`), with key format `csk_live_<prefix>_<secret>` so secret scanners can detect it;
    - `scopes text[] NOT NULL` (subset of the profile's effective permissions; trigger);
    - `created_by_profile_id`, `approved_by_profile_id`, and CHECK they differ;
    - `pepper_version smallint NOT NULL`;
    - `rotated_from_id uuid`;
    - `last_used_ip inet`;
    - `auth_method text` (`bearer`, `mtls`, `http_sig`);
    - `client_cert_sha256 bytea` and `public_key_jwk jsonb`.
  - Trigger `api_key_money_ck`: if `scopes` contains any `money` slug, then:
    - `ip_allow_list` is NOT NULL and non-empty;
    - `auth_method IN ('mtls','http_sig')`;
    - `expires_at ≤ created_at + 90 days`;
    - only `*:initiate` and `*:create` money actions are allowed — never `approve` or `release`.
  - Section 14, StepUpAssertion replacement text: "Service principals never satisfy step-up. A service principal may hold only `initiate`/`create` money actions. Every money record it creates enters the approval workflow and needs human approval with transaction authorisation (finding 2)."
  - Revoke keys unused for 90 days (PCI 8.2.6 analogue).
  - Creating, rotating or widening a key is dual-controlled (finding 3, gate 5).

**6. High. Sections 7.3, 25.x (`phone` "SMS one-time codes"), 14 and 26.2: the MFA factor model is inconsistent and too weak for privileged and money roles.**

- **Problem.**
  - `user_authenticator.type` allows `email_otp` as a second factor. NIST SP 800-63B-4 says "Email SHALL NOT be used for out-of-band authentication".
  - Section 25 says `phone` is for "SMS one-time codes", but there is no `sms_otp` type. This is a contradiction.
  - `amr` values `pwd, otp, hwk` cannot tell TOTP from email OTP, or a synced passkey (`swk`) from a hardware-bound one.
  - There is no rule that TENANT_ADMIN, money roles or platform staff must use MFA at sign-in. It is only required at step-up time, or for "MFA-required tenants".
  - Nigeria's SIM-swap and social-engineering fraud profile makes SMS unsuitable for authorisation. NIBSS reports social engineering as the leading technique [S].
- **Reference.**
  - NIST SP 800-63B-4 (Final, Jul 2025): email prohibited for OOB; PSTN "restricted", verifiers SHOULD check "device swap, SIM change, number porting"; AAL2 verifiers SHALL offer a phishing-resistant option; AAL3 needs a non-exportable key: https://pages.nist.gov/800-63-4/sp800-63b/authenticators/ and https://pages.nist.gov/800-63-4/sp800-63b/aal/
  - PCI DSS v4.0.1 8.4.1–8.4.2 (MFA for all administrative and non-console access) and 8.5.1 (MFA not replayable, not bypassable, at least two different factor types).
  - ISO A.8.5.
  - RFC 8176 (amr values): https://www.rfc-editor.org/rfc/rfc8176
- **Fix.**
  - `user_authenticator.type` becomes `totp`, `passkey`, `device_key`, `transaction_pin`, `backup_codes`, `email_otp`, `sms_otp`. Add the columns:
    - `aal smallint`;
    - `phishing_resistant boolean`;
    - `hardware_bound boolean` (from WebAuthn attestation or BE/BS flags);
    - `usable_for_money_after timestamptz` (finding 7);
    - `enc_key_id text` (finding 10).
  - `email_otp` and `sms_otp` are **recovery or notification only**. They never satisfy StepUpAssertion for `high` or `money`.
  - JWT `amr` uses RFC 8176 values (`pwd`, `otp`, `sms`, `hwk`, `swk`, `pin`, `user`, `mfa`) plus a private `email` value. Add `acr` ∈ {`aal1`, `aal2`, `aal2-pr`, `aal3`}.
  - New section 14 table, **assurance by risk**:

    | Permission or role | Required at use |
    |---|---|
    | low | aal1 |
    | high | aal2 (non-email, non-SMS), `auth_time` ≤ 15 min |
    | money initiate | aal2, `auth_time` ≤ 15 min |
    | money approve/release, member withdrawal | `aal2-pr` (passkey or bound device key) + transaction authorisation (finding 2) |
    | Any profile holding TENANT_ADMIN or a money role, in any tenant | aal2-pr at every sign-in |
    | Platform roles | aal3 (hardware-bound key) at every sign-in |

  - Use the RFC 9470 challenge (`WWW-Authenticate: Bearer error="insufficient_user_authentication", acr_values="aal2-pr", max_age=300`) for the existing 401 `step_up_required`: https://www.rfc-editor.org/rfc/rfc9470

**7. High. Sections 7.3, 13.3 and 18 (`/api/me/authenticators`, password reset): MFA enrolment and account recovery can bypass step-up.**

- **Problem.**
  - Nothing says a *new* authenticator needs an existing one, so a password thief can enrol their own TOTP and then pass step-up.
  - Password reset is by email link, so an email takeover becomes a Colsynq takeover.
  - The spec says nothing about whether reset clears MFA, and there is no cooling-off or all-channel notice after a credential change.
  - The CBN March 2026 circular caps transactions at ₦20,000 for 24 h after a new device activation and requires extra authentication on a new device.
- **Reference.**
  - NIST SP 800-63B-4 §4.1 (binding an additional authenticator requires authentication at the highest AAL in use, or a recovery process) and §4.2: https://pages.nist.gov/800-63-4/sp800-63b.html
  - CBN Instant Payment circular (2026) [S]: https://globallawexperts.com/cbns-new-instantpayment-security-minimums-live-2/
- **Fix.**
  - `app_user` adds `security_hold_until timestamptz` and `security_hold_reason text`.
  - Rules for section 7.3:
    1. Adding an authenticator needs a session authenticated at least at the highest AAL the user already has.
    2. Recovery without a factor (backup code, or administrator-assisted with identity re-proofing) sets `security_hold_until = now() + 72 h`.
    3. Every new authenticator, phone change, email change or password reset sets `usable_for_money_after = now() + 24 h` on the new authenticator. It also sets `security_hold_until = now() + 24 h`. During the hold, money approve/release, beneficiary changes and member withdrawals above ₦20,000 are denied with `security_hold`.
    4. Password reset never removes or bypasses authenticators.
    5. Every such change notifies through all verified channels, and the notice includes a "this wasn't me" link that revokes sessions.
  - Add `SecurityHoldAssertion` (IAM) to section 14.

**8. High. No device binding, and SIM swap is not handled for mobile (new requirement).**

- **Problem.**
  - Members will use a mobile app to request withdrawals and loan disbursements, and officers to approve.
  - The spec stores refresh tokens with no device identity. It has no binding, attestation or new-device rule.
  - CBN's 2026 rules for banks (one device at a time; restricted first 24 h) set the market expectation, and partners are likely to flow it down.
  - Nigerian operators recycle numbers, and the BVN-linked phone can now change only once. Both make the phone a poor identity anchor.
- **Reference.**
  - CBN Instant Payment circular, 12 Mar 2026 [S]: https://openafricapod.substack.com/p/cbn-releases-new-circulars-on-instant
  - NIST SP 800-63B-4 PSTN restrictions (above).
  - OWASP MASVS-AUTH: https://mas.owasp.org/MASVS/
- **Fix.**
  - New table `iam.user_device`:
    - `id`, `user_id` (FK);
    - `platform` (`ios`, `android`, `web`);
    - `device_public_key bytea` (key generated in Secure Enclave or StrongBox);
    - `attestation_type` (`app_attest`, `play_integrity`, `none`) and `attested_at`;
    - `bound_at`, `last_seen_at`;
    - `status` (`active`, `revoked`, `superseded`);
    - `revoked_at`, `revoked_reason`.
    - Partial unique `(user_id) WHERE status = 'active' AND platform <> 'web'`: one mobile device at a time.
  - `refresh_token` adds `device_id` (FK user_device).
  - A `device_key` authenticator (finding 6) is the bound key plus an app PIN or biometric (user verification). A new binding supersedes the old one, revokes its sessions, and sets the 24 h hold (finding 7).
  - SMS is sent only after a SIM-swap or porting check through an aggregator's telco API, and never as an authorisation factor.
  - A phone-number change counts as binding a new authenticator.

**9. High. Sections 26.1 and 26.3: there are no idle timeouts, and privileged sessions last 14–30 days.**

- **Problem.**
  - Every session gets a 14-day rotating refresh token with a 30-day cap. Nothing ends an idle finance officer's or admin's session.
- **Reference.**
  - PCI DSS v4.0.1 8.2.8: re-authenticate after more than 15 minutes idle.
  - NIST SP 800-63B-4: AAL2 SHOULD ≤ 1 h idle and ≤ 24 h overall; AAL3 SHALL ≤ 12 h overall and SHOULD ≤ 15 min idle: https://pages.nist.gov/800-63-4/sp800-63b/aal/
  - OWASP ASVS 5.0 V7 (session management).
- **Fix.**
  - `refresh_token` (gesdinet entity extension) adds:
    - `acr text`;
    - `device_id uuid`;
    - `ip inet`, `user_agent text`;
    - `last_activity_at timestamptz`;
    - `idle_timeout interval`;
    - `absolute_expires_at timestamptz`.
  - Section 26.1 policy table:

    | Session type | Idle timeout | Absolute cap |
    |---|---|---|
    | Platform roles | 15 min | 12 h |
    | Any user holding TENANT_ADMIN or a money/high role in any tenant | 15 min | 12 h |
    | Members on web | 30 min | 24 h |
    | Members on a bound mobile device | 30-day refresh; app re-lock (PIN or biometric) after 5 min in background | — |

  - A user's strictest profile sets the policy.
  - Maximum 5 concurrent sessions per user; the oldest is evicted.
  - `/api/me/sessions` lists `device_id`, IP, ASN and country, and each session can be revoked.
  - Redis-backed checks (jti blocklist, refresh-reuse cache, OTP replay cache) **fail closed** on money and high routes when Redis is unavailable.

**10. High. Section 5 (Secrets) and section 7.3 (`secret_encrypted`, `payload_encrypted`, `lookup_hash` pepper), 26.2 (JWT key): key management is undefined.**

- **Problem.**
  - Encrypted columns have no key identifier, so keys cannot rotate.
  - There are no per-tenant keys, even though the tenancy notes recommend a per-tenant DEK and KMS key, with crypto-shredding on offboarding (`tenancy/db_per_tenant_operations.md`).
  - The ES256 private key is loaded from the vault into the application, so it is exportable.
  - No HSM or KMS boundary is stated for keys that sign money-moving sessions or hold BaaS credentials.
- **Reference.**
  - PCI DSS v4.0.1 Req. 3.6–3.7 (key-management procedures: generation, distribution, storage, cryptoperiod, retirement, split knowledge, dual control).
  - ISO A.8.24 (use of cryptography).
  - NIST SP 800-57 Pt 1 Rev 5: https://csrc.nist.gov/pubs/sp/800/57/pt1/r5/final
- **Fix.**
  - Section 5, Secrets row, replacement text: "Envelope encryption. Each tenant has a DEK wrapped by a tenant KMS key (`tenant_database.kms_key_ref`). Landlord secrets use a landlord key. Every encrypted column has a sibling `enc_key_id text NOT NULL`. Peppers carry `pepper_version`. The JWT signing key, transaction-authorisation verification keys and BaaS mTLS keys are non-exportable keys in an HSM-backed KMS (FIPS 140-3 Level 3 for production signing), or in Vault Transit with an HSM seal. The application calls sign or decrypt and never holds the key. Cryptoperiods: JWT signing 90 days (kid overlap of one token lifetime), DEKs 1 year with lazy re-encryption, peppers 1 year (dual-lookup during rotation). Key ceremonies need two custodians."
  - `tenant_database` adds:
    - `kms_key_ref text NOT NULL`;
    - `backup_kms_key_ref text`;
    - `encryption_status` (`ok`, `rotating`, `shredded`);
    - `key_rotated_at`.
  - Tenant offboarding destroys the tenant key only after the retention period (finding 14).

**11. High. Section 5 (Secrets) and 6.2: outbound bank and BaaS credentials, egress and request signing are not modelled.**

- **Problem.**
  - The only external secret modelled is the database credential.
  - The platform will hold Embedly `x-api-key`, Anchor `x-anchor-key`, VFD tokens, Providus credentials, possibly per-society Kuda or Lenco keys, PGP keys (Remita SOAP), mTLS client certificates (Wema) and webhook secrets.
  - Several of these can debit **any** wallet under the organisation (`bank_wallet_payout_authorisation.md`).
  - Banks ask for static egress IPs and a key-rotation runbook (`integration_onboarding_golive.md`).
- **Reference.**
  - CBN Risk-Based Cybersecurity Framework 2024, third-party risk and Part 5.2 API security [S].
  - PCI DSS 8.6.2/8.6.3.
  - ISO A.5.17, A.8.24.
  - Paystack IP whitelisting (owner emailed on change): https://support.paystack.com/en/articles/2130946
- **Fix.**
  - New table `tenancy.tenant_integration`, or `integration.provider_credential` if an Integration module owns it:
    - `id`
    - `tenant_id` (null = platform-level)
    - `provider`
    - `environment` (`sandbox`, `live`)
    - `credential_kind` (`api_key`, `oauth_client`, `mtls_cert`, `pgp_key`, `webhook_secret`)
    - `secret_ref text` (vault path; never the value)
    - `key_version`
    - `scopes_at_provider text[]`
    - `egress_profile text` (`static-egress-a`)
    - `provider_ip_whitelist_confirmed_at`
    - `rotate_by timestamptz`, `rotated_at`
    - `created_by_profile_id`, `approved_by_profile_id` (CHECK they differ)
    - `status`
  - All calls to banks and BaaS go through one egress proxy with fixed IPs and a hostname allow-list. This also blocks SSRF via tenant-supplied URLs (`logo_url`, custom domains, future webhook endpoints).
  - Use mTLS where the provider supports it. Sign requests (RFC 9421 or the provider's scheme) where offered.
  - Mirror Colsynq's limits at the provider wherever the provider offers them: Paystack approval URL, Flutterwave Payout Rules, IP whitelisting.
  - Creating, rotating or widening a credential is dual-controlled.

**12. High. Section 5 (Audit), 8.6 (audit_event) and 25.24: the audit log is not tamper-evident, does not cover security events or money events, and has no time-source rule.**

- **Problem.**
  - "INSERT and SELECT only" stops the application, but not a DBA, the maintenance role (which can drop partitions) or a compromised superuser.
  - There is no hash chain, no external anchor and no WORM copy.
  - The audit table lives in the landlord database and is "written in the same transaction as each change". That is impossible for money events in tenant databases, and the spec does not say where money-movement audit lives.
  - Missing events:
    - authorisation denials on high and money routes;
    - step-up and transaction-authorisation outcomes;
    - reads and exports of `audit_event`;
    - webhook verification failures;
    - API-key use on money routes.
  - There is no clock-synchronisation requirement.
- **Reference.**
  - PCI DSS v4.0.1:
    - 10.2.1.x (events to log, including access to audit logs, invalid access attempts, and changes to authentication credentials);
    - 10.3.2–10.3.4 (protect logs from modification; file-integrity or change detection);
    - 10.5.1 (keep 12 months, 3 months immediately available);
    - 10.6.x (time synchronisation).
  - ISO A.8.15 (logging), A.8.16 (monitoring), A.8.17 (clock synchronisation).
  - SOC 2 CC7.2.
- **Fix.**
  - `audit_event` adds:
    - `chain_id uuid` (tenant ID, or the platform constant);
    - `seq bigint` (per chain, assigned under `pg_advisory_xact_lock(chain)`);
    - `prev_hash bytea`;
    - `row_hash bytea` = SHA-256(canonical_json(row without row_hash) ‖ prev_hash);
    - `recorded_at timestamptz DEFAULT clock_timestamp()` (DB clock, alongside the application's `occurred_at`).
  - A job `audit:checkpoint` runs every 15 minutes. It signs (chain_id, last seq, last row_hash) with a KMS asymmetric key and writes the checkpoint to object storage with Object Lock in compliance mode (or a local equivalent in Nigeria; finding 13).
  - Every event is streamed to a SIEM within 1 minute.
  - A partition is dropped only after its export has been verified against the checkpoints.
  - Each tenant database gets the same `audit_event` table and chain (actor = profile ID) for money and ledger events, written in the business transaction. The landlord chain records IAM events only. State this in section 3.
  - Add the missing events listed above.
  - Section 5, new row: "**Time:** all hosts sync with chrony to at least two internal NTP sources fed from national or public stratum-1 sources. Alert on drift above 100 ms. The JWT `iat`/`exp` leeway is at most 30 s."

**13. High. Sections 1, 5 and 6.1 (`region`) and 6.2: there is no data-residency rule, despite the 1 Jan 2027 CBN deadline, and no cross-border transfer control.**

- **Problem.**
  - `tenant.region = ng` is cosmetic. Nothing constrains where landlord and tenant databases, backups, Redis, the vault, logs, the SIEM, email/SMS providers or error tracking run.
  - The CBN localisation directive requires Nigerian payment and transaction data to be processed and stored in Nigeria from 1 Jan 2027. It reaches non-licensed participants by contract.
  - The NDPA restricts cross-border transfer to an adequacy decision, a CBDTI approved by the Commission, or another lawful basis (GAID Schedule 5). Cross-border transfer is also a mandatory DPIA trigger (GAID Art. 28(3)(o)).
  - CBN open banking requires CBN approval to disclose customer data to non-Nigerian participants [S].
- **Reference.**
  - CBN circular of 15 Jun 2026 [S]: https://www.mondaq.com/nigeria/financial-services/1813412/cbns-data-localisation-mandate-what-nigerian-fintechs-need-to-know
  - GAID 2025, Art. 45 and Schedule 5: https://ndpc.gov.ng/wp-content/uploads/2025/07/NDP-ACT-GAID-2025-MARCH-20TH.pdf
- **Fix.**
  - Section 5, new row: "**Residency.** Production data planes (landlord, tenant databases, replicas, backups, Redis, vault, object storage, logs, SIEM) are hosted in Nigeria. `tenant_database.hosting_zone text NOT NULL` is checked against an allow-list table `tenancy.hosting_zone (code, country = 'NG', provider, is_active)`. Sub-processors outside Nigeria (email, SMS, monitoring) receive no transaction data and only the minimum personal data, under a CBDTI. They are listed in a sub-processor register reviewed every quarter."
  - Make `tenant.region` NOT NULL with a CHECK (`region = 'ng'`) until a multi-country design exists.

**14. High. Sections 5 (Retention and erasure), 12.1 (TenantDeletedV1) and 13.3 (Erase): erasure and tenant deletion conflict with AML record-keeping and the spec's own "rows never deleted".**

- **Problem.**
  - "Erase blanks email, phone, names" and "IAM removes its profiles, grants, keys" on tenant deletion conflict with:
    - section 8.4 ("rows never deleted");
    - the 7-year audit;
    - AML record-keeping of at least 5 years after the relationship ends (MLPPA 2022 [S]);
    - a cooperative's statutory financial records.
  - GAID allows retention that is "consistent with relevant laws", but it requires a written retention policy that is communicated to data subjects.
- **Reference.**
  - NDPA s.24(1)(d) and GAID retention guidance (Schedule): https://ndpc.gov.ng/wp-content/uploads/2025/07/NDP-ACT-GAID-2025-MARCH-20TH.pdf
  - MLPPA 2022 record-keeping [S]: https://lawpadi.com/money-laundering-prevention-and-prohibition-act-2022/
- **Fix.**
  - Section 13.3, Erase row, replacement text: "Erasure request → `legal_hold_check`. If any tenant holds financial records for the user within the retention window, processing is **restricted**: the account is deactivated, contact data is encrypted under a hold key, and the data is excluded from all non-statutory use. It is blanked when the hold ends. Otherwise blank as today."
  - `app_user` adds `restricted_at` and `legal_hold_until`.
  - Section 12.1 replacement text: "Deleting a tenant marks its profiles `removed` and revokes grants and keys. No IAM rows are deleted. After the longest retention period (7 years for audit), the tenant key is destroyed (crypto-shred) and the rows are anonymised."
  - Add a retention schedule table to section 5:

    | Data | Retention |
    |---|---|
    | audit_event | 7 years |
    | Transaction records and KYC | 5 years after relationship end, or longer per the society's statutory books |
    | refresh_token | 30 days after expiry |
    | one_time_token | 7 days after use or expiry |
    | inbound_webhook raw body | 13 months |
    | Session telemetry | 13 months |

**15. High. Section 12.1 (tenant creation): platform staff alone choose the first administrators of a society that will control money.**

- **Problem.**
  - One PLATFORM_ADMIN posts "two admin emails". Those become TENANT_ADMINs, who (finding 3) can then create money roles.
  - A platform insider, or a phished platform admin, can seed attacker-controlled admins.
  - Society signatories are never verified against the cooperative's registration or the partner bank's KYB.
- **Reference.**
  - CBN CDD Regulations 2023 (beneficial owners and signatories) [S]: https://www.templars-law.com/app/uploads/2023/07/CBN-Customer-Due-Diligence-Regulations-2023-%E2%80%93-Key-Highlights-and-Implications-for-Businesses-and-Financial-Institutions.pdf
  - Partner KYB packs (`integration_onboarding_golive.md`).
  - ISO A.5.16 (identity management).
- **Fix.**
  - `tenant` adds:
    - `kyb_status` (`pending`, `verified`, `rejected`);
    - `kyb_verified_at`;
    - `kyb_verified_by_profile_id` (COMPLIANCE_OFFICER; CHECK ≠ `created_by_profile_id`);
    - `kyb_reference`.
  - Provisioning needs `tenants:create` (PLATFORM_ADMIN) **and** `tenants:approve` (COMPLIANCE_OFFICER, a different person).
  - Bootstrap admins must be named signatories or officers on the registration documents. Their identity is verified through the licensed partner (BVN/NIN match) before their grants activate.
  - New Tenancy assertion `KybVerifiedAssertion`: deny every `money` slug while `kyb_status <> 'verified'`.

**16. High. Sections 13 and 13.1 (self sign-up with SELF_SERVICE): money self-service has no identity-assurance gate, and there is no BVN/NIN data handling rule.**

- **Problem.**
  - Anyone who can verify an email can join an `open` tenant and hold `me-*` permissions.
  - Nothing ties a member's withdrawal or loan disbursement to a proofed identity, or to a payout account whose name matches it.
  - `require_email_verification` can be false for `open` tenants.
  - Under the March 2026 BVN addendum, only CBN-licensed FIs may access the BVN database, so Colsynq must verify through a partner and must not build its own BVN store.
- **Reference.**
  - CBN BVN addendum (2026) [S]: https://openafricapod.substack.com/p/cbn-releases-new-circulars-on-instant
  - CBN tiered KYC and AML/CFT/CPF Regulations 2022 [S].
  - NIST SP 800-63A-4 (IAL2).
- **Fix.**
  - Shared kernel adds `IdentityAssurance` (`none`, `email_verified`, `bvn_verified`, `bvn_nin_verified`).
  - The membership module owns `member.kyc_level` and exposes it through a provider.
  - New assertion `KycLevelAssertion`: `me-*` money actions need `bvn_verified` or higher, and the payout account must pass name enquiry with a fuzzy match to the KYC name (threshold configured; mismatches go to review).
  - Store only a partner verification reference, a peppered HMAC of the BVN/NIN for duplicate detection, and the last 4 digits. Never store the raw value. Never log it.
  - CHECK on `tenant`: `require_email_verification = true`; drop the option.

### Medium

**17. Medium. Section 16 and 8.6 (support_access_grant): break-glass cannot be recorded and has no second approver.**

- **Problem.**
  - Emergency access "without a grant" by PLATFORM_OWNER has no row: `granted_by_profile_id` must be a society profile.
  - There is no second approver, incident reference or post-use review.
  - Direct production database access by engineers (outside the API) is not covered at all.
- **Reference.** ISO A.8.2, A.8.18; PCI DSS 7.2.5, 8.2.2, 10.2.1.2; SOC 2 CC6.3.
- **Fix.**
  - `support_access_grant` adds:
    - `kind` (`tenant_granted`, `emergency`);
    - `approved_by_platform_profile_id` (required for `emergency`; a different PLATFORM_OWNER);
    - `incident_ref`;
    - `review_due_at` (= expires_at + 2 business days);
    - `reviewed_by_profile_id`, `reviewed_at`.
  - Relax the FK: `granted_by_profile_id` is required when `kind = 'tenant_granted'`.
  - Section 16, new text: "Human access to production databases and the vault is just-in-time through a PAM gateway. It needs a ticket and a second approver, is session-recorded, uses a personal identity (no shared superuser), and expires within 4 hours. Every session writes `audit_event` `privileged_session.*`. Break-glass superuser credentials are sealed in the vault with two-person retrieval and rotated after use."

**18. Medium. Sections 8.5, 8.6, 15.4 and 18 (tenant settings): security-relevant tenant configuration changes need only one admin and give no notice.**

- **Problem.**
  - One TENANT_ADMIN can do each of these alone:
    - open the sign-up policy;
    - add a custom domain (which hosts the society's sign-in page and binds refresh-token origins);
    - add tenant `allow` edges;
    - add tenant conflict rows;
    - issue an `impersonate` support grant;
    - create service accounts.
- **Reference.** ISO A.5.3, A.8.32 (change management); SOC 2 CC8.1.
- **Fix.**
  - Generic table `iam.dual_control_request`:
    - `id`, `tenant_id`;
    - `action`;
    - `subject_type`, `subject_id`;
    - `payload jsonb`, `payload_hash bytea`;
    - `requested_by_profile_id`, `approved_by_profile_id` (CHECK they differ);
    - `status` (`pending`, `approved`, `rejected`, `expired`);
    - `expires_at` (≤ 72 h);
    - `decided_at`.
  - Commands that need it: signup-policy change, tenant-domain add, allow-edge add, support grant with mode `impersonate`, service-account or API-key create or widen, provider-credential change, and grant of money or high roles (finding 3).
  - The approver re-authenticates at aal2-pr.
  - All TENANT_ADMINs are notified on every request.
  - This sits in IAM, not the business approval module, so IAM security does not depend on that module.

**19. Medium. Section 21 (Governance, "access recertification" out of scope): there is no periodic access review and no dormancy control.**

- **Reference.**
  - ISO A.5.18 (review access rights at planned intervals).
  - PCI DSS v4.0.1 7.2.4 (review all user accounts at least every 6 months) and 8.2.6 (inactive accounts removed or disabled within 90 days).
  - SOC 2 CC6.2.
- **Fix.**
  - `user_role` adds `last_reviewed_at`, `reviewed_by_profile_id` and `review_due_at`.
  - Job `iam:access-review:due`: money roles every 90 days, high roles every 180 days, others yearly. 30 days past due, the grant is suspended (`status = 'pending_approval'`) and admins are notified.
  - Job `iam:dormancy`: profiles holding money or high roles with no sign-in for 90 days have those grants suspended. API keys unused for 90 days are revoked.
  - Move "access recertification" out of section 21 into section 15.

**20. Medium. Section 7.1 (`failed_login_count`, `locked_until`) and the referenced Authentication tab: lockout and password policy are not stated for privileged users.**

- **Problem.**
  - A "short, growing back-off (minutes)" is fine for members, but the PCI benchmark for administrative users is lock after ≤ 10 failures for ≥ 30 min, or until identity is confirmed.
  - The minimum password length is not stated in this spec.
  - Transaction PINs (added in finding 6) need their own counter.
- **Reference.**
  - PCI DSS v4.0.1 8.3.4 and 8.3.6 (minimum 12 characters).
  - NIST SP 800-63B-4 §3.1.1 (15 characters if single-factor, 8 if part of MFA; blocklist of breached passwords; no composition rules).
  - PSD2 RTS Art. 4(3)(d) (at most 5 failed attempts for an authentication code).
- **Fix.**
  - Section 7.1 replacement text: "Passwords: minimum 12 characters (15 when the account has no second factor), maximum at least 64, checked against a breached-password list, with no composition rules. Lockout: members get exponential back-off from 5 failures; profiles with admin, money or high roles lock for 30 min after 10 failures and alert the user. Transaction PIN: 5 failures lock money authorisation until re-binding through recovery."
  - Add `user_authenticator.failed_count smallint` and `locked_until timestamptz`.

**21. Medium. Section 19 and `audit_event`: there are no fraud or anomaly signals, and no incident-notification path.**

- **Problem.**
  - Sign-in events exist, but there are no risk fields: new device, new ASN or country, impossible travel, many profiles per device, many failed transaction PINs, or a change of payee followed by a payout.
  - CBN's 2026 circular requires enterprise fraud monitoring of inflows and outflows from licensees. CBN's AML baseline standards (10 Mar 2026; full compliance 18 months for DMBs and 24 months for OFIs) require real-time monitoring integrated with KYC. Partners will ask Colsynq for both signals and evidence.
  - Breach notice: NDPA s.40(2) and GAID Art. 33 require notice to the NDPC within 72 h. Data subjects must be told immediately where there is a high risk, which includes any risk of fraud or identity theft.
  - CBN-supervised partners must report cyber incidents to the CBN within 24 h [S], so contracts will demand faster notice from Colsynq.
- **Reference.**
  - GAID Art. 33: https://ndpc.gov.ng/wp-content/uploads/2025/07/NDP-ACT-GAID-2025-MARCH-20TH.pdf
  - CBN AML Baseline Standards [S]: https://businessday.ng/news/article/banks-fintechs-face-new-cbn-deadline-for-automated-aml-systems/
  - Mondaq on the CBN cyber framework (24 h) [S]: https://www.mondaq.com/nigeria/security/1518574/
- **Fix.**
  - New table `iam.sign_in_attempt`:
    - `id`, `occurred_at`;
    - `user_id` (nullable), `email_hmac`;
    - `tenant_id`;
    - `ip inet`, `asn int`, `country text`;
    - `device_id`;
    - `result`;
    - `risk_score smallint`, `risk_reasons text[]`.
    - Retained for 13 months.
  - Rule engine outcomes: allow, step-up, deny, or hold (finding 7).
  - Velocity limits are applied at the money boundary per profile, per tenant and per beneficiary. The money module owns them, but they are listed here as a precondition.
  - Section 19 adds `SecurityIncidentSuspectedV1`, consumed by on-call.
  - New runbook reference: "Partner notice within 24 h of detection (or the contractual figure), NDPC within 72 h, data subjects immediately when the risk is high."

**22. Medium. Sections 26.8 and 26.9 (partners and banks via OAuth, later): the stated design is below FAPI 2.0 and the CBN open-banking consent rules.**

- **Problem.**
  - `client_credentials` with private_key_jwt or mTLS *client authentication* still issues bearer access tokens.
  - There is no sender-constrained token, no PAR, no consent model, and no request or response signing for payment initiation.
  - If partners act for members (data or payment access), CBN open-banking rules need:
    - proof of consent;
    - revocation at any time;
    - re-validation every year or after 180 days unused;
    - a monthly transcript to the customer [S].
- **Reference.**
  - FAPI 2.0 Security Profile (Final, Feb 2025): https://openid.net/specs/fapi-security-profile-2_0-final.html
  - FAPI 2.0 Message Signing (Final, Sep 2025): https://openid.net/specs/fapi-message-signing-2_0-final.html
  - RFC 9700 (OAuth 2.0 Security BCP, Jan 2025): https://www.rfc-editor.org/rfc/rfc9700
  - RFC 9449 (DPoP): https://www.rfc-editor.org/rfc/rfc9449
  - RFC 8705 (mTLS): https://www.rfc-editor.org/rfc/rfc8705
  - RFC 9126 (PAR): https://www.rfc-editor.org/rfc/rfc9126
  - CBN Operational Guidelines for Open Banking (2023) [S]: https://www.mondaq.com/nigeria/financial-services/1374830/
- **Fix.**
  - Section 26.9 replacement text: "Partners follow FAPI 2.0 Security Profile. Client authentication is private_key_jwt or mTLS. Access tokens are sender-constrained (mTLS-bound per RFC 8705, or DPoP per RFC 9449), audience-restricted to `colsynq-api`, and live at most 5 minutes; client_credentials gets no refresh token. Any user-delegated flow uses PAR + PKCE S256 + `iss` in the response. Payment-initiation requests and responses are signed per FAPI 2.0 Message Signing (HTTP Message Signatures, RFC 9421) for non-repudiation."
  - New table `iam.oauth_consent`:
    - `id`, `tenant_id`;
    - `user_profile_id` (the member);
    - `client_id`;
    - `scopes text[]`;
    - `data_tier` (CBN tier);
    - `granted_at`, `expires_at` (≤ 1 year), `last_used_at`;
    - `revoked_at`, `revocation_reason`;
    - `evidence_hash`.
  - A job expires consents unused for 180 days.

**23. Medium. Sections 18 and 20 (Idempotency-Key): the semantics are unsafe for money.**

- **Problem.**
  - "Repeated request with a used idempotency key returns the original response." The key is not bound to principal, tenant or request body, and no storage or lifetime is stated.
  - A key reused with a different payload silently returns the old result, and an unscoped key can collide across tenants.
- **Reference.**
  - IETF draft-ietf-httpapi-idempotency-key-header: https://datatracker.ietf.org/doc/draft-ietf-httpapi-idempotency-key-header/
  - Stripe and Paystack practice.
- **Fix.**
  - New table `iam.idempotency_record`, kept in each tenant database for money routes:
    - `tenant_id`, `principal_id`, `route`, `key`, PK over all four;
    - `request_hash bytea`;
    - `status` (`in_progress`, `completed`);
    - `response_status`, `response_body_encrypted`;
    - `created_at`, `expires_at`.
  - Retention: at least 7 days for money routes, 24 h otherwise.
  - Same key with a different `request_hash` → 422 `idempotency_key_reused`. A concurrent duplicate → 409 `request_in_progress`.

**24. Medium. Security assurance (absent from the spec): pen-testing, vulnerability management and secure SDLC.**

- **Problem.**
  - The spec has governance tests for schema invariants only. It does not mention pen-testing, scanning, dependency or secrets scanning, or threat modelling.
  - Bank vendor questionnaires ask for a recent third-party pen-test and ISO 27001 alignment.
  - The CBN framework makes SFIs require ISO 27001, PCI DSS and the NDPR from third parties, and use quarterly scans, annual vulnerability assessments and third-party pen-tests [S].
- **Reference.**
  - Mondaq summary of the CBN 2024 framework [S].
  - PCI DSS v4.0.1 11.3 (quarterly scans) and 11.4 (annual pen-test, and after significant change).
  - ISO A.8.8 (technical vulnerabilities), A.8.25–A.8.29 (secure development and testing).
  - GAID Art. 28 (DPIA mandatory and filed for "financial services involving the processing of personal data through digital devices", vetted by a certified DPO).
- **Fix.** Add section 27, "Security assurance (go-live gates)":
  1. A DPIA covering IAM, payouts and cross-border sub-processors, filed with the NDPC through the CAR process before the first live tenant.
  2. A third-party pen-test of auth, tenancy isolation, webhooks and money flows before go-live, then yearly and after major change. Findings tracked to closure: critical in 15 days, high in 30.
  3. Weekly authenticated external scans; SAST, dependency and secret scanning in CI, blocking on high.
  4. A threat model refreshed per release for money flows.
  5. An ISO 27001 ISMS roadmap, with the Statement of Applicability covering A.5.15–5.18, A.8.2, A.8.5, A.8.15–8.17 and A.8.24.
  6. A yearly incident-response tabletop with a partner bank.

### Low

**25. Low. Section 26.1 and Section 5: PCI DSS applicability is not stated.**

- **Problem.** If any page ever embeds a gateway's card form (iframe or inline JS), PCI SAQ A eligibility applies. Since the January 2025 SAQ A update, this includes confirming the site is not susceptible to script attacks.
- **Fix.**
  - State: "Colsynq stores, processes and transmits no PAN or SAD. Card payments use gateway-hosted pages or iframes only. No PAN may appear in logs, `metadata` or audit payloads (a CI test greps fixtures and log samples for PAN patterns with a Luhn check)."
  - Keep a CSP with `script-src` hashes on pages that embed payment iframes.
  - Reference: https://www.pcisecuritystandards.org/document_library/

**26. Low. Sections 8.6 and 25.24 (`audit_event.ip`, `user_agent`): personal data is kept 7 years.**

- **Fix.** Truncate IPv4 to /24 and IPv6 to /48 in `audit_event` after 13 months. Keep full values in `sign_in_attempt` (13 months). This follows GAID minimisation.

**27. Low. Section 11 (AUDITOR, EXTERNAL_AUDITOR "read and export everything"): bulk export of member personal data is a breach path.**

- **Fix.**
  - `*:export` permissions get `risk = high`, so they need step-up.
  - Exports are asynchronous, encrypted, delivered by a link valid for at most 24 h, watermarked with the requester, and audited with row counts.
  - Exports of more than 10,000 personal records need dual control (finding 18).

**28. Low. Section 12.2 (host and `X-Tenant` resolution), 6.1 (`logo_url`): the trusted-proxy and SSRF boundaries are implicit.**

- **Fix.**
  - Section 12.2 adds: "`framework.trusted_proxies` and `trusted_headers` list only the load balancer. The host is validated against `tenant_domain` and `*.colsynq.app` before use."
  - `logo_url` is uploaded to Colsynq storage, never fetched from a tenant-supplied URL.
  - All server-side fetches go through the egress proxy (finding 11).

**29. Low. Section 11 (PLATFORM_OWNER, platform roles): platform grants are standing.**

- **Fix.**
  - PLATFORM_ADMIN, FINANCE_OPERATIONS and COMPLIANCE_OFFICER grants get `max_grant_duration = 90 days`, and re-grant needs dual control.
  - PLATFORM_OWNER stays standing (two-holder rule), but it is reviewed every quarter (finding 19) and needs aal3 (finding 6).

---

## Consolidated schema delta

| Table | Change |
|---|---|
| app_user | + security_hold_until, security_hold_reason, restricted_at, legal_hold_until |
| user_authenticator | type set {totp, passkey, device_key, transaction_pin, backup_codes, email_otp, sms_otp}; + aal, phishing_resistant, hardware_bound, usable_for_money_after, failed_count, locked_until, enc_key_id |
| refresh_token | + acr, device_id, ip, user_agent, last_activity_at, idle_timeout, absolute_expires_at |
| api_key | + environment, scopes, created_by_profile_id, approved_by_profile_id, pepper_version, rotated_from_id, last_used_ip, auth_method, client_cert_sha256, public_key_jwk; trigger api_key_money_ck |
| user_role | + status, requested_by_profile_id, approved_by_profile_id, approved_at, effective_from, last_reviewed_at, reviewed_by_profile_id, review_due_at; trigger user_role_dual_control_ck |
| support_access_grant | + kind, approved_by_platform_profile_id, incident_ref, review_due_at, reviewed_by_profile_id, reviewed_at |
| audit_event (landlord and each tenant DB) | + chain_id, seq, prev_hash, row_hash, recorded_at |
| tenant | + kyb_status, kyb_verified_at, kyb_verified_by_profile_id, kyb_reference; region CHECK = 'ng'; require_email_verification forced true |
| tenant_database | + kms_key_ref, backup_kms_key_ref, encryption_status, key_rotated_at, hosting_zone |
| new | iam.transaction_authorisation, iam.user_device, iam.dual_control_request, iam.sign_in_attempt, iam.oauth_consent, iam.idempotency_record (tenant DB), tenancy.hosting_zone, tenancy.tenant_integration (or integration.provider_credential), integration.provider_account_link, integration.inbound_webhook |
| new assertions | DualControlAssertion, TransactionAuthorisationAssertion, SecurityHoldAssertion, KybVerifiedAssertion, KycLevelAssertion |
| new governance tests | DualControlTest, TransactionAuthorisationBindingTest (amount or payee change invalidates), WebhookTenantResolutionTest (header ignored), WebhookReplayTest, AuditChainTest (tamper detected), ServiceAccountMoneyScopeTest, SessionTimeoutTest, ResidencyConfigTest |

## Sources

- CBN Risk-Based Cybersecurity Framework for DMBs and PSBs (2024), PDF (403 to automated fetch): https://www.cbn.gov.ng/Out/2024/BSD/CBN%20Risk-Based%20Cybersecurity%20Framework%20for%20DMBs%20and%20PSBs_2024.pdf ; summary: https://www.mondaq.com/nigeria/security/1518574/
- CBN Instant Payment and BVN circulars (12 Mar 2026): https://openafricapod.substack.com/p/cbn-releases-new-circulars-on-instant ; https://globallawexperts.com/cbns-new-instantpayment-security-minimums-live-2/
- CBN circular PSM/DIR/PUB/CIR/01/006 (MFA and indemnity, 2022): https://regalert.today/document/d706c17d-8187-4b16-9a2f-f9c8172a1cd0/text
- CBN data localisation (15 Jun 2026; effective 1 Jan 2027): https://www.mondaq.com/nigeria/financial-services/1813412/ ; https://www.lawyard.org/news/cbn-reaffirms-january-2027-data-localisation-deadline-for-financial-sector/
- CBN Open Banking Operational Guidelines (2023): https://www.mondaq.com/nigeria/financial-services/1374830/ ; Framework circular (2021): https://www.cbn.gov.ng/Out/2021/PSMD/Circular%20on%20the%20Regulatory%20Framework%20on%20Open%20Banking%20in%20Nigeria.pdf
- CBN Baseline Standards for Automated AML Solutions (10 Mar 2026): https://businessday.ng/news/article/banks-fintechs-face-new-cbn-deadline-for-automated-aml-systems/
- NDPC GAID 2025 (Art. 28 DPIA, Art. 33 breach 72 h, Art. 45 and Sched. 5 cross-border, retention guidance): https://ndpc.gov.ng/wp-content/uploads/2025/07/NDP-ACT-GAID-2025-MARCH-20TH.pdf
- NIST SP 800-63B-4: https://pages.nist.gov/800-63-4/sp800-63b/aal/ ; https://pages.nist.gov/800-63-4/sp800-63b/authenticators/
- PCI DSS v4.0.1: https://www.pcisecuritystandards.org/document_library/
- ISO/IEC 27001:2022: https://www.iso.org/standard/27001 ; SOC 2 TSC: https://www.aicpa-cima.com/resources/download/2017-trust-services-criteria-with-revised-points-of-focus-2022
- FAPI 2.0 Security Profile (Final): https://openid.net/specs/fapi-security-profile-2_0-final.html ; FAPI 2.0 Message Signing (Final, Sep 2025): https://openid.net/fapi-2-message-signing-final-specification-approved/
- RFC 8176, 8705, 9126, 9421, 9449, 9470, 9700: https://www.rfc-editor.org/
- PSD2 RTS (EU) 2018/389: https://eur-lex.europa.eu/eli/reg_del/2018/389/oj
- Paystack webhooks: https://paystack.com/docs/payments/webhooks/ ; Monnify webhooks: https://developers.monnify.com/docs/webhooks ; Embedly webhooks: https://developer.embedly.ng/api-reference/webhooks.md ; Standard Webhooks: https://www.standardwebhooks.com/
- NIBSS fraud trends (secondary): https://techcabal.com/2024/05/24/fraud-losses-decline-q1-2024/
