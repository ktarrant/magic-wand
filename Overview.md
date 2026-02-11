# Hardware Design Document v3.2 (Finalized Prototype)

## Throwable Motion-Reactive LED Baton

**Tube:** 38 mm ID (comfortable fit)
**Charger:** JacobsParts USB-C TP4056 + protection (direct USB alignment)
**MCU:** RP2040 (Raspberry Pi Pico)
**IMU:** LSM6DSR (SPI)
**LEDs:** 36× WS2812
**External UI:** USB-C + Power Switch + Mode Button + Charge Indicator (from charger board)
**Charging rule:** Baton OFF while charging

---

# 1) Mechanical Layout — Center Control Cluster

## Tube cutout (“control window”)

Single rectangular opening centered on baton:

* **Length (along baton):** 60 mm
* **Width (around tube):** 18 mm

Inside this window lives a small **bezel insert** (3D printed or acrylic) holding:

| Feature                 | Position (relative to center) | Opening size          |
| ----------------------- | ----------------------------- | --------------------- |
| USB-C (charger)         | X = 0 mm                      | 11 × 5 mm             |
| Charge indicator window | Above USB                     | 10 × 3 mm translucent |
| Power switch slot       | X = −18 mm                    | 8 × 2.5 mm            |
| Mode button hole        | X = +18 mm                    | 6 mm round            |

The bezel absorbs USB insertion force, not the PCB.

---

# 2) Center Sled (Internal Electronics Carrier)

## Sled size

* Length: **170 mm**
* Width: **25–28 mm**
* Thickness: **2–3 mm**

Material: 3D printed / FR4 / acrylic

## Component placement along sled (X-axis)

| Component                  | Position                         |
| -------------------------- | -------------------------------- |
| Charger board (26×17 mm)   | **X = 0 mm**, USB facing outward |
| Power switch               | X = −18 mm                       |
| Mode tact switch           | X = +18 mm                       |
| Bulk capacitor (1000 µF)   | X = −40 mm                       |
| Buck/boost (Pololu S7V8F3) | X = +40 mm                       |
| RP2040 Pico                | X = +70 → +120 mm                |
| LSM6DSR                    | Near Pico, rigidly mounted       |
| Battery                    | Centered near X = 0              |

All mounted **flat**, not stacked.

---

# 3) Electrical Architecture (Locked)

## Power flow

```
USB-C → Charger/Protection → OUT+ → Power Switch → VBAT_STAR
                                            ├─→ LED V+ (WS2812)
                                            └─→ Buck/Boost → 3.3V → Pico + IMU
```

Ground returns meet at **GND_STAR**.

---

# 4) Critical Electrical Components

## Battery & Charging

* JacobsParts TP4056 USB-C charger + protection
* Use **OUT+/OUT−** (protected output) if available

## 3.3 V Logic Rail

* **Pololu S7V8F3** buck/boost (2.7–11.8 V → 3.3 V)

Decoupling:

* 10–22 µF + 0.1 µF near Pico
* 10 µF + 0.1 µF near IMU

---

## WS2812 Stability Kit (mandatory)

At LED strip input:

* **1000 µF electrolytic**
* **330 Ω series resistor** on LED data

Optional if needed:

* Schottky diode in LED V+
* AHCT buffer for data

---

# 5) MCU + IMU Wiring

## Pico Pins

| Function                 | Pin  |
| ------------------------ | ---- |
| WS2812 Data (PIO)        | GP0  |
| SPI SCK                  | GP18 |
| SPI MOSI                 | GP19 |
| SPI MISO                 | GP16 |
| IMU CS                   | GP17 |
| IMU INT (optional)       | GP20 |
| Mode button              | GP2  |
| Battery sense (optional) | GP26 |

Mode button: GPIO → GND (use pull-up).

---

# 6) External Controls

## Power Switch

* SPST via SPDT slide switch (C&K OS102011MA1QN1)
* Placed in series with **OUT+**
* Recessed to prevent accidental toggle

## Mode Button

* 6×6×5 mm tact switch behind bezel
* Actuated via plunger or silicone boot

## Charge Indicator

* Use charger module LEDs via translucent window
* No LED strip use while charging

---

# 7) Wiring Rules (for durability)

* No Dupont wires
* All soldered + heatshrink
* Strain relief:

  * LED strip pads
  * battery leads
  * module connections
* Foam cradle prevents sled movement
* Wires tied to sled within ~15 mm of joints

---

# 8) Expected Electrical Performance

| Metric              | Expected                                  |
| ------------------- | ----------------------------------------- |
| Runtime             | ~30–60 min (brightness capped ~20–25%)    |
| Max current typical | ~0.3–0.8 A                                |
| Peak bursts         | ~1–1.5 A                                  |
| Stability           | No resets if star routing + bulk cap used |

---

# 9) Hardware Bring-Up Checklist

1. Charger works, LEDs indicate correctly
2. Switch fully kills system power
3. 3.3 V rail stable from full → near empty battery
4. WS2812 stable at low brightness
5. IMU SPI stable at full data rate
6. Shake test passes (no intermittent wiring)
7. Soft drop test passes

---

# 10) What is now fully locked

* Tube size: **38 mm ID**
* Control cluster layout
* Charger module type (JacobsParts)
* Power topology
* Switch + button approach
* Module placement
* Wiring architecture
* LED rail design

No major hardware unknowns remain.
