# IAM v2: security review of JWT, refresh tokens, API keys, OAuth2/OIDC and sessions

Reviewed 2026-10-13 against `iam_v2_snapshot.md`, mainly sections 7.3, 12.2, 13, 14, 16, 18, 20, 25.8–25.14, 25.23 and 26. The earlier notes in `../auth/` (jwt_symfony.md, oauth2_oidc_symfony.md, user_entity_symfony.md) were also consulted.

## Standards status checked for this review (October 2026)

| Standard | Status | URL |
|---|---|---|
| RFC 8725 JWT BCP | In force. **draft-ietf-oauth-rfc8725bis-10** (2026-08-21) has been submitted to the IESG and will obsolete it. Main changes: `aud` MUST be present and validated when one issuer serves several RPs; `sub` must map to a valid subject; `kid` must be sanitised; `jku`/`x5u` fetches need SSRF protection; one algorithm per key; `typ` "JWT" does not count as explicit typing | https://datatracker.ietf.org/doc/draft-ietf-oauth-rfc8725bis/ |
| RFC 9068 JWT access tokens | `typ: at+jwt`. Required claims: `iss, exp, aud, sub, client_id, iat, jti`. Optional: `auth_time, acr, amr` | https://www.rfc-editor.org/rfc/rfc9068 |
| OAuth 2.1 | draft-ietf-oauth-v2-1-16 (2026-09-03); IESG submission planned for Dec 2026 | https://datatracker.ietf.org/doc/draft-ietf-oauth-v2-1/ |
| RFC 9700 OAuth Security BCP | BCP 240, Jan 2025 | https://www.rfc-editor.org/rfc/rfc9700 |
| RFC 10017 Browser-based apps | BCP, Aug 2026 (BFF > token-mediating backend > browser-only) | https://www.rfc-editor.org/rfc/rfc10017 |
| NIST SP 800-63B-4 (final) | AAL1 overall reauth SHOULD be ≤ 30 days. AAL2 overall SHOULD be ≤ 24 h, with inactivity SHOULD ≤ 1 h. AAL3 overall SHALL be ≤ 12 h, with inactivity SHOULD ≤ 15 min. "Email SHALL NOT be used for out-of-band authentication". Session secrets SHOULD NOT be in localStorage. Bearer session secrets SHOULD NOT survive an app restart (PoP secrets may). §5.1.2: an access token is not proof that the subscriber is present | https://pages.nist.gov/800-63-4/sp800-63b.html |
| OWASP ASVS 5.0 | V6, V7, V8, V9 and V10 requirements below were read from the 5.0 sources | https://github.com/OWASP/ASVS/tree/master/5.0/en |
| OWASP API Security Top 10 2023 | API1 BOLA, API2 Broken Auth, API4 Resource consumption, API5 BFLA, API7 SSRF, API8 Misconfig | https://owasp.org/API-Security/editions/2023/en/0x11-t10/ |

Severity scale:
- **critical**: exploitable account takeover or MFA bypass as designed.
- **high**: a likely exploit, or a mandatory standard is violated.
- **medium**: defence in depth missing, or a spec gap that will produce a bug.
- **low**: hygiene or documentation.

---

## Findings

### 1. critical: Social-login and account-link callbacks are not bound to the browser (login CSRF, then account takeover through linking)
- **Section:** 26.7 (Start/Callback rows), 18 (`/api/me/external-identities`).
- **Problem:** `state` and the PKCE verifier are stored in Redis keyed only by `state`. Nothing ties them to the user agent that started the flow. An attacker can start a flow, sign in at Google with an account they control, stop before the callback, and send the victim the callback URL `…/callback?code=X&state=Y`.
  - **Sign-in variant:** the victim is logged into the attacker's account (login CSRF), and money or data they then enter goes to the attacker.
  - **Linking variant** (`POST /api/me/external-identities` while signed in): the attacker's Google subject is linked to the victim's Colsynq user. The attacker can then sign in as the victim forever. That is a full account takeover. Step-up does not help, because the victim performs it.
  - PKCE does not prevent this either, because the server holds the verifier and redeems the code itself.
- **Reference:** ASVS 5.0 10.1.2 (state, nonce and code_verifier "securely bound to both the client and the user agent session"). RFC 9700 §4.7 (CSRF) https://www.rfc-editor.org/rfc/rfc9700#section-4.7. RFC 6749 §10.12.
- **Fix:**
  - At `/start`, generate `tx_id` (128-bit) and set the cookie `__Host-oauth_tx=<tx_id>; Secure; HttpOnly; Path=/; SameSite=Lax`. Use `SameSite=None` for providers using `response_mode=form_post` (Apple), because a Lax cookie is not sent on a cross-site POST.
  - Redis key `oauth_tx:{tx_id}` (TTL 10 min) holds: `state`, `code_verifier`, `nonce`, `provider`, `purpose` (`login|link|signup`), `tenant_id`, `user_id` (for `link`, the signed-in user), `return_to` (validated against the tenant's hosts), `created_at`.
  - On callback, require the cookie, `GETDEL` the record, compare `state` in constant time, and check that the callback's provider equals `provider`.
  - For `purpose=link`, require an authenticated session whose `sub` equals `user_id` and whose `auth_time` is within 5 min.
  - Replace the 26.7 Start row with: "state, nonce, PKCE verifier, provider, purpose and tenant saved in Redis under a random transaction id that is also set as a `__Host-` cookie; the callback is refused unless the cookie, state and provider all match (single use)."

### 2. critical: Refresh does not carry `amr`/`auth_time`, so MFA and step-up are either lost or laundered
- **Section:** 7.3 refresh_token, 25.12, 26.2 (`amr, auth_time`), 26.3 Refresh row, 14 (StepUpAssertion).
- **Problem:** gesdinet re-issues the JWT through lexik's `JWT_CREATED` with only the User object. The refresh_token row has no `amr`, `auth_time` or `acr`. The implementer must choose one of two outcomes:
  - (a) recompute them: `auth_time = now` and `amr` = whatever the listener guesses. This is a **step-up bypass**: any refresh makes the session look freshly MFA-authenticated, so StepUpAssertion's "recent authentication" check is defeated for 30 days.
  - (b) drop them, which forces MFA every 10 minutes.
  - The spec does not say which, and (a) is the natural bug.
- **Reference:** RFC 9068 §2.2.1 (auth_time/acr/amr describe the *original authentication event*). OIDC Core §2 (`auth_time` = time of end-user authentication). ASVS 10.3.4 and 6.8.4. NIST 800-63B-4 §5.1 (a session SHALL NOT be at a higher AAL than its authentication event).
- **Fix:**
  - Extend the refresh-token entity with a custom class: `session_id uuid NOT NULL` (= family; exposed as JWT `sid`), `client_id text NOT NULL`, `auth_time timestamptz NOT NULL`, `amr text[] NOT NULL`, `acr text NOT NULL`, `mfa_at timestamptz NULL`, `session_started_at timestamptz NOT NULL`, `last_used_at timestamptz NOT NULL`, `user_agent_hash`, `ip_first`, `ip_last`, `dpop_jkt text NULL` (see 6), `used_at timestamptz NULL` (see 9).
  - Copy these values unchanged into each rotated row.
  - A `RefreshTokenAuthenticatedListener` puts them into the new JWT.
  - Only `/api/auth`, `/api/auth/mfa` and the step-up endpoint (see 12) may change `auth_time`/`amr`/`acr`, and they write the new values into the current family row.
  - Add a governance test: a refresh must never increase `acr` or move `auth_time` forward.

### 3. high: MFA-pending state is unspecified; a full access token may be issued before the second factor
- **Section:** 13.2, 18 (`/api/auth`, `/api/auth/mfa`), 26.3 (Second factor row), 14 (StepUpAssertion only for money/high/MFA-tenant).
- **Problem:** The API is stateless, but the spec never says what `/api/auth` returns to a user with a confirmed authenticator, or how `/api/auth/mfa` knows which user is mid-login.
  - If `/api/auth` returns a normal JWT with `amr=[pwd]`, then for users with MFA enrolled in tenants that don't require MFA, every non-money endpoint is reachable with the password alone. That includes `/api/me/*`, profile data and session listing.
  - No attempt limit for `/api/auth/mfa` is specified. A 6-digit TOTP has 10^6 values.
- **Reference:** NIST 800-63B-4 §3.2.2 (limit consecutive failures per authenticator to ≤ 100). ASVS 6.3.1, 6.5.1, 6.3.4 (no undocumented pathways). OWASP API2:2023.
- **Fix:**
  - When the user has any confirmed authenticator, or the resolved tenant requires MFA, `/api/auth` returns `401 {"error":"mfa_required","mfa_token":…,"methods":[…]}` and **no** access or refresh token.
  - `mfa_token` is a `one_time_token` row with purpose `mfa_challenge` (add to the enum), lasting 5 min. It is bound to `user_id`, `tenant_id`, `client_id` and the password `auth_time`. It allows at most 5 verification attempts (`attempts smallint` column), after which the row is burned and the user must restart the login.
  - Count failures per authenticator, and disable it at 100 consecutive failures (column `user_authenticator.failed_count`).
  - The `mfa_token` must never be accepted by the JWT authenticator (it is opaque, so there is no type confusion).

### 4. high: `email_otp` counts as a second factor, which NIST and ASVS forbid; it is also the password-reset channel
- **Section:** 7.3 / 25.10 (`type … email_otp`), 26.2 (`amr … otp`), 14 StepUpAssertion.
- **Problem:** NIST 800-63B-4 says "Email SHALL NOT be used for out-of-band authentication", and ASVS 6.3.6 says email must not be an authentication factor. Password reset is also email-based (one_time_token `password_reset`). Taking over one mailbox therefore yields both factors, which then authorise money actions.
- **Reference:** https://pages.nist.gov/800-63-4/sp800-63b.html (out-of-band section). ASVS 5.0 6.3.6, 6.4.3.
- **Fix:**
  - Remove `email_otp` from the authenticator `type` CHECK, or keep it only as a recovery or low-assurance signal that emits `amr=["email"]` (custom, not "otp") and **never** satisfies StepUpAssertion or tenant MFA policy.
  - Allowed MFA: `totp` (`amr` otp), `passkey` (`hwk` or `swk` plus `user`), `backup_codes` (recovery only, re-enrol afterwards).
  - Password reset must not bypass MFA (ASVS 6.4.3): after the reset link, a user with an authenticator must still pass it before tokens are issued.

### 5. high: Session lifetime is not related to NIST AAL; no inactivity timeout; step-up windows are undefined
- **Section:** 26.1 (14 days rotating, 30-day cap), 14 StepUpAssertion ("recent authentication"), 13.2.
- **Problem:** The 30-day refresh chain with a 14-day idle window is an **AAL1** session (overall ≤ 30 d). Money and high permissions in a cooperative-finance product need AAL2: overall SHOULD ≤ 24 h, inactivity SHOULD ≤ 1 h. Platform staff are at AAL3 level: overall ≤ 12 h, inactivity ≤ 15 min.
  - "Recent" in StepUpAssertion has no number.
  - There is no inactivity rule.
  - The required ASVS 7.1.1 documentation of timeouts and deviations is missing.
- **Reference:** NIST 800-63B-4 §2.1.3/2.2.3/2.3.3 and §5.2. ASVS 7.1.1, 7.3.1, 7.3.2, 7.5.3, 10.4.8.
- **Fix:** Add a "Session policy" table to section 26:

  | Client (`client_id`) | Refresh idle | Absolute (`max_session_lifetime`) | AAL2 window for StepUpAssertion |
  |---|---|---|---|
  | colsynq-web (member) | 30 min (no refresh for 30 min → family ends) | 12 h | MFA `auth_time` ≤ 12 h **and** last activity ≤ 30 min; money: MFA ≤ 15 min |
  | colsynq-mobile | 14 d (with DPoP; see 6) | 30 d | Same, using the biometric-unlocked key; money: fresh MFA/passkey ≤ 5 min |
  | colsynq-console (platform staff) | 15 min | 12 h | Phishing-resistant (`hwk`/`swk`) ≤ 12 h; impersonation start needs a fresh passkey |

  - Activity is `refresh_token.last_used_at` (an access token lives 10 min, so a refresh gap above the idle time means the user was inactive).
  - When the idle limit is exceeded but the absolute limit is not, NIST allows reauthentication with a password or biometric only.
  - Write this as ASVS 7.1.1 documentation, including the concurrent-session limit (ASVS 7.1.2): `max_tokens_per_user: 10`, oldest evicted.

### 6. high: Mobile refresh tokens are bearer tokens; "origin" binding gives no protection outside browsers
- **Section:** 7.3 (refresh_token `origin`), 25.12, 26.1, 26.5 "Origin binding".
- **Problem:**
  - `Origin` is a browser-enforced header. Native apps, curl and any thief can set it to anything, so `origin` is not a sender constraint.
  - Mobile apps resolve the tenant with `X-Tenant` against one API host, so there is no meaningful origin.
  - Because access tokens are tenant-free (26.5), a token bound to society A's host is usable against society B with `X-Tenant: b`. The binding therefore does not limit blast radius either.
  - gesdinet v3 has no `origin` column; this is custom code the spec doesn't define.
  - RFC 9700 accepts rotation for public clients, but ASVS L3 and FAPI-style finance profiles prefer sender-constrained tokens.
  - NIST 800-63B-4 §5.1: bearer session secrets SHOULD NOT persist across app restart; PoP secrets may.
- **Reference:** RFC 9700 §4.14.2 https://www.rfc-editor.org/rfc/rfc9700#section-4.14.2. RFC 9449 (DPoP) https://www.rfc-editor.org/rfc/rfc9449. ASVS 10.4.5, 10.3.5, 10.4.14.
- **Fix:**
  - Drop `origin` as a security control. Keep it only as `issued_host` for display.
  - Mobile (phase 1): DPoP for refresh. The app generates a non-exportable P-256 key (iOS Secure Enclave / Android StrongBox or TEE) and sends a `DPoP` proof on `/api/auth` and `/api/token/refresh`. The server stores `refresh_token.dpop_jkt` (RFC 7638 thumbprint) and requires the proof (`htm`, `htu`, `iat` ±60 s, `jti` replay cache in Redis 120 s, signature by the same `jkt`).
  - Phase 2: DPoP-bound access tokens (`cnf.jkt`, `Authorization: DPoP`, `ath`).
  - Web: see 7.
  - Optional: App Attest / Play Integrity at login for client attestation.

### 7. high: The web SPA keeps tokens in JavaScript; an httpOnly refresh cookie does not stop XSS from minting tokens
- **Section:** 26.1 (access token "client memory", refresh "httpOnly cookie").
- **Problem:** RFC 10017 §5.1.3 ("acquisition of new tokens"): any XSS can call `/api/token/refresh`, because the cookie is sent automatically. It receives fresh access tokens, and with them a fresh refresh rotation, then exfiltrates them. httpOnly only hides the *string*. For a finance app, RFC 10017 ranks BFF first.
- **Reference:** RFC 10017 §6.1 (BFF), §5.1. ASVS 10.1.1. NIST 800-63B-4 §5.1 (no session secrets in localStorage).
- **Fix:**
  - Preferred: a BFF on each society host. Symfony keeps a server-side session (`__Host-csq_sid`, HttpOnly, Secure, SameSite=Strict, Path=/) and attaches the JWT internally, so no token reaches JS. The lexik JWT then becomes an internal credential.
  - Minimum if the BFF is deferred:
    - strict CSP (nonce-based, no `unsafe-inline`, Trusted Types);
    - the access token only in memory;
    - DPoP with a non-extractable WebCrypto key (`extractable:false`), so stolen tokens cannot be used off-device;
    - refresh cookie rules as in 8.

### 8. high: Cookie scoping, CSRF and CORS for the refresh/logout cookie are unspecified, and the likely scoping breaks logout
- **Section:** 26.1, 18 (`/api/token/refresh`, `/api/logout`), 12.2 (custom domains).
- **Problem:**
  - (a) The earlier research recommends `path=/api/token/refresh`. gesdinet's logout listener needs the refresh token on `/api/logout`, so with that path the cookie is not sent, logout returns an error and the refresh token stays alive.
  - (b) gesdinet cookie settings are global, and with `remove_token_from_body: true` mobile clients receive **no** refresh token in the body.
  - (c) All `<slug>.colsynq.app` hosts are *same-site*. A sibling society subdomain bypasses SameSite=Strict for CSRF (forced refresh, forced logout).
  - (d) Custom domains: if the API is served from `api.colsynq.app`, its cookie is third-party on `coop.example.org` and browsers block it.
  - (e) No CORS policy is stated. Reflecting arbitrary origins with credentials is the classic mistake (API8:2023).
- **Reference:** NIST 800-63B-4 §5.1.1 (`__Host-` prefix, minimum host and path, SameSite). RFC 6265bis. ASVS 3.3.x (cookie attributes), 3.5.x (CSRF). OWASP API8:2023.
- **Fix:**
  - Cookie: `__Host-csq_rt; Secure; HttpOnly; SameSite=Strict; Path=/`. `__Host-` requires `Path=/`, which removes the path problem. It is host-only on the society host, and the API is served same-origin under `https://<society-host>/api` (required for custom domains).
  - Move logout to `POST /api/token/revoke` (see 10).
  - Turn `cookie.remove_token_from_body` off, and add a `RefreshResponseListener` that removes the body token when `client_id = colsynq-web` and keeps it for mobile.
  - On every cookie-authenticated endpoint (refresh, revoke, BFF), require `Origin` ∈ the resolved tenant's verified hosts and a custom header `X-Csq-Csrf: 1`, and reject `Content-Type` values other than `application/json`.
  - CORS: deny by default. Only the hosts of the resolved tenant (from tenant_domain, status verified) get `Access-Control-Allow-Origin` (exact echo) plus `Allow-Credentials`; never `*`; `Vary: Origin`.

### 9. medium: Refresh rotation on unreliable mobile networks will cause false theft detections; reuse detection depends on Redis
- **Section:** 26.3 (Refresh and reuse rows), 26.5.
- **Problem:**
  - In Nigeria, many refresh responses are lost after the server has rotated. The client retries with the previous token, reuse detection fires, the family is killed, the user is logged out and "notified of theft". Users learn to ignore the alert.
  - gesdinet keeps spent-token digests only in the cache. If Redis evicts them or restarts, reuse goes undetected.
- **Reference:** RFC 9700 §4.14.2 (rotation; the AS "may" choose detection strategies). ASVS 10.4.5.
- **Fix:**
  - Add `refresh_token.used_at` and `replaced_by` (hash), and keep used rows until family expiry. This makes the DB the source of truth.
  - Grace window: within 30 s of `used_at`, a replay of the immediately previous token **from the same DPoP `jkt`** (mobile) or the same session cookie (web) returns the already-issued successor instead of revoking. Anything else revokes the family.
  - Redis instances holding the jti blocklist, reuse digests and DPoP jti must use `maxmemory-policy noeviction` (or `volatile-ttl` on a dedicated instance) with AOF persistence.
  - Clients must refresh single-flight.

### 10. medium: No revocation endpoint usable with an expired access token; logout semantics are incomplete
- **Section:** 18, 26.3 (Log out row).
- **Problem:**
  - If `/api/logout` sits behind `IS_AUTHENTICATED_FULLY`, a user whose 10-minute JWT has expired cannot revoke their refresh token.
  - Logout blocks only the *presented* jti. Other access tokens issued in the same session (other tabs) live for up to 10 min.
  - There is no "end session X" by id for `/api/me/sessions`. ASVS 7.5.2 requires reauthentication before terminating sessions.
  - Admins cannot end a user's sessions (ASVS 7.4.5).
- **Reference:** RFC 7009 https://www.rfc-editor.org/rfc/rfc7009. ASVS 7.4.1, 7.4.5, 7.5.2. NIST 800-63B-4 §5.1 (secrets invalidated at logout).
- **Fix:**
  - `POST /api/token/revoke` (PUBLIC_ACCESS, rate-limited): takes the refresh token from the cookie or the `token` body field and always returns 200 (RFC 7009 §2.2). It revokes the **whole family** (`family_valid = now`) and clears the cookie.
  - Add a `sid` claim (= `refresh_token.session_id`) to every JWT, and keep a Redis set `revoked_sid:{sid}` with TTL = access-token lifetime + skew. The JWT authenticator rejects revoked `sid`s. This replaces per-jti logout and covers all tabs.
  - `DELETE /api/me/sessions/{sessionId}` requires `auth_time` ≤ 5 min.
  - `POST /api/platform/users/{id}/sessions/revoke` (platform permission) bumps `token_version`.
  - A tenant admin's equivalent is disabling the profile; document this.

### 11. medium: The access token is not RFC 9068-conformant and validation is only partly specified
- **Section:** 26.2, 26.1 ("iss, aud, exp, ver checks").
- **Problem:**
  - RFC 9068 requires `client_id`, which is missing, so per-client policy (5, 8) is impossible.
  - `typ` validation is not listed, and lexik's default lcobucci encoder checks only signature and time. It does **not** check `iss`, `aud` or `typ` (see ../auth/jwt_symfony.md).
  - `jti` exists only when lexik `blocklist_token.enabled: true`; the spec doesn't state this.
  - `clock_skew` is unspecified, and lexik's default is 0.
  - The `aud` value `colsynq-api` is not a resource URI.
  - There is no `acr`.
  - `nbf` is not set.
- **Reference:** RFC 9068 §2.2, §4. RFC 8725 §3.8, 3.9, 3.11 and 8725bis §3.9/3.11. ASVS 9.1.1–9.2.3.
- **Fix:** Replace the 26.2 claims table with:
  - **Header:** `alg ES256`, `typ at+jwt`, `kid`.
  - **Claims:** `iss=https://api.colsynq.app`, `aud=https://api.colsynq.app/` (or the per-tenant value from 13), `sub`, `client_id` (`colsynq-web` | `colsynq-mobile` | `colsynq-console`), `iat`, `nbf`, `exp` (+600), `jti` (128-bit), `sid`, `ver`, `auth_time`, `amr` (RFC 8176 values), `acr` (`urn:colsynq:aal1|aal2|aal3`, or the OIDC EAP values `phr`/`phrh` for passkeys), optional `cnf.jkt`.
  - **Validation**, in a `JWT_DECODED` listener or the web-token encoder with checkers:
    - `typ == at+jwt` (case-insensitive per RFC 9068 §4);
    - `alg` ∈ {ES256} only;
    - exact `iss`;
    - `aud` contains ours;
    - `exp`/`nbf`/`iat` with `clock_skew: 30`;
    - reject `iat` > now + 30 s;
    - `client_id` ∈ the registered list;
    - `sub` is a UUID that resolves to an active user of kind person;
    - `ver` == token_version;
    - `sid` not revoked.
  - Config: `blocklist_token.enabled: true`, `allow_no_expiration: false`, query-parameter and cookie extractors disabled.

### 12. medium: The step-up error is not RFC 9470; there is no step-up endpoint and no `acr`
- **Section:** 14 (StepUpAssertion → `401 step_up_required`), 20.
- **Problem:** The proprietary 401 code gives the client no machine-readable statement of *what* is needed. RFC 9470 defines this exactly.
- **Reference:** RFC 9470 https://www.rfc-editor.org/rfc/rfc9470. ASVS 10.3.4, 7.5.1, 7.5.3. RFC 6750 §3 (WWW-Authenticate on every 401).
- **Fix:**
  - The response is `401` with `WWW-Authenticate: Bearer error="insufficient_user_authentication", error_description="…", acr_values="urn:colsynq:aal2", max_age=900`. The body is a problem+json with `code: step_up_required` and the same `acr_values`/`max_age`.
  - Add `POST /api/auth/step-up`. It is authenticated, takes a TOTP or passkey assertion, writes the new `auth_time`/`amr`/`acr` into the session's family row (see 2) and returns a new access token.
  - Every other 401 carries `WWW-Authenticate: Bearer error="invalid_token"` (RFC 6750).
  - Section 20: change "Step-up needed | 401 | step_up_required" to "401 + WWW-Authenticate insufficient_user_authentication (RFC 9470)".

### 13. medium: One tenant-free audience for all societies (blast radius, X-Tenant confused deputy)
- **Section:** 2 (Tokens), 26.2 (aud), 26.5 (Tenant-free tokens), 12.2 (X-Tenant first).
- **Problem:** One stolen token, or an XSS on any one society's frontend (including a custom domain the society controls DNS for), works against every society the user belongs to, and against platform routes if the user is staff. RFC 8725bis and ASVS 9.2.4 want an audience that identifies the intended recipient. RFC 8707 resource indicators exist for exactly this.
- **Reference:** RFC 8707 https://www.rfc-editor.org/rfc/rfc8707. ASVS 9.2.3, 9.2.4, 8.4.1. OWASP API1/API5:2023.
- **Fix:** Keep sign-in global but make access tokens tenant-scoped:
  - The refresh family stays user-level.
  - `POST /api/token/refresh` takes the resolved tenant and mints an access token with `aud = https://<tenant-host>/` (or a claim `tid` that must equal `ResolvedTenant.id`), but only if an active profile exists.
  - The JWT authenticator rejects a token whose `aud`/`tid` differs from the resolved tenant (401 invalid_token).
  - Platform-console tokens get `aud = https://console.colsynq.app/` and `client_id = colsynq-console`, and are never valid on society hosts.
  - Switching society means one refresh call.
  - Update decision row "Tokens": "JWT carries the user ID, authentication facts and the tenant audience it was minted for; roles and profile stay server-side."

### 14. high: Support impersonation has no target, no token marker and no RFC 8693 `act`; revocation is not immediate
- **Section:** 16, 14 step 3b, 25.23.
- **Problem:**
  - `support_access_grant.mode = impersonate` names the support person but **not whom** they impersonate.
  - The staffer's ordinary JWT is reused, so resource code and downstream services cannot tell an impersonated call from a normal one (ASVS 8.3.3).
  - Revoking the grant (`revoked_at`) is not part of the access-cache key, which only mentions expiry, so access lasts up to 5 more minutes (ASVS 8.3.2).
  - Nothing stops impersonating a TENANT_ADMIN.
- **Reference:** RFC 8693 §4.1 (`act` claim), §4.4 (`may_act`) https://www.rfc-editor.org/rfc/rfc8693. ASVS 8.3.2, 8.3.3. OWASP API5:2023.
- **Fix:**
  - Table: add `target_profile_id uuid NULL` with FK `(target_profile_id, tenant_id) → user_profile(id, tenant_id)`, and a CHECK `(mode='impersonate') = (target_profile_id IS NOT NULL)`. Add a trigger: the target holds no TENANT_ADMIN and no platform roles.
  - Add `POST /api/support/token`, a token-exchange endpoint using RFC 8693 semantics:
    - `subject_token` = the staffer's access token, which needs `acr ≥ aal3`/`hwk` and `auth_time` ≤ 5 min;
    - `requested_token_type=urn:ietf:params:oauth:token-type:access_token`;
    - it returns a JWT with `sub = target user`, `act = {"sub": "<staff user id>", "iss": "https://api.colsynq.app"}`, `aud = tenant host`, `client_id = colsynq-console`, `support_grant_id`, and `exp = min(now+15 min, grant.expires_at)`;
    - no refresh token.
  - Read-only mode issues `sub = staff`, `act` omitted, `support_grant_id` set.
  - The pipeline checks `support_grant_id` live on every request (one indexed lookup `WHERE id=? AND revoked_at IS NULL AND now() < expires_at`) instead of trusting the cache.
  - AccessDecision denies money, access-change and export slugs whenever `act` is present.
  - audit_event records `actor = act.sub` and `subject = sub`.
  - Notify the target user and the tenant admins at start and end.

### 15. high: Future per-tenant IdPs (26.8) would let one society authenticate users into other societies; JWKS/discovery SSRF
- **Section:** 26.8, 25.11 (external_identity is user-global), 26.6.
- **Problem:**
  - `external_identity` maps `(provider, subject)` to a global `app_user`. If a society may configure its own OIDC issuer (enterprise SSO), that society's IdP admin can mint any `sub` and/or email. A user linked through it would then sign in to **all** of that user's societies, including platform staff accounts.
  - Discovery and JWKS URLs from tenant configuration are an SSRF vector, and unknown-`kid` refetch is a DoS vector.
  - There is no `logout+jwt` handling.
- **Reference:** ASVS 6.8.1, 9.1.3, 10.5.3, 10.5.5. RFC 8725bis §3.10 (SSRF on key URLs). OWASP API7:2023 (SSRF), API4:2023. OIDC Back-Channel Logout 1.0 https://openid.net/specs/openid-connect-backchannel-1_0.html. NIST 800-63C-4 (federation; the RP decides reauth).
- **Fix:**
  - Add `identity_provider (id, kind platform|tenant, tenant_id NULL, issuer UNIQUE, discovery_url, allowed_algs text[], client_id, status)`, and `external_identity.identity_provider_id FK`.
  - A login through a `kind = tenant` IdP may only open that tenant: ProfileLoginValidator rejects other tenants, and the token is minted with that tenant's `aud` (see 13). It never signs in platform staff.
  - Outbound fetches go through a dedicated HttpClient:
    - https only;
    - DNS resolved and private, loopback, link-local and metadata ranges (169.254.169.254, fd00::/8, …) rejected;
    - no redirects;
    - 2 s timeout;
    - 64 KB body limit.
  - JWKS cache with refetch on unknown `kid` at most once per 5 min per issuer.
  - Metadata `issuer` must exactly equal the configured value.
  - Back-channel logout: validate `typ=logout+jwt`, the `events` claim, no `nonce`, `jti` replay, `exp` ≤ 2 min, then revoke matching `sid`s.
  - The same SSRF client is used for custom-domain verification (`/api/tenant/domains/verify`) and any webhook.

### 16. medium: Provider email trust rules are wrong or vague ("verified it and owns the domain")
- **Section:** 26.7 (New person row), 25.11.
- **Problem:** Each provider has its own rules.
  - **Google:** `email_verified` is reliable. `hd` identifies Workspace domains.
  - **Microsoft Entra / MSA:** the `email` claim is mutable and unverified (the "nOAuth" takeover). Trust it only with the optional claim `xms_edov=true`, or not at all. On the `common`/`organizations` endpoints, `iss` contains the token's `tid` and must be validated against the template. Microsoft's own guidance keys users by `tid`+`oid`, and `sub` is pairwise per app.
  - **Apple:** `email` may be a private relay (`@privaterelay.appleid.com`, `is_private_email=true`). `email_verified` may be the string `"true"`. Name arrives only on the first authorization, in the `user` form field (`response_mode=form_post`).
  - A verified provider email must also not *auto-link* to an existing Colsynq account. The spec is correct about that; keep it.
- **Reference:** https://learn.microsoft.com/en-us/entra/identity-platform/migrate-off-email-claim-authorization. ASVS 6.8.1. OIDC Core §5.7 (email is not a stable identifier).
- **Fix:** Replace the 26.7 New person row text with a provider table:
  - Google: verified if `email_verified===true`.
  - Microsoft: verified only if `xms_edov===true`, else the user must verify via a Colsynq email link. Subject key = `tid:oid`. `iss` must equal `https://login.microsoftonline.com/{tid}/v2.0`.
  - Apple: verified if `email_verified` ∈ {true, "true"}; store `is_private_email`; relay addresses never count for `email_domain` sign-up policy.
  - Add columns `external_identity.is_private_email bool`, `external_identity.issuer text`, and use `(issuer, subject)` as the unique key instead of `(provider, subject)`.
  - Apple server-to-server notifications endpoint (`email-disabled`, `consent-revoked`, `account-delete`) unlinks the identity and revokes its sessions.

### 17. medium: ID-token validation list incomplete; nonce never stored; PKCE support varies by provider
- **Section:** 26.7 (sequence step "validated: signature, issuer, audience, nonce, expiry").
- **Problem:**
  - The nonce is validated but not stored at Start.
  - `azp` is not checked.
  - `iat` freshness is not checked.
  - There is no `alg` allow-list per provider.
  - The JWKS source is not pinned.
  - league/oauth2-client does not validate id_tokens itself (../auth/oauth2_oidc_symfony.md).
  - Apple does not document PKCE support [unverified], so `state` + `nonce` must carry the protection there.
  - Mix-up across three providers is not addressed.
- **Reference:** OIDC Core §3.1.3.7 https://openid.net/specs/openid-connect-core-1_0.html#IDTokenValidation. RFC 9700 §2.1, §4.4 (mix-up; RFC 9207). ASVS 10.5.1–10.5.4, 10.2.2.
- **Fix:** Validate in this order:
  1. JWS with `alg` ∈ the provider allow-list (RS256);
  2. key from the configured JWKS (discovery URL hard-coded per provider);
  3. `iss` exact (Microsoft template per 16);
  4. `aud` contains our client_id;
  5. if multiple audiences, `azp` == client_id;
  6. `exp`, and `iat` ≤ 5 min old with skew 60 s;
  7. `nonce` == stored nonce (Apple: SHA-256 of the raw nonce when the native SDK is used);
  8. `at_hash` if present;
  9. `auth_time` if `max_age` was sent.

  Also:
  - Use a distinct `redirect_uri` per provider and check the callback provider against the transaction (see 1); also check RFC 9207 `iss` where supplied.
  - Use PKCE S256 for every provider that supports it, even though Colsynq is a confidential client (RFC 9700 §2.1.1).
  - Store `nonce` in the transaction record.

### 18. high: Callback, redirect URI and token delivery do not work for per-society hosts; X-Tenant cannot be sent on a navigation
- **Section:** 26.7 (`/start` "+ X-Tenant", final step "Colsynq JWT + refresh token").
- **Problem:**
  - A top-level browser navigation cannot carry `X-Tenant`.
  - Providers require exact, pre-registered redirect URIs (no wildcards). Registering every `<slug>.colsynq.app` and custom domain at Google, Apple and Microsoft is impossible.
  - The final step returns tokens in response to a GET navigation, so it is undefined how they reach the SPA. Tokens in the URL would violate RFC 9700 §4.3 and leak through Referer and history.
- **Reference:** RFC 9700 §2.1, §4.1 (exact redirect matching), §4.3. RFC 8252 (native apps: system browser, claimed https redirects). ASVS 10.4.1.
- **Fix:**
  - Use one registered callback per provider on a central host: `https://auth.colsynq.app/oauth/{provider}/callback`. The tenant comes from the society host at `/start` and travels in the server-side transaction (see 1), never in `state` itself.
  - After success, mint a **handoff code**: a one_time_token with purpose `oauth_handoff`, 60 s, single use, bound to `tx_id` and tenant.
  - Redirect (303) to `https://<society-host>/auth/complete#code=…`. That host's SPA or BFF POSTs the code to `/api/auth/oauth/redeem` on its own host, which sets the `__Host-` refresh cookie and returns the access token.
  - Mobile: use the native SDKs (Google Credential Manager, Sign in with Apple) and add `POST /api/auth/oauth/{provider}/id-token`, which takes `id_token` and the raw nonce, validates them as in 17 and issues tokens with DPoP (see 6).

### 19. medium: Social sign-in can bypass Colsynq MFA and tenant MFA policy
- **Section:** 26.7, 13.2, 14 StepUpAssertion.
- **Problem:** Neither the sequence nor the text says that a user with Colsynq authenticators, or in an MFA-required tenant, must still pass the Colsynq second factor after a provider login. Providers' `amr` values are absent or unreliable (Google), so they cannot be mapped to AAL2. NIST 800-63B-4 §5.2 and 800-63C make the RP the authority on reauthentication.
- **Reference:** ASVS 6.1.3, 6.3.4 (consistent strength across pathways), 6.8.4. NIST 800-63C-4 https://pages.nist.gov/800-63-4/sp800-63c.html.
- **Fix:**
  - A social sign-in yields `amr=["fed"]` (Colsynq-local value, documented) and `acr=aal1`. It then follows the same `mfa_required` path as finding 3.
  - Step-up for social-only users uses a passkey or TOTP. If they have none, they are forced to enrol before any AAL2 action.
  - Never derive `acr` from provider claims unless the provider is a configured enterprise IdP that returns `acr` and is trusted for it.

### 20. medium: Tenant MFA policy enforced only as an assertion on permission slugs
- **Section:** 13.2, 14 (StepUpAssertion "and MFA-required tenants").
- **Problem:** Assertions run only when a permission slug is checked. Endpoints guarded only by "authenticated" in an MFA-required tenant return data with `amr=[pwd]`. Those endpoints are `GET /api/me/profile`, `/api/me/access`, `/api/me/profiles`, session listing and authenticator listing.
- **Reference:** ASVS 6.3.4, 8.1.4. OWASP API5:2023.
- **Fix:** Enforce tenant MFA in pipeline step 3 (profile check). If `ResolvedTenant.requires_mfa` and the token's `acr` < aal2, return 401 per finding 12. The only exceptions are `/api/me/authenticators` enrolment routes, marked `#[AllowWithoutTenantMfa]` and covered by an architecture test.

### 21. medium: The trusted-device cookie may stand in for MFA
- **Section:** 7.1 (`trusted_device_version`).
- **Problem:** There is no rule for what a trusted device allows. NIST 800-63B-4 §5.1.1: "remember my browser" cookies SHALL NOT replace authentication, except for AAL2 reauthentication after an inactivity timeout.
- **Reference:** NIST 800-63B-4 §5.1.1, §2.2.3.
- **Fix:**
  - A trusted device may skip the *second* factor only when an AAL2 session exists and only its inactivity timer has expired (not the overall timer).
  - It never sets `amr` to include `otp`/`hwk`, and it never satisfies money step-up.
  - Store the trusted-device record server-side (`user_trusted_device(id, user_id, token_hash, created_at, expires_at ≤ 30 d, revoked_at)`) instead of a version-only cookie, so devices can be listed and revoked individually.

### 22. medium: API keys are full-power static bearer secrets with no scopes, no format, no rotation overlap and no proof of possession for money flows
- **Section:** 7.3, 25.13, 26.1, 26.3 (Call with an API key).
- **Problem:**
  - A key acts as the whole service-account profile; there is no per-key permission subset.
  - There is no identifiable format for leak scanning.
  - There is no pepper version, so the pepper cannot be rotated without invalidating every key.
  - `ip_allow_list` depends on `trusted_proxies` configuration (X-Forwarded-For spoofing).
  - `last_used_at` is written on every call (write amplification).
  - There are no per-key rate limits, no expiry reminders and no `created_by_profile_id`.
  - Static bearer keys have no replay protection, which matters for money-moving integrations (payroll uploads).
- **Reference:** ASVS 7.2.2 (static keys not for user sessions; acceptable for machines only with compensating controls), 6.4.5, 6.5.6. OWASP API2, API4:2023. RFC 9700 §2.2 (prefer sender-constrained). RFC 7523 / RFC 9421 (HTTP Message Signatures) https://www.rfc-editor.org/rfc/rfc9421.
- **Fix:** Columns to add:
  - `scopes text[] NOT NULL` (permission slugs ⊆ the profile's effective permissions, checked at use: effective = profile ∩ key scopes);
  - `pepper_version smallint NOT NULL`;
  - `created_by_profile_id uuid NOT NULL`;
  - `rate_limit_per_min int`;
  - `last_used_ip inet`;
  - `rotated_from_id uuid NULL`.

  Behaviour:
  - Format `csq_{env}_{prefix8}_{secret32 base62}_{crc32}`. Register the pattern with GitHub secret scanning.
  - Update `last_used_at` at most once per 5 min.
  - Send expiry reminders at 30 and 7 days.
  - Allow two active keys per account for rotation.
  - Money-moving slugs (`*:disburse`, `*:post`) refuse API-key auth unless the request carries an RFC 9421 signature with a registered public key (`api_key.public_jwk jsonb`, `created`/`nonce` replay cache of 5 min in Redis), or (later) a private_key_jwt/mTLS client-credentials token per 26.9.
  - Document `framework.trusted_proxies` and `trusted_headers`.

### 23. medium: Signed-URL join links are replayable and unrevocable; the email-change flow is unspecified
- **Section:** 7.3 "Not stored: email-verification and join links (signed URLs)", 13.1, 25.15.
- **Problem:**
  - A join link (a credential that adds a profile and SELF_SERVICE) is usable repeatedly until expiry and cannot be revoked. Forwarded links let the wrong verified account join, unless the signature binds the user ID and email.
  - Email-change verification and old-address notification are not defined (ASVS 6.3.7, 7.5.1).
  - Signed URLs probably share `APP_SECRET` with other signatures, against the RFC 8725 one-key-per-purpose principle.
- **Reference:** ASVS 6.4.1, 6.3.7, 7.5.1. RFC 8725 §3.12.
- **Fix:**
  - Move join links to `one_time_token` (purpose `join`, bound to `user_id` + `tenant_id`, 7 d, single use).
  - Add purpose `email_change` (bound to user and new email, 1 h), with a notice to the old address that includes a 7-day revert link (`email_change_revert`).
  - Keep stateless signatures only for plain email verification, signed with a dedicated key (`URI_SIGNER_SECRET`, not `APP_SECRET`) and binding `user_id`, `email` and `exp`.
  - Lifetimes: `password_reset` 30 min, `set_password` 72 h, `signup` 24 h. Revoke on use.

### 24. medium: Signing keys and other secrets: "vault" is not specified; `kid` selection conflicts with lexik's lcobucci encoder
- **Section:** 26.1, 26.2 (kid "picks the current or previous key"), 25.10 (`secret_encrypted`), 25.13 (pepper).
- **Problem:**
  - lexik's lcobucci encoder ignores `kid` and tries `additional_public_keys` in order. Only the web-token encoder selects by `kid`.
  - The private key is loaded into PHP memory from env or a file. No KMS/HSM, rotation cadence, compromise runbook or separation of keys per purpose is defined.
  - TOTP seeds and the API-key pepper have no key version, so they cannot be rotated.
  - OAuth client secrets and the Apple `.p8` key live in env. The Apple client-secret JWT expires after at most 6 months.
- **Reference:** RFC 7515 §4.1.4, RFC 7517 (kid/JWKS). RFC 8725bis §3.1 (one alg per key), §3.10 (sanitise kid). ASVS 9.1.3, 11.x (cryptographic inventory, key management), 13.3 (secrets management).
- **Fix:**
  - Use lexik with the **web-token encoder**, a JWKSet with `kid` = RFC 7638 thumbprint, `use=sig`, `alg=ES256`, and `kid` looked up only in the in-memory set (never as a path or SQL).
  - Rotate every 90 days. Add the new public key 1 TTL before signing with it, and remove the old one after TTL + skew. Emergency rotation (key compromise) removes the old `kid` from the verification set at once. That signs everyone out within one refresh, with no user-side bump needed, because access tokens live 10 minutes.
  - Keep the private key in a cloud KMS/HSM if a custom `JWSProvider` that calls KMS Sign is acceptable. Otherwise use Symfony secrets or Vault with a passphrase-encrypted PEM, never `.env`.
  - Add `user_authenticator.key_version smallint` and `api_key.pepper_version smallint`.
  - Keep Apple, Google and Microsoft secrets in the vault, with a scheduled job that regenerates the Apple client secret monthly.
  - Use separate keys for access tokens, URI signing, the handoff code and the DPoP nonce (if used).

### 25. medium: Platform-staff sessions share client, audience and lifetime with members
- **Section:** 12.2 (Platform routes), 26.1, 16.
- **Problem:** A staff member's token from browsing a society site is valid on `_tenant = platform` routes, which are BFLA-sensitive (`tenants:manage`, `plans:manage`). Staff need AAL3-like sessions: phishing-resistant authentication, 12 h maximum and 15 min idle.
- **Reference:** NIST 800-63B-4 §2.3.3. ASVS 6.3.3 (L3 hardware authenticator), 8.4.2. OWASP API5:2023.
- **Fix:**
  - Use a separate client `colsynq-console` on `console.colsynq.app` with `aud` per finding 13.
  - Platform routes require `client_id=colsynq-console`, `acr=aal3` (`amr` ⊇ {hwk|swk, user}) and `auth_time` ≤ 12 h. Use the session policy from 5.
  - Society hosts reject console tokens and vice versa.

### 26. medium: Support grant revocation and access cache
- **Section:** 14 step 4 (TTL "5 minutes or earliest grant or support-grant expiry").
- **Problem:** `revoked_at` on a support grant or a user_role takes effect only at cache expiry. The fact that user_role changes bump `access_version` covers grants, but nothing covers the support grant.
- **Reference:** ASVS 8.3.2.
- **Fix:**
  - Revoking a support grant bumps a `support_access_version` on the tenant (or uses the live check from 14).
  - Add a test: after revocation, the next request returns 404.

### 27. medium: Rate limits and resource consumption are under-specified for auth endpoints
- **Section:** 13.1 (only sign-up/join/resend), 26.3.
- **Problem:** No limits are stated for:
  - `/api/auth/mfa` and step-up (see 3);
  - `/api/token/refresh` (gesdinet `rate_limiter`);
  - `/api/token/revoke`;
  - password reset;
  - the OAuth start and callback endpoints;
  - the id-token endpoint;
  - API-key calls.

  `locked_until` lockout by account allows targeted lockout, which ASVS 6.1.1 says to prevent. NIST requires ≤ 100 consecutive failures per authenticator.
- **Reference:** ASVS 6.1.1, 6.3.1. NIST 800-63B-4 §3.2.2. OWASP API4:2023.
- **Fix:** Add a table:

  | Endpoint | Limit |
  |---|---|
  | login | 5 per (ip, email) per 15 min; 50 per ip per 15 min; global per email 100 per day, then require CAPTCHA-free proof (a passkey, or an emailed link to unlock) instead of a lock |
  | mfa | 5 per mfa_token; 100 consecutive per authenticator → disable |
  | refresh | 30 per session per min |
  | revoke | 20 per ip per min |
  | reset | 3 per email per hour; 20 per ip per hour |
  | oauth start | 20 per ip per min |
  | API key | `rate_limit_per_min` per key (default 600) |

  All limits return 429 with `Retry-After`. Counters live in Redis with `noeviction`.

### 28. low: Password change or reset signs out the current session too, with no option
- **Section:** 26.3 (Sign out everywhere / password change).
- **Problem:** ASVS 7.4.3 expects the user to be offered termination of *other* sessions after a factor change. NIST and ASVS also expect a notification (6.3.7). Bumping `token_version` kills the current session as well, forcing a re-login.
- **Reference:** ASVS 7.4.3, 6.3.7.
- **Fix:**
  - On password change (authenticated, step-up): bump `token_version`, revoke all families except the current one, and re-issue the current access token with the new `ver`.
  - On reset (unauthenticated): revoke everything.
  - Always notify by email.
  - Adding or removing an authenticator offers "sign out other sessions" and notifies.

### 29. low: Refresh writes `audit_event` on every rotation
- **Section:** 26.3 (Refresh row "audit_event").
- **Problem:** One row per user every 10 minutes would swamp the append-only audit store and its retention costs, and would bury security events.
- **Fix:** Log refreshes to the security log (Monolog channel `security`, without tokens). Write `audit_event` only for sign-in, step-up, logout, reuse detection, revocation and session-policy expiry.

### 30. low: Service users and person users must be held to their own credential types
- **Section:** 7.1 (`kind`), 14 step 2.
- **Problem:** Nothing states that the JWT authenticator refuses `kind = service` or that the API-key authenticator refuses `kind = person`.
- **Fix:** Add both checks in `UserChecker::checkPreAuth`, with an architecture test.

### 31. low: Concurrent-session limits and session metadata are undocumented
- **Section:** 26.1, `/api/me/sessions`.
- **Reference:** ASVS 7.1.2, 7.5.2.
- **Fix:**
  - Set `max_tokens_per_user: 10` (gesdinet); signing in an eleventh time evicts the least recently used session.
  - Session list rows show `client_id`, the device label (client-supplied, display only), `ip_first`/`ip_last` (truncated /24), `session_started_at` and `last_used_at`.
  - Notify the user on a sign-in from a new client or device (ASVS 6.3.5).

### 32. low: Protected-resource metadata and OAuth discovery for future partners
- **Section:** 26.6, 26.9.
- **Problem:** When Colsynq becomes a resource server for an external AS, clients need to discover the AS and required scopes.
- **Reference:** RFC 9728 https://www.rfc-editor.org/rfc/rfc9728. RFC 8414. RFC 8705.
- **Fix:**
  - Publish `/.well-known/oauth-protected-resource` (resource, authorization_servers, `dpop_signing_alg_values_supported`, `tls_client_certificate_bound_access_tokens`).
  - Add `resource_metadata="…"` to `WWW-Authenticate`.
  - Section 26.9: require `cnf.x5t#S256` (mTLS) or `cnf.jkt` (DPoP) on partner tokens, and no refresh tokens for client_credentials.

### 33. low: Error and enumeration details
- **Section:** 20, 12.2.
- **Problem:**
  - `400 tenant_mismatch` for an API key used at another tenant confirms that the key is valid.
  - The ProfileLoginValidator increments `failed_login_count` on a *correct* password at a tenant with no profile, which locks users who try the wrong society.
- **Fix:**
  - An API key used at the wrong tenant returns `401 invalid_token` with the same timing as an unknown key.
  - A correct password with no profile in the tenant returns the same error but does not increment `failed_login_count`. It audits `sign_in.no_profile`.

### 34. low: Redis is a security dependency (blocklist, reuse cache, OAuth transactions, DPoP jti)
- **Section:** 25.15, 26.10.
- **Problem:** If Redis is evicted, flushed or unavailable, the system fails open: blocked jtis are accepted again and reuse goes undetected.
- **Fix:**
  - Use a dedicated Redis with TLS, AUTH/ACL, `noeviction`, AOF and keys prefixed `sec:`.
  - On Redis failure, the JWT authenticator fails **closed** for blocklist lookups (503) and refresh is refused.
  - Document this under section 20.

---

## Proposed table changes (summary)

| Table | Change |
|---|---|
| `iam.refresh_token` | **Add** `session_id uuid`, `client_id text`, `auth_time timestamptz`, `amr text[]`, `acr text`, `mfa_at timestamptz`, `session_started_at timestamptz`, `last_used_at timestamptz`, `used_at timestamptz`, `replaced_by text`, `dpop_jkt text`, `issued_host text`, `ip_first inet`, `ip_last inet`, `device_label text`. **Drop** `origin` as a control. Index `(username, family)`. CHECK `client_id IN ('colsynq-web','colsynq-mobile','colsynq-console')` |
| `iam.one_time_token` | purpose enum += `mfa_challenge`, `oauth_handoff`, `join`, `email_change`, `email_change_revert`. **Add** `attempts smallint DEFAULT 0`, `client_id text`, `bound_tx text` |
| `iam.user_authenticator` | **Remove** `email_otp` as an MFA type (or flag `counts_as_mfa=false`). **Add** `failed_count int`, `key_version smallint` |
| `iam.external_identity` | **Add** `identity_provider_id uuid FK`, `issuer text`, `is_private_email bool`. Unique key becomes `(issuer, subject)`; for Microsoft the subject is `tid:oid` |
| `iam.identity_provider` (new) | `id, kind (platform|tenant), tenant_id NULL, issuer UNIQUE, discovery_url, allowed_algs text[], client_id, status, created_at`. CHECK `(kind='tenant') = (tenant_id IS NOT NULL)` |
| `iam.api_key` | **Add** `scopes text[] NOT NULL`, `pepper_version smallint`, `created_by_profile_id uuid`, `rate_limit_per_min int`, `last_used_ip inet`, `rotated_from_id uuid`, `public_jwk jsonb NULL` (RFC 9421 signing) |
| `iam.support_access_grant` | **Add** `target_profile_id uuid NULL` (composite FK), CHECK `(mode='impersonate') = (target_profile_id IS NOT NULL)`, `ended_at`. Trigger: target is not TENANT_ADMIN or platform staff |
| `iam.user_trusted_device` (new) | `id, user_id, token_hash, label, created_at, expires_at, last_used_at, revoked_at` (replaces the version-only approach) |
| `tenancy.tenant` | **Add** `requires_mfa bool`, or confirm it lives in sign-up policy (needed by pipeline step 3) |

## Proposed replacement for 26.2 (access token)

```
Header: {"alg":"ES256","typ":"at+jwt","kid":"<RFC 7638 thumbprint>"}
Claims: iss=https://api.colsynq.app, aud=https://<tenant-host>/ (or console host), sub=<app_user.id>,
        client_id, iat, nbf, exp=iat+600, jti (128-bit), sid=<refresh session_id>, ver=<token_version>,
        auth_time, amr (RFC 8176 + documented local values "fed","email"), acr=urn:colsynq:aal1|aal2|aal3,
        cnf.jkt (DPoP, mobile/web when enabled), act={sub,iss} (support impersonation only),
        support_grant_id (support only)
Validation: alg allow-list {ES256}; kid lookup in in-memory JWKSet; typ==at+jwt; iss exact; aud == resolved
            tenant host; exp/nbf/iat with 30 s skew; client_id registered and allowed for the route family;
            ver == app_user.token_version; sid not revoked; jti not blocked; cnf.jkt matches DPoP proof when present;
            user.kind == person; on failure 401 + WWW-Authenticate: Bearer error="invalid_token".
```

## Sources

- RFC 7519 https://www.rfc-editor.org/rfc/rfc7519 · RFC 7515 https://www.rfc-editor.org/rfc/rfc7515 · RFC 7517 https://www.rfc-editor.org/rfc/rfc7517 · RFC 7638 https://www.rfc-editor.org/rfc/rfc7638
- RFC 8725 https://www.rfc-editor.org/rfc/rfc8725 · draft-ietf-oauth-rfc8725bis-10 https://datatracker.ietf.org/doc/draft-ietf-oauth-rfc8725bis/
- RFC 9068 https://www.rfc-editor.org/rfc/rfc9068 · RFC 8176 https://www.rfc-editor.org/rfc/rfc8176 · RFC 9470 https://www.rfc-editor.org/rfc/rfc9470 · RFC 8693 https://www.rfc-editor.org/rfc/rfc8693
- RFC 6749 https://www.rfc-editor.org/rfc/rfc6749 · RFC 6750 https://www.rfc-editor.org/rfc/rfc6750 · OAuth 2.1 draft-16 https://datatracker.ietf.org/doc/draft-ietf-oauth-v2-1/ · RFC 9700 https://www.rfc-editor.org/rfc/rfc9700
- RFC 7009 https://www.rfc-editor.org/rfc/rfc7009 · RFC 7662 https://www.rfc-editor.org/rfc/rfc7662 · RFC 8414 https://www.rfc-editor.org/rfc/rfc8414 · RFC 9126 https://www.rfc-editor.org/rfc/rfc9126 · RFC 9207 https://www.rfc-editor.org/rfc/rfc9207 · RFC 8707 https://www.rfc-editor.org/rfc/rfc8707 · RFC 9728 https://www.rfc-editor.org/rfc/rfc9728
- RFC 9449 DPoP https://www.rfc-editor.org/rfc/rfc9449 · RFC 8705 mTLS https://www.rfc-editor.org/rfc/rfc8705 · RFC 9421 https://www.rfc-editor.org/rfc/rfc9421
- RFC 10017 browser-based apps https://www.rfc-editor.org/rfc/rfc10017 · RFC 8252 native apps https://www.rfc-editor.org/rfc/rfc8252
- OIDC Core 1.0 https://openid.net/specs/openid-connect-core-1_0.html · OIDC Back-Channel Logout https://openid.net/specs/openid-connect-backchannel-1_0.html
- NIST SP 800-63B-4 https://pages.nist.gov/800-63-4/sp800-63b.html · SP 800-63C-4 https://pages.nist.gov/800-63-4/sp800-63c.html
- OWASP ASVS 5.0 (V6–V10) https://github.com/OWASP/ASVS/tree/master/5.0/en · OWASP API Security Top 10 2023 https://owasp.org/API-Security/editions/2023/en/0x11-t10/
- Microsoft: migrate off email claim https://learn.microsoft.com/en-us/entra/identity-platform/migrate-off-email-claim-authorization
- gesdinet/jwt-refresh-token-bundle v3.0.0 README (cookie, logout, block_previous_jwt, max_tokens_per_user) https://github.com/markitosgv/JWTRefreshTokenBundle/blob/v3.0.0/README.md
