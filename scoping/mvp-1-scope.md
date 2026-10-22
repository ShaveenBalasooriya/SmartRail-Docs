# SmartRail — MVP 1 Scope and Epics

This is the "separate, later artifact" that `moscow-scope.md` refers to. `moscow-scope.md` prioritises
requirements for the whole platform by necessity to the product; this document selects the subset of
that system-wide MoSCoW which forms the first deliverable MVP for a single engineer (brief §25).
`moscow-scope.md` stays the product scope; this document is the delivery selection. Items may be
delivered in a deliberately stripped-down form; where they are, the stripped shape is stated. Owner
decisions are recorded turn by turn in `docs/ai-usage/AI-INTERACTION-LOG.md` (rows 74 onward).

**Status:** agreed by the owner, living document. Cut down to a lean core on 2026-10-19 (owner
decision): the timetable is seed data, three staff roles remain, and passenger accounts are deferred.

---

## MVP 1 goal

MVP 1 proves one thing end to end: a change to a running train is made once and reaches every channel.
A small two-line network and its timetable are loaded as seed data, and each day's journeys are
generated from that timetable. The Guard's per-station departure logs drive live progress; the Network
Controller publishes delays, cancellations and platform changes, and corrects mistakes; and every change
propagates from one source of truth to the passenger app, a station platform display, the Guard's app
and the external catering provider, each copy carrying a timestamp and provenance label, audited, and
only possible for the right role.

**Why this is MVP 1**

- It addresses the root cause named in every discovery file: no shared integration layer that collects
  network state once and distributes it (M-01).
- It is small enough for one engineer to build to a good standard (brief §25: "a smaller, coherent,
  well-engineered solution is preferable to a large collection of incomplete features").
- It still covers every brief §23 artifact (backend, operations web, passenger mobile, operational
  mobile, station experience, catering integration), each in its thinnest useful form.
- It gives the brief §27 demonstration a story: a change made at one point, traced to every channel and
  to the audit record.

---

## Owner decisions for MVP 1

| Decision | Effect on scope |
|---|---|
| Lean core (2026-10-19): MVP 1 is cut to the live-change loop; anything not needed to prove it moves to a later MVP | Scheduling tooling, three staff roles, passenger accounts and the journey planner are deferred; see below |
| The timetable is seed data, like the network (2026-10-19) | No Scheduler role in MVP 1; services, stops, times, platforms and guard assignment are seeded; today's journeys are generated from them. M-20 tooling, S-21 and the M-19 24-hour boundary are deferred |
| Staff roles in MVP 1 are Network Controller, Guard and Auditor, with seeded accounts (2026-10-19) | Scheduler, Driver (M-22) and System Admin are deferred; no account-management UI |
| No passenger accounts in MVP 1 (2026-10-19) | All passenger features are public; optional accounts, the journey planner and saved journeys (S-03) are deferred |
| Two lines, not one, sharing one interchange station | Onward departures from the interchange to the other line (M-12) are possible; the domain model is multi-line from day one |
| Each line has 5 stations | The network is two 5-station lines sharing one interchange station |
| Four change types: delay, cancellation, early termination, platform change; corrections kept | M-16 is stripped to four types (early termination added by the owner, 2026-10-21); S-12 corrections are in |
| Live updates come from the Guard app's per-station departure logs | M-24 and M-23 are in; M-08 progress is derived from guard logs plus the timetable |
| Crew work-device location tracking stays out of MVP 1 | M-08 progress comes only from guard departure logs plus the timetable |
| The Station Master (platform manager) is deferred | M-17, M-18 and C-08 are deferred; the Network Controller can still make platform changes |
| Catering is delivered as an integration capability external providers connect to, not as passenger ordering | M-26 is in, stripped down; M-27 (Should) and all ordering are out |
| No server push notifications in MVP 1 | Server push, SMS and preferences (S-01) are out |
| Features may be stripped-down versions rather than the full described capability | Each item below states its MVP 1 shape |

Further trims made with the lean cut and not separately confirmed by the owner are marked **[A]** where
they appear: the passenger watch and local alert (M-13), Controller notices to crew, signed webhooks, the
provider menu feed, and a smaller set of metrics.

---

## The MVP 1 network and timetable

- Two lines, each running in both directions, with **5 stations per line**, sharing one interchange
  station (9 distinct stations in total). Each station has one or more numbered platforms.
- The timetable is seed data too: services on each line and direction, their stops, times per stop,
  platform per stop, and the guard assigned. Each day's operational journeys are generated from it.
- Station names, the timetable and staff accounts are decided by the owner.
- No interface edits the network or the timetable in MVP 1. The domain model supports more lines and a
  later Scheduler without change.

---

## Roles in MVP 1

| Role | Surface | Does in MVP 1 |
|---|---|---|
| Network Controller | Operations web | Sees today's live picture; publishes delay, cancellation and platform change; issues corrections; owns every change to today's journeys |
| Guard | Crew mobile app | Sees assigned journeys; logs the actual departure at every station stop, with a delay cause from a fixed list; works offline; sees changes to their journey |
| Auditor | Operations web (read-only) | Views the audit record |
| Passenger | Passenger mobile app, station display | Reads live information; no account |
| Catering provider | Machine client (API key) | Pulls journey data and receives pushed status changes |

This satisfies brief §9: operations are split across distinct roles, not a single Administrator. Staff
accounts are seed data, one role per account. The Scheduler, Driver and System Admin roles return in a
later MVP.

---

## MoSCoW items in MVP 1

Three statuses are used below. **In** — built as described. **Stripped** — built in the reduced shape
stated. **Deferred** — not in MVP 1, with the reason stated.

| ID | Capability (short) | MVP 1 status | MVP 1 shape or reason deferred | Epic |
|---|---|---|---|---|
| **M-01** | Shared real-time layer | In | One backend domain model and event stream; every client reads the same state over a real-time channel | EP-01, EP-09 |
| **M-02** | Freshness and offline honesty | In | "Last updated" on every live item; clients keep last-known state with a stale marker; no blank screens | EP-07, EP-09, EP-10, EP-11 |
| **M-03** | Audit record | Stripped | Append-only, hash-chained table of every staff state change and every denied cross-role attempt; Auditor read-only list. Verified exports and SoD reports (S-15) deferred | EP-03 |
| **M-04** | Role-based access, separation of duties | Stripped | Fixed role catalogue (Network Controller, Guard, Auditor) enforced at the API; seeded staff accounts, one role per account. Account management and admin governance (S-14) deferred | EP-02 |
| **M-05** | Accessibility baseline | Stripped | Text-first, semantic structure, contrast, scalable text, nothing by colour alone, on every passenger surface. Formal audit and user testing (S-10) deferred | EP-13 |
| **M-06** | Public info without account | In | Passenger app and display use public endpoints only; there are no passenger accounts in MVP 1 (owner decision) | EP-10 |
| **M-07** | Next departures | In | From station A to station B: scheduled and expected time, minutes early/late, platform, lifecycle status, freshness | EP-10 |
| **M-08** | Live train progress | Stripped | Last station passed, next station and ETA derived from the guard's logged departures plus the timetable. Crew work-device location is out (owner decision) | EP-07 |
| **M-09** | Platform change on every surface | In | Flagged prominently with the time it became known, on app and display | EP-08, EP-10, EP-11 |
| **M-10** | Calling stops | In | Full stop list per journey | EP-10 |
| **M-11** | Disruption "what to do next" | Stripped | Template text per disruption type plus the next service to the same destination. Full recovery alternatives (S-05) deferred | EP-10 |
| **M-12** | Available connections | Stripped | Onward departures from any station, including the interchange to the other line. Journey planner and recovery alternatives (S-05) deferred | EP-10 |
| **M-13** | Proactive alert | Deferred **[A]** | Watch and local notification cut in the lean cut; passengers see changes when they open the app or look at the display | — |
| **M-14** | Station surface | Stripped | One platform display (browser kiosk), large high-contrast text, same data and stale behaviour as the app. Interactive terminal deferred | EP-11 |
| **M-15** | Operations live view | Stripped | Controller's view of today's journeys with status and freshness. Incidents and restrictions deferred | EP-08 |
| **M-16** | Structured status publication | Stripped | Delay, cancellation, early termination and platform change, with actor, timestamp and an "automatically published" provenance label, propagated to every channel and audited. Early termination (owner decision 2026-10-21) is limited to ending a running journey that cannot continue. Skipped stop deferred | EP-08, EP-09 |
| **M-17** | Platform availability lifecycle | Deferred | Station Master deferred by the owner | — |
| **M-18** | Incident logging | Deferred | Station Master deferred by the owner | — |
| **M-19** | Controller to Scheduler handoff, 24-hour window | Deferred | No Scheduler in MVP 1, so there is no boundary to enforce (owner decision) | — |
| **M-20** | Plan reference data and scheduling | Stripped | Network and timetable seeded (services, stops, times, platforms, guard assignment); daily journeys generated from the seeded timetable. Scheduler tooling deferred (owner decision) | EP-04 |
| **M-21** | CIO messaging tooling | Deferred | Trilingual draft-and-approve authoring is a product of its own; MVP 1 proves only the auto-published path and its label | — |
| **M-22** | Driver app | Deferred | Driver role deferred by the owner; the Guard's app is MVP 1's operational mobile app | — |
| **M-23** | Offline queued writes | Stripped | Guard departure logs queued on device with client-generated ID and original timestamp, synced idempotently, queue status visible | EP-06 |
| **M-24** | Guard app | Stripped | Departure logged at every station stop with a delay cause from a fixed list; never blocks departure. Full stop-state model and location verification deferred | EP-06 |
| **M-26** | Catering integration | Stripped | API-key authenticated, least-privilege provider API: pull journey, station, platform and arrival data; receive pushed status-change webhooks with sequence numbers, duplicate handling and retry; a small simulated provider app consumes them. Menu feed **[A]**, ordering and passenger data sharing deferred | EP-12 |
| **C-08** | Temporary station closure | Deferred | Follows the Station Master model, deferred with M-17 | — |
| **S-12** | Corrective re-issue | In | A correction is a new event referencing the original; the original is kept; the correction reaches every channel that received the original, including the provider | EP-08 |
| **S-20** | Privacy and data classification | Stripped | Data classification document plus enforcement that no passenger PII enters audit or diagnostic logs. Retention and purge jobs deferred | EP-14 |
| **S-19** | Observability | Stripped | Structured logs without passenger PII, health endpoint, publish-to-channel latency and webhook delivery failure metrics **[A]** | EP-17 |

**Must items in MVP 1: 19** (In or Stripped). **Deferred Musts: 6** (M-13, M-17, M-18, M-19, M-21,
M-22), plus C-08. **Should items pulled in: 3** (S-12, S-19, S-20).

---

## Epics

Epics are listed in suggested build order. Each states its goal, the surfaces it touches, what is in
MVP 1, what is out, and when it is done. EP-05 (Scheduling and the 24-hour boundary) is deferred and
its number is kept free so later documents can reuse it.

### EP-01 Platform foundation and architecture

**Goal:** build the backend foundation every other epic depends on: one domain model and persistence
layer for the railway's state, a domain event stream that records every change once, and a real-time
channel that pushes those changes to every connected client. **Surfaces:** backend.

**In MVP 1:** service structure; database and migrations; domain event log; real-time push channel;
seed-data mechanism; CI pipeline.

**Out of MVP 1:** multi-region, scaling work. **Done when:** an event written once is received by two
different subscribed clients.

### EP-02 Identity and access control

**Goal:** give each kind of user the access their work needs and no more. Staff sign in and can act only
within their own role, enforced by the server on every request. Passengers read public information
without signing in. The catering provider connects with its own API key. **Surfaces:** backend,
operations web, crew app.

**In MVP 1:** staff login; fixed role catalogue (Network Controller, Guard, Auditor); seeded staff
accounts, one role per account; API-level authorisation on every write; anonymous read for public data;
API key for the catering provider; denied attempts recorded via EP-03.

**Out of MVP 1:** account management by a System Admin, passenger accounts, four-eyes approval,
time-bound roles, single sign-on. **Done when:** every write endpoint rejects a wrong-role token with
403 and the denial is audited.

### EP-03 Audit record

**Goal:** keep a tamper-evident, append-only record of every state-changing action taken by staff,
including denied attempts, so that it can be proven who did what, in which role and when, without
recording any passenger data. **Surfaces:** backend, operations web (Auditor view).

**In MVP 1:** append-only, hash-chained audit table; actor, role, before and after values, server
timestamp; denied attempts; no passenger data; Auditor read-only list, filterable by actor and time.

**Out of MVP 1:** verified exports, separation-of-duties reports, external anchoring of the chain.
**Done when:** tampering with a stored row is detected by the chain check.

### EP-04 Network and timetable

**Goal:** model the railway the system runs on, two lines of five stations each, in both directions,
joined at a shared interchange, and the timetable that runs on it, all loaded as seed data, and generate
each day's operational journeys from that timetable. **Surfaces:** backend.

**In MVP 1:** lines; stations; platforms; station order per line and direction; the interchange
station; seeded services with stops, times, platforms and guard assignment; daily journey generation.

**Out of MVP 1:** the Scheduler role and any interface for editing the network or timetable, running-day
calendars **[A]**, the 24-hour boundary, station accessibility data. **Done when:** both lines, the
interchange and today's generated journeys are queryable through the public API.

### EP-06 Guard mobile app

**Goal:** give the Guard an operational mobile app that is also the source of live running data. The
Guard logs the actual departure at every station stop, even without connectivity, and the logs sync
exactly once, with their original times, when the connection returns. **Surfaces:** crew mobile app.

**In MVP 1:** Guard sees their assigned journeys and stops; logs a departure at each stop, with an
optional delay cause from a fixed list; offline queue with client-generated ID and original timestamp;
idempotent sync; visible queue status; changes to their journey shown live.

**Out of MVP 1:** the Driver view, Controller notices to crew **[A]**, crew device location, a full
stop-state model, any safety-event capture (radio remains the safety channel). **Done when:** departures
logged offline sync exactly once on reconnect with original timestamps.

### EP-07 Live running state

**Goal:** turn the Guard's departure logs and the timetable into the live state of every journey: last
station left and next station, expected times at every later stop, minutes early or late, lifecycle
status and freshness. **Surfaces:** backend.

**In MVP 1:** actual departure per stop; expected times at downstream stops; minutes early or late; last
station passed and next station; lifecycle status (scheduled, running, cancelled, completed, terminated); freshness
timestamp per journey.

**Out of MVP 1:** predictive arrival estimates, GPS tracking. **Done when:** logging a late departure
updates expected times at every downstream stop.

### EP-08 Network Controller operations

**Goal:** give the Network Controller a live view of today's railway and the tools to change it: publish
a delay, cancellation or platform change that goes out immediately labelled as automatically published,
and correct a wrong change without erasing the original. **Surfaces:** operations web.

**In MVP 1:** live view of today's journeys with status and freshness; publish delay, cancellation,
early termination of a running journey that cannot continue (owner decision 2026-10-21) and platform
change; provenance label; correction that keeps the original.

**Out of MVP 1:** notices to crew **[A]**, skipped stop, incident management,
human-authored passenger messaging. **Done when:** a published change and its correction are both
visible on every channel and in the audit.

### EP-09 Live update distribution

**Goal:** make one change reach every channel. Each journey or disruption event is fanned out to the
passenger app, the station display, the Guard's app and the catering provider; every client keeps its
last-known state, marks it stale when it can no longer confirm it, and catches up on reconnect.
**Surfaces:** backend, all clients.

**In MVP 1:** fan-out to the passenger app, the display, the Guard's app and the provider webhook
dispatcher; client-side last-known cache; stale marker after a defined threshold **[A]**; reconnect and
catch-up.

**Out of MVP 1:** SMS and server push notifications, per-user notification preferences. **Done when:** a
client that was offline catches up on reconnect and shows stale data as stale meanwhile.

### EP-10 Passenger mobile app

**Goal:** help passengers answer their journey questions without an account: which train to take and
when it leaves, whether it is late, where it is now, which stations it stops at, whether its platform has
changed, what to do when something goes wrong, and which onward departures leave from another station,
including the interchange. **Surfaces:** passenger mobile app.

**In MVP 1:** station-to-station departures; journey detail with calling stops and progress; platform
change flag; disruption notice with the next service; departures from any station, including the
interchange; offline last-known with a stale marker.

**Out of MVP 1:** passenger accounts, the journey planner, saved journeys, watch and alerts **[A]**, food
ordering, recovery alternatives. **Done when:** a passenger can find a train, follow it live and see its
platform change without an account.

### EP-11 Station platform display

**Goal:** a large, high-contrast platform screen that shows the next departures for its station and
platform from the same live data as the app, makes platform changes prominent, and shows last-known
departures marked stale when the connection drops. **Surfaces:** station platform display.

**In MVP 1:** one platform display in kiosk mode for a chosen station and platform; platform change
prominence; freshness and stale behaviour on connection loss.

**Out of MVP 1:** interactive station terminal, audio output. **Done when:** the display keeps showing
last-known departures marked stale when disconnected and recovers on reconnect.

### EP-12 Catering provider integration

**Goal:** let an external catering provider, which remains a separate system, connect to SmartRail and
get the railway data it needs: a narrow, key-authenticated API for journeys, stations, platforms and
arrival times, and pushed status changes that arrive reliably, in order and without duplicates.
**Surfaces:** provider API, simulated provider app.

**In MVP 1:** API-key authenticated narrow API; webhook subscription for status changes, with sequence
numbers, idempotency and retry; a simulated provider app that consumes and displays them.

**Out of MVP 1:** menu feed **[A]**, signed webhooks **[A]**, passenger food ordering, order callbacks,
sharing passenger data, reconciliation snapshots. **Done when:** a cancellation reaches the simulated
provider once, a replayed duplicate is ignored and an out-of-order event is handled by sequence.

### EP-13 Design system and accessibility baseline

**Goal:** one shared design language with accessibility built in: text-first information, sufficient
contrast, scalable text and status never shown by colour alone. **Surfaces:** all user-facing surfaces.

**In MVP 1:** shared tokens (type, colour, spacing, status colours with text labels); component rules
for status and freshness; wireframes and key flows for each MVP surface.

**Out of MVP 1:** formal WCAG audit and testing with disabled users, a multi-language interface.
**Done when:** every MVP surface uses the shared tokens and passes an automated accessibility check.

### EP-14 Privacy and data classification

**Goal:** classify every kind of data the platform holds, decide who may see it, what goes into logs and
what is shared with the catering provider, and enforce that no passenger personal data enters audit or
diagnostic logs. **Surfaces:** backend, documentation.

**In MVP 1:** data classification table (public, operational, staff, provider-shared); what goes in
logs, what the provider receives, what each role can view; enforcement that no passenger PII enters
logs.

**Out of MVP 1:** retention and purge jobs. **Done when:** the classification document exists and a test
proves logs contain no passenger PII.

### EP-15 Security engineering

**Goal:** model the main threats to the MVP surfaces, record the important security decisions,
implement a control for each top risk, and prove those controls with tests. **Surfaces:** all.

**In MVP 1:** basic threat model; security decisions record; controls (authorisation, input
validation, API-key handling, rate limiting on public and provider endpoints); security tests.

**Out of MVP 1:** penetration testing, SIEM. **Done when:** the threat model's top risks each map to an
implemented control and a test.

### EP-16 Quality engineering

**Goal:** a test strategy, functional test scenarios, unit and API tests, and the five most critical
end-to-end flows automated in CI, with defects logged and a final quality assessment. **Surfaces:** all.

**In MVP 1:** test strategy; functional test scenarios; unit and API tests; the five automated
end-to-end flows below; defect log; quality assessment.

**Out of MVP 1:** load and performance testing, formal accessibility testing. **Done when:** the five
E2E flows run green in CI.

### EP-17 Observability and operations

**Goal:** make the platform operable: structured logs without passenger personal data, a health
endpoint, and metrics showing how quickly a change reaches each channel and whether provider
deliveries are failing. **Surfaces:** backend.

**In MVP 1:** structured logs without passenger PII; health endpoint; metrics for publish-to-channel
latency and webhook delivery failures **[A]**.

**Out of MVP 1:** dashboards, alerting, guard sync backlog metric **[A]**. **Done when:**
publish-to-channel latency can be read for a live change.

---

## The five automated end-to-end flows

1. **Delay reaches every channel** — the Network Controller publishes a delay on a running journey;
   expected times at every later stop update in the passenger app, the station display and the Guard's
   app, and the simulated provider receives the event.
2. **Offline guard logs sync once** — the Guard logs departures while offline, reconnects, the logs sync
   exactly once with their original timestamps, and the passenger app and display show updated progress.
3. **Platform change everywhere** — the Network Controller publishes a platform change; the passenger
   app and display show it with its time and provenance label, and the simulated provider receives it.
4. **Separation of duties denials** — a Guard token attempting to publish a disruption and an Auditor
   token attempting any change both get 403; both denials are audited.
5. **Wrong cancellation corrected** — the Network Controller publishes a wrong cancellation and corrects
   it; every channel, including the provider, shows the correction, and the audit keeps the original.

---

## Suggested build order

| Phase | Epics | Outcome |
|---|---|---|
| 1. Foundation | EP-01, EP-02, EP-03, EP-13 (tokens only) | Backend, auth and audit in place |
| 2. Network and timetable | EP-04 | Seeded two-line network with today's journeys |
| 3. Live running | EP-06, EP-07 | Guard logs drive live journey state |
| 4. Control | EP-08 | Controller changes and corrections |
| 5. Distribution and channels | EP-09, EP-10, EP-11, EP-12 | Every channel receives the same changes |

Cross-cutting throughout: EP-14, EP-15, EP-16, EP-17.

---

## Deferred from MVP 1

| Item | Why deferred | Likely MVP |
|---|---|---|
| Scheduler role, scheduling tooling and the 24-hour boundary (M-19, M-20 tooling, S-21, EP-05) | Timetable is seed data in the lean cut (owner decision) | MVP 2 |
| Driver app (M-22) and Controller notices to crew | Driver deferred by the owner; the Guard's app covers operational mobile | MVP 2 |
| System Admin and staff account management (part of M-04) | Staff accounts are seeded (owner decision) | MVP 2 |
| Optional passenger accounts, journey planner, saved journeys (M-06 account, M-12 planner, S-03) | Deferred by the owner in the lean cut | MVP 2 |
| Watch and local alert (M-13) **[A]** | Not needed to prove the live-change loop | MVP 2 |
| M-17, M-18, C-08 (Station Master) | Owner deferred the platform manager role | MVP 2 |
| M-21 CIO messaging | Trilingual authoring workflow is a product of its own | Later |
| M-16 skipped stop (early termination moved into MVP 1 by the owner, 2026-10-21) | Four change types are enough to prove propagation | MVP 2 |
| Crew work-device location (M-08, M-22, M-24) | Owner chose guard departure logs as the live source | Later |
| M-27 passenger food ordering, provider menu feed | Catering proven as an integration first | Later |
| Interactive station terminal (M-14 option, S-22) | One display is enough for the station artifact | Later |
| Server push and SMS notifications (S-01) | Owner decision | Later |

---

## Open points

None at present. The seed data (station names, timetable, staff accounts) is left to the owner to
define.
