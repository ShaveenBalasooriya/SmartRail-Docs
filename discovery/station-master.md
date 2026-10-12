# Station Master — Problem & Impact Analysis

**Snapshot.** The person accountable for a single station — the local face of the railway to
passengers and the local eyes and ears for operations. The station master keeps the station safe and
usable day to day: deciding when a platform can be used, reporting what is happening on the ground to
control, and seeing the same live arrival and departure information the passengers see.

## 1. Discovery

### Assumptions

**Qualitative**

- The station master is the railway's **local authority at one station**, accountable for its safe and
  continuous operation and for what passengers experience there.
- **One station master per station, scoped to their own station only.** They act on their station, not
  on the wider network.
- The role is the station's **link to operations**: local conditions that control cannot see from the
  centre — overcrowding, security events, a passenger medical emergency, a facility failure — are
  reported up by the station master.
- The station master **owns platform availability** — whether a platform at their station is usable.
  They know when a platform is blocked or out of use, and they are the ones who would mark it out of
  use or back in service.
- **Platform allocation is separate, and not theirs.** Which train uses which platform is decided by
  the Network Controller, not the Station Master. Availability ("is this platform usable?") and
  allocation ("which train uses this platform?") are two different decisions and must not be confused.
- **Out-of-use is verified before it is published.** A platform marked out of use is published only
  after the Network Controller verifies it.
- **Local display notices are station-scoped.** Local issues at the station — for example, a lift out
  of order or a facility closure — are the Station Master's notices, shown on their own station's
  displays only; a local notice affects no other station and not the network.
- The station master needs **the same live arrival and departure information the passenger displays
  show**. They answer the same questions passengers ask, and should not be working from a different,
  staler picture.
- The station master can maintain **basic station information for their own station only** — a small
  part of the role, and not a facilities or service-management feature.
- **Station staff** support the station master with a read-only view of the same information.
- Today all of this is verbal, radio and local knowledge: there is no structured way to mark a platform
  out of use or report a station incident, so the state of a station lives in people's heads and in
  ad-hoc calls.
- Station events are **local and transient**, so if they are not captured where they happen, control
  and passengers keep working from a station picture that is already out of date.

**Assumed figures (illustrative — must be validated)**

- The network is on the order of **~150 stations** and **~350 daily services**.
- Typically **one station master per station**.
- On the order of **1–5 reportable station events per station per day** (overcrowding, security,
  medical, facility faults), varying widely by station and time of day.
- Assume a meaningful share of station events (**~30–40%**) are never captured in any structured
  system today.

### Responsibilities

The station application spans two directions: information the station master needs to **consume**, and
local facts they should be able to **record or update**.

**What the station master needs to see (consume)**

- **Live arrivals and departures for their station** — the same information the passenger displays
  show.
- The current **availability** of each platform at their station: usable, out of use, or changed.
- The **platform allocation** for each service (which train uses which platform), set by the Network
  Controller.
- The timetable and planned stopping pattern for their station.
- Live operational notifications and instructions from control (delays, cancellations, diversions,
  platform changes).
- The state of incidents and reports they have raised, and whether they are being handled.

**What the station master should be permitted to record or update (produce)**

- **Platform availability** — mark a platform at their station out of use, or back in service.
- **Local display notices** — raise notices about local issues at their station (for example, a lift
  out of order or a facility closure), shown only on their own station's displays.
- **Basic station information** — keep the basic details of their own station current, and only their
  own station. This is not a facilities or service-management feature.
- **Local incident reports to the Controller** — overcrowding, security, a passenger medical emergency,
  a facility failure.

### 5 Whys

1. **Why does the station master struggle to keep their station and the wider operation on the same
   picture?**
   → Because station-level reality — a platform out of use or an incident — is held locally and passed
   on by voice, not captured as structured data.
2. **Why is it only passed on by voice?**
   → Because there is no tool at the station to mark a platform out of use or log an incident where and
   when it happens.
3. **Why does that matter?**
   → Because control keeps planning and passengers keep arriving on a picture that no longer matches
   the station, and the station master has no authoritative view to work from either.
4. **Why can't the station master see a reliable picture themselves?**
   → Because the live arrivals, departures and platform state they depend on are not exposed to the
   station in the same way the passenger displays get them, so the station works from separate, staler
   sources.
5. **Why has this never been solved?**
   → Because there is **no unified platform or integration layer** to capture a station event once and
   distribute it, in real time, to control, passengers, displays and staff.

### Root Cause

A station's local reality is not captured where it happens and is never connected to the shared
operational picture, so the station and control each work from a partial, out-of-date view of the same
place. A platform can be out of use or an incident underway and still not reach the people planning the
railway or waiting on it. **The root cause is an integration and architecture gap at the point of
capture — here at the station — not a lack of local diligence.**

### Problem Statement

Station masters struggle to keep their station and the wider operation on the same picture when a
platform goes out of use or an incident happens, because there is no structured, connected way to
record those local facts where they happen, causing control and passengers to work from an out-of-date
station, delayed responses, and a station master answering the same questions as the displays but from
staler information. A successful solution would let the station master mark platform availability and
report local incidents to the Controller — while seeing the same live arrivals and departures the
displays show, with a read-only view for station staff — targeting a rise in station events captured
in a structured system from an assumed ~60% to ≥95% and a fall in the time from a station event to
control and passenger awareness from an assumed ~5–10 minutes to under ~1 minute.

## 2. Understanding the Impact

| Stakeholder | How the problem affects them |
|---|---|
| **Station masters** (primary) | No structured way to mark a platform out of use or raise an incident; the station picture lives in their head and in phone calls; answer passengers from staler information than the displays; local knowledge is lost at shift handover. |
| **Station staff** | Share the same gap with no shared view; field the same passenger questions; a read-only view would give them the same truth as the station master. |
| **Passengers at the station** | Arrive to a platform that is out of use or crowding they were not warned about; information conflicts between what they see and what staff say. |
| **Network controllers / operations** | Cannot see local station conditions (overcrowding, security, medical) unless someone calls; the operational picture is missing the ground level; slower, inconsistent response. |
| **Passenger messaging** | Cannot message accurately about station conditions it has not been told about. |
| **Drivers, guards and onboard staff** | Arrive at a station to find it blocked or overcrowded, with no advance warning. |
| **The railway organisation** | No structured record of station incidents or platform availability; safety risk from an out-of-use platform; poor evidence for reporting and improvement. |
