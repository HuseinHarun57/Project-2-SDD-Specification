# Test Plan: Autonomous Robotics Management and Navigation Software Platform

Sources: `docs/spec.md` (the edited version, with its Definitions list) and `architecture.md`. I did not read any implementation code.

## Conventions

**Test IDs.** Each scenario carries a unique ID tag, `@TP-<AC>-<nn>`, plus one type tag:

| Tag | Meaning |
|---|---|
| `@spec` | A scenario from the acceptance criteria, with values added |
| `@inside` / `@outside` | The case just inside or just outside a limit in the requirements |
| `@control` | A valid request of the same shape as a rejected one. `controls:` names the rejection it supports |
| `@extra` | A gap-checklist case the criteria miss. `probes:` names the requirement and `gap:` names the category |

**Outcomes.** The spec defines no HTTP status codes, so outcomes are written as the spec words them: *registered*, *rejected*, *recorded*, *not recorded*, *displayed*.

**Channels.** Telemetry and sensor status arrive from the robot through Robot Integration (ARCH §2), so they are written as "robot transmits".

**Freshness timeout.** The spec leaves the value to the project team. This plan runs with **10 seconds**. If another value is chosen, shift every time marked † by the difference.

**Fixture and controls.** Every scenario starts from Background F0 unless it says otherwise. A control runs on a fresh F0 unless its steps say it follows the rejection. All times are 2026-01-01 UTC, and each robot's reported time equals the platform's receipt time (see Q7).

**Nothing created.** Every rejection asserts the counts below explicitly.

## Background F0

```gherkin
Background: F0
  Given the status freshness timeout is 10 seconds
  And these users exist:
    | user | role           | logged in |
    | eng  | Engineer       | yes       |
    | op   | Robot Operator | yes       |
    | anon | (none)         | no        |
  And these robots are registered:
    | robot | software | operating parameters | sensors                                            |
    | r-001 | 1.4.2    | max_speed_mps = 1.5  | lidar-1 LiDAR, cam-1 camera, gps-1 GPS, imu-1 IMU  |
    | r-002 | 1.4.2    | (none)               | gps-2 GPS                                          |
    | r-003 | 1.4.2    | (none)               | (none)                                             |
    | r-004 | 1.4.2    | (none)               | (none)                                             |
    | r-005 | 1.4.2    | (none)               | (none)                                             |
  And robot "r-999" is not registered
  And these telemetry records have been received:
    | robot | received at  | location                | battery | operational | mission   |
    | r-001 | 12:00:00.000 | warehouse-A (12.5, 3.0) | 76      | operational | idle      |
    | r-002 | 12:00:00.000 | warehouse-A (2.0, 8.0)  | 90      | operational | idle      |
    | r-003 | 11:59:45.000 | warehouse-A (0.0, 0.0)  | 40      | operational | idle      |
    | r-004 | 12:00:00.000 | warehouse-A (5.0, 5.0)  | 60      | operational | executing |
  And these sensor statuses have been received:
    | robot | sensor  | status | fault description |
    | r-001 | lidar-1 | ok     |                   |
    | r-001 | cam-1   | fault  | lens obstructed   |
    | r-001 | gps-1   | ok     |                   |
    | r-001 | imu-1   | ok     |                   |
  And no mission assignments exist
  And the clock reads 12:00:05.000
```

**F0 counts:** robots 5. Telemetry records: r-001 1, r-002 1, r-003 1, r-004 1, r-005 0, r-999 0. Mission assignments 0.

**Worked by hand at 12:00:05.000:**

| Robot | Status age | Status | Mission status | Available? (Def "Available robot") |
|---|---|---|---|---|
| r-001 | 5.000 s | current | idle | yes |
| r-002 | 5.000 s | current | idle | yes |
| r-003 | 20.000 s | unavailable (over 10 s) | idle | no, status not current |
| r-004 | 5.000 s | current | executing | no, not idle |
| r-005 | none | unavailable | none | no, status not current |
| r-999 | — | — | — | no, not registered |

---

## AC-001: Register and Configure Robot (FR-001)

```gherkin
Feature: Robot registration and configuration

  @TP-001-01 @spec
  Scenario: Register a robot with valid information
    Given I am logged in as "eng"
    And robot identifier "r-010" is not already registered
    When I submit the registration:
      | robot identifier | software version | operating parameters | sensors      |
      | r-010            | 2.0.0            | max_speed_mps = 1.2  | lidar-10 LiDAR |
    Then the system registers robot "r-010"
    And the registered robot count is 6
    And the fleet dashboard lists "r-010" exactly once
    And the dashboard shows "r-010" status as unavailable

  @TP-001-02 @spec
  Scenario: Reject duplicate robot registration
    Given I am logged in as "eng"
    And robot "r-001" is registered with software version "1.4.2"
    When I submit the registration:
      | robot identifier | software version |
      | r-001            | 9.9.9            |
    Then the system rejects the registration
    And the explanation states that identifier "r-001" already exists
    And the registered robot count is 5
    And robot "r-001" still has software version "1.4.2"
    And the fleet dashboard lists "r-001" exactly once

  @TP-001-03 @control  # controls: TP-001-02
  Scenario: Registration of the same shape with a new identifier is accepted
    Given I am logged in as "eng"
    When I submit the registration:
      | robot identifier | software version |
      | r-017            | 9.9.9            |
    Then the system registers robot "r-017"
    And the registered robot count is 6

  @TP-001-04 @spec
  Scenario: Reject registration missing the robot identifier
    Given I am logged in as "eng"
    When I submit the registration:
      | robot identifier | software version |
      | (absent)         | 2.0.0            |
    Then the system rejects the registration
    And the explanation identifies "robot identifier" as missing
    And the registered robot count is 5

  @TP-001-05 @spec @outside  # limit: Def Required configuration (software version present)
  Scenario: Reject registration missing the software version
    Given I am logged in as "eng"
    When I submit the registration:
      | robot identifier | software version |
      | r-011            | (absent)         |
    Then the system rejects the registration
    And the explanation identifies "software version" as missing
    And the registered robot count is 5
    And robot "r-011" is not registered

  @TP-001-06 @control @inside  # controls: TP-001-04, TP-001-05; limit: Def Required configuration
  Scenario: Registration with exactly the required configuration is accepted
    Given I am logged in as "eng"
    When I submit the registration:
      | robot identifier | software version |
      | r-018            | 2.0.0            |
    Then the system registers robot "r-018"
    And the registered robot count is 6

  @TP-001-07 @inside  # limit: Def Required configuration (optional fields may be empty)
  Scenario: Registration with an empty sensor list and no parameters is accepted
    Given I am logged in as "eng"
    When I submit the registration:
      | robot identifier | software version | operating parameters | sensors |
      | r-016            | 2.0.0            | (none)               | []      |
    Then the system registers robot "r-016"
    And the registered robot count is 6

  @TP-001-08 @extra  # probes: Security NFR, ARCH §4; gap: identity
  Scenario: Reject registration by a user who is not logged in
    Given I am not logged in ("anon")
    When I submit the registration:
      | robot identifier | software version |
      | r-012            | 2.0.0            |
    Then the system rejects the registration
    And the registered robot count is 5
    And robot "r-012" is not registered

  @TP-001-09 @control  # controls: TP-001-08; runs after TP-001-08 on the same fixture
  Scenario: The same registration by an Engineer is accepted after the rejection
    Given TP-001-08 has run
    And I am logged in as "eng"
    When I submit the registration:
      | robot identifier | software version |
      | r-012            | 2.0.0            |
    Then the system registers robot "r-012"
    And the registered robot count is 6

  @TP-001-10 @extra  # probes: Security NFR "only authorized users can configure robots", ARCH §4; gap: identity
  Scenario: Reject registration by a Robot Operator
    Given I am logged in as "op"
    When I submit the registration:
      | robot identifier | software version |
      | r-013            | 2.0.0            |
    Then the system rejects the registration
    And the registered robot count is 5
    And robot "r-013" is not registered

  @TP-001-11 @control  # controls: TP-001-10; runs after TP-001-10 on the same fixture
  Scenario: The same registration by an Engineer is accepted
    Given TP-001-10 has run
    And I am logged in as "eng"
    When I submit the registration:
      | robot identifier | software version |
      | r-013            | 2.0.0            |
    Then the system registers robot "r-013"
    And the registered robot count is 6

  @TP-001-12 @extra  # probes: AC-001 duplicate rule; gap: concurrency
  Scenario: Two simultaneous registrations of the same identifier
    Given "eng" is logged in on two sessions
    When both sessions submit {robot identifier "r-014", software version "2.0.0"} at 12:00:05.000
    Then exactly one registration is accepted
    And exactly one is rejected with the explanation that identifier "r-014" already exists
    And the registered robot count is 6
    And the fleet dashboard lists "r-014" exactly once

  @TP-001-13 @extra @inside  # probes: Scalability NFR "at least 100 robots"; gap: listing
  Scenario: The 100th robot can be registered and listed
    Given I am logged in as "eng"
    When I register robots "r-100" through "r-194" with software version "2.0.0"
    Then all 95 registrations are accepted
    And the registered robot count is 100
    And the fleet dashboard lists 100 robots, each exactly once
```

**No outside case for the 100-robot limit.** "At least 100" sets no upper limit, so a 101st robot has no expected outcome.

---

## AC-002: View Robot Configuration (FR-002)

```gherkin
Feature: Robot configuration viewing

  @TP-002-01 @spec @control  # controls: TP-002-03, TP-002-04, TP-002-05
  Scenario: View an existing robot configuration
    Given I am logged in as "eng"
    And robot "r-001" is registered
    When I open the configuration page for "r-001"
    Then the system displays:
      | robot identifier | operating parameters | sensors                                           | software version |
      | r-001            | max_speed_mps = 1.5  | lidar-1 LiDAR, cam-1 camera, gps-1 GPS, imu-1 IMU | 1.4.2            |
    And the registered robot count is 5

  @TP-002-02 @extra  # probes: AC-002 "recorded software version"; gap: what is returned
  Scenario: Configuration is displayed exactly as registered
    Given I am logged in as "eng"
    And I have registered {robot identifier "r-020", software version "3.1.0", operating parameters "max_speed_mps = 0.8"}
    When I open the configuration page for "r-020"
    Then the system displays software version "3.1.0"
    And the system displays operating parameters "max_speed_mps = 0.8"

  @TP-002-03 @extra  # probes: FR-002; gap: errors
  Scenario: Reject viewing the configuration of an unregistered robot
    Given I am logged in as "eng"
    When I open the configuration page for "r-999"
    Then the system rejects the request and displays no configuration
    And the registered robot count is 5
    And robot "r-999" is not registered

  @TP-002-04 @extra  # probes: ARCH §4 (Engineer for configuration viewing); see Q22; gap: identity
  Scenario: Reject viewing configuration as a Robot Operator
    Given I am logged in as "op"
    When I open the configuration page for "r-001"
    Then the system rejects the request and displays no configuration
    And the registered robot count is 5

  @TP-002-05 @extra  # probes: ARCH §4, Security NFR; gap: identity
  Scenario: Reject viewing configuration when not logged in
    Given I am not logged in ("anon")
    When I open the configuration page for "r-001"
    Then the system rejects the request and displays no configuration
    And the registered robot count is 5
```

**Limits:** none in AC-002.

---

## AC-003: Monitor Robot Status (FR-003)

```gherkin
Feature: Robot status monitoring

  @TP-003-01 @spec
  Scenario: View the current status of a robot
    Given I am logged in as "op"
    And the clock reads 12:00:05.000
    When I open the fleet dashboard
    Then the dashboard shows:
      | robot | status  | location                | battery | operational | mission |
      | r-001 | current | warehouse-A (12.5, 3.0) | 76      | operational | idle    |
      | r-002 | current | warehouse-A (2.0, 8.0)  | 90      | operational | idle    |

  @TP-003-02 @spec
  Scenario: The dashboard shows the latest reported values
    Given I am logged in as "op"
    And robot "r-001" transmitted at 12:00:01.000:
      | location                | battery | operational | mission |
      | warehouse-A (13.0, 3.0) | 75      | operational | idle    |
    And the clock reads 12:00:05.000
    When I open the fleet dashboard
    Then "r-001" shows location "warehouse-A (13.0, 3.0)" and battery 75
    And "r-001" does not show battery 76

  @TP-003-03 @spec
  Scenario: Display unavailable status for a robot that has never reported
    Given I am logged in as "op"
    And the clock reads 12:00:05.000
    When I view "r-005" on the dashboard
    Then "r-005" status is shown as unavailable
    And no location, battery, operational status, or mission status is presented as current for "r-005"

  @TP-003-04 @spec
  Scenario: Display unavailable status for a robot whose status is older than the timeout
    Given I am logged in as "op"
    And the clock reads 12:00:05.000  # r-003 age = 20.000 s, over 10 s
    When I view "r-003" on the dashboard
    Then "r-003" status is shown as unavailable
    And battery 40 is not presented as current for "r-003"

  @TP-003-05 @inside  # limit: Def Status freshness timeout (age equal to the timeout is not "older")
  Scenario: Status exactly at the freshness timeout is current
    Given I am logged in as "op"
    And the clock reads 12:00:10.000 †  # r-001 age = 10.000 s
    When I open the fleet dashboard
    Then "r-001" status is shown as current with battery 76

  @TP-003-06 @outside  # limit: Def Status freshness timeout
  Scenario: Status just past the freshness timeout is unavailable
    Given I am logged in as "op"
    And the clock reads 12:00:10.001 †  # r-001 age = 10.001 s
    When I open the fleet dashboard
    Then "r-001" status is shown as unavailable
    And battery 76 is not presented as current for "r-001"

  @TP-003-07 @extra  # probes: AC-001 "appear in the fleet dashboard", AC-003; gap: listing
  Scenario: The dashboard lists every registered robot exactly once
    Given I am logged in as "op"
    And the clock reads 12:00:05.000
    When I open the fleet dashboard
    Then the dashboard lists exactly "r-001", "r-002", "r-003", "r-004", "r-005", each once
    And "r-999" is not listed
    And robots shown as current are "r-001", "r-002", "r-004"
    And robots shown as unavailable are "r-003", "r-005"

  @TP-003-08 @extra  # probes: AC-003; gap: listing (empty)
  Scenario: The dashboard of an empty fleet lists no robots
    Given no robots are registered
    And I am logged in as "op"
    When I open the fleet dashboard
    Then the dashboard lists 0 robots

  @TP-003-09 @extra  # probes: AC-003; gap: state (unavailable becomes current)
  Scenario: A stale robot becomes current when it reports again
    Given I am logged in as "op"
    And robot "r-003" transmitted at 12:00:05.000:
      | location               | battery | operational | mission |
      | warehouse-A (0.0, 1.0) | 41      | operational | idle    |
    And the clock reads 12:00:07.000  # age 2.000 s
    When I open the fleet dashboard
    Then "r-003" status is shown as current with battery 41 and location "warehouse-A (0.0, 1.0)"

  @TP-003-10 @extra  # probes: Performance NFR "within 2 seconds"; gap: time; needs a real-time harness
  Scenario Outline: Dashboard update latency
    Given robot "r-001" transmits battery 75, received at 12:00:01.000
    When the dashboard first shows battery 75 for "r-001" at <shown at>
    Then the latency verdict is <verdict>

    Examples:
      | id         | type     | shown at     | measured latency | verdict |
      | TP-003-10a | @inside  | 12:00:03.000 | 2.000 s          | pass    |
      | TP-003-10b | @outside | 12:00:03.001 | 2.001 s          | fail    |
```

**Rejections:** none in AC-003, so there are no controls.

---

## AC-004: Collect Robot Telemetry (FR-004)

```gherkin
Feature: Robot telemetry collection

  @TP-004-01 @spec
  Scenario: Receive valid telemetry
    Given robot "r-001" is registered and connected
    When robot "r-001" transmits at 12:00:01.000:
      | location                | battery | operational | mission |
      | warehouse-A (13.0, 3.0) | 75      | operational | idle    |
    Then the system records the telemetry
    And robot "r-001" has 2 telemetry records
    And the latest telemetry for "r-001" has battery 75, received at 12:00:01.000

  @TP-004-02 @spec
  Scenario: Reject invalid telemetry
    Given TP-004-01 has run
    When robot "r-001" transmits at 12:00:02.000:
      | location                | battery | operational | mission |
      | warehouse-A (13.0, 3.0) | 101     | operational | idle    |
    Then the system does not record the telemetry
    And robot "r-001" has 2 telemetry records
    And the latest telemetry for "r-001" has battery 75, received at 12:00:01.000
    And the dashboard shows battery 75 for "r-001"
    And at 12:00:11.001 † "r-001" status is shown as unavailable  # age measured from 12:00:01.000

  Scenario Outline: Battery level limits (Def Valid telemetry: 0 to 100 percent)
    Given TP-004-01 has run
    When robot "r-001" transmits at 12:00:02.000 location "warehouse-A (13.0, 3.0)", battery <battery>, operational "operational", mission "idle"
    Then the telemetry is <outcome>
    And robot "r-001" has <records> telemetry records
    And the latest telemetry for "r-001" has battery <latest>

    Examples:
      | id        | type                              | battery | outcome      | records | latest |
      | TP-004-03 | @inside                           | 0       | recorded     | 3       | 0      |
      | TP-004-04 | @inside @control (TP-004-02, -05, -06) | 100     | recorded     | 3       | 100    |
      | TP-004-05 | @outside                          | -0.1    | not recorded | 2       | 75     |
      | TP-004-06 | @outside                          | 100.1   | not recorded | 2       | 75     |

  @TP-004-07 @inside  # limit: Def Valid telemetry "any reported battery level"
  Scenario: Telemetry without a battery level is valid
    Given robot "r-001" is registered and connected
    When robot "r-001" transmits at 12:00:01.000 location "warehouse-A (13.0, 3.0)", operational "operational", mission "idle", and no battery level
    Then the system records the telemetry
    And robot "r-001" has 2 telemetry records

  @TP-004-08 @spec  # Def Valid telemetry "from a registered robot"
  Scenario: Reject telemetry from an unregistered robot
    When robot "r-999" transmits at 12:00:01.000 location "warehouse-A (1.0, 1.0)", battery 50, operational "operational", mission "idle"
    Then the system does not record the telemetry
    And robot "r-999" has 0 telemetry records
    And the registered robot count is 5
    And the fleet dashboard does not list "r-999"

  @TP-004-09 @control  # controls: TP-004-08
  Scenario: The same telemetry from a registered robot is recorded
    When robot "r-001" transmits at 12:00:01.000 location "warehouse-A (1.0, 1.0)", battery 50, operational "operational", mission "idle"
    Then the system records the telemetry
    And robot "r-001" has 2 telemetry records

  @TP-004-10 @extra  # probes: Def Valid telemetry; gap: input types
  Scenario: Reject a non-numeric battery level
    When robot "r-001" transmits at 12:00:01.000 location "warehouse-A (13.0, 3.0)", battery "full", operational "operational", mission "idle"
    Then the system does not record the telemetry
    And robot "r-001" has 1 telemetry record
    And the latest telemetry for "r-001" has battery 76

  @TP-004-11 @control  # controls: TP-004-10
  Scenario: The same telemetry with a numeric battery level is recorded
    When robot "r-001" transmits at 12:00:01.000 location "warehouse-A (13.0, 3.0)", battery 50, operational "operational", mission "idle"
    Then the system records the telemetry
    And robot "r-001" has 2 telemetry records

  @TP-004-12 @extra  # probes: FR-004 "display", domain model Telemetry Record; gap: what is returned
  Scenario: The Engineer can view the latest telemetry
    Given TP-004-01 has run
    And I am logged in as "eng"
    When I view the latest telemetry for "r-001"
    Then the system displays:
      | received at  | location                | battery | operational | mission |
      | 12:00:01.000 | warehouse-A (13.0, 3.0) | 75      | operational | idle    |
```

**Battery precision.** −0.1 and 100.1 assume decimal battery levels. If the level must be an integer, the just-outside values become −1 and 101 (Q12); TP-004-02 already covers 101.

---

## AC-005: Access Sensor Information (FR-005)

```gherkin
Feature: Sensor information access

  @TP-005-01 @spec @control  # controls: TP-005-05, TP-005-06, TP-005-07
  Scenario: View sensor status
    Given I am logged in as "eng"
    And robot "r-001" has registered sensors
    When I open the sensor information for "r-001"
    Then the system displays:
      | sensor  | type   | status | fault description |
      | lidar-1 | LiDAR  | ok     |                   |
      | cam-1   | camera | fault  | lens obstructed   |
      | gps-1   | GPS    | ok     |                   |
      | imu-1   | IMU    | ok     |                   |
    And the sensors identified as reporting a fault are exactly "cam-1"

  @TP-005-02 @extra  # probes: AC-005; gap: state (fault cleared)
  Scenario: A cleared fault is no longer identified
    Given I am logged in as "eng"
    And sensor "cam-1" on "r-001" has reported status "ok"
    When I open the sensor information for "r-001"
    Then no sensor is identified as reporting a fault

  @TP-005-03 @extra  # probes: AC-005; gap: listing (several faults)
  Scenario: Several faulted sensors are all identified
    Given I am logged in as "eng"
    And sensor "lidar-1" on "r-001" has reported status "fault" with description "no returns"
    When I open the sensor information for "r-001"
    Then the sensors identified as reporting a fault are exactly "lidar-1" and "cam-1"

  @TP-005-04 @extra  # probes: AC-005; gap: listing (empty)
  Scenario: A robot with no sensors shows no sensors
    Given I am logged in as "eng"
    When I open the sensor information for "r-005"
    Then the system displays 0 sensors
    And no sensor is identified as reporting a fault

  @TP-005-05 @extra  # probes: ARCH §4 (Engineer for sensor information); see Q22; gap: identity
  Scenario: Reject sensor information for a Robot Operator
    Given I am logged in as "op"
    When I open the sensor information for "r-001"
    Then the system rejects the request and displays no sensor information
    And the registered robot count is 5

  @TP-005-06 @extra  # probes: ARCH §4, Security NFR; gap: identity
  Scenario: Reject sensor information when not logged in
    Given I am not logged in ("anon")
    When I open the sensor information for "r-001"
    Then the system rejects the request and displays no sensor information
    And the registered robot count is 5

  @TP-005-07 @extra  # probes: FR-005; gap: errors
  Scenario: Reject sensor information for an unregistered robot
    Given I am logged in as "eng"
    When I open the sensor information for "r-999"
    Then the system rejects the request and displays no sensor information
    And the registered robot count is 5
    And robot "r-999" is not registered
```

**Limits:** none in AC-005.

---

## AC-006: Assign a Mission (FR-006)

The missions used below:

- **M1:** task "pick tote 17", destination "dock-3"
- **M2:** task "pick tote 18", destination "dock-4"
- **M3:** task "pick tote 20", no destination

```gherkin
Feature: Mission assignment

  @TP-006-01 @spec @control  # controls: TP-006-17, TP-006-18
  Scenario: Assign a mission to an available robot
    Given I am logged in as "op"
    And the clock reads 12:00:05.000  # r-001 is available (see the F0 table)
    When I assign mission M1 to "r-001"
    Then the system records the mission assignment
    And the mission assignment count is 1
    And the system displays assigned robot "r-001" for M1
    And M1 is marked "assigned" or "pending execution"

  @TP-006-02 @spec @outside  # limit: Def Available robot "no mission assigned since that status was reported"
  Scenario: Prevent assignment to a robot assigned since its last status report
    Given TP-006-01 has run  # M1 recorded at 12:00:05.000; r-001 last reported at 12:00:00.000
    And the clock reads 12:00:06.000
    When I assign mission M2 to "r-001"
    Then the system rejects the assignment
    And the explanation states that a mission was assigned after the robot's last status report
    And the mission assignment count is 1
    And M1 is still assigned to "r-001"

  @TP-006-03 @control  # controls: TP-006-02
  Scenario: The same mission to an available robot is accepted
    Given TP-006-01 has run
    And the clock reads 12:00:06.000
    When I assign mission M2 to "r-002"
    Then the system records the mission assignment
    And the mission assignment count is 2

  Scenario Outline: Prevent assignment to an unavailable robot
    Given I am logged in as "op"
    And the clock reads 12:00:05.000
    When I assign mission M2 to "<robot>"
    Then the system rejects the assignment
    And the explanation states that <reason>
    And the mission assignment count is 0
    And the registered robot count is 5

    Examples:
      | id        | type  | robot | why unavailable (by hand)  | reason                                    |
      | TP-006-04 | @spec | r-003 | status age 20.000 s > 10 s | the robot's status information is unavailable |
      | TP-006-05 | @spec | r-004 | mission status "executing" | the robot reports an active mission       |
      | TP-006-06 | @spec | r-005 | no status ever received    | the robot's status information is unavailable |
      | TP-006-07 | @spec | r-999 | not registered             | the robot is not registered               |

  @TP-006-08 @control  # controls: TP-006-04 to TP-006-07
  Scenario: The same mission to an available robot is accepted
    Given I am logged in as "op"
    And the clock reads 12:00:05.000
    When I assign mission M2 to "r-002"
    Then the system records the mission assignment
    And the mission assignment count is 1

  @TP-006-09 @spec
  Scenario: Reject a mission without task or destination information
    Given I am logged in as "op"
    And robot "r-002" is available
    When I assign a mission with task (absent) and destination (absent) to "r-002"
    Then the system rejects the assignment
    And the explanation states that a task or destination is required
    And the mission assignment count is 0

  @TP-006-10 @control  # controls: TP-006-09, TP-006-13; also shows r-002 was still available after TP-006-09
  Scenario: The same assignment with a task is accepted
    Given TP-006-09 has run
    When I assign a mission with task "pick tote 19" and destination (absent) to "r-002"
    Then the system records the mission assignment
    And the mission assignment count is 1

  Scenario Outline: Task or destination limits (Def "at least one ... is not empty")
    Given I am logged in as "op"
    And the clock reads 12:00:05.000
    When I assign a mission with task <task> and destination <destination> to "r-002"
    Then the assignment is <outcome>
    And the mission assignment count is <count>

    Examples:
      | id        | type     | task   | destination | outcome  | count |
      | TP-006-11 | @inside  | "x"    | (absent)    | recorded | 1     |
      | TP-006-12 | @inside  | (absent) | "x"       | recorded | 1     |
      | TP-006-13 | @outside | ""     | ""          | rejected | 0     |

  Scenario Outline: Freshness limit for availability (Def Status freshness timeout)
    Given I am logged in as "op"
    And the clock reads <now>  # r-002 last reported at 12:00:00.000
    When I assign mission M3 to "r-002"
    Then the assignment is <outcome>
    And the mission assignment count is <count>

    Examples:
      | id        | type                        | now            | r-002 age | outcome                                                  | count |
      | TP-006-14 | @inside @control (TP-006-15) | 12:00:10.000 † | 10.000 s  | recorded                                                 | 1     |
      | TP-006-15 | @outside                    | 12:00:10.001 † | 10.001 s  | rejected; explanation: status information is unavailable | 0     |

  @TP-006-16 @inside  # limit: Def Available robot "since that status was reported"; see Q19
  Scenario: A robot that reports idle after an assignment is available again
    Given TP-006-01 has run  # M1 recorded at 12:00:05.000
    And robot "r-001" transmitted mission status "idle", received at 12:00:06.000
    And the clock reads 12:00:07.000
    When I assign mission M2 to "r-001"
    Then the system records the mission assignment
    And the mission assignment count is 2

  @TP-006-17 @extra  # probes: ARCH §3 step 3, Security NFR "only authorized users can ... assign missions"; gap: identity
  Scenario: Reject mission assignment by an Engineer
    Given I am logged in as "eng"
    And the clock reads 12:00:05.000
    When I assign mission M1 to "r-001"
    Then the system rejects the assignment
    And the mission assignment count is 0

  @TP-006-18 @extra  # probes: Security NFR; gap: identity
  Scenario: Reject mission assignment when not logged in
    Given I am not logged in ("anon")
    And the clock reads 12:00:05.000
    When I assign mission M1 to "r-001"
    Then the system rejects the assignment
    And the mission assignment count is 0

  @TP-006-19 @extra  # probes: ARCH §3 "one coordinated operation", Def Available robot; gap: concurrency
  Scenario: Two simultaneous assignments to the same available robot
    Given "op" is logged in on two sessions
    And the clock reads 12:00:05.000
    When one session assigns M1 and the other assigns M3 to "r-001" at the same instant
    Then exactly one assignment is recorded
    And the other is rejected with the explanation that a mission was assigned after the robot's last status report
    And the mission assignment count is 1
```

---

## Gap checklist coverage

| Category | Scenarios |
|---|---|
| Boundaries | TP-001-05/06/07/13, TP-003-05/06/10, TP-004-03 to 07, TP-006-02, TP-006-11 to 16 |
| Time | TP-003-04/05/06/09/10, TP-004-02 (age measured from the last *recorded* report), TP-006-14/15/16 |
| Identity | TP-001-02/08/10, TP-002-04/05, TP-004-08, TP-005-05/06, TP-006-07/17/18 |
| State | TP-001-02 (no overwrite), TP-003-09, TP-005-02, TP-006-02/16 |
| Concurrency | TP-001-12, TP-006-19 |
| Input types | TP-004-07, TP-004-10 |
| Errors and precedence | Errors: TP-002-03, TP-005-07, TP-006-07. **Precedence is not tested.** It needs a request with two faults, which the one-fault rule forbids, and the spec gives no order (Q20). |
| Listing | TP-001-13, TP-003-07/08, TP-005-03/04 |
| What is returned | TP-001-01, TP-002-01/02, TP-004-12, TP-006-01 |

### Extra probes held back

These checklist cases need an expected value the spec does not give, so they are not written as scenarios yet.

| Probe | Category | Requirement | Blocked by |
|---|---|---|---|
| Register with identifier `""` or software version `""` | Input types | Def Required configuration | Q1 |
| Register `R-001` or `" r-001 "` when `r-001` exists | Identity | FR-001 | Q2 |
| Contents of a successful registration or assignment response, including any mission identifier | What is returned | AC-001, AC-006 | Q3 |
| `anon` or `eng` opens the dashboard | Identity | FR-003, Security NFR | Q5 |
| A record reported at 12:00:01.000 arrives after one reported at 12:00:02.000; or the reported time is in the future | Time | AC-003, AC-004 | Q7, Q10 |
| `op` views the latest telemetry | Identity | FR-004 | Q9 |
| Two r-001 records arrive at the same instant | Concurrency | AC-004 "latest" | Q10 |
| A fresh record with no battery: is battery 76 still shown? | State | AC-003 | Q11 |
| Battery `"75"` sent as a string; a mission task sent as the number `123` | Input types | Def Valid telemetry, Def Valid task or destination | Q13 |
| r-002's gps-2 has never reported: is it listed? | Listing | AC-005 | Q14 |
| A status arrives for `sonar-9`, which is not registered on r-001 | Identity | AC-005 | Q15 |
| cam-1's fault was reported more than 10 s ago | Time | AC-005 | Q16 |
| Task `" "` (whitespace only) | Boundaries | Def Valid task or destination | Q17 |
| An assignment recorded at the same instant as a status report | Time | Def Available robot | Q18 |

---

## Spec gaps

### Requirements with no acceptance criterion

1. **Availability NFR (99.9% during scheduled hours).** No acceptance criterion.
2. **Performance NFR (2 s dashboard update).** No acceptance criterion; probed only by extra TP-003-10.
3. **Emergency stop and pause NFR (1 s, acknowledgment shown).** No acceptance criterion, and no functional requirement defines a stop or pause operation.
4. **Security NFR (role-based access, authenticated communication).** Roles appear only as "Given I am logged in" steps. No criterion rejects an unauthenticated or wrong-role user (probed by TP-001-08/10, TP-002-04/05, TP-005-05/06, TP-006-17/18).
5. **Scalability NFR (at least 100 robots).** No acceptance criterion; probed by TP-001-13.
6. **Parts of functional requirements with no criterion:**
   - **FR-001** "configure its basic operating parameters": no criterion covers changing the configuration of a robot that is already registered.
   - **FR-003** says "current location"; AC-003 says "latest reported location".
   - **FR-004** "identify abnormal operating conditions": AC-004 covers only recording and viewing.
   - **FR-006** "so that the robot can autonomously perform its designated task": no criterion covers delivering the mission to the robot (Q21).
7. **Duties in the goals and actor descriptions with no functional requirement:** alerts, escalation (problem reports), pause/stop, software updates, error logs, maintenance records, mission history, and routes and waypoints.

### Acceptance criteria with no requirement

None. AC-001 through AC-006 each name a related requirement. Note one point about the Definitions: "Status freshness timeout" and the idle and no-assignment-since conditions of "Available robot" add rules that no requirement's text states.

### Points the spec does not decide (referenced as Q#)

| Q | Question | Where |
|---|---|---|
| Q1 | Does an empty string count as a missing identifier or software version? | Held-back probes |
| Q2 | Are robot identifiers case-sensitive? Is surrounding whitespace significant? | Held-back probes |
| Q3 | What does a successful registration or assignment return (for example, a mission identifier)? | Held-back probes |
| Q4 | How is a request for an unregistered robot reported on reads? The scenarios assert only "rejected, nothing displayed". | TP-002-03, TP-005-07 |
| Q5 | Who may view the dashboard? AC-003 has no login step; FR-003 names the Robot Operator; the NFR requires authenticated communication. | Held-back probes |
| Q6 | What is the clock resolution for "just outside" a time limit? This plan assumes 1 ms. | TP-003-06, TP-006-15 |
| Q7 | Is age measured from the robot's reported time or the platform's receipt time? How is a future reported time handled? | F0 sets them equal |
| Q8 | What are "normal operating conditions" for the 2 s target? | TP-003-10 |
| Q9 | Who besides Engineers may view the latest telemetry? | Held-back probes |
| Q10 | Is "latest" telemetry decided by receipt order or reported time, including for simultaneous arrivals? | Held-back probes |
| Q11 | If a fresh record omits the battery, may the previous battery value be shown? | Held-back probes |
| Q12 | Is the battery level an integer or a decimal? This sets the "just outside" values. | TP-004-05/06 |
| Q13 | Are numeric strings or numbers accepted where text or numbers are expected? | Held-back probes |
| Q14 | Is a registered sensor that has never reported listed in "available sensor statuses"? | Held-back probes |
| Q15 | What happens to a status reported for an unregistered sensor? | Held-back probes |
| Q16 | Does the freshness timeout apply to sensor status as well as robot status? | Held-back probes |
| Q17 | Is a whitespace-only task or destination "empty"? | Held-back probes |
| Q18 | If an assignment and a status report have the same instant, is the robot available? | Held-back probes |
| Q19 | Under the Def, a robot that reports idle after an assignment but before starting it becomes available again (TP-006-16). Is that intended? | TP-006-16 |
| Q20 | Which explanation wins when a request has two faults? | Precedence (not tested) |
| Q21 | Is an accepted assignment sent to the robot, and what happens if delivery fails? | FR-006 |
| Q22 | ARCH §4 restricts configuration viewing and sensor information to Engineers, but the Security NFR lists only configuring, assigning, stopping, and updating. This plan follows ARCH §4. | TP-002-04, TP-005-05 |
