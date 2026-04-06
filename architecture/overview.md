# Architecture Overview

The firmware is split into four layers. The rule is simple: no layer calls into a non-adjacent layer. All inter-layer communication goes through FreeRTOS queues. This means a crash or bug in the BLE layer cannot directly affect the motor driver, and swapping the MQTT library doesn't require touching motor code.

---

## Why four layers

Embedded firmware tends to collapse into a single monolithic loop when nobody enforces separation. The symptoms are familiar: interrupt handlers that call MQTT publish, Wi-Fi event callbacks that directly set GPIO pins, motor control interleaved with JSON parsing.

This architecture uses a hospital analogy. Doctors don't mop floors. Cleaners don't write prescriptions. Each layer has one job.

---

## Layer overview

```
┌───────────────────────────────────────────────────────────────┐
│  Layer 3 — Application                                        │
│                                                               │
│  app_main.c                                                   │
│  Orchestrates the boot sequence in strict order.              │
│  Owns the 30 s watchdog. Triggers OTA. No business logic.     │
├───────────────────────────────────────────────────────────────┤
│  Layer 2 — Communication                                      │
│                                                               │
│  ble_app/     — NimBLE GATT server, 4 characteristics         │
│  mqtt_app/    — ESP-MQTT client, pub/sub, LWT                 │
│                                                               │
│  Translates network events into curtain commands.             │
│  Translates curtain state into network messages.              │
│  Knows nothing about GPIO or motor steps.                     │
├───────────────────────────────────────────────────────────────┤
│  Layer 1 — Logic                                              │
│                                                               │
│  curtain_controller/                                          │
│  Five-state FSM: IDLE / OPENING / CLOSING / STOPPED / STALLED │
│  Tracks position in steps and percent. Decides when to stop.  │
│  Knows nothing about BLE, MQTT, or GPIO registers.            │
├───────────────────────────────────────────────────────────────┤
│  Layer 0 — Driver                                             │
│                                                               │
│  tmc2209_driver/  — UART register writes, step pulse gen      │
│  button_handler/  — debounce, long-press detection            │
│  led_indicator/   — pattern-based LED state machine           │
│  limit_sensor/    — AUX1/AUX2 hardware end-stop detection     │
│                                                               │
│  The only layer that touches hardware registers directly.     │
└───────────────────────────────────────────────────────────────┘
```

---

## Data flow — one command, full path

Here is what happens when a phone sends `"OPEN"` via BLE:

1. NimBLE stack receives GATT WRITE on characteristic `0x1235`
2. `ble_app` callback posts `CURTAIN_CMD_OPEN` event to the curtain FSM queue
3. `curtain_controller` FSM task wakes, validates current state, posts `MOTOR_START` to the driver queue
4. `tmc2209_driver` sets GPIO21 LOW (enable), GPIO6 HIGH (direction), starts GPTimer alarm for STEP pulses on GPIO5
5. Curtain moves. Each timer ISR increments the step counter
6. When GPIO16 (DIAG) goes HIGH — stall detected — driver posts `STALL` event to FSM queue
7. FSM stops motor, records position = 100%, transitions to IDLE
8. `ble_app` reads new state and sends BLE NOTIFY on `0x1236`
9. `led_indicator` sets LED3 GREEN

Total time from step 1 to step 4 (motor start): ~47 ms measured.

---

## Inter-layer interfaces

Layers talk through typed events posted to FreeRTOS queues. No direct function calls across layer boundaries for control flow.

```c
// Layer 2 → Layer 1
typedef enum {
    CURTAIN_CMD_OPEN,
    CURTAIN_CMD_CLOSE,
    CURTAIN_CMD_STOP,
    CURTAIN_CMD_SET_POS,
    CURTAIN_CMD_CALIBRATE,
} curtain_cmd_t;

// Layer 1 → Layer 0
typedef enum {
    MOTOR_CMD_START_OPEN,
    MOTOR_CMD_START_CLOSE,
    MOTOR_CMD_STOP,
} motor_cmd_t;

// Layer 0 → Layer 1 (events up)
typedef enum {
    MOTOR_EVT_STALL,
    MOTOR_EVT_POSITION_UPDATE,
    MOTOR_EVT_LIMIT_HIT,
} motor_event_t;
```

---

## Project component map

```
main/
└── app_main.c              Layer 3

components/
├── ble_app/                Layer 2
│   ├── ble_app.c
│   └── include/ble_app.h
├── mqtt_app/               Layer 2
│   ├── mqtt_app.c
│   └── include/mqtt_app.h
├── curtain_controller/     Layer 1
│   ├── curtain_controller.c
│   └── include/curtain_controller.h
├── tmc2209_driver/         Layer 0
│   ├── tmc2209_driver.c
│   └── include/tmc2209_driver.h
├── button_handler/         Layer 0
│   ├── button_handler.c
│   └── include/button_handler.h
├── led_indicator/          Layer 0
│   ├── led_indicator.c
│   └── include/led_indicator.h
└── limit_sensor/           Layer 0
    ├── limit_sensor.c
    └── include/limit_sensor.h
```

---

See individual layer docs for implementation detail:

- [Layer 0 — Driver](./layer-0-driver.md)
- [Layer 1 — Logic](./layer-1-logic.md)
- [Layer 2 — Communication](./layer-2-communication.md)
- [Layer 3 — Application](./layer-3-application.md)
