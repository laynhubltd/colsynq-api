# Database-per-tenant multi-tenancy: Symfony 8.1, Doctrine, API Platform 5, PostgreSQL

Research date: 2026-10-12. Each tenant has its own PostgreSQL database. A central ("landlord") database holds the tenant registry.
`[U]` marks syntax I have not checked against source. Everything else was read from the source or docs linked.

## 0. Version snapshot (GitHub releases API, 2026-10-12)

| Package | Latest | Notes |
|---|---|---|
| symfony/symfony | v8.1.8 (2026-09-29) | 7.4.20 is the LTS line |
| doctrine/dbal | 4.5.0 (2026-09-26) | 4.4.5 also exists; 3.10.6 is still maintained |
| doctrine/orm | 3.7.5 (2026-10-09) | 3.7.0 came out 2026-09-07 |
| doctrine/doctrine-bundle | 3.3.2 (2026-09-09) | Requires php ^8.4, dbal ^4.0, persistence ^4, symfony ^6.4\|^7\|^8 |
| api-platform/core | v5.0.3 (2026-10-09) | 5.0.0 came out in Sept 2026; requires php >=8.2, symfony ^7.4\|^8.0 |
| hakam/multi-tenancy-bundle | v3.1.0 (2026-04-26) | **v3.1.0 does NOT support Symfony 8 or DoctrineBundle 3** (needs symfony ^6.4\|^7.0, doctrine-bundle ^2.7). Symfony 8 support exists only on the untagged `v4` branch |
| stancl/tenancy (archtechx) | v3.10.1 (2026-08-05) | Laravel reference |
| spatie/laravel-multitenancy | 4.2.1 (2026-09-22) | Laravel reference |

Sources: https://github.com/doctrine/dbal/releases, https://github.com/doctrine/orm/releases, https://github.com/doctrine/DoctrineBundle/releases, https://github.com/api-platform/core/releases, https://packagist.org/packages/hakam/multi-tenancy-bundle

---

## 1. Runtime connection switching

### 1.1 Facts about DBAL 4 `Connection` (read from `src/Connection.php` at 4.5.0)

- `class Connection` is not final and carries `@phpstan-consistent-constructor`, so `wrapper_class` is still supported.
- `protected ?DriverConnection $_conn = null;` is protected, so a subclass can set it.
- `private array $params;` is private. There is **no setter**.
- `getParams(): array` is marked `@internal` and returns `$this->params`.
- `protected function connect(): DriverConnection` does `$this->_conn = $this->driver->connect($this->params)`, then handles autoCommit. It is protected and returns the driver connection (BC break in 4.0, see below).
- `close(): void` does only `$this->_conn = null; $this->transactionNestingLevel = 0;`. The next query reconnects lazily through `connect()`.
- `getDatabase()` now **queries the server** (`SELECT CURRENT_DATABASE()` through the platform) and no longer reads params.
- The platform is cached in `private ?AbstractPlatform $platform`. It is not reset on `close()`. That is fine when every tenant uses the same PostgreSQL major version, which you should enforce anyway. If the version differs, set `serverVersion` explicitly.
- `convertException()` is `final public`, so a subclass can reuse it.

DBAL 4.0 BC breaks (UPGRADE.md) that broke the old Symfony "selectDatabase" / `changeDatabase()` wrapper trick:
- "Removed `Connection::getWrappedConnection()`, `Connection::connect()` made `protected`... must return the underlying driver-level connection." The old pattern (`public function connect(): bool` plus calling `parent::__construct($params, $driver, $config, $eventManager)` from `selectDatabase()`) no longer fits. The EventManager constructor argument is gone in 4.0, and `connect()` has a different signature and return type.
- Re-calling `__construct()` on a live object is still technically possible, but it is a hack.
- In practice, the old wrapper_class trick broke because it relied on `connect()` returning bool, the 4-arg constructor and `getWrappedConnection()`. RamyHakam's PR #88 says the switch was needed because DBAL 4 removed `_conn`/`_driver` (only partly accurate: `$_conn` is still protected in 4.0–4.5).

Sources: https://github.com/doctrine/dbal/blob/4.5.0/src/Connection.php, https://github.com/doctrine/dbal/blob/4.5.x/UPGRADE.md

### 1.2 Option A: a Driver Middleware that injects params (recommended, matches hakam v4)

DBAL middlewares wrap `Driver::connect(array $params)`. The middleware overrides `dbname/user/password/host` when a tenant is active. Switching to a tenant means setting the override, then calling `$connection->close()`. The next query reconnects with the new params.

DoctrineBundle registers any service that implements `Doctrine\DBAL\Driver\Middleware` (with autoconfigure). The scope is limited with `#[AsMiddleware(connections: [...], priority: N)]` (DoctrineBundle docs/en/middlewares.rst).

Code, with signatures checked against hakam v4 and the DoctrineBundle docs:

```php
// src/Tenancy/Doctrine/TenantDriverMiddleware.php
use Doctrine\Bundle\DoctrineBundle\Attribute\AsMiddleware;
use Doctrine\DBAL\Driver;
use Doctrine\DBAL\Driver\Middleware;
use Symfony\Contracts\Service\ResetInterface;

#[AsMiddleware(connections: ['tenant'])]
final class TenantDriverMiddleware implements Middleware, ResetInterface
{
    private ?array $override = null;
    public function wrap(Driver $driver): Driver { return new TenantDriver($driver, $this); }
    public function set(?array $params): void { $this->override = $params; }
    public function get(): ?array { return $this->override; }
    public function reset(): void { $this->override = null; }   // see §4: must also fail closed
}

// src/Tenancy/Doctrine/TenantDriver.php
use Doctrine\DBAL\Driver\Connection;
use Doctrine\DBAL\Driver\Middleware\AbstractDriverMiddleware;

final class TenantDriver extends AbstractDriverMiddleware
{
    public function __construct(\Doctrine\DBAL\Driver $d, private readonly TenantDriverMiddleware $m) { parent::__construct($d); }
    public function connect(#[\SensitiveParameter] array $params): Connection
    {
        $o = $this->m->get();
        if ($o === null) {
            throw new NoTenantSelected();          // fail closed: never connect to the placeholder DB
        }
        return parent::connect(array_merge($params, $o));
    }
}

// Switching
$middleware->set(['dbname' => $t->dbName, 'user' => $t->dbUser, 'password' => $secret, 'host' => $t->host]);
$em->clear();                       // drop the identity map before the switch
$tenantConnection->close();         // next query reconnects with the new params
```

Caveats:
- `Connection::getParams()` still returns the static config. hakam patches the private `params` property through reflection so `getParams()` and tools agree. Avoid that unless a tool needs it, for example Migrations `DependencyFactory`, which reads `getParams()['dbname']` in some commands [U].
- Do not switch while `$conn->isTransactionActive()`. Guard against it and throw.
- With PHP 8.4, the middleware instance is shared across the connection's lifetime. One middleware instance belongs to one connection, which is fine.

### 1.3 Option B: `wrapper_class` subclass overriding `connect()` (works in DBAL 4.5, less clean)

```php
// [U] not tested; relies on protected $_conn and protected connect()
final class TenantConnection extends \Doctrine\DBAL\Connection
{
    private ?array $tenantParams = null;
    public function switchTo(array $p): void
    {
        if ($this->isTransactionActive()) { throw new \LogicException('switch inside tx'); }
        $this->close();
        $this->tenantParams = $p;
    }
    protected function connect(): \Doctrine\DBAL\Driver\Connection
    {
        if ($this->_conn !== null) { return $this->_conn; }
        if ($this->tenantParams === null) { throw new NoTenantSelected(); }
        try {
            $this->_conn = $this->getDriver()->connect(array_merge($this->getParams(), $this->tenantParams));
        } catch (\Doctrine\DBAL\Driver\Exception $e) { throw $this->convertException($e); }
        if (!$this->isAutoCommit()) { $this->beginTransaction(); }
        return $this->_conn;
    }
}
```
```yaml
doctrine: { dbal: { connections: { tenant: { url: '%env(TENANT_TEMPLATE_URL)%', wrapper_class: App\Tenancy\Doctrine\TenantConnection } } } }
```
Downside: it copies internal logic of `connect()`, which changes between minors. Middleware (A) only uses public extension points, so prefer A.

### 1.4 `doctrine.dbal.connection_factory`

- `Doctrine\Bundle\DoctrineBundle\ConnectionFactory` is **`final` and `@internal`** (3.3.2). Its `createConnection(array $params, ?Configuration $config = null, array $mappingTypes = [])` runs **once, when the connection service is instantiated**.
- Decorating it only helps for boot-time params, such as building a connection per tenant at boot in a CLI process. It cannot switch at runtime. Not recommended.

### 1.5 One EM vs two EMs

The recommended setup is **two connections and two EMs**:
- `default`: the central DB, with Central entities such as Tenant, Plan, global users if any.
- `tenant`: the switchable connection, with Tenant entities only.

The mappings must be disjoint. `ManagerRegistry::getManagerForClass()` (used by API Platform and the `#[MapEntity]` resolver) then picks the right EM automatically.

```yaml
doctrine:
  dbal:
    default_connection: central
    connections:
      central: { url: '%env(resolve:CENTRAL_DATABASE_URL)%', server_version: '17' }
      tenant:  { url: '%env(resolve:TENANT_PLACEHOLDER_URL)%', server_version: '17' }   # placeholder; middleware overrides
  orm:
    default_entity_manager: central
    entity_managers:
      central:
        connection: central
        mappings: { Central: { type: attribute, dir: '%kernel.project_dir%/src/Entity/Central', prefix: 'App\Entity\Central' } }
      tenant:
        connection: tenant
        mappings: { Tenant: { type: attribute, dir: '%kernel.project_dir%/src/Entity/Tenant', prefix: 'App\Entity\Tenant' } }
```
- DoctrineBundle 3 removed `auto_generate_proxy_classes`, `proxy_dir` and `proxy_namespace`, which are no-ops with native lazy objects. ORM 3.5+ deprecates *not* using native lazy objects on PHP 8.4 (`$config->enableNativeLazyObjects(true)`). On PHP 8.4 with ORM 3.7 there are no proxy files at all, so no proxy cache needs per-tenant keying.
- One EM for everything, switching only the DB, works only if central tables are duplicated or reached through a second connection with DBAL. Avoid it.

### 1.6 EM reset and identity map

- `EntityManager::clear()` detaches everything, so the UnitOfWork and identity map are emptied. **Always call `clear()` on the tenant EM before switching.** Identity-map entries keyed by id would otherwise collide across tenants. id=5 in tenant A and id=5 in tenant B is exactly the leak to fear. Using UUIDv7 ids reduces the impact but does not remove the need.
- `ManagerRegistry::resetManager('tenant')` replaces a *closed* EM, for example after an exception during flush. DoctrineBundle 3.3.2 `Registry::reset()` (tagged `kernel.reset`) works like this for each initialized manager:
  - it skips the manager if it is an uninitialized lazy object;
  - it calls `clear()` if the EM is open;
  - otherwise it calls `resetManager()`.
  It does **not** close DBAL connections. Source: https://github.com/doctrine/DoctrineBundle/blob/3.3.2/src/Registry.php
- After `resetManager`, services holding the old EM get a new instance through the lazy proxy. Repositories fetched before the reset can still reference the old EM, so inject `ManagerRegistry`/repositories and don't cache EMs in custom state.

### 1.7 Caches

- **Metadata cache**: safe to share. All tenants use the same schema, which migrations must guarantee.
- **Query cache** (DQL to SQL): safe to share, because the SQL is identical with the same platform.
- **Result cache**: **must be namespaced per tenant**. Use a cache pool decorator that prefixes keys with the tenant id (hakam v4 ships `Cache/TenantAwareCacheDecorator`), or use explicit `->enableResultCache(ttl, "t{$tenantId}_...")`.
- **Second-level cache**: region keys are `entity class + id`, with no DB dimension, so **shared SLC equals a cross-tenant leak**. Disable SLC on the tenant EM. Alternatively, give each tenant its own region cache namespace. That is fiddly: the region cache is built at EM construction, so you would need EM recreation per tenant. Recommendation: leave SLC off.
- **App caches** (`cache.app`, rate limiters, lock keys, Messenger dedupe keys, API Platform HTTP cache tags): all need a tenant prefix.

---

## 2. Existing bundles

### 2.1 hakam/multi-tenancy-bundle (RamyHakam) at https://github.com/RamyHakam/multi_tenancy_bundle

- **v3.1.0** (2026-04-26) is the latest tag. It requires `php ^8.1`, `symfony/* ^6.4|^7.0`, `doctrine/doctrine-bundle ^2.7`, `doctrine/orm ^2.15|^3.0`. **It cannot be installed on Symfony 8.1 or DoctrineBundle 3.** About 56k installs and 132 stars.
- **`v4` branch**: PR #88 "Upgrade to Symfony 8 and Doctrine 3 (v4.0.0)" was merged 2026-07-22. The last commit on the branch is 2026-07-22, and **no v4 tag exists**. Issue #96 "Symfony 8" (2026-08-24) is still open. On 2026-08-25 the maintainer said v4 is "a dedicated version for Symfony 8" and gave no date.
  - v4 composer requires `php ^8.2`, `symfony/* ^8.0`, `doctrine/dbal ^4.0`, `doctrine/orm ^3.0`, `doctrine/doctrine-bundle ^3.0`, `doctrine/doctrine-migrations-bundle ^3.4|^4.0`.
  - It drops Symfony 6.4/7.0, ORM 2 and DBAL 3, the `wrapper_class` `TenantConnection`, and `server_version` config.
  - It moves services.xml to services.php.
- v4 internals (read from source):
  - `Doctrine/DBAL/TenantDriverMiddleware` holds `overrideParams` and has **no reset**.
  - `TenantDriver` extends `AbstractDriverMiddleware` and does `array_merge($params, $override)` in `connect()`.
  - `TenantConnectionSwitcher::switchConnection()` sets the override, patches the private `Connection::$params` through reflection, then calls `close()`.
  - `EventListener/DbSwitchEventListener` (`ResetInterface`) listens to `SwitchDbEvent(?string $dbIndex)`. It loads config through `TenantConfigProviderInterface` (default `hakam_tenant_config_provider.doctrine`, which reads your `TenantDbConfig` entity using `TenantDbConfigTrait`), calls `tenantEntityManager->clear()`, switches, and dispatches `TenantSwitchedEvent`.
  - `TenantResolutionListener` runs on `kernel.request` at **priority 32**, which is before the firewall at 8 and the same as RouterListener. It sets request attrs `_tenant` and `_tenant_resolved`. Resolvers: subdomain, host (`host_map`), path, header (`X-Tenant-ID`), chain. Custom resolvers implement `TenantResolverInterface { resolve(Request): ?string; supports(Request): bool }`.
  - Events: `SwitchDbEvent`, `TenantCreatedEvent`, `TenantDeletedEvent`, `TenantMigratedEvent`, `TenantBootstrappedEvent`, `TenantSwitchedEvent`.
  - Commands: `CreateDatabaseCommand`, `tenant:migrations:migrate <init|update> [dbId] [version]` (`MigrateCommand`), `DiffCommand`, `LoadTenantFixtureCommand`, and a `TenantCommand` base. Migrations build `DependencyFactory::fromEntityManager(..., new ExistingEntityManager($registry->getManager('tenant')))`.
  - Config root `hakam_multi_tenancy`, with these keys:
    - `tenant_database_className`, `tenant_database_identifier` [U, from v3 README]
    - `tenant_config_provider`
    - `tenant_connection { url, host, port (default 3306!), driver (default pdo_mysql!), charset }`
    - `tenant_migration { tenant_migration_namespace, tenant_migration_path }`
    - `tenant_entity_manager { tenant_naming_strategy, dql, mapping { type, dir, prefix, alias } }`
    - `resolver { enabled, strategy, throw_on_missing (false), excluded_paths, options { subdomain_position, base_domain, path_segment, header_name, host_map, chain_order } }`
    - `cache { prefix_separator }`
  - **For PostgreSQL, set `driver: pdo_pgsql` and `port: 5432`** explicitly.
  - **Leak risk found in v4 source.** `DbSwitchEventListener::reset()` only nulls the cached identifier. It does not clear `TenantDriverMiddleware::overrideParams` or close the connection. In a long-running worker (FrankenPHP/RoadRunner/Messenger), a later request or message that doesn't trigger a switch reuses the previous tenant's DB. That happens with `throw_on_missing: false` (default), an excluded path, or an unsupported resolver. Mitigations:
    - set `throw_on_missing: true`;
    - add your own `ResetInterface` service that nulls the override and closes the `tenant` connection;
    - or fork the bundle.
- **Verdict**: well documented and has the most features, but for Symfony 8.1 you would pin `dev-v4` (untagged, unreleased since July). Good as a reference implementation. For production, either wait for the v4.0.0 tag or write the ~300-line DIY version in §1.2 and borrow hakam's design.

### 2.2 Others (Packagist, 2026-10)

- **zhortein/multi-tenant-bundle**: v1.0.0-rc.12 (2026-09-13). Requires symfony ^7.4|^8.0, ORM ^3.5, DBAL ^3.8|^4.4, DoctrineBundle ^2.19|^3.3. Supports shared-DB or DB-per-tenant. About 557 downloads, 6 stars. Still a release candidate with low adoption.
- **saifulferoz/multi-tenancy-bundle**: v1.1.0 (2026-08-06). Symfony ^7.4|^8.0, DBAL ^4, ORM ^3. 0 downloads, 1 star. Not viable.
- fds/multi-tenancy-bundle, apajo/symfony-multi-tenancy-bundle, syeedalireza/multi-tenant-bundle: old, early-stage or experimental.
- None of them is a de-facto standard. In the Symfony ecosystem, DIY is common.

### 2.3 Laravel design references (lifecycle)

- **stancl/tenancy v3.10.1**:
  - `TenantCreated` maps to `JobPipeline::make([CreateDatabase::class, MigrateDatabase::class, SeedDatabase::class])->send(fn($e) => $e->tenant)->shouldBeQueued(true)->toListener()`. `TenantDeleted` maps to `DeleteDatabase`.
  - "Bootstrappers" run on `TenancyInitialized` and revert on `TenancyEnded`: DB connection switch, cache tag prefix, filesystem suffix, queue tenant-id injection.
  - Docs: https://tenancyforlaravel.com/docs/v3/event-system
- **spatie/laravel-multitenancy 4.2.1**:
  - The `Tenant` model `makeCurrent()` runs `SwitchTenantTask`s (for example `SwitchTenantDatabaseTask`); `forget()` reverts them.
  - Queues: `queues_are_tenant_aware_by_default`, `TenantAware`/`NotTenantAware` markers. The tenant is restored after the job, and a job fails with `CurrentTenantCouldNotBeDeterminedInTenantAwareJob` if the tenant is gone.
  - Docs: https://spatie.be/docs/laravel-multitenancy/v4/basic-usage/making-queues-tenant-aware
- **Symfony translation**:
  1. Persist `Tenant(status=provisioning)` in central.
  2. Dispatch Messenger `ProvisionTenant` (async).
  3. The handler runs `CreateDatabase`, then `CreateRole/Grant`, then `Migrate`, then `Seed`, then `status=active`. Make each step idempotent and record `schema_version`.
  4. Dispatch `TenantProvisioned`.
  - Deletion is soft by default: mark `suspended`, then archive with `pg_dump`, then `DROP DATABASE` later.

---

## 3. Tenant resolution

### Strategies
| Strategy | Notes |
|---|---|
| Subdomain `acme.app.com` | Needs a wildcard TLS certificate and DNS. The most common choice for SaaS UIs. |
| Custom domain | Look up `tenant_domains(host)`. Needs on-demand TLS, for example Caddy/FrankenPHP `on_demand_tls` [U]. |
| Header `X-Tenant-ID` | Simple for APIs, but **must be authorized**: check that the authenticated user is a member of that tenant. |
| JWT claim `tid` | Good for an API-first setup. Chicken-and-egg problem: the JWT must be verified before resolving. That only works if the user store or IdP is central, or if the token is self-contained (stateless JWT, no DB user lookup). |
| Path `/t/acme/...` | Clutters API Platform IRIs. Avoid it for API Platform. |

### Central registry table
```sql
CREATE TABLE tenant (
  id uuid PRIMARY KEY,                 -- UUIDv7
  slug citext UNIQUE NOT NULL,
  status text NOT NULL CHECK (status IN ('provisioning','active','suspended','archived','failed')),
  db_host text NOT NULL, db_port int NOT NULL DEFAULT 5432,
  db_name text UNIQUE NOT NULL,        -- e.g. t_<uuid-no-dashes>; never user-provided
  db_user text NOT NULL,               -- role per tenant
  db_secret_ref text NOT NULL,         -- pointer to Vault path / encrypted blob id, NOT the password
  schema_version text,                 -- last applied migration version
  region text, plan text,
  created_at timestamptz NOT NULL, updated_at timestamptz NOT NULL
);
CREATE TABLE tenant_domain (host citext PRIMARY KEY, tenant_id uuid NOT NULL REFERENCES tenant(id), is_primary bool NOT NULL DEFAULT false);
```

### Caching the lookup
- Cache host or slug to a `TenantConnectionConfig` DTO with a `cache.app` pool, TTL around 60 to 300 s, plus explicit invalidation on tenant update or suspension. Use Symfony cache tags (`TagAwareCacheInterface`) `tenant_{id}`.
- Do not cache the decrypted password in shared cache. Keep it in process memory only, for example in an in-memory array in a service that implements `ResetInterface`, or re-fetch it.

### Listener placement (Symfony 8.1 priorities checked in source)
- `IdleConnection\Listener` (doctrine-bridge): `kernel.request` priority **192**. It closes connections idle longer than `idle_connection_ttl` (DoctrineBundle default **600 s**).
- `RouterListener`: priority **32**.
- `FirewallListener` (SecurityBundle): priority **8**.
- **The tenant resolver must run before the firewall (> 8)** when the user provider reads users from the tenant DB. Use 32 to 64 for host/subdomain/header strategies, which don't need routing. Use about 31 if you need route attributes such as `_route`.
- For JWT-claim resolution with users in tenant DBs, decode or verify the JWT yourself in the resolver (priority > 8), or keep identities central.
- After authentication, add a membership check: a listener on `Security\Http\Event\CheckPassportEvent`, or a voter that asserts `user.tenantId === context.tenantId`.

```php
#[AsEventListener(event: KernelEvents::REQUEST, priority: 40)]
final class ResolveTenantListener { public function __invoke(RequestEvent $e): void { if (!$e->isMainRequest()) return; /* resolve → $this->tenantContext->switchTo($tenant) */ } }
```

### Messenger
- Add a `TenantStamp(string $tenantId)` on dispatch through a bus middleware: if the context has a tenant and no stamp exists, add one.
- On receive (`ReceivedStamp` present), switch to the stamp's tenant. Always clear or reset afterwards, in a `finally` block.
- Messages without a stamp run "central only" (the tenant connection fails closed).
- Use the documented middlewares `doctrine_ping_connection`, `doctrine_close_connection` (closes after each message, so the next message connects fresh with the new tenant params), `doctrine_transaction`, and `doctrine_open_transaction_logger`.

```yaml
framework:
  messenger:
    buses:
      messenger.bus.default:
        middleware:
          - App\Tenancy\Messenger\TenantMiddleware   # stamp on dispatch / switch on receive
          - doctrine_ping_connection
          - doctrine_close_connection
          - doctrine_transaction
```

### Console
- Add a global `--tenant` option in a `ConsoleEvents::COMMAND` listener:
  - `$event->getCommand()->getDefinition()->addOption(new InputOption('tenant', null, InputOption::VALUE_REQUIRED))`
  - then `$event->getInput()->bind($event->getCommand()->getDefinition())` [U]
  - Alternatively, use an explicit `#[AsCommand]` base class with the option.
- Add a `tenants:run <command> [--all|--tenant=]` wrapper that iterates active tenants and runs the inner command for each, resetting between tenants. stancl's `tenants:run` is the reference.

### Scheduler
- A Symfony Scheduler `RecurringMessage` runs once globally. Its handler fans out one message per tenant, each with a `TenantStamp`.

---

## 4. Preventing cross-tenant leaks (long-running processes)

1. **`kernel.reset`** (`ResetInterface`, `services_resetter`) runs between requests in FrankenPHP worker mode and RoadRunner, and between messages in `messenger:consume`.
   - Symfony 8.1 adds `--no-reset=N` to reset every N messages. Do not use it with tenancy.
   - Symfony 7.4+ supports FrankenPHP worker mode natively (`runtime/frankenphp-symfony` is no longer needed). Sources: https://frankenphp.dev/docs/symfony/, https://symfony.com/doc/current/messenger.html
2. On reset, every tenant-stateful service must:
   - (a) null the middleware override, so the next connect throws `NoTenantSelected` (**fail closed**);
   - (b) `close()` the `tenant` connection, rolling back any open transaction first;
   - (c) clear the tenant context.
   The `doctrine` Registry reset clears or resets EMs but **does not close connections**, so (b) is your job.
3. Never fall back to "the last tenant". Make the placeholder `TENANT_PLACEHOLDER_URL` point at a non-existent DB or a role with no CONNECT rights.
4. Messenger: use `doctrine_close_connection` plus the tenant middleware with try/finally.
5. Tag every log record with `tenant_id` through a Monolog processor, which helps detect leaks.
6. Add tests: a worker-mode integration test that sends A, then B, then a request with no tenant, and asserts DB isolation. Also test that the identity map is empty after the switch.
7. Tenant-prefix every cache pool, lock store, rate limiter, session (if any) and Mercure topic.
8. Watch connection counts. Each worker holds at most one tenant connection at a time, but PgBouncer pools per (db, user) pair. N tenants × per-tenant roles means many small pools. Use transaction pooling carefully (no session state, no prepared statements unless PgBouncer ≥1.21 `max_prepared_statements` [U]).

---

## 5. API Platform 5 (v5.0.3)

- The Doctrine ORM state providers use `ManagerRegistry::getManagerForClass($entityClass)`. For example, `ItemProvider` at v5.0.3 has the constructor `(ResourceMetadataCollectionFactoryInterface, ManagerRegistry $managerRegistry, iterable $itemExtensions, ?ContainerInterface $handleLinksLocator)`. Processors do the same.
- Because the switch happens at the DBAL level *before* the provider runs (kernel.request), **API Platform works unchanged** with either the single-switchable-connection setup or the two-EM setup with disjoint mappings.
- Caveats:
  - The resource metadata cache is shared across tenants. That is fine.
  - **The HTTP cache** (Souin/Varnish, `api_platform.http_cache.invalidation`) keys on IRIs, and IRIs are identical across tenants. The cache must vary on Host or tenant header, and purge tags must be tenant-prefixed [U].
  - **Mercure** topics and IRIs need a tenant scope.
  - **Pagination counts and filters**: no issue.
  - **Unique validators** (`UniqueEntity`) use the class's EM, which is the tenant EM. Fine.
  - **Tenant resolution must not rely on route params** for header or subdomain strategies. Path-based tenancy changes IRIs.
  - Requires php >=8.2 and symfony ^7.4|^8.0.

---

## 6. Security and credentials

- **Role per tenant** (recommended for DB-per-tenant): `CREATE ROLE t_<id> LOGIN PASSWORD '...'`, then `REVOKE CONNECT ON DATABASE t_<id> FROM PUBLIC; GRANT CONNECT ON DATABASE t_<id> TO t_<id>;`. The tenant role owns or uses only its own DB.
  - A bug in the resolver then fails at the auth level instead of silently reading another tenant.
  - Keep migrations under a separate `migrator` role that owns the schema. The runtime role gets DML only: `GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES` plus `ALTER DEFAULT PRIVILEGES`.
- **One shared app role**: simpler and better for PgBouncer pooling, but gives no defense in depth. If chosen, at least `REVOKE CONNECT ... FROM PUBLIC` and grant only to the app role.
- **Provisioning role**: has `CREATEDB` and `CREATEROLE` (PG16+ CREATEROLE semantics are narrower: the creator gets ADMIN OPTION). It is used only by the provisioning worker and never by web workers.
  - `CREATE DATABASE` cannot run inside a transaction, so run it over a connection to the `postgres` maintenance DB.
  - A fast path is `CREATE DATABASE t_x TEMPLATE tenant_template`. The template must have no active connections.
- **Storing credentials**:
  - Symfony secrets (`secrets:set`) are deploy-time and static, so they are good for the central DB URL and the master encryption key, **not** for per-tenant passwords created at runtime.
  - Options for per-tenant passwords:
    - (a) Encrypt them in the central `tenant` table with libsodium `sodium_crypto_secretbox`, keyed from a Symfony secret, and support key rotation with a key id column.
    - (b) HashiCorp Vault KV, or the Vault database secrets engine (dynamic short-lived creds). Dynamic creds conflict with long-lived worker connections and pooling.
    - (c) AWS Secrets Manager or GCP Secret Manager referenced by `db_secret_ref`.
  - Never log params (DBAL uses `#[SensitiveParameter]` on `connect`).
- Validate `db_name` and role names server-side with `^[a-z0-9_]{1,63}$` and generate them yourself. Quote identifiers with `$platform->quoteSingleIdentifier()` in DDL.
- In the template DB, run `REVOKE ALL ON SCHEMA public FROM PUBLIC` and grant usage only to the tenant and migrator roles. PG15+ already removed the default CREATE on public.

---

## Recommended architecture (summary)

- Two connections (`central`, `tenant`) and two EMs with disjoint mappings.
- A `TenantDriverMiddleware` scoped with `#[AsMiddleware(connections:['tenant'])]` that fails closed.
- A `TenantContext` service (`ResetInterface`) with `switchTo(Tenant)`:
  - assert no active transaction;
  - `em('tenant')->clear()`;
  - set the override;
  - `conn('tenant')->close()`.
- The resolver listener runs at `kernel.request` priority around 40, before the firewall, with a membership check after authentication.
- A Messenger `TenantStamp` middleware plus `doctrine_close_connection`.
- A global console `--tenant` option and a `tenants:migrate --all` command that loops tenants. Use the Doctrine Migrations `DependencyFactory::fromConnection` / `ExistingConnection` per tenant, with a separate `migrations/Tenant` path.
- SLC off. Result cache and app caches tenant-prefixed.
- A role per tenant with credentials encrypted in central or in Vault.
- Use hakam v4 as a reference, or adopt it once `v4.0.0` is tagged, fixing the reset gap.
