# User entity & identity processes — Symfony 8.1 / API Platform 5 / Doctrine ORM 3 / PHP 8.4

Researched 2026-10-12. Sources: symfony/symfony `8.1` branch source and CHANGELOGs, symfony-docs `8.1` branch (rst), Packagist p2 metadata, vendor docs, OWASP, NIST SP 800-63B-4.
Legend: **[V]** verified against source/docs fetched today; **[U]** unverified syntax or design sketch (compile-check before use).

---

## 0. Version matrix (from Packagist, 2026-10-12) [V]

| Package | Latest | Released | Symfony constraint | PHP |
|---|---|---|---|---|
| symfony/security-bundle | v8.1.8 | 2026-09-24 | 8.1 (accepts ^7.4\|^8.0 deps) | >=8.4.1 |
| symfony/validator / uid | v8.1.8 | 2026-09 | — | >=8.4.1 |
| symfony/rate-limiter | v8.1.6 | 2026-08-07 | — | >=8.4.1 |
| api-platform/core, api-platform/symfony | v5.0.3 | 2026-10-09 | http-kernel/security-core ^7.4 \|\| ^8.0 | >=8.2 |
| doctrine/orm | 3.7.5 | 2026-10-09 | — | ^8.1 |
| doctrine/doctrine-bundle | 3.3.2 | 2026-09-09 | framework-bundle ^6.4\|\|^7\|\|^8 | ^8.4 |
| symfony/maker-bundle | 1.68.0 | 2026-09-12 | **^7.4\|^8.1** | >=8.2 |
| symfonycasts/verify-email-bundle | v1.18.0 | 2025-11-29 | ^5.4\|^6\|^7\|**^8.0** (first 8.x-compatible) | >=8.1 |
| symfonycasts/reset-password-bundle | v1.25.0 | 2026-03-26 | ^5.4\|^6\|^7\|**^8.0** (8.x since v1.24.0) | >=8.1.10 |
| scheb/2fa-bundle (+ -totp, -email, -backup-code, -trusted-device) | v8.6.1 | 2026-07-10 | ^7.4 \|\| ^8.0 | ~8.4.0 \|\| ~8.5.0 |
| web-auth/webauthn-symfony-bundle (+ webauthn-lib) | 5.3.9 | 2026-09-10 | ^6.4\|^7\|^8.0 | >=8.2 |
| lexik/jwt-authentication-bundle | v3.2.0 | 2025-12-20 | ^6.4\|^7\|^8.0 | >=8.2 |
| gesdinet/jwt-refresh-token-bundle | v3.0.0 | 2026-08-05 | **^8.0 only** (v2.2.2 for 6.4/7.x/8) | ^8.4 |

Notes: Symfony 8.1.0 released ~29 May 2026; 8.2 is dev. scheb v8.6.0 contained security fixes (constant-time `hash_equals` for code comparison #318, code-reuse cache pre-poisoning fix #317) → require `^8.6`.
Symfony 8.1 RC-era security fixes include CVE-2026-45066 (HtmlSanitizer), CVE-2026-45069 (OIDC token handler missing claims), CVE-2026-45075 (HEAD request bypassing a security check in HttpKernel) — stay on latest 8.1.x patch.

---

## 1. User model

### 1.1 Interfaces (Symfony 8.1 source) [V]
```php
interface UserInterface {
    public function getRoles(): array;            // string[]
    public function getUserIdentifier(): string;  // non-empty-string
}
interface PasswordAuthenticatedUserInterface { public function getPassword(): ?string; }
interface LegacyPasswordAuthenticatedUserInterface extends PasswordAuthenticatedUserInterface { public function getSalt(): ?string; } // only for legacy salted hashers
interface EquatableInterface { public function isEqualTo(UserInterface $user): bool; }
interface UserCheckerInterface {
    public function checkPreAuth(UserInterface $user): void;
    public function checkPostAuth(UserInterface $user, ?TokenInterface $token = null): void; // $token added (7.2 opt, 8.0 signature)
}
interface UserProviderInterface { // @template-covariant TUser
    public function refreshUser(UserInterface $user): UserInterface;
    public function supportsClass(string $class): bool;
    public function loadUserByIdentifier(string $identifier): UserInterface;
}
interface AttributesBasedUserProviderInterface extends UserProviderInterface {
    public function loadUserByIdentifier(string $identifier, array $attributes = []): UserInterface; // attrs come from UserBadge($id, $loader, $attributes)
}
```
- `getUserIdentifier()` returning `''` was deprecated in 7.2; in 8.0 `new UserBadge('')` throws `BadCredentialsException`.
- **eraseCredentials() lifecycle** [V]:
  - 7.3: `UserInterface::eraseCredentials()` + `TokenInterface::eraseCredentials()` deprecated; add `#[\Deprecated]` on your method → Symfony stops calling it.
  - **8.0: removed** from both interfaces.
  - 8.1: `security.erase_credentials` config option + `security.authentication.manager.erase_credentials` param + `AuthenticatorManager::__construct($eraseCredentials)` deprecated (no effect; removed in 9.0). → **delete `erase_credentials:` from security.yaml.**
  - Replacement: don't hold plaintext on the entity (use a DTO/form model for `plainPassword`), and control session payload via `__serialize()`.
- **Session serialization** (docs "Understanding how Users are Refreshed from the Session") [V]: the user object is serialized into the session each request (stateful firewalls), then refreshed via provider; `AbstractToken` compares `getPassword()`, `getSalt()`, `getUserIdentifier()` → mismatch = logout. Recommended: store a **crc32c of the hash** instead of the hash:
  ```php
  public function __serialize(): array
  {
      $data = (array) $this;
      $data["\0".self::class."\0password"] = hash('crc32c', $this->password);
      return $data;
  }
  ```
  Effect: password change invalidates other sessions; real hash never sits in session storage. Also exclude Doctrine relations from serialization. 8.1 RC also "initialize lazy users before serializing" (relevant w/ ORM 3 native lazy objects).
- `EquatableInterface::isEqualTo()` — implement to also compare e.g. `isEnabled`, `tenantId`, `securityStamp/tokenVersion` so admin disable / role change forces re-auth. Don't compare by id only (docs warn).

### 1.2 Providers [V]
```yaml
security:
  providers:
    app_user_provider:
      entity: { class: App\Entity\User, property: email }   # or omit property + repo implements UserLoaderInterface
      # manager_name: customer
    # memory: { users: { admin: { password: '...', roles: ['ROLE_ADMIN'] } } }
    # chain:  { providers: ['in_memory', 'app_user_provider'] }
    # custom: { id: App\Security\UserProvider }
```
- `Symfony\Bridge\Doctrine\Security\User\UserLoaderInterface::loadUserByIdentifier(string): ?UserInterface` on the repository for custom lookups (email OR username, case-folding, tenant filter).
- `UserChainProvider` implements `AttributesBasedUserProviderInterface` since 8.0.
- Custom provider must also implement `PasswordUpgraderInterface` to get rehash-on-login.

### 1.3 UserChecker [V]
- Throw `AccountStatusException` subclasses (`DisabledException`, `LockedException`, `AccountExpiredException`, `CredentialsExpiredException`, or `CustomUserMessageAccountStatusException`).
- **Called on every authentication**, incl. every remember-me auth and **every request on a stateless firewall**; `checkPostAuth()` also on impersonation. Keep fast, no side effects. Use `LoginSuccessEvent` for "last login" writes.
- Typical split: `checkPreAuth` → deleted/disabled/locked (before password check — but see enumeration note in §4.9); `checkPostAuth` → unverified email, password expired-by-compromise, tenant suspended.
- Multiple checkers: tag `security.user_checker.<firewall>` with priority → `ChainUserChecker`.
- Config: `firewalls.main.user_checker: App\Security\UserChecker`.

### 1.4 make:user [V/U]
- `symfony/maker-bundle` 1.68.0 now requires Symfony ^8.1 on the 8 line. `bin/console make:user` generates entity/repo implementing `UserInterface` + `PasswordAuthenticatedUserInterface`, repo with `PasswordUpgraderInterface`, and security.yaml provider/hasher. [U] check whether 1.68 still emits `eraseCredentials()` — delete it if present (removed in 8.0).
- Attribute-based Doctrine mapping is what Maker generates; for DDD use XML (see §2).

### 1.5 Roles [V]
- `getRoles()` must guarantee `ROLE_USER`; `role_hierarchy:` in security.yaml. 7.4: `debug:security:role-hierarchy` (Mermaid). 8.1: `RoleHierarchyInterface::getParentRoleNames()`.
- Special attributes: `PUBLIC_ACCESS`, `IS_AUTHENTICATED`, `IS_AUTHENTICATED_FULLY`, `IS_AUTHENTICATED_REMEMBERED`, `IS_IMPERSONATOR`, `IS_REMEMBERED`.
- Roles are global strings — **don't encode tenant roles as `ROLE_TENANT_42_ADMIN`**; use voters with a membership lookup (see §5).

### 1.6 Current user / Security helper [V]
- Controllers: `#[CurrentUser] ?User $user` (`Symfony\Component\Security\Http\Attribute\CurrentUser`), supports union types.
- Services: `Symfony\Bundle\SecurityBundle\Security` → `getUser()`, `isGranted()`, `getAccessDecision()` (7.4), `isGrantedForUser()` / `getAccessDecisionForUser()` (7.3/7.4, auth check without session — good for workers/messenger), `getToken()`, `getFirewallConfig()`, `login($user, $authenticator, $firewall, $badges, $attributes)`, `logout($validateCsrf = true)`.

### 1.7 Impersonation (switch_user) risks [V + analysis]
- `switch_user: { role: CAN_SWITCH_USER, parameter: X-Switch-User, target_route: ... }`; default role `ROLE_ALLOWED_TO_SWITCH`; `_switch_user=_exit`; `IS_IMPERSONATOR` attribute; `SwitchUserEvent`.
- Risks: privilege escalation (admin impersonating super-admin), cross-tenant access, actions mis-attributed in audit logs, header form on a stateless API firewall = any request can switch.
- Controls: custom voter on `CAN_SWITCH_USER` with subject = target user (deny higher-privilege targets, different tenant), require `IS_AUTHENTICATED_FULLY` + MFA-completed, audit `SwitchUserEvent` incl. `SwitchUserToken::getOriginalToken()`, record impersonator id on every write (audit log "actor" vs "subject"), block sensitive ops (password/MFA/email change) while `IS_IMPERSONATOR`. `checkPostAuth()` runs on switch; `LoginSuccessEvent` does not.

---

## 2. DDD / hexagonal

### 2.1 Pattern [U — design sketch]
- **Domain** (`src/Identity/Domain`): `User` aggregate — framework-free (no Symfony interfaces, no Doctrine attributes). Value objects: `UserId` (UUIDv7), `EmailAddress` (normalized lower-case, NFC), `HashedPassword` (opaque string), `UserStatus` enum (`PendingVerification|Active|Locked|Disabled`), `TenantMembership` (tenantId, roles/permissions). Domain events: `UserRegistered`, `EmailVerified`, `PasswordChanged`, `UserLocked`.
- **Domain ports**: `UserRepository`, `PasswordHasher` (port; adapter wraps `PasswordHasherInterface`), `Clock`.
- **Infrastructure** (`src/Identity/Infrastructure/Security`): `SecurityUser implements UserInterface, PasswordAuthenticatedUserInterface, EquatableInterface` — a small immutable read-model built from the aggregate (id, identifier, hash, roles, status, tenantId, tokenVersion). `SecurityUserProvider implements UserProviderInterface, PasswordUpgraderInterface` loads aggregate via `UserRepository` and maps.
- Why: keeps Symfony session serialization, `getRoles()` semantics, and rehash plumbing out of the domain; SecurityUser is tiny → small session payload and no lazy Doctrine relations serialized.
```php
// [U]
final readonly class SecurityUser implements UserInterface, PasswordAuthenticatedUserInterface, EquatableInterface
{
    /** @param list<string> $roles */
    public function __construct(
        public string $id, private string $identifier, private ?string $passwordHash,
        private array $roles, public ?string $tenantId, public int $tokenVersion, public bool $active,
    ) {}
    public function getUserIdentifier(): string { return $this->identifier; } // use the UUID, not email (see 2.3)
    public function getRoles(): array { return array_values(array_unique([...$this->roles, 'ROLE_USER'])); }
    public function getPassword(): ?string { return $this->passwordHash; }
    public function isEqualTo(UserInterface $user): bool {
        return $user instanceof self && $user->id === $this->id && $user->tokenVersion === $this->tokenVersion
            && $user->active === $this->active && $user->getRoles() == $this->getRoles();
    }
    public function __serialize(): array { $d = get_object_vars($this); $d['passwordHash'] = $this->passwordHash ? hash('crc32c', $this->passwordHash) : null; return $d; }
    // NB: readonly + __unserialize needs care; or drop readonly. [U]
}

final class SecurityUserProvider implements UserProviderInterface, PasswordUpgraderInterface
{
    public function __construct(private UserRepository $users) {}
    public function loadUserByIdentifier(string $identifier): UserInterface { /* find by id or email; throw UserNotFoundException */ }
    public function refreshUser(UserInterface $user): UserInterface { /* reload by id, re-map */ }
    public function supportsClass(string $class): bool { return $class === SecurityUser::class; }
    public function upgradePassword(PasswordAuthenticatedUserInterface $user, string $newHashedPassword): void { /* load aggregate, ->rehashPassword(new HashedPassword(...)), save */ }
}
```
- Gotcha: with crc32c in session, `refreshUser()` must return a user whose `getPassword()` is the real hash; Symfony hashes the refreshed one and compares [V doc behavior].
- `#[CurrentUser] SecurityUser $user` in controllers; application layer receives `UserId` only.

### 2.2 Doctrine mapping [V for bundle facts, U for XML]
- DoctrineBundle 3.x: **YAML and annotation drivers dropped**; use `attribute` or `xml`. Requires PHP 8.4, ORM 3, persistence 4. Proxy options removed (native lazy objects on PHP 8.4).
```yaml
# config/packages/doctrine.yaml [U]
doctrine:
  orm:
    mappings:
      Identity:
        type: xml
        is_bundle: false
        dir: '%kernel.project_dir%/src/Identity/Infrastructure/Persistence/Doctrine/Mapping'
        prefix: 'App\Identity\Domain'
        alias: Identity
```
```xml
<!-- User.orm.xml [U] -->
<doctrine-mapping xmlns="http://doctrine-project.org/schemas/orm/doctrine-mapping"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://doctrine-project.org/schemas/orm/doctrine-mapping https://www.doctrine-project.org/schemas/orm/doctrine-mapping.xsd">
  <entity name="App\Identity\Domain\User" table="users" repository-class="...">
    <id name="id" type="user_id"/>                       <!-- custom DBAL type wrapping symfony/uid Uuid -->
    <field name="tenantId" column="tenant_id" type="uuid"/>
    <embedded name="email" class="App\Identity\Domain\EmailAddress" use-column-prefix="false"/>
    <field name="passwordHash" column="password" type="string" length="255" nullable="true"/>
    <field name="status" type="string" enum-type="App\Identity\Domain\UserStatus"/>
    <field name="tokenVersion" type="integer"/>
    <unique-constraints><unique-constraint columns="tenant_id,email" name="uniq_user_tenant_email"/></unique-constraints>
  </entity>
</doctrine-mapping>
```
- Password column `varchar(255)` (docs: auto hash length may change).

### 2.3 Identifiers [V facts + recommendation]
- `symfony/uid` 7.4: `UuidFactory` defaults to **UuidV7** (+µs precision). 8.1: `Uuid47Transformer` converts UUIDv7↔UUIDv4-looking ids (SipHash-2-4 timestamp masking) so public ids don't leak creation time.
- Doctrine: use `Symfony\Bridge\Doctrine\Types\UuidType` (`uuid`) and **assign id in the aggregate constructor** (`Uuid::v7()`), no DB generator.
- Recommendation: **user identifier for security = UUID** (stable; email can change; avoids PII in remember-me cookies/JWT `sub`/logs). Login still by email → `UserBadge($email, fn($e) => $provider->loadByEmail($e))`, while `getUserIdentifier()` returns UUID [U: verify remember-me/login_link reload by identifier works with your provider — they call `loadUserByIdentifier(getUserIdentifier())`, so the provider must accept UUID]. OWASP: random, non-enumerable IDs.
- Email normalisation: lower-case + trim + NFC; unique index on normalized value (per tenant if users are tenant-scoped).

### 2.4 Multi-tenant users [U — design]
- Two models: (a) **global identity + memberships** (`User` 1–N `Membership{tenantId, roles}`) — one login, switch org; (b) **tenant-scoped users** (`unique(tenant_id,email)`) — tenant resolved before auth (subdomain/header), pass it via `new UserBadge($email, null, ['tenant' => $tenantId])` to an `AttributesBasedUserProviderInterface` provider [V for API].
- Store active tenant in token attributes / JWT claim; validate membership on every request (voter or listener), never trust client-sent tenant header without membership check.
- Tenant-scoped roles: keep `getRoles()` for global/platform roles only (`ROLE_USER`, `ROLE_PLATFORM_ADMIN`); per-tenant roles/permissions in `Membership`, evaluated by voters (§5). Doctrine SQL filter for tenant isolation as defense-in-depth (and API Platform query extensions).

---

## 3. Password hashing & policy

### 3.1 Config [V]
```yaml
security:
  password_hashers:
    Symfony\Component\Security\Core\User\PasswordAuthenticatedUserInterface: 'auto'
    # named hasher (7.4+ injectable via #[Target('recovery_code')] PasswordHasherInterface)
    recovery_code: 'auto'
when@test:
  security:
    password_hashers:
      Symfony\Component\Security\Core\User\PasswordAuthenticatedUserInterface:
        algorithm: auto
        cost: 4          # bcrypt min
        time_cost: 3     # argon min
        memory_cost: 10  # argon min
```
- `auto` "currently Bcrypt" (8.1 docs) — bcrypt default cost **13** (range 4–31). Sodium = Argon2id via libsodium (96-char hash). PBKDF2 not recommended.
- To force Argon2id: `algorithm: sodium` (or `argon2id` [U naming in 8.x; docs list auto/bcrypt/sodium/pbkdf2]) with `migrate_from: [bcrypt]`. auto/native/bcrypt/argon automatically migrate from PBKDF2 and message-digest.
- **Bcrypt 72-byte truncation**: bcrypt silently uses only first 72 bytes; NIST says SHALL NOT truncate. Mitigation: use `sodium`/Argon2id, or cap max length at 64 chars (NIST "SHOULD permit at least 64") — 64 *chars* of multibyte UTF-8 can exceed 72 bytes, so prefer Argon2id if accepting Unicode. [Analysis; verify current Symfony bcrypt pre-hash behavior — U]
- Argon2id tuning: OWASP min `m=19 MiB, t=2, p=1` (or 46 MiB t=1). Symfony `memory_cost` is in KiB [U for exact units mapping].
- 8.1: `security:hash-password` warns when password passed as argv; supports `-` to read stdin.

### 3.2 Rehash on login [V]
- Automatic when authenticator uses `PasswordCredentials` (adds `PasswordUpgradeBadge`) and provider/repo implements `PasswordUpgraderInterface::upgradePassword(PasswordAuthenticatedUserInterface $user, string $newHashedPassword): void`.
- Custom hasher: return true from `needsRehash()`.
- Outside security: `PasswordHasherInterface::needsRehash()` + `hash()`.

### 3.3 NIST SP 800-63B-4 (final, Aug 2025) [V]
- Min length **15** if password is single-factor; **8** if only used in MFA. Max: SHOULD allow **≥64**. Accept ASCII printable + space, SHOULD accept Unicode (code point = 1 char), SHOULD NFC-normalize before hashing.
- **SHALL NOT** impose composition rules, require periodic change (force change on evidence of compromise), allow hints, use KBA/security questions, truncate.
- **SHALL** check against blocklist (breached, dictionary, context-specific e.g. service name/username) — whole-password compare; tell user why rejected.
- SHALL allow password managers/autofill; SHOULD allow paste; SHOULD offer show-password.
- Storage: salted (≥32-bit salt) + password hashing scheme; SHOULD add keyed op (pepper in HSM/secret).
- Rate limiting SHALL: ≤**100 consecutive failures** per authenticator; after success SHOULD disregard prior failures.
- AAL2 SHALL offer ≥1 **phishing-resistant** option (WebAuthn/passkeys); passwords aren't phishing-resistant.

### 3.4 Validator constraints [V]
```php
use Symfony\Component\Validator\Constraints as Assert;
final class RegisterUserInput {
    #[Assert\NotBlank, Assert\Length(min: 15, max: 128)]   // 15 if password-only; 8 if MFA mandatory
    #[Assert\NotCompromisedPassword(skipOnError: false)]    // HIBP k-anonymity: only first 5 hex of SHA-1 sent; threshold default 1
    #[Assert\PasswordStrength(minScore: Assert\PasswordStrength::STRENGTH_MEDIUM)] // entropy (length + unique chars); optional, NIST discourages composition rules
    public string $plainPassword;
}
```
- Disable HIBP calls in dev/test: `framework.validation.not_compromised_password: { enabled: false }` [U exact key path — docs reference `not_compromised_password` option].
- `skipOnError: true` = fail-open if HIBP down (availability vs security trade-off; log it).
- Add a context-specific blocklist validator (email local-part, product name) [U custom].

---

## 4. Flows & packages

### 4.1 Registration [U design]
- API Platform `Post` on a DTO input (`RegisterUserInput`) → processor → domain command. Never expose `password` in output groups.
- Uniform response "If this address can be used, we've sent a verification email" (enumeration); handle unique-constraint violations silently + send "you already have an account" email to existing user.
- Throttle by IP + email (`symfony/rate-limiter`).

### 4.2 Email verification — symfonycasts/verify-email-bundle **v1.18.0** (first with Symfony ^8.0) [V]
- `VerifyEmailHelperInterface::generateSignature(string $routeName, string $userId, string $userEmail, array $extraParams = []): VerifyEmailSignatureComponents`
- Validate: `validateEmailConfirmationFromRequest(Request $request, string $userId, string $userEmail)` (since 1.17; `validateEmailConfirmation(signedUrl,...)` deprecated).
- Config `config/packages/verify_email.yaml`: `symfonycasts_verify_email: { lifetime: 3600 }` (default 3600s).
- Stateless/signed-URL design: signature binds userId+email → changing email invalidates link. "Anonymous validation" pattern: put `id` in extraParams and load user in the verify route (works for SPAs/API).
- Unverified users: block in `UserChecker::checkPostAuth` or allow login with limited scope.

### 4.3 Password reset — symfonycasts/reset-password-bundle **v1.25.0** [V]
```yaml
symfonycasts_reset_password:
  request_password_repository: App\Repository\ResetPasswordRequestRepository
  lifetime: 3600
  throttle_limit: 3600        # >= lifetime → one active request per user
  enable_garbage_collection: true
```
- `ResetPasswordHelper`: `generateResetToken(object $user, ?int $lifetime = null)`, `validateTokenAndFetchUser(string)`, `removeResetRequest(string)`, `getTokenLifetime()`, **`generateFakeResetToken()`** (use when user not found → same response/timing).
- Token = selector + hashed verifier stored (split-token). After reset: hash new password, bump `tokenVersion`/invalidate sessions, revoke refresh tokens, notify user by email, require re-auth for MFA reset.
- For DDD: implement `ResetPasswordRequestInterface` in infrastructure; repo uses `ResetPasswordRequestRepositoryTrait` [V names from bundle; U for exact trait usage w/ XML mapping].

### 4.4 MFA — scheb/2fa-bundle **v8.6.1** (require ^8.6 for security fixes) [V]
- Packages: `scheb/2fa-bundle`, `scheb/2fa-totp`, `scheb/2fa-email`, `scheb/2fa-backup-code`, `scheb/2fa-trusted-device` (Google Authenticator provider also exists).
- User interfaces [V source 8.x]:
  - `Scheb\TwoFactorBundle\Model\Totp\TwoFactorInterface`: `isTotpAuthenticationEnabled()`, `getTotpAuthenticationUsername(): ?string`, `getTotpAuthenticationConfiguration(): ?TotpConfigurationInterface`
  - `Scheb\TwoFactorBundle\Model\Email\TwoFactorInterface`: `isEmailAuthEnabled()`, `getEmailAuthRecipient()`, `getEmailAuthCode(): ?string`, `setEmailAuthCode(string)`
  - `Scheb\TwoFactorBundle\Model\BackupCodeInterface`: `isBackupCode(string)`, `invalidateBackupCode(string)` — **store backup codes hashed** (use named `recovery_code` hasher), compare via hasher.
  - `Scheb\TwoFactorBundle\Model\TrustedDeviceInterface`: `getTrustedTokenVersion(): int` (bump to revoke all trusted devices).
  - In DDD these go on `SecurityUser` (infra), not the domain aggregate. TOTP secret encrypted at rest [U: custom Doctrine type / sodium secretbox].
- Bundle config (scheb_two_factor): `totp {enabled, issuer, server_name, leeway, parameters}`, `email {enabled, sender_email, digits, mailer, code_generator}`, `backup_codes {enabled, manager}`, `trusted_device {enabled, lifetime: 5184000, extend_lifetime, key, cookie_name, cookie_secure, cookie_same_site, cookie_domain, cookie_path}`, `security_tokens: [UsernamePasswordToken, PostAuthenticationToken]` (**must add `PostAuthenticationToken` for json_login/custom authenticators**), `ip_whitelist`, **`code_reuse_cache: cache.app`** (+ `code_reuse_cache_duration: 60`) to block code replay (NIST).
- Firewall `two_factor:` `auth_form_path`, `check_path`, `post_only`, `auth_code_parameter_name` (dot notation for JSON e.g. `data.authCode`), `trusted_parameter_name`, `multi_factor`, `enable_csrf`, `csrf_header`, `prepare_on_login`, `prepare_on_access_denied`, `success_handler`, `failure_handler`, `authentication_required_handler`.
- **API/JSON**: needs a **stateful firewall** (2FA-in-progress state in session). Flow: `json_login` success_handler returns `{"login":"success","two_factor_complete":false}` when token is `TwoFactorTokenInterface` → client POSTs code to check_path → success_handler issues real token (e.g. JWT). Handlers default to HTTP 200 — set 401/403 yourself. Since v8.3: optional stateless providers via `needsPreparation`; v8.5 event `scheb_two_factor.authentication.skipped`.
- Pure stateless JWT + MFA alternative [U design]: issue short-lived "mfa_pending" token (scope-limited) after password, exchange for full access token after TOTP verify via custom authenticator.

### 4.5 Passkeys / WebAuthn — web-auth/webauthn-symfony-bundle **5.3.9** [V]
- `composer require web-auth/webauthn-symfony-bundle`; `webauthn.yaml`: `credential_repository` (implements `Webauthn\Bundle\Repository\PublicKeyCredentialSourceRepositoryInterface`; Doctrine helper `DoctrineCredentialSourceRepository`, entity extends `PublicKeyCredentialSource`), `user_repository` (`PublicKeyCredentialUserEntityRepositoryInterface`: `findOneByUsername`, `findOneByUserHandle`; registration via `CanRegisterUserEntity`), `creation_profiles.default.rp {name,id}`, `authenticator_selection_criteria`, `request_profiles.default.rp_id`, `user_verification`, metadata (MDS) optional.
- Firewall: `webauthn: ~` → routes `POST /login/options`, `POST /login`; `registration: { enabled: true }` → `/register/options`, `/register`. Token attributes `IS_USER_PRESENT`, `IS_USER_VERIFIED` usable in access_control (`roles: [ROLE_ADMIN, IS_USER_VERIFIED]`).
- Supports usernameless/discoverable credentials ("Authentication without username" page), fake credentials (anti-enumeration), counter checks. Use **UUID bytes as userHandle, never email**. Passkeys satisfy NIST AAL2 phishing-resistant requirement.

### 4.6 Magic login links (core `login_link`) [V]
```yaml
firewalls:
  main:
    login_link:
      check_route: login_check
      signature_properties: ['id', 'tokenVersion']  # any change invalidates links; 8.1 supports enum props
      lifetime: 300     # default 600
      max_uses: 1       # needs cache; default unlimited within lifetime
```
- `LoginLinkHandlerInterface::createLoginLink($user)` → `LoginLinkDetails::getUrl()`. Never display link in response; **configure `framework.trusted_hosts`** (Host-header poisoning leaks links). Email scanners may pre-fetch GET links → use an intermediate confirm page that POSTs [docs suggest; V general].

### 4.7 Login throttling / lockout [V]
- `composer require symfony/rate-limiter`; `login_throttling: { max_attempts: 5, interval: '1 minute' }` (defaults 5/min). Default `DefaultLoginRateLimiter`: `max_attempts` per **IP+username** and `5*max_attempts` per **IP**; custom limiter via `limiter:` service id (`DefaultLoginRateLimiter($globalFactory, $localFactory, $secret)`), 7.4+: `cache_pool` / `storage_service` options → use Redis-backed pool in multi-node.
- Throttling ≠ lockout. Add per-account lockout counter (OWASP: track per account, exponential backoff, cap; allow reset-password to unlock; avoid DoS). NIST: hard cap ≤100 consecutive failures. Implement via `LoginFailureEvent` listener incrementing `failedAttempts` + `lockedUntil`, `UserChecker` throws `LockedException`, reset on `LoginSuccessEvent`. 8.1 RateLimiter: `FixedWindowLimiter` `$anchorAt`.
- json_login only throttled if `login_throttling` on that firewall; custom authenticators need `RateLimiter` manually or rely on `login_throttling` (works for authenticators using `UserBadge` — [U] confirm applies to custom).

### 4.8 remember_me, logout, session vs stateless [V]
- remember_me: `secret`, `lifetime` (default 31536000 = 1y → shorten to e.g. 14d), `signature_properties` (default `['password']`; add `tokenVersion`), `token_provider` (Doctrine persistent tokens) for revocability, `httponly: true` default, set `secure: true`, `samesite: lax|strict`. 8.0: user FQCN removed from cookie; `RememberMeBadge` required in custom authenticators.
- logout: `logout: { path: app_logout, clear_site_data: ['cookies','storage'] }` (8.1 adds `clientHints`,`prefetchCache`,`prerenderCache`); `invalidate_session: true` default; `LogoutEvent`. For APIs: revoke refresh token + denylist access-token jti.
- `session_fixation_strategy`: default `MIGRATE`.
- API Platform: prefer `stateless: true` firewall with bearer tokens (`access_token` authenticator with custom `AccessTokenHandlerInterface::getUserBadgeFrom()`, or OIDC handler, or lexik JWT v3.2 + gesdinet refresh v3.0). Docs warn: JWT handler must verify signature and `sub`,`iat`,`nbf`,`exp`. Stateful only for BFF/cookie SPA (then CSRF: `CsrfTokenBadge`/SameSite + `#[IsCsrfTokenValid]`), or scheb 2FA.
- Stateless note: UserChecker runs every request → keep cheap or cache status.

### 4.9 User-enumeration prevention [V + OWASP]
- `security.expose_security_errors`: `none` (default — all user-related exceptions → generic `BadCredentialsException`), `account_status` (show locked/disabled only if password correct), `all`. (`hide_user_not_found` removed in 8.0.)
- `CustomUserMessageAccountStatusException` messages ARE shown subject to that setting — keep `none` for public APIs.
- Timing: when user not found, Symfony's `CheckCredentialsListener` doesn't hash [U — verify; historically Symfony hashes a dummy to equalize? not confirmed]; add a dummy hash verify on not-found in custom providers to equalize timing.
- Same response/status for register, reset (use `generateFakeResetToken()`), verify-email resend, login-link request. Rate-limit those endpoints too.

### 4.10 Audit logging of auth events [V events]
- `CheckPassportEvent` (do custom checks), `AuthenticationTokenCreatedEvent` (modify token, MFA), `AuthenticationSuccessEvent` (last chance to fail), `LoginSuccessEvent` (log success, last-login, reset counters, rehash done), `LoginFailureEvent` (log failure + `getPassport()?->getBadge(UserBadge::class)?->getUserIdentifier()`), `InteractiveLoginEvent`, `LogoutEvent`, `TokenDeauthenticatedEvent` (session killed by user change), `SwitchUserEvent`. Plus scheb 2FA events (`scheb_two_factor.authentication.success/failure/require/skipped`) [U names except skipped].
- Firewall-scoped dispatchers: `security.event_dispatcher.<firewall>`.
- Log: event type, user id (not raw email/password), tenant, IP, UA, authenticator, MFA method, impersonator id, result reason; never log passwords/tokens/OTP. Append-only table or SIEM via Monolog channel `security`.

---

## 5. Authorization

### 5.1 Voters (8.x signatures) [V]
```php
protected function supports(string $attribute, mixed $subject): bool;
protected function voteOnAttribute(string $attribute, mixed $subject, TokenInterface $token, ?Vote $vote = null): bool;
// $vote?->addReason('...') — reasons surface in profiler / API Platform 403 detail in debug
```
- 8.0: `$vote` arg on `VoterInterface::vote()`; `$accessDecision` on `AccessDecisionManagerInterface::decide()` / `isGranted()`. 8.1: voter priority via `#[AsTaggedItem(priority: 10)]`.
- Strategies: `affirmative` (default), `consensus`, `unanimous`, `priority`. For tenant isolation consider `unanimous` or a high-priority TenantVoter that denies cross-tenant subjects [U design].
- `#[IsGranted]`: `#[IsGranted('ROLE_ADMIN')]`, `#[IsGranted('POST_EDIT', subject: 'post')]`, expressions `new Expression('...')`, 7.3+ closures `#[IsGranted(static fn (IsGrantedContext $ctx, mixed $subject) => ...)]`, **8.1: `this` variable** in subject expressions.
- `#[IsCsrfTokenValid]` (8.1 closures + `this` for id).

### 5.2 API Platform 5 security [V docs 5.0]
```php
#[ApiResource(security: "is_granted('ROLE_USER')")]
#[Get(security: "is_granted('BOOK_READ', object)")]
#[Put(securityPostDenormalize: "is_granted('BOOK_EDIT', object) and previous_object.owner == user")]
#[Post(securityPostDenormalize: "is_granted('BOOK_CREATE', object)", securityPostDenormalizeMessage: '...')]
#[Get(security: "...", securityMessage: 'Sorry...')]
#[ApiProperty(security: "is_granted('ROLE_ADMIN')", securityPostDenormalize: "is_granted('UPDATE', object)")]
// extraProperties: ['throw_on_access_denied' => true]  → 403 instead of silently dropping property
```
- Variables: `user`, `object`, `previous_object` (postDenormalize; implement `__clone` for deep), `request` (resource level), `access_decision` (4.4+).
- **Voters are not applied to GetCollection** → filter collections with Doctrine query extensions (`QueryCollectionExtensionInterface`/`QueryItemExtensionInterface`) — this is where tenant filtering lives. Don't combine `security` and `securityPostDenormalize` on same op.
- Throw `ApiPlatform\Metadata\Exception\AccessDeniedException($msg, detail: ...)` (old Symfony-bridge class deprecated since 4.4). Voter reasons only exposed with debug; keep messages non-revealing.
- Use 404 vs 403 for cross-tenant objects (prefer 404 to avoid existence leak) — implement in provider/extension.

### 5.3 RBAC + per-tenant permissions [U design]
- Global roles in `getRoles()` (+ `role_hierarchy`). Per-tenant: `Membership{userId, tenantId, roles[]}` + role→permission map (config or table). `TenantPermissionVoter` supports attributes like `project.edit`; subject carries `tenantId`; checks token user has membership in subject's tenant AND permission. Cache memberships per request.
- ACL-like (per-object grants): `ResourceGrant{subjectType, subjectId, principalId, permission}` checked in voters; Symfony ACL bundle is dead — don't use.
- Messenger/CLI: `Security::isGrantedForUser($user, 'project.edit', $project)` (no session).

---

## 6. Custom authenticators [V]

```php
use Symfony\Component\Security\Http\Authenticator\AbstractAuthenticator;
use Symfony\Component\Security\Http\Authenticator\Passport\{Passport, SelfValidatingPassport};
use Symfony\Component\Security\Http\Authenticator\Passport\Badge\{UserBadge, CsrfTokenBadge, RememberMeBadge, PasswordUpgradeBadge, PreAuthenticatedUserBadge};
use Symfony\Component\Security\Http\Authenticator\Passport\Credentials\{PasswordCredentials, CustomCredentials};

final class ApiPasswordAuthenticator extends AbstractAuthenticator
{
    public function supports(Request $request): ?bool { return $request->isMethod('POST') && $request->getPathInfo() === '/api/login'; }
    public function authenticate(Request $request): Passport
    {
        $data = $request->getPayload();
        return new Passport(
            new UserBadge((string) $data->get('email'), null /* or loader closure */, ['tenant' => $tenantId] /* attrs for AttributesBased provider */),
            new PasswordCredentials((string) $data->get('password')),   // auto adds PasswordUpgradeBadge if provider is PasswordUpgrader
            [/* new CsrfTokenBadge('login', $token), new RememberMeBadge() */],
        );
    }
    public function onAuthenticationSuccess(Request $r, TokenInterface $t, string $firewallName): ?Response { return null; /* or issue token */ }
    public function onAuthenticationFailure(Request $r, AuthenticationException $e): ?Response
    { return new JsonResponse(['message' => strtr($e->getMessageKey(), $e->getMessageData())], 401); }
}
```
- `UserBadge(string $userIdentifier, ?callable $userLoader = null, ?array $attributes = null, ?\Closure $identifierNormalizer = null)` [V source] — normalizer for case-folding/NFC.
- `SelfValidatingPassport(new UserBadge($id))` for API keys/tokens; `CustomCredentials(fn($cred, $user) => bool, $cred)`.
- `createToken()` default → `PostAuthenticationToken` (add to scheb `security_tokens`).
- `AuthenticationEntryPointInterface::start()` for 401 responses on stateless APIs.
- Register: `firewalls.main.custom_authenticators: [App\Security\ApiPasswordAuthenticator]`.

### json_login [V]
```yaml
firewalls:
  api_login:
    pattern: ^/api/login
    stateless: true
    json_login:
      check_path: api_login
      username_path: email          # default 'username'; dot paths allowed e.g. security.credentials.login
      password_path: password
      success_handler: lexik_jwt_authentication.handler.authentication_success  # or custom
      failure_handler: lexik_jwt_authentication.handler.authentication_failure
    login_throttling: { max_attempts: 5 }
```
- Client POSTs JSON (`Content-Type: application/json`); invalid creds → 401 JSON; success → controller (`#[CurrentUser] ?User`) or success_handler. Internally `JsonLoginAuthenticator` builds `Passport(UserBadge, PasswordCredentials, [RememberMeBadge])` → `CheckPassportEvent` listeners (`CheckCredentialsListener` verifies hash + upgrades; `UserCheckerListener` pre-auth; `LoginThrottlingListener`) → token → `LoginSuccessEvent`.

---

## 7. Security-expert checklist (summary)
1. Remove `eraseCredentials()` + `erase_credentials`; implement `__serialize()` crc32c strategy (stateful) and `EquatableInterface` with `tokenVersion`.
2. Separate `SecurityUser` adapter from domain `User`; XML mapping (DoctrineBundle 3 dropped YAML/annotations); UUIDv7 ids, security identifier = UUID.
3. Hasher: `auto` (bcrypt cost 13) or prefer `sodium` (Argon2id) + `migrate_from: [bcrypt]` to avoid bcrypt 72-byte truncation; `PasswordUpgraderInterface`; pepper optional.
4. NIST rev4: min 15 (8 if MFA mandatory), max ≥64, no composition rules, `NotCompromisedPassword` + context blocklist, no periodic rotation, allow paste.
5. `expose_security_errors: none`; uniform responses + fake reset token; timing equalization.
6. `login_throttling` (Redis pool) + per-account lockout ≤100 hard cap; throttle register/reset/link endpoints.
7. MFA via scheb ^8.6 (TOTP + hashed backup codes + trusted device w/ version), `code_reuse_cache`; offer passkeys (webauthn bundle 5.3) for phishing resistance; step-up (re-auth) for email/password/MFA changes.
8. Login links: short lifetime, `max_uses: 1`, `signature_properties`, `trusted_hosts`.
9. Stateless API firewalls with short-lived tokens + rotating refresh tokens (gesdinet v3 / lexik v3.2), revocation on password reset/logout.
10. Voters for all object/tenant checks; API Platform collection filtering via extensions; 404 for cross-tenant.
11. Impersonation gated by voter, tenant-bound, audited, sensitive ops blocked.
12. Audit all security events (no secrets in logs).

## Sources
- Symfony security-core CHANGELOG (8.1): https://github.com/symfony/symfony/blob/8.1/src/Symfony/Component/Security/Core/CHANGELOG.md
- Security-http / SecurityBundle / PasswordHasher / Uid / Validator / RateLimiter CHANGELOGs (same repo, `8.1` branch)
- UserInterface etc.: https://github.com/symfony/symfony/tree/8.1/src/Symfony/Component/Security/Core/User
- Docs (8.1): https://symfony.com/doc/current/security.html, /security/passwords.html, /security/user_providers.html, /security/user_checkers.html, /security/login_link.html, /security/custom_authenticator.html, /security/remember_me.html, /security/impersonating_user.html, /security/voters.html, /security/expressions.html, /security/access_token.html, /reference/configuration/security.html
- https://symfony.com/blog/new-in-symfony-7-3-security-improvements
- https://symfony.com/doc/current/reference/constraints/NotCompromisedPassword.html ; /PasswordStrength.html
- Packagist metadata: https://repo.packagist.org/p2/<vendor>/<package>.json
- verify-email: https://github.com/SymfonyCasts/verify-email-bundle ; reset-password: https://github.com/SymfonyCasts/reset-password-bundle
- scheb 2FA: https://symfony.com/bundles/SchebTwoFactorBundle/current/configuration.html , /api.html , https://github.com/scheb/2fa/releases
- WebAuthn v5: https://webauthn-doc.spomky-labs.com/v5.0/symfony-bundle/firewall.md , /configuration-references.md
- API Platform 5 security: https://api-platform.com/docs/symfony/security/
- DoctrineBundle 3 upgrade: https://github.com/doctrine/DoctrineBundle/blob/3.3.x/UPGRADE-3.0.md
- NIST SP 800-63B-4: https://pages.nist.gov/800-63-4/sp800-63b.html
- OWASP Authentication Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
