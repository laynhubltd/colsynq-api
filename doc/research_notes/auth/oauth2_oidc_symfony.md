# OAuth 2.0 / 2.1 / OIDC for Symfony 8.1 + API Platform 5 + PHP 8.4 — research notes

Researched 2026-10-12. Versions come from Packagist p2 metadata, GitHub source tarballs and the release APIs. Mark legend: **[U]** = unverified syntax or claim, check it before use. Anything without [U] was read from the source, docs or registry listed next to it.

---

## 0. TL;DR / recommendations

| Role | Recommended | Why |
|---|---|---|
| Resource server (validate tokens on API) | Symfony built-in `access_token` authenticator + `oidc` handler with `discovery` | First-party code, JWKS discovery and caching, multiple issuers, JWE. Needs only `web-token/jwt-library` + `symfony/cache` |
| Authorization server (issue tokens) | **Buy/run an IdP** (Keycloak 26.8 / Zitadel v4 / Authentik 2026.8 / Ory Hydra v26 / Auth0) unless the requirements are minimal | league server lacks OIDC, discovery, introspection, revocation, PAR, DPoP, mTLS and private_key_jwt. See §1.4 |
| If you must self-host the AS in-app | `league/oauth2-server-bundle` **v1.2.2** (stable, supports SF 8) on `league/oauth2-server` **9.4.1**; bundle 2.x (SF ^7.4\|^8.1, PHP ^8.2) is still dev-only | Auth code+PKCE, client_credentials, refresh (rotation), device code |
| Social login / OAuth client | `knpuniversity/oauth2-client-bundle` **v2.20.2** + `league/oauth2-client` **2.9.1** + providers | `OAuth2Authenticator`, `OAuth2PKCEClient` |
| Advanced client (private_key_jwt, introspection, JAR, mTLS) | `facile-it/php-openid-client` **1.0.0** (PHP ^8.2) | Spec coverage is far beyond league client |
| SPA frontend | BFF pattern (RFC 10017, BCP, Aug 2026). Keep tokens out of JS | See §5 |
| Financial/partner APIs | FAPI 2.0 Security Profile (Final, Feb 2025) at the IdP: PAR + PKCE + sender-constrained tokens (mTLS or DPoP) + private_key_jwt/mTLS client auth | See §4–5 |

---

## 1. Symfony as an OAuth2 AUTHORIZATION SERVER

### 1.1 Packages and versions (from Packagist, 2026-10-12)

| Package | Latest | Key constraints |
|---|---|---|
| `league/oauth2-server-bundle` | **v1.2.2** (2026-07-26) | php ^8.1; `symfony/framework-bundle` & `security-bundle` `^6.4\|^7.4\|^8.0`; `league/oauth2-server ^9.2` → **works on Symfony 8.1** |
| `league/oauth2-server-bundle` 2.x-dev | unreleased (last commit 2026-10-09) | php ^8.2, Symfony `^7.4\|^8.1`, `lcobucci/jwt ^5.6`, Doctrine ORM 3 / DBAL 4 / DoctrineBundle 2.15+ (UPGRADE-2.x.md). 1.x is in maintenance |
| `league/oauth2-server` | **9.4.1** (2026-06-25) | php `~8.2–~8.5`, `lcobucci/jwt ^5.6`. 9.4.0 dropped PHP 8.1. 9.3.0 added `#[SensitiveParameter]` and a case-insensitive Bearer header |
| `steverhoades/oauth2-openid-connect-server` | v3.0.1 (2024-09-26) | `league/oauth2-server ^8.4.2\|^9.0`, `lcobucci/jwt ^4\|^5`. Stale (no release in 2 years) |
| `lcobucci/jwt` | 5.6.1 (2026-10-10) | |

Sources: https://github.com/thephpleague/oauth2-server-bundle (docs/ dir, branches 1.x & 2.x), https://github.com/thephpleague/oauth2-server/blob/master/CHANGELOG.md, https://packagist.org/packages/league/oauth2-server-bundle

### 1.2 Grants (league/oauth2-server 9.4.1 `src/Grant/`)
`AuthCodeGrant` (PKCE S256/plain verifiers), `ClientCredentialsGrant`, `RefreshTokenGrant`, `DeviceCodeGrant` (RFC 8628, since 9.0), `ImplicitGrant` (legacy), `PasswordGrant` (legacy). Custom grants are possible (bundle doc `implementing-custom-grant-type.md`).

**Not in league server 9.4.1:** OIDC id_token, discovery/JWKS endpoint, RFC 7662 introspection, RFC 7009 revocation endpoint, RFC 8414 metadata, PAR (RFC 9126), DPoP (RFC 9449), mTLS (RFC 8705), `private_key_jwt`/`client_secret_jwt` client auth (RFC 7523), JAR, RFC 9207 `iss` response param. I grepped the 9.4.1 source and found none of these. You build every one of them yourself.

Redirect URIs: `RedirectUriValidator` does **exact match**, with loopback URIs matched excluding the port. This complies with RFC 9700.

**Access-token JWT shape (`AccessTokenTrait::convertToJWT`)**: `aud` = client_id, `jti`, `iat`, `nbf`, `exp`, `sub`, custom claim **`scopes`** (array). There is **no `iss`** and it is **not RFC 9068** (`at+jwt` typ, `scope` string, `client_id`). Consequence: Symfony's `oidc` token handler (which *requires* `issuers` and `audience`) will reject league tokens unless you customise the token entity or add claims. Bundle 2.x adds the `ACCESS_TOKEN_EXTRA_CLAIMS_RESOLVE` event for this. Otherwise protect APIs with the bundle's own `oauth2: true` firewall.

### 1.3 Bundle config (from bundle `docs/index.md` + `src/DependencyInjection/Configuration.php`)

```yaml
# config/packages/league_oauth2_server.yaml
league_oauth2_server:
    authorization_server:
        private_key: '%env(resolve:OAUTH_PRIVATE_KEY)%'      # RSA private key path
        private_key_passphrase: '%env(OAUTH_PASSPHRASE)%'
        encryption_key: '%env(OAUTH_ENCRYPTION_KEY)%'          # defuse key / plain string (encrypts auth codes & refresh tokens)
        encryption_key_type: defuse                           # plain|defuse
        access_token_ttl: PT15M                               # default PT1H
        refresh_token_ttl: P14D                               # default P1M
        auth_code_ttl: PT1M                                   # default PT10M; RFC 9700 advises short
        device_code_ttl: PT10M
        enable_client_credentials_grant: true
        enable_auth_code_grant: true
        enable_refresh_token_grant: true
        enable_password_grant: false        # MUST set explicitly; unset = deprecated, defaults true in 1.x
        enable_implicit_grant: false        # MUST set explicitly; same
        require_code_challenge_for_public_clients: true
        revoke_refresh_tokens: true         # refresh-token rotation (default true)
        persist_access_token: true
        response_type_class: ~              # custom ResponseType, e.g. an OIDC IdTokenResponse
        enable_device_code_grant: false
        device_code_verification_uri: ''
        device_code_polling_interval: 5
    resource_server:
        public_key: '%env(resolve:OAUTH_PUBLIC_KEY)%'
        jwt_leeway: PT5S                    # clock skew
    scopes:
        available: ['openid', 'profile', 'email', 'members:read', 'contributions:write']
        default: ['profile']                # a client with no scopes gets ALL scopes unless defaults are set!
    persistence:
        doctrine:
            entity_manager: default
            table_prefix: oauth2_
    role_prefix: ROLE_OAUTH2_
    client:
        classname: League\Bundle\OAuth2ServerBundle\Model\Client   # or a custom subclass
    # allow_plaintext_secrets: false     # secrets hashed since 1.2 [option present in Configuration.php]
```

Routes: `oauth2: { resource: '@LeagueOAuth2ServerBundle/config/routes.php', type: php }` → `/authorize`, `/token`, `/device-code`.

Security:
```yaml
security:
    firewalls:
        api_token:       { pattern: ^/token$,       security: false }
        api_device_code: { pattern: ^/device-code$, security: false }
        api:
            pattern: ^/api
            stateless: true
            oauth2: true            # bundle authenticator; scopes become ROLE_OAUTH2_<SCOPE>
    access_control:
        - { path: ^/authorize, roles: IS_AUTHENTICATED_REMEMBERED }   # mandatory since 0.9
```
Per-route scope: route `defaults: { oauth2_scopes: ['foo','bar'] }` (all are required). Errors: 401 for an invalid or expired token, 403 for insufficient scope.

**Commands** (2.x `src/Command`): `league:oauth2-server:create-client <name> [id] [secret] --redirect-uri= --grant-type= --scope= --public --allow-plain-text-pkce`, `update-client` (`--add/remove-redirect-uri|grant-type|scope`, `--activate/--deactivate`), `delete-client`, `list-clients`, `clear-expired-tokens [-a -r -c]`, `generate-keypair`, `rehash-client-secrets`.

**Events** (`OAuth2Events`):
- `USER_RESOLVE` (`league.oauth2_server.event.user_resolve`): user resolution for the password grant.
- `SCOPE_RESOLVE`: alter or finalise the granted scopes (enforce per-client and per-user scope minimisation here).
- `AUTHORIZATION_REQUEST_RESOLVE`: **consent screen hook**. The listener gets the authenticated user and the requested scopes and calls `resolveAuthorization(true|false)`, or sets a response that renders a consent page. Persist consent per (user, client, scopes).
- `TOKEN_REQUEST_RESOLVE`: modify the token response.
- `PRE_SAVE_CLIENT`
- `ACCESS_TOKEN_EXTRA_CLAIMS_RESOLVE` (2.x): add JWT claims such as `iss` or tenant.

Raw league events (`RequestAccessTokenEvent`, `RequestRefreshTokenEvent`, `CLIENT_AUTHENTICATION_FAILED`) are covered in `docs/listening-to-league-events.md`. Use them for audit logs and brute-force throttling.

Custom persistence: `persistence.custom: { access_token_manager, authorization_code_manager, client_manager, refresh_token_manager, device_code_manager, credentials_revoker }`.

Test speed-up: override `league.oauth2_server.password_hasher` with a low-cost `NativePasswordHasher` under `when@test`.

### 1.4 OIDC on top of league
`steverhoades/oauth2-openid-connect-server` provides `IdTokenResponse(IdentityRepository, ClaimExtractor)`, `ClaimSet`, and an `openid` scope. Wire it via the bundle's `response_type_class` [U: exact wiring with bundle DI]. Gaps: no discovery doc, no JWKS endpoint, no userinfo endpoint, and `nonce` handling is limited [U]. You would hand-build `/.well-known/openid-configuration`, `/jwks.json`, `/userinfo`, end-session and key rotation, and you would never get OpenID certification. **Verdict: avoid building an OIDC provider on league for a fintech product.**

### 1.5 Build vs buy (latest releases from GitHub, 2026-10)

| IdP | Latest | Notes |
|---|---|---|
| Keycloak | **26.8.0** (2026-10-01) | Java, Apache-2. OIDC certified. Supports PAR, mTLS, client policies/FAPI profiles, DPoP [U: GA level], token exchange, introspection, revocation, device flow, organisations. Heavy to operate (JVM, Postgres) |
| Zitadel | **v4.19.4** (2026-10-01) | Go, multi-tenant (orgs), cloud or self-host. Licence changed to AGPL in v3 [U] |
| Authentik | **2026.8.3** (2026-09-17) | Python. Good for SSO/workforce, less so for FAPI |
| Ory Hydra | **v26.2.0** (2026-03-20) | Headless, OpenID certified. **You keep the login and consent UI in Symfony** (login/consent challenge APIs). Good middle ground |
| Auth0 / Okta / Cognito / Entra External ID | SaaS | Fastest to ship. Per-MAU cost. Check data residency (NDPA 2023 for Nigeria) |

Trade-off: build (league) is only cheap for the simple case of first-party clients plus client_credentials partners. Everything RFC 9700 / FAPI recommends (sender constraint, PAR, asymmetric client auth, metadata, key rotation, introspection, revocation) is DIY on league. Buy or run an IdP when you need OIDC SSO, social federation, MFA, FAPI or partner onboarding. For a Symfony API, the most pragmatic option is **Keycloak or Hydra as the AS, with Symfony as a pure resource server (§2)**.

---

## 2. Symfony as a RESOURCE SERVER (`access_token` authenticator)

Doc: https://symfony.com/doc/current/security/access_token.html (current = 8.1)

Packages: `web-token/jwt-library` **4.2.3** (php >=8.2) for `oidc`; `symfony/cache` for discovery; `symfony/http-client` for `oidc_user_info` and `cas`. **Install ext-gmp** in production, because `brick/math` falls back to pure PHP and becomes much slower.

Feature history: multiple signature `algorithms` (7.1, replaced `algorithm`); OIDC discovery with JWKS fetch and cache (7.3); JWE decryption `encryption` (7.3); multiple discovery endpoints/issuers (7.4); `discovery.enforce_key_usage_verification` (8.1, strict by default: only keys with `use: sig` or `key_ops` containing sign/verify).

### 2.1 `oidc` (local JWT validation) — recommended
```yaml
security:
    firewalls:
        api:
            pattern: ^/api
            stateless: true
            access_token:
                token_handler:
                    oidc:
                        claim: sub                       # user identifier claim (default sub)
                        algorithms: ['RS256', 'ES256']   # whitelist; never 'none'/HS* with public keys
                        audience: 'colsynq-api'          # required → validates aud
                        issuers: ['https://auth.colsynq.com/realms/colsynq']   # required → validates iss (exact string)
                        discovery:
                            base_uri: https://auth.colsynq.com/realms/colsynq/
                            cache: { id: cache.app }     # JWKS cached; plan for key rotation (cache TTL/clear)
                            # enforce_key_usage_verification: true   # 8.1, default strict
                        # encryption:                     # JWE (7.3+)
                        #     enabled: true
                        #     enforce: false
                        #     algorithms: ['ECDH-ES', 'A128GCM']
                        #     keyset: '%env(OIDC_DECRYPTION_JWKS)%'
```
Static keys: replace `discovery` with `keyset: '{"keys":[...]}'`. For multiple IdPs, list `issuers: [...]` and `discovery.base_uri: [...]`. Each token is checked only against its own issuer's keys, and two discovery docs cannot announce the same issuer.

Test tokens: `php bin/console security:oidc:generate-token john@example.com --firewall=api --algorithm=HS256 --issuer=https://example.com`.

### 2.2 `oidc_user_info` (opaque tokens; calls the IdP on every request, so cache it)
```yaml
access_token:
    token_handler:
        oidc_user_info:
            claim: email
            base_uri: https://auth.example.com/realms/demo/
            discovery: { cache: { id: cache.app } }
            # client: oidc.client
```

### 2.3 `cas`
`cas: { validation_url: https://cas.example.com/cas/validate, prefix: cas, http_client: ~ }`.

### 2.4 Mapping claims to a user
- The default produces an `OidcUser` that holds the claims.
- Custom: implement `AttributesBasedUserProviderInterface` (`loadUserByIdentifier($id, array $attributes)`) to upsert a local `User`/`Member` (JIT provisioning) and map roles from `realm_access.roles`, `scope` or `groups`.
- Or write a custom `AccessTokenHandlerInterface::getUserBadgeFrom()` returning `new UserBadge($id, fn() => ..., $claims)`.
- Map **scopes → Symfony roles or voters** (e.g. `SCOPE_members:read`). In API Platform use `security: "is_granted('SCOPE_members:read')"` on operations, backed by a custom voter [U: role naming is your choice].
- Always check `aud`, `iss`, `exp`/`nbf` (the handler does this) and, for M2M, the `client_id`/`azp` claim to know which partner is calling.

### 2.5 Token introspection (RFC 7662)
Not built into Symfony. For opaque or reference tokens (Keycloak lightweight tokens, Hydra opaque tokens), write a custom `AccessTokenHandlerInterface` that POSTs to `introspection_endpoint` with client auth, requires `active: true`, checks `aud` and `scope`, and caches positive results until `min(exp, 60s)`. A library option is `facile-it/php-openid-client` `IntrospectionServiceBuilder` (also RFC 7009 revocation and JWT introspection responses). Trade-off: introspection gives instant revocation at the cost of latency and IdP coupling. JWTs plus a short TTL (5–15 min) is the usual compromise.

### 2.6 API Platform 5 (OpenAPI/Swagger UI auth)
`api_platform.oauth` (root node in core `Configuration.php`):
```yaml
api_platform:
    oauth:
        enabled: true
        clientId: 'swagger-ui'
        pkce: true                 # use PKCE; never put clientSecret in prod (config's own warning)
        type: oauth2
        flow: authorizationCode    # default 'application' (= client_credentials)
        tokenUrl: 'https://auth.colsynq.com/.../token'
        authorizationUrl: 'https://auth.colsynq.com/.../auth'
        refreshUrl: ''
        scopes: { 'members:read': 'Read members' }
    swagger:
        api_keys:
            JWT: { name: Authorization, type: header }
```

---

## 3. Symfony as an OAuth CLIENT / social login

| Package | Latest | Notes |
|---|---|---|
| `knpuniversity/oauth2-client-bundle` | **v2.20.2** (2026-02-12) | php >=8.1, Symfony `^6.4\|^7.3\|^8.0` ✔ |
| `league/oauth2-client` | **2.9.1** (2026-09-16) | php `>=8.0 <8.6`. PKCE since 2.7.0 (`getPkceMethod()` override → S256) |
| `league/oauth2-google` | **5.0.0** (2026-03-23) | php ^8.0 |
| `thenetworg/oauth2-azure` (Microsoft Entra) | v2.2.6 (2026-06-22) | knpu type `azure` |
| `patrickbussmann/oauth2-apple` | 0.4.0 (2026-02-26) | knpu type `apple`. Needs the ES256 client-secret JWT (p8 key) |
| `hwi/oauth-bundle` | **2.5.0** (2026-02-19) | php ^8.3, Symfony `^6.4\|^7.4\|^8.0`. Maintained and config-driven (resource owners), but less flexible than knpu. Choose knpu for an API |
| `hvatum/oauth2-openid-connect-client` | v0.5.0 (2026-08) | generic OIDC provider for league client with PAR, DPoP, private_key_jwt. Low adoption, so vet it first |
| `facile-it/php-openid-client` | 1.0.0 (2026-06-12) | Full OIDC RP: discovery, id_token validation, private_key_jwt, JAR, JARM, introspection, revocation, mTLS (draft) |

Config (knpu README):
```yaml
knpu_oauth2_client:
    clients:
        google:
            type: google
            client_id: '%env(OAUTH_GOOGLE_CLIENT_ID)%'
            client_secret: '%env(OAUTH_GOOGLE_CLIENT_SECRET)%'
            redirect_route: connect_google_check
            redirect_params: {}
            # use_state: true          # default true → keep it
        azure:
            type: azure
            client_id: '%env(OAUTH_AZURE_CLIENT_ID)%'
            client_secret: '%env(OAUTH_AZURE_CLIENT_SECRET)%'
            redirect_route: connect_azure_check
            # tenant: 'common'   [U]
        apple:
            type: apple
            client_id: '%env(OAUTH_APPLE_CLIENT_ID)%'
            team_id: '%env(OAUTH_APPLE_TEAM_ID)%'      # [U] key names
            key_file_id: '%env(OAUTH_APPLE_KEY_ID)%'
            key_file_path: '%kernel.project_dir%/config/apple.p8'
            redirect_route: connect_apple_check
```
PKCE: `KnpU\OAuth2ClientBundle\Client\OAuth2PKCEClient` stores a 64-byte verifier in the session and sends S256. It is used by `keycloak_pkce`. For other providers, wrap or decorate the client, or subclass the league provider and override `getPkceMethod(): string { return self::PKCE_METHOD_S256; }` [U for knpu wiring].

Authenticator pattern: `class GoogleAuthenticator extends OAuth2Authenticator implements AuthenticationEntryPointInterface`. `supports()` checks the `_route` of the check route. `authenticate()` calls `$this->fetchAccessToken($client)` (validates `state`) and returns `new SelfValidatingPassport(new UserBadge($token->getToken(), fn() => findOrCreate...))`.

**Security notes for social login:**
- **Account linking:** the knpu README example links by email (`findOneBy(['email'=>...])`). This is account-takeover-prone. Only auto-link when the IdP asserts `email_verified=true` *and* the IdP is authoritative for that domain (Google for gmail, Entra for tenant domains). Otherwise require the user to log in to the existing account and link explicitly. Store `(provider, sub)` as the immutable key, never the email.
- league/oauth2-client **does not validate id_tokens**. If you rely on id_token claims (Apple sends user data only in the id_token), verify signature, iss, aud, exp and nonce yourself with `web-token`/`lcobucci`, or use facile-it.
- Use the `nonce` param for OIDC, keep `use_state: true`, and use PKCE everywhere (RFC 9700).
- For an API-first backend with a SPA or mobile client: the **mobile app** does native PKCE against Google or Apple and sends the id_token to your API. The API verifies it via the `oidc` handler with Google/Apple JWKS discovery (`issuers: ['https://accounts.google.com']`, `audience: <your client id>`) and exchanges it for your own session or token. **SPA**: use BFF (§5).

---

## 4. Machine-to-machine and partner integrations

- **client_credentials** for partners (banks, payroll, PSPs):
  - one client per partner per environment
  - least-privilege scopes
  - short access tokens (5–15 min) and no refresh tokens (RFC 6749 §4.4.3)
  - restrict the `aud` per API
  - log `client_id` on every call
  - rotate credentials, with IP allow-listing as defence in depth
- **Scope design:** use `resource:action` (`members:read`, `contributions:write`, `payouts:initiate`), with coarse role-style scopes avoided. Separate high-risk scopes (money movement) so they need stronger client auth or step-up. Use RFC 8707 `resource` indicators or `aud` to bind tokens to one API. Consider RFC 9396 Rich Authorization Requests (`authorization_details`) for payment consents (FAPI/open banking style) [U: IdP support varies, Keycloak partial].
- **Client authentication strength** (RFC 9700 says prefer asymmetric):
  - `client_secret_basic`: weakest, a shared secret.
  - `private_key_jwt` (RFC 7523 §2.2 / OIDC Core §9): the client signs a JWT assertion `{iss=sub=client_id, aud=token endpoint/issuer, jti, exp≤60s}` and sends `client_assertion_type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer`. In PHP, build it with `web-token/jwt-library` or `lcobucci/jwt`. facile-it and hvatum support it out of the box. league/oauth2-server does **not** accept it (DIY via a custom `ClientRepository::validateClient` [U]).
  - `tls_client_auth` / `self_signed_tls_client_auth` (RFC 8705 mTLS): terminate TLS at nginx/Envoy, forward the client cert (e.g. `X-SSL-Client-Cert`), and bind tokens with the `cnf.x5t#S256` claim. The RS must compare the cert thumbprint with `cnf`. Symfony has no built-in support, so you need a custom token handler or listener.
- **DPoP (RFC 9449):** the client signs a per-request proof JWT (`htm`, `htu`, `iat`, `jti`, `ath`) and the token is bound with `cnf.jkt`. The header is `Authorization: DPoP <token>` plus a `DPoP: <proof>` header. **No Symfony bundle exists.** PHP libraries: `danielburger1337/oauth2-dpop` v1.2.0 (php ^8.4, about 580 downloads, so low adoption) and `labrodev/laravel-dpop` (Laravel only). To implement it on the RS:
  - a custom `AccessTokenExtractorInterface` for the `DPoP` scheme
  - verify the proof signature with the JWK in the header
  - `htm`/`htu` match, `iat` window, `jti` replay cache (Redis)
  - `ath` = SHA-256 of the token, `jkt` = thumbprint matches `cnf.jkt`
  - optional `DPoP-Nonce`

  Keycloak supports issuing DPoP-bound tokens [U: version/GA].
- **PAR (RFC 9126):** the client POSTs the authorization params to `/par` with client auth, gets a `request_uri`, then redirects with only `client_id` + `request_uri`. This prevents parameter tampering and is mandatory in FAPI 2.0. league server has no support. Keycloak, Hydra [U] and Auth0 (enterprise) do.
- **Outbound M2M (your API calling a bank or payroll provider):** use `symfony/http-client` with a token-caching decorator (cache until `exp - 30s`), store secrets and private keys in Symfony secrets or a vault, and support both `private_key_jwt` and mTLS (`HttpClient` options `local_cert`/`local_pk`).

---

## 5. Security best practices

### Standards status (Oct 2026)
- **RFC 9700 — Best Current Practice for OAuth 2.0 Security** (BCP 240, Jan 2025). Updates RFC 6749/6750/6819. https://www.rfc-editor.org/info/rfc9700
- **OAuth 2.1**: still an Internet-Draft, `draft-ietf-oauth-v2-1-16` (2026-09-03), WG Document. The milestone is to submit to the IESG by Dec 2026, and no RFC number exists. https://datatracker.ietf.org/doc/draft-ietf-oauth-v2-1/
- **OAuth 2.0 for Browser-Based Applications**: **published as RFC 10017** (BCP, Aug 2026; from draft-27). https://datatracker.ietf.org/doc/draft-ietf-oauth-browser-based-apps/, https://www.rfc-editor.org/info/rfc10017. The BCP number shows as 212 on rfc-editor, but the page also lists RFC 8252 as BCP 212 [U].
- **FAPI 2.0 Security Profile**: OpenID Foundation **Final** (Feb 2025), along with the Attacker Model. Final conformance tests have been available since Jul 2025, and Message Signing is a separate spec. https://openid.net/specs/fapi-security-profile-2_0-final.html
- Related RFCs:
  - 7636 PKCE, 7009 revocation, 7662 introspection, 8414 AS metadata, 8628 device, 8705 mTLS, 8707 resource indicators
  - 9068 JWT access-token profile, 9126 PAR, 9207 `iss` response param, 9396 RAR, 9449 DPoP, 7523 JWT client auth
  - 9728 Protected Resource Metadata (2025) [U date]

### Checklist (RFC 9700 / OAuth 2.1)
1. **Grants:** auth code + PKCE (S256) for every client, including confidential ones. client_credentials for M2M. device code only for input-constrained devices. **No implicit, no ROPC/password**. In the league bundle, set `enable_password_grant: false` and `enable_implicit_grant: false` **explicitly**, because the 1.x defaults are true.
2. **PKCE mandatory.** The AS must reject a `code_verifier` without a challenge and stop downgrade (no `plain`, so never pass `--allow-plain-text-pkce`).
3. **Redirect URIs:** pre-registered, **exact string match** (except loopback port), no wildcards, no open redirectors. Use 303 rather than 307 after credential POSTs.
4. **CSRF/mix-up:** `state` and/or PKCE, OIDC `nonce`. With multiple AS, validate the `iss` response param (RFC 9207).
5. **Access tokens:** short-lived (5–15 min), audience-restricted (`aud` = your API), minimal scopes, and sender-constrained where possible (mTLS or DPoP; FAPI requires it). Validate `iss`, `aud`, `exp`, `alg` whitelist and `typ` (`at+jwt` per RFC 9068).
6. **Refresh tokens:**
   - public clients must use **rotation with reuse detection** (revoke the whole family on reuse) or sender constraint
   - set an absolute lifetime (e.g. 14–30 days) plus an idle timeout
   - bind to the client
   - league: `revoke_refresh_tokens: true` gives rotation, but reuse-detection family revocation is DIY [U]
7. **Auth codes:** single use, lifetime ≤ 1 min (BCP guidance is short; league default is 10 min, so lower it), bound to client + redirect_uri + PKCE.
8. **Client auth:** prefer asymmetric methods (`private_key_jwt`, mTLS). Hash stored secrets (league bundle ≥1.2 does). Rotate them.
9. **Keys:** RS256/ES256/PS256, publish JWKS, rotate keys with an overlap window, and keep private keys in a KMS/HSM or Symfony secrets. Never accept `none` or HS256 alongside public keys.
10. **Consent:** an explicit screen for third-party clients that lists scopes in plain language. Store consent records per (user, client, scope set, timestamp) for audit and revocation (NDPA 2023 for Nigeria requires demonstrable consent). Skip consent only for first-party clients.
11. **Scope minimisation:** clients get an `allowed_scopes` list, and the AS intersects requested ∩ allowed ∩ user-entitled (`SCOPE_RESOLVE` event in the league bundle). Never "no scopes = all scopes" (league bundle warning: set `scopes.default`).
12. **SPA → BFF (RFC 10017):** Symfony acts as a confidential client and holds tokens server-side. The browser gets only an `HttpOnly; Secure; SameSite=Lax/Strict` session cookie (`__Host-` prefix), with CSRF protection on state-changing requests. Ranking: BFF > token-mediating backend > browser-only client with in-memory tokens + DPoP. Never put tokens in localStorage.
13. **Native/mobile:** use RFC 8252 (system browser, claimed HTTPS redirect / app links, PKCE).
14. **Ops:**
    - rate-limit `/token` and `/authorize` (`symfony/rate-limiter`)
    - log token issuance and auth failures, never logging tokens or secrets (`#[SensitiveParameter]`)
    - publish `/.well-known/oauth-authorization-server` metadata
    - revocation endpoint plus a logout/session-end path

### FAPI 2.0 and Nigeria open banking
- FAPI 2.0 baseline:
  - auth code + **PAR** + PKCE (S256)
  - confidential clients only, using `private_key_jwt` or mTLS
  - sender-constrained tokens (mTLS or DPoP)
  - `iss` response param
  - no refresh token rotation required when tokens are sender-constrained [U]
  - auth code lifetime ≤ 60 s
  - PS256/ES256/EdDSA only
  
  This is realistic only with a FAPI-capable IdP (Keycloak client policies `fapi-2-security-profile` [U], Authlete, Ping, Connect2id, Curity).
- **Nigeria (CBN):**
  - Regulatory Framework for Open Banking (Feb 2021) and Operational Guidelines (Mar 2023). They prescribe REST + OAuth 2.0 + TLS, a common Banking Industry API standard, and risk-tier access (Tier 0–3).
  - NIBSS runs the Open Banking Registry and the consent system.
  - Go-live was approved for 1 Aug 2025, but reports conflict: TechCabal (Oct 2025) quoted the CBN as saying it was "not live yet", and Mondaq says it was postponed to early 2026 with phased rollout through mid-2026.
  - A vendor blog (WSO2) claims FAPI is mandatory. **I did not verify this against the CBN standard text [U].**
  - Design partner and bank integrations to be FAPI-2-ready (PAR, private_key_jwt/mTLS, sender-constrained tokens, granular consent with RAR).

  Sources: https://www.cbn.gov.ng/Out/2021/PSMD/Circular%20on%20the%20Regulatory%20Framework%20on%20Open%20Banking%20in%20Nigeria.pdf, https://www.cbn.gov.ng/Out/2022/CCD/OPERATIONAL%20GUIDELINES%20FOR%20OPEN%20BANKING%20IN%20NIGERIA_APPROVED%20EXPOSURE%20DRAFT.pdf, https://techcabal.com/2025/10/16/nigerias-open-banking-still-in-the-works/, https://wso2.com/library/blogs/nigerias-open-banking-a-technical-blueprint-for-cios-and-cisos/
- OWASP references: OAuth2 Cheat Sheet https://cheatsheetseries.owasp.org/cheatsheets/OAuth2_Cheat_Sheet.html and the WSTG OAuth testing section [U: not re-fetched for this note].

---

## 6. Suggested architecture for Colsynq (Symfony 8.1 + API Platform 5)

1. **AS = Keycloak 26.x** (self-host, or managed if ops capacity is low), or Ory Hydra with the login/consent UI in Symfony. Realm `colsynq`. Clients:
   - `colsynq-web-bff` (confidential, auth code + PKCE)
   - `colsynq-mobile` (public, PKCE, refresh rotation)
   - `partner-<name>` (client_credentials, `private_key_jwt`)
   - `swagger-ui` (public, PKCE, dev only)
2. **API = resource server:** `access_token` → `oidc` handler with discovery, `audience: colsynq-api`, an `AttributesBasedUserProviderInterface` provider for JIT local users, and scope→role voters for API Platform `security:` expressions.
3. **Web = BFF** (separate Symfony app or firewall) using knpu/facile-it as the OIDC client, with the session cookie on the browser.
4. **Social login:** federate Google, Apple and Microsoft **in the IdP** (identity brokering) rather than in the API, so that account linking policy lives in one place.
5. **Fallback when no external IdP is allowed:** `league/oauth2-server-bundle` 1.2.x with password/implicit off, PKCE, short TTLs, rotation, `SCOPE_RESOLVE` minimisation and an `AUTHORIZATION_REQUEST_RESOLVE` consent screen. Customise the access token so it has `iss` and RFC 9068 shape, or protect APIs with `oauth2: true`. Upgrade to bundle 2.x once it is tagged.
