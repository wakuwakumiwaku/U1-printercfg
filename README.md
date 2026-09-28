# Snapmaker U1 Klipper Configuration Backup

This repository contains the Klipper configuration and backups for the **Snapmaker U1** CoreXY 3D printer.

## Key Configuration Changes

### 1. Nozzle Fan Chamber Guard Optimization (E0 – E3)
* **Problem:** Stock configuration set `external_temp_guard_range: -15.0, 45.0` with `external_temp_guard_fan_speed: 1.0`. When heat-soaking the chamber for ASA/ABS ($>45^\circ\text{C}$), all four docked nozzle fans automatically screamed at 100% continuous speed.
* **Modification:**
  * Raised upper chamber guard limit: `external_temp_guard_range: -15.0, 55.0` (prevents fans from triggering prematurely during heat-soaks).
  * Reduced guard fan speed: `external_temp_guard_fan_speed: 0.5` (runs quietly at 50% instead of 100% if triggered).
  * Applied across all 4 toolhead fan sections: `e0_nozzle_fan`, `e1_nozzle_fan`, `e2_nozzle_fan`, and `e3_nozzle_fan`.

---

## Included Custom Macros

### 2. Heat Soak Macros (`HEAT_SOAK_20`, `HEAT_SOAK_30`, `HEAT_SOAK_110`)
* **Automated Sequence:**
  1. Performs homing (`G28`).
  2. Positions nozzle 20 mm above bed at center (`G1 Z20 F1000`, `G1 X135 Y150 F3000`).
  3. Activates `cavity_fan` at 15% and the active toolhead's part cooling fan at 15% to circulate chamber heat evenly.
  4. Heats bed to 110°C (`M190 S110`).
  5. Dwells for 20 minutes (`HEAT_SOAK_20`), 30 minutes (`HEAT_SOAK_30`), or 10 minutes (`HEAT_SOAK_110`).
  6. Sends live 1-minute countdown reports to the console using Klipper's `[respond]` module (`RESPOND MSG=...` and `M117`).

### 3. `SPREAD_GREASE`
* Maintenance macro to cycle the Z-axis lead screws (20 cycles, Z10 to Z230) and X/Y linear rails in diagonal and box sweeps.
* Used for breaking in and evenly distributing grease after maintenance.

---

## File Overview
* `printer.cfg`: Active configuration file including the custom maintenance and heat soak macros.
* `printer_backup_20260928_072953.cfg`: Timestamped backup before adding the 20 & 30 min heat soak macros.
* `printer_backup_with_macros_20260927_112555.cfg`: Timestamped backup including initial macros and fan guard adjustments.
* `printer_backup_20260927_112337.cfg`: Original stock backup before modifications.
