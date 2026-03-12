# Schematic Checklist — ESP32 LED Controller PCB

Detailed reference for schematic capture in KiCad. Read this file when the user asks about component selection, circuit block design, pin assignments, or schematic review.

---

## 1. Power Input Block

### USB-C Power Input
```
VBUS (5V) → PTC Fuse (500mA-2A, e.g., MF-MSMF050) → Schottky Diode (SS34/SS54) → +5V_USB
GND → GND
CC1 → 5.1kΩ → GND    (required for USB-C to negotiate 5V)
CC2 → 5.1kΩ → GND
D+ → 22Ω → USB_DP
D- → 22Ω → USB_DM
```
Without the 5.1kΩ resistors on CC1/CC2, USB-C power sources will not provide voltage.

### DC Barrel Jack / Screw Terminal (7-24V)
```
VIN → Reverse polarity protection (P-MOSFET or Schottky) → Bulk cap (100µF/25V electrolytic) → Buck Converter
```

### LiPo Battery Input
```
LiPo → TP4056 Module (with DW01 protection IC) → VBAT
VBAT → Boost Converter (MT3608, XL6009) → 5V
    or
VBAT → LDO (XC6220, MCP1700) → 3.3V (direct to ESP32, no LED power)
```

---

## 2. Voltage Regulation

### 5V Rail (for LEDs)

**Buck Converter (from >7V input):**
Recommended ICs: MP1584EN, LM2596S, MP2315, TPS5430

MP1584EN typical application:
```
VIN → C_in (22µF ceramic) → VIN pin
BST pin → 100nF → SW pin
SW pin → Inductor (4.7µH-10µH) → VOUT
FB pin → Resistor divider to set 5V output
EN pin → VIN (always on) or control signal
GND → solid ground plane
VOUT → C_out (22µF ceramic + 100µF electrolytic)
```

**Boost Converter (from LiPo 3.7V):**
MT3608 typical application:
```
VIN (3.7V) → C_in (22µF) → VIN pin
SW pin → Inductor (22µH) → Schottky diode (SS34) → VOUT
FB pin → Resistor divider for 5V
EN → VIN
VOUT → C_out (22µF)
```

### 3.3V Rail (for ESP32)

**LDO from 5V:**

AMS1117-3.3 (dropout ~1.1V, max 1A):
```
5V → C_in (10µF tantalum + 100nF ceramic) → VIN pin
VOUT pin → C_out (10µF tantalum + 100nF ceramic) → 3.3V
GND tab → thermal pad on PCB
```

ME6211C33 (dropout 100mV, max 500mA, SOT-23-5):
Better choice for battery-powered designs where 5V rail may sag.

XC6220B331MR (dropout 100mV, 1A, SOT-25):
Premium choice with very low dropout and low quiescent current (8µA).

**Key rule:** Always provide ≥500mA on the 3.3V rail for ESP32 (WiFi TX bursts draw 250-350mA).

---

## 3. ESP32 Module Circuit

### Power Decoupling (Critical)

For ESP32-WROOM-32E:
```
Pin 2 (3V3) → 10µF + 100nF to GND
Pin 1 (GND) → Ground plane
All exposed GND pads → Via-stitched to ground plane
```

For bare ESP32 chip (advanced):
```
VDD3P3_CPU (pin 37) → 100nF to GND
VDD3P3_RTC (pin 19) → 100nF to GND
VDDA (pin 1)        → 100nF + 1µF to GND (analog supply, keep noise-free)
VDD_SDIO (pin 26)   → 100nF + 4.7µF (1.8V or 3.3V mode per eFuse)
```

### Reset & Boot Circuit
```
EN (CHIP_PU):
  → 10kΩ pull-up to 3.3V
  → 100nF cap to GND (RC delay ~1ms for clean startup)
  → RESET button to GND (active low)
  → Optional: supervisor IC (e.g., MAX809) for brown-out reset

GPIO0 (Boot Mode):
  → 10kΩ pull-up to 3.3V (normal boot = high)
  → BOOT button to GND (hold during reset to enter download mode)
```

### Strapping Pins — Avoid Using These for LED Data

| Pin | Boot Function | Safe Default |
|-----|--------------|-------------|
| GPIO0 | Boot mode select | Pull-up 10kΩ (HIGH = normal boot) |
| GPIO2 | Must be LOW or floating for download | Leave floating or pull-down |
| GPIO12 (MTDI) | VDD_SDIO voltage select | Leave floating for 3.3V flash |
| GPIO15 (MTDO) | JTAG/silencing boot log | Pull-up 10kΩ (suppress boot log noise) |
| GPIO5 | SDIO timing | Default pull-up (internal) |

**Recommendation:** Use GPIO16, GPIO17, GPIO18, GPIO19, GPIO21, GPIO22, GPIO23, GPIO25, GPIO26, GPIO27, GPIO32, GPIO33 for LED data/clock lines. These are safe GPIOs with no boot-mode side effects.

### Crystal (bare chip only)
```
XTAL_P (pin 30) → 40MHz crystal → XTAL_N (pin 29)
Load capacitors: typically 10-22pF to GND (check crystal datasheet)
Keep traces short (<5mm), symmetric, and surrounded by ground guard
```
Note: WROOM modules have the crystal built-in. No external crystal needed.

### USB-UART Bridge (for non-native-USB modules)
```
CP2102N:
  VDD → 3.3V + 100nF + 4.7µF
  D+ → USB_DP (via 22Ω)
  D- → USB_DM (via 22Ω)
  TXD → ESP32 RXD0 (GPIO3)
  RXD → ESP32 TXD0 (GPIO1)
  DTR → 100nF → EN (auto-reset)
  RTS → 100nF → GPIO0 (auto-boot)
```
The DTR/RTS auto-reset circuit uses capacitor coupling so the programmer tool (esptool.py) can automatically put the ESP32 into download mode.

---

## 4. Level Shifter Circuit

### Option A: SN74HCT125 (Quad Buffer, recommended)
```
VCC → 5V + 100nF
GND → GND
OE (1OE, 2OE, etc.) → GND (always enabled)
Input (1A) → ESP32 GPIO (3.3V logic)
Output (1Y) → 330Ω → LED DIN
```
The HCT family accepts 3.3V as logic HIGH (V_IH = 2.0V). Output drives 5V rail. One IC handles 4 channels.

### Option B: 74LVC1T45 (Single-bit bidirectional)
```
VCCA → 3.3V + 100nF
VCCB → 5V + 100nF
DIR → VCCA (A→B direction)
A → ESP32 GPIO
B → 330Ω → LED DIN
```

### Option C: Single MOSFET (BSS138 / 2N7002)
```
3.3V → 10kΩ pull-up → Source (connected to ESP32 GPIO)
5V → 4.7kΩ pull-up → Drain (output to LED)
Gate → 3.3V
```
This is the lowest-cost option but has limited speed. Adequate for WS2812B at 800kHz but marginal for APA102 at high clock rates.

---

## 5. LED Output Circuit

### WS2812B Output
```
+5V → Thick trace / copper pour → LED strip VCC
GND → Thick trace / copper pour → LED strip GND
Level Shifter Output → 330Ω → DIN

Per-LED decoupling (on PCB-mounted LEDs):
  Each WS2812B: 100nF ceramic between VDD and VSS pads
  Place cap within 2mm of LED package
```

### APA102 Output
```
+5V → LED strip VCC
GND → LED strip GND
Level Shifter Data Output → 330Ω → SDI (data in)
Level Shifter Clock Output → 330Ω → CKI (clock in)
```

### Power Injection Points
For strips >30 LEDs, add power injection connectors:
```
Screw Terminal J_PWR1:
  Pin 1 → +5V rail (from regulated supply)
  Pin 2 → GND

Screw Terminal J_PWR2 (mid-strip injection):
  Pin 1 → +5V rail
  Pin 2 → GND
```
Size these connectors for the expected current (e.g., AWG18 for 5A+).

---

## 6. Protection Circuits

### ESD Protection
```
On all external connectors (USB, LED output, power input):
  TVS diode array (e.g., PRTR5V0U2X for USB, PESD5V0S1BA for data lines)
  or individual TVS diodes (SMAJ5.0A for 5V rail)
```

### Reverse Polarity Protection
```
P-MOSFET method (recommended, low loss):
  VIN → Source of P-FET → Drain → rest of circuit
  Gate → GND (via 100kΩ) + Zener (15V) for gate protection
  Body diode provides initial conduction path

Schottky diode method (simple, ~0.3V drop):
  VIN → Schottky (SS34) → rest of circuit
```

### Overcurrent Protection
```
PTC Resettable Fuse:
  5V rail: MF-MSMF200 (2A hold, 4A trip)
  or standard fuse for higher currents

Current limiting resistors on all GPIO-connected outputs
```

---

## 7. Schematic Review Checklist

Before moving to PCB layout, verify:

- [ ] All ESP32 power pins have decoupling caps (100nF + bulk)
- [ ] EN pin has RC circuit (10kΩ + 100nF)
- [ ] GPIO0 has pull-up and BOOT button
- [ ] No strapping pins used for LED data without understanding boot implications
- [ ] Level shifter present between ESP32 and LED data lines
- [ ] 330Ω series resistor on each LED data/clock line
- [ ] Power injection points for long LED chains
- [ ] USB CC resistors (5.1kΩ) if using USB-C
- [ ] Auto-reset circuit (DTR/RTS via caps) if using USB-UART bridge
- [ ] ESD protection on external connectors
- [ ] Reverse polarity protection on DC input
- [ ] PTC fuse or fuse on main power input
- [ ] PWR_FLAG placed on power nets (to suppress KiCad ERC warnings)
- [ ] Net labels consistent and meaningful
- [ ] All component values annotated (resistance, capacitance, voltage rating)
- [ ] Test points on key signals: 3.3V, 5V, GND, LED_DATA, UART TX/RX
- [ ] 3D models assigned to footprints for visual verification

---

## 8. Bill of Materials — Typical Components

### Core Components
| Component | Part Number | Package | Purpose |
|-----------|------------|---------|---------|
| ESP32 Module | ESP32-WROOM-32E | Module | MCU with WiFi/BT |
| LDO 3.3V | AMS1117-3.3 | SOT-223 | 3.3V regulation |
| Level Shifter | SN74HCT125N | DIP-14/SOIC-14 | 3.3V→5V logic |
| USB-UART | CP2102N | QFN-28 | USB serial |
| LED | WS2812B | 5050 | Addressable RGB |
| TVS Diode | PRTR5V0U2X | SOT-143B | USB ESD protection |
| Schottky | SS34 | SMA | Power protection |
| PTC Fuse | MF-MSMF110 | 1812 | Overcurrent |
| Crystal | 40MHz | 3.2×2.5mm | ESP32 clock (bare chip only) |

### Passive Components (Minimum Set)
| Value | Quantity | Package | Purpose |
|-------|----------|---------|---------|
| 100nF MLCC | 10+ | 0402/0603 | Decoupling |
| 10µF MLCC | 4+ | 0805 | Bulk decoupling |
| 100µF Electrolytic | 1-2 | 6.3×5.4mm | Power entry filter |
| 10kΩ | 5+ | 0402/0603 | Pull-ups |
| 5.1kΩ | 2 | 0402/0603 | USB-C CC |
| 330Ω | 2-4 | 0402/0603 | LED data series |
| 22Ω | 2 | 0402/0603 | USB data series |
