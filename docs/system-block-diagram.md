# System Block Diagram

Text description of the system architecture. The diagram has four levels from top to bottom: power input, the ESP32-S3 processing core, peripheral connections, and the bottom-row output devices.

---

## Level 1 — Power input

```
USB-C Charger (18 W or higher, USB-PD capable)
         │
         ▼
   CH224K USB-PD Negotiator
   CFG1=LOW, CFG2=LOW, CFG3=HIGH
   → Negotiates 12 V output from charger
         │
         ├──► 12 V ──► TMC2209 Motor Driver (motor coils)
         │
         └──► 12 V ──► NPN3613-33 Buck Converter
                              │
                              └──► 3.3 V ──► ESP32-S3 + logic
```

The CH224K doesn't convert voltage. It tells the charger what voltage to output. The charger provides 12 V directly. A separate buck converter steps 12 V down to 3.3 V for the ESP32-S3 and all logic-level components.

---

## Level 2 — ESP32-S3-MINI-1 (central block)

The largest block in the diagram. Contains the four firmware layers represented as horizontal strips inside the chip outline.

```
┌─────────────────────────────────────────────────────────┐
│                  ESP32-S3-MINI-1                         │
│       Dual-core LX7 @ 240 MHz  |  8 MB Flash            │
│       BLE 5.0  |  Wi-Fi 802.11 b/g/n                    │
│                                                          │
│  ┌─────────────────────────────────────────────────────┐ │
│  │ App Layer  (Boot · WDT · OTA · NVS)                 │ │
│  ├─────────────────────────────────────────────────────┤ │
│  │ Comm Layer  (Mode 1: Wi-Fi+MQTT | Mode 2: BLE+MQTT) │ │
│  ├─────────────────────────────────────────────────────┤ │
│  │ Logic Layer  (Curtain FSM · Position · Stall)       │ │
│  ├─────────────────────────────────────────────────────┤ │
│  │ Driver Layer  (GPIO · UART · ADC · I2C)             │ │
│  └─────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────┘
```

**Left side connection:** TMC2209 Motor Driver
- STEP pulses: GPIO5 (GPTimer hardware)
- DIR: GPIO6
- ENABLE: GPIO21 (active LOW)
- PDN_UART: GPIO18 (half-duplex UART1, 115200 baud)
- DIAG: GPIO16 (stall interrupt input, rising edge)
- Double arrow (↔) indicates bidirectional: ESP32 sends commands, TMC2209 sends diagnostic data

**Right side connections:** Wireless modules (both internal to ESP32-S3)
- Wi-Fi MQTT block: cloud control channel, also handles OTA
- BLE NimBLE block: local control channel, also handles Wi-Fi provisioning
- Double arrow (↔) indicates the communication layer interfaces with both

---

## Level 3 — Output devices (bottom row)

```
TMC2209          ESP32-S3 GPIO            Wireless
   │                 │                      │
   ▼            ┌────┴───────────────┐      │
Stepper      4 LEDs + 3 Buttons   Smartphone   MQTT Broker
Motor        (Manual control &    (App / BLE  (Cloud / Local)
(Curtain rod) status feedback)     / MQTT)
```

**Stepper Motor (Curtain Rod):** Connected to TMC2209 motor coils (A1, B1, A2, B2). Physically turns the curtain rod.

**4 LEDs + 3 Buttons:** PCB-level manual interface. Buttons (GPIO35/36/37) post events to the same FSM queue as BLE and MQTT. LEDs (GPIO10/12/18/19) reflect current curtain and network state.

**Smartphone:** Connects via BLE in Mode 2, or via MQTT broker in Mode 1. Sends the same command strings either way.

**MQTT Broker:** Can be local (Mosquitto on Raspberry Pi) or cloud (HiveMQ). The device's topic structure is the same regardless.

---

## Level 4 — Supporting storage (bottom small blocks)

```
NVS Storage          OTA Dual Partition
(Creds / Config)     (ota_0 + ota_1)
```

**NVS Storage:** 16 KB flash partition storing Wi-Fi credentials, MQTT broker URI, motor configuration, and last curtain position. Read at boot, updated on every state change.

**OTA Dual Partition:** Two 3 MB app partitions. Active firmware runs from one; new firmware downloads to the other. Bootloader switches between them. If new firmware doesn't call `mark_app_valid` within 30 s, bootloader reverts automatically.

---

## Complete data flow — one command

Starting from a phone tap on "OPEN" in BLE mode:

```
Phone (BLE WRITE) → char 0x1235 "OPEN"
  → NimBLE callback (Layer 2)
  → post CURTAIN_CMD_OPEN to FSM queue
  → curtain FSM (Layer 1): validate state → post MOTOR_START
  → tmc2209_driver (Layer 0): GPIO21 LOW, GPIO6 HIGH, start GPTimer
  → Stepper motor turns → curtain opens
  → GPIO16 DIAG HIGH (motor stalls at end)
  → Layer 0 ISR posts MOTOR_EVT_STALL to FSM queue
  → FSM: stop motor, position = 100%, state = IDLE
  → Layer 2: BLE NOTIFY on 0x1236 → phone shows "Fully Open"
  → led_indicator: LED3 GREEN
```

Total time from phone tap to motor start: ~47 ms measured.
