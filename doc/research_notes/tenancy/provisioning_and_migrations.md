# DB-per-tenant on PostgreSQL + Symfony 8.1 / Doctrine: provisioning & fleet migrations

Researched 2026-10-12. Legend: **[V]** = verified against the primary source or source code listed; **[U]** = unverified (my design/sketch, or not checked against the source). Treat all code below as [U] unless it is marked [V].

---

## 0. Versions as of Oct 2026 (verified)

| Package | Latest | Notes | Source |
|---|---|---|---|
| doctrine/migrations | **3.9.8** (04 Oct) | 3.9.5 allowed Symfony 8 components. 3.9.8 fixed a DBAL 4.5 deprecation. Requires `doctrine/dbal ^3.6 \|\| ^4` and `symfony/console ^5.4..^8.0`. | https://github.com/doctrine/migrations/releases, composer.json on the 3.9.x branch |
| doctrine/doctrine-migrations-bundle | **4.0.1** (26 Aug), also 3.7.1 | 4.0.0 is a BC break: PHP ^8.4, `doctrine/dbal ^4`, `doctrine/doctrine-bundle ^3`, Symfony ^6.4/^7/^8. It drops container-aware migrations and makes classes final/internal. 3.6.0 added Symfony 8 support. 3.7.0 added "service migrations". 3.4.2 added "multiple connections managed by migrations" (#590). | https://github.com/doctrine/DoctrineMigrationsBundle/releases |
| doctrine/dbal | **4.5.0** (26 Sep) | 4.5 deprecates `Column::getType()` in favour of `getTypeName()`, plus table/column mutators; a new `SchemaEditor` replaces them. 4.4 deprecated `AbstractAsset::getName()`. | https://github.com/doctrine/dbal/releases |
| Symfony | **8.1** is current (8.2-dev) | | symfony.com/doc/current |
| PostgreSQL | **18.x** is current (18.6); 19 is in beta | | postgresql.org/docs/current |
| PgBouncer | 1.2x | The docs reference v1.21 features and PG18. | pgbouncer.org/config.html |

Implication: with Symfony 8.1, DBAL 4 and PHP 8.4+, use **DoctrineMigrationsBundle 4.0.x**.

---

## A. PROVISIONING

### A1. PostgreSQL primitives [V] (PG18 docs: https://www.postgresql.org/docs/current/sql-createdatabase.html)

```
CREATE DATABASE name [WITH] [OWNER user] [TEMPLATE t] [ENCODING e] [STRATEGY s] [LOCALE ..]
  [LC_COLLATE ..] [LC_CTYPE ..] [BUILTIN_LOCALE ..] [ICU_LOCALE ..] [ICU_RULES ..] [LOCALE_PROVIDER ..]
  [COLLATION_VERSION ..] [TABLESPACE ..] [ALLOW_CONNECTIONS b] [CONNECTION LIMIT n] [IS_TEMPLATE b] [OID n]
```

**Transactions and privileges**
- `CREATE DATABASE` "cannot be executed inside a transaction block". The same applies to `DROP DATABASE`. Run either on an autocommit admin connection, and never inside a Doctrine migration or `transactional()`.
- It needs superuser or the **CREATEDB** attribute. Setting `OWNER x` requires that you can `SET ROLE x`, so the provisioner must be a member of the tenant role (PG16+ semantics).

**Templates**
- `CREATE DATABASE` **fails if any other session is connected to the template** ("source database ... is being accessed by other users"). While the copy runs, new connections to the template are locked out.
- So keep the golden template **never connected**. Set `ALLOW_CONNECTIONS false` after building it, and toggle it back on only while you migrate it.
- Not copied from the template: database-level `ALTER DATABASE ... SET` settings and database-level GRANTs (CONNECT, etc.). Re-apply both per tenant.
- `IS_TEMPLATE true` lets any CREATEDB role clone the database. Otherwise only a superuser or the owner can.
- Encoding/locale must match the template unless the template is `template0`.
- The docs say a non-`template1` copy is "not (yet) intended as a general-purpose COPY DATABASE facility". It is still widely used for test and tenant cloning.

**STRATEGY** (added in PG15)
- `WAL_LOG` (default since 15): block-by-block copy, WAL-logged. Best for small templates and needs no checkpoints.
- `FILE_COPY`: filesystem-level directory copy. It **forces a checkpoint before and after**, which hurts on a busy primary, but is faster for big templates.
  - Benchmark (Aiven forum): ~97 s with `WAL_LOG` vs ~33 s with `FILE_COPY` on a large DB.
  - A tenant template holding only schema plus seed data is small, so `WAL_LOG` is fine. Avoid `FILE_COPY` on a busy primary because of the checkpoint storms.
- **PG18 `file_copy_method = COPY|CLONE`** [V] (runtime-config-resource). `CLONE` uses `copy_file_range()`/`copyfile` reflinks, which makes `STRATEGY=FILE_COPY` near-instant on XFS-reflink, ZFS or APFS.
  - boringSQL (2025-12-22) measured ~212 ms vs ~67 s for 6 GB.
  - **Not usable on RDS/Cloud SQL/Azure** (no filesystem control); it is self-hosted only.
- PG15 had a bug where `WAL_LOG` lost pg_class changes on templates. It was fixed and back-patched, so stay on current minor releases.

**Schema/security defaults** [V] (ddl-schemas)
- PG15+: `CREATE` on `public` is no longer granted to PUBLIC in new clusters. Databases upgraded from ≤14 still have it, so run `REVOKE CREATE ON SCHEMA public FROM PUBLIC` explicitly in the template.
- Every database grants `CONNECT` and `TEMP` to PUBLIC by default. That is why you need `REVOKE CONNECT ON DATABASE t FROM PUBLIC` per tenant; it is not inherited from the template.

**Timeouts** [V] (runtime-config-client)
- `lock_timeout`: per lock-acquisition wait.
- `statement_timeout`.
- `transaction_timeout`: PG17+ (the version is from my general knowledge [U]).
- `idle_in_transaction_session_timeout`.

**Recommended per-tenant SQL** (run on the admin connection to the `postgres` DB, autocommit) [U]

```sql
-- 1. role (LOGIN, no inheritance surprises); password from a secret manager / generated
CREATE ROLE "t_7f3a" LOGIN PASSWORD '…' NOSUPERUSER NOCREATEDB NOCREATEROLE CONNECTION LIMIT 20;
GRANT "t_7f3a" TO provisioner;                 -- needed so provisioner may set OWNER (PG16+)
-- 2. database from pre-migrated template
CREATE DATABASE "tenant_7f3a" OWNER "t_7f3a" TEMPLATE "tenant_template" STRATEGY WAL_LOG;
-- 3. lock down
REVOKE CONNECT, TEMPORARY ON DATABASE "tenant_7f3a" FROM PUBLIC;
GRANT CONNECT ON DATABASE "tenant_7f3a" TO "t_7f3a";
ALTER DATABASE "tenant_7f3a" SET lock_timeout = '5s';      -- per-db defaults are NOT copied from template
ALTER DATABASE "tenant_7f3a" SET statement_timeout = '30s';
-- 4. inside tenant DB (connect to it): ownership of objects cloned from template
--    objects keep template's owner -> run REASSIGN OWNED BY template_owner TO "t_7f3a" (inside tenant db)
--    or have the template objects owned by a shared "app_migrator" role and GRANT to tenant role.
CREATE EXTENSION IF NOT EXISTS pgcrypto;  -- usually already in template; gen_random_uuid() is core since PG13
```

**Owner patterns** (choose one) [U]
1. **Shared roles** (simplest; works best with PgBouncer):
   - `app_owner` (NOLOGIN) owns all objects in every tenant DB.
   - `app_migrator` (LOGIN, member of app_owner) runs migrations.
   - `app_runtime` (LOGIN) has DML only, through `ALTER DEFAULT PRIVILEGES FOR ROLE app_owner GRANT SELECT,INSERT,UPDATE,DELETE ON TABLES TO app_runtime`.
   - Isolation then comes from the app choosing the DB, not from credentials.
2. **Role per tenant**: stronger blast-radius isolation (a leaked credential exposes one tenant). The costs:
   - N roles.
   - PgBouncer needs `auth_query`.
   - Pools are per (user, db) anyway.
   - Ownership of objects cloned from the template must be reassigned.
   - Roles are cluster-global, so 10k tenants means 10k roles in pg_authid. That is fine, but backup/restore across clusters must carry roles (pg_dumpall -r).
3. Hybrid: shared owner/migrator plus a per-tenant runtime LOGIN role with `CONNECT` granted only on its DB.

Extensions:
- `gen_random_uuid()` is core since PG13, so pgcrypto/uuid-ossp are only needed for other functions.
- Install extensions in the **template** so tenants inherit them.
- On RDS, extensions need rds_superuser [U]. Pre-installing them in the template avoids needing that role at runtime.

### A2. Doing it from Symfony / DBAL 4 [V for API facts]

- `AbstractSchemaManager::createDatabase(string $database): void`, `dropDatabase(string)` and `listDatabases(): array` exist in DBAL 4.5 [V].
- **But** `AbstractPlatform::getCreateDatabaseSQL()` is literally `'CREATE DATABASE ' . $name`, with **no quoting, no OWNER/TEMPLATE/STRATEGY** [V, AbstractPlatform.php 4.5.x].
- PostgreSQL's `getListDatabasesSQL()` = `SELECT datname FROM pg_database`.
- So for tenancy, use raw `executeStatement()` with `$platform->quoteSingleIdentifier($name)`. In DBAL 4.5, `quoteIdentifier()` is deprecated in favour of `quoteSingleIdentifier()` [V].
- **Also validate tenant DB names against a regex** such as `^[a-z][a-z0-9_]{2,62}$`, because identifiers are max 63 bytes.

```php
// [U] admin connection: separate DBAL connection to "postgres" db with CREATEDB/CREATEROLE role
final class PostgresTenantDatabaseProvisioner
{
    public function __construct(private Connection $adminConnection) {}

    public function createDatabase(string $databaseName, string $ownerRole, string $templateName): void
    {
        $platform = $this->adminConnection->getDatabasePlatform();
        if (!preg_match('/^[a-z][a-z0-9_]{2,62}$/', $databaseName)) { throw new \InvalidArgumentException(); }
        // idempotency: check first, CREATE DATABASE has no IF NOT EXISTS
        $exists = (bool) $this->adminConnection->fetchOne('SELECT 1 FROM pg_database WHERE datname = ?', [$databaseName]);
        if ($exists) { return; }
        $this->adminConnection->executeStatement(sprintf(
            'CREATE DATABASE %s OWNER %s TEMPLATE %s STRATEGY WAL_LOG',
            $platform->quoteSingleIdentifier($databaseName),
            $platform->quoteSingleIdentifier($ownerRole),
            $platform->quoteSingleIdentifier($templateName),
        ));   // must NOT be inside $conn->beginTransaction()
        $this->adminConnection->executeStatement(sprintf('REVOKE CONNECT, TEMPORARY ON DATABASE %s FROM PUBLIC',
            $platform->quoteSingleIdentifier($databaseName)));
    }

    public function dropDatabase(string $databaseName): void
    {
        // PG13+: DROP DATABASE ... WITH (FORCE) terminates connections
        $this->adminConnection->executeStatement(sprintf('DROP DATABASE IF EXISTS %s WITH (FORCE)',
            $this->adminConnection->getDatabasePlatform()->quoteSingleIdentifier($databaseName)));
    }
}
```

**Per-tenant DBAL connection** [U]
- Build it with `DriverManager::getConnection([...landlordTemplateParams, 'dbname' => $tenant->databaseName, 'host' => $tenant->dbHost, 'user' => ..., 'password' => ...])`.
- Keep a small LRU of open connections in a `TenantConnectionFactory`.
- For the request-scoped ORM, the common Symfony approach is a DBAL `wrapper_class` or a connection-factory decorator that swaps params before connect, or one "tenant" EntityManager whose connection is (re)created per request or message.
- Workers are long-running, so close the connection after each message. `doctrine_close_connection` / `DoctrinePingConnectionMiddleware` exist in DoctrineBundle [U for 3.x names].

### A3. Running migrations programmatically (doctrine/migrations 3.9) [V: class and method names from source/docs]

Classes:
- `Doctrine\Migrations\Configuration\Configuration`: `addMigrationsDirectory(ns, path)`, `setAllOrNothing(bool)`, `setTransactional(bool)` [U for setTransactional name], `setCheckDatabasePlatform(bool)`, `setMetadataStorageConfiguration(...)`.
- `Doctrine\Migrations\Metadata\Storage\TableMetadataStorageConfiguration`: `setTableName('doctrine_migration_versions')`.
- `Doctrine\Migrations\Configuration\Migration\ExistingConfiguration` (implements `ConfigurationLoader`) [V, source]. The docs page's 2nd example uses a wrong namespace, `Configuration\Configuration\ExistingConfiguration`.
- `Doctrine\Migrations\Configuration\Connection\ExistingConnection` (implements `ConnectionLoader`) [V].
- `Doctrine\Migrations\DependencyFactory::fromConnection(ConfigurationLoader, ConnectionLoader, ?LoggerInterface)` and `::fromEntityManager(ConfigurationLoader, EntityManagerLoader, ?LoggerInterface)` [V].
- DependencyFactory getters [V]:
  - `getMetadataStorage()`
  - `getMigrationRepository()`
  - `getVersionAliasResolver()`
  - `getMigrationPlanCalculator()`
  - `getMigrationStatusCalculator()`
  - `getMigrator()`
  - `getConsoleInputMigratorConfigurationFactory()`
  - `getSchemaProvider()`
  - `getDiffGenerator()`
  - `setService(id, obj)`
  - `freeze()`
- `Doctrine\Migrations\MigratorConfiguration`: `setDryRun()`, `setTimeAllQueries()`, `setNoMigrationException()`, `setAllOrNothing()`, `setFromSchema()` [V].
- `Migrator::migrate(MigrationPlanList $plan, MigratorConfiguration $cfg): array` returns SQL keyed by version [V].

What `MigrateCommand::execute()` does, which you should replicate [V, 3.9.x source]:
1. `getMetadataStorage()->ensureInitialized()` creates `doctrine_migration_versions` if it is missing.
2. `getMigrationRepository()->getMigrations()`: exit if there are none.
3. `getVersionAliasResolver()->resolveVersionAlias('latest')`. This catches `UnknownMigrationVersion` and `NoMigrationsToExecute | NoMigrationsFoundWithCriteria`.
4. `getMigrationStatusCalculator()->getExecutedUnavailableMigrations()` finds versions executed in the DB that are unknown in code. **This is a drift signal**: abort the tenant.
5. `getMigrationPlanCalculator()->getPlanUntilVersion($version)`. If it is empty, the tenant is up to date.
6. `getMigrator()->migrate($plan, $migratorConfiguration)`.
7. **No locking** is done by doctrine/migrations itself [V]. You must add your own lock.

```php
// [U] per-tenant migration runner
use Doctrine\Migrations\Configuration\Configuration;
use Doctrine\Migrations\Configuration\Connection\ExistingConnection;
use Doctrine\Migrations\Configuration\Migration\ExistingConfiguration;
use Doctrine\Migrations\DependencyFactory;
use Doctrine\Migrations\Exception\NoMigrationsToExecute;
use Doctrine\Migrations\Metadata\Storage\TableMetadataStorageConfiguration;
use Doctrine\Migrations\MigratorConfiguration;

final class TenantMigrationRunner
{
    public function __construct(private TenantConnectionFactory $connections, private LoggerInterface $logger) {}

    /** @return array{from: ?string, to: string, executed: list<string>} */
    public function migrate(Tenant $tenant, bool $dryRun = false, string $target = 'latest'): array
    {
        $conn = $this->connections->forTenant($tenant);           // fresh DBAL Connection, autocommit
        $conn->executeStatement("SET lock_timeout = '5s'");      // session-level; NOT through PgBouncer txn mode
        $conn->executeStatement("SET statement_timeout = '15min'");

        $config = new Configuration();
        $config->addMigrationsDirectory('App\\Migrations\\Tenant', __DIR__.'/../../migrations/Tenant');
        $config->setAllOrNothing(true);          // all pending migrations in ONE txn (PG has transactional DDL)
        $config->setCheckDatabasePlatform(false);
        $storage = new TableMetadataStorageConfiguration();
        $storage->setTableName('doctrine_migration_versions');
        $config->setMetadataStorageConfiguration($storage);

        $df = DependencyFactory::fromConnection(new ExistingConfiguration($config), new ExistingConnection($conn), $this->logger);
        $df->getMetadataStorage()->ensureInitialized();

        if (count($df->getMigrationStatusCalculator()->getExecutedUnavailableMigrations()) > 0) {
            throw new TenantSchemaDriftException($tenant->id);
        }
        $current = (string) $df->getVersionAliasResolver()->resolveVersionAlias('current');
        try {
            $version = $df->getVersionAliasResolver()->resolveVersionAlias($target);
        } catch (NoMigrationsToExecute) {
            return ['from' => $current, 'to' => $current, 'executed' => []];
        }
        $plan = $df->getMigrationPlanCalculator()->getPlanUntilVersion($version);
        if (count($plan) === 0) { return ['from' => $current, 'to' => $current, 'executed' => []]; }

        $sql = $df->getMigrator()->migrate($plan, (new MigratorConfiguration())->setDryRun($dryRun)->setAllOrNothing(true));
        return ['from' => $current, 'to' => (string) $version, 'executed' => array_map('strval', array_keys($sql))];
    }
}
```

**Caveats on `all_or_nothing`**
- It wraps all pending migrations in one transaction. A migration with `isTransactional(): false` breaks this. Run CONCURRENTLY migrations as a separate wave, or don't combine them with all-or-nothing.
- Recent 3.x versions behave as follows [U]: when `transactional=false` or a non-transactional migration is in the plan, all-or-nothing is not applied to it.
- `Migrator::migrate` returns `array<string, Query[]>` [U, exact type].

### A4. Faster path: clone a pre-migrated template [U design]
- Keep `tenant_template` with `ALLOW_CONNECTIONS false` and `IS_TEMPLATE true`.
- On each deploy, after the tenant migration rollout:
  1. `ALTER DATABASE tenant_template ALLOW_CONNECTIONS true`
  2. Run the migrate and reference-data seed.
  3. `SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE datname='tenant_template'`
  4. `ALTER ... ALLOW_CONNECTIONS false`
- Template migration must be serialized with provisioning by the **same advisory lock**, otherwise CREATE DATABASE fails because the template is in use.
- After `CREATE DATABASE ... TEMPLATE`, **still run the migrator** against the new DB. It is a no-op if the template is current and catches up if the template lagged the code. This makes the template an optimisation, never a correctness dependency.
- The template must live **on each DB server** (shard), so maintain one template per host.

### A5. Orchestration with Symfony Messenger [U design]
State machine on the landlord `tenant` row:
- States: `pending → provisioning_db → migrating → seeding → active`.
- Any step can go to `failed` (with `failed_step` and `last_error`) and later to `deleting → deleted`.
- Use the Symfony Workflow component (state_machine type), or a plain enum plus a guarded update: `UPDATE tenant SET status=:next WHERE id=:id AND status=:expected`. The guarded update is an optimistic transition and gives idempotency.

Messages (each handler idempotent; chain by dispatching the next message on success):

| Message | Handler does | Idempotency |
|---|---|---|
| `ProvisionTenantDatabase(tenantId)` | Picks a host (least-loaded), writes `db_host` and `db_name` to the landlord **before** creating, then CREATE ROLE and CREATE DATABASE | Checks `pg_roles`/`pg_database` existence first. The DB name is deterministic (`tenant_<uuid-short>`). |
| `MigrateTenantDatabase(tenantId)` | `TenantMigrationRunner::migrate()` | Doctrine's metadata table makes it a no-op on re-run |
| `SeedTenant(tenantId)` | Seeds reference data and the first admin user | Upsert (`INSERT ... ON CONFLICT DO NOTHING`), or a `tenant_seed_versions` table |
| `MarkTenantActive(tenantId)` | Status → active, emits TenantActivated, sends the welcome email | Guarded update |

- **Retries**: `framework.messenger.transports.provisioning.retry_strategy: {max_retries: 5, delay: 2000, multiplier: 3, max_delay: 600000}`.
  - Throw `UnrecoverableMessageHandlingException` for validation errors.
  - Use a `failure_transport` for the dead-letter queue (DLQ). [V-ish: standard Messenger config keys; check against the 8.1 docs.]
- **Timeouts**: set `statement_timeout` on the admin connection (e.g. 120s for CREATE DATABASE, which can wait on checkpoint and template locks). Set the worker `--time-limit` and `--memory-limit`.
- **Compensation**: after N failures on step `provisioning_db` or `migrating`, a `CompensateFailedProvisioning` handler:
  - Runs `DROP DATABASE IF EXISTS ... WITH (FORCE)` (PG13+) and `DROP ROLE IF EXISTS`.
  - Marks the tenant `failed`, then lets the user retry.
  - Never auto-drop a DB that ever reached `active`.
- **Warm pool**: a cron or Scheduler (`symfony/scheduler`) keeps K unassigned DBs per host (`pool_db_001..`) already cloned and migrated.
  - Signup claims one with `UPDATE pool SET tenant_id=:t WHERE id = (SELECT id FROM pool WHERE host=:h AND tenant_id IS NULL ORDER BY id FOR UPDATE SKIP LOCKED LIMIT 1) RETURNING db_name`.
  - Then `ALTER DATABASE pool_db_001 RENAME TO tenant_x`, which requires no connections to it, plus `ALTER DATABASE ... OWNER TO`, then a migrate catch-up. Signup becomes sub-second.
  - The pool refiller must also run the fleet migrations on pool DBs, so treat pool DBs as tenants in the registry.

### A6. Managed Postgres limits [V where cited]

**Databases per instance**
- No documented hard cap on any of AWS RDS, Cloud SQL or Azure Flexible.
- RDS's quota page has no databases-per-instance quota for PG. Cloud SQL lists none, and only collects metrics "for up to 500 databases". Azure's limits page lists none.
- Practical limits come from files and inodes, catalog size, autovacuum workers and backups.
- **Atlassian (Jira)** [V, Atlassian eng blog]:
  - ~4,000,000 DBs on ~3,000 instances in 13 regions.
  - "up to about 4,000 databases" per cluster.
  - ~5,000 files per Jira DB caused Aurora conversion timeouts. AWS advised fewer DBs per instance.
  - Normal tenant moves run ~1,000/day. Moves during the migration averaged 38k/day, peaking ~90k/day.
  - Orchestration was done with AWS Step Functions and preflight/rollback checks.
- Rule of thumb [U]: keep a few hundred to ~2k small DBs per instance, and register `db_host` per tenant from day one.

**max_connections**
- RDS PG [V]: default `LEAST({DBInstanceClassMemory/9531392}, 5000)`, range 6–262143.
- Azure Flexible [V via search of MS Learn]:
  - Range 25–5000, static (needs a restart). Defaults depend on SKU, e.g. B1ms has 50 (35 user) and the 16-vCore E-series has 5000.
  - 15 connections are reserved.
- Cloud SQL [V]: the default depends on the machine type, and the combined settings cap is 262142.
- Each PG backend is a process costing several MB (work_mem is extra). DB-per-tenant multiplies idle connections, so a pooler is mandatory.

**RDS privileges** [V, AWS docs]
- The master user is `LOGIN NOSUPERUSER CREATEDB CREATEROLE` and a member of `rds_superuser`.
- Create a dedicated `provisioner` role with CREATEDB and CREATEROLE. Don't use master in the app.

**RDS Proxy**
- Supports many DBs, but session pinning on `SET` breaks multiplexing [U].
- For thousands of DBs, PgBouncer is the more common choice.

**PgBouncer** [V, pgbouncer.org/config.html and features.html]
- Pools are per **(database, user)** pair. `default_pool_size` defaults to 20, so with 1,000 tenants the server can see up to 20k connections.
  - Cap with `max_db_connections` (per DB) and `max_user_connections`.
  - Use small per-tenant pools (`default_pool_size=2..5`), `server_idle_timeout` (default 600 s), and `min_pool_size=0`.
- `[databases] * = host=pg1 port=5432` is a fallback: any DB name is auto-created as a pool and freed after `autodb_idle_timeout` (default 3600 s). This means no config file edit per tenant, but it only supports one host per `*`.
  - Multi-host sharding: run one PgBouncer per DB server, with the tenant registry storing the PgBouncer host. Or list each tenant DB explicitly and `RELOAD` (SIGHUP) [U].
- `auth_user` + `auth_query` look up passwords from the server, for per-tenant roles without an `auth_file` entry for each. The docs recommend a SECURITY DEFINER function instead of reading `pg_authid` directly.
  - `auth_dbname` sets which DB the lookup runs in. The function must exist in every DB unless `auth_dbname` is set [U].
- **Transaction pooling does NOT support**: `SET`/`RESET`, `LISTEN`, SQL `PREPARE`, WITH HOLD cursors, **session-level advisory locks**.
  - Protocol-level prepared statements work if `max_prepared_statements > 0` (default 200 in recent versions).
  - Consequences:
    - Set timeouts via `ALTER DATABASE/ROLE ... SET` or `SET LOCAL`.
    - **Run migrations and advisory locks over a direct (or session-mode) connection, not via the transaction pooler.**
    - Give the provisioning/migration admin path a separate DSN.
- `max_client_conn` default is 100; raise it along with the file-descriptor ulimit.

---

## B. MIGRATIONS FOR ALL TENANTS

### B1. Two migration sets
- Directory layout: `migrations/Landlord/` (namespace `DoctrineMigrations\Landlord`) and `migrations/Tenant/` (namespace `App\Migrations\Tenant`).
- **Bundle reality (4.0.x)** [V, Configuration.php and DoctrineCommand.php]:
  - The bundle config has **one** `doctrine_migrations` tree with single `connection` and `em` keys. There is **no multiple named configurations feature**.
  - The bundle's `doctrine:migrations:*` commands accept `--em`/`--conn` (mutually exclusive), which call `getConfiguration()->setEntityManagerName()` / `setConnectionName()`.
  - `--configuration=path` can swap in a different migrations config file (php/xml/yml) through `setConfigurationLoader()`.
- Practical setup:
  1. Configure the bundle for the **landlord** (`connection: landlord`, `migrations_paths: {'DoctrineMigrations\Landlord': '%kernel.project_dir%/migrations/Landlord'}`), so `bin/console doctrine:migrations:migrate` = landlord.
  2. For tenants, build your own `DependencyFactory` per tenant connection (A3) inside a custom command. Do not use the bundle's DI-wired factory, because it is bound to one connection.
  3. For **generating** tenant migrations (`diff`):
     - Point a dev-only "tenant_dev" connection and EM at a reference tenant DB, using an EM whose mappings cover `src/Entity/Tenant`.
     - Run `doctrine:migrations:diff --em=tenant --configuration=config/migrations/tenant.yaml` [U, verify that `--configuration` + `--em` combine as expected in bundle 4].
     - Alternatively, write a thin `tenants:migrations:diff` command that creates `DependencyFactory::fromEntityManager(new ExistingConfiguration($tenantConfig), new ExistingEntityManager($tenantEm))` [U: `ExistingEntityManager` class in `Configuration\EntityManager`] and runs `new DiffCommand($df)`.
     - This mirrors the hakam/multi-tenancy-bundle approach (`tenant:migration:diff`, `tenant:migration:migrate`, with `migrations/Main` and `migrations/Tenant` dirs).
- Each tenant DB has its own `doctrine_migration_versions` table, which is the per-tenant source of truth.
- Mirror the latest version into the landlord as `tenant.schema_version` and `tenant.schema_migrated_at` for fleet queries.

### B2. `tenants:migrate` command design [U]

```
bin/console tenants:migrate [--tenant=ID ...] [--wave=canary|1|2|all] [--host=pg1] [--concurrency=8]
                            [--dry-run] [--resume] [--to=VERSION] [--lock-timeout=5s] [--stop-after-failures=3]
```

1. **Global lock**:
   - `Symfony\Component\Lock\Store\DoctrineDbalPostgreSqlStore` (PG advisory locks: native blocking, non-expiring, released at session end) [V, Symfony 8.1 docs].
   - Lock key `tenants-migrate` on the **landlord direct connection, not via the PgBouncer txn pool**.
   - This prevents concurrent rollouts and also serializes against template refresh and provisioning.
2. **Per-tenant lock**: the runner takes `pg_try_advisory_lock(hashtext('migrate'))` inside the tenant DB, so provisioning and fleet migration can't both migrate the same tenant.
3. **Select tenants** from the landlord:
   - `status IN ('active','migrating')`, ordered by wave (`tenant.migration_wave`: 0=canary/internal, 1=small/free, 2=rest, 3=VIP last or first by policy), then by size.
   - `--resume` skips tenants with `schema_version = target`.
4. **Execution**:
   - (a) In-process sequential loop for small fleets.
   - (b) `symfony/process` fan-out: spawn `bin/console tenants:migrate:one <id>` with a pool of N concurrent `Process`, polling `isRunning()`. This isolates memory and PHP state, and each child gets its own DependencyFactory.
   - (c) Messenger fan-out: dispatch `MigrateTenantDatabase(id, targetVersion, rolloutId)` to a `tenant_migrations` transport consumed by N workers, with a `rollout` table tracking counts. Pros: durable, resumable, scales across hosts.
   - Cap concurrency **per DB host**, not globally. Atlassian found destination load was the binding constraint.
5. **Per tenant**:
   - `SET lock_timeout='5s'`, `SET statement_timeout`.
   - all-or-nothing transaction: Postgres DDL is transactional, so a failure leaves the tenant on the old version cleanly. Exceptions are CONCURRENTLY statements, `ALTER TYPE ... ADD VALUE` (allowed inside a transaction since PG12, but the new value cannot be used in the same transaction), and CREATE/DROP DATABASE.
   - Retry on `lock_timeout` (SQLSTATE 55P03 lock_not_available) with backoff, up to 3 times.
6. **Record results**:
   - `tenant_migration_run(rollout_id, tenant_id, from_version, to_version, status, started_at, finished_at, error, duration_ms)`.
   - Update `tenant.schema_version`.
7. **Failure policy**:
   - In the canary wave, any failure aborts the rollout.
   - In later waves, abort after `--stop-after-failures` failures (circuit breaker).
   - Failed tenants are marked `schema_status=failed` and may be put in maintenance mode by the app, or stay on old code if code is backward compatible.
8. **Dry run**: `MigratorConfiguration::setDryRun(true)` logs the SQL without executing it. Pair it with the `--write-sql` equivalent: `$df->getQueryWriter()->write($path, $direction, $sql)`.
9. Exit non-zero if any tenant failed. Print a summary table.

### B3. Zero-downtime / mixed-version fleet
- During a rollout, tenants are on version N or N+1. **Code must work with both schemas.** Use expand/contract:
  1. **Expand** (migration): add nullable column or new table, or create an index CONCURRENTLY. This is additive only.
  2. Deploy code that writes both and reads the new with a fallback, or that tolerates both shapes.
  3. **Backfill** as a background job per tenant (Messenger, batched `UPDATE ... WHERE id BETWEEN`), not inside the migration.
  4. Switch reads; enforce `NOT NULL` via `ADD CONSTRAINT ... CHECK (col IS NOT NULL) NOT VALID` then `VALIDATE CONSTRAINT`, then `SET NOT NULL`. PG12+ uses the validated check to skip the table scan [U, from memory].
  5. **Contract** (a later release, after 100% of tenants are migrated): drop the old column/table.
- **Deploy order**:
  1. Landlord migrations.
  2. Deploy N+1 code (compatible with N and N+1).
  3. Tenant expand migrations (waves).
  4. Refresh the template.
  5. Enable feature flags.
  6. Contract in the next release.
- Alternative: gate code paths per tenant on `tenant.schema_version` (feature flag by schema version). This is the Atlassian-style feature flag approach.
- Guard against a newly provisioned tenant cloned from a stale template by always running a migrate catch-up (A4).
- **CREATE INDEX CONCURRENTLY** [V]:
  - Cannot run in a transaction block.
  - On failure it leaves an **INVALID** index: drop it and retry, or `REINDEX INDEX CONCURRENTLY`.
  - Waits for existing transactions.
  - Only one per table at a time.
  - Not supported on partitioned parents.
  - In Doctrine, put it in its own migration with `public function isTransactional(): bool { return false; }` [V: `AbstractMigration::isTransactional()` exists].
  - Make it idempotent: `DROP INDEX CONCURRENTLY IF EXISTS idx; CREATE INDEX CONCURRENTLY IF NOT EXISTS idx ...`. Check `pg_index.indisvalid` for leftovers from a partial run.
- `lock_timeout` is the key safety net. An `ALTER TABLE` waiting for an ACCESS EXCLUSIVE lock behind a long query blocks all subsequent queries on that table. With `lock_timeout=2-5s`, fail fast and retry.

### B4. Drift detection & rollback [U design, V API names]
- Per tenant, run `getMigrationStatusCalculator()->getExecutedUnavailableMigrations()` and `getNewMigrations()` [U name], which mirror `migrations:status` / `migrations:up-to-date`. Also run `SchemaValidator` / `doctrine:schema:validate --skip-mapping` against a tenant EM [U].
- Stronger check: compare a schema fingerprint. Hash the output of `pg_dump --schema-only --no-owner --no-privileges` per tenant, or query `information_schema.columns` and `pg_indexes`, ordered. Compare against the template's fingerprint. Run nightly with the Scheduler.
- Also flag tenants whose `doctrine_migration_versions` max differs from the landlord `schema_version`.
- **Rollback**:
  - Prefer **roll-forward** (a new fixing migration).
  - `down()` across a fleet is risky (data loss, partial fleet).
  - Because migrations are additive (expand), you usually only roll back **code**.
  - For catastrophic cases, use a per-tenant PITR or a pre-migration `pg_dump`. That is cheap for small tenants: optionally snapshot before contract migrations.

### B5. Real-world lessons (any stack)
- **Atlassian Jira** (https://www.atlassian.com/blog/atlassian-engineering/migrating-jira-database-platform-to-aws-aurora): per-tenant DB at 4M scale works only with heavy orchestration. Lessons:
  - File and inode counts per instance matter.
  - Per-tenant endpoint and credentials make cutover complex.
  - Concurrency must be limited per source and destination.
  - Start with the smallest, least-used tenants.
  - Feature flags handle endpoint overrides.
- **PlanetScale** "Approaches to tenancy in Postgres" (https://planetscale.com/blog/approaches-to-tenancy-in-postgres): db-per-tenant is the most isolated, but it is at odds with Postgres's connection model. Their estimate: "likely won't scale beyond a few hundred tenants" without serious tooling.
- **Crunchy Data** (https://www.crunchydata.com/blog/designing-your-postgres-database-for-multi-tenancy): rates it high management overhead.
- **Arkency** survey (https://blog.arkency.com/peoples-experiences-with-approaches-to-multitenancy/):
  - Thousands of per-tenant DBs led to schema "locked in place" because fleet migrations were too painful.
  - Naive loop scripts that stop at the first failure leave the fleet split.
  - You need per-DB tracking.
- **stancl/tenancy** (Laravel, https://tenancyforlaravel.com/docs/v3/event-system):
  - Canonical pipeline: `TenantCreated → JobPipeline::make([CreateDatabase, MigrateDatabase, SeedDatabase])->send(fn($e)=>$e->tenant)->shouldBeQueued(true)->toListener()`.
  - `tenants:migrate` loops tenants. It is a good reference for the A5 design.
- **hakam/multi-tenancy-bundle** (Symfony, https://github.com/RamyHakam/multi_tenancy_bundle):
  - db-per-tenant, with `migrations/Main` + `migrations/Tenant` and `tenant:migration:diff|migrate <id>`.
  - Bulk all-tenant migrate was "soon" in older READMEs, so check the current version.
  - Other bundles: `apajo/symfony-multi-tenancy-bundle` (`tenants:migrations:*`, early stage).

---

## Checklist / decisions for Colsynq [U]
1. The landlord `tenant` table has: `id`, `slug`, `status` enum, `db_host`, `db_port`, `db_name`, `db_role`, `secret_ref`, `schema_version`, `schema_migrated_at`, `migration_wave`, `failed_step`, `last_error`.
2. Connections:
   - Admin DSN: direct and session-level, with a CREATEDB/CREATEROLE role, used for provisioning, migrations and locks.
   - Runtime DSN: via PgBouncer in transaction mode with `max_prepared_statements>0`. No session `SET`, so timeouts go in `ALTER DATABASE ... SET`.
3. Use a template per host plus a migrate catch-up. Add a warm pool later if signup latency matters.
4. Messenger async provisioning with a guarded state machine, idempotent handlers, a DLQ, and compensation only for never-active tenants.
5. `tenants:migrate`:
   - Advisory lock, waves, per-host concurrency.
   - Per-tenant all-or-nothing transaction with `lock_timeout`.
   - Non-transactional migrations isolated.
   - Results recorded in the landlord, plus resume and dry-run.
6. Expand/contract discipline enforced in code review. Contract migrations only after the fleet is at 100%.
7. Nightly drift check: migration versions plus a schema fingerprint.

## Sources
- PG CREATE DATABASE (PG18): https://www.postgresql.org/docs/current/sql-createdatabase.html
- PG file_copy_method: https://www.postgresql.org/docs/current/runtime-config-resource.html
- PG schemas/public privileges: https://www.postgresql.org/docs/current/ddl-schemas.html
- PG CREATE INDEX: https://www.postgresql.org/docs/current/sql-createindex.html
- PG client timeouts: https://www.postgresql.org/docs/current/runtime-config-client.html
- PG15 STRATEGY / WAL_LOG notes: https://aiven.io/community/forum/t/faq-why-is-database-creation-from-templates-slower-for-large-databases-in-postgresql-15/421
- PG18 clone: https://boringsql.com/posts/instant-database-clones/
- doctrine/migrations custom config: https://www.doctrine-project.org/projects/doctrine-migrations/en/3.9/reference/custom-configuration.html
- doctrine/migrations source (3.9.x): https://github.com/doctrine/migrations/tree/3.9.x/src (MigrateCommand.php, DoctrineCommand.php, DependencyFactory.php, MigratorConfiguration.php, Configuration/Migration/ExistingConfiguration.php, Configuration/Connection/ExistingConnection.php, AbstractMigration.php)
- doctrine/migrations releases: https://github.com/doctrine/migrations/releases
- DoctrineMigrationsBundle releases/config: https://github.com/doctrine/DoctrineMigrationsBundle/releases, https://github.com/doctrine/DoctrineMigrationsBundle/blob/4.0.x/src/DependencyInjection/Configuration.php
- DBAL 4.5 source: https://github.com/doctrine/dbal/blob/4.5.x/src/Schema/AbstractSchemaManager.php, https://github.com/doctrine/dbal/blob/4.5.x/src/Platforms/AbstractPlatform.php; releases https://github.com/doctrine/dbal/releases
- Symfony Lock (8.1): https://symfony.com/doc/current/components/lock.html
- PgBouncer config/features: https://www.pgbouncer.org/config.html, https://www.pgbouncer.org/features.html
- AWS RDS quotas: https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/CHAP_Limits.html
- AWS RDS master user: https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/UsingWithRDS.MasterAccounts.html
- Cloud SQL quotas: https://docs.cloud.google.com/sql/docs/postgres/quotas
- Azure Flexible limits: https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/concepts-limits
- Atlassian: https://www.atlassian.com/blog/atlassian-engineering/migrating-jira-database-platform-to-aws-aurora
- PlanetScale: https://planetscale.com/blog/approaches-to-tenancy-in-postgres
- Crunchy Data: https://www.crunchydata.com/blog/designing-your-postgres-database-for-multi-tenancy
- Arkency: https://blog.arkency.com/peoples-experiences-with-approaches-to-multitenancy/
- stancl/tenancy: https://tenancyforlaravel.com/docs/v3/event-system
- hakam bundle: https://github.com/RamyHakam/multi_tenancy_bundle
