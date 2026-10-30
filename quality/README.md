# SmartRail — Quality deliverables: start here

This folder holds the QA evidence for the assessment. Four documents, one job each.

| Document | What it answers | Read it if you want to know |
|---|---|---|
| **[test-strategy.md](test-strategy.md)** | Why these tests, at these layers, with these tools | The reasoning, the risk ranking, the rejected alternatives, how to add a flow |
| **[functional-scenarios.md](functional-scenarios.md)** | What is tested, scenario by scenario | Which scenarios pass today, which are planned, which are blocked |
| **[defect-log.md](defect-log.md)** | What was found and what happened to it | The four confirmed defects, including one we caused |
| **This page** | The map and the commands | How to run it, and what is still missing |

---

## 1. How the documents answer the assessment

`final-assessment-brief.md` §18 asks for a structured approach and a justification of what was
selected. §23 lists the QA artifacts. This table maps each bullet to where it is answered.

| Assessment asks for | Where it is answered |
|---|---|
| Test strategy covering the full spectrum of quality (§23 QA) | `test-strategy.md` §3 (six layers, what each catches and does not catch) |
| Why these tests provide the most value (§18) | `test-strategy.md` §4 (five flows, risk table, blast radius) |
| Justify the selected approach and tools (§18, §24) | `test-strategy.md` §2 (decisions), §5 (tool choice and eight rejected alternatives) |
| Functional test scenarios (§23 QA) | `functional-scenarios.md` (IDs F1-*, F2-*, F3-*, F4-*, F5-*, X-*) |
| Automating the five critical end-to-end flows (§23 QA, EP-16) | `test-strategy.md` §4 and §12; `functional-scenarios.md` §1–5 |
| Maintainability and scalability of the test code | `test-strategy.md` §6 (four parts, and exactly which files a sixth flow adds) |
| Test data and independence | `test-strategy.md` §7 (own journey, unique ids, baselines, cleanup) |
| Determinism and flakiness | `test-strategy.md` §8 (frozen clock, polling with a deadline, no sleeps, no order) |
| Negative scenarios | `test-strategy.md` §9; `functional-scenarios.md` (F4-01 to F4-18 are the refusal and no-side-effect scenarios) |
| White-box testing | `test-strategy.md` §10 (transaction, outbox, replay, ordering, resume, chain, policies) |
| Security testing | `test-strategy.md` §11 (mapped to the threat model's 18 risks) |
| Identified defects (§23 QA) | `defect-log.md` (DEF-001 to DEF-004) |
| Evidence of testing (§23 QA) | `test-strategy.md` §12; `functional-scenarios.md` status column; the run records in `../../SmartRail-QA/COORDINATION.md` |
| How AI was used, and where it was wrong (§21, §23) | `test-strategy.md` §12, pointing at `../ai-usage/AI-INTERACTION-LOG.md` rows 385–406 |
| Quality assessment of the final product | **Not written yet.** It is a separate deliverable from these four |

**The assertion-failure demo.** A test suite is only evidence if a failure is readable. The recorded
demo is in `../../SmartRail-QA/browser/FEASIBILITY.md` §2: the smoke test passed, then a
deliberately wrong expected heading produced one precise `expect(locator).toBeVisible()` failure
naming the locator it waited for, then passed again after the revert. At API level the same rule is
built into `Flows/Support/Verify.cs`: every helper takes the behaviour as a sentence, so a failure
reads "A Guard's POST delay should be refused with 403, but it answered 201 Created. Body: …".
There is no recorded API-level failure demo yet; that is listed as pending below.

---

## 2. How to run

`SmartRail-QA/COORDINATION.md` owns these commands. They were recorded by the agents that ran them.
Reuse them; do not invent a second way.

| What | Command | Where | Recorded result |
|---|---|---|---|
| Build the Host test project | `dotnet build tests\SmartRail.Host.IntegrationTests\SmartRail.Host.IntegrationTests.csproj -warnaserror` | A throwaway copy of `SmartRail-API` | 0 warnings, 0 errors, about 5 s incremental (`COORDINATION.md` line 39) |
| Run the five-flow suite | `dotnet test tests\SmartRail.Host.IntegrationTests --filter "FullyQualifiedName~Flows" --logger "console;verbosity=normal"` | The same copy | 40/40 passed, about 2.04 min (`COORDINATION.md` line 40) |
| Run the whole integration project | `dotnet test tests\SmartRail.Host.IntegrationTests` | The same copy | 311 tests, about 4.06 min (`COORDINATION.md` line 40); 311/311 on two consecutive runs (`line 43`) |
| Prove nothing is left behind | `dotnet test tests\SmartRail.Host.IntegrationTests --filter "FullyQualifiedName~Flows\|FullyQualifiedName~Public\|FullyQualifiedName~LiveOperations\|FullyQualifiedName~Authorization" --logger "trx;LogFileName=order.trx"` | The same copy | 218/218, Flows first, the eight denial theories after (`COORDINATION.md` line 44) |
| Browser suite | `npx playwright test` | `SmartRail-QA/browser` | 1 smoke test passes (`FEASIBILITY.md` §2). Set `OPSWEB_URL` if OpsWeb is not on 5173 |
| Browser, signed in (constraint) | Run on `http://localhost:5173` only | `SmartRail-QA/browser` | The Authentik blueprint pins the redirect URI (`FEASIBILITY.md` §3, §6) |

**The copy-path caveat, in plain words.** The build and test commands above were run in
`C:\Users\ShaveenBalasooriya\AppData\Local\Temp\qa-a1-copy`, a throwaway copy of `SmartRail-API`,
because the real tree's `src` was being edited at the time. The copy holds an **older** `src`
snapshot. Do not sync `src` or `tests` into it wholesale; copy only your own new files in, or work in
the real tree once the Backend says `src` is quiet. Commands run in the real tree are the same minus
the copy path.

**Port note.** OpsWeb's dev server is on 5173. Another agent may hold it. Use 5174 for an unsigned
browser check, but any check that signs in through Authentik must use 5173.

**What the numbers mean.** `40/40` is the Flows namespace only. `311` is the whole Host integration
project as recorded on 2026-10-29. Neither is the whole solution: unit and architecture projects run
separately, and Layer B does not exist.

**The project has grown since.** The catering migration added four integration tests, and AI log row
406 records the whole Host integration project at **321/321** (281/281 with Flows excluded). Read
`311` as "the size when the flow work was accepted", not as today's count.

---

## 3. What is pending

Listed so nobody mistakes a plan for a result.

| Pending item | Why | What is true today |
|---|---|---|
| Layer A flows 1, 2, 3 and 5 | Being built after Flow 4, the pilot | Flow 4 is built and recorded green at 40/40 (`COORDINATION.md` line 40); the other four have integration coverage of their rules and no flow test |
| Layer B (real stack: Compose, Authentik, provider) | Not built | Design only. It is the only place deployment wiring and PKCE are proven |
| Provider receipt assertions in flows 1, 3 and 5 | Catering and the simulated provider are **not built** | Marked `pending` in `functional-scenarios.md`; never faked |
| Playwright flow checks (delay, platform change, Auditor gate) | No disposable journey for the shared dev database | Playwright is installed and smoke-tested; no flow spec written |
| API-level assertion-failure demo | Would strengthen the evidence | The helpers are built for it; no recorded run |
| Quality assessment of the final product | Separate deliverable | Not written |
| Load, performance, formal accessibility | Out of MVP 1 (`mvp-1-scope.md` EP-16 Out) | Not planned; stated as a gap |
| A fix for DEF-004 (denial lookup fragility) | The owner's decision | Open; the flow layer already does it the right way |

---

## 4. Where the code is

| What | Path |
|---|---|
| Layer A framework and Flow 4 | `SmartRail-API/tests/SmartRail.Host.IntegrationTests/Flows/` |
| Test host, database fixture, token minting | `SmartRail-API/tests/SmartRail.Host.IntegrationTests/Infrastructure/` |
| Existing integration tests | `SmartRail-API/tests/SmartRail.Host.IntegrationTests/` (`Audit`, `Authorization`, `Public`, `LiveOperations`, …) |
| Playwright project | `SmartRail-QA/browser/` |
| Shared agent notes and run records | `SmartRail-QA/COORDINATION.md` |
| AI interaction log | `docs/ai-usage/AI-INTERACTION-LOG.md` |

---

## 5. One-page summary

SmartRail's product claim is that a change is made once and reaches every channel. Five flows test
that claim from five angles. They rank by who is hurt if they break: a delay everywhere, a wrong
cancellation corrected, a role refused, an offline sync, a platform change.

They run in two layers. Layer A is fast and deterministic: real API, real PostgreSQL, fake identity,
frozen clock. Layer B is thin and real: Compose, Authentik, provider. A browser checks only the three
UI paths where the UI is the risk.

Four defects are logged. Two were real product bugs found by the Backend team. One was our tests
polluting a shared audit table, found by the Backend team running the whole suite. One is an open
fragility in older tests, waiting on the owner.

The strategy is built to be added to: a sixth flow is a new test file plus, at most, one step method
and one driver method.

---

## 6. Flows run result (2026-10-29, real tree)

The five flow classes were built and run in the real `SmartRail-API` tree (the Backend said `src` was
quiet), not in the throwaway copy. One whole-project run:

```
dotnet test SmartRail-API\tests\SmartRail.Host.IntegrationTests --logger "trx;LogFileName=whole-qa-a3.trx"
```

| Result | Value |
|---|---|
| Passed | **375** |
| Failed | **0** |
| Skipped | **0** |
| Duration | **18 m 19 s** (test run; 18 m 36 s wall including the incremental build) |

Per flow class, from the same trx:

| Flow | Class | Cases | Result |
|---|---|---|---|
| 1 — delay reaches every channel | `DelayEverywhereFlowTests` | 15 | all passed |
| 2 — offline Guard logs sync once | `OfflineGuardSyncFlowTests` | 8 | all passed |
| 3 — platform change everywhere | `PlatformChangeFlowTests` | 10 | all passed |
| 4 — separation of duties | `SeparationOfDutiesFlowTests` | 40 | all passed |
| 5 — wrong cancellation corrected | `WrongCancellationCorrectedFlowTests` | 13 | all passed |
| | Flows total | **86** | all passed |

That leaves 289 non-flow tests in the project. The "357 pass, 1 skipped" baseline quoted in the QA
brief is not this project's test count today (375 = 289 + 86), so it is treated as a solution-wide
figure; the only counts asserted here are the ones in this trx.

**Nothing left behind.** The same run contains the eight `*_and_the_denial_is_audited` theories and
every other audit test, and all of them passed. Those theories look a denial up by actor and action,
so a flow that left an older denial in the shared, append-only record would have broken them.

**Planted-failure evidence.** Five deliberately wrong expectations, one per flow class, produced five
messages that name the behaviour; the changes were reverted and the same tests passed again. The
messages and the before/after lines are in `../../SmartRail-QA/FAILURE-PROOFS.md`.
