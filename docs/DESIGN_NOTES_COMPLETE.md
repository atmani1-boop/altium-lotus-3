# LiDAR ToF 905nm System - Design Notes

## Table of Contents

1. [System Architecture Overview](#system-architecture-overview)
2. [Block-by-Block Design Rationale](#block-by-block-design-rationale)
3. [Component Selection Criteria](#component-selection-criteria)
4. [Layout Guidelines](#layout-guidelines)
5. [Power Budget Calculations](#power-budget-calculations)
6. [Thermal Analysis](#thermal-analysis)
7. [EMC/EMI Considerations](#emcemi-considerations)
8. [Laser Safety (IEC 60825)](#laser-safety-iec-60825)

---

## System Architecture Overview

### High-Level Block Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                      LiDAR ToF System                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────┐    ┌──────────────┐    ┌─────────────┐         │
│  │ Zhaga-18 ├───►│ Power Supply ├───►│ 5V & 3.3V   │         │
│  │  24V DC  │    │ Protection   │    │ Regulators  │         │
│  └──────────┘    └──────────────┘    └──────┬──────┘         │
│                                              │                 │
│  ┌───────────────────────────────────────────┼─────────┐      │
│  │                                           │         │      │
│  │  ┌────────────┐     ┌────────────┐      │         │      │
│  │  │ Laser      │◄────┤ ESP32-C6   │◄─────┤         │      │
│  │  │ Driver     │     │ MCU        │      │         │      │
│  │  │ (905nm)    │     └─────┬──────┘      │         │      │
│  │  └─────┬──────┘           │             │         │      │
│  │        │                  │             │         │      │
│  │        ▼ Optical          ▼ Control     │         │      │
│  │        Pulse              SPI            │         │      │
│  │        │                  │             │         │      │
│  │  ┌─────▼──────┐     ┌─────▼──────┐      │         │      │
│  │  │ Photodiode │     │  TDC7200   │      │         │      │
│  │  │ + TIA      ├────►│  Time      │      │         │      │
│  │  │            │     │  Measure   │      │         │      │
│  │  └────────────┘     └────────────┘      │         │      │
│  │                                          │         │      │
│  │  ┌────────────┐     ┌────────────┐      │         │      │
│  │  │ ALS Sensor │────►│   DALI     │      │         │      │
│  │  │ (I2C)      │     │ Interface  │      │         │      │
│  │  └────────────┘     └────────────┘      │         │      │
│  │                                          │         │      │
│  │  ┌────────────┐                         │         │      │
│  │  │ Supercap   │◄────────────────────────┘         │      │
│  │  │ Buffer     │                                   │      │
│  │  └────────────┘                                   │      │
│  │                                                    │      │
│  │  ┌──────────────────────────────────────┐         │      │
│  │  │ Lighting Control Outputs             │◄────────┘      │
│  │  │ (0-10V, 1-10V, PWM)                  │                │
│  │  └──────────────────────────────────────┘                │
│  │                                                          │
│  └──────────────────────────────────────────────────────────┘
│                                                               │
└───────────────────────────────────────────────────────────────┘
```

### System Specifications

| Parameter | Specification | Notes |
|-----------|--------------|-------|
| Input voltage | 24V DC ±10% | Zhaga-18 standard |
| Input current | 2A maximum | Including lighting loads |
| Measurement range | 0.2m to 10m | Configurable |
| Range resolution | 1mm | At 55ps time resolution |
| Measurement rate | Up to 100Hz | Limited by laser duty cycle |
| Laser wavelength | 905nm | Near-infrared |
| Laser peak power | 75W | 10-50ns pulses |
| Laser average power | <1mW | Eye-safe Class 1M |
| Ambient light sensor | 1-65535 lux | 16-bit resolution |
| DALI compliance | IEC 62386-104 (D4i) | DALI-2 with diagnostics |
| Wireless | WiFi, BLE, Zigbee | ESP32-C6 |
| Operating temp | -20°C to +70°C | Extended industrial range |

### Design Philosophy

**Key Principles:**
1. **Safety First**: Laser safety compliance (IEC 60825) is paramount
2. **Reliability**: Robust power supply with brownout protection
3. **Accuracy**: High-precision time measurement with calibration
4. **Integration**: Zhaga-18 and DALI-2/D4i standard compliance
5. **Modularity**: Clean interfaces between functional blocks
6. **Manufacturability**: SMT-friendly design with standard components

---

## Block-by-Block Design Rationale

### 1. Power Supply

**Design Goals:**
- Convert 24V Zhaga-18 input to 5V and 3.3V
- Protection against overvoltage, overcurrent, reverse polarity
- Low noise for analog circuits (TIA, TDC)
- Sufficient current for laser pulses (peak 75A, average <100mA)

**Architecture Selection:**
- **Buck converters** chosen over LDOs for efficiency (>90% vs <50%)
- **Two-stage**: 24V→5V (TPS54231) then 5V→3.3V (TPS62840)
- **Separation**: Analog 5V filtered separately for TIA/comparator

**Component Rationale:**

| Component | Rationale |
|-----------|-----------|
| TPS54231 | 28V max input, 2A output, synchronous buck (high efficiency) |
| TPS62840 | Low quiescent current (60nA), small solution size |
| Littelfuse RXEF050 | Resettable PTC fuse, 500mA hold current |
| SMAJ24A TVS | Fast response (<1ns), 400W surge capability |
| Würth 744773 inductors | Shielded, low EMI, automotive-grade |

**Trade-offs:**
- **Cost vs Efficiency**: Synchronous buck costs more but saves 5-10% efficiency
- **Size vs Performance**: Larger inductors reduce ripple but increase board space
- **Complexity**: Two-stage conversion adds cost but improves noise performance

### 2. Laser Driver

**Design Goals:**
- Generate 10-50ns pulses at 905nm
- Peak current up to 75A with precise control
- Fast rise/fall times (<5ns) for accurate ToF
- Current sensing for calibration and safety

**Architecture Selection:**
- **Direct MOSFET drive**: High-speed gate driver (UCC27321) + low-RDS(on) MOSFET (BSC340N08NS)
- **Current sense**: Shunt resistor (0.1Ω) + differential ADC

**Component Rationale:**

| Component | Rationale |
|-----------|-----------|
| UCC27321 | 9A peak drive, 14ns prop delay, cheap (~$0.50) |
| BSC340N08NS | 3.4mΩ RDS(on), 80V breakdown, low Qg (20nC) |
| OSRAM SPL S1L90A_3 | High-power 905nm, proven automotive-grade |
| WSL2512R1000FEA | 0.1Ω ±1%, 1W, low inductance (<5nH) |

**Trade-offs:**
- **Peak current vs Safety**: 75A gives 10m range but requires careful PCB design
- **Pulse width vs Range**: Shorter pulses improve resolution but reduce SNR
- **Cost**: High-power laser diode is most expensive component (~$8-12)

### 3. Analog Reception

**Design Goals:**
- Detect weak reflected 905nm pulses (nW to µW)
- Convert photocurrent to voltage with low noise
- Fast response (<10ns) for accurate stop signal
- Wide dynamic range (60dB+)

**Component Rationale:**

| Component | Rationale |
|-----------|-----------|
| Hamamatsu S5973 | 0.65A/W @ 905nm, fast rise time (10ns), low dark current |
| OPA857 | 1.5GHz GBW, 1.3pA/√Hz input noise, optimized for photodiodes |
| TLV3502 | 4.5ns prop delay, rail-to-rail, dual comparator |

---

## Component Selection Criteria

### General Criteria

1. **Availability**: All components available from major distributors
2. **Lifecycle**: Active parts with >5 year projected lifecycle
3. **Second source**: Preferred to have alternate manufacturers for critical parts
4. **Package**: SMT packages suitable for automated assembly
5. **Temperature**: -40°C to +85°C minimum (industrial grade)
6. **Lead-free**: RoHS compliant
7. **Cost target**: <$30 BOM cost at 1000 pcs

---

## Layout Guidelines

### 4-Layer PCB Stackup

```
Layer 1 (Top):    Signal + Components (1oz copper, 35µm)
Layer 2 (Inner):  GND Plane (1oz copper, 35µm) - continuous
Layer 3 (Inner):  Power Planes (1oz copper, 35µm) - 5V, 3.3V
Layer 4 (Bottom): Signal + Connectors (1oz copper, 35µm)

Total thickness: 1.6mm (62 mil)
Dielectric: FR-4, Er=4.3, Tg=135°C
```

### Critical Trace Impedance

| Signal | Target Z₀ | Width (mil) | Spacing (mil) | Layer |
|--------|----------|-------------|---------------|-------|
| LASER_TRIG | 50Ω | 10 | 20 | Top |
| TDC.CLOCK | 50Ω | 15 | 30 | Top |
| SPI (SCLK/MOSI/MISO) | 50Ω | 8 | 16 | Top |

---

## Power Budget Calculations

### Power Consumption by Block

| Block | Voltage | Current | Power | Average |
|-------|---------|---------|-------|---------|
| **ESP32-C6** Active | 3.3V | 120mA | 396mW | 396mW |
| **TDC7200** Active | 3.3V | 5mA | 16.5mW | 1.7mW |
| **Laser** Pulse | 1.6V | 75A | 120W | 12mW |
| **TIA + Comp** | 5V | 15mA | 75mW | 75mW |
| **ALS Sensor** | 3.3V | 0.3mA | 1mW | 1mW |

**Total Power:**
- **Idle**: ~500mW
- **Active ToF**: ~540mW
- **Full load**: ~1.13W

---

## Thermal Analysis

### Component Thermal Characteristics

| Component | Power | θ_JA (°C/W) | ΔT (°C) | T_J (°C) | T_J,max (°C) | Status |
|-----------|-------|-------------|---------|----------|--------------|--------|
| U1 (TPS54231) | 1.5W | 45 | 67.5 | 92.5 | 150 | OK |
| U2 (TPS62840) | 0.5W | 60 | 30 | 55 | 150 | OK |
| Q1 (laser) | 0.8W | 50 | 40 | 65 | 175 | OK |

---

## EMC/EMI Considerations

### Emission Sources

| Source | Frequency | Mitigation |
|--------|-----------|------------|
| Buck 5V | 400kHz | Input/output filtering, shielding |
| Laser pulse | 20MHz BW | Guard traces, ground plane |
| ESP32 WiFi | 2.4GHz | FCC-approved module |

### Mitigation Techniques

1. **Input filtering**: Common-mode choke, capacitors
2. **Spread spectrum**: Enable for WiFi/BLE
3. **Shielding**: Optional copper shield over laser driver
4. **PCB layout**: Minimize loop areas, controlled impedance

---

## Laser Safety (IEC 60825)

### Classification: Class 1M (Eye-safe)

**Laser parameters:**
- Peak power: 75W
- Pulse width: 50ns
- Repetition rate: 100Hz (max)
- Wavelength: 905nm

**Average power:**
```
P_avg = P_peak × τ × f = 75W × 50ns × 100Hz = 375µW < 1mW ✓
```

### Safety Measures

**Hardware:**
1. Collimating lens with 5° to 30° divergence
2. Optional interlock for production
3. Brownout disables laser
4. Current limit enforcement

**Firmware:**
1. Maximum 1% duty cycle enforced
2. Watchdog timer with laser disable
3. Current monitoring and calibration

**Labeling:**
```
⚠ CAUTION - INVISIBLE LASER RADIATION
CLASS 1M LASER PRODUCT
905nm, <1mW average
IEC 60825-1:2014
```

---

**Document Version:** 1.0  
**Last Updated:** 2024-12-14  

For implementation details, see:
- [Schematic Documentation](/docs/SCHEMATICS_LIDAR_COMPLETE.md)
- [Bill of Materials](/bom/BOM_LIDAR_COMPLETE.csv)
- [Integration Guide](/docs/INTEGRATION_GUIDE.md)
