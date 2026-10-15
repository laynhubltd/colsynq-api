# v4.1 Research Notes: Auth (JWT/OAuth2), Multi-tenant Identity, Instant Tenant Onboarding

Target stack: Symfony 8.1 / API Platform 5, PostgreSQL database-per-tenant, Nigerian cooperative finance SaaS (Colsynq).
Date: 2026-10-14. Sources are primary where possible (IETF RFCs, symfony.com, vendor docs). Items marked **[unverified]** came from secondary sources or were inferred; check before relying on them.

---

## Part A. JWT and OAuth2 in Symfony 8

### A.1 Standards baseline (what "standard" means in 2026)

| Standard | What it fixes for us | URL |
|---|---|---|
| RFC 9700 (BCP 240) OAuth 2.0 Security BCP, Jan 2025 | Password grant (ROPC) and implicit grant are deprecated / MUST NOT be used. Auth code + PKCE (S256) for all clients, incl. confidential. Refresh tokens for public clients must be rotated or sender-constrained. Exact redirect URI matching. | https://www.rfc-editor.org/rfc/rfc9700.html |
| RFC 9068 JWT profile for access tokens | Header `typ: at+jwt`; required claims `iss, exp, aud, sub, client_id, iat, jti`; optional `scope`, `auth_time, acr, amr` (fixed across refreshed tokens from the same auth event); `roles/groups/entitlements` names from RFC 7643. RS validation: check typ, iss exact match, aud contains self, signature, exp (few minutes skew). | https://www.rfc-editor.org/rfc/rfc9068.html |
| RFC 8725 JWT BCP | Pin one alg per key, reject `none`, validate every crypto op, validate iss/sub/aud, explicit `typ`, do not trust claims blindly (esp. for DB lookups), sanitise `kid`, allowlist `jku/x5u`. | https://www.rfc-editor.org/rfc/rfc8725.html |
| RFC 7636 PKCE | `code_challenge` S256; AS must enforce for public clients. | https://www.rfc-editor.org/rfc/rfc7636.html |
| RFC 9126 PAR | Push the authorization request to the AS first; `request_uri` returned; prevents tampering with redirect/scope. Nice-to-have for mobile; required in FAPI profiles. | https://www.rfc-editor.org/rfc/rfc9126.html |
| RFC 9449 DPoP | Proof JWT in `DPoP` header: `typ: dpop+jwt`, `jwk` header, claims `jti, htm, htu, iat`, plus `ath` (SHA-256 of access token) when calling an RS, `nonce` when the server issued one. Token response `token_type: DPoP`; access token carries `cnf.jkt` (JWK thumbprint). Refresh tokens of public clients are bound to the DPoP key. Server nonce via `DPoP-Nonce` header + `use_dpop_nonce` error (400 at AS, 401 at RS). RS must verify 8 steps (single header, typ/alg/sig, no private key in jwk, htm/htu match, nonce/iat window, ath, cnf.jkt equals thumbprint, jti replay). Bind the auth code with `dpop_jkt` or DPoP header on PAR. | https://www.rfc-editor.org/rfc/rfc9449.html |
| RFC 8693 Token Exchange | `grant_type=urn:ietf:params:oauth:grant-type:token-exchange`, `subject_token`, `actor_token`, `audience/resource`, `act`/`may_act` claims; delegation vs impersonation. Use case: BFF/gateway swaps a user token for a narrower downstream-audience token. | https://www.rfc-editor.org/rfc/rfc8693.html |
| RFC 10017 (BCP 212) OAuth 2.0 for Browser-Based Apps, Aug 2026 | Patterns from most to least secure: **BFF** (confidential client on the server, tokens never reach the browser, cookie session, Secure+HttpOnly cookies, CSRF protection, proxy allowlist) — strongly recommended for business/sensitive/personal data; **Token-Mediating Backend**; **Browser-based OAuth client** (public client, PKCE, rotated/sender-constrained RT, least secure, DPoP does not fully save it). | https://datatracker.ietf.org/doc/draft-ietf-oauth-browser-based-apps/ |
| RFC 8252 OAuth for Native Apps | Native apps MUST NOT use embedded web views; use system browser / in-app browser tab (ASWebAuthenticationSession, Android Custom Tabs); PKCE; custom scheme or claimed https (App Links / Universal Links) redirect. AppAuth is the reference SDK. | https://www.rfc-editor.org/rfc/rfc8252.html , https://openid.net/code/AppAuth |
| RFC 9470 Step-Up | RS returns 401 `WWW-Authenticate: Bearer error="insufficient_user_authentication", acr_values="...", max_age=N`; client re-authorizes with `acr_values`/`max_age`; `max_age` is enforceable (forces re-auth), `acr_values` is advisory (RS must re-check `acr` claim). `auth_time` must not change on refresh. | https://datatracker.ietf.org/doc/rfc9470/ |
| RFC 7009 Revocation / RFC 7662 Introspection | Revocation endpoint (`token`, `token_type_hint`); introspection for opaque tokens. | https://www.rfc-editor.org/rfc/rfc7009.html , https://www.rfc-editor.org/rfc/rfc7662.html |
| RFC 8414 AS metadata / OIDC Discovery | `/.well-known/oauth-authorization-server` and `/.well-known/openid-configuration` with `jwks_uri`. Symfony `oidc` handler can consume discovery. | https://www.rfc-editor.org/rfc/rfc8414.html , https://openid.net/specs/openid-connect-discovery-1_0.html |
| OIDC Core 1.0 | ID token, `nonce`, `auth_time`, `acr`, `amr`, `email_verified` (5.7: do not use `email` as identity key unless `email_verified` is true), `sub` is the only stable identifier within an issuer. | https://openid.net/specs/openid-connect-core-1_0.html |
| NIST SP 800-63B-4 | AAL1: reauth ≤30 days. AAL2: reauth ≤24h, inactivity ≤1h, must offer at least one phishing-resistant option. AAL3: reauth ≤12h, inactivity ≤15min, phishing-resistant required, syncable (cloud-synced) passkeys NOT allowed at AAL3 (ok at AAL2). Passwords: ≥15 chars if single factor (≥8 inside MFA), no composition rules, blocklist/breach check, no periodic expiry. SMS/PSTN OTP is "restricted" (allowed with warnings/alternatives); email OTP is NOT an acceptable out-of-band authenticator; "approve-only" push prompts no longer acceptable. | https://pages.nist.gov/800-63-4/sp800-63b.html |
| OWASP ASVS 5.0 (V2 Authentication, V3 Session, V8 Authorization, V14 Config) and cheat sheets | Credential lifecycle (rotate/revoke), rate limiting/lockout, hashed stored secrets, JWT cheat sheet, Session Management cheat sheet. | https://github.com/OWASP/ASVS/tree/master/5.0 , https://cheatsheetseries.owasp.org/cheatsheets/JSON_Web_Token_for_Java_Cheat_Sheet.html , https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html |
| WebAuthn L3 / Passkeys | Discoverable credentials (resident keys), `userVerification: required`, `authenticatorAttachment` platform/cross-platform, conditional UI (`autocomplete="username webauthn"`), backup eligibility/backup state flags (BE/BS) tell you if the passkey is synced. | https://www.w3.org/TR/webauthn-3/ , https://passkeys.dev |

### A.2 Symfony 8 building blocks

**Native `access_token` authenticator** (symfony.com/doc/8.1/security/access_token.html):
- `security.firewalls.api.access_token.token_handler: App\Security\AccessTokenHandler` where the handler implements `AccessTokenHandlerInterface::getUserBadgeFrom(string $accessToken): UserBadge`. You MUST validate signature + `sub, iat, nbf, exp` yourself for self-contained tokens.
- `token_extractors`: `header` (default `Authorization: Bearer`), `query_string`, `request_body`, or custom `AccessTokenExtractorInterface` service. Docs say avoid query/body (logged).
- Built-in handlers: `oidc` (local JWS/JWE validation via `web-token/jwt-library`; options `algorithms`, `keyset` OR `discovery.base_uri` (+ `cache.id`), `audience`, `issuers`, `claim` (default `sub`), `encryption.{enabled,enforce,algorithms,keyset}`; multiple discovery base_uris allowed; **new in 8.1**: `discovery.enforce_key_usage_verification` (default true, only `use: sig` / `key_ops: verify` keys accepted)), `oidc_user_info` (calls userinfo endpoint each request; options `base_uri`, `discovery`, `claim`, `client`), `cas`.
- User loading: `UserBadge($identifier, fn => ...)` closure lets you skip the user provider on stateless firewalls; or implement `AttributesBasedUserProviderInterface::loadUserByIdentifier(string $id, array $attributes = [])` to build the user from claims.
- `php bin/console security:oidc:generate-token user --firewall=api --algorithm=ES256 --issuer=...` for tests.
- No built-in revocation/denylist: do it in your handler (check `jti` against a revocation table / cache).
- URL: https://symfony.com/doc/8.1/security/access_token.html ; blog: https://symfony.com/blog/new-in-symfony-7-1-improved-access-token-authenticator

**LexikJWTAuthenticationBundle** (https://symfony.com/bundles/LexikJWTAuthenticationBundle/current/index.html): issues + validates JWTs from PEM keys (HS/RS/ES256..512, default RS256), lcobucci/jwt encoder, optional web-token bridge for JWK/JWKSet. Good for "I just want a /login_check that returns a JWT". It is a token *issuer+validator*, not an OAuth2 AS: no auth-code/PKCE, no DPoP, no OIDC provider, no JWKS endpoint out of the box, `kid` support **[unverified]**. Refresh tokens need gesdinet/jwt-refresh-token-bundle. Verdict: fine for a mono-client API; not the standard for a multi-client (web BFF + mobile + third-party) SaaS.

**league/oauth2-server-bundle** (https://github.com/thephpleague/oauth2-server-bundle , core https://oauth2.thephpleague.com/): full RFC 6749 AS. Grants: client_credentials, refresh_token, authorization_code (PKCE, `require_code_challenge_for_public_clients: true` default), device_code (off by default), password and implicit (on by default in 1.x, will default to false in 2.0 — turn them OFF). Config keys: `authorization_server.private_key`, `private_key_passphrase`, `encryption_key` (+ `encryption_key_type`), `access_token_ttl` (PT1H), `refresh_token_ttl` (P1M), `auth_code_ttl` (PT10M), `enable_*_grant`, `persist_access_token` (true), `revoke_refresh_tokens` (true = rotation), `resource_server.public_key`, `resource_server.jwt_leeway`, `scopes.available/default`, `role_prefix` (`ROLE_OAUTH2_`), `persistence.doctrine.{entity_manager,table_prefix}` (default prefix `oauth2_`). Tables: `oauth2_client`, `oauth2_access_token`, `oauth2_refresh_token`, `oauth2_authorization_code`. Firewall: `oauth2: true` authenticator; events `AuthorizationRequestResolveEvent` (approve/deny + set user, where you render consent or auto-approve first-party), `UserResolveEvent`; command `league:oauth2-server:create-client`. Access tokens are RS256 JWTs signed with the configured private key (`aud` = client id, `sub` = user id, `scopes`, `jti`) **[verify claim names vs RFC 9068; league does not emit `typ: at+jwt` or `client_id` natively — wrap/extend `AccessTokenEntity`/`Bearer`/`DPoP` response type]**. Native DPoP support: **none** in league core as of 9.x **[unverified]** — implement as a custom `ResponseTypeInterface` + request validator, or run DPoP validation in your own `AccessTokenHandler`.
- The bundle has no OIDC `id_token` either; add it via the `AuthorizationRequestResolveEvent`/custom response type or use an OIDC layer (e.g. `spomky-labs/oidc` **[unverified maintenance]**) — or keep OIDC-provider duty minimal (first-party only).

**knpuniversity/oauth2-client-bundle** (https://github.com/knpuniversity/oauth2-client-bundle): RP side; `knpu_oauth2_client.clients.<name>: { type: google|azure|generic, client_id, client_secret, redirect_route, use_state: true }`; `OAuth2Authenticator` base class; `fetchUserFromToken()`. PKCE only via `keycloak_pkce` type; no id_token/nonce validation (Google type has `use_oidc_mode`). Verdict: convenient but not an OIDC-certified RP; for Google/Microsoft prefer a small custom RP using `web-token` to validate the `id_token` (`iss, aud, nonce, exp, email_verified`) or the Symfony `oidc` handler when the *mobile app* obtains the id_token itself.

**web-auth/webauthn-symfony-bundle** (https://webauthn-doc.spomky-labs.com/symfony-bundle): `webauthn.controllers.request.login.options_path` (`/login/webauthn/options`) and result handled by a custom authenticator extending `WebauthnAuthenticator` returning `WebauthnPassport(WebauthnBadge(host, assertion))`; `IS_USER_PRESENT`/`IS_USER_VERIFIED` attributes for access_control; conditional UI via `useBrowserAutofill: true`; options/challenge are stored between the two calls (session by default; for a stateless API store the challenge keyed by an opaque id in cache **[inference]**). Registration docs incomplete ("TO BE WRITTEN") — use `Webauthn\PublicKeyCredentialCreationOptions` with `residentKey: required`, `userVerification: required`.

**scheb/2fa** (https://symfony.com/bundles/SchebTwoFactorBundle/current/api.html): REQUIRES a stateful (session) firewall; its API how-to assumes `json_login` + session. It does not fit a stateless bearer API. Use it only on the BFF/auth.colsynq.app session firewall, or implement MFA as a step in the AS login flow (recommended: MFA lives in the AS login UI, so every client gets it for free).

### A.3 Recommended topology for Colsynq

```
auth.colsynq.app   = Symfony app acting as OAuth2 AS + OIDC OP (league/oauth2-server-bundle + custom id_token/DPoP/JWKS)
                      - /authorize, /token, /par, /revoke, /introspect, /userinfo, /.well-known/openid-configuration, /.well-known/jwks.json
                      - login UI (password + passkey + TOTP step), consent auto-approved for first-party clients
                      - RP for Google/Microsoft: /oidc/callback/{provider}  (central callback, one per provider)
api.colsynq.app    = API Platform resource server; Symfony `access_token` authenticator + custom handler (JWS ES256 + DPoP check + jti denylist)
app.colsynq.app    = SPA + BFF (RFC 10017 BFF). BFF is a confidential client of auth.colsynq.app; holds AT/RT server-side; browser only has `__Host-session` cookie
mobile             = public client, auth code + PKCE + PAR + DPoP in ASWebAuthenticationSession / Custom Tabs (AppAuth)
```

Decisions:
1. **Run a real AS** (league) rather than a bespoke `/login` that returns a JWT. Reasons: RFC 9700 forbids password grant; you have three client types; the AS gives you consent-free first-party auth-code+PKCE, refresh rotation, client_credentials for service accounts, revocation, and a single place for MFA/passkeys/step-up. Lexik alone would force the password grant pattern.
2. **Access token format**: JWS, `alg: ES256`, header `kid` + `typ: at+jwt`, claims per RFC 9068 (`iss=https://auth.colsynq.app`, `aud=["https://api.colsynq.app"]`, `sub=user ULID`, `client_id`, `scope`, `exp` (10–15 min), `iat`, `jti`, `auth_time`, `acr` (`urn:colsynq:aal1|aal2`), `amr` (`["pwd"]`, `["pwd","otp"]`, `["webauthn","user"]`), `cnf.jkt` when DPoP, plus `tid` (active tenant) and `mid` (membership id) as private claims). Keys stored in a `signing_keys` table (or KMS), published at `/.well-known/jwks.json` with `use: sig`, rotate by adding a new `kid` and keeping the old one until all tokens expire.
3. **Resource server validation** (api): Symfony `access_token` with custom `AccessTokenHandler` (not the built-in `oidc` handler, because it cannot check DPoP/`cnf.jkt`, `jti` denylist, or tenant claim). Load keys from the local `signing_keys` table / cached JWKS. Return `UserBadge(sub, closure)` producing a `TenantUser` with `tenantId`, `membershipId`, `role`, `acr`, `auth_time`.
4. **Refresh tokens**: opaque 256-bit random, store `sha256(token)` only, one-time use with rotation and family-reuse detection (reuse → revoke whole family), absolute lifetime (e.g. 30 days mobile, 8h BFF sliding/24h absolute per AAL2), bound to DPoP key for mobile (`jkt` column).
5. **Revocation**: RFC 7009 `/revoke` for RT (delete family) and AT (`jti` into `revoked_jti` with TTL = remaining exp; RS checks cache). Logout-everywhere = bump `users.token_version`/`sessions` and revoke all families.
6. **Mobile**: auth code + PKCE(S256) + PAR + DPoP, system browser (ASWebAuthenticationSession with `prefersEphemeralWebBrowserSession` decided per UX; Android Custom Tabs via AppAuth), claimed https redirect (`https://auth.colsynq.app/app/callback` Universal/App Link) with custom-scheme fallback. DPoP key in Secure Enclave/Android Keystore (non-extractable).
7. **Web SPA**: BFF pattern. Cookie `__Host-colsynq_sess` (Secure, HttpOnly, SameSite=Lax, Path=/, no Domain), server session row holds AT/RT; BFF proxies `/api/*` adding `Authorization: Bearer`; CSRF via double-submit or `Origin`/`Sec-Fetch-Site` check on unsafe methods. SPA never sees tokens.
8. **Third-party IdP (Google/Microsoft)**: auth.colsynq.app is the RP. One central callback per provider; the AS runs the upstream auth-code+PKCE+nonce flow, validates `id_token` (iss, aud, nonce, exp, iat), reads `sub`, `email`, `email_verified`. **Link only by (`iss`,`sub`)** in `user_identities`. If no identity row: if `email_verified===true` AND a user with that email exists → require link-on-login (user must authenticate with existing credential or an email OTP, then link); if `email_verified` false/missing → treat as unverified, never auto-link, create/claim only after our own email verification. Microsoft often omits `email_verified` → treat as false.
9. **Service accounts / API keys**: primary = OAuth2 `client_credentials` (per-tenant `oauth2_client` with `confidential=true`, `client_secret` hashed with argon2id, scopes limited, short-lived AT with `sub = client_id`, `tid`). Also offer **API keys** for integrators who cannot do OAuth: format `csq_live_<keyid>_<secret>` (prefix for secret scanning + lookup), store `sha256(secret)` (128+ bits entropy; fast hash is fine), `scopes[]`, `tenant_id`, `last_used_at`, `expires_at`, rotation with overlap (two active keys, 24–72h grace), IP allowlist optional. The API-key authenticator maps to the same `TenantUser`/scope model (another `token_extractors` on the same `access_token` firewall, or a second firewall).
10. **MFA / passkeys / step-up**: MFA happens inside the AS login. Factors: passkeys (WebAuthn, UV required) as primary phishing-resistant AAL2 factor; TOTP as fallback; SMS only as "restricted" fallback with warnings (NIST), never email OTP as a second factor. Tenant policy `mfa_policy: off|optional|required|required_phishing_resistant`. `acr`/`amr`/`auth_time` are written into AT and ID token. Step-up: API resources with `#[StepUp(acr: 'aal2', maxAge: 300)]` → voter returns 401 `WWW-Authenticate: Bearer error="insufficient_user_authentication", error_description="...", acr_values="urn:colsynq:aal2", max_age=300` (RFC 9470); client re-runs `/authorize?prompt=login&acr_values=...&max_age=300` (mobile) or BFF redirects (web); AS re-authenticates with a strong factor, issues new tokens with fresh `auth_time`/`acr`; `auth_time` is preserved across refreshes. Session timeouts: AAL2 inactivity 1h (BFF/mobile RT sliding), absolute 24h (RT family max age), AAL1 30 days (only for low-risk read scopes if you choose to offer it).

### A.4 Tables / fields (control-plane DB, "central")

```
users                      id ULID PK, email citext UNIQUE, email_verified_at, password_hash (argon2id, nullable), display_name, phone, phone_verified_at,
                           status (active|suspended|deleted), token_version int, locale, created_at, updated_at, last_login_at
user_identities            id, user_id FK, issuer (varchar), subject (varchar), email_at_link, email_verified_at_link bool, raw_claims jsonb,
                           created_at, last_login_at, UNIQUE(issuer, subject)
user_credentials_password  (or in users) password_hash, password_updated_at, must_change bool, breach_checked_at
webauthn_credentials       id, user_id FK, credential_id bytea UNIQUE, public_key bytea, aaguid uuid, sign_count bigint, transports text[],
                           backup_eligible bool, backup_state bool, uv_supported bool, nickname, created_at, last_used_at, revoked_at
totp_secrets               id, user_id FK, secret_encrypted bytea, confirmed_at, last_used_counter, created_at
recovery_codes             id, user_id FK, code_hash, used_at
email_verification_tokens  id, user_id, token_hash, purpose (verify|link|reset|invite), expires_at, consumed_at
auth_events (audit)        id, user_id, tenant_id?, event (login|logout|mfa_ok|mfa_fail|link|refresh|revoke|step_up), amr text[], acr, ip inet, user_agent,
                           client_id, dpop_jkt, created_at   (append-only)
oauth2_client              identifier (client_id) PK, name, secret_hash (nullable => public), redirect_uris text[], grants text[], scopes text[],
                           active bool, allow_plain_text_pkce bool(false), first_party bool, tenant_id (nullable: service accounts), require_dpop bool,
                           created_at, rotated_at
oauth2_authorization_code  identifier PK (hash), client_id, user_id, scopes, expiry, revoked bool, code_challenge, code_challenge_method, nonce,
                           dpop_jkt (nullable), redirect_uri, acr_requested, max_age_requested, tenant_id (requested)
oauth2_access_token        jti PK, client_id, user_id, scopes, expiry, revoked bool, dpop_jkt, tenant_id, membership_id, session_id, acr, auth_time
                           (persist_access_token may be false if you rely on revoked_jti cache; keep for audit)
oauth2_refresh_token       id PK (hash of opaque token), access_token_jti, family_id, parent_id (nullable), client_id, user_id, session_id, expiry,
                           absolute_expiry, revoked_at, reused_at, dpop_jkt, created_at
revoked_jti (cache+table)  jti PK, expires_at
signing_keys               kid PK, alg ('ES256'), public_jwk jsonb, private_key_encrypted bytea (or KMS ref), status (next|active|retiring|retired),
                           not_before, not_after, created_at
dpop_nonces (cache)        nonce -> issued_at  (short TTL)
dpop_jti_seen (cache)      jti -> ts (TTL = iat window, replay protection)
auth_sessions (BFF/AS)     id ULID PK, user_id, client_id, tenant_id (active), membership_id, acr, auth_time, amr text[], ip, user_agent,
                           last_seen_at, absolute_expires_at, idle_expires_at, revoked_at, refresh_family_id
                           (BFF cookie `__Host-colsynq_sess` = opaque session id, or a signed/encrypted id; AS may reuse symfony session for login UI)
api_keys                   id, tenant_id, created_by_user_id, name, key_id (public prefix part) UNIQUE, secret_hash (sha256), scopes text[],
                           ip_allowlist cidr[], status (active|rotating|revoked), expires_at, last_used_at, replaced_by_id, created_at, revoked_at
step_up_challenges (opt.)  id, session_id, required_acr, max_age, issued_at, satisfied_at
```

Identity providers config: `identity_providers (id, slug google|microsoft|<tenant-saml>, issuer, client_id, client_secret_enc, scopes, tenant_id nullable for per-tenant enterprise SSO, enabled)`.

### A.5 Flows (sequence steps)

**1. Sign-in, web (BFF)**
1. SPA hits `GET app.colsynq.app/auth/login?return=/dashboard` (BFF).
2. BFF creates `state`, `nonce`, PKCE verifier; stores them in a short-lived pre-auth cookie/session; redirects (or PAR → `request_uri`) to `auth.colsynq.app/authorize?response_type=code&client_id=web-bff&redirect_uri=https://app.colsynq.app/auth/callback&scope=openid profile offline_access api&code_challenge=…&state=…&nonce=…`.
3. AS: existing AS session? If not → login page: email → (passkey conditional UI | password) → MFA step per user/tenant policy → `auth_time`, `amr`, `acr` recorded. First-party client → consent auto-approved (`AuthorizationRequestResolveEvent::resolveAuthorization(true)`).
4. AS redirects `…/auth/callback?code=…&state=…`.
5. BFF verifies `state`, posts `/token` (client_secret_basic + `code_verifier`), gets AT (10–15m), RT (rotating), ID token; validates ID token `nonce`.
6. BFF creates `auth_sessions` row (AT/RT encrypted at rest), sets `Set-Cookie: __Host-colsynq_sess=<id>; Secure; HttpOnly; SameSite=Lax; Path=/`, redirects to `/dashboard`.
7. SPA calls `/api/...` on same origin; BFF looks up session, auto-refreshes AT if <60s left, forwards with `Authorization: Bearer`. CSRF: reject unsafe methods unless `Sec-Fetch-Site: same-origin` or custom header `X-Requested-With`.
8. Tenant switch: SPA `POST /auth/switch-tenant {tenant_id}` → BFF calls AS `/token` with `grant_type=refresh_token&tenant_id=…` (custom param) or token exchange (RFC 8693, `resource=api`, `tenant_id`) → new AT with `tid`/`mid`/`role` claims; session row updated. AS checks membership exists and is active, and tenant MFA policy; if the target tenant requires AAL2 and session `acr` is lower → return `insufficient_user_authentication` → step-up.

**2. Sign-in, mobile (PKCE + DPoP)**
1. App generates DPoP keypair (Secure Enclave/Keystore), PKCE verifier/challenge, `state`, `nonce`.
2. `POST /par` with DPoP header (binds `dpop_jkt`), gets `request_uri` (60s).
3. Open `ASWebAuthenticationSession` / Custom Tab at `/authorize?client_id=mobile&request_uri=…`.
4. AS login + MFA as above; redirects to `https://auth.colsynq.app/app/callback?code=…&state=…` (Universal Link) → app.
5. `POST /token` with `grant_type=authorization_code, code, code_verifier, client_id=mobile` + `DPoP` proof (htm POST, htu /token, jti, iat). AS: if no nonce yet → 400 `use_dpop_nonce` + `DPoP-Nonce`; app retries. AS checks PKCE, `dpop_jkt` matches proof thumbprint, issues `token_type: DPoP`, AT with `cnf.jkt`, RT bound to jkt.
6. API calls: `Authorization: DPoP <at>` + `DPoP: <proof with ath>`; RS validates per RFC 9449 §4.3 and `cnf.jkt`.

**3. Refresh**
1. Client `POST /token grant_type=refresh_token&refresh_token=…` (+ DPoP proof for mobile; client secret for BFF).
2. AS: hash → lookup; if `revoked_at` or `reused_at` set or `expiry` passed → `invalid_grant` AND if the token was already rotated (`reused`) → revoke the whole `family_id` + audit `refresh_reuse`.
3. Check `absolute_expiry` (family max) and idle window; check user `status`, `token_version`, membership still active.
4. Mark old RT `revoked_at=now`, insert new RT with same `family_id`, `parent_id=old`; issue new AT keeping `auth_time/acr/amr` from the session.

**4. Revoke / logout**
1. `POST /revoke token=<rt> token_type_hint=refresh_token` (RFC 7009) → revoke family, mark session revoked, add live AT `jti`s to `revoked_jti` (TTL remaining).
2. BFF logout: delete session row, clear cookie, call `/revoke`, optionally RP-initiated logout `/logout?id_token_hint`.
3. "Sign out everywhere": `users.token_version++`, revoke all families; RS compares `token_version` claim (optional) or relies on short AT TTL.

**5. Third-party IdP login (Google/Microsoft) at the AS**
1. User on AS login page clicks "Continue with Google" → AS stores `state`, `nonce`, PKCE in AS session; redirects to Google `/authorize` (scope `openid email profile`).
2. Google → `auth.colsynq.app/oidc/callback/google?code&state`.
3. AS exchanges code (client secret + PKCE), validates `id_token` (sig via Google JWKS, `iss`, `aud`, `nonce`, `exp`, `iat`, `azp`).
4. Lookup `user_identities(issuer, sub)`:
   - found → login as that user; update `last_login_at`.
   - not found & `email_verified===true` & user with same email exists → "link-on-login": show "An account with this email exists, sign in to link" → user authenticates with existing factor (password/passkey) → insert identity row.
   - not found & no user → create user (email_verified_at = now only if `email_verified===true`, else send our own verification), insert identity.
   - `email_verified` missing/false & user exists → do NOT link; offer "sign in with your password, then link in settings".
5. Continue the original `/authorize` request (resume stored auth request) → code → client.
6. If the IdP is a tenant's enterprise SSO (per-tenant SAML/OIDC), the `/authorize` request carries `tenant_id`/`login_hint` domain → AS picks the tenant's `identity_providers` row (home-realm discovery by email domain) and after login JIT-creates the membership with the tenant's default role.

**6. Step-up**
1. API action marked `requires acr=aal2, max_age=300` (e.g. approve loan disbursement, change payout account).
2. Voter inspects token `acr`/`auth_time`; if insufficient → 401 `WWW-Authenticate: Bearer error="insufficient_user_authentication", error_description="A strong, recent authentication is required", acr_values="urn:colsynq:aal2", max_age=300`.
3. Mobile: re-run `/authorize` with `prompt=login&acr_values=urn:colsynq:aal2&max_age=300` (PAR+DPoP, same session). Web: BFF intercepts 401, redirects browser to `/authorize?...prompt=login&acr_values&max_age`.
4. AS forces a fresh strong factor (passkey UV / TOTP); issues new tokens with new `auth_time`, `acr=aal2`, `amr` incl. the factor; session row updated.
5. Client retries the API call; voter passes; `auth_events(step_up)` written.

**7. API key / service account**
- Client credentials: tenant admin creates `oauth2_client` (confidential, scopes, `tenant_id`) → secret shown once → `POST /token grant_type=client_credentials&scope=…` (client_secret_basic or `private_key_jwt`) → AT with `sub=client_id`, `tid`, `scope`, 15–60 min → API calls as Bearer (DPoP optional).
- API key: `POST /tenants/{id}/api-keys {name, scopes, expires_at}` → returns `csq_live_<keyid>_<secret>` once; store `sha256(secret)`. Request: `Authorization: Bearer csq_live_…` → extractor recognises prefix → lookup `key_id` → constant-time compare hash → check status/expiry/IP/scopes → `UserBadge('apikey:'.$id)` → `TenantUser` with `actorType=service`. Rotation: `POST /api-keys/{id}/rotate` creates replacement, old key `status=rotating` until `grace_until` (24–72h) then revoked. `last_used_at` updated async (Messenger) to avoid write on every request.

---

## Part B. Multi-tenant identity patterns

### B.1 How vendors do it (evidence)

| Vendor | Account model | Roles per membership | Duplicate-email at sign-up | Multi-tenant flag | Switching | Per-tenant MFA | Invites | SCIM/JIT | URL |
|---|---|---|---|---|---|---|---|---|---|
| Auth0 Organizations | One user (per connection) → many orgs via membership; `org_id` claim in tokens; org selected at `/authorize` (`organization` param) | **Multiple** roles per member per org (`POST /organizations/{id}/members/{uid}/roles` takes array); org roles are separate from global RBAC roles and apply only when logged in through that org | Invitation requires login/sign-up with the invited email; accepting marks email verified | membership rows | re-authorize with `organization=` (silent auth can fail if org needs MFA) | Not native: tenant-level MFA + Post-Login Action reading org metadata to call `api.multifactor.enable` | invitation ticket `?invitation=&organization=`; expires; roles pre-assigned | Enterprise connections per org; SCIM via Okta/Entra connections | https://auth0.com/docs/manage-users/organizations/organizations-overview , https://auth0.com/docs/manage-users/organizations/configure-organizations/invite-members |
| WorkOS AuthKit | One user → many orgs via `organization_membership` | **Exactly one role by default**; "multiple roles" is an opt-in environment setting; role + permissions in JWT claims; default role `member` | sign-up with existing email → sign-in path; invitations tie to email | memberships list | `organization_id` on session; switch = new session | per-organization MFA/SSO policies in Admin Portal **[partly unverified]** | invitations with role; expiry | Directory Sync (SCIM) group→role mapping overrides SSO/manual; "use one role source per org" | https://workos.com/docs/authkit/roles-and-permissions |
| Clerk Organizations | One user → many orgs; "active organization" in session token (`org_id`, `org_role`) | **One role per membership** (`role` string on `OrganizationMembership`, `org:admin`, `org:member`); permissions attached to role | invitations by email; verified domains auto-invite/suggest | memberships | `setActive({organization})`; token reflects active org; background requests must use `getToken()` | Enterprise SSO per org; MFA app-level **[unverified per-org]** | yes | SCIM via Enterprise connections | https://clerk.com/docs/guides/organizations/overview , https://clerk.com/docs/reference/backend/organization/create-organization-membership |
| Microsoft Entra B2B | Home identity + **guest user object** (`#EXT#`) in each resource tenant | roles/groups per tenant on the guest object | invitation redemption; email OTP fallback | guest object per tenant | tenant switcher; re-auth to home tenant (since 2025 guests redirected to home tenant sign-in) | resource tenant Conditional Access can require its own MFA, or **trust MFA claims** from the home tenant via cross-tenant access settings | invitation API | entitlement management | https://learn.microsoft.com/en-us/entra/external-id/what-is-b2b |
| Slack | One Slack account per email; workspace membership; "sign in to join" when invited | one role per workspace membership (member/admin/owner/guest) | email code sign-in; existing account reused | workspace switcher | per-workspace 2FA requirement | invite link / email | SCIM (Enterprise) | https://slack.com/help/articles/212675257 **[not fetched]** |
| GitHub | one account; org membership; invite by username or **verified** email, 7-day expiry; org can require 2FA before accept | one base role (member/owner) + team-based permissions | n/a (account first) | org list | per-org 2FA enforcement | invitations | SCIM (Enterprise) | https://help.github.com/articles/inviting-users-to-join-your-organization |
| Atlassian | one Atlassian account (email) → many sites/orgs | product roles per site | account exists → sign-in | site switcher | org-level auth policies (Atlassian Guard) | invite; approved domains | SCIM (Guard) | https://community.atlassian.com/forums/Atlassian-Account-questions/Multiple-Jira-sites-and-licences-under-one-Atlassian-Admin-Page/qaa-p/1525495 |

Account linking rule (industry consensus): key on `(iss, sub)`, email is profile data; never merge on email unless `email_verified === true` (treat missing as false); prefer link-on-login. Sources: Ory https://www.ory.sh/blog/secure-account-linking-iam-sso-oidc-saml , WorkOS https://workos.com/blog/stop-using-email-as-a-primary-key , Budibase advisory GHSA-hp6v-6jw7-gv2f (CVE-2026-73302), OIDC Core §5.7.

### B.2 Evaluation: "exactly ONE role per tenant membership"
- Default at WorkOS and Clerk; Slack/GitHub base roles are also single. Auth0 allows many but most apps use one.
- Pros: simple claims (`role` string), simple UI, trivially answers "what can this person do here", maps cleanly to cooperative governance (member, treasurer, secretary, chair, auditor, staff). Permission granularity comes from the **role → permissions** table, not from stacking roles.
- Cons: a person who is both "treasurer" and "loan committee member" needs either a composite role or per-feature permission overrides. Solve with (a) a small set of composite roles, (b) optional `membership_permissions` overrides (grant/deny) kept rare, or (c) a flag column for committee seats. Keep "multiple roles" as a future toggle (WorkOS-style env setting) rather than modelling it now.
- Recommendation: **one role per membership + optional permission overrides**, role is a tenant-scoped FK (`roles.tenant_id` nullable for system roles), permissions are code-defined slugs.

### B.3 Minimal data model (central DB)

```
tenants                 id ULID, slug UNIQUE, name, status (provisioning|ready|sandbox|verified|suspended|closed), kyb_status, db_name, region,
                        mfa_policy (off|optional|required|required_phishing_resistant), session_max_age, allowed_email_domains text[], created_at
tenant_memberships      id ULID, tenant_id FK, user_id FK, role_id FK, status (invited|active|suspended|left), member_profile_id (tenant-db ref),
                        is_default bool, joined_at, last_active_at, invited_by, UNIQUE(tenant_id, user_id)
roles                   id, tenant_id (nullable = system role), slug, name, is_default bool, UNIQUE(tenant_id, slug)
role_permissions        role_id, permission_slug (PK pair)
membership_permissions  membership_id, permission_slug, effect (allow|deny)   -- rare overrides
tenant_invitations      id, tenant_id, email citext, role_id, invited_by, token_hash UNIQUE, expires_at (7d), accepted_at, accepted_user_id, revoked_at,
                        UNIQUE(tenant_id, email) WHERE accepted_at IS NULL
tenant_identity_providers (per-tenant SSO) id, tenant_id, type (oidc|saml), issuer, client_id, secret_enc, email_domains text[], jit_default_role_id, scim_token_hash
users.active_tenant_id  (last used) -- or keep in session only
```

Flag "belongs to several tenants": derived (`COUNT(memberships) > 1`), expose `memberships[]` in `/me`; the ID token can carry `tenants: [{id, slug, role}]` (keep it short) and the AT carries only the active `tid`/`mid`/`role`.

Sign-up detection flow: `POST /signup {email}` → if user exists → respond generically "check your email" and send "you already have an account — sign in to create/join a workspace" (avoids enumeration); front-end shows "sign in to join". Invitation acceptance: must be logged in as a user whose verified email equals invitation email (GitHub/Auth0 rule) → create membership with invitation's role.

Tenant switching: pick membership → issue new AT via refresh/token-exchange with `tid`; enforce tenant `mfa_policy` at switch time (step-up if needed); the API resolves the tenant DB connection from `tid` claim, never from a user-supplied header alone.

SCIM/JIT: start with JIT (membership created on first SSO login with `jit_default_role_id`), add SCIM 2.0 (RFC 7643/7644) `/scim/v2/Users|Groups` per tenant when enterprise customers need deprovisioning; group→role mapping; SCIM is authoritative over manual role when enabled (WorkOS rule). https://www.rfc-editor.org/rfc/rfc7644.html , https://clerk.com/articles/scim-vs-jit-provisioning-when-to-use-each

---

## Part C. Instantly-ready self-service tenant onboarding

### C.1 Evidence
- AWS SaaS Lens: onboarding must be frictionless, repeatable, fully automated (IaC), shared control plane even for silo tenants; "SaaS OPS" pillar. https://docs.aws.amazon.com/wellarchitected/latest/saas-lens/software-and-architecture-patterns.html , https://docs.aws.amazon.com/wellarchitected/latest/saas-lens/silo-pool-and-bridge-models.html , https://docs.aws.amazon.com/prescriptive-guidance/latest/patterns/tenant-onboarding-in-saas-architecture-for-the-silo-model-using-c-and-aws-cdk.html
- Azure multitenant guidance: control plane separate from tenant workloads; deployment stamps; IaC; automated onboarding expected for Marketplace. https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/approaches/deployment-configuration , https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/considerations/tenant-lifecycle
- Pre-provisioned pool pattern (bridge/schema and silo/db variants): keep N empty, migrated databases; at sign-up claim one (rename/assign), background job replenishes; migrate any idle unit to the current schema version before hand-off. https://www.techinterview.org/post/3233472258/lld-tenant-provisioning/ (secondary)
- PostgreSQL `CREATE DATABASE … TEMPLATE tmpl STRATEGY = WAL_LOG|FILE_COPY` (PG15+). WAL_LOG default, good for small templates (no checkpoint); FILE_COPY faster for large templates but forces 2 checkpoints; template must have no connections (FILE_COPY). Benchmark (Aiven): 97.7s WAL_LOG vs 33.5s FILE_COPY for a large DB. WAL_LOG template corruption bug fixed in minor releases — stay current. For a ~few-MB empty tenant template, `CREATE DATABASE` is sub-second either way. https://www.postgresql.org/docs/current/sql-createdatabase.html , https://aiven.io/community/forum/t/faq-why-is-database-creation-from-templates-slower-for-large-databases-in-postgresql-15/421 , https://boringsql.com/posts/instant-database-clones/
- Neon branches: copy-on-write, seconds, scale-to-zero, API-driven; real DB-per-tenant case (LuBot). Supabase branching is dev/preview oriented. https://neon.com/docs/guides/branching , https://neon.com/blog/inside-lubots-database-per-tenant-architecture
- Symfony Workflow (state_machine, marking_store method, guard/transition/completed events, `audit_trail`, `workflow:dump`). https://symfony.com/doc/8.1/workflow.html
- Messenger retry: `retry_strategy: { max_retries: 3, delay: 1000, multiplier: 2, max_delay: 0, jitter: 0.1 }`, `failure_transport`, `messenger:failed:show|retry`, `DispatchAfterCurrentBusMiddleware` + `doctrine_transaction` middleware; Doctrine transport as outbox table + relay worker; handlers must be idempotent (at-least-once). https://symfony.com/doc/current/messenger.html , https://symfony.com/doc/current/messenger/dispatch_after_current_bus.html , https://dev.to/gabrielanhaia/symfony-messenger-transports-building-a-transactional-outbox-on-top-3op3
- Stripe Connect verification state machine (`requirements.currently_due / eventually_due / past_due / pending_verification / errors`, `current_deadline`, `disabled_reason`, `account.updated` webhook) as the KYB gating model. https://docs.stripe.com/connect/handling-api-verification
- "Provisional onboarding" (limited access until verification completes, with expiry and containment). https://nhimg.org/glossary/provisional-onboarding/

### C.2 Recommended design

Time budget: synchronous path ≤ 2s (claim warm DB + write tenant row + membership + issue tokens). Everything else async with a status endpoint and a "tenant ready" event. No page should wait on provisioning; show the workspace immediately in `provisioning` state with a progress banner only if the warm pool is empty.

Warm pool:
```
tenant_db_pool   id, db_name UNIQUE, server_id, schema_version, status (creating|ready|claimed|failed|retired), created_at, claimed_at, claimed_by_tenant_id
```
- `app:tenant-pool:replenish` (scheduler every minute, and triggered by `TenantDbClaimed`): keep `ready >= 5` (tune to peak sign-ups/hour). Create: `CREATE DATABASE "t_pool_<ulid>" TEMPLATE colsynq_tenant_template STRATEGY = WAL_LOG` (template is empty + migrated; refresh template after each deploy; lock the template from connections), then verify `schema_version`, set `ready`.
- Claim is a single `UPDATE … WHERE status='ready' … LIMIT 1 FOR UPDATE SKIP LOCKED RETURNING`. Rename is optional (keep the pool name; store mapping in `tenants.db_name`).
- Fallback when pool empty: synchronous `CREATE DATABASE` (sub-second on empty template) with a 5s budget; if that fails → async path, tenant stays `provisioning`.
- After each release: migrate pool DBs (`app:tenant-pool:migrate`) or retire and recreate them.

Saga / state machine (Symfony Workflow `state_machine` on `TenantProvisioning`):
```
places: requested → db_claimed → schema_verified → seeded → storage_ready → search_ready → owner_linked → ready
        (+ failed, compensating, rolled_back)
tenant_provisionings   id, tenant_id UNIQUE, state, step_attempts jsonb, last_error, started_at, finished_at, idempotency_key UNIQUE
tenant_provisioning_steps  id, provisioning_id, step, status, attempt, started_at, finished_at, error, payload jsonb  (one row per step attempt, audit)
outbox (= messenger doctrine transport table) messenger_messages with queue_name 'tenant_provisioning'
```
- Each step = one Messenger message, idempotent (check step row before acting; use `ON CONFLICT DO NOTHING`; `CREATE … IF NOT EXISTS`; S3 bucket/prefix creation idempotent), retried with `retry_strategy: { max_retries: 5, delay: 2000, multiplier: 2, max_delay: 60000 }`; `failure_transport: failed` → alert (Slack/email) + dashboard; `messenger:failed:retry` for manual replay.
- Dispatch the first message from the sign-up transaction via the Doctrine transport (outbox), so the tenant row and the job commit together; the relay worker forwards it.
- Compensation on terminal failure: `compensating` → drop/return DB to pool (`retired`), delete storage prefix, mark tenant `failed`, notify ops; user sees "we're finishing setup, we'll email you" (never an error wall).
- `TenantReady` domain event → email, in-app toast (Mercure/websocket), analytics, and unlocks writes; the API returns `503 tenant_provisioning` with `Retry-After` for tenant-scoped write routes until ready (reads of control-plane data are fine).
- Observability: metrics `tenant_pool_ready`, `provisioning_duration_seconds`, `provisioning_failures_total`; alert when pool < 2 or any provisioning > 2 min.

KYB gating (sandbox/limited mode):
```
tenants.verification_status  unverified | pending_review | verified | rejected | action_required
tenant_verification          id, tenant_id, kind (cac_rc|tin|bvn_director|address|bank_account), status, provider_ref, submitted_at, verified_at, rejected_reason, documents jsonb
tenant_capabilities (derived/cached) tenant_id, capability (members_invite|contributions_record|loans_disburse|payouts|api_keys_live|limits jsonb), enabled bool, reason
```
- On sign-up the tenant is immediately `ready` + `verification_status=unverified` → **sandbox/limited mode**: full UI, demo data allowed, member invites capped (e.g. 10), no money-movement, no live API keys, "test" badge; daily/weekly reminders; auto-expire unverified tenants after N days (archive, not delete).
- Submit CAC/TIN/director ID → `pending_review` (automated provider check + manual queue) → `verified` unlocks capabilities; `action_required` lists `currently_due[]` Stripe-style with `deadline`.
- Capabilities are computed from one function (`TenantCapabilityResolver`) and enforced by a voter/attribute on API operations, not scattered `if`s.

---

## Top URLs (primary)
1. https://symfony.com/doc/8.1/security/access_token.html
2. https://www.rfc-editor.org/rfc/rfc9700.html
3. https://www.rfc-editor.org/rfc/rfc9068.html
4. https://www.rfc-editor.org/rfc/rfc9449.html
5. https://datatracker.ietf.org/doc/draft-ietf-oauth-browser-based-apps/ (RFC 10017 / BCP 212)
6. https://datatracker.ietf.org/doc/rfc9470/
7. https://www.rfc-editor.org/rfc/rfc8725.html
8. https://github.com/thephpleague/oauth2-server-bundle
9. https://pages.nist.gov/800-63-4/sp800-63b.html
10. https://workos.com/docs/authkit/roles-and-permissions
11. https://auth0.com/docs/manage-users/organizations/configure-organizations/invite-members
12. https://www.postgresql.org/docs/current/sql-createdatabase.html
13. https://symfony.com/doc/8.1/workflow.html
14. https://symfony.com/doc/current/messenger.html
15. https://docs.stripe.com/connect/handling-api-verification
16. https://learn.microsoft.com/en-us/entra/external-id/what-is-b2b
17. https://www.ory.sh/blog/secure-account-linking-iam-sso-oidc-saml
18. https://webauthn-doc.spomky-labs.com/symfony-bundle/firewall
19. https://symfony.com/bundles/SchebTwoFactorBundle/current/api.html
20. https://neon.com/docs/guides/branching
