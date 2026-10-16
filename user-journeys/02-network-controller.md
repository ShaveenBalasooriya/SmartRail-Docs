# Network Controller (Mahesh) — MVP 1 Journeys and Stories

**Surface:** Operations web. **Owns:** every journey inside the 24-hour window.

## Context

SmartRail MVP 1 is a single source of truth for live train running on a small network: two lines of five stations each, running in both directions and sharing one interchange station, nine stations in total, all loaded as seed data. A change is made once and reaches the passenger app, a station platform display, the crew app and an external catering provider, each copy carrying a timestamp and a provenance label; every staff change is audited and only possible for the right role.

Mahesh is the Network Controller. He works on the operations web and owns every journey inside the 24-hour window: he is the only role who can change a journey once its departure is within 24 hours. His core pain is that there is no single live view, changes are relayed by hand, and there is no way to correct a wrong publish. His live picture comes from the Guard's per-station departure logs; he sends operational notices to a journey's crew; and every change he publishes reaches the passenger app, the station display, the crew app and the catering provider while the audit record keeps the original.

## Key terms

- **Lifecycle status** — whether a journey is `scheduled`, `running`, `cancelled` or `completed`.
- **Freshness and the stale marker** — every live item shows its "last updated" time; when a client can no longer confirm the state it keeps the last-known data marked stale rather than showing a blank screen.
- **Provenance label** — the tag on a change saying how it was produced; the Controller's structured changes carry the label "automatically published" with the actor and the time.
- **Correction** — a new event that references the original change; the original is never overwritten, and every channel that received the original receives the correction.
- **Notice and acknowledgement** — a time-stamped operational instruction sent to a journey's crew, which the driver acknowledges with one tap.
- **24-hour window** — the boundary between the Scheduler's plan (more than 24 hours ahead) and the Network Controller's ownership (inside 24 hours); enforced by the server.
- **Audit record** — an append-only, hash-chained record of every staff state change and every denied cross-role attempt, with actor, role, before and after values and a server timestamp; it holds no passenger data.
- **Role-based access** — the server enforces the fixed role catalogue on every request, so a wrong-role call is refused with 403 and the attempt is audited rather than merely hidden in the interface.

## End-to-end scenarios this role takes part in

- **Offline guard logs sync once** — a Guard logs departures while offline, reconnects, the logs sync exactly once with their original timestamps, and the passenger app and display show updated progress. The Controller's live view is fed by the same logs.
- **Platform change everywhere** — the Network Controller publishes a platform change; the passenger app and station display show it with its time and provenance label, and the simulated catering provider receives it.
- **Separation of duties denials** — a Scheduler edit to a journey inside the 24-hour window is rejected by the API, and a Guard token attempting to publish a disruption gets 403; both denials are audited.
- **Wrong cancellation corrected** — the Network Controller publishes a wrong cancellation and then corrects it; every channel, including the provider, shows the correction, and the audit keeps the original.

---

## Journeys

### J1 — Start of shift: read the live railway

| # | Stage | Mahesh does | System responds | Pain addressed |
|---|---|---|---|---|
| 1 | Sign in | Logs in | Opens today's live view | — |
| 2 | Scan | Reviews all today's journeys | Status, owner, minutes early/late, last/next station, freshness | Reading radio + phone + separate systems |
| 3 | Spot | Notices a journey with old freshness | Journey flagged as stale | Acting on lagging data unknowingly |

### J2 — Publish a delay

1. Selects a running journey that guard logs show is late, or which he knows will be held.
2. Enters delay minutes.
3. System publishes immediately, labelled **automatically published**, with his name and timestamp.
4. Expected times update downstream; passenger app, display, crew app and provider receive it; audit records it.

### J3 — Platform change (end-to-end scenario: Platform change everywhere)

1. Selects journey and stop, picks a new platform from that station's platforms.
2. System publishes with the time the change became known.
3. App and display flag it prominently; provider receives a webhook; crew app updates.

### J4 — Cancel, then correct a mistake (end-to-end scenario: Wrong cancellation corrected)

1. Cancels a journey; it propagates to every channel.
2. Realises it was the wrong journey; issues a correction referencing the original.
3. Every channel that got the cancellation shows the corrected state; the audit keeps both events.

### J5 — Tell the crew

1. Writes an operational notice for a journey.
2. Driver (and guard **[A]**) receive it; Mahesh sees when the driver acknowledges **[A]**.

---

## User stories

### NC-01 Live view of today — Must
As a Network Controller, I want one live view of all today's journeys, so that I am not assembling the picture by hand.
- Each journey shows: status (scheduled/running/cancelled/completed), owner, last station passed, next station, minutes early/late, last-updated time.
- Updates arrive in real time without page reload.
**Delivers:** one live view of today's journeys that updates in real time

### NC-02 Stale journeys visible — Must
As a Network Controller, I want journeys with old data marked stale, so that I don't act on a confident but wrong picture.
- When a journey's freshness exceeds the stale threshold **[A]**, then it is visibly marked stale with its last-updated time.
**Delivers:** honest handling of stale live data

### NC-03 Publish a delay — Must
As a Network Controller, I want to publish a delay, so that every channel shows the same expected times.
- Given a journey in my window, when I publish a delay of N minutes, then expected times at all downstream stops update.
- The change carries actor, timestamp and the label `automatically published`, reaches app, display, crew app and provider, and is audited.
**Delivers:** publishing a delay, with expected times updating on every channel

### NC-04 Publish a cancellation — Must
As a Network Controller, I want to cancel a journey, so that passengers and the provider stop planning around it.
- Journey status becomes `cancelled` on every channel; provider receives a cancellation webhook; audited.
**Delivers:** publishing a cancellation to every channel, including the provider

### NC-05 Platform change — Must
As a Network Controller, I want to change a journey's platform at a stop, so that passengers go to the right place.
- Only that station's platforms are selectable.
- App and display show the new platform prominently with the time it became known; provider and crew app receive it (end-to-end scenario: Platform change everywhere).
**Delivers:** a platform change flagged prominently on every surface

### NC-06 Correct a wrong change — Must
As a Network Controller, I want to issue a correction that references the original, so that a mistake doesn't persist anywhere.
- The correction is a new event linked to the original; the original is never overwritten.
- Every channel that received the original, including the provider, receives the correction.
- Audit shows both (end-to-end scenario: Wrong cancellation corrected).
**Delivers:** correcting a wrong change without erasing the original

### NC-07 Operational notice to crew — Must
As a Network Controller, I want to send a notice to a journey's crew, so that instructions don't depend only on relayed calls.
- Notice is time-stamped and delivered to the assigned driver's (and guard's **[A]**) app.
- I can see whether the driver has acknowledged it **[A]**.
**Delivers:** operational notices to a journey's crew, with one-tap driver acknowledgement

### NC-08 In-window ownership — Must
As a Network Controller, I want to be the only role able to change journeys inside 24 hours, so that ownership is unambiguous.
- I can change in-window journeys; the Scheduler cannot (see SCH-10).
**Delivers:** in-window ownership of journeys, kept apart from scheduling

### NC-09 Progress from guard logs — Must
As a Network Controller, I want progress derived from guard departure logs, so that I see actual rather than assumed running.
- When a guard logs a late departure, then minutes late and downstream expected times update in my view.
**Delivers:** live progress derived from the Guard's departure logs

### NC-10 Stay in my lane — Must
As a Network Controller, I want the system to reject actions outside my role, so that separation of duties is provable.
- Calls to Scheduler plan edits (outside the window), account management or audit write return 403 and are audited.
**Delivers:** role-based access and audit of denied cross-role attempts
