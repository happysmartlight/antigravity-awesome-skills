---
name: kicad-esp32-led-pcb
description: >
  KiCad PCB design for ESP32 + addressable LEDs (WS2812B, APA102, SK6812). Trigger for: KiCad schematic/layout/Gerber/BOM/DRC, ESP32 PCB design, power supply (LDO, buck, boost, LiPo charging, IP5306, SW6106, TPS61022), level shifters, antenna keep-out, LED driver circuits. Also trigger for portable LED props, cosplay electronics, push-button on/off power control (LTC2954, soft-latch), battery-powered high-current designs, and compact PCB layout. Vietnamese: thiet ke mach, mach in, mach nguon, mach dieu khien LED, layout PCB, dao cu LED, pin lipo, nut nhan bat tat. Use this skill even without explicit KiCad mention if discussing ESP32 hardware, PCB layout, or battery-powered LED projects.
---

# KiCad ESP32 LED PCB Design Skill

Expert guidance for designing ESP32-based PCBs that drive addressable LEDs (WS2812B, APA102, SK6812, WS2813, WS2815) using KiCad. Covers the full workflow from schematic capture to manufacturing-ready Gerbers.

## When to Read Reference Files

This SKILL.md covers the overall workflow and key decisions. For deep-dive details, read the appropriate reference file:

- **`references/schematic-checklist.md`** — Detailed component selection, circuit blocks, pin assignments, and schematic review checklist. Read this when the user asks about schematics, component selection, or circuit design.
- **`references/pcb-layout-guide.md`** — PCB layout rules, routing strategies, thermal management, DRC setup, antenna keep-out, and manufacturing output. Read this when the user asks about layout, routing, DRC errors, Gerber generation, or fabrication.
- **`references/smart-power-management.md`** — Smart power management for portable LED props: LiPo battery management, high-current boost converters (TPS61022, TPS61032), all-in-one power bank ICs (IP5306, SW6106, IP5328P), push-button on/off controllers (LTC2954, soft-latch circuits), charging ICs, battery protection, and compact PCB layout for props. **Read this first** when the user asks about battery-powered LED projects, push-button power control, portable/wearable electronics, or high-current boost from LiPo.

---

## Project Architecture Overview

A typical ESP32 LED controller PCB has these functional blocks:

1. **Power Input** — USB-C, DC barrel jack, screw terminal, or battery (LiPo via TP4056)
2. **Voltage Regulation** — 5V rail for LEDs + 3.3V rail for ESP32
3. **ESP32 Module** — WROOM-32E, S3-WROOM, or C3-MINI with supporting circuits
4. **Level Shifting** — 3.3V→5V logic for LED data lines (critical for WS2812B)
5. **LED Output** — Connectors for LED strips/matrices with power injection points
6. **Protection** — ESD, reverse polarity, overcurrent (fuses/PTC)
7. **Programming/Debug** — USB-UART bridge or native USB, BOOT/RESET buttons
8. **Smart Power Control** — Push-button on/off with long-press shutdown (for portable/prop applications)

## Portable LED Prop Design

For battery-powered LED props (cosplay, stage, art), the architecture shifts significantly. Read `references/smart-power-management.md` for full details.

### Key Design Challenges

- **High current from low voltage:** A single 3.7V LiPo boosting to 5V/3A means ~4.5A from the battery. At 5A output, battery current exceeds 9A at low charge. This demands careful inductor selection, thick traces, and proper battery protection.
- **Smart on/off:** Props need a single push button that powers on with a short press and requires a deliberate long-press (2-5 seconds) to power off — preventing accidental shutdown during performance.
- **Compact form factor:** The PCB must fit inside the prop. Consider 2-layer or 4-layer boards in the 25×50mm range.
- **Charge-in-place:** USB-C charging without disassembly. Use a power bank IC or charger with power-path management.

### Quick Architecture Selection

| LED Count | Peak Current | Recommended Approach |
|-----------|-------------|---------------------|
| ≤50 | ≤3A | IP5306 module (built-in push button, charger, boost) |
| 50-150 | 3-5A | TPS61022 boost + LTC2954 button + TP4056/BQ24075 charger |
| 150+ | >5A | 2S LiPo pack + buck converter (much more efficient) |

### Smart Push Button Options

| Solution | Complexity | Off Timer | µP Integration | Cost |
|----------|-----------|-----------|---------------|------|
| IP5306 KEY pin | Very low | Double-press | I2C (some variants) | $0.50 |
| LTC2954 | Medium | 1-32s (adjustable R) | INT + KILL pins | $2-3 |
| DIY soft latch (MOSFET + ESP32) | Low | Firmware-controlled | Full control | $0.20 |
| TPS3422 | Low | Cap-adjustable | Reset output | $1 |

The **LTC2954** is the professional choice: adjustable on/off timers via resistors (PDT pin sets long-press duration, e.g., 3.3MΩ = 3.3s to turn off), INT pin notifies ESP32 for graceful shutdown, and KILL pin lets the ESP32 confirm power-off. The IC draws only 6µA in standby — negligible battery drain.

The **DIY soft latch** is the cheapest and most flexible: ESP32 detects the button press, decides when to shut down (any timing logic you want in firmware), and releases a GPIO that controls a P-MOSFET power switch. The tradeoff is that firmware bugs can prevent shutdown.

## Decision Framework

### Choosing the LED Type

| Feature | WS2812B | APA102 / SK9822 | SK6812 RGBW |
|---------|---------|-----------------|-------------|
| Protocol | Single-wire NZR (800kHz) | SPI (data + clock) | Single-wire NZR |
| Refresh Rate | ~400Hz | ~19kHz | ~400Hz (SK6812) |
| Timing Sensitivity | High (±150ns tolerance) | Low (clock-based) | High |
| ESP32 Pins Needed | 1 GPIO per strip | 2 GPIO (MOSI + CLK) | 1 GPIO per strip |
| Best For | General projects, cost-effective | POV displays, video, flicker-free | Warm white + RGB |
| Level Shifter? | Strongly recommended | Strongly recommended | Strongly recommended |

**Why level shifting matters:** ESP32 outputs 3.3V logic. WS2812B datasheet specifies V_IH (logic high threshold) = 0.7 × VDD = 3.5V at 5V supply. This means 3.3V is technically below spec. While many LEDs work at 3.3V in short runs, it is unreliable for production. Use a level shifter (SN74HCT125, SN74AHCT125, 74LVC1T45, or a single-MOSFET circuit with pull-up).

### Choosing the Power Architecture

The power design depends on input voltage and total LED current:

**Rule of thumb:** Each WS2812B pixel draws up to 60mA at full white (20mA × 3 colors). For 60 LEDs = 3.6A at 5V = 18W. Always design for at least 80% of theoretical max.

| Input Source | 5V Rail Solution | 3.3V Rail Solution |
|-------------|-----------------|-------------------|
| USB-C (5V) | Direct (with PTC fuse) | AMS1117-3.3 or ME6211 LDO |
| 12V DC adapter | Buck converter (MP1584, LM2596, MP2315) → 5V | LDO from 5V rail |
| 24V DC | Buck → 5V (ensure Vin rating) | LDO from 5V rail |
| LiPo 3.7V | Boost (MT3608) → 5V | LDO (XC6220, MCP1700) |
| LiPo + USB | TP4056 charge → Boost → 5V | LDO from 5V |

**Critical:** Never power ESP32 directly from 12V/24V. Always regulate down to 3.3V (operating range: 2.3V–3.6V). Espressif recommends 3.3V supply with ≥500mA output capability.

### ESP32 Module Selection

| Module | Flash | PSRAM | USB | Best For |
|--------|-------|-------|-----|----------|
| ESP32-WROOM-32E | 4/8/16MB | No | No (need CP2102/CH340) | Cost-effective, proven |
| ESP32-S3-WROOM-1 | 4/8/16MB | 2/8MB | Native USB-OTG | WLED, complex animations |
| ESP32-C3-MINI-1 | 4MB | No | Native USB-JTAG | Minimal BLE+WiFi, small form |

## Schematic Design Workflow

### Step 1: KiCad Setup

Install the Espressif KiCad library via Plugin & Content Manager (PCM):
1. Open KiCad → Plugin and Content Manager
2. Search "Espressif" → Install
3. Symbols appear as `PCM_Espressif:ESP32-WROOM-32E` etc.
4. Footprints follow Espressif's recommended land patterns from datasheets

For addressable LED footprints (WS2812B-5050, APA102-5050), use the built-in KiCad library or create custom ones matching the LED datasheet. The standard `LED_SMD` library has `LED_WS2812B_PLCC4_5.0x5.0mm_P3.2mm`.

### Step 2: Power Supply Schematic

Read `references/schematic-checklist.md` for detailed circuit blocks and component values.

Key principles:
- Place 100nF ceramic (MLCC) decoupling caps on every power pin, as close as possible
- Add 10µF+ bulk capacitors at power entry points
- ESP32 requires decoupling on: VDD3P3_CPU, VDD3P3_RTC, VDD_SDIO, VDDA (analog)
- For LEDs: place a 100nF cap per LED (on the PCB near each LED's VCC/GND pads) — this is essential for signal integrity on long chains
- Separate analog and digital grounds, connect at a single star point

### Step 3: ESP32 Supporting Circuits

Essential circuits around the ESP32 module:
- **EN (CHIP_PU) pin:** 10kΩ pull-up to 3.3V + 100nF cap to GND (RC delay for stable boot)
- **GPIO0 (boot mode):** 10kΩ pull-up to 3.3V + BOOT button to GND
- **RESET button:** connected to EN pin through a button to GND
- **USB-UART bridge** (if no native USB): CP2102N or CH340C, with 22Ω series resistors on D+/D−
- **Strapping pins:** Be careful with GPIO0, GPIO2, GPIO12, GPIO15 — check Espressif hardware design guidelines for boot mode configuration

### Step 4: LED Driver Output

For WS2812B (single-wire):
```
ESP32 GPIO → Level Shifter → 330Ω series resistor → DIN of first LED
```
The 330Ω resistor dampens reflections on the data line. Some designs use 470Ω. Place it close to the LED, not the MCU.

For APA102 (SPI):
```
ESP32 MOSI → Level Shifter → 330Ω → Data In
ESP32 CLK  → Level Shifter → 330Ω → Clock In
```
APA102 can use hardware SPI (VSPI/HSPI) for maximum throughput.

### Step 5: Connectors & Power Injection

For strips longer than ~30 LEDs, inject 5V power at multiple points (every 30-50 LEDs) to prevent voltage drop and color shift. Use thick traces or dedicated power bus (≥2mm trace width for 3A, or use copper pours).

Typical connector options:
- **JST-SM 3-pin** (5V, GND, Data) — standard for LED strips
- **Screw terminals** — for high-current power injection
- **JST-XH** — for internal board-to-board connections

## PCB Layout Workflow

Read `references/pcb-layout-guide.md` for detailed routing rules and DRC setup.

### Critical Layout Rules

1. **Antenna keep-out zone:** No copper (traces, pours, components) within the antenna area of the ESP32 module. Maintain ≥15mm clearance. The antenna must extend beyond the board edge or have a clear ground-free zone.

2. **Decoupling capacitor placement:** Place 100nF caps within 3mm of ESP32 power pins. Shorter traces = better high-frequency noise suppression.

3. **Ground plane:** Use a solid ground plane on the bottom layer (2-layer board) or on an inner layer (4-layer). Pour ground copper on both layers and stitch with vias every 3-5mm.

4. **LED data line routing:** Keep data lines short and direct. Avoid running data lines parallel to high-current power traces. Use ground guards if possible.

5. **Power trace sizing:** Use the IPC-2152 calculator or KiCad's built-in trace width calculator. For external layers at 1oz copper: ~0.5mm per 1A as a starting point (varies with temperature rise requirements).

6. **Thermal relief on ground pads:** Use thermal relief connections (not solid fills) for through-hole component ground pads to allow hand soldering.

### Layer Stack Recommendation

For most ESP32 LED controller boards, a **2-layer board** is sufficient:
- **Top (F.Cu):** Components, signal routing, LED data traces
- **Bottom (B.Cu):** Ground plane (as continuous as possible), some routing

For complex designs (many IOs, tight spacing, RF-sensitive):
- **4-layer:** Signal / GND / Power / Signal

### Manufacturing Output

1. Run DRC (Design Rule Check) — fix ALL errors, review warnings
2. Run ERC (Electrical Rule Check) — fix unconnected nets, missing power flags
3. Generate Gerbers: F.Cu, B.Cu, F.SilkS, B.SilkS, F.Mask, B.Mask, Edge.Cuts
4. Generate drill files (Excellon format)
5. Generate BOM (from schematic) and pick-and-place / position file (CPL)
6. Review in KiCad's Gerber Viewer or an online viewer before ordering
7. Common fab houses: JLCPCB, PCBWay, ALLPCB, OSHPark

### DRC Settings for Common Fab Houses (JLCPCB Standard)

| Parameter | Value |
|-----------|-------|
| Min trace width | 0.127mm (5mil) |
| Min clearance | 0.127mm (5mil) |
| Min via drill | 0.3mm |
| Min via annular ring | 0.15mm (via diameter ≥ 0.6mm) |
| Min hole-to-hole | 0.5mm |
| Min solder mask clearance | 0.05mm |
| Board edge clearance | 0.3mm |

Import these as a DRC template in KiCad: File → Board Setup → Import Settings from Another Board.

## Common Pitfalls

1. **Forgetting the level shifter** — LEDs work intermittently or not at all on long runs
2. **Insufficient power bus** — Voltage drop causes LEDs at the end of the chain to appear dim or wrong color
3. **No decoupling on LEDs** — Signal glitches cause flickering or random colors
4. **Antenna blocked by ground pour** — WiFi range drops dramatically
5. **Wrong boot strapping pins** — ESP32 won't enter normal boot mode
6. **Undersized power traces** — Traces heat up, potentially delaminating the PCB
7. **Missing ESD protection** — Connector pins exposed to static discharge
8. **No series resistor on data line** — Signal reflections corrupt the first LED's data
9. **USB differential pair not matched** — Unreliable programming/serial connection
10. **Ground loop between power injection points** — Creates noise, solve with star grounding

## Firmware Compatibility Notes

The PCB design should support popular firmware options:
- **WLED** — Most popular for addressable LEDs, supports WS2812B/APA102/SK6812 via configurable GPIO
- **ESP-IDF (native C/C++)** — For custom firmware, use RMT peripheral for WS2812B or SPI for APA102
- **FastLED / NeoPixelBus** — Arduino framework libraries, widely supported
- **ESPHome** — YAML-based, great for Home Assistant integration

When designing the PCB, expose the GPIO pins that these firmware options commonly use and provide test points for debugging.

## Quick Reference: Espressif Resources

- Hardware Design Guidelines: https://docs.espressif.com/projects/esp-hardware-design-guidelines/
- KiCad Library (PCM): https://github.com/espressif/kicad-libraries
- ESP32 Datasheet & Technical Reference Manual: https://www.espressif.com/en/support/documents/technical-documents
