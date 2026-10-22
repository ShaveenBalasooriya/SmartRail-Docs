# SmartRail MVP 1 — User Journeys and User Stories (Index)

SmartRail MVP 1 is a single source of truth for live train running on a small network: two lines of five stations each, running in both directions and sharing one interchange station, nine stations in total. The network and its timetable are loaded as seed data, and each day's journeys are generated from the timetable. A change is made once and reaches the passenger app, a station platform display, the Guard's app and an external catering provider, each copy carrying a timestamp and a provenance label; every staff change is audited and only possible for the right role.

Each file below is written to be read on its own: it carries the context, the scope and the end-to-end picture a reader needs without referring to another planning document. Anything inferred rather than found in the sources is marked **[A]**.

## Stakeholders in MVP 1

| File | Stakeholder | Persona | Surface | Story IDs |
|---|---|---|---|---|
| `02-network-controller.md` | Network Controller | Mahesh | Operations web | NC-01…08 |
| `03-guard.md` | Guard | Ishara | Crew mobile app | GRD-01…09 |
| `06-auditor.md` | Auditor | Wasantha | Operations web (read-only) | AUD-01…07 |
| `07-passenger.md` | Passenger (public, no account) | Nimali, Hannah, Ishan, Farah, Dinesh, Poorvi | Passenger app, station display | PAX-01…07, PAX-09…16 |
| `08-catering-provider.md` | Catering provider | RailBites | Provider API, simulated provider app | CAT-01…12 |

## Stakeholders not in MVP 1

| Stakeholder | Reason |
|---|---|
| Scheduler (Nadini) | The timetable is loaded as seed data, so no one edits the plan in MVP 1 and there is no 24-hour boundary to enforce |
| Driver (Nuwan) | The Guard's app is MVP 1's operational mobile app; the Driver's read-only view and notice acknowledgement come later |
| System Admin (Chathura) | Staff accounts are loaded as seed data, one role per account, so there is no account management in MVP 1 |
| Passenger with an account | Optional passenger accounts, the journey planner, saved journeys and watch-and-alert come later; MVP 1 passengers use public information only |
| Station Master (Rangana) | Platform availability, incident logging and temporary station closure come later; the Network Controller can still change platforms |
| Customer Information Officer (Tharushi) | Human-authored passenger messaging, including the trilingual draft-and-approve workflow, is a product of its own; MVP 1 proves only the automatically published path and its provenance label |
| Passenger who orders food (Kavindya) | Passenger food discovery, ordering and tracking are out; catering is proved as an integration capability first |

## End-to-end scenarios → stories

| End-to-end scenario | Stories |
|---|---|
| Delay reaches every channel — the Network Controller publishes a delay on a running journey; expected times at every later stop update in the passenger app, the station display and the Guard's app, and the simulated catering provider receives the event | NC-03, PAX-04, PAX-14, GRD-08, CAT-05 |
| Offline guard logs sync once — a Guard logs departures while offline, reconnects, the logs sync exactly once with their original timestamps, and the passenger app and display show updated progress | GRD-02, GRD-05…07, PAX-04, PAX-14 |
| Platform change everywhere — the Network Controller publishes a platform change; the passenger app and station display show it with its time and provenance label, and the simulated catering provider receives it | NC-05, PAX-05, PAX-15, CAT-05 |
| Separation of duties denials — a Guard token attempting to publish a disruption and an Auditor token attempting any change both get 403; both denials are audited | GRD-09, NC-08, AUD-04, AUD-06 |
| Wrong cancellation corrected — the Network Controller publishes a wrong cancellation and then corrects it; every channel, including the provider, shows the correction and the audit keeps the original | NC-04, NC-06, PAX-12, CAT-09, AUD-03 |

## Story format

Each story: `As a … I want … so that …`, Given/When/Then acceptance criteria, and a **Delivers** line naming the capability the story provides. Priority within MVP 1: **Must** = needed for an end-to-end scenario or for the capability to be considered done; **Should** = in MVP 1 but can slip last.
