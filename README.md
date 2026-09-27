# Snapmaker U1 Klipper Configuration Backup

This repository contains the Klipper configuration and backups for the **Snapmaker U1** CoreXY 3D printer.

## Included Custom Macros
* **`HEAT_SOAK_110`**: Automatically heats the bed to 110°C and dwells for 10 minutes with live 1-minute countdown reports sent to the console.
* **`SPREAD_GREASE`**: Exercises the Z-axis (20 cycles) and X/Y linear rails to evenly distribute lubricant and clean the motion system.

## File Overview
* `printer.cfg`: Active configuration file including the custom maintenance and heat soak macros.
* `printer_backup_with_macros_20260927_112555.cfg`: Timestamped backup including the new macros.
* `printer_backup_20260927_112337.cfg`: Original stock backup before modifications.
