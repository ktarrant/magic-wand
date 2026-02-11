# Updated Hardware Design Document

## Motion-Reactive Throwable LED Baton

**Platform:** RP2040 (Raspberry Pi Pico form factor)
**IMU:** ST **LSM6DSR** over **SPI**
**LEDs:** ~36× **WS2812** (strip inside ~2 ft diffuser tube)
**Power:** **Single salvaged vape Li-ion cell (1S)**, “safe + simple”

---

## 1) Goals and constraints

### Goals

* Durable baton you can toss; IMU detects motion/impact/free-fall and drives LED effects.
* Hardware that’s **solderable by hand** (prefer through-hole / big pads).
* Avoid complex battery pack management: **single cell only**.

### Constraints

* Battery rail is **3.0–4.2 V**, with big current spikes from LEDs.
* WS2812 worst-case current is huge; we will enforce **firmware brightness limits**.
* Mechanical shocks require: strain relief, robust wiring, minimal connectors.

---

## 2) Finalized architecture

### Power rails (two-rail system)

* **Rail A: LED power (VBAT)**
  Battery (after protection) directly powers the WS2812 strip.
* **Rail B: Logic power (3.3 V)**
  Battery → **3.3 V buck/boost regulator** → RP2040 + LSM6DSR breakout.

This is the simplest design that prevents MCU brownouts caused by LED sag.

---

## 3) Hardware selections (nailed down)

### 3.1 Controller board: RP2040

**Recommended:** **Raspberry Pi Pico** (or compatible) + **0.1" headers you solder yourself**
Why:

* Through-hole header soldering is easy.
* Tons of proven examples for **WS2812 via PIO**.
* Easy debug/program via USB.

Notes:

* You can mount it on a small perfboard or custom carrier with screw/zip-tie strain relief.

---

### 3.2 IMU: LSM6DSR over SPI

**Recommended:** LSM6DSR breakout that includes:

* **0.1" header holes**
* **3.3 V-compatible IO**
* Optional onboard regulator/level shifting is fine, but not required if it’s a 3.3 V breakout.

SPI is the right call here: robust signal integrity, higher rates, less I²C fuss.

---

### 3.3 LED strip: WS2812

* 36 LEDs total is solid for a 2 ft baton (looks great with diffusion).
* Recommended density: **60 LED/m** and use ~0.6 m (or cut to length).
* Plan for **power injection at the start**; consider a second injection at the far end if you see gradient.

---

### 3.4 Battery charging + protection

For “safe + simple”, don’t rely on mystery vape charge PCBs unless you’ve identified them well.

**Recommended approach:** Use a known **1S Li-ion charger module with protection** (common TP4056+protection style is fine).

* USB in, BAT+/BAT-, OUT+/OUT- (if present) makes wiring clean.
* Include a **physical power switch** on the protected output.

If your charger module does **not** have “load sharing/power path”, that’s still okay for this project (you can avoid using it while charging).

---

### 3.5 3.3 V buck/boost regulator module (big pads / hand solderable)

Pick a **buck/boost** (not just an LDO) so the RP2040 stays stable as the cell drains.

**Target specs**

* Input: **2.7–4.5 V** (or wider)
* Output: **3.3 V**
* Continuous: **≥300 mA** (more is fine)
* Prefer a module with **through-hole pins / large pads**.

This is the single most important “stability” part besides capacitors.

---

## 4) Wiring plan (pin-level)

### 4.1 Proposed Pico pin usage

You can change pins later, but it’s helpful to commit now.

#### WS2812 (PIO output)

* **LED_DATA:** Pico **GP0** (PIO-friendly, simple)

#### SPI to LSM6DSR (SPI0 suggested)

* **SCK:**  GP18
* **MOSI:** GP19
* **MISO:** GP16
* **CS:**   GP17  (any GPIO works as CS)

(These map cleanly to SPI0 on many Pico examples.)

#### Optional battery sense (future-proof)

* **VBAT_SENSE:** GP26 / ADC0 via resistor divider (optional; can leave unpopulated)

---

### 4.2 Power wiring and grounding (must-follow)

Create a **star point** at the protected battery output:

* Branch 1 (high current): **VBAT → LED +**
* Branch 2 (clean): **VBAT → buck/boost → 3.3V → Pico + IMU**

Grounds return separately to the star point:

* LED GND returns directly to battery/protection GND
* Logic GND returns separately to the same point

Do **not** daisy-chain logic ground through the LED ground path.

---

## 5) Required passives and “stability kit”

### 5.1 LED rail components (at the strip input)

* **Bulk capacitor:** **1000 µF** (470 µF minimum) across LED + / GND at the strip input
* **Data series resistor:** **330 Ω** (acceptable range 220–470 Ω) in series with LED_DATA near the Pico

These two components prevent 90% of “random flicker / reset” issues.

### 5.2 Logic rail decoupling

* At Pico 3.3 V input: **10–22 µF + 0.1 µF**
* At IMU breakout: **10 µF + 0.1 µF**

### 5.3 Optional but useful

* **LED power injection at far end** (VBAT and GND) if brightness drops along the strip.

---

## 6) WS2812 data-level reliability (important decision)

Because you’re powering LEDs from **VBAT up to 4.2 V** and the Pico outputs **3.3 V**, most WS2812 strips will still work, but some batches are picky.

### Baseline plan (keep it simple)

* No level shifter initially.
* Keep LED_DATA wire short.
* Use the 330 Ω series resistor + strong ground.

### If you see flicker/corruption (especially right after charging at 4.2 V)

Pick one of these *hardware* mitigations:

**Mitigation A (still simple): drop LED V+ slightly**

* Add a **Schottky diode** in series with LED V+ to shave ~0.2–0.3 V at typical currents.
* This increases your logic margin without adding a new rail.

**Mitigation B (most robust): add a tiny “data-only 5 V” rail**

* Add a small boost set to **5 V** powering only a **single-gate buffer** (not the LED strip).
* Use a proper WS2812 buffer (AHCT-style) at 5 V.
* This adds parts, but still avoids a high-current 5 V LED rail.

Documenting this up front keeps you from getting stuck later.

---

## 7) Brightness and current constraints (hardware-informed)

### LED current reality

* WS2812 worst case: 36 × 60 mA = **2.16 A** (don’t run this)
* We will enforce firmware limits: **~20–25% global brightness** typical.

### Consequences of single-cell

* **Runtime**: typically ~30–60 minutes depending on patterns and your specific cell capacity.
* **Brightness**: plenty bright in a diffuser at night, but not “max brightness floodlight.”
* **Reliability**: higher, because fewer battery failure modes.

---

## 8) Mechanical plan (hardware-driven)

### Body

* Polycarbonate tube (diffused) about 2 ft
* Foam end caps for impact

### Internal layout

* LED strip along the tube
* **Battery + Pico + power modules near the center**
* Strain relief on:

  * LED strip solder joints
  * battery leads
  * module-to-module wiring

### Service access

* One end cap removable for:

  * USB charging access
  * firmware USB access
  * battery inspection/replacement

---

## 9) Final BOM checklist (buildable by hand)

**Core**

* Raspberry Pi Pico + headers
* LSM6DSR breakout w/ 0.1" header holes
* WS2812 strip cut to 36 LEDs

**Power**

* 1S Li-ion USB charger module **with protection**
* 3.3 V **buck/boost** regulator module (hand solderable)
* Physical power switch (latching)

**Passives**

* 1000 µF electrolytic (LED rail)
* 10–22 µF ceramic + 0.1 µF ceramic (logic rail)
* 10 µF + 0.1 µF (IMU rail)
* 330 Ω resistor (LED data)

**Wiring/mech**

* Thicker wire for LED VBAT/GND
* Perfboard or small carrier board
* Heatshrink, silicone/epoxy for strain relief

---

## 10) Hardware bring-up tests (before software features)

1. **Power sanity**

* Verify 3.3 V rail holds steady across battery range (fresh charge to near-empty).
* Confirm power switch cuts everything.

2. **WS2812 smoke test**

* Run a low-brightness rainbow/chase (cap brightness in code immediately).
* Confirm no resets when patterns change quickly.

3. **SPI IMU test**

* Read WHOAMI/registers reliably.
* Stream gyro/accel at your target ODR.

4. **Combined stress**

* Worst-case animation transitions at your brightness cap.
* Shake test (vibration) before any throws.
