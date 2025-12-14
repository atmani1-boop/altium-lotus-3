# LiDAR ToF 905nm System - Energy Buffer Design

## Table of Contents

1. [Supercapacitor Sizing Calculations](#supercapacitor-sizing-calculations)
2. [Soft-Charge Circuit Analysis](#soft-charge-circuit-analysis)
3. [OR-ing Diode Operation](#or-ing-diode-operation)
4. [Brownout Detection](#brownout-detection)
5. [Runtime Estimates](#runtime-estimates)

---

## Supercapacitor Sizing Calculations

### Energy Storage Requirements

**Design Goals:**
- Provide ride-through for short power interruptions (1-10 seconds)
- Allow graceful shutdown on extended outage (save state, close files)
- Support continuous operation during brief brownouts (<100ms)

### Energy Calculation

**Stored Energy in Capacitor:**
```
E = (1/2) × C × (V_max² - V_min²)

Where:
- E = energy stored (Joules)
- C = capacitance (Farads)
- V_max = maximum voltage (5.5V)
- V_min = minimum usable voltage (2.5V)
```

**Selected Supercapacitor: CE5R5155VF-ZJ**
```
Capacitance: C = 1.5F
Voltage range: V_max = 5.5V (rated), V_min = 2.5V (3.3V reg dropout + margin)

E = 0.5 × 1.5F × (5.5² - 2.5²)
E = 0.5 × 1.5 × (30.25 - 6.25)
E = 0.5 × 1.5 × 24
E = 18 Joules
```

### Power Consumption Modes

| Mode | Description | Power (W) | Typical Duration |
|------|-------------|-----------|-----------------|
| **Active** | ToF @ 50Hz, WiFi on, ALS reading | 1.5 W | Continuous |
| **Idle** | WiFi idle, no ToF | 0.5 W | Standby |
| **Sleep** | Deep sleep, brownout monitor only | 0.01 W | Power-down |
| **Shutdown** | Saving state, closing NVS | 0.8 W | 2-3 seconds |

### Runtime Calculation

**Runtime Formula:**
```
t = E / P = (C × (V_max² - V_min²)) / (2 × P)

Or more precisely, accounting for voltage drop:
t = (C × (V_max - V_min)) / I_avg

Where:
- I_avg = P / V_avg
- V_avg = (V_max + V_min) / 2 = (5.5 + 2.5) / 2 = 4.0V
```

**Runtime at Different Power Levels:**

```
Active mode (1.5W):
  I_avg = 1.5W / 4.0V = 375mA
  t = (1.5F × (5.5 - 2.5)) / 0.375A = 12 seconds
  
Idle mode (0.5W):
  I_avg = 0.5W / 4.0V = 125mA
  t = (1.5F × 3.0V) / 0.125A = 36 seconds
  
Shutdown mode (0.8W):
  I_avg = 0.8W / 4.0V = 200mA
  t = (1.5F × 3.0V) / 0.200A = 22.5 seconds
```

**Conclusion:** 
- 12-second runtime in active mode exceeds 10-second requirement ✓
- 36-second runtime in idle mode provides ample margin ✓

### Supercapacitor Specifications

**CE5R5155VF-ZJ (Panasonic):**

| Parameter | Value | Notes |
|-----------|-------|-------|
| Capacitance | 1.5F ±20% | At 25°C |
| Rated voltage | 5.5V | Absolute maximum |
| ESR | 60mΩ @ 1kHz | Low internal resistance |
| Leakage current | <10µA @ 5.5V | After 72h |
| Operating temp | -25°C to +70°C | Matches system spec |
| Lifetime | 1000 hours @ 70°C | >10 years at 25°C |
| Package | Radial 10mm diameter | Through-hole mount |

**Temperature Derating:**

```
Capacitance vs Temperature:
  @ -25°C: C = 1.5F × 0.7 = 1.05F (30% reduction)
  @ +25°C: C = 1.5F × 1.0 = 1.50F (nominal)
  @ +70°C: C = 1.5F × 1.1 = 1.65F (10% increase)

Runtime @ -25°C (worst case):
  E = 0.5 × 1.05F × 24V² = 12.6J
  t_active = 12.6J / 1.5W = 8.4 seconds (still acceptable)
```

### Alternative Capacitor Options

| Part Number | Capacitance | Voltage | ESR | Cost | Notes |
|-------------|-------------|---------|-----|------|-------|
| CE5R5155VF-ZJ | 1.5F | 5.5V | 60mΩ | $3.50 | **Selected** |
| EECS0HD224V | 0.22F | 5.5V | 100mΩ | $1.20 | Too small, 2.6J |
| EECF5R5U105 | 1.0F | 5.5V | 90mΩ | $2.80 | Marginal, 12J |
| DXE5R5H104 | 10F | 5.5V | 30mΩ | $12.00 | Overkill, large |

**Selection Rationale:**
- CE5R5155VF-ZJ provides optimal balance of capacity, cost, and size
- 1.5F gives comfortable margin over minimum 1.0F requirement
- Automotive-qualified (AEC-Q200) for reliability

---

## Soft-Charge Circuit Analysis

### Purpose

Supercapacitors present a **short-circuit load** when uncharged (V=0). Without current limiting, inrush current can:
1. Trip PTC fuse F1
2. Cause voltage droop on 5V rail
3. Damage OR-ing diode D_OR2
4. Stress power supply U1

**Soft-charge circuit limits inrush current to safe level.**

### Circuit Topology

```
5V_OUT ──┬──────────────────────────┬─── D_OR1 ─┬─── 5V_SYS
         │                          │           │
         │                          └─── D_OR2 ─┘
         │                                │
         └─── R19 (100Ω) ─── Q2 ───────── C_SC1+ (1.5F)
                           (MOSFET)       │
                              │           │
                              │           C_SC1-
                     GPIO15 ──┘           │
                                         GND
```

### Soft-Start Sequence

**Phase 1: Resistor Pre-charge (0-4.5V)**
```
State: Q2 OFF (GPIO15 = LOW)
Current path: 5V_OUT → R19 → C_SC1 → GND

Initial inrush current:
  I_peak = V / R = 5V / 100Ω = 50mA ✓ (safe for all components)

Time constant:
  τ = R × C = 100Ω × 1.5F = 150 seconds

Voltage vs time:
  V_cap(t) = V_supply × (1 - e^(-t/τ))
  
Charging time to 4.5V (90% of 5V):
  4.5 = 5 × (1 - e^(-t/150))
  0.9 = 1 - e^(-t/150)
  e^(-t/150) = 0.1
  t = -150 × ln(0.1) = 345 seconds ≈ 5.8 minutes
```

**Phase 2: Direct Charge (4.5V-5.5V)**
```
State: Q2 ON (GPIO15 = HIGH after 5 minutes)
Current path: 5V_OUT → Q2 (low RDS) → C_SC1 → GND

Remaining charge:
  ΔQ = C × ΔV = 1.5F × (5.5V - 4.5V) = 1.5 Coulombs
  
At 500mA charging current:
  t = Q / I = 1.5C / 0.5A = 3 seconds
  
Total charging time: 5.8 min + 3 sec ≈ 6 minutes (acceptable)
```

### Component Selection

**R19: Soft-Start Resistor**
```
Resistance: 100Ω
Power rating: P = V² / R = 5² / 100 = 0.25W (use 1W for margin)
Package: 2512 (large thermal mass)
Tolerance: 5% (not critical)
```

**Q2: Bypass MOSFET (BSS138)**
```
Type: N-channel enhancement
V_DS: 50V (5V × 10 safety factor)
I_D: 200mA continuous (500mA pulsed)
R_DS(on): 1.5Ω @ V_GS=4.5V
Package: SOT-23

Power dissipation in Q2:
  P = I² × R_DS = (0.5A)² × 1.5Ω = 0.375W (transient, OK)
```

**R20: Gate Drive Resistor**
```
Resistance: 100kΩ (slow turn-on, prevents shoot-through)
Function: Rate-limit gate charge to prevent inrush
```

### Control Algorithm

```c
#define PRECHARGE_TIME_MS 300000  // 5 minutes
#define PRECHARGE_VOLTAGE_MV 4500 // 4.5V threshold

typedef enum {
    CHARGE_IDLE,          // Not charging
    CHARGE_SOFT_START,    // R19 pre-charge phase
    CHARGE_FAST,          // Q2 bypass phase
    CHARGE_COMPLETE       // Fully charged
} charge_state_t;

charge_state_t supercap_charge_state = CHARGE_IDLE;
uint32_t charge_start_time = 0;

void supercap_charge_init() {
    // Configure GPIO15 as output, set LOW (Q2 OFF)
    gpio_set_direction(GPIO_NUM_15, GPIO_MODE_OUTPUT);
    gpio_set_level(GPIO_NUM_15, 0);
    
    charge_start_time = millis();
    supercap_charge_state = CHARGE_SOFT_START;
}

void supercap_charge_update() {
    uint16_t vcap_mv = adc_read_supercap_voltage();
    uint32_t elapsed_ms = millis() - charge_start_time;
    
    switch (supercap_charge_state) {
        case CHARGE_SOFT_START:
            if (vcap_mv >= PRECHARGE_VOLTAGE_MV || 
                elapsed_ms >= PRECHARGE_TIME_MS) {
                // Activate Q2, enter fast charge
                gpio_set_level(GPIO_NUM_15, 1);
                supercap_charge_state = CHARGE_FAST;
            }
            break;
            
        case CHARGE_FAST:
            if (vcap_mv >= 5300) { // 5.3V = ~95% charged
                supercap_charge_state = CHARGE_COMPLETE;
            }
            break;
            
        case CHARGE_COMPLETE:
            // Keep Q2 ON for trickle charging
            break;
            
        default:
            break;
    }
}
```

### Safety Considerations

**Overcurrent Protection:**
- PTC fuse F1 (500mA) protects against Q2 short-circuit
- R19 limits fault current even if Q2 fails shorted

**Reverse Current:**
- Body diode of Q2 prevents discharge of C_SC1 back to 5V rail if 5V fails
- However, D_OR2 provides primary reverse blocking

**Voltage Monitoring:**
- ADC monitors V_supercap to detect charging progress
- Watchdog timer ensures Q2 activated even if ADC fails

---

## OR-ing Diode Operation

### Purpose

Automatically select highest voltage source between:
1. **5V_OUT**: Main 5V rail from buck converter U1
2. **5V_SUPERCAP**: Supercapacitor buffer voltage

**No active control required** - passive diode OR-ing.

### Circuit Topology

```
5V_OUT ────────┬─── D_OR1 (SS14) ───┬─── 5V_SYS ─── System loads
               │                    │
         C5 (100µF)                C_sys (bulk)
               │                    │
              GND                   │
                                    │
5V_SUPERCAP ───┴─── D_OR2 (SS14) ───┘
               │
          C_SC1 (1.5F)
               │
              GND
```

### Operating Modes

**Mode 1: Normal Operation (Main power present)**
```
Conditions:
  V_5V_OUT = 5.0V
  V_SUPERCAP = 5.3V (fully charged)

Diode voltages:
  V_D_OR1 = V_5V_OUT - V_D_drop = 5.0V - 0.3V = 4.7V
  V_D_OR2 = V_SUPERCAP - V_D_drop = 5.3V - 0.3V = 5.0V

Result: D_OR2 conducts, D_OR1 reverse-biased
  V_SYS = 5.0V (from supercap)
  
Note: Supercap supplies system to reduce ripple on 5V rail
```

**Mode 2: Power Failure (Main power lost)**
```
Conditions:
  V_5V_OUT = 0V (drops rapidly)
  V_SUPERCAP = 5.3V → 2.5V (discharging)

Diode voltages:
  V_D_OR1 = 0V - 0.3V = -0.3V (reverse-biased)
  V_D_OR2 = V_SUPERCAP - 0.3V (forward-biased)

Result: Only D_OR2 conducts
  V_SYS = V_SUPERCAP - 0.3V (5.0V → 2.2V over 12 seconds)
  
System continues operating until V_SYS < 2.7V (brownout threshold)
```

**Mode 3: Supercap Discharged (After extended outage)**
```
Conditions:
  V_5V_OUT = 5.0V (restored)
  V_SUPERCAP = 2.0V (depleted)

Diode voltages:
  V_D_OR1 = 5.0V - 0.3V = 4.7V (forward-biased)
  V_D_OR2 = 2.0V - 0.3V = 1.7V (reverse-biased)

Result: D_OR1 conducts, system powered from main rail
  V_SYS = 4.7V
  Supercap begins recharging via soft-start circuit
```

### Diode Selection

**SS14 Schottky Diode:**

| Parameter | Value | Notes |
|-----------|-------|-------|
| Forward voltage | 0.25-0.35V @ 1A | Low drop preserves energy |
| Reverse voltage | 40V | 5.5V × 7 safety margin |
| Forward current | 1A continuous | System peak ~500mA |
| Reverse leakage | <100µA @ 25°C | Minimal loss |
| Package | SMA (DO-214AC) | 2.5×1.6mm, easy assembly |

**Power Dissipation:**
```
Worst case: 1A forward current
  P_diode = V_F × I_F = 0.35V × 1A = 0.35W

Thermal resistance:
  θ_JA = 100°C/W (SMA package, no heatsink)
  
Temperature rise:
  ΔT = P × θ_JA = 0.35W × 100°C/W = 35°C
  
Junction temp (@ 70°C ambient):
  T_J = 70 + 35 = 105°C < 150°C (rated max) ✓
```

**Thermal Management:**
- Add copper pour around D_OR1, D_OR2 for heat spreading
- Use 4 thermal vias under each diode to bottom layer
- Consider larger package (DO-214AB) if continuous 1A operation

### Efficiency Analysis

**Energy Loss in OR-ing Diodes:**
```
Average system current: I_avg = 300mA
Diode forward drop: V_F = 0.3V

Power loss: P_loss = V_F × I_avg = 0.3V × 0.3A = 90mW

System power: P_system = 5V × 0.3A = 1.5W

Efficiency: η = (1.5W - 0.09W) / 1.5W = 94% (acceptable)
```

**Alternative: Ideal Diode Controller**

For higher efficiency, consider replacing Schottky diodes with active ideal diode controller (e.g., LTC4352):
- Forward drop: ~10mV (vs 300mV)
- Power loss: ~3mW (vs 90mW)
- Cost: +$2 per channel
- **Decision: Use Schottky for simplicity, cost savings**

---

## Brownout Detection

### Purpose

Detect imminent power failure and trigger graceful shutdown:
1. Save system state to non-volatile storage
2. Close open files, flush buffers
3. Signal DALI master of impending shutdown
4. Disable laser for safety
5. Enter deep sleep to preserve remaining energy

### Brownout Monitor Circuit

**TPS3840PL27: Precision Voltage Supervisor**

```
3V3_OUT ────┬─── U11.VDD (pin 1)
            │
            ├─── U11.SENSE (pin 3) ─── Threshold = 2.7V
            │
            │    U11.RESET (pin 4) ────┬─── R21 (10kΩ) ─── 3V3_OUT
            │                          │
            │                          └─── ESP32.GPIO13 (BROWNOUT_INT)
            │
            └─── U11.CT (pin 5) ─── C28 (0.1µF) ─── GND
            
         U11.GND (pin 2) ─── GND
```

### Threshold Selection

**3.3V Rail Monitoring:**
```
Nominal voltage: 3.3V
Brownout threshold: 2.7V (82% of nominal)
Hysteresis: 100mV (typ)

Reset assertion: V_SENSE < 2.7V (falling)
Reset de-assertion: V_SENSE > 2.8V (rising)
```

**Rationale for 2.7V Threshold:**

| Component | Min Voltage | Margin @ 2.7V |
|-----------|-------------|---------------|
| ESP32-C6 | 2.3V | +400mV |
| TDC7200 | 2.2V | +500mV |
| OPA857 | 2.2V | +500mV |
| TLV3502 | 2.7V | 0mV (limiting) |
| BH1750 | 2.4V | +300mV |

**TLV3502 comparator has highest minimum voltage (2.7V)**, so brownout must trigger before rail drops below this level.

### Timing Characteristics

**Assertion Delay (Power Fail → RESET Low):**
```
Delay time = R_CT × C_CT / 1.2µA

With C28 = 0.1µF:
  t_delay = 10kΩ × 0.1µF / 1.2µA ≈ 833ms

Too long! Reduce to 10ms:
  C28 = 1.2µA × 10ms / 10kΩ = 1.2nF

Use C28 = 10nF for 8ms delay (margin for supply fluctuations)
```

**Reset Pulse Width:**
```
Minimum pulse: 200ms (adjustable via C_CT)
  
Ensures ESP32 has sufficient time to enter ISR and save critical state.
```

### Interrupt Service Routine

```c
#define GPIO_BROWNOUT GPIO_NUM_13

volatile bool brownout_detected = false;

void IRAM_ATTR brownout_isr(void *arg) {
    // CRITICAL: This ISR must execute quickly (<1ms)
    brownout_detected = true;
    
    // Disable laser immediately
    gpio_set_level(GPIO_LASER_TRIG, 0);
    
    // Wake main task
    xTaskNotifyFromISR(main_task_handle, BROWNOUT_EVENT, eSetValueWithOverwrite, NULL);
}

void brownout_init() {
    // Configure GPIO13 as input with interrupt
    gpio_config_t io_conf = {
        .pin_bit_mask = (1ULL << GPIO_BROWNOUT),
        .mode = GPIO_MODE_INPUT,
        .pull_up_en = GPIO_PULLUP_ENABLE,
        .pull_down_en = GPIO_PULLDOWN_DISABLE,
        .intr_type = GPIO_INTR_NEGEDGE  // Trigger on falling edge
    };
    gpio_config(&io_conf);
    
    // Install ISR
    gpio_install_isr_service(0);
    gpio_isr_handler_add(GPIO_BROWNOUT, brownout_isr, NULL);
}

void brownout_handler_task() {
    uint32_t notification;
    
    while (1) {
        // Wait for brownout event
        xTaskNotifyWait(0, 0xFFFFFFFF, &notification, portMAX_DELAY);
        
        if (notification & BROWNOUT_EVENT) {
            ESP_LOGW(TAG, "Brownout detected! Initiating shutdown...");
            
            // 1. Save critical state (time budget: ~100ms)
            nvs_commit_all();  // Flush NVS buffers
            
            // 2. Send DALI shutdown notification
            dali_send_emergency_message();
            
            // 3. Close open files
            file_system_sync();
            
            // 4. Log event
            event_log_write(EVENT_BROWNOUT, millis());
            
            // 5. Enter deep sleep
            ESP_LOGI(TAG, "Entering deep sleep mode");
            esp_deep_sleep_start();
        }
    }
}
```

### Brownout Recovery

**Wake-Up Sequence:**
```c
void app_main() {
    // Check reset reason
    esp_reset_reason_t reset_reason = esp_reset_reason();
    
    if (reset_reason == ESP_RST_BROWNOUT || 
        reset_reason == ESP_RST_DEEPSLEEP) {
        
        // Check if power is stable
        uint16_t v3v3_mv = adc_read_3v3_rail();
        
        if (v3v3_mv < 3200) {
            // Still brownout, go back to sleep
            ESP_LOGW(TAG, "Power not restored, sleeping...");
            esp_deep_sleep(1000000); // Sleep 1 second, retry
        }
        
        // Power restored, check supercap charge
        uint16_t vcap_mv = adc_read_supercap_voltage();
        
        if (vcap_mv < 4500) {
            ESP_LOGI(TAG, "Supercap charging, limited functionality");
            // Enter safe mode with reduced features
            enter_safe_mode();
        }
        
        // Full recovery
        ESP_LOGI(TAG, "Brownout recovery complete");
        restore_system_state();
    }
}
```

---

## Runtime Estimates

### Detailed Power Budget

| Component | Mode | Voltage | Current | Power | Duty | Avg Power |
|-----------|------|---------|---------|-------|------|-----------|
| ESP32-C6 | WiFi TX | 3.3V | 190mA | 627mW | 10% | 63mW |
| ESP32-C6 | WiFi RX | 3.3V | 90mA | 297mW | 20% | 59mW |
| ESP32-C6 | Modem idle | 3.3V | 25mA | 82mW | 70% | 57mW |
| ESP32-C6 | CPU active | 3.3V | 45mA | 148mW | 100% | 148mW |
| **ESP32 Total** | | | | | | **327mW** |
| TDC7200 | Active | 3.3V | 5mA | 16.5mW | 1% | 0.17mW |
| TDC7200 | Sleep | 3.3V | 10µA | 33µW | 99% | 0.03mW |
| **TDC Total** | | | | | | **0.2mW** |
| Laser pulse | Peak | 1.6V | 75A | 120W | 0.005% | 6mW |
| Laser driver | Active | 5V | 10mA | 50mW | 0.5% | 0.25mW |
| **Laser Total** | | | | | | **6.25mW** |
| OPA857 TIA | Active | 5V | 7mA | 35mW | 100% | 35mW |
| TLV3502 Comp | Active | 5V | 3mA | 15mW | 100% | 15mW |
| **RX Total** | | | | | | **50mW** |
| BH1750 ALS | Active | 3.3V | 0.2mA | 0.66mW | 10% | 0.07mW |
| DALI Trans | Idle | 3.3V | 1mA | 3.3mW | 90% | 3mW |
| DALI Trans | TX | 3.3V | 5mA | 16.5mW | 10% | 1.7mW |
| **Sensors Total** | | | | | | **4.77mW** |
| MCP4725 DAC | Active | 3.3V | 0.4mA | 1.3mW | 100% | 1.3mW |
| OPA2350 OpAmp | Active | 3.3V | 2mA | 6.6mW | 100% | 6.6mW |
| **Output Total** | | | | | | **7.9mW** |
| Buck U1 quiescent | Switching | 24V | 0.8mA | 19.2mW | 100% | 19.2mW |
| Buck U2 quiescent | Switching | 24V | 60µA | 1.4mW | 100% | 1.4mW |
| **Power Supply** | | | | | | **20.6mW** |
| **GRAND TOTAL** | | | | | | **416.7mW** |

### Runtime Scenarios

**Scenario 1: Active Monitoring (ToF @ 50Hz, WiFi connected)**
```
Power consumption: 420mW
Supercap energy: 18J
Runtime: 18J / 0.42W = 42.8 seconds
```

**Scenario 2: Idle (WiFi idle, no ToF)**
```
Components active:
  - ESP32 modem idle: 82mW
  - ESP32 CPU: 148mW
  - RX chain: 50mW
  - Sensors: 5mW
  - Outputs: 8mW
  - Power supply: 21mW
  
Total: 314mW

Runtime: 18J / 0.314W = 57.3 seconds
```

**Scenario 3: Graceful Shutdown (Save & sleep)**
```
Phase 1: Save state (2 seconds @ 400mW)
  Energy: 0.4W × 2s = 0.8J
  
Phase 2: Deep sleep (remaining time @ 10mW)
  Energy remaining: 18J - 0.8J = 17.2J
  Runtime: 17.2J / 0.01W = 1720 seconds = 28.7 minutes
```

### Voltage Decay Profile

**Discharge Curve (Active Mode, 420mW):**
```
Time  | Voltage | Current | Power  | Energy Remaining
------|---------|---------|--------|------------------
0s    | 5.3V    | 79mA    | 420mW  | 18.0J
5s    | 5.0V    | 84mA    | 420mW  | 15.9J
10s   | 4.7V    | 89mA    | 420mW  | 13.8J
15s   | 4.4V    | 95mA    | 420mW  | 11.7J
20s   | 4.1V    | 102mA   | 420mW  | 9.6J
25s   | 3.8V    | 110mA   | 420mW  | 7.5J
30s   | 3.5V    | 120mA   | 420mW  | 5.4J
35s   | 3.2V    | 131mA   | 420mW  | 3.3J
40s   | 2.9V    | 145mA   | 420mW  | 1.2J
42s   | 2.7V    | 155mA   | 420mW  | 0J (brownout)
```

**Conclusion: 42-second runtime from 5.3V to 2.7V brownout threshold**

### Design Margin

```
Required runtime: 10 seconds (specification)
Calculated runtime: 42 seconds (active mode)
Design margin: 42 / 10 = 4.2× (320% margin) ✓

Worst-case runtime (cold temp, aged cap):
  Capacitance derating: -30% (cold) -20% (aging) = 0.7 × 0.8 = 0.56
  Runtime: 42s × 0.56 = 23.5 seconds
  Margin: 23.5 / 10 = 2.35× (135% margin) ✓ ACCEPTABLE
```

---

**Document Version:** 1.0  
**Last Updated:** 2024-12-14  

For related documentation, see:
- [Schematic Documentation](/docs/SCHEMATICS_LIDAR_COMPLETE.md)
- [Design Notes](/docs/DESIGN_NOTES_COMPLETE.md)
- [Integration Guide](/docs/INTEGRATION_GUIDE.md)
