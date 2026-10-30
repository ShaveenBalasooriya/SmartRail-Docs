# How to run the SmartRail tests

## Before you start

- .NET 10 SDK installed (`dotnet --version`).
- **Docker running.** The integration and flow tests start their own temporary PostgreSQL in a
  container, so you do not need the API, Authentik or the Compose stack running.
- Open a terminal in the `SmartRail-API` folder for every `dotnet` command below.

## Where the tests are

| What | Location |
|---|---|
| Five flow tests (86) | `SmartRail-API/tests/SmartRail.Host.IntegrationTests/Flows/` |
| Existing integration tests | `SmartRail-API/tests/SmartRail.Host.IntegrationTests/` (other folders) |
| Unit and architecture tests | `SmartRail-API/tests/SmartRail.*.UnitTests/` and `SmartRail.ArchitectureTests/` |
| Browser tests (smoke test only) | `SmartRail-QA/browser/` |
| Evidence of planted failures | `SmartRail-QA/FAILURE-PROOFS.md` |

## The five flow tests

| Flow | Class | Tests |
|---|---|---|
| 1 Delay everywhere | `DelayEverywhereFlowTests` | 15 |
| 2 Offline Guard sync | `OfflineGuardSyncFlowTests` | 8 |
| 3 Platform change | `PlatformChangeFlowTests` | 10 |
| 4 Separation of duties | `SeparationOfDutiesFlowTests` | 40 |
| 5 Wrong cancellation corrected | `WrongCancellationCorrectedFlowTests` | 13 |

## Commands (run from `SmartRail-API`)

```
# Build everything with warnings as errors
dotnet build SmartRail.sln -warnaserror

# One flow (swap the class name for any row in the table above)
dotnet test tests\SmartRail.Host.IntegrationTests --filter "FullyQualifiedName~OfflineGuardSyncFlowTests"

# All five flows
dotnet test tests\SmartRail.Host.IntegrationTests --filter "FullyQualifiedName~Flows"

# One test by part of its name
dotnet test tests\SmartRail.Host.IntegrationTests --filter "FullyQualifiedName~The_guard_view_shows_the_platform_the_controller_published"

# Whole integration project (the existing tests plus the flows)
dotnet test tests\SmartRail.Host.IntegrationTests

# Unit and architecture tests only (fast, no Docker needed)
dotnet test tests\SmartRail.ArchitectureTests
dotnet test tests\SmartRail.LiveOperations.UnitTests
dotnet test tests\SmartRail.Distribution.UnitTests
dotnet test tests\SmartRail.Audit.UnitTests

# Everything in the solution
dotnet test SmartRail.sln
```

### See the progress in the console

The commands above print to the console. For more detail while the tests run, add a logger:

```
dotnet test tests\SmartRail.Host.IntegrationTests --filter "FullyQualifiedName~Flows" --logger "console;verbosity=normal"
```

Optional: also save a result file as evidence of which tests passed. It is still printed to the console:

```
dotnet test tests\SmartRail.Host.IntegrationTests --filter "FullyQualifiedName~Flows" --logger "trx;LogFileName=flows.trx" --logger "console;verbosity=normal"
```

The result file is written to `tests\SmartRail.Host.IntegrationTests\TestResults\flows.trx`.

### Approximate times on the development machine

| Run | Time |
|---|---|
| Offline flow | about 1.5 min |
| Delay flow | about 3.6 min |
| Platform + cancellation flows | about 4 min |
| Delay + platform + cancellation together | about 7 min |
| Whole integration project (375 tests) | about 18 min |

Most of the time is Docker and container start-up.

## Browser smoke test (Playwright)

From `SmartRail-QA/browser`:

```
npm install
npx playwright install chromium
npm test
```

It needs the operations website (OpsWeb) running. If it is not on `http://localhost:5173`, set the
address first. PowerShell: `$env:OPSWEB_URL = "http://localhost:5174"`.

Other scripts: `npm run test:headed` (see the browser), `npm run report` (open the last report),
`npm run typecheck`.

## Show an assertion failing

`SmartRail-QA/FAILURE-PROOFS.md` lists, for each flow, which line to change in the test and the exact
failure message to expect. To repeat one: change that expectation, run the single test with the
`--filter` command above, read the message, then change it back.

## Known caveat

Do not run `dotnet build` or `dotnet test` on the API while someone is editing the API source: the
build can collide with their edits.
