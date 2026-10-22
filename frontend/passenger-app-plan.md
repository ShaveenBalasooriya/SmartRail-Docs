# SmartRail Passenger App — Plan

Plan for the Flutter passenger app (`SmartRail-PassengerApp`, EP-10). It extends `docs/frontend-architecture.md`
§5 and §6 and `docs/scoping/mvp-1-scope.md`. Written 2026-10-22 from the documents and the running API
(`http://localhost:5080`). Decisions the owner made are marked **[Owner]**; my choices are marked **[A]** and
open to change.

**Owner decisions:** Flutter; Clean Architecture; Riverpod for state management; one repo per app; build
against the real API where an endpoint exists and mock the rest behind interfaces; passengers have no
accounts (public endpoints only).

---

## 1. What exists in the API today

Read from the running API's `/openapi/v1.json` and the OpenSpec change `add-live-distribution`.

| Need | Endpoint | Status |
|---|---|---|
| Stations (platforms, lines, interchange flag) | `GET /api/v1/network/stations`, `/network/stations/{id}` | **Real** |
| Lines with ordered stations | `GET /api/v1/network/lines?direction=` | **Real** |
| Scheduled journeys with stops (`stationId`, scheduled arrival/departure, `platformId`, `isCalling`) | `GET /api/v1/timetable/journeys?date=` | **Real**, scheduled data only: no expected time, status, progress or freshness, and stops carry ids, not names |
| Live departures A to B (M-07) | `GET /api/v1/public/departures?from=&to=` | **Not built** (`add-live-distribution`, open) |
| Departures from a station / interchange (M-12) | `GET /api/v1/public/stations/{id}/departures?windowMinutes=` | **Not built** |
| Journey detail, live progress, platform change, correction (M-08, M-09, M-10) | `GET /api/v1/public/journeys/{id}` | **Not built** |
| Disruption guidance and next service (M-11) | inside the public journey view (`guidance`) | **Not built** |
| Live stream (SSE, `Last-Event-ID`, `reset`, 15 s heartbeat) | `GET /api/v1/public/stream?station=&journey=` | **Not built** |

Errors are RFC 9457 problem details (`application/problem+json`: `type`, `title`, `status`, `detail`,
`traceId`), for example a bad `date` returns `400`. Public responses are `Cache-Control: no-store`, so the app's
own cache decides what is shown offline. At the time of writing the running API's `stations` and
`timetable/journeys` return `[]` (no seed loaded), so the "real" slice is wired and works but shows empty
data until the backend seed is loaded; the mock provides the demo data.

**Consequence:** the first real slice is the stations list plus *timetable-derived* departures (scheduled
times only, shown honestly as "scheduled, live status not available"). Everything else is mocked behind
interfaces shaped after the OpenSpec `passenger-departures`, `disruption-guidance` and `live-event-streams`
specs. Those shapes have no wire examples yet, so mock DTOs are provisional **[A]** and must be re-checked
against `/openapi/v1.json` when `add-live-distribution` lands.

---

## 2. Architecture: clean architecture, feature first

Three layers inside each feature, with a shared `core/`.

```
lib/
  main.dart                  entry: ProviderScope + runApp
  app/                       MaterialApp.router, go_router config, theme wiring
  core/
    config/                  AppConfig (base URL, per-endpoint DataSourceMode), from --dart-define
    error/                   AppFailure (sealed), ProblemDetails
    network/                 dio client factory, problem-details interceptor, ApiClient
    sse/                     SseClient, SseEvent parser, reconnect with Last-Event-ID
    cache/                   CacheStore interface + shared_preferences impl, Cached<T> {value, savedAt}
    time/                    Asia/Colombo helpers, Clock provider, relative/minutes formatting
    theme/                   design tokens (colour, type, spacing), status styles
    a11y/                    shared semantic widgets: StatusChip, FreshnessBanner, PlatformBadge
  features/
    stations/                list stations, station picker
    departures/              next departures A to B, departures from a station (M-07, M-12)
    journey_detail/          calling stops, progress, platform change (M-08..M-10)
    disruption/              guidance + next service (M-11)  [may live inside journey_detail]
      each feature:
        domain/
          entities/          pure Dart, immutable
          repositories/      abstract interfaces
          usecases/          only where there is logic (e.g. DirectJourneysFilter)
        data/
          dto/               wire types, fromJson, toEntity()
          datasources/       remote (ApiClient) and local (CacheStore)
          repositories/      real impls, plus mock/ fakes with seeded data
        presentation/
          providers/         Riverpod providers and notifiers
          screens/ widgets/
test/                        mirrors lib/
```

**Dependency rule** (inward only): `presentation -> domain <- data`. `domain` imports nothing but Dart core and
`meta`/`collection`; no Flutter, no dio, no Riverpod. `data` imports `domain` and `core`. `presentation`
imports `domain` and `core`, and reaches `data` only through the provider that wires an interface to its
implementation (the composition root, `features/*/presentation/providers/*_wiring.dart` and
`core/config`). Features do not import each other's `data` or `presentation`; shared types
live in `domain` of the feature that owns them (stations are used by departures through the `Station` entity
and the `StationRepository` interface). Enforced by `test/architecture/dependency_rule_test.dart`, which
scans `import` lines and fails on a violation.

**Errors:** repositories return values or throw a typed `AppFailure` (sealed: `NetworkFailure`,
`TimeoutFailure`, `ServerFailure(problem)`, `NotFoundFailure`, `BadRequestFailure(problem)`,
`ParseFailure`, `RateLimitedFailure`, `UnknownFailure`). A dio interceptor maps `DioException` and
`application/problem+json` bodies to `AppFailure`; nothing outside `data` sees dio types. Notifiers expose
failures as `AsyncError`, which the UI renders as text (never a blank screen).

---

## 3. Riverpod design

- **Package:** `flutter_riverpod` 3.x **[Owner]**.
- **Plain providers, no code generation [A].** Hand-written `Provider`, `AsyncNotifierProvider`,
  `NotifierProvider`, `StreamProvider`, and `.family` where parameterised. Rejected: `riverpod_generator` +
  `build_runner`. Why: it adds a generated-file step to every change and to CI, the app has roughly 15
  providers so the boilerplate saving is small, and generated files make small, parallelisable work packages
  harder to review. It is a mechanical migration later if the owner prefers `@riverpod`; the provider names
  and shapes are chosen to make that migration trivial. DTOs are also hand-written (see §5).
- **Server state: `AsyncNotifier`** (`DeparturesController`, `StationsController`, `JourneyDetailController`),
  each wrapping a repository from a provider. State type is `AsyncValue<Freshness<T>>`-like: a small
  `Loaded<T>{ data, lastUpdated, isStale, source }` entity in `core`, where `source` is `live | cache`.
  Refresh is `ref.invalidate` / `refresh()`; pull-to-refresh calls it.
- **UI-only state** (selected from/to station, sort) in `Notifier`s; route parameters carry the ids so deep
  links work.
- **Wiring and mocks by provider overriding.** Every repository has a provider of its interface type:
  `final stationRepositoryProvider = Provider<StationRepository>(...)`. The default builds the real or mock
  implementation from `appConfigProvider`. Tests and the "all mock" build use
  `ProviderScope(overrides: [...])`. Widgets and notifiers never construct implementations.
- **Per-endpoint switching:** `AppConfig.modes` maps each endpoint (`stations`, `lines`, `timetable`,
  `departures`, `stationDepartures`, `journeyDetail`, `stream`) to `DataSourceMode { real, mock, auto }`.
  `auto` = real, fall back to the mock only on connection failure or `404`/`501`-style "not implemented", never on
  data or `4xx` errors (so bugs are not hidden). Defaults today: stations/lines/timetable `auto`; the rest
  `mock`. Setting is `--dart-define=SR_MODE_DEPARTURES=real` etc., and a `--dart-define=SR_ALL_MOCK=true`
  shortcut. When `add-live-distribution` lands a developer flips one default per endpoint. The mode in use is
  visible in debug builds as a small "Mock data" banner so demos are never misleading.
- **Lifecycle:** `autoDispose` for per-screen providers; the SSE stream provider is kept alive while a
  screen needing it is mounted; the app pauses the stream on `AppLifecycleState.paused`.
- **`Clock` provider** (`now()`) so staleness and relative times are testable.

---

## 4. Networking, SSE and caching

- **HTTP: `dio`** (interceptors for problem details and timeouts, cancel tokens, response typing). Base URL
  from config: `--dart-define=SR_API_BASE_URL`, default `http://10.0.2.2:5080` on Android emulator and
  `http://localhost:5080` elsewhere. Paths under `/api/v1`. 10 s connect/receive timeouts. Android needs
  `INTERNET` permission in the main manifest and cleartext traffic allowed for debug only.
  No cookies, no analytics, no passenger identifiers logged (S-20): the logger prints method, path and status only.
- **Typed client:** hand-written DTOs and an `ApiClient` per module for the six read endpoints. The frontend
  architecture doc plans a generated OpenAPI client; with six small endpoints that are not final yet,
  generation (and its build step) costs more than it saves **[A]**. A contract test parses a committed
  snapshot of the real responses (`test/fixtures/openapi_v1.json` for schema names and sample payloads captured
  from the API) so drift is caught. Revisit generation once `add-live-distribution` is stable (open question).
- **SSE (planned, built in WP12):** `SseClient` over a dio streaming `ResponseType.stream` request (not a
  third-party package: need `Last-Event-ID`, explicit reset handling and testability) with a ~60-line parser
  for `id:`, `event:`, `data:` and comment lines. Behaviour per `architecture.md` §6 and the `live-event-streams`
  spec: remember the last event id in memory (and in the cache), reconnect with `Last-Event-ID` and exponential
  backoff with jitter (1 s to 30 s); treat no frame or heartbeat in 40 s as dead; on a `reset` event discard
  the id and reload via the REST endpoints; events carry the full journey live view so they replace the
  matching item in state. Events apply by id, duplicates are ignored. Until the endpoint exists, a
  `MockLiveStream` emits scripted events (delay, platform change, cancellation) for the same interface.
  No polling fallback (frontend-architecture §3), only pull-to-refresh.
- **Local cache (offline last-known, M-02):** `CacheStore` interface (`read(key)`, `write(key, json, savedAt)`)
  with a `shared_preferences` implementation **[A]** (data is a day of nine stations, small). A repository
  decorator `CachingRepository` writes every successful response and, on a `NetworkFailure`/`TimeoutFailure`,
  returns the cached value flagged `source: cache`, so the UI shows last-known data with a stale marker
  instead of an error. An error is shown only if there is no cache. Cache entries keyed by endpoint + params,
  with the service date; entries from a previous day are dropped.
- **Stale threshold [A]:** the backend leaves the client threshold undefined (design Open Questions). Default
  **2 minutes** with the app in the foreground for items with `lastUpdatedAt`, and **any cache-sourced data is
  stale**; one constant in `core/config`, flagged for the owner's decision.

## 5. Time zone

All API timestamps are instants (`date-time`). Display always uses **Asia/Colombo** (UTC+5:30, no DST),
independent of the phone's time zone, using the `timezone` package (`tz.getLocation('Asia/Colombo')`
initialised at startup) plus `intl` for formatting (24-hour `HH:mm`, matching a rail board). Domain entities
store `DateTime` in UTC; conversion happens only in presentation formatters in `core/time`. "Service date" and
"today" are computed in Asia/Colombo (the `date` query parameter is a Colombo date). Tests inject a fixed
`Clock` and cover a phone set to another zone and the midnight boundary.

---

## 6. Screens and navigation (go_router)

`go_router` with typed-ish named routes; all routes public.

| Route | Screen | Requirement |
|---|---|---|
| `/` | **Find a train**: choose From and To stations, then Next departures | M-07 |
| `/stations` | Station list/picker (search by name, interchange marked in text) | EP-10 |
| `/departures?from=&to=` | **Next departures A to B**: scheduled and expected time, minutes early/late, platform, status text, freshness | M-07, M-02 |
| `/stations/:id/departures` | **Departures from a station** including interchange onward services | M-12 |
| `/journeys/:id` | **Journey detail**: calling stops (non-calling marked), live progress (last station, next station), platform-change banner with time known and "automatically published" label, correction note | M-08, M-09, M-10 |
| (section in journey detail, and tile in lists) | **Disruption guidance** and next service to the same destination | M-11 |

Shell: one `Scaffold` with a top app bar, a freshness banner area and a "Mock data" debug banner. No bottom
tab bar needed in MVP 1 (two entry points on the home screen). Unknown routes show a plain "Page not found"
screen with a way home.

## 7. Accessibility baseline (M-05) and freshness (M-02)

Built into the shared widgets so every screen inherits it:

- **Semantics:** every list row is a single merged `Semantics` node with a full sentence ("08:15 to Central,
  platform 2, 4 minutes late, last updated 08:11"); headings marked `header: true`; live regions
  (`liveRegion: true`) for freshness/platform-change banners; decorative icons excluded.
- **Text scaling:** honours the system scale up to at least 200%; no fixed-height text containers, layouts use
  `Wrap`/flexible rows and scroll; golden/widget tests at `textScaler` 2.0 must not overflow.
- **Contrast:** token pairs are at least 4.5:1 (7:1 for the large board text) and verified by a unit test that
  computes contrast for every foreground/background token pair; both light and high-contrast-friendly themes
  follow the system brightness.
- **No colour-only meaning:** status is always an icon **plus text** ("On time", "Delayed 6 min", "Cancelled",
  "Platform changed to 3"); `StatusChip` takes a status enum and renders text and shape; a test asserts every
  status renders non-empty text.
- **Touch targets** at least 48 dp; focus order follows reading order.
- **Freshness:** every live item shows "Updated 08:11" (Colombo time) via `FreshnessLabel`; `FreshnessBanner`
  appears at the top of a screen when data is cache-sourced or older than the stale threshold:
  "Showing last known information from 08:07. Check the station boards." (text, not colour). Items without
  live data (timetable-derived) say "Scheduled time, live information not available".
- **Provenance:** a platform change shows when it became known and the "automatically published" label (M-09,
  M-16); a corrected journey shows "Corrected at HH:mm".
- Automated check: `flutter_test` `meetsGuideline(androidTapTargetGuideline | labeledTapTargetGuideline |
  textContrastGuideline)` on each screen (EP-13 "passes an automated accessibility check").

## 8. Testing strategy

| Level | What | Tools |
|---|---|---|
| Unit | entities, use cases, DTO parsing from fixture JSON (including bad data), problem-details mapping, SSE parser, staleness/time helpers, contrast tokens | `flutter_test`, `mocktail` |
| Data | repositories against a fake dio adapter (real impl), caching decorator behaviour (hit/miss/stale), mode switching (`auto` fallback) | `http_mock_adapter` or a hand-rolled `HttpClientAdapter` |
| Provider | `ProviderContainer` with overridden repositories: loading -> data, error, refresh, stale flag, SSE event application | `flutter_riverpod`, fake `Clock` |
| Widget | each screen with overridden providers: loading, empty, error, stale, text scale 2.0, semantics labels, accessibility guidelines | `flutter_test` |
| Architecture | dependency-rule scan (§2) | plain Dart test |
| Integration (later) | one smoke test against the running API (opt-in via `--dart-define`, excluded from default `flutter test`) | `integration_test` |

Every work package ships its own tests and must leave `flutter analyze` and `flutter test` clean.
Coverage is not a gate, but each use case and repository has tests for its failure paths.

## 9. Lint and tooling

`flutter_lints` plus a stricter `analysis_options.yaml` **[A]**: `strict-casts`, `strict-inference`,
`strict-raw-types` enabled; rules `always_declare_return_types`, `avoid_dynamic_calls`, `prefer_final_locals`,
`prefer_const_constructors`, `require_trailing_commas`, `unawaited_futures`, `use_build_context_synchronously`,
`directives_ordering`, `avoid_print`, `public_member_api_docs` off (not worth it for an app). Rejected:
`very_good_analysis` (large rule set, high friction for AI-assisted small tasks). Formatting with
`dart format`; CI later.

Dependencies: `flutter_riverpod`, `go_router`, `dio`, `intl`, `timezone`, `shared_preferences`, `equatable`
(or plain `==` in entities; use `equatable` **[A]**), `meta`. Dev: `flutter_lints`, `mocktail`. No freezed, no
json_serializable, no build_runner.

---

## 10. Work packages

Each is sized for one OpenCode task; each ends with `flutter analyze` and `flutter test` clean. "Phase 2 now"
marks what I am building in this session.

| # | Package | Acceptance checks | Phase 2 now |
|---|---|---|---|
| WP1 | **Project structure, deps, lint, shell**: rename app, add dependencies, `analysis_options.yaml`, folder skeleton, `AppConfig`, theme tokens, `go_router` with Home and a not-found screen, `ProviderScope`, replace the scaffold counter test | analyze clean; app boots in a widget test showing the Home screen; router test for unknown route; architecture test present; `flutter build apk --debug` builds | yes |
| WP2 | **Core networking + errors**: `AppFailure`, `ProblemDetails` parser, dio client factory with interceptor, `ApiClient`, `Clock` + Asia/Colombo helpers, `CacheStore` interface + in-memory and shared_preferences impls, `Loaded<T>` | unit tests: each status/problem body maps to the right failure; timeout and no-network map; time conversion with another phone zone and midnight; cache round trip | yes |
| WP3 | **Stations slice (real + mock)**: `Station`, `Platform`, `Line` entities; `StationRepository` with `RemoteStationRepository` (real), `MockStationRepository` (9 stations on two lines, one interchange), mode wiring + `auto` fallback, caching decorator; `StationsController`; station list/picker screen | DTO parse test from a captured real payload; repo tests; provider tests; widget tests incl. empty, error, stale, text scale 2.0; works against the live API (returns empty list honestly) | yes |
| WP4 | **Next departures A to B**: `Departure` entity (scheduled and expected time, delay minutes, platform, status, lastUpdatedAt, source), `DeparturesRepository`; real `TimetableDeparturesRepository` (derives from `/timetable/journeys` joined with stations/platforms: calling at A then B, same journey, direction), `MockDeparturesRepository` (live-shaped with delay, cancellation, platform change); `DeparturesController`; Find a train + Departures screens with `StatusChip`, `FreshnessLabel`, `FreshnessBanner` | use-case tests (wrong direction, skipped stop, different lines gives empty list); widget tests (status shown as text, stale banner, 200% text); a11y guideline tests; works in mock mode and real mode | yes |
| WP5 | Departures from a station + interchange onward services (M-12) | list per station; interchange shows both lines and line names | later |
| WP6 | Journey detail: calling stops (M-10) + platform-change banner (M-09) with mock data | non-calling stops marked in text; platform change shows time and label | later |
| WP7 | Live progress (M-08): last/next station | derived from mock live view; semantic sentence | later |
| WP8 | Disruption guidance and next service (M-11) | guidance text shown for cancel/delay/early termination; none when undisrupted; corrected note | later |
| WP9 | Offline: `CachingRepository` on all repos, stale banner everywhere, day roll-over | airplane-mode widget test shows cached data with stale marker, never blank | later |
| WP10 | Accessibility pass: automated guidelines on every screen, 200% text, semantics review | guideline tests green | later |
| WP11 | SSE client + mock stream + live-update application to state | parser, reconnect, `Last-Event-ID`, `reset`, duplicate id ignore, backoff tests | later |
| WP12 | Switch endpoints to real as `add-live-distribution` lands; contract tests vs `/openapi/v1.json` | one mode default flip per endpoint; contract test green | when backend ships |

Stop point for this session: after WP4, if clean.

## 11. Open questions for the owner

1. **Stale threshold** for the passenger app (backend leaves it undefined). Proposed 2 minutes plus any cache-sourced data.
2. **OpenAPI client generation vs hand-written DTOs.** `frontend-architecture.md` says the app generates a client; I deferred it for six endpoints. Confirm or ask for generation.
3. **Riverpod without code generation** (plain providers). Confirm or ask for `riverpod_generator`.
4. **Station names and seed data** are the owner's; the mock's nine stations are placeholders. The running API currently returns empty lists.
5. **Target platforms:** Android first (emulator build verified); iOS cannot be built from Windows.
6. Visual design / tokens (EP-13): Material 3 default tokens with the contrast rules above until a Figma file exists.
