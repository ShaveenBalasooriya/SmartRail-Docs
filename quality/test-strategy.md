# SmartRail — Test Strategy

**Status:** living document, written 2026-10-29 by work package QA-D1 (QA thread). It describes what
is built, what is planned, and why. It does not claim a result that no source records.

**Sources:** `docs/scoping/mvp-1-scope.md` (EP-16, the five flows), `final-assessment-brief.md`
§18 and §23, `docs/architecture.md` §8, `docs/security/threat-model.md`,
`SmartRail-QA/COORDINATION.md`, `SmartRail-QA/browser/FEASIBILITY.md`, and the test code under
`SmartRail-API/tests`.

---

## 1. Purpose

The assessment asks for "a structured quality engineering approach" and for us to "decide which
tests provide the most value and justify the selected approach" (brief §18). It also names the QA
artifacts: a test strategy, functional scenarios, five automated end-to-end flows, identified
defects, evidence, and a quality assessment (brief §23).

This document answers one question: **where should test effort go, and why there rather than
somewhere else?**

Short answer:

- Test the five flows where a mistake hurts a passenger, loses a staff action, or breaks the audit
  chain. Do not test every screen or every endpoint for its own sake.
- Run the flow logic as fast, deterministic API tests (Layer A). Prove deployment wiring later with
  a thin real-stack suite (Layer B). Use a browser only where the browser is the thing at risk.
- Make every test build its own data and clean it up, so tests never depend on order or on a global
  count.

---

## 2. Decisions

Each decision is recorded with its reason. "Owner" means the owner decided it; "QA" means the QA
thread decided it within the owner's approach.

| # | Decision | Reason | Who |
|---|---|---|---|
| D1 | Automate **five** end-to-end flows, not every path | The brief and EP-16 name five. Five broad flows cover the product's one claim: a change is made once and reaches every channel | Owner / brief |
| D2 | Rank the flows by risk, and let the ranking decide what must be automated and reviewed first | Time is fixed. The ranking says which failures matter most. Build order follows what is buildable: Flow 4 was the pilot because it needs no provider and no Guard stream | QA |
| D3 | **Two layers.** Layer A: in-process flow tests. Layer B: thin real-stack suite | Layer A runs in minutes and pins behaviour precisely. Layer B proves Docker Compose, real Authentik and the provider wiring, which Layer A deliberately replaces with fakes | Owner |
| D4 | Playwright for **three OpsWeb checks only**: publish delay, publish platform change, Auditor sees no publish links | These are real UI paths with real forms. Everything else fails faster and more precisely at API level | Owner |
| D5 | Typed API drivers + builders + a flow-step layer; Page Objects only in Playwright | Same idea as Page Object / Screenplay, at API level: a test reads as intent, and a URL or body changes in one place | QA |
| D6 | Each test builds its own journey, uses unique ids, and cleans up — including audit rows, via baselines | The database is shared and the audit table is append-only. Global counts and leftover rows are how suites become order-dependent | QA |
| D7 | Fixed clock, polling with a deadline, no sleeps, no order dependence | A flow test must fail for a real reason, not because a machine was slow | QA |
| D8 | One test, one behaviour, and a failure message that names the behaviour | A red test should say what broke without opening the debugger | QA |
| D9 | White-box checks where the API cannot show the truth: transaction, outbox, ordering, chain | The risk is at the boundary between authorisation, persistence and integration (architecture §8) | QA |

### 2.1 Layer A, in one paragraph

A Layer A test starts the **real Host** in-process (`WebApplicationFactory`), against **real
PostgreSQL** (Testcontainers), with **locally minted staff tokens** and a **frozen clock**. It then
drives the flow through the real HTTP endpoints and reads the database only where the API cannot
answer the question. No Authentik, no Docker Compose, no provider app.

Where this is implemented: `SmartRail-API/tests/SmartRail.Host.IntegrationTests/Infrastructure/`
(`SmartRailApiFactory`, `PostgresFixture`, `TestTokens`) and `.../Flows/`.

### 2.2 Layer B, in one paragraph

Layer B runs the same five journeys against the **real stack** from `docker compose`: the API
container, PostgreSQL, real Authentik sign-in, and the simulated catering provider. It is **not
built**. Its job is not to re-test rules; it is to prove the wiring that Layer A replaces with test
doubles — real token issuance, real redirect URIs, real container networking, real webhook
delivery. Layer B is deliberately thin: one happy path per flow, plus the provider receipt.

---

## 3. Test layers: what each one catches

| Layer | What it catches | Where it lives | How it runs | Status |
|---|---|---|---|---|
| **Unit** | Rules inside one class: expected-time arithmetic, lifecycle transitions, correction rules, value objects, hash chaining | 6 unit projects under `SmartRail-API/tests` (`Audit`, `Distribution`, `Host`, `LiveOperations`, `RailNetwork`, `SharedKernel` `.UnitTests`) | xUnit, no database | Built |
| **Architecture** | A dependency pointing the wrong way, a module reaching across a boundary | `SmartRail-API/tests/SmartRail.ArchitectureTests` (NetArchTest) | xUnit | Built |
| **Integration (module)** | One endpoint to persistence: 400/403/409 contracts, idempotent replay, outbox rows, audit writes, SSE frames and resume | `SmartRail-API/tests/SmartRail.Host.IntegrationTests` (`Audit`, `Authorization`, `Catering`, `Conventions`, `Documentation`, `EventBus`, `Health`, `LiveOperations`, `Logging`, `Persistence`, `Public`, `RateLimiting`, `Seeding`) | `dotnet test`, Testcontainers | Built |
| **Layer A — flow** | A whole journey across modules: publish → state → public view → stream → provider row → audit, including the negative case | `.../SmartRail.Host.IntegrationTests/Flows` | `dotnet test --filter FullyQualifiedName~Flows` | Flow 4 built; Flows 1, 2, 3, 5 planned |
| **Layer B — real stack** | Deployment wiring: Compose networking, real Authentik tokens, provider container, migrations at start-up | New suite, to be created | Docker Compose up, then run | **Not built** |
| **Browser (Playwright)** | The OpsWeb form and gate as a person meets them: review dialog, validation messages, role-hidden actions | `SmartRail-QA/browser` | `npx playwright test` | Installed and smoke-tested; flow checks not written |

**Why this split, in one line each.**

- Unit is cheap and localises a domain bug to one rule.
- Integration is where the real risk sits: authorisation, transactions, persistence, integration
  (architecture §8).
- Layer A proves the *journey* crosses modules correctly without paying for a container stack on
  every run.
- Layer B proves the stack, not the rules.
- Playwright proves the UI, not the contract.

**What each layer deliberately does not do.**

| Layer | Not its job | Covered instead by |
|---|---|---|
| Unit | HTTP status codes, database constraints | Integration |
| Integration | Cross-module journeys, provider receipt end to end | Layer A, Layer B |
| Layer A | Real Authentik, real containers, real provider | Layer B |
| Layer B | Edge cases, negatives, precise rule diagnosis | Layer A |
| Playwright | API contracts, 401/403 details, SSE frames | Layer A |

---

## 4. Why only five end-to-end flows

An end-to-end flow is expensive. It needs a database, a clock, seeds and cleanup. Its failure is
also blunt: "the journey broke somewhere". So it earns its cost only when the thing being proven is
*the journey itself*.

The product's single claim is that a change is made once and reaches every channel
(`mvp-1-scope.md`, MVP 1 goal). Five flows test that claim from five different angles: time
recalculation, offline replay, a field with metadata, refusal, and reversal. Those are the five
ways the claim can be false.

Everything else is cheaper lower down:

| Question | Best layer |
|---|---|
| Is 15 minutes a valid delay and 721 not? | Integration (`PublishDelayTests`) |
| Does a stop that already departed refuse a delay? | Integration |
| Does the display order departures by time? | Unit (`PublicDepartureSelectionTests`) |
| Does the audit chain detect an edited row? | Integration (`AuditIntegrityTests`) |
| Does the Auditor see a publish button? | Browser |

### 4.1 Risk ranking

Ranked by impact (what goes wrong), likelihood (how easily it goes wrong), and blast radius (who is
hurt). Ranking is the QA thread's, accepted by the owner.

| Rank | Flow | Impact if it breaks | Likelihood | Blast radius: who is hurt | Why this rank |
|---|---|---|---|---|---|
| 1 | **Delay reaches every channel** | Passengers wait for a train that is late, at the wrong time, on every surface | High: it is the most-used change and it recalculates times at every later stop | Every passenger at every affected stop, plus the Guard and the provider | Highest reach, highest frequency, and the arithmetic is the easiest thing to get subtly wrong |
| 2 | **Wrong cancellation corrected** | A train that is running is shown as cancelled, or the correction never arrives and passengers stay away | Medium: requires a human mistake first, but the recovery path is the product's promise | Passengers who abandon travel, staff who must explain, the audit trail | A broken correction turns one mistake into a permanent one, and it is the flow that proves the audit keeps the original |
| 3 | **Separation of duties denials** | The wrong role can change the railway, or a refusal leaves no trace | Low for a working policy, high for a new endpoint added without one | Passengers (bad data published), the organisation (no defensible record) | Smallest code path, largest consequence: it is the security control the whole audit story rests on |
| 4 | **Offline guard logs sync once** | Departures are lost, duplicated, or re-timed; progress and ETAs drift | High: offline queues are where duplicate and out-of-order writes live | Passengers (wrong progress), the Guard (work lost), controllers (bad picture) | Genuinely hard (idempotency, original timestamps), but a smaller audience per failure than 1–3 |
| 5 | **Platform change everywhere** | Passengers stand at the wrong platform | Medium | Passengers at one station, one platform | Narrow blast radius: one stop, one value, no time arithmetic |

**Why Flow 4 leads the build even though it ranks third.** It needs no provider and no Guard
stream, so it could be built and proven first. Risk rank orders *attention*, not *build order*. See
§13.1.

### 4.2 Flows 1 and 3 overlap, and why both stay

Both are "the controller publishes, and every channel must show it". They share drivers, steps and
fixtures. They are not the same test.

| | Flow 1 — delay | Flow 3 — platform change |
|---|---|---|
| What changes | Times at **every later stop** | One field at **one stop** |
| What is really proven | Recalculation: the delay is applied from a stop onward, and downstream expected times move | Propagation with metadata: the new platform, the time it became known, and the provenance label |
| What breaks if untested | Passengers get a late train at the old ETA; the Guard's next stop is wrong | Passengers stand at the old platform; the change cannot be told apart from the timetable |
| Adds to the suite | Live-state arithmetic across stops, Guard stream content | Platform field, `knownAt`-style time, automatic-publication label, display prominence data |

Merging them would save one fixture and lose one of two different failure classes. Keeping both is
cheap because the machinery is shared.

---

## 5. Tool choice, and the alternatives we rejected

**Chosen:** xUnit + `WebApplicationFactory` + Testcontainers (PostgreSQL) for Layer A;
Playwright (`@playwright/test`) for the browser layer; Docker Compose for Layer B.

Why xUnit and not another runner: it is already the project's runner in every existing test project,
so there is no second convention, no second CI step and no migration. Why `WebApplicationFactory`
and not a live server: it starts the real Host with the real middleware, DI and endpoints, in
process, with no port or container orchestration. Why Testcontainers and not an in-memory database:
the flows depend on PostgreSQL behaviour — schemas, `ON CONFLICT`, triggers, sequence ordering —
which an in-memory provider would fake away.

| Alternative | Why it was rejected |
|---|---|
| **Postman / Newman** | Collections are JSON, not code. No types, no refactoring, no shared builders, and assertions are strings. It cannot read the database for white-box checks, and it duplicates what the integration project already does |
| **REST Assured / Karate** | JVM toolchains alongside a .NET test suite: a second runtime and a second CI job for no gain. Karate's DSL would also hide the C# intent the assessment asks us to explain |
| **Cypress** | Browser-only. Our risk is mostly below the browser, and Cypress cannot do the in-process API/database checks Layer A needs. Playwright already covers the browser part with a smaller footprint |
| **Selenium** | Lower-level than Playwright, needs more glue (waits, drivers, grid) for the same three checks. Playwright's auto-waiting and trace viewer are why we picked it over Selenium; `FEASIBILITY.md` records the installed version and constraints, not a comparison |
| **Raw `HttpClient` calls inside each test** | Works, but repeats URLs, bodies, auth and parsing in every test. The first endpoint rename touches dozens of files, and failure messages become status codes with no intent |
| **Full Screenplay framework** | Correct in principle, too much ceremony for one engineer and five flows: actors, abilities, tasks and questions for what two small layers already do. We borrowed its idea (intent in the test, mechanics in the layer), not its structure |
| **SpecFlow / Reqnroll (Gherkin)** | Adds a feature-file layer and step bindings that mostly restate the test name. The scenarios in `functional-scenarios.md` serve the same communication purpose without a second language |
| **A mocking library for the provider** | The provider row is written by a real consumer of the real outbox. A mock would test the mock. Until the provider exists, those assertions are marked pending, not faked |

**Playwright is narrow on purpose.** Three checks only, because the browser adds time and flakiness
and proves nothing about the 403 body, the SSE frame or the outbox row. The honest split is recorded
in `SmartRail-QA/browser/FEASIBILITY.md` §5.

**Known Playwright constraints** (from `FEASIBILITY.md`): the session lives in `sessionStorage`, so
`storageState` cannot carry it; the Authentik blueprint pins the redirect URI to
`http://localhost:5173`, so a signing-in browser test must run on that port; there are no
`data-testid` attributes, so locators use role and label.

---

## 6. Framework shape and maintainability

### 6.1 The four parts

| Part | Job | Example |
|---|---|---|
| **Drivers** | One class per surface. Knows URLs, verbs, bodies, token, and how to parse the answer into a typed record | `ControllerDriver`, `GuardDriver`, `AuditorDriver`, `PublicApiDriver`, `AuditDriver` |
| **Builders** | Create valid domain data for a test, with unique ids | `JourneyBuilder` publishes a real `JourneyGenerated` announcement and waits for the live state |
| **Steps** | The flow-step layer: what a journey must look like for a step to be a real attempt, and what the attempt answered | `ControllerWriteSteps.ArrangeAsync` / `AttemptAsync` |
| **Support** | The world a flow runs in, and the assertions | `FlowContext`, `Eventually`, `Verify`, `IdentityOutbox`, `JourneyInternals`, `StaffSessions`, `SeededNetwork` |

A test file contains intent only. It names a write, a role and a behaviour. It contains no URL, no
SQL and no status-code literal. That is why `SeparationOfDutiesFlowTests.cs` reads as eleven
sentences about access control.

**Page Objects exist only in Playwright** (`SmartRail-QA/browser/pages/`), because a browser page is
the only place where "a page" is a real concept. There is no Page Object for an API.

### 6.2 Adding flow six

Flow six (say, "early termination reaches every channel" — today termination is exercised only as a
refusal case in Flow 4 and as integration tests) needs **one new test file and, at most, one new step
method**. Concretely:

| Add | Why | Does anything else change? |
|---|---|---|
| `Flows/EarlyTerminationFlowTests.cs` | The flow itself, one class, `[Collection(PostgresCollection.Name)]`, `IAsyncLifetime` for the fixture | No |
| A method on `ControllerWriteSteps` (or a new `...Steps` class) **only if** the journey needs a new arrangement | Steps own "what the journey must look like" | No |
| A method on an existing driver **only if** a surface is not yet covered | Drivers own URLs and bodies | No |
| A property on `FlowContext` **only if** the flow needs a new surface or a new baseline | `FlowContext` composes the world and owns cleanup | No |

What does **not** change: the factory, the Postgres fixture, the token minting, the frozen clock,
`Eventually`, `Verify`, `JourneyBuilder`, the collection definition, or any existing test. That is
the point of the split: a new flow is new intent, not new plumbing.

### 6.3 Scaling limits, stated honestly

- Every flow test starts its own Host instance. That is a few seconds each; it is why the suite is
  minutes, not seconds (`COORDINATION.md` line 40: 40 flow tests in about 2 minutes, dominated by
  container start).
- The database is shared across the run. Cleanup is per test, by baseline. A test that forgets its
  baseline pollutes its neighbours — this happened once and is recorded as `DEF-003`.
- Layer A cannot prove container wiring. That is Layer B's job and Layer B does not exist yet.

---

## 7. Test data

Rules, all of them visible in `Flows/`:

| Rule | How | Why |
|---|---|---|
| Each test builds its own journey | `JourneyBuilder.CreateAsync()` publishes a real `JourneyGenerated` event with a fresh `Guid.CreateVersion7()` | No test depends on seed rows another test might change |
| Unique identifiers per journey | New journey id, new stop rows, new change ids | Two tests can run in any order, or in parallel, without colliding |
| The journey is created the real way | Through the network module's outbox and the Live Operations consumer, not by inserting rows | The test exercises the same path production uses |
| Cleanup names what it made | `JourneyInternals.CleanUpAsync(journey.Id)`; `IdentityOutbox.CleanUpAfterAsync(baseline)`; `AuditDriver.ClearAfterAsync(baseline)` | The shared database is left as the next class expects to find it |
| Baselines, never global counts | `FlowContext` records an identity-outbox position and an audit chain position at start, and removes rows **above** them at the end | Other agents and other tests write at the same time; a global count would be a moving target |
| Audit rows are cleaned deliberately | Delete above the baseline with the append-only trigger disabled, then re-enable in a `finally` | The record is append-only by design; a test that writes denials must return the record to its prior shape |
| No seeded E2E journey | There is none (`COORDINATION.md` line 28). Tests create what they need | A fixed seed would be shared mutable state |

**Why cleanup matters more than it sounds.** The audit table is append-only and shared. A leftover
denial row is not inert: existing tests look up "the newest denial for this actor and action", so a
leftover row makes them assert against the wrong entry. That is exactly `DEF-003`.

---

## 8. Determinism

| Rule | Implementation | Why |
|---|---|---|
| **Fixed clock** | `FlowContext` replaces `TimeProvider` with `FrozenTimeProvider` at noon on the railway's today | A refusal's recorded time can be asserted exactly, and no test crosses midnight mid-run |
| **No sleeps** | `Eventually.ValueAsync` polls every 100 ms up to 20 s and fails with the last value seen | The event bus delivers after the response; a fixed sleep is either slow or flaky |
| **No "nothing happened" by waiting a fixed time** | `Eventually.NothingAsync` watches a 2 s quiet window, longer than one dispatcher poll | Silence is a real assertion only if the window is longer than a delivery would take |
| **No order dependence** | Each test creates and removes its own data; the frozen clock is per factory | xUnit may run classes in any order; the suite must not care |
| **No parallel-write assumptions** | Where concurrency matters, the test asserts the contract (one 201 and one replay, or one 201 and one 409) rather than a race outcome | The rule is "exactly once", not "whichever won" |
| **Fresh token per read** | `StaffSession.Token` mints on each access | A test token lives five minutes; a suite can run longer |

The suite has been run whole twice (`COORDINATION.md` line 43: 311/311 both times), and once with four
namespaces filtered so Flows ran first (`line 44`: 218/218, with the eight denial tests running after
the flows). Both are recorded; neither is a run of mine.

---

## 9. Negative scenarios

Every flow has negatives, and they are part of the flow, not an afterthought. A positive-only flow
suite proves the happy path and hides the refusal path, which is where authorisation and validation
live.

| Flow | Negative scenarios that must exist | Level |
|---|---|---|
| 1 Delay | Wrong role (Guard, Auditor) → 403; no token → 401; delay at a departed stop → 409; 0 or 721 minutes → 400; unknown journey → 404; closed journey → 409; failing audit step → no change | Layer A + existing integration |
| 2 Offline guard logs | Replay of the same log id → no second record; same stop with a new id → 409; a stop the service does not call → 400; future timestamp beyond the skew → 400; another Guard's journey → 403 and audited; stop logged out of order → refused | Layer A + existing integration |
| 3 Platform change | Platform of another station → 400; unknown platform → 400; same platform → 409; departed stop → 409; non-controller → 403 | Layer A + existing integration |
| 4 Separation of duties | Guard and Auditor refused on **all five** publishing routes → 403 with problem details; no token → 401; expired token → 401; foreign-key token → 401; refusal leaves journey, events, outbox and public view untouched; exactly one denial entry; chain still verifies | Layer A (built) |
| 5 Wrong cancellation corrected | Correcting an already-corrected change → 409; correcting a correction → 409; unknown change → 404; completed journey → 409; non-controller → 403; failing audit step → no correction | Layer A + existing integration |

**A negative must not pass vacuously.** Flow 4 includes a controller **positive control on the same
route**, so a broken route cannot make the 403 assertions pass by refusing everyone. This rule
applies to every new flow: for each refusal, prove the same request succeeds for the right role.

---

## 10. White-box checks

White-box here means "ask the database or the internal state a question the API cannot answer". It
is used sparingly, where the risk is real.

| Target | What is proven | Where it is proven |
|---|---|---|
| **Shared request transaction** | A change and its audit entry commit together, or neither does | `Persistence/RequestTransactionTests.cs`; `A_failing_audit_step_leaves_no_change_event_outbox_row_or_state_change` in the publish suites |
| **Outbox at-least-once** | A published change writes an outbox row in the same transaction and survives a restart | `EventBus/JourneyGeneratedOutboxTests.cs`, `EventBus/RestartRedeliveryTests.cs`, `LiveOperations/JourneyStateChangedOutboxTests.cs` |
| **Idempotent replay** | The same client id replays without a second record | `LogDepartureTests` (`A_new_log_..._a_replay_changes_nothing`, `Two_identical_requests_at_once_...`) |
| **Ordering by StreamPosition** | Events arrive in stored position order, once | `Public/PublicStreamTests.cs` (resume in order, once, then live) |
| **Resume and reset** | A missed window replays; an unknown, future or malformed id gets `event: reset` first | `PublicStreamTests` (`A_reconnecting_stream_receives_the_events_it_missed_in_order_once_then_goes_live`, `A_stream_whose_position_is_unknown_starts_with_reset_then_continues_live`) |
| **Audit hash chain** | An edited value or hash breaks the chain at the right position | `Audit/AuditIntegrityTests.cs` |
| **Authorisation policies** | Every staff write refuses a wrong role with 403; the fallback policy closes unclassified endpoints | `Authorization/StaffAuthorizationTests.cs`, `Authorization/ProviderAuthenticationTests.cs` |
| **Denial audit** | A refusal is announced once through Identity's outbox and recorded once with actor, role, action and server time | `Flows/SeparationOfDutiesFlowTests.cs` (10 cases), `Audit/AccessDeniedConsumerTests.cs` |

Two defects were found by exactly this kind of check, both by the Backend team's LD6 work:
`JourneyStateChanged.StreamPosition` was 0, and the outbox transport mishandled a removed message so
later deliveries in the same cycle were skipped (`COORDINATION.md` line 30). They are recorded as
`DEF-001` and `DEF-002`. We did not find them.

---

## 11. Security-relevant tests

The threat model (`docs/security/threat-model.md`) lists 18 risks and names where each control is
verified (mostly a spec task). This table maps those to the test that exists today. The QA
contribution is to keep those tests honest and to add the flow-level proof.

| Threat model risk | Control | Test evidence |
|---|---|---|
| 1 Forged or expired token | Signature, issuer, audience and lifetime validated; 30 s skew | Flow 4: expired token, foreign-key token, no token → 401 |
| 2 Two roles, or the wrong role | Exactly one catalogue group; none or many means no role | `StaffAuthorizationTests` (`A_token_with_two_role_groups_has_no_role`); Flow 4 per-route |
| 3 Provider key used as staff credential | Two schemes, policies named per endpoint | `ProviderAuthenticationTests` |
| 4 Repudiation of a change or a refusal | Append-only hash-chained entry with staff id and role | `AuditIntegrityTests`; Flow 4 denial entry |
| 5 Token in a log or an error | No tokens, codes or verifiers logged; scrubber | `Logging/PiiLogTests.cs`; `Audit/AuditIntegrityTests.No_entry_holds_a_token_an_api_key_or_an_address` |
| 6 Staff or audit data on a public endpoint | Public view only, `no-store`, no cookies | Unit: `PublicJourneyViewJsonTests.A_staff_identifier_in_the_live_state_never_reaches_the_public_payload`; integration: `PublicJourneyDetailTests` (no-store, no cookie), `PublicStreamTests` |
| 7 Public endpoint flooded | 120 requests a minute per IP, 429 with `Retry-After` | `RateLimiting/PublicRateLimitTests.cs` |
| 9 Provider key guessed or replayed | Prefix + hash stored, constant-time compare, same 401 | `ProviderAuthenticationTests` (revoked key, wrong secret, same body as unknown prefix) |
| 10 One provider starves another | 60 a minute per key | **No test yet.** Only the public 120/min limit is covered (`RateLimiting/PublicRateLimitTests.cs`) |
| 14 Audit rewritten | Append-only trigger plus chain check | `Audit/AuditEntriesStorageTests.cs`, `AuditIntegrityTests` |
| 15 Module reaches another module's tables | Schema per module, no cross-schema FK | `SmartRail.ArchitectureTests` |
| 17 Authorization code intercepted | Public client + PKCE S256 + exact redirect URI | Layer B (real Authentik) — **pending**, because Layer A never touches Authentik |
| 18 Stolen refresh token | 15-minute access tokens, role re-read per request | Layer B — **pending** |

**The hole to be explicit about:** PKCE, refresh-token behaviour and real key discovery are only
proven in Layer B. Layer A mints its own tokens (`TestTokens`) and never contacts Authentik. Until
Layer B exists, threat-model risks 17 and 18 have design evidence, not test evidence.

The provider rate limit (risk 10) is also unproven: the limiter is configured, but only the public
limit has a test. That is a gap in coverage, not a known failure. It is listed here rather than
quietly omitted.

---

## 12. How AI was used in test automation

This section lists only what the AI log records. Each row is a source.

| What AI did | Evidence | How it was validated |
|---|---|---|
| Wrote a readiness report on the flows, risk ranking, spec gaps and tool choice, with no test run | Log row 385 | Read-only; the row states nothing was executed |
| Installed Playwright 1.64.1 and `@playwright/cli` 0.1.22, scaffolded `SmartRail-QA/browser`, and proved a smoke test can fail | Log row 392; `FEASIBILITY.md` §2 | Typecheck clean; smoke passed; a deliberately wrong expected heading produced one precise failure, then reverted |
| Built the Layer A framework and Flow 4: 16 new files, 40 tests | Log row 402; `COORDINATION.md` lines 38–45 | Built with `-warnaserror`; 40/40 Flows; 311/311 whole project twice; 218/218 ordered run |
| Found that AI-written tests left denied audit rows and broke 8 existing theories | `COORDINATION.md` line 42; log rows 396, 397 | Found by the **Backend** team running the full suite, not by the QA tests themselves |
| Corrected its own factual error (claimed Authentik sent no CORS header, then re-read the output) | Log row 404 | The orchestrator re-read its own curl output and rejected the claim |

The owner's decisions that shape this strategy — spec is outdated, two layers, fresh stack and
unique ids, test every Guard and Auditor route, use Playwright, document simply, follow best
practices — are recorded in log row 387.

**A number that has moved.** The `311` above is the whole Host integration project as recorded in
`COORDINATION.md` lines 40 and 43. The catering migration later added four integration tests, and AI
log row 406 records the project at **321/321** (281/281 with Flows excluded, meaning the 40 flow cases
still pass). Neither figure is a run of mine.

**What this shows, plainly.** AI wrote most of the test code and all of the first draft of this
document. AI also wrote a suite that passed alone and broke its neighbours, and the mistake was
caught by a different team running the whole suite. That is the reason this strategy insists on
baselines, cleanup and ordered runs rather than trusting a green filtered run.

---

## 13. Status, gaps and known risks

### 13.1 Status of the five flows

| Rank | Flow | Layer A | Layer B | Browser | Note |
|---|---|---|---|---|---|
| 3 | Separation of duties (built first) | **Built**, 40 cases | Pending | Pending (Auditor gate) | The pilot; no missing backend pieces |
| 1 | Delay everywhere | Planned | Pending | Pending | Needs the provider row to be asserted end to end |
| 2 | Wrong cancellation corrected | Planned | Pending | Pending | Correction already exists at integration level |
| 4 | Offline guard logs | Planned | Pending | Not applicable (no Guard web app) | Idempotency already exists at integration level |
| 5 | Platform change | Planned | Pending | Pending | Shares steps with Flow 1 |

Flow numbers here are the ranking order (D2), not the order in `mvp-1-scope.md`. The mapping is:
rank 1 = scope flow 1, rank 2 = scope flow 5, rank 3 = scope flow 4, rank 4 = scope flow 2, rank 5 =
scope flow 3.

### 13.2 Blocked work, named

| Item | Why it is blocked | What is true today |
|---|---|---|
| Provider receipt assertions (Flows 1, 3 and 5) | Catering and the simulated provider are **not built** (`COORDINATION.md` line 27) | Marked pending; never faked |
| Layer B | Needs the Compose stack plus real Authentik and the provider; not built | Design only |
| Real Authentik PKCE / refresh (threat risks 17, 18) | Layer A mints its own tokens | Design evidence only |
| Playwright flow checks | No disposable seeded journey for the shared dev database (`FEASIBILITY.md` §6) | Smoke test only |

**Not blocked, because they landed.** The public stream with `Last-Event-ID` resume and reset
(`COORDINATION.md` line 25; AI log row 390), and the Guard stream scoped to the assigned Guard (AI
log rows 395–396, `Public/GuardStreamTests.cs`). The planned flow tests for Flows 1 and 2 can use
both. Note that `COORDINATION.md` line 26 still says the Guard stream has not landed; that line was
written before LD8 finished and is superseded by AI log row 396.

### 13.3 Known limits and risks of this strategy

| Limit | Consequence | Mitigation |
|---|---|---|
| Layer A fakes Authentik and the provider | Wiring bugs survive it | Layer B, once built |
| The audit table is shared and append-only | Leftover rows can break unrelated tests | Per-test baselines; ordered run proof; `DEF-003` recorded |
| Existing denial tests find a denial by actor + action | A stale newer row wins, and the test fails for the wrong reason | `DEF-004`, open, owner's decision |
| Flow tests cost minutes, not seconds | Slow feedback if run on every edit | Run flows on the change, not on every save; integration suite stays the fast gate |
| No load or performance testing | Publish-to-channel latency under load is unproven | Out of MVP 1 (EP-16 Out) |
| No formal accessibility testing | Contrast and screen-reader behaviour are design claims | Out of MVP 1 (EP-16 Out) |
| Counting test cases is manual | "40 cases" is a count of `InlineData` rows | Every number in this document cites a file or a log row |

### 13.4 What would make this strategy wrong

Stated so the reasoning can be challenged:

- If the platform grows past one engineer and many flows, the per-test Host start becomes the
  bottleneck and a shared, long-lived stack would be worth the isolation risk.
- If the provider becomes a real external party, Layer B stops being optional and signed webhooks
  become a test target.
- If order dependence cannot be eliminated, the suite must run serially and the time cost accepted.
  Right now it does not need to.
