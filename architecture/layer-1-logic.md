# Layer 1 — Logic (Curtain Controller)

The curtain state machine. This layer decides what the curtain should do based on commands and sensor events. It knows nothing about GPIO pins, BLE characteristics, or MQTT topics. It receives typed events from a queue and posts events back down to the driver queue or up to the communication layer.

---

## States

```
                    ┌─────────────────────────────────┐
                    │                                  │
            CMD_OPEN│                                  │CMD_CLOSE
                    ▼                                  ▼
   ┌──────┐    ┌─────────┐   CMD_STOP    ┌─────────┐    ┌──────┐
   │ IDLE │───►│ OPENING │─────────────►│ STOPPED │◄───│CLOSNG│
   │      │◄───│         │              │         │    │      │
   └──────┘    └────┬────┘              └─────────┘    └──┬───┘
                    │                                      │
               STALL│                                 STALL│
               or   │                                 or   │
               LIMIT│                                 LIMIT│
                    ▼                                      ▼
              ┌──────────┐                          ┌──────────┐
              │  STALLED  │                         │  STALLED  │
              │ pos=100%  │                         │ pos=0%   │
              └──────────┘                          └──────────┘
                    │                                      │
                    └──────────────┬───────────────────────┘
                                   ▼
                                ┌──────┐
                                │ IDLE │
                                └──────┘
```

Five states:
- **IDLE** — motor stopped, position known
- **OPENING** — motor running toward fully open
- **CLOSING** — motor running toward fully closed
- **STOPPED** — motor halted mid-travel by explicit STOP command
- **STALLED** — motor stalled (end-stop reached or obstruction detected)

STALLED is a transient state. Once the FSM records position (100% for open stall, 0% for close stall), it transitions back to IDLE. If a stall happens mid-travel — not at an end-stop — it records the current step count as the last known position and transitions to IDLE.

---

## Position tracking

Position is tracked in two units: steps and percent.

Step tracking: the driver reports step count via `MOTOR_EVT_POSITION_UPDATE` events posted to the FSM queue during motor movement. The FSM accumulates these into a running total.

Percent: `position_pct = (current_steps * 100) / total_travel_steps`. `total_travel_steps` is set during calibration.

On power-on, the FSM reads last known position from NVS namespace `curtain`. If no calibration data exists, the device enters a calibration-required state and moves to a known reference point on first command.

---

## Calibration

On first run (no NVS calibration data), the FSM runs a calibration sequence:

1. Move CLOSE direction at 40% speed until stall detected (pos = 0%, step_count = 0)
2. Move OPEN direction at 40% speed until stall detected — record total step count
3. Store `open_steps` and `close_steps` in NVS namespace `curtain`
4. Transition to IDLE with position = 100%

SET_POS commands use the calibrated step counts to calculate the number of steps to move.

---

## Command processing

Commands arrive from the queue and are validated against current state before execution:

| Command | Valid from states | Action |
|---------|------------------|--------|
| OPEN | IDLE, STOPPED, STALLED | Start motor OPEN direction |
| CLOSE | IDLE, STOPPED, STALLED | Start motor CLOSE direction |
| STOP | OPENING, CLOSING | Stop motor, record position |
| SET_POS:N | IDLE, STOPPED | Calculate delta steps, move in correct direction |
| CALIBRATE | IDLE | Run calibration sequence |

Commands received in invalid states are dropped with a log warning. This prevents, for example, a CLOSE command being processed while the curtain is already closing.

---

## NVS persistence

After every state change that settles (IDLE or STOPPED), the FSM writes current position to NVS. This means position survives power cycles. The write is non-blocking — it uses a deferred write queue to avoid holding the FSM task on flash operations.

NVS namespace: `curtain`  
Keys: `position` (int, 0–100), `open_steps` (uint32), `close_steps` (uint32)
