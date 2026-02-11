# Hardware Design Document v3.0

## Throwable Motion-Reactive LED Baton (Prototype)

**Tube:** 38 mm inner diameter (ID), diffused polycarbonate
**MCU:** RP2040 (Raspberry Pi Pico)
**IMU:** ST LSM6DSR (SPI) breakout
**LEDs:** 36× WS2812 (strip)
**Battery:** Single salvaged 1S vape Li-ion cell
**External UI:** One central cluster: **USB-C charging**, **power switch**, **mode button**, **charge indicator LEDs**
**Charging behavior:** Baton **must be OFF** while charging (no LED array indication during charging)

---

## 1) Requirements and decisions

### Locked decisions

* **One external USB port**: USB-C for charging only
* **Programming**: open baton to access Pico USB internally
* **Battery**: single 1S cell only (no parallel/series packs)
* **Off while charging**: enforced by user behavior + design assumption
* **Control cluster** in grip area includes:

  * USB-C port (charger module)
  * latching power switch
  * mode button (momentary)
  * charge status indicator light(s) (from charger module)

---

## 2) System architecture

### Electrical block diagram

```
USB-C (charging) → 1S Charger+Protection → OUT+ → [Power Switch] → VBAT_STAR
                                                        |
                                                        +→ LED rail (VBAT) → WS2812 strip
                                                        |
                                                        +→ 3.3V buck/boost → Pico + LSM6DSR
```

### Key design principles

* **Two rails**:

  * LED rail = **VBAT** (3.0–4.2 V)
  * Logic rail = **3.3 V** from buck/boost
* **Star power routing** at VBAT_STAR to prevent LED current spikes from resetting logic

---

## 3) Mechanical design

### Tube

* 38 mm ID gives comfortable clearance for:

  * low-profile sled
  * foam shock mounting
  * tidy wire routing
  * a robust central “bezel” for the control cluster

### Central control cluster (“bezel”)

A reinforced mid-body area (where your hand goes) with:

1. **USB-C receptacle opening** (charger board)
2. **Power switch opening** (recessed)
3. **Mode button opening** (plunger or silicone boot)
4. **Indicator window(s)** aligned to charger board LEDs (CHRG/DONE)

**Bezel goals**

* Take insertion forces from USB plug without stressing PCB solder joints
* Prevent accidental switch toggles
* Provide water/dust resistance (optional later)

### Center sled

A rigid internal carrier ~150–200 mm long holding all electronics.
Recommended materials:

* 3D printed sled (best)
* FR4 strip / acrylic strip (good)
* Perfboard only as a mounting base (avoid tall stacks)

**Retention**

* Foam wrap + zip ties/screws so the sled cannot rattle

### Component placement (centered)

* Battery located at the **center of mass**
* Charger board placed so its USB-C aligns to the bezel
* Pico + buck/boost + IMU adjacent, all on the sled
* Bulk capacitor near the LED power branch

---

## 4) Electrical design details

## 4.1 Battery subsystem

* Single 1S cell salvaged from vape (3.0–4.2 V)
* Use a **known charger+protection module** with:

  * USB-C input
  * charge status LED(s) onboard
  * protection for undervoltage/overcurrent/short

**Critical requirement:** Protected output (OUT+/OUT−) is preferred.

---

## 4.2 Power switching (hard OFF)

### Component

* **Latching slide switch** (SPST) recommended, small, recessed

### Wiring

* Charger/protection **OUT+ → switch → VBAT_STAR**
* Charger/protection **OUT− → GND_STAR** (do not switch ground)

This ensures:

* zero parasitic drain when off
* charging state indication remains visible (charger LEDs)
* predictable behavior: OFF means everything is off

---

## 4.3 3.3 V logic rail

### Regulator type

* **Buck/boost 3.3 V** (not LDO)

### Requirements

* Input: down to ~2.7–3.0 V
* Output: 3.3 V
* ≥300 mA continuous
* Module with large pads / through-hole pins

### Decoupling

* At regulator output near Pico: **10–22 µF + 0.1 µF**
* At IMU: **10 µF + 0.1 µF**

---

## 4.4 LED subsystem (VBAT rail)

* WS2812 V+ tied to **VBAT_STAR**
* WS2812 GND tied to **GND_STAR**

### Mandatory stability parts

At the LED strip input (closest to the sled):

* **1000 µF electrolytic** across LED V+/GND
* **330 Ω series resistor** on LED_DATA near Pico output

### Power injection

* Inject at start.
* If you see voltage drop/gradient, inject VBAT/GND at the far end too.

---

## 4.5 WS2812 data voltage margin

Baseline: direct Pico 3.3 V data, short wire, good ground, series resistor.

If corruption occurs (esp. at 4.2 V battery):

* Add a **Schottky diode** in series with LED V+ *or*
* Add a **74AHCT buffer** powered from LED V+ (most robust)

Not required for v1 unless you see issues.

---

## 5) MCU + IMU interface

### Pico mounting

* Pico with through-hole headers soldered
* Mount flat to sled, with strain relief on all wires

### SPI pin plan (suggested)

* WS2812 data (PIO): **GP0**
* SPI0:

  * SCK:  **GP18**
  * MOSI: **GP19**
  * MISO: **GP16**
  * CS:   **GP17**
* Optional IMU interrupt: **GP20**
* Optional battery sense: **GP26/ADC0** (divider footprint can be left unpopulated)

### IMU mounting orientation

* Mount the IMU breakout rigidly to sled (no flex)
* Document axis orientation for software (label X/Y/Z relative to baton)

---

## 6) User interface behavior

### Power switch

* OFF: device fully off
* ON: device runs normally

### Mode button

* Short press: cycle lighting mode
* Long press (optional): brightness step or “lock” mode

### Charge indication

* Provided solely by charger module LEDs:

  * Charging: CHRG LED
  * Full: DONE LED
* Baton must be OFF while charging (no LED array output during charge)

---

## 7) BOM (prototype-ready categories)

**Core**

* Raspberry Pi Pico + 0.1" headers
* LSM6DSR SPI breakout (0.1" headers)
* WS2812 strip cut to 36 LEDs

**Power**

* 1S Li-ion USB-C charger+protection module with onboard status LED(s)
* 3.3 V buck/boost module (hand solderable)
* SPST latching slide switch (panel/recess mount style)

**UI**

* Momentary tactile switch (on sled) + external actuator:

  * silicone boot or printed plunger

**Passives**

* 1000 µF electrolytic (LED rail bulk)
* 10–22 µF ceramic + 0.1 µF ceramic (logic rail)
* 10 µF + 0.1 µF (IMU rail)
* 330 Ω resistor (LED data series)

**Mechanical**

* 38 mm ID diffused tube
* foam end caps
* foam wrap/padding for sled
* bezel/lightpipe material (3D print or epoxy + translucent insert)

---

## 8) Hardware bring-up checklist (before “real” throwing)

1. **Charger + protection**

* Plug USB-C in: CHRG LED lights
* Battery reaches full: DONE LED
* No overheating

2. **Power switch**

* OFF: no current draw to system
* ON: stable VBAT_STAR and 3.3 V rail

3. **LED test at low brightness**

* No flicker, no resets during rapid pattern changes
* Bulk cap installed and close to strip input

4. **SPI IMU test**

* WHOAMI reads reliably
* Streaming accel/gyro at target ODR works

5. **Shake test**

* No intermittent power/data
* No rattles

6. **Soft drop test**

* On carpet/foam first
* Re-check solder joints and connectors
