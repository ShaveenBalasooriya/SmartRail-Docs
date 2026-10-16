# Scheduler (Nadini) — MVP 1 Journeys and Stories

**Surface:** Operations web. **Owns:** the plan for journeys more than 24 hours ahead.

## Context

SmartRail MVP 1 is a single source of truth for live train running on a small network: two lines of five stations each, running in both directions and sharing one interchange station, nine stations in total, all loaded as seed data. A change is made once and reaches the passenger app, a station platform display, the crew app and an external catering provider, each copy carrying a timestamp and a provenance label; every staff change is audited and only possible for the right role.

Nadini is the Scheduler. She works on the operations web and owns the plan: the services on the network and everything about them for journeys more than 24 hours ahead. Her core pain is that the plan and today are disconnected, and that edits race at the 24-hour boundary. She depends on the System Admin to create the driver and guard accounts she assigns, and the Network Controller owns a journey once its departure comes within 24 hours.

## Key terms

- **Service vs operational journey** — a service is the plan: line, direction, stops, times and running days; journeys are the daily instances generated from it.
- **24-hour window** — the boundary between the Scheduler's plan (more than 24 hours ahead) and the Network Controller's ownership (inside 24 hours); enforced by the server.
- **Lifecycle status** — whether a journey is `scheduled`, `running`, `cancelled` or `completed`.
- **Audit record** — an append-only, hash-chained record of every staff state change and every denied cross-role attempt, with actor, role, before and after values and a server timestamp; it holds no passenger data.
- **Role-based access** — the server enforces the fixed role catalogue on every request, so a wrong-role call is refused with 403 and the attempt is audited rather than merely hidden in the interface.

## End-to-end scenarios this role takes part in

- **New service reaches passengers** — the Scheduler creates a service, daily journeys are generated from it, and a journey appears in the passenger app's departures.
- **Separation of duties denials** — a Scheduler edit to a journey inside the 24-hour window is rejected by the API, and a Guard token attempting to publish a disruption gets 403; both denials are audited.

---

## Journeys

### J1 — Put a new train service onto the network

**Trigger:** a new service is needed on a line. **Outcome:** tomorrow's journeys exist and passengers can see them.

| # | Stage | Nadini does | System responds | Pain addressed |
| --- | --- | --- | --- | --- |
| 1 | Sign in | Logs in | Shows Scheduler functions only | Role clarity |
| 2 | Define | Picks line and direction, names service | Offers seeded stations in line order | No reference-data rework |
| 3 | Stops | Chooses all stops or limited pattern | Validates stops belong to the line | Wrong-pattern errors |
| 4 | Times | Enters arrival/departure per stop | Rejects non-increasing times **[A]** | Invalid timetable |
| 5 | Days | Sets running-days calendar | Shows which dates the service runs | — |
| 6 | Platforms | Picks a platform per stop | Lists only that station's platforms | Wrong platform allocation |
| 7 | Crew | Assigns a driver and a guard | Lists active accounts with those roles | Crew assigned informally |
| 8 | Save | Saves the service | Generates daily operational journeys; change audited | — |
| 9 | Confirm | Opens upcoming timetable view | Shows new journeys outside the 24-hour window | Cannot see own plan |

### J2 — Adjust an upcoming service

**Trigger:** a platform or crew change is needed for a future date. **Outcome:** plan updated before the window closes.

1. Opens upcoming timetable view (only journeys > 24 h ahead are editable).
2. Selects journey or service, changes platform, time or crew.
3. System saves, regenerates affected future journeys, audits before/after values.

### J3 — Hit the 24-hour boundary

**Trigger:** Nadini tries to change a journey that has entered the Controller's window. **Outcome:** blocked, clearly told why.

1. The journey no longer appears in her editable view.
2. A direct edit (e.g. stale tab, API call) is rejected by the server.
3. The denial is recorded in the audit; ownership now sits with the Network Controller.

**Gap carried forward:** there is no tracked handoff from the Controller to her in MVP 1; the handoff workflow is deferred to MVP 2.

---

## User stories

### SCH-01 Role-scoped sign-in — Must

As a Scheduler, I want to sign in and see only scheduling functions, so that I cannot act outside my role.

- Given an active Scheduler account, when I sign in, then I see service and timetable functions only.
- Given my token, when I call any Controller, Admin or Auditor endpoint, then I get 403 and the attempt is audited.
**Delivers:** role-scoped sign-in, with wrong-role calls refused and audited

### SCH-02 Create a service — Must

As a Scheduler, I want to create a service on a line and direction, so that trains can run on the network.

- Given the seeded two-line network, when I create a service, then I must choose one line and one direction.
- Stations are offered in that line's station order for that direction.
**Delivers:** scheduling of services on the network (plan tooling)

### SCH-03 Stopping pattern — Must

As a Scheduler, I want to set all-stops or a limited stopping pattern, so that passengers get correct calling stops.

- A pattern must include the first and last station of the service **[A]**.
- Skipped stations are stored as not calling, and are not shown as calling stops to passengers.
**Delivers:** correct calling stops for passengers from the stopping pattern

### SCH-04 Times per stop — Must

As a Scheduler, I want to enter scheduled times per stop, so that live running can be compared against the plan.

- Each calling stop has a scheduled arrival and/or departure.
- When times are not increasing along the route, then save is rejected with the offending stop named **[A]**.
**Delivers:** scheduled times per stop in the plan

### SCH-05 Running days — Must

As a Scheduler, I want to set the days a service runs, so that journeys are generated only on those dates.

- Given running days Mon–Fri, then no journeys are generated for Sat/Sun.
**Delivers:** running days, and journeys generated only on those dates

### SCH-06 Platform per stop — Must

As a Scheduler, I want to allocate a platform at each stop, so that passengers and the provider know where to go.

- Only platforms belonging to that station are selectable.
**Delivers:** a platform allocation per stop, known to passengers and the provider

### SCH-07 Crew assignment — Must

As a Scheduler, I want to assign a driver and a guard to journeys, so that crew see their work in the crew app.

- Only active accounts with the Driver or Guard role are offered for the matching slot.
- When assigned, the journey appears in that crew member's app.
**Delivers:** crew assignment, and the assigned journey appearing in the crew member's app

### SCH-08 Daily journey generation — Must

As a Scheduler, I want operational journeys generated from my services, so that each day's running exists without manual entry.

- Given a saved service running tomorrow, then tomorrow's journeys exist with status `scheduled` and appear in passenger departures (end-to-end scenario: New service reaches passengers).
**Delivers:** daily operational journeys generated from services, visible to passengers

### SCH-09 Upcoming timetable view — Should

As a Scheduler, I want a list/timetable view of upcoming services outside the 24-hour window, so that I can check my plan.

- Shows service, line, direction, dates, stops, platforms and crew.
- Journeys inside 24 hours are not shown as editable.
**Delivers:** the upcoming-timetable view of services outside the 24-hour window (plan tooling)

### SCH-10 24-hour lock — Must

As a Scheduler, I want the system to stop me editing journeys inside 24 hours, so that I never race the Controller.

- Given a journey departing within 24 hours, when I submit an edit, then the API rejects it and the denial is audited (end-to-end scenario: Separation of duties denials).
- In-window journeys are hidden from my editable view.
**Delivers:** the server-enforced 24-hour boundary, and audit of denied edits

### SCH-11 Audited plan changes — Must

As a Scheduler, I want every plan change recorded with before/after values, so that my changes are attributable.

- Each create/edit writes an audit entry with actor, role, before, after and server timestamp.
**Delivers:** an audited record of plan changes with before and after values
