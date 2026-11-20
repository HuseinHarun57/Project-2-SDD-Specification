# Implementation Plan: Autonomous Robotics Management and Navigation Software Platform

Sources: `docs/spec.md`, `architecture.md`, `contracts.md`, `test_plan.md`, `tests/`, and the skeleton (read only). No implementation code was written.

Section references:
- **FR / AC / Def / NFR** refer to `docs/spec.md`. "Def" is the Definitions list at the top of Acceptance Criteria.
- **ARCH §n** refers to `architecture.md`.
- **C §n** refers to the Part 1 section numbers in `contracts.md`, and **C-REST** to its Part 2.

## Baseline (test run before any slice)

```
65 failed, 2 skipped in 0.19s
```

All 65 failures are the same error: `NotImplementedError: access_control.AccessControl.authenticate`. No failure is an assertion failure.

| Criterion | Failing tests | Count |
|---|---|---|
| AC-001 | test_TP_001_01 … test_TP_001_13 | 13 |
| AC-002 | test_TP_002_01 … test_TP_002_05 | 5 |
| AC-003 | test_TP_003_01 … test_TP_003_09 (test_TP_003_10a, test_TP_003_10b skipped) | 9 |
| AC-004 | test_TP_004_01 … test_TP_004_12 | 12 |
| AC-005 | test_TP_005_01 … test_TP_005_07 | 7 |
| AC-006 | test_TP_006_01 … test_TP_006_19, with test_TP_006_04_to_06 run three times | 19 |

## The constraint that shapes the order

Every test except `test_TP_003_08` begins with `build_f0()`. That setup makes these contract calls:

- `authenticate`
- `register_robot` ×5
- `record_telemetry` ×4
- `record_sensor_status` ×4

So no other test can pass until all three recording paths work for valid input. Slice 2 exists for that reason. It is wider than a single rule, and a defect in it will fail about 64 tests at once (see Test issues 1).

## Rule for placing tests

Each test is listed in the slice where it first **turns green**. Where a later slice owns the rule a test guards, that slice says the test must stay green.

---

## Slices

### Slice 1: Thin path, empty fleet dashboard

- **Goal:** An authenticated Robot Operator receives an empty fleet list through the full flow: route, authentication, module, store.
- **Spec sources:** FR-003; AC-003; ARCH §2 (User Interface, Access Control, Robot Status and Telemetry, Persistence); ARCH §3 (request flow pattern); C §1, §2, §5, §8; C-REST FR-003.
- **Criteria:** none completed. This is the first part of AC-003.
- **Tests that should turn green:** `test_TP_003_08`
- **Skeleton to fill in:**
  - `AccessControl.authenticate`: known credential returns the user. The rejection path is Slice 3.
  - `PersistenceStore.list_robots`
  - `RobotStatusTelemetry.get_fleet_status`: empty fleet only.
  - Route `GET /api/v1/dashboard`: success response only.
- **Depends on:** —
- **Done when:** the listed test passes and no previously passing test fails. Also check by hand that `GET /api/v1/dashboard` returns 200 `{"robots": []}`, because no test covers routes (see Design question 2).
- **Status:** done (test_TP_003_08 passes)

### Slice 2: Fixture path, valid registration and recording

- **Goal:** Valid robots can be registered, and valid telemetry and sensor status from registered robots are stored, so that `build_f0()` completes.
- **Spec sources:** FR-001, FR-004, FR-005; AC-001 scenario 1; AC-004 scenario 1; ARCH §2 (Robot Registry, Persistence); C §2, §4, §5, §6; C-REST FR-001.
- **Criteria:** none completed.
- **Tests that should turn green:** `test_TP_001_03`, `test_TP_001_06`, `test_TP_001_07`, `test_TP_004_07`, `test_TP_004_09`, `test_TP_004_11`
- **Skeleton to fill in:**
  - `AccessControl.require_role`: success path.
  - `PersistenceStore.insert_robot`, `get_robot`, `append_telemetry`, `count_telemetry`, `upsert_sensor_status`
  - `RobotRegistry.register_robot`: valid input, Engineer user.
  - `RobotRegistry.get_robot`, including `RobotNotFound`.
  - `RobotStatusTelemetry.record_telemetry`: valid message from a registered robot; `received_at = clock()`.
  - `SensorInformation.record_sensor_status`: valid message.
  - Route `POST /api/v1/robots`: success response.
- **Depends on:** Slice 1
- **Done when:** the listed tests pass, and `test_TP_003_08` still passes.
- **Status:** not started

### Slice 3: Authentication and role rejection

- **Goal:** A missing or unknown credential is rejected, and a user without the required role cannot register a robot.
- **Spec sources:** Security NFR; ARCH §4 (role-based access, authorization enforced in the application); C §1, §4.
- **Criteria:** none completed.
- **Tests that should turn green:** `test_TP_001_08`, `test_TP_001_09`, `test_TP_001_10`, `test_TP_001_11`
- **Skeleton to fill in:**
  - `AccessControl.authenticate`: `AuthenticationRequired` path.
  - `AccessControl.require_role`: `PermissionDenied` path.
  - `RobotRegistry.register_robot`: calls the Engineer role check.
  - Route mapping for `POST /api/v1/robots`: 401 and 403 (Design question 2).
- **Depends on:** Slice 2
- **Done when:** the listed tests pass and no previously passing test fails.
- **Status:** not started

### Slice 4: Robot status, reported vs never reported

- **Goal:** The dashboard lists every registered robot exactly once. A robot with telemetry shows its latest values as current; a robot with none shows as unavailable, with every status field empty.
- **Spec sources:** FR-003; AC-003 both scenarios (not-received case); AC-001 "appear in the fleet dashboard"; ARCH §4 "Distinguish unavailable status from current status"; C §5.
- **Criteria:** none completed.
- **Tests that should turn green:** `test_TP_003_01`, `test_TP_003_02`, `test_TP_003_03`, `test_TP_003_09`, `test_TP_001_01`, `test_TP_001_13`
- **Skeleton to fill in:**
  - `RobotStatusTelemetry.get_fleet_status`: non-empty fleet.
  - `RobotStatusTelemetry.get_robot_status`
  - `PersistenceStore.get_latest_telemetry`
- **Depends on:** Slice 2
- **Done when:** the listed tests pass and no previously passing test fails.
- **Note:** The age rule is deliberately left out of this slice (Slice 5), so a failure here points at selecting the latest record or building the view.
- **Status:** not started

### Slice 5: Status freshness timeout

- **Goal:** Status older than the freshness timeout is shown as unavailable. Age equal to the timeout is still current.
- **Spec sources:** Def "Status freshness timeout"; AC-003 scenario 2 ("older than the status freshness timeout"); ARCH §5.2; C §5 (the definition of current).
- **Criteria:** **AC-003**
- **Tests that should turn green:** `test_TP_003_04`, `test_TP_003_05`, `test_TP_003_06`, `test_TP_003_07`
- **Skeleton to fill in:**
  - The age comparison inside `RobotStatusTelemetry.get_fleet_status` and `get_robot_status`: `clock() - received_at <= status_freshness_timeout`.
  - The `StatusFreshnessTimeoutNotSet` path.
- **Depends on:** Slice 4
- **Done when:** the listed tests pass and no previously passing test fails.
- **Why it is its own slice:** this is the rule easiest to get subtly wrong. Using `<` instead of `<=` fails only `test_TP_003_05`. Truncating to whole seconds fails only `test_TP_003_06`.
- **Status:** not started

### Slice 6: Configuration view

- **Goal:** An Engineer can view a registered robot's configuration and software version. Other users and unknown robots are rejected.
- **Spec sources:** FR-002; AC-002; ARCH §4 (Engineer for configuration viewing); C §4; C-REST FR-002.
- **Criteria:** **AC-002**
- **Tests that should turn green:** `test_TP_002_01`, `test_TP_002_02`, `test_TP_002_03`, `test_TP_002_04`, `test_TP_002_05`
- **Skeleton to fill in:**
  - `RobotRegistry.get_robot_configuration`: Engineer check and `RobotNotFound`.
  - Route `GET /api/v1/robots/{robot_id}/configuration`: success, 401, 403, 404.
- **Depends on:** Slice 3
- **Done when:** the listed tests pass and no previously passing test fails.
- **Status:** not started

### Slice 7: Required configuration

- **Goal:** A registration with an empty robot identifier or software version is rejected, the missing fields are named, and nothing is stored.
- **Spec sources:** Def "Required configuration"; AC-001 scenario 3; C §4 (`missing_fields` order).
- **Criteria:** none completed.
- **Tests that should turn green:** `test_TP_001_04`, `test_TP_001_05`
- **Skeleton to fill in:**
  - `RobotRegistry.register_robot`: `InvalidRegistration` path.
  - Route mapping for `POST /api/v1/robots`: 400.
- **Depends on:** Slice 2
- **Done when:** the listed tests pass. `test_TP_001_06` and `test_TP_001_07` (the controls) must stay green.
- **Status:** not started

### Slice 8: Duplicate identifier

- **Goal:** Registering an identifier that already exists is rejected with an explanation, and the existing robot is not overwritten.
- **Spec sources:** AC-001 scenario 2; ARCH §4 "Reject duplicate robot identifiers"; C §2 (`insert_robot`), C §4.
- **Criteria:** none completed.
- **Tests that should turn green:** `test_TP_001_02`
- **Skeleton to fill in:**
  - `RobotRegistry.register_robot`: `DuplicateRobotIdentifier` path.
  - `PersistenceStore.insert_robot`: `UniqueIdentifierViolation` path.
  - Route mapping for `POST /api/v1/robots`: 409.
- **Depends on:** Slice 4 (dashboard listing) and Slice 6 (configuration view), both used in the assertions.
- **Done when:** the listed test passes. `test_TP_001_03` (the control) must stay green.
- **Status:** not started

### Slice 9: Duplicate identifier under concurrency

- **Goal:** Of two simultaneous registrations of one identifier, exactly one is accepted.
- **Spec sources:** AC-001 scenario 2; ARCH §4 "backed by a uniqueness constraint in Persistence".
- **Criteria:** **AC-001**
- **Tests that should turn green:** `test_TP_001_12`
- **Skeleton to fill in:**
  - `PersistenceStore.insert_robot`: atomic uniqueness.
  - `RobotRegistry.register_robot`: converts `UniqueIdentifierViolation` into `DuplicateRobotIdentifier`.
- **Depends on:** Slice 8
- **Done when:** the listed test passes repeatedly (see Test issues 7) and no previously passing test fails.
- **Why it is its own slice:** a check-then-insert implementation passes Slice 8 and can still fail here.
- **Status:** not started

### Slice 10: Latest telemetry view

- **Goal:** An Engineer can view the latest recorded telemetry for a robot.
- **Spec sources:** FR-004 "display"; AC-004 scenario 1 ("make the latest telemetry available for viewing"); C §5; C-REST FR-004.
- **Criteria:** none completed.
- **Tests that should turn green:** `test_TP_004_01`, `test_TP_004_12`, `test_TP_004_03`, `test_TP_004_04`
- **Skeleton to fill in:**
  - `RobotStatusTelemetry.get_latest_telemetry`: Engineer check and `RobotNotFound`.
  - Route `GET /api/v1/robots/{robot_id}/telemetry/latest`: success with 200 or `null`, plus 401, 403, 404.
- **Depends on:** Slice 3, Slice 4
- **Done when:** the listed tests pass and no previously passing test fails.
- **Note:** `test_TP_004_03` and `test_TP_004_04` are the *inside* cases of the battery limit. They turn green here because nothing rejects 0 or 100 yet. Slice 11 must keep them green.
- **Status:** not started

### Slice 11: Battery range

- **Goal:** Telemetry with a numeric battery level outside 0–100 is not recorded, and the previous record stays the latest.
- **Spec sources:** Def "Valid telemetry"; AC-004 scenario 2; C §5 (`record_telemetry`).
- **Criteria:** none completed.
- **Tests that should turn green:** `test_TP_004_02`, `test_TP_004_05`, `test_TP_004_06`
- **Skeleton to fill in:**
  - `RobotStatusTelemetry.record_telemetry`: `InvalidTelemetry` path, validated before anything is stored.
- **Depends on:** Slice 5 (`test_TP_004_02` checks freshness), Slice 10
- **Done when:** the listed tests pass. `test_TP_004_03`, `test_TP_004_04` and `test_TP_004_07` must stay green.
- **Why it is its own slice:** using an exclusive range fails only `test_TP_004_03` or `test_TP_004_04`.
- **Status:** not started

### Slice 12: Non-numeric battery

- **Goal:** Telemetry whose battery level is not a number is not recorded.
- **Spec sources:** Def "Valid telemetry"; AC-004 scenario 2.
- **Criteria:** none completed.
- **Tests that should turn green:** `test_TP_004_10`
- **Skeleton to fill in:**
  - `RobotStatusTelemetry.record_telemetry`: a type check before the range check.
- **Depends on:** Slice 11
- **Done when:** the listed test passes. `test_TP_004_11` (the control) must stay green.
- **Why it is its own slice:** in Python, comparing `"full"` with a number raises `TypeError`, not `InvalidTelemetry`. A plain range check therefore fails this test with the wrong error.
- **Status:** not started

### Slice 13: Telemetry from an unregistered robot

- **Goal:** Telemetry from a robot that is not registered is not recorded and creates nothing.
- **Spec sources:** Def "Valid telemetry" ("from a registered robot"); C §5.
- **Criteria:** **AC-004**
- **Tests that should turn green:** `test_TP_004_08`
- **Skeleton to fill in:**
  - `RobotStatusTelemetry.record_telemetry`: `RobotNotFound` path, checked before anything is stored.
- **Depends on:** Slice 4 (the dashboard assertion)
- **Done when:** the listed test passes. `test_TP_004_09` (the control) must stay green.
- **Status:** not started

### Slice 14: Sensor status view

- **Goal:** An Engineer sees the latest status of each registered sensor, with faulted sensors identified. Other users and unknown robots are rejected.
- **Spec sources:** FR-005 (narrowed); AC-005; ARCH §4 (Engineer for sensor information); C §6; C-REST FR-005.
- **Criteria:** **AC-005**
- **Tests that should turn green:** `test_TP_005_01`, `test_TP_005_02`, `test_TP_005_03`, `test_TP_005_04`, `test_TP_005_05`, `test_TP_005_06`, `test_TP_005_07`
- **Skeleton to fill in:**
  - `SensorInformation.get_sensor_statuses`
  - `PersistenceStore.list_sensor_statuses`
  - `PersistenceStore.upsert_sensor_status`: replacement semantics, checked by `test_TP_005_02`.
  - Route `GET /api/v1/robots/{robot_id}/sensors`: success, 401, 403, 404.
- **Depends on:** Slice 3
- **Done when:** the listed tests pass and no previously passing test fails.
- **Status:** not started

### Slice 15: Assign a mission to an available robot

- **Goal:** A Robot Operator can assign a valid mission to an available robot, and the assignment is recorded with its robot, status and time.
- **Spec sources:** FR-006; AC-006 scenario 1; ARCH §3 steps 1–8; C §2, §5, §7; C-REST FR-006.
- **Criteria:** none completed.
- **Tests that should turn green:** `test_TP_006_01`, `test_TP_006_03`, `test_TP_006_07`, `test_TP_006_08`, `test_TP_006_11`, `test_TP_006_12`, `test_TP_006_14`, `test_TP_006_17`, `test_TP_006_18`
- **Skeleton to fill in:**
  - `MissionAssignment.assign_mission`: Operator check, `RobotNotFound`, recording.
  - `RobotStatusTelemetry.get_mission_availability`: available path only.
  - `PersistenceStore.insert_mission_assignment`, `list_mission_assignments`
  - `PersistenceStore.run_in_transaction`: non-concurrent behavior.
  - Route `POST /api/v1/missions`: success, 401, 403, 404.
- **Depends on:** Slice 3, Slice 5
- **Done when:** the listed tests pass and no previously passing test fails.
- **Status:** not started

### Slice 16: Mission must have a task or destination

- **Goal:** A mission whose task and destination are both missing or empty is rejected, and nothing is recorded.
- **Spec sources:** Def "Valid task or destination information"; AC-006 scenario 3; C §7.
- **Criteria:** none completed.
- **Tests that should turn green:** `test_TP_006_09`, `test_TP_006_10`, `test_TP_006_13`
- **Skeleton to fill in:**
  - `MissionAssignment.assign_mission`: `InvalidMission` path.
  - Route mapping for `POST /api/v1/missions`: 422.
- **Depends on:** Slice 15
- **Done when:** the listed tests pass. `test_TP_006_11` and `test_TP_006_12` must stay green.
- **Why it is its own slice:** checking only for `None` passes `test_TP_006_09` and fails `test_TP_006_13` (`""`).
- **Status:** not started

### Slice 17: Unavailable because status is not current

- **Goal:** A mission cannot be assigned to a robot whose status is stale or was never reported, and the reason says so.
- **Spec sources:** Def "Available robot" (status current); AC-006 scenario 2; C §5 (`get_mission_availability` reasons).
- **Criteria:** none completed.
- **Tests that should turn green:** `test_TP_006_04_to_06[TP-006-04]`, `test_TP_006_04_to_06[TP-006-06]`, `test_TP_006_15`
- **Skeleton to fill in:**
  - `RobotStatusTelemetry.get_mission_availability`: "status information is unavailable" path. It must reuse the Slice 5 age rule.
  - `MissionAssignment.assign_mission`: raises `RobotUnavailable`.
  - Route mapping for `POST /api/v1/missions`: 409.
- **Depends on:** Slice 15 (and the Slice 5 age rule)
- **Done when:** the listed tests pass. `test_TP_006_14` (inside the limit) must stay green.
- **Status:** not started

### Slice 18: Unavailable because the robot reports an active mission

- **Goal:** A mission cannot be assigned to a robot whose latest mission status is not idle.
- **Spec sources:** Def "Available robot" (latest reported mission status is idle); C §3 (`IDLE_MISSION_STATUS`), C §5.
- **Criteria:** none completed.
- **Tests that should turn green:** `test_TP_006_04_to_06[TP-006-05]`
- **Skeleton to fill in:**
  - `RobotStatusTelemetry.get_mission_availability`: "robot reports an active mission" path.
- **Depends on:** Slice 17
- **Done when:** the listed test passes and no previously passing test fails.
- **Status:** not started

### Slice 19: Unavailable because a mission was assigned since the last report

- **Goal:** A robot that has been assigned a mission since its latest status report is unavailable. A newer idle report makes it available again.
- **Spec sources:** Def "Available robot" ("no mission has been assigned since that status was reported"); test_plan Q19; C §2 (`get_latest_mission_assignment`), C §5.
- **Criteria:** none completed.
- **Tests that should turn green:** `test_TP_006_02`, `test_TP_006_16`
- **Skeleton to fill in:**
  - `PersistenceStore.get_latest_mission_assignment`
  - `RobotStatusTelemetry.get_mission_availability`: compares the assignment's `recorded_at` with the telemetry's `received_at`.
- **Depends on:** Slice 18
- **Done when:** the listed tests pass. `test_TP_006_03` (the control) must stay green.
- **Why it is its own slice:** comparing in the wrong direction, or against the robot's reported time instead of the receipt time, fails exactly one of these two tests.
- **Status:** not started

### Slice 20: Concurrent assignment

- **Goal:** Of two simultaneous assignments to the same available robot, exactly one is recorded.
- **Spec sources:** ARCH §3 ("one coordinated operation"); Def "Available robot"; C §2 (`run_in_transaction`), C §7.
- **Criteria:** **AC-006**
- **Tests that should turn green:** `test_TP_006_19`
- **Skeleton to fill in:**
  - `PersistenceStore.run_in_transaction`: atomic check-and-record.
  - `MissionAssignment.assign_mission`: runs the availability check and the insert inside it.
- **Depends on:** Slice 19
- **Done when:** the listed test passes repeatedly (Test issues 7) and no previously passing test fails.
- **Status:** not started

---

## Coverage check

**Criteria:** each acceptance criterion is completed in exactly one slice.

| Criterion | Slice |
|---|---|
| AC-001 | 9 |
| AC-002 | 6 |
| AC-003 | 5 |
| AC-004 | 13 |
| AC-005 | 14 |
| AC-006 | 20 |

**Tests:** all 65 failing tests appear in exactly one slice:

| Criterion | Tests per slice | Total |
|---|---|---|
| AC-001 | Slice 2: 3, Slice 3: 4, Slice 4: 2, Slice 7: 2, Slice 8: 1, Slice 9: 1 | 13 |
| AC-002 | Slice 6: 5 | 5 |
| AC-003 | Slice 1: 1, Slice 4: 4, Slice 5: 4 | 9 |
| AC-004 | Slice 2: 3, Slice 10: 4, Slice 11: 3, Slice 12: 1, Slice 13: 1 | 12 |
| AC-005 | Slice 14: 7 | 7 |
| AC-006 | Slice 15: 9, Slice 16: 3, Slice 17: 3, Slice 18: 1, Slice 19: 2, Slice 20: 1 | 19 |

### Criteria or tests that fit no slice

- `test_TP_003_10a` and `test_TP_003_10b` are skipped rather than failing. They need a real-time latency harness for the 2-second Performance NFR, and no contract function provides one.
- No acceptance criterion is left out.

### Functions or routes that no slice fills

| Item | Why no slice fills it |
|---|---|
| `RobotIntegration.subscribe_telemetry` | No test. The tests call `record_telemetry` directly. |
| `RobotIntegration.subscribe_sensor_status` | No test. The tests call `record_sensor_status` directly. |
| `RobotIntegration.send_mission_assignment` | No test. When to send, and what to do on failure, are undecided (test_plan Q21). |
| `UnconfiguredIdentityRecords.find_user_by_credential` (`main.py`) | A placeholder, not a contract function. Every route returns 501 until a real identity source exists (Design question 4). |
| `SensorInformation.record_sensor_status`: `SensorNotRegistered` and `RobotNotFound` paths | No test; the probe is held back (test_plan Q15). |
| `RobotStatusTelemetry.get_robot_status` / `get_latest_telemetry`: `RobotNotFound` paths | No test calls them with an unknown robot. |
| `StatusFreshnessTimeoutNotSet` | No test. Every test sets a 10 s timeout. Slice 5 implements the path, but nothing verifies it. |
| `StorageUnavailable` | No test. |
| All six routes | No test exercises any route. Slices list the routes they fill, but "Done when" cannot be confirmed by the suite (Design question 2). |

---

## Design questions

1. **Database and test isolation (blocks Slice 1).** The contract does not name a database technology. The tests create a new `PersistenceStore` for every test with the same `TEST_DATABASE_URL`, and the contract has no reset function. The suite only works if each store instance starts empty, as an in-memory store would. A persistent shared database would make the second F0 registration fail with a duplicate. Decide the technology, and whether "each instance starts empty" becomes a contract rule or a test-only setting. **Resolved for Slice 1:** in-memory storage; each `PersistenceStore` instance starts empty and `database_url` is unused.
2. **Route error mapping and route tests.** C-REST calls its status codes "a proposal", and no test covers any route. Should the proposed codes be confirmed, and should API tests be added before routes count as done? Adding tests would be new work, so it is not in this plan.
3. **Who may view the dashboard and robot status (test_plan Q5).** The contract lists `PermissionDenied` for `get_fleet_status` and `get_robot_status` but names no role. The tests only require that a Robot Operator is allowed. Slices 1 and 4 need a decision, or the role check stays open. **Resolved for Slice 1 (dashboard only):** any authenticated user may call `get_fleet_status`. The rule for `get_robot_status` is still open.
4. **Identity source for the running server.** The tests inject a fake identity source. `main.py` wires `UnconfiguredIdentityRecords`, so the live routes cannot authenticate anyone. The identity provider and credential format are unspecified (ARCH §5.5). **Resolved for Slice 1:** keep the placeholder for now. Live routes return 501 until an identity source is specified.
5. **Order of availability reasons.** The contract lists three reasons but no order for a robot that fails more than one condition. The tests are single-fault, so Slices 17–19 can pass under any order (test_plan Q20).
6. **Transaction and concurrency semantics (Slices 9 and 20).** These depend on the database chosen in question 1. The contract says `run_in_transaction` is atomic but not how concurrent callers are serialized.

## Test issues

The tests were left unchanged.

1. **Shared setup couples everything to Slice 2.** About 64 tests depend on `build_f0()`. A defect in registration, telemetry recording or sensor recording therefore fails almost the whole suite, so early on a failure will not point at one cause.
2. **"Not logged in" tests only test `authenticate`.** `test_TP_001_08`, `test_TP_002_05`, `test_TP_005_06` and `test_TP_006_18` pass as soon as `authenticate(None)` raises. They cannot show that the module functions themselves refuse unauthenticated calls, because the contract only accepts users from `authenticate`.
3. **`test_TP_006_07` differs from test_plan.md.** The plan expects `RobotUnavailable` with the reason "the robot is not registered". The test expects `RobotNotFound`, following contracts.md. One of the two documents should be updated.
4. **`test_TP_002_01` checks sensors outside the configuration view.** It uses `RobotRegistry.get_robot`, because `RobotConfigurationView` has no sensors field. The plan says the sensors are displayed on the configuration page.
5. **`test_TP_004_10` relies on an interpretation.** The contract says `InvalidTelemetry` when the battery is "outside 0–100". The test treats a non-numeric value as outside. test_plan.md says this was derived rather than stated.
6. **Dashboard tests assume the Robot Operator role.** If Design question 3 restricts the dashboard to Engineers, `test_TP_003_01` through `test_TP_003_09` would need different users. **Resolved for Slice 1:** any authenticated user may view the dashboard, so these tests are unaffected.
7. **The concurrency tests can pass by luck.** `test_TP_001_12` and `test_TP_006_19` use two threads and a barrier. With Python's interpreter lock, a non-atomic implementation may still pass most runs. Run them repeatedly (for example 50 times) before calling Slice 9 or 20 done.

---

**Next action:** Slice 2 (fixture path). Make `test_TP_001_03` pass.
