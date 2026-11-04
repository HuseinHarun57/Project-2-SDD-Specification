# Architecture: Autonomous Robotics Management and Navigation Software Platform

## 1. Overview

The platform is proposed as a **centralized, modular application** that provides Engineers and Robot Operators with a shared interface for robot registration, configuration, monitoring, sensor information, and mission assignment. Its application modules share a persistent data store and communicate with registered robots through a robot-integration boundary, while low-level navigation, perception, and safety control remain on the robots themselves.

## 2. Modules

| Module | Responsibility | May depend on |
|---|---|---|
| User Interface | Present the fleet dashboard and the configuration, sensor, and mission-assignment views used by Engineers and Robot Operators. | Access Control; Robot Registry; Robot Status and Telemetry; Sensor Information; Mission Assignment |
| Access Control | Authenticate users and enforce Engineer and Robot Operator permissions at the application boundary. | User identity records |
| Robot Registry | Register robots, reject duplicate identifiers, and provide stored robot configuration and software-version information. | Persistence; Access Control |
| Robot Status and Telemetry | Receive robot telemetry, retain the latest reported status, and provide the dashboard with current or explicitly unavailable status information. | Robot Integration; Persistence; Access Control |
| Sensor Information | Provide Engineers with the available sensor status and identify sensors reporting faults. | Robot Integration; Persistence; Access Control |
| Mission Assignment | Check robot availability, record valid mission assignments, and expose the assignment status and assigned robot. | Robot Registry; Robot Status and Telemetry; Robot Integration; Persistence; Access Control |
| Robot Integration | Exchange the supported telemetry, status, and mission-assignment information between the platform and robot-side systems through robot-compatible interfaces. | Robot-side APIs or communication protocols |
| Persistence | Store and retrieve robot identifiers, configurations, software-version records, telemetry, sensor status, and mission-assignment records. | Persistent database |

The module boundaries are logical boundaries within one application; the specification does not require separate deployable services. The Robot Integration module is an interface boundary rather than a new robot-control system: it adapts communication with robot-side systems, but does not perform onboard navigation or safety-critical control.

## 3. Request flow

### Scenario: Assign a mission to an available robot (FR-006, AC-006)

1. The Robot Operator signs in and opens the mission-assignment view.
2. The User Interface sends the assignment request to the Mission Assignment module.
3. Access Control verifies that the signed-in user has the Robot Operator role and is permitted to assign missions.
4. Mission Assignment obtains the selected robot's current availability from Robot Status and Telemetry and confirms that the submitted mission contains the task or destination information required by the specification.
5. If the robot is unavailable, Mission Assignment rejects the request and returns the reason; the User Interface displays that explanation. This is the rejection path in AC-006.
6. If the robot is available, Mission Assignment records the association between the mission and the selected robot through Persistence.
7. Mission Assignment makes the recorded assignment and its assigned-or-pending-execution status available to the User Interface.
8. The User Interface displays the assigned robot and the mission's assignment status to the Robot Operator.

The availability check and recording of the assignment must be treated as one coordinated operation so that the platform does not knowingly accept an assignment based on an availability result that has already changed. 

## 4. Design decisions

| Decision | Reason |
|---|---|
| Centralized fleet management through one application and dashboard. | The project description and domain model describe a centralized interface; FR-001 and AC-001 require registered robots to appear in the fleet dashboard, and FR-003/AC-003 use that dashboard for status monitoring. |
| Role-based access for Engineers and Robot Operators. | The acceptance criteria require an Engineer to be logged in for registration, configuration viewing, and sensor information (AC-001, AC-002, AC-005), and a Robot Operator to be logged in for mission assignment (AC-006). The security NFR requires role-based access control and authenticated communication. |
| Modular application structure, without requiring microservices. | The maintainability and scalability goals support separation of responsibilities. The security NFR and FR-001 through FR-006 can be met by modules within one application; the specification does not require independent deployment, so a modular monolith is the simpler preference. |
| Keep high-level fleet functions in the platform and low-level navigation, perception, and safety control onboard the robot. | The project description says the platform integrates with robotic software to support perception, navigation, and decision-making, while the implementation constraint explicitly states that onboard safety mechanisms must remain responsible for safety-critical control. |
| Provide a robot-integration boundary for robot-specific communication. | The project description requires integration with autonomous robots and sensor information, and the hardware-compatibility constraint identifies multiple robot models and sensor types. The exact protocols are not specified; using an integration boundary is a design preference to isolate those differences. |
| Separate robot status/telemetry handling from sensor-information presentation. | FR-003/AC-003 and FR-004/AC-004 concern robot status and telemetry, while FR-005/AC-005 specifically concerns sensor status and fault identification. Separate modules keep these responsibilities distinct. |
| Use event-triggered evaluation for abnormal conditions only when alert requirements are specified. | The goals mention prompt alerts, but the supplied functional requirements and acceptance criteria do not define alert behavior. An event-based alerting module is therefore not included in this minimum architecture; adding it would be a future design decision once the missing requirements are specified. |
| Persist robot registration, configuration, telemetry, and mission-assignment records. | AC-001 requires registration to be retained and shown on the dashboard; AC-004 requires received telemetry to be recorded; AC-006 requires the mission assignment to be recorded. |
| Distinguish unavailable status from current status. | AC-003 explicitly requires unavailable information to be identified and outdated information not to be presented as current. The Robot Status and Telemetry module enforces this rule when providing data to the dashboard. |
| Reject duplicate robot identifiers. | AC-001 explicitly requires duplicate registration to be rejected with an explanation. The Robot Registry enforces uniqueness when registering a robot, backed by a uniqueness constraint in Persistence. |
| Enforce authorization in the application, not only by hiding interface controls. | The security NFR requires role-based access control. Access Control checks permissions on each protected operation so that direct requests cannot bypass the role restrictions. |
| Use a single application data store for the specified records. | AC-001, AC-004, and AC-006 require records to be retained, but the specification does not require a particular database technology or distributed storage. A single persistent store is the simplest preference for the current scope. |
| Use simulation before connecting physical robots. | This follows the stated simulation-first implementation decision and the academic development constraint. It is a development preference; the supplied functional requirements do not make simulation a runtime system capability. |

## 5. Constraints and known limits

1. **Hardware compatibility:** The platform must work with different robot models and onboard systems, including LiDAR, cameras, GPS, and IMUs. The architecture isolates robot-specific communication in Robot Integration, but the specification does not identify supported robot models, APIs, protocols, or sensor data formats. Each additional robot family may require a compatible integration implementation and validation.

2. **Communication limitations:** Telemetry reception and robot interaction depend on network connectivity. AC-003 requires unavailable status to be shown as unavailable rather than presenting stale information as current. The specification does not define a freshness timeout, reconnection policy, offline command behavior, or how delayed and out-of-order telemetry is handled.

3. **Real-time data processing:** The performance NFR sets a two-second dashboard update target after data is received under normal operating conditions. This design places the update path through Robot Integration, Robot Status and Telemetry, Persistence, and the User Interface. The target does not define behavior during network outages or under peak load.

4. **Safety dependencies:** The platform does not replace onboard emergency-stop, collision-avoidance, or other safety mechanisms. The supplied functional requirements do not include a pause/stop operation or its acceptance criteria, despite the safety goal and the one-second command NFR. A production design for those operations would need explicit command, acknowledgement, failure, and safety requirements, plus validation against the robot's onboard safety system.

5. **Security and access control:** Access Control enforces authenticated access and role permissions. The specification identifies roles and protected activities but does not define the identity provider, credential lifecycle, session policy, encryption standards, audit requirements, or detailed permission matrix.

6. **Integration dependencies:** External warehouse-management and robot-system integration depends on compatible APIs and communication protocols. The external warehouse system is optional for the initial MVP in the domain model, so it is not a separate module in this minimum architecture. If warehouse integration becomes mandatory, its data ownership, interface contracts, error handling, and synchronization requirements must be added.

7. **Fleet scale:** The scalability NFR specifies support for at least 100 robots without architectural redesign while retaining the stated performance target. The specification does not provide telemetry frequency, payload sizes, concurrent user counts, retention periods, or peak-load assumptions. Those values are needed to size and validate the database and application deployment.

8. **Growth in requirements:** If the system later needs alert generation, problem escalation, diagnostics workflows, software updates, maintenance records, mission-history analysis, or route planning, the specification must first define the relevant functional requirements and acceptance criteria. The corresponding responsibilities can then be added as modules or extended within existing modules. The current supplied functional scope does not justify adding separate services or infrastructure for those capabilities.

## 6. Out of scope
Project Non-Goals
1. Physical Robot Design: The project will not involve designing, manufacturing, or modifying the physical structure or hardware of autonomous robots.

2. Low-Level Robot Control: The platform will not replace onboard robot controllers responsible for motor control, navigation, and movement execution.

3. Safety-Critical Control Implementation: The platform will not independently implement collision avoidance or replace the robots' built-in emergency-stop and safety mechanisms.

4. Sensor Hardware Development: The project will not involve developing or manufacturing sensors such as LiDAR, cameras, GPS, or IMUs; it will use data provided by existing sensors.

5. Full Warehouse Management: The platform will not replace a warehouse management system (WMS) or manage broader business operations such as inventory tracking, order processing, and supply-chain management.
