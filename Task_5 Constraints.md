# Task 5 – Constraints, Formalisation & Violations (Railway Crossing)

## 5.1 Constraints (13)

| ID | Constraint (simple English) | Why necessary |
|---|---|---|
| C-01 | While a train is approaching or present the barrier must be closed | Prevents road users and robots meeting a train |
| C-02 | The robot may enter the crossing only if the barrier is open and no train is approaching or present | Core crossing safety rule |
| C-03 | The barrier may open only after the crossing is confirmed clear and no train is present | Avoids opening under a passing train |
| C-04 | While a train is approaching the warning lights and audible alarm must be on | Warns anyone near the crossing |
| C-05 | If a close command is not followed by a closed barrier within 10 s, a barrier failure must be declared | Detects barrier failure; triggers fail-safe |
| C-06 | If the train-detection sensor has failed, the barrier must be closed and no crossing permit given (fail-safe) | A blind system must assume a train |
| C-07 | If the ARLCCS link is down or its status is older than 2 s, the robot must have no crossing permit | Robot must not act on stale data |
| C-08 | If redundant sensors disagree, a train is assumed present | Handles incorrect readings conservatively |
| C-09 | A robot on the crossing must keep moving to clear it, even if Safe Mode is triggered | A stopped robot on the tracks is a collision hazard |
| C-10 | The chamber door must be closed while the robot is NAVIGATING, AVOIDING_OBSTACLE or RETURNING (robot–chamber) | Protects the batch and avoids spillage while moving |
| C-11 | If the robot is DELIVERING and the door is open, removal must be authorised, the chamber safe and no protection response active (robot–chamber) | A batch may be removed only in a safe condition |
| C-12 | If chamber or battery temperature stays above the critical threshold for 3 s the robot must be in SAFE_MODE (CR-01, robot–chamber) | Real-time safety bound |
| C-13 | In LINK_LOST the robot must not enter a crossing or begin a delivery | Operator cannot supervise |

## 5.2 Formalisation

| ID | Formal expression |
|---|---|
| C-01 | `(Train_Present ∨ Train_Approaching) → ¬Barrier_Open` |
| C-02 | `Robot_Entering_Crossing → (Barrier_Open ∧ ¬Train_Approaching ∧ ¬Train_Present)` |
| C-03 | `Barrier_Open_Cmd → (Crossing_Clear ∧ ¬Train_Present)` |
| C-04 | `Train_Approaching → (Warning_Lights_On ∧ Audible_Alarm_On)` |
| C-05 | `(Close_Cmd ∧ Close_Cmd_Age > 10 s ∧ ¬Barrier_Closed) → Barrier_Fault` |
| C-06 | `¬Train_Sensor_OK → (Barrier_Closed ∧ ¬Crossing_Permit)` |
| C-07 | `(¬ARLCCS_Link_OK ∨ Status_Age > 2 s) → ¬Crossing_Permit` |
| C-08 | `(Sensor_A_Train ≠ Sensor_B_Train) → Train_Assumed_Present` |
| C-09 | `(Robot_On_Crossing ∧ Safe_Mode) → Robot_Moving_To_Exit` |
| C-10 | `(Robot_State = NAVIGATING ∨ Robot_State = AVOIDING_OBSTACLE ∨ Robot_State = RETURNING) → Door_Closed` |
| C-11 | `(Robot_State = DELIVERING ∧ Door_Open) → (Batch_Removal_Authorised ∧ Chamber_Safe ∧ ¬Protection_Active)` |
| C-12 | `(Chamber_Temp > T_crit ∨ Batt_Temp > T_bcrit ∨ Batt_Cap < 5%) for ≥ 3 s → Robot_State = SAFE_MODE` |
| C-13 | `Robot_State = LINK_LOST → (¬Robot_Entering_Crossing ∧ ¬Delivery_Start)` |

## 5.3 and 5.4 Violation scenarios, detection and expected response

| ID | Violation scenario (variable values) | How the violation is detected | Expected response |
|---|---|---|---|
| C-01 | `Train_Approaching = TRUE, Barrier_Open = TRUE` (barrier stuck open) | ARLCCS safety monitor compares train sensor with barrier position sensor | Alarm on, barrier close retried, robot stopped before crossing, operator alert, event logged |
| C-02 | `Robot_Entering_Crossing = TRUE, Barrier_Open = FALSE, Train_Approaching = TRUE` | Robot monitor compares position with crossing zone and the ARLCCS status | Robot emergency stop (if before the zone), Safe Mode if inside, operator alert, event logged |
| C-03 | `Barrier_Open_Cmd = TRUE, Crossing_Clear = FALSE, Train_Present = TRUE` | Safety unit interlock checks clear-confirmation before executing the command | Command blocked, barrier stays closed, alarm kept on, event logged |
| C-04 | `Train_Approaching = TRUE, Warning_Lights_On = FALSE` (lamp failure) | Lamp current monitor and the audible-alarm feedback | Barrier closed, fault alarm to Mission Control, crossing marked unavailable, robot stops, event logged |
| C-05 | `Close_Cmd = TRUE, Close_Cmd_Age = 12 s, Barrier_Closed = FALSE` | Barrier timeout timer in the safety unit | `Barrier_Fault` raised, crossing marked unsafe, robot stopped, operator alerted, event logged |
| C-06 | `Train_Sensor_OK = FALSE, Barrier_Closed = FALSE, Crossing_Permit = TRUE` | Sensor self-diagnostic (heartbeat/range check) | Barrier closed, permit withdrawn, robot stops, operator alert, event logged |
| C-07 | `ARLCCS_Link_OK = FALSE, Status_Age = 6 s, Crossing_Permit = TRUE` | Robot status-age watchdog (limit 2 s) | Permit cleared, robot holds before the crossing, alert after 30 s, LINK_LOST if the heartbeat is lost, event logged |
| C-08 | `Sensor_A_Train = TRUE, Sensor_B_Train = FALSE, Train_Assumed_Present = FALSE` | Redundancy voter compares the two sensors | Train assumed present, barrier closed, robot held, maintenance alert, event logged |
| C-09 | `Robot_On_Crossing = TRUE, Safe_Mode = TRUE, Robot_Moving_To_Exit = FALSE` | Robot monitor: speed = 0 inside the crossing zone | Safe Mode stop overridden, robot drives out, operator alert, barrier exit-lane request, event logged |
| C-10 | `Robot_State = NAVIGATING, Door_Open = TRUE` | Door sensor evaluated on every state sample | Robot stops, `CriticalCondition` → SAFE_MODE, chamber enters OP-12, operator alert, event logged |
| C-11 | `Robot_State = DELIVERING, Door_Open = TRUE, Protection_Active = TRUE, Batch_Removal_Authorised = FALSE` | Door lock monitor and chamber state | Door re-locked / handover refused, alert, no transition to RETURNING until closed, event logged |
| C-12 | `Chamber_Temp = 11 °C (limit 10 °C) for 4 s, Robot_State = NAVIGATING` | Real-time monitor timing the condition | Robot forced to SAFE_MODE, chamber protection active, operator alerted, event logged (test TC-07) |
| C-13 | `Robot_State = LINK_LOST, Robot_Entering_Crossing = TRUE` | Robot monitor on state and crossing zone | Robot stops before the crossing, event buffered and logged, reported when the link is restored |

## 5.5 Required constraint categories

| Requirement of Task 5.5 | Constraints |
|---|---|
| Robot–chamber links | C-10, C-11, C-12 |
| Sensor failure / communication loss | C-06, C-07, C-08, C-13 |

## 5.6 Decidability

Every constraint uses only observable Boolean variables, state names, or timers with numeric limits (3 s, 10 s, 2 s, 30 s), so a test or runtime monitor can decide from the observed values whether it holds.
