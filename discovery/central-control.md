# Central Control — Problem & Impact Analysis

**Snapshot.** The function that plans, runs, messages, administers and audits the railway from a
central operations centre. It is the railway's head — the place that must know, at any moment, what is
happening across the whole network and act on it — yet it has no single live view of the network and
works instead from radio, phone and separate systems. Within Central Control the authority to change
the plan, to change today's running, to manage access, to message passengers, and to audit the others
is split across distinct roles — Scheduler, Network Controller, System Admin, Customer Information
Officer and Auditor.

## 1. Discovery

### Assumptions

**Qualitative**

- **Central Control's core need is a live view of the network.** It must be able to see, in one place
  and in real time, what every train and station is doing, what is disrupted, and what needs a
  decision. Today that view does not exist: the picture is assembled from radio, phone and separate
  systems, so Central Control is always working from a partial, lagging account of its own network.
- **The roles are separated.** Central Control is one function made of separated roles, each owning a
  different part of the railway and each deliberately denied the others' powers.
- **There is no super-admin.** Access management is separated from operations, so no role can both
  alter the railway and alter the record of what it did.
- **Plan versus instance.** The Scheduler edits the *template* of what the railway should do; the
  Network Controller edits what it *is* doing today. These are different data with different owners.
- **Time-based handling.** The Network Controller acts immediately on anything inside its window (up to
  24–48 hours ahead). Anything beyond that window becomes a handoff to the Scheduler.
- **Cause-based restriction.** A single recorded cause (for example, "Station X closed Mon–Thu") is
  captured once; the system identifies the affected journeys, a human decides the resolution, and the
  approved changes are published consistently everywhere.
- **A discovered gap: today and the plan are not connected.** A live restriction handled by the Network
  Controller (who owns today) often has to become a future plan change owned by the Scheduler (who owns
  the plan). There is no shared, tracked path that turns a live restriction into a future plan change,
  and no owned handoff between the two roles.
- **The failure it creates.** When a handoff is missed, the plan carries on unaware and future trains
  are routed against a plan that no longer reflects reality — a closed station, a platform temporarily
  shut, a line or network under maintenance, or a route that has changed.
- **Handoff controls.** Every handoff is governed so that it cannot be silently dropped:
  - **Deadline.** Each handoff carries a deadline: it must be resolved before the first affected future
    journey.
  - **Escalation.** If the handoff is not accepted in time, the Controller is alerted.
  - **Reassignment.** The Controller can reassign the handoff to another Scheduler.
  - **Fallback.** If no Scheduler is available, the Controller can temporarily extend their own window
    to cover that restriction; the extension is logged, and the handoff stays open until it is resolved.
- **Restriction clearance.** A restriction is marked as cleared by a Controller or a Scheduler, and the
  clearance records who cleared it and the maintenance confirmation.
- **Maintenance is outside SmartRail.** Maintenance is a separate system and is not a SmartRail user;
  maintenance communicates with Central Control outside SmartRail.
- **Structured status changes publish automatically.** Delays, cancellations, platform changes and
  skipped stops are structured data, so the system publishes them straight away — to passengers,
  station displays and the catering provider — with no human in the loop. A review step would only
  slow down a message that is already correct.
- **What the system cannot do well stays human.** The system knows *what* changed; it does not know
  what passengers should do about it. That gap, and the situations below, are the Customer Information
  Officer's remit:
  - **Context and advice** — for example, "use the replacement bus from Platform 2" or "expect
    crowding on the next service".
  - **Notices not tied to a journey** — planned works announced weeks ahead, strikes, weather warnings
    and station facility closures.
  - **Sensitive incidents** — after a fatality or a security event the wording matters, and raw
    operational notes such as a controller's incident log must never be published as they stand.
  - **Languages** — messages need Sinhala, Tamil and English, and a human writes or checks these.
  - **Corrections** — someone can pause or fix a wrong automatic message.
- **Audit everything.** Every role's actions are logged, including the admin's, and the logs cannot be
  changed.
- **Scoped reporting.** Each role sees reports for its own area only; the Auditor sees everything. The
  Auditor is an internal role; external regulators receive exported reports.
- **A security requirement, not a nicety.** Separation of duties between roles, and audit logs that
  are tamper-proof — no one, including administrators, can edit or delete them.
- Today this coordination is verbal, email-based or simply assumed, so a decision that crosses roles
  is only as reliable as the people remembering to pass it on.

**Assumed figures (illustrative — must be validated)**

- The network is on the order of **~150 stations** and **~350 daily services**.
- A small central team runs it: on the order of a few Network Controllers per shift, a Scheduler team,
  one or two Customer Information Officers, a rotating Auditor, and shared administrative support.
- On the order of **a few dozen reportable incidents per day**, of which roughly **1 in 10** extends
  beyond the controller's window and becomes a Scheduler handoff.
- Assume **~30%** of disruptions are currently visible to the centre as they happen; the rest are
  learned second-hand.
- Assume **~15–20%** of handoffs are currently missed, duplicated or completed without a record
  because there is no shared tracking.
- A manual handoff (call, email, re-key) consumes roughly **10–20 minutes** of combined controller and
  scheduler time.

### Responsibilities

Central Control is a single stakeholder whose powers are split by role. The separation is the point:
each role can act only in its own domain, and none can act in all of them.

**Roles and their authority**

| Role | Owns | Can do | Cannot do |
|---|---|---|---|
| System Admin | Access | Manage users, roles and configuration; view all logs and data | Change operational or schedule data; edit or delete logs |
| Scheduler | The plan | Stations, platforms, routes, trains, timetables, calendars, planned works, temporary date-limited changes; clear restrictions | Change a journey already running |
| Network Controller | Today | Monitor trains, log incidents, delay, cancel, skip stop, terminate early, allocate and change platforms (up to 24–48h ahead), manage handoffs, clear restrictions | Change the timetable or reference data |
| Customer Information Officer | Passenger messaging | Write advisories and notices; correct published messages | Change operational data |
| Auditor | Oversight | Read-only access to logs and reports | Change anything |

**What Central Control needs to see (consume)**

- The live, consolidated view of the network: every service, station, disruption and restriction.
- The plan it is held to: timetable, rolling stock/diagram, platform allocations, calendars and
  planned works.
- The state of every incident, restriction, delay and disruption, and who owns it.
- The handoff queue: items owed to the Scheduler, with status, owner and deadline.
- The restriction list and its clearance state.
- The current passenger-messaging state: what has been told to passengers, and where.
- Logs and reports, scoped per role, with the Auditor seeing everything.

**What each role can change (produce)**

- **System Admin** — manage users, roles and configuration. No operational or schedule data; logs
  remain immutable.
- **Scheduler** — change the plan: stations, platforms, routes, trains, timetables, calendars, planned
  works and temporary date-limited changes; clear restrictions. Not a journey already running.
- **Network Controller** — change today: monitor, log incidents, delay, cancel, skip stop, terminate
  early, and allocate or change platforms within its 24–48h window; create and manage handoffs
  (deadline, escalation, reassignment, logged fallback); clear restrictions. Not the timetable or
  reference data.
- **Customer Information Officer** — write optional advisories attached to a disruption and
  network-wide notices, and correct or pause published messages. Not operational data.
- **Auditor** — produce oversight reports by reading logs and data. Changes nothing.

### 5 Whys

1. **Why does Central Control struggle to run the network well?**
   → Because it has no single, live view of what is happening across the network.
2. **Why can't it see the network?**
   → Because the live picture is assembled from radio, phone and separate systems rather than from one
   source of truth.
3. **Why does that matter?**
   → Because decisions, disruption handling and passenger messaging are made from a partial, lagging
   account, so responses are slow and inconsistent.
4. **Why does that also produce the handoff problem?**
   → Because with no shared, tracked workflow, the running day (owned by the Network Controller) and
   the future plan (owned by the Scheduler) are never connected, so a restriction can fail to become a
   plan change.
5. **Why has this never been solved?**
   → Because there is **no unified platform or integration layer** to collect the network's state once
   and distribute it, in real time, to every role that needs it, with a tracked and auditable workflow.

### Root Cause

Central Control has no single live view of the network. The state of trains, stations and disruptions is
scattered across radio, phone and separate systems, so the centre plans, decides and informs from a
partial, lagging picture. The missed Controller-to-Scheduler handoff is a symptom of the same gap: two
roles working from disconnected views, with no shared, tracked workflow to move a restriction from
today into the plan. **The root cause is an integration and architecture gap that leaves the centre
without a unified, real-time view of its own network — not a lack of diligence by any single role.**

### Problem Statement

Central Control struggles to run and oversee the railway when disruptions, restrictions and incidents
occur, because it has no single, live view of what is happening across the network and relies instead
on radio, phone and separate systems, causing slow and inconsistent decisions, delays in informing
passengers, and problems that are noticed late or not at all. A secondary, discovered problem is the
handoff between the Network Controller (who owns today) and the Scheduler (who owns the plan): with no
shared, tracked path, a live restriction can fail to become a future plan change, so trains are routed
against a plan that no longer reflects reality. A successful solution would give every Central Control
role a single, real-time view of the network and the tools to act on it — including automatic
publishing of structured status changes, a tracked handoff with a deadline, escalation, reassignment
and a logged fallback, and restriction clearance — targeting a rise in disruptions visible to the
centre as they happen from an assumed ~30% to ≥95% and a fall in missed or untracked handoffs from an
assumed ~15–20% to under 2%.

## 2. Understanding the Impact

| Stakeholder | How the problem affects them |
|---|---|
| **Central Control — Network Controllers** (primary) | Work without a live view of the network; act on partial, lagging information; incidents they resolve can silently stop being anyone's problem; may authorise or permit running against a plan that no longer reflects reality (a closed station, a shut platform, a line under maintenance, a changed route); decisions relayed by hand. |
| **Central Control — Schedulers** | Receive changes informally, if at all; may not learn a station or platform has been taken out of use until trains are already scheduled there; forced into rework and firefighting; own the plan but cannot see today's reality. |
| **Central Control — System Admin** | Pressured to grant broad access because the roles are not cleanly separated; cannot guarantee an audit trail; carries configuration risk without operational authority. |
| **Central Control — Customer Information Officer** | Writes passenger messaging that can contradict the actual running state; no single source of truth to message from. |
| **Central Control — Auditor** | Cannot give assurance because logs are incomplete or editable; oversight is retrospective and unprovable; no evidence of separation of duties. |
| **Drivers, guards and onboard staff** | Can be dispatched toward a closed station, a shut platform, a line under maintenance or a changed route; receive contradictory or missing instructions; absorb passenger frustration on board. |
| **Station staff** | Manage a station or platform that operations may still believe is in use; face passengers with information that conflicts with the plan. |
| **Passengers** | Sent to or waiting at a station or platform that is no longer in use; unpredictable cancellations or diversions; lost time and trust. |
| **External regulators** | Receive exported reports that may be incomplete or unverifiable; assurance and compliance exposure. |
| **The railway organisation** | Runs a national network from a centre that cannot see it in real time: slow and inconsistent decisions, duplicated effort, no defensible audit trail, safety and regulatory risk from running against a plan that no longer reflects reality, and eroded confidence in central control. |
