# Layer 2 — Communication

Two sub-modules: `ble_app` and `mqtt_app`. They translate between the wireless protocols and the curtain FSM queue. Neither sub-module knows about GPIO pins or motor steps.

---

## Operating modes

The device runs in one of two modes, selected by what's stored in NVS:

**Mode 1 — Wi-Fi + MQTT:** The device joins the configured access point. All curtain commands come from the MQTT broker. BLE advertising is stopped once Wi-Fi connects in this mode.

**Mode 2 — BLE + MQTT:** BLE is the primary control interface. Wi-Fi stays active for MQTT telemetry only — it does not subscribe to the command topic. The phone must be within BLE range to send commands.

The modes are mutually exclusive as control interfaces. The reason is the shared 2.4 GHz antenna: running BLE connection events and MQTT command processing simultaneously on one antenna adds unpredictable latency. For curtain control — where 47 ms already feels instant — that trade-off isn't worth making.

---

## ble_app — NimBLE GATT server

Stack: NimBLE (`CONFIG_BT_NIMBLE_ENABLED=y`).

**Service UUID:** `0x1234`  
**Advertising name:** `CosmicX-Curtain`  
**Max connections:** 1  
**Bonding:** not required

### Characteristics

**0x1235 — CURTAIN_COMMAND (Write)**

Accepts plaintext strings:

```
"OPEN"
"CLOSE"
"STOP"
"SET_POS:50"       ← 0–100, any integer
"CALIBRATE"
"CLEAR_JAM"        ← stop + retry last direction
```

On write, the callback parses the string and posts the corresponding `curtain_cmd_t` event to the FSM queue.

**0x1236 — CURTAIN_STATE (Read + Notify)**

JSON payload, updated on every FSM state change:

```json
{"state": "OPENING", "position": 45, "source": "ble"}
```

`source` is either `"ble"` or `"mqtt"` — indicates which interface triggered the last command. Connected BLE clients receive a notification automatically on every state change via the NOTIFY property.

**0x1237 — CURTAIN_CONFIG (Read + Write)**

Exposes tunable motor parameters:

```json
{"speed": 2000, "current_ma": 800}
```

Writes are validated against safe ranges before being passed to the motor driver. Out-of-range values are rejected with a GATT error response.

**0x1238 — WIFI_CONFIG (Write only)**

The provisioning channel. The phone writes Wi-Fi and MQTT credentials here:

```json
{
  "ssid": "HomeNetwork",
  "password": "mypassword",
  "broker": "mqtt://192.168.1.100:1883"
}
```

The callback parses the JSON, writes each field to NVS namespace `wifi_creds`, then triggers a Wi-Fi connect attempt. If the connect succeeds, the device transitions to Mode 1 on next boot. This characteristic is write-only by design — the credentials cannot be read back.

### Advertising

BLE advertising starts on boot and restarts automatically after a client disconnects. In Mode 1 (active Wi-Fi control), advertising is suppressed.

---

## mqtt_app — ESP-MQTT client

Stack: ESP-MQTT (`CONFIG_MQTT_PROTOCOL_311=y`).

**Base topic:** `cosmicx/devices/{device_id}/`

The device ID is derived from the ESP32-S3 MAC address.

### Topics

| Topic suffix | Direction | QoS | Retained |
|-------------|-----------|-----|----------|
| `state` | Device → Broker | 1 | Yes |
| `command` | Broker → Device | 1 | No |
| `telemetry` | Device → Broker | 0 | No |
| `status` | Device → Broker (LWT) | 1 | Yes |

**state:** Published on every FSM state change. Payload mirrors the BLE state JSON plus a timestamp.

**command:** Subscribed at connection. Payload is the same string format as BLE 0x1235 (`"OPEN"`, `"SET_POS:75"`, etc.). Internally it uses the same `curtain_cmd_t` queue — the FSM does not know or care whether a command came from BLE or MQTT.

**telemetry:** Published every 30 seconds.

```json
{
  "temp_c": 42.3,
  "vbus_v": 12.03,
  "uptime_s": 3600,
  "position": 75,
  "rssi": -58
}
```

**status (LWT):** Configured at connection time. If the device drops off the network, the broker publishes `"offline"` to this topic within the keep-alive window. The device publishes `"online"` when it reconnects.

### Wi-Fi reconnection

Wi-Fi uses `WIFI_EVENT_STA_DISCONNECTED` to trigger reconnection. The implementation uses exponential backoff: 1 s, 2 s, 4 s, 8 s, up to a maximum of 10 attempts before setting a fault flag. LED2 blinks a specific error pattern when all retries are exhausted. Measured mean reconnection time: 3.2 s from link-loss detection to IP assignment.

After IP is assigned, the MQTT client reconnects automatically and re-subscribes to the command topic. The LWT `"online"` message is published at reconnect time with `retain=true`, overwriting the LWT `"offline"` message on the broker.

### OTA trigger

A `"OTA"` command on the command topic with a URL field triggers the OTA process via `esp_https_ota`. OTA is only available in Mode 1 — it needs Wi-Fi for the HTTPS download.
