# Data classification

What SmartRail holds in MVP 1, how sensitive it is, who may see it, and what may be written to
logs. It backs the logging rule in `../architecture.md` §7 and the threat model's information
disclosure rows.

## Classes

| Class | What it covers | Handling |
|---|---|---|
| **Public** | Lines, stations, platforms, the seeded timetable; each journey's live state as passengers see it — expected times, platform, lifecycle status, freshness timestamp, provenance label, guidance text | Served anonymously; no personal data; `Cache-Control: no-store` on responses that carry live state |
| **Operational** | Journey and stop live state with their baselines, departure logs (device time, delay cause), controller changes and corrections, the journey event stream, webhook subscriptions and delivery attempts | Internal to SmartRail; exposed to a provider only through the provider payload, which excludes staff names, staff IDs, guard assignments, audit data and passenger data |
| **Staff** | Staff accounts (`authentik_subject`, username, display name), group membership that assigns the role, the caller's staff ID and role on a request, audit actor IDs and roles | Never on a public or provider response; shown only to the authenticated staff member (`/identity/me`) and, for the audit record, to the Auditor |
| **Provider-shared** | Stations and today's journeys in the provider payload; pushed status events with event ID, sequence number, provenance and correction link; the provider's own callback URL and key prefix | Scoped to the one provider that owns the key; never includes another provider's data |
| **Secret** | Staff sign-in passwords and app passwords, the provider API key's plaintext and secret part, bearer and refresh tokens, authorization codes and PKCE verifiers, Authentik's secret key and database password | Configuration or issuer only. The staff client is public, so it has no OAuth client secret; SmartRail stores an API key prefix and hash and nothing else, and no staff credential is stored at all |

There is no passenger personal data in MVP 1: no accounts, no names, no contact details and no
location beyond what a journey's public progress already shows.

## What each role sees

| Role | Sees | Can change |
|---|---|---|
| Anonymous (passenger app, station display) | Public | Nothing |
| Guard | Their own assigned journeys from the operational class, and their own staff record | Departure logs for their own journeys only; changes to their journey are read-only |
| Network Controller | Today's journeys and their live state, and their own staff record | Delays, cancellations, platform changes and corrections on today's journeys |
| Auditor | The audit record (read-only) and their own staff record | Nothing |
| Catering provider (API key) | Provider-shared data for itself | Its own webhook subscription |
| SmartRail operator (database, hosting) | Everything, out of band | Everything, through tooling outside the API |

Access follows the role catalogue only. A client that hides a button is not a control; the server
refuses the request (see the threat model, rows 2 and 3).

## Logs

Logs are for operating the service, not for reconstructing its data. They may contain:

- correlation and request IDs, the endpoint path, the HTTP method, status code and duration;
- the staff ID and role of an authenticated caller, the journey or station ID being handled, and
  the event type;
- a webhook callback URL **without its query string** and the provider's key prefix, never the
  key itself.

They must never contain:

- an `Authorization` header, a bearer token or any part of a token's payload, and equally a refresh
  token, an authorization code or a PKCE code verifier;
- an `X-Api-Key` value, the plaintext provider key or the secret half of one;
- a staff sign-in password or app password, an OAuth client secret (the public staff client has
  none), Authentik's secret key, or a database password;
- a request or response body from a token or authentication call;
- passenger personal data — there is none in MVP 1, and adding it would require this file to
  change first.

Error reporting follows the same rule: Sentry runs with `SendDefaultPii = false` and a before-send
scrubber that removes headers, cookies and the client IP (`add-critical-e2e-flows` 1.1).
