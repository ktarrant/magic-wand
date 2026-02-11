Perfect — **one visible USB (charging only)** is the cleanest/strongest baton mechanically, and it keeps the electrical design dead simple.

Here’s the **hardware-locked update** of the design doc with that decision baked in, including a concrete mid-body layout for **30 mm ID** (and an easy fallback to 38 mm).

---

# Hardware Design Document v2.1

## RP2040 + LSM6DSR (SPI) + 36× WS2812

### Single visible USB for charging (programming requires opening baton)

## 1) Summary

A 2-ft throwable LED baton using:

* **Raspberry Pi Pico (RP2040)** for control + WS2812 via **PIO**
* **LSM6DSR** IMU on **SPI**
* **36× WS2812** LEDs on a strip inside a diffused tube
* **Single 1S vape Li-ion cell** with **known 1S charger+protection module**
* Only external port: **charger USB**
* Pico USB is internal; open baton to reflash

Primary design priorities:

* **Battery safety & simplicity**
* **No random resets** from LED current spikes
* **Mechanical robustness** for throwing
* **Hand-solderable modules** (through-hole headers / large pads)

---

## 2) Mechanical envelope and layout

### Tube size targets

* **Ideal:** 30 mm inner diameter (ID)
* **Easy mode:** 38 mm ID (more padding, easier module selection)

30 mm works if we keep the middle package low-profile and avoid tall stacks.

### Overall physical layout

* **LED strip** runs nearly full length
* **Electronics “center sled”** located at the baton midpoint (center of mass)
* **Single charging USB window** on the tube at the center (grip area)
* Ends are sealed with foam end caps; electronics are not at the ends

### Center sled concept (recommended)

A rigid internal carrier ~**140–180 mm long** that holds:

* battery (centered)
* charger/protection module (USB aligned to window)
* 3.3 V buck/boost module
* Pico (internal-only USB)
* IMU breakout
* bulk capacitor + series resistor

**Sled materials (pick one):**

* 3D printed sled (best packaging control)
* FR4 strip / acrylic strip (simple, stiff)
* perfboard only as a mounting plate (avoid tall component stacks)

**Retention:** foam cradle + 2–3 zip ties or screws → sled cannot rattle.

---

## 3) Electrical architecture

### 3.1 Rails

* **VBAT (protected):** battery voltage after protection, ~3.0–4.2 V

  * Powers WS2812 strip directly
  * Feeds buck/boost regulator
* **3.3 V:** regulated logic rail for Pico + IMU

### 3.2 Star power topology (must-follow)

At the **protected battery output**, create a star point:

* Branch A: **VBAT → LED strip V+**
* Branch B: **VBAT → 3.3V buck/boost → Pico + IMU**

Ground returns also star back to the same point:

* LED ground return does **not** share thin traces/wires with logic ground.

This is your main defense against resets/flicker.

---

## 4) Modules and parts (hand-solderable)

### 4.1 MCU

* **Raspberry Pi Pico**

  * Solder on 0.1" headers (through-hole)
  * Mount flat to sled (minimize flex)

### 4.2 IMU

* **LSM6DSR breakout** with:

  * 0.1" header holes
  * SPI pins clearly labeled
  * 3.3V supply supported (ideal is “no onboard level shifting needed”)

### 4.3 Battery charger + protection (single visible USB)

Use a **known-good 1S Li-ion charger module with protection** (USB-C or micro-USB). Requirements:

* Charges 1S Li-ion safely
* Includes **undervoltage cutoff** and short protection (or separate protection board)
* Has a USB connector you can align to a tube window

**Important:** This is the **only external USB**.

### 4.4 3.3V regulator

Use a **buck/boost** module (not LDO) so the Pico is stable down near empty battery:

* Vin down to ~2.7–3.0 V
* 3.3 V out
* ≥300 mA continuous

Choose a module with **through-hole pins / large pads** and a low profile.

---

## 5) LED subsystem details

### 5.1 LED strip

* WS2812 strip cut to **36 LEDs**
* Place at least one **power injection** at the near end (closest to sled)

If brightness gradient appears, inject power at the far end too:

* VBAT and GND to far end pads (still one cell; just reducing voltage drop).

### 5.2 Required “stability kit” (non-negotiable)

At the LED strip input (near the sled):

* **1000 µF electrolytic** across LED V+ / GND
* **330 Ω series resistor** on LED data line near Pico output

These reduce sag and ringing that cause flicker or resets.

---

## 6) Data level considerations (3.3V Pico → WS2812 at VBAT)

Baseline approach:

* Drive WS2812 data directly from Pico (3.3 V)
* Keep data wire short and referenced to the same ground
* Use series resistor

**If you see corruption (especially at fresh charge 4.2 V):**
Hardware mitigation path (in order of simplicity):

1. **Schottky diode in LED V+** to drop LED rail slightly (improves logic margin)
2. Add **74AHCT** buffer powered from LED V+ (most robust, still small)

We won’t add these unless needed.

---

## 7) Concrete pin plan (Pico)

### WS2812 (PIO)

* LED_DATA: **GP0** (or any convenient GPIO)

### SPI to LSM6DSR (SPI0 suggested)

* SCK:  **GP18**
* MOSI: **GP19**
* MISO: **GP16**
* CS:   **GP17** (GPIO)

### IMU interrupt (optional, recommended)

* INT1: **GP20** (any GPIO)

### Optional battery sense (future-proof)

* VBAT_SENSE: **GP26 / ADC0** via divider (can be unpopulated in v1)

---

## 8) Physical placement inside 30 mm ID

### Recommended orientation

Place everything “flat” on the sled so the thickness stays low:

* **Pico** mounted flat
* **Charger module** mounted flat with USB connector facing outward to the tube window
* **Buck/boost** mounted flat
* **Bulk capacitor** laid **sideways** if height is tight (or choose a shorter can)

### Tube window (charging)

* One rectangular cutout aligned with **charger USB**
* Add a small **bezel** (3D print or epoxy-formed) so plug forces push into the tube, not into the PCB solder joints.

### Service access for programming

* One endcap is removable
* Sled can slide out for Pico USB access

---

## 9) Wiring rules for baton survivability

* No Dupont jumpers.
* Use soldered joints + heatshrink.
* Add strain relief where wires meet:

  * LED strip pads
  * battery leads
  * module pads
* Immobilize wires to the sled (zip tie anchors or adhesive) so solder joints aren’t load-bearing.

---

## 10) Bring-up tests (hardware before software)

1. **Power test**

* Charger charges battery.
* Protection behaves (no weird heat).
* 3.3 V rail is stable across battery range.

2. **LED test (low brightness)**

* Confirm no resets/flicker when patterns change quickly.
* Confirm bulk cap and data resistor are installed.

3. **IMU SPI test**

* Read WHOAMI.
* Stream accel/gyro at target rate.

4. **Shake test**

* No intermittent power/data issues under vibration.

Only after this do you start throwing.
