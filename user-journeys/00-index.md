# SmartRail MVP 1 — User Journeys and User Stories (Index)

SmartRail MVP 1 is a single source of truth for live train running on a small network: two lines of five stations each, running in both directions and sharing one interchange station, nine stations in total, all loaded as seed data. A change is made once and reaches the passenger app, a station platform display, the crew app and an external catering provider, each copy carrying a timestamp and a provenance label; every staff change is audited and only possible for the right role.

Each file below is written to be read on its own: it carries the context, the scope and the end-to-end picture a reader needs without referring to another planning document. Anything inferred rather than found in the sources is marked **[A]**.

## Stakeholders in MVP 1

| File | Stakeholder | Persona | Surface | Story IDs |
|---|---|---|---|---|
| `01-scheduler.md` | Scheduler | Nadini | Operations web | SCH-01…11 |
| `02-network-controller.md` | Network Controller | Mahesh | Operations web | NC-01…10 |
| `03-guard.md` | Guard | Ishara | Crew mobile app | GRD-01…09 |
| `04-driver.md` | Driver | Nuwan | Crew mobile app (read-only) | DRV-01…07 |
| `05-system-admin.md` | System Admin | Chathura | Operations web | ADM-01…07 |
| `06-auditor.md` | Auditor | Wasantha | Operations web (read-only) | AUD-01…07 |
| `07-passenger.md` | Passenger (public, optional account) | Nimali, Hannah, Ishan, Farah, Dinesh, Poorvi | Passenger app, station display | PAX-01…24 |
| `08-catering-provider.md` | Catering provider | RailBites | Provider API, simulated provider app | CAT-01…14 |

## Stakeholders not in MVP 1

| Stakeholder | Reason |
|---|---|
| Station Master (Rangana) | Platform availability, incident logging and temporary station closure are deferred to MVP 2 by the owner; the Network Controller can still change platforms |
| Customer Information Officer (Tharushi) | Human-authored passenger messaging, including the trilingual draft-and-approve workflow, is a product of its own; MVP 1 proves only the automatically published path and its provenance label |
| Passenger who orders food (Kavindya) | Passenger food discovery, ordering and tracking are out; catering is proved as an integration capability first |

## End-to-end scenarios → stories

| End-to-end scenario | Stories |
|---|---|
| New service reaches passengers — the Scheduler creates a service, daily journeys are generated from it, and the journey appears in the passenger app's departures | SCH-02…08, PAX-02 |
| Offline guard logs sync once — a Guard logs departures while offline, reconnects, the logs sync exactly once with their original timestamps, and the passenger app and display show updated progress | GRD-02, GRD-05…07, PAX-04, PAX-14 |
| Platform change everywhere — the Network Controller publishes a platform change; the passenger app and station display show it with its time and provenance label, and the simulated catering provider receives it | NC-05, PAX-05, PAX-15, CAT-05 |
| Separation of duties denials — a Scheduler edit to a journey inside the 24-hour window is rejected by the API and a Guard token attempting to publish a disruption gets 403; both denials are audited | SCH-10, NC-10, AUD-04 |
| Wrong cancellation corrected — the Network Controller publishes a wrong cancellation and then corrects it; every channel, including the provider, shows the correction and the audit keeps the original | NC-04, NC-06, PAX-12, CAT-09, AUD-03 |

## Story format

Each story: `As a … I want … so that …`, Given/When/Then acceptance criteria, and a **Delivers** line naming the capability the story provides. Priority within MVP 1: **Must** = needed for an end-to-end scenario or for the capability to be considered done; **Should** = in MVP 1 but can slip last.
