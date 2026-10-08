# Driver / Train Operator — Problem & Impact Analysis

**Snapshot.** Professional operational staff who run the trains — the person (or crew) responsible
for operating an assigned journey safely and to schedule. They are the only people who know, in real
time, what is actually happening on the train and on the line ahead, yet they have no structured
digital way to capture or share that reality.

## 1. Discovery

### Assumptions

**Qualitative**

- A driver's core job is to operate an assigned journey safely and to schedule: follow the route and
  stopping pattern, depart and arrive on time, and respond to instructions from control.
- To do that well, a driver needs assigned journeys, route information, scheduled station stops,
  timing information, operational notifications and journey status visible in one place.
- The driver is the **ground truth** for a journey. Control and passengers can only know what is
  happening because the driver (or a system on board) tells them.
- Today that ground truth is relayed **verbally** (radio/phone) and then re-keyed or re-broadcast by
  hand by controllers and station staff. It is slow, unstructured and easy to lose.
- The driver's updates never enter a shared, machine-readable source of truth, so passenger-facing
  channels cannot be updated from them automatically.
- The driver receives little in return: no consolidated digital view of their duty, the timetable,
  the current disruption picture, or onward/connection information.
- The most acute pain is the **reporting loop**: a delay or obstacle happens, the driver has no quick
  structured way to report it, and the information reaches passengers late, inconsistently, or not
  at all.
- Trains pass through areas of weak connectivity, so any driver tool must tolerate losing and
  regaining network without losing reports.
- During disruption the driver becomes the de-facto information desk for confused passengers, with no
  tooling to help.
- Drivers operate under **safety and regulatory** duties; any update mechanism must not interfere
  with, or distract from, operating the train.

**Assumed figures (illustrative — must be validated)**

- Roughly **~150 stations** and **~350 daily services**, served by a few hundred drivers/operators
  across shifts.
- A driver records on the order of **6–15 status events per journey** (departure, each station
  pass/stop, arrival, plus exceptions).
- **~20–30%** of journeys experience at least one reportable delay or incident on a typical day.
- A verbal-only report consumes roughly **1–3 minutes** of combined driver and controller time per
  event (radio call, clarification, manual re-broadcast).
- A meaningful share of delay/incident reports (**~30–40%**) are never captured in any structured
  system today.

### Responsibilities

The driver application spans two directions: information the driver needs to **consume**, and
updates the driver should be able to **produce**.

**What the driver needs to see (consume)**

- Assigned duty/journey and shift roster; the train/unit to operate.
- Route, planned stops and stopping pattern; platform/station information.
- Timetable and timing points (scheduled vs actual).
- Live operational notifications and instructions from the controller.
- Current journey status and any known disruption on the route.
- Onward journey/connection and handover information.

**What the driver should be permitted to update (produce)**

- **Journey status** — departed, arrived at each stop, journey complete. *(The core duty.)*
- **Delay and disruption reports** — reason (signal, obstruction, incident, weather, asset fault) and
  magnitude, raised from the cab.
- **Obstacles / incidents** on the line or at stations, including safety-critical events.
- **Acknowledgement** of instructions and notifications from control.
- **Equipment / fault reports** (train or infrastructure).
- **Handover notes** to the next crew.

### 5 Whys

1. **Why do drivers/operators struggle to run and communicate a journey?**
   → Because they have no digital tool to view their duty or report what is happening in real time.
2. **Why can't they report or view digitally?**
   → Because the only channel is verbal radio/phone, and the information is not captured in a shared
   system.
3. **Why is it only verbal and unstructured?**
   → Because no driver-facing application exists to capture the ground truth and feed it into
   operations and passenger information.
4. **Why doesn't it exist?**
   → Because systems were built for operational control, not for real-time field data capture and
   distribution; integration was never prioritised.
5. **Why was field integration never built?**
   → Because there is **no unified platform or integration layer** to capture a driver's status once
   and distribute it, in real time, to control, passengers, displays and partners.

### Root Cause

The driver sits at the very start of the railway's information pipeline, but there is **no
structured, connected way to capture what the driver knows**. The ground truth of a journey is
locked in verbal reports and individual memory and never enters a shared source of truth, so control,
passengers and displays are always reacting to a picture that is already out of date. **The root
cause is an integration and architecture gap at the point of data capture, not a lack of driver
discipline or goodwill.**

### Problem Statement

Train drivers and onboard operators struggle to keep control and passengers informed of a journey's
true status when delays, obstacles or disruptions occur, because they have no simple, connected way
to report what is happening from the train, causing late and inconsistent information, repeated
verbal reporting, and an operational picture that is always behind reality. A successful solution
would give drivers a simple way to see their assigned journey and publish status, delay and incident
updates that flow to the controller and onward to passengers, targeting a reduction in the time from
an event to a passenger-visible update from an assumed ~10–15 minutes to under ~2 minutes, and a rise
in journeys with structured live-status capture from an assumed ~60% to ≥95%.

## 2. Understanding the Impact

| Stakeholder | How the problem affects them |
|---|---|
| **Drivers / train operators** (primary) | No single digital view of their duty or the network; manual, verbal reporting; added workload during disruption; they become the de-facto information desk for confused passengers with no tooling to help; frustration and safety-distraction risk. |
| **Railway controllers / operations** | Depend on radio and manual re-keying for situational awareness; recurring, time-consuming clarification calls; struggle to re-broadcast updates consistently; their operational picture is always behind. |
| **Passengers** | Receive stale, inconsistent or missing live information; the driver-to-control loop is where the passenger's information gap actually originates. |
| **Station staff** | Field questions they cannot answer because the fresh status never left the cab; pulled from safety and service duties. |
| **Customer service / contact centre** | Handle avoidable "what is happening" contacts with no reliable live status to quote. |
| **Onboard / other operational staff** | Coordination during incidents relies on the same verbal channel; no shared view. |
| **Catering provider** | Cannot plan preparation or collection because journey status is not captured or shared (downstream dependency). |
| **Engineering / maintenance** | Faults reported informally and slowly; no structured feed for reliability or asset management. |
| **The railway organisation** | No reliable, structured record of delays/incidents for reporting, regulation and improvement; slower recovery; eroded trust; the live-data foundation for every other surface is never seeded. |
