# SmartRail — Functional Test Scenarios

**Status:** living document, first written 2026-10-29 by QA-D1. It is a register, not a claim. Rows
for flows not yet built say `planned`; rows blocked on missing software say `pending`.

**How to read the status column**

| Status | Meaning |
|---|---|
| `passing` | It exists in code and a recorded run reported it green. The source is named in the row |
| `planned` | The scenario is agreed and the test is not written yet |
| `pending` | The test cannot be written honestly until a missing piece exists (provider, Layer B) |
| `gap` | No test is planned for this at MVP 1; the reason is stated |
| `manual` | Checked by a person during the demo, not automated |

**Levels:** `Layer A` = in-process flow test; `Layer B` = real Docker Compose stack; `browser` =
Playwright against OpsWeb; `integration` = an existing endpoint-to-database test that already covers
the step (supporting coverage, not a flow test).

**Test-name shorthand.** `SeparationOfDutiesFlowTests` is
`SmartRail-API/tests/SmartRail.Host.IntegrationTests/Flows/SeparationOfDutiesFlowTests.cs`. Other
files are named in full.

---

## 1. Flow 4 — Separation of duties (built first)

Scope flow 4. Risk rank 3. It was built first because it needs no provider and no Guard stream.
The file has 11 test methods and 40 cases; `COORDINATION.md` line 40 records **40/40 passed**.

| ID | Flow | Scenario | Steps in plain words | Expected result | Level | Test name or file | Status |
|---|---|---|---|---|---|---|---|
| F4-01 | 4 | Guard cannot publish a delay | Sign in as the Guard; POST a delay on a journey that has not departed | 403 with problem details | Layer A | `A_guard_token_is_refused_on_every_controller_write` (delay case) | passing |
| F4-02 | 4 | Guard cannot cancel a journey | Sign in as the Guard; POST a cancellation on a journey with no departure | 403 with problem details | Layer A | same method (cancellation case) | passing |
| F4-03 | 4 | Guard cannot change a platform | Sign in as the Guard; POST a platform change on an undeparted stop | 403 with problem details | Layer A | same method (platform change case) | passing |
| F4-04 | 4 | Guard cannot terminate a journey | Sign in as the Guard; log a departure so the journey runs; POST a termination | 403 with problem details | Layer A | same method (termination case) | passing |
| F4-05 | 4 | Guard cannot correct a change | Sign in as the Guard; publish a delay as the controller first; POST a correction of it | 403 with problem details | Layer A | same method (correction case) | passing |
| F4-06 | 4 | Auditor cannot publish a delay | Sign in as the Auditor; POST a delay | 403 with problem details | Layer A | `An_auditor_token_is_refused_on_every_controller_write` (delay case) | passing |
| F4-07 | 4 | Auditor cannot cancel | Sign in as the Auditor; POST a cancellation | 403 with problem details | Layer A | same method (cancellation case) | passing |
| F4-08 | 4 | Auditor cannot change a platform | Sign in as the Auditor; POST a platform change | 403 with problem details | Layer A | same method (platform change case) | passing |
| F4-09 | 4 | Auditor cannot terminate | Sign in as the Auditor; POST a termination on a running journey | 403 with problem details | Layer A | same method (termination case) | passing |
| F4-10 | 4 | Auditor cannot correct | Sign in as the Auditor; POST a correction | 403 with problem details | Layer A | same method (correction case) | passing |
| F4-11 | 4 | A refused write changes no stored state | Snapshot the journey's changes, events and outbox rows; attempt a Guard write; wait for the refusal to be delivered; snapshot again | The two snapshots are identical | Layer A | `A_refused_write_leaves_the_journeys_changes_events_and_outbox_rows_untouched` (5 cases) | passing |
| F4-12 | 4 | A refused write changes nothing a passenger sees | Read the public journey detail; attempt a Guard write; read it again | Byte-for-byte identical payload | Layer A | `A_refused_write_leaves_the_public_journey_detail_unchanged` (5 cases) | passing |
| F4-13 | 4 | A denial is recorded once, naming the actor | Attempt the write as Guard or Auditor; wait for Identity's outbox announcement; read the entry it produced | One entry, outcome `denied`, the right staff id, role, action and frozen server time | Layer A | `A_denied_write_is_recorded_once_naming_the_actor_role_action_and_time` (10 cases) | passing |
| F4-14 | 4 | Reading the audit list is not a denial | Auditor reads the audit list | 200; no new audit entry | Layer A | `The_auditor_can_read_the_audit_list_and_reading_records_no_denial` | passing |
| F4-15 | 4 | No token is refused and not recorded | POST a delay with no credential | 401 with problem details; no audit entry | Layer A | `A_controller_write_with_no_token_is_refused_and_not_recorded_as_a_denial` | passing |
| F4-16 | 4 | An expired token is refused | POST a delay with a token that expired an hour ago | 401 | Layer A | `An_expired_controller_token_is_refused` | passing |
| F4-17 | 4 | A forged token is refused | POST a delay with a token signed by a key the Host does not hold | 401 | Layer A | `A_controller_token_signed_with_the_wrong_key_is_refused` | passing |
| F4-18 | 4 | The chain still verifies after every denial | Attempt all five writes as the Guard; wait for all denials; ask the Auditor for the integrity report | Status `intact` | Layer A | `The_audit_chain_verifies_after_every_denied_write` | passing |
| F4-19 | 4 | The controller can publish on the same routes | POST each of the five writes as the controller | 201 and the published change | Layer A | `The_controller_can_publish_every_controller_write` (5 cases) | passing |
| F4-20 | 4 | Deployment rejects the same roles | Run Flow 4's core cases against Compose with real Authentik tokens | Same 403 and same audit entry | Layer B | Not written | pending |
| F4-21 | 4 | The Auditor sees no publish links in OpsWeb | Sign in as Auditor; open `/audit`; try `/board` | `/audit` shows no publish links; `/board` is refused | browser | `SmartRail-QA/browser` — no spec yet | planned |

Why F4-19 matters: without it, F4-01 to F4-10 could pass because every request is refused. A
positive control on the same route is what makes the refusals meaningful.

---

## 2. Flow 1 — Delay reaches every channel

Scope flow 1. Risk rank **1**. Layer A is planned; the provider step is pending because catering
does not exist (`COORDINATION.md` line 27).

| ID | Flow | Scenario | Steps in plain words | Expected result | Level | Test name or file | Status |
|---|---|---|---|---|---|---|---|
| F1-01 | 1 | A delay moves every later expected time | Controller publishes a 10-minute delay from stop 2 on a 3-stop journey | Expected times at stops 2 and 3 move by 10 minutes; stop 1 does not change | Layer A | `DelayEverywhereFlowTests.The_public_journey_detail_moves_the_delay_stop_and_later_stops_only` (15 min from stop 3 of a 4-stop journey) | passing |
| F1-02 | 1 | The passenger view shows the delay | Read `GET /api/v1/public/journeys/{id}` after the publish | Delay status, minutes late, new expected times, freshness timestamp | Layer A | `DelayEverywhereFlowTests.The_public_journey_detail_moves_the_delay_stop_and_later_stops_only` | passing |
| F1-03 | 1 | The display shows the delay | Read the public display endpoint for the station and platform | The journey appears with its delay and expected time | Layer A | `DelayEverywhereFlowTests.The_station_display_moves_the_delay_stop_and_later_stops_only` | passing |
| F1-04 | 1 | The Guard's stream carries the delay | Subscribe to `/api/v1/guard/stream` as the assigned Guard; publish the delay | One `delay_published` frame with the new state, for that journey only | Layer A | `DelayEverywhereFlowTests.The_guard_stream_delivers_the_delay_to_the_assigned_guard_and_not_to_another` | passing |
| F1-05 | 1 | The provider receives the delay | Publish the delay; read the provider delivery rows | One delivery with a sequence number and the new state | Layer A | `DelayEverywhereFlowTests.The_provider_receives_the_delay_published_event_once_with_its_minutes` and `A_provider_that_is_down_is_retried_until_the_delay_is_applied_once` | passing |
| F1-06 | 1 | A late departure also moves later times | Guard logs a late departure at stop 1 | Downstream expected times recalculate from the actual departure | Layer A | Planned: `DelayFlowTests` | planned |
| F1-07 | 1 | The change is audited with provenance | Publish the delay; read the audit entry | Actor, role, before/after values, server time, automatic-publication label | Layer A | `DelayEverywhereFlowTests.Publishing_a_delay_appends_exactly_one_audit_entry_for_the_change` | passing |
| F1-08 | 1 | A delay on a departed stop is refused | Publish a delay from a stop the journey has left | 409; no state change | Layer A | `PublishDelayTests.A_delay_at_a_departed_stop_is_a_conflict_and_changes_nothing` | integration (exists) |
| F1-09 | 1 | A delay outside 1–720 minutes is refused | Publish 0 or 721 minutes | 400 naming the range | Layer A | `PublishDelayTests.A_delay_outside_one_to_720_minutes_is_a_bad_request` | integration (exists) |
| F1-10 | 1 | A failing audit step rolls the change back | Make the audit recorder throw; publish a delay | 500; no change, no event, no outbox row, no state change | Layer A | `PublishDelayTests.A_failing_audit_step_leaves_no_change_event_outbox_row_or_state_change` | integration (exists) |
| F1-11 | 1 | Two delays at once both succeed in order | Publish two delays concurrently | Both 201; consecutive stream positions | Layer A | `PublishDelayTests.Two_delays_published_at_once_both_succeed_with_consecutive_events` | integration (exists) |
| F1-12 | 1 | Delay end to end on the real stack | Same as F1-01 to F1-03 against Compose | Same values through a real API container and real tokens | Layer B | Not written | pending |
| F1-13 | 1 | Controller publishes a delay in OpsWeb | Sign in as controller; open a journey; fill the delay drawer; review; confirm | Review dialog shows the delay; after confirm the board shows it | browser | Not written | planned |

---

## 3. Flow 2 — Offline guard logs sync once

Scope flow 2. Risk rank **4**. There is no Guard web app, so the browser column does not apply
(`FEASIBILITY.md` §5). Layer A is planned; the sync rules already have integration coverage.

| ID | Flow | Scenario | Steps in plain words | Expected result | Level | Test name or file | Status |
|---|---|---|---|---|---|---|---|
| F2-01 | 2 | Queued logs sync once | Queue departures at stops 1 and 2 while "offline"; send them on reconnect | Two records; the passenger view and display show updated progress | Layer A | `OfflineGuardSyncFlowTests.Two_queued_logs_each_record_one_departure_at_the_device_time_they_were_tapped` and `Replaying_the_queued_logs_answers_ok_and_creates_nothing_again` | passing |
| F2-02 | 2 | Original device times survive | Log a departure with a device time earlier than the receipt time | `departedAt` is the device time; `receivedAt` is the server time | Layer A | `LogDepartureTests.Offline_logs_keep_their_device_times_and_record_the_server_receipt_time` | integration (exists) |
| F2-03 | 2 | Replaying the same log changes nothing | Send the same log id twice | One record; the replay is accepted without a duplicate | Layer A | `LogDepartureTests.A_new_log_..._a_replay_changes_nothing` | integration (exists) |
| F2-04 | 2 | A double tap with a new id is a conflict | Send two different ids for the same stop | First 201; second 409 | Layer A | `LogDepartureTests.A_double_tap_with_two_identifiers_accepts_the_first_and_conflicts_the_second` | integration (exists) |
| F2-05 | 2 | Two identical requests at once | Send the same id concurrently | One 201 and one replay, never two records | Layer A | `LogDepartureTests.Two_identical_requests_at_once_answer_one_created_and_one_replayed` | integration (exists) |
| F2-06 | 2 | Arrival at a later stop advances progress | Sync departures at stops 1 and 2 | Last station passed and next station update; ETAs follow | Layer A | `OfflineGuardSyncFlowTests.The_public_journey_detail_shows_the_last_logged_stop_as_last_passed` | passing |
| F2-07 | 2 | Out-of-order logs are refused | Log stop 3 before stop 2 | Refused as out of order; nothing stored | Layer A | `LogDepartureTests.A_departure_earlier_than_a_preceding_stop_is_out_of_order` | integration (exists) |
| F2-08 | 2 | A future time is refused | Log a departure well ahead of the server clock | 400 beyond the allowed skew | Layer A | `LogDepartureTests.A_future_timestamp_is_refused_and_exactly_two_minutes_ahead_is_accepted` | integration (exists) |
| F2-09 | 2 | Another Guard's journey is refused and audited | Log a departure on a journey assigned to a different Guard | 403; the denial is announced and audited | Layer A | `LogDepartureTests.A_log_for_another_guards_journey_is_forbidden_and_the_denial_is_announced_and_audited` | integration (exists) |
| F2-10 | 2 | The Guard's stream shows the journey change | Subscribe as the assigned Guard; publish a controller change | The frame arrives once and only for an assigned journey | Layer A | `GuardStreamTests` (5 tests, read) | integration (exists) |
| F2-11 | 2 | Sync once on the real stack | Same as F2-01 against Compose | Same result through a real API container | Layer B | Not written | pending |

---

## 4. Flow 3 — Platform change everywhere

Scope flow 3. Risk rank **5**. Shares its machinery with Flow 1; both stay (see
`test-strategy.md` §4.2). Provider step is pending.

| ID | Flow | Scenario | Steps in plain words | Expected result | Level | Test name or file | Status |
|---|---|---|---|---|---|---|---|
| F3-01 | 3 | The platform changes at one stop | Controller publishes a platform change for stop 1 | Stop 1 shows the new platform; times and other stops do not change | Layer A | `PlatformChangeFlowTests.The_guard_view_shows_the_platform_the_controller_published` and `The_public_journey_detail_shows_the_new_platform_with_its_time_and_provenance` | passing |
| F3-02 | 3 | The passenger view carries the change with its time | Read the public journey detail | New platform, the time it became known, the automatic-publication label | Layer A | `PlatformChangeFlowTests.The_public_journey_detail_shows_the_new_platform_with_its_time_and_provenance` | passing |
| F3-03 | 3 | The display makes the change prominent | Read the station/platform display view | The change is flagged, not just silently replaced | Layer A | `PlatformChangeFlowTests.The_display_feed_moves_the_journey_to_the_new_platform_and_off_the_old_one` | passing |
| F3-04 | 3 | The provider receives the change | Publish; read the provider delivery rows | One delivery with the new platform and a sequence number | Layer A | `PlatformChangeFlowTests.The_provider_receives_the_platform_change_with_the_new_platform_number` | passing |
| F3-05 | 3 | A platform at another station is refused | Publish a platform that belongs to a different station | 400 | Layer A | `PublishPlatformChangeTests.A_platform_of_another_station_is_a_bad_request` | integration (exists) |
| F3-06 | 3 | The platform already in use is refused | Publish the platform the stop already has | 409 | Layer A | `PublishPlatformChangeTests.The_platform_the_stop_already_uses_is_a_conflict` | integration (exists) |
| F3-07 | 3 | A departed stop cannot change platform | Publish a platform change for a stop the journey has left | 409 | Layer A | `PublishPlatformChangeTests.A_stop_that_has_already_departed_is_a_conflict` | integration (exists) |
| F3-08 | 3 | Only the controller may change a platform | Attempt as Guard and as Auditor | 403; denial audited | Layer A | `PublishPlatformChangeTests.Only_the_network_controller_may_change_a_platform` | integration (exists) |
| F3-09 | 3 | Platform change on the real stack | Same as F3-01 to F3-03 against Compose | Same values through the real stack | Layer B | Not written | pending |
| F3-10 | 3 | The controller changes a platform in OpsWeb | Sign in; open a journey; pick stop and new platform; review; confirm | Review shows both; after confirm the journey shows the new platform | browser | Not written | planned |

---

## 5. Flow 5 — Wrong cancellation corrected

Scope flow 5. Risk rank **2**. Correction already has strong integration coverage; the flow adds the
fan-out and the audit-keeps-the-original proof.

| ID | Flow | Scenario | Steps in plain words | Expected result | Level | Test name or file | Status |
|---|---|---|---|---|---|---|---|
| F5-01 | 5 | A wrong cancellation is withdrawn | Controller cancels a scheduled journey; then corrects that cancellation | Journey is no longer cancelled; both changes remain in history | Layer A | `PublishCorrectionTests.A_wrong_cancellation_is_withdrawn_and_both_changes_stay_in_the_history` | integration (exists) |
| F5-02 | 5 | Every channel shows the correction | After the correction, read the passenger view, display and Guard stream | Each shows the journey running again, not cancelled | Layer A | `WrongCancellationCorrectedFlowTests.A_cancelled_journey_is_shown_cancelled_and_the_correction_restores_it_with_its_time`, `The_controller_and_guard_views_agree_about_the_corrected_journey`, `The_display_feed_lists_the_journey_again_once_the_cancellation_is_corrected`, `The_public_stream_delivers_the_cancellation_then_the_correction_in_stream_order` | passing |
| F5-03 | 5 | The provider receives the correction | Read the provider delivery rows after the correction | A correction event arrives after the original, with a later sequence number | Layer A | `WrongCancellationCorrectedFlowTests.The_provider_receives_the_cancellation_then_the_correction_that_references_it` | passing |
| F5-04 | 5 | The audit keeps the original | Read the audit entries for the original and the correction | Original entry unchanged; correction links to it | Layer A | `PublishCorrectionTests.A_correction_links_its_audit_entry_to_the_original_and_leaves_it_unchanged` | integration (exists) |
| F5-05 | 5 | The correction names the original | Inspect the correction event | It references the original change and its event | Layer A | `PublishCorrectionTests.A_correction_event_names_the_original_change_and_its_event` | integration (exists) |
| F5-06 | 5 | A change already corrected cannot be corrected again | Correct the same change twice | Second attempt 409 | Layer A | `PublishCorrectionTests.A_change_that_has_already_been_corrected_is_a_conflict` | integration (exists) |
| F5-07 | 5 | A correction cannot itself be corrected | Correct the correction | 409 | Layer A | `PublishCorrectionTests.A_correction_cannot_itself_be_corrected` | integration (exists) |
| F5-08 | 5 | A later change survives an earlier correction | Publish two changes; correct the first | The second is still in force and reported as the after value | Layer A | `PublishCorrectionTests.A_later_platform_change_survives_the_correction_of_an_earlier_one` | integration (exists) |
| F5-09 | 5 | Only the controller may correct | Attempt as Guard and Auditor | 403; denial audited | Layer A | `PublishCorrectionTests.Only_the_network_controller_may_correct_and_the_denial_is_audited` | integration (exists) |
| F5-10 | 5 | A failing audit step leaves no correction | Make the recorder throw; correct a change | 500; journey unchanged | Layer A | `PublishCorrectionTests.A_failing_audit_step_leaves_no_correction_and_the_journey_unchanged` | integration (exists) |
| F5-11 | 5 | Correction on the real stack | Cancel then correct against Compose | Same result through the real stack | Layer B | Not written | pending |

---

## 6. Cross-cutting scenarios

| ID | Area | Scenario | Steps | Expected result | Level | Test name or file | Status |
|---|---|---|---|---|---|---|---|
| X-01 | Audit | Tampering is detected | Edit a stored audit value, then run the integrity check | Broken at that position | integration | `Audit/AuditIntegrityTests.cs` | integration (exists) |
| X-02 | Audit | No secrets in the record | Read every audit entry | No token, API key or address | integration | `Audit/AuditIntegrityTests.No_entry_holds_a_token_an_api_key_or_an_address` | integration (exists) |
| X-03 | Privacy | No passenger PII in logs | Run a journey and inspect captured logs | No passenger personal data | integration | `Logging/PiiLogTests.cs` | integration (exists) |
| X-04 | Auth | A token with two roles has no role | Call a staff endpoint | 403 | integration | `StaffAuthorizationTests.A_token_with_two_role_groups_has_no_role` | integration (exists) |
| X-05 | Auth | An unclassified endpoint closes | Call an endpoint with no policy, anonymously | 401 | integration | `StaffAuthorizationTests.An_endpoint_with_no_classification_refuses_an_anonymous_caller` | integration (exists) |
| X-06 | Streams | Missed events replay in order, once | Reconnect with `Last-Event-ID` after two publishes | Both arrive in position order, then live | integration | `Public/PublicStreamTests.A_reconnecting_stream_receives_the_events_it_missed_in_order_once_then_goes_live` | integration (exists) |
| X-07 | Streams | An unknown id resets first | Connect with a future or malformed id | `event: reset` with `data: {}`, then live | integration | `Public/PublicStreamTests.A_stream_whose_position_is_unknown_starts_with_reset_then_continues_live` | integration (exists) |
| X-08 | SSE wiring | The stream works through the real stack | Connect through Compose | `text/event-stream` with CORS for the OpsWeb origin | Layer B | Not written | pending |
| X-09 | Accessibility | Every MVP surface uses shared tokens | Inspect each surface | Shared tokens; status never by colour alone | manual | No automated check in MVP 1 | gap (EP-16 Out) |
| X-10 | Performance | Publish-to-channel latency under load | Load test | — | none | No test planned in MVP 1 | gap (EP-16 Out) |

---

## 7. What already exists today (read, not run)

These files were read for this register. They are the raw material the planned flow tests will
reuse, and they already prove the rules the flows depend on. They are **not** flow tests.

| File | What it covers | Flow |
|---|---|---|
| `Flows/SeparationOfDutiesFlowTests.cs` | All of Flow 4, 11 methods, 40 cases | 4 |
| `LiveOperations/PublishDelayTests.cs` | Delay validation, state, outbox, audit, concurrency, rollback | 1 |
| `LiveOperations/PublishPlatformChangeTests.cs` | Platform change rules and refusal | 3 |
| `LiveOperations/PublishCorrectionTests.cs` | Correction rules, history, audit link, concurrency | 5 |
| `LiveOperations/LogDepartureTests.cs` | Idempotent sync, device times, ordering, refusals | 2 |
| `Public/PublicStreamTests.cs` | Stream filters, replay, reset | 1, 2 |
| `Public/GuardStreamTests.cs` | Guard assignment scoping, 401/403, resume | 2 |
| `Audit/AuditIntegrityTests.cs` | Chain verification, tamper detection, no secrets | cross-cutting |
| `Authorization/StaffAuthorizationTests.cs` | Role policies, 401/403, fallback policy | 4 |

`COORDINATION.md` line 40 records the whole Host integration project at **311 tests** and the Flows
namespace at **40/40**.

**That count has moved.** The catering migration added four integration tests; AI log row 406 records
the project at **321/321** (281/281 with Flows excluded). The Flow 4 figures in §1 are unaffected:
they are 11 methods and 40 cases in the file itself.
