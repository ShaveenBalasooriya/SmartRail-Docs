# Food Ordering Service Providers — Problem & Impact Analysis

**Snapshot.** External catering businesses that sell food to passengers and fulfil orders against a
railway journey. They are separate companies with their own platforms and kitchens, not part of the
railway, and there can be more than one — SmartRail shows passengers the options available to them. To
prepare an order and hand it over correctly, a provider needs just enough of the railway's journey
picture, and today it does not have it in a timely, reliable form.

## 1. Discovery

### Assumptions

**Qualitative**

- Catering is provided by **external businesses**, not by the railway. Each provider runs its own
  platform and operations, and there may be several offering different menus and different stations.
- The provider stays a **separate external system**. The railway does not own the provider's menu,
  stock, pricing or kitchen; it integrates with the provider and shows the provider's options to
  passengers.
- The provider's job is to **receive an order, prepare it, and hand it to the passenger** at the right
  station and platform as the train passes through.
- To do that, the provider needs a **minimum slice of railway information** — the service and route,
  the station, the platform, the scheduled and expected arrival, and any service change — plus the
  order and **just enough passenger detail to identify and deliver it correctly**.
- The provider should **not** receive the railway's full operational picture, nor more passenger
  information than an order requires. The integration is deliberately narrow.
- The railway's journey data (arrival estimates, platform, delays and changes) is the provider's signal
  for when to prepare and when to be on the platform. Without it, preparation and handover are
  guesswork.
- Handover happens against a **moving train and a short platform window**, so timing accuracy is the
  thing that matters most.
- Food availability, menus and prices change on the provider's side, so the passenger-facing options
  depend on the provider publishing those changes through the integration.
- Many parts of the ordering and fulfilment process are **undefined** today and have to be decided:
  which stations support catering; when an order can be placed and how close to arrival it can change;
  what happens when a train is delayed or the passenger misses the handover point; who may cancel; who
  owns payment; who owns the authoritative menu; how availability changes are communicated; how orders
  are tied to a passenger and a journey; and how successful handover is confirmed.

**Assumed figures (illustrative — must be validated)**

- The network is on the order of **~150 stations** and **~350 daily services**.
- On the order of **a handful of providers**, each serving a subset of stations and services.
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
- **Confirmation of successful handover**, and cancellation or failure where an order cannot be
  fulfilled.

### 5 Whys

1. **Why do providers struggle to fulfil food orders for train passengers reliably?**
   → Because they act on incomplete, late or missing journey information, so they cannot know when and
   where to prepare and hand over.
2. **Why is the journey information incomplete or late?**
   → Because the railway's live journey data is not connected to them in a timely, structured way, and
   changes are not pushed to them.
3. **Why does that matter to the order?**
   → Because an order is bound to a moving train and a short platform window, so even small timing
   errors become a missed or wasted order.
4. **Why is the ordering process itself unclear?**
   → Because the responsibilities across the railway, the provider and the passenger — menu, pricing,
   payment, cancellation and confirmation — were never defined or connected.
5. **Why has this never been solved?**
   → Because there is **no unified integration platform** to exchange the right journey and order
   information, in real time, between the railway and separate external providers.

### Root Cause

A provider is asked to fulfil orders against a journey it cannot see clearly, and the ownership of the
ordering process and its data is not connected across the railway, the provider and the passenger. So
food is prepared on guesswork and handed over against a moving train, and the passenger cannot tell
whether the order will arrive. **The root cause is an integration and process-ownership gap between the
railway and its external food-ordering providers — not a lack of catering capability.**

### Problem Statement

Food ordering service providers struggle to prepare and hand over orders for passengers travelling on
the railway when they cannot reliably see which train is coming, when it will arrive, and which station
and platform it will use, or the order and just enough passenger detail to deliver it, because the
railway's live journey data and the order details are not connected to them in a timely, structured
way, causing food prepared for the wrong time or place, missed or late handovers, wasted food and
effort, and passengers left unsure whether their order will arrive. A successful solution would give
each provider exactly the journey, station, platform, arrival and order information needed to fulfil an
order — and nothing more — through a reliable integration with the railway, with order status flowing
back so passengers can track their food, targeting a fall in late or missed handovers from an assumed
~5–10% to under 1% and a rise in orders confirmed as handed over from an assumed ~70% to ≥98%.

## 2. Understanding the Impact

| Stakeholder | How the problem affects them |
|---|---|
| **Food ordering service providers** (primary) | Cannot plan preparation or be on the right platform at the right time; risk preparing for a train that is late, changed or cancelled; missed handovers waste food and money; unclear ownership of payment, cancellation and confirmation; their reputation suffers for delays that were not their fault. |
| **Passengers ordering food** | Order but cannot rely on it arriving; may lose the order when the train is late or they cannot reach the handover point; wasted money and lost trust. |
| **Station master / station staff** | Providers and passengers converge on the platform around handover; crowding, access and safety at the platform; passenger queries about a provider's order they cannot answer. |
| **Railway operations / controllers** | A new dependency on accurate, timely journey data leaving the railway; a delay now carries a catering consequence; no structured way to tell providers about changes. |
| **Passenger messaging** | Must carry order and catering information too, or the passenger is left uninformed about their own order. |
| **The railway organisation** | An underperforming revenue and partnership opportunity when fulfilment is unreliable; brand risk attached to a partner's failure; no structured integration to grow catering sensibly; data-sharing and privacy duties toward passengers. |
