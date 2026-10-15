# Symfony Mechanics for an Event-Only Modular Monolith (as of 12 October 2026)

Stack: Symfony 8.1, API Platform 5, Doctrine ORM 3.7 / DBAL 4, PHP 8.4/8.5, PostgreSQL.
Companion to `event_driven_modular_monolith.md` (the patterns); this file covers the framework mechanics.

Markers:
- **[P]** primary: official docs, the project's own changelog/UPGRADE file, or its Packagist/GitHub release metadata, read during this research.
- **[S]** secondary: reputable blog, tutorial, news site or third-party README.
- **[U]** unverified: from general knowledge or inference, not confirmed against a source in this session. Check before relying on it.

Installed in this repo (composer.lock, 12 Oct 2026): `symfony/framework-bundle` v8.1.8, `api-platform/symfony` v5.0.2, `doctrine/orm` 3.7.4, `doctrine/dbal` 4.5.0, `doctrine/doctrine-bundle` 3.3.2. `symfony/messenger`, `symfony/object-mapper`, `symfony/lock`, Deptrac and PHPat are **not yet installed**.

---

## 1. Versions

### 1.1 Symfony release timeline

| Version | Released | End of support | PHP | Latest patch (12 Oct 2026) |
|---|---|---|---|---|
| 7.4 LTS | Nov 2025 | bugs Nov 2028, security Nov 2029 | >= 8.2 | 7.4.20 |
| 8.0 | Nov 2025 | July 2026 (unmaintained) | >= 8.4 | 8.0.16 |
| 8.1 | May 2026 (8.1.0 on 29 May 2026) | January 2027 | >= 8.4 | 8.1.8 |
| 8.2 | November 2026 (in development) | July 2027 | >= 8.4 | none yet (`next: 8.2.0`) |
| 8.4 LTS | expected Nov 2027 | | | |

- [P] Dates, PHP minimums and patch numbers come from https://symfony.com/roadmap and https://symfony.com/releases.json (`"lts":"7.4.20","stable":"8.1.8","next":"8.2.0"`, `maintained_versions: 6.4, 7.4, 8.1, 8.2`).
- [P] 8.1.0 release date: symfony.com blog, releases category.
- [S] The next LTS is 8.4, due November 2027 (endoflife.date and a secondary source). Symfony's cadence (minor releases every six months, x.4 is LTS) supports this.
- **Implication.** 8.1 is unsupported from January 2027, so plan the upgrade to 8.2 (November 2026). Several features below (the Messenger `outbox` option, the bus `messages` option, `deduplication.lock_factory`, `serializedTypeNameAliases`) are **8.2-only**.

### 1.2 API Platform 5

- [P] v5.0.0 was released on 16 Sep 2026. The latest is v5.0.3 (9 Oct 2026). 4.x is still patched (4.4.x), and 4.3.x gets occasional fixes. Sources: GitHub releases and Packagist.
- [P] 5.0 "contains every change shipped in v4.4.0, plus the removals and breaking changes" (CHANGELOG.md, v5.0.0). Requirements: Symfony `^7.4 || ^8.0` (6.4 dropped), PHP >= 8.2, and `api-platform/doctrine-orm` needs `doctrine/orm ^3.3` for PARTIAL support.
- [P] Breaking changes and defaults, from https://api-platform.com/docs/core/upgrade-guide/ :
  - `ObjectMapperProcessor` is **removed**. Use `ObjectMapperInputProcessor` plus `ObjectMapperOutputProcessor`.
  - `DeserializeProvider` drops `TranslatorInterface`. It no longer derives `api_assign_object_to_populate` from the HTTP method, so custom controllers that call `provide()` must set that flag.
  - `SerializerAwareProviderInterface` and `SerializerAwareProviderTrait` are removed. Inject `SerializerInterface` instead.
  - `ValidationException` only accepts a `ConstraintViolationListInterface`.
  - The legacy PropertyInfo `Type` APIs are removed (`getBuiltinTypes` becomes `getNativeType`).
  - `UniqueConstraintViolationException` now maps to 422 by default.
  - A denormalization error on a property with a constraint now returns 422 with a ConstraintViolation instead of 400.
  - `jsonapi.use_iri_as_id` now defaults to `false`.
  - These config keys are removed: `resource_class_directories` (classes are autoconfigured), `enable_link_security` (always on), `validator.query_parameter_validation` (always on), and `http_cache.invalidation.varnish_urls`/`xkey`.
  - `ApiTestCase` moves to the `api-platform/test` package.
  - Doctrine filters: `RangeFilter` is deprecated in favour of `ComparisonFilter [between]`. `AbstractFilter`-based filters and `FilterInterface::getDescription()` are deprecated, and their removal is deferred to 6.0.
- [P] New features in 4.4/5.0: the HTTP `QUERY` method, OpenAPI 3.2, `routePriority`, `Parameter` attributes on properties, the state option `repositoryMethod` for the Doctrine query builder (#7115), `ChainFilter`, `throwOnNotFound`, and voter reasons exposed.

**State providers/processors and resources separated from entities** [P] (https://api-platform.com/docs/core/dto/, 5.0 docs)
- The `#[ApiResource]` class is a DTO that defines the API contract. It is linked to the entity with `stateOptions: new Options(entityClass: X::class)` (`ApiPlatform\Doctrine\Orm\State\Options`). With `stateOptions`, the built-in Doctrine provider and processor load and persist the **entity**, and Symfony **Object Mapper** maps entity and DTO in both directions.
- This requires `symfony/object-mapper ^7.4 || ^8.0` and a class-level `#[Map(source: Entity::class)]` on the resource. Property-level `#[Map(source: 'title')]` renames fields, and `transform:` converts values. Input DTOs use `#[Map(target: Entity::class)]` with `input:`/`output:` on operations.
- For custom business actions, write a processor. It reads the loaded entity from `$context['request']->attributes->get('read_data')`, applies domain logic, and delegates to `api_platform.doctrine.orm.state.persist_processor`.
- [P] **Doc inconsistency.** The DTO page still mentions `ObjectMapperProcessor` decorating the chain, while the upgrade guide says it was removed in 5.0. Trust the upgrade guide: wire `ObjectMapperInputProcessor` and `ObjectMapperOutputProcessor`.
- [P] A known 4.2 issue (#7540) noted that automatic mapping could not be disabled. Check this if you want manual control.
- **Recommendation for an event-only modular monolith** [U]. The design inference: do **not** let the generic Doctrine persist processor write aggregates, because that bypasses the domain and records no events. Use `stateOptions` plus Object Mapper for **read** models and collections. For writes, give operations a custom `ProcessorInterface` that maps input to a command and dispatches it on `command.bus`. The command handler loads the aggregate, calls the domain method and records events. The processor then returns the output DTO.

### 1.3 Doctrine ORM 3.x

- [P] Versions: 3.7.0 (7 Sep 2026), 3.7.5 latest (9 Oct 2026), and 3.6.9. Source: GitHub releases.
- [P] Changes in 3.7:
  - native lazy objects for partial objects
  - cursor-based pagination and `OffsetPaginator` (the old `Tools\Pagination\Paginator` is deprecated)
  - configurable foreign key names and default string length
  - `NULL` allowed in `SELECT`/`CASE`
  - `EntityRepository::expr()`
  - explicit identifier position
  - deprecations: `#[JoinColumns]`, `TypedExpression`, `null`/string sort directions, and `LockMode::NONE` (now a no-op)
- [P] 3.5: running on PHP 8.4+ without native lazy objects is **deprecated**, so call `$config->enableNativeLazyObjects(true)`. The upgrade notes advise going straight to PHP 8.4 and ORM 3.5+ to skip lazy-ghost proxy generation. Source: https://github.com/doctrine/orm/blob/3.7.x/UPGRADE.md
- [P] 3.4: PHP 8.4 native lazy objects behind a flag (PR #11853). `ClassMetadata::$reflFields` is deprecated in favour of `PropertyAccessors` "to better support property hooks". Property hooks can be mapped (PR #11852, "Final tests and adjustments to allow mapping properties with hooks"), and `private(set)` properties are handled (#11872).
- [P] Property-hook edge cases are still being fixed: #12401 was merged in June 2026, and #12646 ("Do not unset a hooked property when a typed field is set to null") is still open. Virtual (hook-only) properties cannot be mapped.
- [P] ORM 3.0 removed the YAML and annotation drivers. **The XML and attribute drivers remain.** DoctrineBundle 3.0 likewise dropped YAML/annotation support, and its `doctrine.orm.enable_lazy_ghost_objects` option is gone.

### 1.4 PHP 8.4/8.5 features relevant to DDD

- [P] PHP 8.5 was released on 20 Nov 2025 (https://www.php.net/releases/8.5/). Its features:
  - the pipe operator `|>`
  - `clone($obj, ['prop' => $v])`, which helps withers on `readonly` classes
  - `#[\NoDiscard]` (a warning when a return value is ignored; cast with `(void)` to discard on purpose)
  - closures and first-class callables in constant expressions and attributes
  - `array_first()` and `array_last()`
  - the URI extension
- [U] PHP 8.4 (Nov 2024) features:
  - property hooks (`get`/`set` on properties)
  - asymmetric visibility (`public private(set) string $status`)
  - native lazy objects (used by Doctrine)
  - `#[\Deprecated]`
  - `new Foo()->bar()` without wrapping parentheses
  - These are well known; see php.net/releases/8.4.
- [U] `readonly` classes arrived in PHP 8.2.
- DDD usage notes [U]:
  - `public private(set)` gives aggregates readable state without getters while keeping mutation in domain methods. Doctrine 3.4+ handles `private(set)`.
  - `readonly` classes suit value objects and events.
  - `clone with` (8.5) makes VO withers trivial.
  - `#[\NoDiscard]` suits methods like `releaseEvents()` or `Result`-returning domain operations whose result must not be dropped.
  - Avoid hooks on Doctrine-mapped properties until the open issues settle. Use hooks on non-mapped or derived properties.
  - The repo requires `php >= 8.4`. Using pipe, `clone with` or NoDiscard would require raising this to 8.5.

---

## 2. Symfony Messenger in depth (8.1, with 8.2 notes)

Source for everything in this section unless noted: https://symfony.com/doc/8.1/messenger.html and https://symfony.com/doc/8.2/messenger.html [P].
Install with `composer require symfony/messenger symfony/doctrine-messenger symfony/lock`. Lock enables deduplication.

### 2.1 Multiple buses

```yaml
# config/packages/messenger.yaml
framework:
    messenger:
        default_bus: command.bus           # injected for plain MessageBusInterface
        buses:
            command.bus:
                middleware:
                    - doctrine_ping_connection
                    - doctrine_close_connection
                    - doctrine_open_transaction_logger
                    - validation
                    - doctrine_transaction
            query.bus:
                middleware: [validation]
            event.bus:
                default_middleware:
                    enabled: true
                    allow_no_handlers: true   # an event with zero subscribers is legal
                    allow_no_senders: true
                middleware:
                    - doctrine_ping_connection
                    - doctrine_close_connection
                    - doctrine_transaction    # each async event handler = one tx
```

- [P] Each bus is a service, autowired by a variable name matching the bus (`MessageBusInterface $eventBus`, `$queryBus`). The docs advise adding a bus only when its middleware stack differs.
- [P] `allow_no_handlers` defaults to **false**. Set it to `true` on the event bus so that publishing an event that no module (yet) subscribes to does not throw `NoHandlerForMessageException`.
- [P] The docs' `allow_no_senders` comments contradict each other (YAML says the default is true, PHP says false). Set it explicitly.
- [P] There are two ways to restrict handlers to a bus:
  - the `bus:` option on `#[AsMessageHandler]` or the `messenger.message_handler` tag
  - `_instanceof` on a marker interface
- [P] **8.2** adds a bus `messages:` option (a class or interface list), for example `command.bus: { messages: App\Shared\CommandInterface }`. Dispatching any other message type on that bus then throws. The check covers app dispatches only, not messages received from transports. Mailer, Notifier and Webhook use the default bus, so allow-list their messages or point them to another bus.
- [P] `bin/console debug:messenger [bus]` lists the handlers per bus.

### 2.2 Middleware

- [P] Doctrine middleware:
  - `doctrine_ping_connection` reconnects when the connection was closed (long-running workers).
  - `doctrine_close_connection` closes the connection after handling.
  - `doctrine_open_transaction_logger` logs transactions left open.
  - `doctrine_transaction` wraps **all handlers** in one transaction, so handlers do not need `flush()` and any error rolls back. Use `doctrine_transaction: ['other_em']` for another entity manager.
- [P] Other middleware:
  - `validation` throws `ValidationFailedException`. Groups are set via `ValidationStamp`.
  - `router_context`.
  - **8.2**: a `logging` middleware.
- [P] **`dispatch_after_current_bus`** is enabled by default as part of the default middleware. If you list middleware manually, it **must come before `doctrine_transaction`** and be present on every bus involved.
  - A message dispatched inside a handler with `DispatchAfterCurrentBusStamp` is held until the outer handler finishes **without exception** (and, given the ordering, after commit).
  - If a deferred handler throws, the exceptions are wrapped in `DelayedMessageHandlingException::getWrappedExceptions()`.
  - The docs' example (RegisterUser and UserRegistered) shows the two failure modes this fixes:
    1. A failing email handler rolls back the user insert.
    2. A failing insert still sends the email.
  - Usage: `$eventBus->dispatch(new Envelope($event)->with(new DispatchAfterCurrentBusStamp()));`
- [P] **8.2** `unwrap_exceptions` option. Background: a sync handler failure is wrapped in `HandlerFailedException`, and this option unwraps it in sync handling. It is kept wrapped for multi-handler failures and for worker-received messages, because retry relies on the wrapper.
- [P] **Middleware order note.** The default middleware (add_bus_name, dispatch_after_current_bus, failed_message_processing, send_message, handle_message...) wraps your custom `middleware:` list. Your list is inserted between the defaults' "before" and "after" parts. [U] Treat the exact internal order as an implementation detail. Use `debug:container messenger.bus.command.bus.middleware` [U] or a test to verify it.

### 2.3 Handler and message attributes

- [P] `#[AsMessageHandler]` takes the following parameters:

  | Parameter | Meaning |
  |---|---|
  | `bus` | Restrict to one bus |
  | `fromTransport` | Only for messages received from that transport (lets one async event have per-transport handlers) |
  | `handles` | FQCN, when it can't be inferred from the `__invoke` type-hint |
  | `method` | Handling method |
  | `priority` | Higher runs first, sequentially |
  | `sign` | HMAC-sign messages with `kernel.secret`. Headers `Body-Sign`/`Sign-Algo`; an invalid or missing signature throws `InvalidMessageSignatureException` |

  The attribute can also go on several methods of one class.
  - **8.2** adds a `transport` option ("name of the transport to route the handled messages to... as with from_transport") and a transport-level `sign` option.
- [P] `#[AsMessage('async')]` or `#[AsMessage(['async','audit'])]` routes a message class to transports. YAML/PHP `routing:` overrides the attribute. Routing matches parent classes and interfaces, and `'*'` is the fallback.
  - **8.1**: `#[AsMessage(serializedTypeName: 'order.placed')]` sets a stable wire `type` header instead of the FQCN.
  - **8.2**: `serializedTypeNameAliases: ['App\Message\OrderPlaced']` reads old type names. Aliases apply to decoding only, so deploy consumers before producers.
- **Modular-monolith routing tip** [U]. Make every integration event implement a shared `IntegrationEvent` marker interface in `src/Shared/...` and route that interface once: `routing: { 'App\Shared\Integration\IntegrationEvent': events }`. Each subscribing module gets its own handler with `fromTransport` or `bus: event.bus`. Give events stable `serializedTypeName`s (e.g. `billing.invoice_paid.v1`) so classes can be renamed or moved between modules.

### 2.4 Transports

**Doctrine** (`doctrine://<connection>?options`) [P]

| Option | Default | Notes |
|---|---|---|
| `table_name` | `messenger_messages` | |
| `queue_name` | `default` | A column in the table, so one table can serve many transports |
| `auto_setup` | | Creates the table on send or get. Prefer a migration in production [U] |
| `redeliver_timeout` | 3600 s | Must exceed your longest handler |
| `use_notify` | `true` | PostgreSQL only |
| `check_delayed_interval` | 60000 ms | |
| `get_notify_timeout` | 60000 ms | |

- With PostgreSQL `use_notify`, do **not** share a `queue_name` between transports, because LISTEN/NOTIFY becomes much less effective.
- **8.1** `PostgreSqlNotifyOnIdleListener`: when one worker consumes several PG transports, they must share the DBAL connection and `table_name`, or a `LogicException` is thrown.
- Doctrine supports `--keepalive` (it refreshes `delivered_at`).
- [S] Doctrine transport reads use `SELECT ... FOR UPDATE SKIP LOCKED`, so multiple workers are safe (dev.to, Gabriel Anhaia).

**Redis**: streams. Requires ext-redis >= 4.3 and Redis 5+. Options include `stream`, `group`, `consumer`, `auto_setup`, `stream_max_entries`, `dbindex`, and Sentinel/Cluster settings (**8.1** `cluster=true` discovers nodes). Supports `--keepalive`. [P]

**AMQP**: exchanges and queues with bindings. **8.1** adds `delay[daily_delay_queues]` and `queues: false`. **8.2** adds `queues[name][bindings]`, `prefetch_count` and `--keepalive` support. `PriorityStamp` (**8.1**) works on AMQP and Beanstalkd only. [P]

For a single-deployable monolith on PostgreSQL, the Doctrine transport is enough and gives you the outbox for free (section 3). Add AMQP or Redis later only if throughput demands it. [U]

### 2.5 Failures, retries, deduplication, stamps

- [P] `failure_transport` is set globally and can be overridden per transport. Manage failures with `messenger:failed:show|retry|remove [--transport=]`. **8.2** adds `messenger:failed:retry --redispatch` (re-routes the message to its senders via `RedispatchStamp`).
- [P] `retry_strategy` options per transport: `max_retries` (3), `delay` (1000 ms), `multiplier` (2), `max_delay` (0 means unlimited), `jitter` (0.1), and `service` for a custom `RetryStrategyInterface`.
  - `UnrecoverableMessageHandlingException` means never retry.
  - `RecoverableMessageHandlingException` means always retry and accepts a `retryDelay`. **8.1** adds `forceRetry: false`, which respects `max_retries`.
  - **8.1** routes decode failures to the failure transport instead of dropping them.
- [P] **Deduplication** (`DeduplicateMiddleware`, symfony/symfony#54141, **added in 7.3**).
  - It is enabled automatically when symfony/lock is installed.
  - `new DeduplicateStamp(string $key, int $ttl = 300, bool $onlyDeduplicateInQueue = false)`.
  - By default the lock is held while the message is queued **and** while it is processing. It is released on success, kept across retries, and (**8.1**) released on definitive failure or when the message moves to the failure transport. With the third argument set to `true`, the lock is released when a worker receives the message.
  - The key is a business choice, e.g. `'recalc.project.'.$id`.
  - **8.2** `framework.messenger.deduplication.lock_factory: 'lock.deduplication.factory'` points it at a shared (e.g. Redis or PG) named lock. The default lock store may be local to each server.
  - **Not a substitute for an inbox** [S]: it prevents duplicate enqueue/concurrent processing, not a redelivered message running twice after a crash (bytespin/messenger-dedupe-bundle README; consistent with the docs' lock semantics).
- [P] Stamps:
  - `TransportNamesStamp(['async'])` overrides routing per envelope.
  - `DelayStamp` delays delivery.
  - `ValidationStamp` sets validation groups.
  - `SerializerStamp` sets serializer context.
  - `ScheduledStamp` marks Scheduler-originated messages.
  - `HandledStamp` carries handler results (query bus).

### 2.6 Scheduler [P] (https://symfony.com/doc/current/scheduler.html)

- Implement a `ScheduleProviderInterface` marked `#[AsSchedule('name')]` that returns `RecurringMessage::cron('@daily', $msg)` or `::every('1 hour', $msg)`. Alternatively, put `#[AsCronTask('0 0 * * *', jitter:, timezone:, transports: 'async')]` or `#[AsPeriodicTask(frequency: '1 day', from:, until:)]` directly on a handler or service.
- Each schedule becomes a transport named `scheduler_<name>`. Run it with `messenger:consume scheduler_default` (regex receiver names, e.g. `scheduler_.*`, are **8.1**). Messages carry `ScheduledStamp`.
- [U] With multiple worker instances, make schedules `->stateful($cache)->lock($lock)` so each tick fires once. Use the Scheduler for outbox-pruning and inbox-pruning jobs.

### 2.7 `messenger:consume` options [P]

| Option | Effect / notes |
|---|---|
| `--limit` | Stop after N messages |
| `--time-limit` | Stop after N seconds |
| `--memory-limit` | Stop above a memory threshold |
| `--failure-limit` | Stop after N failures |
| `--queues` | Receivers with `QueueReceiverInterface` only |
| `--all` / `--exclude-receivers` (`-eq`, only with `--all`) | Consume every receiver except the excluded ones |
| `--keepalive` | Doctrine, Redis, SQS, Beanstalkd; AMQP from 8.2 |
| `--no-reset[=N]` | Numeric value since 8.1 |
| `--fetch-size` | 8.1 |
| `--bus` | |
| `--sleep` | |
| `--concurrency` | 8.2. Handles messages in parallel. Messages with the same bus, transport and class stay sequential |

- Receivers listed on the command are consumed in the given order (priority). **8.2** adds a transport `priority` option, which applies with `--all`.
- **8.2**: a second SIGINT exits immediately.
- Run the worker under a process manager (supervisor or systemd) with `--time-limit`/`--memory-limit` so it restarts.

### 2.8 Serialization and versioning

- [P] The default serializer is PHP native `serialize()`. Switch to `messenger.transport.symfony_serializer` with `format: json`, either as `serializer.default_serializer` or per transport. Avoid `xml`, which loses stamp types.
  - **8.2** `symfony_serializer.service: serializer.messages` uses a named serializer.
  - **8.1/8.2** `serializedTypeName` and aliases provide stable wire names.
- [U] Recommendations for persisted events (the outbox and Doctrine queues outlive deploys):
  - Use the JSON Symfony serializer and stable `serializedTypeName`s with a version suffix.
  - Give events primitive or scalar-only constructor properties: no entities and no domain VOs that may change shape.
  - Use additive changes only. Introduce `...v2` as a new type and keep upcasting handlers for v1 until queues drain.
  - PHP `serialize()` ties stored messages to exact class definitions and private property names. [P] The 8.2 docs warn that the relay "must be able to load the classes of the stored messages and stamps".

---

## 3. Transactional outbox in Symfony

### 3.1 Symfony 8.2 native support [P] (https://symfony.com/doc/8.2/messenger.html, "Sending Messages Through a Transactional Outbox"; "The outbox transport option was introduced in Symfony 8.2")

```yaml
framework:
    messenger:
        transports:
            orders:
                dsn: '%env(MESSENGER_TRANSPORT_DSN)%'   # e.g. amqp:// or another doctrine queue
                outbox: outbox                           # store in "outbox" first
            outbox: 'doctrine://default?queue_name=outbox'
```

```bash
php bin/console messenger:consume outbox   # relay: forwards to "orders", never calls handlers
php bin/console messenger:consume orders   # normal handling
```

Rules from the docs:
- The outbox transport **must be a Doctrine transport on the same DB connection** as your writes. Symfony does not check this.
- Dispatch **while the transaction is open**. With the `doctrine_transaction` middleware, dispatch from the handler **without `DispatchAfterCurrentBusStamp`**. Otherwise the message is stored after commit.
- `DelayStamp` delays apply while the message waits in the outbox.
- A forwarding failure uses the **outbox** transport's retry and failure transport. A handling failure on the target retries directly to the target.
- Only transports that declare `outbox:` are protected.
- Order is not guaranteed.
- Lower `redeliver_timeout` on the outbox so a crashed relay resumes sooner.
- [P] 8.2 also adds **claim check**: a transport `claim_check` option stores oversized messages in a PSR-6 cache pool. Secondary coverage: thedroptimes.com, "Symfony 8.2 to Add Transactional Outbox and Claim Check to Messenger".

### 3.2 Symfony 8.1 (this repo today): the Doctrine transport *is* the outbox [P mechanics, U composition]

In a single-database monolith you do not need a relay. Route events to a Doctrine transport on the same `default` connection and dispatch them inside the command's transaction:

```yaml
framework:
    messenger:
        transports:
            events:
                dsn: 'doctrine://default?queue_name=events&auto_setup=false'
                retry_strategy: { max_retries: 5, delay: 1000, multiplier: 3 }
            failed: 'doctrine://default?queue_name=failed'
        failure_transport: failed
        routing:
            'App\Shared\Integration\IntegrationEvent': events
```

Flow:
1. The API Platform processor dispatches a command on `command.bus` (sync).
2. `doctrine_transaction` opens a transaction.
3. The handler mutates the aggregate. The aggregate records events.
4. The events are dispatched to `event.bus` **in the handler, before return, without `DispatchAfterCurrentBusStamp`**. The `SendMessageMiddleware` INSERTs them into `messenger_messages` on the same connection, so they are inside the transaction.
5. The middleware flushes and commits. Either the state and the events are both committed, or neither is.
6. `messenger:consume events` workers deliver to the subscriber handlers. Each handler runs in its own `doctrine_transaction` (event.bus stack).

Caveats:
- The atomicity rests on the Doctrine transport using the same DBAL `Connection` instance as the EntityManager. With DoctrineBundle, `doctrine://default` resolves to the `doctrine.dbal.default_connection` service. [U] Verify with an integration test: throw after dispatch and assert the `messenger_messages` row is absent.
- **Don't** mix in a sync transport for the same event, or the subscriber runs inside the producer's transaction. Module isolation requires async delivery or `fromTransport`-scoped handlers.
- [S] Delivery is at-least-once (crash between handler commit and ack → redelivery). Consumers must be idempotent (3.4).
- [S] Background: dev.to "Symfony Messenger Transports: Building a Transactional Outbox on Top" (Gabriel Anhaia) describes the same "Doctrine transport on the same connection" approach. A levelup.gitconnected article covers the RabbitMQ variant.

### 3.3 Bundles and alternatives

| Option | What it is | Maturity |
|---|---|---|
| Symfony 8.2 `outbox:` option | Native, documented | **Preferred** once on 8.2 (Nov 2026) [P] |
| `goetas/messenger-doctrine-outbox` | Middleware plus `outbox-doctrine://` transport | Packagist currently returns **no releases**, so treat it as abandoned or unpublished [P metadata] |
| `velpl/symfony-messenger-outbox-bundle` | `doctrine://...?queue_name=outbox` plus attribute routing | Small community bundle [S] |
| `alexandrebulete/ddd-outbox-bundle` | "The transport is the outbox", no separate table | Small community bundle [S] |
| Ecotone (`ecotone/symfony-bundle` 1.328.2, `ecotone/dbal`) | Full message framework. `DbalBackedMessageChannelBuilder` channels commit with the command's transaction. "Combined message channels" use the DB as outbox and a broker for execution. Also provides a dead-letter store and sagas | Active, frequent releases [P Packagist]. Outbox mechanics [P docs.ecotone.tech]. "High Throughput Outbox Forwarding" is **Enterprise-only** [P]. It replaces Messenger's programming model, so adopting it is a framework-level decision [U] |

### 3.4 Inbox / idempotent consumer [S pattern, U schema]

```sql
CREATE TABLE inbox_processed (
    consumer     text        NOT NULL,       -- e.g. 'billing.on_order_placed'
    message_id   uuid        NOT NULL,       -- event id generated by producer
    processed_at timestamptz NOT NULL DEFAULT now(),
    PRIMARY KEY (consumer, message_id)
);
```

- Put an `eventId` (UUIDv7) **in the event payload**, not only in a stamp, so it survives serialization changes.
- Inside the subscriber's `doctrine_transaction`, run `INSERT ... ON CONFLICT DO NOTHING`. If no row was inserted, return early. Otherwise do the work. Both commit together.
- A small Messenger middleware on `event.bus` can do this generically, keyed by the handler class (`HandlerDescriptor`) [U]. With multiple handlers per event, per-handler keys need care: one transaction wraps all handlers, unless you split transports per module with `fromTransport`.
- Prune old rows with a Scheduler task.
- Use `DeduplicateStamp` (2.5) as an *additional* guard against duplicate enqueue, not instead of the inbox.

---

## 4. Collecting domain events from aggregates

Pattern [S] (headsnet/domain-events-bundle, phauthentic/domain-events-symfony-bundle, fusonic/ddd-extensions):

```php
abstract class AggregateRoot
{
    /** @var list<DomainEvent> */
    private array $recordedEvents = [];

    protected function recordThat(DomainEvent $e): void { $this->recordedEvents[] = $e; }

    /** @return list<DomainEvent> */
    #[\NoDiscard]                       // PHP 8.5; drop on 8.4
    public function releaseEvents(): array { $e = $this->recordedEvents; $this->recordedEvents = []; return $e; }
}
```

(`$recordedEvents` is not mapped, so Doctrine ignores it.)

| Option | How | Verdict |
|---|---|---|
| **A. Explicit dispatch in the command handler** (recommended) | The handler calls `$repo->save($agg)` and then `foreach ($agg->releaseEvents() as $e) $eventBus->dispatch($e)`, inside `doctrine_transaction` and without `DispatchAfterCurrentBusStamp`. With the Doctrine `events` transport, this is the atomic outbox from 3.2 | Explicit, testable, atomic [U composition of P mechanics] |
| B. Doctrine `onFlush` listener | Collect events from UnitOfWork scheduled insertions/updates/deletions and dispatch them (Doctrine transport INSERT) during the flush, still inside the transaction | Atomic, but magic. Dispatching from inside `onFlush` must not touch the ORM UoW. The Doctrine transport uses DBAL directly, so it is fine [U] |
| C. Doctrine `postFlush` listener | `#[AsDoctrineListener(event: Events::postFlush)]` walks the identity map and dispatches | **Risky**: in implicit-transaction mode postFlush runs after commit, so a crash means lost events. It walks every managed entity and re-enters on nested flushes. Inside `doctrine_transaction` it actually runs before the outer commit, which means its behaviour depends on context [S, U] |
| D. `kernel.terminate` publishing (headsnet) | Publishes after the response is sent | Does not fire in console or worker processes. Failures are invisible [S] |

Pitfalls:
- **Dispatching before commit to a sync or broker transport.** The subscriber sees state that later rolls back, or the email is sent for a rolled-back user. Fix this with the Doctrine transport on the same connection (atomic) or with `DispatchAfterCurrentBusStamp` (after commit, but not crash-safe).
- **Dispatching after commit to a broker without an outbox.** A crash between commit and send loses the event. Fix this with the outbox (3.1/3.2).
- **Forgetting `releaseEvents()` clearing**, which leads to duplicate publishes.
- **Events recorded on entities loaded in another module.** Forbid this with architecture rules (section 5).
- **Using domain events as integration events.** Map them to versioned, scalar-only integration events at the module boundary before dispatch [U].

---

## 5. Architecture enforcement

### 5.1 Deptrac

- [P] Install with `composer require --dev deptrac/deptrac`. The latest is **4.7.2** (15 Sep 2026) and it requires PHP ^8.2. `qossmic/deptrac` (last release 2.0.4, Nov 2024) is **abandoned in favour of `deptrac/deptrac`** (Packagist metadata).
- [P] Docs: https://deptrac.github.io/deptrac/. Config is `deptrac.yaml`, or `deptrac.php` with `DeptracConfig` and the builders `paths()`, `excludeFiles()`, `layers()` (`Layer::withName()->collectors(ClassLikeConfig::create(...))`), and `rulesets()` (`Ruleset::forLayer()->accesses()`). A layer without `accesses` may depend on nothing else. The Graphviz formatter is optional.
- Example ruleset for "modules talk only via events" [U syntax; structure per docs]:

```yaml
# deptrac.yaml
deptrac:
  paths: [./src]
  layers:
    - name: Shared_Kernel            # IDs, IntegrationEvent marker, clock, etc.
      collectors: [{ type: directory, value: src/Shared/.* }]
    - name: Billing_Domain
      collectors: [{ type: directory, value: src/Billing/Domain/.* }]
    - name: Billing_Application
      collectors: [{ type: directory, value: src/Billing/Application/.* }]
    - name: Billing_Infrastructure
      collectors: [{ type: directory, value: src/Billing/Infrastructure/.* }]
    - name: Billing_Contracts        # published integration events ONLY
      collectors: [{ type: directory, value: src/Billing/Contracts/.* }]
    # ...repeat per module
    - name: Vendor_Symfony_Doctrine
      collectors: [{ type: classNameRegex, value: '#^(Symfony|Doctrine|ApiPlatform)\\#' }]
  ruleset:
    Billing_Domain:         [Shared_Kernel]                       # pure: no framework
    Billing_Application:    [Billing_Domain, Shared_Kernel, Billing_Contracts, Orders_Contracts]
    Billing_Infrastructure: [Billing_Application, Billing_Domain, Shared_Kernel, Billing_Contracts, Vendor_Symfony_Doctrine]
    Billing_Contracts:      [Shared_Kernel]
```

- The rule that matters: a module's layers may reference **another module's `Contracts` (events) namespace only**, never its Domain, Application or Infrastructure.
- Run `vendor/bin/deptrac analyse --fail-on-uncovered` in CI [U flag].
- Use `skip_violations` only for temporary, ticketed exceptions [U].

### 5.2 PHPat (PHPStan extension)

- [P] Install with `composer require --dev phpat/phpat`. It is at **0.12.5** (15 Sep 2026) and still 0.x, so breaking changes ship in minor versions. It requires PHPStan ^2.1. Enable it via `phpstan/extension-installer` or `includes: [vendor/phpat/phpat/extension.neon]`.
- [P] Assertions include:
  - `should()->dependOn()` and `shouldNot()->dependOn()`
  - `canOnlyDependOn()`
  - `beFinal()`, `beReadonly()`, `beInvokable()`
  - `haveOnlyOnePublicMethod()`
  - `extend()`, `implement()`
  - Docs: https://github.com/carlosas/phpat/tree/main/docs
- Example [U exact selector API]:

```php
final class ModuleBoundaryTest
{
    public function test_billing_domain_is_pure(): Rule
    {
        return PHPat::rule()
            ->classes(Selector::inNamespace('App\Billing\Domain'))
            ->canOnlyDependOn()
            ->classes(Selector::inNamespace('App\Billing\Domain'), Selector::inNamespace('App\Shared\Kernel'));
    }

    public function test_billing_uses_only_other_modules_contracts(): Rule
    {
        return PHPat::rule()
            ->classes(Selector::inNamespace('App\Billing'))
            ->shouldNotDependOn()
            ->classes(Selector::inNamespace('App\Orders'))
            ->excluding(Selector::inNamespace('App\Orders\Contracts'));
    }

    public function test_events_are_readonly_final(): Rule
    {
        return PHPat::rule()->classes(Selector::implements(IntegrationEvent::class))->should()->beReadonly();
    }
}
```

Register the test class as a service tagged `phpat.test` in `phpstan.neon` [U].

### 5.3 PHPStan extensions and other options

- [P] Versions:
  - `phpstan/phpstan` 2.3.1
  - `phpstan/phpstan-symfony` 2.1.0, which understands the container, `MessageBusInterface` results and console
  - `phpstan/phpstan-doctrine` 2.0.28, which provides DQL validation, repository and entity checks, and column-type vs property-type checks
  - `spaze/phpstan-disallowed-calls` v4.14.0
- [U] Use disallowed-calls for:
  - `disallowedNamespaces` (e.g. `Doctrine\ORM\*` in `App\*\Domain`)
  - `disallowedMethodCalls` (e.g. `MessageBusInterface::dispatch` outside Application)
  - `disallowedFunctionCalls` (e.g. `new \DateTimeImmutable()` in Domain, so a clock is used instead)
- [P] Pest **v5.3.1** (requires PHP ^8.4) has `arch()` tests (`expect('App\Billing\Domain')->not->toUse(['Doctrine', 'Symfony'])`). [U] Pest works in Symfony projects but is Laravel-centric, and two test runners add friction. In a Symfony repo, prefer Deptrac (module graph) plus PHPat (class-level rules inside PHPStan).
- [P] `carthage-software/mago` 1.56.2 is a Rust PHP linter and analyzer with architectural guard rules. It is an optional newer alternative [U capability claim].

---

## 6. Symfony practices for a DDD folder layout

- **Layout** [U]: `src/<Module>/{Domain,Application,Infrastructure,Contracts}` plus `src/Shared/{Kernel,Integration}`. API Platform resources go in `src/<Module>/Infrastructure/Api` or `.../Presentation/Api`.
- **Doctrine mapping per module** [U syntax, P concept: DoctrineBundle `mappings`]:

```yaml
doctrine:
  orm:
    enable_native_lazy_objects: true   # [U key name; required behaviour on PHP 8.4+ per ORM 3.5 UPGRADE]
    mappings:
      Billing:
        type: xml                      # or attribute
        is_bundle: false
        dir: '%kernel.project_dir%/src/Billing/Infrastructure/Persistence/Doctrine/Mapping'
        prefix: 'App\Billing\Domain'
        alias: Billing
      Orders: { type: xml, is_bundle: false, dir: '...', prefix: 'App\Orders\Domain' }
```

  [P] DoctrineBundle 3 supports only the attribute, XML and PHP drivers (YAML and annotations were removed).
- **XML vs attributes for a pure Domain**:
  - [P] Both drivers are first-class in ORM 3 and XML is not deprecated.
  - [U] Assessment: **XML** keeps `Doctrine\ORM\Mapping` imports out of `Domain`. It also allows embeddables and custom DBAL types for VOs (`<embedded>`, `type="billing_money"`) and lets Deptrac/PHPat enforce "Domain depends on nothing". The cost is verbosity and refactor drift; mitigate it with `doctrine:schema:validate` and `doctrine:mapping:info` in CI.
  - [U] **Attributes** are inert metadata at runtime (no Doctrine code runs in the domain). Many teams accept them as a pragmatic exception. If you go that way, whitelist `Doctrine\ORM\Mapping` in the domain rules explicitly.
  - Recommendation for "event-only, strictly enforced": **XML mapping in Infrastructure**, plus custom types for VOs. Never use attributes in Domain.
- **Service config** [P recipe]: the Symfony 7.4/8.1 `framework-bundle` Flex recipe's `services.yaml` is now just `App\: resource: '../src/'`, and **the old `exclude` list (Entity, DependencyInjection, Kernel.php) is gone**.
  - [P] 7.3 added the `container.excluded` tag and resource tags (`addResourceTag()`/`findTaggedResourceIds()`) for non-service classes (symfony.com blog, "New in Symfony 7.3: DI improvements").
  - [U] Unused definitions are removed anyway. Still add an explicit `exclude: ['../src/*/Domain/', '../src/*/Contracts/', '../src/Kernel.php']`, so domain objects never get autowired by accident and dev-container build time improves.
  - Domain services that need DI can be registered explicitly, or excluded paths can be narrowed to `Domain/Model`.
- **Attribute-based DI** [U names; long-standing Symfony attributes]:
  - `#[AsAlias(InvoiceRepository::class)]` on the Doctrine implementation binds the Domain port interface to its adapter.
  - `#[Autoconfigure(tags: [...])]`, `#[AutoconfigureTag]`, `#[Autowire(service: 'event.bus')]` or naming the argument `$eventBus`, and `#[Exclude]` on a class.
  - `#[AsMessageHandler(bus: 'event.bus', fromTransport: 'events')]` is the per-module subscriber.
  - Keep these attributes in Application/Infrastructure classes only.

---

## 7. Multi-tenancy (Symfony and Doctrine)

### 7.1 Doctrine SQL filters [P] (https://www.doctrine-project.org/projects/doctrine-orm/en/3.6/reference/filters.html)

- Extend `Doctrine\ORM\Query\Filter\SQLFilter::addFilterConstraint(ClassMetadata $target, string $alias): string`. The filter is applied wherever the ORM generates SQL: DQL, persisters, lazy loading and extra-lazy collections.
- Set parameters via `$em->getFilters()->enable('tenant')->setParameter('tenant_id', $id)` **before** use, never inside `addFilterConstraint`, because the query cache depends on it. `getParameter()` quotes the value.
- `disable()` loses parameters. Use `suspend()` and `restore()` instead.
- Gaps [U]:
  - It does not apply to raw DBAL or native SQL, nor to entities already in the identity map.
  - [U] Check how it interacts with DQL UPDATE/DELETE.
  - It is application-level only, so one bug equals a leak. Hence add RLS.
- Register the filter in config (`doctrine.orm.filters.tenant: { class: ..., enabled: false }`). Enable it from a kernel.request or security listener once the tenant is resolved [U].

### 7.2 Tenant context through Messenger [U design; Messenger stamp/middleware APIs are P]

Async handlers run in a worker with **no HTTP request**, so the tenant must travel on the envelope:

```php
final readonly class TenantStamp implements StampInterface { public function __construct(public string $tenantId) {} }

final class TenantMiddleware implements MiddlewareInterface
{
    public function __construct(private TenantContext $ctx, private EntityManagerInterface $em, private Connection $db) {}

    public function handle(Envelope $envelope, StackInterface $stack): Envelope
    {
        if ($envelope->last(ReceivedStamp::class) === null) {          // dispatching side
            if (!$envelope->last(TenantStamp::class) && $this->ctx->has()) {
                $envelope = $envelope->with(new TenantStamp($this->ctx->id()));
            }
            return $stack->next()->handle($envelope, $stack);
        }
        $stamp = $envelope->last(TenantStamp::class) ?? throw new UnrecoverableMessageHandlingException('No tenant');
        $previous = $this->ctx->swap($stamp->tenantId);                 // receiving side (worker)
        $this->em->getFilters()->enable('tenant')->setParameter('tenant_id', $stamp->tenantId);
        try {
            return $stack->next()->handle($envelope, $stack);
        } finally {
            $this->ctx->restore($previous);
            $this->em->getFilters()->disable('tenant');
        }
    }
}
```

- Put the middleware on every bus **before `doctrine_transaction`**, so RLS `set_config` runs inside the transaction it opens. Alternatively, do the `set_config` in a DBAL middleware or a `doctrine_transaction` decorator.
- The stamp must be serializable. With the JSON serializer, stamps go into the `X-Message-Stamp-*` headers [U header name].
- **Also include `tenantId` in every integration event payload** (belt and braces, and needed for auditing and replay). The stamp is the transport-level carrier.
- Reset services between messages. Workers reset services by default; `--no-reset` disables that, so don't use it with tenant state. Make `TenantContext` implement `ResetInterface` [U].
- Scheduler-originated messages have no tenant. Fan out one message per tenant explicitly.

### 7.3 PostgreSQL row-level security as defence in depth [S PG pattern, U Doctrine wiring]

```sql
ALTER TABLE invoice ENABLE ROW LEVEL SECURITY;
ALTER TABLE invoice FORCE ROW LEVEL SECURITY;          -- applies to table owner too
CREATE POLICY tenant_isolation ON invoice
  USING      (tenant_id = current_setting('app.tenant_id', true)::uuid)
  WITH CHECK (tenant_id = current_setting('app.tenant_id', true)::uuid);
CREATE INDEX ON invoice (tenant_id);
```

- [S] Use **transaction-scoped** settings: `SELECT set_config('app.tenant_id', :id, true)` (equivalent to `SET LOCAL`, and it accepts bound params). Session-level `SET` leaks across pooled connections (PgBouncer). Sources: leapcell.io RLS article; edana.ch (June 2026).
- [S] The app must connect as a non-superuser, non-owner role, or use `FORCE ROW LEVEL SECURITY`. Index `tenant_id`.
- [U] Doctrine wiring:
  - Use a DBAL 4 driver middleware (`Doctrine\DBAL\Driver\Middleware`) or a decorator of the `doctrine_transaction` middleware that runs `set_config(..., true)` right after `BEGIN`.
  - Reads outside explicit transactions are autocommit, so wrap HTTP reads in a transaction or use the API Platform provider decorator. Otherwise `current_setting` returns NULL and RLS returns zero rows. That fails closed, which is safe but surprising.
- **Messenger tables.** `messenger_messages` is shared across tenants. Do **not** put RLS on it (workers must see all rows). Tenant isolation for messages comes from the stamp and payload. A separate migrations/maintenance role may use `BYPASSRLS` [U].
- Test it: connect as the app role with no tenant set and assert zero rows. Set tenant A and assert none of tenant B's rows appear, including via lazy-loaded associations.

---

## Sources

Primary:
- Symfony roadmap and release data: https://symfony.com/roadmap, https://symfony.com/releases.json, https://symfony.com/releases/8.1
- Messenger docs: https://symfony.com/doc/8.1/messenger.html, https://symfony.com/doc/8.2/messenger.html (outbox, claim check, bus `messages`, dedup lock factory)
- Scheduler: https://symfony.com/doc/current/scheduler.html
- New in Symfony 7.3, Messenger and DI: https://symfony.com/blog/new-in-symfony-7-3-messenger-improvements, https://symfony.com/blog/new-in-symfony-7-3-dependency-injection-improvements
- Symfony Flex recipe: https://github.com/symfony/recipes/blob/main/symfony/framework-bundle/8.1/config/services.yaml
- API Platform:
  - https://api-platform.com/docs/core/upgrade-guide/
  - https://api-platform.com/docs/core/dto/
  - https://github.com/api-platform/core/blob/main/CHANGELOG.md
  - https://github.com/api-platform/core/releases
  - https://github.com/api-platform/core/issues/7540
- Doctrine:
  - https://github.com/doctrine/orm/releases
  - https://github.com/doctrine/orm/blob/3.7.x/UPGRADE.md
  - https://github.com/doctrine/orm/issues/11624 (PRs #11852, #11853, #12401, #12646)
  - https://github.com/doctrine/DoctrineBundle/blob/3.3.x/UPGRADE-3.0.md
  - https://www.doctrine-project.org/projects/doctrine-orm/en/3.6/reference/filters.html
- PHP 8.5: https://www.php.net/releases/8.5/en.php
- Deptrac: https://github.com/deptrac/deptrac/releases, https://packagist.org/packages/deptrac/deptrac, https://deptrac.github.io/deptrac/
- PHPat: https://github.com/carlosas/phpat (docs/examples.md, docs/documentation/assertions.md)
- Ecotone: https://docs.ecotone.tech/modelling/recovering-tracing-and-monitoring/resiliency/outbox-pattern
- Packagist metadata (versions and dates, 12 Oct 2026): phpstan/phpstan, phpstan-symfony, phpstan-doctrine, spaze/phpstan-disallowed-calls, pestphp/pest, ecotone/*, goetas/messenger-doctrine-outbox, carthage-software/mago

Secondary:
- https://dev.to/gabrielanhaia/symfony-messenger-transports-building-a-transactional-outbox-on-top-3op3
- https://www.thedroptimes.com/node/72431 (Symfony 8.2 outbox and claim check news)
- https://endoflife.date/symfony
- Packagist READMEs: velpl/symfony-messenger-outbox-bundle, alexandrebulete/ddd-outbox-bundle, bytespin/messenger-dedupe-bundle, headsnet/domain-events-bundle, phauthentic/domain-events-symfony-bundle, fusonic/ddd-extensions
- PostgreSQL RLS: https://leapcell.io/blog/achieving-robust-multi-tenant-data-isolation-with-postgresql-row-level-security, https://edana.ch/en/2026/06/25/ensuring-multi-tenant-data-isolation-with-postgresql-row-level-security-rls/
- SymfonyCasts, API Platform "stateOptions + entityClass": https://symfonycasts.com/screencast/api-platform-extending/state-options
