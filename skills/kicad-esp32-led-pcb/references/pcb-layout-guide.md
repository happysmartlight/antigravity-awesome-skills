# PCB Layout Guide — ESP32 LED Controller

Detailed reference for PCB layout, routing, DRC configuration, and manufacturing output in KiCad. Read this when the user asks about layout placement, trace routing, DRC/ERC errors, antenna design, thermal management, or Gerber generation.

---

## 1. Board Setup in KiCad

### Layer Configuration (2-Layer Standard)

```
Board Setup → Board Stackup:
  F.Cu    — Top copper (components, signals)
  B.Cu    — Bottom copper (ground plane, routing)
  F.SilkS — Top silkscreen (component labels)
  B.SilkS — Bottom silkscreen
  F.Mask  — Top solder mask
  B.Mask  — Bottom solder mask
  Edge.Cuts — Board outline
```

For 4-layer (advanced):
```
  F.Cu    — Signal + components
  In1.Cu  — Ground plane (continuous)
  In2.Cu  — Power plane (5V, 3.3V zones)
  B.Cu    — Signal + some components
```

### Design Rules (JLCPCB Standard Process)

Configure in Board Setup → Design Rules → Constraints:

```
Minimum clearance:         0.15mm (6mil)
Minimum track width:       0.15mm (6mil)  [use 0.2mm+ for reliability]
Minimum via drill:         0.3mm
Minimum via diameter:      0.6mm (annular ring ≥0.15mm)
Minimum through-hole:      0.3mm drill
Copper to edge clearance:  0.3mm
Silkscreen line width:     0.15mm minimum
Silkscreen text height:    0.8mm minimum (1.0mm recommended for readability)
Solder mask expansion:     0.05mm
Solder mask min width:     0.1mm
```

### Net Classes

Configure in Board Setup → Net Classes:

| Net Class | Track Width | Clearance | Via Drill | Via Diameter |
|-----------|------------|-----------|-----------|-------------|
| Default | 0.25mm | 0.15mm | 0.3mm | 0.6mm |
| Power_5V | 1.0mm+ | 0.2mm | 0.4mm | 0.8mm |
| Power_3V3 | 0.5mm | 0.2mm | 0.3mm | 0.6mm |
| LED_Data | 0.3mm | 0.2mm | 0.3mm | 0.6mm |
| USB_DP/DM | 0.3mm | 0.15mm | 0.3mm | 0.6mm |

Assign nets to classes in Board Setup → Net Classes → Net Class Assignments.

---

## 2. Component Placement Strategy

### Placement Priority Order

1. **ESP32 module** — Place first, centered or offset to allow antenna clearance
2. **USB connector** — Edge-mounted, aligned with board edge
3. **Power components** — Group regulator + inductors + caps together, away from RF
4. **Crystal** (if bare chip) — Within 5mm of ESP32 crystal pins
5. **Level shifters** — Between ESP32 and LED output connectors
6. **LED connectors** — Board edge, close to level shifter output
7. **Buttons** (BOOT, RESET) — Accessible edge location
8. **Decoupling caps** — Immediately adjacent to their associated IC power pins
9. **Protection components** — Near connectors they protect

### ESP32 Module Placement Rules

**Antenna orientation:** The antenna portion of the WROOM module must extend beyond the PCB edge OR have a keep-out zone. Position the module so the antenna side is at a board edge with no copper underneath or nearby.

```
┌──────────────────────────────┐
│                              │
│   [Components]   [ESP32    ] │
│                  [WROOM    ] │
│   [Power]        [........]◄── Antenna at board edge
│                  [........] │
│   [LED Output]              │
│                              │
└──────────────────────────────┘
```

**Ground pad:** The WROOM module has a large exposed ground pad (GND) underneath. Connect this to the ground plane with a grid of thermal vias:
- Via pattern: 3×3 or 4×4 grid
- Via drill: 0.3mm
- Via spacing: ~1mm apart
- Connect to ground plane on B.Cu

### Power Component Grouping

Keep all components of one power stage together:
```
[Input cap] → [Regulator IC] → [Inductor] → [Output cap]
                                              ↓
                                         [Feedback divider]
```

Route the power loop (input → switch → inductor → output → return through ground) as tightly as possible. The area enclosed by this loop is the source of switching noise EMI. Minimize it.

---

## 3. Routing Guidelines

### Trace Width Calculator

Use KiCad's built-in calculator: Inspect → Board Statistics, or the standalone PCB Calculator tool.

Quick reference for external layers (1oz copper, 10°C rise, ambient 25°C):

| Current (A) | Min Width (mm) | Recommended Width (mm) |
|-------------|---------------|----------------------|
| 0.5 | 0.25 | 0.4 |
| 1.0 | 0.5 | 0.8 |
| 2.0 | 1.0 | 1.5 |
| 3.0 | 1.5 | 2.0 |
| 5.0 | 2.5 | 3.5 |

For internal layers, widths need to be ~2× wider due to reduced heat dissipation.

### Signal Routing Rules

**LED Data Lines (WS2812B / APA102):**
- Keep data traces as short as possible
- Avoid routing parallel to power traces (crosstalk)
- Use 0.3mm width minimum
- Route directly from level shifter output to connector
- Place 330Ω series resistor within 5mm of the connector/first LED
- If routing on both layers, keep via count to minimum (each via adds ~0.5nH inductance)

**USB Differential Pair:**
- Route D+ and D- as a 90Ω differential pair
- Keep traces equal length (length matching within 0.15mm)
- Maintain consistent spacing between the pair
- Place 22Ω series resistors close to the ESP32/CP2102 USB pins
- Avoid vias in the differential pair if possible
- Do not route under or near switching power supply inductors

**Power Traces:**
- Use copper pours for high-current paths (5V LED power)
- Minimum 1mm width for 1A, scale linearly
- Place bulk capacitors at power entry and distribution nodes
- Use wide traces or pours from regulator output to distribution points

**SPI (for APA102):**
- Route CLK and MOSI with matched lengths if running >4MHz
- Keep traces short (<50mm total)
- Place series resistors near the source (ESP32 side)

### Ground Plane

**2-layer board:**
- Dedicate B.Cu primarily as a ground plane
- Pour copper fill on B.Cu assigned to GND net
- Keep the ground plane as continuous as possible — avoid splitting it with signal traces
- Where you must route on B.Cu, keep the trace narrow and bridge the ground plane split with nearby vias

**Ground stitching:**
- Place GND vias around the board perimeter every 3-5mm
- Place GND vias near all decoupling capacitors
- Place GND vias along high-speed signal traces for return current paths

**Analog/Digital ground separation:**
- For the ESP32's ADC pins, route the analog ground return separately from the digital ground
- Connect analog and digital grounds at a single point near the ESP32's VDDA pin

---

## 4. Antenna Keep-Out Zone

This is one of the most critical aspects of ESP32 PCB design. Getting it wrong severely degrades WiFi/Bluetooth range.

### WROOM Module Antenna

The ESP32-WROOM-32E has a meandering inverted-F antenna (MIFA) printed on the module PCB. The antenna area is the last ~7mm of the module length.

**Keep-out rules:**
- No copper (traces, pours, ground plane) on ANY layer directly beneath the antenna
- No copper within 15mm on either side of the antenna (on the same layer)
- No components within the antenna projection area
- The antenna should protrude beyond the main board edge, or have a generous cut-out in the ground plane beneath it
- No metal enclosure, battery, or wiring near the antenna

**In KiCad:**
1. Draw a keep-out zone on F.Cu and B.Cu layers in the antenna area
2. Use "Rule Area" (right-click → Add Rule Area) with "Keep out copper fill" checked
3. Verify in 3D viewer that no copper is under the antenna

### U.FL / IPEX Antenna Variant (WROOM-32UE)

If using an external antenna variant:
- Route a 50Ω microstrip from the module's RF pad to the U.FL connector
- Use a coplanar waveguide with ground (CPWG) structure
- Keep the RF trace short and straight
- Place the U.FL connector at the board edge

---

## 5. Copper Zones (Fills)

### Creating Ground Plane

1. Select B.Cu layer
2. Draw a copper zone (Add Filled Zone tool, or press 'B' then click)
3. Set net to GND
4. Set zone properties:
   - Clearance: 0.2mm
   - Min width: 0.2mm
   - Pad connections: Thermal relief (for hand soldering) or Solid (for reflow)
   - Thermal relief gap: 0.25mm
   - Thermal relief spoke width: 0.25mm
5. Draw the zone outline covering the entire board
6. Press 'B' to refill zones after routing changes

### Power Zones (5V distribution)

For high-current LED power:
1. On F.Cu, draw a copper zone for the +5V net
2. Limit it to the area between the regulator output and LED connectors
3. This provides a low-impedance path for LED current
4. Stitch with vias if you also have 5V on B.Cu

---

## 6. DRC Troubleshooting

### Common DRC Errors and Fixes

**"Clearance violation"**
- Trace too close to another trace, pad, or zone
- Fix: Reroute with more spacing, or adjust the trace manually
- Check: Are your design rules too tight for the fab?

**"Track width too small"**
- A trace is narrower than the minimum
- Fix: Re-route with the correct width, check net class assignments

**"Via too small / drill too small"**
- Via dimensions don't meet minimum manufacturing specs
- Fix: Update via settings in Board Setup → Design Rules

**"Unconnected items"**
- Ratsnest lines remain — nets not fully routed
- Fix: Route the remaining connections, or check if schematic is correct

**"Pad near pad / Pad too close"**
- Component pads violate clearance
- Fix: Move components apart, or check if footprint is correct

**"Courtyard overlap"**
- Two component courtyards intersect
- Fix: Move components apart — courtyards define physical clearance for assembly

**"Board has malformed outline"**
- Edge.Cuts layer has gaps or intersections
- Fix: Ensure the board outline is a single closed shape

**"Silkscreen overlap"**
- Silkscreen text overlaps pads or other silkscreen
- Fix: Move or resize text, or place on the other side

**"Thermal relief incomplete"**
- Pad in a zone has fewer than the required spoke connections
- Fix: Adjust zone fill settings, or manually add a trace to connect

### ERC Common Warnings

**"Power pin not driven"**
- A VCC or +5V net has no power source
- Fix: Add a PWR_FLAG symbol on that net in the schematic

**"Pin not connected"**
- An IC pin is floating
- Fix: Either connect it or mark as "Not Connected" (place a no-connect flag ×)

---

## 7. Manufacturing Output

### Gerber Generation

File → Fabrication Outputs → Gerbers (.gbr):

| Layer | File Suffix | Purpose |
|-------|------------|---------|
| F.Cu | .gtl | Top copper |
| B.Cu | .gbl | Bottom copper |
| F.SilkS | .gto | Top silkscreen |
| B.SilkS | .gbo | Bottom silkscreen |
| F.Mask | .gts | Top solder mask |
| B.Mask | .gbs | Bottom solder mask |
| Edge.Cuts | .gm1 | Board outline |
| F.Paste | .gtp | Top paste (for stencil) |
| B.Paste | .gbp | Bottom paste |

Settings:
- Format: Gerber X2 (or 4.6 format for compatibility)
- Use protel filename extensions: Yes (for JLCPCB compatibility)
- Generate drill file separately

### Drill File Generation

File → Fabrication Outputs → Drill Files (.drl):
- Format: Excellon
- Drill units: mm
- Zeros format: Decimal
- Merge PTH and NPTH: depends on fab (JLCPCB accepts merged)

### BOM Generation

From schematic editor: Tools → Edit Symbol Fields → Export as CSV
Or use a BOM plugin: Tools → Generate BOM

Essential columns:
- Reference (R1, C1, U1...)
- Value (10kΩ, 100nF, ESP32-WROOM-32E...)
- Footprint (0402, SOT-223...)
- Quantity
- Manufacturer Part Number (for JLCPCB SMT assembly)

### Pick-and-Place / CPL File

File → Fabrication Outputs → Component Placement (.pos):
- Format: CSV
- Side: Both
- Units: mm

**Important:** JLCPCB and other fabs may have different rotation conventions than KiCad. Always verify component rotations in the online viewer after uploading. Common issues:
- Diodes, ICs, and polarized capacitors often have wrong rotations
- Manually correct rotations in the CPL file or in the fab's online editor

### Using the KiCad Fabrication Toolkit Plugin (Optional)

The "Fabrication Toolkit" plugin generates all output files (Gerbers, drill, BOM, CPL) in one click with the correct settings for popular fabs. Install via KiCad PCM.

---

## 8. Pre-Fabrication Checklist

Run through this before placing the order:

- [ ] DRC passes with 0 errors (warnings reviewed individually)
- [ ] ERC passes with 0 errors
- [ ] All ratsnest lines cleared (no unconnected nets)
- [ ] Board outline is a closed shape on Edge.Cuts
- [ ] Mounting holes added (M3 = 3.2mm drill, with GND or unconnected pad)
- [ ] Fiducials placed (for SMT assembly, at least 2-3 fiducial marks)
- [ ] Silkscreen readable: component references, board name, revision, date
- [ ] Polarity marks on diodes, electrolytic caps, ICs
- [ ] Test points accessible (3.3V, 5V, GND, key signals)
- [ ] Antenna keep-out zone verified in 3D viewer
- [ ] No copper under antenna on any layer
- [ ] Ground stitching vias present around board perimeter
- [ ] ESP32 module ground pad has via matrix
- [ ] Decoupling caps placed within 3mm of IC power pins
- [ ] Power trace widths sufficient for expected current
- [ ] USB differential pair length-matched
- [ ] LED data line has series resistor near connector
- [ ] All footprints verified against actual component datasheets
- [ ] 3D model check — no collisions, connectors at correct positions
- [ ] Gerbers opened in external viewer (KiCad Gerber Viewer or online)
- [ ] Drill file verified — correct hole sizes and positions
- [ ] BOM cross-checked with fab house component availability
- [ ] CPL file rotations verified in fab's online viewer

---

## 9. Thermal Considerations

### LED Power Dissipation

For boards with WS2812B LEDs mounted directly on the PCB:
- Each LED dissipates ~0.3W at full white
- 10 LEDs = 3W, 64 LEDs (8×8 matrix) = 19.2W
- Use copper pours on both layers connected to LED ground pads
- Add thermal vias under LED pads for heat transfer to the back layer
- Consider aluminum-core PCB for high-density LED matrices

### Voltage Regulator Cooling

LDO thermal dissipation: P = (Vin - Vout) × Iload
Example: AMS1117-3.3 with Vin=5V, Iload=500mA → P = (5-3.3) × 0.5 = 0.85W

Provide:
- Copper pour on the thermal/ground pad (minimum 20×20mm)
- Thermal vias (array of 0.3mm vias) connecting to B.Cu ground plane
- Check SOT-223 tab connection — it's usually the output pin, verify per datasheet

### Buck Converter

- Keep the switching loop area (Vin → IC → inductor → Cout → GND back to IC) minimal
- Place input and output capacitors as close as possible to the IC
- Use thermal vias under the IC's exposed pad
- Route the feedback trace away from the inductor and switching node
