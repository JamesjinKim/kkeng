# Operation Sequence Table
(Based on "1. 전기 시퀜스 운전조건")

================================================================================
  SYSTEM / TANK          | CONDITION / TRIGGER    | ACTION
--------------------------------------------------------------------------------
  Flow Equalization Tank | Level Sensor High      | Start Flow EQ Pump (A/B)
  (유량조정조)              |                        |
                         |------------------------|---------------------------
                         | Level Sensor Low       | Stop Flow EQ Pump
--------------------------------------------------------------------------------
  DAF System             | Flow EQ Pump ON        | Start DAF System:
  (가압부상조)              |                        |   - PAC Pump
                         |                        |   - Polymer Pump
                         |                        |   - NaOH Pump (if pH Low)
                         |                        |   - Agitators
                         |                        |   - Circulation Pump
                         |------------------------|---------------------------
                         | pH Sensor High         | Stop NaOH Pump
--------------------------------------------------------------------------------
  1st Treatment Tank     | Level Sensor High      | Start 1st Treatment
  (1차 처리수조)            |                        | Pump (A/B)
                         |------------------------|---------------------------
                         | Level Sensor HH        | [INTERLOCK]
                         |                        | Stop Flow EQ Pump
--------------------------------------------------------------------------------
  Membrane Tank          | Level Sensor High      | Start Timer (7m ON/2m OFF)
  (분리막조)               |                        | -> Start Suction Pump
                         |------------------------|---------------------------
                         | Level Sensor Low       | Stop Suction Pump
                         |------------------------|---------------------------
                         | Level Sensor HH        | [INTERLOCK]
                         |                        | Stop 1st Treatment Pump
                         |------------------------|---------------------------
                         | Level Sensor HHAL      | [ALARM]
                         |                        | Trigger High Level Alarm
--------------------------------------------------------------------------------
  Filtrate Tank          | Level Sensor High      | Start Filtrate Pump (A/B)
  (여과수조)               |                        |
                         |------------------------|---------------------------
                         | Level Sensor H         | [INTERLOCK]
                         |                        | Stop Suction Pump
--------------------------------------------------------------------------------
  Discharge Tank         | Level Sensor High      | Start Discharge Pump (A/B)
  (방류조)                 |                        |
                         |------------------------|---------------------------
                         | Level Sensor HH        | [INTERLOCK]
                         |                        | Stop Filtrate Pump
--------------------------------------------------------------------------------
  Other Systems          | PLC / Timer            | Control Turbo Blower
                         |------------------------|---------------------------
                         | PLC Timer              | Control Return Pump
                         |------------------------|---------------------------
                         | PLC Schedule           | Control Cleaning Chem Pump
================================================================================
