# SmartRail — MVP 1 Scope and Epics

This is the "separate, later artifact" that `moscow-scope.md` refers to. `moscow-scope.md` prioritises
requirements for the whole platform by necessity to the product; this document selects the subset of
that system-wide MoSCoW which forms the first deliverable MVP for a single engineer (brief §25).
`moscow-scope.md` stays the product scope; this document is the delivery selection. Items may be
delivered in a deliberately stripped-down form; where they are, the stripped shape is stated. Owner
decisions are recorded turn by turn in `docs/ai-usage/AI-INTERACTION-LOG.md` (rows 74 onward).

**Status:** agreed by the owner, living document.

---

## MVP 1 goal

MVP 1 builds the foundation first and proves it end to end. A small two-line network exists; the
Scheduler puts trains (services) onto it; each day's operational journeys run, with live progress
coming from the Guard's per-station departure logs; the Network Controller makes changes (delay,
cancellation, platform change) and those changes propagate from one source of truth to the passenger
app, a station platform display and the external catering provider, each copy carrying a timestamp and
provenance label, audited, and only possible for the right role.

**Why this is MVP 1**

- It addresses the root cause named in every discovery file: no shared integration layer that collects
  network state once and distributes it (M-01).
- Every later feature only reads or writes the network, the schedule and the live state, so they are
  built and proven first.
- One flow exercises every brief §23 artifact (backend, operations web, passenger mobile, operational
  mobile, station experience, catering integration) with no padding.
- It satisfies brief §25: small and coherent, but not reduced to a single application.
- It gives the brief §27 demonstration a story: a change made at one point, traced to every channel and
  to the audit record.

---

## Owner decisions for MVP 1

| Decision | Effect on scope |
|---|---|
| Foundation first: the network, scheduling, live running and Controller changes are the base of MVP 1 | Scheduler plan tooling (M-20) is in MVP 1 rather than running on fixture data only |
| Two lines, not one, sharing at least one interchange station | Real onward connections (M-12) are possible; the domain model is multi-line from day one |
| Live updates come from the Guard app's per-station departure logs | M-24 and M-23 are in; M-08 progress is derived from guard logs plus the timetable |
| The 24-hour Scheduler/Controller boundary is enforced server-side now | The window lock from M-19 is in; the handoff workflow of M-19 is deferred |
| The Station Master (platform manager) is deferred | M-17, M-18 and C-08 are deferred to a later MVP; the Network Controller can still make platform changes |
| Catering is delivered as an integration capability external providers connect to, not as passenger ordering | M-26 is in, stripped down; M-27 (Should) and all ordering are out |
| Features may be stripped-down versions rather than the full described capability | Each item below states its MVP 1 shape |
| Crew work-device location tracking stays out of MVP 1 | M-08 progress comes only from guard departure logs plus the timetable |
| No server push notifications in MVP 1 | M-13 alerts are local device notifications only; server push, SMS and preferences (S-01) are out |
| Each line has 5 stations | The network is two 5-station lines sharing one interchange station |
| Staff accounts are created in the system rather than only seeded; the owner decides what seed data is loaded | A System Admin role with basic account management is in MVP 1 (part of M-04) |
| Passengers can create an optional account; live information and journey planning stay open without one, and saving a journey needs an account | Passenger self-registration, sign-in, account details, journey planning across both lines and saved journeys are in MVP 1 (M-06, M-12, S-03); no live-information feature requires sign-in |

---

## The MVP 1 network

- Two lines, each running in both directions, with **5 stations per line**, sharing one interchange
  station (9 distinct stations in total). Station names and the rest of the seed data are decided by the
  owner.
- Each station has one or more numbered platforms.
- The network itself (lines, stations, platforms, station order) is loaded as seed data in MVP 1. The
  Scheduler adds services onto it. A UI for editing network reference data is deferred.
- The domain model supports more lines without change.

---

## Roles in MVP 1

| Role | Surface | Does in MVP 1 |
|---|---|---|
| Scheduler | Operations web | Creates and edits services, stopping patterns, times, running days, platform allocation and crew assignment for journeys more than 24 hours ahead |
| Network Controller | Operations web | Sees today's live picture; publishes delay, cancellation and platform change; issues corrections; sends operational notices to a journey's crew; owns everything inside the 24-hour window |
| Guard | Crew mobile app | Logs the actual departure at every station stop, with a delay cause from a fixed list; works offline |
| Driver | Crew mobile app (read-only view) | Sees assigned journey, stops and notices; acknowledges notices with one tap |
| Auditor | Operations web (read-only) | Views the audit record |
| System Admin | Operations web | Creates and deactivates staff accounts and assigns each a role from the fixed catalogue; sees no operational, schedule or passenger data |
| Passenger | Passenger mobile app, station display | Reads live information without an account; may create an optional passenger account to sign in, see account details, and plan and save journeys |
| Catering provider | Machine client (API key) | Pulls journey data and receives pushed status changes |

This satisfies brief §9 — operations are split across distinct roles, not a single Administrator. Staff
accounts are created by the System Admin in the system; the owner decides which accounts, if any, are
included in the seed data. The System Admin manages accounts only and cannot act as an operational role.

---

## MoSCoW items in MVP 1

Three statuses are used below. **In** — built as described. **Stripped** — built in the reduced shape
stated. **Deferred** — not in MVP 1, with the reason stated.

| ID | Capability (short) | MVP 1 status | MVP 1 shape or reason deferred | Epic |
|---|---|---|---|---|
| **M-01** | Shared real-time layer | In | One backend domain model and event stream; every client reads the same state over a real-time channel | EP-01, EP-09 |
| **M-02** | Freshness and offline honesty | In | "Last updated" on every live item; clients keep last-known state with a stale marker; no blank screens | EP-07, EP-09, EP-10, EP-11 |
| **M-03** | Audit record | Stripped | Append-only, hash-chained table of every staff state change and every denied cross-role attempt; Auditor read-only list. Verified exports and SoD reports (S-15) deferred | EP-03 |
| **M-04** | Role-based access, separation of duties | Stripped | Fixed role catalogue enforced at the API for the MVP 1 roles; staff accounts created and deactivated by the System Admin, one role per account. Admin governance (S-14) deferred | EP-02 |
| **M-05** | Accessibility baseline | Stripped | Text-first, semantic structure, contrast, scalable text, nothing by colour alone, on every passenger surface; every change is text so it has a visual twin. Formal audit and user testing (S-10) deferred | EP-13 |
| **M-06** | Public info without account | In | Passenger app and display use public endpoints for all live information. Passengers may also create an optional account (owner decision) to sign in, see account details, and plan and save journeys; passenger accounts are separate from staff roles | EP-02, EP-10 |
| **M-07** | Next departures | In | From station A to station B: scheduled and expected time, minutes early/late, platform, lifecycle status, freshness | EP-10 |
| **M-08** | Live train progress | Stripped | Last station passed, next station and ETA derived from the guard's logged departures plus the timetable. Crew work-device location is out of MVP 1 (owner decision) | EP-07 |
| **M-09** | Platform change on every surface | In | Flagged prominently with the time it became known, on app and display | EP-08, EP-10, EP-11 |
| **M-10** | Calling stops | In | Full stop list per journey | EP-10 |
| **M-11** | Disruption "what to do next" | Stripped | Template text per disruption type plus the next service to the same destination. Full recovery alternatives (S-05) deferred | EP-10 |
| **M-12** | Available connections | Stripped | Onward departures from any station, including the interchange to the other line, plus a journey planner from any station to any other (direct, or one change at the interchange, offering only changes with enough time). Recovery alternatives during disruption (S-05) deferred | EP-10 |
| **M-13** | Proactive alert | Stripped | Watch a journey on the device, no account; local notification when a change for it arrives. Server push, SMS and preferences (S-01) deferred (owner decision) | EP-10 |
| **M-14** | Station surface | Stripped | One platform display (browser kiosk), large high-contrast text, same data and stale behaviour as the app. Interactive terminal deferred | EP-11 |
| **M-15** | Operations live view | Stripped | Controller's view of today's journeys with status, owner and freshness. Incidents and restrictions deferred | EP-08 |
| **M-16** | Structured status publication | Stripped | Delay, cancellation and platform change, with actor, timestamp and an "automatically published" provenance label, propagated to every channel and audited. Skipped stop and early termination deferred to MVP 2 | EP-08, EP-09 |
| **M-17** | Platform availability lifecycle | Deferred | Station Master deferred by the owner | — |
| **M-18** | Incident logging | Deferred | Station Master deferred by the owner; not needed for the MVP 1 flow | — |
| **M-19** | Controller to Scheduler handoff, 24-hour window | Stripped | Only the fixed 24-hour window, enforced server-side: the Scheduler cannot edit any journey inside 24 hours and in-window journeys are hidden from the Scheduler's editable view. Handoff records, deadlines, escalation and accept/reject deferred | EP-05 |
| **M-20** | Plan reference data and scheduling | Stripped | Network seeded; Scheduler creates services, stopping patterns, times, running days, platform allocation and crew assignment (driver and guard) outside the 24-hour window; daily operational journeys generated from them. Reference-data editing UI and restriction guard deferred | EP-04, EP-05 |
| **M-21** | CIO messaging tooling | Deferred | Trilingual draft-and-approve authoring is a product of its own; MVP 1 proves only the auto-published path and its label | — |
| **M-22** | Driver app | Stripped | Read-only assigned journey, stops and Controller notices with one-tap acknowledgement. Delay-cause reports and passive location deferred | EP-06 |
| **M-23** | Offline queued writes | Stripped | Guard departure logs queued on device with client-generated ID and original timestamp, synced idempotently, queue status visible. Other frontline writes later | EP-06 |
| **M-24** | Guard app | Stripped | Departure logged at every station stop with a delay cause from a fixed list; never blocks departure. Full stop-state model and location verification deferred | EP-06 |
| **M-26** | Catering integration | Stripped | Authenticated (API key), least-privilege provider API: pull journey, station, platform and arrival data; receive pushed status-change webhooks with sequence numbers, duplicate handling and retry; a small simulated provider app consumes them. Menu feed optional; order create/callbacks and passenger data sharing deferred with M-27 | EP-12 |
| **C-08** | Temporary station closure | Deferred | Follows the Station Master model, deferred with M-17 | — |
| **S-12** | Corrective re-issue | In | A correction is a new event referencing the original; the original is kept; the correction reaches every channel that received the original, including the provider | EP-08 |
| **S-20** | Privacy and data classification | Stripped | Data classification document plus enforcement that no passenger PII enters audit or diagnostic logs. Retention and purge jobs deferred | EP-14 |

**Must items in MVP 1: 24** (In or Stripped). **Deferred: 4** (M-17, M-18, M-21, C-08). The M-19
handoff workflow is also deferred even though its window lock is in, so M-19 counts as Stripped above.

**Should items pulled into MVP 1 (stripped)**

| ID | Capability (short) | MVP 1 shape | Epic |
|---|---|---|---|
| **S-19** | Observability | Structured logs without passenger PII, health endpoint, publish-to-channel latency and offline sync backlog metrics | EP-17 |
| **S-03** | Saved journey | A planned journey (direct, or one change at the interchange) saved to the passenger's optional account rather than the device, showing the next trains with live status, platform and freshness. One-tap watch from a saved journey and "check before you travel" deferred | EP-10 |
| **S-21** | Scheduler plan tooling | Basic list/timetable view of upcoming services outside the 24-hour window. Draft-to-approved workflow and concurrency control deferred | EP-05 |

---

## Epics

Epics are listed in suggested build order. Each states its goal, the surfaces it touches, what is in
MVP 1, what is out, and when it is done.

### EP-01 Platform foundation and architecture

**Goal:** build the backend foundation every other epic depends on: one domain model and persistence
layer for the railway's state, a domain event stream that records every change once, and a real-time
channel that pushes those changes to every connected client, so that all surfaces read the same single
source of truth. **Surfaces:** backend.

**In MVP 1:**
- Service structure
- Database and migrations
- Domain event log
- Real-time push channel (WebSocket or SSE, decided in architecture)
- Seed-data mechanism
- CI pipeline

**Out of MVP 1:** multi-region, scaling work. **Done when:** an event written once is received by two
different subscribed clients.

### EP-02 Identity and access control

**Goal:** give each kind of user the access their work needs and no more. Staff sign in and can act only
within their own role, enforced by the server on every request rather than only hidden in the interface.
Staff accounts are created and deactivated in the system by a System Admin who manages accounts but
cannot see or change operational data. Passengers read public railway information without an account,
and may create an optional passenger account of their own, kept separate from staff roles. The catering
provider connects with its own API key. **Surfaces:** backend, operations web, crew app.

**In MVP 1:**
- Staff login
- Fixed role catalogue (Scheduler, Network Controller, Guard, Driver, Auditor, System Admin)
- Staff account management by the System Admin: create, deactivate and assign one role per account
- Seed data for accounts as decided by the owner
- API-level authorisation on every write
- Anonymous read for public data
- Optional passenger accounts: self-registration, sign-in and sign-out, with password reset and account deletion as Should
- Passenger accounts separate from staff roles: a passenger token is refused on every staff endpoint, and the System Admin neither manages nor sees passenger accounts
- API keys for the catering provider
- Denied attempts recorded via EP-03

**Out of MVP 1:** four-eyes approval for privileged roles, time-bound role assignments, access-request
workflow, single sign-on, admin self-service password reset. **Done when:** a System Admin can create a
staff account that signs in with only its assigned role, every write endpoint rejects a wrong-role token
with 403, and both the account change and the denial are audited; and a passenger can register and sign in
without gaining access to any staff endpoint.

### EP-03 Audit record

**Goal:** keep a tamper-evident, append-only record of every state-changing action taken by staff,
including attempts that were denied, so that it can be proven who did what, in which role and when,
without recording any passenger data. **Surfaces:** backend, operations web (Auditor view).

**In MVP 1:**
- Append-only, hash-chained audit table
- Actor, role, before and after values, server timestamp
- Denied attempts
- No passenger data
- Auditor read-only list, filterable by actor and time

**Out of MVP 1:** verified audit exports, separation-of-duties reports, external anchoring of the hash
chain. **Done when:** tampering with a stored row is detected by the chain check.

### EP-04 Network model

**Goal:** model the railway network the system runs on: two lines of five stations each, running in both
directions, with their stations, numbered platforms and station order, joined at a shared interchange
station. The network
is loaded as seed data, and the model allows more lines to be added without changing it.
**Surfaces:** backend.

**In MVP 1:**
- Lines
- Stations
- Platforms
- Station order per line and direction
- The interchange station
- Seed data

**Out of MVP 1:** an interface for editing network reference data, station accessibility data (step-free
access, lifts). **Done when:** both lines and the interchange are queryable through the public API.

### EP-05 Scheduling and the 24-hour boundary

**Goal:** let the Scheduler put trains onto the network by defining services, their stopping patterns,
times, running days, platforms and crew, and generate each day's operational journeys from them. The
server enforces a fixed 24-hour boundary: any journey within 24 hours belongs to the Network Controller
and cannot be edited by the Scheduler. **Surfaces:** operations web.

**In MVP 1:**
- Create and edit services on a line and direction
- Stopping patterns (all stops or limited)
- Times per stop
- Running days calendar
- Platform per stop
- Driver and guard assignment
- Daily operational journey generation
- Server-side 24-hour lock, and hiding in-window journeys from the Scheduler's editable view
- Basic upcoming-timetable view

**Out of MVP 1:** a Controller-to-Scheduler handoff workflow, restriction records, a draft-to-approved
workflow, bulk import. **Done when:** a service created by the Scheduler appears as tomorrow's journeys,
and an edit to a journey inside 24 hours is rejected by the API.

### EP-06 Crew mobile app (Guard and Driver)

**Goal:** give train crew an operational mobile app that is also the source of live running data. The
Guard logs the actual departure at every station stop, even without connectivity, and the logs sync
exactly once, with their original times, when the connection returns. The Driver sees the assigned
journey and acknowledges the Network Controller's notices. **Surfaces:** crew mobile app.

**In MVP 1:**
- Guard sees their assigned journey and stops
- Log a departure at each stop, with an optional delay cause from a fixed list
- Offline queue with client-generated ID and original timestamp
- Idempotent sync
- Visible queue status
- Driver: read-only journey and notices, with one-tap acknowledgement

**Out of MVP 1:** crew device location tracking, a full stop-state model, driver delay reports, any
safety-event capture (radio remains the safety channel). **Done when:** departures logged offline sync
exactly once on reconnect with original timestamps.

### EP-07 Live running state

**Goal:** turn the Guard's departure logs and the timetable into the live state of every journey: which
station the train last left and which is next, its expected times at every later stop, how early or late
it is running, whether it is scheduled, running, cancelled or completed, and how fresh that information
is. **Surfaces:** backend.

**In MVP 1:**
- Actual departure per stop
- Expected times at downstream stops
- Minutes early or late
- Last station passed and next station
- Lifecycle status (scheduled, running, cancelled, completed)
- Freshness timestamp per journey

**Out of MVP 1:** predictive arrival estimates beyond the schedule plus reported delay, GPS tracking from
the train itself. **Done when:** logging a late departure updates expected times at every downstream
stop.

### EP-08 Network Controller operations

**Goal:** give the Network Controller a live view of today's railway and the tools to change it: publish
a delay, cancellation or platform change that goes out immediately and is labelled as automatically
published, correct a wrong change without erasing the original, and send operational notices to a
journey's crew. **Surfaces:** operations web.

**In MVP 1:**
- Live view of today's journeys with status, owner and freshness
- Publish delay, cancellation and platform change
- Provenance label
- Correction that keeps the original
- Operational notice to a journey's crew

**Out of MVP 1:** skipped-stop and early-termination changes (MVP 2), incident management, human-authored
passenger messaging, handoffs to the Scheduler. **Done when:** a published change and its correction are
both visible on every channel and in the audit.

### EP-09 Live update distribution

**Goal:** make one change reach every channel. Each journey or disruption event is fanned out to the
passenger app, the station display, the crew app and the catering provider; every client keeps its
last-known state, marks it as stale when it can no longer confirm it, and catches up when it reconnects.
**Surfaces:** backend, all clients.

**In MVP 1:**
- Fan-out of journey and disruption events to the passenger app, the display, the crew app and the provider webhook dispatcher
- Client-side last-known cache
- Stale marker after a defined threshold **[A]**
- Reconnect and catch-up

**Out of MVP 1:** SMS and server push notifications, per-user notification preferences. **Done when:** a
client that was offline catches up to the current state on reconnect and shows stale data as stale
meanwhile.

### EP-10 Passenger mobile app

**Goal:** help passengers answer their journey questions without needing an account: which train to take
and when it leaves, whether it is running late, where it is now, which stations it stops at, when it will
reach their station, whether its platform has changed, what to do when something goes wrong, and which
onward departures are available from another station, including the interchange. Passengers can plan a
journey between any two stations, including a change of line at the interchange, and are alerted when a
journey they are watching changes. A passenger who creates an optional account can sign in, see their
account details and save planned journeys to reopen on any device. **Surfaces:** passenger mobile app.

**In MVP 1:**
- Station-to-station departures
- Journey detail with calling stops and progress
- Platform change flag
- Disruption notice with the next service
- Departures from any station, including the interchange
- Watch a journey with a local notification
- Offline last-known with a stale marker
- Journey planner between any two stations, direct or with one change at the interchange
- Optional passenger account: register, sign in and out, account details, saved journeys

**Out of MVP 1:** food ordering, recovery alternatives during disruption, watches and alerts synced to
the account (watches stay on the device). **Done when:** a passenger can find a train, follow it live and
be alerted to its platform change without an account, and a signed-in passenger can save a journey
planned across both lines and reopen it on another device.

### EP-11 Station platform display

**Goal:** provide the station passenger-information display: a large, high-contrast platform screen that
shows the next departures for its station and platform from the same live data as the app, makes platform
changes prominent, and stays honest when the connection drops by showing last-known departures marked as
stale. **Surfaces:** station platform display.

**In MVP 1:**
- One platform display in kiosk mode, showing next departures for a chosen station and platform
- Platform change prominence
- Freshness and stale behaviour on connection loss

**Out of MVP 1:** interactive station terminal, audio output. **Done when:** the display keeps showing
last-known departures marked stale when disconnected and recovers on reconnect.

### EP-12 Catering provider integration

**Goal:** let an external catering provider, which remains a separate system, connect to SmartRail and
get the railway data it needs to coordinate food with train journeys: a narrow, key-authenticated API for
journeys, stations, platforms and arrival times, and pushed notifications of status changes that arrive
reliably, in order and without duplicates. **Surfaces:** provider API, simulated provider app.

**In MVP 1:**
- API-key authenticated narrow API for journeys, stations, platforms and scheduled and expected arrivals
- Webhook subscription for status changes, with sequence numbers, idempotency and retry
- A simulated provider app that consumes and displays them
- Optional: provider menu feed with per-item preparation time

**Out of MVP 1:** passenger food ordering, order status callbacks, sharing the passenger's name, phone or
order PIN, reconciliation snapshots and stale-feed alerting. **Done when:** a cancellation reaches the
simulated provider once, a replayed duplicate is ignored and an out-of-order event is handled by
sequence.

### EP-13 Design system and accessibility baseline

**Goal:** give every SmartRail surface one shared design language with accessibility built in from the
start: text-first information, sufficient contrast, scalable text and status never shown by colour
alone, so that every change shown on screen also has a readable text form. **Surfaces:** all user-facing
surfaces.

**In MVP 1:**
- Shared tokens (type, colour, spacing, status colours with text labels)
- Component rules for status and freshness
- Wireframes and key flows for each MVP surface

**Out of MVP 1:** a formal WCAG audit and testing with disabled users, a multi-language interface.
**Done when:** every MVP surface uses the shared tokens and passes an automated accessibility check.

### EP-14 Privacy and data classification

**Goal:** identify every kind of data the platform collects, classify it, and decide who may see it,
what goes into logs and what is shared with the catering provider, and enforce that no passenger
personal data enters audit or diagnostic logs. **Surfaces:** backend, documentation.

**In MVP 1:**
- Data classification table (public, operational, staff, passenger account, provider-shared)
- What goes in logs, what the provider receives, what each role can view; passenger account data (name,
  email, saved journeys) is visible only to that passenger
- Enforcement that no passenger PII enters audit or diagnostic logs

**Out of MVP 1:** retention and purge jobs. **Done when:** the classification document exists and a test
proves logs contain no passenger PII.

### EP-15 Security engineering

**Goal:** build security in from the start rather than adding it at the end: model the main threats to
the MVP surfaces, record the important security decisions, implement a control for each top risk, and
prove those controls with security tests. **Surfaces:** all.

**In MVP 1:**
- Basic threat model for the MVP surfaces
- Security decisions record
- Controls (authorisation, input validation, API-key handling, rate limiting on public and provider endpoints, passenger password storage and sign-in rate limiting **[A]**)
- Security tests

**Out of MVP 1:** penetration testing, security event monitoring (SIEM). **Done when:** the threat
model's top risks each map to an implemented control and a test.

### EP-16 Quality engineering

**Goal:** show a structured approach to quality: a test strategy, functional test scenarios, unit and API
tests, and the five most critical end-to-end flows automated and running in CI, with defects logged and a
final quality assessment. **Surfaces:** all.

**In MVP 1:**
- Test strategy
- Functional test scenarios
- Unit and API tests
- The five automated end-to-end flows below
- Defect log
- Quality assessment

**Out of MVP 1:** load and performance testing, formal accessibility testing with disabled users.
**Done when:** the five E2E flows run green in CI.

### EP-17 Observability and operations

**Goal:** make the platform operable in production: structured logs that contain no passenger personal
data, a health endpoint, and metrics that show how quickly a change reaches each channel, how many guard
logs are waiting to sync, and whether notifications to the provider are failing. **Surfaces:** backend.

**In MVP 1:**
- Structured logs without passenger PII
- Health endpoint
- Metrics for publish-to-channel latency, guard sync backlog and webhook delivery failures

**Out of MVP 1:** dashboards beyond a basic view, alert-fatigue controls. **Done when:**
publish-to-channel latency can be read for a live change.

---

## The five automated end-to-end flows

1. Scheduler creates a service, journeys are generated, and the journey appears in passenger app
   departures.
2. Guard logs departures while offline, reconnects, the logs sync exactly once with their original
   timestamps, and the passenger app and display show updated progress.
3. Network Controller publishes a platform change; the passenger app and display show it with its time
   and provenance label, and the simulated provider receives it.
4. Separation of duties: a Scheduler edit to a journey inside the 24-hour window is rejected by the API,
   and a Guard token attempting to publish a disruption gets 403; both denials are audited.
5. Network Controller publishes a wrong cancellation and corrects it; every channel, including the
   provider, shows the correction, and the audit keeps the original.

---

## Suggested build order

| Phase | Epics | Outcome |
|---|---|---|
| 1. Foundation | EP-01, EP-02, EP-03, EP-13 (tokens only) | Backend, auth and audit in place |
| 2. Network and schedule | EP-04, EP-05 | Two-line network with scheduled trains and daily journeys |
| 3. Live running | EP-06, EP-07 | Guard logs drive live journey state |
| 4. Control | EP-08 | Controller changes and corrections |
| 5. Distribution and channels | EP-09, EP-10, EP-11, EP-12 | Every channel receives the same changes |

Cross-cutting throughout: EP-14, EP-15, EP-16, EP-17.

---

## Deferred from MVP 1

| Item | Why deferred | Likely MVP |
|---|---|---|
| M-17, M-18, C-08 (Station Master) | Owner deferred the platform manager role | MVP 2 |
| M-19 handoff workflow | Window lock is enough to keep Scheduler and Controller apart for MVP 1 | MVP 2 |
| M-21 CIO messaging | Trilingual authoring workflow is a product of its own | MVP 2 |
| M-16 skipped stop and early termination | Three change types are enough to prove propagation | MVP 2 |
| Crew work-device location (M-08, M-22, M-24) | Owner chose guard departure logs as the live source | Later |
| M-27 passenger food ordering | Already Should; catering proven as an integration first | Later |
| Network reference-data editing UI (M-20) | Network seeded for MVP 1 | Later |
| Interactive station terminal (M-14 option, S-22) | One display is enough for the station artifact | Later |
| Server push and SMS notifications (S-01) | Owner decided alerts are local device notifications only in MVP 1 | Later |

---

## Open points

None at present. The four earlier open points (crew device location, server push, network size and staff
accounts) were settled by the owner and are recorded in **Owner decisions for MVP 1**. The seed data
(station names, platforms, staff accounts) is left to the owner to define.
