# Auditor (Wasantha) — MVP 1 Journeys and Stories

**Surface:** Operations web, read-only.

## Context

SmartRail MVP 1 is a single source of truth for live train running on a small network: two lines of five stations each, running in both directions and sharing one interchange station, nine stations in total, all loaded as seed data. A change is made once and reaches the passenger app, a station platform display, the crew app and an external catering provider, each copy carrying a timestamp and a provenance label; every staff change is audited and only possible for the right role. Wasantha is the Auditor and works in the operations web with read-only access, viewing the audit record; he has no operational surface and can change nothing. The core pain addressed is that he cannot prove the record wasn't changed, and that he has no evidence roles stayed in their lanes. His assurance depends on every other role writing to the audit record — Scheduler and Network Controller changes, Guard departure logs, Driver notice acknowledgements and System Admin account actions — and on the server refusing and recording wrong-role writes.

## Key terms

- **Audit record** — an append-only, hash-chained record of every staff state change and every denied attempt, holding actor, role, before and after values and a server timestamp, and no passenger data.
- **Hash chain** — each entry is chained to the one before it, so altering a stored row breaks the chain and is detected.
- **Denied attempt** — a write the server refused with 403 because the caller's role was wrong; it is recorded like any other audit entry.
- **Correction** — a new event that references the original and leaves it in place, so both the wrong change and its correction stay in the record.
- **Separation of duties** — each staff role acts only within its own powers, and the audit record is the evidence that this held.

## End-to-end scenarios this role takes part in

**Separation of duties denials** — a Scheduler's edit to a journey inside the 24-hour window is rejected by the API and a Guard token attempting to publish a disruption gets 403; both denials appear in the audit record with the actor, role and timestamp.

**Wrong cancellation corrected** — the Network Controller cancels a journey by mistake and then corrects it; the record shows the cancellation and its correction, linked, with the original unchanged.

---

## Journeys

### J1 — Review who changed what

| # | Stage | Wasantha does | System responds | Pain addressed |
|---|---|---|---|---|
| 1 | Sign in | Logs in | Audit list, read-only | — |
| 2 | Filter | Filters by actor and time range | Matching entries | Incomplete logs |
| 3 | Inspect | Opens an entry | Actor, role, action, before/after, server timestamp | — |

### J2 — Check a separation-of-duties breach attempt (end-to-end scenario: Separation of duties denials)

1. Filters for denied attempts in a period.
2. Sees a Scheduler's in-window edit and a Guard's attempted publish, both denied with role and timestamp.

### J3 — Trace a correction (end-to-end scenario: Wrong cancellation corrected)

1. Finds a cancellation and its correction; both are present, linked, original unchanged.

### J4 — Trust the record

1. Views the chain integrity status **[A]**.
2. If any stored row has been altered, the check reports the break.

---

## User stories

### AUD-01 Read the full audit — Must
As an Auditor, I want a read-only list of every staff state change, so that I can see what happened across all roles.
- Includes Scheduler, Controller, Guard, Driver acknowledgements and System Admin actions.
**Delivers:** a read-only view of every staff change

### AUD-02 Filter — Must
As an Auditor, I want to filter by actor and time, so that I can answer specific questions quickly.
**Delivers:** filtering the record by actor and time

### AUD-03 Entry detail — Must
As an Auditor, I want each entry to show actor, role, before/after values and server timestamp, so that the change is fully explained.
- Corrections show a link to the original event.
**Delivers:** fully explained entries with a link to a corrected original

### AUD-04 Denied attempts — Must
As an Auditor, I want denied cross-role attempts recorded, so that I can show roles stayed in their lanes.
- Each 403 on a write records actor, role, attempted action and time.
**Delivers:** denied attempts recorded and visible

### AUD-05 Tamper evidence — Must
As an Auditor, I want the hash-chain check to detect altered rows, so that I can vouch for the record.
- When a stored row is modified outside the application, then the chain check fails and identifies the break point.
- Integrity status visible in my view **[A]**.
**Delivers:** tamper evidence in the audit record

### AUD-06 Read-only — Must
As an Auditor, I want no ability to change anything, so that my independence is evident.
- Every write endpoint returns 403 for my role.
**Delivers:** an Auditor role with no write powers

### AUD-07 No passenger data — Must
As an Auditor, I want the audit to contain no passenger personal data, so that the record raises no privacy conflict.
- Audit covers staff actions only; a test proves no passenger PII is written.
**Delivers:** an audit that holds no passenger data
