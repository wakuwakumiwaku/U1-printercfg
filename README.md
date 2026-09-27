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

### 2. `HEAT_SOAK_110`
* Automatically heats the bed to 110°C and dwells for 10 minutes.
* Sends live 1-minute countdown reports to the console using Klipper's `[respond]` module.

### 3. `SPREAD_GREASE`
* Maintenance macro to cycle the Z-axis lead screws (20 cycles, Z10 to Z230) and X/Y linear rails in diagonal and box sweeps.
* Used for breaking in and evenly distributing grease after maintenance.

---

## File Overview
* `printer.cfg`: Active configuration file including the custom maintenance and heat soak macros.
* `printer_backup_with_macros_20260927_112555.cfg`: Timestamped backup including the new macros and fan guard adjustments.
* `printer_backup_20260927_112337.cfg`: Original stock backup before modifications.
