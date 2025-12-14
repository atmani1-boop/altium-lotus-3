# LiDAR ToF 905nm System - Complete Schematic Documentation

This document provides detailed textual schematics for the LiDAR Time-of-Flight system integration.

## Table of Contents

1. [Page 1: Power Supply](#page-1-power-supply)
2. [Page 2: Laser Driver](#page-2-laser-driver)
3. [Page 3: Analog Reception](#page-3-analog-reception)
4. [Page 4: TDC7200 & ESP32-C6](#page-4-tdc7200--esp32-c6)
5. [Page 5: Sensors & DALI](#page-5-sensors--dali)
6. [Page 6: Energy Buffer](#page-6-energy-buffer)
7. [Page 7: Lighting Outputs](#page-7-lighting-outputs)
8. [Page 8: Layout Guidelines & Integration Notes](#page-8-layout-guidelines--integration-notes)

---

## Page 1: Power Supply

### Block Description
Converts Zhaga-18 24VDC input to regulated 5V and 3.3V rails with protection and filtering.

### Component List

| Designator | Part Number | Description | Value/Rating | Package |
|-----------|-------------|-------------|--------------|---------|
| J1 | Zhaga-18 | Connector for 24VDC input | 24V, 2A | Zhaga-18 |
| F1 | Littelfuse RXEF050 | PTC resettable fuse | 500mA hold | 1206 |
| D_TVS1 | SMAJ24A | TVS diode for ESD/surge | 24V, 400W | DO-214AC |
| C1 | Generic | Input bulk capacitor | 47µF, 50V, X7R | 1210 |
| C2 | Generic | Input filter capacitor | 10µF, 50V, X7R | 0805 |
| L1 | Würth 744773110 | Inductor for 5V buck | 10µH, 2.8A | 5.8x5.2mm |
| U1 | TPS54231 | 24V→5V buck converter | 3.5-28Vin, 2A | SOIC-8 |
| C3 | Generic | U1 input capacitor | 10µF, 50V, X7R | 0805 |
| C4 | Generic | U1 bootstrap capacitor | 0.1µF, 50V, X7R | 0603 |
| R1 | Generic | U1 feedback upper | 100kΩ, 1% | 0603 |
| R2 | Generic | U1 feedback lower | 22kΩ, 1% | 0603 |
| C5 | Generic | 5V output bulk | 100µF, 10V, X7R | 1210 |
| C6 | Generic | 5V output filter | 10µF, 10V, X7R | 0805 |
| L2 | Würth 744773047 | Inductor for 3.3V buck | 4.7µH, 1.5A | 5.8x5.2mm |
| U2 | TPS62840 | 24V→3.3V buck converter | 3-17Vin, 750mA | VSON-10 |
| C7 | Generic | U2 input capacitor | 10µF, 50V, X7R | 0805 |
| R3 | Generic | U2 feedback upper | 680kΩ, 1% | 0603 |
| R4 | Generic | U2 feedback lower | 200kΩ, 1% | 0603 |
| C8 | Generic | 3.3V output bulk | 47µF, 6.3V, X7R | 0805 |
| C9 | Generic | 3.3V output filter | 10µF, 6.3V, X7R | 0603 |

### Net Connections

```
24V_IN (J1.1) → F1.1
F1.2 → D_TVS1.1 (anode)
F1.2 → C1+ → C2+ → U1.VIN (pin 1)
D_TVS1.2 (cathode) → GND

U1 (TPS54231):
  Pin 1 (VIN) → 24V_FILT (via C3+)
  Pin 2 (GND) → GND
  Pin 3 (SW) → L1.1
  Pin 4 (BOOT) → C4+ → L1.1
  Pin 5 (FB) → R1 (to 5V_OUT) and R2 (to GND)
  Pin 6 (EN) → 24V_FILT (via 100k pullup)
  Pin 7 (SS) → GND (via 10nF)
  Pin 8 (NC)
  
L1.2 → 5V_OUT → C5+ → C6+ → distributed to system

U2 (TPS62840):
  Pin 1 (SW) → L2.1
  Pin 2 (GND) → GND
  Pin 3 (FB) → R3 (to 3V3_OUT) and R4 (to GND)
  Pin 4 (EN) → 5V_OUT (enable from 5V rail)
  Pin 5 (PG) → LED indicator (optional)
  Pin 6 (VIN) → 24V_FILT (via C7+)
  Pin 7 (NC)
  Pin 8 (NC)
  Pin 9 (GND) → GND
  Pin 10 (GND) → GND
  
L2.2 → 3V3_OUT → C8+ → C9+ → distributed to system
```

### Layout Notes
- **Ground plane**: Solid GND plane on layer 2
- **Power plane**: 5V and 3.3V pours on layer 3
- **Trace width**: 24V_IN minimum 30 mil, 5V minimum 40 mil, 3.3V minimum 30 mil
- **TVS placement**: D_TVS1 as close as possible to J1
- **Inductor orientation**: Keep magnetic field parallel to PCB surface
- **Capacitor placement**: Input caps within 5mm of IC VIN pins, output caps within 10mm
- **Thermal vias**: 4x vias under U1/U2 thermal pads to inner ground plane

### Test Points
- TP1: 24V_IN
- TP2: 5V_OUT
- TP3: 3V3_OUT
- TP4: GND

---

## Page 2: Laser Driver

### Block Description
High-speed gate driver, MOSFET switch, and 905nm laser diode with current sensing for ToF pulse generation.

### Component List

| Designator | Part Number | Description | Value/Rating | Package |
|-----------|-------------|-------------|--------------|---------|
| U3 | UCC27321 | High-speed MOSFET driver | 9A peak, 4.5V-18V | SOT-23-5 |
| Q1 | BSC340N08NS | N-channel MOSFET | 80V, 17A, 3.4mΩ | PG-TDSON-8 |
| D1 | OSRAM SPL S1L90A_3 | 905nm laser diode | 75W peak, 1.6V @ 75A | SMD |
| R_SENSE | WSL2512R1000FEA | Current sense resistor | 0.1Ω, 1%, 1W | 2512 |
| R5 | Generic | Gate pulldown | 10kΩ | 0603 |
| R6 | Generic | Current limit resistor | 22Ω | 0603 |
| C10 | Generic | Driver bypass | 0.1µF, 25V, X7R | 0603 |
| C11 | Generic | Laser supply decoupling | 100µF, 10V, X5R | 1210 |
| C12 | Generic | High-freq decoupling | 10µF, 10V, X7R | 0805 |
| C13 | Generic | HF ceramic cap | 0.1µF, 10V, X7R | 0603 |

### Net Connections

```
5V_OUT → U3.VDD (pin 2)
U3.GND (pin 3) → GND

LASER_TRIG (from ESP32) → R6 → U3.IN (pin 4)
U3.OUT (pin 5) → Q1.GATE (pin 4)
Q1.GATE → R5 → GND (pulldown)

5V_OUT → C11+ → C12+ → C13+ → Q1.DRAIN (pins 5,6,7,8)
Q1.SOURCE (pin 1) → R_SENSE.1
R_SENSE.2 → D1.CATHODE (pin 1)
D1.ANODE (pin 2) → GND

I_SENSE_P (to ADC) → R_SENSE.1
I_SENSE_N (to ADC) → R_SENSE.2
```

### Timing Characteristics
- Pulse width: 10-50ns (configurable)
- Rise time: <5ns
- Fall time: <5ns
- Repetition rate: Up to 100kHz
- Peak current: 75A (limited by laser diode)
- Average current: <100mA (1% duty cycle max)

### Layout Notes
- **Critical traces**: Keep LASER_TRIG to U3.IN < 50mm with ground guard traces
- **Gate drive**: Q1.GATE trace < 20mm from U3.OUT, width 10 mil minimum
- **Power loop**: Minimize loop area C11→Q1.DRAIN→D1→R_SENSE→GND
- **Current sense**: Kelvin connection for R_SENSE, differential routing to ADC
- **Thermal management**: Q1 thermal pad to GND with 6x thermal vias
- **Ground plane**: Continuous under entire laser driver section
- **Clearance**: 1mm clearance around laser diode for optical assembly

### Test Points
- TP5: Q1.GATE
- TP6: I_SENSE_P
- TP7: I_SENSE_N
- TP8: LASER_TRIG

### Safety Notes
- IEC 60825 Class 1M compliance requires proper optical design
- Laser pulse energy: <0.5µJ per pulse (eye-safe with diffuser)
- Interlock circuit recommended for production

---

## Page 3: Analog Reception

### Block Description
Photodiode receiver with transimpedance amplifier and comparator for ToF signal detection.

### Component List

| Designator | Part Number | Description | Value/Rating | Package |
|-----------|-------------|-------------|--------------|---------|
| D2 | Hamamatsu S5973 | PIN photodiode | 905nm optimized, 0.65A/W | SMD |
| U4 | OPA857 | Transimpedance amplifier | 1.5GHz GBW, low noise | SOT-23-5 |
| U5 | TLV3502 | Dual comparator | 4.5ns prop delay | SOIC-8 |
| R7 | Generic | TIA feedback resistor | 10kΩ, 1% | 0603 |
| C14 | Generic | TIA feedback capacitor | 0.5pF, C0G | 0402 |
| C15 | Generic | U4 power supply decoupling | 0.1µF, 10V, X7R | 0603 |
| C16 | Generic | U4 HF decoupling | 10nF, 10V, X7R | 0402 |
| R8 | Generic | Photodiode load | 1kΩ | 0603 |
| R9 | Generic | Comparator threshold upper | 10kΩ, 1% | 0603 |
| R10 | Generic | Comparator threshold lower | 10kΩ, 1% | 0603 |
| R11 | Generic | Comparator hysteresis | 100kΩ | 0603 |
| C17 | Generic | U5 power supply decoupling | 0.1µF, 10V, X7R | 0603 |
| C18 | Generic | Threshold filter | 10nF, 10V, X7R | 0603 |

### Net Connections

```
D2 photodiode:
  ANODE → GND
  CATHODE → R8 → 5V_OUT (reverse bias)
  CATHODE → U4.IN- (pin 2)

U4 (OPA857) TIA configuration:
  Pin 1 (OUT) → R7 → Pin 2 (IN-)  (feedback)
  Pin 1 (OUT) → C14 → Pin 2 (IN-)  (compensation)
  Pin 2 (IN-) → D2.CATHODE
  Pin 3 (IN+) → VREF_MID (2.5V via resistor divider)
  Pin 4 (V-) → GND
  Pin 5 (V+) → 5V_OUT (via C15 and C16 to GND)
  
TIA_OUT (U4.OUT) → U5.IN+ (pin 3, comparator A)

U5 (TLV3502) dual comparator:
  Pin 1 (OUT_A) → STOP_SIGNAL (to TDC7200)
  Pin 2 (IN-_A) → VREF_THR (threshold from R9/R10 divider)
  Pin 3 (IN+_A) → TIA_OUT
  Pin 4 (GND) → GND
  Pin 5 (IN+_B) → NC (unused)
  Pin 6 (IN-_B) → NC (unused)
  Pin 7 (OUT_B) → NC (unused)
  Pin 8 (VCC) → 3V3_OUT (via C17)

VREF_MID generation:
  5V_OUT → R9 (10k) → VREF_THR → R10 (10k) → GND
  VREF_THR → C18 → GND

Hysteresis:
  U5.OUT_A → R11 → U5.IN+_A
```

### Signal Chain Performance
- Photodiode responsivity: 0.65 A/W @ 905nm
- TIA transimpedance gain: 10kΩ
- TIA bandwidth: ~160MHz (limited by C14)
- Noise equivalent power: ~10pW/√Hz
- Comparator threshold: 2.5V (adjustable via R9/R10)
- Comparator hysteresis: ~50mV (via R11)
- Propagation delay: <10ns total

### Layout Notes
- **Photodiode placement**: D2 positioned for optical alignment, 45° angle recommended
- **TIA critical path**: Minimize parasitic capacitance at U4.IN-, guard with GND
- **Feedback components**: R7, C14 within 3mm of U4, no vias in feedback path
- **Ground plane**: Solid GND under entire RX section, separate from digital GND
- **Power supply**: Star connection for analog 5V, separate from digital 5V
- **Shielding**: Optional copper pour around D2 and U4 to reduce EMI pickup
- **Optical window**: PCB cutout or transparent window for D2

### Test Points
- TP9: TIA_OUT
- TP10: VREF_THR
- TP11: STOP_SIGNAL
- TP12: D2.CATHODE

---

## Page 4: TDC7200 & ESP32-C6

### Block Description
Time-to-digital converter for ToF measurement and ESP32-C6 microcontroller for system control.

### Component List

| Designator | Part Number | Description | Value/Rating | Package |
|-----------|-------------|-------------|--------------|---------|
| U6 | TDC7200 | Time-to-digital converter | 55ps resolution | VQFN-16 |
| U7 | ESP32-C6-WROOM-1U | WiFi/BLE/Zigbee MCU | 160MHz RISC-V | Module |
| Y1 | Generic | TCXO for TDC7200 | 16MHz, ±1ppm | 7x5mm |
| C19 | Generic | TDC7200 VCC decoupling | 0.1µF, 10V, X7R | 0603 |
| C20 | Generic | TDC7200 HF decoupling | 10nF, 10V, X7R | 0402 |
| C21 | Generic | TCXO decoupling | 0.1µF, 10V, X7R | 0603 |
| R12 | Generic | SPI CS pullup | 10kΩ | 0603 |
| R13 | Generic | TDC ENABLE pulldown | 10kΩ | 0603 |
| R14 | Generic | ESP32 EN pullup | 10kΩ | 0603 |
| C22 | Generic | ESP32 EN filter | 0.1µF, 10V, X7R | 0603 |
| C23-C26 | Generic | ESP32 power decoupling | 0.1µF, 10V, X7R | 0603 |
| U8 | Generic | LDO for TDC7200 | 3.3V, 100mA | SOT-23-5 |

### Net Connections

```
U6 (TDC7200):
  Pin 1 (START) → LASER_TRIG (shared with laser driver)
  Pin 2 (STOP1) → STOP_SIGNAL (from comparator U5)
  Pin 3 (STOP2) → NC (not used)
  Pin 4 (TRIG1) → GND (internal trigger mode)
  Pin 5 (TRIG2) → GND
  Pin 6 (CS) → ESP32.GPIO10 (via R12 pullup to 3V3)
  Pin 7 (SCLK) → ESP32.GPIO6
  Pin 8 (MOSI) → ESP32.GPIO7
  Pin 9 (MISO) → ESP32.GPIO2
  Pin 10 (INT) → ESP32.GPIO3
  Pin 11 (ENABLE) → ESP32.GPIO4 (via R13 pulldown)
  Pin 12 (CLKSEL) → 3V3_OUT (select external clock)
  Pin 13 (CLOCK) → Y1.OUT
  Pin 14 (VCC) → 3V3_TDC (clean 3.3V from U8)
  Pin 15 (GND) → AGND
  Pin 16 (GND_PAD) → AGND (thermal pad)

Y1 (16MHz TCXO):
  VCC → 3V3_OUT (via C21)
  OUT → U6.CLOCK (pin 13)
  GND → AGND

U7 (ESP32-C6-WROOM-1U):
  Pin 1 (GND) → GND
  Pin 2 (3V3) → 3V3_OUT (via C23-C26)
  Pin 3 (EN) → R14 → 3V3_OUT, R14 → C22 → GND
  Pin 4 (GPIO0) → BOOT button (optional)
  Pin 5 (GPIO1) → UART TX (programming)
  Pin 6 (GPIO2) → TDC.MISO
  Pin 7 (GPIO3) → TDC.INT
  Pin 8 (GPIO4) → TDC.ENABLE
  Pin 9 (GPIO5) → I2C_SDA (sensors)
  Pin 10 (GPIO6) → TDC.SCLK
  Pin 11 (GPIO7) → TDC.MOSI
  Pin 12 (GPIO8) → DALI TX
  Pin 13 (GPIO9) → DALI RX
  Pin 14 (GPIO10) → TDC.CS
  Pin 15 (GPIO11) → I2C_SCL (sensors)
  Pin 16 (GPIO12) → LASER_TRIG
  Pin 17 (GPIO13) → BROWNOUT_INT
  Pin 18 (GPIO14) → STATUS_LED
  Pins 19-30 → Additional GPIO for outputs, ADC, etc.

SPI Bus (TDC7200):
  SCLK: GPIO6, max 20MHz
  MOSI: GPIO7
  MISO: GPIO2
  CS: GPIO10

I2C Bus (Sensors):
  SDA: GPIO5, 400kHz
  SCL: GPIO11
```

### Software Configuration
- TDC7200 measurement mode 1 (single stop)
- Clock frequency: 16MHz (62.5ns period)
- Time resolution: ~55ps (interpolated)
- Measurement range: 0-8ms
- Calibration: Two-point calibration every 10 measurements

### Layout Notes
- **Clock routing**: Y1 to U6.CLOCK, 50Ω controlled impedance, length < 30mm
- **SPI signals**: Equal length matching ±5mm, series termination at ESP32 end
- **Ground plane**: Separate AGND island for U6, connected via ferrite bead
- **Power supply**: Dedicated LDO (U8) for TDC7200, pi-filter before U6.VCC
- **Thermal pad**: U6 thermal pad to AGND with 4x vias
- **Crystal placement**: Y1 within 20mm of U6, guard traces around clock signal
- **EMI reduction**: GND pour around U6 and Y1

### Test Points
- TP13: TDC.START
- TP14: TDC.STOP1
- TP15: TDC.CLOCK
- TP16: TDC.CS

---

## Page 5: Sensors & DALI

### Block Description
Ambient light sensor (ALS) and DALI communication interface for D4i compliance.

### Component List

| Designator | Part Number | Description | Value/Rating | Package |
|-----------|-------------|-------------|--------------|---------|
| U9 | BH1750 or VEML7700 | Ambient light sensor | I2C, 1-65535 lux | 6-pin SMD |
| U10 | Generic | DALI transceiver IC | ISO compliant | SOIC-8 |
| R15 | Generic | I2C pullup SDA | 4.7kΩ | 0603 |
| R16 | Generic | I2C pullup SCL | 4.7kΩ | 0603 |
| R17 | Generic | DALI current limit | 220Ω | 0603 |
| R18 | Generic | DALI bias resistor | 10kΩ | 0603 |
| C27 | Generic | ALS power decoupling | 0.1µF, 10V, X7R | 0603 |
| D3 | Generic | DALI protection diode | 3.3V Zener | SOD-323 |
| D4 | Generic | DALI reverse protection | Schottky | SOD-323 |
| J2 | Generic | DALI connector | 2-pin terminal block | 3.5mm pitch |

### Net Connections

```
U9 (BH1750/VEML7700) Ambient Light Sensor:
  Pin 1 (VDD) → 3V3_OUT (via C27)
  Pin 2 (ADDR) → GND (I2C address 0x23)
  Pin 3 (GND) → GND
  Pin 4 (SCL) → I2C_SCL (ESP32.GPIO11, via R16 to 3V3)
  Pin 5 (SDA) → I2C_SDA (ESP32.GPIO5, via R15 to 3V3)

U10 (DALI Transceiver):
  Pin 1 (VCC) → 3V3_OUT
  Pin 2 (GND) → GND
  Pin 3 (TX) → ESP32.GPIO8 (DALI_TX)
  Pin 4 (RX) → ESP32.GPIO9 (DALI_RX)
  Pin 5 (DALI_A) → R17 → J2.1 (DALI line)
  Pin 6 (DALI_B) → J2.2 (DALI line)
  Pin 7 (NC)
  Pin 8 (NC)

DALI bus protection:
  J2.1 → D4.anode, D4.cathode → R17
  J2.1 → R18 → 3V3_OUT
  J2.1 → D3.cathode, D3.anode → GND (3.3V Zener clamp)

I2C bus pullups:
  3V3_OUT → R15 → I2C_SDA
  3V3_OUT → R16 → I2C_SCL
```

### I2C Device Addresses
- BH1750: 0x23 (ADDR=GND) or 0x5C (ADDR=VDD)
- VEML7700: 0x10 (fixed)
- Future expansion: Reserved addresses for temperature, humidity sensors

### DALI Protocol
- Baud rate: 1200 bps (Manchester encoding)
- Voltage: 16V nominal (9.5V-22.5V operating range)
- Current: 2mA typical
- Compliance: IEC 62386 (DALI-2), IEC 62386-104 (D4i)

### Layout Notes
- **ALS placement**: U9 positioned for ambient light measurement, avoid shadows
- **I2C routing**: SDA/SCL parallel routing, equal length, 30 mil spacing minimum
- **DALI isolation**: Optional: Use isolated DALI transceiver for enhanced protection
- **Connector placement**: J2 at board edge for easy access
- **ESD protection**: D3, D4 close to J2 connector

### Test Points
- TP17: I2C_SDA
- TP18: I2C_SCL
- TP19: DALI_A
- TP20: DALI_B

---

## Page 6: Energy Buffer

### Block Description
Supercapacitor energy storage for brownout ride-through and OR-ing diode network for power source selection.

### Component List

| Designator | Part Number | Description | Value/Rating | Package |
|-----------|-------------|-------------|--------------|---------|
| C_SC1 | CE5R5155VF-ZJ | Supercapacitor | 1.5F, 5.5V | Radial |
| U11 | TPS3840PL27 | Brownout supervisor | 2.7V threshold | SOT-23-5 |
| D_OR1 | SS14 | Schottky OR-ing diode | 40V, 1A | SMA |
| D_OR2 | SS14 | Schottky OR-ing diode | 40V, 1A | SMA |
| R19 | Generic | Soft-start resistor | 100Ω, 1W | 2512 |
| Q2 | BSS138 | N-channel MOSFET | Soft-start switch | SOT-23 |
| R20 | Generic | Soft-start gate resistor | 100kΩ | 0603 |
| R21 | Generic | Brownout reset pullup | 10kΩ | 0603 |
| C28 | Generic | Brownout filter capacitor | 0.1µF, 10V, X7R | 0603 |
| R22 | Generic | Supercap discharge resistor | 1kΩ, 1/4W | 0603 |

### Net Connections

```
Power source selection:
  5V_OUT (from U1) → D_OR1.anode
  5V_SUPERCAP (from C_SC1) → D_OR2.anode
  D_OR1.cathode → 5V_SYS
  D_OR2.cathode → 5V_SYS
  
Supercapacitor charging:
  5V_OUT → R19 → Q2.DRAIN
  Q2.SOURCE → C_SC1.POS
  C_SC1.NEG → GND
  C_SC1.POS → R22 → GND (bleed resistor)
  
Soft-start control:
  ESP32.GPIO15 → R20 → Q2.GATE
  Q2.SOURCE → Q2.GATE (bootstrap)
  
Brownout detection:
  U11 (TPS3840PL27):
    Pin 1 (VDD) → 3V3_OUT
    Pin 2 (GND) → GND
    Pin 3 (SENSE) → 3V3_OUT (monitors 3.3V rail)
    Pin 4 (RESET) → ESP32.GPIO13 (BROWNOUT_INT, via R21 pullup)
    Pin 5 (CT) → C28 → GND (delay capacitor)
```

### Supercapacitor Sizing

**Energy storage calculation:**
- Capacitance: 1.5F
- Voltage range: 5.5V (charged) to 2.5V (discharged)
- Energy stored: E = 0.5 × C × (V²_max - V²_min)
- E = 0.5 × 1.5F × (5.5² - 2.5²) = 18.4 Joules

**Runtime estimation:**
- Average system power: 2W (ToF active)
- Minimum power: 0.5W (idle)
- Runtime at 2W: ~9 seconds
- Runtime at 0.5W: ~37 seconds

**Charging time:**
- Soft-start current limit: 50mA (via R19)
- Charge time (0V to 5V): τ = C × V / I = 1.5F × 5V / 0.05A = 150 seconds
- Fast charge mode: Bypass Q2, charge in ~30 seconds

### OR-ing Diode Operation
- Forward voltage drop: ~0.3V @ 1A (Schottky)
- Selection logic: Highest voltage source powers the system
- Reverse leakage: <100µA @ 25°C
- No active control required (passive OR-ing)

### Brownout Detection
- Threshold: 2.7V on 3.3V rail (~18% undervoltage)
- Assertion delay: 10ms (via C28)
- Reset pulse width: 200ms minimum
- ISR action: Save state, shutdown non-critical loads

### Layout Notes
- **Supercapacitor placement**: C_SC1 near power input, short connections
- **OR-ing diodes**: D_OR1, D_OR2 with thermal vias, handle 1A continuous
- **Soft-start path**: R19, Q2 rated for peak current
- **Brownout supervisor**: U11 close to 3V3 rail sense point
- **Ground connection**: C_SC1.NEG to main GND with thick trace (100 mil)

### Test Points
- TP21: 5V_SYS
- TP22: 5V_SUPERCAP
- TP23: BROWNOUT_INT
- TP24: C_SC1.POS

---

## Page 7: Lighting Outputs

### Block Description
Multiple output modes for lighting control: 0-10V, 1-10V analog dimming, and PWM switching.

### Component List

| Designator | Part Number | Description | Value/Rating | Package |
|-----------|-------------|-------------|--------------|---------|
| U12 | MCP4725 | 12-bit DAC | I2C, 0-VDD output | SOT-23-6 |
| U13 | OPA2350 | Dual op-amp | Rail-to-rail | SOIC-8 |
| Q3 | Si2301 | P-channel MOSFET | -20V, -2.3A | SOT-23 |
| Q4 | BSS138 | N-channel MOSFET | 50V, 0.2A | SOT-23 |
| R23 | Generic | DAC load resistor | 10kΩ | 0603 |
| R24 | Generic | 0-10V scale resistor | 3.3kΩ, 1% | 0603 |
| R25 | Generic | 0-10V scale resistor | 1kΩ, 1% | 0603 |
| R26 | Generic | 1-10V offset resistor | 10kΩ, 1% | 0603 |
| R27 | Generic | 1-10V offset resistor | 1.1kΩ, 1% | 0603 |
| R28 | Generic | PWM pulldown | 10kΩ | 0603 |
| C29 | Generic | DAC decoupling | 0.1µF, 10V, X7R | 0603 |
| C30 | Generic | Op-amp decoupling | 0.1µF, 10V, X7R | 0603 |
| C31 | Generic | Output filter | 1µF, 25V, X7R | 0805 |
| J3 | Generic | Output connector | 4-pin terminal block | 3.5mm pitch |

### Net Connections

```
U12 (MCP4725) DAC:
  Pin 1 (VOUT) → R23 → GND, → U13A.IN+ (pin 3)
  Pin 2 (GND) → GND
  Pin 3 (VDD) → 3V3_OUT (via C29)
  Pin 4 (SDA) → I2C_SDA
  Pin 5 (SCL) → I2C_SCL
  Pin 6 (A0) → GND (I2C address 0x60)

U13A (op-amp, 0-10V output):
  Pin 1 (OUT_A) → J3.1 (0-10V output, via C31)
  Pin 2 (IN-_A) → R24 → Pin 1 (OUT_A)
  Pin 3 (IN+_A) → U12.VOUT
  Pin 4 (GND) → GND
  R25 → Pin 3 (IN+_A) → GND (input bias)
  
U13B (op-amp, 1-10V output):
  Pin 5 (IN+_B) → U12.VOUT
  Pin 6 (IN-_B) → R26 → Pin 7 (OUT_B)
  Pin 7 (OUT_B) → J3.2 (1-10V output, via C31)
  Pin 8 (VCC) → 12V_EXT (external supply for >3.3V output)
  R27 → Pin 6 (IN-_B) → GND (offset adjust)

PWM outputs:
  ESP32.GPIO20 → Q3.GATE (P-channel, high-side switch)
  Q3.SOURCE → 12V_EXT
  Q3.DRAIN → J3.3 (PWM_OUT_1)
  
  ESP32.GPIO21 → Q4.GATE (N-channel, low-side switch)
  Q4.GATE → R28 → GND
  Q4.DRAIN → J3.4 (PWM_OUT_2)
  Q4.SOURCE → GND

Output connector J3:
  Pin 1: 0-10V analog output
  Pin 2: 1-10V analog output
  Pin 3: PWM high-side output (12V switched)
  Pin 4: PWM low-side output (sinking)
```

### Output Specifications

**0-10V Mode:**
- Range: 0.0V to 10.0V
- Resolution: 12-bit (2.44mV per step)
- Accuracy: ±1% (with 1% resistors)
- Load: 10kΩ minimum
- Update rate: 100Hz via I2C

**1-10V Mode:**
- Range: 1.0V to 10.0V
- Resolution: 12-bit (2.20mV per step)
- Accuracy: ±1%
- Load: 10kΩ minimum
- Offset error: <0.1V

**PWM Mode:**
- Frequency: 1kHz default (adjustable 100Hz-20kHz)
- Duty cycle: 0-100%
- Resolution: 10-bit (0.1% per step)
- Max current: Q3: 2A, Q4: 200mA
- Rise/fall time: <1µs

### Layout Notes
- **Analog outputs**: Star ground for U12, U13, keep analog traces separate
- **Output filtering**: C31 close to J3 connector
- **Power supply**: U13 requires 12V external supply for >3.3V outputs
- **PWM switching**: Q3, Q4 with flyback protection diodes if driving inductive loads
- **Connector placement**: J3 at board edge

### Test Points
- TP25: DAC_OUT (U12.VOUT)
- TP26: 0-10V_OUT
- TP27: 1-10V_OUT
- TP28: PWM_OUT_1

---

## Page 8: Layout Guidelines & Integration Notes

### PCB Stackup (4-Layer)

```
Layer 1 (Top):       Signal, components
Layer 2 (Inner):     GND plane (continuous)
Layer 3 (Inner):     Power planes (5V, 3.3V, partitioned)
Layer 4 (Bottom):    Signal, connectors
```

### Design Rules

| Parameter | Value | Notes |
|-----------|-------|-------|
| Board thickness | 1.6mm | Standard FR-4 |
| Copper weight | 1oz (35µm) | 2oz for high-current traces |
| Minimum trace width | 6 mil (0.15mm) | Signal traces |
| Power trace width | 30-100 mil | Based on current |
| Minimum clearance | 8 mil (0.2mm) | Signal-to-signal |
| Via size | 12 mil drill, 24 mil pad | Standard |
| Thermal via | 8 mil drill, 16 mil pad | Under ICs |

### Ground Plane Strategy

**Partitioning:**
- AGND island: TDC7200, photodiode, TIA, comparator
- DGND: ESP32, digital logic, DALI
- PGND: Power supply, laser driver
- Connection: Single-point star connection near main power input

**Vias:**
- Ground stitching vias every 10mm around perimeter
- Thermal vias under all IC thermal pads (4-6 vias per IC)
- Via-in-pad for exposed pads (filled and plated over)

### Power Plane Strategy

**5V Plane:**
- Pour on Layer 3, avoid crossing under AGND signals
- Separate pour for 5V_ANALOG (filtered)
- Connect via ferrite bead or 0Ω resistor

**3.3V Plane:**
- Shared pour for digital 3.3V
- Separate pour for 3V3_TDC (filtered via pi-filter)

### Critical Trace Routing

| Signal | Length | Width | Impedance | Notes |
|--------|--------|-------|-----------|-------|
| LASER_TRIG | <50mm | 10 mil | 50Ω | Guard traces, minimize stubs |
| Q1.GATE | <20mm | 10 mil | - | Short as possible |
| TDC.CLOCK | <30mm | 15 mil | 50Ω | Controlled impedance |
| TIA feedback | <5mm | 10 mil | - | No vias, minimize capacitance |
| SPI bus | ±5mm match | 8 mil | 50Ω | Equal length matching |
| I2C bus | Parallel | 10 mil | - | Equal length, 30 mil spacing |

### Component Placement

**Top side:**
- U1, U2 (power supply) near input connector
- U6 (TDC7200), Y1 (TCXO) in center, low-noise area
- U7 (ESP32-C6) in center
- U3, Q1, D1 (laser driver) together, near optics
- U4, U5, D2 (RX path) together, near photodiode

**Bottom side:**
- Connectors (J1, J2, J3)
- Output drivers (Q3, Q4)
- Supercapacitor (C_SC1)

### Thermal Management

| Component | Power Dissipation | Cooling Method |
|-----------|------------------|----------------|
| U1 (TPS54231) | 1-2W | Thermal vias, optional heatsink |
| U2 (TPS62840) | 0.5W | Thermal vias |
| Q1 (laser MOSFET) | <1W (pulsed) | Thermal vias, copper pour |
| U4 (OPA857) | 0.3W | Thermal vias |
| D1 (laser diode) | <1W (average) | Thermal coupling to heatsink |

**Thermal vias:**
- Drill: 8 mil (0.2mm)
- Spacing: 1mm on-center
- Placement: Grid pattern under thermal pads

### EMC/EMI Considerations

**Emissions reduction:**
- Ferrite beads on power lines (24V input, 5V, 3.3V)
- Pi-filters for sensitive analog supplies
- Ground plane stitching around perimeter
- Shielding over high-speed sections (optional)

**Immunity:**
- TVS diodes on all external connections
- ESD protection on connectors (DALI, outputs)
- Filtering on power input (C1, C2)

### Optical Alignment

**Laser diode (D1):**
- Position: 45° angle to PCB surface
- Collimating lens: 5mm diameter, 10mm focal length
- Beam divergence: <5° full angle

**Photodiode (D2):**
- Position: 45° angle, opposite side from D1
- Focusing lens: 10mm diameter, 15mm focal length
- Field of view: 30° full angle

**Baseline distance:**
- Laser to photodiode: 100mm baseline
- Range: 0.2m to 10m (configurable)

### Test and Debug Features

**Test points (all on top side):**
- Power rails: 24V, 5V, 3.3V, GND
- Critical signals: LASER_TRIG, STOP_SIGNAL, TDC.CLOCK
- Analog signals: TIA_OUT, I_SENSE, DAC_OUT
- Digital buses: SPI, I2C, DALI

**Debug headers:**
- UART: TX, RX, GND for ESP32 programming
- JTAG: Optional 10-pin header for debugging
- ISP: Boot mode selection jumper

### Assembly Notes

**SMT assembly:**
- Stencil thickness: 0.125mm (5 mil)
- Solder paste: SAC305 lead-free
- Reflow profile: IPC/J-STD-020 compliant

**Hand assembly:**
- Supercapacitor C_SC1: Through-hole, solder and clip leads
- Connectors: Through-hole terminal blocks
- Optical components: Post-SMT assembly, alignment jig required

### Inspection and Testing

**AOI (Automated Optical Inspection):**
- All SMT components
- Solder joint quality
- Polarity verification

**Functional tests:**
1. Power-on test: Verify 5V, 3.3V rails
2. Laser safety test: Confirm <1mW average power
3. ToF test: Measure known distance target
4. DALI test: Send/receive DALI commands
5. Output test: Verify 0-10V, PWM outputs

### Compliance and Safety

**Laser safety (IEC 60825-1):**
- Class 1M: Eye-safe with proper diffuser/collimation
- Warning label: "Invisible laser radiation"
- Interlock: Optional for development boards

**Electrical safety:**
- Isolation: DALI interface isolated (optional)
- ESD: All external connections ESD protected
- Overvoltage: TVS diodes on power input

**EMC compliance:**
- EN 55015: Limits for lighting equipment
- EN 61547: Immunity requirements
- FCC Part 15: Radiated/conducted emissions

### Bill of Materials Summary

- Total components: ~100 (including passives)
- Unique parts: ~50
- Estimated cost: $25-35 (1000 pcs)
- Assembly: SMT + hand-soldered connectors

### Revision History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2024-01 | Initial release |

---

**Document End**

For detailed component specifications, refer to:
- `/bom/BOM_LIDAR_COMPLETE.csv`
- `/docs/DESIGN_NOTES_COMPLETE.md`
- Component datasheets (linked in BOM)
