# JWT auth for Symfony 8.1 + API Platform 5 + PHP 8.4 (as of 2026-10-12)

Legend: [V] = checked against primary source (source code, tagged composer.json, official docs) during this research. [U] = unverified syntax/behaviour; check it before relying on it.

## 0. Version matrix (verified 2026-10-12)

| Package | Latest | Released | Requires | Notes |
|---|---|---|---|---|
| symfony/* | 8.1 (current docs), 8.2 = dev | – | PHP 8.4+ | symfony.com docs show 8.1 as "current" [V] |
| api-platform/core | **v5.0.3** | 2026-10-09 | php >=8.2, Symfony ^7.4 \|\| ^8.0 | docs "Version 5.0 (current)" [V] |
| lexik/jwt-authentication-bundle | **v3.2.0** | 2025-12-20 | php >=8.2, ext-openssl, lcobucci/jwt ^5.0, symfony/* ^6.4\|^7.0\|^8.0 | v3.2.0 added Symfony 8 and web-token 4.x support. There is no newer tag. The 3.x branch has unreleased fixes, see below [V] |
| gesdinet/jwt-refresh-token-bundle | **v3.0.0** | 2026-08-05 | php ^8.4, symfony/* ^8.0 (config ^8.0.9), lexik ^3.0, doctrine/persistence ^3.1\|^4.0 | **v3 supports Symfony 8 only.** v2.2.x is the line for Symfony 6.4/7.x [V] |
| web-token/jwt-bundle / jwt-library | 4.2.3 | 2026-09-12 | – | Needed for lexik's `web_token` encoder (JWE, JWKSet, EdDSA) and for Symfony's `oidc` access-token handler [V] |
| lcobucci/jwt | 5.6.1 | 2026-10-10 | – | Used by lexik's default encoder [V] |

Repo/package URLs:
- https://packagist.org/packages/lexik/jwt-authentication-bundle, https://github.com/lexik/LexikJWTAuthenticationBundle/releases
- https://packagist.org/packages/gesdinet/jwt-refresh-token-bundle, https://github.com/markitosgv/JWTRefreshTokenBundle (CHANGELOG.md, UPGRADE-3.0.md)
- https://packagist.org/packages/api-platform/core
- https://github.com/web-token/jwt-framework/releases

"markitosgv" and "gesdinet" are the same bundle: the GitHub repo is `markitosgv/JWTRefreshTokenBundle` and the Composer name is `gesdinet/jwt-refresh-token-bundle`. No separate alternative is needed.

**Caveats found:**
- Lexik 3.2.0's `require-dev` tests against `api-platform/core ^3.0|^4.0` only, so API Platform 5 is **not CI-tested** by lexik. I checked that every class lexik's `OpenApi/OpenApiFactory.php` imports (`ApiPlatform\OpenApi\Factory\OpenApiFactoryInterface`, `Model\Operation`, `PathItem`, `RequestBody`, `MediaType`, `OpenApi`) still exists in api-platform/core v5.0.3. It should work. [V for class existence; runtime behaviour U]
- The lexik 3.x branch has unreleased fixes as of 2026-10:
  - #1308: `JWTUserProvider` keeps a stale user cache in long-running runtimes (FrankenPHP/RoadRunner/Swoole), so a user can get roles from an old request. This matters if you use the DB-less `lexik_jwt` user provider under worker mode.
  - #1311: support for `AttributesBasedUserProviderInterface`.
  - #1319: `#[\Deprecated]` on `eraseCredentials`.
  - Pin to `3.x-dev` or wait for 3.2.1/3.3.0 if you need these. [V via GitHub commits API]
- API Platform 5 moved `ApiTestCase` to `ApiPlatform\Test\ApiTestCase`. The old `ApiPlatform\Symfony\Bundle\Test\ApiTestCase` triggers a deprecation since 5.0 [V, source].

---

## 1. lexik/jwt-authentication-bundle 3.2

### Install
```bash
composer require lexik/jwt-authentication-bundle   # Flex registers the bundle
php bin/console lexik:jwt:generate-keypair          # --skip-if-exists | --overwrite
# keys -> config/jwt/private.pem, config/jwt/public.pem
```
`.env`:
```
JWT_SECRET_KEY=%kernel.project_dir%/config/jwt/private.pem
JWT_PUBLIC_KEY=%kernel.project_dir%/config/jwt/public.pem
JWT_PASSPHRASE=changeme
```
Source: https://github.com/lexik/LexikJWTAuthenticationBundle/blob/3.x/Resources/doc/index.rst [V]

### Algorithms. Important limitation
- The default encoder is `lexik_jwt_authentication.encoder.lcobucci`. It supports **only HS256/384/512, RS256/384/512, ES256/384/512**. There is **no EdDSA and no PS\***. Source: `Services/JWSProvider/LcobucciJWSProvider.php` signer map [V].
- `lexik:jwt:generate-keypair` accepts the same list. Defaults are RS256 = 2048-bit and RS512 = 4096-bit. It does **not** support EdDSA [V, `Command/GenerateKeyPairCommand.php`]. It takes the algorithm from `encoder.signature_algorithm` config.
- For ES256 with the lcobucci encoder:
  ```yaml
  lexik_jwt_authentication:
      encoder: { signature_algorithm: ES256 }
  ```
  Then run `lexik:jwt:generate-keypair`, or use openssl manually:
  ```bash
  openssl ecparam -name prime256v1 -genkey -noout | openssl pkcs8 -topk8 -v2 aes-256-cbc -out config/jwt/private.pem   # [U] flags
  openssl ec -in config/jwt/private.pem -pubout -out config/jwt/public.pem
  ```
- **EdDSA (Ed25519)** and **PS256** require the **web-token encoder** (`lexik_jwt_authentication.encoder.web_token` plus `composer require web-token/jwt-bundle`). Generate keys with `jose.phar` ([web-token/jwt-app](https://github.com/web-token/jwt-app)), e.g. `key:generate:okp --random_id --use=sig --alg=EdDSA Ed25519` [U]. The lexik docs literally write `--alg=ED256`, which is not a registered JOSE alg. The JOSE name is `EdDSA` (RFC 8037), and RFC 9864 adds fully-specified `Ed25519`. The web-token 4.x alg name is probably `EdDSA` or `Ed25519` [U].
- Recommendation: **ES256** with the lcobucci encoder is small, fast, and the simplest choice. Use **RS256 at 3072+ bits** if consumers require RSA. Use EdDSA only if you adopt the web-token encoder anyway.

### Full config reference (3.2.0 `DependencyInjection/Configuration.php`) [V]
```yaml
# config/packages/lexik_jwt_authentication.yaml
lexik_jwt_authentication:
    secret_key: '%env(resolve:JWT_SECRET_KEY)%'   # path or raw key; required to create tokens
    public_key: '%env(resolve:JWT_PUBLIC_KEY)%'   # required to verify tokens
    pass_phrase: '%env(JWT_PASSPHRASE)%'          # default ''
    additional_public_keys:                       # tried if public_key fails (rotation) - not for HMAC
        - '%kernel.project_dir%/config/jwt/public_prev.pem'
    token_ttl: 900                                # default 3600 (s). Recommend 5-15 min
    clock_skew: 30                                # default 0 (s) - leeway for exp/iat/nbf
    allow_no_expiration: false                    # default false - keep false
    encoder:
        service: lexik_jwt_authentication.encoder.lcobucci   # default; or ...encoder.web_token
        signature_algorithm: ES256                            # default RS256
    user_id_claim: username                       # default 'username' - claim that holds getUserIdentifier()
    remove_token_from_body_when_cookies_used: true # default true
    token_extractors:
        authorization_header: { enabled: true, prefix: Bearer, name: Authorization }  # canBeDisabled
        cookie:               { enabled: false, name: BEARER }                         # canBeEnabled
        query_parameter:      { enabled: false, name: bearer }                         # keep disabled (logs/Referer leak)
        split_cookie:         { enabled: false, cookies: [jwt_hp, jwt_s] }
    set_cookies: {}                               # see cookie section
    blocklist_token:                              # canBeEnabled -> DEFAULT DISABLED
        enabled: false
        cache: cache.app
    api_platform:                                 # canBeEnabled
        enabled: true
        check_path: /api/auth
        username_path: email
        password_path: password
    access_token_issuance:      { enabled: false, ... }   # web-token encoder only
    access_token_verification:  { enabled: false, ... }   # web-token encoder only
```
**Gotcha:** the docs' "Full default configuration" block shows `blocklist_token: enabled: true`, but the code uses `canBeEnabled()`, so the blocklist is **off by default**. When it is off, **no `jti` claim is added** [V].

### What the default (lcobucci) encoder validates [V, LcobucciJWSProvider.php]
- It validates only `SignedWith(signer, key)` and `LooseValidAt(clock, clock_skew)` (exp/nbf/iat).
- The signer is fixed by config, so the token's `alg` header cannot pick the algorithm. lcobucci's `SignedWith` rejects a token whose header `alg` differs from the configured signer. This blocks alg-confusion and `none` attacks. [V for design; header-alg check is lcobucci 5 behaviour, U on exact code path]
- **It does NOT validate `iss`, `aud`, or `typ`.** RFC 8725 §3.8/3.9/3.11 want these checks. Add them yourself with a `JWT_DECODED` listener (below), or use the web-token encoder with `mandatory_claims` and claim checkers.
- `sub` is emitted only if you put `sub` in the payload.

### security.yaml (Symfony 8.1, single firewall, stateless)
```yaml
security:
    password_hashers:
        Symfony\Component\Security\Core\User\PasswordAuthenticatedUserInterface: 'auto'
    providers:
        app_user_provider:
            entity: { class: App\Entity\User, property: email }
    firewalls:
        dev:
            pattern: ^/(_profiler|_wdt|assets|build)/
            security: false
        api:
            pattern: ^/api
            stateless: true
            provider: app_user_provider
            entry_point: jwt
            json_login:
                check_path: /api/auth            # route must exist (no controller)
                username_path: email
                password_path: password
                success_handler: lexik_jwt_authentication.handler.authentication_success
                failure_handler: lexik_jwt_authentication.handler.authentication_failure
            jwt: ~                               # provider/authenticator overridable
            refresh_jwt:                          # gesdinet v3 (see section 2)
                check_path: /api/token/refresh
            login_throttling:
                max_attempts: 5
                interval: '15 minutes'
            logout:
                path: /api/logout                 # triggers LogoutEvent -> lexik blocklist + gesdinet invalidation
    access_control:
        - { path: ^/api/(auth|token/refresh)$, roles: PUBLIC_ACCESS }
        - { path: ^/api/docs, roles: PUBLIC_ACCESS }
        - { path: ^/api, roles: IS_AUTHENTICATED_FULLY }
```
```yaml
# config/routes.yaml
api_auth:          { path: /api/auth, methods: [POST] }
api_refresh_token: { path: /api/token/refresh, methods: [POST] }
api_logout:        { path: /api/logout, methods: [POST] }   # [U] logout listener matches path; method restriction is optional
```
- The lexik docs show a two-firewall layout (`login` with `^/api/login`, plus `api`). With a single firewall, json_login, jwt, and refresh_jwt work together. The API Platform docs use a single `main` firewall with `json_login` + `jwt: ~` [V].
- Under Apache, Authorization headers are stripped unless you add `SetEnvIf Authorization "(.*)" HTTP_AUTHORIZATION=$1`.
- `login_throttling` requires `composer require symfony/rate-limiter`. The default is 5 attempts per minute. The limit is `max_attempts` per IP+username, and `5*max_attempts` per IP. A custom limiter can be set with `limiter: <service id>` [V, symfony.com/doc/current/security.html].

### Events. Constants in `Lexik\Bundle\JWTAuthenticationBundle\Events` [V]
| Constant | Event name | Event class | Use |
|---|---|---|---|
| `JWT_CREATED` | `lexik_jwt_authentication.on_jwt_created` | `JWTCreatedEvent` (`getData/setData/getHeader/setHeader/getUser`) | add claims (`iss`, `aud`, `sub`, `typ` header, `ver`) |
| `JWT_ENCODED` | `...on_jwt_encoded` | `JWTEncodedEvent` | get the token string |
| `JWT_DECODED` | `...on_jwt_decoded` | `JWTDecodedEvent` (`getPayload`, `markAsInvalid()`) | validate iss/aud/token-version |
| `JWT_AUTHENTICATED` | `...on_jwt_authenticated` | `JWTAuthenticatedEvent` | customise security token |
| `JWT_INVALID` / `JWT_NOT_FOUND` / `JWT_EXPIRED` | ... | | customise 401 bodies |
| `AUTHENTICATION_SUCCESS` | `...on_authentication_success` | `AuthenticationSuccessEvent` (`getData/setData/getUser/getResponse`) | add user info to the login response |
| `AUTHENTICATION_FAILURE` | `...on_authentication_failure` | `AuthenticationFailureEvent` | generic error body |

Claims plus validation listener (PHP 8.4, attribute-based) [U on exact method names other than those listed in the docs]:
```php
namespace App\Security\Jwt;

use App\Entity\User;
use Lexik\Bundle\JWTAuthenticationBundle\Event\JWTCreatedEvent;
use Lexik\Bundle\JWTAuthenticationBundle\Event\JWTDecodedEvent;
use Lexik\Bundle\JWTAuthenticationBundle\Events;
use Symfony\Component\DependencyInjection\Attribute\Autowire;
use Symfony\Component\EventDispatcher\Attribute\AsEventListener;

final readonly class JwtClaimsListener
{
    public function __construct(
        #[Autowire('%env(JWT_ISSUER)%')] private string $issuer,
        #[Autowire('%env(JWT_AUDIENCE)%')] private string $audience,
    ) {}

    #[AsEventListener(event: Events::JWT_CREATED)]
    public function onCreated(JWTCreatedEvent $event): void
    {
        $user = $event->getUser();
        $payload = $event->getData();
        $payload['iss'] = $this->issuer;
        $payload['aud'] = $this->audience;
        if ($user instanceof User) {
            $payload['sub'] = (string) $user->getId();       // stable, non-PII id (UUID)
            $payload['ver'] = $user->getTokenVersion();       // revocation-by-version
        }
        unset($payload['roles']);                            // optional: don't trust roles from token; reload from DB
        $event->setData($payload);
        $event->setHeader(['typ' => 'at+jwt'] + $event->getHeader()); // RFC 9068 explicit typing [U header merge]
    }

    #[AsEventListener(event: Events::JWT_DECODED)]
    public function onDecoded(JWTDecodedEvent $event): void
    {
        $p = $event->getPayload();
        $aud = (array) ($p['aud'] ?? []);
        if (($p['iss'] ?? null) !== $this->issuer || !\in_array($this->audience, $aud, true)) {
            $event->markAsInvalid();
        }
    }
}
```
- Token-version check: compare `ver` against `$user->getTokenVersion()` after the user loads. Do this in a `JWT_AUTHENTICATED` listener or a custom authenticator extending `JWTAuthenticator` (doc 6-extending-jwt-authenticator.rst). JWT_DECODED does not have the user yet.
- If you set `user_id_claim: sub`, the provider must load by id (e.g. `property: id`, or `uuid`). Otherwise keep `username` = email, but then the email (PII) is in every token. **Recommendation: `user_id_claim: sub` with a UUID.** [U that lexik won't also add a `username` claim when user_id_claim changes. It uses user_id_claim as the key.]

### Cookies (set_cookies / split cookie) [V docs]
```yaml
lexik_jwt_authentication:
    token_extractors:
        cookie: { enabled: true, name: BEARER }
    set_cookies:
        BEARER: ~    # defaults: lifetime null(=token_ttl), samesite lax, path /, domain null, secure true, httpOnly true, partitioned false
```
Split cookie: `jwt_hp` (header.payload, httpOnly false, readable by JS) plus `jwt_s` (signature, httpOnly true, lifetime 0 = session cookie). Both use `samesite: strict` and `split: [header, payload]` / `[signature]`. With cookies, the login response is **204 with an empty body** unless `remove_token_from_body_when_cookies_used: false`.

### Logout / blocklist (`jti`) [V, 11-invalidate-token.rst + Configuration.php]
```yaml
lexik_jwt_authentication:
    blocklist_token: { enabled: true, cache: jwt_blocklist_pool }
framework:
    cache:
        pools:
            jwt_blocklist_pool: { adapter: cache.adapter.redis, provider: '%env(REDIS_URL)%' }  # [U] provider key form
```
Enabling the blocklist does three things:
- It adds a random `jti` via `RandomJtiEnrichment`.
- `BlockJWTListener` blocks the token on `LogoutEvent`, and on `DisabledException` at login.
- `RejectBlockedTokenListener` rejects blocked tokens.

Entries expire at the token's `exp`. The cache must be **shared across processes and nodes** (Redis, not the filesystem `cache.app` in multi-node setups). You can also dispatch `new LogoutEvent($request, $tokenStorage->getToken())` manually.

**Limit:** the blocklist only withdraws the token *presented* at logout. For "log out everywhere" or a password reset, use gesdinet `block_jwts_on_revocation`, or a user `tokenVersion`.

### Web-token encoder, JWE, key rotation [V, 10-web-token.rst + Configuration.php]
```bash
composer require web-token/jwt-bundle
php bin/console lexik:jwt:migrate-config      # dev only: converts PEM config to JWK/JWKSet
php bin/console lexik:jwt:enable-encryption   # dev only: interactive JWE setup
```
```yaml
lexik_jwt_authentication:
    encoder:
        service: lexik_jwt_authentication.encoder.web_token
    access_token_issuance:
        enabled: true
        signature:
            algorithm: 'ES256'
            key: '%env(file:SIGNATURE_KEY)%'            # docs write 'env(file:SIGNATURE_KEY)' [U - probably needs %...%]
        encryption:
            enabled: false
            key_encryption_algorithm: 'ECDH-ES+A256KW'   # [U] alg choice
            content_encryption_algorithm: 'A256GCM'
            key: '%env(file:ENCRYPTION_KEY)%'
    access_token_verification:
        enabled: true
        signature:
            allowed_algorithms: ['ES256']
            keyset: '%env(file:SIGNATURE_KEYSET)%'
            header_checkers: []                               # default []
            claim_checkers: ['exp_with_clock_skew', 'iat_with_clock_skew', 'nbf_with_clock_skew']  # default
            mandatory_claims: ['exp', 'iat', 'iss', 'aud', 'sub']   # default []
        encryption:
            enabled: false
            continue_on_decryption_failure: false   # keep false
            allowed_key_encryption_algorithms: ['ECDH-ES+A256KW']
            allowed_content_encryption_algorithms: ['A256GCM']
            keyset: '%env(file:ENCRYPTION_KEYSET)%'
```
- **Rotation with web-token:** keep a private `signature.jwk` (with `kid`) and a public `signature.jwkset`. Rotate with `jose.phar keyset:rotate "$(cat signature.jwkset)" "$(cat signature.jwk)"`. This puts the new key first and drops the oldest. Overlap must be at least the max token TTL.
- **Rotation with lcobucci:** sign with the new `secret_key`/`public_key`, and keep the old public key in `additional_public_keys` until all old tokens have expired (token_ttl plus skew). Not supported for HMAC [V].
- JWE is only worthwhile if claims are confidential. Prefer simply not putting sensitive data in the token (OWASP).
- The `iss`/`aud` checks in `mandatory_claims` only check presence. Value checks need a custom claim checker or the JWT_DECODED listener [U].

### API Platform 5 OpenAPI integration [V]
```yaml
# lexik: adds the POST check_path to OpenAPI
lexik_jwt_authentication:
    api_platform: { check_path: /api/auth, username_path: email, password_path: password }
# api_platform.yaml: "Authorize" button
api_platform:
    swagger:
        api_keys:
            JWT: { name: Authorization, type: header }
```
Lexik auto-integrates when API Platform is detected. The gesdinet `api_platform.enabled: true` option documents `/api/token/refresh` (off by default). Remove any hand-written OpenAPI decorator to avoid duplicate entries. Source: https://api-platform.com/docs/symfony/jwt/

---

## 2. Refresh tokens: gesdinet/jwt-refresh-token-bundle v3.0.0 (Symfony 8-only) [V, README/UPGRADE-3.0 at tag v3.0.0]

```bash
composer require gesdinet/jwt-refresh-token-bundle doctrine/orm doctrine/doctrine-bundle
```
```php
// src/Entity/RefreshToken.php
namespace App\Entity;
use Doctrine\ORM\Mapping as ORM;
use Gesdinet\JWTRefreshTokenBundle\Entity\RefreshToken as BaseRefreshToken;

#[ORM\Entity]
#[ORM\Table(name: 'refresh_tokens')]
#[ORM\Index(fields: ['family'])]          // recommended in UPGRADE-3.0 (family lookups)
class RefreshToken extends BaseRefreshToken {}
```
v3 adds the columns `family VARCHAR(32) NULL` and `family_valid DATETIME NULL`. Generate them with a migration.

Recommended v3 config:
```yaml
gesdinet_jwt_refresh_token:
    refresh_token_class: App\Entity\RefreshToken
    ttl: 1209600                 # 14 days (default 2592000 = 30d); must be > 0
    ttl_update: false
    single_use: true             # rotation (new RT on each use)
    single_use_ttl_update: true  # default true; false => chain dies one ttl after login
    max_session_lifetime: 2592000  # v3: absolute cap on a refresh chain
    max_tokens_per_user: 10      # concurrent sessions; oldest evicted
    token_parameter_name: refresh_token
    return_expiration: true      # adds refresh_token_expiration
    hash_tokens:
        enabled: true            # store sha256$..., not raw tokens
        accept_stored_in_the_clear: false   # once legacy tokens expired
    reuse_detection:             # v3; requires single_use; not with cache_pool
        enabled: true
        cache: cache.app         # shared store (Redis)
        ttl: 2592000
    block_previous_jwt: true     # v3; needs lexik blocklist_token.enabled
    block_jwts_on_revocation:    # v3; revokeAllForUser() also kills outstanding JWTs by iat
        enabled: true
        cache: cache.app
        ttl: 900                 # >= lexik token_ttl
        user_claim: username     # = lexik user_id_claim
    rate_limiter:                # v3; needs symfony/rate-limiter; 429 + Retry-After
        enabled: true
        limiter: limiter.refresh
        key: ip                  # or token
    cookie:
        enabled: true
        same_site: strict        # default lax
        path: /api/token/refresh # scope the RT cookie to the refresh endpoint [U recommended practice]
        domain: null
        http_only: true
        secure: true
        partitioned: false
        remove_token_from_body: true
    api_platform:
        enabled: true
framework:
    rate_limiter:
        refresh: { policy: sliding_window, limit: 20, interval: '1 minute' }
```
- **Firewall:** `refresh_jwt: { check_path: /api/token/refresh }`. `check_path` is **required in v3**. Per-firewall overrides are allowed (ttl, single_use, etc.), but cookie settings are global. `invalidate_token_on_logout: true` is the default on the authenticator, and `logout_firewall` is deprecated.
- **Reuse detection:** a spent token is remembered by digest. On replay, the whole family is revoked and `RefreshTokenReuseDetectedEvent` (`gesdinet.refresh_token_reuse_detected`) is dispatched. The bundle cannot tell theft from a client racing itself, so serialise refreshes in the frontend (single-flight).
- **APIs:** `revokeAllForUser()`, `findAllForUser()`, `revokeFamily()`, `Session\SessionLister` for a "your devices" UI. Schedule `gesdinet:jwt:clear` (cron or Symfony Scheduler).
- **Alternative storage:** `cache_pool: <psr6 pool>` instead of Doctrine. It cannot be combined with `reuse_detection` or `max_tokens_per_user`. The DBAL backend uses `dbal_connection`, `dbal_table_name`, `dbal_columns`, and `dbal_auto_create_table`.
- Tokens are keyed by `getUserIdentifier()`. **Revoke them when the identifier (email) changes.** This is another reason to use a UUID identifier.
- 2.2.0+ also allows exchanging an expired JWT when `jwt` and `refresh_jwt` are on the same firewall.
- Upgrades: Rector sets ship for each version hop. Symfony 7 users stay on 2.2.x.

Alternatives: none maintained and comparable. You could roll your own opaque refresh token table plus endpoint, or use an OAuth2 server (`league/oauth2-server-bundle`) if you need third-party clients or the authorization-code flow [U current version].

---

## 3. Symfony built-in `access_token` authenticator (6.2+, docs 8.1) [V, symfony.com/doc/current/security/access_token.html]

```yaml
security:
    firewalls:
        api:
            stateless: true
            access_token:
                token_handler: App\Security\ApiTokenHandler
                token_extractors: [header]        # header (default) | query_string | request_body | service id
                # success_handler / failure_handler: service ids
```
```php
use Symfony\Component\Security\Http\AccessToken\AccessTokenHandlerInterface;
use Symfony\Component\Security\Http\Authenticator\Passport\Badge\UserBadge;
use Symfony\Component\Security\Core\Exception\BadCredentialsException;

final readonly class ApiTokenHandler implements AccessTokenHandlerInterface
{
    public function __construct(private ApiTokenRepository $tokens) {}
    public function getUserBadgeFrom(string $accessToken): UserBadge
    {
        $t = $this->tokens->findValidByHash(hash('sha256', $accessToken));  // store hashes only
        if (null === $t) { throw new BadCredentialsException('Invalid credentials.'); }
        return new UserBadge($t->getUserIdentifier());
        // or new UserBadge($id, fn (string $id) => new User(...)) for provider-less
    }
}
```
- **Built-in handlers:**
  - `oidc_user_info` calls the IdP userinfo endpoint and needs http-client.
  - `oidc` validates JWTs locally. It needs `web-token/jwt-library` and **ext-gmp** in production for performance. Options: `algorithms`, `keyset` or `discovery: {base_uri, cache: {id}}`, required `audience` and `issuers`, `claim` (default `sub`), and `encryption: {enabled, enforce, algorithms, keyset}`. `enforce_key_usage_verification` is **new in 8.1**.
  - `cas`.
- `php bin/console security:oidc:generate-token` (with `--firewall`, `--algorithm`, `--issuer`) makes test tokens for the oidc handler.
- The `query_string` and `request_body` extractors SHOULD NOT be used (RFC 6750; tokens end up in logs).

**When to use what:**
| Scenario | Choice |
|---|---|
| Your API issues its own tokens to its own SPA/mobile clients via password login | **lexik + gesdinet**: login endpoint, refresh rotation, and cookies come built in |
| Tokens are issued by an external IdP (Keycloak, Auth0, Entra, Zitadel) | **built-in `access_token` + `oidc` handler**: validates iss/aud/alg against a JWKS. No lexik needed |
| Machine/API keys, PATs, server-to-server | **built-in `access_token`** with opaque random tokens (≥256-bit) stored hashed, so revocation is instant |
| First-party browser app only | Consider **no JWT at all**: a Symfony session cookie (stateful) or the BFF pattern. JWTs bring revocation pain without stateless benefits for a single-origin app |

- **Opaque tokens:** require a DB/cache lookup on every request, but revocation is instant and no claims leak.
- **JWT:** stateless verification and cross-service friendly. Revocation needs a deny-list, a token version, or a short TTL.
- A hybrid is common: **short-lived JWT access token plus opaque, rotating, hashed refresh token**, which is exactly lexik + gesdinet.

---

## 4. Security best practices (checklist)

### Standards
- **RFC 8725 JWT BCP** (https://www.rfc-editor.org/rfc/rfc8725):
  - §3.1: verify the algorithm against an allow-list; one key = one alg.
  - §3.2: no `none`.
  - §3.5: HMAC keys need real entropy, never passwords.
  - §3.8: validate iss/sub.
  - §3.9: validate `aud` when the issuer serves more than one relying party.
  - §3.10: sanitise `kid`; allow-list `jku`/`x5u` (SSRF).
  - §3.11: explicit `typ`.
  - §3.12: mutually exclusive validation rules across token kinds.
- **draft-ietf-oauth-rfc8725bis** (rev -10, 2026-08-21) will **obsolete RFC 8725**. It tightens aud/iss/sub/typ validation and restricts `alg: none`. Status is still an Internet-Draft. See https://datatracker.ietf.org/doc/draft-ietf-oauth-rfc8725bis/
- **RFC 9068 JWT access-token profile** (https://www.rfc-editor.org/rfc/rfc9068):
  - Header `typ: at+jwt`.
  - Required claims: `iss`, `exp`, `aud`, `sub`, `client_id`, `iat`, `jti`.
  - Resource servers MUST check `typ`, `iss`, `aud`, signature, and `exp`, and MUST reject `alg: none`.
  - Following it is cheap even when not doing full OAuth: add typ, aud, jti, and client_id. [knowledge; not re-fetched]
- **RFC 9700 OAuth 2.0 Security BCP** (Jan 2025): refresh token rotation or sender-constraining for public clients; prefer sender-constrained tokens (DPoP RFC 9449 / mTLS).
- **OWASP JWT Cheat Sheet** (https://cheatsheetseries.owasp.org/cheatsheets/JSON_Web_Token_Cheat_Sheet.html):
  - Reject `none` and hard-code algorithms.
  - Key the deny-list on `jti` (+`iss`), not on a token hash (malleability).
  - Keep sensitive data out of claims (they are only base64url-encoded); use JWE only if necessary.
  - HMAC secrets: ≥ hash-size bits from a CSPRNG, never reused across purposes.
  - Consider DPoP/mTLS sender-constraint.
  - (The old Java cheat sheet "fingerprint cookie" sidejacking pattern is gone. The current page points to DPoP.)

### Attacks to cover, and how this stack handles them
| Attack | Mitigation in this stack |
|---|---|
| `alg: none` / alg confusion (RS→HS with public key as HMAC secret) | lexik/lcobucci uses a config-fixed signer. web-token uses `allowed_algorithms: [ES256]`. **Never allow HS\* and RS/ES\* together.** Never mix key types under one config |
| `kid` injection (path traversal or SQLi via kid lookup) | Never use `kid` as a file path or SQL input. Look kid up in an in-memory JWKSet only |
| `jku`/`x5u`/embedded `jwk` header trust | Ignore them. Only use locally configured keys. The oidc discovery URL comes from config |
| Weak HMAC secret / brute force offline | Prefer asymmetric (ES256). If HS256: ≥256-bit random secret from the vault |
| Token replay after logout | Short TTL (5–15 min) + lexik blocklist (jti) + gesdinet `block_previous_jwt` |
| Stolen refresh token | `single_use` + `reuse_detection` + `max_session_lifetime` + `hash_tokens` + httpOnly cookie scoped to the refresh path |
| Credential stuffing | `login_throttling` (+ app-level rate limiter on /api/auth), uniform error messages, timing-safe user lookup (Symfony hashes dummy passwords for unknown users [U]) |
| Token in URL | Disable lexik `query_parameter` and the access_token `query_string` extractor |
| Cross-token confusion (ID token, email-verify token, or reset token accepted as access token) | Distinct `typ`/`aud`, and separate keys for different token purposes (RFC 8725 §3.12) |
| Stale privileges | Don't trust `roles` in the JWT: reload the user from the DB (the default entity provider does), or keep TTL very short. Use a `tokenVersion` bump on role/password change |

### Validation settings
- `token_ttl`: 300–900 s for access tokens.
- Refresh TTL: 7–30 days, with `max_session_lifetime`.
- `clock_skew`: 30–60 s (lexik default 0 is strict; small skew avoids multi-node clock drift 401s).
- Always issue and validate `iss`, `aud`, `exp`, `iat`, `sub` (opaque UUID), `jti`, and the `typ` header.
- Keep `allow_no_expiration: false`.

### Browser token storage
- **localStorage/sessionStorage:** any XSS can exfiltrate the token. Avoid this for anything long-lived. Never put the refresh token there.
- **httpOnly + Secure + SameSite cookies:** XSS can still *use* the session while the page is open, but cannot steal the token. Requires CSRF thinking.
  - lexik `set_cookies` for the access token.
  - gesdinet `cookie.enabled` for the refresh token, with path scoped to `/api/token/refresh`.
- **In-memory access token + httpOnly refresh cookie:** a common SPA compromise. The access token lives in a JS variable and is restored on reload via a silent refresh.
- **BFF (Backend-for-Frontend):** recommended by the IETF draft *OAuth 2.0 for Browser-Based Applications* (draft-ietf-oauth-browser-based-apps). Tokens never reach the browser. The BFF holds them and uses a session cookie to talk to the SPA. It is the strongest option for high-value apps, at the cost of one more component (or Symfony itself acting as the BFF with a stateful session).
- **Mobile:** use Keychain/Keystore and Bearer headers; cookies are not needed.

### CSRF when using cookies
- Cookie auth brings CSRF back. Mitigations:
  - `SameSite=Strict` (or Lax plus no state-changing GET).
  - Require a custom header (`X-Requested-With` / `Content-Type: application/json` forces CORS preflight).
  - Strict CORS allow-list (`nelmio/cors-bundle`) with `allow_credentials` only for known origins.
  - Optionally a double-submit token. Symfony's stateless CSRF (`framework.csrf_protection.stateless_token_ids`, 7.2+) checks the Origin/Referer header plus a double-submit cookie [U applicability to API Platform].
- SameSite depends on the *site*, not the origin, so subdomains under the same registrable domain count as same-site. Lock down sibling subdomains.

### Revocation strategies (combine them)
1. Short access TTL (the baseline).
2. Refresh rotation + reuse detection (gesdinet v3).
3. jti deny-list in Redis for logout (lexik `blocklist_token`).
4. Per-user cutoff: gesdinet `block_jwts_on_revocation`, which refuses JWTs whose `iat` is at or before the revocation moment. Alternatively, a `tokenVersion` int on User, embedded as `ver` and compared on each request. Bump it on password change, role change, account disable, or "log out all devices".
5. Disable the account: Symfony `UserChecker::checkPostAuth()` runs on JWT auth too [U for lexik jwt authenticator; it is a standard Passport flow so it should].

### Key management
- Never commit production keys. `config/jwt/*.pem` should be in `.gitignore` (Flex adds it).
- Load keys from a secrets manager (Vault, AWS Secrets Manager/KMS, GCP Secret Manager) or Symfony secrets (`secrets:set JWT_PASSPHRASE`). Raw key content is also accepted in `secret_key`/`public_key` [V: "path ... OR raw key"].
- Encrypt the private key with a passphrase. Use separate keys per environment. Use dedicated test keys in `config/packages/test/`.
- **Rotate** on a schedule (e.g. every 90 days) and immediately on compromise, using an overlap window ≥ max TTL (`additional_public_keys` or a JWKSet). Give each key a `kid` (web-token).
- If other services verify your tokens, publish a JWKS endpoint (web-token makes this easy) [U, no lexik built-in].

### Logging / PII
- Never log tokens, refresh tokens, Authorization headers, or passwords. Configure Monolog processors and redact headers in the access logs of your proxy/APM.
- Keep PII out of claims: use a UUID `sub`, not the email. Remember the JWT payload is readable by anyone holding it.
- Log authentication events: login success/failure, refresh, reuse detection (security alert), logout, and revocation. Include user id, IP, user-agent, and jti, not the token.
- Uniform failure messages; no user enumeration.

### Rate limiting
- `login_throttling` on the firewall (symfony/rate-limiter).
- gesdinet `rate_limiter` on refresh.
- Also limit registration, password reset, and email verification with `#[RateLimit]`-style listeners or `RateLimiterFactory` [U attribute availability in 8.x; the factory approach is V].
- Set `framework.trusted_proxies` behind a load balancer, or every request appears to come from the same IP.

---

## 5. Testing (Pest/PHPUnit + API Platform 5)

Test keys:
```yaml
# config/packages/test/lexik_jwt_authentication.yaml
lexik_jwt_authentication:
    secret_key: '%kernel.project_dir%/config/jwt/private-test.pem'
    public_key: '%kernel.project_dir%/config/jwt/public-test.pem'
    pass_phrase: 'test'
```
Committing test keys for CI is fine. Alternatively, use HMAC in tests with `secret_key: <random>` and `signature_algorithm: HS256`, though keeping the same alg as production is better.

API Platform 5 test base class: `ApiPlatform\Test\ApiTestCase` (the old `ApiPlatform\Symfony\Bundle\Test\ApiTestCase` is deprecated in 5.0) [V].
- `protected static ?bool $alwaysBootKernel = false;`
- `createClient(array $kernelOptions = [], array $defaultOptions = [])`
- The client supports the HttpClient options `auth_bearer`/`auth_basic`, `loginUser($user, $firewallContext = 'main')`, and `disableReboot()` [V, source v5.0.3].

```php
use ApiPlatform\Test\ApiTestCase;
use Lexik\Bundle\JWTAuthenticationBundle\Services\JWTTokenManagerInterface;
use Zenstruck\Foundry\Test\{Factories, ResetDatabase};

final class BookTest extends ApiTestCase
{
    use ResetDatabase, Factories;
    protected static ?bool $alwaysBootKernel = true;

    private function tokenFor(User $user): string
    {
        return static::getContainer()->get(JWTTokenManagerInterface::class)->create($user);  // fires JWT_CREATED (claims listener runs)
    }

    public function testRequiresAuth(): void
    {
        static::createClient()->request('GET', '/api/books');
        $this->assertResponseStatusCodeSame(401);
    }

    public function testListWithJwt(): void
    {
        $user = UserFactory::createOne()->_real();   // [U] Foundry 2 API
        static::createClient()->request('GET', '/api/books', ['auth_bearer' => $this->tokenFor($user)]);
        $this->assertResponseIsSuccessful();
    }

    public function testLoginFlow(): void
    {
        UserFactory::createOne(['email' => 'a@b.c', 'plainPassword' => 'secret']);
        $r = static::createClient()->request('POST', '/api/auth', ['json' => ['email' => 'a@b.c', 'password' => 'secret']]);
        $this->assertResponseIsSuccessful();
        $this->assertArrayHasKey('token', $r->toArray());
    }
}
```
- Other ways to authenticate in tests:
  - `JWTEncoderInterface::encode($claims)` for a raw custom payload. Use it to test expired, wrong-aud, wrong-iss, or tampered tokens.
  - `$client->loginUser($user, 'api')` bypasses JWT entirely. It is fine for authorization tests, not for auth tests.
- Security tests worth writing:
  - expired token → 401
  - `alg: none` token → 401
  - token signed with another key → 401
  - wrong `aud`/`iss` → 401
  - token after logout (blocklist) → 401
  - refresh reuse → family revoked
  - refresh with rotated token → old one is rejected
  - login throttling → 429 after N failures
  - query-string token is ignored
- Clock: lexik 3.x uses `symfony/clock` when available (PR #1204). In tests, use `Symfony\Component\Clock\Test\ClockSensitiveTrait` / `MockClock` to time-travel past `exp` [U that every lexik code path honours the clock].
- Blocklist and reuse detection in tests need a cache pool that is shared within the test kernel (`cache.adapter.array` per kernel). With `disableReboot()`, multi-request flows keep state.
- `security:oidc:generate-token` produces tokens if you use Symfony's oidc handler instead.

---

## Recommended baseline for this project
1. Install lexik 3.2 (or `3.x-dev` if running FrankenPHP worker mode with the DB-less provider) and gesdinet 3.0.
2. Keys: ES256 via lcobucci. Keys come from the secrets vault, are encrypted with a passphrase, and are rotated with `additional_public_keys`.
3. Access token: `token_ttl: 600`, `clock_skew: 30`. Claims: `iss`, `aud`, `sub`=UUID, `jti`, `ver`, and `typ: at+jwt`. Validate iss/aud in a `JWT_DECODED` listener. Reload the user from the DB; don't trust roles in the token.
4. Refresh: `single_use`, `reuse_detection`, `hash_tokens`, `max_session_lifetime`, `block_previous_jwt`, `block_jwts_on_revocation`, `rate_limiter`. Refresh cookie: httpOnly, Secure, SameSite=Strict, path=/api/token/refresh.
5. Web SPA:
   - Option 1: access token in memory, refresh token in an httpOnly cookie.
   - Option 2: lexik `set_cookies` for the access token plus CSRF defenses.
   - Option 3: a BFF for high-value apps.
6. Mobile: Bearer header with the refresh token in the body (gesdinet cookie off, or per-client handling; note cookie config is global in v3).
7. Firewall: single `stateless` firewall with `json_login` + `jwt` + `refresh_jwt` + `login_throttling` + `logout`, plus the Redis blocklist.
8. Docs: lexik `api_platform`, gesdinet `api_platform.enabled`, and `swagger.api_keys.JWT`.

## Sources
- https://github.com/lexik/LexikJWTAuthenticationBundle (tag v3.2.0: composer.json, DependencyInjection/Configuration.php, Services/JWSProvider/LcobucciJWSProvider.php, Command/GenerateKeyPairCommand.php, Events.php, Resources/doc/*.rst on branch 3.x)
- https://github.com/lexik/LexikJWTAuthenticationBundle/releases
- https://packagist.org/packages/lexik/jwt-authentication-bundle
- https://github.com/markitosgv/JWTRefreshTokenBundle (tag v3.0.0: README.md, UPGRADE-3.0.md, CHANGELOG.md, composer.json)
- https://packagist.org/packages/gesdinet/jwt-refresh-token-bundle
- https://symfony.com/doc/current/security/access_token.html (8.1)
- https://symfony.com/doc/current/security.html (8.1: login_throttling, json_login, logout)
- https://api-platform.com/docs/symfony/jwt/ (5.0)
- https://api-platform.com/docs/symfony/testing/ (5.0)
- https://github.com/api-platform/core (tag v5.0.3: src/Test/ApiTestCase.php, src/Test/Client.php)
- https://github.com/web-token/jwt-framework/releases
- https://www.rfc-editor.org/rfc/rfc8725.html
- https://datatracker.ietf.org/doc/draft-ietf-oauth-rfc8725bis/ (rev -10, 2026-08-21)
- https://www.rfc-editor.org/rfc/rfc9068.html
- https://www.rfc-editor.org/rfc/rfc9700.html
- https://datatracker.ietf.org/doc/draft-ietf-oauth-browser-based-apps/
- https://cheatsheetseries.owasp.org/cheatsheets/JSON_Web_Token_Cheat_Sheet.html
