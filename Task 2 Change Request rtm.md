# Task 2 – Change Requests & Traceability Matrix (RTM)

IDs used here come from Task 1 (FR/NFR/SR), Task 3 (S, E, T, X), Task 4 (OP, CS) and Task 5 (C).

## 2.1 Version-controlled change log

| Version | Date | CR | Req. ID | Status | Old text | New text | Reason |
|---|---|---|---|---|---|---|---|
| v1.0 | baseline | – | all | Approved | – | Task 1 baseline | – |
| v1.1 | CR-01 | CR-01 | FR-05 | **Modified** | The robot shall enter Safe Mode when a critical battery or thermal condition is detected. | The robot shall enter Safe Mode within 3 seconds when chamber or battery temperature exceeds the critical threshold or battery capacity falls below the emergency level. | Safety-critical behaviour needs a time bound and defined conditions |
| v1.2 | CR-02 | CR-02 | NFR-03 | **Modified** | The system shall support communication with multiple robots simultaneously. | The system shall support at least 20 simultaneously connected robots. | Pilot size is 20 robots; "multiple" cannot be tested |
| v1.3 | CR-03 | CR-03 | FR-01 | **Modified** | Only authenticated operators shall be permitted to issue robot commands. | The system shall require authenticated and role-authorised operators before accepting any robot command. | Authentication alone does not stop an unauthorised role from issuing commands |

Parameter values for FR-05 (Task 1): critical chamber temperature > 2 °C outside the profile limit, critical battery temperature > 60 °C, emergency battery level 5 %.

## 2.2 Why the new wording is more measurable / testable

| CR | Original weakness | Why the new wording is better |
|---|---|---|
| CR-01 | "critical" is undefined and there is no time limit, so no pass/fail time can be measured | It adds **a time limit (3 s)**, three **explicit trigger conditions** (chamber temperature, battery temperature, battery capacity) with numeric thresholds, and an **observable result** (state = SAFE_MODE). A test injects a value above the threshold, measures the time to SAFE_MODE and passes if ≤ 3 s. Boundary tests are possible (just below / just above) |
| CR-02 | "multiple" means 2 or 200; no test can fail | It gives **a number (≥ 20)**. A load test connects 20 simulated robots, checks that all send telemetry and receive commands, and passes or fails objectively. TC-14 tests above the minimum |
| CR-03 | "authenticated" covers only identity | It adds a **second checkable condition** (role authorisation) and a precise point (**before accepting any command**). Test matrix: {authenticated, not} × {authorised, not} → the command is accepted only in one of four cases, and rejection is observable (error code, log entry) |

## 2.3 System-level operations and invariants (supplementary IDs)

| ID | System operation | ID | System invariant |
|---|---|---|---|
| SOP-01 | AuthenticateOperator | INV-01 | `Command_Executed → (Authenticated ∧ Role_Authorised)` |
| SOP-02 | AuthoriseCommand (role + validation check) | INV-02 | `Critical_Condition for ≥ 3 s → Robot_State = SAFE_MODE` |
| SOP-03 | ExecuteCommand (apply transition) | INV-03 | `Event_Occurred → Log_Record_Exists` (on restore of link if buffered) |
| SOP-04 | ReportStatus (ack + telemetry) | INV-04 | `Connected_Robots ≤ 20 → Telemetry_Age ≤ 10 s` for every robot |
| SOP-05 | LogEvent | INV-05 | `¬(Prev_State = IDLE ∧ Next_State = DELIVERING)` |
| SOP-06 | EnterSafeMode | INV-06 | `(Battery ≤ 20% ∧ State ∈ {NAVIGATING, AVOIDING_OBSTACLE}) → State = RETURNING` within 2 s |
| SOP-07 | EvaluateCrossingPermit | INV-07 | `Chamber_State = CONSERVING → (Door_Closed ∧ Profile_Loaded)` |
| – | – | INV-08 | `Power_Lost → (Emergency_Power_On ∨ (Safe_Shutdown ∧ Incident_Logged))` |

## 2.4 Test cases (20)

Scenario steps: **S1** command received → **S2** authenticated → **S3** executed → **S4** status reported → **S5** logged.

| TC-ID | Title | Steps (summary) | Expected result | Req. | S1 | S2 | S3 | S4 | S5 |
|---|---|---|---|---|---|---|---|---|---|
| TC-01 | Valid delivery request | Authorised dispatcher sends DeliveryRequest to IDLE robot | Robot → NAVIGATING (T01), ack ≤ 10 s, log entry | FR-01, FR-03, FR-09, FR-10 | ✔ | ✔ | ✔ | ✔ | ✔ |
| TC-02 | Unauthenticated command | Send command with no / expired token | Rejected ERR_AUTH, state unchanged, logged | FR-01, FR-02, NFR-02 | ✔ | ✔ | – | ✔ | ✔ |
| TC-03 | Authenticated but unauthorised role | Viewer role sends DeliveryRequest | Rejected ERR_FORBIDDEN, state unchanged, logged (CR-03) | FR-01 | ✔ | ✔ | – | ✔ | ✔ |
| TC-04 | Status reporting delay | Execute command under 5 s link delay | Ack ≤ 10 s (95 %), telemetry every 5 s ± 1 s | FR-09, NFR-01 | ✔ | – | ✔ | ✔ | – |
| TC-05 | Event logging completeness | Run TC-01 to TC-03, then restart the system | Every step has a record with timestamp, operator, result; none lost | FR-10, NFR-04 | ✔ | ✔ | ✔ | ✔ | ✔ |
| TC-06 | Obstacle handling | Inject obstacle during NAVIGATING and RETURNING; send DestinationReached while avoiding; hold the obstacle 120 s | T02, T03, T10, T11; X02 ignored; T29 → SAFE_MODE | FR-04, SR-02 | – | – | ✔ | ✔ | ✔ |
| TC-07 | Safe Mode within 3 s (CR-01) | Inject chamber temp above critical, battery temp > 60 °C, capacity 4 %; also boundary values | SAFE_MODE ≤ 3 s in all states, alert, log; just below the threshold no entry | FR-05 | – | – | ✔ | ✔ | ✔ |
| TC-08 | CriticalBattery handling | Set battery to 20 % in NAVIGATING, AVOIDING_OBSTACLE, DELIVERING | T07, T08, T09 | FR-06 | – | – | ✔ | ✔ | ✔ |
| TC-09 | Link loss and recovery | Cut the link 30 s in each state; send a command during outage; restore | T19–T28; command not executed; buffered events delivered | SR-03, NFR-04 | ✔ | – | ✔ | ✔ | ✔ |
| TC-10 | Chamber correction and protection | Raise temperature; (a) correction works, (b) block the actuator | (a) OP-07 confirms and returns to CONSERVING; (b) PROTECTION after the recovery period, light reduced, alert | FR-07 | – | – | ✔ | ✔ | ✔ |
| TC-11 | Door open interlock | Open the door in NAVIGATING and DELIVERING | C-10 violation → SAFE_MODE; no DELIVERING with the door open and no authorisation | SR-02 | – | – | ✔ | ✔ | ✔ |
| TC-12 | Railway crossing | Train present, barrier closed, stale status (> 2 s), sensor disagreement | Robot never enters; stops before crossing | SR-01 | – | – | ✔ | ✔ | ✔ |
| TC-13 | Load test – 20 robots | 20 simulated robots send telemetry and receive commands for 1 h | All connected, ack ≤ 10 s, no loss (CR-02) | NFR-03, NFR-01, FR-09 | ✔ | ✔ | ✔ | ✔ | ✔ |
| TC-14 | Stress test – 25 robots | 25 robots, then drop and reconnect 5 | No degradation for the first 20; graceful behaviour above | NFR-03 | ✔ | – | ✔ | ✔ | – |
| TC-15 | Commands in SAFE_MODE | Send DeliveryRequest; send reset as authorised and as unauthorised operator | Request rejected (X04); reset accepted only for an authorised operator | FR-01, FR-05 | ✔ | ✔ | ✔ | ✔ | ✔ |
| TC-16 | Invalid transitions | Send DestinationReached in IDLE; DeliveryRequest in RETURNING | Ignored / rejected (X01, X05), state unchanged, logged | FR-02, FR-03, SR-02 | ✔ | – | – | ✔ | ✔ |
| TC-17 | Chamber start-up | Fail one sensor in self-check; start with door open; start with no profile | No MONITORING / CONSERVING in each case | FR-08 | – | – | ✔ | ✔ | ✔ |
| TC-18 | Security controls | 6 failed logins; idle 16 min; sniff the link | Lockout after 5; session expires; traffic encrypted | NFR-02 | ✔ | ✔ | – | – | ✔ |
| TC-19 | Usability | 10 operators identify an alerting robot and send a recall | ≤ 5 s to identify, ≤ 3 interactions, ≥ 90 % success | NFR-05 | ✔ | – | – | ✔ | – |
| TC-20 | Power loss | Cut the main power (with and without emergency power) | Emergency power ≤ 1 s, or incident logged and safe shutdown | SR-04 | – | – | ✔ | ✔ | ✔ |

## 2.5 Multi-layer RTM

Requirement → State/Transition → Operation → Constraint → Test Case → Risk Level

| Req. ID | State / Transition | Operation | Constraint / Invariant | Test Case | Risk Level |
|---|---|---|---|---|---|
| FR-01 (v1.3) | T01, T17, X03, X04 | SOP-01, SOP-02 | INV-01 | TC-01, TC-02, TC-03, TC-15 | High |
| FR-02 | T01, X01, X05 | SOP-02, SOP-03 | INV-01, INV-05 | TC-02, TC-16 | High |
| FR-03 | T01, T06, X01 | SOP-03 | INV-05 | TC-01, TC-16 | High |
| FR-04 | T02, T03, T10, T11, T29, X02 | SOP-03 | INV-05 | TC-06 | High |
| FR-05 (v1.1) | T12–T18, T25, T28 | SOP-06, OP-08 | INV-02, C-12 | TC-07, TC-15 | Critical |
| FR-06 | T07, T08, T09, T06 | SOP-03 | INV-06 | TC-08 | High |
| FR-07 | CS3 → CS4 → CS3 / CS5 | OP-05, OP-06, OP-07, OP-08, OP-09 | C-12 | TC-10 | Critical |
| FR-08 | CS1 → CS2 → CS3 | OP-01, OP-02, OP-03, OP-04 | INV-07 | TC-17 | High |
| FR-09 | all robot states | SOP-04, OP-02 | INV-04 | TC-04, TC-13 | Medium |
| FR-10 | all transitions | SOP-05, OP-18 | INV-03 | TC-05 | Medium |
| NFR-01 | T01 | SOP-02, SOP-03, SOP-04 | INV-04 | TC-04, TC-13 | Medium |
| NFR-02 | T01, X03 | SOP-01 | INV-01 | TC-02, TC-03, TC-18 | High |
| NFR-03 (v1.2) | all | SOP-04 | INV-04 | TC-13, TC-14 | High |
| NFR-04 | T19–T28 | SOP-05 | INV-03 | TC-05, TC-09 | High |
| NFR-05 | T12–T16 | OP-17 | – | TC-19 | Low |
| SR-01 | NAVIGATING / RETURNING motion guard | SOP-07 | C-01 … C-09 | TC-12 | Critical |
| SR-02 | T04, X01, X02, X04, CS7 | OP-12, OP-13, OP-16 | C-10, C-11, INV-05 | TC-11, TC-16, TC-06 | Critical |
| SR-03 | T19–T28, X04 | SOP-04, SOP-06 | C-07, C-13 | TC-09 | Critical |
| SR-04 | CS3 → CS8 / CS9 | OP-14, OP-15, OP-18 | INV-08 | TC-20 | Critical |

## 2.6 Re-run list per Change Request

| Test Case | CR-01 (FR-05) | CR-02 (NFR-03) | CR-03 (FR-01) | Reason |
|---|---|---|---|---|
| TC-01 | – | ✔ | ✔ | Command path changes with role check and with 20 robots |
| TC-02 | – | – | ✔ | Authentication rejection path |
| TC-03 | – | – | ✔ | New role-authorisation check (new test for CR-03) |
| TC-04 | ✔ | ✔ | – | Status of Safe Mode entry; telemetry at 20 robots |
| TC-05 | ✔ | ✔ | ✔ | Logging of Safe Mode, 20-robot load, rejected commands |
| TC-07 | ✔ | – | – | Directly tests the new 3 s limit (updated for new thresholds) |
| TC-08 | ✔ | – | – | Battery levels interact with the emergency level (20 % vs 5 %) |
| TC-09 | – | ✔ | – | Link handling for many robots |
| TC-10 | ✔ | – | – | Chamber thermal path feeds into Safe Mode |
| TC-13 | – | ✔ | – | Load test updated to exactly 20 robots |
| TC-14 | – | ✔ | – | Above-minimum stress |
| TC-15 | ✔ | – | ✔ | Commands and reset while in Safe Mode |
| TC-16 | – | – | ✔ | Command validation order (auth, role, then state) |
| TC-18 | – | – | ✔ | Security regression on the authentication path |
