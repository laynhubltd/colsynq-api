# RBAC + Identity + Tenancy Control-Plane Schema: Standard Practice (PostgreSQL 17/18, Doctrine)

Research date: 2026-10-12. Perspective: senior DBA / data modeller. Sources are listed per section and at the end.
Legend: **[V]** = verified against the fetched primary source this session; **[P]** = practitioner consensus / reviewer judgement (not a single citable rule).

---

## 0. TL;DR recommendations

1. `user` (global identity) → `membership` (user × tenant, carries status) → `membership_role` (membership × role). Org roles attach to the **membership**, never to the user. Every vendor surveyed does this (Auth0, WorkOS, Clerk, OpenFGA). [V]
2. System roles: **one shared row with `tenant_id NULL`** plus tenant-custom rows with `tenant_id NOT NULL`. Do not copy system roles per tenant. Enforce uniqueness with two partial unique indexes (or `UNIQUE NULLS NOT DISTINCT (tenant_id, key)`). [V/P]
3. Permissions are a global, code-defined catalogue (immutable slug). `role_permission` is a pure join table. [V – WorkOS/Clerk]
4. Model "who can assign which role" as data (`role_assignable_role` / ARBAC `can_assign`), not as `if` statements; model SoD only if a real requirement exists (`sod_constraint` + `sod_constraint_role`). [V – NIST/ARBAC]
5. Copy `tenant_id` onto child rows **only** when it buys a composite FK `(tenant_id, x_id)` that makes cross-tenant references impossible, or RLS / partition / index locality. Otherwise it is redundancy. [P]
6. Don't put scope on both role and assignment; don't keep `status` + `is_active` + `deleted_at` on the same row; keep `access_version` as a column on the subject it versions (membership/user), not a separate table. [P]
7. Two modules sharing one PostgreSQL database: prefer a **view in the owning module's schema + `GRANT SELECT`** over a copied projection table; use a projection only if you need independent lifecycle, different shape for hot-path indexes, or the modules will split databases. [P]
8. Audit: one append-only, partitioned `audit_event` table (jsonb payload) + domain rows with `revoked_at/revoked_by`. Don't build per-domain history tables unless you need point-in-time reconstruction queries. [P]
9. PG18 temporal keys (`WITHOUT OVERLAPS`, `PERIOD`) are real and usable for `valid_during tstzrange` on role assignments, but the FK side doesn't support `ON DELETE CASCADE`, needs `btree_gist`, and Doctrine has no mapping for it — use raw-SQL migrations. Most SaaS products don't need it; `granted_at/expires_at/revoked_at` suffices. [V]
10. Credentials: keep **password hash on its own table or user**, and use one `authenticator` table (type discriminator + `public_data jsonb` + `secret_data`) for TOTP/WebAuthn/backup-code-set — the Keycloak `CREDENTIAL` model. Keep **refresh tokens/sessions, API keys, external identities (federated links), and password-reset/verification tokens in separate tables** — they are not authenticators and have very different volume, lifecycle and indexes. [V – Keycloak/NIST]
11. Tenancy: `tenant`, `tenant_domain` (1:N), `plan`, `feature`, `plan_feature` (entitlement with limit), `subscription` (tenant × plan, time-bounded), `tenant_entitlement_override`, `provisioning_run`/`migration_run` (append-only job logs). Merge a 1:1 `tenant_database` into `tenant` **unless** it holds secrets with a different grant/encryption policy, or is optional/sparse (silo-only tenants). [P]
12. PostgreSQL: UUIDv7 PKs (`DEFAULT uuidv7()` on PG18, app-generated on PG17), `timestamptz` everywhere, `text` + `CHECK` for small status sets (or lookup table if it carries metadata), nondeterministic ICU collation instead of `citext` for email (PG docs recommend it; PG18 added `LIKE` support), snake_case singular table names, index every FK, partial unique indexes for "one active row". [V]

---

## 1. Canonical RBAC tables and the membership entity

### 1.1 NIST Core RBAC → relational mapping
ANSI INCITS 359-2004 (revised 2012) defines Core RBAC (USERS, ROLES, PERMS = OPS×OBS, UA, PA, SESSIONS), Hierarchical RBAC (RH partial order; limited = tree, general = DAG), SSD (constraints at assignment time) and DSD (constraints at session activation). Core is mandatory; the others are independent options. [V]

Relational mapping for multi-tenant SaaS (the "user" in NIST corresponds to a **membership** in SaaS, because a human has different roles per tenant):

| NIST element | Table |
|---|---|
| USERS | `app_user` (global identity: email, name, status) |
| (tenant subject) | `membership` (user_id, tenant_id, status, joined_at) |
| ROLES | `role` (id, tenant_id NULL = system, key, name, is_system) |
| PERMS | `permission` (id/key slug e.g. `project:write`, description) — global catalogue |
| PA | `role_permission` (role_id, permission_id) PK composite |
| UA | `membership_role` (membership_id, role_id, granted_by, granted_at, expires_at?) |
| RH | `role_inheritance` (senior_role_id, junior_role_id) — only if needed |
| SESSIONS | session / refresh-token table; DSD rarely implemented in SaaS |

### 1.2 How vendors model "user belongs to many orgs"
- **Auth0 Organizations**: roles are defined at tenant (Auth0 tenant = your app) level *or* per-organization, but always **assigned to a member within one Organization**; they apply when the user logs in through that org. Up to 50 role IDs per member via Management API. [V]
- **WorkOS**: Permissions = immutable slugs. Environment-level roles apply to every org by default (with a default `member` role); orgs may define custom roles (`org`-prefixed slug) and then get their own default + priority order. `organization_membership` links user↔org and holds the role(s); by default exactly one role per membership, optional multi-role = union of permissions. Every membership must have ≥1 role. Roles may be driven by SCIM/SSO group mappings. [V]
- **Clerk**: org roles (`org:admin`, `org:member` defaults, custom roles), permissions tied to features (`org:<feature>:<perm>`), system permissions `org:sys_*`; role set per org; one role per membership in practice. [V]
- **Keycloak**: realm = tenant boundary (strong isolation, heavy); `USER_ENTITY` belongs to one realm; realm roles + client roles; `USER_ROLE_MAPPING`, `COMPOSITE_ROLE` for hierarchy, `GROUP` + `GROUP_ROLE_MAPPING`. Keycloak 26 added "Organizations" inside a realm (members + domains + IdP links) — again a membership entity. [P]
- **Zitadel**: Instance → Organization → Project; users belong to one org; cross-org access is via "user grants"/"project grants" (a membership-like grant carrying role keys). [P]
- **OpenFGA / Zanzibar / SpiceDB**: roles are relations on the `organization` object (`user:anne` is `admin` of `organization:A`); tenant context is passed per request (contextual tuples) to restrict to the active org. The tuple `(org, relation, user)` is exactly a `membership_role` row. [V]

**Conclusion:** org-level roles attach to the membership (user × tenant), not the user. Platform-operator roles (staff of the SaaS vendor) are a separate, tenant-less assignment table (`platform_role_assignment(user_id, role_id)`) or memberships in a reserved "platform" tenant — don't make `membership.tenant_id` nullable for that. [P]

### 1.3 System roles: shared rows vs copied per tenant

| | Shared (`tenant_id IS NULL`) | Copied per tenant |
|---|---|---|
| Change system role permissions | one UPDATE, all tenants | N updates / data migration per release |
| Tenant customises a system role | impossible (must clone into custom role) | trivially possible, but then drift |
| Row count | O(roles) | O(tenants × roles) |
| RLS | policy must allow `tenant_id IS NULL OR tenant_id = current` | simple equality |
| Composite FK `(tenant_id, role_id)` from membership_role | doesn't work for NULL tenant (FK not checked when any column NULL) | works |

Recommendation [P]: **shared rows** for system roles (vendors do this: WorkOS environment roles, Auth0 tenant roles, Clerk default roles). Tenants that need variation create a custom role (optionally "cloned from" a system role, `cloned_from_role_id`). Enforce "custom role belongs to membership's tenant" via a trigger or an app check, because a composite FK can't express "same tenant OR NULL".

Unique constraints with NULL tenant_id:
```sql
-- Option A (PG15+): single constraint, NULL treated as equal
ALTER TABLE role ADD CONSTRAINT role_tenant_key_uq UNIQUE NULLS NOT DISTINCT (tenant_id, key);
-- Prevents two system roles with same key AND two custom roles with same key in one tenant.
-- Does NOT prevent a tenant custom role from shadowing a system key; add a check/trigger if required.

-- Option B: two partial unique indexes (works on any version, clearer intent)
CREATE UNIQUE INDEX role_system_key_uq ON role (key) WHERE tenant_id IS NULL;
CREATE UNIQUE INDEX role_tenant_key_uq ON role (tenant_id, key) WHERE tenant_id IS NOT NULL;
```
Plain `UNIQUE (tenant_id, key)` is **wrong**: default NULLS DISTINCT allows duplicate system roles. [V – PG docs]
Doctrine note: Doctrine ORM can't declare `NULLS NOT DISTINCT` or partial indexes reliably via attributes (`options: {where: ...}` exists for indexes on PostgreSQL platform; NULLS NOT DISTINCT needs raw SQL in the migration). Exclude such indexes from schema-diff noise via `schema_filter` or by accepting manual migrations. [P]

---

## 2. Role hierarchy, can_assign, separation of duty

### 2.1 Hierarchy
- Limited hierarchy (tree) vs general (DAG) per NIST. [V]
- SaaS practice: most products **avoid** hierarchy and use flat roles with explicit permission sets (Auth0/WorkOS/Clerk have no role inheritance); permissions expansion is cheaper to reason about and audit. [P]
- If needed: `role_inheritance(senior_role_id, junior_role_id, PRIMARY KEY(senior, junior), CHECK (senior <> junior))`; prevent cycles with a trigger or app check; resolve with recursive CTE and cache the effective permission set per role (materialised table refreshed on change) since role graphs change rarely.

### 2.2 Who can assign which role (ARBAC97 / URA97)
ARBAC97 = URA97 (user-role assignment), PRA97 (permission-role), RRA97 (role-role). `can_assign(admin_role, prerequisite_condition, role_range)`: a member of admin role (or senior) may assign a user satisfying the prerequisite to roles in the range; `can_revoke(admin_role, role_range)`. ARBAC02 criticised role ranges as fragile when the hierarchy changes. [V]

Practical relational form (drop range/prerequisite generality):
```sql
CREATE TABLE role_assignable_role (           -- can_assign
  assigner_role_id uuid NOT NULL REFERENCES role(id) ON DELETE CASCADE,
  assignable_role_id uuid NOT NULL REFERENCES role(id) ON DELETE CASCADE,
  PRIMARY KEY (assigner_role_id, assignable_role_id)
);
```
Alternative common in SaaS: a numeric `rank`/`priority` on role and the rule "you can only grant roles with rank ≤ your highest rank" (WorkOS keeps a role priority order). Simpler; less expressive. Pick one, never both. Also enforce "can't remove the last owner" (app-level, or deferred constraint trigger). [P]

Permission-based alternative: a permission `member:assign_role` plus the rank rule. Most products do this. [P]

### 2.3 Separation of duty (NIST SSD/DSD)
SSD = mutually exclusive roles at assignment (with cardinality n: no user may hold ≥ n roles from the set); DSD = same at session activation. [V]
```sql
CREATE TABLE sod_constraint (id uuid PK, tenant_id uuid NULL, kind text CHECK (kind IN ('static','dynamic')), cardinality smallint CHECK (cardinality >= 2), name text);
CREATE TABLE sod_constraint_role (sod_constraint_id uuid REFERENCES sod_constraint ON DELETE CASCADE, role_id uuid REFERENCES role ON DELETE CASCADE, PRIMARY KEY (sod_constraint_id, role_id));
```
Enforce SSD in the assignment command handler (in a transaction with `SELECT ... FOR UPDATE` on the membership row to avoid races). Only build this if there's a compliance requirement (finance/approval workflows); YAGNI otherwise. [P]

---

## 3. Redundancy pitfalls

### 3.1 Denormalised `tenant_id` on child rows — when justified
Justified when it is **used**:
1. **Composite FK enforcing same tenant**: parent has `UNIQUE (tenant_id, id)`; child has `FOREIGN KEY (tenant_id, membership_id) REFERENCES membership (tenant_id, id)`. This makes a cross-tenant reference structurally impossible — the only DB-level way to guarantee it. [P, widely recommended for pool model]
2. **RLS**: policies `USING (tenant_id = current_setting('app.current_tenant')::uuid)` need the column on the row; joining to parent inside a policy is slow and fragile. AWS pool-model guidance puts tenant id on every table. [V – AWS]
3. **Partitioning / index locality**: leading `tenant_id` in composite indexes for tenant-scoped lists.
Not justified: copying `tenant_id` "for convenience" with no composite FK, RLS, or index using it — it becomes a second source of truth that can disagree with the parent. If you copy it, **always** pair it with the composite FK so it cannot drift.

RLS pitfalls [V – AWS]: superusers/BYPASSRLS skip policies; table owners skip unless `FORCE ROW LEVEL SECURITY`; app must connect as non-owner role; session variables vs PgBouncer transaction pooling — use `SET LOCAL` / `set_config(..., true)` inside each transaction.

### 3.2 Scope stored in both role and assignment
If `role.scope` ('platform'|'tenant'|'project') and `membership_role.scope` both exist, they will disagree. Rule: **the role defines what it can apply to (its type); the assignment defines where it is applied (the resource id)**. E.g. `role.applies_to = 'project'` and `assignment.resource_id`; CHECK/trigger that the assignment's resource type matches the role's. Don't repeat the type on the assignment. [P]

### 3.3 Duplicate status columns
Anti-pattern: `status`, `is_active`, `is_deleted`, `deleted_at`, `suspended_at` on the same table. Pick **one** state column (`status text CHECK (...)`) plus event timestamps that are facts (`suspended_at`, `deactivated_at`) only if they're needed for queries; or derive status from timestamps — never both independently writable. If both exist, add a CHECK tying them: `CHECK ((status = 'suspended') = (suspended_at IS NOT NULL))`. Prefer status for lifecycle; soft-delete via `deleted_at` only for "trash/restore" semantics. [P]

### 3.4 `access_version` placement
`access_version` (a monotonic counter bumped when a subject's effective permissions change, embedded in tokens to invalidate caches/JWTs) belongs **on the row whose change it versions** — `membership.access_version` (per-tenant permissions) and maybe `app_user.security_stamp` (password change / global logout). A separate 1:1 `access_version` table adds a join and a second row to lock for no benefit, unless the update rate is so high it causes HOT/bloat contention on a wide row (rare). Bumping: `UPDATE membership SET access_version = access_version + 1 WHERE id = $1` in the same transaction as the role change. Role-permission edits affect all holders → either bump all affected memberships (one UPDATE with join) or include a `role.version`/tenant-level `authz_version` in the cache key. [P]

### 3.5 Projections inside one database
When module B needs module A's data and both live in the same PostgreSQL database:
- **Default: a view owned by module A** (`CREATE VIEW identity_api.membership_v AS SELECT ...` in A's schema) and `GRANT SELECT` to B's DB role. A controls the contract (columns), B can't write, no sync lag, no consistency bugs. This is the "database view as published interface" pattern. [P]
- A projection **table** (populated by domain events/outbox) is justified when: modules will be split into separate databases soon; B needs a different shape with its own indexes on hot paths (e.g. flattened effective-permission table); B must keep working if A's tables are being migrated/locked; or A's data is in another DB. Then it's a deliberate read model with an owner, rebuild command, and lag tolerance.
- A projection that's a 1:1 copy of a table in the same DB, kept in sync synchronously, is pure redundancy — replace with a view.
- Doctrine: map the view as a read-only entity (`#[ORM\Entity(readOnly: true)]`) and exclude it from schema tool (`schema_filter`), create via migration.

---

## 4. Audit and temporal validity

### 4.1 Generic append-only audit vs per-domain audit tables
Common SaaS practice [P]: one **append-only `audit_event`** table:
```
id uuid (v7) PK, occurred_at timestamptz NOT NULL, tenant_id uuid NULL, actor_type text, actor_id uuid NULL,
actor_ip inet NULL, action text NOT NULL (e.g. 'membership.role_granted'), target_type text, target_id uuid,
request_id uuid, payload jsonb NOT NULL DEFAULT '{}'
```
- Partition by range on `occurred_at` (monthly) for retention/drop; index `(tenant_id, occurred_at DESC)`, `(target_type, target_id, occurred_at DESC)`.
- Make it append-only: `REVOKE UPDATE, DELETE ON audit_event FROM app_role`; optionally a trigger raising on UPDATE/DELETE.
- Write in the same transaction as the change (or via outbox) so audit can't be lost.
- No FKs to domain tables (targets may be deleted; partitioned table FKs are costly).
Per-domain audit/history tables (`membership_role_history`) are warranted only when you must query state "as of time T" relationally (compliance access reviews). Trigger-based history (`*_history` with `tstzrange`) is the classic approach; PG has no built-in system-versioned tables (SQL:2011 SYSTEM_TIME is not implemented as of PG18). [V – PG18 release notes list application-time constraints only]

### 4.2 `revoked_at` soft delete vs hard delete + history
For role assignments: keep row with `revoked_at`, `revoked_by` → active-assignment uniqueness via partial unique index:
```sql
CREATE UNIQUE INDEX membership_role_active_uq ON membership_role (membership_id, role_id) WHERE revoked_at IS NULL;
```
Every query must filter `revoked_at IS NULL` (Doctrine SQL filter or a view `membership_role_active`). Alternative: hard-delete the assignment and rely on `audit_event` for history — simpler queries, smaller hot table; the audit table is the history. Both are accepted; don't do both soft delete AND a history table. [P]

### 4.3 PostgreSQL 18 temporal constraints
PG18: `PRIMARY KEY/UNIQUE (..., valid_at WITHOUT OVERLAPS)` and `FOREIGN KEY (..., PERIOD valid_at) REFERENCES t (..., PERIOD valid_at)`. [V]
- Last column must be a range/multirange; empty ranges disallowed; GiST index; `btree_gist` needed for uuid/int equality columns ("the expected way to use this feature").
- `UNIQUE (id, valid_at WITHOUT OVERLAPS)` ≈ `EXCLUDE USING gist (id WITH =, valid_at WITH &&)` — which already works on PG17.
- Temporal FKs do **not** support `ON DELETE/UPDATE CASCADE/SET NULL/SET DEFAULT/RESTRICT` actions.
Application to role assignment:
```sql
CREATE EXTENSION IF NOT EXISTS btree_gist;
CREATE TABLE membership_role (
  membership_id uuid NOT NULL REFERENCES membership(id) ON DELETE CASCADE,
  role_id uuid NOT NULL REFERENCES role(id),
  valid_during tstzrange NOT NULL DEFAULT tstzrange(now(), NULL),
  granted_by uuid, ...,
  CONSTRAINT membership_role_no_overlap UNIQUE (membership_id, role_id, valid_during WITHOUT OVERLAPS)
);
-- active now:  WHERE valid_during @> now()
```
Recommendation: use it only if time-bound grants (JIT access, scheduled expiry, contractor access windows) are a product feature. Otherwise `granted_at`, `expires_at NULL`, `revoked_at NULL` + partial unique index is simpler and fully supported by Doctrine. Doctrine has no `tstzrange` type — needs a custom DBAL type and raw-SQL migration. On PG17, use the equivalent `EXCLUDE USING gist` constraint.

---

## 5. Credentials / authenticators

### 5.1 NIST SP 800-63B (rev 4) vocabulary
Authenticator types: memorized secret/password, look-up secret (backup codes), out-of-band device, single-/multi-factor OTP, single-/multi-factor cryptographic (passkeys/WebAuthn, hardware keys). CSP binds authenticators to the subscriber account; must be able to invalidate an authenticator immediately on reported loss/compromise; rate-limit (≤100 consecutive failures); verifiers store hashed/derived values, not raw secrets. [V]
So in NIST terms password, TOTP, WebAuthn, backup codes are all **authenticators**; sessions, refresh tokens, API keys are not (they are session secrets / service credentials), and federated identities are **federation** (800-63C), not authenticators.

### 5.2 Keycloak
Single `CREDENTIAL` table: `id, user_id, type, created_date, user_label, secret_data (JSON, never leaves server), credential_data (JSON, public metadata), priority`. Password, OTP, WebAuthn, recovery codes all live here, discriminated by `type`. Federated links are separate (`FEDERATED_IDENTITY`), sessions are separate (`OFFLINE_USER_SESSION`/Infinispan), required actions separate. [V]

### 5.3 Recommended split
| Table | Why separate |
|---|---|
| `user_authenticator` (type: `password`?, `totp`, `webauthn`, `backup_codes`, `sms`?) with `public_data jsonb`, `secret_data` (encrypted text/bytea), `label`, `last_used_at`, `created_at`, `revoked_at` | Same lifecycle: bind / use / revoke; MFA listing UI is one query. Keycloak model. |
| password: either in the authenticator table (`type='password'`, partial unique index one per user) or `app_user.password_hash` | Both fine; a column is simpler for Symfony `PasswordAuthenticatedUserInterface`. |
| backup codes | One authenticator row (set) + child `backup_code(authenticator_id, code_hash, used_at)` — or a hashed array in secret_data; individual codes need `used_at`, so a child table is cleaner. |
| WebAuthn credential id lookup | WebAuthn needs lookup by `credential_id` (bytea) → unique index on extracted column; store `credential_id` as a real column, not only in jsonb. |
| `refresh_token` / `user_session` | High volume, hot, expiring, rotation family (`family_id`, `replaced_by_id`), purge job. Never mix with authenticators. |
| `api_key` (service credential: prefix, hash, scopes, owner membership/tenant, expires_at, last_used_at) | Different owner (often tenant/service account), scopes, lookup by prefix. |
| `external_identity` (provider, subject, user_id, UNIQUE(provider, subject)) | Federation link, not a secret; uniqueness on provider subject. |
| `one_time_token` (purpose: `password_reset`,`email_verify`,`invite`; token_hash; expires_at; consumed_at) | Consolidating the short-lived single-use tokens into **one** table with a `purpose` column is common and reasonable. |

Rule of thumb: consolidate rows that share lifecycle, access pattern and security policy; separate those that don't. [P]

---

## 6. Tenancy control plane

### 6.1 Standard entities
- `tenant` (id, slug UNIQUE, name, status, isolation_model `pool|bridge|silo`, region, created_at). AWS SaaS: silo = DB per tenant, bridge = schema per tenant, pool = shared tables + tenant_id. [V – AWS]
- `tenant_domain` (tenant_id, domain UNIQUE (case-insensitive), is_primary, verified_at, verification_token_hash) — 1:N; "one primary" via partial unique index `(tenant_id) WHERE is_primary`.
- `tenant_database` / connection info for silo/bridge.
- `plan` (key, name, is_public, version), `feature` (key, type `boolean|limit|metered`), `plan_feature` (plan_id, feature_id, limit_value NULL) = entitlements by plan.
- `subscription` (tenant_id, plan_id, status, current_period tstzrange or start/end, external_ref e.g. Stripe id). One active per tenant: partial unique index `(tenant_id) WHERE status IN ('active','trialing','past_due')`.
- `tenant_entitlement_override` (tenant_id, feature_id, limit_value, expires_at) for sales deals; effective entitlement = override ?? plan_feature.
- `provisioning_run`, `migration_run` (tenant_id, kind, target_version, status, started_at, finished_at, error jsonb, attempt) — append-only operational logs; one running per tenant via partial unique index `(tenant_id) WHERE status = 'running'` as a lock.

### 6.2 1:1 tables — merge or split?
Merge by default (1:1 mandatory = same entity). Split is justified when [P]:
- different **security** posture: connection secrets (DSN/password ref) in `tenant_database` with column/table grants restricted to the provisioning service; app role can't read it;
- **optional/sparse**: only silo/bridge tenants have a dedicated database (pool tenants would have all-NULL columns);
- different **write frequency/lock contention** (e.g., migration status updated constantly vs tenant row read on every request);
- likely to become 1:N (read replicas, multiple shards per tenant).
If `tenant_database` exists for every tenant, is never secret-separated, and only holds `db_name`, merge it. Store secrets as references to a secret manager (Vault/KMS), not plaintext.

### 6.3 Plan/feature/entitlement
Keep plans versioned (new plan row rather than editing an active one subscribers are on), entitlements resolved in one place (view `tenant_effective_entitlement`), usage counters for metered features in a separate table (high write rate). Billing provider (Stripe) is source of truth for money; your DB is source of truth for entitlements. [P]

---

## 7. PostgreSQL standards

| Topic | Recommendation |
|---|---|
| PK type | UUIDv7. PG18: `DEFAULT uuidv7()` (time-ordered: ms timestamp + sub-ms + random; `uuid_extract_timestamp()` works). PG17: generate in app (symfony/uid `UuidV7`, Doctrine `uuid` type) — app-side generation is also fine on PG18 and lets aggregates know their id before flush. Don't expose v7 ids if creation time is sensitive. [V] |
| Timestamps | `timestamptz` always; never `timestamp` without tz. Doctrine: `datetimetz_immutable`. [P/PG wiki "Don't Do This"] |
| Case-insensitive text (email, slug, domain) | PG docs tip: "Consider using nondeterministic collations instead of [citext]" — they handle Unicode cases citext's lower() misses. `CREATE COLLATION case_insensitive (provider = icu, locale = 'und-u-ks-level2', deterministic = false);` PG18 allows `LIKE` with nondeterministic collations. Costs: slower comparisons, no B-tree dedup. Alternative widely used: store normalised `email_normalized text` (lower/casefold, PG18 `casefold()`) with a plain unique index — simplest with Doctrine. Avoid citext for new designs. [V] |
| Enums | `text` + `CHECK (status IN (...))` for small, code-owned sets (easy to change in a migration; Doctrine maps to PHP backed enum via `enumType`). Native `CREATE TYPE ... AS ENUM` — can't remove values, awkward in Doctrine. Lookup table when values carry data (label, sort, is_terminal) or are user-managed (e.g. permissions, plans). [P] |
| Naming | snake_case, lowercase, no quoted identifiers; singular table names are the more common modern convention for ORM-backed apps (`membership`, `role`), but consistency matters more than the choice; avoid reserved words (`user` is reserved in PG → use `app_user` / `user_account`). Constraint names: `{table}_{cols}_{pk|fk|uq|ck|idx}`. [P] |
| FK indexes | PG does **not** auto-index the referencing side; index every FK column (or have it as leading column of a composite index) — required for cascade deletes and joins. Doctrine creates them for ManyToOne by default. [P] |
| "One active row" | Partial unique index: `CREATE UNIQUE INDEX ... ON subscription (tenant_id) WHERE status = 'active';` [V – PG docs indexes-partial] |
| jsonb | For truly variable/opaque data (audit payload, provider metadata, authenticator public data, settings blobs). Don't put FKs, things you filter/join on frequently, or uniqueness-bearing values inside jsonb; extract them to columns (or generated columns). Size: TOASTed above ~2 kB; whole-value rewrite on any update. [P] |
| Generated columns | PG18 virtual generated columns are now default (`GENERATED ALWAYS AS (...)` virtual); stored ones still needed if indexed. [V] |
| RETURNING OLD/NEW | PG18 supports `RETURNING old.*, new.*` — handy for writing audit events in one statement. [V] |

---

## 8. Sources

PostgreSQL
- citext (tip recommending nondeterministic collations): https://www.postgresql.org/docs/current/citext.html
- Collations / nondeterministic: https://www.postgresql.org/docs/18/collation.html
- CREATE TABLE (WITHOUT OVERLAPS, PERIOD, NULLS NOT DISTINCT): https://www.postgresql.org/docs/18/sql-createtable.html
- UUID functions (uuidv7): https://www.postgresql.org/docs/18/functions-uuid.html
- PG18 release notes: https://www.postgresql.org/docs/18/release-18.html
- Partial indexes: https://www.postgresql.org/docs/current/indexes-partial.html
- Don't Do This (timestamp, etc.): https://wiki.postgresql.org/wiki/Don%27t_Do_This
- Row security: https://www.postgresql.org/docs/current/ddl-rowsecurity.html

RBAC standards
- ANSI INCITS 359-2004 text: https://www.cs.purdue.edu/homes/ninghui/readings/AccessControl/ANSI+INCITS+359-2004.pdf
- Apache Fortress ANSI RBAC overview: https://directory.apache.org/fortress/user-guide/1.3-what-rbac-is.html
- Li et al., critique of RBAC standard: https://www.cs.purdue.edu/homes/ninghui/papers/aboutRBACStandard.pdf
- ARBAC97 PRA97: https://profsandhu.com/confrnc/rbac/r98pra97.pdf
- ARBAC02 (Oh, Sandhu, Zhang): https://profsandhu.com/zhang/pub/arbac02.pdf
- ARBAC lecture (Purdue): https://www.cs.purdue.edu/homes/ninghui/courses/Spring05/lectures/lecture13.pdf

Vendors
- Auth0 org member roles: https://auth0.com/docs/manage-users/organizations/configure-organizations/add-member-roles
- WorkOS roles & permissions: https://workos.com/docs/authkit/roles-and-permissions
- Clerk org roles/permissions: https://clerk.com/docs/guides/organizations/roles-and-permissions
- Keycloak server development (Credentials table): https://www.keycloak.org/docs/latest/server_development/index.html
- OpenFGA org-context authorization: https://openfga.dev/docs/modeling/organization-context-authorization
- AWS RLS multi-tenant isolation: https://aws.amazon.com/blogs/database/multi-tenant-data-isolation-with-postgresql-row-level-security/

NIST identity
- SP 800-63B-4: https://pages.nist.gov/800-63-4/sp800-63b.html

Caveats: Zitadel, Ory Keto, Azure multitenant guidance and SpiceDB items are from reviewer knowledge (not re-fetched this session); claims marked [P] are practitioner judgement.
