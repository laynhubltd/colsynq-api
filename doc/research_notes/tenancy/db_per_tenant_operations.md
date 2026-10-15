# Database-per-tenant (PostgreSQL) — operations and architecture notes

Context: Symfony 8.1 modular monolith, Messenger + Doctrine transport, transactional outbox/inbox per module, event-only inter-module comms. Researched 2026-10-12. Current versions seen: Symfony 8.1 (8.2 dev), PostgreSQL 18.6 (19 in beta 4).
`[U]` = unverified snippet or claim (my synthesis, or a secondary source I did not confirm).

---

## 1. Tenancy model trade-offs

### Azure SQL "Multitenant SaaS patterns" comparison table
Source: https://learn.microsoft.com/en-us/azure/azure-sql/database/saas-tenancy-app-design-patterns (updated 2026-08)

| Measure | Standalone app | DB-per-tenant | Sharded multitenant |
|---|---|---|---|
| Scale | 1–100s | 1–100,000s | 1–1,000,000s |
| Isolation | High | High | Low (except for single-tenant shards) |
| DB cost/tenant | High (sized for peaks) | Low (pools) | Lowest |
| Perf monitoring | Per-tenant only | Aggregate + per-tenant | Aggregate |
| Dev complexity | Low | Low | Medium (sharding) |
| Ops complexity | Low–High | Low–Medium ("patterns address complexity at scale") | Low–High |

- Azure SQL says it manages "well over 100,000 databases". Example: one 1,000-tenant DB with 20 indexes becomes 20,000 indexes across 1,000 tenant DBs. Azure relies on automatic indexing for that; self-hosted Postgres has no equivalent.
- Per-tenant point-in-time restore (PITR) is trivial with DB-per-tenant and "unacceptably slow" with a shared DB.
- Hybrid model: every DB keeps a `tenant_id` column, so a tenant can move between a shared DB and its own DB. Example tiers: free trial → shared, basic → less dense shared, premium → dedicated.
- Pools: "1,000 databases per pool, with 1,000 pools" could reach millions of tenants, "at the risk of becoming unwieldy to manage."

### Azure Architecture Center: storage/data approaches
Source: https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/approaches/storage-data (updated 2026-10)

- Isolate a tenant, or group it with similar tenants, when it needs **its own encryption keys**, **its own backup/restore policy**, or **a different geography**.
- Antipatterns:
  - Table-per-tenant.
  - Per-tenant column customization.
  - **Manual schema changes.** Track each tenant's schema version in a lookup table.
  - **Version dependencies.** The app must stay backward compatible with at least one previous schema version, because tenants migrate at different times.
- At larger fleets, the app tier can detect a DB's schema version and trigger the upgrade, instead of a deployment pipeline looping over every DB.
- Reporting: "it becomes cumbersome to run queries on each database individually". Have tenants publish data to a central warehouse instead.
- RLS: "Many multitenant solutions don't use row-level security because of those complexities."
- Recommends Deployment Stamps even when using shared or sharded DBs.

Azure PostgreSQL service guide (https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/service/postgresql):
- Elastic clusters (Citus) schema-based sharding is described as "well-suited for workloads that have between 1 and 10,000 tenants".
- PgBouncer is built in. Postgres' process-per-connection model makes many idle connections expensive.
- Customer-managed keys (CMK) per server. Deleting a tenant's key makes its data inaccessible (crypto-shredding).

### AWS: silo / pool / bridge
- SaaS Lens: https://docs.aws.amazon.com/wellarchitected/latest/saas-lens/silo-pool-and-bridge-models.html
  - Silo = dedicated resources, but still with shared identity, onboarding and ops. Pool = shared. Bridge = a mix per service or tier.
- PostgreSQL guide: https://docs.aws.amazon.com/prescriptive-guidance/latest/saas-multitenant-managed-postgresql/partitioning-models.html
  - AWS **silo** = an *instance* per tenant.
  - AWS **bridge** = a database (or schema) per tenant on a *shared instance*. So **DB-per-tenant on a shared server is "bridge" in AWS terms.** It still has noisy-neighbour issues at instance level, adds per-tenant provisioning, and needs a tenant→DB mapping. https://docs.aws.amazon.com/prescriptive-guidance/latest/saas-multitenant-managed-postgresql/bridge.html
  - Silo pros: compliance, no cross-tenant impact, tenant-level tuning and availability. Cons: agility, no central management, deployment complexity, cost.

### Threshold numbers
| Figure | Source |
|---|---|
| DB-per-tenant suits "5 or 50 tenants"; for thousands, use shared tables with a tenant column | Citus: https://docs.citusdata.com/en/latest/articles/designing_saas.html |
| Schema changes go "from seconds to minutes to hours across thousands of different tenants" | Citus lessons: https://www.citusdata.com/blog/2016/12/18/schema-sharding-lessons/ |
| Framework connection pools need extra work to span DBs or schemas | Citus lessons (same link) |
| Practitioners report breakdown at about 500 tenants | Generic guide, no named case `[U]` |
| A 100+ DB migration fan-out was "complex and error-prone"; author later called DB-per-tenant an overreaction | dev.to restaurant SaaS write-up `[U]` |
| Atlassian: up to about 4,000 DBs per cluster | Section 6 |

Practical read `[U]`:
- Self-managed Postgres DB-per-tenant is comfortable at tens to low hundreds of tenants per server.
- It is workable into the low thousands per server, but only with automation for provisioning, migration fan-out, monitoring, pooling and rebalancing.
- Beyond that, go hybrid: pool small or trial tenants and silo the big or regulated ones.

### Comparison summary `[U synthesis]`
| | Shared tables + RLS | Schema-per-tenant | DB-per-tenant | Hybrid |
|---|---|---|---|---|
| Isolation | Logical; a bug or missing policy can leak data | Namespace; `search_path` mistakes leak data | Strong; connection is scoped to one DB | Per tier |
| Noisy neighbour | Highest | High (same DB and catalog) | Instance-level only | Isolate the big tenants |
| Cost/tenant | Lowest | Low | Medium (catalog, connections, about 5k files per DB at Atlassian) | Tuned |
| Tenant count limit | Millions | Catalog bloat at thousands of schemas | Low thousands per server | High |
| Cross-tenant reporting | Plain SQL | `UNION` across schemas | Needs ETL/CDC or FDW | ETL |
| Per-tenant restore | Hard (restore aside, extract rows) | `pg_dump -n` | `pg_dump -d` or a per-DB clone | Mixed |
| Erasure | `DELETE` by `tenant_id` across all tables, plus backups | `DROP SCHEMA` | `DROP DATABASE` | Mixed |
| Residency | One region | One region | DB can sit on a server in the required region | Yes |

---

## 2. Messaging (Messenger, outbox, inbox) with DB-per-tenant

### Key Symfony facts
Source: https://symfony.com/doc/current/messenger.html (8.1)

- **Doctrine transport DSN:** `doctrine://<connection_name>?table_name=...&queue_name=...`.
  - Defaults: `table_name=messenger_messages`, `queue_name=default`, `redeliver_timeout=3600`.
  - Set `auto_setup: false` in production and create the table through a migration.
- **PostgreSQL LISTEN/NOTIFY:**
  - `use_notify` defaults to true. `check_delayed_interval` and `get_notify_timeout` default to 60000 ms.
  - `PostgreSqlNotifyOnIdleListener` is new in 8.1.
  - Docs: "When consuming from multiple PostgreSQL transports, all transports must share the same DBAL connection and the same `table_name`. Otherwise, a `LogicException` is thrown when the worker starts."
  - **So a single `messenger:consume` process cannot consume N per-tenant-DB Postgres transports.** Options: one worker per tenant DB, or one central queue.
  - The docs also advise against several transports sharing the same `queue_name` with `use_notify`.
- **NOTIFY is per database.** It reaches listeners "in the current database" (https://www.postgresql.org/docs/current/sql-notify.html), so there is no cross-DB wakeup.
- **Consume flags:**
  - `messenger:consume` accepts regex transport names (8.1), e.g. `scheduler_.*`.
  - Worker limits: `--limit`, `--memory-limit`, `--time-limit`, `--failure-limit`, `--keepalive` (Doctrine).
- **Doctrine middleware:** `doctrine_ping_connection`, `doctrine_close_connection` (frees connections, which matters with many DBs), `doctrine_transaction`.
  - `dispatch_after_current_bus` must come before `doctrine_transaction`.
- **`DeduplicateStamp`:**
  - Signature `(key, ttl=300, onlyDeduplicateInQueue=false)`. Uses the Lock component.
  - Lock is released after final failure (8.1).
  - **Lock keys must include the tenant id** if the lock store is shared.
- **Custom middleware:** runs on dispatch *and* on receive (check `ReceivedStamp`). This is the hook to set and restore tenant context.
- **Stamps:** must be serializable by the configured serializer. `DefaultStampsProviderInterface` lets messages add stamps automatically.

### Where to put the transport, outbox and inbox

**Option A — everything in the tenant DB (recommended for atomicity).**
- Outbox table, inbox table (processed message ids) and `messenger_messages` all live in each tenant DB.
- Business write + outbox insert commit in one local transaction. Inbox dedupe happens in the same transaction as the handler's writes.
- Consequence: workers must reach N DBs:
  - (A1) One supervisor-managed worker per tenant, with transports generated from the registry. This scales badly past about 50–100 tenants; each idle worker holds a connection and polls or LISTENs.
  - (A2) **Per-tenant outbox + relay to a central transport**, recommended `[U]`:
    - A small relay iterates tenant DBs fairly (round robin, per-tenant cursor, `FOR UPDATE SKIP LOCKED`, batch N).
    - It forwards rows to a central transport: a Doctrine transport in the landlord or a dedicated "bus" DB, or a real broker (RabbitMQ, Redis Streams, SQS).
    - It stamps each message with `TenantStamp(tenantId)`.
    - Consumers receive from the central transport. Middleware switches the tenant connection from the stamp, then the handler writes and records the inbox entry in the **tenant** DB, in one transaction.
- NServiceBus maintainer guidance: polling multiple tenant DBs from one connector is fine for "<10" tenant DBs. Beyond that, polling cost grows with tenant count "regardless if that tenant is busy or not". Shard the relay by tenant group instead. https://discuss.particular.net/t/outbox-in-an-asp-net-core-scenario/966/27
- Alternative relay: logical decoding/CDC per tenant DB. But that means one replication slot per DB, and `max_replication_slots` defaults to 10 — see section 3.
- Idle-cost mitigation `[U]`: keep a central "dirty tenants" set (Redis SET or a landlord table) written after commit. The relay polls only dirty tenants, plus a slow sweep of all tenants to catch lost signals.

**Option B — central transport, outbox in the tenant DB.** Same as A2. The outbox must stay in the tenant DB to keep atomicity.

**Option C — central outbox and transport only.**
- You lose atomic commit with tenant data, because Postgres has no cross-DB transactions; you would need 2PC or prepared transactions (avoid).
- Only acceptable for landlord-originated events (tenant created, plan changed).

Precedents:
- **Tenancy for Laravel** stores queued jobs centrally ("stored in the `jobs` table in the central database"). Its `QueueTenancyBootstrapper` re-initializes the tenant before each job. Note it does not use a per-tenant outbox. https://tenancyforlaravel.com/docs/v3/queues
- **Ecotone** (PHP, supports Symfony):
  - `MultiTenantConfiguration::create(tenantHeaderName: 'tenant', tenantToConnectionMapping: [...])`.
  - The tenant header propagates through messages and switches the DBAL connection or ObjectManager per message.
  - Multi-tenant dead letter, tenant-aware console commands.
  - "Dynamic Message Channels" (Enterprise) do round-robin over channels and per-tenant throttling or isolation.
  - https://docs.ecotone.tech/messaging/multi-tenancy-support/getting-started/symfony-and-doctrine-orm
  - https://docs.ecotone.tech/modelling/asynchronous-handling/dynamic-message-channels
  - Header-based DB routing is free in 1.x and moves to Enterprise in the next major.
- **Ibexa** (Symfony): runs one worker per siteaccess/repository with the Doctrine transport. https://doc.ibexa.co/en/1.7/infrastructure_and_maintenance/background_tasks

### Sketch `[U]`

```php
final readonly class TenantStamp implements StampInterface {
    public function __construct(public string $tenantId) {}
}

final class TenantContextMiddleware implements MiddlewareInterface {
    public function __construct(private TenantContext $ctx, private TenantConnectionSwitcher $switcher) {}

    public function handle(Envelope $e, StackInterface $stack): Envelope {
        if (null === $e->last(ReceivedStamp::class)) {
            // dispatch side: require/attach tenant
            if (null === $e->last(TenantStamp::class) && $this->ctx->has()) {
                $e = $e->with(new TenantStamp($this->ctx->id()));
            }
            return $stack->next()->handle($e, $stack);
        }
        $tenant = $e->last(TenantStamp::class)
            ?? throw new UnrecoverableMessageHandlingException('missing tenant');
        $previous = $this->ctx->current();
        $this->switcher->switchTo($tenant->tenantId); // close + reconnect tenant conn, clear EM
        try {
            return $stack->next()->handle($e, $stack);
        } finally {
            $this->switcher->restore($previous);
        }
    }
}
```

Middleware order on the receiving bus `[U]`: `tenant_context` → `doctrine_ping_connection` → `dispatch_after_current_bus` → `doctrine_transaction` → handler (inbox check inside the transaction) → `doctrine_close_connection`.

Rules:
- Always `clear()` the EntityManager and reset services (`services_resetter`, `kernel.reset`) between tenants.
- Never keep tenant-scoped services alive across messages.
- Inbox key = `(message_id, handler)` in the **tenant** DB.
- Dead letter / failure transport: keep it central, with the tenant stamp. Per-tenant replay tooling filters on the stamp.

### Scheduler per tenant `[U]`
- Symfony Scheduler is process-level.
- Option 1: a single central schedule emits `RunForAllTenants(job)`, and a fan-out handler dispatches one message per active tenant (with `TenantStamp`, and `DeduplicateStamp` key `job:tenant:period`).
- Option 2, for per-tenant cron config: store schedules in the landlord DB, and have a central ticker dispatch the due tenant jobs.
- Avoid one scheduler process per tenant.

---

## 3. Landlord (central) database

Contents `[U synthesis]`:
- **tenants**: id, slug, status (provisioning / active / suspended / offboarding / deleted), plan, region, `db_host`/`db_name`/`db_secret_ref`, `schema_version`, `created_at`, `deleted_at`.
- domains / subdomains.
- plans, subscriptions, billing, usage metering.
- **global identity**: users, credentials, MFA, plus a membership table (user↔tenant, role).
  - Global identity enables one login across tenants and is the common SaaS choice (AWS: silos still share identity and onboarding).
  - Per-tenant users give stronger erasure (drop DB) but duplicate accounts.
- feature flags.
- provisioning / migration job ledger.
- relay cursors.
- central failure transport.
- platform audit log (admin actions, impersonation, tenant lifecycle).
- tenant-scoped audit stays in the tenant DB.

Cross-tenant queries:
- Postgres: "It is not possible to access more than one database per connection". Use `postgres_fdw` (recommended) or `dblink` (older). https://www.postgresql.org/docs/current/manage-ag-overview.html
- The same page warns that DBs in one cluster share WAL, which "impacts backup and recovery options".
- Teamdash (MySQL → Postgres): cross-DB queries became `dblink(...) UNION dblink(...)`, with a new connection per dblink. https://karlerss.com/migrating-from-mysql-to-postgres/
- → Use FDW/dblink only for ad-hoc ops. For reporting, use **ETL/CDC into a warehouse** (Azure guidance above).

CDC limits:
- **Debezium PostgreSQL connector = one database per connector, one replication slot each.** A multi-DB connector request was closed "Can't Do" (https://issues.redhat.com/browse/DBZ-6374).
- `max_replication_slots` defaults to 10. Raise it, plus `max_wal_senders`.
- **Abandoned slots retain WAL indefinitely.** Monitor `pg_replication_slots` and drop slots at offboarding.
- At hundreds of tenants, prefer the app's own event stream (the outbox relay already centralizes integration events → warehouse sink), or batch ETL per tenant DB (nightly incremental by `updated_at`), over per-DB CDC `[U]`.

Logical replication restrictions (PG 18, https://www.postgresql.org/docs/current/logical-replication-restrictions.html):
- No DDL replication.
- Sequences not replicated; fix them before cutover.
- No large objects.
- Tables only.
- TRUNCATE fails across foreign keys.

---

## 4. Operations

### Backup and restore
- **PITR / WAL archiving is whole-cluster only.** Docs: "can only support restoration of an entire database cluster, not a subset". pg_dump output cannot be used with WAL replay. https://www.postgresql.org/docs/current/continuous-archiving.html
- Single-tenant PITR therefore means restoring the cluster to a scratch instance at time T, then `pg_dump -d tenant_x` → restore into production (rename/swap) `[U]`. Automate and rehearse this.
- Add nightly logical `pg_dump -Fc -d tenant_x` per DB for quick single-tenant restore and export. Also `pg_dumpall --globals-only` for roles.
- Fan the dumps out in parallel with a concurrency cap; total backup time grows linearly with tenant count `[U]`.
- Managed options: Azure SQL/HorizonDB/Neon offer per-DB or per-project restore. Neon claims restoring a tenant DB "within seconds" and branching (https://neon.com/use-cases/database-per-tenant).
- RDS/Aurora PITR is instance-level, same as above.

### Offboarding, erasure and export (NDPA 2023, Nigeria)
- Data-subject rights are in Part VI (s.34 rights of a data subject, s.35 withdrawal of consent, s.36 objection, s.37 automated decisions, s.38 portability).
- s.38 portability depends on future Commission regulations.
- Exact sub-paragraph for erasure is unclear: secondary sources cite s.34(1) or s.35(d) `[U]`. Verify against the gazetted Act / NDPC GAID 2025 (https://ndpc.gov.ng/wp-content/uploads/2025/07/NDP-ACT-GAID-2025-MARCH-20TH.pdf).
- s.24 principles include **storage limitation**.
- s.40: breach notification to the Commission within **72 hours**.
- Sources: https://www.mondaq.com/nigeria/data-protection/1366934/the-nigerias-data-protection-act-2023-a-look-at-key-provisions , https://www.afriwise.com/blog/data-breaches-compliance-obligations-under-the-nigerian-data-protection-act-2023 , https://lawpavilion.com/blog/safeguarding-privacy-a-descriptive-analysis-of-the-nigeria-data-protection-act-2023

Offboarding runbook `[U]`:
1. Mark the tenant `offboarding`, block logins, stop the relay and scheduler for it.
2. Export: `pg_dump` plus a JSON/CSV export through the app (portability), and storage objects under the tenant prefix. Deliver through a signed URL that expires.
3. Retention grace period, e.g. 30–90 days, by contract.
4. `DROP DATABASE tenant_x WITH (FORCE)` (PG13+).
5. Delete the storage prefix, Redis keys and search index/docs. Drop replication slots and PgBouncer entries.
6. Delete the tenant KMS key: crypto-shreds backups that still contain the tenant.
7. Keep a minimal landlord tombstone for legal or billing retention.

Backups are the hard part. Either expire them by retention policy and document it, or rely on per-tenant key destruction.

### Monitoring per tenant
- `pg_stat_database` has one row per DB, so with DB-per-tenant it is the per-tenant view:
  - Current state: `numbackends` is the only current-state column; all others are cumulative, so diff samples.
  - Workload: `xact_commit`/`xact_rollback`, `blks_hit`/`blks_read`, `tup_*`, `temp_bytes`, `deadlocks`, `session_time`/`active_time`/`idle_in_transaction_time`, `sessions_*`.
  - https://www.postgresql.org/docs/current/monitoring-stats.html
- Size: `pg_database_size(datname)`.
- Query detail: `pg_stat_statements` keyed by `dbid` `[U]`.
- Add app-level metrics tagged with `tenant_id`: relay lag, outbox depth, inbox duplicates, migration version.
- Watch out for exporter label cardinality at thousands of DBs `[U]`.

### Connection pooling (PgBouncer) at many-DB scale
https://www.pgbouncer.org/config.html
- **Pools are per (database, user) pair.**
- Defaults: `default_pool_size=20`, `max_client_conn=100`, `max_db_connections=0` (unlimited), `pool_mode=session`.
- `"*"` fallback DB entry auto-creates pools. These are freed after `autodb_idle_timeout=3600`s; `server_idle_timeout=600`.
- `auth_user` + `auth_query` let you avoid listing every tenant role.
- Transaction mode supports protocol-level prepared statements (`max_prepared_statements` default 200).
- **Math:** N tenant DBs × pool size can exceed Postgres `max_connections`, e.g. 500 × 20 = 10,000.
  - Use small per-DB pools (`pool_size` 2–5), `max_db_connections` per DB, a global cap via Postgres `max_connections`, `min_pool_size=0`, short `server_idle_timeout`, transaction mode.
  - Use the `doctrine_close_connection` middleware in workers.
- Using one app role for all tenant DBs keeps the pool count at N. Per-tenant roles also give N pools, but credentials are easier to scope.
- Atlassian connects with **per-tenant credentials** to tenant endpoints that refresh periodically.

### Moving a tenant between servers
- Atlassian rebalancing: small DBs by backup and restore; large DBs by logical replication (DB stays online); about 1,000 moves per day routinely.
- Procedure: mark the tenant read-only/offline in the landlord → copy → fix sequences → flip `db_host` in the registry → invalidate connection caches → online.
- The Azure split/merge tool does the same: "marking affected tenants as offline before moving them", then updates the catalog.

### Tenant-specific encryption keys
- Postgres has no per-database TDE.
- Options:
  - App-level envelope encryption: a per-tenant DEK, wrapped by a KMS key per tenant, for sensitive columns and files.
  - Disk/instance-level CMK (Azure CMK, RDS KMS), which is per server, so dedicated servers for "bring your own key" tenants.
- Deleting a key gives crypto-shredding for erasure.

### Cache, Redis, files, search
- Redis/cache:
  - Prefix every key with `t:{tenantId}:` and include the tenant in the Symfony cache pool namespace / `prefix_seed`, or use a per-tenant pool decorator `[U]`.
  - Lock (Deduplicate) and RateLimiter keys must include the tenant.
  - Erasure: `SCAN MATCH t:{id}:*` + `UNLINK`.
- Files: object storage key prefix `tenants/{tenantId}/...`, or a bucket per tenant for residency/BYOK.
  - Use valet-key / signed URLs scoped to the prefix (Azure Valet Key pattern).
- Search:
  - Meilisearch recommends **one shared index + tenant filter or tenant tokens** over index-per-tenant, because tasks across indexes are processed sequentially. https://www.meilisearch.com/blog/multi-tenancy-guide
  - Elasticsearch: `cluster.max_shards_per_node` default 1,000 (replicas count), so index-per-tenant consumes shards linearly. https://www.elastic.co/docs/reference/elasticsearch/configuration-reference/miscellaneous-cluster-settings
  - Use shared indices with filtered aliases, and a dedicated index for big tenants `[U]`.

### Schema migrations fan-out `[U]`
- Use separate Doctrine Migrations configs: `migrations/Landlord` and `migrations/Tenant`.
- A `tenants:migrate` command iterates the registry with bounded concurrency. It records `schema_version` per tenant, continues on failure, and supports retry.
- Use expand/contract changes so app version N works with schema N-1 (Azure antipattern guidance).
- Provision new tenants from `CREATE DATABASE ... TEMPLATE tenant_template` (template kept at latest schema), or run migrations on create.
- Symfony bundle precedent: `hakam/multi-tenancy-bundle` (SwitchDbEvent, separate Main/Tenant migrations, `TenantTestTrait::runInTenant()`). Listed for Symfony 6/7; check Symfony 8 support. https://github.com/RamyHakam/multi_tenancy_bundle

---

## 5. Testing in Symfony

- **DAMA doctrine-test-bundle** (https://github.com/dmaicher/doctrine-test-bundle):
  - Wraps each test in a transaction and rolls back. Supports PHPUnit 11, 12 and 13.
  - `enable_static_connection: true` covers all configured DBAL connections. A map enables only some:
    ```yaml
    dama_doctrine_test:
      enable_static_connection:
        landlord: true
        tenant: true
      connection_keys:   # optional: share/split driver connections
        tenant: tenant
    ```
  - Only *configured* connections are covered. A connection whose params are switched at runtime (wrapper class or reconnect to a different dbname) reuses the static driver connection keyed by connection name, so switching tenants inside a test may silently hit the same physical DB `[U]`.
  - Prefer fixed test tenants: `tenant_a_test`, `tenant_b_test`, pre-created and pre-migrated, each a configured connection or using an explicit connection key.
  - Debug with `StaticDriver::commit()`.
  - DBAL < 4 needs `use_savepoints`.
- **Cross-tenant tests:** "tenant A cannot see tenant B" requires two real DBs.
- **Outbox/relay tests:** DAMA's rollback hides committed outbox rows from a separate relay connection, so mark such tests to disable DAMA (or use a real commit + truncate) `[U]`.
- **Parallel runs (paratest):**
  - Suffix DB names with `TEST_TOKEN`, e.g. `dbname_suffix: '_test%env(default::TEST_TOKEN)%'` `[U]`. That is a common recipe; the current Symfony testing page only shows the `_test` suffix.
  - Create landlord + N tenant DBs per worker in bootstrap, from a migrated `TEMPLATE` DB, which is fast.
- **Messenger in tests:** use the `in-memory://` or `sync://` transport for unit tests, and assert the `TenantStamp` is present. Use one integration test with a real Doctrine transport + relay.

---

## 6. Case studies

**Atlassian Jira** (https://www.atlassian.com/blog/atlassian-engineering/migrating-jira-database-platform-to-aws-aurora; coverage https://www.theregister.com/2025/07/07/asia_tech_news_in_brief/):
- Scale: about **4 million Postgres DBs** (one per Jira tenant) across about 3,000 instances in 13 regions, an average of about 1,300 DBs per instance (computed).
- Clusters host **up to about 4,000 DBs**. Each Jira DB is about **5,000 files** on disk.
- Bridge model: most tenants on shared instances, very large tenants on dedicated Aurora.
- Routine rebalancing about 1,000 DB moves per day. During the Aurora migration, 38,000 per day average, peak about 90,000.
- Pitfall: replica startup timed out because a status check enumerated **millions of files**. AWS's fix was to reduce tenants per instance ("draining").
- Pitfall: too much destination concurrency degraded clusters.
- Cutover under 3 minutes per instance: lock SQL users, promote, unlock on the destination, feature-flag endpoint override.
- Result: m5.4xlarge RDS → r6.2xlarge Aurora, up to 15 readers autoscaled.

**Postgres catalog/object-count pitfalls:**
- A report of about 200k tables in one DB showed `pg_attribute` > 200 GB of bloat. Connections hung at startup.
- More autovacuum workers did not help, because the I/O cost limit is shared; lower `autovacuum_vacuum_cost_delay` instead.
- https://www.postgresql.org/message-id/20151030133252.3033.4249%40wrigleys.postgresql.org
- This is relevant mostly to schema-per-tenant, but shared catalogs (`pg_database`, `pg_authid`) and per-DB files grow with DB-per-tenant too.

**Citus / Apartment (Rails):** schema-per-tenant pain at thousands of tenants — migrations, pooling. https://www.citusdata.com/blog/2016/12/18/schema-sharding-lessons/

**Teamdash:** DB-per-tenant on Postgres; the main pain was cross-DB queries via dblink. https://karlerss.com/migrating-from-mysql-to-postgres/

**Neon:** markets project-per-tenant: scale-to-zero, millisecond provisioning, per-tenant restore, quotas. Customer Opusflow. https://neon.com/use-cases/database-per-tenant

---

## 7. Recommendations for Colsynq `[U synthesis]`

1. Start **hybrid-ready**:
   - DB-per-tenant (bridge: many DBs per Postgres server), but keep `tenant_id` on aggregate roots so you can pool tiny or trial tenants later.
   - Keep the landlord in its own DB, ideally on a separate server.
2. Per tenant DB: module tables, per-module outbox and inbox tables, no Messenger consume tables.
3. A **relay** (sharded by tenant hash, fair round robin, dirty-tenant hints) forwards outbox rows to a **central transport** with `TenantStamp`.
   - The central transport is a Doctrine table in a dedicated "bus" DB, or RabbitMQ/Redis later.
   - Consumers switch tenant connection via middleware; inbox dedupe and writes commit in the tenant DB.
4. Central scheduler fans out per tenant. Central failure transport, keyed by tenant.
5. PgBouncer in transaction mode:
   - Small per-DB pools.
   - `max_db_connections` per DB.
   - `"*"` fallback + `auth_query`.
   - `doctrine_close_connection` in workers.
6. Ops tooling from day one:
   - `tenant:provision` from `TEMPLATE`.
   - `tenants:migrate` fan-out with a per-tenant version.
   - Per-tenant `pg_dump` nightly + cluster PITR, with a documented single-tenant restore path.
   - `tenant:export`, `tenant:offboard` (drop DB, storage prefix, Redis prefix, search docs, KMS key).
   - `tenant:move` (dump/restore, or logical replication for big tenants).
7. Reporting through the event stream / ETL into a warehouse. No runtime cross-DB queries.
8. Plan a per-server ceiling (e.g. a few hundred to about 1,000 DBs, well under Atlassian's about 4,000) and add servers or rebalance before hitting it.
