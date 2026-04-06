# OTA Firmware Update Process

The device supports wireless firmware updates over HTTPS. The update process is fail-safe: a bad firmware that crashes on boot will always revert to the previous working version. The device cannot be permanently bricked by an OTA update.

---

## Prerequisites

- Device must be in Mode 1 (Wi-Fi + MQTT active)
- Firmware binary must be hosted over HTTPS
- Binary must be compiled for ESP32-S3 with matching partition layout

---

## Flash partition layout

```
Offset     Name       Size     Purpose
0x1000     bootloader  4 KB    Reads otadata, decides which app to boot
0xD000     otadata     8 KB    Tracks which OTA partition is active
0x10000    nvs        16 KB    Wi-Fi creds, config, position
0x20000    ota_0       3 MB    First application partition
0x320000   ota_1       3 MB    Second application partition
```

At any given time, one partition is active (running) and one is inactive (available for download).

---

## Update sequence

```
Step 1 — Trigger
  Dashboard publishes to .../command:
  "OTA:https://updates.thecosmicx.com/curtain/v2.bin"

Step 2 — Download
  esp_https_ota opens HTTPS connection to the URL
  Binary downloads in chunks (4 KB blocks) directly to the
  INACTIVE partition — if ota_0 is running, ota_1 receives the data
  The running firmware continues operating during download

Step 3 — Verify
  SHA-256 hash of downloaded binary verified against the hash
  embedded in the firmware image header
  If hash mismatch → download rejected, device stays on current firmware

Step 4 — Commit
  otadata partition updated to mark the new partition as bootable
  Device publishes {"status":"rebooting","reason":"ota"} to telemetry topic

Step 5 — Reboot
  Device reboots
  Bootloader reads otadata, boots from the new partition

Step 6 — Validate
  New firmware has 30 seconds to call:
  esp_ota_mark_app_valid_cancel_rollback()

  This function signals to the bootloader that the new firmware
  started successfully and should be kept permanently.

Step 7 — Rollback (if step 6 doesn't happen)
  If the watchdog fires (30 s timeout) before mark_app_valid is called,
  the bootloader sees the partition was never validated.
  On next boot, the bootloader reverts to the previous partition.
  The device comes back online running the old firmware.
```

---

## Rollback guarantee

The 30-second window between reboot and `mark_app_valid` is the critical safety window. If new firmware:
- Crashes immediately → watchdog fires → rollback
- Hangs in app_main → watchdog fires → rollback
- Connects to Wi-Fi but fails to call mark_valid → watchdog fires → rollback
- Starts normally and calls mark_valid → kept permanently

There is no way to deploy a firmware that permanently prevents the device from rebooting into a working state, as long as the bootloader and otadata partitions are intact (they are never written during OTA).

---

## Partition switching

After a successful OTA update, the active partition alternates:

| Before OTA | Download goes to | After OTA |
|-----------|-----------------|-----------|
| ota_0 (active) | ota_1 | ota_1 (active) |
| ota_1 (active) | ota_0 | ota_0 (active) |

The previously active partition is not erased. It remains available as the rollback target for the next OTA cycle.

---

## Monitoring OTA progress

During download, the device publishes to `.../telemetry`:

```json
{"ota_progress": 45, "ota_state": "downloading"}
```

After reboot and successful validation:

```json
{"ota_state": "complete", "fw_version": "2.1.0"}
```

After rollback:

```json
{"ota_state": "rolled_back", "reason": "validation_timeout"}
```
