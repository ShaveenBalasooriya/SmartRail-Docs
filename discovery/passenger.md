# Passenger — Problem & Impact Analysis

**Snapshot.** Commuters who travel regularly and know the network, plus occasional travellers who
do not. Both are trying to answer the same simple questions — *which train, when, from where, is it
on time, where is it now, will I make my connection* — and both currently struggle to get a
trustworthy answer in the moment they need it.

## 1. Discovery

### Assumptions

**Qualitative**

- Passengers currently rely on a mix of static station displays, PA announcements, printed
  timetables, staff, and word of mouth. There is no passenger-facing real-time information channel.
- The railway's operational data (schedules, running times, platform allocations, disruptions)
  largely *exists internally* but is **not exposed to passengers in real time**.
- Disruption information reaches passengers late, inconsistently, or not at all — and often differs
  between channels (display vs announcement vs staff).
- The most acute pain is the **information gap**: the unknown wait, a missed connection, a platform
  change, or a cancellation discovered too late.
- **Occasional travellers** are affected most (unfamiliar with the network and alternatives).
  **Regular commuters** are affected mainly during peak times and disruptions.
- Passengers with **accessibility needs** (visual, hearing, cognitive, mobility) are
  disproportionately affected by information that is only visual, only auditory, or only in one place.
- The information passengers value most: next train to a destination, live running status, platform,
  delay/arrival estimates, and **what to do when something has gone wrong**.
- Beyond information, passengers want to act on their journey. A clear example is **food ordering**:
  discovering the food options available on their route, ordering, and having the order delivered or
  collected at a suitable station during the journey.
- For a food order to be of any use, the passenger needs to **track its progress** — accepted,
  preparing, on its way, arrived — in the same way they need live information about their train.
- **Food ordering depends on the journey's real timing.** If a train is delayed or changed and the
  passenger is not told, the passenger and the provider fall out of step, so ordering only works on
  top of trustworthy live journey information.

**Assumed figures (illustrative — must be validated)**

- The brief says only "thousands of passengers per day". For sizing, assume an order of magnitude of
  **~20,000 passenger journeys/day**, ~150 stations, ~350 daily services.
- Assume **~55%** of passengers consider information during a disruption inadequate.
- Assume information-related enquiries consume **~20–25%** of station staff interactions.
- Assume **~30%** of disruption-affected passengers first learn of the change *at the station*.
- Assume a minority of passengers — on the order of **~10–15%** — would order food on a given
  journey, concentrated around mealtimes and on longer journeys.

### 5 Whys

1. **Why do passengers have a difficult experience getting accurate, up-to-date train information?**
   → Because they cannot reliably find the service, platform and live status information they need,
   at the moment they need it.
2. **Why can't they find it?**
   → Because reliable, real-time operational data is not available to passengers through any channel
   they use.
3. **Why isn't it available?**
   → Because operational systems are internal and disconnected — there is no single source of truth
   exposed to passenger-facing channels.
4. **Why are those systems internal and disconnected?**
   → Because the organisation has historically built and maintained systems for *operations*, not for
   *passenger information*; integration across them was never prioritised or possible.
5. **Why was passenger-facing integration never built?**
   → Because **no shared platform or integration layer exists** to collect network data once and
   distribute it, in real time, to mobile, station displays, and terminals.

### Root Cause

The railway has **no unified, real-time information platform**. Operational data is siloed,
internal-only, and not exposed through any shared integration layer, so passengers cannot receive
accurate, timely, consistent information about services, disruptions, and live train progress —
even though the underlying data largely exists. **The root cause is an integration and architecture
gap, not a data-availability problem.**

### Problem Statement

Passengers struggle to plan and complete their journeys when service and disruption information is
missing, delayed, or inconsistent, causing uncertainty, missed trains and connections, wasted time,
and loss of trust in the railway. A successful solution would deliver accurate, real-time, consistent
information across all passenger touchpoints — mobile, station displays, and terminals — **from a
single source of truth**, targeting a rise in information satisfaction from an assumed ~45% to ≥80%
and a ~30% reduction in information-related complaints.

## 2. Understanding the Impact

| Stakeholder | How the problem affects them |
|---|---|
| **Passengers — commuters** (primary) | Daily uncertainty; wasted time on platforms; missed connections; standing on the wrong platform; no confidence whether to wait or re-plan. |
| **Passengers — occasional travellers** | Highest anxiety; unfamiliar with routes and alternatives; more likely to miss trains or give up and not travel. |
| **Passengers with accessibility needs** | Excluded when information is single-channel (visual-only or audio-only); worse outcomes during disruption. |
| **Passengers ordering food** | Cannot discover or order food against their journey, or know whether their order will arrive; a service that could ease their journey is unavailable or unreliable. |
| **Station staff** | Field constant "is my train on time / which platform" questions they cannot reliably answer; pulled away from safety and service duties. |
| **Customer service / contact centre** | High volume of information and complaint contacts; repeated, avoidable queries; reputational pressure. |
| **Train drivers & onboard staff** | Boarding confusion and passenger frustration transferred to them; they become the de-facto information source with no passenger-facing tools. |
| **Railway operations / controllers** | Disruption messaging must be repeated manually across channels; no reliable way to reach affected passengers; passenger confusion adds to operational load. |
| **Food ordering service providers** | Cannot coordinate preparation or collection because they have no reliable journey/arrival data (a downstream dependency of the same root cause). |
| **The railway organisation** | Lost revenue from abandoned trips; brand and trust damage ("unreliable / uninformative" reputation); regulatory and accessibility exposure; operational inefficiency; poor foundation for future digital services. |
