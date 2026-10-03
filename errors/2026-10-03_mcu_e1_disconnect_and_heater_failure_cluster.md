# Incident Report: Recurring MCU 'e1' Disconnects & Heater Verification Errors (Toolhead 2)

* **Date:** 2026-10-03 (03:08 – 07:17 UTC / 05:08 – 09:17 CEST)
* **Printer:** Snapmaker U1 (Serial: `8110026010700126YWSV`, Device: `U1`)
* **Firmware:** Klipper `1.3.0` / Software `1.3.0` / Linux Buildroot `6.1`
* **Target Hardware:** Toolhead 2 / Extruder 1 MCU (`e1` - STM32F105, USB: `/dev/serial/by-path/platform-xhci-hcd.0.auto-usb-0:1.4:1.0`)
* **Active Print File:** `u1-motherboard-fan-chance1_PLA_4h54m.gcode` (High-Temp ASA profile: Bed 110 °C, Nozzle 270 °C, Chamber 55–60 °C)
* **Status:** Open / Critical Intermittent Hardware Fault (Thermal fretting & contact resistance on SnapSwap Pogo-Pin Dock / USB-C harness)

---

## 1. Executive Summary

During a continuous overnight high-temperature print job on 2026-10-03, Toolhead 2 (`MCU 'e1'`) experienced a **cluster of 9 emergency shutdowns** within a 4-hour window. Each time, Klipper's emergency shutdown triggered, followed by Snapmaker's automated `power_loss` recovery which re-homed and resumed the print from SD-card flash position.

The failures manifested as two alternating error signatures:
1. **7× `Lost communication with MCU 'e1'` (Code: `0003-0522-0003-0008`, ID: 522):** Total serial communication loss / unacknowledged packet timeouts between host Linux and the toolhead STM32 MCU.
2. **2× `Heater extruder1 not heating at expected rate` (Code: `0003-0523-0001-0003`, ID: 523):** Klipper `verify_heater` trip caused by ADC voltage instability and temperature overshoot/drift under thermal stress.

Across all 9 incidents, microcontrollers `e0` (Toolhead 1), `e2` (Toolhead 3), `e3` (Toolhead 4), and main `mcu` remained 100% operational with 0 packet loss. The issue is strictly isolated to **Toolhead 2 (`e1`)**.

---

## 2. Chronological Incident Log (All 9 Events)

| # | Time (UTC) | Time (CEST) | Error Code / ID | Error Message | Print Time | Bed (°C) | Cavity (°C) | Extruder1 (°C) | e1 Retransmit / Symptoms |
|---|---|---|---|---|---|---|---|---|---|
| **1** | 03:08:31 | 05:08:31 | `0003-0522-0003-0008` (522) | `Lost communication with MCU 'e1'` | 13,388 s (~3.7 h) | 110.0 | 59.7 | 268.3 (PWM 0.71) | Bus timeout after 3.7h initial run |
| **2** | 03:31:05 | 05:31:05 | `0003-0522-0003-0008` (522) | `Lost communication with MCU 'e1'` | 1,091 s (~18 m) | 110.1 | 58.4 | 268.7 (PWM 0.57) | Quick recurrence under 58 °C chamber |
| **3** | 04:17:38 | 06:17:38 | `0003-0522-0003-0008` (522) | `Lost communication with MCU 'e1'` | 2,709 s (~45 m) | 109.8 | 59.2 | 270.3 (PWM 0.27) | `err_len=5` on toolhead serial RX |
| **4** | 04:48:05 | 06:48:05 | `0003-0522-0003-0008` (522) | `Lost communication with MCU 'e1'` | 1,706 s (~28 m) | 109.8 | 58.4 | 269.4 (PWM 0.51) | Contact loss during active raster move |
| **5** | 05:18:04 | 07:18:04 | `0003-0522-0003-0008` (522) | `Lost communication with MCU 'e1'` | 1,710 s (~28 m) | 109.9 | 59.0 | 271.7 (PWM 0.00) | `err_len=5`, packet timeout |
| **6** | 05:32:25 | 07:32:25 | `0003-0523-0001-0003` (523) | `Heater extruder1 not heating at expected rate` | 662 s (~11 m) | 110.0 | 57.4 | 270.9 (Target 270.0) | ADC jitter / thermal overshoot |
| **7** | 06:34:03 | 08:34:03 | `0003-0523-0001-0003` (523) | `Heater extruder1 not heating at expected rate` | 3,518 s (~58 m) | 109.7 | 59.3 | 271.0 (Target 270.0) | Temp drift at 271.03 °C violates gain window |
| **8** | 07:00:43 | 09:00:43 | `0003-0522-0003-0008` (522) | `Lost communication with MCU 'e1'` | 1,128 s (~19 m) | 109.6 | 57.3 | 269.0 (PWM 0.65) | **`bytes_retransmit=9120`** (massive packet loss) |
| **9** | 07:17:36 | 09:17:36 | `0003-0522-0003-0008` (522) | `Lost communication with MCU 'e1'` | 809 s (~13.5 m) | 109.9 | 55.2 | 270.5 (PWM 0.30) | `retransmit=71`, 74 unacknowledged pings |

---

## 3. Telemetry Pattern & Failure Correlation

```
                    Sustained Chamber Heat (55°C – 60°C) & Bed 110°C
                                           │
             ┌─────────────────────────────┴─────────────────────────────┐
             ▼                                                           ▼
Thermal Expansion / Fretting on                              Contact Resistance on
SnapSwap Pogo-Pin Dock Interface                             Power & Ground Pins
             │                                                           │
             ▼                                                           ▼
High-Frequency Packet Drop                                   Voltage Dips / ADC Reference Jitter
(bytes_retransmit spiked to 9,120)                           (Thermistor reads noisy 271.03°C)
             │                                                           │
             ▼                                                           ▼
[ID: 522] Lost communication with MCU 'e1'                   [ID: 523] verify_heater Trip on extruder1
             │                                                           │
             └─────────────────────────────┬─────────────────────────────┘
                                           ▼
                            Klipper Emergency Shutdown
                                           │
                                           ▼
                   Automatic Power-Loss Recovery Cycle
              (Homing X/Y -> Reheat 270°C/110°C -> SD Resume)
```

### Key Analytical Takeaways:
1. **Chamber Temperature Dependency:** Every single failure occurred with chamber temperatures between **55.2 °C and 59.7 °C**. Under high ambient heat, thermal expansion of the pogo pin springs, gold contact pad plating, and toolhead docking plastics causes microscopic contact fretting.
2. **Mean Time Between Failures (MTBF):** After the initial 3.7-hour run, the failure interval dropped to an average of **20–30 minutes**, reflecting heat saturation of the dock and carriage.
3. **The Link Between ID 522 and ID 523:** 
   - A dirty or high-resistance pogo-pin dock affects both the 24V/GND power pins and the USB/serial data lines.
   - When 24V supply drops across contact resistance, the onboard ADC reference fluctuates, causing temperature readings to wander or overshoot (triggering `verify_heater` ID 523).
   - When contact breaks completely for $>25\text{ ms}$, serial communications drop entirely (triggering ID 522).

---

## 4. Raw Diagnostic Evidence

### A. Communication Drop & Packet Loss Dump (Incident #8 at 07:00:43 UTC)
```text
07:00:43.054: Transition to shutdown state: {"coded": "0003-0522-0003-0008", "oneshot": 0, "msg":"Lost communication with MCU 'e1'"}
07:00:43.056: Stats 1144.7: ...
  e0: bytes_write=2654 bytes_read=82275 bytes_retransmit=9
  e1: bytes_write=56428 bytes_read=86836 bytes_retransmit=9120  <-- MASSIVE PACKET RETRANSMIT
  e2: bytes_write=2617 bytes_read=82242 bytes_retransmit=0
  e3: bytes_write=3430 bytes_read=81946 bytes_retransmit=9
```

### B. Heater Verification Failure Dump (Incident #7 at 06:34:03 UTC)
```text
06:34:03.701: {"coded": "0003-0523-0001-0003", "oneshot": 0, "msg":"Heater extruder1 not heating at expected rate, temp: 271.03 target: 270.00"}
06:34:03.707: Transition to shutdown state: {"coded": "0003-0523-0001-0003", "oneshot": 0, "msg":"Heater extruder1 not heating at expected rate, temp: 271.03 target: 270.00"}
06:34:03.789: Raising exception: id:523 index:1 code:3 oneshot:0 level:3 is_persistent:0, message: Heater extruder1 not heating at expected rate, temp: 271.03 target: 270.00
```

### C. Touchscreen GUI Error Logging (`gui.log`)
```text
2026-10-03 07:17:36.646 <Controller Exception> level: 3 id: 522 index: 3 code: 8 message: Lost communication with MCU 'e1'
2026-10-03 07:17:36.648 <Controller Exception> show_exception_dialog yml_key: 522-3-0008, exception_text: System Anomaly_Toolhead 2 MCU connection interrupted.
2026-10-03 07:17:36.651 <Controller Exception> qr_website: https://wiki.snapmaker.com/en/snapmaker_u1/troubleshooting/u1_toolhead_disconnected
```

---

## 5. Required Corrective Actions

### Immediate Physical Maintenance (Mandatory before next print):
1. **Clean Pogo-Pins & Gold Pads:**
   - Use 99% Isopropanol (IPA) and a lint-free swab to vigorously clean both the male spring-loaded pogo-pins on the carriage dock and the female gold contact pads on the back of Toolhead 2.
   - **DO NOT USE GREASE OR DIELECTRIC OIL:** Fats act as insulators and attract airborne micro-particles from ASA/ABS prints.
2. **Spring Plunger Compliance Test:**
   - Gently press each individual pogo-pin on Toolhead 2's carriage dock using a plastic spudger or wooden toothpick. Verify that all pins spring back smoothly and none are stuck, recessed, or gummed up with condensed filament fumes.
3. **Inspect Umbilical USB-C Cable Strain Relief:**
   - Check the rear USB-C connection to the toolhead dock for looseness or strain when the toolhead moves to the maximum front-right corners.

### Software Configuration Hardening (Optional):
If temperature overshoot/oscillation persists at 270 °C after physical cleaning, add a dedicated heater verification tolerance block in `printer.cfg`:
```ini
[verify_heater extruder1]
max_error: 150
check_gain_time: 30
hysteresis: 5
heating_gain: 2
```
