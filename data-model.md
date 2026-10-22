# SmartRail — MVP 1 Data Model

This is the data model for SmartRail MVP 1. It follows the architecture in `docs/architecture.md`: six
modules, one PostgreSQL schema for each module that keeps data, and no foreign key that crosses a
schema boundary. It covers only what the lean `docs/scoping/mvp-1-scope.md` and `docs/user-journeys/`
put in MVP 1. Anything inferred rather than found in those sources is marked **[A]**.

**Status:** draft for the owner's review. Revised on 2026-10-19 for the lean MVP 1 cut: the timetable is
seed data, the only staff roles are the Network Controller, the Guard and the Auditor, there are no
passenger accounts, and other modules read live state through module contracts rather than copied
tables. The model went from 28 tables to 18. On 2026-10-21 (event bus, no message broker in MVP 1,
`architecture.md` §3) it gained the event bus outbox, `outbox_messages` and `outbox_deliveries` in each
of the three publishing schemas **[A]**, so it now has 24.

---

## How to view it in Eraser

Eraser's entity-relationship diagrams use Eraser's own diagram-as-code syntax, not JSON Schema. Create
a new **Entity Relationship Diagram** in Eraser and paste the code block from the last section of this
document into its code editor.

**Reading the diagram**

- **The prefix is the schema.** `network_journeys` is the `journeys` table in the `network` schema.
  Eraser can't use dots in entity names, so `schema.table` is written `schema_table`.
- **The colour is the module.** Purple is Identity & Access, blue is Rail Network, orange is Live
  Operations, red is Catering Integration, and grey is Audit. Distribution keeps no data, so it has
  no tables.
- **There are two blocks of relationships.** The first block is real foreign keys inside one schema.
  The second is references between modules. Those are stored as plain IDs with **no foreign key**,
  and they are drawn only so the diagram shows how the modules connect. Delete the second block to see
  only the enforced structure.
- Types such as `uuid`, `timestamp` and `jsonb` are PostgreSQL types. The values of each `enum` column
  are listed in the module notes below.

---

## Module notes

### Identity & Access (`identity`)

- **This schema holds no credentials.** Authentik holds every staff login. SmartRail keeps
  `authentik_subject`, the OIDC `sub` claim, to link a token to a local record. Staff accounts are
  seed data, defined in the Authentik blueprint with matching rows here.
- **There is no role column.** The role is membership of one Authentik group (`network_controller`,
  `guard` or `auditor`), read from the token's `groups` claim; a token naming none or more than one is
  refused, which keeps "one role per account" (`mvp-1-scope.md` EP-02) (owner decision, 2026-10-20).
  The staff row exists so that changes and audit entries reference a stable staff ID.
- No deactivation columns: account management is deferred with the System Admin.
- `provider_api_keys` stores only a hash of the key, plus a short prefix that identifies which key was
  used. The key is seed data (owner decision). `revoked_at` lets a key be refused with 401 (CAT-01). The
  architecture places API keys in this module, so the table lives here even though the provider itself
  is a Catering record.

### Rail Network (`network`)

- `lines`, `stations`, `platforms` and `line_stations` are the seeded network. `line_stations.sequence`
  stores each line's station order once. The reverse direction is the same order read backwards
  **[A]**. The interchange is not a stored flag. It is the station that appears on both lines.
- `services` is the seeded timetable. `direction` is `outbound` (in sequence order) or `inbound`
  (reverse order) **[A]**, because the sources don't name the directions. `guard_staff_id` is the
  guard assigned in the seed data. There are no running-day columns, because running-day calendars are
  out of MVP 1 (owner decision): every service runs every day.
- `service_stops` keeps a row for every station on the line, including skipped ones, with
  `is_calling = false`, so passengers see only true calling stops (PAX-03). Times are times of day. A
  service that runs past midnight is not handled **[A]**.
- `journeys` is one row per service per day, generated from the timetable and unique on
  (`service_id`, `service_date`). Generation can therefore run again without creating duplicates.
  `scheduled_departure_at` is the first stop's departure, and the Controller's view of today's journeys
  is ordered by it. There are no journey-level stop or crew tables. Those copies existed so that a
  Scheduler could change a single date, and the Scheduler is deferred. Each journey's stops and guard
  are those of its service.

### Live Operations (`operations`)

- `journey_live_states` and `journey_stop_live_states` hold each journey's live state. They are created
  from Rail Network's journey-generated event, with the service's scheduled times, platforms and guard
  copied in as the baseline. Live Operations then owns every change to them.
  - `lifecycle_status` is `scheduled`, `running`, `cancelled`, `completed` or `terminated` (ended early
    by the Network Controller) **[A]**.
  - `last_updated_at` is the freshness time shown on every channel.
  - `guard_staff_id` is checked so that only the assigned guard sees the journey and logs its departures
    (GRD-01).
- `departure_logs.id` is **generated on the device**. The server accepts each ID once, which makes the
  offline sync idempotent (GRD-07).
  - `departed_at` is the device time at the tap, and `received_at` is the server time at sync.
    Progress is computed from `departed_at`.
  - A unique constraint on (`journey_id`, `stop_sequence`) means each stop can be logged once **[A]**
    (GRD-02).
  - `delay_cause` is optional, from a fixed list (GRD-03). The sources don't define the values yet
    **[A]**.
- `status_changes` holds the Network Controller's published changes.
  - `change_type` is `delay`, `cancellation`, `termination`, `platform_change` or `correction`.
    `termination` ends a running journey early (owner decision 2026-10-21) **[A]**. Skipped stops are
    deferred, so they are not in the list.
  - A correction is a new row whose `corrects_change_id` points at the original, so the original is
    never overwritten (NC-06).
  - For a platform change, `stop_sequence` is the stop that changed. For a delay, it is the first stop
    affected **[A]**. For a termination, it is the last stop served **[A]**.
  - `provenance` has one value in MVP 1, `automatically_published`. `human_authored` arrives with
    Customer Information Officer messaging.
- `journey_events` is the ordered event stream that `mvp-1-scope.md` EP-01 calls the domain event log.
  Every departure log and every status change produces one event.
  - `sequence_number` orders the events within one journey, and catering webhooks carry it (CAT-07).
  - `stream_position` orders all events across the system. It is the SSE event ID, so a client that
    reconnects resumes from the last position it saw (EP-09).
  - It is the query and catch-up store, not the bus. Other modules read it through `ILiveJourneys`, and
    SSE `Last-Event-ID` resume reads it. Cross-module delivery of a new event goes through the outbox
    (below), whose `JourneyStateChanged` message names the event.

### Distribution (no schema)

Distribution keeps no data. It serves the public passenger and display API by calling Live Operations
(live state) and Rail Network (station names, platform numbers, line order) through their in-process
contracts. It holds open SSE connections in memory. Catch-up after a reconnect reads `journey_events`
from the client's last `stream_position`. The last-known cache and the stale marker are on the client
(EP-09).

### Catering Integration (`catering`)

- `providers` is the external provider: RailBites in MVP 1.
- `webhook_subscriptions` is the callback URL a provider registers (CAT-04). Signed webhooks are deferred
  **[A]**, so no signing secret is stored.
- `webhook_deliveries` holds one row per event per subscription, unique on (`subscription_id`,
  `event_id`).
  - It drives retry with backoff through `attempt_count` and `next_attempt_at` (CAT-08). The webhook
    delivery-failure metric is counted from it (EP-17).
  - `status` is `pending`, `delivered`, `failed` or `abandoned` (the retry limit was reached) **[A]**.
  - Only delay, cancellation, early termination, platform change and correction events are pushed
    (CAT-05, CAT-09).
    Departure events are not, so a provider will see gaps in `sequence_number`. That is expected: the
    ordering rule only needs the numbers to increase.
- `webhook_deliveries` rows are created by Catering's consumer of the `JourneyStateChanged` bus message,
  and the unique (`subscription_id`, `event_id`) index makes a redelivered message harmless **[A]**.
- The provider's pull API (CAT-02, CAT-03) has no table of its own. It reads stations and platforms
  from Rail Network, and journeys and expected arrivals from Live Operations, through their contracts.
- No passenger data appears anywhere in this schema (CAT-10).

### Event bus outbox (`identity`, `network`, `operations`)

Added 2026-10-21 (owner decision, `architecture.md` §3 "Event delivery"). Every module that publishes a
cross-module event keeps two tables in its own schema, so an event is saved in the same transaction as the
change and no write crosses a schema boundary. Today those modules are Identity (`AccessDenied`), Rail
Network (`JourneyGenerated`) and Live Operations (`JourneyStateChanged`) **[A]**.

- `outbox_messages` holds one row per published event: `id` is the event ID, `position` the dispatch
  order, `ordering_key` the journey ID for journey events (null otherwise), `payload` the event as JSON,
  and `published_at` is null until the dispatcher hands it to the transport.
- `outbox_deliveries` holds one row per message and consumer, unique on (`message_id`, `consumer`), with
  `status` (`pending`, `delivered` or `failed`), `attempt_count` and `next_attempt_at`. It is what makes one
  failing consumer independent of the others, and it drives retry with backoff.
- Neither table is read by another module. A NATS JetStream transport would keep `outbox_messages` as the
  relay's source and replace `outbox_deliveries`.

### Audit (`audit`)

- `entries` is append-only. No role has an update or delete path, at the application level or in the
  database.
  - Each entry records the actor (`actor_staff_id`), the actor's role at the time (`actor_role`), the
    `action`, the `outcome` (`succeeded` or `denied`), the target, `before` and `after`, and the server
    timestamp `occurred_at` (AUD-03, AUD-04).
  - Denials come from Guard and Auditor tokens attempting writes outside their role (NC-08, GRD-09,
    AUD-06).
- **Hash chain (owner decision):** one chain across all entries, in `chain_position` order. `entry_hash` is the hash
  of the entry's content plus `previous_hash`, so changing a row breaks every hash after it, and the
  check reports the first position that fails (AUD-05).
- `related_entry_id` links a correction's entry to the original change's entry (AUD-03).
- `source_event_id` is the bus event ID of the denial that produced the entry; it is null for staff state
  changes, unique, and not part of the hash, so a redelivered denial creates no second entry **[A]**.
- Staff actions only. There are no passenger accounts in MVP 1, so there is no passenger actor to
  record, and M-03 forbids logging passenger activity in any case.

---

## Eraser diagram code

```
// ============================================================
// SmartRail MVP 1 (lean) — Entity Relationship Diagram
// Prefix = PostgreSQL schema. Colour = module.
// Distribution keeps no data, so it has no tables.
// ============================================================

// ---------- Identity & Access (schema: identity) ----------

identity_staff_members [icon: user, color: purple] {
  id uuid pk
  authentik_subject string
  username string
  display_name string
  created_at timestamp
}

identity_provider_api_keys [icon: key, color: purple] {
  id uuid pk
  provider_id uuid
  key_prefix string
  key_hash string
  created_at timestamp
  revoked_at timestamp
}

// ---------- Rail Network (schema: network) ----------

network_lines [icon: git-commit, color: blue] {
  id uuid pk
  code string
  name string
}

network_stations [icon: map-pin, color: blue] {
  id uuid pk
  code string
  name string
}

network_platforms [icon: hash, color: blue] {
  id uuid pk
  station_id uuid
  number string
}

network_line_stations [icon: list, color: blue] {
  id uuid pk
  line_id uuid
  station_id uuid
  sequence int
}

network_services [icon: calendar, color: blue] {
  id uuid pk
  line_id uuid
  direction enum
  name string
  guard_staff_id uuid
}

network_service_stops [icon: list, color: blue] {
  id uuid pk
  service_id uuid
  station_id uuid
  sequence int
  is_calling bool
  scheduled_arrival time
  scheduled_departure time
  platform_id uuid
}

network_journeys [icon: navigation, color: blue] {
  id uuid pk
  service_id uuid
  service_date date
  scheduled_departure_at timestamp
  created_at timestamp
}

// ---------- Live Operations (schema: operations) ----------

operations_journey_live_states [icon: activity, color: orange] {
  journey_id uuid pk
  service_id uuid
  line_id uuid
  service_date date
  guard_staff_id uuid
  lifecycle_status enum
  delay_minutes int
  last_departed_station_id uuid
  next_station_id uuid
  last_sequence_number bigint
  last_updated_at timestamp
}

operations_journey_stop_live_states [icon: list, color: orange] {
  id uuid pk
  journey_id uuid
  sequence int
  station_id uuid
  is_calling bool
  scheduled_arrival timestamp
  scheduled_departure timestamp
  expected_arrival timestamp
  expected_departure timestamp
  actual_departure timestamp
  platform_id uuid
  platform_changed_at timestamp
}

operations_departure_logs [icon: check-circle, color: orange] {
  id uuid pk
  journey_id uuid
  stop_sequence int
  station_id uuid
  guard_staff_id uuid
  departed_at timestamp
  delay_cause enum
  received_at timestamp
}

operations_status_changes [icon: alert-triangle, color: orange] {
  id uuid pk
  journey_id uuid
  change_type enum
  stop_sequence int
  delay_minutes int
  new_platform_id uuid
  previous_platform_id uuid
  corrects_change_id uuid
  provenance enum
  published_by uuid
  published_at timestamp
}

operations_journey_events [icon: zap, color: orange] {
  id uuid pk
  stream_position bigint
  journey_id uuid
  sequence_number bigint
  event_type enum
  status_change_id uuid
  departure_log_id uuid
  payload jsonb
  occurred_at timestamp
  recorded_at timestamp
}

// ---------- Catering Integration (schema: catering) ----------

catering_providers [icon: coffee, color: red] {
  id uuid pk
  name string
  is_active bool
  created_at timestamp
}

catering_webhook_subscriptions [icon: link, color: red] {
  id uuid pk
  provider_id uuid
  callback_url string
  is_active bool
  created_at timestamp
}

catering_webhook_deliveries [icon: send, color: red] {
  id uuid pk
  subscription_id uuid
  event_id uuid
  journey_id uuid
  sequence_number bigint
  event_type enum
  payload jsonb
  status enum
  attempt_count int
  next_attempt_at timestamp
  last_attempt_at timestamp
  last_response_status int
  delivered_at timestamp
  created_at timestamp
}

// ---------- Audit (schema: audit) ----------

audit_entries [icon: shield, color: gray] {
  id uuid pk
  chain_position bigint
  occurred_at timestamp
  actor_staff_id uuid
  actor_role enum
  action string
  outcome enum
  target_type string
  target_id string
  before jsonb
  after jsonb
  related_entry_id uuid
  source_event_id uuid
  previous_hash string
  entry_hash string
}

// ---------- Event bus outbox (one pair per publishing schema: identity, network, operations) ----------

identity_outbox_messages [icon: inbox, color: purple] {
  id uuid pk
  position bigint
  event_type string
  ordering_key string
  payload jsonb
  occurred_at timestamp
  published_at timestamp
}

identity_outbox_deliveries [icon: send, color: purple] {
  message_id uuid
  consumer string
  status enum
  attempt_count int
  next_attempt_at timestamp
  last_attempt_at timestamp
  delivered_at timestamp
}

network_outbox_messages [icon: inbox, color: blue] {
  id uuid pk
  position bigint
  event_type string
  ordering_key string
  payload jsonb
  occurred_at timestamp
  published_at timestamp
}

network_outbox_deliveries [icon: send, color: blue] {
  message_id uuid
  consumer string
  status enum
  attempt_count int
  next_attempt_at timestamp
  last_attempt_at timestamp
  delivered_at timestamp
}

operations_outbox_messages [icon: inbox, color: orange] {
  id uuid pk
  position bigint
  event_type string
  ordering_key string
  payload jsonb
  occurred_at timestamp
  published_at timestamp
}

operations_outbox_deliveries [icon: send, color: orange] {
  message_id uuid
  consumer string
  status enum
  attempt_count int
  next_attempt_at timestamp
  last_attempt_at timestamp
  delivered_at timestamp
}

// ============================================================
// Relationships INSIDE a schema (real foreign keys)
// ============================================================

network_platforms.station_id > network_stations.id
network_line_stations.line_id > network_lines.id
network_line_stations.station_id > network_stations.id
network_services.line_id > network_lines.id
network_service_stops.service_id > network_services.id
network_service_stops.station_id > network_stations.id
network_service_stops.platform_id > network_platforms.id
network_journeys.service_id > network_services.id

operations_journey_stop_live_states.journey_id > operations_journey_live_states.journey_id
operations_departure_logs.journey_id > operations_journey_live_states.journey_id
operations_status_changes.journey_id > operations_journey_live_states.journey_id
operations_status_changes.corrects_change_id > operations_status_changes.id
operations_journey_events.journey_id > operations_journey_live_states.journey_id
operations_journey_events.status_change_id - operations_status_changes.id
operations_journey_events.departure_log_id - operations_departure_logs.id

catering_webhook_subscriptions.provider_id > catering_providers.id
catering_webhook_deliveries.subscription_id > catering_webhook_subscriptions.id

audit_entries.related_entry_id > audit_entries.id

identity_outbox_deliveries.message_id > identity_outbox_messages.id
network_outbox_deliveries.message_id > network_outbox_messages.id
operations_outbox_deliveries.message_id > operations_outbox_messages.id

// ============================================================
// Cross-module references (ID only, NO foreign key)
// Shown for readability. Delete this block to see enforced FKs only.
// ============================================================

identity_provider_api_keys.provider_id > catering_providers.id

network_services.guard_staff_id > identity_staff_members.id

operations_journey_live_states.journey_id - network_journeys.id
operations_journey_live_states.guard_staff_id > identity_staff_members.id
operations_departure_logs.guard_staff_id > identity_staff_members.id
operations_status_changes.published_by > identity_staff_members.id
operations_status_changes.new_platform_id > network_platforms.id

catering_webhook_deliveries.event_id > operations_journey_events.id

audit_entries.actor_staff_id > identity_staff_members.id
```
