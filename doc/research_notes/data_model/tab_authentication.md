Authentication and identity: JWT, OAuth2 and users in Symfony
Written from: security and lead backend engineering view · Stack: PHP 8.4 · Symfony 8.1 · API Platform 5 · Doctrine ORM 3 · PostgreSQL · Checked: 12 October 2026 against Packagist, GitHub source, symfony.com, IETF, NIST and OWASP
This tab covers authentication: how people and systems prove who they are, how tokens are issued and checked, and every login integration. It is aligned with the User, access control and tenancy tab, which owns the data model (User, UserProfile, Role, Permission, RolePermission, UserRole, UserRoleAssignRole, Tenant, TenantDatabase) and authorisation. Items marked (verify) use syntax not confirmed in a primary source.
1. Decisions in one table
Topic
 | Decision
 | Why
 | 
Identity model
 | Global User (login and credentials only) in the landlord; one UserProfile per tenant says which societies the user belongs to
 | Profile pattern; one login across societies
 | 
Who issues tokens (now)
 | Colsynq issues its own: lexik/jwt-authentication-bundle 3.2 for access tokens, gesdinet/jwt-refresh-token-bundle 3.0 for refresh tokens
 | Only first-party clients today (web, mobile, admin)
 | 
Who issues tokens (later)
 | When third parties, banks or open banking need delegated access, an external identity provider (Keycloak 26 or similar) issues tokens; the API validates them with Symfony's oidc handler
 | A full OAuth2/OIDC server in-app means writing discovery, JWKS, introspection, PAR, DPoP and mTLS yourself
 | 
Token subject
 | sub = User ID (UUID v7). Never email
 | Email can change and is personal data
 | 
Tenant in the token
 | No. The tenant comes from the host or X-Tenant on each request; the server checks the user has an active profile there
 | One token works across the user's societies; switching society needs no new login
 | 
Roles in the token
 | None. Access is loaded from UserRoles on the current profile at every request
 | Revocation takes effect at once
 | 
Machine access
 | Service accounts: a User of kind service with a UserProfile in one tenant and UserRoles like anyone else; it authenticates with an API key now, client_credentials later
 | One access model for people and systems
 | 
Signing
 | ES256 with the default (lcobucci) encoder
 | Small, fast, asymmetric; rotation by old public keys
 | 
Access token life
 | 10 minutes, 30 s clock skew
 | Limits damage from a leak
 | 
Refresh token
 | Opaque, hashed, single use with rotation and reuse detection, 14 days, absolute cap 30 days
 | Detects theft
 | 
Passwords
 | Argon2id (sodium), migrating from bcrypt; NIST SP 800-63B-4
 | Bcrypt cuts passwords at 72 bytes
 | 
Second factor
 | TOTP + hashed backup codes (scheb/2fa-bundle ^8.6); passkeys for administrators and finance roles
 | NIST AAL2 needs a phishing-resistant option
 | 
Browser apps
 | Access token in memory, refresh token in an httpOnly, Secure, SameSite=Strict cookie on the refresh path; back-end-for-frontend for the platform console
 | Keeps tokens away from scripts (RFC 10017)
 | 
Support staff
 | Platform profile; tenant data only through a SupportAccessGrant
 | No standing access to societies
 | 
2. Packages and versions
Package
 | Version
 | Notes
 | 
symfony/security-bundle
 | 8.1.8
 | Needs PHP ≥ 8.4.1
 | 
api-platform/core
 | 5.0.3
 | Test base class ApiPlatform\Test\ApiTestCase
 | 
lexik/jwt-authentication-bundle
 | 3.2.0
 | First with Symfony 8; not tested by its authors on API Platform 5 but compatible. 3.x-dev only for FrankenPHP or RoadRunner worker mode
 | 
gesdinet/jwt-refresh-token-bundle
 | 3.0.0
 | PHP 8.4 / Symfony 8 only; reuse detection, session cap, rate limiter; new family columns
 | 
web-token/jwt-library
 | 4.2.3
 | For the oidc handler; GMP extension in production
 | 
league/oauth2-server-bundle
 | 1.2.2 (2.x unreleased)
 | Only if Colsynq must be its own OAuth2 server; turn password and implicit grants off explicitly
 | 
knpuniversity/oauth2-client-bundle
 | 2.20.2
 | Social login, if done in the API
 | 
symfonycasts/verify-email-bundle
 | 1.18.0
 | First with Symfony 8
 | 
symfonycasts/reset-password-bundle
 | 1.25.0
 |  | 
scheb/2fa-bundle
 | 8.6.1
 | Require ^8.6 (security fixes in 8.6.0)
 | 
web-auth/webauthn-symfony-bundle
 | 5.3.9
 | Passkeys
 | 
symfony/rate-limiter
 | 8.1
 | Login throttling and refresh limits
 | 
Installing any of these needs approval.
3. Principals: who can authenticate
Principal
 | Stored as
 | Authenticates with
 | Gets access through
 | 
Person
 | User (kind = person) + one UserProfile per society
 | Password + MFA, passkey, magic link, social login (later)
 | UserRoles on the profile of the society named in the request
 | 
Colsynq staff
 | User (kind = person) + a profile in the platform tenant
 | Password + MFA and passkey (required)
 | Platform UserRoles; tenant data only under a SupportAccessGrant
 | 
Service account (partner system, payroll file upload bot, integration)
 | User (kind = service, no password) + one UserProfile in its tenant
 | API key now; OAuth client_credentials later
 | UserRoles on its profile, usually narrow and unit-scoped
 | 
Webhook sender (bank, BaaS, payment gateway)
 | Connector configuration in the tenant
 | Signature (HMAC or JWS) + timestamp + replay check
 | Fixed handler; not a user
 | 
All principals except webhook senders resolve to a User, then to a UserProfile for the request's tenant, then to UserRoles. That keeps one path for every access decision.
4. How authentication connects to tenancy and access
sequenceDiagram
    autonumber
    participant C as Client
    participant TR as Tenant Resolver (kernel.request 20)
    participant AUTH as POST /api/auth (IAM)
    participant FW as Firewall (JWT)
    participant PR as Profile resolver
    participant ACC as Access context + voter
    C->>TR: society host or X-Tenant: slug
    TR->>TR: Tenant found and active? set TenantContext, switch tenant database
    C->>AUTH: email + password (+ MFA step)
    AUTH->>AUTH: ProfileLoginValidator: active profile in this tenant? (generic error if not)
    AUTH-->>C: access token (sub = userId) + refresh token; body: profile + access for this tenant
    C->>TR: next request, same host or X-Tenant
    TR->>FW: verify token, load User (status, token_version)
    FW->>PR: active UserProfile for (user, tenant)?
    PR-->>C: 404 if none
    PR->>ACC: UserRoles + RolePermissions (cached by access_version)
    ACC-->>C: 200 or 403
Sign-in on the main Colsynq domain (no tenant) is also allowed; the client then calls GET /api/me/profiles and picks a society. Sign-up always happens inside a resolved tenant and creates the UserProfile automatically (User tab, section 11).
Step
 | Module
 | Reads
 | 
Login, MFA, tokens
 | IAM
 | iam.app_user, iam.user_authenticator, iam.refresh_token
 | 
Society switcher
 | IAM
 | iam.user_profile (active profiles)
 | 
Tenant resolution
 | Tenancy
 | tenancy.tenant, tenancy.tenant_domain, tenancy.tenant_database
 | 
Profile resolution
 | IAM
 | iam.user_profile for (user, tenant)
 | 
Access context
 | IAM
 | iam.user_role, iam.role, iam.role_permission, iam.permission
 | 
Decision
 | IAM + owning module
 | AccessVoter, scope filters, assertions
 | 
All of these tables are in the landlord, so authentication and authorisation never open a tenant database; only business handlers do.
5. The User model in code
5.1 Domain and Symfony adapter
Class
 | Layer
 | Job
 | 
User aggregate (table iam.app_user)
 | IAM Domain
 | Register, verify email or phone, change password, lock and unlock, deactivate. No Symfony interfaces
 | 
UserProfile aggregate
 | IAM Domain
 | Create (at sign-up), disable, enable, remove; per-tenant details
 | 
UserAuthenticator aggregate
 | IAM Domain
 | TOTP, email OTP, passkey and backup-code authenticators of a User
 | 
SecurityUser
 | IAM Infrastructure
 | What Symfony security needs: ID, password hash, token_version, status, kind, platform-staff flag
 | 
SecurityUserProvider
 | IAM Infrastructure
 | Loads the User by ID (tokens) or email (login); derives the platform-staff flag from an active platform-tenant profile; rehashes passwords
 | 
CurrentProfile
 | IAM Infrastructure
 | Request-scoped: the active UserProfile for the resolved tenant
 | 
User columns (full list in the User, access control and tenancy tab, section 7.1): id, kind (person, service), email (normalised), phone, password_hash, status (active, deactivated), token_version, failed_login_count, locked_until, email_verified_at, phone_verified_at, trusted_device_version, last_login_at. Derived, not stored: locked (locked_until in the future), verified, MFA enabled (an active authenticator), platform staff (an active platform-tenant profile).
IAM authentication events: UserRegisteredV1, EmailVerifiedV1, PasswordChangedV1, UserLockedV1, UserUnlockedV1, AuthenticatorAddedV1, AuthenticatorRevokedV1, SessionsRevokedV1, UserDeactivatedV1, plus the profile and role events in the other tab. Every change is also written to iam.audit_event.
5.2 Symfony 8.1 interfaces
UserInterface has only getRoles() and getUserIdentifier(); the identifier must be non-empty.
eraseCredentials() was removed in 8.0; delete the security.erase_credentials option (deprecated, no effect in 8.1). Plain passwords live only in input DTOs.
EquatableInterface::isEqualTo() compares ID, token_version and status, so changes sign the user out.
UserCheckerInterface runs on every request of a stateless firewall: keep it fast and write-free.
5.3 The adapter (sketch, verify)
final readonly class SecurityUser implements UserInterface, PasswordAuthenticatedUserInterface, EquatableInterface
{
    public function __construct(
        public string $id,                 // UUID v7, the security identifier and JWT sub
        private ?string $passwordHash,     // null for service accounts and passkey-only users
        public int $tokenVersion,
        public string $status,             // active | deactivated
        public ?\DateTimeImmutable $lockedUntil,
        public bool $emailVerified,        // email_verified_at is not null
        public string $kind,               // person | service
        public bool $isPlatformStaff,      // derived: active profile in the platform tenant
    ) {}
    public function getUserIdentifier(): string { return $this->id; }
    /** Symfony roles are coarse markers only; permissions come from UserRoles via AccessVoter. */
    public function getRoles(): array
    {
        return $this->isPlatformStaff ? ['ROLE_USER', 'ROLE_PLATFORM_STAFF'] : ['ROLE_USER'];
    }
    public function getPassword(): ?string { return $this->passwordHash; }
    public function isLocked(\DateTimeImmutable $now): bool { return null !== $this->lockedUntil && $this->lockedUntil > $now; }
    public function isEqualTo(UserInterface $user): bool
    {
        return $user instanceof self
            && $user->id === $this->id
            && $user->tokenVersion === $this->tokenVersion
            && $user->status === $this->status;
    }
}
SecurityUserProvider::loadUserByIdentifier() accepts a UUID (token requests) or a normalised email (login); throws UserNotFoundException otherwise. It also implements PasswordUpgraderInterface.
UserChecker: checkPreAuth refuses deactivated users, users whose locked_until is in the future, and service accounts on password login; checkPostAuth refuses users without email_verified_at when the tenant requires verification.
CurrentProfile is resolved after the firewall (section 4). ROLE_PLATFORM_STAFF only opens /api/platform/* routes; what staff can do there still comes from their UserRoles in the platform tenant.
Doctrine XML mapping in IAM Infrastructure; table iam.app_user (user is reserved in PostgreSQL); unique normalised email.
6. Passwords
security:
    password_hashers:
        Symfony\Component\Security\Core\User\PasswordAuthenticatedUserInterface:
            algorithm: sodium          # Argon2id
            migrate_from: [bcrypt]
        recovery_code: 'auto'          # named hasher for backup codes
        api_key: 'auto'                # (optional) or SHA-256 lookup hash, see section 11.4
when@test:
    security:
        password_hashers:
            Symfony\Component\Security\Core\User\PasswordAuthenticatedUserInterface:
                algorithm: auto
                cost: 4
                time_cost: 3
                memory_cost: 10
NIST SP 800-63B-4 (August 2025): at least 15 characters when the password is the only factor (8 if MFA is always on), allow at least 64; no composition rules, no forced periodic change, no hints or security questions; check breached and context-specific lists (NotCompromisedPassword, plus words like "colsynq" and the society name); allow paste and password managers; normalise Unicode to NFC; at most 100 consecutive failures; rehash on login through PasswordCredentials and PasswordUpgraderInterface.
7. JWT access tokens (lexik 3.2)
7.1 Keys
composer require lexik/jwt-authentication-bundle
php bin/console lexik:jwt:generate-keypair
Set encoder.signature_algorithm: ES256 before generating. The default encoder supports only HS, RS and ES; EdDSA or PS256 need the web-token encoder.
Production keys live in the secrets vault, encrypted with a passphrase; never committed.
Rotation: publish the new key, keep the old public key in additional_public_keys for one token lifetime, then remove it.
7.2 Configuration
# config/packages/lexik_jwt_authentication.yaml
lexik_jwt_authentication:
    secret_key: '%env(resolve:JWT_SECRET_KEY)%'
    public_key: '%env(resolve:JWT_PUBLIC_KEY)%'
    pass_phrase: '%env(JWT_PASSPHRASE)%'
    additional_public_keys: []
    token_ttl: 600                      # 10 minutes (default 3600)
    clock_skew: 30                      # default 0
    allow_no_expiration: false
    encoder:
        signature_algorithm: ES256
    user_id_claim: sub                  # User ID; the default 'username' would put the email in every token
    token_extractors:
        authorization_header: { enabled: true, prefix: Bearer, name: Authorization }
        query_parameter: { enabled: false }
    blocklist_token:
        enabled: true                   # OFF by default; also adds the jti claim
        cache: jwt_blocklist_pool       # shared Redis pool
    api_platform:
        enabled: true
        check_path: /api/auth
        username_path: email
        password_path: password
The bundle docs show the blocklist as on by default, but the code leaves it off; without it tokens have no jti and logout cannot withdraw them.
7.3 Claims
Claim
 | Value
 | Checked
 | 
sub
 | User ID
 | Reload the User; must be active
 | 
iss
 | https://api.colsynq.app
 | JWT_DECODED listener
 | 
aud
 | colsynq-api
 | JWT_DECODED listener
 | 
iat, exp
 | 10-minute life
 | Encoder, 30 s skew
 | 
jti
 | Random
 | Blocklist
 | 
ver
 | User token_version
 | Must equal the reloaded User's value
 | 
header typ
 | at+jwt
 | RFC 9068
 | 
Not in the token: email, phone, names, tenant, profile, roles, permissions. The tenant is chosen per request; profile and access are loaded on the server.
final readonly class JwtClaimsListener
{
    public function __construct(private string $issuer, private string $audience) {}
    #[AsEventListener(event: Events::JWT_CREATED)]
    public function onCreated(JWTCreatedEvent $event): void
    {
        /** @var SecurityUser $user */
        $user = $event->getUser();
        $data = $event->getData();
        $data['iss'] = $this->issuer;
        $data['aud'] = $this->audience;
        $data['ver'] = $user->tokenVersion;
        unset($data['roles']);
        $event->setData($data);
        $header = $event->getHeader();
        $header['typ'] = 'at+jwt';
        $event->setHeader($header);
    }
    #[AsEventListener(event: Events::JWT_DECODED)]
    public function onDecoded(JWTDecodedEvent $event): void
    {
        $payload = $event->getPayload();
        if (($payload['iss'] ?? null) !== $this->issuer
            || !in_array($this->audience, (array) ($payload['aud'] ?? []), true)) {
            $event->markAsInvalid();
        }
    }
}
The ver comparison runs after the User is reloaded (in the JWT_AUTHENTICATED listener or the user checker, verify event name): a mismatch means sign-out everywhere, a password change, or a lock.
7.4 Firewall
security:
    providers:
        users:
            id: App\Iam\Infrastructure\Security\SecurityUserProvider
    firewalls:
        dev:
            pattern: ^/(_profiler|_wdt|assets|build)/
            security: false
        api:
            pattern: ^/api
            stateless: true
            provider: users
            user_checker: App\Iam\Infrastructure\Security\UserChecker
            entry_point: jwt
            json_login:
                check_path: /api/auth
                username_path: email
                password_path: password
                success_handler: App\Iam\Infrastructure\Security\AuthenticationSuccessHandler   # wraps lexik's; adds profile + access for the resolved tenant
                failure_handler: lexik_jwt_authentication.handler.authentication_failure
            jwt: ~
            access_token:                                   # service accounts (section 11.4)
                token_handler: App\Iam\Infrastructure\Security\ApiKeyHandler
                token_extractors: App\Iam\Infrastructure\Security\ApiKeyHeaderExtractor
            refresh_jwt:
                check_path: /api/token/refresh
            login_throttling:
                max_attempts: 5
                interval: '15 minutes'
            logout:
                path: /api/logout
    access_control:
        - { path: ^/api/(auth|auth/register|token/refresh|password-reset.*|verify-email|tenants/resolve)$, roles: PUBLIC_ACCESS }
        - { path: ^/api/docs, roles: PUBLIC_ACCESS }
        - { path: ^/api/platform, roles: ROLE_PLATFORM_STAFF }
        - { path: ^/api, roles: IS_AUTHENTICATED_FULLY }
The Tenant Resolver Service runs on kernel.request at priority 20: after the router (32), so it can read the matched route's _tenant requirement (required, optional, none or platform, compiled from #[TenantRequirement]), and before the firewall (8). It reads the X-Tenant slug or the host, checks status and switches the tenant database (User tab section 12.2, Multi-tenancy tab). REPROMAS uses 512 with a path bypass list; Colsynq moved it so the route attribute is the single source of truth.
ProfileLoginValidator listens to CheckPassportEvent: when a tenant is resolved, sign-in needs an active profile there, else the generic "Invalid email or password". The profile resolver runs after the firewall on every other request.
/api/auth, /api/token/refresh and /api/logout need routes without controllers. Set trusted_proxies so throttling sees real IPs.
7.5 Where clients keep tokens
Client
 | Access token
 | Refresh token
 | 
Society web app
 | In memory
 | httpOnly, Secure, SameSite=Strict cookie, path /api/token/refresh
 | 
Platform console
 | Server side (back-end-for-frontend); the browser holds a session cookie
 | Server side
 | 
Mobile app
 | Keychain / Keystore, sent as Bearer
 | Keychain / Keystore, sent in the body
 | 
Service account
 | No JWT: API key per request
 | None
 | 
Never
 | localStorage, sessionStorage, URLs
 | Same
 | 
8. Refresh tokens (gesdinet 3.0)
gesdinet_jwt_refresh_token:
    refresh_token_class: App\Iam\Infrastructure\Security\RefreshToken
    ttl: 1209600                    # 14 days
    ttl_update: false
    single_use: true                # rotate on every use
    max_session_lifetime: 2592000   # 30-day cap
    max_tokens_per_user: 10
    return_expiration: true
    hash_tokens: { enabled: true, accept_stored_in_the_clear: false }
    reuse_detection: { enabled: true, cache: cache.app }
    block_previous_jwt: true
    block_jwts_on_revocation: { enabled: true, cache: cache.app, ttl: 600, user_claim: sub }
    rate_limiter: { enabled: true, limiter: limiter.refresh, key: ip }
    cookie:
        enabled: true
        same_site: strict
        path: /api/token/refresh
        http_only: true
        secure: true
        remove_token_from_body: true
    api_platform: { enabled: true }
Refresh tokens belong to the User, not to a profile: one sign-in serves all the user's societies.
A reused refresh token revokes the whole chain and raises a security alert; clients must send one refresh at a time.
Cookie settings are global in v3; the mobile app's body-based token is handled in a custom success handler (verify).
gesdinet:jwt:clear runs daily through Symfony Scheduler. findAllForUser() feeds the "your devices" screen.
9. Revocation
Event
 | Effect
 | 
Sign out on one device
 | Current jti blocklisted; that refresh token revoked
 | 
Sign out everywhere / password changed or reset
 | revokeAllForUser(); User token_version bumped; SessionsRevokedV1
 | 
User locked or deactivated
 | User checker refuses the next request; token_version bumped
 | 
Profile disabled or removed in one society
 | No token change: the profile check refuses that society on the next request; other societies keep working
 | 
Role revoked
 | No token change: the profile's access_version changes and the next request sees it
 | 
Refresh token reused
 | Whole chain revoked; alert to user and Audit
 | 
Signing key compromised
 | New key; old key removed from additional_public_keys at once; everyone signs in again
 | 
API key revoked or expired
 | Refused on the next call
 | 
10. OAuth2 and OpenID Connect
10.1 Colsynq as a resource server (later, tokens from an identity provider)
security:
    firewalls:
        api:
            access_token:
                token_handler:
                    oidc:
                        claim: sub
                        algorithms: ['ES256', 'RS256']
                        audience: 'colsynq-api'
                        issuers: ['https://auth.example.com/realms/colsynq']
                        discovery:
                            base_uri: 'https://auth.example.com/realms/colsynq/'
                            cache: { id: cache.app }
The identity provider's sub is mapped to a Colsynq User through ExternalIdentity (section 11.1). The rest of the pipeline (tenant, profile, access) is unchanged.
Profiles and roles stay in Colsynq; the identity provider never decides society access.
Needs web-token/jwt-library, symfony/cache, GMP in production. security:oidc:generate-token makes test tokens. No built-in introspection; opaque tokens need a custom handler or facile-it/php-openid-client.
10.2 Colsynq as an authorization server
Prefer an external identity provider (Keycloak 26.8, Zitadel v4, Authentik 2026.8, Ory Hydra v26, or hosted). In-app fallback: league/oauth2-server-bundle 1.2.2 with password and implicit grants explicitly off, scopes.default set, one-minute authorization codes, consent through AUTHORIZATION_REQUEST_RESOLVE, and the known gaps (OIDC id_token, discovery, JWKS, introspection, revocation endpoint, PAR, DPoP, mTLS, private_key_jwt) built in-house.
11. Integrations
11.1 ExternalIdentity
Column
 | Notes
 | 
id, user_id
 | The Colsynq User
 | 
provider
 | google, microsoft, apple, oidc:<issuer>, client:<issuer>
 | 
subject
 | The provider's stable user or client ID; unique with provider
 | 
email, email_verified
 | As asserted by the provider
 | 
linked_at, last_used_at, unlinked_at
 |  | 
One User can have several external identities. Linking always happens from a signed-in session or a verified invitation, never by matching email alone.
11.2 Social login
Best done in the identity provider (brokering). In-app option: knpuniversity/oauth2-client-bundle with PKCE.
Link on the provider's subject, never on email. A matching email is trusted only if verified by a provider that owns that domain; even then the user confirms the link while signed in.
league/oauth2-client does not validate id_tokens; validate them yourself.
Social login signs in the User; it never creates profiles or roles.
11.3 Sign-up and external sign-in
Sign-up happens inside a resolved tenant (POST /api/auth/register) and creates the User and its UserProfile together (User tab section 11). A person may also sign up with a social or identity-provider account: the provider's verified identity creates the User and the ExternalIdentity, and the profile is created in the resolved tenant under the same sign-up policy. If the external identity's email already belongs to a Colsynq account, nothing is linked automatically; the owner gets a join link and links the provider from a signed-in session.
11.4 Service accounts and API keys
Rule
 | Detail
 | 
Principal
 | User with kind = service, no password, no authenticators, exactly one UserProfile in its tenant
 | 
Created by
 | Tenant administrator (service-accounts:create) for a named integration (payroll upload, partner, POS)
 | 
Key format
 | csk_live_ or csk_test_ + 256 random bits; shown once
 | 
Storage
 | iam.api_key.lookup_hash = SHA-256 of the key (fast lookup; the key is high-entropy), plus prefix for display
 | 
Bound to
 | The service account's profile: api_key(user_profile_id, tenant_id) references user_profile(id, tenant_id), so the key's tenant always equals the profile's; a key used with another tenant's host or X-Tenant is refused
 | 
Rights
 | UserRoles on the service profile, usually narrow and unit-scoped; same gates as people
 | 
Extras
 | expires_at (max 1 year), ip_allow_list, last_used_at, two active keys during rotation, every call logged
 | 
final readonly class ApiKeyHandler implements AccessTokenHandlerInterface
{
    public function __construct(private ApiKeyReader $keys, private TenantContext $tenant) {}
    public function getUserBadgeFrom(#[\SensitiveParameter] string $accessToken): UserBadge
    {
        $key = $this->keys->findActiveByLookupHash(hash('sha256', $accessToken));
        if (null === $key || !$key->isUsableFor($this->tenant->currentOrNull(), new \DateTimeImmutable())) {
            throw new BadCredentialsException('Invalid credentials.');
        }
        return new UserBadge($key->userId());   // loads the service User; profile resolution follows as usual
    }
}
The extractor reads Authorization: ApiKey csk_live_… (or X-Api-Key). Later: the service account authenticates with OAuth client_credentials at the identity provider, mapped through ExternalIdentity (client:<issuer>), with private_key_jwt or mTLS client authentication.
11.5 Webhooks (banks, BaaS, gateways)
Not principals. The Integrations module verifies each call by signature (HMAC or JWS), timestamp window and replay cache, stores the event, and publishes an integration event. Webhook endpoints are outside the JWT firewall and use their own per-connector secrets.
11.6 Financial-grade future
FAPI 2.0 (PAR, PKCE, sender-constrained tokens with mTLS or DPoP, strong client authentication) is realistic only through a capable identity provider; there is no Symfony DPoP bundle. CBN open banking prescribes OAuth 2.0; whether FAPI is mandatory is unconfirmed. Partner integrations should be designed to be FAPI-ready.
12. Account processes
Process
 | How
 | Security points
 | 
Self sign-up
 | POST /api/auth/register with the tenant resolved by host or X-Tenant; RegisterUserInTenant creates the User (email not yet verified) and UserProfile in one transaction; default role from the tenant's policy
 | Allowed only if the tenant's signup_mode is open or the email domain matches; tenant never taken from the body; uniform 202; throttled
 | 
Existing account joins a society
 | Join link (signed, User ID + tenant ID, 72 hours) or POST /api/me/profiles on an open tenant while signed in
 | A profile is never attached to an account by someone who only knows its email
 | 
User created by an administrator
 | POST /api/profiles: User without password + profile + set-password link (reset-password bundle)
 | No default passwords; initial role through the gates
 | 
First tenant administrators
 | Created with the tenant (two), set-password link, MFA before first use
 |  | 
Email or phone verification
 | verify-email-bundle 1.18 signed link carrying the User ID; OTP for phone
 | Required before sign-in when the tenant requires it
 | 
Sign-in
 | POST /api/auth with a tenant (needs an active profile there) or without (society switcher next)
 | Generic error when the profile is missing
 | 
Password reset
 | reset-password-bundle 1.25 (selector + hashed verifier, stored in iam.one_time_token)
 | generateFakeResetToken() when no user; after reset bump token_version, revoke refresh tokens, notify
 | 
MFA
 | scheb 2fa ^8.6: TOTP, email code, hashed backup codes, trusted devices
 | Required for tenant administrators, finance roles and staff; "MFA pending" step before tokens (verify); code_reuse_cache on; secrets encrypted
 | 
Passkeys
 | webauthn-symfony-bundle 5.3
 | User handle = UUID bytes; required for staff
 | 
Magic login links
 | Core login_link
 | 300 s, single use, signature_properties: [id, tokenVersion], trusted_hosts, confirm page that POSTs
 | 
Throttling and lockout
 | login_throttling (Redis) + per-User counter
 | Never more than 100 failures; unlock by reset or locked_until
 | 
Step-up
 | StepUpAssertion (User tab)
 | Email, password, MFA, API keys, role assignments, money actions
 | 
Switching society
 | Pick a profile from GET /api/me/profiles; send X-Tenant or use the society host
 | No new sign-in; profile checked on every request
 | 
Support access
 | Platform profile + SupportAccessGrant created by a tenant administrator
 | Impersonation limits apply
 | 
Service accounts
 | Section 11.4
 | Key bound to one tenant
 | 
Leaving a society
 | Profile removed
 | The User and other profiles remain
 | 
Closing an account
 | User deactivated; all roles, sessions and keys revoked; data kept per NDPA retention
 | Re-activation through platform support
 | 
Hide whether an account exists: keep security.expose_security_errors: none; same message and status for login, reset, verification resend and invitation acceptance; a dummy hash check when no User is found.
13. Handoff to authorisation
Authentication ends when the firewall has a SecurityUser. Everything after belongs to the User, access control and tenancy tab:
CurrentProfile = active UserProfile for (User, tenant); none → 404.
EffectiveAccess from UserRole → Role → RolePermission → Permission, cached by access_version.
AccessVoter: permission → scope → assertions.
Symfony roles from getRoles() stay coarse (ROLE_USER, ROLE_PLATFORM_STAFF) and are used only in access_control; every real decision uses permission slugs.
14. API documentation
lexik and gesdinet api_platform.enabled: true add login and refresh to OpenAPI.
Security schemes: JWT (Bearer) and ApiKey (header) under api_platform.swagger.api_keys.
With an identity provider later: api_platform.oauth with flow: authorizationCode, pkce: true, no clientSecret in production.
OpenAPI customisation lives in Infrastructure (Architecture guide).
15. Audit and monitoring
Security events (LoginSuccessEvent, LoginFailureEvent, LogoutEvent, TokenDeauthenticatedEvent, refresh reuse, authenticator added or removed, API key used or created, support access) are written to the single append-only iam.audit_event table and published as IAM integration events: UserSignedInV1, SignInFailedV1, SessionsRevokedV1, RefreshTokenReuseDetectedV1, AuthenticatorAddedV1, AuthenticatorRevokedV1, ExternalIdentityLinkedV1, ApiKeyCreatedV1, ApiKeyRevokedV1.
Record User ID, profile and tenant when known, IP, user agent, method, reason. Never passwords, tokens, keys, OTP codes or full emails; mark secrets #[SensitiveParameter].
Alert on refresh reuse, credential stuffing patterns, MFA disabled, passkey added to a platform account, API key used from a new IP, emergency support access.
16. Threats and controls
Threat
 | Control
 | 
alg: none or algorithm swap
 | Algorithm fixed by configuration
 | 
Forged kid, jku, x5u
 | Keys only from configuration or the identity provider's JWKS
 | 
Token theft from the browser
 | Memory-only access token, httpOnly refresh cookie, back-end-for-frontend for the platform console
 | 
Refresh token theft
 | Rotation, reuse detection, hashing, 30-day cap
 | 
Token for another API
 | iss and aud checks
 | 
Credential stuffing
 | Throttling, breached-password check, MFA, passkeys
 | 
Account enumeration
 | Uniform responses, fake reset tokens, timing equalised
 | 
Cross-society access
 | Tenant from host or header, active profile required, scope filters, 404 responses
 | 
API key used in the wrong society
 | Key bound to one tenant; checked against the resolved tenant
 | 
Takeover through social login
 | Link by provider subject, not email
 | 
Stale rights
 | No roles in tokens; token_version for the User, access_version for the profile
 | 
Host header poisoning in emailed links
 | framework.trusted_hosts
 | 
Signing key leak
 | Vault storage, rotation runbook, short tokens
 | 
17. Testing
Valid tokens from JWTTokenManagerInterface::create($user); bad ones from JWTEncoderInterface::encode($claims). ApiPlatform\Test\ApiTestCase with auth_bearer.
Negative tests: expired token; wrong aud or iss; tampered signature; alg: none; token after logout; token after password change (ver); reused refresh token; refresh rate limit; locked and deactivated users; valid token but no profile in the requested tenant (404); disabled profile in one society while another still works; API key used with another tenant; expired API key; service account trying password login; enumeration responses identical; impersonation blocked for money and access changes.
18. Checklist
sub = User ID; no email, tenant, profile or roles in tokens; iss, aud, typ, ver checked
ES256 keys in a vault; rotation tested; token_ttl 600, clock_skew 30, blocklist on Redis
gesdinet: single use, reuse detection, hashing, session cap, rate limit
SecurityUser adapter; Domain User framework-free; no eraseCredentials()
Tenant resolved before the firewall; profile resolved after it
Service accounts: one profile, tenant-bound hashed keys, narrow roles
ExternalIdentity linking by subject only
MFA for administrators, finance roles and staff; passkeys for staff; step-up assertion
Webhooks verified by signature, timestamp and replay check
Security events to Audit; no secrets in logs; negative tests in CI
19. Sources
Symfony security and access tokens: https://symfony.com/doc/current/security.html, https://symfony.com/doc/current/security/access_token.html
LexikJWTAuthenticationBundle: https://github.com/lexik/LexikJWTAuthenticationBundle · JWTRefreshTokenBundle: https://github.com/markitosgv/JWTRefreshTokenBundle
API Platform JWT: https://api-platform.com/docs/symfony/jwt/
league/oauth2-server-bundle: https://github.com/thephpleague/oauth2-server-bundle · knpuniversity/oauth2-client-bundle: https://github.com/knpuniversity/oauth2-client-bundle
scheb/2fa: https://github.com/scheb/2fa · WebAuthn: https://github.com/web-auth/webauthn-framework
RFC 8725 and its update draft · RFC 9068 · RFC 9700 · RFC 10017 · OAuth 2.1 draft 16 · FAPI 2.0 Security Profile
NIST SP 800-63B-4: https://pages.nist.gov/800-63-4/sp800-63b.html · OWASP cheat sheets: https://cheatsheetseries.owasp.org/
Research notes: doc/research_notes/auth/
