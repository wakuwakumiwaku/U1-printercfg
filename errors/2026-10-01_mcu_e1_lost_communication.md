# Incident Report: Lost communication with MCU 'e1' (Toolhead 1)

* **Date / Time:** 2026-10-01 08:18:40 UTC (10:18:40 Local)
* **Printer:** Snapmaker U1 (Serial: `8110026010700126YWSV`)
* **Firmware:** Klipper `1.3.0` / Linux Buildroot `6.1`
* **Operating Hours:** ~938 hours
* **Target Hardware:** Toolhead 1 / Extruder 1 MCU (`e1` - STM32F105)
* **Active Print File:** `u1-motherboard-fan-chance1_PLA_5h40m.gcode`
* **Status:** Resolved / Hardware intact (Intermittent bus/contact timeout)

---

## 1. Incident Summary

During the execution of a multi-hour print using Extruder 1 (ASA filament), Klipper abruptly transitioned into an emergency shutdown state with the error:

```text
!! Lost communication with MCU 'e1'
// Klipper state: Disconnect
```

Inspection of the printer and telemetry confirmed:
* **Toolhead 1 hardware is not damaged.** After a firmware restart / power cycle, MCU `e1` reconnected immediately, loaded all 138 commands, and reported valid thermistor telemetry (~65 °C cooling down).
* The shutdown was triggered by a communication loss exceeding 25 ms between the host and the toolhead 1 board during a high-acceleration travel move.

---

## 2. Telemetry at Disconnect

At the moment of the crash (08:18:38 - 08:18:40 UTC):
* **Extruder 1 Temperature:** 268.3 °C (Target: 267.0 °C)
* **Heated Bed Temperature:** 109.9 °C (Target: 110.0 °C)
* **Chamber / Cavity Temperature:** 58.0 °C
* **Toolhead Coordinates:** $X = 177.50\text{ mm},\; Y = 207.61\text{ mm},\; Z = 24.40\text{ mm}$
* **Motion State:** High-speed nozzle wipe / travel move with acceleration `ACCEL=10000 mm/s²`, `ACCEL_TO_DECEL=5000`, feedrate `F9000` (150 mm/s).

---

## 3. Raw Log Excerpts (`klippy.log`)

### Pre-crash motion queue & stats:
```text
08:18:38.022: Stats 2084229.8: gcodein=0 mcu: ... e1: mcu_awake=0.008 mcu_task_avg=0.000011 mcu_task_stddev=0.000010 
bytes_write=123433077 bytes_read=79960423 send_seq=2374726 receive_seq=2374726 freq=144000344 
heater_bed: target=110 temp=109.9 pwm=0.101 cavity: temp=58.0 extruder1: target=267 temp=268.3 pwm=0.074

08:18:40.051: Virtual sdcard (15112774): '... G1 X178.142 Y207.361 E.00921\nSET_VELOCITY_LIMIT ACCEL=10000 ACCEL_TO_DECEL=5000\nG1 E-1.1 F1800\n;WIPE_START\nG1 F9000\nG1 X177.826 Y207.446 E-.06529\n'
08:18:40.052: gcode state: last_position=[177.5023, 207.6117, 24.4016, 119904.009] speed=150.0
```

### The Timeout & Emergency Shutdown:
```text
08:18:40.045: Timeout with MCU 'e1' (eventtime=2084231.798893)
08:18:40.045: Transition to shutdown state: {"coded": "0003-0522-0003-0008", "oneshot": 0, "msg":"Lost communication with MCU 'e1'"}
08:18:40.047: Force record power loss print file env
...
08:19:51.896: Lost communication with MCU 'e1'
08:19:51.898: Raising exception: id:522 index:3 code:8 oneshot:0 level:3 is_persistent:0, message: Lost communication with MCU 'e1'
```

### Reconnection on Restart:
```text
08:20:11.532: [gcode] FIRMWARE_RESTART
08:20:13.089: Start printer at Thu Oct  1 08:20:13 2026
08:20:17.943: mcu 'e1': Starting serial connect
08:20:18.547: mcu 'e1': got {'count': 335, 'sum': 419304, 'sumsq': 3553226, 'err_len': 1, 'err_dest': 1, 'err_sync': 0, 'err_crc': 0, ...}
08:20:18.654: Loaded MCU 'e1' 138 commands
08:20:20.091: Sending MCU 'e1' printer configuration...
08:20:20.097: Configured MCU 'e1' (1024 moves)
...
Printer state: READY
```

---

## 4. Root Cause Analysis

1. **SnapSwap Pogo Pin Contact Resistance / Vibration:**
   Toolhead 1 mates with the carriage via spring-loaded pogo pins. At ~938 print hours, micro-debris (condensed filament aerosol from ASA/ABS prints at 110 °C bed / 58 °C chamber) can accumulate on the pads. Under extreme jerk and acceleration (`10,000 mm/s²`), micro-vibration can momentarily open the circuit.
2. **Toolhead Cable Flex / Strain:**
   The dynamic cable harness flexing during rapid travel moves (`F9000` / 150 mm/s) at high coordinates ($X\approx 178, Y\approx 208$) can trigger intermittent packet loss if the USB/CAN connector housing is loose or under tension.
3. **Chamber Thermal Load:**
   The chamber temperature sustained at 58.0 °C increases contact resistance and thermal expansion on the PCB docking interface.

---

## 5. Corrective & Preventive Action

* [x] **Clean Pogo Pins & Pads:** Clean the spring-loaded pins on the dock and the contact pads on Toolhead 1 with Isopropanol (IPA) and a lint-free swab.
* [x] **Verify Pin Spring Action:** Inspect all pins to ensure none are sticking or recessed.
* [x] **Check Cable Housing Screws:** Verify the rear toolhead connector is seated firmly without over-tightening (screws into soft plastic).
* [x] **1,000-Hour Preventive Maintenance:** Perform full cleaning of linear rails, re-greasing with synthetic grease (`SPREAD_GREASE`), CoreXY belt tension check, and extruder gear de-dusting.
