# Catering Provider (RailBites) — MVP 1 Journeys and Stories

**Surface:** Provider API (machine client, API key) and a simulated provider app.

## Context

SmartRail MVP 1 is a single source of truth for live train running on a small network: two lines of five stations each, running in both directions and sharing one interchange station, nine stations in total, all loaded as seed data. A change is made once and reaches the passenger app, a station platform display, the crew app and an external catering provider, each copy carrying a timestamp and a provenance label; every staff change is audited and only possible for the right role. RailBites is that external catering provider: a separate system that stays separate, connecting to SmartRail over an API authenticated with an API key, with a small simulated provider app to show the integration working. RailBites uses the data to time its preparation and handovers at its station; it owns nothing inside SmartRail and cannot change anything there. The core pain addressed is acting on late, incomplete journey data, changes that are not pushed, and being unable to attribute a railway-caused failure. What it receives depends on the Scheduler's services becoming daily journeys, the Guard's departure logs driving live progress, and the Network Controller's published changes and corrections being distributed to it.

The "user" here is RailBites' kitchen team working through their system, so the stories below are written for the provider as a system and for its staff.

## Key terms

- **API key** — the credential the provider presents on every request; it grants read access to provider endpoints only.
- **Webhook** — an HTTP callback from SmartRail to a URL the provider registers, carrying each status change as an event.
- **Sequence number and idempotency** — every event carries an event ID and a sequence number: the same event ID delivered twice is ignored, and an event with a lower sequence than the last applied for that journey is not applied.
- **Retry with backoff** — failed deliveries are attempted again at increasing intervals until they succeed or reach the retry limit.
- **Provenance label** — the marker on a published change saying that it was automatically published, with its published time.
- **Last updated / freshness** — the timestamp on each record showing how current it is.
- **Correction** — a new event that references the original change and leaves it in place.
- **Operational journey** — a daily instance of a scheduled service, generated from the service the Scheduler created.
- **Data classification** — the rule that the provider receives railway operational data only: no passenger personal data is shared with it, and none enters audit or diagnostic logs.

## End-to-end scenarios this role takes part in

**Platform change everywhere** — the Network Controller publishes a platform change; the passenger app and the station display show it with its time and provenance label, and the simulated catering provider receives it as a status-change event.

**Wrong cancellation corrected** — the Network Controller publishes a wrong cancellation and then corrects it; the correction reaches the provider as a new event referencing the original, the journey is restored on the provider's side, and the audit keeps the original.

---

## Journeys

### J1 — Connect to SmartRail

| # | Stage | Provider does | SmartRail responds | Pain addressed |
|---|---|---|---|---|
| 1 | Credential | Receives an API key **[A: issuer TBC]** | Key scoped to provider endpoints only | Unstructured data access |
| 2 | Pull reference | Requests stations and platforms | Station list with platforms | — |
| 3 | Subscribe | Registers a webhook URL | Subscription active | Changes not pushed |

### J2 — Plan the day's preparation

1. Pulls today's journeys calling at its station.
2. Gets scheduled and expected arrival, platform and status per journey.
3. Kitchen screen in the simulated app lists them in arrival order.

### J3 — A train is delayed or re-platformed

1. Controller publishes a change.
2. Provider receives a webhook with event ID, sequence number, journey, stop, new expected arrival/platform.
3. Kitchen shifts preparation or moves the handover point.

### J4 — A train is cancelled, then un-cancelled (end-to-end scenario: Wrong cancellation corrected)

1. Cancellation webhook arrives once → kitchen stops preparation; cause is clearly the railway.
2. Correction webhook referencing the original arrives → journey restored.

### J5 — Delivery problems

1. Provider endpoint is down → SmartRail retries.
2. A duplicate is delivered → ignored by event ID.
3. Events arrive out of order → the provider applies them by sequence; older events don't overwrite newer state.

---

## User stories

### CAT-01 Key-authenticated access — Must
As a catering provider, I want to authenticate with an API key, so that only I get my railway data.
- No/invalid/revoked key → 401; key grants read access to provider endpoints only (least privilege).
**Delivers:** least-privilege API-key access to provider endpoints

### CAT-02 Stations and platforms — Must
As a catering provider, I want to pull stations and their platforms, so that I know where handovers can happen.
**Delivers:** stations and platforms the provider can pull

### CAT-03 Journey data — Must
As a catering provider, I want journeys calling at a station with scheduled and expected arrival, platform and status, so that I can time preparation.
- Filter by station and date **[A]**.
- Each record carries a last-updated time.
**Delivers:** journey arrival data with status and freshness

### CAT-04 Webhook subscription — Must
As a catering provider, I want to register a webhook endpoint, so that changes are pushed to me instead of polled.
**Delivers:** a webhook subscription for pushed changes

### CAT-05 Status-change events — Must
As a catering provider, I want delay, cancellation and platform-change events pushed to me, so that my kitchen stays in step with the train.
- Each event: event ID, sequence number, type, journey, stop, new values, published time, provenance label.
- Platform change reaches me in the same flow as passengers (end-to-end scenario: Platform change everywhere).
**Delivers:** delay, cancellation and platform-change events pushed to the provider

### CAT-06 Duplicate handling — Must
As a catering provider, I want duplicate deliveries ignorable, so that one cancellation isn't processed twice.
- Same event ID on replay → no state change.
**Delivers:** safe handling of duplicate deliveries

### CAT-07 Ordering by sequence — Must
As a catering provider, I want events ordered by sequence number, so that a late-arriving old event can't overwrite newer state.
- Out-of-order event with lower sequence than last applied for that journey is not applied.
**Delivers:** events applied in sequence order

### CAT-08 Retry on failure — Must
As a catering provider, I want SmartRail to retry failed deliveries, so that a brief outage on my side doesn't lose a change.
- Retries with backoff until success or a retry limit **[A]**; failures counted in metrics.
**Delivers:** reliable delivery with retries and failure metrics

### CAT-09 Corrections — Must
As a catering provider, I want corrections pushed with a reference to the original event, so that a wrong cancellation doesn't cost me an order.
**Delivers:** corrections that reference the original event

### CAT-10 Signed deliveries — Should **[A]**
As a catering provider, I want webhook payloads signed, so that I can reject spoofed events.
**Delivers:** signed webhook payloads

### CAT-11 No passenger data — Must
As a catering provider, I want to receive railway data only, so that I hold no passenger personal data in MVP 1.
**Delivers:** railway data only, with no passenger data

### CAT-12 Rate limits — Must
As SmartRail, I want provider endpoints rate-limited, so that one client can't degrade the service.
- Over-limit requests → 429.
**Delivers:** rate-limited provider endpoints

### CAT-13 Simulated provider app — Must
As the provider's kitchen team, I want a screen listing incoming journeys and live changes, so that I can see what to prepare and when.
- Shows journeys with expected arrival, platform, status and received events in real time.
**Delivers:** a simulated kitchen screen showing journeys and live changes

### CAT-14 Menu feed with prep time — Could (optional in MVP 1)
As a catering provider, I want to publish menu items with preparation time, so that later ordering can gate on it.
**Delivers:** an optional menu feed carrying preparation times
