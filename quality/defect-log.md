# SmartRail — Defect Log

**Status:** living document, opened 2026-10-29 by QA-D1. It contains only defects that a recorded
source in this project names. Nothing here is reconstructed from memory or invented.

**Severity scale**

| Severity | Meaning |
|---|---|
| **Critical** | Passengers are misled, staff actions are lost, or the audit chain is broken |
| **High** | A flow in the five is wrong or a suite cannot pass |
| **Medium** | Wrong behaviour in a narrow case, or a test that can fail for the wrong reason |
| **Low** | Cosmetic, or a test that is slower or noisier than it should be |

**Status values:** `fixed`, `open`, `wont-fix` (with reason).

---

## 1. Confirmed defects

| ID | Found by | Date | Severity | Area | Description | Expected vs actual | Status | Fix reference |
|---|---|---|---|---|---|---|---|---|
| **DEF-001** | Backend team, LD6 work (recorded in `SmartRail-QA/COORDINATION.md` line 30) | 2026-10-29 | Critical | Live Operations / Distribution | The `JourneyStateChanged` integration event carried `StreamPosition` = 0. The stream's frame `id`, resume with `Last-Event-ID` and catch-up all depend on that number | Expected: the position PostgreSQL assigns to the appended journey event. Actual: 0 | fixed | The event now takes the appended event's position (`ApplyJourneyFactCommand.cs` line 117, `JourneyStateChanged.cs` line 14). Recorded as fixed in `COORDINATION.md` line 30 |
| **DEF-002** | Backend team, LD6 work (recorded in `COORDINATION.md` line 30) | 2026-10-29 | High | Shared Kernel event bus | The outbox transport did not cope with a delivery whose message row had been removed. The lookup threw, and the whole module's outbox cycle failed, so the messages after it were not delivered in that cycle | Expected: an orphan delivery is skipped and later messages still go out. Actual: later messages were skipped for that cycle, repeatedly | fixed | The transport now builds a dictionary of existing messages and delivers only those with a message (`PostgresEventTransport.cs` lines 82–93, including the comment that explains the skip). Recorded as fixed in `COORDINATION.md` line 30 |
| **DEF-003** | Backend team, running the full integration suite (recorded in `COORDINATION.md` lines 41–43 and AI log rows 396–397). **Introduced by the QA Flow 4 tests** | 2026-10-29 | High (suite-blocking, not a product defect) | Tests: Flow 4 cleanup / audit data | Flow 4 tests deliberately provoke denials, which write denied rows into the shared, append-only audit record. The tests did not remove those rows. Existing tests find "the newest denial for this actor and action", so they matched the leftover row and failed | Expected: the flow leaves the audit record as it found it. Actual: `317 = 309 pass + 8 fail`; the eight failures are the `Only_the_network_controller_*_and_the_denial_is_audited` theories (four files, two roles each). The project was `277/277` with the Flows namespace excluded | fixed | `FlowContext` takes an identity-outbox and audit-chain baseline at start and removes rows above them on dispose (`FlowContext.cs` lines 77, 94, 112–113; `IdentityOutbox.CleanUpAfterAsync`; `AuditDriver.ClearAfterAsync`). Verified `311/311` twice and `218/218` with Flows first (`COORDINATION.md` lines 43–44) |
| **DEF-004** | Reported by QA-A1 (recorded in `COORDINATION.md` line 41) | 2026-10-29 | Medium | Existing integration tests (`PublishDelayTests`, `PublishCancellationTests`, `PublishTerminationTests`, `PublishCorrectionTests`) | The denial theories find a denial by actor and action and take the newest row by `chain_position`, instead of finding the entry by the source event id the request produced. A stale newer row wins, so the test can assert against the wrong entry and fail for the wrong reason | Expected: the test asserts the entry caused by its own request. Actual: it asserts the newest entry for that actor and action, which may belong to another test | **open** — owner's decision | No fix yet. The pattern is visible in `PublishDelayTests.WaitForDenialAsync` (`WHERE actor_staff_id = ... AND outcome = 'denied' AND action = 'PublishDelay' ORDER BY chain_position DESC LIMIT 1`). The flow layer already does it the right way: it names the outbox announcement first (`IdentityOutbox.WaitForSingleDenialAsync`) and then waits for the entry with that source event id |
| **DEF-005** | Frontend/orchestrator browser check of OpsWeb with a real Authentik login as the Network Controller (recorded in AI log row 407, the entry that starts with First real browser verification) | 2026-10-29 | Medium | OpsWeb 403 page / audit | The 403 page shown when the controller opens /audit says the attempt was recorded in the audit log, but the audit record has no entry for that visit, because the page guard never calls the API | Expected: either the denial is audited or the page does not claim it was. Actual: the page claims an audit entry that does not exist | open (QA did not find it; added by the QA thread from the recorded row, not yet reproduced by QA) | not fixed |
| **DEF-006** | QA-A3 flow tests (recorded in SmartRail-QA/COORDINATION.md and AI log row 432) | 2026-10-29 | None (test harness artifact, not a product defect) | Our Flows SSE reader under WebApplicationFactory | Live frames appeared to reach the client only at the next heartbeat (15.2-15.5 s public, 17.6 s Guard) | Expected: about 1 s. Backend measured 1.0-1.2 s from publish to frame with the stream open before the publish and a 20 s heartbeat. Cause: under TestServer the response reaches the client only when the server writes its first body byte; a quiet stream writes nothing until an event or heartbeat, so SseStream.OpenAsync itself blocked for a heartbeat and our clock started before it | closed: artifact, explained (verdict from the Backend thread) | Flows/Support/FlowContext.cs now sets Distribution:StreamHeartbeatSeconds=1 unless a flow sets its own; a publish-to-frame latency test with a long heartbeat is still to add |

### 1.1 Why DEF-003 is in this log at all

It is a defect in our tests, not in the product. It belongs here because the assessment asks for
identified defects and honest evidence, and because it changed the strategy: every flow now takes
baselines and cleans up, and the suite was run in a fixed order to prove nothing is left behind.

**The lesson, in one line:** a filtered green run (`40/40` Flows alone) is not evidence that the
suite is green. Only the whole suite, run together, showed the pollution. The Backend team found it,
not the QA tests.

---

## 2. New defects

Append new rows here. One row per defect. Do not edit an earlier row; supersede it with a new row
if the understanding changes.

| ID | Found by | Date | Severity | Area | Description | Expected vs actual | Status | Fix reference |
|---|---|---|---|---|---|---|---|---|
| | | | | | | | | |

**Rules for adding a defect**

1. Name the test, command or activity that found it. "Found in review" is not enough unless the
   review is named.
2. State expected and actual as two different things.
3. Do not log a defect you have not reproduced, or whose source you cannot name.
4. If a test was wrong rather than the product, say so. DEF-003 is the precedent.
5. If a defect is closed by a decision rather than a fix, write `wont-fix` and the reason.

---

## 3. Defects we know about but have not logged as product defects

These are recorded elsewhere as accepted gaps, not defects, because the owner decided them. They are
listed here so nobody re-discovers them and calls them bugs.

| Item | Where it is recorded | Why it is not a defect |
|---|---|---|
| Webhook payloads are unsigned | `docs/security/threat-model.md` risk 12 and "Known gaps" | Owner lean cut; the only receiver is the local simulated provider |
| Callback URLs are not blocked from private addresses | `threat-model.md` "Known gaps" | Same reason; a real provider must not connect before this is added |
| Redirect URIs are the local stack's | `threat-model.md` "Known gaps" | Configuration, not code; a deployment must register its own |
| Role changes happen in Authentik, not in SmartRail's audit | `threat-model.md` "Known gaps"; `architecture.md` §5 | Accepted trade-off while roles are seed data |
| The stream holds response headers until the first heartbeat (first byte after about 15 s) | AI log row 403 | Reported to the Backend thread as a fix, not logged here; no user-visible defect was confirmed |
| No MFA, no account deactivation, no four-eyes approval | `threat-model.md` "Known gaps" | Deferred with the System Admin role |
