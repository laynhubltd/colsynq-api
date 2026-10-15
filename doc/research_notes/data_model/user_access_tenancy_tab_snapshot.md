User, access control and tenancy: data model and architecture
Owner: lead backend engineering, reviewed by data science and database administration · Applies to: the Tenancy and IAM (identity and access) modules and every protected endpoint · Pattern: REPROMAS profile pattern and RBAC, cleaned up against NIST RBAC (ANSI INCITS 359), ARBAC97, NIST SP 800-63B-4, PostgreSQL 18 documentation and the data models of Auth0, WorkOS, Clerk and Keycloak
The model in one sentence: a global User gets one UserProfile per tenant; UserRole grants a catalogue Role to a profile; RolePermission gives each role its catalogue Permissions; UserRoleAssignRole says which roles a role holder may assign; the landlord holds Tenant, TenantDatabase and the rest of the tenancy registry.
Out of scope (section 21): the approval workflow and cooperative membership.
1. Review: what was removed or merged, and why
The previous version had 27 tables, with many facts stored twice. This version has 26 tables, each fact stored once; three of them (feature, plan_feature, tenant_entitlement_override) replace list columns on plans.
#
 | Finding
 | Problem
 | Change
 | 
1
 | Identity and Access were separate modules in the same landlord database
 | Access copied profiles into ProfileProjection; Access wrote access_version on Identity's table; disabling a profile needed Access's admin count (cross-module rule)
 | Merged into one IAM module (schema iam): real foreign keys, no projection, one transaction for profile + roles. Vendors model users, memberships and roles as one realm (Keycloak, Auth0)
 | 
2
 | UserProfile.tenant_id nullable for platform staff
 | Nullable tenant breaks uniqueness and every "per tenant" query
 | Colsynq itself is a platform tenant (tenant.kind = platform, Keycloak's master-realm idea); staff have normal profiles there; tenant_id NOT NULL
 | 
3
 | User.is_platform_staff
 | Same fact as "has an active profile in the platform tenant"
 | Removed
 | 
4
 | User.mfa_enabled
 | Derivable from authenticators
 | Removed
 | 
5
 | User status locked and pending_verification
 | Same facts as locked_until and email_verified_at
 | Status is now active or deactivated only
 | 
6
 | Profile email, phone, display_name
 | Copies of the User's login identity; display name derivable
 | Removed; society-specific contact details belong to the future Membership module
 | 
7
 | Profile disabled_at, removed_at; Tenant suspended_at, archived_at, suspension_reason, failed_step, last_error
 | Several timestamps for one status; errors duplicated from job rows
 | One status, status_reason, status_changed_at; history in audit_event; errors in tenant_job
 | 
8
 | Role.is_system, Role.is_platform, Permission.is_platform
 | Same facts as tenant_id IS NULL and scope = PLATFORM
 | Removed
 | 
9
 | UserRole.scope
 | Always equal to the role's scope
 | Removed; the role says what kind of resource, the grant says which one
 | 
10
 | UserRole.granted_by, revoked_by, revoke_reason; RolePermission.granted_by
 | Same facts as the audit log
 | Kept only timestamps on the rows; who and why live in audit_event
 | 
11
 | AccessAudit plus separate authentication audit events
 | Two audit stores
 | One append-only, monthly-partitioned iam.audit_event
 | 
12
 | tenant_edge_override table
 | A second table to switch off a default edge
 | effect column (allow or deny) on UserRoleAssignRole; a tenant row overrides the default
 | 
13
 | RoleConflict allowed (A,B) and (B,A)
 | Symmetric duplicates
 | CHECK (role_a_id < role_b_id)
 | 
14
 | Tenant.plan_code and Subscription.plan_code
 | Two sources for the current plan
 | Current plan = the tenant's one active subscription
 | 
15
 | Plan.limits and Plan.features as lists
 | Unqueryable, unvalidated
 | Normalised into feature and plan_feature
 | 
16
 | TenantProvisioningRun and TenantMigrationRun
 | Two job logs with the same lifecycle
 | One tenant_job table with kind
 | 
17
 | MfaMethod and Passkey as separate tables
 | Both are NIST "authenticators" with the same lifecycle
 | One user_authenticator table (Keycloak's credential model); backup codes as its child rows
 | 
18
 | Subdomains stored in TenantDomain
 | <slug>.colsynq.app is derivable from the slug
 | TenantDomain holds verified custom domains only
 | 
19
 | Permission.id plus unique slug
 | Two keys for one catalogue entry
 | slug is the primary key
 | 
20
 | Table name user
 | Reserved word in PostgreSQL
 | Table iam.app_user (entity still User)
 | 
21
 | citext, native enum types, varchar(n)
 | PostgreSQL 18 docs prefer nondeterministic collations or normalised text over citext; native enums are hard to change
 | Normalised text with plain unique indexes; text + CHECK mapped to PHP enums (section 7)
 | 
22
 | Duplicate sentence about Redis; "still to update" notes
 | Stale text
 | Removed
 | 
2. Key decisions
Topic
 | Decision
 | 
Planes
 | Landlord database (control plane): Tenancy and IAM. Tenant databases (data plane): business modules (Multi-tenancy tab)
 | 
Modules
 | Tenancy (tenancy schema) and IAM (iam schema). They exchange events; IAM reads tenant facts it needs from the resolved TenantContext, never from Tenancy's tables
 | 
Membership of a society
 | UserProfile (user_id, tenant_id), unique per pair, created automatically at sign-up
 | 
Platform staff
 | Profiles in the platform tenant; no nullable tenant anywhere
 | 
Roles
 | System roles shared (tenant_id null), custom roles per tenant; uniqueness with NULLS NOT DISTINCT
 | 
Permissions
 | One global catalogue keyed by slug, generated from code
 | 
Assignment authority
 | UserRoleAssignRole edges with allow/deny, plus the can_assign_all wildcard for tenant administrators
 | 
History
 | Rows keep current state and timestamps; who, why and before/after go to audit_event
 | 
Tokens
 | No roles or tenant in the JWT; access loaded per request, cached by access_version (Authentication tab)
 | 
Record-level rules
 | Assertions registered by the owning module (section 14)
 | 
3. Modules
graph TD
    subgraph Landlord [Landlord database: control plane]
        TEN[Tenancy: tenant, tenant_database, tenant_domain, plan, feature, plan_feature, subscription, tenant_entitlement_override, tenant_job, Tenant Resolver]
        IAM[IAM: app_user, user_profile, user_authenticator, backup_code, external_identity, refresh_token, api_key, one_time_token, permission, role, role_permission, user_role, user_role_assign_role, role_conflict, support_access_grant, scope_reference, audit_event]
    end
    subgraph Tenants [Tenant databases: data plane]
        BIZ[Business modules: units, ventures, requests, inventory, ledger ...]
    end
    TEN -- TenantProvisioned, TenantStatusChanged, TenantPlanChanged --> IAM
    IAM -- UserProfileCreated, UserProfileUpdated --> BIZ
    BIZ -- UnitActivated, UnitClosed, VentureActivated, VentureClosed --> IAM
Module
 | Schema
 | Owns
 | 
Tenancy
 | tenancy
 | Tenant registry and sign-up policy, tenant databases, custom domains, plans, features, subscriptions, entitlement overrides, tenant jobs, the Tenant Resolver Service
 | 
IAM
 | iam
 | Identity (users, profiles, authenticators, external identities, tokens, API keys) and access (permissions, roles, grants, assignment rules, conflicts, support grants, audit)
 | 
Inside IAM the code keeps two namespaces, Iam\Identity and Iam\Access, for readability; they share one schema and one transaction. Business modules never import IAM; they name permission slugs and register assertions through shared-kernel interfaces (PermissionProvider, AccessAssertion).
4. Entity–relationship model
erDiagram
    TENANT ||--o| TENANT_DATABASE : "stored in (society tenants)"
    TENANT ||--o{ TENANT_DOMAIN : "custom domains"
    TENANT ||--o{ SUBSCRIPTION : "subscribes"
    PLAN ||--o{ SUBSCRIPTION : "chosen in"
    PLAN ||--o{ PLAN_FEATURE : "includes"
    FEATURE ||--o{ PLAN_FEATURE : "in"
    TENANT ||--o{ TENANT_ENTITLEMENT_OVERRIDE : "overrides"
    TENANT ||--o{ TENANT_JOB : "runs"
    APP_USER ||--o{ USER_PROFILE : "one per tenant"
    APP_USER ||--o{ USER_AUTHENTICATOR : "proves identity with"
    USER_AUTHENTICATOR ||--o{ BACKUP_CODE : "holds"
    APP_USER ||--o{ EXTERNAL_IDENTITY : "federates"
    APP_USER ||--o{ REFRESH_TOKEN : "sessions"
    APP_USER ||--o{ ONE_TIME_TOKEN : "links"
    USER_PROFILE ||--o{ API_KEY : "service accounts"
    USER_PROFILE ||--o{ USER_ROLE : "holds"
    ROLE ||--o{ USER_ROLE : "granted as"
    ROLE ||--o{ ROLE_PERMISSION : "includes"
    PERMISSION ||--o{ ROLE_PERMISSION : "is in"
    ROLE ||--o{ USER_ROLE_ASSIGN_ROLE : "may assign"
    ROLE ||--o{ ROLE_CONFLICT : "conflicts with"
    USER_PROFILE ||--o{ SUPPORT_ACCESS_GRANT : "support access"
Foreign keys are real inside a schema. Links from iam to tenancy (tenant_id) are logical UUIDs, as the two modules are independent; tenant_id values are validated against TenantProvisionedV1 and the resolved TenantContext.
5. PostgreSQL conventions
Topic
 | Standard
 | 
Names
 | snake_case, singular tables, schema per module; constraints `{table}{columns}{pk
 | 
Primary keys
 | UUID v7 generated in the application (Uuid::v7()); DEFAULT uuidv7() once on PostgreSQL 18
 | 
Time
 | timestamptz everywhere; UTC in the application
 | 
Case-insensitive values
 | Store normalised text (email lower-cased and NFC, slug lower-case) with a plain unique index; no citext
 | 
Fixed value sets
 | text + CHECK (col IN (…)), mapped to PHP backed enums; no native ENUM types
 | 
Free text
 | text; length limits as CHECK (char_length(col) <= n) only where a rule needs them
 | 
Nullable tenant (system rows)
 | UNIQUE NULLS NOT DISTINCT (PostgreSQL 15+) in raw-SQL migrations
 | 
"One active row"
 | Partial unique indexes (WHERE revoked_at IS NULL, WHERE status = 'active')
 | 
Foreign keys
 | Every referencing column indexed (PostgreSQL does not do it automatically)
 | 
jsonb
 | Only for variable or opaque data (metadata, public_data, audit payloads); never for keys, filters or unique values
 | 
Secrets
 | Hashes or encrypted bytes (bytea); never plaintext; references to vault paths for database credentials
 | 
Audit
 | Append-only, monthly partitions, no UPDATE or DELETE for the application role
 | 
Money
 | Not stored here
 | 
6. Tenancy tables (tenancy schema)
6.1 tenant
Column
 | Type
 | Constraints and notes
 | 
id
 | uuid
 | PK, v7
 | 
kind
 | text
 | platform or society; partial unique index: exactly one platform row
 | 
slug
 | text
 | Unique; lower-case, ^[a-z0-9][a-z0-9-]{1,62}$; sent in X-Tenant, used as subdomain
 | 
legal_name, short_name
 | text
 |  | 
registration_number, registering_authority, state
 | text
 | Cooperative registration details; null for the platform tenant
 | 
status
 | text
 | provisioning, active, suspended, archived, deleting, deleted, failed
 | 
status_reason
 | text
 | Required for suspended, archived, failed (CHECK)
 | 
status_changed_at
 | timestamptz
 |  | 
activated_at
 | timestamptz
 | First activation
 | 
signup_mode
 | text
 | open, email_domain, closed (default)
 | 
signup_email_domains
 | text[]
 | Non-empty only when signup_mode = email_domain (CHECK)
 | 
signup_default_role_code
 | text
 | Default SELF_SERVICE; validated by IAM when the policy changes
 | 
require_email_verification
 | boolean
 | Default true
 | 
region, timezone, currency
 | text
 | ng, Africa/Lagos, NGN
 | 
logo_url, primary_colour
 | text
 | Public branding
 | 
created_by_profile_id
 | uuid
 | A platform-tenant profile (logical link)
 | 
created_at, updated_at
 | timestamptz
 |  | 
6.2 tenant_database (one per society tenant)
Kept as its own 1:0..1 table because it holds credential references under a stricter grant, exists only for society tenants, and is written by provisioning and migration jobs far more often than tenant.
Column
 | Type
 | Constraints and notes
 | 
tenant_id
 | uuid
 | PK, FK tenant
 | 
db_host, db_port
 | text, integer
 | Database server
 | 
db_name
 | text
 | Unique; generated
 | 
db_role
 | text
 | Per-tenant runtime role
 | 
secret_ref
 | text
 | Vault path; never the password
 | 
schema_version
 | text
 | Cached from the tenant database's doctrine_migration_versions (the source of truth)
 | 
schema_status
 | text
 | ok, migrating, failed, drift
 | 
migration_wave
 | smallint
 | 0 canary to 3 last
 | 
provisioned_at
 | timestamptz
 |  | 
6.3 Other tenancy tables
Table
 | Columns
 | Constraints and notes
 | 
tenant_domain
 | host PK, tenant_id FK, is_primary, verification_token_hash, verified_at
 | Custom domains only; partial unique (tenant_id) WHERE is_primary
 | 
feature
 | code PK, name, description
 | Plan features (procurement, ventures, …)
 | 
plan
 | id PK, code, version, name, is_active
 | Unique (code, version); a price or feature change makes a new version
 | 
plan_feature
 | plan_id FK, feature_code FK, limit_value (integer, nullable)
 | PK (plan_id, feature_code); null limit = unlimited
 | 
subscription
 | id, tenant_id FK, plan_id FK, status (trialing, active, past_due, cancelled, expired), starts_at, ends_at
 | Partial unique (tenant_id) WHERE status IN ('trialing','active','past_due'): one current subscription
 | 
tenant_entitlement_override
 | tenant_id FK, feature_code FK, is_enabled, limit_value, expires_at, reason
 | PK (tenant_id, feature_code); one-off deals
 | 
tenant_job
 | id, tenant_id FK, kind (provision, migrate, move, archive, delete), rollout_id, step, status (queued, running, succeeded, failed), attempt, from_version, to_version, details jsonb, error, started_at, finished_at
 | Replaces provisioning and migration run tables; partial unique (tenant_id) WHERE status = 'running' acts as a per-tenant lock; index (rollout_id)
 | 
Usage counters (users, units, storage) are kept by the modules that own those things and checked by PlanFeatureAssertion; they are not stored here.
7. IAM tables: identity
7.1 app_user (entity User)
Column
 | Type
 | Constraints and notes
 | 
id
 | uuid
 | PK, v7; JWT sub
 | 
kind
 | text
 | person or service
 | 
email
 | text
 | Normalised (lower-case, NFC); unique where not null; required for person (CHECK)
 | 
phone
 | text
 | E.164; unique where not null
 | 
password_hash
 | text
 | Argon2id; null for service users and passkey-only users
 | 
status
 | text
 | active or deactivated
 | 
token_version
 | integer
 | JWT ver; bumped to sign out everywhere
 | 
failed_login_count
 | integer
 | Reset on success
 | 
locked_until
 | timestamptz
 | Temporary lock; "locked" = locked_until > now()
 | 
email_verified_at, phone_verified_at
 | timestamptz
 | Null = unverified
 | 
trusted_device_version
 | integer
 | Bumped to forget all trusted devices (scheb)
 | 
last_login_at, created_at, updated_at
 | timestamptz
 |  | 
Derived, not stored: MFA enabled (an active TOTP or passkey authenticator exists), platform staff (an active profile in the platform tenant), locked, verified.
7.2 user_profile (entity UserProfile)
Column
 | Type
 | Constraints and notes
 | 
id
 | uuid
 | PK, v7
 | 
user_id
 | uuid
 | FK app_user, NOT NULL
 | 
tenant_id
 | uuid
 | NOT NULL (platform staff use the platform tenant)
 | 
first_name, last_name
 | text
 | As known in this society
 | 
status
 | text
 | active, disabled, removed
 | 
status_reason, status_changed_at
 | text, timestamptz
 |  | 
source
 | text
 | self_signup, admin_created, tenant_bootstrap, join_link
 | 
access_version
 | integer
 | Bumped in the same transaction as any change to this profile's grants or status
 | 
created_by_profile_id
 | uuid
 | FK user_profile; null for self sign-up
 | 
metadata
 | jsonb
 | Any extra information a society wants to collect about its users (staff number, branch, next of kin, …); free-form, not used for sign-in, access or uniqueness
 | 
created_at, updated_at
 | timestamptz
 |  | 
Constraints: unique (user_id, tenant_id); unique (id, tenant_id) so grants can reference both; index (tenant_id, status).
7.3 Credentials and sessions
Table
 | Columns
 | Constraints and notes
 | 
user_authenticator
 | id, user_id FK, type (totp, email_otp, passkey, backup_codes), label, credential_id (bytea, passkeys), public_data jsonb (passkey public key, transports, sign count, AAGUID), secret_encrypted bytea (TOTP seed), confirmed_at, last_used_at, revoked_at, created_at
 | One table for NIST authenticators (Keycloak credential model); unique (credential_id) where not null; index (user_id) WHERE revoked_at IS NULL
 | 
backup_code
 | authenticator_id FK, code_hash, used_at
 | Child rows of a backup_codes authenticator, one row per code
 | 
external_identity
 | id, user_id FK, provider, subject, email, email_verified, linked_at, last_used_at, unlinked_at
 | Unique (provider, subject)
 | 
refresh_token
 | gesdinet columns: hashed token, username (= user ID), valid, family, family_valid
 | Index (family); per User, not per profile
 | 
api_key
 | id, user_profile_id, tenant_id, name, prefix, lookup_hash, ip_allow_list cidr[], expires_at, last_used_at, revoked_at, created_at
 | FK (user_profile_id, tenant_id) → user_profile(id, tenant_id); unique lookup_hash; the profile's user must be kind = service
 | 
one_time_token
 | id, purpose (password_reset, set_password), selector, verifier_hash, user_id FK, expires_at, used_at, created_at
 | Unique selector; mapped to the reset-password bundle's request interface (verify)
 | 
Not stored: email-verification and join links (stateless signed URLs), the JWT blocklist and refresh-reuse cache (Redis).
api_key now hangs off the service account's profile (with its tenant enforced by the composite foreign key) instead of carrying a separate user_id and tenant_id that could disagree.
8. IAM tables: access
8.1 permission (catalogue)
Column
 | Type
 | Constraints and notes
 | 
slug
 | text
 | PK; resource:action, e.g. purchase-orders:create
 | 
resource, action
 | text
 | CHECK slug = resource || ':' || action
 | 
module
 | text
 | Owning module
 | 
name, description
 | text
 | For the role editor
 | 
risk
 | text
 | low, high, money
 | 
allowed_scopes
 | text[]
 | Subset of the role scopes (section 9); a platform-only permission has {PLATFORM}
 | 
feature_code
 | text
 | Plan feature, nullable (logical link to tenancy.feature)
 | 
deprecated_at
 | timestamptz
 | Removed from code; kept one release
 | 
Global; written only by iam:catalogue:sync from code.
8.2 role (catalogue)
Column
 | Type
 | Constraints and notes
 | 
id
 | uuid
 | PK, v7
 | 
tenant_id
 | uuid
 | Null = system role shared by all tenants; otherwise a custom role of that tenant
 | 
code
 | text
 | UNIQUE NULLS NOT DISTINCT (tenant_id, code)
 | 
name, description
 | text
 |  | 
scope
 | text
 | PLATFORM, TENANT, BUSINESS_UNIT, VENTURE, SELF; CHECK scope <> 'PLATFORM' OR tenant_id IS NULL
 | 
can_assign_all
 | boolean
 | Wildcard for assignment authority within the tenant
 | 
based_on_role_id
 | uuid
 | FK role; a custom role copied from a system role
 | 
archived_at
 | timestamptz
 | Archived roles cannot be granted
 | 
created_at, updated_at
 | timestamptz
 |  | 
System role = tenant_id IS NULL; platform role = scope = 'PLATFORM'. No flags repeat these facts.
8.3 role_permission
Column
 | Type
 | Constraints and notes
 | 
role_id
 | uuid
 | FK role ON DELETE CASCADE
 | 
permission_slug
 | text
 | FK permission
 | 
created_at
 | timestamptz
 |  | 
PK (role_id, permission_slug); index (permission_slug). A trigger or the application checks that the permission's allowed_scopes contains the role's scope.
8.4 user_role (granting a role to a profile)
Column
 | Type
 | Constraints and notes
 | 
id
 | uuid
 | PK, v7
 | 
user_profile_id
 | uuid
 | NOT NULL
 | 
tenant_id
 | uuid
 | NOT NULL; with user_profile_id forms FK (user_profile_id, tenant_id) → user_profile(id, tenant_id), so the grant's tenant can never disagree with the profile's
 | 
role_id
 | uuid
 | FK role; the role is a system role or a custom role of the same tenant (checked on insert)
 | 
scope_ref_id
 | uuid
 | Which unit or venture; the user ID for SELF; null for TENANT and PLATFORM (CHECK against the role's scope on insert)
 | 
granted_at
 | timestamptz
 |  | 
expires_at
 | timestamptz
 | Required for time-limited roles
 | 
revoked_at
 | timestamptz
 | Rows are never deleted; who and why are in audit_event
 | 
Indexes: UNIQUE NULLS NOT DISTINCT (user_profile_id, role_id, scope_ref_id) WHERE revoked_at IS NULL (one active grant); (tenant_id, role_id) WHERE revoked_at IS NULL (two-administrator count); (role_id). tenant_id is the only copied column, kept because it backs the composite foreign key and the per-tenant indexes.
8.5 user_role_assign_role (who may assign which role)
Column
 | Type
 | Constraints and notes
 | 
id
 | uuid
 | PK, v7
 | 
tenant_id
 | uuid
 | Null = default edge for every tenant; set = that tenant's edge
 | 
assigner_role_id
 | uuid
 | FK role (role A)
 | 
assignable_role_id
 | uuid
 | FK role (role B)
 | 
effect
 | text
 | allow or deny; a tenant deny row switches off a default allow edge
 | 
created_at
 | timestamptz
 |  | 
UNIQUE NULLS NOT DISTINCT (tenant_id, assigner_role_id, assignable_role_id); CHECK assigner_role_id <> assignable_role_id. Resolution: a tenant row wins over the default row for the same pair; holders of a can_assign_all role may assign any non-platform role in their tenant.
8.6 Other access tables
Table
 | Columns
 | Constraints and notes
 | 
role_conflict
 | id, tenant_id (null = default), role_a_id, role_b_id, same_scope_only, severity (block, warn, off)
 | CHECK role_a_id < role_b_id (no mirrored duplicates); UNIQUE NULLS NOT DISTINCT (tenant_id, role_a_id, role_b_id); a tenant row overrides the default
 | 
support_access_grant
 | id, tenant_id, support_profile_id FK (a platform-tenant profile), granted_by_profile_id FK, mode (read_only, impersonate), reason, ticket_ref, starts_at, expires_at, revoked_at
 | CHECK expires_at <= starts_at + interval '4 hours'
 | 
scope_reference
 | tenant_id, scope_ref_id, kind (business_unit, venture), name, status, updated_at
 | PK (tenant_id, scope_ref_id); the only projection left: unit and venture IDs live in tenant databases, so IAM keeps this copy fed by their events
 | 
audit_event
 | id, occurred_at, tenant_id, actor_user_id, actor_profile_id, action, subject_type, subject_id, payload jsonb (before, after, reason), ip, user_agent, correlation_id
 | Append-only; PARTITION BY RANGE (occurred_at) monthly; no FKs; UPDATE and DELETE revoked from the application role; indexes (tenant_id, occurred_at), (subject_type, subject_id)
 | 
9. Role scopes
Scope
 | Rank
 | scope_ref_id
 | Applies to
 | Example roles
 | 
PLATFORM
 | −1
 | null
 | Colsynq; grantable only in the platform tenant
 | Platform Owner, Platform Admin, Support Agent
 | 
TENANT
 | 0
 | null
 | The whole society
 | Tenant Administrator, Finance Officer, Accountant, Auditor, Viewer
 | 
BUSINESS_UNIT
 | 1
 | unit ID
 | One business unit
 | Unit Manager, Storekeeper, Cashier, Procurement Officer
 | 
VENTURE
 | 1
 | venture ID
 | One venture
 | Venture Manager, Venture Accountant
 | 
SELF
 | 2
 | the user's ID
 | Records owned by the user
 | Self-service User
 | 
Grants combine by union, per permission. BUSINESS_UNIT and VENTURE grants with several references give a set of IDs.
10. Permission catalogue
Slugs are resource:action; actions are list, read, create, update, delete, execute, export, manage. Approval permissions will come from the approval module.
#[AutoconfigureTag('colsynq.permission_provider')]
final class ProcurementPermissions implements PermissionProvider   // example module
{
    public function permissions(): array
    {
        return [
            PermissionDefinition::of('purchase-orders', ['list', 'read', 'create', 'update'], risk: 'low',
                scopes: [RoleScope::Tenant, RoleScope::BusinessUnit], feature: 'procurement'),
            PermissionDefinition::of('supplier-invoices', ['execute'], risk: 'money',
                scopes: [RoleScope::Tenant], feature: 'procurement'),
        ];
    }
}
iam:catalogue:sync on every deploy: upsert permission rows, mark missing slugs deprecated, apply SystemRoleCatalogue (system roles, their permissions, default assignment edges and conflicts), never add high or money permissions to a role automatically, remove deprecated slugs one release later with an audit_event, and bump a global catalogue_version used in the access cache key.
11. System roles and defaults
Role code
 | Scope
 | Main permissions
 | Notes
 | 
TENANT_ADMIN
 | TENANT
 | Profiles, roles, role permissions, user roles, assignment edges, conflicts, tenant settings and sign-up policy, support grants; read of all modules
 | can_assign_all; no money permissions
 | 
FINANCE_OFFICER
 | TENANT
 | money execute permissions; read ledger and reconciliation
 | Conflicts with Auditor
 | 
ACCOUNTANT
 | TENANT
 | Journals, statements, reconciliation; read finance
 |  | 
AUDITOR
 | TENANT
 | Read and export everything, including audit_event
 | No create, update, delete or execute
 | 
EXTERNAL_AUDITOR
 | TENANT
 | Read and export finance
 | expires_at required
 | 
VIEWER
 | TENANT
 | Read non-sensitive resources
 |  | 
UNIT_MANAGER
 | BUSINESS_UNIT
 | Unit operations; may assign unit roles in own unit
 |  | 
STOREKEEPER
 | BUSINESS_UNIT
 | Goods received, stock movements and counts, deliveries
 |  | 
CASHIER
 | BUSINESS_UNIT
 | Sales, refunds, cash-ups, petty cash
 |  | 
PROCUREMENT_OFFICER
 | BUSINESS_UNIT
 | Suppliers, quotations, purchase orders
 |  | 
VENTURE_MANAGER
 | VENTURE
 | Venture operations and KPIs; may assign venture roles in own venture
 |  | 
VENTURE_ACCOUNTANT
 | VENTURE
 | Venture journals and returns
 |  | 
SELF_SERVICE
 | SELF
 | me-*; own records where a module offers self-service
 | Default sign-up role
 | 
PLATFORM_OWNER
 | PLATFORM
 | All platform permissions
 | At least two holders
 | 
PLATFORM_ADMIN
 | PLATFORM
 | tenants:*, plans:*, platform profiles
 |  | 
SUPPORT_AGENT
 | PLATFORM
 | Platform read; tenant data only under a support grant
 |  | 
COMPLIANCE_OFFICER
 | PLATFORM
 | Tenant registration details, platform audit
 |  | 
FINANCE_OPERATIONS
 | PLATFORM
 | Plans, subscriptions, billing
 |  | 
Default assignment edges (allow, tenant_id null): UNIT_MANAGER → STOREKEEPER, CASHIER, PROCUREMENT_OFFICER, SELF_SERVICE; VENTURE_MANAGER → VENTURE_ACCOUNTANT, SELF_SERVICE; PLATFORM_OWNER → every platform role; PLATFORM_ADMIN → SUPPORT_AGENT, COMPLIANCE_OFFICER, FINANCE_OPERATIONS.
Default conflicts (tenant_id null):
Role A
 | Role B
 | Same scope only
 | Severity
 | 
FINANCE_OFFICER
 | AUDITOR
 | no
 | block
 | 
FINANCE_OFFICER
 | EXTERNAL_AUDITOR
 | no
 | block
 | 
TENANT_ADMIN
 | FINANCE_OFFICER
 | no
 | warn
 | 
CASHIER
 | ACCOUNTANT
 | no
 | block
 | 
CASHIER
 | VENTURE_ACCOUNTANT
 | no
 | block
 | 
PROCUREMENT_OFFICER
 | STOREKEEPER
 | yes
 | block
 | 
Tenants create custom roles (for example President, Treasurer) by copying a system role (based_on_role_id).
12. Tenancy lifecycle and the Tenant Resolver
12.1 Creating a society tenant
sequenceDiagram
    autonumber
    participant PA as Platform Admin (platform tenant profile)
    participant TEN as Tenancy
    participant JOB as tenant_job workers
    participant IAM as IAM
    participant A as First administrators
    PA->>TEN: POST /api/platform/tenants (details, plan, sign-up policy, two admin emails)
    TEN->>TEN: tenant (provisioning), tenant_database, subscription, tenant_job (provision)
    TEN->>JOB: provision: role + database from template, migrate, seed
    JOB->>TEN: tenant active
    TEN-->>IAM: TenantProvisionedV1 (first administrators)
    IAM->>IAM: RegisterUserInTenant x2: app_user (new or existing) + user_profile + user_role TENANT_ADMIN, one transaction
    IAM-->>A: set-password link (one_time_token) or notice
Status
 | Sign-in
 | Reads
 | Writes
 | Money actions
 | 
provisioning, failed
 | No (503)
 | No
 | No
 | No
 | 
active
 | Yes
 | Yes
 | Yes
 | Yes
 | 
suspended
 | Yes
 | Yes
 | No, except paying the subscription
 | No
 | 
archived
 | Administrators and auditors
 | Read-only
 | No
 | No
 | 
deleting, deleted
 | No (404)
 | No
 | No
 | No
 | 
12.2 Tenant Resolver Service
Follows REPROMAS (TenantResolveListener, CurrentTenantProvider, stamp and context middleware, TenantScopedRunner, tenant validate endpoint).
Sources, in order: X-Tenant header (slug), then host (verified custom domain, then <slug>.colsynq.app). Header and host naming different tenants → 400 tenant_mismatch.
Route requirement: #[TenantRequirement(Required | Optional | None)], default Required; a configured bypass list covers routes without attributes.
Listener: kernel.request priority 512, before the router (32) and firewall (8); skips OPTIONS.
Result: an immutable ResolvedTenant (id, kind, slug, status, sign-up policy, plan features) in TenantContext. IAM reads the sign-up policy and plan features from it, so it never queries Tenancy's tables.
Errors: 400 tenant_required, 400 tenant_mismatch, 404 tenant_not_found, 503 tenant_unavailable.
Workers and webhooks: TenantStampMiddleware, TenantContextMiddleware (reset in finally), TenantScopedRunner.
Public lookup: GET /api/tenants/resolve?slug= returns name, logo, colours, status and signup_mode.
#[AsEventListener(event: KernelEvents::REQUEST, priority: 512)]
final readonly class TenantResolveListener
{
    public function __invoke(RequestEvent $event): void
    {
        $request = $event->getRequest();
        if (!$event->isMainRequest() || $request->isMethod('OPTIONS')) {
            return;
        }
        $requirement = $this->requirements->for($request);
        if (TenantRequirement::None === $requirement) {
            return;
        }
        $tenant = $this->resolver->resolve($request);   // header, then host; cached registry; may throw TenantMismatch
        if (null === $tenant) {
            if (TenantRequirement::Required === $requirement) {
                throw new TenantRequired();
            }
            return;
        }
        $tenant->assertReachable();                      // 503 provisioning/failed, 404 deleting/deleted
        $this->context->switchTo($tenant);               // refuses a second tenant; switches the tenant database for society tenants
    }
}
13. Sign-up and profile lifecycle
Every way a User enters a tenant goes through RegisterUserInTenant, whose tenant comes from TenantContext, never from the request body. In one transaction it creates or reuses the app_user, creates the user_profile, and grants the initial or default role through the gates (section 15), then writes audit_event rows and publishes events.
Path
 | Endpoint
 | Result
 | 
Self sign-up
 | POST /api/auth/register (tenant required)
 | Allowed if signup_mode is open, or email_domain with a matching verified domain. New email: app_user + user_profile (self_signup) + default role; verification email. Existing email: nothing created, the owner gets a join link. Always the same 202 response
 | 
Join with an existing account
 | POST /api/tenants/{slug}/join (join-link token) or POST /api/me/profiles on an open tenant
 | user_profile (join_link) + default role
 | 
Created by an administrator
 | POST /api/profiles (profiles:create)
 | New email: app_user without password + profile (admin_created) + optional initial role + set-password link. Existing email: profile + notice
 | 
First tenant administrators
 | Section 12.1
 | Profile (tenant_bootstrap) + TENANT_ADMIN
 | 
Service account
 | POST /api/service-accounts
 | app_user (kind = service) + its only profile + first api_key
 | 
Platform staff
 | POST /api/platform/profiles
 | Profile in the platform tenant + platform role
 | 
Sign-in at a tenant requires an active profile there (ProfileLoginValidator, generic error otherwise); sign-in without a tenant ends at the society switcher (GET /api/me/profiles). The login response body adds the profile and access for the UI; the JWT stays tenant-free.
Profile change
 | Effect
 | Tokens
 | 
Disable
 | status = disabled; grants kept but ignored; access_version bumped
 | Unchanged; the next request to that tenant gets 404
 | 
Enable
 | status = active; access_version bumped
 | Unchanged
 | 
Remove
 | status = removed; active grants revoked; a service account's keys revoked
 | Unchanged for persons
 | 
Deactivate the User
 | app_user.status = deactivated; token_version bumped; all grants of all profiles revoked
 | Refresh tokens revoked
 | 
The two-administrator rule (section 15.3) is checked in the same transaction for disable, remove and deactivate, which is possible because profiles and grants are in one module.
14. Request pipeline and assertions
graph TD
    REQ[Bearer token or ApiKey + society host or X-Tenant] --> TR[1. Tenant Resolver, 512]
    TR --> FW[2. Firewall, 8: JWT or API key, load app_user]
    FW --> PR[3. Active user_profile for user + tenant, else 404]
    PR --> AC[4. EffectiveAccess: user_role, role, role_permission; cached by tenant, profile, access_version, catalogue_version]
    AC --> SF[5. Scope filters on tenant entity manager]
    SF --> V[6. AccessVoter: permission, scope, assertions]
    V --> H[Handler or 403 with reason code]
The access query reads only iam tables: active, unexpired grants of non-archived roles for the profile, joined to their permissions.
Assertions decide whether a permitted action is allowed for this record, in this state, now. The owning module registers them (AccessAssertion, tag colsynq.access_assertion); they are read-only, fast and deterministic. Order: permission held → scope covers the subject → every assertion for the slug; any Deny denies, with its reason code in the 403.
Assertion
 | Owner
 | Rule
 | 
TenantWritableAssertion
 | Tenancy
 | Deny writes when the tenant is suspended or archived
 | 
PlanFeatureAssertion
 | Tenancy
 | Deny permissions whose feature_code is not in the plan or overrides
 | 
OwnRecordAssertion
 | IAM
 | SELF grants cover only records the user owns
 | 
NotSelfAssertion
 | IAM
 | Nobody changes their own grants
 | 
StepUpAssertion
 | IAM
 | money and high permissions need MFA and recent re-authentication
 | 
DraftOnlyAssertion (example)
 | Procurement
 | Purchase orders change only while draft
 | 
(later) approval state
 | Approval module
 | Deny until the workflow completes
 | 
15. Granting and revoking roles
15.1 Five gates (UserRoleAssignmentPolicy)
Own profile: nobody grants or revokes their own roles.
Authority: an allow edge in user_role_assign_role from one of the caller's active roles to the target role (tenant row first, then the default), unless the caller's role has can_assign_all. Platform roles only from platform roles, only in the platform tenant.
Scope containment: unit callers assign only with their own unit IDs, venture callers only their venture; TENANT roles only by can_assign_all holders; the reference must be active in scope_reference.
Target: the profile is active and in the caller's tenant (enforced by the composite foreign key); the role is a system role or the tenant's own; time-limited roles have expires_at.
Conflicts: no block row in role_conflict (tenant row first, then default) with the profile's active roles; warn needs a reason, recorded in audit_event.
Revoking uses gates 0–2, sets revoked_at, bumps access_version, writes audit_event, publishes UserRoleRevokedV1. Changing a custom role's permissions re-runs gate 4 for its holders.
15.2 Taking effect
Tokens carry no roles, so a grant or revocation applies on the next request through access_version.
15.3 Two-administrator rule
Each society keeps at least two active profiles with TENANT_ADMIN; the platform tenant keeps two PLATFORM_OWNER. Checked with the (tenant_id, role_id) WHERE revoked_at IS NULL index, under a row lock, for revoke, disable, remove and deactivate; otherwise 409 last_administrators.
16. Support access
A society administrator creates a support_access_grant for a named platform-tenant profile (reason, ticket, read_only or impersonate, at most 4 hours). Emergency access without a grant: PLATFORM_OWNER only, MFA and passkey, read-only, 1 hour, administrators notified. Impersonation follows the Authentication tab (no money actions, no access changes, no exports, both identities in audit_event).
17. Code layout
src/
├── Tenancy/
│   ├── Domain/        Tenant, TenantDatabase, TenantDomain, Plan, Feature, Subscription, TenantJob
│   ├── Application/   CreateTenant, ChangeTenantStatus, ChangePlan, UpdateSignupPolicy, provisioning and migration jobs,
│   │                  TenantWritableAssertion, PlanFeatureAssertion
│   ├── Contracts/     TenantProvisionedV1, TenantStatusChangedV1, TenantPlanChangedV1 ...
│   └── Infrastructure/ TenantResolveListener, TenantResolver, TenantContext, DBAL middleware, messenger middleware
├── Iam/
│   ├── Domain/Identity/  User, UserProfile, UserAuthenticator, ExternalIdentity, ApiKey
│   ├── Domain/Access/    Permission, Role, UserRole, UserRoleAssignRole, RoleConflict, SupportAccessGrant,
│   │                     UserRoleAssignmentPolicy, TwoAdministratorRule, RoleScope
│   ├── Application/      RegisterUserInTenant, profile and grant commands, EffectiveAccessLoader, AccessDecision,
│   │                     catalogue sync, scope_reference projection, StepUpAssertion
│   ├── Contracts/        UserRegisteredV1, UserProfileCreatedV1, UserRoleGrantedV1 ...
│   └── Infrastructure/   SecurityUser, user provider, ProfileLoginValidator, JWT and API key handlers, AccessVoter,
│                         scope filters, assertion registry, audit writer, API resources
└── Shared/Access/        PermissionProvider, PermissionDefinition, AccessAssertion, AssertionResult, RoleScope, ScopedBy
18. API endpoints
Area
 | Endpoints
 | Permission or access
 | 
Tenant lookup
 | GET /api/tenants/resolve?slug=
 | Public, no tenant
 | 
Sign-up and join
 | POST /api/auth/register; POST /api/tenants/{slug}/join; POST /api/me/profiles
 | Public (tenant required) / authenticated
 | 
Authentication
 | POST /api/auth, /api/auth/mfa, /api/token/refresh, /api/logout, /api/password-reset, /api/verify-email; passkey routes
 | Authentication tab
 | 
Me
 | GET/PATCH /api/me; GET /api/me/profiles; GET/PATCH /api/me/profile; GET /api/me/access; /api/me/authenticators, /api/me/external-identities, /api/me/sessions
 | Authenticated; step-up for credential changes
 | 
Platform tenants
 | POST/GET /api/platform/tenants; GET/PATCH /api/platform/tenants/{id}; POST /api/platform/tenants/{id}/status; GET /api/platform/tenant-jobs; GET/POST /api/platform/plans
 | tenants:*, tenant-jobs:list, plans:*
 | 
Platform staff
 | POST/GET /api/platform/profiles; /api/platform/profiles/{id}/roles
 | platform-profiles:*, user-roles:*
 | 
Tenant settings
 | GET/PATCH /api/tenant; PATCH /api/tenant/signup-policy; GET/POST/DELETE /api/tenant/domains (+ /verify)
 | tenant:read, tenant:manage, tenant-domains:manage
 | 
Profiles
 | GET/POST /api/profiles; GET/PATCH/DELETE /api/profiles/{id}
 | profiles:*
 | 
Service accounts
 | GET/POST /api/service-accounts; POST/DELETE /api/service-accounts/{profileId}/api-keys[/{keyId}]
 | service-accounts:*, api-keys:* (step-up)
 | 
Roles and permissions
 | GET /api/permissions; GET/POST /api/roles; GET/PATCH/DELETE /api/roles/{id}; PUT /api/roles/{id}/permissions
 | permissions:list, roles:*, role-permissions:manage
 | 
Assignment rules and conflicts
 | GET/POST/DELETE /api/user-role-assign-roles; GET/POST/PATCH /api/role-conflicts
 | user-role-assign-roles:*, role-conflicts:*
 | 
Grants
 | GET/POST /api/profiles/{id}/roles; DELETE /api/profiles/{id}/roles/{userRoleId}
 | user-roles:* (five gates)
 | 
Audit and support
 | GET /api/audit-events; GET/POST/DELETE /api/support-access-grants
 | audit-events:read, support-access-grants:*
 | 
19. Events
Event (V1)
 | Producer
 | Consumers
 | 
TenantCreatedV1, TenantProvisionedV1, TenantStatusChangedV1, TenantPlanChangedV1, TenantSignupPolicyChangedV1
 | Tenancy
 | IAM, business modules, Audit, Notification
 | 
TenantJobFailedV1
 | Tenancy
 | Monitoring
 | 
UserRegisteredV1, UserDeactivatedV1
 | IAM
 | Audit, Notification
 | 
UserProfileCreatedV1, UserProfileUpdatedV1, UserProfileStatusChangedV1
 | IAM
 | Business modules (profile-name projection), Notification
 | 
UserRoleGrantedV1, UserRoleRevokedV1, ProfileAccessChangedV1
 | IAM
 | Notification, Reporting
 | 
RoleChangedV1, PermissionCatalogueChangedV1
 | IAM
 | Monitoring
 | 
Authentication events (UserSignedInV1, SignInFailedV1, SessionsRevokedV1, RefreshTokenReuseDetectedV1, AuthenticatorAddedV1, AuthenticatorRevokedV1, ExternalIdentityLinkedV1, ApiKeyCreatedV1, ApiKeyRevokedV1)
 | IAM
 | Notification, Security monitoring
 | 
SupportAccessGrantedV1, SupportAccessEndedV1
 | IAM
 | Notification
 | 
UnitActivatedV1, UnitClosedV1, VentureActivatedV1, VentureClosedV1
 | Business modules
 | IAM (scope_reference; closing revokes scoped grants)
 | 
Events between IAM's identity and access parts are gone: they share one transaction. Every IAM change also writes audit_event.
20. Errors and caching
Situation
 | HTTP
 | Code
 | 
Tenant missing, mismatched, unknown, unavailable
 | 400 / 400 / 404 / 503
 | tenant_required, tenant_mismatch, tenant_not_found, tenant_unavailable
 | 
No active profile in the tenant
 | 404
 | tenant_not_found
 | 
Tenant read-only
 | 403
 | tenant_read_only
 | 
Missing permission or outside scope
 | 403
 | access_denied
 | 
Assertion denied
 | 403
 | the assertion's reason code
 | 
Gate failure
 | 403
 | user_role_forbidden (gate number)
 | 
Role conflict
 | 409
 | role_conflict
 | 
Last administrators
 | 409
 | last_administrators
 | 
Step-up needed
 | 401
 | step_up_required
 | 
Access cache key: access:{tenant}:{profile}:{access_version}:{catalogue_version}. Expired grants are ignored by the loader. Assertions read live state. All keys are prefixed by tenant.
21. Out of scope for now
Topic
 | Plan
 | 
Approval workflow
 | Its own module; plugs in through assertions and defines its own permissions
 | 
Cooperative membership
 | Members, staff, employers; a profile will link to a member record, and society-specific contact details live there
 | 
Governance
 | Elections, resolutions, access recertification
 | 
Temporal grant validity
 | PostgreSQL 18 WITHOUT OVERLAPS on a tstzrange, only if time-bound access becomes a product feature
 | 
22. Governance tests
Test
 | Asserts
 | 
SchemaConventionTest
 | Singular snake_case tables, no native enums or citext, every FK column indexed, timestamps are timestamptz
 | 
PermissionCatalogCoverageTest
 | Every API operation has a catalogued slug or is public by list
 | 
SystemRoleCatalogueTest
 | System roles use only existing permissions in their scopes; no money on TENANT_ADMIN
 | 
UniquenessTest
 | One profile per user and tenant; one active grant per profile, role and reference; no duplicate system role codes (NULLS NOT DISTINCT); no mirrored conflicts
 | 
CompositeTenantKeyTest
 | A grant or API key can never point at a profile of another tenant
 | 
SignUpTest
 | User and profile created together; tenant from the resolver; existing email gets a join link; policy enforced
 | 
UserRoleAssignmentPolicyTest
 | Each gate, tenant deny overriding a default allow, can_assign_all, platform roles only in the platform tenant
 | 
TwoAdministratorsTest
 | Revoke, disable, remove and deactivate keep two administrators
 | 
AuditTest
 | Every IAM change writes one audit_event; the application role cannot update or delete it
 | 
AssertionEvaluationTest
 | Deny wins; reason codes reach the 403
 | 
CrossTenantTest
 | No access, cache or key leaks between tenants
 | 
NoRolesInTokenTest, RevocationTimingTest, SupportAccessTest
 | As in the Authentication tab
 | 
23. Checklist for new features
New tables follow section 5 (names, UUID v7, timestamptz, text + CHECK, indexed FKs, partial unique indexes)
No fact stored twice; derived values computed, not stored; history in audit_event
Permissions declared in the module's PermissionProvider; every operation has a security expression; collections use scope filters
Record-level rules written as assertions in the owning module
Events listed in section 19 with schemas; governance tests updated
