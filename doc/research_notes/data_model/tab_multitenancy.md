Multi-tenancy: one database per tenant, end to end in Symfony
Stack: PHP 8.4 · Symfony 8.1 · Doctrine DBAL 4.5 / ORM 3.7 / DoctrineBundle 3.3 · doctrine/migrations 3.9 / DoctrineMigrationsBundle 4.0 · API Platform 5 · PostgreSQL 17–18 · Checked: 12 October 2026 against Doctrine and Symfony source, PostgreSQL docs, cloud provider docs, PgBouncer docs, Azure and AWS SaaS guidance, and published engineering write-ups
Model: every society (tenant) gets its own PostgreSQL database. One central landlord database holds the tenant registry, identities and platform data. No society's business data shares a database with another's.
This replaces the shared-database model (society filter plus row-level security) described in the Architecture guide (section 14) and the Authentication tab. Items marked (verify) use syntax not confirmed in a primary source.
1. Decisions in one table
Topic
 | Decision
 | 
Isolation
 | One PostgreSQL database per society, on shared database servers. A server holds a few hundred to about 1,000 tenant databases
 | 
Central data
 | A landlord database: tenant registry, domains, plans and billing, global user identities and memberships, platform audit, relay cursors
 | 
Doctrine
 | Two connections and two entity managers: landlord and tenant. Module entities map to tenant
 | 
Switching
 | A DBAL driver middleware sets the tenant's connection details; switching = clear the entity manager, set details, close the connection. Fails closed when no tenant is set
 | 
Bundle
 | Build our own (about 300 lines). hakam/multi-tenancy-bundle has no Symfony 8 release yet; use it only as a reference
 | 
Tenant creation
 | Asynchronous chain on Messenger: create role and database from a pre-migrated template → migrate → seed → activate. Each step idempotent; failures compensated
 | 
Migrations
 | Two sets: migrations/Landlord (bundle commands) and migrations/Tenant (our tenants:migrate, one DependencyFactory per tenant). Waves, locks, per-tenant results, expand/contract
 | 
Messaging
 | Each tenant database keeps its outboxes and inboxes. A relay forwards outbox rows to a central transport with a tenant stamp. Workers switch to the tenant before handling
 | 
Pooling
 | PgBouncer in transaction mode with small pools per database; migrations and locks use a direct connection
 | 
Keep for later
 | Keep a society_id column on core records, so small tenants could be pooled into shared databases later if needed
 | 
2. Why database per tenant, and its cost
 | Shared tables + row-level security
 | Schema per tenant
 | Database per tenant
 | 
Isolation
 | Logical; a missed filter leaks
 | Better
 | Strong: separate catalogs, credentials and connections
 | 
Restore one tenant
 | Hard
 | Hard
 | Easy: dump or restore one database
 | 
Delete or export one tenant
 | Row-by-row
 | Drop schema
 | Drop database
 | 
Noisy neighbour
 | High
 | High
 | Medium (shared server)
 | 
Schema changes
 | One run
 | N runs
 | N runs (needs fleet tooling)
 | 
Cross-tenant reporting
 | Easy SQL
 | Possible
 | Not in SQL; use a warehouse
 | 
Connections
 | Few
 | Few
 | Many; pooler required
 | 
Tenant count
 | Unlimited
 | Thousands
 | 1 to 100,000s (Azure), with a registry and tooling
 | 
Azure recommends isolating a tenant when it needs its own keys, backup policy or data location. AWS calls this the "bridge" model (own database, shared server). Atlassian runs about 4 million Jira databases on about 3,000 servers, up to about 4,000 per cluster, and had to lower that density when per-database file counts slowed failover. Citus notes schema changes going "from seconds to minutes to hours" across thousands of tenants. The tooling below exists to make that cost manageable.
3. What lives where
graph TD
    subgraph App [Symfony app and workers]
        R[Tenant resolver] --> CTX[Tenant context]
        CTX --> MW[DBAL tenant middleware]
        EML[landlord entity manager]
        EMT[tenant entity manager]
    end
    subgraph Landlord [Landlord database]
        T[(tenant, tenant_domain)]
        U[(users, memberships)]
        Q[(central message transport, failed, scheduler)]
        RUN[(migration runs, relay cursors)]
    end
    subgraph Server1 [DB server 1]
        A[(society_a: module schemas, outbox, inbox)]
        B[(society_b)]
        TPL[(tenant_template)]
    end
    subgraph Server2 [DB server 2]
        C[(society_c)]
        TPL2[(tenant_template)]
    end
    EML --> Landlord
    EMT --> MW --> A
    MW --> B
    MW --> C
Landlord database (control plane)
 | Each tenant database (data plane)
 | 
Tenancy (tenancy schema): tenant (incl. sign-up policy), tenant_database, tenant_domain, plan, feature, plan_feature, subscription, tenant_entitlement_override, tenant_job
 | One PostgreSQL schema per business module (requests, ledger, inventory, …), as in the Architecture guide
 | 
IAM (iam schema): app_user, user_profile (one per user per tenant), authenticators, external identities, refresh tokens, API keys, one-time tokens, permission, role, role_permission, user_role, user_role_assign_role, permission_conflict, platform_tenant, support_access_grant, scope_reference, audit_event
 | Module outbox and inbox tables; doctrine_migration_versions
 | 
Central Messenger transport, failed transport, scheduler state, relay cursors
 | Module projections and process-manager state, including a read-only projection of profile names
 | 
 | Per-tenant reference data (chart of accounts, settings)
 | 
Identity and access stay central: one person signs in once and has one UserProfile per society; sign-in, the society switcher and every access check read only the landlord. Business handlers are the only code that opens a tenant database (User, access control and tenancy tab).
4. The tenant registry
CREATE TABLE tenant (
  id              uuid PRIMARY KEY,                 -- UUID v7
  slug            citext UNIQUE NOT NULL,
  status          text NOT NULL CHECK (status IN
                    ('pending','provisioning_db','migrating','seeding','active',
                     'suspended','failed','deleting','deleted')),
  failed_step     text,
  last_error      text,
  db_host         text NOT NULL,
  db_port         int  NOT NULL DEFAULT 5432,
  db_name         text UNIQUE NOT NULL,             -- generated, e.g. t_0192f3a7c1; never user input
  db_user         text NOT NULL,
  db_secret_ref   text NOT NULL,                    -- vault path or encrypted blob id, never the password
  schema_version  text,
  schema_status   text NOT NULL DEFAULT 'ok',       -- ok | migrating | failed | drift
  migration_wave  smallint NOT NULL DEFAULT 2,      -- 0 canary, 1 small, 2 rest, 3 last
  region          text NOT NULL DEFAULT 'ng',
  plan            text,
  created_at      timestamptz NOT NULL,
  updated_at      timestamptz NOT NULL
);
CREATE TABLE tenant_domain (
  host       citext PRIMARY KEY,
  tenant_id  uuid NOT NULL REFERENCES tenant(id),
  is_primary boolean NOT NULL DEFAULT false
);
Cache lookups (host or slug → connection config) for 60–300 seconds with tag tenant_{id}; invalidate on suspend or update.
Never cache the decrypted database password in a shared cache. Keep it in process memory only, cleared on reset.
5. Doctrine set-up
# config/packages/doctrine.yaml
doctrine:
    dbal:
        default_connection: landlord
        connections:
            landlord:
                url: '%env(resolve:LANDLORD_DATABASE_URL)%'
                server_version: '17'
            tenant:
                # placeholder: points at nothing usable; the middleware supplies real details
                url: '%env(resolve:TENANT_PLACEHOLDER_URL)%'
                server_version: '17'
    orm:
        default_entity_manager: landlord
        entity_managers:
            landlord:
                connection: landlord
                mappings:
                    Landlord:
                        type: xml
                        is_bundle: false
                        dir: '%kernel.project_dir%/src/Landlord/Infrastructure/Persistence/Mapping'
                        prefix: 'App\Landlord\Domain'
            tenant:
                connection: tenant
                second_level_cache: { enabled: false }   # its keys ignore the database: would leak
                mappings:
                    Requests:
                        type: xml
                        is_bundle: false
                        dir: '%kernel.project_dir%/src/Requests/Infrastructure/Persistence/Mapping'
                        prefix: 'App\Requests\Domain\Model'
                    # one entry per module (examples)
Mappings must not overlap. ManagerRegistry::getManagerForClass() then picks the right manager, so API Platform 5 works unchanged.
Metadata and query caches can be shared (same schema everywhere). Result cache and every application cache must be prefixed by tenant.
PHP 8.4 with ORM 3.7 uses native lazy objects: no proxy files to worry about.
6. Switching the connection safely
DBAL 4 made connect() protected and the stored params private, so the old "select database" wrapper trick is gone. The clean way is a driver middleware, scoped to the tenant connection:
#[AsMiddleware(connections: ['tenant'])]
final class TenantDriverMiddleware implements Middleware, ResetInterface
{
    /** @var array{host:string, port:int, dbname:string, user:string, password:string}|null */
    private ?array $override = null;
    public function wrap(Driver $driver): Driver { return new TenantDriver($driver, $this); }
    public function set(?array $params): void { $this->override = $params; }
    public function get(): ?array { return $this->override; }
    public function reset(): void { $this->override = null; }
}
final class TenantDriver extends AbstractDriverMiddleware
{
    public function __construct(Driver $driver, private readonly TenantDriverMiddleware $middleware)
    {
        parent::__construct($driver);
    }
    public function connect(#[\SensitiveParameter] array $params): DriverConnection
    {
        $override = $this->middleware->get();
        if (null === $override) {
            throw new NoTenantSelected();     // fail closed: never fall back to the last tenant
        }
        return parent::connect(array_merge($params, $override));
    }
}
final class TenantContext implements ResetInterface
{
    private ?TenantId $current = null;
    public function __construct(
        private readonly TenantDriverMiddleware $middleware,
        private readonly ManagerRegistry $doctrine,
        private readonly TenantConnectionConfigProvider $configs,
    ) {}
    public function switchTo(TenantId $tenantId): void
    {
        $connection = $this->doctrine->getConnection('tenant');
        if ($connection->isTransactionActive()) {
            throw new \LogicException('Cannot switch tenant inside a transaction.');
        }
        $this->doctrine->getManager('tenant')->clear();   // empty the identity map first
        $this->middleware->set($this->configs->for($tenantId)->toParams());
        $connection->close();                              // next query reconnects to the new database
        $this->current = $tenantId;
    }
    public function current(): TenantId
    {
        return $this->current ?? throw new NoTenantSelected();
    }
    public function reset(): void
    {
        $connection = $this->doctrine->getConnection('tenant');
        if ($connection->isTransactionActive()) {
            $connection->rollBack();
        }
        $connection->close();
        $this->middleware->reset();
        $this->current = null;
    }
}
Doctrine's own reset clears entity managers between requests but does not close connections. TenantContext::reset() does that.
Point TENANT_PLACEHOLDER_URL at a database that does not exist, or a role without CONNECT rights, so an unswitched query fails loudly.
7. Resolving the tenant
The Tenant Resolver Service follows REPROMAS (TenantResolveListener + CurrentTenantProvider) and is specified in full in the User, access control and tenancy tab (section 10.3).
Source (in order)
 | Use
 | 
X-Tenant header with the tenant slug
 | Mobile app, society switcher, integrations, sign-up from a shared app
 | 
Host: verified custom domain, then <slug>.colsynq.app
 | Web apps
 | 
TenantStamp on messages
 | Workers (TenantContextMiddleware)
 | 
TenantScopedRunner::run($tenantId, fn)
 | Webhooks, console commands, scheduler fan-out
 | 
Each route declares #[TenantRequirement(Required | Optional | None)] (default Required), with a configured bypass list for routes without attributes. Header and host naming different tenants → 400 tenant_mismatch.
Avoid path-based tenancy (/t/{slug}/…): IRIs then collide in API Platform HTTP cache and Mercure unless every key is prefixed.
Listener order on kernel.request in Symfony 8.1: idle-connection listener 192, router 32, tenant resolver 20, firewall 8. Running after the router lets the resolver read the matched route's _tenant requirement; running before the firewall means sign-in and sign-up already know the tenant. The profile check runs after the firewall.
#[AsEventListener(event: KernelEvents::REQUEST, priority: 20)]   // after RouterListener (32), before Firewall (8)
final readonly class TenantResolveListener
{
    public function __construct(
        private TenantResolver $resolver,
        private TenantContext $context,
    ) {}
    public function __invoke(RequestEvent $event): void
    {
        $request = $event->getRequest();
        if (!$event->isMainRequest() || $request->isMethod('OPTIONS')) {
            return;
        }
        $requirement = TenantRequirement::from($request->attributes->get('_tenant', 'required'));
        if (TenantRequirement::None === $requirement) {
            return;
        }
        $tenant = TenantRequirement::Platform === $requirement
            ? $this->resolver->platform($request)
            : $this->resolver->resolve($request);      // X-Tenant slug, then host; cached registry
        if (null === $tenant) {
            TenantRequirement::Required === $requirement ? throw new TenantRequired() : null;
            return;
        }
        $tenant->assertReachable();                     // 503 provisioning/failed, 404 deleting/deleted
        $this->context->switchTo($tenant);              // sets current tenant + DB middleware; refuses a second tenant
    }
}
Users are global, so the firewall can authenticate after the tenant is resolved. After authentication, the profile resolver requires an active UserProfile for the user in this tenant; otherwise 404.
Sign-in at a tenant also requires that profile (ProfileLoginValidator); sign-up in a tenant creates the User and the UserProfile together.
A tenant slug from a client is a request, never proof. The profile decides.
8. No leaks in long-running processes
FrankenPHP worker mode, RoadRunner and messenger:consume keep PHP alive across requests and messages.
Every service holding tenant state implements ResetInterface and fails closed on reset (clear settings, roll back, close the connection).
Messenger middleware order: tenant middleware → doctrine_ping_connection → doctrine_close_connection → doctrine_transaction. Switch in a try, reset in finally.
Do not use Symfony 8.1's messenger:consume --no-reset=N with tenancy.
Prefix every shared store by tenant: cache pools, locks, rate limiters, Messenger deduplication keys, API Platform cache tags, Mercure topics, Redis keys, file storage paths, search documents.
A Monolog processor adds tenant_id to every log line.
A worker-mode test handles tenant A, then B, then a request with no tenant, and asserts that each saw only its own data and the last one failed.
9. Creating a tenant (provisioning)
9.1 Flow
sequenceDiagram
    autonumber
    participant API as Platform admin API (landlord)
    participant L as Landlord DB
    participant W as Provisioning worker
    participant PG as Tenant DB server (admin connection)
    participant T as New tenant DB
    API->>L: insert tenant (status pending, host chosen, db_name generated)
    API->>W: ProvisionTenantDatabase(tenantId)
    W->>PG: CREATE ROLE, CREATE DATABASE ... TEMPLATE tenant_template
    W->>L: status migrating
    W->>T: MigrateTenantDatabase (catch-up to latest)
    W->>L: status seeding, schema_version
    W->>T: SeedTenant (chart of accounts, settings, first officers)
    W->>L: status active
    W-->>API: TenantActivatedV1 (event)
9.2 Steps and state machine
Message
 | Handler does
 | Safe to repeat because
 | 
ProvisionTenantDatabase
 | Writes host and database name to the registry before creating; creates the role and database; locks down access
 | Checks pg_roles and pg_database first; names are deterministic
 | 
MigrateTenantDatabase
 | Runs tenant migrations to the latest version
 | The tenant's migration table records what ran
 | 
SeedTenant
 | Reference data and the first officers
 | INSERT … ON CONFLICT DO NOTHING, or a seed-version table
 | 
MarkTenantActive
 | Status active; publishes TenantActivatedV1
 | Guarded update
 | 
Status changes are guarded updates: UPDATE tenant SET status = :next WHERE id = :id AND status = :expected. Zero rows means another worker already did it.
Retries: exponential backoff (for example 5 retries from 2 s, capped at 10 minutes), then the failure transport. Validation errors throw UnrecoverableMessageHandlingException.
Compensation: after repeated failure in provisioning_db or migrating, drop the database and role (DROP DATABASE IF EXISTS … WITH (FORCE)), mark failed, and allow a retry. Never automatically drop a database that ever reached active.
9.3 The SQL (admin connection to the postgres database, autocommit)
-- 1. Runtime role for this tenant
CREATE ROLE "t_0192f3a7c1" LOGIN PASSWORD '…' NOSUPERUSER NOCREATEDB NOCREATEROLE CONNECTION LIMIT 20;
GRANT "t_0192f3a7c1" TO provisioner;          -- PG16+: needed to set OWNER
-- 2. Database cloned from the pre-migrated template
CREATE DATABASE "t_0192f3a7c1" OWNER app_owner TEMPLATE "tenant_template" STRATEGY WAL_LOG;
-- 3. Lock it down (database-level grants and settings are NOT copied from the template)
REVOKE CONNECT, TEMPORARY ON DATABASE "t_0192f3a7c1" FROM PUBLIC;
GRANT CONNECT ON DATABASE "t_0192f3a7c1" TO "t_0192f3a7c1", app_migrator;
ALTER DATABASE "t_0192f3a7c1" SET lock_timeout = '5s';
ALTER DATABASE "t_0192f3a7c1" SET statement_timeout = '30s';
PostgreSQL facts that shape this:
CREATE DATABASE and DROP DATABASE cannot run inside a transaction. Never call them from a migration or transactional().
It needs CREATEDB. On AWS RDS the master user has it but is not a superuser; use a dedicated provisioner role, never the app role.
It fails if anyone is connected to the template. Keep the template at ALLOW_CONNECTIONS false except while refreshing it, and serialise refresh and provisioning with the same lock.
STRATEGY WAL_LOG (default since PG15) suits small templates; FILE_COPY forces checkpoints. PG18's file_copy_method = CLONE is very fast but needs a filesystem you control, so not managed clouds.
DBAL's createDatabase() builds a bare CREATE DATABASE name with no quoting or options. Use executeStatement() with quoteSingleIdentifier() and validate names against ^[a-z][a-z0-9_]{2,62}$.
9.4 Role patterns
Pattern
 | How
 | Fit
 | 
Shared roles
 | app_owner owns objects everywhere; app_migrator runs migrations; app_runtime has data access only
 | Simplest; best with PgBouncer; isolation by the app's choice of database
 | 
Hybrid (recommended)
 | Shared owner and migrator, plus a per-tenant runtime login role with CONNECT only on its own database
 | A leaked runtime credential exposes one society
 | 
Full per-tenant
 | Each tenant role owns its database and objects
 | Most isolation; ownership must be reassigned after cloning; heavier ops
 | 
Per-tenant passwords: generated, stored encrypted (sodium) in the landlord or in a vault, referenced by db_secret_ref. PgBouncer authenticates them with auth_user + auth_query through a SECURITY DEFINER function.
9.5 Making creation fast
Template: one pre-migrated tenant_template per database server, refreshed after every fleet migration. Always run a migrate catch-up after cloning, so a stale template never matters for correctness.
Warm pool (optional): keep a few unassigned, migrated databases per server. Signup claims one (… FOR UPDATE SKIP LOCKED), renames it, changes ownership and runs the catch-up. Pool databases are registered like tenants so fleet migrations reach them.
Placement: pick the least-loaded server under its ceiling; store db_host from day one so tenants can later be spread and moved.
10. Migrations for every database
10.1 Two migration sets
migrations/
├── Landlord/   # namespace DoctrineMigrations\Landlord  → landlord database only
└── Tenant/     # namespace App\Migrations\Tenant         → every tenant database and the templates
DoctrineMigrationsBundle 4.0 supports one configuration only (one connection or entity manager). --em and --conn only pick names; there are no named configurations.
So: configure the bundle for the landlord. bin/console doctrine:migrations:migrate migrates the landlord.
For tenants, our own commands build a DependencyFactory per tenant connection: tenants:migrate, tenants:migrate:one, tenants:migrations:status, and tenants:migrations:diff (diff against a reference tenant database using the tenant entity manager).
Each tenant database has its own doctrine_migration_versions table, the source of truth. The landlord mirrors the latest version in tenant.schema_version for fleet queries.
# config/packages/doctrine_migrations.yaml (landlord only)
doctrine_migrations:
    connection: landlord
    migrations_paths:
        'DoctrineMigrations\Landlord': '%kernel.project_dir%/migrations/Landlord'
10.2 Migrating one tenant from code
final readonly class TenantMigrationRunner
{
    public function __construct(private TenantConnectionFactory $connections, private LoggerInterface $logger) {}
    /** @return array{from: string, to: string, executed: list} */
    public function migrate(Tenant $tenant, bool $dryRun = false, string $target = 'latest'): array
    {
        $connection = $this->connections->direct($tenant);           // direct, not through PgBouncer
        $connection->executeStatement("SET lock_timeout = '5s'");
        $connection->executeStatement("SET statement_timeout = '15min'");
        $configuration = new Configuration();
        $configuration->addMigrationsDirectory('App\\Migrations\\Tenant', $this->tenantMigrationsPath());
        $configuration->setAllOrNothing(true);                       // PostgreSQL DDL is transactional
        $storage = new TableMetadataStorageConfiguration();
        $storage->setTableName('doctrine_migration_versions');
        $configuration->setMetadataStorageConfiguration($storage);
        $factory = DependencyFactory::fromConnection(
            new ExistingConfiguration($configuration),
            new ExistingConnection($connection),
            $this->logger,
        );
        $factory->getMetadataStorage()->ensureInitialized();
        if ([] !== iterator_to_array($factory->getMigrationStatusCalculator()->getExecutedUnavailableMigrations())) {
            throw new TenantSchemaDrift($tenant->id());              // ran in DB but unknown to code
        }
        $current = (string) $factory->getVersionAliasResolver()->resolveVersionAlias('current');
        try {
            $version = $factory->getVersionAliasResolver()->resolveVersionAlias($target);
        } catch (NoMigrationsToExecute) {
            return ['from' => $current, 'to' => $current, 'executed' => []];
        }
        $plan = $factory->getMigrationPlanCalculator()->getPlanUntilVersion($version);
        if (0 === count($plan)) {
            return ['from' => $current, 'to' => $current, 'executed' => []];
        }
        $sql = $factory->getMigrator()->migrate(
            $plan,
            (new MigratorConfiguration())->setDryRun($dryRun)->setAllOrNothing(true),
        );
        return ['from' => $current, 'to' => (string) $version, 'executed' => array_map('strval', array_keys($sql))];
    }
}
Class names checked against doctrine/migrations 3.9 source: ExistingConfiguration is in Configuration\Migration, ExistingConnection in Configuration\Connection (one docs example uses a wrong namespace). doctrine/migrations takes no lock of its own; the command adds them. The drift check uses an iterable result (verify exact return type).
10.3 The fleet command
bin/console tenants:migrate [--tenant=ID ...] [--wave=canary|1|2|all] [--host=pg1]
                            [--concurrency=8] [--dry-run] [--resume] [--to=VERSION]
                            [--stop-after-failures=3]
Global lock tenants-migrate with Symfony Lock's DoctrineDbalPostgreSqlStore (PostgreSQL advisory lock) on a direct landlord connection. It also serialises template refresh and provisioning.
Per-tenant lock: pg_try_advisory_lock(hashtext('migrate')) inside the tenant database, so provisioning and the fleet run never migrate the same tenant at once.
Select tenants from the landlord: status IN ('active','migrating'), ordered by migration_wave (canary first: internal and pilot societies), then size. --resume skips tenants already at the target.
Run: small fleets loop in process; larger fleets fan out, either with symfony/process (a child tenants:migrate:one <id> each, clean memory) or with Messenger (MigrateTenantDatabase(tenantId, target, rolloutId) on a tenant_migrations transport). Cap concurrency per database server, not globally.
Per tenant: one all-or-nothing transaction; on lock_timeout (SQLSTATE 55P03) retry up to 3 times with backoff.
Record every result in tenant_migration_run (rollout_id, tenant_id, from_version, to_version, status, started_at, finished_at, duration_ms, error) and update tenant.schema_version.
Stop rules: any canary failure aborts; later waves stop after --stop-after-failures. Failed tenants get schema_status = failed and stay on the old schema, which the code still supports (10.4).
Dry run prints SQL per tenant. Exit non-zero if any tenant failed, with a summary table.
After success: refresh tenant_template on each server (allow connections → migrate → terminate sessions → disallow connections), under the same lock.
10.4 Changing the schema without downtime
During a rollout some societies are on version N and others on N+1, so the code must work with both:
Step
 | Example
 | 
1. Expand (migration)
 | Add a nullable column or a new table; create an index concurrently. Additive only
 | 
2. Deploy code
 | Writes both shapes, reads the new one with a fallback
 | 
3. Backfill
 | A background job per tenant in batches, never inside the migration
 | 
4. Tighten
 | ADD CONSTRAINT … CHECK (col IS NOT NULL) NOT VALID → VALIDATE CONSTRAINT → SET NOT NULL
 | 
5. Contract (a later release)
 | Drop the old column only when 100% of tenants are migrated
 | 
Deploy order: landlord migrations → deploy code compatible with N and N+1 → tenant migrations in waves → template refresh → enable feature flags → contract in the next release.
Indexes on big tables: CREATE INDEX CONCURRENTLY cannot run in a transaction. Put it alone in a migration with public function isTransactional(): bool { return false; }, write it idempotently (DROP INDEX CONCURRENTLY IF EXISTS …; CREATE INDEX CONCURRENTLY IF NOT EXISTS …), and check pg_index.indisvalid because a failure leaves an invalid index. Run such migrations as their own wave, outside all-or-nothing.
lock_timeout is the safety net: an ALTER TABLE waiting behind a long query blocks everything after it. Fail fast in 5 seconds and retry.
Roll forward, not back. Running down() across a fleet is riskier than a new corrective migration.
Never change a tenant schema by hand. Azure lists manual changes and single-version apps as anti-patterns.
10.5 Drift detection
Nightly job per tenant: migration status (executed-but-unknown and pending versions) plus a schema fingerprint (hash of information_schema columns, indexes and constraints) compared with the template. Differences set schema_status = drift and alert.
doctrine:schema:validate --skip-mapping against a tenant connection (verify option use with our runner).
11. Messaging and events across tenant databases
The Architecture guide requires the outbox to commit in the same transaction as the data, so outbox and inbox tables live in each tenant database. But one Messenger worker cannot read Doctrine transports in many databases: Symfony 8.1 requires PostgreSQL transports consumed together to share one connection and one table, otherwise it throws. PostgreSQL NOTIFY also only reaches the same database.
Design:
sequenceDiagram
    autonumber
    participant H as Handler (tenant A)
    participant OA as Outbox in tenant A DB
    participant RL as Relay (shard of tenants)
    participant CQ as Central transport (landlord or Redis)
    participant W as Worker
    participant IB as Inbox in tenant A DB
    H->>OA: business rows + outbox row, one commit
    RL->>OA: read unsent rows (fair round-robin across tenants)
    RL->>CQ: forward with TenantStamp(A), mark sent
    W->>CQ: consume
    W->>W: switch to tenant A
    W->>IB: inbox check + handler writes, one commit in tenant A
Relay processes each own a group of tenants (by hash), poll fairly, and record cursors in the landlord. Delivery stays at least once; the inbox removes duplicates.
Central transport in the landlord database (Doctrine) or Redis. Failed transport and scheduler are central too.
Worker middleware reads TenantStamp, switches tenant, handles, resets in finally. Messages without a stamp may only touch the landlord.
Lock and deduplication keys always include the tenant ID.
Events between modules of the same society go through this path; there are no events between societies.
Precedents: Tenancy for Laravel uses a central queue; an NServiceBus maintainer reports that one process polling outboxes in many tenant databases stops scaling beyond about 10, hence relay shards.
Simpler start (acceptable for few tenants): run one relay per server that loops over tenants every second; split into shards when lag grows.
12. Connections and limits
Item
 | Fact
 | Action
 | 
PgBouncer pools
 | One per (database, user); default size 20 → 1,000 tenants could mean 20,000 server connections
 | default_pool_size 2–5, max_db_connections, min_pool_size 0, server_idle_timeout
 | 
New tenants
 | [databases] * = host=pg1 creates pools on demand, freed after autodb_idle_timeout (3600 s)
 | One PgBouncer per database server; registry stores which
 | 
Transaction pooling
 | No SET/RESET, LISTEN, SQL PREPARE, session advisory locks
 | Per-database defaults via ALTER DATABASE … SET; migrations, locks and provisioning use a direct DSN
 | 
AWS RDS
 | max_connections default LEAST(memory/9531392, 5000); no documented cap on databases
 | Ceiling per server in our placement logic
 | 
Azure Flexible
 | 25–5000 connections by size, 15 reserved, change needs restart
 | Same
 | 
Cloud SQL
 | Metrics only for up to 500 databases
 | Plan monitoring accordingly
 | 
Catalog size
 | Very many tables per server bloats catalogs (one report: 200k tables, pg_attribute over 200 GB, hung connections)
 | Few hundred to ~1,000 tenant databases per server; module schemas kept lean
 | 
13. Backup, restore, move, offboard
Point-in-time recovery is whole-server. To restore one society: recover the server elsewhere to the time needed, then pg_dump -d <tenant> and restore that database. Add a nightly pg_dump per tenant for simple single-tenant restores.
Move a tenant between servers: dump and restore for small databases; logical replication for large ones (sequences and DDL are not replicated, so fix sequences before cutover). Steps: mark read-only → copy → verify counts → update db_host → invalidate caches → reopen.
Offboarding (NDPA 2023: rights in ss.34–38, storage limitation s.24, 72-hour breach notice s.40; confirm the erasure section in the gazetted text): export → grace period → DROP DATABASE … WITH (FORCE) → remove role, storage prefix, Redis keys, search documents, replication slots and the tenant's encryption key → registry deleted, keep the audit record. Keep backups only as long as policy allows; deleting the tenant key makes encrypted fields in old backups unreadable.
Encryption: PostgreSQL has no per-database encryption. Use application-level per-tenant keys for sensitive fields (BVN, bank details).
Monitoring: pg_stat_database per tenant (size, transactions, deadlocks); its counters are cumulative, so compare samples. Alert on relay lag, migration failures, drift and per-server tenant count.
14. Reporting across societies
PostgreSQL cannot query several databases in one statement; postgres_fdw and dblink are for ad-hoc work only. Platform reporting comes from the event stream (integration events copied to a warehouse) or scheduled ETL. Per-database change capture (Debezium) needs one replication slot per database; PostgreSQL allows 10 by default and abandoned slots keep WAL forever, so it scales poorly here.
15. Security
Roles: provisioner (CREATEDB, CREATEROLE; used only by the provisioning worker), app_migrator (DDL), per-tenant runtime roles (data only), app_landlord (landlord data).
REVOKE CONNECT … FROM PUBLIC on every tenant database; grant only its runtime role and the migrator.
Database names are generated, validated and quoted; never built from user input.
Symfony secrets hold only static values (landlord URL, master key). Tenant passwords are encrypted in the landlord or held in a vault.
Membership is checked after authentication on every request; cross-society access returns 404.
Logs carry tenant_id, never credentials.
16. Testing
Keep two or three fixed, pre-migrated test tenant databases created from a migrated template; for parallel runs suffix names with the test token (verify for your runner).
dama/doctrine-test-bundle rolls back per test on configured connections; check it wraps the tenant connection after switching (verify), and turn it off for relay tests, because rolled-back rows never reach the relay.
Tests to keep: provisioning end to end including a forced failure and compensation; tenants:migrate across three tenants with one failing; drift detection; switching A → B → none in one process; message handled in the right tenant; cache keys prefixed; cross-society 404.
17. Bundle status
Package
 | Status (Oct 2026)
 | Use
 | 
hakam/multi-tenancy-bundle
 | v3.1.0 supports only Symfony 6.4/7; Symfony 8 work sits on an untagged v4 branch; its reset does not clear connection settings (risk in workers); MySQL defaults
 | Reference design only, or adopt once v4 is tagged and fixed
 | 
zhortein/…, saifulferoz/…
 | Release candidates, very few installs
 | No
 | 
stancl/tenancy, spatie/laravel-multitenancy (Laravel)
 | Mature
 | Design reference for the create → migrate → seed lifecycle and central queue
 | 
Our own code: TenantContext, TenantDriverMiddleware, ResolveTenantListener, Messenger TenantMiddleware and stamp, TenantMigrationRunner, tenants:* commands, provisioning handlers, relay. Installing symfony/lock, symfony/process or PgBouncer needs approval.
18. Checklist
Landlord and tenant connections and entity managers, mappings disjoint, second-level cache off on tenant
Driver middleware fails closed; placeholder URL unusable
TenantContext::reset() rolls back and closes; worker-mode leak test passes
Tenant resolved by host or header; membership checked after login
Every cache, lock, limiter, cache tag, topic, key and file path prefixed by tenant
Provisioning chain idempotent, guarded status updates, compensation, never drops active tenants
Template per server, refreshed after each rollout, catch-up migrate after clone
tenants:migrate with global and per-tenant locks, waves, per-server concurrency, results recorded, resume
Expand/contract only; non-transactional index migrations isolated; roll forward
Nightly drift check
Outbox and inbox per tenant database, relay shards, central transport with tenant stamp
PgBouncer small pools; direct DSN for migrations and locks
Nightly per-tenant dumps; restore and move runbooks tested
Offboarding runbook meets NDPA
19. Sources
Doctrine DBAL 4.5 Connection source and UPGRADE notes: https://github.com/doctrine/dbal
DoctrineBundle 3.3 (Registry reset, middlewares): https://github.com/doctrine/DoctrineBundle
doctrine/migrations 3.9 (DependencyFactory, MigrateCommand): https://github.com/doctrine/migrations · DoctrineMigrationsBundle 4.0
PostgreSQL CREATE DATABASE, CREATE INDEX CONCURRENTLY, pg_stat_database: https://www.postgresql.org/docs/current/
PgBouncer configuration and features: https://www.pgbouncer.org/config.html
Symfony Messenger, Lock, FrankenPHP worker mode: https://symfony.com/doc/current/messenger.html, https://frankenphp.dev/docs/symfony/
Azure multitenant SaaS database patterns: https://learn.microsoft.com/azure/azure-sql/database/saas-tenancy-app-design-patterns · AWS SaaS Lens (silo, bridge, pool)
Atlassian engineering blog on Jira's Postgres fleet · Citus multi-tenant guidance
hakam/multi-tenancy-bundle: https://github.com/RamyHakam/multi_tenancy_bundle · Tenancy for Laravel: https://tenancyforlaravel.com · spatie/laravel-multitenancy
AWS RDS, Azure Flexible Server and Cloud SQL quota and limits pages
Nigeria Data Protection Act 2023
Research notes: doc/research_notes/tenancy/db_per_tenant_symfony.md, provisioning_and_migrations.md, db_per_tenant_operations.md
