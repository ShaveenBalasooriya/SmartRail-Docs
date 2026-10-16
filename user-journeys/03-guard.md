# Guard (Ishara) — MVP 1 Journeys and Stories

**Surface:** Crew mobile app. **Role in MVP 1:** the source of live running data.

## Context

SmartRail MVP 1 is a single source of truth for live train running on a small network: two lines of five stations each, running in both directions and sharing one interchange station, nine stations in total, all loaded as seed data. A change is made once and reaches the passenger app, a station platform display, the crew app and an external catering provider, each copy carrying a timestamp and a provenance label; every staff change is audited and only possible for the right role.

Ishara is the Guard, working in the crew mobile app, and the Guard is the source of live running data: the actual departure is logged at every station stop, with an optional delay cause from a fixed list, and the app works offline. The core pain addressed is that a departure is confirmed verbally and never recorded, so the Guard carries responsibility without evidence, in weak connectivity. The Guard's work comes from the Scheduler's plan, and notices and changes to that work come from the Network Controller; the departures logged feed the Controller's live view, the passenger app and the station display.

## Key terms

- **Offline queue** — departure logs held on the device with a client-generated ID and the original timestamp of the tap, synced idempotently once the connection returns, with pending and synced status shown.
- **Delay cause** — an optional reason picked from a fixed, short list when a departure is late; it records service impact only.
- **Freshness and the stale marker** — every live item shows its "last updated" time; when the app cannot confirm the state it keeps the last-known data marked stale rather than showing a blank screen.
- **Notice** — a time-stamped operational instruction from the Network Controller to a journey's crew.

## End-to-end scenarios this role takes part in

- **Offline guard logs sync once** — the Guard logs departures while offline, reconnects, and the logs sync exactly once with their original timestamps; the passenger app and display show updated progress.
- **Platform change everywhere** — the Network Controller publishes a platform change; the passenger app and station display show it with its time and provenance label, the simulated catering provider receives it, and the change also appears in the crew app.
- **Separation of duties denials** — a Scheduler edit to a journey inside the 24-hour window is rejected by the API, and a Guard token attempting to publish a disruption gets 403; both denials are audited.
- **Wrong cancellation corrected** — the Network Controller publishes a wrong cancellation and then corrects it; every channel, including the provider, shows the correction, and the audit keeps the original.

---

## Journeys

### J1 — Before the shift

| # | Stage | Ishara does | System responds | Pain addressed |
|---|---|---|---|---|
| 1 | Sign in | Logs in on work device | Shows today's assigned journeys | No digital view of own work |
| 2 | Review | Opens a journey | Stops, scheduled times, platforms | — |

### J2 — Log departures along the route (connected)

| # | Stage | Ishara does | System responds | Pain addressed |
|---|---|---|---|---|
| 1 | Platform | Gives the verbal/visual all-clear as today | App does not gate departure | App must not block or distract |
| 2 | Log | One tap "Departed" once the train leaves | Captures device time; next stop highlighted | Departure never recorded |
| 3 | Cause | If late, optionally picks a delay cause | Cause stored with the log | Delay cause lost |
| 4 | Effect | — | Downstream expected times update for passengers, display, Controller | Live picture inferred |

### J3 — Log departures with no signal (end-to-end scenario: Offline guard logs sync once)

1. Taps "Departed" at two stops while offline.
2. App stores each log with a client-generated ID and original timestamp; queue shows "2 pending".
3. Network returns; logs sync once; queue shows "synced"; passengers see updated progress, using original times.

---

## User stories

### GRD-01 See assigned journeys — Must
As a Guard, I want to see my assigned journeys and their stops, so that I know my work in one place.
- Only journeys I am assigned to are listed, with stops, scheduled times and platforms.
**Delivers:** the Guard's assigned journeys and stops in one place

### GRD-02 Log a departure at each stop — Must
As a Guard, I want to log the actual departure at every station stop with one tap, so that the real departure time becomes data.
- When I tap "Departed" for a stop, then the device time at the tap is recorded as that stop's actual departure.
- A stop can be logged once; the next stop is then active **[A]**.
**Delivers:** the actual departure at every stop as live data

### GRD-03 Delay cause — Must
As a Guard, I want to optionally attach a delay cause from a fixed list, so that the reason for lateness is captured consistently.
- Cause list is fixed and short **[A]**; selecting one is optional and never required to log.
- Causes record service impact only; the app is not a safety channel.
**Delivers:** a consistent, optional delay reason on each departure log

### GRD-04 Never block departure — Must
As a Guard, I want the app never to gate or delay departure, so that a dead device or mis-tap can't hold a train.
- Logging is after the fact; no app state prevents the train from leaving.
**Delivers:** departure logging that never gates or delays the train

### GRD-05 Offline logging — Must
As a Guard, I want logs saved on my device when there is no signal, so that no departure is lost.
- Each log gets a client-generated ID and its original timestamp at the tap.
- Logs survive app restart **[A]**.
**Delivers:** offline departure logging with no departure lost

### GRD-06 Queue status — Must
As a Guard, I want to see which logs are pending and which are synced, so that I know my record reached control.
- Queue shows count pending and per-log status (pending / synced).
**Delivers:** visible queue status for pending and synced logs

### GRD-07 Exactly-once sync — Must
As a Guard, I want queued logs to sync exactly once with their original times, so that the record is accurate and not duplicated.
- On reconnect, the server accepts each client ID once; a resent log is ignored without error.
- Progress is computed from the original timestamps, not the sync time (end-to-end scenario: Offline guard logs sync once).
**Delivers:** exactly-once sync of queued logs with their original times

### GRD-08 See changes to my journey — Must
As a Guard, I want delays, cancellations and platform changes for my journey to appear in my app, so that I'm not relying only on radio.
- Changes published by the Controller appear in real time; when offline, last-known state shows with a stale marker.
**Delivers:** journey changes in the app, with honest last-known state offline

### GRD-09 Receive Controller notices — Should **[A]**
As a Guard, I want to see operational notices sent to my journey's crew, so that I have the same instructions as the driver.
- Notices addressed to the journey appear time-stamped in my app.
**Delivers:** Controller notices addressed to the Guard's journey

---

**Open point [A]:** correcting a mis-tapped departure is not in MVP 1 scope (full stop-state model deferred). Recommend the owner confirms whether a Controller-side correction covers this.
