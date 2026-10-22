# Passenger (public) — MVP 1 Journeys and Stories

**Surfaces:** Passenger mobile app (no account), station platform display.

## Context

SmartRail MVP 1 is a single source of truth for live train running on a small network: two lines of five stations each, running in both directions and sharing one interchange station, nine stations in total. The network and its timetable are loaded as seed data, and each day's journeys are generated from the timetable. A change is made once and reaches the passenger app, a station platform display, the Guard's app and an external catering provider, each copy carrying a timestamp and a provenance label; every staff change is audited and only possible for the right role. The passenger is the public user of this system, reading the same live railway data from the passenger mobile app or, at the station, from the platform display, with no account and no sign-in. They own nothing in the system; they use it to answer their journey questions — which train to take, whether it is late, where it is now, which platform, and what to do when something goes wrong. The core pain addressed is changes learned late, channels that disagree, wrong platforms, no guidance when things go wrong, and exclusion when there is only one channel. Their information depends on the seeded timetable being turned into daily journeys, the Guard's departure logs for live progress, and the Network Controller's published changes and corrections.

**Personas covered:** Nimali (commuter), Hannah (tourist), Ishan (vision loss), Farah (hearing loss), Dinesh (cognitive), Poorvi (mobility).

## Key terms

- **Timetabled service** — a train in the seeded timetable on a line and direction: its stops, times and platforms.
- **Operational journey** — the daily instance generated from a timetabled service; this is what a passenger follows and what the Network Controller changes.
- **Lifecycle status** — scheduled, running, cancelled or completed.
- **Provenance label** — the marker that says a change was "automatically published", with the time it was published.
- **Freshness / stale marker** — the "last updated" time on live data, and the mark shown when a client can no longer confirm the current state.
- **Correction** — a new event that references the original change and leaves the original in place.
- **Interchange station** — the one station the two lines share, where a passenger changes from one line to the other.
- **Calling stops** — the full list of stations a journey stops at.

## End-to-end scenarios this role takes part in

**Delay reaches every channel** — the Network Controller publishes a delay on a running journey; expected times at every later stop update in the app and on the station display, and the Guard's app and the simulated catering provider receive it too.

**Offline guard logs sync once** — a Guard logs departures while offline, reconnects, and the logs sync exactly once with their original timestamps; the app and the display then show updated progress.

**Platform change everywhere** — the Network Controller publishes a platform change; the app and the station display show it prominently with its time and provenance label, and the simulated catering provider receives it too.

**Wrong cancellation corrected** — the Network Controller publishes a wrong cancellation and then corrects it; every channel, including the provider, shows the correction and the audit keeps the original.

---

## Journeys

### J1 — Morning commute check (Nimali)

| # | Stage | Passenger does | System responds | Pain addressed |
|---|---|---|---|---|
| 1 | Open | Opens app, no sign-in | Last-used stations **[A]** | Login barrier |
| 2 | Search | Picks station A → B | Departures: scheduled, expected, ±min, platform, status, last updated | "Is it on time?" |
| 3 | Re-check | Opens the app again while walking to the station | Her train flagged: "Platform changed 2 → 3 at 07:41" | Learns at the station |
| 4 | Platform | Arrives | Display shows platform 3 flagged as changed | Display vs app mismatch |

### J2 — Following a train live (Hannah)

1. Opens journey detail: full calling stops, last station passed, next station, ETA, minutes late.
2. Checks her station is a calling stop before boarding.
3. Progress updates as the guard logs departures.

### J3 — Cancellation (Hannah)

1. Opens her journey and finds it cancelled.
2. Journey shows `Cancelled`, a plain-language "what to do next" and the next service to the same destination.
3. Label shows the change was automatically published and when.
4. If it was published in error, a correction restores the journey with a note that it was corrected.

### J4 — Change lines at the interchange (Nimali, Hannah)

1. Opens departures from the interchange station.
2. Sees onward departures on the other line with expected times and platforms.

### J5 — Weak signal on the platform (all)

1. Connection drops; app and display keep last-known departures marked **stale** with last-updated time.
2. On reconnect, both catch up automatically.

### J6 — Accessible use of the same journeys

| Persona | MVP 1 path | Known gap in MVP 1 |
|---|---|---|
| Ishan (vision) | App with screen reader; semantic order; text status | Station display has no audio (deferred) |
| Farah (hearing) | Every change is text on app and display | — |
| Dinesh (cognitive) | Plain template wording; change stays on screen with its time | No "what changed since you looked" (old → new) marker |
| Poorvi (mobility) | Platform change shown in the app before she reaches the station | No step-free or lift data; no alert unless she opens the app |

---

## User stories — Passenger app

### PAX-01 Live information without an account — Must
As a passenger, I want live information without signing in, so that nothing stands between me and my answer.
- Departures, journey detail, live progress and the station display use public endpoints only; there is no sign-in.
**Delivers:** live information without signing in

### PAX-02 Departures A → B — Must
As a passenger, I want departures from my station to my destination, so that I know which train to take.
- Each row: scheduled time, expected time, minutes early/late, platform, status, last-updated time.
- Only journeys calling at both stations are listed.
**Delivers:** station-to-station departures with times, platform, status and freshness

### PAX-03 Calling stops — Must
As a passenger, I want the full stop list for a journey, so that I don't board a train that skips my station.
**Delivers:** the full calling-stop list for a journey

### PAX-04 Live progress — Must
As a passenger, I want last station passed, next station and ETA, so that I know where my train is now.
- Updates when the guard's departure logs arrive (end-to-end scenario: Offline guard logs sync once).
**Delivers:** live progress derived from the guard's departure logs

### PAX-05 Platform change flag — Must
As a passenger, I want platform changes flagged prominently with the time they became known, so that I go to the right platform.
- Flag uses text and icon, never colour alone.
**Delivers:** platform changes flagged with the time they became known

### PAX-06 What to do next — Must
As a passenger, I want disruption notices to tell me what to do and the next service to my destination, so that I can decide to wait or re-plan.
- Template text per type (delay, cancellation, platform change) **[A]** plus next service to the same destination when one exists.
**Delivers:** plain guidance on what to do next after a disruption

### PAX-07 Departures from any station — Must
As a passenger, I want onward departures from any station including the interchange, so that I can see my connection.
**Delivers:** onward departures from any station, including the interchange

### PAX-09 Offline honesty — Must
As a passenger, I want last-known data marked stale when I'm offline, so that I'm never misled or shown a blank screen.
**Delivers:** last-known data marked stale rather than hidden

### PAX-10 Provenance and freshness — Must
As a passenger, I want to see when a change was published and that it was automatically published, so that I can trust which source is current.
**Delivers:** the time and provenance of every published change

### PAX-11 Accessible by default — Must
As a passenger with a disability, I want screen-reader support, scalable text, sufficient contrast and no meaning by colour alone, so that I get the same information as everyone.
- Passes automated accessibility check.
**Delivers:** an accessible passenger experience

### PAX-12 See corrections — Must
As a passenger, I want a wrongly published change to be corrected on my screen, so that I'm not left with a false cancellation.
- Corrected state replaces the wrong one with a "corrected at hh:mm" note **[A]** (end-to-end scenario: Wrong cancellation corrected).
**Delivers:** corrections shown on every passenger screen

## User stories — Station platform display

### PAX-13 Next departures on the platform — Must
As a passenger at the station, I want a large, high-contrast list of next departures for this platform, so that I can read it from a distance.
- Same data as the app: destination, scheduled, expected, ±min, status, last updated.
**Delivers:** large, high-contrast next departures on the platform

### PAX-14 Progress reflected on display — Must
As a passenger, I want the display's expected times to update as the train progresses, so that it agrees with the app.
**Delivers:** display times that agree with the app

### PAX-15 Platform change on display — Must
As a passenger, I want a platform change shown prominently on the display, so that I see it even without my phone.
- Shown with time known; stays visible, does not flash away **[A]**.
**Delivers:** a platform change shown prominently on the display

### PAX-16 Display stale and recovery — Must
As a passenger, I want the display to keep showing last-known departures marked stale when it loses connection, and recover on reconnect, so that it never goes blank or lies.
**Delivers:** a display that marks stale data honestly and recovers on reconnect
