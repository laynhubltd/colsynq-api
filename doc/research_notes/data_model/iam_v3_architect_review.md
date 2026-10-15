# IAM and tenancy v3: architect review of v2 (r22) and lean data model

Reviewed: "User, access control and tenancy (v2)" snapshot r22 (sections 1–27, `iam_v2_snapshot_r22.md`), cross-checked with `tab_authentication.md`, `tab_multitenancy.md` and `tab_architecture_guide.md`. Date: 13 October 2026.
Context: Symfony 8.1 modular monolith, Doctrine ORM 3, PostgreSQL 17/18, database per tenant plus a landlord (control plane), Tenancy and IAM in the landlord, modules talk only through events, cooperative finance in Nigeria.

## 0. Summary

1. Section 27 is binding but was added on top of the earlier text. Sections 2, 3, 4, 7, 11, 13, 16 and 25 and the Authentication and Multi-tenancy tabs still describe the older design. There are **at least 34 contradictions** (part b). The document also describes provisioning in two different ways, with two status sets, two job tables and two activation events.
2. The model has **30 landlord tables plus 2 for Integrations**. **23 are enough** (6 in tenancy, 17 in iam) with every needed section 27 control kept. Integration tables move to the Integrations module. Code-time configuration moves into code: plans, features, default assignment rules and default SoD conflicts. Duplicated facts are merged: device_key versus user_device, refresh-token family versus auth_session, platform_tenant versus tenant.kind, and access_version spread over every profile.
3. Five flow gaps stand out:
   - An administrator can attach any existing global account to their society by knowing its email. There is no invitation entity, and this contradicts the Authentication tab.
   - The first administrators come from an undefined "encrypted invitation record" that crosses the module boundary. The tenant becomes `active` with zero administrators.
   - Tenant-free global sessions clash with two section 27 rules: per-role session policies and per-host `__Host-` BFF cookies. The society switcher and "no new login" break on the web.
   - The Multi-tenancy tab offboarding runbook (`DROP DATABASE` after a grace period) breaks the AML retention rule in 27.8.
   - Three items appear in the text but have no column: tenant MFA policy, legal hold, and dormancy tracking.
4. Naming problems:
   - `kind` and `type` are mixed, and so are `slug` and `code`.
   - FKs are written as both `user_profile_id` and `profile_id`.
   - `state` (a Nigerian state) sits next to `status`.
   - The `app_user` prefix collides with the `app_*` PostgreSQL role names.
   - `user_role_assign_role`, `scope_reference` and `one_time_token` are unclear.
   - Role scope values are uppercase while every other value set is lowercase.
   - A gesdinet column called `valid` holds a timestamp.

   Part (c) fixes all of these with a full rename map.

---

## (a) Findings

Severity scale: Critical (exploitable or data-loss), High (control missing or broken flow), Medium (redundancy or contradiction that will cause bugs), Low (naming or documentation).

### F1. High: an administrator can attach an existing user to a society without consent (13.1, Auth tab 12)
- **Problem.** In 13.1, "Created by an administrator" handles an existing email with "profile + notice". Anyone with `profiles:create` can add any Colsynq account to their society by email. That account then appears in the person's society switcher, and the API reveals that the account exists, since a new email and an existing email give different outcomes. The Authentication tab (12) says the opposite: "A profile is never attached to an account by someone who only knows its email."
- **Evidence.** Mature systems all model a pending invitation that the invitee must accept:
  - WorkOS keeps `pending` memberships that only invitation acceptance can activate (https://workos.com/docs/reference/authkit/organization-membership). Its invitation object has `state` pending/accepted/expired/revoked, `inviter_user_id`, `accepted_user_id` and `role_slug` (https://workos.com/docs/reference/authkit/invitation).
  - Clerk's OrganizationInvitation has `status` pending/accepted/revoked/expired and `role` (https://clerk.com/docs/reference/backend/types/backend-organization-invitation).
  - In Keycloak, inviting an existing user only "asks the user to confirm they want to join" (https://docs.redhat.com/ja/documentation/red_hat_build_of_keycloak/26.2/html/server_administration_guide/managing_organizations).
  - Auth0 invitations bind to the invited email (https://auth0.com/docs/manage-users/organizations/configure-organizations/invite-members).
- **Fix.** Add `iam.invitation`. Every administrator-created person and every bootstrap administrator goes through it. A profile is created only when the invitation is accepted, by the account that owns the email. The API returns the same 202 whether or not the account exists.

### F2. High: the first administrators depend on an undefined record that crosses modules; the tenant is active with no administrators (12.1, 15.3, 27.6)
- **Problem.**
  - 12.1 sends "administrator emails … in a short-lived encrypted invitation record referenced by the event", but no table holds it and no module owns it. If Tenancy owns it, IAM reads Tenancy data, which breaks Architecture rule 1.
  - The tenant reaches `active` before any TENANT_ADMIN exists. The two-administrator rule (15.3) is violated from day one.
  - 27.6 requires "bootstrap administrators matched to registered signatories, checked by a second platform person". The data model has no signatories and no approver.
- **Evidence.** AWS separates tenant management (tenant record) from user management (creating the tenant admin in the identity provider) inside the control plane (https://aws.amazon.com/blogs/apn/tenant-onboarding-best-practices-in-saas-with-the-aws-well-architected-saas-lens).
- **Fix.**
  - Tenancy creates the tenant and stores no personal data.
  - IAM learns about the tenant from `TenantCreatedV1`, which creates the `iam.tenant_policy` row.
  - The console then calls an IAM platform endpoint to create two `bootstrap` invitations. A different platform profile with the Compliance role must approve them (`approved_by_profile_id`). The signatory check is a manual step recorded in the approval reason.
  - The system path grants TENANT_ADMIN on acceptance.
  - The two-administrator rule is restated as "no operation may reduce the count below min(2, current count)", so bootstrap is never in violation.

### F3. High: tenant-free sessions conflict with per-host BFF cookies and per-role session policy (2, 13.2, 27.2, Auth tab 1 and 12)
- **Problem.**
  - 27.2 puts a BFF "on each society host" with a `__Host-csq_session` cookie. `__Host-` cookies are bound to one host, so `ikeja.colsynq.app` and `lekki.colsynq.app` have separate sessions. "Switching society needs no new login" (Auth tab 12) then fails on the web.
  - 27.2's policy table has members at 24 h max and 1 h idle, and administrators at 12 h max and 15 min idle. A user who is an administrator in society A and a member in B has one global session, so the policy is undefined.
- **Fix.**
  - Keep `auth_session` global and evaluate the policy per request: a `SessionPolicyCheck` in pipeline step 3 compares session age, idle time and `assurance_level` with the requirement of the current profile's roles. A failed check returns a 401 step-up, not a session limit.
  - For the web, serve all societies from one app host (`app.colsynq.app`, society chosen by `X-Tenant`).
  - Custom domains get a BFF session through the existing one-time handoff code from 27.4, redeemed on the custom host.
  - Write this down once in a "Sessions" section.

### F4. High: tenant offboarding destroys records that must be kept (Multi-tenancy 13, v2 12.1, 27.8)
- **Problem.**
  - The Multi-tenancy runbook goes "export → grace period → DROP DATABASE … → … tenant's encryption key".
  - 27.8 says "Financial and AML records kept at least 5 years after the relationship ends … tenant deletion follows the same rule".
  - v2 12.1 `deleting` closes profiles immediately.
  - Section 5 says "Audit kept 7 years", but 27.8 says "≥ 5 years after the relationship ends", which can run past 7 years.
- **Evidence.** Azure separates deactivation (temporary) from offboarding, which has a retention period (https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/considerations/tenant-life-cycle).
- **Fix.** One lifecycle: `active → suspended ↔ active → archived` (read-only, database kept, TENANT_ADMIN and AUDITOR read) → `deleting` → `deleted`. The move from archived to deleting happens only after `retention ends` and with no legal hold. Audit retention becomes "until the later of 7 years or the end of the AML retention".

### F5. High: the audit hash chain conflicts with IP truncation and adds a per-tenant write bottleneck (5, 8.6, 27.8)
- **Problem.**
  - `audit_event` stores `ip` and `user_agent` and is append-only and hash-chained, yet 27.8 truncates IPs after 13 months. That needs an UPDATE, which breaks the chain and the INSERT-only grant.
  - A per-tenant chain serialises every audit insert for that tenant, because each row needs the previous row's hash, and that needs an advisory lock per tenant per write.
  - `tenant_id` is null for platform-wide events, so the chain key is undefined for them.
- **Fix.**
  - Remove `ip` and `user_agent` from `audit_event` and store `session_id`. The IP lives in `auth_session`, which is pruned after 13 months, and in logs.
  - Chain on `coalesce(tenant_id, platform_tenant_id)`.
  - Or chain per daily batch: a signed checkpoint job hashes each day's rows in id order. This keeps tamper evidence without lock contention, and is the recommended option.

### F6. High: two different provisioning specifications (12.1 vs Multi-tenancy 4 and 9)
- **Problem.**

  | Topic | v2 / section 27 | Multi-tenancy tab |
  |---|---|---|
  | Status values | `provisioning, active, suspended, archived, deleting, deleted, failed` | `pending, provisioning_db, migrating, seeding, active, suspended, failed, deleting, deleted` |
  | Connection details | separate `tenant_database` | `db_*` columns on `tenant`, plus `plan text` |
  | Failure detail | `tenant_job.step`, `error` | `failed_step`, `last_error` |
  | Migration results | `tenant_job` (kind migrate, rollout_id) | `tenant_migration_run` |
  | Case-insensitive values | "no citext" | `citext` for slug and host |
  | Activation event | `TenantProvisionedV1` | `TenantActivatedV1` |
  | Who creates first officers | IAM | "SeedTenant … first officers" in the tenant database |
  | Route bypass | "no path bypass list" | "a configured bypass list" |
  | Resolver reference | — | points to "User tab section 10.3", which does not exist (it is 12.2) |

- **Fix.** The Multi-tenancy tab keeps only the mechanics: driver middleware, template, migration runner, relay and PgBouncer. The registry, the state machine and the bootstrap belong to the IAM and tenancy tab and are written once (part d). Step progress belongs in `tenant_job.step`, not in tenant status.

### F7. High: webhook tenant resolution cannot work as modelled (27.7, Auth tab 11.5)
- **Problem.**
  - 27.7 resolves the tenant "from the provider account or merchant ID in the verified payload". `integration.tenant_integration` has `tenant_id, provider, vault_ref, key_id, rotated_at, next_rotation_due, status` and no external account ID, so the lookup is impossible.
  - `_tenant = webhook` is not in the `TenantRequirement` enum (12.2: Required | Optional | None | Platform) or in TenantRequirementTest.
  - The Authentication tab says webhook senders use "connector configuration in the tenant".
  - The Architecture guide says a module belongs to exactly one plane, but no plane is given for Integrations.
- **Fix.**
  - Integrations is a landlord module with `integration.provider_account (provider, external_account_id UNIQUE, tenant_id, vault_ref, …)`.
  - `inbound_webhook` deduplicates in the landlord and is pruned after 30 days.
  - Verified events are published with a TenantStamp.
  - Add `Webhook` to `TenantRequirement`.

### F8. Medium: `user_device` duplicates the `device_key` authenticator (7.3, 27.3, 27.9)
- **Problem.** 27.3 adds authenticator type `device_key` ("attested mobile key") and also a table `user_device` (public_key, attestation, revoked_at). Both hold the same key with the same lifecycle. `auth_session.device_id` points at one of them and the authenticator's `credential_id` at the other.
- **Fix.**
  - Merge them into `authenticator` with `type = 'device_key'`, keeping the attestation in `credential_data`.
  - Express "one active mobile device for money" as a partial unique index `(user_id) WHERE type='device_key' AND revoked_at IS NULL`.
  - The ₦20,000 limit for a new device derives from `confirmed_at`.
  - Browser "trusted devices" only skip re-authentication after idle (27.3), which is a session property, so no row is needed.

### F9. Medium: refresh-token family duplicates auth_session; three revocation mechanisms (7.3, 25.12, 26, 27.2, Auth tab 7–9)
- **Problem.**
  - In the gesdinet entity, `family` is the session, `family_valid` is `auth_session.revoked_at`, `username` is `auth_session.user_id`, and `valid` is an expiry timestamp despite its name.
  - Revocation runs through three paths: `token_version` (all sessions), a Redis `sid` deny-list (one session), and a `jti` blocklist (one token, plus `block_previous_jwt`).
  - The gesdinet configuration is global: one TTL, one SameSite=Strict cookie path. That cannot express 27.2's per-client policy or DPoP binding. The Auth tab already marks the mobile variant "(verify)".
- **Evidence.** RFC 9700 §4.14 (refresh token rotation and reuse detection) needs only "token → grant/session" plus a used marker (https://www.rfc-editor.org/rfc/rfc9700#section-4.14).
- **Fix.**
  - Use a small IAM-owned `refresh_token (token_hash PK, session_id, created_at, expires_at, used_at)`. Reuse outside the 30 s grace period revokes the session.
  - Keep `token_version`, which is checked on the user row already loaded per request, and the `sid` deny-list. Drop the `jti` blocklist: logout revokes the session.
  - Dropping gesdinet is a dependency change, so it needs approval.

### F10. Medium: the Authentication tab contradicts section 27 in at least 10 places
- **Problem.**
  - Email OTP is still a factor (1, 5.1, 12). 27.3 removed it.
  - SPA access token in memory, refresh cookie SameSite=Strict on `/api/token/refresh` (1, 7.5, 8). 27.2 uses a BFF everywhere with `__Host-csq_session` (Lax, Path=/).
  - The lcobucci default encoder is used, and the claims list has no sid, acr, amr, client_id or cnf. 27.2 says the default encoder does not check iss, aud or typ.
  - API key lookup is plain SHA-256 (11.4, and the `hash('sha256', …)` in the code sample). v2 7.3 says HMAC with a pepper.
  - Sign-up "creates the User (email not yet verified) and UserProfile in one transaction; default role from the tenant's policy" (12). v2 13.1 creates nothing before verification, and the role is fixed to SELF_SERVICE.
  - Magic login links (3, 12) are a single factor that bypasses the MFA rules in 27.3 and are not in v2.
  - `login_throttling: max_attempts 5 / 15 minutes` locks users out. 27.3 says "lockouts never apply to the IP-throttled anonymous path".
  - `trusted_device_version` is still listed.
  - Signing keys are "in the vault with a passphrase". 27.8 says non-exportable KMS or HSM.
  - First administrators are "created with the tenant (two)". 27.6 requires KYB and approval.
- **Fix.** The Authentication tab keeps package configuration only and points to the single specification. Delete magic links, or allow them only as a step before MFA.

### F11. Medium: `require_email_verification` is dead (6.1, 25.1)
- **Problem.** Every path that creates an account now proves the email first:
  - Self sign-up uses a pending registration (13.1).
  - Social sign-in requires a provider-verified email (26.7, 27.4).
  - The admin path uses a set-password link, which "verifies the email".

  The flag can never be false in practice, and a CHECK already forces it for `email_domain`.
- **Fix.** Drop the column. State the invariant: **a person account exists only with a proven email.** Then `email_verified_at` is always set at creation and can be dropped too, together with the "unverified" branch of UserChecker. An email change takes effect only after the new address is verified, through a signed link.

### F12. Medium: controls referenced but never stored
- **Problem.**
  - "MFA-required tenants" (14 StepUpAssertion, 27.4 "Tenant MFA policy is enforced in pipeline step 3") has no column.
  - "legal hold flag" (27.1 S22) has no column.
  - "profiles dormant for 90 days", "grants unused for 90 days" (27.6) have no activity data, and per-grant usage cannot be measured.
  - "Suspended automatically" (27.6) is not a profile or grant state.
- **Fix.**
  - Add `tenant_policy.mfa_required`.
  - Model legal hold as `tenant.legal_hold_until`, because AML retention is per tenant. For persons, `user_account.erasure_requested_at` plus a retention calculation from profile `status_changed_at`.
  - Add `profile.last_seen_at`, written at most once a day.
  - Define dormancy as "disable the profile (status `disabled`, reason `dormant`) unless it would breach the two-administrator rule".
  - Replace "grants unused" with the 90-day access review of financial and high-risk grants.

### F13. Medium: money permissions contradict each other (10, 11, 27.6)
- **Problem.**
  - Sections 10 and 11 still use `execute` for money (`supplier-invoices:execute`, FINANCE_OFFICER "money execute permissions").
  - The section 11 conflict table has `payments:execute × user-roles:create = warn`.
  - 27.6 replaces `execute` with `initiate` and `release`, and makes "any money permission × user-roles:create" a block.
  - The risk value `money` is a category, not a level.
- **Fix.**
  - Actions for money resources become `initiate` and `release`.
  - The `risk` values become `low | high | financial`.
  - Default conflicts are listed once, in code (SystemRoleCatalogue), and the doc shows only examples.

### F14. Medium: support access is defined three times (14 step 3b, 16, 27.5; Auth tab 12)
- **Problem.**
  - Section 16 has emergency access "without a grant: PLATFORM_OWNER only … 1 hour".
  - 27.5 has emergency grants as rows with `approved_by_profile_id` of a second owner.
  - Section 14 points to "the impersonation limits of the Authentication tab", which 27.5 replaces with RFC 8693 token exchange.
  - `granted_by_profile_id` (society administrator) and `approved_by_profile_id` (second platform owner) are the same fact, "who approved", stored in two columns.
- **Fix.** Use one table with a `type` (standard, emergency) and one `approved_by_profile_id`:
  - for `standard`, a profile in the target tenant;
  - for `emergency`, a different PLATFORM_OWNER profile.

  CHECK constraints cover the window (4 h standard, 1 h emergency) and the impersonation target.

### F15. Medium: crypto-shredding cannot apply to the login email (13.3, 27.8)
- **Problem.** "Crypto-shred personal data after retention" needs per-user keys. `user_account.email` must stay plaintext or deterministic for the unique index and for login lookup, and no per-user key or blind index is modelled. 27.8's "per-tenant data keys" do not fit global users.
- **Fix.**
  - In the landlord, erasure at the end of retention overwrites personal columns (email, phone, names) with tombstones, keeps the IDs, and sets `status = 'erased'`.
  - Crypto-shredding applies to tenant-database encrypted fields (BVN, bank details) and to backups through the tenant key.
  - Write this as one rule. "Restrict processing" is `status = 'deactivated'` plus `erasure_requested_at`.

### F16. Medium: `transaction_authorisation` is in the wrong database and module (27.6)
- **Problem.**
  - Payments and the release record live in the tenant database. The signature evidence sits in the landlord IAM table, so the "single use" and "recompute at release" checks cannot run in the same transaction as the release.
  - The challenge itself is 5-minute, single-use, short-lived state.
- **Fix.**
  - Put the challenge nonce in Redis, with the noeviction policy already required by 27.2.
  - IAM's `TransactionSigningAssertion` verifies the signature with the device-key or passkey public key from `authenticator`.
  - The Payments module stores the evidence (method, signature, payload hash) on its release record in the tenant database, in the same transaction.
  - This removes one landlord table and keeps non-repudiation evidence beside the money record.

### F17. Medium: API key columns duplicate authority and tenant (7.3, 27.7)
- **Problem.**
  - `api_key.scopes` narrows a service account that already has exactly one profile with its own role assignments. That is two authority mechanisms.
  - `environment` (live, test) has no meaning without a sandbox deployment. The prefix `csk_live_` / `csk_test_` already encodes it.
  - `tenant_id` is derivable from the profile, and the key is always resolved through the profile.
- **Fix.**
  - Drop `scopes`: give a narrow role to a separate service account.
  - Derive "money key" rules (IP allow-list required, 90-day maximum, request signing) from the profile's effective permissions having `risk = financial`.
  - Drop `environment` and `tenant_id`.

### F18. Medium: `access_version` on every profile causes fan-out writes (7.2, 8.3, 15.4)
- **Problem.** Editing a role's permissions or archiving a role bumps `access_version` on every holder's profile, which is an N-row update under a lock. A role edit also changes rows in another aggregate (Profile), breaking "one aggregate per transaction".
- **Fix.**
  - Keep one counter per tenant: `iam.tenant_policy.access_version`, bumped by any access change in that tenant. The cache key becomes `access:{tenant}:{profile}:{tenant_access_version}:{catalogue_version}`.
  - Cooperatives have tens to low thousands of profiles, so a tenant-wide cache miss costs little, and the per-holder bump logic and its tests disappear.
  - Profile status changes also bump it.

### F19. Medium: global defaults stored as nullable-tenant rows (5, 8.5, 8.6, 10)
- **Problem.**
  - Default assignment edges and default conflicts are `tenant_id IS NULL` rows. That needs partial unique pairs, precedence logic and catalogue sync, even though they change only with a release.
  - `role.can_assign_all` repeats what a tenant-scope permission already says.
- **Evidence.** System roles must be rows because assignments reference them. Edges and conflicts are only read by the application.
- **Fix.**
  - Default edges and default conflicts live in `SystemRoleCatalogue` (code).
  - `role_assignment_rule` holds tenant rows only (`tenant_id NOT NULL`).
  - "Assign any non-platform role" becomes the permission `role-assignments:assign-any` at TENANT scope.
  - Tenant-defined stricter SoD rules are deferred. Defaults in code keep the security control, so `permission_conflict` is dropped from v3 core.

### F20. Medium: membership policy lives in Tenancy, and `platform_tenant` exists only because IAM cannot read Tenancy (6.1, 8.6, 3)
- **Problem.**
  - `signup_mode`, `signup_email_domains` and `require_email_verification` are IAM rules (who may join) stored in `tenancy.tenant` and read back by IAM through TenantContext. That needs an extra event (`TenantSignupPolicyChangedV1`) and a Tenancy endpoint for an IAM concept.
  - `iam.platform_tenant` is a singleton copy of `tenant.kind`.
- **Fix.** Create one IAM-owned row per tenant, **`iam.tenant_policy`**, made by `TenantCreatedV1`. Its columns:
  - `is_platform` (partial unique), replacing `platform_tenant`;
  - `join_mode` and `join_email_domains`;
  - `mfa_required` (F12);
  - `access_version` (F18).

  It becomes the real FK target for `profile.tenant_id` and `role.tenant_id` inside IAM, so links that were logical become enforced.

### F21. Medium: personal data on the profile is duplicated per tenant (7.2, 25.9)
- **Problem.**
  - `first_name` and `last_name` are copied into every profile of the same person, and the global account has no name for platform emails.
  - `metadata` jsonb ("staff number, branch, next of kin") holds personal and third-party data in IAM. That contradicts "keeps IAM generic", widens erasure scope, and overlaps the future Membership module (21).
- **Evidence.** WorkOS, Clerk and Keycloak keep names on the user. Organization-specific attributes go to membership metadata, or in Colsynq's case to a business module (https://clerk.com/docs/reference/backend/types/backend-organization-membership).
- **Fix.**
  - Put `given_name` and `family_name` (OIDC claim names) on `user_account`.
  - Drop `metadata`. Society-collected member data belongs to the Membership module in the tenant database.

### F22. Low: profile provenance is duplicated (7.2)
- **Problem.** `source` and `created_by_profile_id` repeat the audit `profile.created` event and the invitation (`invited_by_profile_id`, `type`). The bootstrap case needs a trigger to let a platform profile appear as creator in a society.
- **Fix.** Drop both columns and the trigger.

### F23. Medium: DDD aggregate boundaries are implicit (3, 13, 15)
- **Problem.**
  - `RegisterUserInTenant` writes UserAccount, UserProfile and UserRole in one transaction. Architecture rule 10 says "one aggregate per command".
  - The two-administrator rule is a set-based invariant across profiles.
  - Role edits affect holders (F18).
  - Tenancy owns the registry, resolver, domains, jobs, plans and subscriptions, which mixes SRP concerns.
- **Fix.** Declare the aggregates:

  | Aggregate | Contains |
  |---|---|
  | `UserAccount` | account, authenticators, external identities |
  | `AuthSession` | session, refresh tokens |
  | `Profile` | profile, role assignments, so SoD on effective permissions is a Profile invariant |
  | `Role` | role, permission codes |
  | `Invitation` | invitation |
  | `SupportAccessGrant` | grant |
  | `TenantPolicy` | policy row |
  | `Tenant` | tenant, domains |
  | `TenantDatabase` | connection record |
  | `TenantJob` | job |
  | `Subscription` | subscription |

  - Creating an account and its first profile is a documented exception: the account is created first and a domain event creates the profile in the same transaction (Architecture 5.1 allows in-module domain events).
  - The two-administrator rule is a domain service under a tenant advisory lock.
  - Plans and subscriptions are a candidate Billing module later. Until then their catalogue is code.

### F24. Medium: who may act on a global account is unclear (13.3, Auth tab 12)
- **Problem.** "Deactivate the User" and "Erase the User" are listed among profile changes, with no actor named. A tenant administrator must not be able to deactivate an account that also belongs to other societies. "Re-activation through platform support" (Auth tab) has no flow in v2.
- **Fix.**
  - Tenant administrators manage only profiles: disable, enable, remove, and assignments.
  - The account holder, or platform staff with `user-accounts:manage`, manages the global account: deactivate, reactivate, erasure.

### F25. Low: the platform bootstrap is not specified (2, 8.6 platform_tenant "Seeded at bootstrap", 15.3)
- **Problem.** The design does not say how the platform tenant and the first two PLATFORM_OWNERs come to exist, given that the gates require an existing grantor.
- **Fix.**
  - A one-off console command creates the platform tenant and its `tenant_policy(is_platform)`, and issues two bootstrap invitations through the system path.
  - It refuses to run when any PLATFORM_OWNER exists, is audited, and is listed in the go-live checklist.

### F26. Medium: three lockout counters (7.1, 7.3, 27.3, Auth tab 7.4)
- **Problem.** Three counters overlap and the policy is spread across them:
  - `app_user.failed_login_count` / `locked_until`;
  - `user_authenticator.failed_count`;
  - Symfony `login_throttling` (5 attempts per 15 minutes).
- **Fix.**
  - Throttle with a Redis limiter per IP and per account. That throttling never sets a lock.
  - Keep `user_account.failed_sign_in_count` and `locked_until` only for the privileged-user lock in 27.3.
  - Keep `authenticator.failed_count` for NIST's 100-failure limit.
  - Write the policy once.

### F27. Low: tenant slug reuse and deleted rows (6.1)
- **Problem.** A `deleted` tenant row keeps a plain `UNIQUE(slug)`, so the slug can never be reused.
- **Fix.** Use a partial unique index on slug `WHERE status <> 'deleted'`. Keep deleted rows for the retention period, because audit references them.

### F28. Low: `external_identity` soft unlink blocks relinking; provider and issuer are mixed (7.3, 25.11, 27.4, 27.9)
- **Problem.**
  - 7.3 has "Unique (provider, subject)" and 27.9 has "unique (issuer, subject)".
  - Combined with `unlinked_at`, an unlinked identity can never be linked again to another account.
- **Fix.**
  - Use `issuer` only. The provider name comes from configuration keyed by issuer.
  - Hard-delete on unlink, with the audit record kept.
  - Drop `unlinked_at`, `linked_at` (use `created_at`) and `email_verified` (it is checked only at link time).

### F29. Low: `one_time_token` is unnecessary (7.3, 25.14)
- **Problem.**
  - The selector/verifier split guards low-entropy tokens. For 256-bit random tokens a SHA-256 lookup is enough.
  - The `signup` purpose stores an encrypted password hash before verification, which needs a key_id under 27.8.
  - `set_password` is replaced by invitations (F1).
  - v2 already makes verification and join links stateless signed URLs.
- **Fix.**
  - Pending sign-up becomes a stateless signed link carrying email, tenant and expiry. The user chooses a password after clicking, so no pending password is stored.
  - Password reset becomes a signed link whose signature includes `token_version` and the password hash, so it is single use, as in Symfony login-link `signature_properties` (https://symfony.com/doc/current/security/login_link.html).
  - Email change becomes a signed link.
  - Drop the table. Rate-limit resends in Redis.
  - Trade-off: no per-link revocation. A credential change bumps `token_version` and invalidates all links.

### F30. Low: plans and features are code-time configuration (6.3, 25.4)
- **Problem.** `feature`, `plan` and `plan_feature` change only with a product release. `permission.feature_code` already refers to code-defined features.
- **Fix.**
  - Use a versioned `PlanCatalogue` in code, the same pattern as the permission catalogue.
  - `subscription` stores `plan_code` and `plan_version`.
  - Keep `entitlement_override`, which is runtime data.

### F31. Low: naming problems
- **Problems.**
  - `kind` and `type` are both used (`tenant.kind`, `app_user.kind`, `tenant_job.kind`, `user_device.kind`, `scope_reference.kind`, `actor_kind`, support `kind`, versus `user_authenticator.type` and `subject_type`).
  - `permission.slug` sits beside `role.code`, `feature.code` and `plan.code`.
  - FKs are written as `user_profile_id` (api_key, user_role) and `profile_id` (transaction_authorisation, support grant).
  - `tenant.state` (a Nigerian state) sits next to `status`.
  - `app_user` reuses the `app_` prefix of PostgreSQL roles (`app_owner`, `app_migrator`, `app_runtime`, `app_landlord`).
  - `user_role_assign_role` joins table names, which SQL style guides advise against (https://www.sqlstyle.guide/).
  - `scope_reference` and `scope_ref_id` are vague.
  - `one_time_token` is vague.
  - gesdinet's `valid` holds a timestamp.
  - Role scope values are UPPERCASE while every other value set is lowercase.
  - `primary_colour` and `authorisation` use UK spelling in identifiers.
  - `db_host` sits in a table already called `tenant_database`.
- **Evidence.**
  - `user` is reserved in PostgreSQL and `role` is not (https://www.postgresql.org/docs/current/sql-keywords-appendix.html).
  - GitLab constraint naming: https://docs.gitlab.com/development/database/constraint_naming_convention/
  - NIST calls user-role links "user assignment (UA)" (https://csrc.nist.gov/projects/role-based-access-control/faqs).
  - ZITADEL renamed "user grant" to "role assignment" (https://zitadel.com/docs/guides/manage/console/roles).
  - Spatie uses `model_has_roles` and `role_has_permissions` (https://github.com/spatie/laravel-permission/blob/main/config/permission.php), which shows join-style names are a framework habit, not a standard.
- **Fix.** Part (c) conventions and rename map. Use `type` for classification, `code` for stable keys, `status` for lifecycle, and `<singular_table>_id` for FKs, with one documented exception (`user_id` → `user_account`). Value sets are lowercase snake_case. Identifiers use US spelling and prose keeps UK spelling, or the reverse, applied consistently.

### F32. Low: stale counts, diagrams and references
- **Problem.**
  - The section 3 and section 4 diagrams omit `auth_session`, `user_device` and `transaction_authorisation`, yet the text says "21 in iam".
  - The section 17 code layout omits them.
  - 25.8 (app_user field guide) stops after `kind`.
  - The Auth tab points to "User tab section 11" (sign-up is 13).
  - The Multi-tenancy tab points to "section 10.3" (it is 12.2).
  - 25.10 still says the factors are "authenticator apps (TOTP), email codes".
  - 7.3 keeps `backup_codes` as a type and also a `backup_code` table.
- **Fix.** Section 25 (the field guide) duplicates sections 6–8. Merge them into one table per entity with a "why" column (part e).

### F33. Low: grant expiry rules conflict (8.4, 15.3, 27.1 S63)
- **Problem.** 15.3 says TENANT_ADMIN and PLATFORM_OWNER grants "cannot expire". S63 lists "platform grant expiry" as a fix.
- **Fix.** Platform roles except PLATFORM_OWNER get `max_grant_duration` of 180 days with re-grant at access review. Owners and tenant administrators do not expire, and are covered by the 90-day review.

### F34. Low: SELF_SERVICE assignment is implicit, so the row is redundant (11, 13, 15.2)
- **Problem.** Every active person profile gets SELF_SERVICE through the system path, and it can never be revoked separately. A stored row that always exists for every profile is a derivable fact.
- **Fix.** This is optional. `EffectiveAccessLoader` adds SELF_SERVICE implicitly for active person profiles, which removes one write per join and rejoin and the "system grant" special case for it. Keep it as a row only if tenants must be able to withhold self-service.

---

## (b) Redundancies and contradictions

| # | Topic | Where | Conflict | Resolution |
|---|---|---|---|---|
| 1 | Tenant status set | v2 6.1 vs MT 4 | 7 values vs 9 (provisioning_db, migrating, seeding) | v2 set; steps in tenant_job |
| 2 | Tenant DB connection | v2 6.2 vs MT 4 | Separate table vs columns on tenant | Separate `tenant_database` |
| 3 | Failure details | v2 tenant_job vs MT failed_step, last_error | Two places | tenant_job only |
| 4 | Migration results | v2 tenant_job(migrate) vs MT tenant_migration_run | Two tables | tenant_job (type migrate) |
| 5 | citext | v2 5 "no citext" vs MT 4 citext | — | No citext |
| 6 | Activation event | TenantProvisionedV1 (v2) vs TenantActivatedV1 (MT) | — | TenantActivatedV1 (state reached), plus TenantCreatedV1 |
| 7 | First officers | IAM (v2 12.1) vs SeedTenant in the tenant DB (MT 9.2) | — | IAM invitations; the seed is reference data only |
| 8 | Route bypass list | v2 "no bypass list" vs MT 7 "configured bypass list" | — | No bypass list |
| 9 | TenantRequirement values | 12.2 (4 values) vs 27.7 `webhook` | — | Add Webhook |
| 10 | Plan on tenant | MT `plan text` vs v2 subscription | — | subscription |
| 11 | Offboarding | MT 13 drops the DB vs 27.8 retention | Data loss | archived → deleted after retention |
| 12 | Audit retention | 5 "7 years" vs 27.8 "≥5 years after relationship end" | — | The later of the two |
| 13 | Erasure | 5 "blank columns" vs 13.3 / 27.8 crypto-shred | — | Overwrite in landlord after retention; crypto-shred tenant fields (F15) |
| 14 | Email OTP factor | Auth 1, 5.1, 12; v2 25.10 vs 27.3 | — | Removed |
| 15 | Web token storage | Auth 1, 7.5, 8 vs 27.2 BFF | — | BFF |
| 16 | Refresh cookie | Auth 8 Strict, path /api/token/refresh vs 27.2 `__Host-`, Lax, Path=/ | — | 27.2 |
| 17 | JWT encoder and claims | Auth 7 default encoder, 7 claims vs 27.2 web-token encoder, 14 claims | — | 27.2 |
| 18 | Signing keys | Auth 7.1 vault plus passphrase vs 27.8 KMS/HSM | — | KMS |
| 19 | API key hash | Auth 11.4 SHA-256 vs v2 HMAC plus pepper | — | HMAC plus pepper_version |
| 20 | Sign-up creates user | Auth 12 "email not yet verified" vs v2 13.1 pending | — | Nothing before verification |
| 21 | Sign-up role | Auth 12 "default role from the tenant's policy" vs v2 6.1 fixed SELF_SERVICE | — | Fixed, or implicit (F34) |
| 22 | Admin adds existing email | v2 13.1 "profile + notice" vs Auth 11.1/12 "never attached by email" | — | Invitation (F1) |
| 23 | Magic links | Auth 3, 12 vs absent in v2 / 27.3 MFA rules | — | Drop |
| 24 | Login throttling | Auth 7.4 lockout 5 per 15 min vs 27.3 "no lockout on anonymous path" | — | Redis throttle only (F26) |
| 25 | trusted_device_version | v2 7.1 and Auth 5.1 vs 27.3 "dropped" | — | Drop |
| 26 | Device key | authenticator.type device_key vs user_device | Same fact twice | Merge (F8) |
| 27 | Refresh family vs session | refresh_token.family / family_valid vs auth_session | Same fact twice | Lean refresh_token (F9) |
| 28 | Revocation | token_version + sid deny-list + jti blocklist | Three paths | token_version + sid |
| 29 | Platform tenant | tenant.kind=platform vs iam.platform_tenant | Same fact twice | tenant_policy.is_platform |
| 30 | Sign-up policy owner | Tenancy columns vs IAM logic | Wrong owner | tenant_policy |
| 31 | Money actions | 10, 11 `execute` vs 27.6 initiate/release; warn vs block | — | 27.6 |
| 32 | Emergency support | 16 "without a grant" vs 27.5 grant row | — | Grant row, type emergency |
| 33 | Impersonation | 14 "limits of the Auth tab" vs 27.5 token exchange | — | 27.5 |
| 34 | Support approver | granted_by_profile_id vs approved_by_profile_id | Same fact twice | approved_by_profile_id |
| 35 | external_identity unique key | 7.3 (provider, subject) vs 27.9 (issuer, subject) | — | issuer |
| 36 | Profile FK names | user_profile_id vs profile_id | — | profile_id |
| 37 | Actor columns | audit actor_profile_id derivable from (actor_user_id, tenant_id) | Same fact twice | Drop actor_profile_id |
| 38 | Profile provenance | source / created_by_profile_id vs audit and invitation | Same fact twice | Drop |
| 39 | Signatories | 27.6 "matched to registered signatories" vs no data model | — | Four-eyes approval on bootstrap invitation |
| 40 | Two-admin rule at bootstrap | 15.3 vs 12.1 (tenant active with 0 admins) | — | min(2, current) |
| 41 | Grant expiry | 15.3 "cannot expire" vs S63 | — | F33 |
| 42 | Table counts and diagrams | 3, 4, 17 vs 27.9 | — | Regenerate |
| 43 | Field guide | 25 duplicates 6–8; 25.8 truncated | — | Merge |
| 44 | ProfileLoginValidator and switcher | 13.2 vs 27.2 BFF per host | Switching breaks | F3 |
| 45 | resource and action | permission.resource/action plus a CHECK equal to slug | Derivable | Drop |
| 46 | can_assign_all | role flag vs a TENANT-scope permission | Same fact twice | Permission |
| 47 | Locale | tenant.region fixed to `ng` plus a CHECK | Constant | Drop region |
| 48 | require_email_verification | Always true in practice | Dead | Drop |
| 49 | provisioned_at / activated_at | tenant_job.finished_at / subscription.starts_at / audit | Derivable | Drop |
| 50 | Integrations | Auth 11.5 "connector config in tenant" vs 27.7 tables in the Integrations module without a plane | — | Landlord Integrations module (F7) |

MT = Multi-tenancy tab, Auth = Authentication tab.

---

## (c) Proposed lean v3 data model

### c.1 Conventions (replaces v2 section 5 naming rows)
- Tables are singular snake_case nouns, one schema per module, with no schema prefix repeated in the table name (`tenancy.subscription`, not `tenancy.tenant_subscription`, unless the name would be ambiguous).
- FK column = `<referenced singular table>_id`. Documented exception: `user_id` → `iam.user_account`, because `user` is reserved and the entity is `User`. Role-qualified FKs use a prefix: `approved_by_profile_id`, `support_profile_id`.
- Timestamps: `*_at timestamptz`, past participle (`created_at`, `revoked_at`, `expires_at`, `starts_at`).
- Booleans: `is_*` or `has_*`. Prefer a nullable `*_at` when the time matters.
- Classification is `type`, lifecycle is `status` (+ `status_reason`, `status_changed_at` when set by people), and stable business keys are `code`. Value sets are lowercase snake_case, as text plus a CHECK mapped to PHP backed enums.
- Secrets: `*_hash` (one-way) or `*_encrypted bytea` + `key_id`.
- Constraints: `<table>_<cols>_pkey|_fkey|_key|_check|_idx` (PostgreSQL's own default suffixes, so generated and hand-written names match), or adopt GitLab's `fk_/index_/unique_/check_`. Pick one.
- Single quotes are used for value literals.

### c.2 `tenancy` schema (6 tables, was 9)

**tenancy.tenant**: registry row, aggregate root `Tenant`.

| Column | Type | Constraints |
|---|---|---|
| id | uuid | PK (v7) |
| type | text | NOT NULL, CHECK in ('platform','society'); unique index `(type) WHERE type='platform'` |
| slug | text | NOT NULL, CHECK `^[a-z0-9][a-z0-9-]{1,62}$`; unique `(slug) WHERE status <> 'deleted'` |
| legal_name | text | NOT NULL |
| display_name | text | NOT NULL |
| registration_number | text | NULL; CHECK `type='platform' OR registration_number IS NOT NULL` |
| registering_authority | text | NULL |
| registration_state | text | NULL (Nigerian state code) |
| status | text | NOT NULL, CHECK in ('provisioning','active','suspended','archived','deleting','deleted','failed') |
| status_reason | text | NULL; CHECK required when status in ('suspended','archived','failed') |
| status_changed_at | timestamptz | NOT NULL |
| kyb_status | text | NOT NULL DEFAULT 'pending', CHECK in ('pending','verified','rejected','not_required') |
| kyb_verified_at | timestamptz | NULL; CHECK `(kyb_status='verified') = (kyb_verified_at IS NOT NULL)` |
| legal_hold_until | timestamptz | NULL (blocks deleting) |
| timezone | text | NOT NULL DEFAULT 'Africa/Lagos' |
| currency_code | char(3) | NOT NULL DEFAULT 'NGN' |
| logo_url | text | NULL (fetched only through the hardened client) |
| brand_color | text | NULL, CHECK `^#[0-9a-f]{6}$` |
| created_at, updated_at | timestamptz | NOT NULL |

**tenancy.tenant_domain**: custom domains only.

| Column | Type | Constraints |
|---|---|---|
| host | text | PK (normalised lower-case) |
| tenant_id | uuid | NOT NULL FK tenant; index |
| is_primary | boolean | NOT NULL DEFAULT false; unique `(tenant_id) WHERE is_primary`; CHECK `NOT is_primary OR verified_at IS NOT NULL` |
| verification_token_hash | text | NULL |
| verified_at | timestamptz | NULL |
| created_at | timestamptz | NOT NULL |

**tenancy.tenant_database**: 1:0..1, stricter grant.

| Column | Type | Constraints |
|---|---|---|
| tenant_id | uuid | PK, FK tenant |
| host | text | NOT NULL |
| port | integer | NOT NULL DEFAULT 5432 |
| database_name | text | NOT NULL UNIQUE, CHECK `^[a-z][a-z0-9_]{2,62}$` |
| login_role | text | NOT NULL UNIQUE |
| secret_ref | text | NOT NULL (vault path) |
| schema_version | text | NULL (mirror of the tenant's doctrine_migration_versions) |
| schema_status | text | NOT NULL DEFAULT 'ok', CHECK in ('ok','migrating','failed','drift') |
| migration_wave | smallint | NOT NULL DEFAULT 2, CHECK 0..3 |
| created_at, updated_at | timestamptz | NOT NULL |

**tenancy.tenant_job**: provisioning, migration, move, archive and delete. Replaces `tenant_migration_run`, `failed_step` and `last_error`.

| Column | Type | Constraints |
|---|---|---|
| id | uuid | PK |
| tenant_id | uuid | NOT NULL FK tenant |
| type | text | NOT NULL, CHECK in ('provision','migrate','move','archive','delete') |
| status | text | NOT NULL, CHECK in ('queued','running','succeeded','failed','cancelled'); unique `(tenant_id) WHERE status IN ('queued','running')`, which is the per-tenant lock and double-submit guard |
| step | text | NULL (last completed step) |
| attempt | smallint | NOT NULL DEFAULT 0 |
| rollout_id | uuid | NULL; index |
| details | jsonb | NULL (from_version, to_version, target host, …) |
| last_error | text | NULL |
| created_at | timestamptz | NOT NULL |
| started_at, finished_at | timestamptz | NULL |

**tenancy.subscription**

| Column | Type | Constraints |
|---|---|---|
| id | uuid | PK |
| tenant_id | uuid | NOT NULL FK tenant |
| plan_code | text | NOT NULL (PlanCatalogue in code) |
| plan_version | smallint | NOT NULL |
| status | text | NOT NULL, CHECK in ('trialing','active','past_due','cancelled','expired'); unique `(tenant_id) WHERE status IN ('trialing','active','past_due')` |
| starts_at | timestamptz | NOT NULL |
| ends_at | timestamptz | NULL |
| created_at, updated_at | timestamptz | NOT NULL |

**tenancy.entitlement_override**

| Column | Type | Constraints |
|---|---|---|
| tenant_id | uuid | PK part, FK tenant |
| feature_code | text | PK part (code catalogue) |
| is_enabled | boolean | NOT NULL |
| limit_value | integer | NULL (null = plan limit) |
| reason | text | NOT NULL |
| expires_at | timestamptz | NULL |
| created_at | timestamptz | NOT NULL |

### c.3 `iam` schema (17 tables, was 21)

**iam.tenant_policy**: IAM's own per-tenant row, created from TenantCreatedV1. Aggregate `TenantPolicy`.

| Column | Type | Constraints |
|---|---|---|
| tenant_id | uuid | PK (logical link to tenancy.tenant) |
| is_platform | boolean | NOT NULL DEFAULT false; unique `(is_platform) WHERE is_platform` |
| join_mode | text | NOT NULL DEFAULT 'closed', CHECK in ('closed','open','email_domain') |
| join_email_domains | text[] | NULL; CHECK `(join_mode='email_domain') = (cardinality(join_email_domains) > 0)` |
| mfa_required | boolean | NOT NULL DEFAULT false (true for the platform tenant) |
| access_version | bigint | NOT NULL DEFAULT 1 (bumped by any access change in the tenant) |
| created_at, updated_at | timestamptz | NOT NULL |

**iam.user_account**: entity `User`, the global login and JWT `sub`.

| Column | Type | Constraints |
|---|---|---|
| id | uuid | PK |
| type | text | NOT NULL, CHECK in ('person','service') |
| email | text | NULL, normalised; unique `(email) WHERE email IS NOT NULL`; CHECK `type='service' OR email IS NOT NULL` |
| phone | text | NULL (E.164); unique `(phone) WHERE phone_verified_at IS NOT NULL` |
| phone_verified_at | timestamptz | NULL |
| given_name, family_name | text | NULL (person names; overwritten at erasure) |
| password_hash | text | NULL; CHECK `type='person' OR password_hash IS NULL` |
| status | text | NOT NULL DEFAULT 'active', CHECK in ('active','deactivated','erased') |
| status_changed_at | timestamptz | NOT NULL |
| token_version | integer | NOT NULL DEFAULT 1 |
| failed_sign_in_count | smallint | NOT NULL DEFAULT 0 |
| locked_until | timestamptz | NULL |
| erasure_requested_at | timestamptz | NULL |
| created_at, updated_at | timestamptz | NOT NULL |

Invariant: a person row exists only after its email was proven (F11).

**iam.authenticator**: replaces user_authenticator, backup_code and user_device.

| Column | Type | Constraints |
|---|---|---|
| id | uuid | PK |
| user_id | uuid | NOT NULL FK user_account; index `(user_id) WHERE revoked_at IS NULL` |
| type | text | NOT NULL, CHECK in ('totp','passkey','device_key','backup_codes') |
| label | text | NULL |
| credential_id | bytea | NULL; unique where not null (passkey and device key lookup) |
| credential_data | jsonb | NULL (public key, sign count, attestation, transports; for backup_codes `[{hash, used_at}]`) |
| secret_encrypted | bytea | NULL (TOTP seed) |
| key_id | text | NULL; CHECK `(secret_encrypted IS NULL) = (key_id IS NULL)` |
| confirmed_at | timestamptz | NULL |
| last_used_at | timestamptz | NULL |
| failed_count | smallint | NOT NULL DEFAULT 0 (≥100 = locked) |
| revoked_at | timestamptz | NULL |
| created_at | timestamptz | NOT NULL |

Extra indexes: unique `(user_id) WHERE type='device_key' AND revoked_at IS NULL` (one money device); unique `(user_id) WHERE type='backup_codes' AND revoked_at IS NULL`. A trigger refuses rows for `type='service'` users.

**iam.external_identity**

| Column | Type | Constraints |
|---|---|---|
| id | uuid | PK |
| user_id | uuid | NOT NULL FK user_account; index |
| issuer | text | NOT NULL (OIDC iss, or `client:<issuer>`) |
| subject | text | NOT NULL (Microsoft: `tid:oid`); unique `(issuer, subject)` |
| email | text | NULL (as asserted; display only) |
| created_at | timestamptz | NOT NULL |
| last_used_at | timestamptz | NULL |

Unlinking deletes the row and writes an audit record.

**iam.auth_session**: the `sid`.

| Column | Type | Constraints |
|---|---|---|
| id | uuid | PK |
| user_id | uuid | NOT NULL FK user_account; index `(user_id) WHERE revoked_at IS NULL` |
| client_type | text | NOT NULL, CHECK in ('member_web','member_mobile','console','partner') |
| authenticated_at | timestamptz | NOT NULL (auth_time; changed only by sign-in, MFA, step-up) |
| auth_methods | text[] | NOT NULL (amr) |
| assurance_level | text | NOT NULL, CHECK in ('aal1','aal2','aal3') (acr) |
| authenticator_id | uuid | NULL FK authenticator (device key for mobile) |
| dpop_jkt | text | NULL |
| ip | inet | NULL (truncated after 13 months) |
| user_agent | text | NULL |
| created_at | timestamptz | NOT NULL |
| last_used_at | timestamptz | NOT NULL |
| revoked_at | timestamptz | NULL |
| revoke_reason | text | NULL; CHECK `(revoked_at IS NULL) = (revoke_reason IS NULL)` |

**iam.refresh_token**: replaces the gesdinet entity.

| Column | Type | Constraints |
|---|---|---|
| token_hash | bytea | PK (SHA-256 of a 256-bit token) |
| session_id | uuid | NOT NULL FK auth_session ON DELETE CASCADE; index |
| created_at | timestamptz | NOT NULL |
| expires_at | timestamptz | NOT NULL |
| used_at | timestamptz | NULL (reuse after used_at + 30 s → revoke session) |

**iam.api_key**

| Column | Type | Constraints |
|---|---|---|
| id | uuid | PK |
| profile_id | uuid | NOT NULL FK profile; index; trigger: the profile's user is `service` |
| name | text | NOT NULL |
| prefix | text | NOT NULL UNIQUE (`csk_live_…` display prefix) |
| key_hash | bytea | NOT NULL UNIQUE (HMAC-SHA-256 with a pepper) |
| pepper_version | smallint | NOT NULL |
| ip_allow_list | cidr[] | NULL (required by the application when the profile holds financial permissions) |
| expires_at | timestamptz | NOT NULL (≤1 year; ≤90 days if financial) |
| last_used_at | timestamptz | NULL |
| revoked_at | timestamptz | NULL |
| created_at | timestamptz | NOT NULL |

**iam.invitation**: new. Covers bootstrap administrators and administrator-created people.

| Column | Type | Constraints |
|---|---|---|
| id | uuid | PK |
| tenant_id | uuid | NOT NULL FK tenant_policy |
| email | text | NOT NULL (normalised) |
| type | text | NOT NULL, CHECK in ('bootstrap','standard') |
| role_id | uuid | NULL FK role (bootstrap: TENANT_ADMIN or PLATFORM_OWNER) |
| scope_id | uuid | NULL; FK `(tenant_id, scope_id)` → access_scope |
| invited_by_profile_id | uuid | NULL FK profile (null only for the platform bootstrap command) |
| approved_by_profile_id | uuid | NULL FK profile; CHECK `type='standard' OR approved_by_profile_id IS NOT NULL` before acceptance (application) and `approved_by_profile_id <> invited_by_profile_id` |
| token_hash | bytea | NOT NULL UNIQUE |
| expires_at | timestamptz | NOT NULL |
| accepted_at | timestamptz | NULL |
| accepted_user_id | uuid | NULL FK user_account; CHECK `(accepted_at IS NULL) = (accepted_user_id IS NULL)` |
| revoked_at | timestamptz | NULL |
| created_at | timestamptz | NOT NULL |

Unique `(tenant_id, email) WHERE accepted_at IS NULL AND revoked_at IS NULL`. Status (pending, accepted, expired, revoked) is derived from the timestamps.

**iam.profile**: a user's presence in one tenant; aggregate root with its role assignments.

| Column | Type | Constraints |
|---|---|---|
| id | uuid | PK; unique `(id, tenant_id)` (composite FK target) |
| user_id | uuid | NOT NULL FK user_account |
| tenant_id | uuid | NOT NULL FK tenant_policy; unique `(user_id, tenant_id)`; index `(tenant_id, status)` |
| status | text | NOT NULL, CHECK in ('active','disabled','removed') |
| status_reason | text | NULL (for example 'dormant', 'left') |
| status_changed_at | timestamptz | NOT NULL |
| last_seen_at | timestamptz | NULL (updated at most daily) |
| created_at, updated_at | timestamptz | NOT NULL |

Trigger: a `service` user has exactly one profile.

**Why "profile" and not "membership".** WorkOS, Clerk and Keycloak call this "organization membership" or "member". In a cooperative, "member" is a core business term: a shareholder member, owned by the future Membership module (v2 section 21). Keeping "profile" avoids two meanings of "member". The doc should say once: "profile = membership in the IAM sense".

**iam.permission**: synced from code.

| Column | Type | Constraints |
|---|---|---|
| code | text | PK, CHECK `^[a-z0-9-]+:[a-z0-9-]+$` (resource:action) |
| module | text | NOT NULL |
| name | text | NOT NULL |
| description | text | NULL |
| risk | text | NOT NULL, CHECK in ('low','high','financial') |
| allowed_scopes | text[] | NOT NULL |
| feature_code | text | NULL |
| deprecated_at | timestamptz | NULL |

**iam.role**

| Column | Type | Constraints |
|---|---|---|
| id | uuid | PK |
| tenant_id | uuid | NULL FK tenant_policy (null = system role); immutable (trigger) |
| code | text | NOT NULL; unique `(code) WHERE tenant_id IS NULL`, unique `(tenant_id, code) WHERE tenant_id IS NOT NULL`; trigger: no shadowing of system codes |
| name | text | NOT NULL |
| description | text | NULL |
| scope | text | NOT NULL, CHECK in ('platform','tenant','business_unit','venture','self'); immutable; CHECK `scope <> 'platform' OR tenant_id IS NULL` |
| max_grant_duration | interval | NULL |
| archived_at | timestamptz | NULL |
| created_at, updated_at | timestamptz | NOT NULL |

**iam.role_permission** (permission assignment, PA)

| Column | Type | Constraints |
|---|---|---|
| role_id | uuid | PK part, FK role ON DELETE CASCADE |
| permission_code | text | PK part, FK permission ON DELETE CASCADE; index |

**iam.role_assignment** (user assignment, UA)

| Column | Type | Constraints |
|---|---|---|
| id | uuid | PK |
| profile_id | uuid | NOT NULL |
| tenant_id | uuid | NOT NULL; FK `(profile_id, tenant_id)` → profile(id, tenant_id) |
| role_id | uuid | NOT NULL FK role; trigger: system role or the same tenant's, not archived; platform roles only in the `is_platform` tenant |
| scope_id | uuid | NULL; FK `(tenant_id, scope_id)` → access_scope; trigger: scope type matches the role scope |
| created_at | timestamptz | NOT NULL (requested) |
| starts_at | timestamptz | NOT NULL (= created_at, or +24 h for financial or high roles) |
| expires_at | timestamptz | NULL; CHECK `expires_at > starts_at` |
| revoked_at | timestamptz | NULL |

Indexes:
- unique `(profile_id, role_id, coalesce(scope_id, '00000000-0000-0000-0000-000000000000'::uuid)) WHERE revoked_at IS NULL`, one expression index instead of two partial ones (or `NULLS NOT DISTINCT` on PG15+ as a unique index with WHERE);
- `(tenant_id, role_id) WHERE revoked_at IS NULL`;
- `(expires_at) WHERE revoked_at IS NULL AND expires_at IS NOT NULL`.

**iam.role_assignment_rule** (ARBAC97 can_assign, tenant rows only; defaults in code)

| Column | Type | Constraints |
|---|---|---|
| id | uuid | PK |
| tenant_id | uuid | NOT NULL FK tenant_policy |
| assigner_role_id | uuid | NOT NULL FK role |
| assignable_role_id | uuid | NOT NULL FK role; unique `(tenant_id, assigner_role_id, assignable_role_id)` |
| effect | text | NOT NULL, CHECK in ('allow','deny') |
| created_at | timestamptz | NOT NULL |

**iam.access_scope**: projection of business units and ventures, replacing scope_reference.

| Column | Type | Constraints |
|---|---|---|
| tenant_id | uuid | PK part |
| id | uuid | PK part (unit or venture ID) |
| type | text | NOT NULL, CHECK in ('business_unit','venture') |
| name | text | NOT NULL |
| closed_at | timestamptz | NULL (closing revokes scoped assignments) |
| source_version | bigint | NOT NULL (apply only when newer) |
| source_event_id | uuid | NOT NULL (Architecture 8 projection rule) |
| updated_at | timestamptz | NOT NULL |

**iam.support_access_grant**

| Column | Type | Constraints |
|---|---|---|
| id | uuid | PK |
| tenant_id | uuid | NOT NULL FK tenant_policy |
| type | text | NOT NULL, CHECK in ('standard','emergency') |
| mode | text | NOT NULL, CHECK in ('read_only','impersonate'); CHECK `type='standard' OR mode='read_only'` |
| support_profile_id | uuid | NOT NULL FK profile (trigger: in the platform tenant) |
| target_profile_id | uuid | NULL; FK `(target_profile_id, tenant_id)` → profile; CHECK `(mode='impersonate') = (target_profile_id IS NOT NULL)` |
| approved_by_profile_id | uuid | NOT NULL FK profile (standard: same tenant; emergency: another PLATFORM_OWNER) (trigger) |
| reason | text | NOT NULL |
| ticket_ref | text | NOT NULL |
| starts_at | timestamptz | NOT NULL |
| expires_at | timestamptz | NOT NULL; CHECK `expires_at > starts_at AND expires_at <= starts_at + CASE type WHEN 'emergency' THEN interval '1 hour' ELSE interval '4 hours' END` |
| revoked_at | timestamptz | NULL |
| created_at | timestamptz | NOT NULL |

**iam.audit_event**: append-only, partitioned monthly.

| Column | Type | Constraints |
|---|---|---|
| occurred_at | timestamptz | PK part; partition key |
| id | uuid | PK part |
| tenant_id | uuid | NULL (platform-wide events) |
| actor_type | text | NOT NULL, CHECK in ('user','service','system','support') |
| actor_user_id | uuid | NULL |
| on_behalf_of_user_id | uuid | NULL (impersonation target) |
| session_id | uuid | NULL (IP and device via auth_session) |
| action | text | NOT NULL (`role_assignment.created`) |
| subject_type | text | NOT NULL |
| subject_id | uuid | NULL |
| payload | jsonb | NOT NULL DEFAULT '{}' (IDs, before/after, reason; no personal data) |
| correlation_id | uuid | NULL |

Indexes: `(tenant_id, occurred_at)` and `(subject_type, subject_id)`. Tamper evidence: daily signed checkpoint (hash of the day's rows by tenant) to WORM storage, not a per-row chain (F5). If a per-row chain is required, add `prev_hash bytea` and `hash bytea` and serialise under `pg_advisory_xact_lock(tenant)`.

### c.4 Outside IAM and Tenancy
- **integration.provider_account** (Integrations, landlord): id, tenant_id, provider, external_account_id (unique with provider; used for webhook tenant routing), vault_ref, key_id, rotated_at, rotation_due_at, status, created_at, updated_at.
- **integration.inbound_webhook**: (provider, event_id) PK, tenant_id, received_at, processed_at, pruned after 30 days.
- **Payments (tenant DB)**: on the release record, `signed_payload_hash`, `signature`, `signature_method`, `authenticator_id` (replaces iam.transaction_authorisation). The nonce is in Redis.
- **Code catalogues**: PermissionProvider (existing); SystemRoleCatalogue (system roles, default assignment rules, default SoD conflicts); PlanCatalogue (features, plans, limits); SessionPolicy (per client type and role class).
- **Deferred**: `iam.identity_provider` (per-tenant IdP), `iam.permission_conflict` (tenant-added stricter SoD rules).

### c.5 Rename map v2 → v3

| v2 | v3 |
|---|---|
| tenancy.tenant.kind | tenant.type |
| tenant.short_name | tenant.display_name |
| tenant.state | tenant.registration_state |
| tenant.primary_colour | tenant.brand_color |
| tenant.currency | tenant.currency_code |
| tenant_database.db_host / db_port / db_name / db_role | host / port / database_name / login_role |
| tenant_job.kind | tenant_job.type |
| tenant_job.error | tenant_job.last_error |
| tenant_job.from_version, to_version | tenant_job.details |
| tenant_entitlement_override | entitlement_override |
| subscription.plan_id | subscription.plan_code + plan_version |
| iam.app_user | iam.user_account (entity User) |
| app_user.kind | user_account.type |
| app_user.failed_login_count | user_account.failed_sign_in_count |
| user_profile | profile |
| user_profile_id (all tables) | profile_id |
| user_profile.first_name, last_name | user_account.given_name, family_name |
| user_authenticator | authenticator |
| user_authenticator.public_data | authenticator.credential_data |
| backup_code (table) | authenticator (type backup_codes) credential_data |
| user_device (table) | authenticator (type device_key) |
| auth_session.device_id | auth_session.authenticator_id |
| auth_session.client_id | auth_session.client_type |
| auth_session.auth_time / amr / acr | authenticated_at / auth_methods / assurance_level |
| auth_session.session_started_at | auth_session.created_at |
| refresh_token.token / username / valid / family / family_valid / replaced_by | token_hash / (via session) / expires_at / session_id / (session.revoked_at) / dropped |
| external_identity.provider | external_identity.issuer |
| external_identity.linked_at | external_identity.created_at |
| api_key.lookup_hash | api_key.key_hash |
| one_time_token | removed (signed URLs) + iam.invitation |
| permission.slug | permission.code |
| role_permission.permission_slug | role_permission.permission_code |
| permission.risk 'money' | permission.risk 'financial' |
| role.scope 'PLATFORM', 'TENANT', … | 'platform', 'tenant', 'business_unit', 'venture', 'self' |
| user_role | role_assignment |
| user_role.scope_ref_id | role_assignment.scope_id |
| user_role.granted_at / effective_at | role_assignment.created_at / starts_at |
| user_role_assign_role | role_assignment_rule |
| scope_reference | access_scope |
| scope_reference.scope_ref_id / kind / status | access_scope.id / type / closed_at |
| platform_tenant | tenant_policy.is_platform |
| tenancy.tenant.signup_mode / signup_email_domains | iam.tenant_policy.join_mode / join_email_domains |
| user_profile.access_version | tenant_policy.access_version |
| support_access_grant.kind | support_access_grant.type |
| support_access_grant.granted_by_profile_id + approved_by_profile_id | approved_by_profile_id |
| audit_event.actor_kind | audit_event.actor_type |
| audit_event.ip, user_agent | audit_event.session_id |
| Permission slugs `user-roles:*`, `user-role-assign-roles:manage` | `role-assignments:*`, `role-assignment-rules:manage`, `role-assignments:assign-any` |
| Endpoint /api/user-role-assign-roles | /api/role-assignment-rules |
| Endpoint /api/profiles/{id}/roles/{userRoleId} | /api/profiles/{id}/role-assignments/{id} |
| Event UserRoleGrantedV1 / UserRoleRevokedV1 | RoleAssignedV1 / RoleAssignmentRevokedV1 |
| Event TenantProvisionedV1 | TenantActivatedV1 |

### c.6 Removed or merged, with reasons

| Removed | Reason |
|---|---|
| tenancy.feature, plan, plan_feature | Change only with releases; PlanCatalogue in code (F30) |
| tenant.signup_mode, signup_email_domains | Moved to IAM tenant_policy (F20) |
| tenant.require_email_verification | Always true (F11) |
| tenant.region | Constant 'ng'; enforced by deployment (27.8) |
| tenant.activated_at | subscription.starts_at / audit |
| tenant.created_by_profile_id | Audit actor; cross-module link |
| tenant_database.provisioned_at | tenant_job (provision) finished_at |
| iam.platform_tenant | tenant_policy.is_platform (F20) |
| iam.backup_code | authenticator.credential_data |
| iam.user_device | Duplicate of the device_key authenticator (F8) |
| iam.one_time_token | Signed URLs + invitation (F29) |
| iam.transaction_authorisation | Payments module, tenant DB, plus a Redis nonce (F16) |
| iam.permission_conflict | Defaults in code; tenant rules deferred (F19) |
| app_user.trusted_device_version | Superseded (27.3) |
| app_user.last_login_at | max(auth_session.created_at) |
| app_user.email_verified_at | Invariant: accounts only after proof (F11) |
| user_profile.source, created_by_profile_id | Audit and invitation (F22) |
| user_profile.metadata | Personal data out of IAM (F21) |
| user_profile.access_version | Tenant-level counter (F18) |
| user_authenticator.backup_codes rows | Merged |
| external_identity.email_verified, unlinked_at | Checked at link time; hard delete (F28) |
| refresh_token gesdinet columns | Duplicate session facts (F9) |
| api_key.tenant_id, environment, scopes | Derived or duplicate authority (F17) |
| permission.resource, action | Derivable from code |
| role.can_assign_all | Permission `role-assignments:assign-any` (F19) |
| role.based_on_role_id | Provenance in audit; cosmetic |
| role_permission.created_at | Audit |
| user_role_assign_role default rows | Code (F19) |
| scope_reference.status | closed_at |
| audit_event.actor_profile_id | Derivable from (actor_user_id, tenant_id) |
| audit_event.ip, user_agent | auth_session; IP truncation (F5) |
| Redis jti blocklist | Session revocation covers it (F9) |

Result: **tenancy 6, iam 17 = 23 landlord tables** (v2: 30). Integrations: 2 (own module). No section 27 control is lost: MFA challenge, session assurance, DPoP, BFF, dual control, transaction signing, delayed financial roles, KYB gate, four-eyes bootstrap, support token exchange, webhook routing, audit tamper evidence and retention all remain, with fewer places to keep in sync.

---

## (d) Proposed v3 flows

### d.1 Tenant provisioning and bootstrap
1. A platform admin calls `POST /api/platform/tenants` (Tenancy, `_tenant=platform`, Idempotency-Key; the unique slug is the natural key). This creates `tenant` (status `provisioning`, kyb_status `pending`), `subscription` and `tenant_job(provision, queued)`, and publishes **TenantCreatedV1** (id, slug, type) in the same transaction.
2. IAM consumes TenantCreatedV1 and inserts `tenant_policy` (join_mode `closed`).
3. The provisioning worker runs the steps, each idempotent:
   - `create_database`: create the role and database from the template, using deterministic names checked against pg_roles and pg_database;
   - `migrate`: catch-up migration;
   - `seed`: reference data only, `ON CONFLICT DO NOTHING`;
   - `activate`: guarded `UPDATE … WHERE status='provisioning'`, then **TenantActivatedV1**.

   Each step records `tenant_job.step`. Retries use backoff. After repeated failure, compensation drops the database and role only if the tenant was never active, then sets status `failed`. `POST /api/platform/tenants/{id}/retry` starts a new job.
4. Bootstrap administrators: the console calls `POST /api/platform/tenants/{id}/invitations` (IAM) with two emails, creating `invitation(type bootstrap, role TENANT_ADMIN)`. A Compliance Officer, who must be a different profile, approves with `POST …/invitations/{id}/approve` after checking the signatories. Emails are sent only after approval and after TenantActivatedV1.
5. On acceptance, the user signs in or signs up with the same email. The system path then creates the profile and the TENANT_ADMIN assignment. Assignments of financial or high roles start 24 h later.
6. KYB: `POST /api/platform/tenants/{id}/kyb` (Compliance) sets `kyb_status`. PlanFeatureAssertion keeps financial features off until the tenant is `verified`.
7. Offboarding: `archived` (read-only) → `deleting` (after retention, with no legal hold; job `delete`: export, `DROP DATABASE`, delete the tenant key) → `deleted`. IAM, on TenantStatusChangedV1(archived), revokes API keys and support grants. Profiles stay readable for AUDITOR and TENANT_ADMIN.
8. Platform tenant: a one-off console command creates the platform tenant and two bootstrap PLATFORM_OWNER invitations. It refuses if any owner exists.

### d.2 Sign-up, join and invitations
- **Self sign-up** (`join_mode` open or email_domain): `POST /api/auth/register {email}` always returns 202 and sends a signed link (email, tenant, 30 min). Opening it lets the user set name and password, which creates `user_account` and `profile` (and SELF_SERVICE, unless that is implicit). For an existing email, the link instead offers "sign in to join".
- **Existing user joins**: `POST /api/me/profiles` in the resolved tenant. Allowed when join_mode is open, or email_domain with a matching domain, or with a valid invitation. The outcome depends on the existing profile:
  - none → create;
  - `removed` → reactivate with old assignments still revoked;
  - `disabled` → refuse with a neutral response;
  - `active` → no-op.
- **Invitation** (administrator `POST /api/invitations {email, role_id?, scope_id?}`, through the assignment gates): always returns 202. Acceptance requires the account that owns the email, by sign-in or by sign-up through the link. Invitations can be listed, revoked and resent (rate-limited).
- **Social sign-up**: provider-verified email, then the same RegisterUserInTenant path. An existing email is never linked automatically.

### d.3 Sign-in and society switch
- `POST /api/auth` (BFF or mobile) checks the password. If MFA is required by the tenant policy, by any role the user holds, or by risk, it returns `mfa_challenge` (5 min, 5 attempts). After the factor, it creates `auth_session` (assurance_level, auth_methods) and `refresh_token`, and issues a JWT (sub, sid, ver, acr, amr, auth_time, cnf).
- With a resolved tenant, ProfileLoginValidator requires an active profile there and gives a generic error otherwise. Without one, the client calls `GET /api/me/profiles` to show the society switcher.
- **Switch**: the client sends a different `X-Tenant` on the same app host. No new session is needed. On each request the pipeline runs: tenant → JWT → profile active → SessionPolicyCheck (idle and age limits and assurance for this profile's roles; on failure 401 `insufficient_user_authentication`) → effective access, cached by tenant `access_version`. Custom domains obtain a session through the one-time handoff code.
- **Refresh**: look up the hash, then check that the session is not revoked and within its absolute limit, check the DPoP proof, mark `used_at`, and insert the next token. Reuse after 30 s revokes the session, adds the `sid` to the deny-list, and alerts the user.

### d.4 User and profile management

| Action | Who | Effect |
|---|---|---|
| Invite / add | Tenant admin (`profiles:create`) | Invitation (d.2) |
| Disable / enable profile | Tenant admin | profile.status; tenant access_version + 1; two-admin check |
| Remove profile / user leaves | Tenant admin / the user | status `removed`; assignments revoked; service keys revoked; two-admin check |
| Rejoin | User | Reactivate (d.2) |
| Deactivate / reactivate account | Account holder or platform staff (`user-accounts:manage`) | user_account.status; token_version + 1; sessions revoked; assignments kept (profiles become unusable) |
| Erasure request | Account holder | status `deactivated`, `erasure_requested_at`; overwrite personal columns once every profile has been ended longer than the retention period and no legal hold applies |
| Service account | Tenant admin (`service-accounts:manage`, step-up) | user_account(type service) + its single profile + api_key; roles through the gates |
| Platform staff | PLATFORM_ADMIN | Invitation into the platform tenant; passkey required |
| Dormancy | Daily job | Profiles with `last_seen_at` older than 90 days → `disabled` (reason dormant), skipped if it would breach the two-admin rule |

### d.5 Role assignment and revocation
- **Assign** (`POST /api/profiles/{id}/role-assignments`). The gates, in order:
  1. Not the caller's own profile.
  2. Authority, evaluated in this order: tenant deny rule, then `role-assignments:assign-any` at tenant scope, then tenant allow rule, then default rule in code.
  3. Scope containment.
  4. Target active, role valid, expiry within `max_grant_duration`.
  5. SoD on effective permissions, with defaults from code.

  Financial or high roles get `starts_at = now + 24 h`, all administrators are notified, and any of them may revoke before the start. Then access_version + 1 and an audit record.
- **Revoke** runs gates 1–3, then sets `revoked_at`, bumps access_version and writes an audit record. The two-admin rule runs under `pg_advisory_xact_lock(tenant)`.
- **Expiry job** (every minute): revokes expired assignments and bumps access_version.
- **System path**: only bootstrap TENANT_ADMIN or PLATFORM_OWNER on invitation acceptance, and SELF_SERVICE if it is stored. It runs gates 4 and 5.

### d.6 Support access
- **Standard**: a society TENANT_ADMIN creates a `support_access_grant` with type standard, read_only or impersonate (with a target), at most 4 h, and a ticket.
- **Emergency**: a PLATFORM_OWNER requests it and a second PLATFORM_OWNER approves. It is read_only, lasts at most 1 h, and society administrators are notified.
- **Use**: a platform profile calls `POST /api/support/sessions` (RFC 8693 token exchange) and receives a 15-minute token with `sub` = target (impersonate) or the staff member (read_only), `act.sub` = staff, and no refresh token. The grant is checked live on every request, with no cache. Administrators and holders of financial roles cannot be impersonated. Impersonated sessions cannot perform financial actions, access changes or exports. Audit records `actor_type support`, `actor_user_id` (staff) and `on_behalf_of_user_id`.

---

## (e) Proposed v3 document outline (one tab, ~40% shorter)

1. **Scope and planes**: one table of modules, schemas and planes, including Integrations. Out of scope.
2. **Decisions**: one table, final values only. No "superseded by" notes.
3. **Glossary and naming conventions**: user account, profile (= membership in the IAM sense), role assignment, assignment rule, access scope, tenant policy, invitation, session. Naming rules from c.1.
4. **Data model**: ER diagram plus one table per entity with type, constraints and a short "why" column. This replaces sections 6–8 and 25. Code catalogues are listed separately.
5. **Tenant lifecycle**: state machine (status, kyb_status), provisioning steps, compensation, archive and delete with retention, platform bootstrap. The Multi-tenancy tab keeps only the database mechanics.
6. **Tenant resolution**: TenantRequirement (required, optional, none, platform, webhook), sources, errors, worker context.
7. **Identity**: account invariants, authenticators, external identities, invitations, sign-up and join, erasure.
8. **Sessions and tokens**: auth_session, refresh rotation, JWT claims and validation, BFF and DPoP, per-request session policy, revocation (two mechanisms). The Authentication tab keeps only package configuration.
9. **Access control**: permission catalogue, roles and scopes, system roles, assignment gates, assignment rules, SoD, delayed financial roles, two-admin rule, effective-access cache.
10. **Request pipeline and decisions**: pipeline, voter, assertions table, error codes.
11. **Support access and break-glass.**
12. **Financial controls**: dual control, transaction signing (Payments-owned evidence), KYB and KYC gates, access reviews and dormancy.
13. **Integrations boundary**: API keys, webhooks routing, provider accounts (pointer to the Integrations tab).
14. **Audit, retention and data protection**: audit_event, checkpoints, retention matrix (audit, AML, IP, sessions, backups), NDPA and CBN obligations, residency.
15. **API endpoints and events**: two tables.
16. **Governance tests and go-live checklist**: one merged list.
17. **Change history**: v1, v2, v3 (one line each, linking to the review notes).

Rule for keeping it lean: each fact appears in exactly one section. Other sections link to it and do not restate it. Security review findings stay in the research notes. The tab records only the resulting design.

---

## Sources
- WorkOS organization membership: https://workos.com/docs/reference/authkit/organization-membership · invitation: https://workos.com/docs/reference/authkit/invitation
- Clerk organization invitation: https://clerk.com/docs/reference/backend/types/backend-organization-invitation · membership: https://clerk.com/docs/reference/backend/types/backend-organization-membership
- Keycloak organizations (managed and unmanaged members, invitations, identity-first login, multiple organizations): https://docs.redhat.com/ja/documentation/red_hat_build_of_keycloak/26.2/html/server_administration_guide/managing_organizations
- Auth0 Organizations: https://auth0.com/docs/manage-users/organizations/organizations-overview · invitations: https://auth0.com/docs/manage-users/organizations/configure-organizations/invite-members
- ZITADEL role assignments (formerly user grants): https://zitadel.com/docs/guides/manage/console/roles
- Azure tenant lifecycle: https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/considerations/tenant-life-cycle · tenancy models and tenant-to-deployment mapping: https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/considerations/tenancy-models
- AWS control plane vs application plane: https://docs.aws.amazon.com/whitepapers/latest/saas-architecture-fundamentals/control-plane-vs.-application-plane.html · onboarding (tenant management vs user management services): https://aws.amazon.com/blogs/apn/tenant-onboarding-best-practices-in-saas-with-the-aws-well-architected-saas-lens
- Tenancy for Laravel (event pipelines CreateDatabase/MigrateDatabase/SeedDatabase; data column): https://tenancyforlaravel.com/docs/v3/event-system · https://tenancyforlaravel.com/docs/v3/tenants
- Spatie permission table names: https://github.com/spatie/laravel-permission/blob/main/config/permission.php
- NIST RBAC (UA, PA, sessions, SSD/DSD): https://csrc.nist.gov/projects/role-based-access-control/faqs · ARBAC97 (can_assign): https://profsandhu.com/confrnc/rbac/r98pra97.pdf
- GitLab constraint naming: https://docs.gitlab.com/development/database/constraint_naming_convention/ · SQL Style Guide: https://www.sqlstyle.guide/ · PostgreSQL keywords: https://www.postgresql.org/docs/current/sql-keywords-appendix.html · partial indexes: https://www.postgresql.org/docs/current/indexes-partial.html
- RFC 9700 (refresh rotation): https://www.rfc-editor.org/rfc/rfc9700 · Symfony login links (signature_properties): https://symfony.com/doc/current/security/login_link.html · NIST SP 800-63B-4: https://pages.nist.gov/800-63-4/sp800-63b.html
- Earlier notes in this folder: `rbac_tenancy_schema_practice.md`, `iam_tab_architect_review.md`, `iam_v2_security_review_oauth_jwt.md`, `iam_v2_security_review_banking.md`.
