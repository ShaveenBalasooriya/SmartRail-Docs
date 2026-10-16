# System Admin (Chathura) — MVP 1 Journeys and Stories

**Surface:** Operations web. **Owns:** staff accounts and their role, nothing operational.

## Context

SmartRail MVP 1 is a single source of truth for live train running on a small network: two lines of five stations each, running in both directions and sharing one interchange station, nine stations in total, all loaded as seed data. A change is made once and reaches the passenger app, a station platform display, the crew app and an external catering provider, each copy carrying a timestamp and a provenance label; every staff change is audited and only possible for the right role. Chathura is the System Admin and works in the operations web, where he creates and deactivates staff accounts and gives each one a role from the fixed catalogue. He owns staff accounts and their role and nothing operational: he cannot see or act on operational, schedule or passenger data. Passenger accounts are separate: passengers create their own, and the System Admin neither manages nor sees them. The core pain addressed is the pressure to over-grant access, roles that are not separated, and the inability to prove that separation held. His work depends on the role catalogue shared with the Scheduler, Network Controller, Guard, Driver and Auditor, and on the audit record that captures every account change he makes.

## Key terms

- **Fixed role catalogue** — the six MVP 1 roles, Scheduler, Network Controller, Guard, Driver, Auditor and System Admin, with one role per staff account and the catalogue enforced by the server.
- **Role-based access** — the server checks the caller's role on every request and returns 403 to the wrong role, recording the attempt.
- **Audit record** — an append-only, hash-chained record of every staff state change and every denied cross-role attempt, holding actor, role, before and after values and a server timestamp, and no passenger data.
- **Separation of duties** — staff act only inside their own role: the System Admin manages accounts, operational roles operate, and the Auditor reads; no role holds both operational and audit powers.

## End-to-end scenarios this role takes part in

**Separation of duties denials** — a Scheduler's edit to a journey inside the 24-hour window is rejected by the API and a Guard token attempting to publish a disruption gets 403; both denials are written to the audit record, which the System Admin can neither edit nor delete. No part of the scenario is driven from the System Admin's surface; a wrong-role call by the System Admin is refused and audited in the same way.

---

## Journeys

### J1 — Onboard a new staff member

| # | Stage | Chathura does | System responds | Pain addressed |
|---|---|---|---|---|
| 1 | Sign in | Logs in | Account management only | No god user |
| 2 | Create | Enters name, username, picks **one** role from fixed catalogue | Rejects a second role | Broad access under pressure |
| 3 | Hand over | Issues initial credentials **[A]** | Account active | — |
| 4 | Evidence | — | Creation audited with actor and role assigned | Cannot prove control |
| 5 | Result | New user signs in | Sees only their role's functions | — |

### J2 — Offboard a staff member

1. Finds the account, deactivates it.
2. Sign-in and existing sessions are refused **[A]**; deactivation audited.

### J3 — Asked to "just check a journey"

1. Someone asks him to look at today's running.
2. He has no operational, schedule or passenger view; a direct call returns 403 and is audited.

---

## User stories

### ADM-01 Create staff account — Must
As a System Admin, I want to create a staff account with one role, so that each person gets exactly their own powers.
- Role chosen from: Scheduler, Network Controller, Guard, Driver, Auditor, System Admin.
- The new account can sign in and use only that role.
**Delivers:** a staff account that can act in exactly one role

### ADM-02 One role per account — Must
As a System Admin, I want the system to enforce one role per account, so that no account combines powers.
- Attempting to assign a second role is rejected.
**Delivers:** one role enforced per account by the server

### ADM-03 Deactivate account — Must
As a System Admin, I want to deactivate an account, so that leavers lose access immediately.
- Deactivated accounts cannot sign in; active tokens are rejected **[A]**.
**Delivers:** immediate loss of access when an account is deactivated

### ADM-04 No operational access — Must
As a System Admin, I want to be unable to see or change operational, schedule or passenger data, so that I can't be pressured into acting as an operator.
- All such endpoints return 403 for my role and the attempt is audited.
**Delivers:** no operational access for the admin, with the attempt recorded

### ADM-05 Audited account changes — Must
As a System Admin, I want every account change audited, so that I can prove what I granted and when.
- Create, role assignment and deactivation each write actor, role, before/after and server timestamp.
**Delivers:** an audited trail of every account change

### ADM-06 Cannot touch the audit — Must
As a System Admin, I want no ability to edit or delete audit entries, including my own, so that the record is trustworthy.
- No update/delete exists for audit entries for any role.
**Delivers:** an audit record no role can alter or erase

### ADM-07 List accounts — Should **[A]**
As a System Admin, I want a list of staff accounts with role and active status, so that I can review who has access.
- Filter by role and status.
**Delivers:** a reviewable list of accounts and their roles

---

**Open point [A]:** how initial credentials are delivered, and who issues the catering provider's API key, are not stated in scope.
