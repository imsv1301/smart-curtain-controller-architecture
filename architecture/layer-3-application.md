# Layer 3 — Application

`app_main.c`. This layer does one thing: start everything else in the correct order.

No business logic here. No GPIO writes. No MQTT publishes. Just the boot sequence, the watchdog, and a few orchestration tasks.

---

## Boot sequence

Order is not flexible. The USB-PD CFG pins must be configured before any other component draws current from the board.

```c
void app_main(void)
{
    // [0] USB-PD — ABSOLUTE FIRST
    //     CFG1=LOW, CFG2=LOW, CFG3=HIGH → requests 12 V
    //     If this runs after nvs_flash_init(), the ~100 ms NVS delay
    //     means motor coils see 5 V, causing stall false-positives.
    usbpd_12v_init();

    // [1] NVS — read stored Wi-Fi creds, motor config, curtain position
    ESP_ERROR_CHECK(nvs_flash_init());

    // [2] LED — visual feedback during rest of boot
    led_indicator_init();

    // [3] Watchdog — 30 s timeout, panic on trip (generates crash dump)
    esp_task_wdt_reconfigure(&wdt_cfg);
    esp_task_wdt_add(NULL);

    // [4] Hardware inputs
    limit_sensor_init();
    button_handler_init();

    // [5] Motor driver — UART config, register writes, StealthChop2 ON
    tmc2209_init();

    // [6] Curtain FSM — restore position from NVS, start FSM task
    curtain_init();

    // [7] BLE — NimBLE stack, GATT server, start advertising
    ble_app_init();

    // [8] Wi-Fi + MQTT — only if NVS has credentials
    if (nvs_has_wifi_creds()) {
        wifi_start();
        // MQTT connects after IP_EVENT_STA_GOT_IP fires
    }
}
```

### Why USB-PD is first

The CH224K USB-PD negotiator sets the charger output voltage via hardware CFG pins. Negotiation happens at the moment the device powers up — the charger checks the CFG pin states and sets its output accordingly. If any other initialisation runs first and draws enough current to delay the CFG pin setup, the board can miss the negotiation window and fall back to 5 V. At 5 V, the stepper motor has about 40% of its rated torque, which causes the curtain to stall before it fully opens.

---

## Watchdog

30-second hardware watchdog. `trigger_panic=true` means a watchdog trip generates a full crash dump (stack trace, register state, backtrace) before rebooting. This is the only way to debug hang conditions in production without JTAG access.

The main app task feeds the watchdog. If any layer deadlocks — stuck waiting on a queue with no timeout, infinite loop, blocked I2C transaction — the watchdog fires within 30 seconds and the device reboots.

---

## OTA updates

Triggered by an MQTT command: `{"cmd": "OTA", "url": "https://updates.thecosmicx.com/curtain/v2.bin"}`.

The `mqtt_app` layer receives this and calls into an OTA helper (still Layer 3 — it orchestrates the process but delegates the actual download to `esp_https_ota`).

```
1. Receive OTA URL via MQTT
2. esp_https_ota downloads binary to inactive OTA partition
   (if ota_0 is active, download writes to ota_1)
3. SHA-256 hash verified against embedded firmware header
4. otadata partition updated to point to new partition
5. Device reboots
6. Bootloader boots from new partition
7. New firmware calls esp_ota_mark_app_valid_cancel_rollback()
   within 30 s of boot
8. If step 7 doesn't happen → watchdog fires → bootloader
   sees mark_valid was never called → reverts to previous partition
```

The rollback window (step 7–8) is 30 seconds — same as the watchdog timeout. A new firmware that crashes before it can mark itself valid will always revert. The device cannot be permanently bricked by a bad OTA update.

---

## NVS namespaces

| Namespace | Keys | Written by |
|-----------|------|-----------|
| `wifi_creds` | `ssid`, `password`, `broker_uri` | ble_app (char 0x1238) |
| `curtain` | `position`, `open_steps`, `close_steps` | curtain_controller |
| `motor` | `speed`, `current_ma`, `sgthrs` | mqtt_app / ble_app (char 0x1237) |

NVS uses wear-levelling across the 16 KB `nvs` partition — writes survive power cycles indefinitely without wearing out a single flash page.

---

## Flash partition layout

```
# partitions.csv
# Name,   Type, SubType, Offset,   Size,     Flags
nvs,       data, nvs,     0x10000,  0x4000,
otadata,   data, ota,     0x1D000,  0x2000,
ota_0,     app,  ota_0,   0x20000,  0x300000,
ota_1,     app,  ota_1,   0x320000, 0x300000,
```

Two 3 MB app partitions. At 8 MB flash, this leaves ~2 MB free after NVS and bootloader overhead.
