# v4.1 research: API Platform 5 / Symfony 8.1 / Doctrine DBAL 4 for database-per-tenant

Date: 2026-10-14 · Scope: sections 7–12 of `iam_v3_1_snapshot.md` and the architecture guide (`tab_architecture_guide.md`).
Installed versions checked in this repo's `vendor/`: api-platform/* 5.0.2, doctrine/dbal 4.5.0, doctrine/orm 3.7.4, doctrine/doctrine-bundle 3.3.2, doctrine/persistence 4.2.0, symfony/* 8.1.8 (runtime 8.1.0). Facts marked **(verified in vendor)** were read from that source; everything else cites the URL given.

Legend: REC = recommendation, PITFALL = something that will bite, CHANGE = a change to the v3.1 text.

---

## 0. Changes to the current design (summary)

| # | v3.1 text | Finding | Change |
|---|---|---|---|
| C1 | 8.7 "alternatively enable emulated prepares on the tenant connection" | `PDO::ATTR_EMULATE_PREPARES` is only for OCI, Firebird and MySQL (php.net). DBAL's `pdo_pgsql` driver already sets `Pdo\Pgsql::ATTR_DISABLE_PREPARES = true` unless you set it to `false` **(verified in vendor: `Driver/PDO/PgSQL/Driver.php`)**, so Doctrine sends unnamed statements and is transaction-pooling safe without `max_prepared_statements`. | Delete the "emulated prepares" sentence. Keep `max_prepared_statements` (default 200 since PgBouncer 1.21) for other clients. |
| C2 | 8.2 one login role per society; new requirement: ONE central credential | With one credential no PostgreSQL mechanism can verify the application's belief about which tenant it is talking to. The DB-side defence shrinks to: explicit PgBouncer `[databases]` entries, `REVOKE CONNECT … FROM PUBLIC`, owner separation (NOLOGIN owner role per DB, app role is not owner), and a post-connect `current_database()` assertion in `TenantBoundConnection`. Role switching via `ALTER ROLE app IN DATABASE t_x SET role = t_x` is pooling-safe and needs no `SET ROLE` in code, but is a privilege-containment measure, not a cross-tenant guard (section 3.6). | Rewrite 8.2/8.7/8.8: single `app` role, `TenantConnectionResolver` returns only `dbname`; add `current_database()` check; add `application_name = tenant:<id>`; `max_user_connections` on PgBouncer. |
| C3 | 8.6 "Doctrine result and second-level cache regions use a tenant-specific namespace" | DBAL's own result cache key = sha256 of `sql + params + types + connection params` where connection params are the *configured* (placeholder) params, not the ones the middleware swapped in **(verified in vendor: `Connection.php` ~L829, `QueryCacheProfile::generateCacheKeys`)**. Same key for every tenant. | Disable `result_cache` on the `tenant` connection or wrap the pool with a tenant-prefixing decorator. Same rule for ORM second-level cache regions (keys are entity id + region). Metadata and DQL query caches are safe (schema identical across tenants). |
| C4 | 7.4 `TenantContext` implements `ResetInterface` so "the kernel clears it between requests" | Reset runs in `Kernel::boot()` at the start of the **next** main request, not at terminate **(verified in vendor: `Kernel.php` L69–73, L132–139)**; in Messenger it runs on `WorkerRunningEvent` (priority −1024) after each message. `Registry::reset()` clears or resets entity managers but does **not** close DBAL connections **(verified in vendor: `Registry.php`)**. | Add an explicit `kernel.terminate` (or `kernel.finish_request`) listener that closes the tenant connection, and keep `TenantScopedRunner`'s `finally` as the only place that closes it for CLI/workers. Set `FRANKENPHP_RESET_KERNEL=1` only as an emergency fallback (clones the kernel per request, slower). |
| C5 | 7.1 "`#[TenantRequirement]` … compiled into the `_tenant` route default" | API Platform operations have no controller, so a controller attribute never sees them. `ApiLoader` merges `Operation::getDefaults()` into the route defaults (built-ins `_controller`, `_stateless`, `_api_resource_class`, `_api_operation_name` win) **(verified in vendor: `Routing/ApiLoader.php` L116–121)**. | Two loaders, one default: (a) controllers: subclass `AttributeRouteControllerLoader::configureRoute()`; (b) API Platform: a `ResourceMetadataCollectionFactoryInterface` decorator that turns `extraProperties['tenant']` (or a PHP attribute on the resource class) into `defaults: ['_tenant' => …]`. Architecture test iterates `RouterInterface::getRouteCollection()`. |
| C6 | 12 step 2 "fails with 400/404/423" | API Platform's `ErrorListener` handles every non-HTML exception, including ones thrown by `kernel.request` listeners before any operation ran, and renders `application/problem+json` **(verified in vendor: `EventListener/ErrorListener.php` L70–99)**. | Make `TenantNotFound`, `TenantSuspended`, `ConflictingTenantHints` implement `ApiPlatform\Metadata\Exception\ProblemExceptionInterface` (status, type, title, detail) and list them in `exception_to_status` for non-API routes. |

---

## 1. API Platform 5

### 1.1 Request pipeline in 5.0 (where tenant logic belongs)

- `use_symfony_listeners` defaults to **false** **(verified in vendor: `Bundle/DependencyInjection/Configuration.php` L89)**. Requests go through `ApiPlatform\Symfony\Controller\MainController`, which calls the provider chain then the processor chain. Kernel listeners (`ReadListener` 4, `DeserializeListener` 2, etc.) exist only when the flag is true. Docs: https://api-platform.com/docs/core/events/ ("Using Kernel event with API Platform should be mostly limited to tweaking the generated HTTP response").
- Provider chain (service `api_platform.state_provider.main`, decoration priority; higher = closer to the inner provider) **(verified in vendor: `Bundle/Resources/config/state/provider.php`, `security.php`)**:
  - `read` (500) ← decorated by `access_checker.pre_read` (10), `access_checker` (0), `security_parameter` (0), `backed_enum` (0), `object_mapper` (0)
  - `deserialize` (300) ← `access_checker.post_deserialize` (0)
  - `parameter` (180)
  - `content_negotiation` (100, outermost)
  - `validate` ← `access_checker.post_validate` (0)
- Processor chain (`api_platform.state_processor.main`): `serialize` (200), `object_mapper_output` (150), `write` (100), `object_mapper_input` (50); `respond` is separate and decorated by `add_link_header`.
- REC: resolve the tenant **before** API Platform, in a `kernel.request` listener at priority 20 (section 2). Providers, processors and Doctrine extensions only *consume* `TenantContext`. Do not decorate `api_platform.state_provider.main` to resolve tenants: it would not cover non-API routes, it runs after the firewall, and it runs after content negotiation.
- Decorate at the provider level only for **tenant-shaped checks that need the operation**, e.g. a `TenantRequirementProvider` decorating `api_platform.state_provider.read` with priority > 10 (runs before `access_checker.pre_read`) to assert `TenantContext` matches the operation's `extraProperties['tenant']`. This is belt-and-braces; the listener is the primary gate.

### 1.2 `#[ApiResource]` on DTOs vs entities

- REC (matches the architecture guide §13): resources are DTO classes in `Infrastructure`, never domain entities. Three documented strategies: https://api-platform.com/docs/core/dto/
  1. DTO linked to an entity with `stateOptions: new Options(entityClass: BookEntity::class)` + `#[Map(source: BookEntity::class)]` (Object Mapper, Symfony-only, added in 4.2, requires `symfony/object-mapper` ^7.4|^8.0). Good for simple reads of the module's own tables.
  2. Input DTO + built-in persist processor (`#[Map(target: …)]`).
  3. Input DTO + custom processor that dispatches a command (the guide's rule for aggregates, so domain events are not skipped).
- 5.0 change: `ObjectMapperProcessor` removed, replaced by `ObjectMapperInputProcessor` / `ObjectMapperOutputProcessor` (https://api-platform.com/docs/core/upgrade-guide/).
- `ApiPlatform\Doctrine\Orm\State\Options(entityClass, handleLinks, repositoryMethod)` **(verified in vendor)**. `repositoryMethod` must return a `QueryBuilder`; pagination/filters still apply.

### 1.3 `provider` / `processor` composition

- `ProviderInterface::provide(Operation $operation, array $uriVariables = [], array $context = [])`; `ProcessorInterface::process(mixed $data, Operation $operation, array $uriVariables = [], array $context = [])`. Attach with `#[Get(provider: X::class)]` / `#[ApiResource(processor: Y::class)]`. https://api-platform.com/docs/core/state-providers/ , https://api-platform.com/docs/core/state-processors/
- Compose, don't decorate, the Doctrine ones: inject `#[Autowire(service: 'api_platform.doctrine.orm.state.item_provider')]`, `…collection_provider`, `…persist_processor`, `…remove_processor` **(service ids verified in vendor)**. Return the modified object; `write: true` is needed for a processor to run on a safe operation.
- Since 5.0 a `Link` provider's value replaces the URI variable unless the provider implements `PreservesUriVariableInterface`; `ReadLinkParameterProvider` preserves by default. `SerializerContextBuilder::createFromRequest()` no longer sets `uri_variables` (upgrade guide).

### 1.4 Security expressions, voters

Docs: https://api-platform.com/docs/symfony/security/

- Variables available **(verified in vendor: `Security/ResourceAccessChecker.php` L86–91)**: `token`, `user`, `roles`, `trust_resolver`, `auth_checker`, `access_decision`, plus `object`, `previous_object` (post-denormalize) and `request` (resource level only). `access_decision` (4.4+) exposes voter reasons; in prod the 403 `detail` is generic unless `securityMessage` is set.
- `security` runs before denormalization (`object` = stored value); `securityPostDenormalize` after (`object` = submitted, `previous_object` = shallow clone, `null` on create); `securityPostValidation` exists on `Operation` **(verified in vendor: `Metadata/Metadata.php`)**. Since 4.3, a `security` expression that does not reference `object` is evaluated **before** the provider (`access_checker.pre_read`), so `is_granted('ROLE_USER')`-style denials never hit the DB.
- Voters cannot be used on `GetCollection`; filter collections with a Doctrine extension or in the custom provider.
- Failed `securityPostDenormalize` silently discards protected fields unless `extraProperties: ['throw_on_access_denied' => true]`.
- Throw `ApiPlatform\Metadata\Exception\AccessDeniedException('Access Denied.', detail: '…')` from providers/processors/voters (4.4+); the Symfony-bridge `AccessDeniedException` is deprecated.
- REC for this design: one `PermissionVoter` with attribute = permission slug, `is_granted('orders:read', object)` on item operations; `TenantIsActive` / `ProfileIsActive` as assertions inside the voter, not as expressions. Add `Vote::addReason()` only with non-sensitive text.

### 1.5 Doctrine extensions

- `QueryCollectionExtensionInterface::applyToCollection(QueryBuilder, QueryNameGeneratorInterface, string $resourceClass, ?Operation, array $context)` and `QueryItemExtensionInterface::applyToItem(…, array $identifiers, …)`. Tags `api_platform.doctrine.orm.query_extension.collection` / `.item` (autoconfigured). Use a **positive** priority; built-ins are negative (eager_loading item −8, filter −16, filter_eager_loading −17, eager_loading collection −18, order −32, pagination −64). https://api-platform.com/docs/core/extensions/
- PITFALL: an extension that returns early for anonymous users exposes everything; require `IS_AUTHENTICATED_FULLY` via `access_control`.
- With database-per-tenant you do **not** need a tenant extension; you need a `ScopeExtension` (business_unit / venture scope from `AccessEvaluator`) applied to both collection and item.
- `ItemProvider` runs item extensions after `handleLinks()`; an extension implementing `QueryResultItemExtensionInterface` can short-circuit the query **(verified in vendor: `doctrine-orm/State/ItemProvider.php`)**.

### 1.6 uriVariables and Links

- `uriVariables: ['companyId' => new Link(fromClass: Company::class, toProperty: 'company'), 'id' => new Link(fromClass: Employee::class)]`; `Link(security: "is_granted('X', company)")` with `securityObjectName` for the variable name. Link security is always on in 5.0 (`enable_link_security` removed). https://api-platform.com/docs/core/subresources/
- Never put the tenant in the URI (`/tenants/{slug}/…`): it duplicates the header/host source and the design forbids body/query/URI hints.

### 1.7 OpenAPI: documenting `X-Tenant`, per-tenant servers, auth header

- Per-operation header parameters: `new GetCollection(parameters: ['X-Tenant' => new HeaderParameter(description: 'Society slug', required: true, constraints: [new Assert\Regex('/^[a-z0-9-]+$/')], nativeType: new BuiltinType(TypeIdentifier::STRING))])`. Header keys are case-insensitive; `required: true` adds `NotNull`; constraints need `nativeType` or each constraint receives an array. Documented automatically in OpenAPI/Hydra. https://api-platform.com/docs/core/filters/
- Global: decorate `api_platform.openapi.factory` with `#[AsDecorator(decorates: 'api_platform.openapi.factory')]`, iterate `$openApi->getPaths()->getPaths()` and add `new Model\Parameter('X-Tenant', 'header', 'Society slug', required: false)` to every operation whose route default `_tenant` is `required`/`optional`; set `->withServers([new Model\Server('https://{tenant}.colsynq.app', variables: …)])`. https://api-platform.com/docs/core/openapi/
- Auth header in Swagger UI: `api_platform.swagger.api_keys.JWT: { name: Authorization, type: header }` **(verified in vendor: Configuration L311–338)**. https://api-platform.com/docs/symfony/jwt/
- Hide an operation with `openapi: false`; describe one with `openapi: new Model\Operation(summary:, responses: [423 => new Model\Response('Tenant suspended')])`.

### 1.8 Errors (RFC 9457 problem+json)

- API Platform docs still say RFC 7807; RFC 9457 (July 2023) obsoletes it with the same members `type`, `status`, `title`, `detail`, `instance`, extension members, `about:blank`. https://www.rfc-editor.org/rfc/rfc9457.html , https://api-platform.com/docs/core/errors/
- Status resolution order (Symfony): `exception_to_status` → Symfony `HttpExceptionInterface` → `ApiPlatform\Metadata\Exception\ProblemExceptionInterface` → `ApiPlatform\Metadata\Exception\HttpExceptionInterface` → defaults (`RequestExceptionInterface` 400, `ValidationException` 422) → `#[ErrorResource]` status → 500.
- `ProblemExceptionInterface` = `getType(): string`, `getTitle(): ?string`, `getStatus(): ?int`, `getDetail(): ?string` (+ `getInstance`) **(verified in vendor)**. `#[ErrorResource]` extends `ApiResource` **(verified in vendor)**; `ApiPlatform\State\ApiResource\Error::createFromException()` for a custom `api_platform.state.error_provider`.
- PITFALL: `exception_to_status` is **not merged** with the defaults; copy the default entries (Serializer `ExceptionInterface` 400, `InvalidArgumentException` 400, `FilterValidationException` 400, `OptimisticLockException` 409, `UniqueConstraintViolationException` 422 in 5.0). `rfc_7807_compliant_errors` was removed in 3.4 (always on); `error_formats.jsonproblem: ['application/problem+json']`.
- Message visibility: 5xx `detail` is generic in prod; 4xx messages are shown. The listener opts out per operation with `extraProperties: ['_api_error_handler' => false]` **(verified in vendor)**.
- REC: domain marker interfaces (guide §15) map through one `#[AsDecorator]` of `api_platform.state.error_provider` or, simpler, `exception_to_status` entries for the markers (`App\Shared\Domain\NotFound: 404`, `Conflict: 409`, `Unprocessable: 422`, `Forbidden: 403`). Tenant errors implement `ProblemExceptionInterface` with `type: 'https://colsynq.app/problems/tenant-suspended'` and status 423.

### 1.9 Operation-level tenant requirement (`#[TenantRequirement]`)

- `Operation` has `extraProperties: array` with `getExtraProperties()` / `withExtraProperties()`; `HttpOperation` has `defaults`, `requirements`, `options`, `stateless`, `host`, `schemes`, `condition`, `routePrefix`, `routeName` **(verified in vendor: `Metadata/Metadata.php`, `Metadata/HttpOperation.php`)**.
- `ApiLoader` builds each route with `['_controller' => …, '_stateless' => $operation->getStateless(), '_api_resource_class' => …, '_api_operation_name' => …] + ($operation->getDefaults() ?? []) + ['_format' => null]` **(verified in vendor)**. So `defaults: ['_tenant' => 'required']` on an operation lands in `$request->attributes->get('_tenant')` after the router (32), readable at priority 20.
- REC: keep authors writing one thing. Decorate `api_platform.metadata.resource.metadata_collection_factory` (API Platform itself decorates it at priority 100 for the object mapper **(verified in vendor)**): for each operation, read `extraProperties['tenant']` (or a `#[TenantRequirement(Requirement::Required)]` PHP attribute on the resource class via reflection), and return `$operation->withDefaults(($operation->getDefaults() ?? []) + ['_tenant' => $value])`. Fail at cache warm-up when missing (`\LogicException`), so the architecture test is enforced at build time. Set a global default in config: `api_platform.defaults.extra_properties.tenant: required` (defaults apply to every resource), and override to `none`/`platform` on the few others.
- For plain controllers: subclass `Symfony\Bundle\FrameworkBundle\Routing\AttributeRouteControllerLoader` and override `configureRoute(Route $route, \ReflectionClass $class, \ReflectionMethod $method, object $attr): void` to read `#[TenantRequirement]` from method then class and `$route->setDefault('_tenant', …)`; swap the class of service `routing.loader.attribute` in a compiler pass **(service id verified in vendor: framework-bundle `Resources/config/routing.php` L88)**. Hook docs: https://symfony.com/doc/current/routing/custom_route_loader.html
- Architecture test: `$router->getRouteCollection()->all()` → every route not matching `^(_|api_doc|api_entrypoint|api_jsonld_context)` has `_tenant`.

### 1.10 Stateless, GraphQL, pagination, serialization

- Stateless: `api_platform.defaults.stateless: true` → `_stateless` route default; `AbstractSessionListener` throws `UnexpectedSessionUsageException` in debug (warning in prod) if the session is touched **(verified in vendor: `AbstractSessionListener.php` L216, L249)**. A stateless firewall sets `_stateless` only when the attribute is not already set (Symfony 6.3+). Keep `stateless: false` on the BFF web firewall routes that need the session cookie.
- GraphQL: `api_platform.graphql.enabled` defaults false and is only turned on by installing `api-platform/graphql`; do not install it. https://api-platform.com/docs/core/configuration/
- Pagination: `defaults.pagination_enabled`, `pagination_items_per_page` (30), `pagination_maximum_items_per_page`, `pagination_partial` (skips COUNT), `pagination_client_*`; per operation `paginationEnabled`, `paginationMaximumItemsPerPage`, `paginationFetchJoinCollection`, `paginationUseOutputWalkers`. Custom providers must return `PaginatorInterface`/`PartialPaginatorInterface` (`ArrayPaginator`, `TraversablePaginator`, or `ApiPlatform\Doctrine\Orm\Paginator` around `Doctrine\ORM\Tools\Pagination\Paginator`). Cursor pagination (`paginationViaCursor`) documents `RangeFilter` + `OrderFilter`, but `RangeFilter` is deprecated since 4.4 (removed 6.0) in favour of `ComparisonFilter` via `#[QueryParameter]`; run `bin/console api:upgrade-filter`. https://api-platform.com/docs/core/pagination/
- Serialization: `normalizationContext: ['groups' => ['order:read']]` per resource/operation (operation context replaces resource context); decorate `api_platform.serializer.context_builder` (`SerializerContextBuilderInterface::createFromRequest(Request, bool $normalization, ?array $extractedAttributes)`) for role/permission-based groups; per-item ACL groups via a normalizer at priority 64. https://api-platform.com/docs/core/serialization/

### 1.11 Multiple entity managers

- `ItemProvider`/`CollectionProvider`/`PersistProcessor` call `$this->managerRegistry->getManagerForClass($entityClass)` where `$entityClass` = `stateOptions->entityClass` if set, else the resource class; a `null` result throws `RuntimeException` (added in 4.0.20, api-platform/core PR #6995) **(verified in vendor)**.
- `AbstractManagerRegistry::getManagerForClass()` iterates managers in configuration order and returns the first whose `getMetadataFactory()->isTransient($class)` is false **(verified in vendor: persistence `AbstractManagerRegistry.php`)**. PITFALL: overlapping `mappings` (e.g. `auto_mapping: true` on both managers, or a `dir` that contains both landlord and tenant entities) silently routes tenant entities to the landlord EM. Set `doctrine.orm.auto_mapping: false` and one explicit `mappings` entry per module on exactly one manager; add a test asserting `getManagerForClass()` returns `tenant` for every business entity and `landlord` for IAM/Tenancy ones.
- Doctrine SQL filters are per EM: `doctrine.orm.entity_managers.tenant.filters.<name>: { class:, enabled: }` and `$em->getFilters()->enable()` on **that** manager; parameters must be set before use; filters do not affect already-managed entities. https://www.doctrine-project.org/projects/doctrine-orm/en/latest/reference/filters.html . Not needed for database-per-tenant; keep for soft-delete only.
- No associations across managers (Symfony docs: entities cannot define associations across different entity managers). Store other modules' ids as plain UUID columns (guide §12).
- Multiple-EM caches: metadata/query caches are per EM config and safe; result caches are not database-aware (C3).

---

## 2. Symfony 8.1 kernel events, context, reset, Messenger, Scheduler, console

### 2.1 Listener vs subscriber, priority

- Symfony: "usually a matter of personal taste"; subscribers are easier to reuse, listeners (attribute) more flexible. https://symfony.com/doc/current/event_dispatcher.html
- REC: `#[AsEventListener(event: KernelEvents::REQUEST, priority: 20)] final class TenantResolveListener { public function __invoke(RequestEvent $event): void }`. Only act on `$event->isMainRequest()`.
- Priorities **(verified in vendor, symfony 8.1.8)**: `ValidateRequestListener` 256, `SessionListener` 128, `LocaleListener::setDefaultLocale` 100, `RouterListener` 32, API Platform `AddFormatListener` 28 (only with `use_symfony_listeners`), `LocaleListener::onKernelRequest` 16, `Firewall`/`FirewallListener` 8, API Platform `ReadListener` 4 / `DeserializeListener` 2 (listener mode only). Historic list with the API Platform ordering discussion: https://github.com/api-platform/core/issues/3577
- 20 vs 24 vs firewall `request_matcher`:
  - **20 (REC)**: after router (needs `_route`/`_tenant`), before locale (16) so an optional `TenantLocaleListener` at 18 can set `_locale` from tenant branding, before the firewall (8) so authenticators can read `TenantContext`.
  - 24 is equally valid (nothing between 32 and 16 in a non-listener API Platform setup); the only reason to pick 24 is if a listener at 20–23 is added later. Don't go above 28 if `use_symfony_listeners` is ever enabled (format negotiation at 28 is wanted before errors are rendered).
  - Firewall `request_matcher` (`security.firewalls.X.request_matcher: App\Security\TenantHostMatcher`) runs inside the firewall at 8 and only *selects* a firewall; it cannot produce 400/404/423 cleanly and cannot populate context for the router. Use it only to route `/platform/*` or the platform host to a separate firewall. https://symfony.com/doc/current/security/firewall_restriction.html
- Route data after the router: `$request->attributes->get('_route')`, `'_route_params'`, and every default becomes an attribute (`$request->attributes->get('_tenant')`). https://symfony.com/doc/current/routing.html

### 2.2 `Request::attributes` vs a context service

- REC: `TenantContext` service is the source of truth (works in HTTP, Messenger, CLI); the listener additionally sets `$request->attributes->set('_tenant_id', $tenant->id)` for logging processors, firewall matchers and functional-test assertions. Authenticators get `TenantContext` injected. Never read the tenant from `RequestStack` in business code.
- Set-once semantics: `TenantContext::set()` throws `TenantAlreadySet`; `TenantScopedRunner` is the only API that may replace it (push/pop).

### 2.3 Reset in long-running processes

- `ResetInterface` services are auto-tagged `kernel.reset` (`method: reset`); `services_resetter` calls them. Symfony 8.1 also resets non-shared services tagged `kernel.reset`. https://symfony.com/doc/current/reference/dic_tags.html
- Timing **(verified in vendor)**: `Kernel::handle()` sets `resetServices = true`; `Kernel::boot()` runs `services_resetter->reset()` at the start of the next main request when `requestStackSize === 0`. FrankenPHP: `FrankenPhpWorkerRunner` loops `frankenphp_handle_request()` → `handle()` → `terminate()`; with `FRANKENPHP_RESET_KERNEL=1` it `clone`s the kernel after each request instead (slower, Symfony 8.1). https://symfony.com/doc/current/components/runtime.html
- `Doctrine\Bundle\DoctrineBundle\Registry implements ResetInterface`: for each initialised manager, `clear()` if open, else `resetManager()` (native lazy objects since DI 7.3; uninitialised lazy EMs are skipped) **(verified in vendor)**. It never closes connections. `ManagerRegistry::resetManager()` calls `container->resetService()` and returns a fresh EM; with native lazy objects existing injected references are reset in place, so injecting `EntityManagerInterface $tenantEm` is safe; injecting `ManagerRegistry` is still the most robust.
- REC: `TenantContext::reset()` only clears the context. A separate `TenantConnectionCloser` (`#[AsEventListener(KernelEvents::TERMINATE)]` + Messenger `WorkerMessageHandledEvent`/`WorkerMessageFailedEvent`) rolls back any active transaction and `close()`s the tenant connection, so a worker never idles on a tenant's pool and the next request reconnects through the middleware. DoctrineBundle's `IdleConnectionMiddleware` (`doctrine.dbal.connections.tenant.idle_connection_ttl`, default 600) is a safety net, not the mechanism.
- Messenger: `ResetServicesListener` subscribes `WorkerRunningEvent` (−1024) and `WorkerStoppedEvent` (−1024); reset runs after each non-idle loop (`--no-reset` disables; `--no-reset=100` resets every 100 messages, 8.1). https://symfony.com/doc/current/messenger.html

### 2.4 Messenger middleware + stamps

- `final readonly class TenantStamp implements StampInterface { public function __construct(public string $tenantId) {} }` (must be serializable by the Symfony serializer; do not make it `NonSendableStampInterface`).
- `TenantStampMiddleware implements MiddlewareInterface`: if `null === $envelope->last(ReceivedStamp::class)` (dispatch side) and no `TenantStamp`, add one from `TenantContext` or throw `TenantNotResolved` for messages implementing `TenantScopedMessage`. On the worker side (`ReceivedStamp` present) read the stamp and wrap `$stack->next()->handle()` in `TenantScopedRunner::run()`; a tenant message without a stamp → `UnrecoverableMessageHandlingException` (goes to the failure transport without retries).
- Bus order **(docs)**: custom tenant middleware first; `dispatch_after_current_bus` before `doctrine_transaction`; `doctrine_ping_connection: ['tenant']` and `doctrine_transaction: ['tenant']` take the EM name as argument. PITFALL: `doctrine_ping_connection` opens the connection, so it must come **after** the tenant middleware. `doctrine_close_connection: ['tenant']` after handling.
- Events for logging/cleanup: `WorkerMessageReceivedEvent`, `WorkerMessageHandledEvent`, `WorkerMessageFailedEvent`, `SendMessageToTransportsEvent`.

### 2.5 Scheduler

- `#[AsSchedule('tenants')] class TenantJobsProvider implements ScheduleProviderInterface`; `RecurringMessage::trigger(CronExpressionTrigger::fromSpec('@daily'), new CallbackMessageProvider(fn (MessageContext $c) => yield from $this->fanOut(), 'nightly'))` yielding `new RedispatchMessage(new RunTenantJob($tenantId), 'tenant_jobs')` per active tenant; `->stateful($cache)->processOnlyLastMissedRun(true)->lock($lockFactory->createLock('scheduler-tenants'))`. Run `messenger:consume scheduler_tenants`. Keep the scheduler worker light; the per-tenant handler runs under the stamp middleware. https://symfony.com/doc/current/scheduler.html

### 2.6 Console

- `#[AsEventListener(event: ConsoleEvents::COMMAND)]` reads `$event->getInput()->getOption('tenant')` if `hasOption('tenant')` and sets the context (or disables the command with `disableCommand()`, exit 113). `setOption()` in the listener is discarded (input is re-bound). Prefer an explicit `TenantScopedCommand` base/trait that calls `TenantScopedRunner` for `--tenant=<slug>` or loops for `--all`. `ConsoleEvents::TERMINATE` closes the connection. https://symfony.com/doc/current/components/console/events.html

---

## 3. Doctrine DBAL 4 / ORM 3 / PostgreSQL / PgBouncer with ONE credential

### 3.1 DBAL 4 connection lifecycle **(verified in vendor: dbal 4.5.0 `Connection.php`)**

- `protected function connect(): DriverConnection` is lazy: returns `$this->_conn` if set, else `$this->_conn = $this->driver->connect($this->params)`; if `autoCommit === false` it begins a transaction immediately (keep `auto_commit: true`).
- `public function close(): void { $this->_conn = null; $this->transactionNestingLevel = 0; }` — no ROLLBACK, no driver call; the native PDO is released when the last reference goes. `isConnected()` is `$this->_conn !== null`. `getParams()` returns the configured params; `$params` is private and immutable, so **the only way to change `dbname` is in a `Driver\Middleware::connect()` override** (which is what the design does). `getNativeConnection()` exposes the PDO (use only for the `current_database()` assertion if you avoid DBAL there).
- `Doctrine\DBAL\Driver\Middleware\AbstractDriverMiddleware::connect(#[SensitiveParameter] array $params): DriverConnection` is the override point; `AbstractConnectionMiddleware` wraps `prepare/query/exec/beginTransaction/…` (use it for `TenantBoundConnection`). https://www.doctrine-project.org/projects/doctrine-dbal/en/latest/reference/architecture.html
- DoctrineBundle: `#[AsMiddleware(connections: ['tenant'], priority: 100)]` on a class implementing `Doctrine\DBAL\Driver\Middleware` **(verified in vendor: attribute has `array $connections = [], ?int $priority = null`)**; higher priority = applied earlier/outermost (its `connect()` runs first and passes the rewritten params inward, so the Debug/Idle middlewares log the real dbname). Implement `ConnectionNameAwareInterface::setConnectionName(string)` if one middleware serves several connections. Tag fallback `doctrine.middleware`. https://symfony.com/bundles/DoctrineBundle/current/middlewares.html
- `wrapper_class` must extend `Doctrine\DBAL\Connection` (DoctrineBundle `ConnectionFactory` validates) — not needed; the driver-level `Driver\Connection` decorator sees every statement, including ORM ones.
- Config keys (DoctrineBundle 3.3): `doctrine.dbal.connections.tenant.{url|dbname|host|port|user|password|driver|server_version|wrapper_class|driver_class|options|idle_connection_ttl|dbname_suffix|sslmode|…}`; `doctrine.orm.entity_managers.tenant.{connection, mappings, filters, metadata_cache_driver, query_cache_driver, result_cache_driver, second_level_cache, dql}`. `auto_generate_proxy_classes`, `proxy_dir`, `enable_lazy_ghost_objects` are removed in 3.0 (native lazy objects, PHP ≥ 8.4). https://symfony.com/bundles/DoctrineBundle/current/configuration.html , vendor `UPGRADE-3.0.md`.
- REC: set `server_version: '17'` on the tenant connection so DBAL never connects just to detect the platform (a reconnect on a placeholder database would fail closed and confuse cache warm-up).

### 3.2 PostgreSQL cannot switch database on an open connection

- libpq fixes `dbname` at `PQconnectdb*`; `PQreset` reconnects with the same parameters; there is no "use database" call. https://www.postgresql.org/docs/current/libpq-connect.html . Hence switch = `close()` + lazy reconnect through the middleware.
- Transaction state: because `close()` drops the handle without ROLLBACK, `TenantScopedRunner` must `if ($conn->isTransactionActive()) { $conn->rollBack(); }` **before** `close()`; a socket dropped mid-transaction is rolled back server-side, but under PgBouncer transaction pooling the server connection is discarded, which costs a reconnect.
- Entity manager: after `close()` call `$tenantEm->clear()` (identity map leak between societies); if `!$tenantEm->isOpen()` (closed by a flush exception: `EntityManagerClosed`) call `$registry->resetManager('tenant')`.

### 3.3 Caches **(C3)**

- DBAL result cache key = sha1 of `query=<sql>&params=…&types=…&connectionParams=sha256(serialize(configured params minus password))` → identical for all tenants. Disable `result_cache` on `tenant`, or decorate the pool with a tenant-namespaced `CacheItemPoolInterface` (`t:<id>:` prefix), and the same for ORM `second_level_cache.region_cache_driver` and `result_cache_driver`. Metadata and DQL query caches are shared safely.

### 3.4 Prepared statements, PgBouncer **(C1)**

- DBAL `pdo_pgsql` sets `Pdo\Pgsql::ATTR_DISABLE_PREPARES = true` unless the driver option is explicitly `false` **(verified in vendor)** → unnamed statements (`PQexecParams`), which PgBouncer supports in transaction pooling ("Protocol-level prepared plans: Yes (needs non-zero `max_prepared_statements`)" is only for *named* plans). `PDO::ATTR_EMULATE_PREPARES` is documented as available only for OCI, Firebird and MySQL. https://www.php.net/manual/en/pdo.setattribute.php , https://www.pgbouncer.org/features.html
- Keep `max_prepared_statements = 200` (PgBouncer ≥ 1.21, default 200) for other clients; `DEALLOCATE ALL`/`DISCARD ALL` clear tracking. https://www.pgbouncer.org/config.html
- Transaction pooling forbids session state: SET/RESET, LISTEN, `WITH HOLD` cursors, session advisory locks, `PREPARE`, `PRESERVE ROWS` temp tables ("Never"). `server_reset_query` is **not** run in transaction pooling unless `server_reset_query_always = 1` (costly: `DISCARD ALL` after every transaction). `track_extra_parameters` can only track `GUC_REPORT` parameters.
- Pools are per (database, user): with one user you still get one pool per tenant database. Set `default_pool_size`, per-db `max_db_connections`, and **`max_user_connections`** (global cap for the single `app` user, otherwise one hot tenant can starve others), `server_idle_timeout` to drain idle societies. `[databases]`: explicit entries per tenant (generated at provisioning, `RELOAD`) rather than the `*` wildcard, so an unknown dbname is refused by the pooler; `auth_query` is unnecessary with one user in `auth_file`. DSN param `application_name=tenant:<id>` (supported by DBAL's `constructPdoDsn` **(verified in vendor)**) makes `pg_stat_activity` auditable.

### 3.5 Per-tenant server settings without per-tenant roles

- `ALTER ROLE app IN DATABASE tenant_x SET statement_timeout = '5s'` (role-and-database-specific; applied at session start; overrides `ALTER ROLE … SET` and `ALTER DATABASE … SET`). `ALTER DATABASE tenant_x SET …` works for any role. Both are login-time, so PgBouncer-safe. https://www.postgresql.org/docs/current/sql-alterrole.html , https://www.postgresql.org/docs/current/sql-alterdatabase.html

### 3.6 Defence in depth with ONE login role: options evaluated

| Option | Mechanism | Verdict |
|---|---|---|
| A. Plain single role `app` | `REVOKE CONNECT ON DATABASE … FROM PUBLIC; GRANT CONNECT … TO app` per DB; PgBouncer explicit `[databases]`; application-level `TenantBoundConnection` (context check before every statement + `SELECT current_database()` once after connect compared with the context). | **Baseline, required.** `pg_hba.conf` cannot help (same role must reach every DB). The DB cannot detect a wrong-dbname switch; only the app can. |
| B. `SET ROLE t_<id>` at runtime | `CREATE ROLE t_x NOLOGIN; ALTER DATABASE tenant_x OWNER TO t_x; GRANT t_x TO app WITH INHERIT FALSE, SET TRUE;` then `SET ROLE t_x` after connect. | **Rejected under transaction pooling.** `role` is session state, is **not** `GUC_REPORT` (flags: `GUC_IS_NAME | GUC_NO_SHOW_ALL | GUC_NO_RESET_ALL | …`, PG 17 `guc_tables.c`), so PgBouncer cannot track it and `RESET ALL` does not reset it; a session-level `SET ROLE` applies to one server connection while autocommit reads land on others → random `permission denied`. `SET LOCAL ROLE` works only inside explicit transactions (reverts on COMMIT/ROLLBACK; no effect outside a block). https://www.postgresql.org/docs/current/sql-set-role.html , https://www.postgresql.org/docs/current/sql-set.html , https://www.postgresql.org/docs/current/sql-grant.html |
| C. Default role bound at login (**REC if you want owner separation**) | Same roles as B, but no runtime SET: `ALTER ROLE app IN DATABASE tenant_x SET role = 't_x'`. Every session `app` opens to `tenant_x` starts as `current_user = t_x`; `RESET ROLE`/`DISCARD ALL` (which runs `SET SESSION AUTHORIZATION DEFAULT` + `RESET ALL`) revert to the connection-time setting, i.e. still `t_x`. `app` itself owns nothing and has no table grants, so DDL by the app role and accidental privilege creep are impossible; migrations run as `t_x` automatically. | Pooling-safe, zero code. Still **not** a cross-tenant guard: a connection to `tenant_x` is always `t_x`; if the app mis-switched, the data is still `tenant_x`'s. Adds one `NOLOGIN` role per society and `GRANT t_x TO app WITH SET TRUE` at provisioning. |
| D. Per-tenant login roles (v3.1 design) | password per society in the vault. | The only option where the **database** rejects a mis-switch (wrong password for the wrong db). Costs: vault per tenant, rotation job, PgBouncer `auth_query`. If the new requirement stands (one credential), accept that cross-tenant protection is application-level only (A), optionally with C for containment. |

- `GRANT … WITH SET FALSE` is explicitly "not a complete safeguard" (ownership changes and Trojan-horse functions); roles whose privileges should be inherited but not assumed should own no objects.
- `DISCARD ALL` cannot run inside a transaction and includes `DEALLOCATE ALL`, `UNLISTEN *`, `pg_advisory_unlock_all()`, `DISCARD PLANS/TEMP/SEQUENCES`. https://www.postgresql.org/docs/current/sql-discard.html

### 3.7 Credential rotation with one role

- `ALTER ROLE app PASSWORD '…'`: existing server connections stay; PgBouncer re-reads `auth_file` on `RELOAD`; the app's resolver keeps the secret in memory for 10 min and retries once on auth failure (v3.1 8.8 still applies, just once instead of per tenant). Use SCRAM; `scram_iterations` is tracked by PgBouncer by default.

---

## 4. SOLID / shared-kernel design for a modular monolith

### 4.1 What is shared kernel (`src/Shared`)

| Service | Kind | Why shared |
|---|---|---|
| `TenantId` value object, `ResolvedTenant` (readonly DTO: id, slug, type, status, displayName) | Domain types | Every module's events carry `tenantId`; no module may import Tenancy's entities. |
| `TenantContext` (final, `ResetInterface`, set-once, `current(): ResolvedTenant`, `currentOrNull()`) | Shared infrastructure, concrete | One state holder for HTTP, Messenger, CLI. |
| `TenantContextInterface`? | **No.** | One implementation, tested directly; an interface here is speculative generality. Keep concrete and `final`. |
| `TenantResolver` (`resolve(Request): ResolvedTenant|null`) | Tenancy module port + impl | Reads header/host, consults `TenantDirectory`. Interface **yes**: `TenantDirectory` is a port (landlord DB + cache) with an in-memory fake for tests. |
| `TenantConnectionResolver` (`forCurrentTenant(): ConnectionTarget`) | Tenancy infrastructure, interface | Implementations: landlord-table + vault (prod), static map (tests). Consumed only by the DBAL middleware. |
| `TenantScopedRunner` (`run(TenantId, callable): mixed`) | Shared infrastructure, concrete | Only place that pushes/pops context and closes the tenant connection; depends on `TenantContext`, `ManagerRegistry`, `TenantDirectory`. |
| `CurrentUser` / `CurrentProfile` | IAM port (`CurrentActor` interface) implemented by a Symfony `Security`-backed adapter | Business modules need `actorId` for events and scope checks; they must not depend on `Symfony\Bundle\SecurityBundle\Security` directly. |
| `Clock`, `IdGenerator` ports | Shared ports | Already in the guide. |

### 4.2 One context, three adapters (no duplication)

- `Shared/Infrastructure/Tenancy/Http/TenantResolveListener` (kernel.request 20) → `TenantResolver` → `TenantContext::set()` + `_tenant_id` attribute; throws problem exceptions.
- `Shared/Infrastructure/Tenancy/Messenger/TenantStampMiddleware` (dispatch: stamp from context; receive: `TenantScopedRunner`).
- `Shared/Infrastructure/Tenancy/Console/TenantOptionListener` or `TenantScopedCommand` trait (`--tenant`, `--all`) → `TenantScopedRunner`.
- All three call the same two primitives (`TenantDirectory::bySlug()` and `TenantScopedRunner::run()`); the HTTP adapter does not use the runner because the context lives for the whole request and the connection is closed by the terminate listener.
- DBAL side: `TenantDriverMiddleware` (`#[AsMiddleware(connections: ['tenant'], priority: 100)]`) → `TenantDriver extends AbstractDriverMiddleware` → `TenantBoundConnection extends AbstractConnectionMiddleware`.

### 4.3 DI patterns to use

- `#[AsAlias(OrderWriter::class)]` for port → adapter binding (guide §4); `#[AsDecorator(decorates: 'api_platform.openapi.factory')]` + `#[AutowireDecorated]` for API Platform decoration; `#[AutoconfigureTag('app.tenant_assertion')]` on an interface and `#[AutowireIterator('app.tenant_assertion')]` / `#[AutowireLocator('app.permission_assertion', indexAttribute: 'permission')]` with `#[AsTaggedItem(index: 'orders:read', priority: 10)]` for the voter's assertions (open/closed). `getDefaultPriority()` / `default_index_method` are deprecated in 8.1; use `#[AsTaggedItem]`. https://symfony.com/doc/current/service_container/tags.html , https://symfony.com/doc/current/service_container/service_decoration.html , https://symfony.com/doc/current/service_container/service_subscribers_locators.html
- `#[Autowire(service: 'api_platform.doctrine.orm.state.item_provider')]` for composition instead of decoration (section 1.3).
- Exclude `Domain` and `Contracts` from autowiring (guide §12); put `#[AsEventListener]`, `#[AsMiddleware]`, `#[AsDecorator]` only in `Infrastructure`.

### 4.4 Testing strategy (Pest 4 on PHPUnit 12 or plain PHPUnit)

- Pest has no official Symfony plugin; use `uses(KernelTestCase::class)->in('Integration')` and `uses(ApiTestCase::class)->in('Api')`, or stay on PHPUnit. Third-party `bentools/pest-symfony-kernel` exists but declares Symfony ≤ 7.x. API Platform 5 moved `ApiTestCase` to the `api-platform/test` package (`ApiPlatform\Test\ApiTestCase`); `$alwaysBootKernel` now defaults to false. https://api-platform.com/docs/symfony/testing/ , https://api-platform.com/docs/core/upgrade-guide/
- Unit: `TenantContext` set-once/reset; `TenantResolver` with a fake `TenantDirectory` (header/host agreement, normalisation, misses); `TenantScopedRunner` with an in-memory `ManagerRegistry` double asserting rollback → clear → close → restore order; `TenantStampMiddleware` with `Envelope`s with/without `ReceivedStamp`.
- Integration (two PostgreSQL databases `test_tenant_a`, `test_tenant_b` + landlord): `TenantBoundConnection` throws `TenantConnectionMismatch` when the context changes mid-query; `current_database()` assertion; a query without context fails closed (`TenantNotResolved`); `getManagerForClass()` mapping test for every entity; DBAL result cache disabled on `tenant`.
- API: `ApiTestCase` requests with `X-Tenant: a` then `X-Tenant: b` on the same client (`disableReboot()` to prove reset between requests in one process), conflicting header/host → 400 problem+json with the right `type`; suspended → 423; route-table architecture test for `_tenant` defaults; every operation has a `security` expression with a permission slug (guide §22).
- Worker: dispatch with stamp → handler sees tenant; without stamp → failure transport; `in-memory://` transport.
- Use `dama/doctrine-test-bundle` only for the landlord connection; tenant databases are switched by closing the connection, which does not play with DAMA's static connection wrapper — truncate or recreate them in `beforeEach` instead.

---

## 5. Top URLs

1. https://api-platform.com/docs/core/state-providers/ (and `/state-processors/`) — provider/processor contracts, composition with Doctrine service ids
2. https://api-platform.com/docs/symfony/security/ — expression variables, `securityPostDenormalize`, `throw_on_access_denied`, `AccessDeniedException` with `detail`
3. https://api-platform.com/docs/core/errors/ — status resolution order, `ProblemExceptionInterface`, `#[ErrorResource]`, `exception_to_status` not merged
4. https://api-platform.com/docs/core/upgrade-guide/ — 5.0 removals (`enable_link_security`, `resource_class_directories`, ObjectMapper processors split, `api-platform/test`)
5. https://api-platform.com/docs/core/filters/ — `HeaderParameter`/`QueryParameter`, `required`, `constraints` + `nativeType`
6. https://api-platform.com/docs/core/openapi/ — `OpenApiFactoryInterface` decoration, `Model\Parameter`, `Model\Server`
7. https://symfony.com/bundles/DoctrineBundle/current/middlewares.html — `#[AsMiddleware(connections:, priority:)]`, `AbstractDriverMiddleware::connect()`
8. https://www.pgbouncer.org/config.html and https://www.pgbouncer.org/features.html — pool modes, `server_reset_query(_always)`, `max_prepared_statements`, `max_db_connections`, `max_user_connections`, feature matrix
9. https://www.postgresql.org/docs/current/sql-alterrole.html (with `sql-set-role`, `sql-set`, `sql-discard`, `sql-grant`) — `ALTER ROLE … IN DATABASE … SET`, login-time semantics, `SET LOCAL`, `DISCARD ALL`, `WITH SET`
10. https://symfony.com/doc/current/messenger.html and https://symfony.com/doc/current/scheduler.html — middleware/stamps, Doctrine middleware args, reset between messages, `CallbackMessageProvider` fan-out

Also used: https://api-platform.com/docs/core/extensions/ , https://api-platform.com/docs/core/dto/ , https://api-platform.com/docs/core/pagination/ , https://api-platform.com/docs/core/serialization/ , https://api-platform.com/docs/core/subresources/ , https://api-platform.com/docs/core/events/ , https://api-platform.com/docs/core/configuration/ , https://api-platform.com/docs/symfony/testing/ , https://api-platform.com/docs/symfony/jwt/ , https://github.com/api-platform/core/issues/3577 , https://symfony.com/doc/current/event_dispatcher.html , https://symfony.com/doc/current/reference/dic_tags.html , https://symfony.com/doc/current/components/runtime.html , https://symfony.com/doc/current/routing.html , https://symfony.com/doc/current/routing/custom_route_loader.html , https://symfony.com/doc/current/security/firewall_restriction.html , https://symfony.com/doc/current/components/console/events.html , https://symfony.com/doc/current/service_container/tags.html , https://symfony.com/doc/current/service_container/service_decoration.html , https://symfony.com/doc/current/service_container/service_subscribers_locators.html , https://symfony.com/bundles/DoctrineBundle/current/configuration.html , https://www.doctrine-project.org/projects/doctrine-dbal/en/latest/reference/configuration.html , https://www.doctrine-project.org/projects/doctrine-dbal/en/latest/reference/architecture.html , https://www.doctrine-project.org/projects/doctrine-orm/en/latest/reference/filters.html , https://www.doctrine-project.org/projects/doctrine-orm/en/latest/reference/second-level-cache.html , https://www.postgresql.org/docs/current/libpq-connect.html , https://www.pgbouncer.org/2023/10/pgbouncer-1-21-0 , https://www.php.net/manual/en/pdo.setattribute.php , https://www.php.net/manual/en/class.pdo-pgsql.php , https://www.rfc-editor.org/rfc/rfc9457.html , https://symfonycasts.com/screencast/api-platform-security (API Platform 3 course: voters, state processors, query extensions; concepts carry to 5).
