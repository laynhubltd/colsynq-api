# Adversarial architect review: User, access control and tenancy tab

Reviewed: `user_access_tenancy_tab_snapshot.md` (and `rbac_tenancy_schema_practice.md`). Date: 2026-10-12.
Severity: critical = exploitable privilege escalation or a design that cannot be built as written; high = security hole or certain production bug; medium = correctness or maintainability defect; low = clarity or hygiene.

Primary sources checked this session:
- PostgreSQL 18 partitioning limits: https://www.postgresql.org/docs/18/ddl-partitioning.html
- PostgreSQL 18 CREATE TABLE (UNIQUE constraint grammar, MATCH SIMPLE/FULL, FK target rules): https://www.postgresql.org/docs/18/sql-createtable.html
- PostgreSQL 18 CREATE INDEX (NULLS NOT DISTINCT + WHERE): https://www.postgresql.org/docs/18/sql-createindex.html
- Symfony RouterListener (kernel.request priority 32): https://github.com/symfony/symfony/blob/8.0/src/Symfony/Component/HttpKernel/EventListener/RouterListener.php
- Symfony FirewallListener (kernel.request priority 8): https://github.com/symfony/symfony/blob/8.0/src/Symfony/Bundle/SecurityBundle/EventListener/FirewallListener.php
- Symfony voters and decision strategies (default `affirmative`): https://symfony.com/doc/current/security/voters.html
- Doctrine ORM limitation "join columns pointing to non-primary keys": https://www.doctrine-project.org/projects/doctrine-orm/en/current/reference/limitations-and-known-issues.html
- gesdinet JWTRefreshTokenBundle (no token families; hashing opt-in): https://github.com/markitosgv/JWTRefreshTokenBundle
- OWASP ASVS 5.0 V8 Authorization: https://github.com/OWASP/ASVS/blob/master/5.0/en/0x17-V8-Authorization.md

---

## Critical

### 1. Open sign-up can hand out any role, including TENANT_ADMIN (sections 6.1, 13, 15.1)
**Problem.** `tenant.signup_default_role_code` lives in Tenancy. It is "validated by IAM when the policy changes", but the modules only exchange events, so that check runs after the change has already been committed. Nothing stops `signup_default_role_code = 'TENANT_ADMIN'`, or a custom role that later gains money permissions through `PUT /api/roles/{id}/permissions`. With `signup_mode = open`, every anonymous registrant then becomes an administrator. On top of that, the role is stored by **code**, not id. If a tenant creates a custom role with the same code as a system role, it is unclear which one the code refers to.
**Fix.** Move the default role into IAM, which owns roles: create `iam.tenant_signup_role (tenant_id uuid PK, role_id uuid NOT NULL REFERENCES iam.role(id))` and drop the column from `tenant`. Add this rule to 15.1: "A sign-up default role must have scope SELF (or TENANT with only `risk = low` permissions), no `can_assign_all`, and no assignment edges. `role_permission` changes on that role re-run this check and are refused with 409 `signup_role_privileged`." Add the test `SignupRoleNotPrivilegedTest`.

### 2. Custom roles bypass both separation of duty and "no money permissions on TENANT_ADMIN" (sections 8.6, 11, 15.1)
**Problem.** TENANT_ADMIN holds `roles:*` and `role-permissions:manage`. An administrator can create a custom role holding every `money` and `high` permission and have a second administrator grant it to them. Gate 0 only blocks self-grants, and two administrators are, by design, always present. `role_conflict` is keyed by role id, so a custom copy of FINANCE_OFFICER (`based_on_role_id`) has no conflict with AUDITOR. The SoD table is therefore decorative. This is ARBAC97 PRA97 missing: there is no `can_assignp`, so nothing limits which permissions a role editor may put into a role.
**Fix.**
- (a) PRA rule: "A caller may add a permission to a role only if they hold that permission in the same tenant (permission-level can_assignp). `money` and `high` permissions may be added to a custom role only by a holder of `role-permissions:manage-privileged`, which no system role has by default (granted by the platform per tenant contract)."
- (b) Move SoD from role pairs to permission sets: `iam.permission_conflict (tenant_id uuid NULL, permission_a text, permission_b text, same_scope_only bool, severity text, CHECK (permission_a < permission_b), UNIQUE NULLS NOT DISTINCT (tenant_id, permission_a, permission_b))`. Evaluate it on the profile's *effective permission set* after the grant, and again for every holder when a role's permissions change. Keep `role_conflict` only as a convenience that is expanded into permission pairs.

### 3. Tenant Resolver at kernel.request priority 512 cannot read `#[TenantRequirement]` (section 12.2)
**Problem.** The RouterListener sets `_route` and `_controller` at priority 32. At 512 no route has been matched, so `$this->requirements->for($request)` cannot read a route or controller attribute. In practice it falls back to the "bypass list" (path patterns), and the default `Required` then applies to every unlisted route. Public routes (`/api/tenants/resolve`, `/api/auth` without a tenant, the society switcher, `/api/verify-email`, health checks) either fail with 400 tenant_required or depend on a hand-maintained list that drifts from the attributes. The two sources of truth also contradict each other.
**Fix.** Write the requirement into route defaults at compile time. A route loader decorator or compiler pass reads `#[TenantRequirement]` and sets `defaults: { _tenant: required|optional|none }`. Run the listener **after routing and before the firewall**: `#[AsEventListener(event: KernelEvents::REQUEST, priority: 20)]` (after RouterListener 32, before LocaleListener 16 and the Firewall 8), and read `$request->attributes->get('_tenant', 'required')`. Delete the bypass list. Replacement text: "Listener: kernel.request priority 20, after the router (32) and before the firewall (8); the requirement comes from the matched route's `_tenant` default, compiled from `#[TenantRequirement]`; skips OPTIONS." Side effect: unknown paths now give 404 before any tenant lookup, which also stops unauthenticated tenant-database switching for garbage URLs.

### 4. `audit_event` as written cannot be created (section 8.6)
**Problem.** The table has `PRIMARY KEY (id)` and `PARTITION BY RANGE (occurred_at)`. PostgreSQL rejects this. The docs state that "the constraint's columns must include all of the partition key columns" (ddl-partitioning, 5.12.2.3).
**Fix.**
```sql
CREATE TABLE iam.audit_event (
  id uuid NOT NULL, occurred_at timestamptz NOT NULL, ...,
  CONSTRAINT audit_event_pk PRIMARY KEY (occurred_at, id)
) PARTITION BY RANGE (occurred_at);
```
Note that `id` alone is not unique across partitions. UUIDv7 makes that harmless, but say so. Doctrine maps this with `#[ORM\Id]` on both columns, or better as a write-only DBAL insert with no entity. Partition creation and drop need a separate `iam_maintenance` role. The application role has neither DDL nor DELETE.

---

## High

### 5. NULLS NOT DISTINCT used as a partial *constraint* (sections 5, 8.4)
**Problem.** "UNIQUE NULLS NOT DISTINCT (user_profile_id, role_id, scope_ref_id) WHERE revoked_at IS NULL" is not valid as a table constraint, because the UNIQUE constraint grammar has no WHERE (only EXCLUDE does). It is valid as `CREATE UNIQUE INDEX ... NULLS NOT DISTINCT WHERE ...`. A unique index cannot be the target of a foreign key when it is partial, which is acceptable here. Separately, Doctrine cannot express NULLS NOT DISTINCT, so `doctrine:migrations:diff` and `schema:validate` will keep trying to drop or recreate these indexes.
**Fix.** Pick the Doctrine-mappable form everywhere. Use two partial unique indexes, which Doctrine supports through `options: ['where' => ...]`:
```sql
CREATE UNIQUE INDEX user_role_active_unscoped_uq ON iam.user_role (user_profile_id, role_id)
  WHERE revoked_at IS NULL AND scope_ref_id IS NULL;
CREATE UNIQUE INDEX user_role_active_scoped_uq ON iam.user_role (user_profile_id, role_id, scope_ref_id)
  WHERE revoked_at IS NULL AND scope_ref_id IS NOT NULL;
CREATE UNIQUE INDEX role_system_code_uq ON iam.role (code) WHERE tenant_id IS NULL;
CREATE UNIQUE INDEX role_tenant_code_uq ON iam.role (tenant_id, code) WHERE tenant_id IS NOT NULL;
```
Do the same for `user_role_assign_role` and `role_conflict`. Change section 5 to: "Nullable tenant: a pair of partial unique indexes (`WHERE tenant_id IS NULL` / `IS NOT NULL`), mapped in Doctrine; NULLS NOT DISTINCT only where Doctrine is told to ignore the index." Also forbid custom role codes that shadow a system code (`role_tenant_code_uq` does not do that), either with a trigger or with an app check plus a test. This also settles the ambiguity in finding 1.

### 6. Expired grants block re-granting and stay in the cache (sections 8.4, 15.2, 20)
**Problem.** The active-grant unique index filters only on `revoked_at IS NULL`. An expired but unrevoked EXTERNAL_AUDITOR grant therefore blocks a new grant for the same role and scope. The access cache key changes only when `access_version` is bumped, and expiry bumps nothing. A cached grant set therefore outlives `expires_at` until something else changes. The doc's claim that "expired grants are ignored by the loader" is true only on a cache miss. This breaks ASVS 8.3.2.
**Fix.**
- (a) An `iam:grants:expire` scheduler task (every minute) sets `revoked_at = expires_at`, bumps `access_version` and writes `audit_event (action = 'user_role.expired')`. Also index `(expires_at) WHERE revoked_at IS NULL AND expires_at IS NOT NULL`.
- (b) Cache TTL = `min(expires_at of returned grants, support grant expires_at, default TTL)`.
- (c) The grant command treats an existing expired row as "revoke then insert" inside the same transaction.

### 7. Two-administrator rule: "under a row lock" does not prevent write skew (sections 13, 15.3)
**Problem.** You cannot `SELECT count(*) ... FOR UPDATE`. Locking the admin `user_role` rows does not serialise *profile disable*, *profile remove* or *user deactivate*, because those update `user_profile` or `app_user` and never touch the grant rows. Under READ COMMITTED, two concurrent transactions (disable profile A, revoke B's grant) each count 3, and the tenant ends with 1. Further gaps: grants with `expires_at` silently drop the count below two with no transaction at all; the count definition is unspecified (profile active? user active? role not archived? not expired?); and user deactivation touches many tenants, so it can deadlock.
**Fix.** Replacement text: "Every command that can reduce a tenant's administrators takes `pg_advisory_xact_lock(hashtextextended('iam.admins:' || tenant_id::text, 0))` first. Commands that touch several tenants (user deactivation) take the locks in ascending tenant_id order. Then they count `user_role ur JOIN user_profile p ON (p.id, p.tenant_id) = (ur.user_profile_id, ur.tenant_id) JOIN app_user u ON u.id = p.user_id JOIN role r ON r.id = ur.role_id WHERE ur.tenant_id = $t AND r.code = $adminCode AND r.tenant_id IS NULL AND ur.revoked_at IS NULL AND (ur.expires_at IS NULL OR ur.expires_at > now()) AND p.status = 'active' AND u.status = 'active'`, excluding the subject of the change. TENANT_ADMIN and PLATFORM_OWNER grants cannot carry expires_at (gate 3)." The alternative is SERIALIZABLE with retry, but the advisory lock is simpler and cheaper.

### 8. Role-permission edits and role archiving do not invalidate the access cache (sections 8.2, 10, 15.1, 20)
**Problem.** The cache key is `{tenant}:{profile}:{access_version}:{catalogue_version}`. Editing a custom role's permissions (`PUT /api/roles/{id}/permissions`) or archiving a role bumps neither value. Removed permissions stay effective for every holder until some unrelated bump. That is privilege retention.
**Fix.** In the same transaction as any `role_permission` change or role archive/unarchive:
```sql
UPDATE iam.user_profile p SET access_version = access_version + 1
 WHERE p.id IN (SELECT user_profile_id FROM iam.user_role WHERE role_id = $1 AND revoked_at IS NULL);
```
For system roles, the catalogue_version bump covers this. Add to 15.2: "Any change to a role's permissions or archive state bumps access_version for all its active holders."

### 9. Role-to-tenant consistency is not enforced in the database (sections 8.2, 8.4, 8.5, 8.6)
**Problem.**
- `user_role.role_id` has a plain FK. "System role or custom role of the same tenant" and "platform roles only in the platform tenant" are checked only in the application. A bug or a direct SQL fix can grant tenant B's custom role, or PLATFORM_OWNER, to a profile of tenant A.
- The same gap exists for tenant rows in `user_role_assign_role` and `role_conflict` (they can reference another tenant's custom roles) and for `role.based_on_role_id`.
- A composite FK cannot express "NULL or same tenant": with MATCH SIMPLE a NULL column skips the whole check, and MATCH FULL forbids partial nulls.
- IAM does not know which tenant is the platform tenant (the tenant kind lives in Tenancy, linked logically).
**Fix.**
- Add `iam.platform_tenant (singleton boolean PK DEFAULT true CHECK (singleton), tenant_id uuid NOT NULL)`, populated at bootstrap.
- Add a `CONSTRAINT TRIGGER user_role_role_tenant_ck AFTER INSERT OR UPDATE OF role_id, tenant_id ON iam.user_role DEFERRABLE INITIALLY IMMEDIATE`. It raises unless `(r.tenant_id IS NULL OR r.tenant_id = NEW.tenant_id) AND ((r.scope = 'PLATFORM') = (NEW.tenant_id = (SELECT tenant_id FROM iam.platform_tenant)))` and `r.archived_at IS NULL`.
- Add equivalent triggers on `user_role_assign_role` and `role_conflict` (both roles are system or `= tenant_id`, and a tenant row never references another tenant's role) and on `role` (`based_on_role_id` is a system role or the same tenant's; `tenant_id` and `scope` are immutable after insert).
- Also enforce that TENANT/BU/VENTURE/SELF roles are not granted in the platform tenant.
- Change "checked on insert" to "enforced by trigger `user_role_role_tenant_ck` and the application".

### 10. Platform permissions are not bound to the platform tenant context (sections 9, 14, 18)
**Problem.** Nothing in the pipeline says that `/api/platform/*` requires the resolved tenant to be `kind = platform`, or that platform-scope permissions are evaluated only when the request's tenant is the platform tenant. Combined with a tenant-free JWT and a mis-grant (see 9), platform permissions could be exercised from a society host. ASVS 8.4.1 requires cross-tenant controls.
**Fix.** Add to section 14: "Permissions whose `allowed_scopes = {PLATFORM}` are granted only when `TenantContext.kind = platform`. Platform routes carry `#[TenantRequirement(Required)]` plus `#[PlatformOnly]`, and the resolver returns 404 for them on society tenants." Add the test `PlatformPermissionOutsidePlatformTenantTest`.

### 11. Self sign-up contradicts the five gates (sections 13, 15.1)
**Problem.** RegisterUserInTenant "grants the initial or default role through the gates". At self sign-up there is no caller with roles: the caller *is* the target. Gate 0 (own profile) and gate 1 (authority edge from the caller's roles) both fail, and the doc never defines a "system grantor". The same applies to the bootstrap TENANT_ADMIN grants from TenantProvisionedV1 and to join links.
**Fix.** Define two grant paths. "Delegated grant: gates 0–4 with the caller's profile. System grant (sign-up default role, join link, tenant bootstrap, catalogue sync): skips gates 0–2, runs gates 3–4, is limited to the role named by `tenant_signup_role` (sign-up and join) or TENANT_ADMIN (bootstrap only, only from TenantProvisionedV1 and only while the tenant has no administrator), and records `actor = system:<path>` in audit_event." Implement it as a separate `SystemRoleGrant` command, so the policy class never sees an "is system" flag.

### 12. Account pre-hijacking and unverified domain sign-up (section 13)
**Problem.**
- Self sign-up with a new email creates `app_user` with a password before the email is verified. An attacker pre-registers `victim@x` and sets the password. When the victim later signs up, they hit the "existing email, join link" path and log in to an account whose password the attacker knows. This is the classic pre-hijacking pattern (Sudhodanan and Paverd, USENIX Security 2022).
- In `email_domain` mode the profile and default role are created before the email is proven, so anyone can claim `@society.org`. If `require_email_verification = false`, email_domain mode is meaningless.
**Fix.**
- "Self sign-up stores a pending registration (signed, stateless or a one_time_token with purpose = signup). `app_user`, `user_profile` and the default grant are created only when the verification link is used, and the password is set at that step."
- Or: "Until `email_verified_at` is set, the account cannot sign in, and a password reset or verification by the real owner discards the stored password_hash and all sessions."
- CHECK on tenant: `signup_mode <> 'email_domain' OR require_email_verification`.
- Joining an email_domain tenant through `/api/me/profiles` requires `email_verified_at IS NOT NULL` and a domain match.

### 13. Removed profiles cannot rejoin; all creation paths collide with unique (user_id, tenant_id) (sections 7.2, 13)
**Problem.** "Remove" keeps the row with `status = removed`. Every entry path (self sign-up, join link, `/api/me/profiles`, admin create, bootstrap) says "creates user_profile", which raises a unique violation (500) or reveals membership history.
**Fix.** Add to section 13: "RegisterUserInTenant is idempotent per (user, tenant). If no profile exists, create one. If `removed`, reactivate it (status active, status_reason, new audit_event `user_profile.rejoined`, old grants stay revoked, a fresh default grant). If `disabled`, refuse (an administrator must enable it) and return the generic response. If `active`, do nothing and return the same response." Lock with `SELECT ... FOR UPDATE` on the profile row, or `INSERT ... ON CONFLICT (user_id, tenant_id) DO NOTHING RETURNING`.

### 14. API keys: tenant binding and "service user only" not enforced (sections 7.3, 14)
**Problem.**
- "The profile's user must be kind = service" and "its only profile" cannot be enforced by the composite FK.
- The pipeline does not say that the API key's `tenant_id` must equal the resolved tenant. An API key presented with `X-Tenant: other` either resolves another tenant (then step 3 fails, which is fine if implemented) or is used where it should not be.
- The `lookup_hash` algorithm is unspecified.
**Fix.**
- Trigger on `api_key` insert: raises unless `(SELECT kind FROM app_user JOIN user_profile ...) = 'service'`.
- Trigger on `user_profile` insert: a service user may have only one profile (or add `app_user.home_tenant_id` for services, with `CHECK (kind = 'person' OR home_tenant_id IS NOT NULL)`).
- Pipeline rule: "For ApiKey authentication the tenant is the key's tenant. An X-Tenant or host naming another tenant gives 400 tenant_mismatch."
- `lookup_hash = HMAC-SHA-256(server_pepper, secret)`, compared in constant time. Enforce `ip_allow_list` and `expires_at` in the authenticator.

### 15. Support access has no path through the pipeline (sections 14, 16)
**Problem.** Step 3 requires an active `user_profile` for the user in the tenant, else 404. A support agent has a profile only in the platform tenant, so a valid `support_access_grant` can never be used, and nobody says what the effective access is. Also: `support_profile_id` "a platform-tenant profile" and `granted_by_profile_id` "a profile of the tenant" are not enforced, and there is no `CHECK (expires_at > starts_at)`.
**Fix.**
- Step 3b: "If no profile exists and the user holds an active support_access_grant for the tenant, EffectiveAccess is the fixed SUPPORT_READ_ONLY (or impersonation) permission set. The cache key includes the grant id, and the TTL is the grant's expires_at."
- DDL: `FOREIGN KEY (granted_by_profile_id, tenant_id) REFERENCES iam.user_profile (id, tenant_id)`; a trigger checks that `support_profile_id`'s tenant is the platform tenant; `CHECK (expires_at > starts_at AND expires_at <= starts_at + interval '4 hours')`.

### 16. Authorization decision strategy not specified; affirmative default undermines assertion denies (section 14)
**Problem.** Symfony's default strategy is `affirmative`: one granting voter wins. If any other voter (a module's own voter, ExpressionVoter, RoleVoter) votes on the same attribute or subject, AccessVoter's assertion Deny is overruled. AccessVoter also bundles permission lookup, scope evaluation and assertion orchestration, which breaks SRP and is hard to test.
**Fix.** Configure `security.access_decision_manager: { strategy: unanimous, allow_if_all_abstain: false }`. Make AccessVoter a thin adapter that delegates to the application service `AccessDecision`, composed of `PermissionCheck`, `ScopeCheck` and `AssertionRegistry` (tagged iterator). Report denies through `Vote::addReason()` so the reason code reaches the 403 (ASVS 8.1.x). Add an arch test: "no other voter supports permission-slug attributes".

### 17. Doctrine cannot map the composite FKs as associations (sections 7.2, 7.3, 8.4)
**Problem.** Doctrine ORM "cannot use join columns pointing to non-primary keys". `ManyToOne` with join columns `(user_profile_id → id, tenant_id → tenant_id)` would make Doctrine treat `(id, tenant_id)` as the profile's identity and build wrong proxies. Raw-SQL composite FKs are also invisible to the schema tool, which will try to drop them.
**Fix.** Map `UserRole` (and `ApiKey`) with `ManyToOne UserProfile` on `user_profile_id → id` **plus** a plain `tenant_id` column (insertable, not an association). Add the composite FK and `user_profile_id_tenant_id_uq` in a hand-written migration. Exclude them from diff noise by naming convention, with a migration test asserting they exist (extend `CompositeTenantKeyTest` to check `pg_constraint`).

---

## Medium

### 18. Access cache: read-order race and stampede (sections 14, 20)
**Problem.**
- If the loader reads grants first and `access_version` second (or in separate snapshots), a concurrent grant commit can store **old grants under the new key** forever.
- No stampede protection.
- If reads go to a replica, "applies on the next request" is false.
**Fix.**
- "The loader reads `access_version` and the grants in one statement (or one REPEATABLE READ transaction), version first, always on the primary."
- Use Symfony Cache contracts `$cache->get($key, $callback, $beta)`, which has built-in lock and early-expiration stampede protection.
- Use `catalogue_version` from a single landlord row, read with the profile.

### 19. SELF scope stores the user id: a duplicated fact, and it blocks the scope_reference FK (sections 8.4, 9)
**Problem.** `scope_ref_id` = "the user ID" for SELF duplicates the grant's own profile, so the two can disagree (and it is user vs profile, which is ambiguous). It also prevents a real FK to `scope_reference`.
**Fix.** "SELF: scope_ref_id NULL; the owner is the grant's profile (OwnRecordAssertion compares record.owner_profile_id or owner_user_id with the current profile/user)." Then add:
```sql
ALTER TABLE iam.user_role ADD CONSTRAINT user_role_scope_ref_fk
  FOREIGN KEY (tenant_id, scope_ref_id) REFERENCES iam.scope_reference (tenant_id, scope_ref_id); -- MATCH SIMPLE: NULL refs skipped
```
Add a trigger checking `scope_reference.kind` matches `role.scope` and `scope_ref_id IS NULL` iff the scope is TENANT, PLATFORM or SELF.

### 20. scope_reference staleness and event ordering (sections 8.6, 19)
**Problem.** The events are asynchronous. UnitClosedV1 lag leaves scoped grants effective after closure, and the loader does not filter on `scope_reference.status`. Out-of-order delivery (Closed before Activated) can resurrect a closed unit.
**Fix.** "The loader excludes grants whose scope_reference.status <> 'active'. Projection updates are idempotent and ordered by the producer's `occurred_at`/sequence (`UPDATE ... WHERE incoming_version > version`). UnitClosedV1 revokes the scoped grants and bumps access_version in one transaction." Add a `version bigint` column. Gate 2 failing closed while the projection lags is acceptable; document the 409 `scope_not_ready`.

### 21. Role lifecycle: archive vs grants vs delete (sections 8.2, 18)
**Problem.**
- Archived roles are "ignored" but their grants stay unrevoked, so unarchiving silently restores access.
- `DELETE /api/roles/{id}` against `user_role` rows that are "never deleted" means the FK blocks the delete, while `role_permission ON DELETE CASCADE` suggests deletes happen.
- The time-limited flag (EXTERNAL_AUDITOR "expires_at required") is not a column; it lives only in prose.
**Fix.**
- "DELETE /api/roles/{id} archives. Archiving requires zero active grants (409 role_in_use) or revokes them with audit."
- Drop the CASCADE.
- Add `role.max_grant_duration interval NULL` (non-null means expires_at is required and bounded), checked in gate 3, plus `CHECK (max_grant_duration IS NULL OR max_grant_duration > interval '0')`.

### 22. Tenant can switch off SoD defaults (sections 8.6, 18)
**Problem.** A tenant row with `severity = 'off'` overrides a default `block` (FINANCE_OFFICER/AUDITOR). `role-conflicts:*` is held by TENANT_ADMIN, so administrators can disable their own controls.
**Fix.** "A tenant row may tighten (warn→block) but not loosen a default `block`." Enforce this with a trigger or app check plus a test. Loosening needs a platform-side override recorded in audit_event.

### 23. ProfileLoginValidator ordering leaks membership (section 13)
**Problem.** If the listener runs on CheckPassportEvent before credentials are checked (UserCheckerListener's pre-check runs at 256, CheckCredentialsListener at 0), a wrong password and "no profile here" fail differently in time, and possibly in code. That enumerates who belongs to which society. The section-20 table also gives 404 tenant_not_found for "no active profile" on API calls, while login gives a "generic error". The two need a stated contract.
**Fix.** "ProfileLoginValidator listens on CheckPassportEvent at priority -10, after password verification, and throws the same `CustomUserMessageAuthenticationException('Invalid credentials.')` as a bad password. For unknown emails a dummy Argon2id verify is run to equalise timing." Add a test that measures the error code only (timing as a manual check).

### 24. Cross-module boundary leaks (sections 2, 3, 12.2, 17)
**Problem.**
- IAM "reads TenantContext", but TenantContext/ResolvedTenant live in `Tenancy/Infrastructure`, so IAM must import Tenancy, which contradicts "modules never import".
- `RoleScope` is declared in both `Iam/Domain/Access` and `Shared/Access`.
- `NotSelfAssertion` repeats gate 0 (one rule, two places).
- `signup_default_role_code` is an IAM concept stored in Tenancy (see 1).
**Fix.** Move `TenantContext` (interface) and `ResolvedTenant` (read-only DTO) to `Shared/Tenancy`, implemented in Tenancy/Infrastructure. Keep `RoleScope` only in Shared/Access. Remove NotSelfAssertion and leave the rule in gate 0. Add a deptrac/arch test for module imports.

### 25. Long-running workers and Doctrine filters (sections 12.2, 14)
**Problem.**
- `TenantContext::switchTo` "refuses a second tenant". Under FrankenPHP/RoadRunner worker mode the context, the switched connection and the enabled SQL filters survive into the next request unless reset.
- Doctrine SQL filters (step 5) do not apply to DBAL/native queries and do not re-filter entities already in the identity map, so they cannot be the primary scope control.
**Fix.**
- TenantContext implements `Symfony\Contracts\Service\ResetInterface` (tag `kernel.reset`) and is also cleared on `kernel.finish_request` and in exception paths.
- "Scope filters are defense in depth; repositories and handlers apply scope explicitly; native queries must take a `ScopedBy` argument."

### 26. Step-up vs assertion error contract (sections 14, 20)
**Problem.** StepUpAssertion is an assertion (Deny → 403 with reason), but the error table says 401 `step_up_required`.
**Fix.** StepUpAssertion returns `AssertionResult::challenge('step_up_required')`. The voter maps challenges to an `InsufficientAuthenticationException` → 401 with `WWW-Authenticate: Bearer error="insufficient_user_authentication"` (RFC 9470).

### 27. Phone uniqueness and number recycling (section 7.1)
**Problem.** A global unique phone blocks household members who share a phone, which is common in Nigeria. Nigerian operators also recycle inactive numbers, so a phone-based recovery path is an account-takeover risk.
**Fix.** "phone: unique only where phone_verified_at IS NOT NULL (`CREATE UNIQUE INDEX app_user_phone_verified_uq ON iam.app_user (phone) WHERE phone_verified_at IS NOT NULL`). Verifying a number already verified on another account clears it there (audit). Phone is never the sole recovery factor."

### 28. JWT is tenant-free: one token works across all the user's societies (section 2, Tokens)
**Problem.** A token stolen via XSS on one society's custom domain is valid at every society the user belongs to, including ones that require MFA (no per-tenant assurance).
**Fix.** Keep tenant-free roles, but add an `amr`/`auth_time` claim and evaluate per-tenant MFA policy with StepUpAssertion. Bind the refresh token family to the issuing origin (store `origin` on refresh_token). Optionally set `aud` = origin and verify it against the Host.

### 29. Refresh token claims do not match gesdinet (section 7.3)
**Problem.** "gesdinet columns: hashed token, ..., family, family_valid". Gesdinet has `refreshToken`, `username` and `valid` only. Hashing is opt-in (`hash_tokens`), and there are no families or reuse detection.
**Fix.** "Custom RefreshToken entity extending gesdinet's AbstractRefreshToken with `family_id uuid`, `replaced_at`; `hash_tokens: enabled: true`; `single_use: true`; a RefreshTokenReuseListener revokes the family on reuse."

### 30. Erasure (NDPA 2023 / GDPR) and tenant deletion (sections 7, 8.6, 12)
**Problem.**
- There is no TenantDeletedV1 or IAM handling. Profiles, grants, api_keys and scope_reference rows of deleted tenants remain.
- A global app_user cannot be erased per tenant.
- `audit_event` is immutable but stores `ip`, `user_agent` and before/after payloads containing names and emails.
- No retention period is defined.
**Fix.**
- Add TenantDeletedV1 → IAM removes profiles, revokes grants and keys, and deletes scope_reference.
- "User erasure: app_user email/phone/name replaced with tombstones, authenticators and external identities deleted, profiles removed. audit_event payloads reference ids only; PII in payloads is encrypted per user (crypto-shredding key in `iam.user_pii_key`) and destroyed on erasure."
- Retention: "audit_event partitions kept 7 years for money/access events (regulatory), dropped by the maintenance role."

### 31. Abuse controls missing on public endpoints (sections 13, 18)
**Problem.** There are no rate limits on register, join, resolve, login, password reset or verify-email. There are no idempotency keys on POST commands. "Always the same 202" still leaks through timing, because the new-email path does more synchronous work.
**Fix.** Symfony RateLimiter with a `sliding_window` per IP + per normalised email + per tenant on these routes. All sign-up side effects go through Messenger (the response is sent before work). Accept an `Idempotency-Key` header on POST /api/profiles, /roles, /service-accounts, and grants, stored for 24 hours. Use throttling with backoff instead of a hard global `locked_until`: a global lock is a lockout DoS across tenants, and NIST 800-63B-4 requires rate limiting rather than permanent lockout.

---

## Low

32. **role_conflict CHECK role_a_id < role_b_id.** Valid (uuid has btree ordering). The seeding of default conflicts must sort by uuid, not by code; write `LEAST/GREATEST` normalisation in the command. Fix: "the application normalises (a, b) = (least, greatest) before insert."
33. **`tenants:*` style wildcards** in sections 11/18 violate `CHECK slug = resource || ':' || action`. Say "shorthand for all actions of the resource; the catalogue stores explicit slugs; no wildcard matching at runtime."
34. **`CHECK assigner_role_id <> assignable_role_id`** forbids the legitimate "UNIT_MANAGER may appoint a deputy UNIT_MANAGER". Drop it or document it as deliberate.
35. **Gate 2 vs SELF edges.** UNIT_MANAGER → SELF_SERVICE conflicts with "unit callers assign only with their own unit IDs". State that SELF grants need no scope reference and are allowed to edge holders for profiles of their unit (or remove these edges).
36. **can_assign_all vs deny edges.** Precedence is undefined. Add: "deny edges win over can_assign_all; can_assign_all never covers PLATFORM roles or roles with `money` permissions unless the caller holds those permissions."
37. **Revocation authority.** ARBAC97 separates can_revoke from can_assign. Reusing assign edges is acceptable but must be stated, and the grant's scope_ref must be within the caller's scope (not only the caller's own IDs at assign time).
38. **`permission` deletion.** `role_permission.permission_slug` FK needs `ON DELETE CASCADE` (or catalogue sync deletes role_permission rows first, with audit) for "remove deprecated slugs one release later".
39. **`GET /api/tenants/resolve` exposes status.** A society's suspended (billing) state should not be public. Return only `available | unavailable`.
40. **Event payload PII.** TenantProvisionedV1 carrying admin emails sits in the messenger transport. Send ids or a one-time reference instead.
41. **`created_by_profile_id`** on user_profile can point at another tenant's profile. Allow only same-tenant or platform-tenant profiles (trigger), or drop it and rely on audit_event (who/why is in audit per finding 10 of the doc itself).
42. **Archived tenant sign-in "administrators and auditors".** No component is assigned this check. Put it in ProfileLoginValidator (role-aware) and in TenantWritableAssertion.
43. **Email normalisation.** Specify one function used both in PHP and in SQL (`lower()` vs `mb_strtolower` differ for some code points; PG18 `casefold()` is better), and do not strip dots or plus tags.

---

## Missing governance tests to add (section 22)
SignupRoleNotPrivilegedTest, CustomRolePermissionEscalationTest (PRA), PermissionSoDTest, TenantRequirementCompiledTest (listener at 20 reads `_tenant`), AuditPartitionDdlTest, GrantExpirySweepTest, TwoAdministratorsConcurrencyTest (two connections, advisory lock), RolePermissionChangeInvalidatesCacheTest, RoleTenantTriggerTest, PlatformPermissionOutsidePlatformTenantTest, RejoinAfterRemovalTest, ApiKeyTenantMismatchTest, SupportGrantPipelineTest, DecisionStrategyUnanimousTest, PreHijackingTest, RateLimitTest.
