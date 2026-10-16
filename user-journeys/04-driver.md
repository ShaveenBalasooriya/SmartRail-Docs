# Driver (Nuwan) — MVP 1 Journeys and Stories

**Surface:** Crew mobile app, read-only except acknowledgement.

## Context

SmartRail MVP 1 is a single source of truth for live train running on a small network: two lines of five stations each, running in both directions and sharing one interchange station, nine stations in total, all loaded as seed data. A change is made once and reaches the passenger app, a station platform display, the crew app and an external catering provider, each copy carrying a timestamp and a provenance label; every staff change is audited and only possible for the right role.

Nuwan is the Driver. He works in the crew mobile app, which is read-only apart from acknowledging notices: it shows his assigned journeys, their stops and the Network Controller's notices, so that instructions no longer depend on radio alone. His core pain is that there is no consolidated view of assigned journeys, route and notices, because instructions are relayed by radio only. He depends on the Scheduler for his assigned journeys and on the Network Controller for notices and changes to them, and the app shows him the same changes passengers see.

## Key terms

- **Notice and acknowledgement** — a time-stamped operational instruction from the Network Controller to a journey's crew, which the Driver acknowledges with one tap; the acknowledgement records the actor and the time and is audited.
- **Freshness and the stale marker** — every live item shows its "last updated" time; when the app cannot confirm the state it keeps the last-known data marked stale rather than showing a blank screen.
- **Role-based access** — the server enforces the fixed role catalogue on every request, so a wrong-role call is refused with 403 and the attempt is audited rather than merely hidden in the interface.

## End-to-end scenarios this role takes part in

- **Platform change everywhere** — the Network Controller publishes a platform change; the passenger app and station display show it with its time and provenance label, the simulated catering provider receives it, and the change also appears in the Driver's crew app.
- **Separation of duties denials** — a Scheduler edit to a journey inside the 24-hour window is rejected by the API, and a Guard token attempting to publish a disruption gets 403; both denials are audited. The Driver's app relies on the same rule: any write other than an acknowledgement is refused with 403 and audited.
- **Wrong cancellation corrected** — the Network Controller publishes a wrong cancellation and then corrects it; every channel, including the provider, shows the correction, and the audit keeps the original.

---

## Journeys

### J1 — Prepare for the journey

| # | Stage | Nuwan does | System responds | Pain addressed |
|---|---|---|---|---|
| 1 | Sign in | Logs in on work device | Today's assigned journeys | Folded paper diagram |
| 2 | Review | Opens journey | Route, calling stops, times, platforms, any notices | No consolidated view |

### J2 — Notice arrives while stopped

1. Controller sends a notice to his journey.
2. App shows it time-stamped, without demanding interaction while moving.
3. At a stop, Nuwan reads it and taps "Acknowledge"; the Controller sees the acknowledgement **[A]**.

### J3 — Change arrives mid-journey

1. Controller publishes a platform change at a downstream stop.
2. App updates the stop's platform with the change flag; if offline, shows last-known state as stale and refreshes on reconnect.

---

## User stories

### DRV-01 See assigned journeys — Must
As a Driver, I want to see today's assigned journeys, so that I have my work in one place.
- Only journeys I'm assigned to are shown.
**Delivers:** the day's assigned journeys in one place

### DRV-02 Route and stops — Must
As a Driver, I want each journey's route, calling stops, times and platforms, so that I follow the correct stopping pattern.
- Non-calling stations are clearly distinguished from calling stops **[A]**.
**Delivers:** the route, calling stops, times and platforms for each journey

### DRV-03 Receive notices — Must
As a Driver, I want time-stamped notices from the Controller, so that instructions don't rely only on radio.
- Notices appear in real time and are refreshed on reconnect.
**Delivers:** time-stamped Controller notices in the app

### DRV-04 One-tap acknowledgement — Must
As a Driver, I want to acknowledge a notice with one tap, so that control knows I received it.
- Acknowledgement records actor and time and is audited.
- Acknowledgement queues offline and syncs later **[A]**.
**Delivers:** one-tap acknowledgement of a notice, recorded and audited

### DRV-05 Read-only otherwise — Must
As a Driver, I want the app to have no other inputs, so that it never pulls attention from driving.
- No departure logging, publishing or editing; any such API call returns 403 and is audited.
**Delivers:** a read-only app, with wrong-role calls refused and audited

### DRV-06 Offline honesty — Must
As a Driver, I want last-known data shown with a stale marker when I lose signal, so that I don't trust an outdated screen.
- No blank screen; "last updated" shown; refreshes automatically on reconnect.
**Delivers:** honest last-known data with a stale marker when offline

### DRV-07 Live changes to my journey — Must
As a Driver, I want delays, cancellations and platform changes for my journey to show in the app, so that I see what passengers see.
- Changes appear in real time with their published time.
**Delivers:** live delays, cancellations and platform changes for the assigned journey
