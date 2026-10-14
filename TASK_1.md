# Task 1 – Requirement Extraction (SACDCS)

**Role:** Software Quality Engineering Lead  **Baseline:** v1.0  **Pilot scope:** 20 clinics, up to 20 robots

## Assumed parameters (to be confirmed with Mission Control)

| Parameter | Value | Used in |
|---|---|---|
| Link-loss declaration | no heartbeat for 30 s | SR-03 |
| CriticalBattery (return level) | ≤ 20 % | FR-06 |
| EmergencyBattery level | ≤ 5 % | FR-05 / CR-01 |
| Critical battery temperature | > 60 °C | FR-05 / CR-01 |
| Critical chamber temperature | > 2 °C outside the batch profile limit (e.g. > 10 °C for a 2–8 °C profile) | FR-05 / CR-01 |
| Recovery period (chamber) | 10 min (configurable per profile) | FR-07 |
| Vibration stabilisation period | 60 s below threshold | FR-07 |
| ARLCCS status maximum age | 2 s | SR-01 |
| Link delay (one-way) | up to 5 s | NFR-01 |

## 1. Functional Requirements (10)

| Req. ID | Requirement | Type | Priority | Verification Method |
|---|---|---|---|---|
| FR-01 | Only authenticated operators shall be permitted to issue robot commands. | FR | High | Test (TC-01, TC-02) |
| FR-02 | **Command validation:** before executing any command the robot controller shall check the command format, the parameter ranges and that the command is allowed in the current robot state. An invalid command shall be rejected with an error code, the robot state shall stay unchanged and the rejection shall be logged. | FR | High | Test (TC-02, TC-16) + Inspection |
| FR-03 | **Delivery start:** a valid DeliveryRequest received in IDLE shall move the robot to NAVIGATING and send an acknowledgement. A robot in IDLE shall never move directly to DELIVERING. | FR | High | Test (TC-01, TC-16) + State-model verification |
| FR-04 | **Obstacle handling:** while NAVIGATING the robot shall check for obstacles continuously. When one is detected it shall suspend navigation, enter AVOIDING_OBSTACLE and resume NAVIGATING (or RETURNING) only after ObstacleCleared. | FR | High | Test (TC-06) + Simulation |
| FR-05 | The robot shall enter Safe Mode when a critical battery or thermal condition is detected. | FR | Critical | Test (TC-07) |
| FR-06 | **Low-battery return:** when the battery falls to ≤ 20 % during NAVIGATING or AVOIDING_OBSTACLE the robot shall abandon the delivery, go to RETURNING and become IDLE on arrival at the warehouse. | FR | High | Test (TC-08) + Simulation |
| FR-07 | **Chamber correction:** when temperature or humidity leaves the batch profile range the chamber shall start correction and shall confirm from sensor readings that the value is back in range. If it is not back in range within the recovery period the chamber shall enter protection mode, reduce light exposure and alert the operator. | FR | Critical | Test (TC-10) + Fault injection |
| FR-08 | **Chamber start-up:** the chamber shall enter monitoring only when the power-on self-check passes for all essential sensors, and shall start active conservation only when the door is closed and the batch profile is loaded. | FR | High | Test (TC-17) |
| FR-09 | **Status reporting:** every accepted or rejected command shall be acknowledged to Mission Control, and each robot shall send telemetry (state, position, battery, chamber temperature/humidity, door status) every 5 s ± 1 s. | FR | High | Test (TC-04, TC-13) |
| FR-10 | **Event logging:** the system shall log every command (operator ID, command, result), state transition, alarm and Safe Mode entry with a timestamp. | FR | High | Test (TC-05) + Inspection of log |

## 2. Non-Functional Requirements (5)

| Req. ID | Requirement | Type | Priority | Verification Method |
|---|---|---|---|---|
| NFR-01 (Performance) | Command acknowledgement shall reach Mission Control within 10 s for 95 % of commands with a one-way link delay of up to 5 s, and Mission Control shall process an operator command in ≤ 1 s. | NFR | High | Performance test (TC-04, TC-13) |
| NFR-02 (Security) | All links shall use TLS 1.2 or higher; operator sessions expire after 15 min inactivity; an account is locked for 15 min after 5 failed logins. | NFR | High | Security test (TC-18) + Inspection |
| NFR-03 (Scalability) | The system shall support communication with multiple robots simultaneously. | NFR | High | Load test (TC-13, TC-14) |
| NFR-04 (Reliability) | Mission Control shall be available ≥ 99.5 % per month, and each robot shall buffer ≥ 24 h of events during link loss with zero events lost once the link is restored. | NFR | High | Reliability / fault-injection test (TC-05, TC-09) |
| NFR-05 (Usability) | A trained operator shall identify an alerting robot within 5 s and issue a recall command in ≤ 3 interactions; ≥ 90 % of 10 test operators shall complete the task set without error. | NFR | Medium | Usability test (TC-19) |

## 3. High-Risk Requirements (4)

| Req. ID | Requirement | Type | Priority | Verification Method |
|---|---|---|---|---|
| SR-01 (Railway crossing) | The robot shall enter a railway level-crossing only when the ARLCCS reports barrier open **and** no train approaching, with a status age ≤ 2 s. If the status is missing, stale or conflicting the robot shall stop before the crossing. | Safety | Critical | Test (TC-12) + Hazard analysis + Simulation |
| SR-02 (Delivery interlock) | The robot shall enter DELIVERING only from NAVIGATING with the chamber door closed, no protection response active and not in AVOIDING_OBSTACLE or Safe Mode. If the door opens during conservation, normal operation shall be suspended until conditions and sensors are verified. | Safety | Critical | Test (TC-11, TC-16) + State-model verification |
| SR-03 (Link-dependent) | If no heartbeat is received for 30 s the robot shall enter LINK_LOST, accept no new delivery, not enter a crossing and return autonomously to the warehouse; it shall buffer events and report them when the link is restored. | Safety | Critical | Fault-injection test (TC-09) |
| SR-04 (Real-time power) | On power loss the chamber shall switch to emergency power within 1 s if available; otherwise it shall record the incident and shut down safely. | Safety | Critical | Fault-injection test (TC-20) |
