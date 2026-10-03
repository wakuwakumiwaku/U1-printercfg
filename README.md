# Snapmaker U1 Klipper Configuration Backup

This repository contains the Klipper configuration and backups for the **Snapmaker U1** CoreXY 3D printer.

## Key Configuration Changes

### 1. Nozzle Fan Chamber Guard Optimization (E0 – E3)
* **Problem:** Stock configuration set `external_temp_guard_range: -15.0, 45.0` with `external_temp_guard_fan_speed: 1.0`. When heat-soaking the chamber for ASA/ABS ($>45^\circ\text{C}$), all four docked nozzle fans automatically screamed at 100% continuous speed.
* **Modification:**
  * Raised upper chamber guard limit: `external_temp_guard_range: -15.0, 55.0` (prevents fans from triggering prematurely during heat-soaks).
  * Guard fan speed: `external_temp_guard_fan_speed: 1.0` (runs at 100% full airflow above 55 °C to prevent toolhead STM32/TMC2209 thermal throttling and USB packet loss during high-temperature prints).
  * Applied across all 4 toolhead fan sections: `e0_nozzle_fan`, `e1_nozzle_fan`, `e2_nozzle_fan`, and `e3_nozzle_fan`.

### 2. Power Loss Detection Debounce (`power_loss_trigger_time`)
* **Problem:** Stock `power_loss_trigger_time: 0.022` (22 ms) is overly sensitive. High-load prints (110 °C bed + 268 °C hotend) cause transient ripple or micro-dips on the 24V line, triggering false power-off signals or emergency shutdowns.
* **Modification:**
  * Set `power_loss_trigger_time: 0.05` (50 ms) under `[power_loss_check]`, per Snapmaker official troubleshooting guidance, to debounce transient voltage dips without compromising blackout state saves.

---

## Included Custom Macros

### 3. Heat Soak Macros (`HEAT_SOAK_20`, `HEAT_SOAK_30`, `HEAT_SOAK_110`)
* **Automated Sequence:**
  1. Performs homing (`G28`).
  2. Positions nozzle 20 mm above bed at center (`G1 Z20 F1000`, `G1 X135 Y150 F3000`).
  3. Activates `cavity_fan` at 15% and the active toolhead's part cooling fan at 15% to circulate chamber heat evenly.
  4. Heats bed to 110°C (`M190 S110`).
  5. Dwells for 20 minutes (`HEAT_SOAK_20`), 30 minutes (`HEAT_SOAK_30`), or 10 minutes (`HEAT_SOAK_110`).
  6. Sends live 1-minute countdown reports to the console using Klipper's `[respond]` module (`RESPOND MSG=...` and `M117`).

### 4. `SPREAD_GREASE`
* Maintenance macro to cycle the Z-axis lead screws (20 cycles, Z10 to Z230) and X/Y linear rails in diagonal and box sweeps.
* Used for breaking in and evenly distributing grease after maintenance.

---

## File Overview
* `printer.cfg`: Active configuration file including custom maintenance and heat soak macros, 100% nozzle fan chamber guard, and power loss debounce.
* `printer_backup_20261003_010800.cfg`: Timestamped backup with `external_temp_guard_fan_speed: 1.0` (100% cooling above 55 °C chamber).
* `printer_backup_PRE-StealthChop.cfg`: Backup taken before the StealthChop experiment (StealthChop tested and reverted due to CoreXY resonance and phase lag).
* `printer_backup_20261001_235000.cfg`: Timestamped backup before enabling StealthChop.
* `printer_backup_PRE-Timingthreshold.cfg`: Backup taken before changing `power_loss_trigger_time` from 0.022 to 0.05.
* `printer_backup_20260928_072953.cfg`: Timestamped backup before adding the 20 & 30 min heat soak macros.
* `printer_backup_with_macros_20260927_112555.cfg`: Timestamped backup including initial macros and fan guard adjustments.
* `printer_backup_20260927_112337.cfg`: Original stock backup before modifications.
* `errors/`: Documentation and raw logs for printer incidents and diagnostic investigations.

---

## Errors & Incident Logs

### [2026-10-03] Recurring MCU 'e1' Disconnects & Heater Verification Errors (Toolhead 2)
* **Status:** Open / Critical Intermittent Hardware Fault (SnapSwap Pogo-Pin Dock & USB-C Harness).
* **Detailed Report:** [`errors/2026-10-03_mcu_e1_disconnect_and_heater_failure_cluster.md`](errors/2026-10-03_mcu_e1_disconnect_and_heater_failure_cluster.md)
* **Log Signature:**
  ```text
  Lost communication with MCU 'e1' (Code: 0003-0522-0003-0008, ID: 522)
  Heater extruder1 not heating at expected rate, temp: 271.03 target: 270.00 (Code: 0003-0523-0001-0003, ID: 523)
  ```
* **Summary:** Cluster of 10 shutdowns (8x MCU disconnects, 2x heater verify trips) during an overnight high-temp ASA print (`u1-motherboard-fan-chance1_PLA_4h54m.gcode`) under sustained 55–60 °C chamber temps. The printer automatically recovered 10 times via `power_loss` resume. Serial telemetry proved massive packet retransmits (`bytes_retransmit=9120` and `8633` on `e1` vs `9` on others).
* **Root Cause & Action:** Contact resistance and thermal fretting on Toolhead 2 carriage dock pogo-pins. Mandatory IPA cleaning of pins/pads and spring compliance check required.

### [2026-10-02] Internal error on command:"G1" / Lost communication with MCU 'e1' (Toolhead 2)
* **Status:** Open / Intermittent Hardware Fault (USB-C cable vs. Toolhead 2 PCB socket).
* **Detailed Report:** [`errors/2026-10-02_internal_error_g1_mcu_e1_disconnect.md`](errors/2026-10-02_internal_error_g1_mcu_e1_disconnect.md)
* **Log Signature:**
  ```text
  !! Internal error on command:"G1"
  b'stepcompress o=15 i=0 c=8 a=0: Invalid sequence'
  Lost communication with MCU 'e1' (Code: 0003-0522-0003-0008, ID: 522)
  ```
* **Summary:** Occurred ~12.3 hours into `u1-motherboard-fan-chance1_PLA_5h39m.gcode` at 270 °C nozzle, 110 °C bed, 57 °C chamber. The generic `G1` failure was a secondary crash of Klipper's step compression when MCU `e1` disconnected. Toolhead reconnected normally after restart.
* **Diagnosis:** Dynamic USB-C cable fatigue vs. PCB USB-C receptacle contact fretting. Cross-swap test with Toolhead 1 to be performed for RMA.

### [2026-10-01] Lost communication with MCU 'e1' (Toolhead 1)
* **Status:** Resolved / Hardware OK (Intermittent bus/contact timeout).
* **Detailed Report:** [`errors/2026-10-01_mcu_e1_lost_communication.md`](errors/2026-10-01_mcu_e1_lost_communication.md)
* **Log Signature:**
  ```text
  08:18:40.045: Timeout with MCU 'e1' (eventtime=2084231.798893)
  08:18:40.045: Transition to shutdown state: {"coded": "0003-0522-0003-0008", "oneshot": 0, "msg":"Lost communication with MCU 'e1'"}
  ...
  !! Lost communication with MCU 'e1'
  // Klipper state: Disconnect
  ```
* **Summary:** Occurred ~52 min into an ASA print (`u1-motherboard-fan-chance1_PLA_5h40m.gcode`) at 110 °C bed, 58 °C chamber, 268 °C nozzle, during a high-acceleration wipe move (`ACCEL=10000`, `F9000`). MCU reconnected successfully after firmware restart.
* **Root Cause & Action:** Micro-debris/vibration on SnapSwap pogo-pin contacts at ~938 operating hours. Cleaned pogo-pins and contact pads with IPA, verified pin travel, and checked cable harness strain relief.

