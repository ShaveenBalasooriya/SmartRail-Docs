# Driver / Train Operator — Problem & Impact Analysis

**Snapshot.** Professional operational staff who drive the trains — the person (or crew) responsible
for operating an assigned journey safely and to schedule. Their attention belongs on the train and the
line ahead, so the driver consumes the journey picture and raises exceptions. The driver is not the
record of a journey's progress, and cannot be, because they can only report while the train is stopped.

## 1. Discovery

### Assumptions

**Qualitative**

- A driver's core job is to operate an assigned journey safely and to schedule: follow the route and
  stopping pattern, depart and arrive on time, and respond to instructions from control.
- To do that well, a driver needs to see their assigned journeys, the route and stops, operational
  notices and the current disruption picture in one place.
- The driver's attention belongs on operating the train, so any interaction must be minimal and must
  not happen while the train is moving.
- The driver is **not** the record of a journey's progress. Actual departure times are the guard's, and
  the driver's role is to raise exceptions, not to narrate the journey.
- Because the driver can only act while the train is stopped, keeping passengers informed during
  disruption falls to the guard and station staff, not the driver.
- Today the driver's exception reports are relayed **verbally** (radio/phone) and then re-keyed or
  re-broadcast by hand by controllers and station staff. It is slow, unstructured and easy to lose.
- Those reports never enter a shared, machine-readable source of truth, so control and passenger-facing
  channels cannot rely on them.
- The driver receives little in return: no consolidated digital view of their assigned journeys, the
  route, the current disruption picture, or notices from control.
- The most acute pain is the **reporting loop**: an incident, obstruction or fault happens, the driver
  has no quick, structured way to raise it while stopped, and the information reaches control and
  passengers late, inconsistently, or not at all.
- Trains pass through areas of weak connectivity, so any driver tool must tolerate losing and regaining
  network without losing reports.
- Drivers operate under **safety and regulatory** duties; any reporting mechanism must not interfere
  with, or distract from, operating the train.

**Assumed figures (illustrative — must be validated)**

- Roughly **~150 stations** and **~350 daily services**, served by a few hundred drivers/operators
  across shifts.
- **~20–30%** of journeys experience at least one reportable incident, obstruction or fault on a
  typical day.
- A verbal-only report consumes roughly **1–3 minutes** of combined driver and controller time per
  event (radio call, clarification, manual re-broadcast).
- A meaningful share of incident and fault reports (**~30–40%**) are never captured in any structured
  system today.

### Responsibilities

The driver application spans two directions: information the driver needs to **consume**, and
exceptions the driver should be able to **raise**, only while the train is stopped.

**What the driver needs to see (consume)**

- Assigned journeys, including the crew on each journey (the guard).
- Route and planned stops, with platform/station information.
- Operational notices and instructions from the controller.
- Disruptions on the route.

**What the driver should be able to raise (produce, only while the train is stopped)**

- **Incident and obstruction reports** on the line or at stations, including safety-critical events.
- **Fault reports** (train or infrastructure).
- **Acknowledgement** of instructions and notifications from control.

### 5 Whys

1. **Why do drivers struggle to get exceptions into the hands of control and passengers?**
   → Because they have no quick, structured way to report an incident, obstruction or fault, and can
   only do so while the train is stopped.
2. **Why is it verbal and unstructured?**
   → Because the only channel is radio/phone, and the report is not captured in a shared system.
3. **Why does that matter?**
   → Because the report is re-keyed by hand and loses detail, so control and passengers work from a
   picture that is late and incomplete.
4. **Why is there no better channel?**
   → Because no driver-facing application exists to capture exceptions and feed them into operations
   and passenger information; systems were built for operational control, not field reporting.
5. **Why was field integration never built?**
   → Because there is **no unified platform or integration layer** to capture a driver's exception once
   and distribute it, in real time, to control, passengers, displays and partners.

### Root Cause

A driver can only raise an exception from the cab, and only while the train is stopped, yet there is
**no structured, connected way to capture it**. Exceptions are locked in verbal reports and never enter
a shared source of truth, so control, passengers and displays react to a picture that is already out of
date. **The root cause is an integration and architecture gap at the point of exception reporting, not
a lack of driver discipline or goodwill.**

### Problem Statement

Train drivers and onboard operators struggle to get incidents, obstructions and faults into the hands
of control and passengers when they occur, because they have no simple, structured way to raise an
exception from the cab while stopped, causing late and incomplete information, repeated verbal
reporting, and an operational picture that is always behind reality. A successful solution would give
drivers a simple, minimal way to see their assigned journeys, route, notices and disruptions and to
raise incident, obstruction and fault reports while the train is stopped, targeting a reduction in the
time from an exception to a control-visible update from an assumed ~10–15 minutes to under ~2 minutes
and a rise in exceptions captured in a structured system from an assumed ~60% to ≥95%.

## 2. Understanding the Impact

| Stakeholder | How the problem affects them |
|---|---|
| **Drivers / train operators** (primary) | No single digital view of their assigned journeys or the current disruption picture; exceptions relayed by voice; added workload when stopped; frustration and safety-distraction risk if any tooling demands attention while moving. |
| **Railway controllers / operations** | Depend on radio and manual re-keying for situational awareness; recurring, time-consuming clarification calls; their operational picture is always behind. |
| **Passengers** | Receive stale, inconsistent or missing information about incidents on the line; the driver-to-control loop is part of where the passenger's information gap originates. |
| **Guard and station staff** | Carry the burden of keeping passengers informed during disruption, without the driver's fresh detail. |
| **Customer service / contact centre** | Handle avoidable "what is happening" contacts with no reliable live status to quote. |
| **Onboard / other operational staff** | Coordination during incidents relies on the same verbal channel; no shared view. |
| **Food ordering service providers** | Cannot plan preparation or collection because journey status is not captured or shared (downstream dependency). |
| **Engineering / maintenance** | Faults reported informally and slowly; no structured feed for reliability or asset management. |
| **The railway organisation** | No reliable, structured record of incidents and faults for reporting, regulation and improvement; slower recovery; eroded trust; the live-data foundation for every other surface is never seeded. |
