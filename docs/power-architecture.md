# Power Architecture

The board runs entirely from a USB-C port. A USB-PD negotiator requests 12 V from the charger, and a buck converter steps it down to 3.3 V for logic. Two separate power rails serve two different loads.

---

## Full power chain

```
USB-C Charger (18 W or higher, USB-PD capable)
              │
              │  5 V default output
              ▼
       CH224K USB-PD Negotiator
       CFG1 = LOW (GPIO38)
       CFG2 = LOW (GPIO48)
       CFG3 = HIGH (GPIO47)
              │
              │  12 V (after negotiation)
              │
              ├────────────────────────────────────────────►
              │                                             │
              ▼                                             ▼
     NPN3613-33 Buck Converter                  TMC2209 Stepper Driver
     12 V → 3.3 V, ~85–95% efficiency           Motor coils: A1, B1, A2, B2
              │                                  Peak: 0.85 A @ 12 V = 10.2 W
              │  3.3 V
              ▼
     ESP32-S3-MINI-1 + all logic chips
     LEDs, buttons, NTC, VBUS sense ADC
     Typical draw: ~250 mA active
```

---

## CH224K — USB-PD negotiation

The CH224K is a USB Power Delivery sink negotiator. It communicates with the charger over the USB-C CC pins using the USB-PD protocol and requests a specific voltage. The charger switches its output accordingly.

**This is not a voltage converter.** The charger outputs 12 V directly. CH224K just asked for it.

### CFG pin configuration

| Pin | GPIO | Level | Purpose |
|-----|------|-------|---------|
| CFG1 | GPIO38 | LOW | |
| CFG2 | GPIO48 | LOW | → Together: request 12 V output |
| CFG3 | GPIO47 | HIGH | |

These three pins select the voltage request according to the CH224K datasheet voltage configuration table. Setting CFG1=LOW, CFG2=LOW, CFG3=HIGH maps to 12 V.

### Why firmware sets these first

The CFG pins are read by the CH224K at power-up during PD negotiation. If `app_main` runs any initialisation before configuring these GPIO pins, the delay (~50–100 ms for `nvs_flash_init`) can cause the board to miss the negotiation window. The charger then defaults to 5 V.

At 5 V, the TMC2209 motor coils get roughly 40% of rated torque. The first motor move after boot will likely stall before the curtain reaches full open, triggering a false positive on the StallGuard2 interrupt.

This is why USB-PD init is the absolute first call in `app_main`.

### What happens with a non-PD charger

If the charger doesn't support USB Power Delivery, CH224K cannot negotiate. The charger stays at 5 V. The device boots normally, but the motor will not have enough torque for reliable operation. The `vbus_v` field in telemetry will show ~5.0 V instead of ~12.0 V.

---

## Buck converter — NPN3613-33

A synchronous buck converter on the PCB steps 12 V down to 3.3 V. Buck converters work by rapidly switching the input on and off (hundreds of kHz) and using an inductor and capacitor to smooth the pulsed output into a steady DC voltage.

Efficiency: 85–95%. This means for every watt drawn at 3.3 V, ~1.05–1.18 W is consumed from the 12 V input. The remainder (5–15%) becomes heat on the converter.

A linear regulator doing the same job would drop the full 8.7 V difference as heat: at 250 mA draw, that's 2.2 W of heat versus ~0.03 W for the buck converter. For a PCB in an enclosure, this matters.

---

## Power budget

| Component | Voltage | Typical current | Power |
|-----------|---------|----------------|-------|
| ESP32-S3 (active, Wi-Fi TX) | 3.3 V | 240 mA | 0.79 W |
| TMC2209 logic | 3.3 V | 10 mA | 0.03 W |
| LEDs (4 × max) | 3.3 V | 40 mA | 0.13 W |
| Buttons, NTC, ADC | 3.3 V | 5 mA | 0.02 W |
| **Logic total (3.3 V rail)** | | **295 mA** | **0.97 W** |
| Stepper motor (running) | 12 V | 600 mA avg | 7.2 W |
| Stepper motor (peak) | 12 V | 850 mA | **10.2 W** |
| **Total (12 V rail peak)** | | | **~11.2 W** |

An 18 W USB-PD charger provides comfortable headroom. The CH224K will not negotiate with chargers rated below 15 W.

---

## VBUS monitoring

GPIO4 connects to a resistor divider on the 12 V rail. The ESP32-S3 ADC reads this periodically. Normal readings: 12.03 V ± 0.05 V (measured under full motor load).

If VBUS drops below 10 V, the driver posts a `POWER_FAULT` event. The curtain FSM stops the motor and sets LED1 to fast-blink red. The device does not attempt to run the motor under a power fault.
