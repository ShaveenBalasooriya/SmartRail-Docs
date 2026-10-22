# Operations web: feature, page and component plan

Status: draft for the owner, 2026-10-22. Scope is the Operations web (`SmartRail-OpsWeb`) only.
Grounded in the Deskline blueprint, `docs/frontend-architecture.md` §3–§6, `docs/scoping/mvp-1-scope.md`
(M-02, M-05, M-09, M-15, M-16, S-12, EP-03, EP-08), `docs/architecture.md` §5–§6, the running API
(`/openapi/v1.json`, checked 2026-10-22) and the OpenSpec changes that will add the missing endpoints.
Items marked **[A]** are my assumptions where a spec is silent.

The foundation (commit `2a3d20d`) already has the shell, guards, transport, `QueryState`, `ConfirmDialog`,
the live board and a dev-only sign-in. This plan covers what comes next.

## 1. What the API offers today

| Endpoint | State | Used for |
|---|---|---|
| `GET /api/v1/timetable/journeys?date=` | **Real** | Board, journey detail (the list carries `stops`) |
| `GET /api/v1/network/stations`, `/stations/{id}` | **Real** | Station names, platforms, interchange flag |
| `GET /api/v1/network/lines?direction=` | **Real** | Line reference (not needed by a screen yet) |
| `GET /api/v1/identity/me` | **Real** (returns `staffId`, `username`, `role`) | Replaces the dev sign-in; needs a bearer token, which arrives with OIDC |
| `GET /api/v1/operations/journeys`, `/journeys/{id}` | Not built (`add-controller-operations`) | Live board status and freshness, detail with change history |
| `POST /api/v1/operations/journeys/{id}/delays`, `/cancellation`, `/platform-changes`, `/termination` | Not built | Publish forms |
| `POST /api/v1/operations/changes/{id}/corrections` | Not built | Corrections |
| `GET /api/v1/audit/entries`, `/entries/{id}`, `/integrity` | Not built (`add-audit-record`) | Audit view |
| `GET /api/v1/public/stream` (SSE, `Last-Event-ID`, `reset` event) | Not built (`add-live-distribution`) | Live updates |

There is no `GET` journey by id on the real API. The timetable list already contains each journey's stops, so
the detail page reads the list for the journey's date and picks the journey by id (real, no mock). It goes
away when `GET /operations/journeys/{id}` lands.

## 2. Real versus mocked

A mock is a second implementation of the same service function, in `features/<domain>/api/<name>.mock.ts`.
The service file picks one through `shared/api/mockSwitch.ts`, which reads one Vite flag per endpoint group,
for example `VITE_USE_REAL_OPERATIONS=true`. The default for an endpoint that does not exist yet is the mock;
setting the flag switches that group to the real API with no other change. Signatures and wire types are
identical, so hooks, components and pages never know. Tests mock at the service module.

**Deviation from `docs/frontend-architecture.md` §6**, which names Mock Service Worker: MSW is not installed
and the README records "no MSW". A service-level mock adds no dependency, keeps the transport (`httpClient`)
exercised by its own tests, and is removed file by file as each backend change lands. If the owner prefers
MSW, only the `.mock.ts` files change.

| Backend change | Feature it swaps to real | Swap = |
|---|---|---|
| `add-controller-operations` | publishing, corrections, journey live state and history | set `VITE_USE_REAL_OPERATIONS`, delete `*.mock.ts`, update wire types |
| `add-audit-record` | audit | same, `VITE_USE_REAL_AUDIT` |
| `add-live-distribution` | live updates | the hook already targets the real URL; remove the stub |
| `add-identity-access` | auth | replace `sessionService` with OIDC (authorization code + PKCE) |

## 3. Feature plan

Roles: `network_controller` reads and writes; `auditor` reads only; the guard has no web access. The UI hides
what a role may not do, the route guard blocks it, and a 403 from the API is shown and never assumed away.

### 3.1 `auth` (exists)

Purpose: identity and role. Pages: `/login`. Planned change: swap the dev session for OIDC when
`add-identity-access` lands; call `GET /identity/me` to hydrate `User`. No component changes.

### 3.2 `stations` (new, package A)

Purpose: network reference data so every other feature can show a station name and a platform number instead
of a UUID. Read by both roles. No page of its own in MVP 1 (a station list is not a story); it is a data
feature consumed by journeys and publishing.

- **types.ts**: `Station { id, code, name, isInterchange, platforms: Platform[], lines: StationLine[] }`,
  `Platform { id, stationId, number }`, `StationLine { id, code, name }`.
- **api/stations.ts** (real): `getStations(): Promise<Station[]>`, `getStation(id): Promise<Station>`.
- **hooks**: `useStations()` key `['stations']` with `staleTime: Infinity` style long stale time (reference
  data changes rarely; **[A]** 10 minutes), `useStation(id)` key `['stations', id]`.
- **lib/stationLookup.ts** (pure): `indexStations(stations): StationIndex` and
  `stationName(index, id)`, `platformNumber(index, platformId)`, `platformsOf(index, stationId)`. Unknown ids
  return `null`, and callers render a visible fallback ("Unknown station"), never an empty cell.
- **components/StationName.tsx** (pure): `{ name: string | null; isInterchange?: boolean }` renders the name and an
  "Interchange" text tag (not colour alone).
- Four states: not a page; consumers treat a failed station load as degraded and still render ids with the
  fallback label plus a visible notice.

### 3.3 `journeys` (exists; add detail, package B)

Purpose: the board (M-15, exists) and the journey detail (stops, platforms, change history).

Pages and routes
- `LiveBoardPage` `/board` (exists). Planned: row link to detail, a stale marker per row when
  `isStale` is real (M-02), live updates (§3.7).
- `JourneyDetailPage` `/journeys/:journeyId?date=YYYY-MM-DD` (new). Both roles. Defaults `date` to today's
  railway date. `QueryState` four states with `notFoundTitle="Journey not found"`.

Components (pure, props in, JSX out)
- `JourneyDetailHeader { journey: JourneyDetail; freshness: ReactNode }`: service name, line, direction, date,
  status badge, scheduled departure.
- `StopList { stops: StopRow[] }`: ordered calling stops as a native `<table>` with captions and column
  headers: stop number, station, scheduled arrival, scheduled departure, platform. Non-calling stops are shown
  as "passes" in text. A platform that changed is flagged with text "Changed" and its change time (M-09), not
  colour alone.
- `ChangeHistory { changes: ChangeView[] }` (package D, §3.5): newest first; an empty history says "No changes
  published".
- `JourneyStatusBadge { status: JourneyStatus | null }`: wraps `Badge`; `null` renders "Not reported" (the
  board already does this inline; extract so board and detail share it).

Hooks
- `useJourney(journeyId, serviceDate)` key `['journeys', { serviceDate }, journeyId]`, built on the existing
  `['journeys', { serviceDate }]` list query: it reads the list through `useJourneys` data via
  `select` so the board and the detail share one fetch **[A]**; invalidation of `['journeys']` refreshes both.
- Later: `useJourneyChanges(journeyId)` key `['journeys', 'changes', journeyId]` (package D).

API services
- `journeys.ts` (exists): `getJourneys(date)` keeps its list mapping and adds `getJourney(id, date)` returning
  `JourneyDetail` (the list wire type already includes `stops`, so extend the local wire type; do not widen the
  board's `Journey`).
- Later (mock until real): `getOperationsJourney(id)` with status, expected times, `isStale`, `lastUpdatedAt`.

lib
- `journeyStops.ts` (pure): `toStopRows(journey, stationIndex)` joins stops with station names and platform
  numbers, orders by `sequence`, parses the sequence which the API may send as a number or a string.
- `railwayTime.ts` (shared, exists): all times formatted in the railway time zone.

types: `JourneyDetail extends Journey { serviceDate, stops: JourneyStop[] }`,
`JourneyStop { sequence, stationId, isCalling, scheduledArrival, scheduledDeparture, platformId }`.

Accessibility and freshness: heading hierarchy (`h1` journey name, `h2` per section); table has caption and
`scope` headers; a "Last updated" line using the shared `FreshnessNote`; when the detail refetch fails with
data on screen, keep the data and show a `role="alert"` note (same rule as the board).

### 3.4 `publishing` (new, package C; mock until `add-controller-operations`)

Purpose: the controller publishes delay, cancellation and platform change (M-16, NC-03, NC-04, NC-05) and, in
MVP 1, early termination (owner decision 2026-10-21; later package). Access: `network_controller` only; the
route group is guarded with `RequireRole`, and an auditor reaching the URL sees the Forbidden page.

Pages and routes (child of the journey, so the controller keeps context)
- `PublishDelayPage` `/journeys/:journeyId/publish/delay`
- `PublishCancellationPage` `/journeys/:journeyId/publish/cancellation`
- `PublishPlatformChangePage` `/journeys/:journeyId/publish/platform-change`
- Journey detail shows a `PublishActions` block (controller only) linking to these.

Components
- `PublishActions { journeyId: string; status: JourneyStatus | null; canPublish: boolean }`: the declarative
  action list (blueprint `RequestStatusActions` pattern): an `ACTIONS` table of `{ type, label, isAllowed(status) }`;
  renders nothing when `canPublish` is false. Which actions apply by status follows the spec (today's
  `scheduled` or `running` journeys only; cancellation only before departure).
- `DelayForm { stops: StopOption[]; isPending: boolean; error: string | null; onSubmit(input: DelayInput) }`
- `CancellationForm { journeyName: string; isPending; error; onSubmit() }`: a confirm-only form.
- `PlatformChangeForm { stops: StopOption[]; platformsByStop; isPending; error; onSubmit(input) }`
- `PublishConfirmDialog { change: ChangeDraft; isConfirming; error; onConfirm; onCancel }`: wraps
  `ConfirmDialog` and renders the provenance block: "Published immediately to every channel. Published by
  <name>, labelled 'Automatically published'. Cannot be undone; a mistake is fixed with a correction."
- `FormField { id, label, error?, hint?, children }` (shared, see §4): label/`htmlFor`, `aria-describedby`
  for hint and error, `role="alert"` on shown errors.

Forms follow the blueprint: one `useState` per field; validation derived during render into
`Record<Field, string | null>`; a `touched` map plus `attemptedSubmit` decide when an error shows;
`noValidate`; trim on submit; fields cleared only on success; copy changes while pending; unsaved-changes
`useBlocker` guarded by a ref set before `navigate()` on success; `ConfirmDialog` for the irreversible step.

Validation (pure, in `lib/validateChange.ts`, unit-tested):
- Delay: `minutes` integer 1–720 (spec); `fromStop` required and must be a calling stop. "Not yet departed"
  cannot be known until live state exists, so the server's 409 is mapped to a message **[A]**.
- Platform change: `stop` required (calling), `platformId` required, must belong to the stop's station
  (offered list is already filtered), must differ from the current platform.
- Cancellation: no input; the confirm dialog is the gate.

Hooks (mutations return the raw TanStack object; separate instances per form)
- `usePublishDelay(journeyId)`, `usePublishCancellation(journeyId)`, `usePublishPlatformChange(journeyId)`:
  on success invalidate `['journeys']` (prefix, so the board and detail refresh) and
  `['journeys', 'changes', journeyId]`.

API services (`publishing/api/`): `publishDelay(journeyId, input)`, `publishCancellation(journeyId)`,
`publishPlatformChange(journeyId, input)` returning `PublishedChange`. Mock in `*.mock.ts`: validates the same
rules, returns a change with `publishedBy`, `publishedAt` (now), `provenance: 'automatically_published'`, can be
forced to fail for tests. Wire bodies **[A]** (the spec states rules, not field names): delay
`{ fromSequence, minutes }`, platform change `{ sequence, platformId }`, termination `{ lastStopSequence }`.
They are confirmed against `openapi/v1.json` when the endpoint ships.

types: `ChangeType = 'delay' | 'cancellation' | 'termination' | 'platform_change' | 'correction'`;
`PublishedChange { id, type, journeyId, values, publishedBy, publishedAt, provenance, correctedByChangeId | null,
correctsChangeId | null }`; `DelayInput`, `PlatformChangeInput`.

Error mapping (user-facing, "short sentence + Please try again."): 400 validation, 403 "You do not have
permission to publish changes.", 404 journey not found, 409 conflict (already departed, already cancelled,
same platform) with the server's message surfaced because it is meaningful.

Accessibility: dialog follows the `ConfirmDialog` template (focus in and back, trap, Escape blocked while
confirming); every field has a visible label; errors are text with `aria-describedby`; success is announced
with a `role="status"` message on the detail page after redirect.

### 3.5 `corrections` (new, package D; mock)

Purpose: undo a wrongly published change everywhere while keeping the original (S-12, NC-06). Controller only.

- Surface: `ChangeHistory` on the detail page lists each change; a controller sees a "Correct" button on a
  change that is not a correction and not yet corrected. No separate page: the action is a `ConfirmDialog`.
- Components: `ChangeHistoryRow { change; canCorrect; onCorrect }` (shows type, values, publisher, publish
  time, provenance label, and text states "Corrected by …" / "Corrects …"); `CorrectionConfirmDialog`.
- Hook: `useCorrectChange(journeyId)` invalidates `['journeys']` and `['journeys', 'changes', journeyId]`.
- API: `correctChange(changeId): Promise<PublishedChange>`; 409 (already corrected, or target is a correction)
  and 404 are mapped to specific messages.
- lib: `canCorrect(change, role)` pure.

### 3.6 `audit` (exists as a stub; package E; mock)

Purpose: read-only record for the auditor (EP-03). Route `/audit` exists and is already `auditor`-only.

- Page `AuditPage`: filters (actor, from, to, outcome) kept in URL search params (refresh-safe, shareable),
  paginated list newest first, "no data" versus "no matches" empty split, integrity check button.
- Components: `AuditFilterBar`, `AuditTable` (native `<table>`; paginated server-side so no virtualization),
  `AuditEntryRow`, `AuditEntryDetail` (before and after values, link to the corrected entry), `IntegrityResult`.
  Outcome (`succeeded` / `denied`) is a text badge.
- Hooks: `useAuditEntries(filters, page)` key `['audit', 'entries', filters, page]`,
  `useAuditEntry(id)` key `['audit', 'entries', id]`, `useIntegrityCheck()` as a mutation (it is an action, not
  a cached read).
- API: `getAuditEntries`, `getAuditEntry`, `checkIntegrity`. Never displays passenger data (S-20).
- Route addition: `/audit/:entryId`.

### 3.7 `live` (new, package F)

Purpose: keep the board and detail live and honest about connection loss (M-02, EP-09).

- `shared/hooks/useEventStream(url, { onEvent, onReset })`: wraps the native `EventSource`. The browser sends
  `Last-Event-ID` itself on automatic reconnect, so the hook relies on that and exposes
  `status: 'connecting' | 'open' | 'reconnecting' | 'closed'` and `lastEventAt`. A `reset` event triggers
  `queryClient.invalidateQueries({ queryKey: ['journeys'] })`. A normal event invalidates the affected journey's
  keys (live view is re-read; the client never patches status itself, per "clients never infer status").
  Heartbeats are comments and update `lastEventAt` without an event callback; **[A]** no heartbeat for
  more than 45 seconds marks the stream stale.
- `features/journeys/hooks/useLiveJourneys()`: wires the hook to the keys above.
- Component `ConnectionStatus { status; lastEventAt }` in `shared/ui`: text plus icon: "Live", "Reconnecting…",
  "Offline, showing the last update at 10:42:05". Never a blank screen; data stays on screen.
- The stream URL is a configuration value; until `add-live-distribution` ships the hook is not mounted
  (feature flag) rather than faked.
- Polling fallback is not built, as `frontend-architecture.md` §3 states.

## 4. Shared pieces to extract

| Piece | From | To | Why |
|---|---|---|---|
| `FreshnessNote { updatedAt, isFetching, onRefresh }` | inline in `LiveBoardPage` | `shared/ui` | Board, detail and audit all show "Last updated" and refresh (M-02) |
| `FormField` | new | `shared/ui` | Every publish form needs label, hint, error wiring |
| `JourneyStatusBadge` | inline in board row | `features/journeys/components` | Board and detail |
| `useBlockUnsavedChanges(isDirty)` | new, from the blueprint's `useBlocker` + ref | `shared/hooks` | Three forms need it; keeps the ref-before-navigate rule in one place |
| `mockSwitch` | new | `shared/api` | One flag convention for every mock |
| `useEventStream`, `ConnectionStatus` | new | `shared/hooks`, `shared/ui` | Live board and detail |
| Role-aware `useCanPublish()` | new | `features/publishing/hooks` | One place for "controller may publish" |

Naming and conventions follow `docs/frontend-architecture.md` and the blueprint: `PascalCase.tsx`, one CSS
Module per component, `import type`, no enums, `@/` aliases, query keys hierarchical by entity.

## 5. Page and component tree

```
app/
  router.tsx                         routes below
  layout/{AppLayout,Navbar,Footer}   (exists)
  pages/{NotFound,Unauthorized,Forbidden,RouteError}   (exists)
features/
  auth/        pages/LoginPage                      components/LoginForm        api/session
  stations/    types · api/stations(.ts) · hooks/{useStations,useStation}
               lib/stationLookup · components/StationName
  journeys/    pages/{LiveBoardPage, JourneyDetailPage}
               components/{JourneyFilterBar, JourneyTable, JourneyTableRow,
                           JourneyDetailHeader, StopList, JourneyStatusBadge}
               hooks/{useJourneys, useJourney, useJourneyFilters, useLiveJourneys}
               api/journeys · lib/{filterJourneys, journeyStops} · types
  publishing/  pages/{PublishDelayPage, PublishCancellationPage, PublishPlatformChangePage}
               components/{PublishActions, DelayForm, CancellationForm, PlatformChangeForm, PublishConfirmDialog}
               hooks/{usePublishDelay, usePublishCancellation, usePublishPlatformChange, useCanPublish}
               api/{publish(.ts,.mock.ts)} · lib/validateChange · types
  corrections/ components/{ChangeHistory, ChangeHistoryRow, CorrectionConfirmDialog}
               hooks/useCorrectChange · api/corrections(.ts,.mock.ts) · lib/canCorrect
  audit/       pages/{AuditPage, AuditEntryPage}
               components/{AuditFilterBar, AuditTable, AuditEntryRow, AuditEntryDetail, IntegrityResult}
               hooks/{useAuditEntries, useAuditEntry, useIntegrityCheck} · api/audit(.ts,.mock.ts) · types
shared/
  api/{httpClient, client, ApiError, mockSwitch}
  components/{Button, Badge}
  ui/{QueryState, EmptyState, ErrorPage, ConfirmDialog, FilterBar, LoadingIndicator,
      FreshnessNote, FormField, ConnectionStatus}
  hooks/{useDebouncedValue, useBlockUnsavedChanges, useEventStream}
  lib/railwayTime · preferences/appearance
```

Routes

| Path | Guard | Page |
|---|---|---|
| `/login` | none | LoginPage |
| `/board` | authenticated | LiveBoardPage |
| `/journeys/:journeyId` | authenticated | JourneyDetailPage |
| `/journeys/:journeyId/publish/{delay,cancellation,platform-change}` | `network_controller` | publish pages |
| `/audit`, `/audit/:entryId` | `auditor` | audit pages |
| `*` | none | NotFoundPage |

## 6. Work packages and acceptance checks

Every package must leave `npm run lint`, `npx tsc -b`, `npm test` and `npm run build` green, ships with
Vitest + Testing Library tests, and is committed on `dev` with a `[feat]:` subject.

| # | Package | Real/mock | Acceptance |
|---|---|---|---|
| A | `stations` feature + shared `FreshnessNote`, `FormField`, `useBlockUnsavedChanges`, `mockSwitch` | stations real | Lookup returns names and platform numbers and a safe fallback for unknown ids; `FreshnessNote` replaces the board's inline block with no visible change; `FormField` wires label, hint and error; tests for lookup, components and the hook |
| B | Journey detail page and route, board rows link to it | journeys real | `/journeys/:id?date=` shows header, stops with station names and platform numbers, not-found state for an unknown id, kept data plus alert on a failed refresh; tests for `toStopRows`, `StopList`, the page's four states |
| C | Publish forms (delay, cancellation, platform change), `ConfirmDialog` with provenance, controller-only routes | **mock** | Derived validation with touched and attempted-submit; dialog shows provenance and blocks dismissal while confirming; failure keeps input and shows the mapped error; auditor gets Forbidden; unsaved-changes blocker; tests for validators, each form, the dialog flow with a mocked service |
| D | Change history and corrections | mock | History shows type, publisher, time, provenance and corrected state; Correct only for controllers on uncorrected non-correction changes; 409 mapped; tests |
| E | Audit view | mock | Filters in the URL, pagination, two empty states, integrity result, auditor-only; tests |
| F | Live updates and `ConnectionStatus` | mock stream in tests | Reconnect keeps data and shows "Reconnecting"; `reset` invalidates; tests with a fake `EventSource` |
| G | OIDC sign-in | real | Replaces the dev session; `GET /identity/me` hydrates the user |
| H | Swap mocks to real as each OpenSpec change lands | real | Flag flipped, `*.mock.ts` deleted, wire types checked against `openapi/v1.json` |

Phase 2 builds A, B, C in that order. **Reordering note:** the owner's list put the detail page first and the
stations data second, but the detail page must show station names and platform numbers, which come from the
stations data, so A (stations plus shared pieces) goes first.

## 7. Deviations from the blueprint

- `@/` path aliases (the blueprint recommends adding them) and Vitest tests (the blueprint has none).
- Service-level mocks behind a per-endpoint flag instead of MSW (§2).
- A native `<table>` for the stop list and audit list: both are short, paginated or bounded, so virtualization
  is not needed (blueprint reserves it for the one large table).
- `useBlocker` plus ref extracted into `useBlockUnsavedChanges` because three forms need it.

## 8. Open questions for the owner

1. Are request bodies for delay, platform change and termination `{ fromSequence, minutes }`,
   `{ sequence, platformId }` and `{ lastStopSequence }`? The specs state rules, not field names.
2. May the detail page read the timetable list to find a journey until `GET /operations/journeys/{id}` exists?
3. MSW or service-level mocks (§2)?
4. Should early termination ship with the first publish package or separately (it is in MVP 1 per the
   2026-10-21 decision)? This plan puts it after A–F.
5. Which stream does the Operations web use for live updates, the anonymous public stream, or a staff stream
   that also carries the controller-only fields?
