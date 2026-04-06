# CosmicX Smart Curtain Controller

Firmware architecture for a production IoT curtain controller I built during my final-year engineering internship at [CosmicX](https://www.thecosmicx.com), a smart security and automation company based in Anand, Gujarat.

The system runs on an ESP32-S3, uses a TMC2209 stepper driver for silent motor control, and supports two wireless modes: full cloud control over Wi-Fi + MQTT, and local BLE control via NimBLE — without running both as simultaneous control interfaces.

> **Note:** This repository documents the firmware architecture, design decisions, and system behaviour. Source code is proprietary to CosmicX and is not included.

---

## What the system does

A curtain motor controller that you can command from a phone, from anywhere. Open, close, stop, set to a specific position. The device also detects end-stops without physical limit switches, updates its own firmware over Wi-Fi, and provisions its Wi-Fi credentials wirelessly over Bluetooth — no USB cable, no serial terminal.

Numbers from bench testing:
- **47 ms** mean BLE command response (50 trials, 1 m distance)
- **112 ms** mean MQTT round-trip on a local Mosquitto broker
- **34 dB(A)** motor noise with StealthChop2 active (vs 61 dB(A) on SpreadCycle)
- **98%** StallGuard2 detection accuracy at speeds above 40% rated
- **3.2 s** mean Wi-Fi reconnection after deliberate link drop (20 trials)
- **12.03 V ± 0.05 V** USB-PD output under full motor load

---

## Hardware platform

| Component | Part | Role |
|-----------|------|------|
| MCU | ESP32-S3-MINI-1 (N8) | Dual-core LX7, BLE 5.0, Wi-Fi, 8 MB flash |
| Motor driver | TMC2209 | Half-duplex UART, StealthChop2, StallGuard2 |
| Power management | CH224K | USB-PD negotiator — requests 12 V from charger |
| Buck converter | NPN3613-33 | 12 V → 3.3 V for logic rails |
| PCB | PD_Stepper_V1 Rev 1.1 | Custom KiCad board, ~80 × 65 mm |

**GPIO map (PD_Stepper_V1 Rev 1.1)**

| Signal | GPIO | Notes |
|--------|------|-------|
| TMC2209 STEP | GPIO5 | GPTimer hardware pulses |
| TMC2209 DIR | GPIO6 | Direction control |
| TMC2209 ENABLE | GPIO21 | Active LOW |
| TMC2209 PDN_UART | GPIO18 | Half-duplex UART1, 115200 baud |
| TMC2209 DIAG | GPIO16 | Stall interrupt, rising edge |
| Button OPEN | GPIO35 | Active LOW, debounced |
| Button STOP | GPIO36 | Active LOW, debounced |
| Button CLOSE | GPIO37 | Active LOW, debounced |
| LED 1 (error) | GPIO10 | Red |
| LED 2 (network) | GPIO12 | Red/Green |
| LED 3 (open) | GPIO18 | Green |
| LED 4 (closed) | GPIO19 | Green |
| NTC thermistor | GPIO7 | ADC, Steinhart-Hart conversion |
| VBUS sense | GPIO4 | ADC, 12 V rail monitoring |
| AUX1 limit | GPIO14 | Hardware end-stop backup |
| AUX2 limit | GPIO13 | Hardware end-stop backup |
| USB-PD CFG1 | GPIO38 | LOW → 12 V negotiation |
| USB-PD CFG2 | GPIO48 | LOW → 12 V negotiation |
| USB-PD CFG3 | GPIO47 | HIGH → 12 V negotiation |

---

## Firmware architecture

Four layers. Each one knows nothing about the others. They communicate through FreeRTOS queues.

```
┌─────────────────────────────────────────────────┐
│  Layer 3 — Application                          │
│  app_main: boot sequence, watchdog, OTA, NVS    │
├─────────────────────────────────────────────────┤
│  Layer 2 — Communication                        │
│  BLE NimBLE GATT server  |  ESP-MQTT client     │
│  (Mode 1: Wi-Fi+MQTT)    |  (Mode 2: BLE+MQTT)  │
├─────────────────────────────────────────────────┤
│  Layer 1 — Logic                                │
│  Curtain FSM: IDLE / OPENING / CLOSING /        │
│               STOPPED / STALLED                 │
│  Position tracking, stall handling              │
├─────────────────────────────────────────────────┤
│  Layer 0 — Driver                               │
│  TMC2209 UART, GPTimer step pulses, ADC,        │
│  button debounce, LED patterns, limit sensors   │
└─────────────────────────────────────────────────┘
```

Detailed write-ups for each layer are in [`architecture/`](./architecture/).

---

## Boot sequence

Order is strict. The USB-PD CFG pins must be set before anything else draws current — if the board initialises NVS first, the 50–100 ms delay can cause the motor coils to see 5 V instead of 12 V, which causes stall false-positives on first move.

```
[0]  usbpd_12v_init()          — CFG1=LOW, CFG2=LOW, CFG3=HIGH → request 12 V
[1]  nvs_flash_init()           — read stored config, Wi-Fi creds, curtain position
[2]  led_indicator_init()       — visual feedback during boot
[3]  esp_task_wdt_reconfigure() — 30 s watchdog, trigger_panic=true
[4]  limit_sensor_init()        — AUX1/AUX2 hardware limit switches
[5]  button_handler_init()      — OPEN / STOP / CLOSE buttons
[6]  tmc2209_init()             — UART config, StealthChop2 ON, SGTHRS=50
[7]  curtain_init()             — FSM start, restore position from NVS
[8]  ble_app_init()             — NimBLE stack, GATT server, start advertising
[9]  wifi_start()               — only if NVS has credentials; MQTT connects after IP
```

---

## Wireless modes

### Mode 1 — Wi-Fi + MQTT

The device joins the configured 802.11 b/g/n access point. All curtain commands arrive via MQTT. OTA updates also run in this mode. Wi-Fi reconnects automatically using exponential backoff (up to 10 retries).

### Mode 2 — BLE + MQTT

BLE is the primary control interface. Wi-Fi stays active for cloud telemetry only — it does not accept curtain commands. BLE and Wi-Fi never act as simultaneous control radios; they share the 2.4 GHz antenna and time-division is handled by the ESP-IDF coexistence layer.

**Why not both at the same time?** RF coexistence on a single antenna adds latency and makes timing unpredictable. For a curtain — where 47 ms feels instant — the trade-off isn't worth it.

---

## BLE — GATT service

**Service UUID:** `0x1234`  
**Advertising name:** `CosmicX-Curtain`

| Characteristic | UUID | Permissions | Payload |
|----------------|------|-------------|---------|
| CURTAIN_COMMAND | `0x1235` | Write | `"OPEN"` `"CLOSE"` `"STOP"` `"SET_POS:50"` `"CALIBRATE"` |
| CURTAIN_STATE | `0x1236` | Read + Notify | `{"state":"OPENING","position":45,"source":"ble"}` |
| CURTAIN_CONFIG | `0x1237` | Read + Write | `{"speed":2000,"current":800}` |
| WIFI_CONFIG | `0x1238` | Write only | `{"ssid":"...","password":"...","broker":"mqtt://192.168.1.100:1883"}` |

`0x1238` is the provisioning channel. The phone writes Wi-Fi + broker credentials here over BLE. The device stores them in NVS and connects to Wi-Fi on the next boot cycle in Mode 1.

---

## MQTT topics

**Base:** `cosmicx/devices/{device_id}/`

| Topic | Direction | QoS | Retained | Purpose |
|-------|-----------|-----|----------|---------|
| `.../state` | Device → Broker | 1 | Yes | Current position and curtain state |
| `.../command` | Broker → Device | 1 | No | Control commands |
| `.../telemetry` | Device → Broker | 0 | No | Motor temp, VBUS voltage, uptime (30 s interval) |
| `.../status` | Device → Broker | 1 | Yes | LWT: `"offline"` on unexpected disconnect |

The LWT is configured at connection time. When the device drops off the network, the broker publishes `"offline"` to `.../status` within the keep-alive window. The device publishes `"online"` when it reconnects.

---

## NVS namespaces

| Namespace | Keys | Purpose |
|-----------|------|---------|
| `wifi_creds` | `ssid`, `password`, `broker_uri` | Written via BLE char 0x1238 |
| `curtain` | `position`, `open_steps`, `close_steps` | Last known position, calibration |
| `motor` | `speed`, `current_ma`, `sgthrs` | Motor tuning parameters |

---

## TMC2209 — StallGuard2 explained

StallGuard2 monitors back-EMF. When the curtain hits a wall and the motor stalls, back-EMF drops. The TMC2209 compares this against the SGTHRS register (set to 50 in this design). If the load exceeds the threshold, it raises the DIAG pin. The ESP32-S3 has an interrupt on GPIO16 that posts a `STALL` event to the curtain FSM queue.

This replaces physical limit switches at normal operating speeds. The 2% failure rate we measured happens below 5% rated speed — back-EMF at that speed is too small to distinguish from noise. AUX1 (GPIO14) and AUX2 (GPIO13) are wired as hardware backup for low-speed calibration.

---

## OTA firmware updates

The 8 MB flash is partitioned into two 3 MB app slots (`ota_0`, `ota_1`). OTA runs in Mode 1 only — it needs a stable Wi-Fi link.

```
Receive MQTT command with firmware URL
  → esp_https_ota downloads to inactive partition
  → SHA-256 verification
  → otadata updated to point to new partition
  → device reboots
  → new firmware must call esp_ota_mark_app_valid_cancel_rollback() within 30 s
  → if watchdog fires first → bootloader reverts to previous partition
```

A bad firmware update cannot permanently brick the device. The rollback window is hard-coded to 30 seconds — same as the watchdog timeout.

---

## Power rail

```
USB-C charger
    │
    │  default 5 V
    ▼
CH224K USB-PD negotiator
    │  CFG1=LOW, CFG2=LOW, CFG3=HIGH → negotiates 12 V
    │
    ├──► 12 V ──► TMC2209 stepper driver
    │              (motor coils, ~0.85 A peak)
    │
    └──► 12 V ──► NPN3613-33 buck converter
                    │
                    └──► 3.3 V ──► ESP32-S3 + logic
```

The CH224K does not convert voltage — it asks the charger for it. A non-PD charger defaults to 5 V, which starves the motor and causes stall false-positives. The device requires an 18 W or higher USB-PD charger.

---

## Project structure

```
smart-curtain-controller-architecture/
├── README.md                    ← you are here
├── LICENSE                      ← MIT
├── CONTRIBUTING.md
│
├── architecture/
│   ├── overview.md              ← system architecture overview
│   ├── layer-0-driver.md        ← TMC2209, GPTimer, ADC, buttons, LEDs
│   ├── layer-1-logic.md         ← curtain FSM, position tracking
│   ├── layer-2-communication.md ← BLE NimBLE, ESP-MQTT, Wi-Fi
│   └── layer-3-application.md  ← boot sequence, watchdog, OTA, NVS
│
└── docs/
    ├── system-block-diagram.md  ← text description of the block diagram
    ├── ble-protocol.md          ← GATT service spec
    ├── mqtt-protocol.md         ← topic schema and payloads
    ├── ota-process.md           ← OTA flow and rollback logic
    └── power-architecture.md    ← USB-PD, buck converter, rail map
```

---

## My role

I built this firmware during a 6-month final year engineering internship at CosmicX (2025–26). The hardware was designed by CosmicX; my work was the firmware.

Specifically:

- Wrote the TMC2209 UART driver from scratch — no external library. Raw register writes using the TMC2209 datagram format (SYNC byte, slave address, register, 4-byte data, CRC-8).
- Implemented the GPTimer-based step pulse generator with trapezoidal acceleration ramping inside the ISR callback.
- Built the curtain state machine with StallGuard2-based end-stop detection via GPIO16 interrupt.
- Wrote the NimBLE GATT server with all four characteristics including the BLE-to-Wi-Fi provisioning flow.
- Integrated ESP-MQTT with LWT, retained topics, and QoS 1 publish/subscribe.
- Implemented the dual-partition OTA update flow with SHA-256 verification and automatic rollback.
- Designed the four-layer firmware architecture and enforced strict layer separation through FreeRTOS queue interfaces.

---

## Tech stack

- **MCU:** ESP32-S3-MINI-1 (Xtensa LX7, 240 MHz, dual-core)
- **Framework:** ESP-IDF 5.5
- **RTOS:** FreeRTOS (event-driven queues, no polling)
- **BLE:** NimBLE stack (ESP-IDF component)
- **MQTT:** ESP-MQTT component (Protocol 3.1.1)
- **Motor:** TMC2209 (UART mode, 115200 baud, CRC-8)
- **Storage:** NVS flash (key-value, wear-levelled)
- **OTA:** esp_https_ota with dual-partition rollback
- **Build:** CMake + idf.py
- **Language:** C (C11)

---

## Academic context

**Student:** Mohammad Sahil (Enrollment: 21EL002)  
**Degree:** B.E. Electronics Engineering  
**Institution:** Birla Vishvakarma Mahavidyalaya (BVM), V.V. Nagar, Gujarat  
**University:** Gujarat Technological University (GTU)  
**Academic Year:** 2025–26  
**Industry Partner:** CosmicX, Anand, Gujarat  
**Faculty Guide:** Dr. Deepak Vala  
**Co-Guide:** Dr. J. M. Rathod  
**Industry Guide:** Ishwar Sharma (MD), CosmicX

---

## License

MIT — see [LICENSE](./LICENSE).

The firmware source code is proprietary to CosmicX and is not part of this repository.
