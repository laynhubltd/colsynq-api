# Event-Only DDD Modular Monolith: Established Good Practice (as of October 2026)

Scope: a single deployable made of modules (bounded contexts). Modules communicate **only** through published events. There are no synchronous cross-module calls and no cross-module query services. Every finding is marked:

- **[P]** primary source: the author's or standard body's own writing, spec, or repository.
- **[S]** secondary source: third-party summary, blog, or synthesis.
- **[U]** unverified: an inference, practitioner consensus or recollection that I could not check against a source in this session.

Verification note: in this session I checked these sources live: Dudycz's "Internal and external events" article, Shopify's engineering posts, AsyncAPI 3.0 status, and Grzybek's repository README structure. Other primary citations are well-known canonical works cited from knowledge. Their URLs are given, but I did not re-fetch them in October 2026.

---

## 0. Executive position

1. Event-only communication between modules is a legitimate, well-documented style. Its main references are Grzybek, Dudycz, Richardson and Fowler's "event-carried state transfer". It is **stricter than most reference implementations**: Grzybek's reference also uses a small number of commands and queries through module facades, and Shopify uses synchronous public APIs. [P]/[S]
2. To make event-only communication work, the design must:
   - (a) separate private domain events from public integration events;
   - (b) make integration events carry enough state for consumers to build **local read models**;
   - (c) use an outbox and inbox with idempotent consumers;
   - (d) use process managers for multi-module workflows;
   - (e) compose reads across modules at the **API/UI edge**, not module-to-module. [U] (synthesis of the sources below)

---

## 1. Foundational references

### 1.1 Kamil Grzybek, *Modular Monolith with DDD*
- Repository: https://github.com/kgrzybek/modular-monolith-with-ddd. It is a .NET reference app with four modules: Meetings, Payments, Administration and UserAccess. [P]
- Blog series: https://www.kamilgrzybek.com/blog/series/modular-monolith. Posts include "Modular Monolith: Primer", "Architectural Drivers", "Architecture Enforcement", "Integration Styles" and "Domain-Centric Design". [P]
- Key practices in the repository [P, from the README table of contents; the details are from knowledge]:
  - Each module has its own DB schema, and no module reads another module's tables.
  - Modules integrate asynchronously through **integration events** in a separate `IntegrationEvents` contracts assembly per module. This is the module's published language.
  - The repository uses an **in-memory events bus**, with the note that it can be swapped for a broker.
  - **Outbox**: domain events are saved in the same transaction and processed later. **Inbox**: incoming integration events are stored, then processed. Internal commands are also processed asynchronously from a table.
  - **Architecture tests** (NetArchTest) enforce module boundaries.
- "Integration Styles" post [P]. It compares four styles of module integration:
  - file transfer;
  - shared database;
  - direct call;
  - messaging.

  Grzybek recommends **messaging** as the default for autonomy. He accepts direct calls through a module's public API (facade) as a trade-off. Event-only is therefore his preferred style, but he does not require it.
- Caveat: the README carries a non-technical usage notice. Read it before reusing the code. [P]

### 1.2 Eric Evans and Vaughn Vernon
- Evans, *Domain-Driven Design* (2003), Part IV, "Strategic Design" [P]:
  - **Bounded Context** and **Context Map**.
  - The relationships between contexts: Shared Kernel, Customer/Supplier, Conformist, **Anticorruption Layer (ACL)**, Open Host Service (OHS) and **Published Language (PL)**.
  - In an event-only monolith, the public integration-event schema of each module **is** its Published Language, served as an Open Host Service. Each consumer's projection or handler acts as its ACL.
- Evans, *DDD Reference* (2015, free PDF, domainlanguage.com) [P]. It adds **Domain Events** as a first-class building block. Domain Events were not in the 2003 book.
- Vernon, *Implementing DDD* (2013) [P]:
  - Ch. 3, Context Maps.
  - Ch. 8, Domain Events. Ch. 8 covers event stores, publishing with "store and forward" (an early outbox), de-duplication, and the advice to **"design events with enough state"** to be useful to subscribers.
  - Ch. 13, Integrating Bounded Contexts. Ch. 13 covers REST versus messaging, "autonomous services" and the trade-offs of **event-carried data versus callbacks**. Vernon favours copying needed data into the consumer to keep the consumer autonomous.
- Vernon, *Reactive Messaging Patterns with the Actor Model* (2015) [P]. It maps Hohpe's EIP to actors and covers Process Manager, Message Expiration, Idempotent Receiver, Guaranteed Delivery and Correlation Identifier.
- Vernon, *DDD Distilled* (2016) [P]. It is a concise context-mapping summary.

### 1.3 Martin Fowler, "What do you mean by 'Event-Driven'?" (2017)
- URL: https://martinfowler.com/articles/201701-event-driven.html [P]
- It defines four patterns that people confuse:
  1. **Event Notification.** A thin event with an id. The receiver calls back for details. In an event-only monolith, a callback is a forbidden sync query, so pure notification only works if no data is needed.
  2. **Event-Carried State Transfer (ECST).** The event carries the state the consumer needs. The consumer keeps a local copy. Benefits: resilience, lower latency, no load on the source. Costs: data duplication and eventual consistency. ECST is **the core pattern for an event-only modular monolith**.
  3. **Event Sourcing.** State is persisted as an event log.
  4. **CQRS.** Separate read and write models.
- Fowler's warning: event notification makes the logic flow hard to see; "the danger is that it's very easy to make nicely decoupled systems with event notification, without realizing that you're losing sight of that larger-scale flow". This argues for process managers or catalogs (see §4, §6).
- Also from Fowler: "MonolithFirst" (2015) and "CQRS" (2011) bliki entries. [P]

### 1.4 Chris Richardson, microservices.io
These patterns apply equally to modules in a single process. [P] for each pattern page:
- Transactional Outbox: https://microservices.io/patterns/data/transactional-outbox.html. Polling publisher and transaction log tailing.
- Idempotent Consumer: https://microservices.io/patterns/communication-style/idempotent-consumer.html. Record processed message ids in the same transaction as the business update.
- Saga: https://microservices.io/patterns/data/saga.html. Choreography versus orchestration, compensating transactions, and **countermeasures for lack of isolation**: semantic lock, commutative updates, pessimistic view, reread value, version file, by value.
- Domain Event: https://microservices.io/patterns/data/domain-event.html
- CQRS: https://microservices.io/patterns/data/cqrs.html. Maintain a view by subscribing to events. This replaces cross-service queries.
- API Composition: https://microservices.io/patterns/data/api-composition.html. **This is the pattern purists use to avoid module-to-module queries.**
- *Microservices Patterns* (Manning, 2018), chapters 3–7. [P]
- Richardson has also written on modular monoliths and "Assemblage", and describes the modular monolith as a valid architecture. [S/U: posts on chrisrichardson.net 2023–2025]

### 1.5 Gregor Hohpe and Bobby Woolf, *Enterprise Integration Patterns* (2003)
- https://www.enterpriseintegrationpatterns.com [P]
- Patterns relevant here:
  - Event Message, Document Message, Command Message.
  - Publish-Subscribe Channel, Datatype Channel.
  - **Message Translator / Normalizer / Canonical Data Model.** These are the basis for internal-to-external event mapping.
  - **Correlation Identifier**, Message Sequence, Resequencer.
  - **Idempotent Receiver**, Guaranteed Delivery, **Dead Letter Channel**, Invalid Message Channel.
  - **Process Manager.**
  - Content Enricher. This is what a module does when it turns a thin internal event into a fat external event.
  - Claim Check. Use it for large payloads.
- Hohpe's essay "Your Coffee Shop Doesn't Use Two-Phase Commit" (2005) [P] is the classic argument for compensation over distributed transactions.

### 1.6 Oskar Dudycz, event-driven.io
All entries below are [P].
- "Internal and external events, or how to design event-driven API" (2023): https://event-driven.io/en/internal_external_events/. Verified in this session. Its main points:
  - Split events into internal (granular, module-private) and external (the public contract).
  - Treat events as an API.
  - Before exposing data, ask what the consumer actually needs.
  - Publish **summary events** that carry complete data.
  - Use **separate channels** for internal and external events.
  - Map internal to external in a handler that enriches the event, loading state **at the event's stream position** to avoid races.
  - Keep business calculations (such as totals) in the producer.
  - Deliver with an outbox or a subscription.
  - Don't disguise commands as events.
- "Dealing with Race Conditions in EDA using Read Models" (2025): https://event-driven.io/en/dealing_with_race_conditions_in_eda_using_read_models/. It advises:
  - using read models as an ACL;
  - tolerating partial state and out-of-order arrival;
  - splitting internal and external events.
- Further articles [P; titles from knowledge, URLs on event-driven.io]:
  - "Outbox, Inbox patterns and delivery guarantees explained".
  - "Simple patterns for events schema versioning" (upcasting, double write, weak schema).
  - "How to (not) do the events versioning?"
  - "Event versioning with Marten".
  - "What's the difference between a command and an event?"
  - "Saga and Process Manager — distributed processes in practice".
  - "Projections and read models in event-driven architecture".
  - "Rebuilding projections" posts.
  - "Ordering, grouping and consistency in messaging systems".
  - "Anti-patterns in event modelling – Clickbait event".
  - "Passive aggressive events".
  - "State obsession".
- Dudycz's open-source projects:
  - Emmett (TS): https://github.com/event-driven-io/emmett
  - Pongo
  - Marten (.NET), as co-maintainer

  They show these patterns in code. [P]

### 1.7 Mathias Verraes, Messaging Patterns
- https://verraes.net/#blog. The series "DDD and Messaging Architectures" (2019) is collected as *Messaging Patterns* on Leanpub / verraes.net. [P]
- Relevant patterns:
  - **Event Summary**: aggregate many fine-grained events into one.
  - **Completeness Guarantee**: emitted events carry enough to rebuild state.
  - **Fat Event versus Segregated Event**.
  - **Explicit Public Events**: separate public events from private ones.
  - **Domain Query** and **Forgettable Payloads** (GDPR: encrypt personal data or keep it out of the event).
  - **Throttling**, **Change Detection Events**.
  - **Passage of Time Event**: a scheduler publishes "DayHasPassed". This is a clean way to implement timeouts without calls.
  - **Decision Tracking**, **Correcting Events**.
- These are the most directly applicable published-language patterns for event-only design.

### 1.8 Case studies
- **Shopify** [P]:
  - "Deconstructing the Monolith" (2019): https://shopify.engineering/deconstructing-monolith-designing-software-maximizes-developer-productivity. Shopify chose a modular monolith over microservices. The codebase was split into about 37 components (the count comes from a secondary source).
  - "Enforcing Modularity in Rails Apps with Packwerk" (2020): https://shopify.engineering/enforcing-modularity-rails-apps-packwerk. Packwerk enforces boundaries through static analysis.
  - "Under Deconstruction: The State of Shopify's Monolith" (2022): https://shopify.engineering/shopify-monolith. The 2022 post reports that boundary enforcement alone did not create good boundaries. Packwerk's privacy checks were later moved to an extension.

  **Shopify mainly uses synchronous public component APIs, not event-only messaging.** It is a modularity case study, not a case study of event-only communication.
- **Spotify** [U]: there is no well-known primary write-up of a Spotify *modular monolith*. Spotify is usually cited for squads, Backstage and microservices, so don't cite it as a modular-monolith case study.
- **Amazon Prime Video** [P]: "Scaling up the Prime Video audio/video monitoring service and reducing costs by 90%" (2023, primevideotech.com). One service moved from serverless microservices to a single process. This example is about cost and deployment, not event design.
- **Others** [S]:
  - Root Insurance (Rails modular monolith).
  - Gusto ("packwerk" adoption).
  - Kraken Technologies / Octopus Energy (large Django modular monolith, talks by David Seddon; `import-linter` enforces layers) [P for import-linter: https://github.com/seddonym/import-linter].
- **Spring Modulith** [P]: https://spring.io/projects/spring-modulith. As of 2026 it is the most mature *framework-level* reference for event-based module integration:
  - `@ApplicationModuleListener`;
  - an **Event Publication Registry**, which is an outbox that records incomplete publications and resubmits them on restart;
  - event externalization to Kafka, AMQP and other brokers;
  - `Scenario` API for testing event flows;
  - architecture verification;
  - generated module documentation.

  It is worth reading even for PHP teams, because its design maps one-to-one onto Laravel listeners and queues. Oliver Drotbohm's talks on "Modulith" describe event-first module integration as the recommended default.
- **Milan Jovanović** [S]: "Event-Driven Communication Between Modules in .NET" (milanjovanovic.tech). Start with an in-process bus and add an outbox for reliability.
- **ABP.io** [S]: an article on outbox/inbox in a multi-module monolith, with a separate outbox and inbox per module.

---

## 2. Event types and design

### 2.1 Internal domain events versus public integration events
- **Domain event (internal or private).** It is raised by an aggregate and describes a fact in the module's ubiquitous language. It is fine-grained and may change freely. Consumers are the same module's handlers and projections. [P: Evans DDD Reference; Vernon IDDD ch. 8; Dudycz]
- **Integration event (external, public, or "published language").** It is a versioned contract and is coarser. It is often a summary or snapshot, and it is built by a **translator or enricher** inside the producing module. [P: Dudycz; Verraes "Explicit Public Events"; Grzybek `IntegrationEvents` projects]
- Rules:
  1. Never publish internal events directly. Map them explicitly. [P: Dudycz]
  2. Publish integration events in a dedicated **contracts package** per module. Consumers depend only on that package, never on the producer's domain. [P: Grzybek] In PHP, use a namespace such as `Modules\Billing\Contracts\Events`, enforced with architecture tests (§8.5).
  3. Name events in the past tense, using business terms ("InvoiceIssued", not "InvoiceRowInserted"). [P: Vernon; Verraes]
  4. Don't put a command in an event's clothing. If there is exactly one consumer that must act, and the producer cares about the outcome, it is a command. [P: Dudycz] In an event-only system, keep commands **inside** a module. A process manager in the consuming module reacts to a fact. The producer never "tells" another module what to do.

### 2.2 Event-carried state transfer: designing integration events with enough data
- The consumer must be able to do its job with only the events, because no callback is allowed. [P: Fowler ECST; Vernon IDDD ch. 13]
- Design method [U, practitioner synthesis of Dudycz and Verraes]:
  1. List each consumer's **use cases**: what it decides or displays.
  2. Derive the minimal attribute set the consumer needs. Make that attribute set part of the contract.
  3. Include **derived business values** computed by the producer (totals, statuses), so that consumers don't re-implement rules. [P: Dudycz]
  4. Include an **aggregate version or sequence** and **occurred_at**, so that consumers can ignore stale updates.
  5. Prefer **"state snapshot" (document) events** for entity-replication flows, such as `CustomerProfileChanged` with the full public profile. Prefer **"delta" events** for business-process facts, such as `OrderPlaced`.
  6. Keep personal data minimal. Use **Forgettable Payloads** or crypto-shredding for GDPR. [P: Verraes]

### 2.3 Fat versus thin events
| | Thin (notification) | Fat (ECST / document) |
|---|---|---|
| Coupling | Runtime coupling: callback required | Schema coupling only |
| Event-only compatible? | **No**, unless the consumer needs only the id | **Yes** |
| Payload size | Small | Larger. Use Claim Check for blobs. [P: EIP] |
| Stale data risk | Callback reads *current* state, which may be newer than the event | Payload is state *as of the event*, which is consistent with ordering |
| Evolvability | Easy | Each field is a contract. Version carefully. |

Recommendation: in an event-only monolith, integration events default to **fat but purpose-designed** (summary events). Internal domain events can stay thin. [P: Dudycz "summary events"; Verraes "Event Summary"; Fowler]

### 2.4 Envelope metadata
Recommended envelope fields. [P: CloudEvents for the standard fields; correlation and causation from Greg Young / EIP Correlation Identifier; the rest is U, consensus]

| Field | Purpose |
|---|---|
| `event_id` (UUIDv7) | Uniqueness, idempotency key. UUIDv7 is time-ordered (RFC 9562, 2024) [P]. |
| `event_type` | e.g. `billing.invoice.issued` |
| `event_version` / `schema_version` | Integer or semver of the payload schema |
| `occurred_at` | Business time (UTC, RFC 3339) |
| `recorded_at` / `published_at` | Technical time |
| `source` / `producer_module` | e.g. `colsynq/billing` |
| `aggregate_type`, `aggregate_id` | Partition or ordering key |
| `aggregate_version` / `sequence` | Per-aggregate monotonic number, for ordering and stale-update detection |
| `tenant_id` | Multi-tenant routing, isolation and authorization in handlers |
| `correlation_id` | Ties together everything caused by one originating request |
| `causation_id` | `event_id` / `command_id` of the immediate cause |
| `actor` / `initiated_by` | Audit (avoid PII; use an id) |
| `traceparent` | W3C Trace Context, to link OpenTelemetry spans [P: W3C Trace Context Rec.] |

### 2.5 CloudEvents (CNCF, spec v1.0.2)
- Spec: https://github.com/cloudevents/spec (`cloudevents/spec.md`). [P]
- **Required** context attributes:
  - `id`: unique per source;
  - `source`: URI reference;
  - `specversion`: `"1.0"`;
  - `type`.
- **Optional:**
  - `datacontenttype`;
  - `dataschema`: URI of the schema, which is where you put versioning;
  - `subject`: e.g. the aggregate id;
  - `time`.
- **Extensions** (documented in the spec repo):
  - `partitionkey` (Partitioning extension);
  - `sequence` (Sequence extension);
  - `traceparent`/`tracestate` (Distributed Tracing extension);
  - `dataref` (Claim Check);
  - `authcontext`.

  There is no standard `tenantid` or `correlationid`, so add your own extension attributes. Extension names must be lowercase alphanumeric. [P; extension list from knowledge, U for 2026 changes]
- OpenTelemetry semantic conventions define `cloudevents.event_id`, `cloudevents.event_source`, `cloudevents.event_type`, `cloudevents.event_subject` and `cloudevents.event_spec_version` for spans. [P: opentelemetry.io semconv]
- Recommendation: adopt the CloudEvents attribute names in the envelope even in-process. Externalizing events later then costs very little. [U]

Example envelope (in-process, CloudEvents-shaped):
```json
{
  "specversion": "1.0",
  "id": "01928f3c-7a1e-7b3a-9f00-6f1c2d9e4a10",
  "source": "colsynq/billing",
  "type": "colsynq.billing.invoice_issued.v2",
  "subject": "invoice/7f3e...",
  "time": "2026-10-12T09:15:00Z",
  "datacontenttype": "application/json",
  "dataschema": "https://schemas.colsynq.internal/billing/invoice_issued/2.json",
  "partitionkey": "invoice/7f3e...",
  "sequence": "42",
  "tenantid": "t_123",
  "correlationid": "01928f3c-...",
  "causationid": "01928f3b-...",
  "traceparent": "00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01",
  "data": { "invoiceId": "7f3e...", "customerId": "c_9", "totalMinor": 125000, "currency": "NGN", "dueDate": "2026-11-11", "status": "issued" }
}
```

---

## 3. Local read models and projections, instead of synchronous queries

The core idea: every module that needs another module's data keeps its **own projection**, built from integration events and stored in its own schema or tables. The projection is shaped for the consumer's queries. [P: Richardson CQRS view; Fowler ECST; Vernon IDDD ch. 13; Dudycz]

### 3.1 Bootstrapping a new projection
Three established options:
1. **Replay from the producer's durable event log.** This option exists when the producer is event-sourced, or keeps an **append-only published-events table (outbox retained, not deleted)**. A new consumer starts from position 0. [P: Dudycz; Spring Modulith Event Publication Registry keeps completed publications optionally]
2. **Snapshot or backfill event.** The producer emits a one-off series of "state snapshot" events, such as `CustomerSnapshotted` (sometimes named "...Bootstrapped" or "...Synced"), for all current entities. The consumer then switches to live events, de-duplicating by `aggregate_version`. [U, common practice; related: Verraes "Completeness Guarantee"]
3. **Admin-time data migration.** A one-off migration copies data from the producer's tables into the consumer's projection. Allowed only as a **deployment-time** operation owned by the producer team, never at runtime. [U] This is the pragmatic option in a monolith that shares one database server.

Cut-over technique: record the global outbox position (or max `event_id` / time) **before** the snapshot. Apply the snapshot, then replay live events from that position. The handlers are idempotent and version-guarded, so overlap is harmless. [U]

### 3.2 Ordering
- Guarantee order **per aggregate (per partition key)**, not globally. [P: Kafka docs; Richardson; Dudycz "ordering, grouping and consistency"]
- In a Laravel/DB-backed monolith, the options are:
  - a single relay process per partition;
  - `SELECT ... FOR UPDATE SKIP LOCKED` with grouping by aggregate;
  - `ShouldBeUnique` / `WithoutOverlapping` keyed by aggregate id. [U for the Laravel specifics]
- Consumers should be **version-aware**. Store `last_applied_version` per aggregate in the projection. Ignore events with version ≤ stored. If there is a gap, park the event or retry later, or accept it for snapshot-style events where last-writer-wins by version is correct. [U, consensus; Dudycz 2025 on partial state]
- For snapshot-style events, out-of-order delivery is harmless when you apply "only if newer". For delta events, you need strict order or commutative updates. [P: Richardson saga countermeasure "commutative updates"]

### 3.3 Eventual consistency and UX strategies
[U unless marked; consensus patterns]
- **Read-your-own-writes in the originating module**: the module that took the command answers from its own write or read model.
- **Return the result of the command** (id plus status) in the response, so the UI shows it optimistically.
- **Optimistic UI** with a pending state ("Processing…"), plus push updates through WebSockets / Laravel Reverb broadcast when the projection catches up.
- **Version tokens**: the command returns `correlation_id` or the outbox position. The UI or API can poll "has projection X reached position N?", or wait briefly with a timeout. [P: this is the "consistency token" approach in EventStoreDB and Marten async daemon docs]
- **Design the business process around the delay**: "Your order is received" rather than "Your order is confirmed". [P: Hohpe "Coffee Shop"]
- **Avoid cross-module invariants in UX flows.** If an invariant truly spans modules, either the module boundaries are wrong, or the invariant needs a process manager with a reservation pattern. [P: Vernon IDDD; Richardson semantic lock]

### 3.4 Detecting stale projections
[U, consensus; Marten/Emmett and EventStoreDB expose these metrics as P]
- Store a **checkpoint** (last processed global position or event id and time) per projection.
- **Lag metric** = producer head position − projection checkpoint, plus `now − occurred_at` of the last applied event. Alert on thresholds.
- **Heartbeat events**: the producer emits a periodic tick (Verraes "Passage of Time"). If the projection's last tick is old, the projection is stale even when no business events are flowing.
- **Reconciliation jobs**: periodically compare counts or hashes per tenant between producer exports and consumer projections. Do this offline, or through a producer-published "daily summary" event, so that you don't need a sync query.
- Expose `as_of` / `projection_lag_ms` in API responses that are served from projections.

### 3.5 Rebuilding
[P: Dudycz projection rebuild posts; Marten docs; U for blue/green specifics]
- Projections must be **disposable**: rebuildable from the event log or from snapshot events.
- **Blue/green rebuild**: build into a new table (`projection_v2`) while v1 serves traffic. Catch up, swap atomically (rename or view), then drop v1.
- Make projection handlers **pure functions of (state, event)** with no side effects. Side effects such as emails belong in reactors or process managers, which must **not** re-run during a rebuild. Use a "replaying" flag, or keep reactors and projectors as separate subscriptions.
- Keep the retained published-event log long enough to rebuild, or keep a snapshot-event emitter available.

---

## 4. Process managers and sagas for multi-module workflows

### 4.1 Definitions
- **Saga** (Garcia-Molina & Salem, "Sagas", SIGMOD 1987) [P]: a long-lived transaction made of a sequence of local transactions, each with a compensating transaction.
- **Process Manager** (EIP) [P]: a central component that keeps state for a process and decides the next step based on incoming messages.
- Many authors (Dudycz, Vernon, Richardson) note that the terms are used loosely. [P] Practical distinction: a saga is a *pattern for consistency through compensation*, and a process manager is a *stateful router/coordinator*.

### 4.2 Choreography versus orchestration
| | Choreography | Orchestration (process manager) |
|---|---|---|
| Control | Each module reacts to others' events | One stateful coordinator decides the next step |
| Fit with event-only | Native | Fits if the orchestrator **lives inside one module** and reacts to events. It must not send cross-module commands. Instead it emits its own facts that other modules react to. |
| Visibility | Flow is implicit (Fowler's warning) | Explicit state machine |
| Recommended when | 2–3 steps, few participants | 4+ steps, compensations, timeouts, business-visible status |

[P: Richardson saga page; Fowler; U for the event-only adaptation]

Event-only adaptation [U]:
- The workflow owner is the module whose business process it is, for example `Onboarding`. It holds the process state.
- Other modules don't receive commands. They subscribe to the owner's public facts, for example `OnboardingRequested` or `WorkspaceProvisioningRequested`. (A "...Requested" fact is effectively a command in disguise; Dudycz warns about this. It is acceptable only if documented as such in the catalog. The purist alternative is to rethink the boundaries.)
- Each participant publishes its outcome facts (`WorkspaceProvisioned`, `WorkspaceProvisioningFailed`). The process manager consumes them.

### 4.3 Compensating actions
- Compensation is a **semantic undo**: a new business transaction such as `RefundIssued` or `ReservationReleased`, not a rollback. [P: Garcia-Molina; Hohpe "Coffee Shop"; Richardson]
- Order the steps as follows: compensatable steps first, then the **pivot transaction** (the point of no return), then retriable steps that must eventually succeed. [P: Richardson, *Microservices Patterns* ch. 4]
- Compensations must be idempotent and must handle "compensate before original arrived", for example by using tombstones. [U]

### 4.4 Timeouts
- Process managers schedule a **deadline**. In Laravel, use a delayed job or a scheduled `due_at` row checked by the scheduler.
- When the deadline is reached and the process isn't complete, emit `XTimedOut` and compensate. [P: EIP "Message Expiration"; Verraes "Passage of Time Event"; NServiceBus/MassTransit saga timeout docs]
- Store deadlines **in the process manager's state table**, not only in the queue. A queue purge must not lose them. [U]

### 4.5 Process manager state
- Persist the state, keyed by `correlation_id` or a business key. Use optimistic concurrency, such as a `version` column. [P: Vernon; MassTransit/NServiceBus docs]
- The process manager is a good use of an **explicit state machine**: an enum of states and a transition table. This makes the process manager testable as a pure function. [U]

---

## 5. Reliability

### 5.1 Transactional outbox
- Write the business change and the outgoing event to the **same DB transaction**. A relay (polling or CDC/log-tailing) publishes afterwards. This solves the **dual-write problem**. [P: Richardson]
- In a single-database monolith, the "relay" can simply be a queue worker that reads `outbox` rows with `FOR UPDATE SKIP LOCKED`. [U]
- Laravel specifics [U for 2026 status; check the docs]:
  - Laravel's `DB::afterCommit` / `ShouldDispatchAfterCommit` / `$afterCommit = true` on queued listeners delays dispatch until commit.
  - This is **not** an outbox. If the process crashes between commit and dispatch, the event is lost.
  - Use a real outbox table for integration events.
- Spring Modulith's Event Publication Registry is the canonical in-monolith outbox design. [P]

### 5.2 Inbox (idempotent consumer)
- On receive, insert `(consumer_name, event_id)` into an `inbox` / `processed_messages` table with a unique constraint, **in the same transaction** as the handler's state change. A duplicate causes a unique-violation, so the consumer acks and skips. [P: Richardson Idempotent Consumer; EIP Idempotent Receiver; Dudycz outbox/inbox article]
- The key must be **per consumer (handler)**, not per module, because one event may have several handlers in a module. [U, consensus]
- Alternative: **natural idempotency**, an upsert by aggregate id and version. This is often simpler for projections. [U]

### 5.3 At-least-once delivery and idempotency keys
- Assume at-least-once delivery. Exactly-once *delivery* is impossible over an unreliable network (Two Generals). Exactly-once *processing effect* = at-least-once delivery + idempotent handling. [P: Kafka EOS docs say this explicitly; Confluent "Exactly-once semantics" blog 2017; Dudycz]
- The idempotency key is `event_id` (the producer-assigned UUIDv7). For commands that come from HTTP, use the client's `Idempotency-Key` header (IETF draft "The Idempotency-Key HTTP Header Field") [P: draft-ietf-httpapi-idempotency-key-header; status in 2026 U].

### 5.4 Ordering per aggregate (partition key)
- Use the aggregate id as the partition key. All events for one aggregate go through one ordered lane. [P: Kafka partitioning; CloudEvents `partitionkey` extension]
- In a DB queue, process one aggregate at a time:
  - an advisory lock keyed by aggregate id;
  - or a `WithoutOverlapping($aggregateId)` middleware;
  - or a single worker per hashed lane.

  [U]
- Parallelism comes from many aggregates, not from one.

### 5.5 Poison messages and dead-letter queues
- A message that fails deterministically blocks its partition, or loops forever. After N attempts, move it to a **DLQ** together with the error, stack trace and attempt count. [P: EIP Dead Letter Channel / Invalid Message Channel]
- Separate the error types:
  - **transient** errors (DB deadlock, timeout): retry;
  - **permanent** errors (schema invalid, business rule): DLQ immediately.

  [U]
- Ordering caveat: dead-lettering one event for an aggregate means later events for that aggregate should **park** too. Otherwise they are applied out of order. Use a "parked aggregates" list. [U; NServiceBus/Kafka practitioner guidance]
- Provide **redrive tooling**: inspect, fix, replay from the DLQ (in Laravel: `queue:failed`, `queue:retry`, plus custom outbox/inbox admin). [U]

### 5.6 Retries with backoff
- Use exponential backoff **with jitter** to avoid thundering herds. [P: AWS Architecture Blog, "Exponential Backoff And Jitter", Marc Brooker 2015]
- Laravel: `$tries`, `backoff()` returning an array, `retryUntil()`, and the `ThrottlesExceptions` middleware. [P: Laravel queue docs, v13 — check with search-docs]
- Use bounded retries plus a DLQ. Make the retry policy part of each handler's configuration (OCP, §7.2).

### 5.7 Exactly-once illusions
[P: Kafka EOS docs; Tyler Treat "You Cannot Have Exactly-Once Delivery" (2015, bravenewgeek.com); Dudycz]
- "Exactly-once" in brokers means exactly-once *within the broker's transactional boundary*, for example Kafka read-process-write to Kafka. It does **not** cover side effects in your DB or email.
- Side effects outside your DB need their own idempotency: provider idempotency keys (Stripe, Paystack), or "send once" tables.
- Even in a single-process monolith, the outbox → queue → handler path can deliver twice, because a worker can crash after commit and before ack.

---

## 6. Event versioning and schema governance

### 6.1 Strategies
Greg Young, *Versioning in an Event Sourced System* (Leanpub, 2017) [P] is the canonical text. Dudycz's "Simple patterns for events schema versioning" [P] covers the same ground.
1. **Additive changes only** (weak schema): add optional fields. Never rename or remove them, and never change their meaning.
2. **New event type or version** for breaking changes (`InvoiceIssued.v2`). The producer **double-publishes** v1 and v2 during migration. Consumers migrate, and v1 is retired. Track consumers in the catalog.
3. **Upcasting**: transform old stored events into the current shape when they are read or replayed. This is essential for replaying the retained outbox or event log into new projections.
4. **Copy-and-transform**: rewrite the event store. This is a last resort.
5. **Wrong events stay wrong.** Fix them with **correcting events**, not by editing history. [P: Verraes "Correcting events"; Young]

### 6.2 Weak schema and tolerant reader
- **Tolerant Reader** [P: Fowler bliki 2011, https://martinfowler.com/bliki/TolerantReader.html]:
  - Consumers read only what they need.
  - They ignore unknown fields.
  - They apply sensible defaults for missing optional fields.
- **Postel's law** applied to messages. The weak schema in Young's book is the same idea for event stores. [P]
- In PHP: deserialize into consumer-owned DTOs (the ACL), never into producer classes. Don't use `strict unknown properties` on inbound events. [U]

### 6.3 Consumer-driven contract tests
- **Consumer-Driven Contracts** [P: Ian Robinson, martinfowler.com/articles/consumerDrivenContracts.html, 2006].
- **Pact** supports **message pacts** for async events: the consumer defines the expected message shape, and the producer verifies that it can produce it. [P: docs.pact.io; pact-php exists, P: github.com/pact-foundation/pact-php; its message-pact support in 2026 is U]
- In a monolith there's a lighter equivalent [U]:
  - Each consumer keeps a **fixture of the event payload it relies on**, plus a test that its projection accepts it.
  - The producer keeps a test asserting that its serializer output **validates against the published JSON Schema**.
  - CI runs both. Breaking a schema then fails the build.

### 6.4 Event catalogs and AsyncAPI
- **AsyncAPI 3.0.0** (released Nov 2023) is still the latest major version as of these sources. No 3.1 was confirmed in this session. [P: github.com/asyncapi/spec; asyncapi.com/blog/async-api-spec-3.0-release]
  - v3 separates **operations** from **channels** and supports reusable messages.
  - Payloads can be JSON Schema, Avro and others.
  - It is usable for in-process event contracts. Treat each module's public events as a channel set.
  - A Laravel package generates AsyncAPI 3.0 from PHP attributes: `drmmr763/laravel-asyncapi` on Packagist [S; maturity U].
- **EventCatalog** (https://eventcatalog.dev, by David Boyne) [P for existence; 2026 feature set U]:
  - documentation-as-code for domains, services, events, commands, queries, channels and flows;
  - schema versioning per message;
  - "producers/consumers" graphs;
  - generators from AsyncAPI/OpenAPI.

  It is a good fit for documenting which module publishes and which consumes each event, which directly mitigates Fowler's "lost flow" problem.
- **Schema registry** in a monolith [U]: store JSON Schemas in the repository under `contracts/<module>/<event>/<version>.json`. Validate them in tests, and optionally at publish time in non-production.

---

## 7. SOLID in DDD and hexagonal architecture (PHP 8.4/8.5 examples)

References:
- [P] Robert C. Martin, *Clean Architecture* (2017) and *Agile Software Development: Principles, Patterns, and Practices* (2002).
- [P] Alistair Cockburn, "Hexagonal Architecture" (2005, alistair.cockburn.us). Juan Manuel Garrido de Paz & Cockburn, *Hexagonal Architecture Explained* (2024).
- [P] Matthias Noback, *Advanced Web Application Architecture* (Leanpub, 2020). It is PHP-specific and the best PHP reference.
- [P] Carlos Buenosvinos, Christian Soronellas, Keyvan Akbary, *Domain-Driven Design in PHP* (2017).

### 7.1 SRP in handlers: one handler, one reason to change
```php
<?php
declare(strict_types=1);

namespace Modules\Membership\Application\Projections;

use Modules\Billing\Contracts\Events\InvoiceIssuedV2;          // producer's published language only
use Modules\Membership\Application\Ports\BillingStatusProjection;
use Modules\Membership\Application\Ports\Inbox;

/** Projects Billing invoice facts into Membership's local read model. Does nothing else. */
final readonly class ProjectInvoiceIssued
{
    public function __construct(
        private BillingStatusProjection $projection,
        private Inbox $inbox,
    ) {}

    public function __invoke(InvoiceIssuedV2 $event): void
    {
        $this->inbox->once(self::class, $event->eventId, function () use ($event): void {
            $this->projection->upsertIfNewer(
                customerId: $event->customerId,
                invoiceId: $event->invoiceId,
                outstandingMinor: $event->totalMinor,
                aggregateVersion: $event->aggregateVersion,
            );
        });
    }
}
```
Don't put "also send email" or "also call Billing" in this handler. Side effects go in a separate reactor or process manager subscribed to the same event. [U, consensus; Noback]

### 7.2 OCP via strategies and policies
```php
<?php
namespace Modules\Membership\Domain\Policy;

interface SuspensionPolicy
{
    public function shouldSuspend(MemberAccount $account, \DateTimeImmutable $now): bool;
}

final readonly class OverdueDaysPolicy implements SuspensionPolicy
{
    public function __construct(private int $graceDays) {}
    public function shouldSuspend(MemberAccount $account, \DateTimeImmutable $now): bool
    {
        return $account->oldestUnpaidDueDate()?->modify("+{$this->graceDays} days") < $now;
    }
}

/** Composite: new rules are added by new classes, not by editing existing ones. */
final readonly class AnyOf implements SuspensionPolicy
{
    /** @param list<SuspensionPolicy> $policies */
    public function __construct(private array $policies) {}
    public function shouldSuspend(MemberAccount $account, \DateTimeImmutable $now): bool
    {
        foreach ($this->policies as $policy) {
            if ($policy->shouldSuspend($account, $now)) {
                return true;
            }
        }
        return false;
    }
}
```
Other ways to apply OCP in event-driven code:
- Register event-to-handler mappings in a module service provider. Adding a consumer doesn't touch the producer.
- Use an **upcaster chain**. Adding a version adds an upcaster class.

### 7.3 LSP for repository implementations
Every implementation must honour the *same contract*, including exceptions and concurrency semantics, not just the method signatures.
```php
<?php
namespace Modules\Billing\Domain;

interface InvoiceRepository
{
    /** @throws InvoiceNotFound when absent (never returns null). */
    public function get(InvoiceId $id): Invoice;

    /**
     * Persists the aggregate AND its pending domain events atomically (outbox).
     * @throws ConcurrencyConflict if $invoice->version() is stale.
     */
    public function save(Invoice $invoice): void;
}
```
- `EloquentInvoiceRepository` and `InMemoryInvoiceRepository` (tests) must both throw `ConcurrencyConflict` on a stale version. Both must also record pending events.
- An in-memory fake that silently ignores versions violates LSP. Tests that pass against the fake would then lie.
- Run **one shared contract test suite** against both implementations (Pest dataset over implementations). [P: Noback; "contract tests for adapters" in *Hexagonal Architecture Explained*; U for the Pest specifics]

### 7.4 ISP for ports
Use small, role-specific ports rather than one fat gateway.
```php
<?php
namespace Modules\Membership\Application\Ports;

interface BillingStatusReader   { public function outstandingFor(CustomerId $id): Money; }       // query side of local projection
interface BillingStatusProjection { public function upsertIfNewer(string $customerId, string $invoiceId, int $outstandingMinor, int $aggregateVersion): void; }
interface IntegrationEventPublisher { public function publish(IntegrationEvent ...$events): void; } // writes to outbox
interface Clock { public function now(): \DateTimeImmutable; }
```
A use case that only reads billing status depends only on `BillingStatusReader`. It has no access to write methods. [P: Martin; Noback]

### 7.5 DIP via domain ports
- The domain and application layers define interfaces. Infrastructure (Eloquent, Redis, queue) implements them. Bindings live in the module's `ServiceProvider`.
- Dependencies point inward: `Infrastructure → Application → Domain`.
- **Cross-module DIP in an event-only system**: module A never depends on module B's ports or classes. It depends only on B's `Contracts\Events` namespace (data-only DTOs), and on the shared kernel (envelope, ids, Money). [P: Cockburn; Martin; Grzybek]

```php
<?php
// Modules/Billing/Infrastructure/BillingServiceProvider.php
$this->app->bind(InvoiceRepository::class, EloquentInvoiceRepository::class);
$this->app->bind(IntegrationEventPublisher::class, OutboxIntegrationEventPublisher::class);
```

### 7.6 Aggregate recording events (hexagonal core)
```php
<?php
namespace Modules\Billing\Domain;

final class Invoice
{
    /** @var list<DomainEvent> */
    private array $recordedEvents = [];

    private function __construct(private InvoiceId $id, private InvoiceStatus $status, private int $version) {}

    public static function issue(InvoiceId $id, CustomerId $customer, Money $total, \DateTimeImmutable $at): self
    {
        $invoice = new self($id, InvoiceStatus::Issued, 0);
        $invoice->record(new InvoiceWasIssued($id, $customer, $total, $at)); // internal domain event
        return $invoice;
    }

    public function pullRecordedEvents(): array { $e = $this->recordedEvents; $this->recordedEvents = []; return $e; }
    private function record(DomainEvent $e): void { $this->recordedEvents[] = $e; $this->version++; }
}
```
- An application-layer translator maps `InvoiceWasIssued` (internal) to `InvoiceIssuedV2` (public). It enriches the event with derived data **from the same transaction's state**, then writes it to the outbox. [P: Dudycz mapping handler]

---

## 8. Testing an event-driven modular monolith

### 8.1 Unit tests of aggregates emitting events (Given–When–Then)
[P: Greg Young / Event Sourcing testing style; Vernon; Dudycz "testing event-sourced aggregates"]
```php
it('records InvoiceWasIssued when issued', function () {
    $invoice = Invoice::issue(InvoiceId::new(), CustomerId::fromString('c_9'), Money::ngn(125000), new DateTimeImmutable('2026-10-12'));

    expect($invoice->pullRecordedEvents())
        ->toHaveCount(1)
        ->and(fn ($events) => $events[0])->toBeInstanceOf(InvoiceWasIssued::class);
});
```
- Assert on **events**, not on internal state. This keeps tests stable across refactors.

### 8.2 Contract tests of event schemas
- **Producer side**: build each public event from a factory. Serialize it with the real serializer and validate it against `contracts/billing/invoice_issued/2.json` (JSON Schema). Use a snapshot test to detect accidental changes. [U; P for JSON Schema itself, json-schema.org 2020-12]
- **Consumer side**: each consumer stores the *example payloads it depends on*, including old versions. It asserts that its deserializer, upcasters and projection accept them. [P: Robinson CDC; Pact message pacts]
- **Compatibility gate**: CI fails if a schema changes without a version bump, or if a field is removed or renamed in the same version. Tools: `json-schema-diff`, AsyncAPI diff (`@asyncapi/diff`). [S/U]

### 8.3 Projection tests
- Given a sequence of events (including duplicates and out-of-order versions), when the projection runs, then the read model equals X. [U, consensus; Marten/Emmett docs P]
- Test these cases explicitly:
  - idempotency: the same event twice;
  - stale versions: v3 then v2;
  - an unknown extra field (tolerant reader);
  - rebuild-from-scratch equals the incremental result.

### 8.4 Process manager tests
- Treat the process manager as a state machine: `(state, event) → (newState, outgoing events, scheduled deadlines)`. Test it as a pure function with a **fake clock**. [U; P: Spring Modulith `Scenario` API, MassTransit saga test harness are analogous]
- Cover the following scenarios:
  - happy path;
  - each failure, triggering compensation;
  - timeout firing;
  - timeout arriving *after* completion (must be ignored);
  - duplicate events.
- **Integration test** in Laravel: `Event::fake()` / `Queue::fake()` to assert publishing, plus a "run the outbox relay synchronously" helper for end-to-end flows across modules within one test. [U; check Laravel 13 testing docs via search-docs]

### 8.5 Architecture tests
- **Pest `arch()`** (Pest 3/4) [P: pestphp.com/docs/arch-testing]:
```php
arch('modules talk only via contracts')
    ->expect('Modules\Membership')
    ->not->toUse(['Modules\Billing\Domain', 'Modules\Billing\Application', 'Modules\Billing\Infrastructure']);

arch('domain is framework-free')
    ->expect('Modules\*\Domain')    // verify wildcard support in your Pest version
    ->not->toUse(['Illuminate', 'Laravel']);

arch('contracts are data only')
    ->expect('Modules\Billing\Contracts\Events')
    ->toBeFinal()->toBeReadonly();
```
- **Deptrac** (https://github.com/deptrac/deptrac) [P]: a layer and module dependency rule set (YAML). Better for whole-graph rules. It is widely used in PHP modular monoliths.
- **PHPStan/Larastan** custom rules can forbid, for example, `DB::table('billing_*')` outside the Billing module. [U]
- Equivalents in other ecosystems [P]:
  - NetArchTest (Grzybek);
  - ArchUnit and Spring Modulith `ApplicationModules.verify()`;
  - import-linter (Python);
  - Packwerk (Ruby).

---

## 9. Pitfalls and trade-offs of event-only communication

### 9.1 Eventual consistency
- Cross-module invariants become eventual. Sometimes a check is needed *at decision time*, such as "is the customer's credit OK?". Then the deciding module must hold a local projection, and accept a staleness window, **or** the boundary is wrong. [P: Vernon "aggregate design rules"; Fowler ECST; Richardson]
- Racing: a consumer may receive a fact about an entity it has not seen yet. Design projections to accept **partial state** (Dudycz 2025) [P], or to park the event until its prerequisite arrives.

### 9.2 Duplication of data
- Duplication is a deliberate trade for autonomy. [P: Fowler ECST; Vernon ch. 13]
- Costs:
  - storage;
  - GDPR erasure must propagate, through `CustomerErased` events and Forgettable Payloads (Verraes [P]);
  - schema coupling on each duplicated field.
- Mitigate the costs by:
  - projecting **only** what the consumer needs;
  - owning the meaning of the copy in the consumer's own language (ACL);
  - never editing a copy locally. A copy is read-only, and only the owner's events change it.

### 9.3 Debugging and tracing
- Fowler's "losing sight of the larger-scale flow" is the main operational cost. [P]
- **OpenTelemetry**:
  - propagate W3C `traceparent` in the envelope;
  - create producer and consumer spans following the **OTel Messaging semantic conventions** (`messaging.system`, `messaging.operation.type` = publish/receive/process, `messaging.message.id`);
  - use **span links** (not parent-child) for async hops when a batch or fan-out breaks strict parent-child structure.

  [P: opentelemetry.io/docs/specs/semconv/messaging/; messaging conventions reached stable status around 2024–2025, U for exact status in Oct 2026]
- For the PHP OTel SDK and auto-instrumentation (`open-telemetry/opentelemetry-auto-laravel`) [P for existence; queue span maturity in 2026 U]:
  - Propagate context manually into the outbox row and restore it in the worker.
  - Laravel's queue context (`Context` facade, Laravel 11+) can carry `correlation_id` across dispatches. [P: Laravel Context docs; check v13 with search-docs]
- Correlation and causation ids in logs give a cheap "event lineage" view without full tracing. Log one structured line per publish and handle: `event_id`, `type`, `correlation_id`, `causation_id`, `handler` and `duration`. [U]
- Keep an **EventCatalog / AsyncAPI** map of producers and consumers. It is the static complement to dynamic traces. [P]

### 9.4 When a synchronous call is still justified, and how purists avoid it
Cases where practitioners commonly accept a sync call [U, synthesis; Grzybek's "Integration Styles" P for the general trade-off]:
1. **Strong read-after-write UX** on another module's data, immediately after a user action.
2. **Validation needing authoritative current state** at decision time, such as unique usernames across modules or seat-limit checks.
3. **Low-value lookups** where building a projection costs more than the coupling.
4. **Bootstrap or admin tooling.**

How purists avoid module-to-module sync calls:
- **API composition at the edge** [P: Richardson API Composition; also "Backend for Frontend", Sam Newman, samnewman.io/patterns/architectural/bff]:
  - The HTTP controller, GraphQL resolver or BFF calls **each module's own public read API** and merges the results.
  - Modules never call each other. Only the outermost layer, which is not a module, composes them.
  - This keeps module-to-module dependencies event-only while the UI still gets a consistent page.
- **UI composition**: the frontend fetches separate widgets from separate module endpoints. This is Spotify's and IKEA's "micro-frontends" style (P: Cam Jackson, "Micro Frontends", martinfowler.com 2019).
- **Move the invariant**: re-draw the boundary so that the invariant lives inside one aggregate or module. [P: Vernon "Effective Aggregate Design" (2011, dddcommunity.org)]
- **Reservation pattern**: the deciding module publishes `SeatReservationRequested`. The owner replies with `SeatReserved` or `SeatReservationRejected`. A process manager with a timeout handles the rest. [P: Richardson semantic lock; Vernon]
- **Local projection plus compensation**: decide optimistically on the local projection, and compensate if the owner later rejects. [P: Hohpe "Coffee Shop"]

### 9.5 Other pitfalls
[U unless marked]
- **Event soup / CRUD events** ("CustomerUpdated" with everything): consumers can't tell *why* something changed. Prefer intention-revealing events, plus one public snapshot event where replication is truly the need. [P: Verraes, Dudycz "clickbait event", "state obsession"]
- **Passive-aggressive events**: events that expect a specific reaction, which are really commands. [P: Fowler/Verraes talk "The Many Meanings of Event-Driven Architecture", Fowler GOTO 2017; Dudycz]
- **Shared database tables "just for reading"**: this breaks the boundaries silently. Enforce with Deptrac, arch tests and DB schemas or prefixes per module. [P: Grzybek; Shopify Packwerk]
- **Synchronous in-process listeners treated as "events"**: Laravel's default `Event::dispatch` runs listeners in the same transaction. A listener exception then rolls back the producer, which is hidden coupling. For cross-module events, always go through the outbox and an async handler. [U; Spring Modulith makes the same distinction between `@EventListener` and `@ApplicationModuleListener` (async, after-commit) — P]
- **Over-modularising early**: use few, coarse modules and split later. [P: Fowler MonolithFirst; Shopify 2022 "Under Deconstruction"]
- **Projection explosion**: each consumer copying a large share of every other module is a sign that the boundaries are wrong, or that the edge needs API composition instead. [U]

---

## 10. Minimal recommended blueprint (Laravel 13, single DB)

[U, synthesis of the sources above]
1. `Modules/<Name>/{Domain,Application,Infrastructure,Contracts/Events,Http}`, with a dedicated table prefix or schema per module.
2. Aggregates record domain events. The repository `save()` persists the aggregate and writes **mapped integration events** to `<module>_outbox` in one transaction.
3. An outbox relay (a scheduled command or long-running worker) publishes to the Laravel queue. It uses an ordered lane per `partition_key` and marks rows as published, keeping them for replay with a retention period.
4. Each consumer handler is queued and wraps its work in `inbox->once(handler, event_id)` plus a version guard. Use a per-handler retry/backoff policy, then send to a DLQ (failed_jobs plus a parked-aggregate table).
5. Each consumer keeps projections in its own tables. Projections have a checkpoint and lag metrics, and support blue/green rebuild.
6. Process managers live in the module that owns the workflow. They have a state table, deadlines and compensations.
7. The envelope is CloudEvents-shaped and carries tenant, correlation, causation and traceparent.
8. Governance:
   - JSON Schema per public event version;
   - AsyncAPI 3.0 per module;
   - EventCatalog site;
   - CI contract checks;
   - Pest `arch()` and Deptrac boundary rules.
9. Cross-module reads for the UI happen through **API composition in the HTTP layer**, never module to module.

---

## Source list (primary unless marked)
- Grzybek, Modular Monolith with DDD — https://github.com/kgrzybek/modular-monolith-with-ddd ; blog https://www.kamilgrzybek.com
- Evans, *Domain-Driven Design* (2003); *DDD Reference* (2015) — https://www.domainlanguage.com/ddd/reference/
- Vernon, *Implementing DDD* (2013); *Reactive Messaging Patterns* (2015); "Effective Aggregate Design" (2011)
- Fowler, "What do you mean by Event-Driven?" — https://martinfowler.com/articles/201701-event-driven.html ; TolerantReader; MonolithFirst; CQRS
- Robinson, "Consumer-Driven Contracts" — https://martinfowler.com/articles/consumerDrivenContracts.html
- Richardson — https://microservices.io/patterns/ ; *Microservices Patterns* (2018)
- Hohpe & Woolf, *Enterprise Integration Patterns* — https://www.enterpriseintegrationpatterns.com ; "Your Coffee Shop Doesn't Use Two-Phase Commit"
- Dudycz — https://event-driven.io/en/internal_external_events/ ; https://event-driven.io/en/dealing_with_race_conditions_in_eda_using_read_models/
- Verraes, Messaging Patterns — https://verraes.net
- Young, *Versioning in an Event Sourced System* (2017)
- Garcia-Molina & Salem, "Sagas" (SIGMOD 1987)
- CloudEvents — https://github.com/cloudevents/spec ; OTel CloudEvents and Messaging semconv — https://opentelemetry.io/docs/specs/semconv/
- AsyncAPI 3.0 — https://www.asyncapi.com/blog/async-api-spec-3.0-release ; https://github.com/asyncapi/spec
- EventCatalog — https://eventcatalog.dev
- Spring Modulith — https://spring.io/projects/spring-modulith
- Shopify — https://shopify.engineering/deconstructing-monolith-designing-software-maximizes-developer-productivity ; https://shopify.engineering/enforcing-modularity-rails-apps-packwerk ; https://shopify.engineering/shopify-monolith
- Brooker, "Exponential Backoff and Jitter" (AWS Architecture Blog, 2015)
- Noback, *Advanced Web Application Architecture* (2020); Cockburn & Garrido de Paz, *Hexagonal Architecture Explained* (2024)
- Pest arch testing — https://pestphp.com/docs/arch-testing ; Deptrac — https://github.com/deptrac/deptrac
- [S] Milan Jovanović — https://milanjovanovic.tech/blog/event-driven-communication-modules ; ABP outbox/inbox article — https://abp.io/community/articles/using-outboxinbox-pattern-for-reliable-event-handling-in-a-multimodule-monolithic-application-eurs9own ; InfoQ on Grzybek — https://www.infoq.com/news/2019/09/design-ddd-modular-monolith/
