# v4 input: security compliance, redundancy and failure-mode review of IAM/tenancy v3.1

Reviewed 13 October 2026. Reviewer stance: SaaS security architect.

Documents read:

- `data_model/iam_v3_1_snapshot.md` (the design under review, called "v3.1")
- `data_model/tab_authentication.md` ("Auth tab")
- `data_model/tab_multitenancy.md` ("MT tab")
- `data_model/tab_architecture_guide.md` ("Arch guide")
- `data_model/iam_v2_snapshot_r22.md` ("v2", especially section 27)
- the prior reviews `iam_v2_security_review_oauth_jwt.md`, `iam_v2_security_review_banking.md` and `iam_v3_architect_review.md`

Out of scope by instruction: financial controls, the approval workflow, support access, KYB and plans are deferred in v3.1. They are not asked back. Where a deferral leaves a hole, section 1.B proposes a minimal placeholder.

Severity:

- **High**: an exploitable gap, or a mandatory standard missed. Fix in v4.
- **Medium**: missing defence in depth, or a spec gap that will become a bug.
- **Low**: hygiene.

---

## 0. Standards baseline, and what changed in 2025–2026

| Source | Status (Oct 2026) | What matters here | URL |
|---|---|---|---|
| OWASP ASVS 5.0.0 | Released 30 May 2025. About 350 requirements in 17 chapters. New chapters V9 (self-contained tokens) and V10 (OAuth/OIDC). Logging is now V16, configuration and secrets V13 | 6.1.1/6.3.1 (stuffing defences must not allow malicious lockout); 6.3.3 (L2 MFA, L3 hardware phishing-resistant); 6.4.3 (reset must not bypass MFA); 7.3.1/7.3.2 (idle and absolute timeouts); 7.4.2/7.4.3/7.4.5 (session termination); 9.1.2/9.1.3/9.2.2/9.2.3 (alg allow-list, trusted key sources, token type, aud); 13.2.1/13.2.2 (backend least privilege, no static shared credentials); 13.3.1–13.3.4 (vault, HSM at L3, rotation); 16.2.2 (UTC, synced time); 16.2.5 (no sensitive data in logs); 16.3.1/16.3.2 (log authentication and authorisation failures); 16.4.2/16.4.3 (logs immutable and shipped to a separate system) | https://github.com/OWASP/ASVS/tree/master/5.0/en |
| OWASP Multi-Tenant Security Cheat Sheet | Current | Fail closed without tenant context. Treat a client-sent tenant as a selector only. Add a tenant dimension to rate limits and to worker, queue and connection limits. Use per-tenant keys where needed. Revoke sessions and API keys at offboarding. Apply deletion to backups and replicas. Make cross-tenant admin paths explicit and audited. Restore one tenant without overwriting live data. Test cross-tenant denial with the real role and pooler, and test connection-reuse leaks | https://cheatsheetseries.owasp.org/cheatsheets/Multi_Tenant_Security_Cheat_Sheet.html |
| OWASP Password Storage Cheat Sheet | Current | Argon2id minimum m=19 MiB, t=2, p=1 (or the listed equivalents). A pepper is defence in depth and lives in a vault or HSM. bcrypt has a 72-byte limit | https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html |
| OWASP Authorization Cheat Sheet | Current | Deny by default; check every request; fail closed | https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html |
| OWASP Top 10:2025 | Published late 2025 | SSRF folded into A01 Broken Access Control. New A03 Software Supply Chain Failures and A10 Mishandling of Exceptional Conditions (fail-open error paths) | https://owasp.org/Top10/ |
| NIST SP 800-63B-4 | Final, August 2025 | At most 100 consecutive failures per authenticator. Passwords ≥ 15 characters when single-factor, ≥ 8 inside MFA. Salted hash with a suitable scheme; an extra keyed hash with the key in an HSM is recommended. An AAL2 verifier must offer a phishing-resistant option. Syncable passkeys are allowed at AAL2, not AAL3. Reauthentication: AAL1 ≤ 30 days; AAL2 ≤ 24 h and ≤ 1 h idle; AAL3 ≤ 12 h and ≤ 15 min idle | https://pages.nist.gov/800-63-4/sp800-63b.html |
| RFC 9700 (BCP 240) | January 2025 | Refresh tokens for public clients must be sender-constrained or rotated with reuse detection. PKCE everywhere | https://www.rfc-editor.org/rfc/rfc9700 |
| RFC 8725 → draft-ietf-oauth-rfc8725bis-10 | Submitted to IESG, Aug 2026; will obsolete 8725 | `aud` mandatory when one issuer serves several RPs; sanitise `kid`; SSRF-safe `jku`/`x5u`; one algorithm per key; `typ: JWT` is not explicit typing | https://datatracker.ietf.org/doc/draft-ietf-oauth-rfc8725bis/ |
| RFC 9068 | In force | `typ: at+jwt`; required `iss, exp, aud, sub, client_id, iat, jti` | https://www.rfc-editor.org/rfc/rfc9068 |
| RFC 9449 DPoP | In force | The server must keep a `jti` replay window and should use server nonces. The refresh token is bound to the key at the token endpoint | https://www.rfc-editor.org/rfc/rfc9449 |
| RFC 10017 (BCP 212), OAuth 2.0 for Browser-Based Apps | **Published August 2026** (was the draft) | Preference order: BFF > token-mediating backend > browser-only | https://www.rfc-editor.org/rfc/rfc10017 |
| PgBouncer | 1.24.0 (Jan 2025) added per-user and per-database connection limits and turned prepared statements on by default. 1.24.1 fixed CVE-2025-2291 (auth_query ignored `VALID UNTIL`). 1.25.1 (Dec 2025) fixed CVE-2025-12819 (unauthenticated SQL via `search_path` when `auth_user` is set and `auth_query` is not schema-qualified). 1.25.2 (May 2026) fixed CVE-2026-6664 to 6667 (SCRAM parsing crashes, KILL_CLIENT authorisation). 1.26.0 (23 Sep 2026) fixed three more CVEs | **v3.1's "PgBouncer 1.21+" pins versions with known authentication CVEs** | https://www.pgbouncer.org/changelog · https://www.pgbouncer.org/config.html |
| PostgreSQL 16–18 | PG16 changed CREATEROLE: the creator gets ADMIN OPTION but not SET or INHERIT unless `createrole_self_grant` says so. PG17 added `transaction_timeout`. PG18 added OAuth (OAUTHBEARER) authentication, still young | CREATE DATABASE … OWNER needs SET ROLE on the owner; database-level GRANTs and settings are not copied from a template | https://www.postgresql.org/docs/current/sql-createdatabase.html · https://www.postgresql.org/docs/current/runtime-config-client.html |
| AWS SaaS tenant isolation whitepaper and SaaS Lens | Whitepaper marked historical; the Lens is current | Silo isolation through scoped credentials; isolation enforced by a shared mechanism outside developers' code; noisy-neighbour controls | https://docs.aws.amazon.com/whitepapers/latest/saas-tenant-isolation-strategies/silo-isolation.html · https://docs.aws.amazon.com/wellarchitected/latest/saas-lens/saas-lens.html |
| ISO/IEC 27001:2022, 27017:2015 | Voluntary; asked for in bank vendor packs | A.5.15/A.8.2 (access control and privileged access), A.8.15/A.8.16 (logging and monitoring), A.8.24 (cryptography), A.8.13 (backup), A.5.34 (PII). 27017 CLD.9.5.1 (tenant segregation), CLD.12.1.5 (administrator operational security), CLD.12.4.5 (monitoring) | https://www.iso.org/standard/27001 · https://www.bsigroup.com/LocalFiles/EN-AU/ISO-27017-overview.pdf |
| SOC 2 TSC (2017, points of focus 2022) | Voluntary | CC6.1 (logical access, encryption keys), CC6.2/6.3 (provisioning, removal, periodic review), CC6.6 (external threats), CC7.2 (monitoring), A1.2 (backup and recovery) | https://www.aicpa-cima.com/resources/download/2017-trust-services-criteria-with-revised-points-of-focus-2022 |
| PCI DSS v4.0.1 | v4.0 retired 31 Dec 2024. Future-dated requirements binding since **31 March 2025** (8.4.2 MFA for all CDE access; 8.3.4 lockout after ≤ 10 attempts for ≥ 30 min) | Out of scope while cards touch only gateway-hosted pages; used as the bank-questionnaire bar | https://www.pcisecuritystandards.org/document_library/ |
| Nigeria NDPA 2023 + GAID 2025 | GAID issued 20 Mar 2025, **effective 19 Sep 2025**; replaces NDPR 2019 and its framework | DPIA; 72-hour breach notice (s.40); storage limitation; portability; cross-border transfers only through adequacy, SCCs, BCRs or other approved instruments. Data controllers of major importance register and file Compliance Audit Returns by 31 March (Ultra-High and Extra-High tiers) | https://www.mondaq.com/nigeria/privacy-protection/1606106/ · https://www.banwo-ighodalo.com/media-files/2025/04/GAID-Publication.pdf |
| CBN circular PSS/DIR/PUB/CIR/001/004 (15 Jun 2026) | **Data localisation effective 1 Jan 2027** | Payment transaction data generated in Nigeria must be processed and stored in Nigeria. Advisers say to include backups, DR, analytics, logging and vendors. Reaches Colsynq through partner contracts | https://techeconomy.ng/cbn-orders-banks-fintechs-to-store-payment-data-in-nigeria-by-january-2027/ |
| CBN Risk-Based Cybersecurity Framework for DMBs and PSBs | 31 May 2024, effective 1 Jul 2024 | Binds partner banks, which push it to vendors through third-party risk management (Part 2.4), access control and monitoring | https://www.cbn.gov.ng/Out/2024/BSD/CBN%20Risk-Based%20Cybersecurity%20Framework%20for%20DMBs%20and%20PSBs_2024.pdf |

---

## 1. Security compliance gaps in v3.1

v3.1 is a good lean rewrite. It **dropped most of v2 section 27.8** (keys, audit integrity, residency, retention, logs, SSRF, incidents) and the 27.3 rate-limit rules. Those topics were not deferred on purpose; they simply have no home now. Several gaps below come from that loss.

### 1.A Gaps

**G1. High. The tenant runtime role owns its database, so a society's app credential can run DDL and switch off its own safety nets (v3.1 5.2 `create_database … OWNER t_<id>`, 8.5)**

- **Requirement.**
  - ASVS 13.2.2: backend accounts get the fewest privileges needed.
  - ISO 27001 A.8.2: privileged access.
  - The MT tab 9.4 already recommends the "hybrid" pattern.
- **Gap.**
  - `t_<id>` is a LOGIN role, owns the database and, through migrations, every table.
  - An owner can `DROP` tables, `ALTER TABLE … DISABLE TRIGGER` (which defeats the constraint triggers in section 3), `TRUNCATE` an audit or ledger table, and `CREATE` objects in `public`.
  - An SQL-injection bug in one society therefore becomes destruction of that society's data, not just reads.
  - The same role runs migrations, so least privilege cannot be applied.
- **Fix.** Use three roles per tenant:
  - `t_<id>_owner` (NOLOGIN) owns the database, schemas and tables.
  - `t_<id>` (LOGIN, runtime) gets only `USAGE` on module schemas and `SELECT, INSERT, UPDATE, DELETE` (no TRUNCATE, no DDL) through `ALTER DEFAULT PRIVILEGES FOR ROLE t_<id>_owner`.
  - Migrations run as a migrator login that may `SET ROLE t_<id>_owner`, over a direct connection (see R3).
  - Also: `REVOKE CREATE ON SCHEMA public FROM PUBLIC`, `REVOKE TEMPORARY ON DATABASE … FROM PUBLIC`, and append-only tables get `REVOKE UPDATE, DELETE`.
  - Add a test: the runtime role cannot `DROP`, `TRUNCATE` or `ALTER`.

**G2. High. The provisioner login is a master key to every society (v3.1 5.2 Rules, go-live)**

- **Requirement.**
  - ASVS 13.2.1/13.2.2 and 13.3.2.
  - OWASP Multi-Tenant: cross-tenant service identities need their own least-privilege scope.
  - ISO 27017 CLD.12.1.5.
- **Gap.**
  - "Only CREATEDB and CREATEROLE, not superuser" sounds narrow, but it is not. Since PostgreSQL 16, a CREATEROLE user holds ADMIN OPTION on every role it created.
  - It can therefore `ALTER ROLE t_<id> PASSWORD …` or `GRANT t_<id> TO provisioner` for any society and then log in or `SET ROLE` as that society.
  - With a static password, one leak of this credential exposes every tenant.
- **Fix.**
  - Issue the provisioner credential dynamically from the vault (short-lived, for example Vault's database secrets engine).
  - Only the provisioning worker gets it, on its own host or container.
  - `pg_hba.conf` lets it connect only to the `postgres` maintenance database from the worker's address.
  - After `CREATE DATABASE`, revoke any membership it took (see R2).
  - Log role DDL with pgaudit (`ROLE`, `DDL` classes) to the external log store, and alert on `ALTER ROLE` or `GRANT` outside a running `tenant_job`.
  - Per-tenant role passwords are written by the worker and never read back by it.

**G3. High. The landlord is a pooled multi-tenant database with no database-level row checks (v3.1 4.2, 4.3)**

- **Requirement.**
  - OWASP Multi-Tenant: application filters "do not replace database-level boundaries" in a pooled store; scope tenant context to the transaction; fail closed.
  - ASVS 8.2.x (data-specific authorisation).
  - AWS isolation guidance: isolation applied by a shared mechanism, not by each developer.
- **Gap.**
  - `profile`, `role`, `role_assignment`, `role_assignment_rule`, `invitation`, `access_scope`, `api_key` and `audit_event` all hold rows for every society in one database.
  - Society administration endpoints (`/profiles`, `/audit-events`, `/api-keys`) rely only on a `WHERE tenant_id = …` in PHP. One missed filter leaks another society's member list or audit trail.
  - The isolation story covers the data plane only. The control plane holds the PII (emails, names, phones).
- **Fix.**
  - Enable RLS (`ENABLE` and `FORCE`) on every landlord table with a `tenant_id`. Policy: `tenant_id = current_setting('app.tenant_id', true)::uuid`.
  - Set the value with `SET LOCAL` (or `set_config(…, true)`) at the start of each transaction from `TenantContext`. This is safe under PgBouncer transaction pooling.
  - Account endpoints (`/me`, Tenant None) and Platform routes use a separate landlord role with explicit, narrow policies, for example `user_id = app.user_id`. No request role has BYPASSRLS. Only the catalogue sync and the migrator do.
  - Add a CI test that fails when a landlord table with `tenant_id` lacks a policy.
  - The `tenant_database` "only the resolver and worker are granted it" rule needs a **different database login** to mean anything. The resolver runs in the same PHP process as the web app. Either expose a `SECURITY DEFINER` function that returns one row by tenant id, or accept that the grant is documentation only.

**G4. High. PgBouncer `auth_query` set-up is unspecified and pinned to vulnerable versions (v3.1 8.7, go-live "≥ 1.21")**

- **Requirement.**
  - PgBouncer docs: use a non-superuser calling a `SECURITY DEFINER` function.
  - The CVE fixes in 1.24.1, 1.25.1, 1.25.2 and 1.26.0.
  - ASVS 13.2.x; TLS per ASVS 12.3.x.
- **Gap.**
  - "A restricted function" is not specified, so nothing says what it may return. If it returns any role's SCRAM secret, someone with the `auth_user` credential can log in as `provisioner`, a landlord role or a superuser through PgBouncer.
  - Versions 1.21–1.25.0 are exposed to CVE-2025-12819 (unauthenticated SQL when `auth_user` is set and the query is not schema-qualified) and CVE-2025-2291 (expired passwords accepted).
  - TLS between app, PgBouncer and PostgreSQL is not stated.
- **Fix.**
  - Require PgBouncer **≥ 1.26.0** (or latest).
  - `auth_type = scram-sha-256`.
  - Point `auth_dbname` at a dedicated `pgbouncer_auth` database on each cluster (this also fixes R1).
  - The function:
    - `LANGUAGE sql SECURITY DEFINER SET search_path = pg_catalog, pg_temp`;
    - reads `pg_catalog.pg_authid`;
    - returns rows only where `rolname ~ '^t_[0-9a-f]{32}$' AND rolcanlogin AND NOT rolsuper AND (rolvaliduntil IS NULL OR rolvaliduntil > now())`;
    - `REVOKE EXECUTE … FROM PUBLIC` and `GRANT EXECUTE` only to `pgbouncer_auth`.
  - `auth_query = SELECT usename, passwd FROM pgbouncer.get_auth($1)` (schema-qualified).
  - Landlord and system roles go in a static `auth_file` or are not proxied at all.
  - `client_tls_sslmode = require`, `server_tls_sslmode = verify-full`.
  - `admin_users` and `stats_users` restricted; console not exposed beyond localhost.

**G5. Medium. "Vault" and secret handling are undefined, and tenant DB passwords are static (v3.1 4.1 `secret_ref`, 8.2, 8.8, go-live)**

- **Requirement.**
  - ASVS 13.3.1 (secrets manager), 13.3.2 (least-privilege access to secrets), 13.3.4 and 13.1.4 (rotation schedule).
  - ASVS 13.2.1: prefer short-lived credentials over static passwords.
  - SOC 2 CC6.1.
- **Gap.**
  - No product, authentication method, policy or rotation period is named.
  - How the app authenticates to the vault is open (a static vault token in `.env` would undo everything).
  - The rotation flow has a race (R6).
- **Fix.**
  - Name the vault (HashiCorp Vault/OpenBao, or the cloud secrets manager in a Nigerian region; see G14).
  - Authenticate the app by workload identity (Kubernetes or cloud IAM), not by a static token.
  - Path policies:
    - web and workers: read `tenant-db/*`;
    - provisioner: write `tenant-db/*`;
    - nobody but the signer: `jwt-signing/*`.
  - Rotate tenant DB passwords every 90 days and on any suspicion, using alternating users (R6).
  - List the critical secrets in a register: JWT keys, API-key pepper, TOTP encryption key, provider secrets, provisioner and PgBouncer auth.

**G6. High. JWT key management and claims are incomplete (v3.1 2 "Tokens", 10)**

- **Requirement.**
  - ASVS 9.1.2/9.1.3 (allow-listed algorithms and trusted key sources) and 9.2.2/9.2.3 (type and audience).
  - RFC 9068 required claims; RFC 8725bis.
  - ASVS 13.3.3 (L3: keys never leave the security module).
- **Gap.**
  - v3.1 lists only `sub, sid, ver, amr, acr, auth_time, cnf`. There is no `iss`, `aud`, `exp`, `iat`, `jti`, `client_id`, `typ` or `kid`.
  - There is no JWKS, no rotation period and no compromise runbook. The Auth tab 7.1 and 9 have a partial plan, but on the lcobucci encoder, which ignores `kid`.
  - The platform console and member apps share one audience (prior OAuth review #13 and #25).
- **Fix (owner: Auth tab).**
  - Header `{alg: ES256, typ: at+jwt, kid: <RFC 7638 thumbprint>}`.
  - Claims: `iss, aud (colsynq-api | colsynq-console), sub, client_id, sid, jti, iat, nbf, exp, ver, amr, acr, auth_time, cnf.jkt`.
  - Validate in this order: alg allow-list, typ, kid looked up only in the in-memory key set, signature, iss, aud matches the route family, time with 30 s skew, sid, ver, cnf.
  - Use lexik with the web-token encoder, or a custom `JWSProvider` signing through KMS. Signing happens only at sign-in and refresh, so KMS latency is acceptable.
  - Rotate every 90 days: publish the new public key one token lifetime before using it, and remove the old key after TTL + skew.
  - Emergency rotation drops the old `kid` immediately.
  - Use separate keys for access tokens, the handoff code and signed links.

**G7. Medium. Password hashing parameters and policy are not stated in v3.1 (4.2 `password_hash`)**

- **Requirement.**
  - OWASP Password Storage: Argon2id m ≥ 19 MiB, t ≥ 2, p = 1.
  - NIST 800-63B-4: length rules, breached-password check, ≤ 100 failures, keyed hash recommended.
  - ASVS 6.2.1, 6.2.9, 6.2.12.
- **Gap.**
  - v3.1 is silent. The Auth tab uses the `sodium` defaults, which are libsodium INTERACTIVE (64 MiB, ops 2; above the OWASP minimum) but undocumented.
  - The bcrypt migration path truncates at 72 bytes.
  - Nothing ties hashing cost to capacity (R12).
- **Fix (owner: Auth tab; v4 references it).**
  - State Argon2id `memory_cost = 65536 KiB, time_cost = 2–3, p = 1`, benchmarked to about 150–300 ms per hash on production hardware. Fall back to the OWASP 19 MiB/t=2 point if memory is short.
  - `needsRehash` on login.
  - Passwords:
    - minimum 15 characters if single-factor, 8 when MFA is always on;
    - maximum ≥ 64 (accept 128);
    - NFC normalisation;
    - breached check (HIBP k-anonymity or an offline list) at sign-up, change and reset;
    - context words ("colsynq" and the society name).
  - Optional pepper: HMAC-SHA-256 with a key in the vault or KMS, and `pepper_version` kept with the hash.
  - Hashing runs only after the rate limiter (G9) allows the attempt.

**G8. High. The MFA and passkey policy is too thin, and recovery can bypass MFA (v3.1 4.2 `authenticator`, `tenant_policy.mfa_required`, 11.2 RecentAuthentication, 14 `/auth/password/reset`)**

- **Requirement.**
  - ASVS 6.3.3 (L2 MFA; L3 hardware phishing-resistant).
  - ASVS 6.4.3 (reset does not bypass MFA) and 6.4.4 (re-proof on factor loss).
  - ASVS 7.5.1 (re-authenticate before changing sensitive attributes).
  - NIST: an AAL2 verifier must offer a phishing-resistant option; syncable passkeys are AAL2 only.
  - ISO 27017 CLD.12.1.5 (MFA for administrators).
  - PCI 8.4.2 as the bank bar.
- **Gap.**
  - v3.1 never says who must have MFA. The platform tenant can run on password only, so `platform_admin` (which can create, suspend and archive societies) is single-factor.
  - `society_admin` MFA depends on the society's own flag.
  - Password reset by signed link does not demand the enrolled second factor.
  - Adding or removing authenticators (`/me/authenticators`) has no step-up rule.
  - The "MFA pending" state before tokens are issued is missing (OAuth review #3).
- **Fix.**
  - `tenant_policy.mfa_required` is forced true for the platform tenant.
  - Platform roles need a phishing-resistant passkey (device-bound preferred; AAL3-like sessions, 12 h max and 15 min idle, which v3.1 already has for the console).
  - `society_admin` and any role holding a `risk = high` permission need MFA regardless of the society flag (a minimal placeholder for the deferred financial controls).
  - Sign-in returns `mfa_required` plus a one-time challenge (5 minutes, 5 attempts) before any token is issued.
  - Reset needs the second factor when one is enrolled. Losing every factor goes through a documented recovery (backup codes, or platform-assisted re-proofing with a hold).
  - Factor add or remove needs `RecentAuthentication` at AAL2 plus notices on all channels.
  - Backup codes: 10 single-use codes, hashed, and they never satisfy step-up for `high`.

**G9. High. Lockout design allows targeted lockout and has no per-IP or global throttling (v3.1 4.2 `failed_sign_in_count`, `locked_until`; 9.3 "5 failed → 15 minutes, doubling")**

- **Requirement.**
  - ASVS 6.1.1/6.3.1: document and implement stuffing and brute-force defences **that prevent malicious account lockout**.
  - NIST: ≤ 100 consecutive failures.
  - OWASP API4:2023.
  - OWASP Multi-Tenant: a tenant dimension in limits.
- **Gap.**
  - An attacker who knows a treasurer's email can keep them locked out indefinitely; the lock doubles without a stated cap.
  - Credential stuffing (one password across many accounts) never trips a per-account lock.
  - No limits are stated for sign-up, reset, refresh, step-up, OAuth start or callback, invitation acceptance, or API keys. v2 27.3 had them; v3.1 dropped them.
  - Counters on `user_account` are hot rows (R13).
- **Fix (owner: Auth tab, a rate-limit table).** Use Symfony RateLimiter on Redis, keyed:
  - per (IP, account): 5 per 15 min;
  - per IP: 50 per 15 min;
  - per account: 20 per hour, then **soft lock**. Password sign-in pauses, but a passkey or an emailed unlock link still works, so a victim is never locked out.
  - per tenant host: a global ceiling;
  - per authenticator: 100 consecutive failures, then disabled (NIST).
  - Other endpoints:
    - reset: 3 per email per hour, 20 per IP per hour;
    - refresh: 30 per session per minute;
    - step-up and MFA: 5 per challenge;
    - API key: per key and per tenant.
  - Return 429 with `Retry-After`.
  - Set `trusted_proxies` correctly; responses stay uniform.
  - Alert on stuffing patterns (many accounts from one IP or ASN; a high failure ratio).
  - Remove `failed_sign_in_count` and `locked_until` from `user_account`, or keep only `locked_until` for an admin lock.

**G10. Medium. Session management gaps (v3.1 10)**

- **Requirement.** ASVS 7.1.1, 7.3.1/7.3.2, 7.4.2/7.4.3/7.4.5; NIST reauthentication limits; RFC 9449 (DPoP).
- **Gap.**
  - Timeouts are in line with NIST (web 12 h and 30 min idle = AAL2; console 15 min idle; mobile 30 days with DPoP = AAL1, with `high` actions stepping up to AAL2). Not covered:
    - a society administrator cannot end another profile's sessions in their society (ASVS 7.4.5);
    - after a factor change, there is no "end other sessions" option (7.4.3);
    - there is no cap on concurrent sessions per account;
    - the DPoP `jti` replay window and server nonce are not specified;
    - `ver` is cached for 60 s, so "sign out everywhere" is not immediate (R9).
  - Disabling a profile stops that society only. That is correct, but write it down.
- **Fix.**
  - Add the society-admin action "end this person's sessions in this society". It bumps `access_version` for the profile; the session is global, so revocation is per tenant.
  - A platform action revokes all sessions.
  - Offer "end other sessions" on any password or factor change.
  - Cap sessions at 10 per account, evicting the oldest.
  - DPoP: a `jti` cache for 60 s per key, server nonces on the token endpoint, and `iat` within ±60 s.

**G11. Medium. Token revocation depends on Redis without a failure rule (v3.1 4 "Not stored", 10)**

- **Requirement.** OWASP Top 10:2025 A10 (fail-open error paths); ASVS 7.4.x.
- **Gap.** The revoked-sid list is held only in Redis. If Redis is evicted, flushed or down, revoked sessions work again until their tokens expire. The rate limiter fails the same way (prior OAuth review #34).
- **Fix.**
  - Use a dedicated Redis (TLS, ACL, `maxmemory-policy noeviction`, AOF).
  - `auth_session.revoked_at` stays authoritative and is always checked on refresh.
  - If the revoked-sid lookup errors, privileged routes (console, `high` permissions) return 503 (fail closed). Ordinary reads may run on the 10-minute token lifetime only if this is written down as an accepted risk.

**G12. High. Audit is not tamper-evident, covers no security events, and has no separate copy (v3.1 13)**

- **Requirement.**
  - ASVS 16.3.1 (log every authentication, success and failure), 16.3.2 (authorisation failures), 16.4.2 (logs cannot be changed), 16.4.3 (logically separate system), 16.2.2 (UTC, synced).
  - ISO 27001 A.8.15; SOC 2 CC7.2.
  - OWASP Multi-Tenant: platform-wide audit reads are a separate permission.
  - CBN framework monitoring (bank questionnaires).
- **Gap.**
  - `audit_event` records only state changes. Failed sign-ins, MFA failures, authorisation denials, refresh-reuse detection, rate-limit trips, `TenantConnectionMismatch` and cross-tenant attempts are not logged.
  - "Insert only" binds the app role, but the table owner, the migrator and anyone with DBA access can still edit it, and detaching partitions is unprotected.
  - The v2 hash chain was rightly dropped (v3 review F5 bottleneck), but nothing replaced it.
  - The landlord `audit_event` covers IAM and Tenancy only. Business-module audit has no owner.
- **Fix.**
  - Add a `security_event` stream: a Monolog channel plus an outbox. It is shipped asynchronously to an external, append-only store (object storage with Object Lock/WORM in Nigeria) and is not written to `audit_event`.
  - Write a signed daily digest per partition (SHA-256 over ordered rows, signed by a KMS key) to the same WORM bucket. That gives tamper-evidence without a per-row chain.
  - Revoke `TRUNCATE`, `UPDATE` and `DELETE` from every role except a break-glass owner.
  - Alert on `DETACH PARTITION` outside a scheduled job.
  - `/audit-events` is filtered by RLS (G3).
  - Platform audit reads need a distinct permission and are themselves audited.
  - State NTP and UTC.

**G13. Medium. PII in logs and audit has no rule in v3.1 (lost from v2 27.8)**

- **Requirement.** ASVS 16.2.5; NDPA storage limitation and data minimisation; GAID; ISO 27001 A.5.34.
- **Gap.**
  - `auth_session.ip` and `user_agent` and audit payloads keep personal data indefinitely.
  - Application logs have no rule. The Auth tab 15 and the MT tab 15 each state part of one.
- **Fix (owner: Arch guide, Observability).**
  - Never log passwords, tokens, refresh tokens, API keys, OTPs, DB passwords or the DSN (`#[SensitiveParameter]` on `connect`).
  - Mask emails and phones; record the tenant id and user id only.
  - IPs: truncate after 90 days in application logs; `auth_session.ip` is removed when the session ends plus 13 months.
  - Audit `payload` holds ids and changed field names, not values, for PII fields.
  - Retention table: audit 7 years or the AML minimum; security logs 1–2 years; debug 30 days.

**G14. High. No data-residency rule, although placement supports many clusters (v3.1 5.3 placement; 4.1 `tenant_database.host`)**

- **Requirement.**
  - CBN localisation circular: from 1 Jan 2027, payment transaction data is stored in Nigeria, including backups and DR per advisers.
  - NDPA and GAID cross-border transfer instruments; DPIA; registration as a data controller of major importance and Compliance Audit Returns.
  - Azure tenancy guidance: isolate by data location when needed.
- **Gap.** v3.1 says nothing about where clusters, backups, the vault, Redis, logs, email and error tracking run. Placement "from configuration" could put a society anywhere.
- **Fix (owner: a new Security and compliance tab, or v4 section "Hosting").**
  - Every tenant cluster, the landlord, backups, WORM logs, vault and Redis run in Nigerian regions or data centres; placement configuration accepts only clusters tagged `region=ng`.
  - Keep a sub-processor register (email, SMS, error tracking, analytics). Run a transfer assessment for anything offshore and strip PII before sending (for example Sentry scrubbing).
  - File the DPIA; decide whether Colsynq is a data controller of major importance and calendar the 31 March Compliance Audit Return.

**G15. Medium. Per-tenant backup and restore is only a checklist line (v3.1 15)**

- **Requirement.**
  - OWASP Multi-Tenant: authorise the operator for the target tenant, check ownership, never restore over live data, test with two tenants.
  - ISO 27001 A.8.13; SOC 2 A1.2.
  - MT tab 13 (PITR is whole-server).
- **Gap.**
  - No RPO or RTO, no encryption of dumps, no restore authorisation, no landlord-tenant consistency rule.
  - A society restored to 02:00 while the landlord keeps 14:00 profiles and roles can leave profiles pointing at missing members, or outbox and inbox replays.
- **Fix.**
  - Use cluster PITR (WAL archiving) plus a nightly `pg_dump -Fc` per tenant, encrypted with a per-tenant data key (envelope under KMS) and stored in Nigeria.
  - RPO ≤ 15 min (PITR), RTO per tenant ≤ 4 h.
  - Restore runbook:
    1. Restore into a new database `t_<id>_r<n>`.
    2. Run a validation job.
    3. Swap `tenant_database.database_name` under a `tenant_job(type=restore)` while the tenant is suspended.
    4. Replay or reset the relay cursors.
  - Restores need two platform staff (ticket) and are audited.
  - Run a quarterly drill with two tenants.

**G16. High. Deferred support access leaves platform-staff and engineer access to tenant databases ungoverned (v3.1 6.2 "support access is deferred", 16)**

- **Requirement.**
  - ISO 27001 A.8.2 and A.5.18; ISO 27017 CLD.12.1.5.
  - SOC 2 CC6.1/CC6.3.
  - OWASP Multi-Tenant (explicit, separately authorised, audited cross-tenant paths).
  - CBN framework (privileged access, through partner due diligence).
- **Gap.** In-app support is correctly deferred, but people will still need production database access to debug a society. Today that means the provisioner, a DBA superuser or a vault read of a tenant password, all standing and unlogged.
- **Fix (minimal placeholder, no app feature).**
  - No standing human access to tenant databases.
  - Break-glass goes through vault-issued, time-boxed (≤ 1 h), read-only credentials for one tenant role. They are requested with a ticket and approved by a second person outside the app.
  - pgaudit `READ, WRITE` is on for those sessions.
  - The society's `society_admin` is notified afterwards.
  - Run a quarterly review of who used it.
  - Write this as a one-paragraph rule in v4 section 16.

**G17. Medium. Noisy neighbours and DoS: limits exist only at the pool and statement level (v3.1 5.2, 8.7)**

- **Requirement.** OWASP Multi-Tenant (a tenant dimension for rate limits, workers, queues and connections); OWASP API4:2023; ASVS 13.1.2.
- **Gap.**
  - There is no per-tenant HTTP rate limit, Messenger fairness, per-tenant queue depth, storage quota, PgBouncer `max_client_conn` sizing, or per-cluster migration concurrency.
  - One society's bulk import can starve the shared `integration_events` transport and the relay. Plans are deferred, so nothing sets quotas.
- **Fix.**
  - Default per-tenant quotas in configuration until plans exist:
    - API requests per minute per tenant and per profile;
    - concurrent jobs per tenant in workers (a lock keyed `t:<id>:jobs` with a semaphore);
    - relay round-robin (already in the MT tab);
    - `max_db_connections` per tenant database (already present);
    - `max_user_connections` (PgBouncer 1.24+);
    - `query_wait_timeout` of about 15 s, so one tenant's waiters fail fast;
    - database size alerts from `pg_database_size`;
    - object-storage quota per prefix.
  - Migration rollouts cap concurrency per cluster.

**G18. Medium. Offboarding and export are not specified (v3.1 5.5 `archived`, 16)**

- **Requirement.**
  - OWASP Multi-Tenant: revoke sessions and API keys at offboarding; provide export; apply retention to backups and replicas.
  - NDPA portability and storage limitation; AML retention of at least 5 years.
- **Gap.**
  - "archived: read-only for exports" does not say who may export, in what format, or how credentials are cut.
  - Service-account API keys and open invitations for an archived tenant are not revoked.
  - Read-only is not enforced in the database.
- **Fix.**
  - `TenantArchivedV1` handler in IAM: revoke all API keys and open invitations; disable all profiles except export-capable `society_admin`; bump `access_version`.
  - Tenancy: `ALTER DATABASE t_<id> SET default_transaction_read_only = on` and revoke DML from the runtime role.
  - Export: a `tenant_job(type=export)` builds a signed, encrypted archive (CSV or JSON per module plus a manifest with hashes), behind a 24 h single-use link and step-up AAL2.
  - Deletion after retention stays deferred, but write down that backups expire on the same schedule and that crypto-shredding the tenant key finishes erasure.

**G19. Medium. Signed links (reset, verification, join) are replayable within their lifetime (v3.1 4 "Not stored")**

- **Requirement.** ASVS 6.4.1 (short-lived, single use); NIST (reset secrets single use).
- **Gap.**
  - A link bound only to `token_version` stays valid until expiry or a version bump. A verification or join link can be replayed, and a reset link stays live if it leaks after use when the reset did not bump the version (for example a cancelled reset). No TTL is stated.
- **Fix.**
  - Reset links: 30 min, bound to a fingerprint of the current `password_hash` (Symfony `signature_properties`), so a successful reset kills every reset link.
  - Verification and join links: 24 h, with a `jti` stored in Redis on first use.

**G20. Medium. Service accounts and `high` permissions conflict (v3.1 9.3, 11.2 RecentAuthentication)**

- **Requirement.** ASVS 13.2.x; the prior banking review #5.
- **Gap.**
  - Service accounts have no MFA. If an administrator gives one a role holding a `high` permission, `RecentAuthentication` either always challenges (the integration breaks) or gets bypassed for service accounts (a hole).
  - API keys have no scopes.
- **Fix.**
  - A trigger (or the assignment check in 11.3) refuses any role containing a `risk = high` permission on a `type = service` profile.
  - Add `api_key.scopes text[]`, a subset of the profile's permissions.
  - Make the `ip_allow_list` mandatory for keys whose roles hold write permissions.

**G21. Medium. The bootstrap of the first society administrator trusts one email (v3.1 6.2)**

- **Requirement.** ISO 27001 A.5.16 (identity management); NIST IAL concepts; the prior banking review #15.
- **Gap.**
  - `owner_email` is typed by one platform operator. A typo or a compromised mailbox gives full control of the society for 7 days.
  - KYB is deferred, so nothing checks the person.
- **Fix (placeholder).**
  - A second platform person confirms `owner_email` before `TenantActivatedV1` (a `tenant.owner_confirmed_by` audit entry, not a workflow).
  - Shorten the bootstrap invitation to 72 h.
  - The accepting account's verified email must equal `owner_email`.
  - MFA enrolment is required during acceptance.

**G22. Low. Encryption in transit and at rest is not stated (v3.1 silent)**

- **Requirement.** ASVS 12.x (TLS for all connections, including internal) and 11.x (cryptographic inventory); ISO A.8.24.
- **Gap.** v3.1 is silent on TLS to PostgreSQL, PgBouncer and Redis, disk encryption, and per-tenant data keys (v2 had them).
- **Fix.**
  - TLS `verify-full` on every hop.
  - Encrypted volumes.
  - Per-tenant data keys under KMS for sensitive business fields (BVN, bank details), owned by the business modules but named in v4 so `key_id` conventions line up.

**G23. Low. Supply chain and security assurance (OWASP Top 10:2025 A03, A10)**

- **Gap.** v3.1 has no dependency, SBOM, secret-scanning or pen-test line; v2 27.9 had one.
- **Fix.** Keep a short go-live list:
  - `composer audit` and SBOM in CI;
  - secret scanning for `csk_` keys and PEMs;
  - a pen test covering cross-tenant cases;
  - error paths fail closed (resolver, middleware and voter exceptions return 4xx or 5xx, never "allow").

**G24. Low. No periodic access review**

- **Requirement.** SOC 2 CC6.2/CC6.3; ISO A.5.18.
- **Gap.** Access recertification is not mentioned. It is cheap to add even while approvals are deferred.
- **Fix.** A quarterly report:
  - society administrators: holders of `society_admin` and `high` roles in their society;
  - platform: holders of platform roles.
  - Each report is acknowledged in the audit log.
  - Optional: auto-expire unused assignments after 90 days.

### 1.B Deferrals that leave a hole needing a minimal placeholder

| Deferred item | Hole it leaves | Minimal placeholder for v4 |
|---|---|---|
| Support access and break-glass | Engineers reach tenant DBs through standing credentials | G16 out-of-app JIT access rule |
| Approval workflow and financial controls | A custom role could be given a future money permission the day Payments ships | `iam:sync-catalogue` fails the deploy if any permission has `risk = financial` (or module Payments, Ledger or Integrations with a write action) while the approval module is absent. `society_admin` needs MFA (G8). Service accounts cannot hold `high` (G20) |
| KYB | One unverified email becomes the society owner | Second-person confirmation of `owner_email`; MFA at acceptance (G21) |
| Plans and feature flags | No quotas, so noisy neighbours (G17) | Default per-tenant quotas in configuration |
| Tenant database deletion | Archived tenants keep live credentials | Revoke keys, invitations and profiles on archive; read-only DB (G18) |
| Integrations and webhooks | A webhook route could be added with `Optional` and trust `X-Tenant` | The architecture test refuses any route under `/webhooks/*` until the `Webhook` requirement exists |

---

## 2. Redundancy, bulk and contradictions between the tabs

### 2.1 Conflicts (v3.1 should win unless noted)

| # | Topic | Auth tab | MT tab | Arch guide | v2 | v3.1 | Resolution |
|---|---|---|---|---|---|---|---|
| 1 | Table and entity names | User, UserProfile, UserRole, UserRoleAssignRole, `app_user`, `user_authenticator`, `one_time_token`, `lookup_hash`, `user_profile_id` | `app_user`, `user_profile`, `user_role`, `platform_tenant`, `scope_reference`, `permission_conflict` | UserProfile, `UserProfileCreatedV1` | v2 names | `user_account`, `profile`, `role_assignment`, `ProfileCreatedV1` | Rename everywhere to v3 names; other tabs only link |
| 2 | Web token storage | SPA keeps the access token in memory and the refresh in a SameSite=**Strict** cookie on `/api/token/refresh`; BFF only for the console | — | — | 27.2 BFF, `__Host-`, Lax | BFF for all web, `__Host-`, **Lax** + CSRF | v3.1 (RFC 10017) |
| 3 | Refresh lifetime | 14 days, 30-day cap, 10 per user (gesdinet) | — | — | per role | 12 h web, 30 days mobile (SessionPolicy) | v3.1; Auth tab config must match |
| 4 | Revocation mechanisms | lexik `jti` blocklist + gesdinet `block_jwts_on_revocation` + `token_version` | — | — | three paths | sid list + `token_version` | v3.1; remove the blocklist config from the Auth tab |
| 5 | JWT claims | `iss, aud, jti, typ, ver`; no `sid, amr, acr` | — | — | 14 claims, `kid` | `sub, sid, ver, amr, acr, auth_time, cnf`; no `iss/aud/jti/typ/kid` | Merge per G6; Auth tab owns |
| 6 | Lockout | `login_throttling` 5/15 min + per-user counter, ≤ 100 | — | — | no lockout on the anonymous path | per-account 5 → 15 min doubling | Replace with G9 |
| 7 | Second factors | email OTP, trusted devices, magic links | — | — | removed | totp, passkey, backup codes | v3.1 |
| 8 | First administrators | "created with the tenant (two), set-password link" | `SeedTenant` creates "first officers" | — | encrypted invitation record | one bootstrap invitation | v3.1 |
| 9 | Admin creates a user | `POST /api/profiles`, user without password + set-password link | — | — | profile + notice | invitation only (consent) | v3.1 |
| 10 | Sign-up | creates User "email not yet verified"; role from tenant policy | — | — | pending | email proven first; `member` with scope self | v3.1 |
| 11 | No profile in tenant | **404** | 404 | 404 | 404 | **403** | Pick one. 404 avoids revealing membership, but v3.1 says 403 "with no hint". State it once in v4 |
| 12 | Support access | live SupportAccessGrant | `support_access_grant` table | — | 27.5 | deferred | v3.1 + G16 |
| 13 | Tenant registry | — | `tenant` with `db_*` columns, 9 statuses, `region`, `plan`, citext | — | 7 statuses | `tenant` + `tenant_database`, 5 statuses, no citext | v3.1; MT tab section 4 deleted, replaced by a link |
| 14 | Provisioning | — | message chain, template clone, `OWNER app_owner` (hybrid), `CONNECTION LIMIT 20`, compensation drops the DB, 5 retries | — | — | `tenant_job` steps, `template0`, `OWNER t_<id>`, 3 retries, drop only if never active | v3.1 owns the state machine; MT owns the SQL, and the **hybrid role pattern should win** (G1) |
| 15 | Fleet migrations | `tenants:migrate`, `tenant_migration_run`, **direct DSN**, `lock_timeout`, waves 0 canary … 3 last, default 2 | — | — | — | `tenant:migrate`, `tenant_job(migrate)`, **through TenantScopedRunner (PgBouncer)**, waves 0 internal … 3 everyone | Command name and wave meanings must match. Direct DSN wins (R3); `tenant_job` wins over `tenant_migration_run` |
| 16 | Switching code | `TenantContext::switchTo()` pushes params into the middleware; resolver switches per request | — | middleware + reset | — | resolver sets context once; middleware pulls from the resolver; `TenantBoundConnection`; `TenantScopedRunner` | v3.1 (stronger: mismatch check); MT section 6 code replaced |
| 17 | TenantRequirement | Required/Optional/None, **bypass list**, default Required; 503/404 | — | Required/Optional/None | 4 values | + Platform; no bypass; 400/404/423 | v3.1 |
| 18 | Resolver cache | 60–300 s with tags | — | — | — | 60 s, negative caching | v3.1 |
| 19 | Doctrine second-level cache on tenant EM | **disabled** ("keys ignore the database: would leak") | — | — | — | "regions use a tenant-specific namespace" | MT wins (safer). v3.1 8.6 must say disabled |
| 20 | Timeouts | — | `ALTER DATABASE … SET statement_timeout, lock_timeout` | — | — | `ALTER ROLE … SET statement_timeout, idle_in_transaction_session_timeout` | Role-level is fine; add `lock_timeout`; the migrator role gets its own limits (R3, R22) |
| 21 | Tenant DB password storage | — | "encrypted (sodium) in landlord **or** vault" | — | vault | vault only | v3.1 |
| 22 | Postgres roles list | — | provisioner, `app_migrator`, per-tenant runtime, `app_landlord` | — | — | provisioner + `t_<id>` | Merge into one roles table in the MT tab (G1, G2, G3) |
| 23 | Offboarding | — | DROP DATABASE after grace | — | retention + crypto-shred | archived, deletion deferred | v3.1 + G18 |
| 24 | Landlord contents | — | plans, features, subscriptions, `permission_conflict`, `platform_tenant`, `support_access_grant`, `one_time_token` | 2.1 lists plans and features | 30 tables | 20 tables | v3.1; MT section 3 and Arch 2.1 cut down to "see v4 section 4" |
| 25 | API key | SHA-256 `lookup_hash`, `csk_live_` format, `user_profile_id` | — | — | HMAC + pepper, scopes | HMAC + `pepper_version`, `profile_id`, no scopes | v3.1 + scopes (G20); the Auth tab keeps the key format |
| 26 | External identity | `provider`, `email_verified`, `unlinked_at` | — | — | issuer | `issuer`, `subject` | v3.1 |
| 27 | Refresh-token library | gesdinet 3.0 full config | — | — | — | own `refresh_token` table (open item) | Decide in v4; the Auth tab keeps whichever config wins |
| 28 | Messenger tenant middleware | — | middleware order list | `TenantContextMiddleware` on every bus | — | `TenantStamp` dispatch middleware + worker middleware through the runner | v3.1 names; Arch 17 config updated |
| 29 | PgBouncer sizing | — | `default_pool_size` 2–5, wildcard `[databases] *`, one per server | — | — | 5, auth_query, ≥ 1.21 | MT owns; version ≥ 1.26 (G4) |
| 30 | MFA policy | MFA for administrators, finance roles and staff; passkeys for staff | — | — | 27.3 | `tenant_policy.mfa_required` + `high` needs AAL2 | v3.1 + G8 |
| 31 | Keep `society_id` on core rows "for later pooling" | — | yes | — | — | — | Drop, or move to an ADR; it conflicts with "names never from the slug" only by naming. Low |

### 2.2 Pure duplication (same fact, stated in several places)

- **Tenant resolver** (priority 20, sources, requirement values, mismatch → 400): v3.1 7, MT 7, Arch 2.1 and 14, Auth 7.4 and 4. Keep it once in v4 section 7.
- **Connection switching and fail-closed middleware**: v3.1 8, MT 6, Arch 12 and 14.
- **Cache, lock and file prefixing by tenant**: v3.1 8.6, MT 8, Arch 14 and 22.
- **Provisioning steps**: v3.1 5, MT 9.
- **Migrations and expand/contract**: v3.1 5.4, MT 10.
- **Revocation table**: Auth 9, v3.1 10.
- **Account processes** (sign-up, join, reset, close): Auth 12, v3.1 9.
- **Data-model column lists**: Auth 5.1 (User columns), MT 3 (landlord list), Arch 2.1, v3.1 4.
- **Threats and controls**: Auth 16, v2 27, prior reviews.
- **Security of Postgres roles**: MT 9.3, 9.4 and 15; v3.1 5.2.

### 2.3 Topic ownership map (single source of truth for v4)

| Topic | Owner | Others do |
|---|---|---|
| Module rules, events, outbox and inbox, layers, error mapping, CI boundaries, test levels | **Arch guide** | link |
| Logging and observability policy, including the PII-in-logs rules (G13) | **Arch guide, section 18** | link |
| Data model of the landlord (tenancy and iam tables, columns, constraints), tenant lifecycle states and events, provisioning **state machine** and job semantics, bootstrap, tenant resolution rules, access control (roles, permissions, voters), session **policy values**, audit table | **IAM/Tenancy v4** | link; never list columns |
| DB-per-tenant mechanics: rationale, Doctrine connections and middleware code, **PostgreSQL roles and grants** (G1–G3), provisioning **SQL**, PgBouncer config and auth_query (G4), fleet migrations, templates, relay and messaging across DBs, backup, restore and move (G15), noisy-neighbour infra limits (G17), offboarding runbook (G18) | **MT tab** | v4 references the section numbers |
| Credential and token mechanics: packages and config, password hashing params and policy (G7), JWT format, keys, JWKS and rotation (G6), refresh, BFF and DPoP details, MFA and passkey ceremonies and recovery (G8), OAuth/OIDC, API key format, **rate-limit table** (G9), threat table | **Auth tab** (rewritten with v3 names) | v4 states only decisions and links |
| Secrets and vault, KMS and key inventory, residency, retention and erasure, incident notification, platform-staff and engineer access (G16), compliance mapping (ASVS level, ISO, SOC 2, NDPA/GAID, CBN), go-live security checklist | **New "Security and compliance" tab** (the old v2 27.8 content, cut down) | everyone links |
| v2 snapshot r22 | **Archive** (historical; not referenced by v4 except the change history) | — |

Suggested v4 rule: a tab may restate a decision in one line only if it links to the owning section.

---

## 3. Potential runtime errors and failure modes

| # | Where | What fails | Fix |
|---|---|---|---|
| R1 | 8.7 PgBouncer + 5.2 `REVOKE CONNECT … FROM PUBLIC` | `auth_query` runs **inside the target database** by default. `auth_user` has no CONNECT on any tenant database, so every new society connection fails authentication. Installing the function in each tenant DB instead means a migration per DB and a role with CONNECT everywhere | `auth_dbname = pgbouncer_auth` (one auth database per cluster) with the hardened function from G4 |
| R2 | 5.2 `CREATE DATABASE … OWNER t_<id>` | On PG16+ a non-superuser can set OWNER only if it can `SET ROLE` to that owner. The provisioner created the role (ADMIN OPTION) but has no SET membership unless `createrole_self_grant` is set, so it fails with "must be able to SET ROLE" | `GRANT t_<id>_owner TO provisioner WITH SET TRUE, INHERIT FALSE` just before, and `REVOKE` straight after (G2). With the hybrid roles the owner is a NOLOGIN role |
| R3 | 5.2 `migrate` and 8.5 through the runner | Migrations go through PgBouncer in transaction pooling as a role with `statement_timeout = 30s`. Long `ALTER`s and backfills are killed. `CREATE INDEX CONCURRENTLY` and session-level `SET` behave unpredictably across pooled server connections. The MT tab says direct DSN | Migrations use a **direct** connection (port 5432, not PgBouncer) as the migrator, with `SET lock_timeout = '5s'` and `statement_timeout = '15min'`. Concurrent-index migrations are non-transactional and isolated (MT 10.4) |
| R4 | 4.3 `audit_event` monthly partitions | If next month's partition is missing, every IAM write fails (audit is in the same transaction): sign-in, sign-up and role changes all return 500 at midnight on the 1st | Create partitions 3 months ahead (pg_partman or a scheduled job), add a DEFAULT partition as a safety net, and alert if fewer than 2 future partitions exist |
| R5 | 4.1 `tenant_job` partial unique index as the lock | A worker crash (OOM, deploy) leaves the job `running` for ever. The partial index then blocks retries and migrations for that tenant | Add `lease_until` and `heartbeat_at`. A reaper marks expired leases `failed`, and retry allows that. Workers renew the lease per step |
| R6 | 8.8 credential rotation | Neither order of "ALTER ROLE then vault" or "vault then ALTER ROLE" is safe. Processes with a 10-minute in-memory cache, or the other order, fail new connections until they re-read. Many workers re-reading at once stampede the vault | Use two login roles per tenant (`t_<id>_a`, `t_<id>_b`) and rotate the idle one, then flip `secret_ref`. Or set the new password, keep the old valid through `VALID UNTIL` overlap (PgBouncer ≥ 1.24.1 respects it), and re-read with jitter |
| R7 | 4.2 `refresh_token` 30 s grace | Two parallel refreshes (two tabs, a mobile retry) inside the grace window mint two successors, giving two live chains, or one gets a false "reuse". There is no `replaced_by` column, so the server cannot return the same successor | Add `replaced_by`. Lock the row `FOR UPDATE`; inside the grace window, for the same session and `dpop_jkt`, return the already-issued successor (store an encrypted copy for 30 s, or re-derive). Outside it, revoke |
| R8 | Redis outage | Revoked-sid checks, rate limits, the OAuth transaction cookie store and the DPoP `jti` cache all depend on Redis. Default Symfony behaviour throws 500s everywhere, or someone "fixes" it to fail open | Fail-closed rule (G11); the rate limiter degrades to local APCu per node; health check and alerts |
| R9 | 10 `ver` cached 60 s | "Sign out everywhere" and a password change are not immediate: up to 60 s of continued access, despite 4.2 saying "at once" | On a `token_version` bump, also push every active sid to the revoked list in the same operation, or delete the cache key through an event |
| R10 | 4.2 `tenant_policy.access_version` | Every role or assignment change in a tenant updates one row. A bulk import of 2,000 members serialises on that row lock, and the access cache for the whole tenant is invalidated on every change (cache stampede on large societies) | Bump once per command or batch, not per row; warm the cache lazily; consider `(profile_id, access_version)` plus a per-profile version for profile-only changes |
| R11 | Sign-in Argon2id cost | 64 MiB per hash × concurrent sign-ins. libsodium memory is not under PHP `memory_limit`, so a login flood can OOM-kill FrankenPHP/FPM containers, which is a cheap DoS | Rate-limit before hashing; cap concurrent hash operations per node (a semaphore); size the parameters to the container (G7) |
| R12 | 4.2 `failed_sign_in_count` on `user_account` | A stuffing burst writes to the same rows; lost updates between parallel requests undercount failures; row locks slow legitimate sign-ins | Move counters to Redis (G9) |
| R13 | 7.3 negative cache | A cached "miss" for a slug lasts 60 s. Activating a tenant (or adding a domain) right after someone probed the slug returns 404 for a minute. The owner's first click on the bootstrap link can fail | Status, slug and domain changes delete `tenant:slug:*` and `tenant:host:*` **including miss entries**; activation does too |
| R14 | 4.1 slug "unique while not archived" | An archived tenant keeps its slug, which can be reused. A request with that slug then matches two rows, and exports of the archived tenant cannot be resolved by slug | The resolver ignores archived tenants; archived exports go by tenant id on Platform routes only |
| R15 | 5.2 `create_database` idempotency | The step is "done" when the DB exists. A crash after `CREATE DATABASE` but before `REVOKE CONNECT … FROM PUBLIC`/`GRANT` means the retry skips the lock-down. The database stays open to every role with login | Split it into a `lock_down` step that always re-runs its idempotent statements and is verified by checking `pg_database.datacl` |
| R16 | 5.2 `create_role` + vault | If the vault write succeeds and the job crashes before `CREATE ROLE`, a retry that generates a new password diverges from the vault | The rule: generate the password once and store it with the job id, and the retry re-uses the vault value. Or `ALTER ROLE … PASSWORD` from the vault on every retry |
| R17 | 6.2 events | IAM handles `TenantActivatedV1` (create the bootstrap invitation) but may not yet have processed `TenantCreatedV1` (create `tenant_policy`) if the relay or retries reorder them. The invitation then fails its foreign key or policy lookup | Upsert `tenant_policy` in both handlers, or park out-of-order events by `aggregateVersion` (Arch 7.3) |
| R18 | 8.4 "a tenant message without a stamp is rejected" | Landlord-only IAM and Tenancy messages legitimately have no tenant. Without a marker, the worker middleware sends them to the failure transport | A `TenantScopedMessage` marker interface; only those messages need the stamp |
| R19 | 8.6 Doctrine second-level cache "namespaced" | Region keys do not include the database by default. If a namespace is set at boot rather than per tenant, entities leak across societies in worker mode | Disable the second-level cache on the tenant EM (MT 5) |
| R20 | Role `statement_timeout = 30s` for all runtime work | Exports, relay catch-up and nightly jobs running as the runtime role hit 30 s and fail repeatedly | Long jobs use `SET LOCAL statement_timeout` within their transaction, or a separate job role |
| R21 | 6.2 bootstrap acceptance | Nothing checks that the accepting account's email equals `owner_email`. A forwarded link lets anyone become `society_admin` | Require a match, as for every invitation (consent), and MFA enrolment (G21) |
| R22 | 7.1 resolver before firewall | CORS preflight (`OPTIONS`), error sub-requests and `/_profiler` would trip the Required check. The MT tab code skips `OPTIONS` and sub-requests; v3.1 does not say so | State it: main requests only, skip `OPTIONS` |
| R23 | 8.7 PgBouncer upgrade | Moving from 1.21 to 1.24+ turns prepared statements on by default and changes `max_prepared_statements` semantics; per-server LRU memory grows with the tenant count × pool | Set `max_prepared_statements` explicitly (for example 100) and watch PgBouncer and PostgreSQL memory, or enable emulated prepares on the tenant connection |
| R24 | 5.2 statement limits vs `CONNECTION LIMIT` | v3.1 sets no `CONNECTION LIMIT` on the tenant role (the MT tab sets 20). PgBouncer `max_db_connections` covers pooled traffic, but direct connections (migrations, break-glass) are unbounded | `ALTER ROLE t_<id> CONNECTION LIMIT 20`; the migrator has its own limit |

---

## 4. Sources

Standards and guidance

- OWASP ASVS 5.0 sources: https://github.com/OWASP/ASVS/tree/master/5.0/en (V6 `0x15-V6-Authentication.md`, V7 `0x16-V7-Session-Management.md`, V9 `0x18-V9-Self-contained-Tokens.md`, V13 `0x22-V13-Configuration.md`, V16 `0x25-V16-Security-Logging-and-Error-Handling.md`)
- ASVS 5.0 release note: https://assuranceinaction.intertek.com/post/102kghm/owasp-release-version-5-0-of-the-application-security-verification-standard-asvs
- OWASP Multi-Tenant Security Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Multi_Tenant_Security_Cheat_Sheet.html
- OWASP Password Storage Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html
- OWASP Authorization Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html
- OWASP Session Management Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html
- OWASP Top 10:2025: https://owasp.org/Top10/ (summary: https://www.fastly.com/blog/new-2025-owasp-top-10-list-what-changed-what-you-need-to-know)
- OWASP API Security Top 10 2023: https://owasp.org/API-Security/editions/2023/en/0x11-t10/
- NIST SP 800-63B-4: https://pages.nist.gov/800-63-4/sp800-63b.html
- RFC 9700: https://www.rfc-editor.org/rfc/rfc9700 · RFC 8725: https://www.rfc-editor.org/rfc/rfc8725 · draft-ietf-oauth-rfc8725bis: https://datatracker.ietf.org/doc/draft-ietf-oauth-rfc8725bis/ · RFC 9068: https://www.rfc-editor.org/rfc/rfc9068 · RFC 9449: https://www.rfc-editor.org/rfc/rfc9449 · RFC 10017: https://www.rfc-editor.org/rfc/rfc10017 · RFC 7638: https://www.rfc-editor.org/rfc/rfc7638 · RFC 7009: https://www.rfc-editor.org/rfc/rfc7009
- AWS SaaS Tenant Isolation Strategies: https://docs.aws.amazon.com/whitepapers/latest/saas-tenant-isolation-strategies/silo-isolation.html · pool isolation: https://docs.aws.amazon.com/whitepapers/latest/saas-tenant-isolation-strategies/implementing-pool-isolation.html · SaaS Lens: https://docs.aws.amazon.com/wellarchitected/latest/saas-lens/saas-lens.html
- Azure tenancy models: https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/considerations/tenancy-models
- ISO/IEC 27001:2022: https://www.iso.org/standard/27001 · ISO 27017 overview (BSI): https://www.bsigroup.com/LocalFiles/EN-AU/ISO-27017-overview.pdf
- SOC 2 TSC: https://www.aicpa-cima.com/resources/download/2017-trust-services-criteria-with-revised-points-of-focus-2022
- PCI DSS document library: https://www.pcisecuritystandards.org/document_library/ · 4.0 password and MFA summary: https://www.hypr.com/pci-dss-4-password-mfa-requirements

PostgreSQL and PgBouncer

- PgBouncer config: https://www.pgbouncer.org/config.html · changelog (CVE-2025-2291, CVE-2025-12819, CVE-2026-6664..6667, 1.26.0): https://www.pgbouncer.org/changelog
- auth_query with SECURITY DEFINER (EDB): https://www.enterprisedb.com/postgres-tutorials/pgbouncer-authquery-and-authuser-pro-tips · https://www.enterprisedb.com/blog/pg-phriday-securing-pgbouncer
- SCRAM with PgBouncer (Crunchy): https://www.crunchydata.com/blog/pgbouncer-scram-authentication-postgresql
- PostgreSQL CREATE DATABASE: https://www.postgresql.org/docs/current/sql-createdatabase.html · client settings (`createrole_self_grant`, `transaction_timeout`, `idle_session_timeout`): https://www.postgresql.org/docs/current/runtime-config-client.html · RLS: https://www.postgresql.org/docs/current/ddl-rowsecurity.html
- PostgreSQL 18 OAuth: https://www.cybertec-postgresql.com/en/oauth-2-0-in-postgresql-18/
- libsodium Argon2id presets: https://doc.libsodium.org/password_hashing/default_phf

Nigeria

- GAID 2025 summaries: https://www.mondaq.com/nigeria/privacy-protection/1606106/the-nigeria-data-protection-commission-issues-the-general-application-and-implementation-directive-2025-gaid · https://www.afriwise.com/blog/highlights-of-the-nigeria-data-protection-act-general-application-and-implementation-directive · GAID text (Banwo & Ighodalo copy): https://www.banwo-ighodalo.com/media-files/2025/04/GAID-Publication.pdf
- CBN data localisation circular (15 Jun 2026) reporting: https://techeconomy.ng/cbn-orders-banks-fintechs-to-store-payment-data-in-nigeria-by-january-2027/ · https://businessday.ng/technology/article/cbn-ends-offshore-payment-processing-as-banks-fintechs-face-2027-data-localisation-deadline/ · https://youverify.co/en/blogs/cbn-payment-data-storage-regulations-data-localisation-nigeria
- CBN Risk-Based Cybersecurity Framework 2024: https://www.cbn.gov.ng/Out/2024/BSD/CBN%20Risk-Based%20Cybersecurity%20Framework%20for%20DMBs%20and%20PSBs_2024.pdf · control mapping (secondary): https://opensecurityarchitecture.org/frameworks/cbn-csf

Caveats: ASVS requirement numbers come from the 5.0 sources on GitHub (master branch). CBN circular texts were not fetched directly (cbn.gov.ng blocks automated fetches), so those points rest on reputable press and law-firm summaries; confirm them before quoting to a bank. The Argon2id INTERACTIVE ops limit (2) is from libsodium headers, not the docs page.
