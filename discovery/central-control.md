# Central Control — Problem & Impact Analysis

**Snapshot.** The function that plans, runs, messages, administers and audits the railway from a
central operations centre. It is not one type of user: the authority to change the plan, to change
today's running, to manage access, to message passengers, and to audit the others is deliberately
split across distinct roles — Scheduler, Network Controller, System Admin, Customer Information
Officer and Auditor — so that no single person can both change the railway and cover their tracks.

## 1. Discovery

### Assumptions

**Qualitative**

- Central Control is **one stakeholder made of separated roles**, each owning a different part of the
  railway and each deliberately denied the others' powers.
- **There is no super-admin.** Access management is separated from operations, so no role can both
  alter the railway and alter the record of what it did.
- **Plan versus instance.** The Scheduler edits the *template* of what the railway should do; the
  Network Controller edits what it *is* doing today. These are different data with different owners.
- **Time-based handoff.** The Network Controller acts immediately on anything inside its window
  (up to 24–48 hours ahead). Anything that reaches beyond that window is handed to the Scheduler as an
  owned item — tracked, and closed only when resolved.
- **Cause-based restriction.** A single recorded cause (for example, "Station X closed Mon–Thu") is
  captured once; the system identifies the affected journeys, a human decides the resolution, and the
  approved changes are published consistently everywhere.
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
- **A missing process.** There is no defined path that turns a live incident into planned works, and no
  owned handoff between the Network Controller who handles the incident and the Scheduler who owns the
  plan.
- **The failure it creates.** When a handoff is missed, the plan carries on unaware and future trains
  are routed against a plan that no longer reflects reality — a closed station, a platform
  temporarily shut, a line or network under maintenance, or a route that has changed.
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
| Scheduler | The plan | Stations, platforms, routes, trains, timetables, calendars, planned works, temporary date-limited changes | Change a journey already running |
| Network Controller | Today | Monitor trains, log incidents, delay, cancel, skip stop, terminate early, platform change (up to 24–48h ahead) | Change the timetable or reference data |
| Customer Information Officer | Passenger messaging | Write advisories and notices; correct published messages | Change operational data |
| Auditor | Oversight | Read-only access to logs and reports | Change anything |

**What Central Control needs to see (consume)**

- The shared live operational picture: every service, its actual vs planned position, platform and
  status.
- The plan it is held to: timetable, rolling stock/diagram, platform allocations, calendars and
  planned works.
- The state of every incident, delay and disruption, and who owns it.
- The handoff queue: items raised by the Network Controller that are beyond its window and owed to the
  Scheduler, with status and owner.
- The current passenger-messaging state: what has been told to passengers, and where.
- Logs and reports, scoped per role, with the Auditor seeing everything.

**What each role can change (produce)**

- **System Admin** — manage users, roles and configuration. No operational or schedule data; logs
  remain immutable.
- **Scheduler** — change the plan: stations, platforms, routes, trains, timetables, calendars, planned
  works and temporary date-limited changes. Not a journey already running.
- **Network Controller** — change today: monitor, log incidents, delay, cancel, skip stop, terminate
  early, and apply platform changes within its 24–48h window. Not the timetable or reference data.
- **Customer Information Officer** — write optional advisories attached to a disruption and
  network-wide notices, and correct or pause published messages. Not operational data.
- **Auditor** — produce oversight reports by reading logs and data. Changes nothing.

### 5 Whys

1. **Why does Central Control struggle to keep the planned railway and the running railway in
   agreement?**
   → Because the plan and the running day are owned by different roles and held in different systems,
   with no shared, tracked way to pass work between them.
2. **Why is the handoff untracked?**
   → Because an incident handled today by the Network Controller and the planned works it implies
   (owned by the Scheduler) are recorded in separate places, so nothing forces an incident to become a
   plan change.
3. **Why does that cause real failures?**
   → Because a missed handoff leaves the plan unchanged, so future trains are routed against a plan
   that no longer reflects reality — into closed stations, temporarily shut platforms, lines or
   networks under maintenance, or a route that has changed.
4. **Why can't anyone see or prove what happened?**
   → Because there is no single source of truth linking the cause, the affected journeys, the decision
   and the resolution — and no tamper-proof audit of who did what.
5. **Why has this never been solved?**
   → Because the railway's systems were built function by function, not as one connected platform with
   clearly separated, auditable role boundaries.

### Root Cause

The railway's plan and its live operation are held in disconnected systems owned by different roles,
with no shared, tracked, auditable handoff between them. A real-world event can therefore be fully
known to the Network Controller and still never reach the Scheduler's plan, so trains continue to be
routed against a plan that no longer reflects reality — a closed station, a temporarily shut
platform, a line or network under maintenance, or a changed route. **The root cause is an integration
and ownership gap between planning and operations — compounded by missing separation-of-duties and
audit controls — not a lack of diligence by any single role.**

### Problem Statement

Central Control struggles to keep the running railway and the planned railway in step when an incident
needs a change that outlives the controller's window, because the roles that own today's running and
the future plan have no shared, tracked handover, causing missed handoffs, trains routed against a
plan that no longer reflects reality (closed stations, shut platforms, lines or networks under
maintenance, changed routes), duplicated or contradictory changes, and no trustworthy record of who
changed what. A successful solution would give each role the authority and visibility it needs inside
one connected platform — the Scheduler owning the plan, the Network Controller owning today, the
System Admin owning access, the Customer Information Officer owning passenger messaging, and the
Auditor holding independent, read-only oversight — with a tracked Controller-to-Scheduler handoff and
tamper-proof logs of every action, targeting a fall in missed or untracked handoffs from an assumed
~15–20% to under 2% and no published journey routed against a plan that no longer reflects reality.

## 2. Understanding the Impact

| Stakeholder | How the problem affects them |
|---|---|
| **Central Control — Network Controllers** (primary) | Act on today without an owned way to push longer-term consequences into the plan; incidents they resolve can silently stop being anyone's problem; may authorise or permit running against a plan that no longer reflects reality (a closed station, a shut platform, a line under maintenance, a changed route); decisions relayed by hand. |
| **Central Control — Schedulers** | Receive changes informally, if at all; may not learn a station or platform has been taken out of use until trains are already scheduled there; forced into rework and firefighting; own the plan but cannot see today's reality. |
| **Central Control — System Admin** | Pressured to grant broad access because the roles are not cleanly separated; cannot guarantee an audit trail; carries configuration risk without operational authority. |
| **Central Control — Customer Information Officer** | Writes passenger messaging that can contradict the actual running state; no single source of truth to message from. |
| **Central Control — Auditor** | Cannot give assurance because logs are incomplete or editable; oversight is retrospective and unprovable; no evidence of separation of duties. |
| **Drivers, guards and onboard staff** | Can be dispatched toward a closed station, a shut platform, a line under maintenance or a changed route; receive contradictory or missing instructions; absorb passenger frustration on board. |
| **Station staff** | Manage a station or platform that operations may still believe is in use; face passengers with information that conflicts with the plan. |
| **Passengers** | Sent to or waiting at a station or platform that is no longer in use; unpredictable cancellations or diversions; lost time and trust. |
| **External regulators** | Receive exported reports that may be incomplete or unverifiable; assurance and compliance exposure. |
| **The railway organisation** | Safety and regulatory risk from running against a plan that no longer reflects reality; no defensible audit trail; duplicated work between planning and operations; slow, inconsistent change; eroded confidence in central control. |
