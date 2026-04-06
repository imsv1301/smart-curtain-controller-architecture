# MQTT Protocol

Standard MQTT 3.1.1 over TCP. The device subscribes to a command topic and publishes to state, telemetry, and status topics. The same command strings work over both MQTT and BLE — the curtain FSM does not care which interface a command came from.

---

## Topic structure

**Base:** `cosmicx/devices/{device_id}/`

The device ID is the ESP32-S3 MAC address formatted as six lowercase hex pairs joined by colons. Example: `cosmicx/devices/aa:bb:cc:dd:ee:ff/`

| Topic | Direction | QoS | Retained | Purpose |
|-------|-----------|-----|----------|---------|
| `.../state` | Device → Broker | 1 | Yes | Current curtain state and position |
| `.../command` | Broker → Device | 1 | No | Control commands |
| `.../telemetry` | Device → Broker | 0 | No | Sensor data, diagnostics |
| `.../status` | Device → Broker | 1 | Yes | Online/offline presence (LWT) |

---

## Payloads

### `.../state`

Published on every FSM state change. Retained so new subscribers see the current state immediately.

```json
{
  "state": "IDLE",
  "position": 75,
  "source": "mqtt",
  "timestamp": 1712345678
}
```

### `.../command`

Accepted command strings (identical to BLE 0x1235):

```
OPEN
CLOSE
STOP
SET_POS:50
CALIBRATE
CLEAR_JAM
OTA:https://updates.thecosmicx.com/curtain/v2.bin
```

The `OTA:` prefix triggers a firmware update to the specified URL. The URL must be HTTPS. OTA is only processed in Mode 1.

### `.../telemetry`

Published every 30 seconds. QoS 0 — loss of an occasional telemetry message is acceptable.

```json
{
  "temp_c": 42.3,
  "vbus_v": 12.03,
  "uptime_s": 3600,
  "position": 75,
  "rssi": -58,
  "heap_free": 142336,
  "mode": 1
}
```

| Field | Description |
|-------|-------------|
| `temp_c` | TMC2209 PCB temperature from NTC thermistor |
| `vbus_v` | USB-PD rail voltage (should be ~12.03 V) |
| `uptime_s` | Seconds since last boot |
| `position` | Current curtain position (0–100%) |
| `rssi` | Wi-Fi signal strength in dBm |
| `heap_free` | Free heap memory in bytes |
| `mode` | 1 = Wi-Fi+MQTT active, 2 = BLE+MQTT |

### `.../status`

**LWT message:** `"offline"` (published by broker if device disconnects unexpectedly)  
**Online message:** `"online"` (published by device on connect and reconnect, `retain=true`)

A dashboard subscribed to `.../status` gets accurate presence without polling. Offline detection latency is bounded by the MQTT keep-alive window (60 s default).

---

## Connection parameters

| Parameter | Value |
|-----------|-------|
| Protocol | MQTT 3.1.1 |
| Keep-alive | 60 s |
| Clean session | True |
| TLS | Planned (future scope) |
| Authentication | None (local network deployment) |

---

## Reconnection behaviour

On Wi-Fi link loss:
1. `WIFI_EVENT_STA_DISCONNECTED` fires
2. Exponential backoff retry: 1 s, 2 s, 4 s, 8 s... up to 10 attempts
3. On IP assignment: MQTT client reconnects automatically
4. MQTT client re-subscribes to `.../command`
5. Device publishes `"online"` to `.../status` with `retain=true`
6. Device publishes current state to `.../state` with `retain=true`

Mean reconnection time measured: 3.2 s from link drop to IP assignment (20 trials).

---

## Example — full command flow over MQTT

```
Dashboard publishes to:  cosmicx/devices/aa:bb:cc:dd:ee:ff/command
Payload: SET_POS:75

Device receives → posts CURTAIN_CMD_SET_POS(75) to FSM queue
FSM calculates steps needed, starts motor
Device publishes to:     cosmicx/devices/aa:bb:cc:dd:ee:ff/state
Payload: {"state":"OPENING","position":45,"source":"mqtt","timestamp":...}

Motor reaches 75%
Device publishes:        cosmicx/devices/aa:bb:cc:dd:ee:ff/state
Payload: {"state":"IDLE","position":75,"source":"mqtt","timestamp":...}
```
