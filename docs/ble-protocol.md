# BLE Protocol

Custom GATT service over NimBLE. Four characteristics cover curtain control, state readback, configuration, and Wi-Fi provisioning.

---

## Service

| Field | Value |
|-------|-------|
| Stack | NimBLE (ESP-IDF component) |
| Service UUID | `0x1234` |
| Advertising name | `CosmicX-Curtain` |
| Max simultaneous connections | 1 |
| Bonding required | No |

---

## Characteristics

### 0x1235 — CURTAIN_COMMAND

**Permissions:** Write only  
**Payload:** UTF-8 string

| Command | Effect |
|---------|--------|
| `OPEN` | Move curtain to fully open position |
| `CLOSE` | Move curtain to fully closed position |
| `STOP` | Halt motor immediately, record current position |
| `SET_POS:N` | Move to N percent (0–100). Example: `SET_POS:50` |
| `CALIBRATE` | Run full calibration sequence (slow CLOSE → slow OPEN) |
| `CLEAR_JAM` | Stop + retry last direction (for obstruction recovery) |

Commands received while the motor is already moving in the same direction are ignored. Commands in invalid states (e.g., OPEN while already at 100%) are logged and dropped.

---

### 0x1236 — CURTAIN_STATE

**Permissions:** Read + Notify  
**Payload:** JSON

```json
{
  "state": "OPENING",
  "position": 45,
  "source": "ble"
}
```

| Field | Type | Values |
|-------|------|--------|
| `state` | string | `"IDLE"`, `"OPENING"`, `"CLOSING"`, `"STOPPED"`, `"STALLED"` |
| `position` | int | 0–100 (percent open) |
| `source` | string | `"ble"` or `"mqtt"` (last command source) |

Connected clients receive a NOTIFY event on every state change. There is no need to poll.

---

### 0x1237 — CURTAIN_CONFIG

**Permissions:** Read + Write  
**Payload:** JSON

```json
{
  "speed": 2000,
  "current_ma": 800
}
```

| Field | Unit | Range | Default |
|-------|------|-------|---------|
| `speed` | steps/sec | 100–4000 | 2000 |
| `current_ma` | mA RMS | 200–1200 | 800 |

Values outside the safe range are rejected with a GATT `ATT_ERR_VALUE_NOT_ALLOWED` error. Accepted values are written to NVS namespace `motor` and applied immediately to the running TMC2209 driver.

---

### 0x1238 — WIFI_CONFIG

**Permissions:** Write only  
**Payload:** JSON

```json
{
  "ssid": "HomeNetwork",
  "password": "mypassword",
  "broker": "mqtt://192.168.1.100:1883"
}
```

This is the provisioning characteristic. On a fresh device with no Wi-Fi credentials, the phone connects via BLE and writes this payload to `0x1238`. The device:

1. Parses the JSON
2. Writes `ssid`, `password`, and `broker_uri` to NVS namespace `wifi_creds`
3. Triggers a Wi-Fi connect attempt with the new credentials
4. On successful IP assignment, transitions to Mode 1 operation

The characteristic is write-only by design. The stored password cannot be read back over BLE. To change credentials, write new ones to `0x1238` — they overwrite the stored values.

---

## Advertising behaviour

| Condition | Advertising state |
|-----------|------------------|
| No BLE client connected | Advertising |
| BLE client connected | Not advertising |
| Client disconnects | Advertising restarts automatically |
| Mode 1 active (Wi-Fi control) | Advertising suppressed |

---

## Connection sequence

```
Phone scans for "CosmicX-Curtain"
  → CONNECT
  → Discover service 0x1234
  → Subscribe to NOTIFY on 0x1236 (state updates)
  → Write "OPEN" to 0x1235
  → Receive NOTIFY: {"state":"OPENING","position":0,"source":"ble"}
  → Receive NOTIFY: {"state":"IDLE","position":100,"source":"ble"}
  → Done
```
