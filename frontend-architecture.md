# SmartRail — Frontend Architecture

This document records how SmartRail's frontends are built and why. It extends `architecture.md`, which
covers the backend and does not yet say anything about client technology. Decisions are the owner's;
anything inferred rather than decided outright is marked **[A]**.

**Status:** agreed 2026-10-20, living document. Scope comes from `docs/scoping/mvp-1-scope.md`.

---

## 1. Surfaces

MVP 1 names four client surfaces. The owner has asked to start with the first two.

| Surface | Users | Epic | Auth | Status |
|---|---|---|---|---|
| **Operations web** | Network Controller (read/write), Auditor (read-only) | EP-08, EP-03 | Authentik, staff client | Start now |
| **Passenger mobile app** | Passenger, no account | EP-10 | None, public endpoints | Start now |
| Guard crew app | Guard | EP-06 | Authentik, staff client | Later, reuses the mobile approach **[A]** |
| Station platform display | Public, kiosk | EP-11 | None, public endpoints | Later, small web page **[A]** |

The Guard app is the hardest client (offline queue, exactly-once sync). It is deliberately not started
first.

---

## 2. Decisions

| Area | Decision | Rejected alternative | Why |
|---|---|---|---|
| Operations web | React + Vite + TypeScript, a single-page app | Next.js, Angular, Blazor | A staff-only, authenticated tool gains nothing from server rendering. The native `EventSource` handles SSE with no extra library |
| Passenger mobile | Flutter | React Native + Expo (my recommendation) | Owner choice. Strong UI toolkit and accessibility support, which M-05 needs. Cost accepted: Dart means no code or type sharing with the web app, so the mobile app generates its own API client from OpenAPI |
| Repositories | One repository per app, beside `SmartRail-API` | A monorepo; frontends inside `SmartRail-API` | Owner choice. Keeps .NET, Node and Dart tooling, CI and history separate. Cost accepted: the mobile app's OpenAPI client is generated once per repo |
| Backend gap | Build against the real API where an endpoint exists; mock the rest; switch each mock to the real endpoint as its OpenSpec change lands | Wait for the backend; fully mock and integrate at the end | Integration risk stays small and spread out, instead of concentrated at the end |

Proposed repository names **[A]**: `SmartRail-OpsWeb` and `SmartRail-PassengerApp`.

---

## 3. Shared principles

- **The OpenAPI document is the contract.** The backend serves `/openapi/v1.json`. The mobile app
  generates its typed API client from it, so a backend change that breaks that client fails at
  generation or compile time, not in the demo. Generated code is not edited by hand. The Operations
  web generates nothing: its typed services live in `features/<domain>/api`, following the Deskline
  blueprint, and their types are checked against the same document, so a backend change that breaks
  them fails at compile or review time.
- **One source of truth, many channels.** Clients never compute expected times or lifecycle status.
  They render what Distribution returns (MVP 1 goal: a change is made once and reaches every channel).
- **Freshness is always visible (M-02).** Every live item shows "last updated". On a lost
  connection the client keeps the last-known state with a stale marker; no blank screens.
- **Provenance is always visible (M-16, M-09).** A platform change or disruption shows when it became
  known and its "automatically published" label.
- **Live updates use SSE.** On reconnect the client sends `Last-Event-ID` and catches up
  (`architecture.md` §6). A polling fallback is not built **[A]**.
- **Accessibility baseline (M-05) from the first screen:** semantic structure, contrast, scalable
  text, nothing conveyed by colour alone. Design tokens are defined once per client (EP-13).
- **No passenger data in logs or analytics (S-20).**

---

## 4. Operations web

- **Stack:** React, TypeScript, Vite. TanStack Query for server state **[A]**, a router, a component
  library chosen at scaffold time.
- **Auth:** OIDC authorization code with PKCE against the Authentik staff client, using
  `oidc-client-ts` **[A]**. The role comes from the token's `groups` claim (`architecture.md` §5).
  The UI hides what a role may not do, but the API is the enforcement point; a `403` is handled and
  shown, never assumed away.
- **Screens in MVP 1:**
  1. Live board: today's journeys with status, expected times and freshness (M-15).
  2. Journey detail: stops, platforms, and the change history.
  3. Publish: delay, cancellation, platform change (M-16), and issue a correction (S-12).
  4. Audit view, read-only, for the Auditor.
- **Role-driven navigation:** a Controller sees 1 to 3 and may publish; an Auditor sees 1, 2 and 4 with
  no publish controls.

## 5. Passenger mobile app

- **Stack:** Flutter. State management and HTTP/SSE packages chosen at scaffold time.
- **Auth:** none. Public endpoints only (M-06).
- **Screens in MVP 1:**
  1. Next departures from station A to station B (M-07).
  2. Journey detail with calling stops (M-10) and live progress (M-08).
  3. Platform change shown prominently with its time (M-09).
  4. Disruption guidance and onward connections, including the interchange (M-11, M-12).
- **Offline honesty:** last-known departures are cached on the device and shown with a stale marker.
- **Out of MVP 1:** watch and local alert (M-13), accounts, journey planner, food ordering.

---

## 6. Integration plan

Work follows the backend's OpenSpec changes. Each frontend step pairs with the backend change that
makes it real.

| Backend change | Status (2026-10-20) | Frontend work it unlocks |
|---|---|---|
| `add-rail-network-timetable` | Done | Station and journey lists in both clients, against the real API |
| `add-live-running-state` | Done | Live state shapes; still no public endpoint to read them |
| `add-identity-access` | Open | Operations web sign-in and role-driven navigation |
| `add-controller-operations` | Open | Operations web publish and correct screens |
| `add-live-distribution` | Open | Passenger app departures, journey detail and SSE |
| `add-audit-record` | Open | Operations web audit view |
| `add-critical-e2e-flows` | Open | The five flows run across the real clients where practical |

Until a change lands, the client uses a mock for that endpoint that matches the OpenAPI shape
(Mock Service Worker in the web app; a fake repository behind an interface in Flutter **[A]**).

**Done when:** the five end-to-end flows in `mvp-1-scope.md` can be demonstrated through the real
operations web and passenger app against the running stack, with no mocks left.

---

## 7. Open points for the owner

- Component library and visual design for the operations web, and the design tokens for both clients
  (EP-13). A Figma file would help here if you want to design first.
- Flutter state management choice (Riverpod or Bloc), and SSE package.
- Where CI for each new repository runs, and whether they follow the same Graphite and `dev`/`main`
  flow as `SmartRail-API`.
