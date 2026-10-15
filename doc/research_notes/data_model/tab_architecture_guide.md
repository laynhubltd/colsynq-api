Architecture guide: SOLID, DDD, event-only modules
Style: Domain-Driven Design modular monolith · Hexagonal (ports and adapters) · CQRS inside each module · modules talk to each other only through events
Stack: PHP 8.4+ · Symfony 8.1 · API Platform 5 · Doctrine ORM 3 · PostgreSQL · Symfony Messenger
This guide states the rules every contributor and every coding agent must follow. It does not fix a list of modules or folders. Module names, class names and paths below are examples only. Apply the same pattern to whatever module you are working on.
How to read the rules:
MUST or MUST NOT is a hard rule. CI enforces most of them.
SHOULD is the default. Deviating from it needs a reason written in the PR.
<Module>, <Aggregate> and <Event> are placeholders. Examples use names like Orders, Billing and Ledger for illustration only.
Items marked (verify) use syntax that was not confirmed against a primary source. Check them against the installed package version before relying on them.
1. Core rules
#
 | Rule
 | Meaning
 | 
1
 | Events only between modules
 | A module never calls, queries or reads another module. It only reacts to that module's published events.
 | 
2
 | Events carry what consumers need
 | A consumer should never have to ask the producer for more data.
 | 
3
 | State and event commit together
 | A change and its events are saved in the same database transaction.
 | 
4
 | Consumers are idempotent
 | Any event may arrive more than once. Handling it twice must have the same effect as once.
 | 
5
 | Dependencies point inward
 | Infrastructure depends on Application, which depends on Domain. Domain depends on nothing outside itself.
 | 
6
 | Aggregates protect their rules
 | No public setters. State changes only through methods that check the rules.
 | 
7
 | Each module owns its data
 | No other module reads or writes its tables. No foreign keys or joins across modules.
 | 
8
 | One database per tenant
 | Each tenant's business data lives in its own database; the landlord database holds the tenant registry, users, profiles and access control. Every event carries the tenant ID.
 | 
9
 | One owner per business decision
 | Only the module that owns a fact may decide on it, for example whether a balance is enough. Others ask by event and wait for the answer.
 | 
10
 | SOLID
 | Small classes with one job. Extend by adding classes, not by editing switches.
 | 
2. What a module is
A module is one bounded context: a part of the business with its own language, rules and data. Examples: Orders, Inventory, Billing, Ledger, Notification.
Each module:
MUST own its tables, preferably in its own database schema.
MUST publish the facts other modules may care about, as integration events.
MUST keep any data it needs from other modules in its own local tables, filled from their events.
MUST NOT share entities, repositories or services with other modules.
Picking boundaries:
Split where the language changes. For example, "order" means different things to sales and to the warehouse.
Split where the rules change at different speeds or are owned by different people.
Keep things that must change together in one transaction inside one module.
When unsure, start with fewer, larger modules. Splitting later is easier than merging.
The shared kernel is a small common area, for example src/Shared. It holds only what every module agrees on and rarely changes:
base value objects, such as money, IDs and the tenant ID;
the event envelope base class;
exception marker interfaces;
ports for the clock and ID generation.
Two modules both needing a concept is not a reason to put it in the shared kernel. Each module models its own version.
2.1 Control plane and data plane
Modules live in one of two kinds of database (Multi-tenancy tab):
Database
 | Modules (examples)
 | Holds
 | 
Landlord (control plane, one)
 | Tenancy, IAM (identity and access)
 | Tenant registry and sign-up policy, tenant databases, domains, plans and features, tenant jobs; global User and one UserProfile per user per tenant, authenticators, tokens, API keys; permissions, roles, grants, audit events (User, access control and tenancy tab)
 | 
Tenant (data plane, one per society)
 | Business modules: business units, catalogue, requests, inventory, procurement, sales, ventures, assets, payments, ledger, reconciliation
 | That society's business data, its outboxes and inboxes, projections
 | 
Sign-in, the society switcher and every access check read only the landlord.
Business handlers run after the Tenant Resolver Service has switched the tenant connection (X-Tenant slug or host, priority 20, after the router).
A module belongs to exactly one plane. Landlord modules never read tenant databases; business modules never read the landlord. They exchange events like any other modules (for example UserProfileCreatedV1 feeds a profile-name projection in tenant databases).
Every module still keeps its own PostgreSQL schema inside its database.
3. Layers inside a module
3.1 Example layout
Folder names are a suggestion. Keep whatever names the codebase already uses, as long as the four layers stay separate.
src//
├── Domain/            # Pure PHP: aggregates, value objects, domain events, rules, ports
├── Application/       # Commands, queries, handlers, projections, workflows, event translators
├── Contracts/         # Published integration events and their schemas (the only public part)
└── Infrastructure/    # Doctrine, API Platform, Messenger adapters, console commands
Example: src/Orders/Contracts/Event/OrderPlacedV1.php.
3.2 What each layer may contain
Layer
 | Contains
 | MUST NOT contain
 | May depend on
 | 
Domain
 | Aggregates, entities, value objects, domain events, rules and policies, port interfaces, domain exceptions
 | Framework code, ORM attributes, SQL, HTTP, messaging, any Contracts class
 | PHP and the shared kernel's domain types
 | 
Application
 | Commands, queries, handlers, projections, workflows (process managers), translators, DTOs
 | SQL, HTTP request objects, API resources
 | Its own Domain and Contracts, other modules' published events only, the shared kernel
 | 
Contracts
 | Integration events, JSON schemas
 | Behaviour, domain types
 | The shared kernel's event base class only
 | 
Infrastructure
 | Repositories, ORM mapping, API resources and state classes, CLI, messaging adapters
 | Business rules
 | Everything in its own module, the shared kernel, the framework
 | 
4. SOLID in practice
Principle
 | Rule
 | Example
 | 
Single responsibility
 | One handler per command, query, projection or translator. Change one aggregate per transaction.
 | PlaceOrderHandler only places orders. Pricing lives in a separate PricingPolicy.
 | 
Open/closed
 | Add behaviour with new classes (policies, strategies, rules, adapters), not new switch branches.
 | A new discount rule is a new DiscountPolicy class tagged with #[AsTaggedItem].
 | 
Liskov substitution
 | Every implementation of a port passes the same contract test.
 | The Doctrine and in-memory repositories both pass OrderRepositoryContractTest.
 | 
Interface segregation
 | Keep ports small and role-specific.
 | Separate OrderReader and OrderWriter instead of one wide repository.
 | 
Dependency inversion
 | Domain and Application define ports. Infrastructure implements them.
 | OrderWriter lives in Domain. DoctrineOrderRepository implements it and is bound with #[AsAlias].
 | 
// Domain owns the port.
interface OrderWriter
{
    public function save(Order $order): void;
}
// Infrastructure implements it.
#[AsAlias(OrderWriter::class)]
final class DoctrineOrderRepository implements OrderWriter, OrderReader { /* ... */ }
// Open/closed: a new rule is a new class.
interface DiscountPolicy
{
    public function apply(Money $subtotal, Customer $customer): Money;
}
#[AsTaggedItem('loyalty')]
final readonly class LoyaltyDiscount implements DiscountPolicy { /* ... */ }
5. The integration rule: events only
A module learns about another module only by consuming its published integration events. It MUST NOT call another module's services, dispatch its commands, run its queries, read its tables, or import any of its classes except published events.
5.1 Inside versus across modules
Situation
 | Use
 | 
Inside one module
 | Its own command bus and query bus. Internal domain events may be handled in the same transaction.
 | 
Across modules
 | Integration events only, delivered asynchronously through the outbox (section 7).
 | 
5.2 Typical flow
sequenceDiagram
    autonumber
    participant API as API processor (module A)
    participant H as Command handler (module A)
    participant AGG as Aggregate (module A)
    participant OB as Outbox (same transaction)
    participant W as Worker
    participant B as Handler (module B)
    API->>H: Command
    H->>AGG: Business method
    AGG-->>H: Domain events recorded
    H->>OB: Translated integration events (same transaction)
    Note over H,OB: COMMIT: data and events saved together
    W->>OB: Consume
    W->>B: Integration event
    B->>B: Idempotency check, update own data, maybe publish own events
5.3 "But I need that data now"
Need
 | What to do
 | 
Display another module's data
 | Keep a local projection with just the fields you need (section 8).
 | 
Check that a referenced thing exists
 | Check your local projection. If it might be stale, accept provisionally and let the owner confirm or reject by event.
 | 
A decision the other module owns, such as "is the balance enough?"
 | Ask by event, wait for the answer. Publish a request event. The owner publishes a reserved or refused event. Your workflow continues (section 9).
 | 
One screen showing several modules
 | Compose at the edge: the client calls each module's API. Alternatively, build a read-only reporting projection that consumes events.
 | 
Show the result of your own write
 | Return it from your own module. Show downstream effects as "pending" until their events arrive.
 | 
6. Designing events
6.1 Two kinds of events
 | Domain event
 | Integration event
 | 
Where
 | The module's Domain layer
 | The module's Contracts layer
 | 
Who reads it
 | The same module only
 | Other modules
 | 
Content
 | May use domain types
 | Primitives only
 | 
Stability
 | Free to change
 | Versioned. Breaking changes need a new version
 | 
Delivery
 | In process, in the transaction
 | Asynchronous, through the outbox
 | 
Naming
 | Past tense, for example OrderPlaced
 | Past tense plus version, for example OrderPlacedV1. Wire name: <app>.<module>.<event>.v1
 | 
A translator in the Application layer turns domain events into integration events. It adds the data consumers need and leaves out internals.
6.2 Envelope
Every integration event carries these fields. They are shaped after CloudEvents.
Field
 | Meaning
 | 
eventId
 | UUID v7, unique. Used for deduplication
 | 
eventType
 | Stable wire name
 | 
occurredAt
 | When it happened (UTC, ISO 8601)
 | 
tenantId
 | The tenant. Mandatory
 | 
aggregateId, aggregateVersion
 | What changed, and its version, for ordering
 | 
correlationId
 | The original user action, kept through the whole flow
 | 
causationId
 | The message that directly caused this one
 | 
actorId
 | Who or what did it
 | 
traceparent
 | Trace context for OpenTelemetry
 | 
abstract readonly class IntegrationEvent
{
    public function __construct(
        public string $eventId,
        public string $occurredAt,
        public string $tenantId,
        public string $aggregateId,
        public int $aggregateVersion,
        public string $correlationId,
        public ?string $causationId,
        public ?string $actorId,
    ) {}
    abstract public static function eventType(): string;
}
// Example only
#[AsMessage(transport: 'integration_events', serializedTypeName: 'app.orders.order_placed.v1')] // (verify)
final readonly class OrderPlacedV1 extends IntegrationEvent
{
    // envelope arguments, then payload:
    // public string $customerId, public int $totalMinor, public array $lines ...
}
6.3 Payload rules
Use primitives only: strings, integers, booleans, ISO dates, UUID strings, and arrays of these.
Store money as an integer in minor units (for example kobo or cents), with the field name saying so (totalMinor, amountKobo).
Include what consumers act on, so they never need to call back.
Include no personal data beyond what consumers need. Prefer IDs.
State a fact, not an instruction. PaymentRequested is right. PleaseDebitAccount is wrong.
6.4 Versioning
Change
 | How
 | 
Add an optional field
 | In place. Consumers ignore fields they don't know
 | 
Rename, remove or change the type of a field
 | Publish V2 alongside V1. Remove V1 when no consumer uses it
 | 
Change the meaning
 | New event type
 | 
Each event version has a JSON Schema next to the event class. CI validates fixtures against it.
Wire names are fixed with #[AsMessage(serializedTypeName: ...)].
An event catalogue (for example AsyncAPI 3.0) SHOULD be generated from the schemas.
7. Reliable delivery
7.1 Outbox
Goal: an event is published if and only if its change commits.
On Symfony 8.1:
Route integration events to a Doctrine transport on the same database connection as the module data.
Run command handlers inside the doctrine_transaction middleware.
Dispatch integration events inside the handler, without DispatchAfterCurrentBusStamp. The transport's insert then joins the open transaction.
#[AsMessageHandler(bus: 'command.bus')]
public function __invoke(PlaceOrder $command): void
{
    $order = Order::place(/* value objects */);
    $this->orders->save($order);
    foreach ($this->translator->translate($order->releaseEvents()) as $event) {
        $this->eventBus->dispatch($event);   // written in the same transaction
    }
}
On Symfony 8.2 and later: use the native Messenger outbox option and its relay worker. The dispatch rules stay the same.
With one database per tenant: a business module's outbox lives in the tenant database, so the business rows and the outbox row still commit together. One worker cannot consume Doctrine transports spread over many databases, so a relay forwards outbox rows to a central transport in the landlord (or Redis) with a TenantStamp; workers switch to the tenant before handling, and the inbox sits in the tenant database. Landlord modules (Tenancy, IAM) use an outbox in the landlord database directly. Details: Multi-tenancy tab, section 11.
7.2 Inbox (idempotency)
Delivery is at least once.
Each module keeps an inbox table with primary key (consumer, event_id).
A middleware inserts that row with ON CONFLICT DO NOTHING, in the same transaction as the handler's work. If the row already existed, the handler is skipped.
DeduplicateStamp (needs symfony/lock) MAY stop duplicate commands, for example from double clicks. It does not replace the inbox.
7.3 Ordering
Order is guaranteed per aggregate only. Use aggregateVersion to check it.
For snapshot events, which carry the full state, apply an event only if its version is newer than the stored one.
For change events that must apply in sequence, park an event whose earlier version is missing and retry it later. After a time limit, send it to the failure transport.
7.4 Retries and failures
Retry with exponential backoff and jitter (for example 3 retries, starting at 1 s and capped at 60 s).
After retries are exhausted, the message moves to a failure transport. Review that transport regularly and replay with messenger:failed:retry. The inbox makes replays safe.
A failed event SHOULD block later events for the same aggregate and consumer until it is resolved.
7.5 Event log
Keep an append-only log of every published event, written in the same transaction. Use it for audit and to fill new projections.
8. Local read models (projections)
A projection is a table owned by the consuming module. It holds only what that module needs from other modules' events. It also protects the module's own language from the producer's (an anti-corruption layer).
Example: an Orders module keeps orders.customer_status, filled from CustomerRegisteredV1 and CustomerSuspendedV1.
Rules:
One handler per projection. It only updates its table. No emails, no new events.
It is idempotent (inbox) and version-checked per source aggregate.
Each row stores the source event ID, the source version and the time it was projected.
Lag is measured and visible.
Never use a projection as the final word on a decision another module owns. Use ask-and-wait instead (section 9).
Build a new projection by replaying the event log. Rebuild into a new table, then swap it in.
9. Workflows across modules
Approach
 | Use when
 | 
Choreography: each module reacts on its own
 | Short flows with no waiting and no undo
 | 
Process manager: one module runs a state machine
 | Flows that wait, time out or need compensation
 | 
Rules for process managers:
A process manager lives in the module that owns the business outcome.
It stores its state in that module's tables.
It reacts to events and publishes events. It never calls other modules.
Every step is idempotent and keyed by the correlation ID.
Deadlines are stored as data and checked by a scheduled job, never held in memory.
Undoing a step means publishing a new event (for example ReservationReleased), never deleting data.
Example: checking funds before paying out.
sequenceDiagram
    autonumber
    participant P as Payments (process manager)
    participant L as Ledger
    participant X as Integrations
    P-->>L: PayoutRequestedV1
    alt funds available
        L-->>P: HoldPlacedV1
        P->>P: Collect approvals
        P-->>X: PaymentApprovedV1
        X-->>P: BankConfirmedV1 or BankFailedV1
        P-->>L: PaymentSettledV1 or PaymentFailedV1
        L->>L: Post or release the hold
    else not enough
        L-->>P: HoldRefusedV1
        P->>P: Reject and notify
    end
10. Writes (commands)
Commands are final readonly classes with primitive properties and a fromArray(array $data): self factory.
Commands are named in the imperative (PlaceOrder). Events are named in the past tense (OrderPlaced).
The handler turns primitives into value objects, loads the aggregate, calls one intention-revealing method, saves it, and dispatches translated events.
A command changes one aggregate. Other changes happen by reacting to events.
A handler never dispatches another module's commands and never reads another module's data.
If the outcome depends on other modules, return 202 Accepted with a pending state.
11. Reads (queries)
A query reads only its own module's tables and projections.
Handlers return DTOs, never entities.
Validate filters, sorts and includes in the handler. Repositories trust validated input.
Use left joins for optional relations.
Read tables may be denormalised for speed, because they can be rebuilt from events.
12. Persistence
Each module SHOULD have its own database schema. Modules MUST NOT have cross-module foreign keys, joins or reads. Store other modules' IDs as plain UUID columns.
Keep Doctrine mapping out of the Domain layer, for example in XML files in Infrastructure, so domain classes stay plain PHP.
Map value objects with custom DBAL types or embeddables.
Repository interfaces live in Domain. Doctrine implementations live in Infrastructure and are bound with #[AsAlias].
Generate IDs in the application, as UUID v7 through an ID-generator port, so events know IDs before saving.
A migration touches one module's schema only.
Exclude Domain and Contracts folders from service autowiring.
With database per tenant there are two entity managers: landlord (Tenancy and IAM mappings) and tenant (business module mappings, connection switched per request by a DBAL middleware). Mappings never overlap, so ManagerRegistry::getManagerForClass() and API Platform pick the right one. Landlord migrations run with the migrations bundle; tenant migrations run across all tenant databases with tenants:migrate (Multi-tenancy tab, section 10).
# Example: one mapping entry per module
doctrine:
    orm:
        mappings:
            Orders:
                type: xml
                is_bundle: false
                dir: '%kernel.project_dir%/src/Orders/Infrastructure/Persistence/Mapping'
                prefix: 'App\Orders\Domain\Model'
13. HTTP API with API Platform
Resources and documentation: use #[ApiResource] on resource classes in the Infrastructure layer. These classes define the endpoints and generate the OpenAPI documentation. All other OpenAPI customisation also lives in Infrastructure, for example operation summaries and descriptions, openapi: options, #[ApiProperty] examples and an OpenApiFactory decorator. Domain classes stay free of API attributes, so the Domain layer remains plain PHP.
Reads: a state provider dispatches a query. For simple reads of the module's own tables, stateOptions with the Object Mapper is fine.
Writes: a custom processor maps the input to a command and dispatches it. Do not use the generic persist processor for aggregates, or domain events will be skipped.
Every endpoint is mapped to a permission. A test fails if any endpoint is unmapped.
14. Multi-tenancy
Layer
 | Mechanism
 | 
Model
 | One database per tenant plus a landlord database (section 2.1). Full design: Multi-tenancy tab
 | 
HTTP
 | Tenant Resolver Service on kernel.request at priority 20 (after the router at 32, before the firewall at 8): X-Tenant slug, then host; per-route requirement Required, Optional or None; status check; switches the tenant connection. A tenant ID in the request body is never trusted
 | 
Identity
 | After the firewall, the user must have an active UserProfile in the resolved tenant, otherwise 404. Sign-up in a tenant creates the User and UserProfile together (User, access control and tenancy tab)
 | 
Database
 | A DBAL driver middleware supplies the tenant's connection details and fails closed when no tenant is set; per-tenant database roles; REVOKE CONNECT … FROM PUBLIC
 | 
Messaging
 | TenantStampMiddleware stamps the tenant on dispatch; TenantContextMiddleware restores it in workers and resets in finally; TenantScopedRunner for webhooks and console. Every event payload also carries tenantId
 | 
Long-running processes
 | Every tenant-aware service implements ResetInterface; the entity manager is cleared and the connection closed between requests and messages
 | 
Cache and other stores
 | Keys, locks, rate limiters, cache tags, topics, files and search documents prefixed by tenant
 | 
15. Errors
Domain exceptions implement one marker interface from the shared kernel. A kernel listener maps each marker to an HTTP status.
Marker
 | HTTP
 | 
Not found
 | 404
 | 
Conflict
 | 409
 | 
Unprocessable (a business rule was broken)
 | 422
 | 
Forbidden
 | 403
 | 
Generic domain error
 | 400
 | 
Failures that happen later, in another module, are never thrown back to the original caller. They become events (for example HoldRefusedV1) and visible states on the original record.
16. Money and the ledger
This section applies when the system keeps accounts.
Only the ledger module writes accounts and journal entries. Other modules publish business facts, and the ledger turns them into journals.
Each posting rule is a class handling one event type (open/closed principle).
Each journal references its source event. The source event ID is the idempotency key, so a replayed event never posts twice.
Holds are pending entries, placed and released by events.
The ledger publishes its own results, for example JournalPostedV1 and BalanceChangedV1. Other modules project the balances they need from these.
Money is always an integer in minor units.
#[AsTaggedItem(index: 'app.orders.order_placed.v1')]   // example
final readonly class OrderPlacedPostingRule implements PostingRule
{
    public function journal(IntegrationEvent $event, AccountResolver $accounts): JournalDraft
    {
        // Debit receivable / Credit sales
    }
}
17. Messenger setup
Required packages (install only with approval): symfony/messenger, symfony/doctrine-messenger, symfony/lock, symfony/scheduler, symfony/uid.
# config/packages/messenger.yaml (verify option names against the installed version)
framework:
    messenger:
        default_bus: command.bus
        buses:
            command.bus:
                middleware:
                    - App\Shared\Infrastructure\Messaging\TenantContextMiddleware
                    - validation
                    - dispatch_after_current_bus   # before doctrine_transaction
                    - doctrine_ping_connection
                    - doctrine_transaction
            query.bus:
                middleware:
                    - App\Shared\Infrastructure\Messaging\TenantContextMiddleware
            event.bus:
                default_middleware:
                    allow_no_handlers: true
                middleware:
                    - App\Shared\Infrastructure\Messaging\TenantContextMiddleware
                    - App\Shared\Infrastructure\Messaging\EventLogMiddleware
                    - doctrine_ping_connection
                    - doctrine_transaction
                    - App\Shared\Infrastructure\Messaging\InboxMiddleware
                    - doctrine_close_connection
        failure_transport: failed
        transports:
            integration_events:
                dsn: 'doctrine://default?queue_name=integration_events'
                serializer: messenger.transport.symfony_serializer
                options: { use_notify: true }
                retry_strategy: { max_retries: 3, delay: 1000, multiplier: 3, max_delay: 60000 }
            failed:
                dsn: 'doctrine://default?queue_name=failed'
The middleware class names above are examples. Run the worker with bin/console messenger:consume integration_events --time-limit=3600 --memory-limit=256M. A busy consumer can later get its own transport by routing its event types there, with no code changes.
18. Observability
Tracing: OpenTelemetry, with traceparent carried in the envelope.
Logs: structured, always including correlationId, causationId, eventId and tenantId.
Metrics: outbox backlog, consumer lag, failure-transport size, projection lag, workflow timeouts, duplicate deliveries skipped.
Runbooks: poison message, projection rebuild, replay from the event log, worker restart.
19. Testing
Level
 | What to test
 | 
Domain
 | Aggregates and policies, written as given–when–then, with no kernel
 | 
Port contracts
 | One shared test run against every implementation of a port
 | 
Translators
 | Event payload, envelope and type
 | 
Event contracts
 | Fixtures validate against JSON Schemas. Consumers keep fixtures of the events they consume
 | 
Projections
 | Duplicate, out-of-order and older-version events. A rebuild gives the same result
 | 
Process managers
 | Happy path, refusal, timeout and compensation, using a fake clock
 | 
Integration
 | Handlers with the database, using the in-memory:// transport
 | 
API
 | Endpoints, including tenant isolation
 | 
Architecture
 | Boundary rules (section 20)
 | 
20. Enforcing the boundaries
Tools: deptrac/deptrac, phpat/phpat (on PHPStan), phpstan/phpstan-symfony, phpstan/phpstan-doctrine. Write the rules generically, with a regex per module.
Rule
 | Check
 | 
Domain is framework-free
 | App\*\Domain has no Symfony\, Doctrine\ or ApiPlatform\ dependencies
 | 
Domain doesn't see contracts
 | App\*\Domain has no Contracts dependencies
 | 
Cross-module imports only via contracts
 | A module's non-Contracts code depends only on its own code, the shared kernel, and other modules' Contracts\Event
 | 
Contracts are plain
 | Event classes are final readonly and depend only on the shared event base
 | 
Commands are plain
 | Command classes are final readonly
 | 
Projections have no side effects
 | Projection classes don't depend on message buses or mailers
 | 
# deptrac.yaml (pattern; verify syntax)
deptrac:
  layers:
    - name: Orders
      collectors: [{ type: classNameRegex, value: '#^App\\Orders\\(?!Contracts\\)#' }]
    - name: OrdersContracts
      collectors: [{ type: classNameRegex, value: '#^App\\Orders\\Contracts\\#' }]
    # repeat per module
  ruleset:
    Orders: [Shared, OrdersContracts, BillingContracts]   # example
    OrdersContracts: [Shared]
Also test that every event has a schema and a fixture, and that every migration touches only one schema.
21. Recording decisions
Record each significant decision as an ADR in doc/adr/NNN-title.md, with context, decision, consequences and alternatives.
Baseline decisions this guide assumes:
A modular monolith.
Events-only integration.
Local projections instead of shared reads.
Outbox plus inbox (outbox in the tenant database, relayed to a central transport).
Versioned JSON events.
A framework-free domain.
UUID v7 identifiers.
One database per tenant; a landlord database for tenancy, users, profiles and access control.
A Tenant Resolver Service at priority 20 (after routing) with per-route tenant requirements compiled into the route.
Global User with one UserProfile per tenant, created automatically at sign-up.
Only the ledger writes money.
Boundaries enforced in CI.
22. Checklists
Adding a module
Name the module after a business capability, and confirm its boundary (section 2).
Decide its plane: landlord or tenant database (section 2.1).
Create Domain, Application, Contracts and Infrastructure layers.
Give it its own schema, mapping entry on the right entity manager, inbox table and migrations (landlord or tenant set).
Add boundary rules for it.
List the events it publishes and consumes, with schemas.
Declare its permission slugs (PermissionProvider) and any assertions.
Adding an event
Past-tense name, V1, stable wire name.
final readonly class, primitives only, full envelope including tenantId.
JSON Schema and fixture.
Dispatched inside the command transaction.
Adding a consumer
Idempotent through the inbox, with a test for duplicate delivery.
Handles ordering through aggregateVersion.
Updates only its own module's tables.
Every PR
No cross-module imports except published events. No cross-module calls, queries or table access.
The Domain layer has no framework code.
One handler per message, and one aggregate per command.
New behaviour added as new classes, not new switch branches.
Every route declares its tenant requirement; tenant never taken from the request body.
Every operation has a security expression with a permission slug; collections use scope filters.
Tenant-aware services reset between requests; caches and keys prefixed by tenant.
Money stored as integer minor units.
Tests at every level that applies, and boundary checks pass.
23. Glossary
Term
 | Meaning
 | 
Aggregate
 | Objects changed together under one consistency rule
 | 
Anti-corruption layer
 | Translation that keeps another module's language out of yours
 | 
Bounded context
 | A part of the business with its own model and language. In this guide, a module
 | 
Choreography
 | Modules reacting to events with no central coordinator
 | 
Domain event
 | A fact inside one module, not published
 | 
Event-carried state transfer
 | Events include the data consumers need
 | 
Inbox
 | A record of events each consumer has processed
 | 
Integration event
 | A published, versioned fact for other modules
 | 
Outbox
 | Events stored in the same transaction as the change, then delivered
 | 
Process manager
 | A stateful workflow driven by events
 | 
Projection
 | A local read model built from events
 | 
Published language
 | A module's set of integration events and schemas
 | 
Tenant
 | An isolated customer or organisation whose data must never mix with another's
 | 
