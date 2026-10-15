# IAM and Tenancy data model: DBA review of v3.1 and lean v4 proposal

Date: 2026-10-13. Reviewer stance: senior data architect / DBA (PostgreSQL 17-18, Doctrine ORM 3 / DBAL 4, Symfony 8.1).
Input: `iam_v3_1_snapshot.md` (20 tables: 4 tenancy, 16 iam). Context checked: `tab_multitenancy.md`, `tab_authentication.md`, `tab_architecture_guide.md`, `iam_v3_architect_review.md`, `rbac_tenancy_schema_practice.md`, `tenancy/*.md`.
Legend: **[V]** verified in the cited primary source this session; **[P]** practitioner judgement or unverified recollection (check before relying on it).

---

## 0. Summary

1. The model is mostly sound and already lean. The biggest problems are not bulk but **constraints that cannot work as written**:
   - composite tenant FKs to `role` break for system roles (`role.tenant_id IS NULL`);
   - "one active row per (profile, role, scope)" silently allows duplicates when `scope_id` is NULL, and "active" depends on `now()`, which an index predicate cannot use;
   - `tenant_policy.status` has no `provisioning` value, yet section 6.2 inserts `status = provisioning` (CHECK violation on the very first event);
   - Doctrine cannot map an association whose join columns point at a non-primary key, so `profile(id, tenant_id)` composite FKs can only exist in SQL, not in the mapping.
2. **Messaging tables are missing.** The spec writes to "the outbox" and consumes events into `tenant_policy` and `access_scope`, but none of the 20 tables is an outbox or inbox. The Architecture guide requires an inbox with PK `(consumer, event_id)`.
3. **Duplicated facts:** `tenant.owner_email` vs `invitation.email` (plus a cross-module write-back loop), `tenant.status = failed` vs `tenant_job.status = failed`, `tenant_database.schema_status = migrating` vs a running migrate job, `role.scope` vs `access_scope.type` naming, `profile.metadata` (business data in the landlord).
4. **Race conditions** that need explicit locking: last-administrator guard (write skew), refresh-token rotation inside the 30 s grace (forks the token chain), stale `running` tenant job (permanent lock), backup-code consumption, sign-up double submit.
5. **Doctrine 3 / DBAL 4 type gaps:** `interval`, `cidr[]`, `text[]`, `inet` have no built-in DBAL type that maps natively; `NULLS NOT DISTINCT` and constraint triggers are raw-SQL only; the partitioned `audit_event` must be kept out of schema diff.
6. Lean v4: **23 landlord tables** (tenancy 4 + outbox = 5; iam 15 + outbox + inbox = 17; audit 1). The 20 business tables stay at 20; the 3 new ones are messaging plumbing that v3.1 already relied on without listing. A rollout table is avoided by adding `wave` to `tenant_job`.

---

## 1. Findings

Severity: H = data corruption, security or runtime failure; M = inconsistency or maintenance cost; L = naming or tidiness.

### F1 (H). Composite FK to `role` cannot work for system roles
- **Issue.** Section 4.3 says `role_assignment.tenant_id` is repeated "so composite keys keep profile, role and scope in one tenant". A composite FK `(role_id, tenant_id) -> role(id, tenant_id)` needs a matching row. System roles have `role.tenant_id IS NULL`, the referencing row has a non-null `tenant_id`, so under MATCH SIMPLE the row must match and cannot. [V: PG CREATE TABLE, MATCH SIMPLE "if any of them are null, the row is not required to have a match"; the referencing side here has no nulls.] The same applies to `role_assignment_rule.assigner_role_id / assignable_role_id` and `invitation.role_id`.
- **Risk.** Either the FK is dropped silently (then a tenant can assign another tenant's custom role), or every system-role assignment fails at insert.
- **Fix.** Plain FK `role_id -> role(id)` everywhere, plus one constraint trigger `role_tenant_check` (shared function) on `role_assignment`, `role_assignment_rule`, `invitation`: `role.tenant_id IS NULL OR role.tenant_id = NEW.tenant_id`. `role.tenant_id` is immutable, so no trigger is needed on `role`. Fully declarative alternative (sentinel `owner_key` column = coalesce(tenant_id, nil-uuid) + CHECK on the child) works but adds a column to four tables; not worth it.

### F2 (H). "One active assignment per (profile, role, scope)" is unenforceable as written
- **Issue.** A plain UNIQUE treats NULLs as distinct, so tenant-wide grants (`scope_id IS NULL`) can be duplicated. [V: PG "null values are not considered equal, unless NULLS NOT DISTINCT is specified"]. "Active" also depends on `starts_at`/`expires_at` vs `now()`; index predicates must be immutable [V: use-the-index-luke, deterministic functions only].
- **Fix.** `CREATE UNIQUE INDEX role_assignment_active_key ON iam.role_assignment (profile_id, role_id, scope_id) NULLS NOT DISTINCT WHERE revoked_at IS NULL;` Expiry is materialised: a sweeper sets `revoked_at` (and audit reason `expired`) when `expires_at < now()`, and a re-grant revokes the expired row in the same transaction. Drop `starts_at` (no requirement for future-dated grants; it also breaks the "active" definition).

### F3 (H). `tenant_policy.status` CHECK rejects the first event
- **Issue.** 4.2 lists `status (active, suspended, archived)`; 6.2 inserts `status = provisioning`.
- **Fix.** `tenant_policy.status IN ('provisioning','active','suspended','archived')`, same set as `tenant.status` (after F5). Add `source_version bigint` and apply an event only when `event.version > source_version`, because Messenger does not guarantee order across retries (Suspended may overtake Activated).

### F4 (H). Messaging tables are missing from the model
- **Issue.** 5.1 "writes TenantCreatedV1 to the outbox"; 6.2 and 4.3 consume events; Architecture guide 7.1/7.2 requires an outbox per module and an inbox `(consumer, event_id)`. None of the 20 tables is either.
- **Risk.** Without an inbox, a redelivered `ProfileCreatedV1` or `BusinessUnitClosedV1` is applied twice; without a stated outbox, events are dispatched outside the transaction.
- **Fix.** Add `tenancy.outbox`, `iam.outbox`, `iam.inbox`. Tenancy needs no inbox once F6 removes its only consumer. (On Symfony 8.2+ the native Messenger outbox table replaces the two outbox tables; same rules.)

### F5 (M). `tenant.status = failed` duplicates `tenant_job.status = failed`
- **Fix.** Remove `failed` from `tenant.status`. A tenant stays `provisioning` until activated; the failure, attempt and error live on the provision job. Retry restarts the job. Lifecycle becomes `provisioning -> active <-> suspended -> archived` (deleting/deleted later). Emit `TenantProvisioningFailedV1` from the job.

### F6 (M). `tenant.owner_email` duplicates `invitation.email` and creates a write-back loop
- **Issue.** The owner's email is stored in Tenancy, copied into an IAM invitation, then Tenancy consumes `ProfileCreatedV1` to clear it. Personal data in two modules, plus a bidirectional event dependency.
- **Fix.** Drop `tenant.owner_email`. `TenantCreatedV1` carries `ownerEmail`. IAM creates the bootstrap invitation immediately with `sent_at NULL, expires_at NULL`; on `TenantActivatedV1` it sets `sent_at = now()`, `expires_at = now() + 7 days` and publishes `InvitationSentV1`. Tenancy then consumes nothing from IAM.

### F7 (H). Doctrine cannot map composite FKs to `(id, tenant_id)`
- **Issue.** "It is not possible to use join columns pointing to non-primary keys" [V: Doctrine ORM limitations page]. `profile(id, tenant_id)` and `access_scope` composite references are SQL-only.
- **Fix.** In mappings, use `ManyToOne` on the single id (or keep plain uuid ids, per architecture guide), and map `tenant_id` as a scalar column. Declare composite UNIQUE and FK constraints in hand-written migrations. Add a `postGenerateSchema` listener that adds the same constraints to the generated Schema so `doctrine:migrations:diff` and `schema:validate` do not try to drop them [P]. Never map associations across entity managers; landlord and tenant reference each other by uuid only (already the rule).

### F8 (H). Role scope vs assignment scope is not checked
- **Issue.** Nothing ensures `scope_id IS NULL` exactly when the role's scope is `platform/tenant/self`, nor that `access_scope.type` equals the role's scope type, nor that `platform` roles are assigned only in the platform tenant.
- **Fix.** Constraint trigger `role_assignment_scope_check` (and the same on `invitation`). CHECK on role: `scope_type <> 'platform' OR tenant_id IS NULL` (custom roles cannot be platform roles). Composite FK `(tenant_id, scope_id) -> access_scope(tenant_id, id)` works with NULL scope under MATCH SIMPLE [V].

### F9 (H). Last-administrator guard is a write-skew race
- **Issue.** Two administrators revoking each other under READ COMMITTED each see one remaining admin and both commit; locks on different rows do not conflict [V: PG SSI wiki / Cybertec on SELECT FOR UPDATE].
- **Fix.** Every operation that can reduce the society_admin count (revoke assignment, disable/remove profile, account deactivation, expiry sweeper) first takes `SELECT ... FROM iam.tenant_policy WHERE tenant_id = :t FOR UPDATE`, then counts. This row is already bumped (`access_version`) by the same operations, so the lock is free. Account deactivation spans tenants: lock tenant_policy rows ordered by tenant_id to avoid deadlocks. society_admin role gets `max_grant_days NULL` (never expires), so the sweeper cannot remove the last admin.

### F10 (H). Refresh rotation within the 30 s grace forks the session
- **Issue.** Two concurrent refreshes with the same token: both read `used_at IS NULL`, both mint children. With only a hash stored, the grace path cannot return the existing successor. RFC 9700 4.14.2 has no grace period at all [V].
- **Fix.** Mark used atomically: `UPDATE refresh_token SET used_at = now() WHERE token_hash = :h AND used_at IS NULL RETURNING session_id`. Zero rows and `used_at` within 30 s -> answer 409 "retry with your newest token" without minting and without revoking; older than 30 s -> revoke session. Add index `refresh_token_session_id_idx` (FK cascade and sign-out) and `refresh_token_expires_at_idx` (purge). Hash as PK is fine (256-bit random; SHA-256 without salt is correct).

### F11 (H). Stale job keeps the per-tenant lock forever
- **Issue.** Partial unique index on `tenant_job(tenant_id) WHERE status IN ('queued','running')` is a good lock, but a killed worker leaves `running` forever.
- **Fix.** Add `heartbeat_at`; a reaper fails jobs with `heartbeat_at < now() - 5 min`. Fleet rollout inserts with `INSERT ... ON CONFLICT (tenant_id) WHERE status IN ('queued','running') DO NOTHING` (index inference with predicate [V: PG INSERT]). Retry reuses the row (`attempt + 1`, `status = queued`). Add `wave smallint` so `/platform/rollouts` is derived from jobs grouped by `rollout_id` (no rollout table).

### F12 (M). `tenant_database` mixes placement with derived migration state
- **Issues.** (a) `schema_status = migrating/failed` is the state of the migrate job; (b) `schema_version` mirrors `doctrine_migration_versions` (acceptable as a reporting cache, but write it only from the migrate step); (c) `host` points at PgBouncer, but provisioning, migrations, `SET lock_timeout` and advisory locks need a direct, session-mode connection (tenancy/provisioning_and_migrations.md; [V: PgBouncer "clients must not use any session-based features" in transaction mode]).
- **Fix.** Replace `host, port` with `cluster_code` (config maps each cluster to pooler and direct endpoints and to a region). Keep `schema_version` + `migrated_at`; drop `schema_status`, derive "failed" from the last migrate job and "drift" from the nightly check writing a job or an audit event. Moving a tenant becomes one column change.

### F13 (M). `access_version` granularity and catalogue changes
- **Issue.** A system-role change from `iam:sync-catalogue` must invalidate every tenant: N-row update of `tenant_policy`. Per-tenant version also invalidates every profile's cache on any change (acceptable).
- **Fix.** Cache key = `(profile_id, tenant_policy.access_version, catalogue_version)`, `catalogue_version` = hash of the deployed catalogue (code constant). No landlord-wide update on deploy.

### F14 (M). Tenant registry constraints missing
- **Fixes.** One platform row: `CREATE UNIQUE INDEX tenant_platform_key ON tenancy.tenant ((true)) WHERE type = 'platform'`. Society registration: `CHECK (type = 'platform' OR (registration_number IS NOT NULL AND registering_authority IS NOT NULL AND registration_state IS NOT NULL))` and `UNIQUE (registering_authority, registration_number)` (stops the same society being created twice). Slug: CHECK regex as specified; plain `UNIQUE (slug)` (not "while not archived": archived societies are still resolvable for exports, and partial uniqueness would make an archived slug ambiguous; slug reuse belongs to the deferred deletion step). Add `version int` (event ordering, optimistic locking with Doctrine `#[Version]`).

### F15 (M). `tenant_domain` host PK allows squatting
- **Issue.** An unverified host claimed by tenant A blocks the real owner B.
- **Fix.** PK `id`; `UNIQUE (host) WHERE verified_at IS NOT NULL`; `UNIQUE (tenant_id, host)`; `UNIQUE (tenant_id) WHERE is_primary`; `CHECK (NOT is_primary OR verified_at IS NOT NULL)`; `CHECK (host = lower(host))`. Purge unverified rows after 7 days.

### F16 (M). Invitation integrity
- **Issues.** No FKs listed for `role_id`, `scope_id`, `invited_by_profile_id`; "open" depends on `expires_at` (time) so the partial unique index cannot include it; nothing ties `type` to `invited_by_profile_id`.
- **Fix.** `UNIQUE (tenant_id, email) WHERE accepted_at IS NULL AND revoked_at IS NULL`; re-inviting revokes an expired open row in the same transaction. Composite FK `(invited_by_profile_id, tenant_id) -> profile(id, tenant_id)`; FK `accepted_user_id`; triggers from F1/F8. CHECKs: `(type = 'bootstrap') = (invited_by_profile_id IS NULL)`; `(accepted_at IS NULL) = (accepted_user_id IS NULL)`; `NOT (accepted_at IS NOT NULL AND revoked_at IS NOT NULL)`; `email = lower(email)`. On acceptance re-check role not archived and scope not closed.

### F17 (M). Service-account invariants need triggers
- **Issue.** "api_key belongs to a service account's profile" and "a service account has exactly one profile" cross `user_account.type`, which no FK or partial index can see.
- **Fix.** Constraint triggers `api_key_service_profile_check` and `profile_service_account_check`; `user_account.type` immutable (trigger or app rule). Resolve the Authentication tab contradiction (it still has `api_key(user_profile_id, tenant_id)`): tenant comes from the profile; no `tenant_id` on api_key. Lookup by `key_hash` (unique); `key_prefix` is display only, not unique (short random prefixes collide).

### F18 (M). `user_account` CHECKs and uniqueness
- **Fixes.** `CHECK (type = 'service' OR email IS NOT NULL)`, `CHECK (type = 'person' OR (password_hash IS NULL AND email IS NULL))`, `CHECK (email = lower(email))` (or a nondeterministic ICU collation; PG docs prefer it over citext [P, see rbac practice note]). `UNIQUE (email)`, `UNIQUE (phone) WHERE phone_verified_at IS NOT NULL`. Add `status_reason` (naming rule says lifecycle status + reason + changed_at). Sign-up double submit: `INSERT ... ON CONFLICT (email) DO NOTHING` then select; profile `ON CONFLICT (user_id, tenant_id)`.

### F19 (M). Profile reactivation lets a removed person rejoin
- **Issue.** 9.1: sign-up reactivates a `removed` profile. If an administrator removed the person, an `open` join mode lets them straight back in.
- **Fix.** Split: `left` (by the user; self-rejoin allowed) vs `removed` (by an administrator; rejoin only by invitation). Status set `active, disabled, left, removed`.

### F20 (M). `profile.metadata` is business data in the control plane
- **Issue.** "Whatever a society collects" is personal, per-society, schema-less data in the landlord, against "each society's business data lives in its own database" (and prior review F21).
- **Fix.** Drop it; the Membership module in the tenant database owns it, keyed by `profile_id`.

### F21 (M). Authenticator constraints and backup-code race
- **Fixes.** `UNIQUE (user_id) WHERE type = 'backup_codes' AND revoked_at IS NULL`; `UNIQUE (credential_id)` (NULLs distinct, fine); per-type CHECKs (totp needs `secret_encrypted, key_id`; passkey needs `credential_id, credential_data`). Consuming a backup code or updating the passkey sign count must `SELECT ... FOR UPDATE` the row, else two concurrent sign-ins both use the same code.

### F22 (M). Audit table placement and Doctrine
- **Issues.** (a) `audit_event` sits in `iam` but Tenancy writes it too (cross-module write). (b) PK `(occurred_at, id)` is required because a PK on a partitioned table must include the partition key [V: PG partitioning limitations]; nothing should FK to it. (c) No index for `GET /audit-events`. (d) Doctrine diff will try to manage partitions. (e) Missing next-month partition makes inserts fail.
- **Fix.** Move to schema `audit` owned by a shared Audit component (both modules use an `AuditWriter` port). Write with DBAL, not an ORM entity; exclude `audit.*` via `schema_filter`; manage partitions in SQL (pg_partman or a monthly job) plus a DEFAULT partition as a safety net; detach old partitions with `DETACH PARTITION ... CONCURRENTLY` from a separate job (not inside a transaction) [P: Crunchy pg_partman notes]. Indexes: `(tenant_id, occurred_at DESC)`, `(subject_type, subject_id, occurred_at DESC)`. Add `ip inet` (Auth tab 15 requires IP). Align `actor_type` with `user_account.type`: `person, service, system`. Audit vs outbox: not a duplicate; outbox rows are transient and deleted after relay, audit is permanent; do not keep an extra "event log" table.

### F23 (M). DBAL 4 type gaps
- `role.max_grant_duration interval`: DBAL `dateinterval` stores a string, not a native interval [P]. Use `max_grant_days int`.
- `text[]` (`join_email_domains`, `auth_methods`, `allowed_scopes`), `cidr[]` (`ip_allow_list`), `inet` (`ip`): no native DBAL 4 types; `simple_array` is comma-joined text [V: DBAL types]. Either write small in-house DBAL types (no new dependency) or use `jsonb` (`json` type with `platformOptions: {jsonb: true}` [V]). Recommendation: jsonb for `join_email_domains`, `allowed_scopes`, `auth_methods`; in-house `inet`/`cidr[]` types because CIDR matching in SQL is useful (`ip <<= ANY(...)`).
- Enums: PHP backed enums via `enumType`; CHECK constraints in migrations (DBAL does not introspect CHECKs, so no diff churn [P]).
- Partial indexes: `#[Index(options: ['where' => ...])]` [V: ORM attributes reference]. `NULLS NOT DISTINCT` and constraint triggers: raw SQL only; protect with the schema listener of F7 and a test that asserts each constraint exists.
- UUID v7: app-generated (Symfony Uid `UuidV7`, `uuid` type -> native `uuid`). PG18 `uuidv7()` exists [V] and can be the column DEFAULT later for SQL-only inserts.

### F24 (M). `access_scope` lifecycle
- PK `(tenant_id, id)` is a composite identifier; Doctrine advises against composite keys. Use PK `id` (uuid from the tenant DB, globally unique) + `UNIQUE (tenant_id, id)` for the composite FK, same pattern as `profile`. Keep `source_version`. When a scope closes, the consumer revokes its assignments and bumps `access_version` in one transaction. New tenants: backfill by a replay request to the tenant's module, idempotent through the inbox.

### F25 (M). `role_assignment_rule` keys
- Add `UNIQUE (tenant_id, assigner_role_id, assignable_role_id)`; make that the PK and drop `id`. Deny + allow is an extension of ARBAC97 `can_assign` (which is permissive only); keep deny but document "deny wins" (already 11.3).

### F26 (L). Naming
- `auth_session.client_type (member_web, member_mobile, console)` and the system role `member` reuse the word the glossary reserves for the Membership module -> `web, mobile, console`; role `society_member` or `basic`.
- `role.scope` is a classification -> `scope_type` (and `access_scope.type` stays).
- `api_key.prefix` -> `key_prefix`; `permission.risk` -> `risk_level`; `auth_session.revoke_reason` -> `revoked_reason` (pairs with `revoked_at`).
- `actor_type user` vs account type `person` (F22).
- Unique indexes created with CREATE UNIQUE INDEX are not constraints and get no default suffix: name them `<table>_<purpose>_key`.

### F27 (M). Contradictions with other tabs (fix the other tabs)
| Topic | v3.1 | Other tab |
|---|---|---|
| Missing profile in tenant | 403 (7.4) | 404 (Auth 4, 13; Arch guide 444) |
| access_version | per tenant | per profile (Auth 9, 16) |
| Account creation | only after email proven | "User (email not yet verified)" (Auth 591) |
| api_key tenant | via profile | composite (user_profile_id, tenant_id) (Auth 560) |
| Tenant registry | 4 tables, 5 states, db in tenant_database | tenant with db_* inline, 9 states, citext slug, region, plan tables (Multi-tenancy 3-4) |
| migration_wave | 0 internal, 1 canary, 2 early, 3 everyone | 0 canary, 1 small, 2 rest, 3 last, default 2 (Multi-tenancy 4, provisioning notes) |
| Names | user_account, profile, role_assignment | app_user, user_profile, user_role, one_time_token, SupportAccessGrant |
| Join link | signed URL | 72 h signed join link (Auth 595): fine, but state the TTL once |

### F28 (L). Small things
- `auth_session.last_used_at`: update only on refresh (already in 10), never per request.
- `user_account.failed_sign_in_count` is a hot row under credential stuffing; acceptable, update with a single atomic UPDATE.
- `updated_at` maintained by the app (Doctrine lifecycle callback), not triggers; drop it on insert-only tables.
- Landlord entity manager in workers: clear it per message too (stale identity map, memory), not only the tenant EM.

---

## 2. Lean v4 table list

Markers: **+** added, **-** removed, **~** renamed or changed. All ids uuid (v7, app-generated), all times timestamptz, value sets text + CHECK.

### tenancy (5)
- **tenant**: id, type (platform, society), slug, legal_name, display_name, registration_number, registering_authority, registration_state, status (provisioning, active, suspended, archived) ~ (-failed), status_reason, status_changed_at, timezone, currency_code, logo_url, brand_color, +version, created_at, updated_at. **-owner_email**.
- **tenant_domain**: +id PK, tenant_id, host ~ (no longer PK), is_primary, verification_token_hash, verified_at, created_at.
- **tenant_database**: tenant_id PK, +cluster_code ~ (replaces host, port), database_name, login_role, secret_ref, schema_version, +migrated_at, migration_wave, created_at, updated_at. **-host, -port, -schema_status**.
- **tenant_job**: id, tenant_id, type (provision, migrate), status (queued, running, succeeded, failed, cancelled), step, attempt, rollout_id, +wave, details jsonb, last_error, +heartbeat_at, created_at, started_at, finished_at.
- **+outbox**: id (event_id), aggregate_id, event_type, payload jsonb, occurred_at, published_at.
- (no tenancy inbox: Tenancy no longer consumes IAM events)

### iam (17)
- **tenant_policy**: tenant_id PK, is_platform, status ~ (+provisioning), join_mode, join_email_domains ~ jsonb, mfa_required, access_version, +source_version, created_at, updated_at.
- **user_account**: id, type, email, phone, phone_verified_at, given_name, family_name, password_hash, status, +status_reason, status_changed_at, token_version, failed_sign_in_count, locked_until, created_at, updated_at.
- **authenticator**: unchanged columns (id, user_id, type, label, credential_id, credential_data jsonb, secret_encrypted, key_id, confirmed_at, last_used_at, revoked_at, created_at).
- **external_identity**: unchanged.
- **auth_session**: id, user_id, client_type ~ (web, mobile, console), authenticated_at, auth_methods ~ jsonb, assurance_level, dpop_jkt, ip inet, user_agent, created_at, last_used_at, revoked_at, ~revoked_reason.
- **refresh_token**: token_hash PK, session_id, created_at, expires_at, used_at.
- **api_key**: id, profile_id, name, ~key_prefix (not unique), key_hash, pepper_version, ip_allow_list cidr[], expires_at, last_used_at, revoked_at, created_at.
- **invitation**: id, tenant_id, type, email, role_id, scope_id, invited_by_profile_id, token_hash, +sent_at, expires_at ~ (nullable until sent), accepted_at, accepted_user_id, revoked_at, created_at.
- **profile**: id, user_id, tenant_id, status ~ (active, disabled, left, removed), status_reason, status_changed_at, created_at, updated_at. **-metadata**.
- **permission**: code PK, module, name, description, ~risk_level, allowed_scopes ~ jsonb, deprecated_at.
- **role**: id, tenant_id (NULL = system), code, name, description, ~scope_type, ~max_grant_days int (was interval), archived_at, created_at, updated_at.
- **role_permission**: (role_id, permission_code) PK.
- **role_assignment**: id, profile_id, tenant_id, role_id, scope_id, expires_at, revoked_at, created_at. **-starts_at**.
- **role_assignment_rule**: (tenant_id, assigner_role_id, assignable_role_id) PK, effect, created_at. **-id**.
- **access_scope**: id PK ~ (was tenant_id + id), tenant_id, type, name, closed_at, source_version, updated_at.
- **+outbox**: as tenancy.outbox.
- **+inbox**: (consumer, event_id) PK, received_at.

### audit (1)
- **audit_event** ~ (moved from iam): occurred_at + id PK, monthly partitions + DEFAULT; tenant_id, actor_type ~ (person, service, system), actor_user_id, session_id, +ip inet, action, subject_type, subject_id, payload jsonb, correlation_id.

Total 23 (20 business/control tables + 3 messaging). Deferred tables unchanged.

---

## 3. Constraints, triggers and indexes

### tenancy
- `tenant_pkey (id)`; `tenant_slug_key UNIQUE (slug)`; `tenant_slug_check` regex; `tenant_platform_key UNIQUE ((true)) WHERE type='platform'`; `tenant_registration_key UNIQUE (registering_authority, registration_number)`; `tenant_society_registration_check`; `tenant_status_check`, `tenant_type_check`; `currency_code ~ '^[A-Z]{3}$'`.
- `tenant_domain_host_key UNIQUE (host) WHERE verified_at IS NOT NULL` (hot path: resolve by host); `tenant_domain_tenant_host_key UNIQUE (tenant_id, host)`; `tenant_domain_primary_key UNIQUE (tenant_id) WHERE is_primary`; `tenant_domain_primary_check`; `tenant_domain_host_check (host = lower(host))`; FK tenant_id -> tenant.
- `tenant_database_pkey (tenant_id)` FK -> tenant; `database_name_key`, `login_role_key`; `migration_wave_check (0..3)`; name check `database_name = 't_' || replace(tenant_id::text,'-','')`.
- `tenant_job_active_key UNIQUE (tenant_id) WHERE status IN ('queued','running')`; `tenant_job_tenant_created_idx (tenant_id, created_at DESC)`; `tenant_job_rollout_idx (rollout_id, wave) WHERE rollout_id IS NOT NULL`; `tenant_job_running_heartbeat_idx (heartbeat_at) WHERE status='running'`.
- `outbox_unpublished_idx (occurred_at) WHERE published_at IS NULL`.

### iam
- tenant_policy: PK; `tenant_policy_platform_key UNIQUE ((true)) WHERE is_platform`; checks for status, join_mode; `join_mode <> 'email_domain' OR jsonb_array_length(join_email_domains) > 0`.
- user_account: `email_key UNIQUE (email)`; `phone_verified_key UNIQUE (phone) WHERE phone_verified_at IS NOT NULL`; checks of F18; `phone ~ '^\+[1-9][0-9]{6,14}$'`.
- authenticator: `user_id_idx`; `credential_id_key`; `backup_codes_active_key`; per-type checks.
- external_identity: `issuer_subject_key UNIQUE (issuer, subject)`; `user_id_idx`.
- auth_session: `user_active_idx (user_id) WHERE revoked_at IS NULL` (list and revoke-all).
- refresh_token: `session_id_idx`; `expires_at_idx`; FK session_id ON DELETE CASCADE.
- api_key: `key_hash_key UNIQUE`; `profile_id_idx`; trigger `api_key_service_profile_check`; `expires_at <= created_at + interval '1 year'`.
- invitation: `token_hash_key`; `open_key UNIQUE (tenant_id, email) WHERE accepted_at IS NULL AND revoked_at IS NULL`; FK `(invited_by_profile_id, tenant_id) -> profile(id, tenant_id)`; FK `(tenant_id, scope_id) -> access_scope(tenant_id, id)`; FK role_id; triggers `role_tenant_check`, `invitation_scope_check`; checks of F16.
- profile: `user_tenant_key UNIQUE (user_id, tenant_id) INCLUDE (id, status)` (hot path: profile check); `id_tenant_key UNIQUE (id, tenant_id)` (composite FK target); `tenant_status_idx (tenant_id, status)`; trigger `profile_service_account_check`.
- permission: PK code; `code ~ '^[a-z][a-z0-9-]*:[a-z][a-z0-9-]*$'`; risk_level check.
- role: `tenant_code_key UNIQUE NULLS NOT DISTINCT (tenant_id, code)`; `role_code_namespace_check ((tenant_id IS NULL) = (code NOT LIKE 'custom.%'))` (custom codes can never equal a system code); `role_platform_check (scope_type <> 'platform' OR tenant_id IS NULL)`; `id_tenant_key UNIQUE (id, tenant_id)` not needed (F1).
- role_permission: PK (role_id, permission_code); `permission_code_idx`; trigger `role_permission_scope_check`.
- role_assignment: `active_key UNIQUE (profile_id, role_id, scope_id) NULLS NOT DISTINCT WHERE revoked_at IS NULL`; `profile_active_idx (profile_id) INCLUDE (role_id, scope_id, expires_at) WHERE revoked_at IS NULL` (hot path: permission evaluation); `tenant_role_active_idx (tenant_id, role_id) WHERE revoked_at IS NULL` (last-admin count, role usage); `expires_idx (expires_at) WHERE revoked_at IS NULL AND expires_at IS NOT NULL` (sweeper); FK `(profile_id, tenant_id) -> profile(id, tenant_id)`; FK `(tenant_id, scope_id) -> access_scope(tenant_id, id)`; FK role_id; triggers `role_tenant_check`, `role_assignment_scope_check`.
- role_assignment_rule: PK; FKs on both role ids; trigger `role_tenant_check` on both.
- access_scope: PK id; `tenant_id_key UNIQUE (tenant_id, id)`; type check.
- inbox: PK (consumer, event_id); `received_at_idx` for purge.

### audit
- PK (occurred_at, id); `tenant_occurred_idx (tenant_id, occurred_at DESC)`; `subject_idx (subject_type, subject_id, occurred_at DESC)`; `REVOKE UPDATE, DELETE`.

### Hot paths
| Path | Index |
|---|---|
| Resolve by slug | tenant_slug_key (then 60 s cache) |
| Resolve by host | tenant_domain_host_key (partial, verified) |
| Token version | user_account PK (cached 60 s) |
| Profile in tenant | profile_user_tenant_key INCLUDE (id, status) - index-only scan |
| Effective access | role_assignment_profile_active_idx + role_permission PK, cached by (profile, access_version, catalogue_version) |
| Refresh | refresh_token PK |
| API key | api_key_key_hash_key |

---

## 4. Concurrency playbook (summary)
| Case | Mechanism |
|---|---|
| Last admin | Lock `tenant_policy` row FOR UPDATE, count, act, bump access_version |
| Any access change | Same row lock (serialises access writes per tenant; low volume) |
| One open invitation | Partial unique index; revoke expired open row first |
| Sign-up double submit | ON CONFLICT on email and on (user_id, tenant_id); Idempotency-Key on the HTTP layer |
| Refresh rotation | Conditional UPDATE ... WHERE used_at IS NULL RETURNING; 409 in grace, revoke after |
| Tenant job lock | Partial unique index + heartbeat reaper; ON CONFLICT ... WHERE ... DO NOTHING |
| Backup code / sign count | SELECT ... FOR UPDATE on authenticator row |
| Status transitions | Guarded UPDATE ... WHERE status = :expected (or Doctrine #[Version]) |
| Event consumers | Inbox row + state change + outbox in one transaction; source_version check for ordering |

---

## 5. Sources
- PostgreSQL CREATE TABLE (NULLS NOT DISTINCT, MATCH SIMPLE/FULL, SET NULL column list): https://www.postgresql.org/docs/current/sql-createtable.html
- PostgreSQL partitioning limitations (PK must include partition key): https://www.postgresql.org/docs/current/ddl-partitioning.html
- PostgreSQL INSERT ON CONFLICT index inference with predicate: https://www.postgresql.org/docs/current/sql-insert.html
- PostgreSQL 18 uuidv7(): https://www.postgresql.org/docs/18/functions-uuid.html
- PostgreSQL SSI / write skew: https://wiki.postgresql.org/wiki/SSI
- Cybertec, SELECT FOR UPDATE considered harmful: https://www.cybertec-postgresql.com/en/select-for-update-considered-harmful-postgresql/
- Foreign keys and partitioned tables (PG12+): https://enterprisedb.com/blog/postgresql-12-foreign-keys-and-partitioned-tables
- pg_partman (Crunchy Data): https://access.crunchydata.com/documentation/pg-partman/4.5.1/epub/pg_partman.epub
- use-the-index-luke, partial indexes: https://use-the-index-luke.com/sql/where-clause/partial-and-filtered-indexes
- PgBouncer configuration (max_prepared_statements, session features in transaction mode): https://www.pgbouncer.org/config.html
- Doctrine ORM limitations (no join columns to non-primary keys): https://www.doctrine-project.org/projects/doctrine-orm/en/current/reference/limitations-and-known-issues.html
- Doctrine ORM attributes (Index options where, insertable/updatable): https://www.doctrine-project.org/projects/doctrine-orm/en/current/reference/attributes-reference.html
- Doctrine ORM composite keys tutorial: https://www.doctrine-project.org/projects/doctrine-orm/en/3.7/tutorials/composite-primary-keys.html
- Doctrine DBAL 4 types (simple_array, json/jsonb, guid->uuid): https://www.doctrine-project.org/projects/doctrine-dbal/en/4.2/reference/types.html
- RFC 9700 OAuth 2.0 Security BCP, 4.14 refresh token rotation: https://www.rfc-editor.org/rfc/rfc9700
- NIST RBAC unified model (basis of ANSI INCITS 359): https://csrc.nist.gov/pubs/conference/2000/07/26/nist-model-for-rbac-towards-a-unified-standard/final
- NIST RBAC FAQ: https://csrc.nist.gov/projects/role-based-access-control/faqs
- ARBAC97 (Sandhu, Bhamidipati, Munawer): https://profsandhu.com/confrnc/rbac/r98pra97.pdf
- ARBAC02 critique of prerequisite conditions: https://profsandhu.com/zhang/pub/arbac02.pdf
- Microsoft, tenant life cycle: https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/considerations/tenant-life-cycle
- Microsoft, control planes: https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/considerations/control-planes
- Microsoft, database-per-tenant catalog: https://learn.microsoft.com/uk-ua/azure/azure-sql/database/saas-dbpertenant-provision-and-catalog?view=azuresql
- AWS SaaS tenant isolation strategies (silo): https://docs.aws.amazon.com/whitepapers/latest/saas-tenant-isolation-strategies/silo-isolation.html
