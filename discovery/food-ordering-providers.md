# External Catering Provider — Problem & Impact Analysis

**Snapshot.** The external catering business that sells food to passengers and fulfils orders against a
railway journey. It is a separate company with its own platform and kitchen, not part of the railway:
SmartRail exposes its menu to passengers and integrates with it. To prepare an order and hand it over
correctly, the provider needs just enough of the railway's journey picture, and today it does not have
it in a timely, reliable form.

## 1. Discovery

### Assumptions

**Qualitative**

- Catering is provided by **a single external business** — the railway does not cook or sell food
  itself. The provider runs its own platform and operations, and **the design should allow more
  providers to be added later**, each with different menus and stations.
- The provider stays a **separate external system**. The railway does not own the provider's menu,
  stock, pricing or kitchen; it integrates with the provider and shows the provider's options to
  passengers.
- The provider's job is to **receive an order, prepare it, and hand it to the passenger**. Handover
  works in one of two ways: the catering outlet is inside or close to the station, or catering staff
  with special passes meet the passenger inside the train carriage.
- **Catering station staff** — the people who physically prepare and hand over orders — are the
  provider's frontline sub-stakeholder; they are the ones who need the train, platform, arrival and
  order details in the moment.
- To do that, the provider needs a **minimum slice of railway information** — the service and route,
  the station, the platform, the scheduled and expected arrival, and any service change — plus the
  order and **just enough passenger detail to identify and deliver it correctly**.
- The provider should **not** receive the railway's full operational picture, nor more passenger
  information than an order requires. The integration is deliberately narrow.
- The railway's journey data (arrival estimates, platform, delays and changes) is the provider's signal
  for when to prepare and when to be ready to hand over. Without it, preparation and handover are
  guesswork.
- Handover happens while the train is stopped at the station, in a **short window**, so timing accuracy
  is the thing that matters most.
- Food availability, menus and prices change on the provider's side, so the passenger-facing options
  depend on the provider publishing those changes through the integration.
- Many parts of the ordering and fulfilment process are **undefined** today and have to be decided:
  which stations support catering; when an order can be placed and how close to arrival it can change;
  what happens when a train is delayed or the passenger misses the handover point; who may cancel; who
  owns payment; who owns the authoritative menu; how availability changes are communicated; how orders
  are tied to a passenger and a journey; and how successful handover is confirmed.

**Assumed figures (illustrative — must be validated)**

- The network is on the order of **~150 stations** and **~350 daily services**.
- **A single external provider** for now, serving a subset of stations and services; the design allows
  more providers later.
- Only a subset of stations support catering — from a few up to a few dozen.
- Assume a minority of passengers (**~10–15%**) order food, concentrated around mealtimes and on
  longer journeys.
- Assume a preparation-to-handover window on the order of **30–60 minutes** before the train is due.
- Assume a small but material share of orders (**~5–10%**) currently risk being late, missed or wasted
  because arrival and change information does not reach the provider in time.

### Responsibilities

The provider integration spans two directions: the railway information the provider needs to
**consume**, and the order and menu information the provider **produces**.

**What the provider needs to see (consume)**

- The service and route carrying the order, and the **station** where the handover happens.
- The **platform** the train will use.
- The **scheduled and estimated arrival** at that station, and any delay.
- The **journey state** — running, changed, cancelled, terminated early — because that decides whether
  the order can be fulfilled at all.
- The **order**: items, order reference and current status.
- **Just enough passenger detail to deliver correctly** — enough to identify the passenger and hand
  over the order, and no more than the order requires.

**What the provider produces (produce)**

- Its authoritative **menu, prices and availability**, and the changes to them, published for
  passengers to see.
- Order **acceptance, preparation and fulfilment** against the journey.
- **Order status updates** so the passenger can follow progress (accepted, preparing, on its way,
  handed over).
- **Handover to the passenger** by catering station staff — at the outlet, or inside the carriage where
  passes allow.
- **Confirmation of successful handover**, and cancellation or failure where an order cannot be
  fulfilled.

### 5 Whys

1. **Why does the provider struggle to fulfil food orders for train passengers reliably?**
   → Because it acts on incomplete, late or missing journey information, so it cannot know when and
   where to prepare and hand over.
2. **Why is the journey information incomplete or late?**
   → Because the railway's live journey data is not connected to the provider in a timely, structured
   way, and changes are not pushed to it.
3. **Why does that matter to the order?**
   → Because an order is bound to a train stopped at a station for a short window, so even small timing
   errors become a missed or wasted order.
4. **Why is the ordering process itself unclear?**
   → Because the responsibilities across the railway, the provider and the passenger — menu, pricing,
   payment, cancellation and confirmation — were never defined or connected.
5. **Why has this never been solved?**
   → Because there is **no unified integration platform** to exchange the right journey and order
   information, in real time, between the railway and the external provider.

### Root Cause

The provider is asked to fulfil orders against a journey it cannot see clearly, and the ownership of the
ordering process and its data is not connected across the railway, the provider and the passenger. So
food is prepared on guesswork and handed over in a short station window, and the passenger cannot tell
whether the order will arrive. **The root cause is an integration and process-ownership gap between the
railway and its external catering provider — not a lack of catering capability.**

### Problem Statement

The external catering provider struggles to prepare and hand over orders for passengers travelling on
the railway when it cannot reliably see which train is coming, when it will arrive, and which station
and platform it will use, or the order and just enough passenger detail to deliver it, because the
railway's live journey data and the order details are not connected to it in a timely, structured way,
causing food prepared for the wrong time or place, missed or late handovers, wasted food and effort,
and passengers left unsure whether their order will arrive. A successful solution would give the
provider exactly the journey, station, platform, arrival and order information needed to fulfil an
order — and nothing more — through a reliable integration with the railway, with order status flowing
back so passengers can track their food, targeting a fall in late or missed handovers from an assumed
~5–10% to under 1% and a rise in orders confirmed as handed over from an assumed ~70% to ≥98%.

## 2. Understanding the Impact

| Stakeholder | How the problem affects them |
|---|---|
| **External catering provider** (primary) | Cannot plan preparation or be ready at the right time and place; risk preparing for a train that is late, changed or cancelled; missed handovers waste food and money; unclear ownership of payment, cancellation and confirmation; its reputation suffers for delays that were not its fault. |
| **Catering station staff** | The people physically handing over orders: they need the right train, platform, arrival time and order at the moment, and bear the brunt when the train is late, the platform changes or the passenger cannot be found. |
| **Passengers ordering food** | Order but cannot rely on it arriving; may lose the order when the train is late or they cannot reach the handover point; wasted money and lost trust. |
| **Station master / station staff** | Providers and passengers converge on the platform around handover; crowding, access and safety at the station; passenger queries about an order they cannot answer. |
| **Railway operations / controllers** | A new dependency on accurate, timely journey data leaving the railway; a delay now carries a catering consequence; no structured way to tell the provider about changes. |
| **Passenger messaging** | Must carry order and catering information too, or the passenger is left uninformed about their own order. |
| **The railway organisation** | An underperforming revenue and partnership opportunity when fulfilment is unreliable; brand risk attached to a partner's failure; no structured integration to grow catering sensibly; data-sharing and privacy duties toward passengers. |
