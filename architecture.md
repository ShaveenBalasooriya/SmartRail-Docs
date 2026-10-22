# SmartRail — Architecture

This document records the architecture of SmartRail's backend-first build and the reasons for each
choice. It exists so that each decision can be explained and defended on its own, as
`final-assessment-brief.md` §24 and §26 ask: justified against the problem the platform has to solve,
not just named. The decisions are the owner's. Where I (the assistant) argued for a different option
and the owner chose otherwise, this document says so. Anything inferred rather than decided outright is
marked **[A]**.

**Status:** agreed by the owner, living document, written before any code exists. Revised on 2026-10-19
for the lean MVP 1 cut in `docs/scoping/mvp-1-scope.md`, and on 2026-10-21 for event delivery (§2, §3,
§6, §9, §10) and the Shared Kernel project list. Decisions are also recorded turn by turn in `docs/ai-usage/AI-INTERACTION-LOG.md` (rows 87
onward). The data model that follows from this architecture is in `docs/data-model.md`.

---

## 1. Why these choices as a whole

SmartRail is one person building a platform with real breadth: a backend, an operations web app, a
passenger mobile app, a crew mobile app, a station display and an external catering integration, all
within a fixed amount of time (brief §25). Each choice below is weighed against two things at once.
Does it keep the system coherent and genuinely production-minded (brief §19, §26)? And does it avoid
spending scarce time on ceremony that a solo build doesn't need yet (brief §25)? Where those pulled in
different directions, this document says so.

MVP 1 was later cut down to a lean core. The network and its timetable are seed data. The only staff
roles are the Network Controller, the Guard and the Auditor, and there are no passenger accounts. The
architecture was revised with that cut, so that it carries no structure MVP 1 has no use for, while
keeping a clear way to add the deferred parts back.

---

## 2. Backend architecture: Modular Monolith

**Decision:** one ASP.NET Core backend process, divided internally into modules with hard boundaries,
rather than a microservices architecture.

**Why not microservices:** microservices earn their cost through independent scaling, independent
deployment and separate teams, and none of those apply to a one-person build in a fixed timeframe.
They would mean spending the project's time on service discovery, distributed transactions and
network failure handling instead of on the railway domain. That is the over-complication brief §25
warns against: "a smaller, coherent, well-engineered solution is preferable to a large collection of
incomplete features".

**Why not one undivided application either:** SmartRail's requirements already have clear seams.
The network and timetable, live running, staff identity, what is delivered to the outside world,
catering and audit are distinct concerns, with different data and (per `moscow-scope.md` M-04) a hard
rule that no role may reach across them. A modular monolith enforces those seams inside one deployable
unit, and leaves open the option of splitting a module into its own service later, without making
that a requirement now.

**Internal structure, per module: Clean Architecture with a CQRS pipeline.** Each module is four
projects under `src/Modules/<Module>/`, plus its public contracts in `src/Contracts/`:

| Project | Contains | References |
|---|---|---|
| **Domain** | Entities (aggregates) with behaviour, value objects, domain errors, domain services | Shared Kernel only |
| **Application** | Use cases as vertical slices, the ports the module needs (`Abstractions/`), the implementations of the module's public contracts | Domain, Shared Kernel Application, its own and other modules' Contracts |
| **Infrastructure** | The `DbContext` (which is the module's unit of work), EF configurations, migrations, repositories, background jobs, external HTTP clients | Application |
| **Api** | Carter endpoint modules per feature and their request DTOs | Application, Shared Kernel Api |
| **Contracts** (`src/Contracts/SmartRail.<Module>.Contracts`) | The module's public surface: integration events (`INotification`), query contract interfaces and their DTOs | Shared Kernel, MediatR contracts |

Every arrow points inward. Infrastructure depends on Application, not the reverse, so it implements
the interfaces Application declares. The Host references each module's Api and Infrastructure only to
call their `Add<Module>…()` extensions. Domain and Application never reference EF Core, ASP.NET Core
or Authentik.

**Domain.** Entities derive from `Entity<TId>`, have private setters and a private constructor for
EF Core, and are created through a `static Create(...)` that validates and returns `Result<T>`.
Behaviour that changes an entity is a method on it, so its invariants are enforced in one place.
Concepts with rules of their own (a station code, a platform number) are value objects with a
validating `Create`. Each aggregate has a static `<Entity>Errors` class of named `Error`s. A domain
calculation that is not one entity's behaviour (the live-state calculator) is a pure domain service in
the Domain project.

**Application.** One folder per use case, `<Feature>/<UseCase>/`, holding the command or query, its
handler and its validator. The feature root holds the feature's repository interface and response DTOs.
Commands and queries implement `ICommand`, `ICommand<T>` or `IQuery<T>` and return `Result` or
`Result<T>`. Handlers are `internal sealed`; requests and responses are public. `Abstractions/` holds the
module's `IUnitOfWork` and any other port (a clock, a fact reader). The module's public contract
interfaces are implemented here by classes that send the module's own queries, so another module's
call and an HTTP request run through the same handler. Handlers for other modules' integration events
are slices too. Each Application project exposes `Add<Module>Application()`.

**Pipeline.** Cross-cutting concerns are MediatR behaviours in the Shared Kernel Application project,
never code inside handlers: request → `LoggingPipelineBehavior` → `ValidationPipelineBehavior` →
handler. Validation runs the request's FluentValidation validators and short-circuits with a
`ValidationResult` (a failed `Result` carrying every error) without throwing. Logging records the
request name, outcome and duration, never the request body.

**Infrastructure.** The module's `DbContext` implements its `IUnitOfWork`. `Configurations/` holds one
`IEntityTypeConfiguration` per entity and maps the Domain entities directly; there are no separate
persistence models. `Repositories/` implements the Application repository interfaces. SQL that EF Core
cannot express (`ON CONFLICT DO NOTHING`, `SELECT … FOR UPDATE`) lives only inside a repository method.
A background job is a thin `BackgroundService` that sends a command; the work is in the handler. Each
Infrastructure project exposes `Add<Module>Infrastructure(configuration)`.

**Api.** Endpoints are Carter modules (`ICarterModule`), one per feature, using Minimal API route
groups. An endpoint maps its request DTO to a command or query, calls `sender.Send(...)` and returns
`result.IsSuccess ? TypedResults.X(...) : result.ToProblemDetails()`. `ToProblemDetails` (Shared Kernel
Api) maps `ErrorType` to the HTTP status and RFC 7807 body. Endpoints hold no business rules.

**Errors.** Expected failures (not found, validation, conflict, forbidden) travel as `Result` values
and become problem details at the edge. Exceptions are only for bugs and infrastructure failures, and
the Host's global exception handler turns them into a 500.

**Request flow.**

```text
HTTP → Host middleware (correlation ID, auth, policy)
     → Carter endpoint → ISender
     → Logging → Validation → handler
          handler: value objects → entity behaviour → repository
                   → cross-module event: IIntegrationEventPublisher adds an outbox row
                   → IUnitOfWork.SaveChangesAsync (change and outbox row commit together)
                   → the event bus delivers the event to consumers after the commit (§3)
     ← Result<T> → TypedResults.X or ToProblemDetails()
```

**Why four projects and not three:** the first design merged Infrastructure and endpoints to save
projects. In practice that put use-case logic, persistence models and HTTP in one project, left the
Application layer nearly empty and made the dependency rule unenforceable. The owner chose the
standard split so each layer has one job, and the same layout is repeated in every module (owner
decision, 2026-10-20).

**Endpoints, not controllers:** each module maps its own Carter modules inside its own Api project.
There is no shared `Controllers/` folder for every module to reach into, which would be the first
crack in a module boundary.

---

## 3. Module boundaries

Six modules, drawn from the epics in `docs/scoping/mvp-1-scope.md`. Each module that keeps data owns
its own database schema (§4). Other modules reach it only through its published contracts, never by
querying its tables.

| Module | Responsibility | Epics it covers | Schema |
|---|---|---|---|
| **Identity & Access** | Authentik integration for staff identity, token validation, the role catalogue (Network Controller, Guard, Auditor), role-based access enforcement, the catering provider's API key | EP-02 | `identity` |
| **Rail Network** | The seeded network (lines, stations, platforms, station order) and the seeded timetable (services, stops, times, platforms, the assigned guard), and generating each day's operational journeys from it | EP-04 | `network` |
| **Live Operations** | The Guard's departure logs and their idempotent sync; each journey's live state (last and next station, expected times, minutes early or late, lifecycle status, freshness); the Network Controller's delay, cancellation, platform change and correction; the ordered journey event stream | EP-06, EP-07, EP-08 | `operations` |
| **Distribution** | Everything delivered to the outside world: the public read API for the passenger app and the station display, and the live SSE streams for the passenger app, the display and the Guard's app, including catch-up after a reconnect | EP-09, EP-10, EP-11 | none |
| **Catering Integration** | The provider's narrow read API, webhook subscriptions, and webhook delivery with sequence numbers, retry and duplicate handling | EP-12 | `catering` |
| **Audit** | The append-only, hash-chained record of every staff state change and every denied cross-role attempt; the Auditor's read-only view | EP-03 | `audit` |

**Why Passenger is not a module in MVP 1:** the original design had a separate Passenger module for
passenger accounts, saved journeys and the journey planner. All three are deferred in the lean cut, so
the passenger app needs only a public read API, which Distribution now serves. Passenger becomes its
own module again when accounts return in a later MVP. The owner chose this merge over keeping an empty
module in place.

Two cross-cutting pieces are not on the list, because they are not bounded contexts. They are shared
infrastructure that every module uses the same way:

- **Shared Kernel** is three small projects with no business logic, plus a fourth, `SmartRail.SharedKernel.Infrastructure`, that holds the event bus **[A]**. `SmartRail.SharedKernel` holds the
  base entity and value-object types, the domain-event base type, `Result`, `Result<T>`, `Error`,
  `ErrorType`, `ValidationResult` and pagination helpers, and depends on nothing.
  `SmartRail.SharedKernel.Application` holds the CQRS interfaces (`ICommand`, `IQuery`, their handlers)
  and the logging and validation pipeline behaviours. `SmartRail.SharedKernel.Api` holds
  `ToProblemDetails`. `SmartRail.SharedKernel.Infrastructure` holds the outbox mapping, the publisher, the
  dispatcher and the in-process transport of the event bus (§3), because they need EF Core and Npgsql, which
  the Application projects may not reference. Every module may depend on them.
- **Host** is the ASP.NET Core entry point (`Program.cs`) and the composition root. It calls every
  module's `Add<Module>Application`, `Add<Module>Infrastructure` and `Add<Module>Api`, registers MediatR
  and the pipeline behaviours over every module's Application assembly, maps the Carter modules under
  `/api/v1`, and wires the middleware pipeline (correlation ID, authentication, authorisation, the
  global exception handler, OpenAPI and Scalar, health checks, migrations at start-up). It is the only
  project that knows all six modules exist.

**Why Audit is a module and not just a logging call:** it owns its own schema and its own invariants:
hash-chaining, append-only storage, and no update or delete path for any role (brief §23,
`moscow-scope.md` M-03). By the rule used for every other module, that makes it a module. The
difference is direction: every other module depends on Audit's port to record its own state changes,
and Audit depends on none of them.

**How modules talk to each other:** never by one module querying another module's tables, which is
what schema-per-module (§4) makes impossible rather than just discouraged. There are three patterns, and
the choice depends on what the interaction is; the event bus is settled in detail at the end of this
section:

- **"Something happened" inside a module** (a handler reacting to a command's outcome, a lightweight
  side effect) is published as an in-process event through **MediatR**. Interested handlers register as
  `INotificationHandler` and subscribe independently, and the code that raised the event never knows who is
  listening. MediatR was chosen over a hand-rolled dispatcher because it already implements this pattern,
  it is widely understood (so a reviewer or subagent isn't learning a bespoke mechanism), and it costs one
  small library rather than the time to write and test the same thing.
- **"Something happened" for other modules** (a departure was logged, a delay was published, a change was
  corrected, a journey was generated, access was denied) is an integration event on the **event bus**: the
  publishing module saves an outbox row in its own schema in the same transaction as the change, and a
  dispatcher delivers it at least once to each module's consumer, in order per ordering key. This is the one
  mechanism for every cross-module event, so nothing crosses a boundary by a different route. See "Event
  delivery: a generalised event bus on Postgres now, NATS JetStream later" below. The bus keeps a future
  split open: the in-process transport becomes a NATS JetStream transport without changing the modules that
  publish or consume.
- **"What is the current state of X"**, asked by a module that doesn't own X, is answered by a call to
  the owning module's **in-process query contract**: an interface in that module's Contracts project,
  implemented in its Application layer by sending the module's own query, and registered in the DI
  container. For example, Distribution asks Live Operations for a journey's live
  state and Rail Network for station names. This is still not a cross-schema join, because the owning
  module runs its own query on its own schema and returns a result object.

**Why contracts and not read models:** the first design had Distribution and Catering keep their own
copies of live journey data, updated by events. Each copy was a set of tables plus event handlers to
keep them in sync. For MVP 1's scale (two lines, nine stations, one day's journeys) an in-process call
is effectively free, and the copies would have been the largest single piece of plumbing in the build
without fixing any MVP 1 problem. The owner chose the leaner option. If a query path later becomes slow,
a read model can be added behind the same contract without changing its callers.

### Event delivery: a generalised event bus on Postgres now, NATS JetStream later

**Decision (owner, 2026-10-21):** MVP 1 runs no message broker, but one mechanism, an event bus on
PostgreSQL, carries every event that crosses a module boundary. NATS JetStream is the broker the owner
expects to adopt later, as a swap of the bus's transport. The owner first asked for the persisted journey
event stream alone to be the bus ("Postgres only for now, given the time constraints"), then chose the
generalised bus because it solves the major issue that events were being published in different ways
across the system. `operations.journey_events` stays as the query and catch-up store (the live-state
contract, `Last-Event-ID` resume, per-journey sequence numbers); the bus does not replace it.

**The problem it settles:** events reached their consumers in three different ways, and the specs assumed
three different answers to "when does an event fire". `JourneyGenerated` (Rail Network) was published
in-process right after save, `JourneyStateChanged` (Live Operations) after the transaction commits, and
`AccessDenied` (Identity) in-process from the authorization handler. `add-audit-record` wrote its entry in
a transaction shared by every module's `DbContext`, and `add-catering-integration` required a handler to
run *inside* the transaction, which an event published after commit cannot be. None of the three survived
a crash between the change and the reaction.

**How it works [A].** A module publishes an integration event through `IIntegrationEventPublisher`, which
writes an outbox row in that module's own schema (§4: no cross-schema write) in the same transaction as
the change, so a rollback publishes nothing. A generic dispatcher reads each module's unpublished rows in
order and hands each to every registered consumer, through an `IEventTransport`; MVP 1's transport is
in-process on PostgreSQL. After the commit the publisher wakes the dispatcher in-process, and a one-second
poll is the fallback. Delivery is at least once. Each consumer implements `IEventConsumer<TEvent>`, has a
stable name and its own delivery state, is idempotent by event ID, retries with backoff and after five
attempts marks the delivery `failed` and counts it. Order is promised per ordering key (the journey ID for
journey events), not globally. MediatR stays for commands, queries and reactions inside one module. The
detail is `add-event-bus`.

**Audit keeps one exception.** An audit entry for a staff state change commits in the change's own
transaction, through the shared request transaction (`IRequestTransaction`, named so it does not clash with
each module's `IUnitOfWork` **[A]**). The owner decided this on 2026-10-21 (option X) and acknowledged the
small coupling it creates between the modules' persistence, accepting it for the sake of simplicity: AUD-05
needs the entry and the change to stand or fall together. The change's outbox row commits in that same
transaction, so no consumer is woken for work that was rolled back. Access denials do go through the bus
(Identity's outbox, an Audit consumer writes the entry), so a denial is not lost if the audit write fails;
it appears a moment after the denial **[A]**.

**Planned: audit through the bus (option Y).** The owner's stated plan is to send staff state changes
through the bus too: the change commits with a durable to-audit message in its own module's outbox, and an
Audit consumer writes the chain entry shortly after. The shared transaction would go, the entry would
appear a moment after the change, and AUD-05 and the audit requirement that a change and its entry commit
together would need rewording. It is not built now. To keep it possible, staff state changes record through
the `IAuditRecorder` contract only, so only the recorder's implementation changes.

**Catering and Distribution are consumers.** Catering inserts its `webhook_deliveries` rows from a
consumer of `JourneyStateChanged`, replacing the old handler inside the unit of work; the unique
`(subscription_id, event_id)` index makes redelivery harmless. Distribution's stream hub is a consumer of
the same message, and a client that resumes with `Last-Event-ID` reads `journey_events` through the
live-state contract (§6). Consumers never query `operations` directly, so §4's rule holds.

**Why a bus and not a broker now:**

- **Time.** The owner's stated reason: with the given time constraints, no broker is the best choice right
  now, and it makes sense.
- **A broker would not remove the outbox.** Writing to the database and then publishing to a broker is
  two writes. If the process dies between them the change is lost, so a relay reading the same table is
  still needed. A broker adds code; it does not replace the outbox.
- **It adds moving parts MVP 1 does not need.** A container, a client library, a second failure mode and
  a CI service for the end-to-end flows, for one API instance and a handful of consumers (brief §25:
  avoid ceremony a solo build does not need yet).
- **The bus abstraction makes the broker a transport swap.** Publishers and consumers see only
  `IIntegrationEventPublisher` and `IEventConsumer<TEvent>`, so adopting NATS replaces one implementation.
  What the bus implements by hand (delivery state, retry, redelivery) is small at this scale.

**Why NATS JetStream is the later choice:**

- Durable consumers with acknowledgement, redelivery and replay, which is what the bus's per-consumer
  delivery state implements by hand.
- Publish de-duplication on a `Nats-Msg-Id` header, set to the event ID, which makes a relay's retries
  safe. The default window is 2 minutes, so consumers must still de-duplicate by event ID.
- Apache 2.0 licence, an official .NET client (NATS.Net), and one small container.
- It can be the backplane if the API ever runs more than one instance; the bus dispatcher and
  Distribution's SSE hub are in-memory and single-instance today.
- It keeps the door the MediatR paragraph above leaves open: an in-process event becomes a bus event
  without changing the modules that publish or handle it.

**Rejected alternatives:**

| Option | Why not now |
|---|---|
| RabbitMQ | Mature and good for work queues, but not a replayable log by default; RabbitMQ Streams adds that back |
| Redis or Valkey Streams | Persistence and consumer groups, but Redis's licence changed (RSALv2/SSPL, later AGPL for Redis 8) and Valkey is the BSD fork **[A]** |
| Kafka, Apache Pulsar, Redpanda | Durable and replayable, but heavier to operate than MVP 1 needs; Redpanda's core is source-available (BSL), not fully open source **[A]** |
| MassTransit v9 | Brings an outbox, but is commercially licensed (its NuGet pages state v9 must be licensed; older versions stay open source). SmartRail uses MediatR under a commercial licence only because this is a test project (owner decision, 2026-10-19), so a second commercial dependency is avoided |
| Wolverine, Rebus | Not evaluated |

The MassTransit v9 licence position and NATS JetStream's two-minute default de-duplication window were
checked against their official pages when this decision was taken. The Redis, Kafka, Pulsar and
Redpanda licence details come from the owner's own comparison table and were not re-checked **[A]**.

---

## 4. Data layer: PostgreSQL, one schema per module

**Decision:** one PostgreSQL database, with one schema for each module that keeps data, and one EF Core
`DbContext` per module scoped to its own schema. No foreign key crosses a schema boundary. A reference
to something another module owns is stored as a plain identifier and resolved through that module's
contract when needed, never through a join.

**Why not one shared schema:** a shared schema with module names on folders is not actually modular.
Nothing stops a later change, whether made by a subagent or under time pressure, from joining two
modules' tables. By the time anyone notices, the boundary is cosmetic. Separate schemas make the
boundary something the database enforces rather than a code-review convention, and they let each
module be understood, migrated and tested on its own.

**Why PostgreSQL:** it is mature and free, runs the same way in Docker locally as in most production
targets, has first-class EF Core support, and supports schemas natively, which this design depends on.

**Cost accepted knowingly:** a read that would have been one join becomes a call through another
module's contract (§3). That is more design work than one shared schema. It is accepted because a
shared schema gives up the one mechanism that makes the monolith actually modular, in a system where
separation of duties (brief §14, `moscow-scope.md` M-04) is a correctness requirement.

---

## 5. Identity & Access: Authentik

**Decision:** Authentik, an open-source OIDC/OAuth2 identity provider, is SmartRail's single identity
provider. Staff and, in a later MVP, passengers each sign in through their own OAuth2 client on the one
Authentik instance. SmartRail never stores a login credential. Role-based access control lives in
SmartRail's own Identity & Access module, not in Authentik.

**In MVP 1, only the staff client is used.** There are no passenger accounts in MVP 1, so the passenger
app and the station display call public endpoints with no token at all. The staff accounts (Network
Controller, Guard, Auditor) are seed data: they are defined in a checked-in Authentik **blueprint**,
Authentik's declarative YAML configuration format, and come up pre-configured with `docker compose up`.
There is no account-management screen and no System Admin role in MVP 1.

**Why not build authentication ourselves:** storing and hashing passwords, running password-reset
flows, and issuing sessions and tokens securely is a solved problem with real failure modes when done
casually. Building it for an assessment would spend time on something brief §14 does not ask for. §14
asks for an identity *model*, not a hand-written auth server. Authentik is mature and self-hostable,
so it removes that risk without depending on a third-party SaaS's availability or cost.

**Why one Authentik instance for staff and passengers, not two systems:** I originally argued for
keeping passenger accounts in SmartRail's own database, because they are optional and low-trust. The
owner's reasoning is the one recorded here. Running two authentication strategies, one delegated and
one hand-written, is itself an ongoing maintenance and security burden for a solo engineer, heavier than
configuring a second client on the same identity provider. And storing any user's login in SmartRail
is something to avoid on principle. This decision still stands, and it takes effect when passenger
accounts return. The passenger client then gets a different audience from the staff client, and
SmartRail checks the audience on every request before it looks at a role claim, so a passenger token
can never reach a staff endpoint.

**Who administers Authentik:** a single super-user identity outside every application role, used only
to bootstrap and maintain the identity provider. When a System Admin role returns in a later MVP, it
will manage staff accounts through SmartRail's own screens. Those screens will call Authentik's
management API with a service account limited to user management. They will not log in to Authentik's
console. This keeps the identity provider's highest-privilege credential away from every application
role, the same no-god-user rule `moscow-scope.md` M-04 applies to operational data.

**Why RBAC lives in SmartRail, not in Authentik:** Authentik answers "who is this". SmartRail's
Identity & Access module answers "what may they do, and can that be proven": the role catalogue, the
check on every write endpoint, and the audit entry for every denial (brief §14, M-04, the
separation-of-duties end-to-end flow). Keeping that decision in SmartRail's code also keeps it where
§14 and §26 assess it. It is an authorisation model that can be explained, tested and shown working,
rather than a third-party product's configuration screen.

**Where a staff member's role is assigned (owner decision, 2026-10-20):** in Authentik, as membership
of one of the groups `network_controller`, `guard` or `auditor`, defined in the blueprint. SmartRail
reads the role from the token's `groups` claim, ignores groups outside its catalogue, and refuses a
token naming none or more than one. The catalogue, the check and the denial event stay in SmartRail;
only the assignment moved. SmartRail still keeps a seeded staff record per user, so every change and
audit entry has a stable staff ID. The trade-off is that a role change happens in Authentik, so it is
recorded in Authentik's event log rather than SmartRail's audit record.

**The catering provider** authenticates with an API key, not through Authentik. A provider is a
machine client, and an API key is the narrow, least-privilege credential CAT-01 asks for. In MVP 1 the
key is seed data (owner decision, 2026-10-19), and SmartRail stores only a hash of it.

---

## 6. Real-time distribution: Server-Sent Events

**Decision:** every client that receives live updates in MVP 1 (the passenger app, the station display
and the Guard's app) uses **Server-Sent Events (SSE)**, a one-directional stream over plain HTTP. The
Guard's departure logs go to the server as ordinary HTTP requests from the offline queue, not over the
live channel.

**Why SSE:** in MVP 1, no client sends anything back over the live channel. They all only receive. SSE
is designed for exactly that. It runs over plain HTTP and needs no special client library. It also
reconnects automatically and sends the last event ID it saw, so a client that reconnects resumes from
where it left off. That matters for a kiosk display that may sit disconnected for a while
(`moscow-scope.md` M-02, PAX-16) and for a Guard's phone in weak signal.

**Why SignalR was dropped:** the earlier design used SignalR (WebSocket) for the crew app, because the
Driver would acknowledge Controller notices over it. The lean cut deferred both the Driver and notices,
so the Guard's app now only receives updates, and its writes are idempotent HTTP calls that work
better from an offline queue than a held-open socket. Keeping SignalR would have meant building and
testing a second transport for a client that has no two-way need. SignalR returns when a two-way need
returns, with Driver notices and acknowledgements in a later MVP.

Live updates reach the SSE streams because Distribution's stream hub is a consumer of the
`JourneyStateChanged` event on the event bus (§3); a client that resumes with `Last-Event-ID` reads the
persisted `journey_events` stream through Live Operations' contract and then goes live.

---

## 7. Observability and hosting: Sentry, no Aspire

**Decision:** Sentry is the only observability tool. .NET Aspire, which I proposed alongside it for
local orchestration, was dropped.

**Why Aspire was dropped:** Aspire's real value is as a local-development orchestrator and dashboard. It
gives one-command startup and a development-time view over OpenTelemetry traces, logs and metrics. It is
not a production hosting or monitoring product, and its dashboard does not carry over to wherever
SmartRail is deployed. Brief §19 requires the organisation to be "capable of operating the platform in
production", and §26 assesses operational thinking on that basis. A tool whose main value disappears
outside local development works against that requirement. The owner's point, that observability should
be production-ready from the start rather than impressive locally and re-tooled later, settles it.

**Why Sentry alone is enough:** Sentry is the same tool, configured the same way, whether SmartRail runs
in Docker Compose on a laptop or is deployed somewhere real. It provides error tracking and performance
tracing, with no migration step between local and production. It integrates with ASP.NET Core directly
and works with OpenTelemetry, so one set of instrumentation serves both purposes.

**What Sentry does not give for free:** `mvp-1-scope.md` EP-17 asks for two business metrics:
publish-to-channel latency and catering webhook delivery failures **[A]**. These are specific to
SmartRail and need explicit instrumentation (custom OpenTelemetry `Meter` instruments), whatever displays
them. Sentry is where that instrumentation is sent, not a replacement for writing it. The guard
sync-backlog metric is deferred **[A]**.

**Local running:** Docker Compose runs the modular monolith, PostgreSQL, Authentik and the simulated
catering provider app as one stack. ASP.NET Core's health-check middleware serves a `/health` endpoint,
and Compose's own health-check directive uses it. That is a cheap, standard way to know a container is
up, without a separate orchestration product.

---

## 8. Development workflow

### Spec-first with OpenSpec

Requirements discovery, MoSCoW prioritisation and MVP scoping are already done (`docs/discovery`,
`docs/scoping`, `docs/user-journeys`). The clarity that a spec-first approach needs before coding is
therefore already in place, which makes spec-first a reasonable first attempt at this workflow rather
than a risk. **OpenSpec** keeps the change-proposal trail (propose, apply, archive) for each piece of
work. Each OpenSpec change covers a capability or an epic, not a single pull request. Writing a spec per
PR would mean writing more spec than code, which defeats the purpose.

### Trunk-based development, stacked PRs via Graphite, `dev` ahead of `main`

Strictly, this is **stacked-PR trunk-based development**, not the classic form, which has direct
commits to `main` gated only by CI and no PRs at all. The distinction matters because the PR review
step, not the branching model, is what lets the owner check subagent-written work in small pieces
before it lands, and that is why this workflow was chosen. Epics are broken into independently
mergeable stacked changes rather than one PR per epic, which Graphite's stacking encourages.

There are two long-lived branches, not one trunk. **`dev`** receives AI- and subagent-written stacked
PRs after review, and **`main`** only receives promotions from `dev`, never a direct PR. The reason is
security, not ceremony. AI-generated code is reviewed carefully at the PR stage, but a compromised or
over-eager subagent run, a bad merge or a misread spec is cheaper to catch and discard on `dev` than on
the branch treated as authoritative and demo-ready. `main` moves forward only after a batch of work on
`dev` has been exercised together (CI and, once they exist, the five end-to-end flows), so `main` never
holds a single subagent's PR in isolation. This costs one extra promotion step per batch. It is
accepted because the product itself makes separation of duties and provable control over every change a
stated property (brief §14, M-04), and the same discipline should apply to who may change the code that
runs it. Graphite's trunk in the `SmartRail-API` repository is set to `dev`.

### Testing: unit, module integration and five end-to-end flows

Three layers, not an exhaustive suite (brief §18, `mvp-1-scope.md` EP-16):

| Layer | What it covers | How |
|---|---|---|
| **Unit** | Domain logic inside one module: expected-time calculation from a late departure, lifecycle transitions, correction rules | xUnit against the Domain layer directly, no database |
| **Module integration** | One module's endpoint-to-persistence path: 403 on a wrong role, the audit write, idempotent departure-log sync, idempotent webhook delivery | `WebApplicationFactory` against a real PostgreSQL instance (Testcontainers), one module at a time |
| **End-to-end** | The five flows in `mvp-1-scope.md`: a delay reaching every channel, offline guard logs syncing once, a platform change everywhere, separation-of-duties denials, and a wrong cancellation corrected | The full stack in Docker Compose, automated in CI |

The middle layer was added to the "unit tests plus five end-to-end flows" first proposed. Most of
SmartRail's real risk is not in domain arithmetic. It is at the boundary between authorisation,
persistence and integration.

---

## 9. Decisions at a glance

| Area | Decision | Rejected alternative | Why |
|---|---|---|---|
| Backend architecture | Modular monolith, 6 modules; per module Clean Architecture (Domain, Application, Infrastructure, Api) with a MediatR CQRS pipeline | Microservices; one undivided application; three layers with endpoints inside Infrastructure (first design) | Real seams exist and need enforcing, but independent deployment and scaling are not real requirements yet. Owner: the three-layer merge emptied the Application layer and blurred the dependency rule |
| Module set | 6 modules; Passenger merged into Distribution for MVP 1 | 7 modules with an empty Passenger module | With accounts deferred, Passenger had no data of its own; it returns with accounts |
| API style | Minimal APIs through Carter modules in each module's Api project; `Result` mapped to problem details at the edge | MVC controllers | Avoids a shared `Controllers/` folder that would undermine module boundaries |
| Data layer | PostgreSQL, one schema per module, no cross-schema foreign keys | One shared schema | Makes the module boundary something the database enforces, not a convention |
| Cross-module reads | In-process query contracts | Event-fed read models in each consuming module (my original proposal) | At MVP 1's scale the copies were the largest piece of plumbing with no problem to solve; a read model can be added behind a contract later |
| In-process events | MediatR, for commands, queries and reactions inside one module | Hand-rolled dispatcher | MediatR already provides the fan-out pattern, is widely understood, and leaves less to build and test |
| Event delivery | A generalised event bus on Postgres: every cross-module event is an outbox row in the publisher's schema, delivered at least once by a dispatcher to named consumers, ordered per key; NATS JetStream later as a transport swap | Only the persisted journey stream as the bus (the first Postgres-only plan); a broker now; a handler inside the unit of work (the catering outbox as first specified) | Owner: one mechanism solves events being published in different ways, and the time constraints rule out a broker now. A broker would still need the outbox, and `journey_events` stays for `Last-Event-ID` catch-up |
| Audit of staff changes | The entry commits in the change's own transaction through the shared request transaction (option X); denials go through the bus | Writing every audit entry from a bus consumer now | Owner: the small coupling is acknowledged and accepted for simplicity and AUD-05; moving to the bus is planned (option Y, §3) |
| Identity | Authentik, one instance; staff client in MVP 1, passenger client when accounts return; RBAC enforced in SmartRail, roles assigned as Authentik groups | Authentik for staff, passenger accounts stored in SmartRail (my original proposal) | Owner: avoids two authentication strategies, and keeps every user's login out of SmartRail |
| Real-time | SSE for every live client | SignalR for the crew app (earlier decision) | No MVP 1 client sends over the live channel; SignalR returns with Driver notices |
| Observability and hosting | Sentry only; Docker Compose for local running | Sentry plus .NET Aspire (my original proposal) | Owner: Aspire's value is local-only and doesn't carry into production, which the brief asks the platform to be ready for |
| API documentation | OpenAPI with the Scalar reference; staff sign in from Scalar with authorization code + PKCE against the same public staff client the operations web uses; the provider key is entered by hand | Swagger UI; pasting a token obtained outside Scalar (the 2026-10-19 decision, replaced) | Owner choice of Scalar. The staff client has to be public with PKCE anyway, because the frontends sign staff in from the browser, so Scalar reuses it (owner decision, 2026-10-21) |
| Development process | Spec-first with OpenSpec, one change per capability | One spec per PR | Keeps the spec a design tool rather than paperwork larger than the code |
| Commit and review | Stacked-PR trunk-based development via Graphite; reviewed on `dev`, promoted to `main` | Classic trunk-based (no PRs); a single `main` trunk | The PR step reviews subagent output in small pieces; `dev` takes the risk of AI-written code before it reaches the authoritative branch |
| Testing | Unit, module integration, five automated end-to-end flows | Unit and end-to-end only (my original proposal) | Most of the real risk sits at the authorisation, persistence and integration boundary |

---

## 10. What this document deliberately leaves open

- The seed data itself (station names, the timetable and staff accounts), which the owner defines.
- When to adopt NATS JetStream. The trigger is more than one API instance, or a module split into its
  own process **[A]**. Until then the §3 event bus serves.

**Planned (owner, 2026-10-21):** staff state changes will move onto the event bus (option Y, §3): the
change commits with a durable to-audit message in its own outbox and an Audit consumer writes the entry a
moment later. The shared request transaction goes, and the wording of AUD-05 and of the audit requirement
that a change and its entry commit together changes with it.
