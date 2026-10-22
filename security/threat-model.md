# Threat model

A STRIDE view of SmartRail's MVP 1 surfaces: what can go wrong at each trust boundary, what
stops it, and which task or test proves the control exists. It covers the scope in
`../scoping/mvp-1-scope.md`; penetration testing and SIEM are out of MVP 1 (EP-15 Out), so this
is a design review, not a substitute for an attack.

## Surfaces and trust boundaries

| Surface | Crosses the boundary | Trusted side |
|---|---|---|
| Staff API (`/api/v1/identity`, operations, guard, audit) | Bearer token issued by Authentik | SmartRail decides the role; Authentik only says who the caller is |
| Public API and SSE streams (`/api/v1/public`, `/api/v1/network`, `/api/v1/timetable`) | Nothing: anonymous reads | Open by design; no passenger accounts in MVP 1 |
| Provider API and webhook delivery (`/api/v1/provider`) | `X-Api-Key`; outbound HTTP to a registered callback | The key identifies one provider; the callback URL is chosen by that provider |
| Database | The API's connection only | One schema per module, no cross-schema foreign keys |
| Authentik | Browser sign-ins with authorization code + PKCE, and machine tokens; its admin console | A single identity provider for all staff |

## Top risks

| # | Surface | STRIDE | Risk | Control | Where it is verified |
|---|---|---|---|---|---|
| 1 | Staff API | Spoofing | A forged, expired, or foreign-issuer token is accepted | Signature, issuer, audience and lifetime are validated against Authentik's published keys; 30 s clock skew; no introspection call per request (design D1) | `add-identity-access` 2.1 (a seeded token works, no token is 401); 3.2 (locally signed tokens) |
| 2 | Staff API | Elevation of privilege | A caller acts outside its one role, or a misconfigured token grants two roles | The role is exactly one catalogue group from the `groups` claim; none or more than one means no role, and every staff endpoint refuses it with 403 (design D2) | `add-identity-access` 3.2 (a token with two role groups gets 403); `add-critical-e2e-flows` 3.4 |
| 3 | Staff API | Elevation of privilege | A provider key is used as a staff credential, or a staff token as a provider credential | Two authentication schemes and endpoint policies that name exactly one of them; the policies are declared per endpoint and a missing declaration is closed, not open (design D4) | `add-identity-access` 3.1 (fallback policy), 4.1 (key accepted and rejected) |
| 4 | Staff API | Repudiation | A staff member denies a change, or a refused change leaves no trace | Every staff write and every denied write is recorded in the append-only, hash-chained audit record with the staff ID and role | `add-audit-record` 2.1, 2.2; `add-departure-logging` 2.3; `add-controller-operations` 2.2; `add-critical-e2e-flows` 3.4 |
| 5 | Staff API | Information disclosure | A bearer token, refresh token or authorization code ends up in logs or an error trace | Tokens, codes and PKCE verifiers are not logged; the Sentry scrubber removes headers, cookies and the IP; problem details carry no token | `add-critical-e2e-flows` 1.1 (scrubber unit test); `docs/security/data-classification.md` |
| 6 | Public API and streams | Information disclosure | Staff-only, audit or provider data appears on an anonymous endpoint | Public endpoints return the public journey view only, which carries no staff names, IDs, assignments or audit data; responses set `Cache-Control: no-store` and issue no cookies | `add-live-distribution` 1.1, 2.4; `add-catering-integration` provider-data-api spec |
| 7 | Public API and streams | Denial of service | An anonymous client floods the open endpoints | Fixed-window limit of 120 requests a minute per client IP, refused with 429 problem details and `Retry-After` (design D8) | `add-identity-access` 5.1 |
| 8 | Public API and streams | Tampering | Someone writes railway state without a credential | The fallback policy requires an authenticated staff caller, and public endpoints are reads only; a forgotten classification closes an endpoint instead of opening it | `add-identity-access` 3.1 |
| 9 | Provider API | Spoofing | A key is guessed, stolen from a log, or replayed after revocation | Format `srk_<prefix>_<secret>` with a high-entropy secret; only the prefix and a SHA-256 hash are stored; comparison is constant-time; a missing, unknown or revoked key gives the same 401 | `add-identity-access` 1.2 (plaintext key not in the database), 4.1 |
| 10 | Provider API | Denial of service | One provider degrades the API or another provider's access | Fixed-window limit of 60 requests a minute per key, refused with 429 and `Retry-After` | `add-identity-access` 5.1 |
| 11 | Provider API | Information disclosure | One provider reads or replaces another provider's subscription | The provider ID comes from the validated key, not the request body, and a subscription is scoped to that provider | `add-catering-integration` 2.1, 3.1 |
| 12 | Provider webhooks | Spoofing, tampering | Delivered events are unsigned, so the receiver cannot tell a genuine delivery from a forged one | Accepted in MVP 1: the only receiver is the simulated provider on the local Compose network and webhook signing was an owner lean cut | Gap, below |
| 13 | Database | Information disclosure | A password, API key or token is stored in the clear | SmartRail stores no staff credential at all; the provider key is stored as a prefix and a hash; there is no passenger data to store | `add-identity-access` 1.2, 4.1; `data-model.md` Identity |
| 14 | Database | Tampering | The audit record is rewritten to hide a change | `audit.entries` is append-only at the database level and hash-chained; the integrity endpoint recomputes the chain and reports the first broken position | `add-audit-record` 1.2, 3.3 |
| 15 | Database | Tampering | One module reaches into another's tables | One schema per module, no foreign key across a schema, and repository-only data access; a change that spans modules commits in one shared transaction | `add-audit-record` 1.3; architecture tests in `SmartRail.ArchitectureTests` |
| 16 | Authentik | Spoofing | A compromised identity provider can assert any staff identity | One super-user outside every application role; staff users and groups exist only in the checked-in blueprint; access tokens live 15 minutes and refresh tokens a week, and the API checks the audience before it looks at a role | `add-identity-access` 7.1 (fresh stack), `staff-authentication` spec |
| 17 | Authentik, browser sign-in | Spoofing, information disclosure | An authorization code is intercepted on its way back to the client, or delivered to a page the attacker controls | The staff client is public and has no client secret, so the code alone is not enough: every authorization request must carry a PKCE S256 challenge and the token request must present the verifier. The redirect URI must match a registered entry exactly, or the regex entry for the operations-web origin; anything else is refused before a code is issued. Codes are single-use and short-lived | `add-identity-access` 7.3 (PKCE sign-in, wrong verifier refused, unregistered redirect refused) |
| 18 | Authentik, browser sign-in | Elevation of privilege | A stolen refresh token keeps a staff session alive | Refresh tokens are issued only for `offline_access`, live a week, belong to one public client, and can be revoked in Authentik; access tokens still expire after 15 minutes and the role is re-read from `groups` on every request | `add-identity-access` 7.3 (a refresh token is issued and the refresh grant works) |

## Known gaps

- **Callback-address (server-side request) control, deferred.** Webhook registration validates
  that the URL is absolute, `http`/`https`, at most 2,048 characters and free of credentials, and
  redirects are not followed. Blocking private and loopback addresses is not built, because the
  only receiver in MVP 1 is the simulated provider on the local network, which is itself a private
  address (owner decision, 2026-10-19). A real provider must not be connected before that control
  is added; the decision and the risk are recorded in `add-catering-integration` design D4.
- **Role changes made in Authentik do not appear in SmartRail's audit record.** Who changed a
  group membership, and when, is recorded only in Authentik's own event log. SmartRail records
  what a role did with a valid token, not how the role was granted. Acceptable while staff and
  groups are blueprint seed data with no account-management screen; it must be revisited when the
  System Admin role returns and roles are edited at runtime (architecture §5).
- **Webhook payloads are not signed.** The receiver must trust the URL it registered. Signing was
  an explicit lean cut for MVP 1 (`add-catering-integration` proposal, Out of Scope).
- **The registered redirect URIs are the local stack's.** The staff client allows exactly
  `http://localhost:5080/scalar/` and any path under `http://localhost:5173`; both are development
  values. A deployment must register its own HTTPS origins and remove the localhost entries before
  staff sign in anywhere real, otherwise a local process could receive a code meant for the
  deployed client. The blueprint is the only place they are set, so this is a configuration change,
  not a code change.
- **No MFA policy, no account deactivation, no four-eyes approval.** All deferred with the System
  Admin role (mvp-1-scope.md EP-02 Out); Authentik's own policies are configuration, not SmartRail
  behaviour.
- **Authentik is a single instance.** New staff logins stop if it is down; tokens already issued
  keep working until they expire, at most 15 minutes. No high availability is claimed for MVP 1.
