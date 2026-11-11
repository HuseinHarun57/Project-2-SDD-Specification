# Contracts: Autonomous Robotics Management and Navigation Software Platform

Skeleton only. Every function below raises `NotImplementedError`. The "Success" and "Fails with" columns describe the behaviour the implementation must provide.

- "Spec rule" cites `docs/spec.md` (FR / AC / NFR / **Def** = the Definitions list at the top of Acceptance Criteria) or `architecture.md` (`ARCH §n`).
- "Injected" lists the constructor dependencies the function relies on. Every class receives its dependencies through `__init__`, so a test can replace any of them.
- `user: AuthenticatedUser` is always the result of `AccessControl.authenticate`.
- **Not specified** means the spec or architecture still leaves the decision open.

## Shared errors (`app/modules/persistence.py`)

| Error | Raised when | Raised by |
|---|---|---|
| `RobotNotFound(robot_id)` | No robot is registered with that identifier | Robot Registry, Robot Status and Telemetry, Sensor Information, Mission Assignment |
| `SensorNotRegistered(robot_id, sensor_id)` | A sensor status names a sensor not registered on the robot | Sensor Information |
| `UniqueIdentifierViolation` | A unique key already exists | Persistence |
| `StorageUnavailable` | The database cannot be reached | Persistence (any method) |

These live in Persistence because every module that raises them already depends on it, so no new dependency is added.

## Data types

Fields as defined in the skeleton. Import paths are under `app.modules`.

| Type | Module | Fields |
|---|---|---|
| `Role` (enum) | `access_control` | `ENGINEER`, `ROBOT_OPERATOR` |
| `AuthenticatedUser` | `access_control` | `user_id: str`, `roles: frozenset[Role]` |
| `UserIdentityRecords` (protocol) | `access_control` | `find_user_by_credential(credential: str) -> AuthenticatedUser \| None` |
| `SensorType` (enum) | `persistence` | `LIDAR`, `CAMERA`, `GPS`, `IMU`, `OTHER` |
| `SensorHealth` (enum) | `persistence` | `OK`, `FAULT` |
| `AssignmentStatus` (enum) | `persistence` | `ASSIGNED`, `PENDING_EXECUTION` |
| `SensorDescriptor` | `persistence` | `sensor_id`, `sensor_type: SensorType` |
| `RobotConfiguration` | `persistence` | `operating_parameters: Mapping[str, Any]`, `sensor_settings: Mapping[SensorId, Mapping[str, Any]]`, `software_version: str` |
| `RobotRecord` | `persistence` | `robot_id`, `configuration: RobotConfiguration`, `sensors: tuple[SensorDescriptor, ...]` |
| `Location` | `persistence` | `reference_frame: str`, `coordinates: Mapping[str, float]` |
| `TelemetryRecord` | `persistence` | `robot_id`, `received_at`, `reported_at`, `location`, `battery_level`, `speed`, `temperature`, `communication_status`, `operational_status`, `mission_status` |
| `SensorStatusRecord` | `persistence` | `robot_id`, `sensor_id`, `health: SensorHealth`, `fault_description`, `reported_at` |
| `MissionSpec` | `persistence` | `task: str \| None`, `destination: str \| None` |
| `MissionAssignmentRecord` | `persistence` | `mission_id`, `robot_id`, `mission: MissionSpec`, `status: AssignmentStatus`, `recorded_at: datetime` |
| `InboundTelemetry` | `robot_integration` | `robot_id`, `reported_at`, `location`, `battery_level`, `speed`, `temperature`, `communication_status`, `operational_status`, `mission_status` |
| `InboundSensorStatus` | `robot_integration` | `robot_id`, `sensor_id`, `health: SensorHealth`, `fault_description`, `reported_at` |
| `RobotRegistration` | `robot_registry` | `robot_id`, `configuration: RobotConfiguration`, `sensors: tuple[SensorDescriptor, ...]` |
| `RobotConfigurationView` | `robot_registry` | `robot_id`, `configuration: RobotConfiguration`, `software_version: str` |
| `StatusAvailability` (enum) | `robot_status_telemetry` | `CURRENT`, `UNAVAILABLE` |
| `RobotStatusView` | `robot_status_telemetry` | `robot_id`, `availability`, `location`, `battery_level`, `operational_status`, `mission_status`, `last_received_at` |
| `MissionAvailability` | `robot_status_telemetry` | `robot_id`, `available: bool`, `reason: str \| None` |
| `SensorStatusView` | `sensor_information` | `sensor_id`, `sensor_type`, `health: SensorHealth \| None`, `is_faulted: bool`, `fault_description`, `reported_at` |
| `SensorReport` | `sensor_information` | `robot_id`, `sensors: list[SensorStatusView]` |
| `MissionRequest` | `mission_assignment` | `robot_id`, `mission: MissionSpec` |

Error attributes: `DuplicateRobotIdentifier.robot_id`; `InvalidRegistration.missing_fields`; `RobotNotFound.robot_id`; `SensorNotRegistered.robot_id`, `.sensor_id`; `RobotUnavailable.robot_id`, `.reason`.

---

## Part 1: Module functions

### 1. Access Control (`app/modules/access_control.py`)

Constructor: `AccessControl(identity_records: UserIdentityRecords)`

| Function | Success | Fails with | Spec rule | Injected |
|---|---|---|---|---|
| `authenticate(credential: str \| None) -> AuthenticatedUser` | Returns the user and their `roles` | `AuthenticationRequired` when the credential is missing or unknown | Security NFR; AC-001/002/005/006 "Given I am logged in" | `identity_records` |
| `require_role(user: AuthenticatedUser, role: Role, operation: str) -> None` | Returns `None` when `role in user.roles` | `PermissionDenied` naming `operation` | Security NFR; ARCH §4 "enforce authorization in the application" | none |

`Role` = `ENGINEER`, `ROBOT_OPERATOR`. Not specified: the identity provider, credential format, and session policy (ARCH §5.5).

### 2. Persistence (`app/modules/persistence.py`)

Constructor: `PersistenceStore(database_url: str)`. Every method can also fail with `StorageUnavailable`.

| Function | Success | Fails with | Spec rule | Injected |
|---|---|---|---|---|
| `insert_robot(record: RobotRecord) -> None` | Robot is stored | `UniqueIdentifierViolation` | AC-001 duplicate rejection; ARCH §4 uniqueness constraint | database |
| `get_robot(robot_id) -> RobotRecord \| None` | The record, or `None` | — | AC-002, AC-005 | database |
| `list_robots() -> list[RobotRecord]` | All registered robots | — | AC-001 "appear in the fleet dashboard"; AC-003 | database |
| `append_telemetry(record: TelemetryRecord) -> None` | Telemetry is stored | — | AC-004 "record the received telemetry" | database |
| `get_latest_telemetry(robot_id) -> TelemetryRecord \| None` | Newest record by `received_at`, or `None` | — | AC-004 "latest telemetry"; AC-003 | database |
| `count_telemetry(robot_id) -> int` | Number of stored telemetry records for `robot_id`; 0 if there are none or the robot is unknown | — | AC-004 "shall not record the telemetry" (lets a test check that nothing was created) | database |
| `upsert_sensor_status(record: SensorStatusRecord) -> None` | Latest status per `(robot_id, sensor_id)` is stored | — | AC-005 | database |
| `list_sensor_statuses(robot_id) -> list[SensorStatusRecord]` | Latest status for each reporting sensor | — | AC-005 | database |
| `insert_mission_assignment(record: MissionAssignmentRecord) -> None` | Assignment is stored | `UniqueIdentifierViolation` on duplicate `mission_id` | AC-006 "record the mission assignment" | database |
| `list_mission_assignments() -> list[MissionAssignmentRecord]` | Every stored assignment, oldest `recorded_at` first | — | AC-006 "record the mission assignment" (lets a test check that nothing was created) | database |
| `get_latest_mission_assignment(robot_id) -> MissionAssignmentRecord \| None` | Newest assignment by `recorded_at`, or `None` | — | **Def** Available robot ("no mission has been assigned since that status was reported") | database |
| `run_in_transaction(operation: Callable[[], T]) -> T` | Runs `operation` atomically and returns its result | Re-raises whatever `operation` raised, with nothing committed | ARCH §3 coordinated operation | database |

Not specified: database technology, retention, and the ordering of telemetry that arrives out of order (ARCH §4, §5.2, §5.7).

### 3. Robot Integration (`app/modules/robot_integration.py`)

Constructor: `RobotIntegration(robot_endpoints: dict[RobotId, str])`. Constant: `IDLE_MISSION_STATUS = "idle"`.

| Function | Success | Fails with | Spec rule | Injected |
|---|---|---|---|---|
| `subscribe_telemetry(handler: Callable[[InboundTelemetry], object]) -> None` | `handler` is called for each telemetry message. Robot-specific mission-status values are normalized so that "no active mission" is `IDLE_MISSION_STATUS`. The handler's return value is ignored | `RobotCommunicationError` if the transport cannot be opened | FR-004; **Def** Available robot ("mission status is idle") | `robot_endpoints` |
| `subscribe_sensor_status(handler: Callable[[InboundSensorStatus], object]) -> None` | `handler` is called for each sensor-status message | `RobotCommunicationError` | FR-005 | `robot_endpoints` |
| `send_mission_assignment(assignment: MissionAssignmentRecord) -> None` | The assignment is delivered to the robot-side system | `RobotCommunicationError` | FR-006; ARCH §2 | `robot_endpoints` |

Not specified: protocols, message formats, supported robot models, and each family's own "idle" value (ARCH §5.1).

### 4. Robot Registry (`app/modules/robot_registry.py`)

Constructor: `RobotRegistry(store: PersistenceStore, access: AccessControl)`

| Function | Success | Fails with | Spec rule | Injected |
|---|---|---|---|---|
| `register_robot(user, registration: RobotRegistration) -> RobotRecord` | Robot is stored and returned; afterwards it appears in `get_fleet_status` | `PermissionDenied` (not Engineer); `InvalidRegistration(missing_fields)` when `robot_id` or `software_version` is empty; `missing_fields` lists `"robot_id"` and/or `"software_version"`, in that order; `DuplicateRobotIdentifier` ("identifier already exists") | FR-001; AC-001 all three scenarios; **Def** Required configuration | `store`, `access` |
| `get_robot_configuration(user, robot_id) -> RobotConfigurationView` | The configuration and recorded `software_version` | `PermissionDenied` (not Engineer); `RobotNotFound` | FR-002; AC-002 | `store`, `access` |
| `get_robot(robot_id) -> RobotRecord` | The registered robot | `RobotNotFound` | ARCH §2 (used by Mission Assignment) | `store` |

### 5. Robot Status and Telemetry (`app/modules/robot_status_telemetry.py`)

Constructor: `RobotStatusTelemetry(store, integration, access, clock: Callable[[], datetime], status_freshness_timeout: timedelta | None)`

Status is **current** when the latest telemetry exists and `clock() - received_at <= status_freshness_timeout`. Otherwise it is **unavailable**.

| Function | Success | Fails with | Spec rule | Injected |
|---|---|---|---|---|
| `record_telemetry(message: InboundTelemetry) -> TelemetryRecord` | Stores the message with `received_at = clock()` and returns it | `RobotNotFound`; `InvalidTelemetry` when `battery_level` is outside 0–100. Nothing is stored on failure, so the previous record stays the latest | FR-004; AC-004 both scenarios; **Def** Valid telemetry; Performance NFR (2 s) | `store`, `clock` |
| `get_fleet_status(user) -> list[RobotStatusView]` | One view per registered robot | `PermissionDenied`; `StatusFreshnessTimeoutNotSet` | FR-003; AC-003; AC-001 "appear in the fleet dashboard" | `store`, `access`, `clock`, `status_freshness_timeout` |
| `get_robot_status(user, robot_id) -> RobotStatusView` | `CURRENT` with location, battery, operational status, and mission status; or `UNAVAILABLE` with those fields set to `None` | `PermissionDenied`; `RobotNotFound`; `StatusFreshnessTimeoutNotSet` | AC-003 both scenarios; **Def** Status freshness timeout | `store`, `access`, `clock`, `status_freshness_timeout` |
| `get_latest_telemetry(user, robot_id) -> TelemetryRecord \| None` | The latest record, or `None` if none has been received | `PermissionDenied` (not Engineer); `RobotNotFound` | FR-004; AC-004 | `store`, `access` |
| `get_mission_availability(robot_id) -> MissionAvailability` | `available=True` only if status is current, `mission_status == IDLE_MISSION_STATUS`, and the latest assignment (if any) has `recorded_at` earlier than the telemetry's `received_at`. Otherwise `available=False` with `reason` set to one of: "status information is unavailable", "robot reports an active mission", "a mission was assigned after the robot's last status report" | `RobotNotFound`; `StatusFreshnessTimeoutNotSet` | AC-006; **Def** Available robot | `store`, `clock`, `status_freshness_timeout` |

Not specified: which role may view the dashboard. FR-003 names the Robot Operator, AC-003 has no "logged in" step, and Engineer access is unstated.

### 6. Sensor Information (`app/modules/sensor_information.py`)

Constructor: `SensorInformation(store, integration, access)`

| Function | Success | Fails with | Spec rule | Injected |
|---|---|---|---|---|
| `record_sensor_status(message: InboundSensorStatus) -> SensorStatusRecord` | Stores and returns the latest status for that sensor | `RobotNotFound`; `SensorNotRegistered` | FR-005; ARCH §2 | `store` |
| `get_sensor_statuses(user, robot_id) -> SensorReport` | One `SensorStatusView` per registered sensor. `is_faulted=True` when `health == FAULT`. `health=None` when the sensor has not reported | `PermissionDenied` (not Engineer); `RobotNotFound` | FR-005 (narrowed to status); AC-005 | `store`, `access` |

### 7. Mission Assignment (`app/modules/mission_assignment.py`)

Constructor: `MissionAssignment(store, registry, status, integration, access, clock: Callable[[], datetime], new_mission_id: Callable[[], MissionId])`

| Function | Success | Fails with | Spec rule | Injected |
|---|---|---|---|---|
| `assign_mission(user, request: MissionRequest) -> MissionAssignmentRecord` | Inside `store.run_in_transaction`, checks `get_mission_availability`, then stores a record with `recorded_at = clock()` and `status` set to `ASSIGNED` or `PENDING_EXECUTION`. Returns the record with its `robot_id` | `PermissionDenied` (not Robot Operator); `InvalidMission` when both `task` and `destination` are `None` or `""`; `RobotNotFound`; `RobotUnavailable(reason)` | FR-006; AC-006 all three scenarios; **Def** Available robot, Valid task or destination; ARCH §3 | `store`, `registry`, `status`, `integration`, `access`, `clock`, `new_mission_id` |

Not specified: when `send_mission_assignment` is called and what happens if delivery fails (ARCH §3 stops at recording); whether a whitespace-only task counts as empty.

### 8. User Interface (`app/modules/user_interface/`)

Each route handler in `routes.py` calls `access.authenticate(Authorization header)` and then exactly one module function (Part 2). The `schemas.py` `to_domain()` methods only copy fields. Injected: `Services` on `app.state`.

### Startup (`app/main.py`)

| Function | Success | Fails with | Spec rule | Injected |
|---|---|---|---|---|
| `connect_robot_integration(wiring: Wiring) -> None` | Subscribes `record_telemetry` and `record_sensor_status` to Robot Integration | Never stops startup. While Robot Integration is a stub, it logs `Robot Integration not connected; inbound telemetry is disabled` | FR-004, FR-005; ARCH §2 | `Wiring` |
| `status_freshness_timeout_from_env() -> timedelta \| None` | Reads `STATUS_FRESHNESS_TIMEOUT_SECONDS` | `ValueError` if the value is not a number | **Def** Status freshness timeout | environment |

---

## Part 2: REST API

Base path `/api/v1`. Credentials go in the `Authorization` header (format not specified). Every route currently returns:

```
HTTP 501
{"error": "not_implemented", "message": "<METHOD> <path> (<FR> -> <function>) is not implemented; first stub reached: <stub>"}
```

A body that fails schema validation (for example, a missing `software_version`) returns FastAPI's 422 before the handler runs.

| Requirement | Method and path | Request body | Calls | Planned success | Planned errors |
|---|---|---|---|---|---|
| FR-001 | `POST /robots` | `{robot_id, configuration: {software_version, operating_parameters?, sensor_settings?}, sensors?: [{sensor_id, sensor_type}]}` | `RobotRegistry.register_robot` | 201 `RobotOut` | 401; 403; 400 `InvalidRegistration`; 409 `DuplicateRobotIdentifier` |
| FR-002 | `GET /robots/{robot_id}/configuration` | — | `RobotRegistry.get_robot_configuration` | 200 `RobotConfigurationOut` | 401; 403; 404 `RobotNotFound` |
| FR-003 | `GET /dashboard` | — | `RobotStatusTelemetry.get_fleet_status` | 200 `{robots: [RobotStatusOut]}` | 401; 403; 500 `StatusFreshnessTimeoutNotSet` |
| FR-004 | `GET /robots/{robot_id}/telemetry/latest` | — | `RobotStatusTelemetry.get_latest_telemetry` | 200 `TelemetryOut`, or 200 `null` if none received | 401; 403; 404 `RobotNotFound` |
| FR-005 | `GET /robots/{robot_id}/sensors` | — | `SensorInformation.get_sensor_statuses` | 200 `SensorReportOut` | 401; 403; 404 `RobotNotFound` |
| FR-006 | `POST /missions` | `{robot_id, mission: {task?, destination?}}` | `MissionAssignment.assign_mission` | 201 `MissionAssignmentOut` | 401; 403; 404 `RobotNotFound`; 422 `InvalidMission`; 409 `RobotUnavailable` with `reason` |

The status codes are a proposal; the skeleton does not map exceptions to them yet. Telemetry collection (AC-004) reaches `record_telemetry` through `RobotIntegration.subscribe_telemetry`, not through REST.

---

## Resolution of the earlier open issues

| # | Issue | Resolution |
|---|---|---|
| 1 | FR-005 asked for sensor data that nothing provided | `docs/spec.md` FR-005 narrowed to "sensor status and identify sensors reporting a fault". No code change needed |
| 2 | Undefined terms | `docs/spec.md` Definitions list added, plus three rejection scenarios (AC-001 missing configuration, AC-004 invalid telemetry, AC-006 missing task or destination). AC-003 now references the freshness timeout. Contracts above implement each definition |
| 3 | Telemetry subscription not connected | `connect_robot_integration` runs at startup and logs a warning while the stub is in place. Handler types changed to `Callable[..., object]` |
| 4 | Inconsistent not-found errors | One `RobotNotFound` plus `SensorNotRegistered` in Persistence; `UnknownRobot` removed |

## Still open

- **The freshness timeout value.** The spec has a placeholder (`STATUS_FRESHNESS_TIMEOUT`). Until it is set, the dashboard and mission assignment raise `StatusFreshnessTimeoutNotSet`.
- **architecture.md §5.2 is now partly outdated.** It says the spec defines no freshness timeout. The spec now defines one, though without a value. I did not edit architecture.md.
