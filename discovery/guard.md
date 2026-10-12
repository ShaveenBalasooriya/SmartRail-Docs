# Guard — Problem & Impact Analysis

**Snapshot.** The onboard safety and departure authority for a train — the person responsible for
passenger safety during stops, for confirming the train is complete and boarding is finished, and
for giving the driver the all-clear to depart. The guard stands at the platform edge at the exact
moment a train leaves, which makes the guard the most reliable human source of a journey's true
departure events.

## 1. Discovery

### Assumptions

**Qualitative**

- The guard's core duties revolve around the safe movement of the train at stations: confirming
  boarding and alighting are complete, ensuring passenger safety during the stop, then signalling the
  driver to depart.
- The guard checks the train is intact before departure using the tail lamp or tail board — a safety
  check that the whole formation is present and coupled.
- The guard holds an emergency brake and can bring the train to a stand in an emergency; any
  application is a safety-critical event.
- The guard communicates with the driver (signals/radio) and is the driver's counterpart for
  coordinating movement.
- Because the guard is not driving, the guard can safely interact with a device at the moment of
  departure — unlike the driver, who must keep attention on operating the train.
- The guard's "ready to depart" signal maps directly to the real-world moment a train actually leaves
  the station; capturing it gives the live picture an honest, human-confirmed departure time.
- Today that confirmation is verbal and visual only and is not captured as structured data, so the
  live picture must infer a departure rather than be told one.
- **How times are known.** The guard's confirmation gives the actual **departure** time at each stop.
  **Arrival** at a stop is not separately confirmed by a human; it is derived — for example, from the
  previous confirmed departure plus the scheduled running time between the two stops. The guard
  therefore remains the only human source of an actual time.
- The guard also becomes a point of contact for passenger safety and information at stops, with no
  digital tooling to record or escalate what they see.
- The guard operates under **safety and regulatory** duties; any tool must not distract from platform
  safety.
- Weak connectivity on the move means any guard tool must tolerate losing and regaining network
  without losing reports.

**Assumed figures (illustrative — must be validated)**

- Roughly **~150 stations** and **~350 daily services**.
- A guard confirms departure on the order of **6–15 times per journey** (once per station stop), plus
  exceptions.
- A meaningful share of departures (**~30–40%**) are not captured with an accurate actual departure
  time today.
- Because a departure is a per-stop moment, capturing it well improves the accuracy of every
  downstream live estimate.

**Related actor noted.** A **driver's assistant** supports the driver and handles the physical token
used for single-line working. Recorded here so the actor is not lost.

### Responsibilities

The guard application spans two directions: information the guard needs to **consume**, and events
the guard should be able to **record or escalate**.

**What the guard needs to see (consume)**

- Assigned journeys, including the crew on each journey (the driver).
- Route, planned stops, platform and stopping pattern.
- Timetable and timing points (scheduled vs actual).
- Live operational notifications and instructions from control.
- Current journey status and any known disruption on the route.
- Passenger safety and accessibility information relevant to the stop (for example, assistance
  requests).

**What the guard should be permitted to record or update (produce)**

- **Departure confirmation** — boarding and alighting complete, "ready to depart" (the core, honest
  source of actual departure time).
- **Station stop status** — unusually long dwell, blocked doors, overcrowding, assisted boarding.
- **Passenger safety events** — person on the track, medical emergency, disorder, unsafe door
  operation.
- **Train integrity / tail lamp confirmation** before departure where required.
- **Emergency brake application** — a safety-critical event that must be logged and reported.

### 5 Whys

1. **Why is a journey's live progress difficult to capture accurately?**
   → Because the real-world events that define progress, especially departure, are confirmed verbally
   and visually and never recorded as structured data.
2. **Why aren't they recorded?**
   → Because the guard, who owns the departure decision at the platform edge, has no digital tool to
   record it.
3. **Why does that matter for live tracking?**
   → Because without a human-confirmed departure the system can only infer or wait, so live data is
   late or wrong from the first event of every stop.
4. **Why is the guard the right person to capture it?**
   → Because the guard already makes the "ready to depart" call, is present when it happens, and can
   use a device safely where the driver cannot.
5. **Why has this never been captured before?**
   → Because there is **no unified platform or integration layer** to collect a confirmed departure
   once and distribute it, in real time, to control, passengers and displays.

### Root Cause

The railway's live picture depends on real-world events that are confirmed by people, but those
confirmations are never captured where and when they happen. At departure the guard is present,
accountable and able to act, yet with no structured, connected way to record the confirmation the
moment is lost and every downstream estimate starts late. **The root cause is an integration and
architecture gap at the point of confirmation, not a lack of guard diligence.**

### Problem Statement

Guards struggle to turn their safety-critical, on-the-spot decisions into usable information when
they signal a train to depart, because they have no structured, connected way to record that
confirmation at the platform edge, causing live progress to be inferred, late or wrong and leaving
the guard's judgement outside the digital record. A successful solution would let the guard confirm
departure and flag safety events during the stop, with the confirmation flowing to the controller and
onward to passengers, targeting a rise in departures with an accurate actual departure time from an
assumed ~60% to ≥95% and a drop in departure-capture delay from an assumed ~3–5 minutes to under
~1 minute.

## 2. Understanding the Impact

| Stakeholder | How the problem affects them |
|---|---|
| **Guards** (primary) | Their departure and safety decisions exist only in the moment and in verbal exchange; no digital record of what they confirmed or when; responsibility without evidence; extra verbal reporting to control; exposed to blame when something goes wrong. |
| **Train drivers / operators** | Depend on the guard's visible or verbal all-clear to depart; if that moment is not captured, the departure event is lost to the system; the driver's attention stays on the train while the guard is the safer point of capture. |
| **Railway controllers / operations** | Work without a confirmed departure event, so their picture is inferred; rely on manual relay of what the guard reports; slower and less reliable disruption handling. |
| **Passengers** | Live progress starts from an unconfirmed departure, so "has it left", "next station" and arrival estimates are late or wrong. |
| **Station staff** | Cannot answer whether a train has actually departed or is about to; must pull information from the platform by radio. |
| **Customer service / contact centre** | No authoritative departure data to quote. |
| **Catering provider** | Preparation and collection depend on actual departure and arrival, which are unconfirmed. |
| **The railway organisation** | No structured record of departure performance, station dwell or safety events (including emergency brake applications), undermining on-time reporting, regulation and improvement. |
