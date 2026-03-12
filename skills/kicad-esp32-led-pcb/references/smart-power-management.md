# Smart Power Management for Portable LED Props

Reference for designing compact, high-current LiPo-powered LED controller boards with intelligent push-button on/off control. Optimized for wearable/handheld LED props (cosplay, stage, art installations) requiring 5V 3-5A from a single LiPo cell.

---

## 1. Power Architecture for LED Props

### Requirements Profile

Portable LED props have unique demands:
- **Compact form factor** — board must be small enough to hide inside props
- **High burst current** — LED effects can spike to 5A+ during full-white patterns
- **Single-cell LiPo** — 3.7V nominal (3.0V–4.2V range), pouch or 18650
- **Smart on/off** — push button to power on, long-press to power off (prevent accidental shutdown)
- **Charge while assembled** — USB-C charging without disassembling the prop
- **Low standby drain** — battery shouldn't die when prop is "off"

### Architecture Options

**Option A: All-in-One Power Bank IC (Recommended for simplicity)**
```
LiPo → [Power Bank IC (IP5306/SW6106/IP5328P)] → 5V out → LEDs + LDO → 3.3V → ESP32
         ↑ USB-C charge input
         ↑ Push button on K pin
```

**Option B: Discrete Components (Maximum control & current)**
```
LiPo → [Charger IC (TP4056/BQ25895)] → Battery
Battery → [Push Button Controller (LTC2954)] → EN signal
EN → [High-Current Boost (TPS61022/TPS61032)] → 5V → LEDs
5V → [LDO (ME6211/XC6220)] → 3.3V → ESP32
```

**Option C: Hybrid (ESP32-controlled power)**
```
LiPo → [Charger (TP4056)] → Battery
Battery → [Soft Latch Circuit] → P-MOSFET → [Boost Converter] → 5V
ESP32 GPIO → holds latch open (self-sustaining power)
Long-press detected by ESP32 → release latch → power off
```

---

## 2. All-in-One Power Bank ICs

### IP5306 — Budget Option (5V/2.4A max)

Good for up to ~144 LEDs at 40% average brightness.

| Parameter | Value |
|-----------|-------|
| Charge input | 5V USB, up to 2.1A |
| Boost output | 5V, 2.4A max (12W) |
| Switching freq | 500kHz |
| Quiescent (off) | ~50µA |
| Push button | Built-in support on KEY pin |
| I2C | Some variants (programmable) |
| Package | QFN-24 (4×4mm) |

**Push button behavior (default):**
- Short press: Toggle power on
- Double press: Toggle power off
- With I2C variant: configurable, can disable auto-shutdown at low load

**Limitation:** 2.4A output max — not enough for 5A prop applications. Auto-shutdown below 50mA can kill ESP32 in deep sleep (fixable with I2C variant by setting "continuous output" mode).

**Key circuit:**
```
LiPo(+) → VBAT pin
LiPo(-) → GND
USB 5V → VIN pin
KEY pin → Push button → GND (with 10kΩ series resistor)
SW pin → 4.7µH inductor → VOUT
VOUT → 10µF + 100nF → Load
LED1-LED4 → Battery level indicator LEDs (optional)
```

### SW6106 — Mid-Range (5V/3A, 18W total)

Better fit for props with moderate LED count.

| Parameter | Value |
|-----------|-------|
| Charge input | 5V-12V, up to 4A (PD/QC) |
| Boost output | 5V/3A, 9V/2A, 12V/1.5A (18W max) |
| Efficiency | Up to 95% |
| Protocols | PD3.0, QC3.0, AFC, FCP, PE2.0 |
| Battery | Single-cell Li-ion/LiPo |
| Package | QFN-40 |

**Push button:** Connect to KEY pin. Short press = on, double press = off.
Light load auto-off can be an issue — the IC may shut down when LED animation enters a dark pattern.

### IP5328P — High Performance (18W, multi-protocol)

| Parameter | Value |
|-----------|-------|
| Charge input | 5V-12V (PD/QC) |
| Boost output | 5V/3A, 9V/2A, 12V/1.5A |
| Efficiency | ~93% |
| Push button | K pin (short press on, double press off) |
| Shutdown current | <100µA |
| Package | QFN |

**Wiring note:** Use short, thick wires (≥AWG18) from B+/B- to battery — input current can reach 5A+ during heavy load at low battery voltage.

---

## 3. High-Current Boost Converters (Discrete Approach)

For 5A output at 5V from a single LiPo, you need a serious boost converter. At 3.0V battery (near empty), input current = 5V×5A / (0.9 × 3.0V) ≈ 9.3A. The battery and traces must handle this.

### TPS61022 (TI) — 8A Switch, Best for 3-5A Output

| Parameter | Value |
|-----------|-------|
| Input range | 0.5V–5.5V |
| Output | Adjustable 2.2V–5.5V |
| Switch current limit | 6.5A–8A (valley) |
| Switching freq | 1MHz (>1.5V in), 0.6MHz (<1.5V) |
| Efficiency | ~93% at 3.7V→5V/2A |
| Quiescent (shutdown) | 0.1µA (true load disconnect) |
| Package | VQFN 2×2mm |

**Realistic output at 5V from single LiPo:**
- At 4.2V battery: ~3.5A output
- At 3.7V battery: ~3.0A output  
- At 3.0V battery: ~2.0A output (input limited by battery)

**Critical design notes:**
- EN pin controls on/off — connect to push button controller output
- Use 1µH–2.2µH inductor rated for ≥10A saturation current (e.g., XAL7030 series)
- Input caps: 2× 22µF ceramic (X5R/X7R) close to VIN
- Output caps: 2× 22µF + 1× 100µF ceramic
- The IC has been reported to self-destruct when input power is unstable — add proper UVLO
- Use DW01 + MOSFET for battery protection with ≥8A rating

### TPS61032 (TI) — Mature, 4A Switch

| Parameter | Value |
|-----------|-------|
| Input range | 1.8V–5.5V |
| Output | Up to 5.5V, 4A switch |
| Efficiency | ~96% at optimal load |
| Features | Power Good output, programmable soft-start |
| Package | QFN-10 (3×3mm) |

Easier to design with than TPS61022 but lower maximum current capability.

### TPS61023 (TI) — Compact, 3A Switch

| Parameter | Value |
|-----------|-------|
| Input range | 0.5V–5.5V |
| Switch current limit | 3.7A |
| Switching freq | 1MHz |
| Shutdown current | 0.1µA |
| Package | SOT-563 (1.2×1.6mm!) |

Extremely tiny — great for space-constrained props where you need ≤1.5A output.

### Parallel Boost Strategy for 5A+

For >3.5A at 5V from a single cell, consider:
1. **Two parallel TPS61022** with current sharing (complex, not recommended)
2. **2S LiPo pack (7.4V)** + buck converter → 5V/5A (much easier and more efficient)
3. **Single TPS61022** + current-limit LED animations to peak 3A

**Practical recommendation for 5A props:**
Use a 2S LiPo (7.4V) with a buck converter (like MP2315, AP63205) rated for 5A. This is more efficient, generates less heat, and the higher battery voltage means lower input current for the same output power.

---

## 4. Smart Push Button On/Off Controllers

### LTC2954 (Analog Devices) — Professional Grade

The gold standard for push button power control with µP interrupt.

| Parameter | Value |
|-----------|-------|
| Input range | 2.7V–26.4V |
| Quiescent current | 6µA |
| On timer (adjustable) | 32ms–1024ms (via ONT resistor) |
| Off timer (adjustable) | 1s–32s (via PDT resistor) |
| ESD on PB pin | ±10kV HBM |
| Package | DFN-8 (3×2mm), ThinSOT-8 |

**How it works:**
1. User presses button → after tON delay, EN goes active → DC/DC powers up
2. µP boots, asserts KILL pin HIGH within 512ms → system stays on
3. User long-presses button → after tOFF delay, INT goes low → µP receives interrupt
4. µP performs safe shutdown, then pulls KILL low → EN goes inactive → power off

**Adjusting off-time with PDT resistor:**
```
tPDT ≈ 0.1s × (R_PDT / 100kΩ)

Examples:
  R_PDT = 100kΩ  → tOFF ≈ 0.1s (too short)
  R_PDT = 1MΩ    → tOFF ≈ 1.0s
  R_PDT = 3.3MΩ  → tOFF ≈ 3.3s (good for props)
  R_PDT = 6.4MΩ  → tOFF ≈ 6.4s (very deliberate)
```

**Schematic (LTC2954-2 version, drives P-MOSFET):**
```
VBAT ──┬── VIN (LTC2954)
       │
       ├── 100kΩ ── PB pin ── Button ── GND
       │
       ├── ONT pin ── 100kΩ ── GND  (tON = 128ms)
       │
       ├── PDT pin ── 3.3MΩ ── GND  (tOFF = 3.3s)
       │
       ├── EN_bar ──── Gate of P-MOSFET (IRML6344)
       │                Source ← VBAT
       │                Drain → Boost converter VIN
       │
       ├── INT pin ──── ESP32 GPIO (interrupt, active low)
       │
       └── KILL pin ─── ESP32 GPIO (open-drain, pull high to stay on)
```

### TPS3422 (TI) — Simpler Alternative

| Parameter | Value |
|-----------|-------|
| Input range | 1.6V–6.5V |
| Configurable delay | Via capacitor on DEL pin |
| Reset pulse | Configurable width |
| Package | SOT-23-6 |

Simpler than LTC2954 but less control over on/off timing. Good for basic toggle.

### DIY Soft Latch Circuit (No Dedicated IC)

Using a discrete MOSFET latch — cheapest option, works well for ESP32-controlled power.

```
              VBAT
               │
          ┌────┤
          │    │ R1 (100kΩ)
          │    │
Button ───┤    ├──── Gate Q1 (P-MOSFET, e.g., SI2301)
          │    │        │
          │    │     Source ← VBAT
          │    │     Drain → System Power (Boost EN + ESP32 via LDO)
          │    │
          │    └──── R2 (10kΩ) ── Q2 (N-MOSFET, 2N7002)
          │                         │
          │              Gate ← ESP32 GPIO "POWER_HOLD"
          │              Source → GND
          │              Drain → R1 junction
          │
          └── GND (through button)
```

**How it works:**
1. Press button → pulls Q1 gate LOW via R1 → Q1 turns on → system powers up
2. ESP32 boots → sets POWER_HOLD GPIO HIGH → Q2 turns on → holds Q1 gate LOW → self-sustaining
3. To turn off: ESP32 detects long-press on another GPIO → performs shutdown → pulls POWER_HOLD LOW → Q1 turns off → power dies

**Advantages:** Very cheap, fully programmable shutdown behavior in firmware.
**Disadvantages:** No hardware-enforced off timeout, relies on ESP32 firmware.

---

## 5. Battery Charging for Props

### TP4056 + DW01 (Budget, 1A charge)

Simple and proven, but limited to 1A charging.

```
USB-C VBUS → TP4056 VCC
TP4056 BAT+ → DW01 VCC → Battery(+)
DW01 VSS → Sense resistor (0.1Ω) → Battery(-)
DW01 CS- → between resistor and MOSFET pair
```

### BQ25895 (TI) — Advanced Single-Cell Charger

| Parameter | Value |
|-----------|-------|
| Input range | 3.9V–14V (HVDCP support) |
| Charge current | Up to 5A (adjustable via I2C) |
| Buck switching freq | 1.5MHz |
| Features | I2C control, NTC monitoring, BATFET, power path management |
| Package | WQFN-24 (4×4mm) |

**Power path management** means the system can run from USB while simultaneously charging the battery — no interruption when charger is plugged in/out.

### BQ24075 (TI) — Simpler Power Path

| Parameter | Value |
|-----------|-------|
| Charge current | Up to 1.5A |
| System output | Selects between USB and battery automatically |
| Features | Hardware-only (no I2C needed), SYSOFF for power control |
| Package | WSON-10 (2.5×2.5mm) |

The SYSOFF pin can be connected to the push button controller to disconnect the system load while still allowing charging.

---

## 6. Battery Protection

### DW01A + FS8205A (Standard 1S Protection)

Standard single-cell protection. **Critical for safety.**

```
Battery(+) → DW01A VCC
             OD pin → Gate of discharge MOSFET (in FS8205A)
             OC pin → Gate of charge MOSFET (in FS8205A)
             CS pin → Between sense resistor and MOSFETs
Battery(-) → Sense resistor (0.05Ω–0.1Ω for <3A; lower for higher current) → FS8205A → GND
```

**For 5A+ applications:** FS8205A has RDS(on) ~25mΩ per FET. At 5A input: P = 5² × 0.025 = 0.625W per FET. This is marginal. Options:
1. Use parallel FS8205A pair (halves resistance)
2. Use discrete MOSFETs with lower RDS(on) (e.g., CSD17573Q5B, 1.2mΩ)
3. Use higher-rated protection IC (e.g., BQ29728 + external FETs)

### Voltage Monitoring with ESP32

The ESP32's ADC can monitor battery voltage through a resistor divider:
```
VBAT → 100kΩ → ADC_PIN → 100kΩ → GND
ADC reading = VBAT / 2
```
Use this for:
- Battery level indication (LED colors or WLED display)
- Low-battery warning (change LED animation to red blink)
- Software UVLO (shut down boost converter below 3.0V)

---

## 7. Inductor & Component Selection for High-Current Boost

### Inductor Selection (Critical for Boost Converter)

The inductor is the most important passive component in the boost converter.

For TPS61022 at 5V/3A output from 3.7V LiPo:
```
Peak input current ≈ 5V × 3A / (0.9 × 3.0V) ≈ 5.5A (at low battery)
Ripple current ≈ 30-40% of average → peak ≈ 7-8A
```

**Requirements:**
- Inductance: 1.0µH–2.2µH (per IC datasheet)
- Saturation current: ≥10A (must not saturate at peak current!)
- DCR (DC resistance): <20mΩ (lower = less loss = less heat)
- Package: 7×7mm or larger for thermal dissipation

**Recommended inductors:**
| Part Number | Inductance | Isat | DCR | Size |
|-------------|-----------|------|-----|------|
| Coilcraft XAL7030-222 | 2.2µH | 11.5A | 9.7mΩ | 7×7×3mm |
| Coilcraft XAL7070-102 | 1.0µH | 21A | 4.2mΩ | 7×7×7mm |
| Würth 744311100 | 1.0µH | 14A | 7.5mΩ | 7×7×3mm |
| TDK VLCF5020T-1R0N | 1.0µH | 7.8A | 15mΩ | 5×5×2mm |

### Capacitor Selection

**Input capacitors (boost VIN):**
- 2× 22µF ceramic (X5R or X7R, ≥10V rating)
- Place within 3mm of IC VIN pin
- Low ESR is critical for reducing input ripple

**Output capacitors (boost VOUT):**
- 2× 22µF + 1× 47µF ceramic (X5R, ≥10V rating)  
- Or: 22µF ceramic + 100µF polymer/electrolytic
- More output capacitance = less ripple but slower transient response

**DC bias derating:** A 22µF 6.3V 0805 X5R capacitor may only provide ~10µF at 5V DC bias! Use 10V or 16V rated caps, or larger packages (1206/1210), to get the actual capacitance you need.

---

## 8. PCB Layout for High-Current Boost (Compact Props)

### Compact Board Size Targets

| Application | Target Size | Layer Count |
|------------|-------------|-------------|
| Handheld wand/staff | 25×50mm | 2-layer |
| Wearable badge/armor | 30×30mm | 4-layer |
| LED matrix controller | 40×60mm | 2-layer |
| Minimal controller (ESP32-C3) | 20×30mm | 4-layer |

### Layout Rules for High-Current Boost

1. **Switching loop** (VIN → IC SW pin → inductor → VOUT cap → GND → IC GND): Make this loop as small and tight as physically possible. This is THE most critical layout rule. Large loop = EMI radiation + ringing.

2. **Input capacitors:** Place touching the VIN and GND pins. Not 5mm away. Not "close." Touching.

3. **Inductor placement:** Directly adjacent to the IC SW pin. Trace from SW to inductor pad should be short and wide (≥1mm).

4. **Output capacitors:** Place between inductor output and load, close to VOUT pin.

5. **Ground plane:** Solid ground on one layer. All high-current returns through this plane. Stitch with vias densely around the power section.

6. **Thermal pad:** The IC's exposed pad must have a thermal via array connecting to ground plane for heat dissipation.

7. **Keep the boost converter section separate from the ESP32 section.** Route 5V out from the power section to the ESP32 and LED section via a clean trace/pour. Don't let switching noise couple into the ESP32's analog pins.

8. **Battery connector:** Use thick traces (≥3mm or copper pour) from battery connector to boost converter input. At 9A peak, even short traces matter.

---

## 9. Compact Connector Options for Props

| Connector | Current Rating | Purpose | Size |
|-----------|---------------|---------|------|
| JST-PH 2-pin | 2A | LiPo battery | 4.5mm height |
| XT30 | 30A | High-current battery | 15mm |
| XT60 | 60A | Very high current | 22mm |
| JST-SM 3-pin | 3A | LED strip data+power | Small |
| USB-C | 3A (5V) / 5A (20V PD) | Charging | Standard |
| Spring-loaded pogo pins | 2-5A | Magnetic snap-on charge | Custom |

For props: consider **magnetic pogo pin connectors** for charging — they look clean, are easy to connect in the dark, and can be waterproofed.

---

## 10. Design Decision Flowchart

```
How many LEDs at peak? 
├── ≤50 LEDs (~3A max) 
│   └── IP5306 module OR TPS61023 boost + TP4056 charger
│       └── Push button: IP5306 KEY pin or discrete soft latch
│
├── 50-150 LEDs (~3-5A)
│   └── TPS61022 boost + BQ24075 charger + LTC2954 button controller
│       └── OR: SW6106 all-in-one (if 3A is acceptable peak)
│
└── 150+ LEDs (>5A)
    └── Use 2S LiPo + buck converter (MP2315/TPS54531)
        └── Much more efficient at high current
        └── Push button: LTC2954 or ESP32 soft latch
```
