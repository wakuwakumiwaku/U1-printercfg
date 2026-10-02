# Incident Report: `Internal error on command:"G1"` / `Lost communication with MCU 'e1'` (Toolhead 2)

* **Date / Time:** 2026-10-02 20:00:06 UTC (22:00:06 Local CEST)
* **Printer:** Snapmaker U1 (Serial: `8110026010700126YWSV`, Device: `U1`)
* **Firmware:** Klipper `1.3.0` / Software `1.3.0` / Linux Buildroot `6.1`
* **Target Hardware:** Toolhead 2 / Extruder 1 MCU (`e1` - STM32F105, USB: `/dev/serial/by-path/platform-xhci-hcd.0.auto-usb-0:1.4:1.0`)
* **Active Print File:** `u1-motherboard-fan-chance1_PLA_5h39m.gcode` (Line 727,472 / File Pos: 19,911,002 bytes)
* **Status:** Open / Intermittent Hardware Fault (USB-C Umbilical Cable vs. Toolhead 2 Socket)

---

## 1. Incident Summary

During the 12th hour of a high-temperature print job using Toolhead 2 (`extruder1`), the printer console threw a generic motion abort error:

```text
!! Internal error on command:"G1"
22:00:54 
echo: b'{"machine_type": "Snapmaker U1", "nozzle_diameter": [0.4, 0.4, 0.4, 0.4], "serial_number": "8110026010700126YWSV", "device_name": "U1", "firmware_version": "1.3.0", "software_version": "1.3.0"}'
```

Simultaneously, the touchscreen GUI reported:
```text
System Anomaly: Toolhead 2 MCU connection interrupted. 
Power off printer, check Toolhead 2 cable, then restart to resume operation.
Code: 0003-0522-0003-0008 (ID: 522)
```

Detailed log correlation across `klippy.log`, `gui.log`, and `moonraker.log` proved that **`Internal error on command:"G1"` was merely a secondary symptom**. The primary root cause was an immediate hardware bus disconnect of **Toolhead 2 (`MCU 'e1'`)**.

---

## 2. Telemetry at Disconnect

At the exact moment of failure (20:00:06 UTC / 22:00:06 CEST):
* **Extruder 1 (Toolhead 2) Temp:** 269.5 °C (Target: 270.0 °C, PWM: 0.366)
* **Heated Bed Temp:** 109.8 °C (Target: 110.0 °C)
* **Chamber / Cavity Temp:** 56.8 °C
* **Active G-Code Move:** `G1 X80.283 Y183.789 E.02899`
* **Head Coordinates:** $X = 80.40\text{ mm},\; Y = 183.79\text{ mm},\; Z = 27.64\text{ mm}$
* **Print Duration Elapsed:** 44,410 seconds (~12.33 hours)

---

## 3. Root Cause & Failure Mechanism

```
[Hardware Disconnect on Toolhead 2 USB-C]
                 │
                 ▼
[Timeout with MCU 'e1' (ID: 522, Code: 0003-0522-0003-0008)]
                 │
                 ▼
[Klipper Triggers Emergency Shutdown on Main MCU]
                 │
                 ▼
[Lookahead / Step-Compression Aborts on In-Flight Move]
(b'stepcompress o=15 i=0 c=8 a=0: Invalid sequence')
                 │
                 ▼
[mcu.error: Internal error in MCU 'mcu' stepcompress]
                 │
                 ▼
[Console displays: !! Internal error on command:"G1"]
```

### Why did `G1` fail?
When communication to `e1` dropped out, Klipper commanded an immediate emergency shutdown across all connected microcontrollers (`shutdown clock=4198693565 static_string_id=Command request`). 

Because a move (`G1 X80.283 Y183.789 E.02899`) was currently being compressed by Klipper's C-accelerated motion planner (`stepcompress.c`), the unexpected shutdown caused the step sequence buffer to break (`Invalid sequence`). Klipper caught this as a generic `mcu.error: Internal error in MCU 'mcu' stepcompress` and raised `Internal error on command:"G1"`.

---

## 4. Raw Log Excerpts

### A. Primary Disconnect Event (`klippy.log` lines 3327–3984):
```text
20:00:06.188: Timeout with MCU 'e1' (eventtime=50438.422780)
20:00:06.189: Transition to shutdown state: {"coded": "0003-0522-0003-0008", "oneshot": 0, "msg":"Lost communication with MCU 'e1'"}
...
20:00:06.246: Raising exception: id:522 index:3 code:8 oneshot:0 level:3 is_persistent:0, message: Lost communication with MCU 'e1'
```

### B. Secondary Stepcompress Crash on G1 (`klippy.log` lines 4406–4458):
```text
20:00:06.311: b'stepcompress o=15 i=0 c=8 a=0: Invalid sequence'
20:00:06.311: Internal error on command:"G1"
Traceback (most recent call last):
  File "/home/lava/klipper/klippy/gcode.py", line 229, in _process_commands
    handler(gcmd)
  File "/home/lava/klipper/klippy/extras/gcode_move.py", line 159, in cmd_G1
    self.move_with_transform(self.last_position, self.speed)
  File "/home/lava/klipper/klippy/extras/bed_mesh.py", line 243, in move
    self.toolhead.move(split_move, speed)
  File "/home/lava/klipper/klippy/toolhead.py", line 537, in move
    self.lookahead.add_move(move)
  ...
  File "/home/lava/klipper/klippy/mcu.py", line 1037, in flush_moves
    raise error("Internal error in MCU '%s' stepcompress"
mcu.error: Internal error in MCU 'mcu' stepcompress
20:00:06.336: Exiting SD card print, lines=727472, current_line_gcode=G1 X80.283 Y183.789 E.02899
```

### C. Snapmaker Touchscreen UI Record (`gui.log`):
```text
2026-10-02 20:00:06.305 <Controller Exception> level: 3 id: 522 index: 3 code: 8 message: Lost communication with MCU 'e1'
2026-10-02 20:00:06.307 <Controller Exception> show_exception_dialog yml_key: 522-3-0008, exception_text: System Anomaly_Toolhead 2 MCU connection interrupted. Power off printer, check Toolhead 2 cable, then restart to resume operation.
2026-10-02 20:00:06.309 <Controller Exception> qr_website: https://wiki.snapmaker.com/en/snapmaker_u1/troubleshooting/u1_toolhead_disconnected
2026-10-02 20:00:06.459 <Notifications> notify_gcode_response: !! Internal error on command:"G1"
```

---

## 5. Hardware Diagnosis: Cable vs. PCB

Telemetry after reboot confirms:
* **Toolhead 2 board is NOT permanently fried.** Upon reboot, MCU `e1` immediately reconnected via `/dev/serial/by-path/platform-xhci-hcd.0.auto-usb-0:1.4:1.0`, loaded all 138 commands, and heated back to 270.0 °C normally.
* Because the failure occurred after **12.3 hours** of printing at **57 °C chamber temp** and specifically during a move at $X=80, Y=183$, the issue is intermittent:
  1. **USB-C Umbilical Cable Fatigue (~85% probability):** Micro-fracture in the $D+/D-$ high-speed differential signal pair inside the dynamic USB-C cable near the connector strain-relief boot.
  2. **USB-C Receptacle / Solder Joint on Toolhead 2 PCB (~15% probability):** Fretting fatigue or thermal expansion of the spring contacts in the PCB USB-C female port under prolonged 57 °C enclosure heat.

---

## 6. Diagnostic Test: USB-C Cross-Swap

To definitively isolate the cable from the board before submitting an RMA claim to Snapmaker Support:

1. Power off the printer completely.
2. Swap the USB-C cable of **Toolhead 2** with the USB-C cable of **Toolhead 1** (or Toolhead 3).
3. Run a test print or dynamic head exercise.
4. **Conclusion:**
   * **If the error moves to Toolhead 1 (`Lost communication with MCU 'e0'`):**  
     $\rightarrow$ The **USB-C cable is defective** (internal conductor break).
   * **If the error remains on Toolhead 2 (`Lost communication with MCU 'e1'`):**  
     $\rightarrow$ The **Toolhead 2 PCB / USB-C socket is defective** (cold solder joint or loose contact pins).

---

## 7. Support & RMA Action

* **Snapmaker Wiki Reference:** [U1 Toolhead Disconnected Troubleshooting](https://wiki.snapmaker.com/en/snapmaker_u1/troubleshooting/u1_toolhead_disconnected)
* **Error Code:** `0003-0522-0003-0008`
* **Printer Serial:** `8110026010700126YWSV`
* **Action:** Contact Snapmaker Support with the results of the cross-swap test and provide attached `klippy.log` and `gui.log` for warranty replacement of the cable or toolhead board.
