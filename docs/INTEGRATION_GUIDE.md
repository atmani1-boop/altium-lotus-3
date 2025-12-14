# LiDAR ToF 905nm System - Integration Guide

## Table of Contents

1. [Sensor Fusion Logic](#sensor-fusion-logic)
2. [DALI/D4i Data Mapping](#dalid4i-data-mapping)
3. [Lighting Control Algorithms](#lighting-control-algorithms)
4. [Calibration Procedures](#calibration-procedures)
5. [Testing Procedures](#testing-procedures)

---

## Sensor Fusion Logic

### Overview

The system combines three sensor modalities for intelligent lighting control:

1. **ToF (Time-of-Flight)**: Presence detection and distance measurement (0.2m-10m)
2. **ALS (Ambient Light Sensor)**: Daylight harvesting (1-65535 lux)
3. **Presence Detection**: Derived from ToF measurements

### Data Fusion Architecture

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│ ToF Distance │     │  ALS Lux     │     │  DALI Cmd    │
│  (mm)        │     │  (lux)       │     │  (dimming)   │
└──────┬───────┘     └──────┬───────┘     └──────┬───────┘
       │                    │                    │
       ▼                    ▼                    ▼
┌──────────────────────────────────────────────────────────┐
│           Sensor Fusion State Machine                    │
│                                                           │
│  ┌─────────────┐   ┌──────────────┐   ┌──────────────┐ │
│  │  Presence   │──►│  Daylight    │──►│   Output     │ │
│  │  Detection  │   │  Harvesting  │   │   Control    │ │
│  └─────────────┘   └──────────────┘   └──────────────┘ │
└───────────────────────────────┬──────────────────────────┘
                                 │
                                 ▼
                     ┌────────────────────┐
                     │ 0-10V / 1-10V /PWM │
                     │   Output Driver    │
                     └────────────────────┘
```

### Presence Detection Algorithm

**State Machine:**

```c
typedef enum {
    PRESENCE_VACANT,      // No presence detected
    PRESENCE_OCCUPIED,    // Presence detected
    PRESENCE_HOLDOVER,    // Recently occupied, grace period
    PRESENCE_UNKNOWN      // Sensor fault or startup
} presence_state_t;

typedef struct {
    uint16_t distance_mm;       // Current ToF reading
    uint8_t confidence;         // Measurement confidence (0-100%)
    uint32_t timestamp_ms;      // Time of measurement
    presence_state_t state;     // Current presence state
    uint32_t last_motion_ms;    // Time of last detected motion
} presence_data_t;
```

**Detection Logic:**

```c
#define PRESENCE_THRESHOLD_MIN_MM   200    // 20cm minimum distance
#define PRESENCE_THRESHOLD_MAX_MM   10000  // 10m maximum distance
#define PRESENCE_DEBOUNCE_MS        500    // 500ms debounce time
#define PRESENCE_HOLDOVER_MS        300000 // 5 minutes holdover

presence_state_t update_presence(presence_data_t *data, uint16_t new_distance) {
    uint32_t now_ms = millis();
    
    // Update measurement data
    data->distance_mm = new_distance;
    data->timestamp_ms = now_ms;
    
    // Check if distance indicates presence
    bool object_detected = (new_distance >= PRESENCE_THRESHOLD_MIN_MM) && 
                          (new_distance <= PRESENCE_THRESHOLD_MAX_MM);
    
    switch (data->state) {
        case PRESENCE_VACANT:
            if (object_detected && data->confidence > 80) {
                data->state = PRESENCE_OCCUPIED;
                data->last_motion_ms = now_ms;
            }
            break;
            
        case PRESENCE_OCCUPIED:
            if (object_detected) {
                data->last_motion_ms = now_ms; // Update activity timestamp
            } else if ((now_ms - data->last_motion_ms) > PRESENCE_DEBOUNCE_MS) {
                data->state = PRESENCE_HOLDOVER; // Enter grace period
            }
            break;
            
        case PRESENCE_HOLDOVER:
            if (object_detected) {
                data->state = PRESENCE_OCCUPIED; // Return to occupied
                data->last_motion_ms = now_ms;
            } else if ((now_ms - data->last_motion_ms) > PRESENCE_HOLDOVER_MS) {
                data->state = PRESENCE_VACANT; // Timeout, declare vacant
            }
            break;
            
        default:
            data->state = PRESENCE_VACANT;
            break;
    }
    
    return data->state;
}
```

### Daylight Harvesting

**Constant Illuminance Control:**

```c
#define TARGET_ILLUMINANCE_LUX  500  // Target desktop illuminance
#define LIGHT_CONTRIBUTION_LUX  300  // Fixture's contribution at 100%

uint8_t calculate_dimming_level(uint16_t ambient_lux) {
    // Calculate required artificial light
    int16_t required_lux = TARGET_ILLUMINANCE_LUX - ambient_lux;
    
    if (required_lux <= 0) {
        return 0; // Sufficient daylight, lights off
    }
    
    if (required_lux >= LIGHT_CONTRIBUTION_LUX) {
        return 100; // Maximum dimming needed
    }
    
    // Proportional dimming
    uint8_t dimming_pct = (uint8_t)((required_lux * 100) / LIGHT_CONTRIBUTION_LUX);
    
    return dimming_pct;
}
```

### Combined Control Logic

**Master Control Function:**

```c
typedef struct {
    presence_state_t presence;
    uint16_t ambient_lux;
    uint8_t dali_override;    // 0-254 (255 = no override)
    uint8_t manual_override;  // 0-100%, 255 = auto
} lighting_input_t;

typedef struct {
    uint8_t output_level;     // 0-100%
    uint16_t voltage_mv;      // For 0-10V output (0-10000mV)
    uint16_t pwm_duty;        // For PWM output (0-1023)
    uint32_t transition_ms;   // Fade time
} lighting_output_t;

lighting_output_t calculate_lighting_output(lighting_input_t *input) {
    lighting_output_t output = {0};
    uint8_t target_level = 0;
    
    // 1. Check for manual override
    if (input->manual_override != 255) {
        target_level = input->manual_override;
        output.transition_ms = 0; // Instant response to manual control
    }
    // 2. Check for DALI override
    else if (input->dali_override != 255) {
        target_level = (input->dali_override * 100) / 254; // Convert DALI 0-254 to 0-100%
        output.transition_ms = 1000; // 1 second fade
    }
    // 3. Automatic control based on presence and daylight
    else {
        if (input->presence == PRESENCE_OCCUPIED || 
            input->presence == PRESENCE_HOLDOVER) {
            // Occupied: apply daylight harvesting
            target_level = calculate_dimming_level(input->ambient_lux);
            output.transition_ms = 2000; // 2 second fade
        } else {
            // Vacant: lights off
            target_level = 0;
            output.transition_ms = 5000; // 5 second fade out
        }
    }
    
    // Apply dimming curve correction (linear to perceived brightness)
    output.output_level = apply_dimming_curve(target_level);
    
    // Convert to output formats
    output.voltage_mv = (output.output_level * 10000) / 100; // 0-10V mapping
    output.pwm_duty = (output.output_level * 1023) / 100;    // 10-bit PWM
    
    return output;
}
```

**Dimming Curve (Linear to Logarithmic):**

```c
// CIE lightness curve approximation
uint8_t apply_dimming_curve(uint8_t linear_pct) {
    if (linear_pct == 0) return 0;
    if (linear_pct == 100) return 100;
    
    // Piecewise linear approximation of logarithmic curve
    // L* = 116 * (Y/Yn)^(1/3) - 16
    
    const uint8_t curve_lut[11] = {
        0,   // 0%
        1,   // 10% → 1%
        4,   // 20% → 4%
        9,   // 30% → 9%
        16,  // 40% → 16%
        25,  // 50% → 25%
        36,  // 60% → 36%
        49,  // 70% → 49%
        64,  // 80% → 64%
        81,  // 90% → 81%
        100  // 100% → 100%
    };
    
    uint8_t index = linear_pct / 10;
    uint8_t remainder = linear_pct % 10;
    
    // Linear interpolation between LUT points
    uint8_t lower = curve_lut[index];
    uint8_t upper = curve_lut[index + 1];
    
    return lower + ((upper - lower) * remainder) / 10;
}
```

---

## DALI/D4i Data Mapping

### DALI-2 Memory Bank Structure

**Memory Bank 0 (Standard Parameters):**

| Address | Parameter | Value/Range | Description |
|---------|-----------|-------------|-------------|
| 0x00 | Last Light Level | 0-254 | Current dimming level |
| 0x01 | Power On Level | 0-254 | Level on power-up (default: 254 = last) |
| 0x02 | System Failure Level | 0-254 | Level on system fault |
| 0x03 | Min Level | 1-254 | Minimum dimming level (default: 10) |
| 0x04 | Max Level | 1-254 | Maximum dimming level (default: 254) |
| 0x05 | Fade Time | 0-15 | Fade duration (0.7s to 90s) |
| 0x06 | Fade Rate | 1-15 | Fade rate (steps/s) |
| 0x07-0x0F | Scene Levels | 0-254 | 16 programmable scenes |
| 0x10 | Group 0-7 | Bitmask | Group membership |
| 0x11 | Group 8-15 | Bitmask | Group membership |
| 0x12 | Random Address H | 0-255 | Random address (24-bit) |
| 0x13 | Random Address M | 0-255 | Random address |
| 0x14 | Random Address L | 0-255 | Random address |

**Memory Bank 1 (D4i Diagnostic Data):**

| Address | Parameter | Format | Description |
|---------|-----------|--------|-------------|
| 0x00-0x01 | Operating Time | uint16 (hours) | Total operating hours |
| 0x02-0x03 | Start Counter | uint16 | Number of power-on cycles |
| 0x04-0x05 | External Supply Voltage | uint16 (mV) | Input voltage (24V nominal) |
| 0x06-0x07 | Power Consumption | uint16 (mW) | Real-time power usage |
| 0x08 | Overall Failure | Bitmask | System fault flags |
| 0x09 | External Supply | Bitmask | Power supply status |
| 0x0A | Light Source | Bitmask | LED/laser status |
| 0x0B | Thermal Derating | 0-100% | Thermal throttling |
| 0x0C | Thermal Shutdown | Boolean | Overtemperature flag |
| 0x0D | Temperature | int8 (°C) | Internal temperature |

**Memory Bank 2 (D4i Sensor Data):**

| Address | Parameter | Format | Description |
|---------|-----------|--------|-------------|
| 0x00-0x01 | Lux Level | uint16 (lux) | ALS reading |
| 0x02-0x03 | ToF Distance | uint16 (mm) | Distance measurement |
| 0x04 | Presence State | enum | Vacant/Occupied/Holdover |
| 0x05 | Motion Counter | uint8 | Events since last read |
| 0x06-0x09 | Last Motion Time | uint32 (s) | Unix timestamp |

### DALI Command Mapping

**Standard Commands:**

| Command | Value | Function | Implementation |
|---------|-------|----------|----------------|
| OFF | 0x00 | Turn off | Set output = 0% |
| UP | 0x01 | Dim up | Increase by fade_rate |
| DOWN | 0x02 | Dim down | Decrease by fade_rate |
| STEP UP | 0x03 | Step up | Increase by 1 step |
| STEP DOWN | 0x04 | Step down | Decrease by 1 step |
| RECALL MAX | 0x05 | Max level | Set to max_level |
| RECALL MIN | 0x06 | Min level | Set to min_level |
| STEP DOWN AND OFF | 0x07 | Step & off | Decrement, off if min |
| ON STEP UP | 0x08 | On & step | Turn on, increment |
| GO TO SCENE x | 0x10-0x1F | Scene recall | Load scene 0-15 |
| DAPC | 0x20-0xFE | Direct level | Set to value (0-254) |

**D4i Extended Commands:**

| Command | Function | Response |
|---------|----------|----------|
| QUERY STATUS | Read status byte | Bit 0: Lamp failure, Bit 3: Power failure |
| QUERY LIGHT LEVEL | Read current level | 0-254 (actual arc power) |
| QUERY EXTERNAL SUPPLY VOLTAGE | Read input V | Voltage in mV (24000 nominal) |
| QUERY POWER CONSUMPTION | Read power | Power in mW |
| QUERY SENSOR DATA | Read sensors | ALS lux, ToF distance |

### DALI Message Handler

```c
typedef struct {
    uint8_t address;      // Short address (0-63) or group (0-15)
    uint8_t command;      // DALI command byte
    uint8_t data;         // Optional data byte
    bool is_broadcast;    // True if broadcast message
    bool is_group;        // True if group address
} dali_message_t;

void handle_dali_command(dali_message_t *msg) {
    // Check if message is for this device
    if (!is_addressed_to_me(msg)) {
        return;
    }
    
    switch (msg->command) {
        case DALI_OFF:
            set_output_level(0);
            break;
            
        case DALI_RECALL_MAX:
            set_output_level(get_max_level());
            break;
            
        case DALI_RECALL_MIN:
            set_output_level(get_min_level());
            break;
            
        case DALI_GO_TO_SCENE_0 ... DALI_GO_TO_SCENE_15:
            uint8_t scene = msg->command - DALI_GO_TO_SCENE_0;
            set_output_level(get_scene_level(scene));
            break;
            
        case DALI_DAPC_MIN ... DALI_DAPC_MAX:
            uint8_t level = msg->command; // Direct arc power control
            set_output_level(level);
            break;
            
        case DALI_QUERY_STATUS:
            send_dali_response(get_status_byte());
            break;
            
        case DALI_QUERY_LIGHT_LEVEL:
            send_dali_response(get_current_level());
            break;
            
        // D4i extensions
        case DALI_QUERY_EXTERNAL_SUPPLY_VOLTAGE:
            send_dali_response_16bit(get_input_voltage_mv());
            break;
            
        case DALI_QUERY_POWER_CONSUMPTION:
            send_dali_response_16bit(get_power_consumption_mw());
            break;
            
        default:
            // Unknown command, no response
            break;
    }
}
```

---

## Lighting Control Algorithms

### Fade Engine

**Smooth Transitions:**

```c
typedef struct {
    uint8_t current_level;    // Current output (0-100%)
    uint8_t target_level;     // Target output (0-100%)
    uint32_t start_time_ms;   // Fade start time
    uint32_t duration_ms;     // Fade duration
    bool active;              // Fade in progress
} fade_state_t;

fade_state_t fade_state = {0};

void start_fade(uint8_t target, uint32_t duration_ms) {
    fade_state.target_level = target;
    fade_state.start_time_ms = millis();
    fade_state.duration_ms = duration_ms;
    fade_state.active = true;
}

uint8_t update_fade() {
    if (!fade_state.active) {
        return fade_state.current_level;
    }
    
    uint32_t elapsed = millis() - fade_state.start_time_ms;
    
    if (elapsed >= fade_state.duration_ms) {
        // Fade complete
        fade_state.current_level = fade_state.target_level;
        fade_state.active = false;
    } else {
        // Linear interpolation
        int16_t delta = fade_state.target_level - fade_state.current_level;
        fade_state.current_level += (delta * elapsed) / fade_state.duration_ms;
    }
    
    return fade_state.current_level;
}
```

### Adaptive Sensitivity

**Adjust ToF sensitivity based on ambient light:**

```c
uint8_t calculate_tof_confidence(uint16_t distance_mm, uint16_t ambient_lux) {
    // Base confidence from distance measurement
    uint8_t base_confidence = 100;
    
    // Reduce confidence for very close or far objects
    if (distance_mm < 300) {
        base_confidence -= (300 - distance_mm) / 10; // Penalize <30cm
    } else if (distance_mm > 8000) {
        base_confidence -= (distance_mm - 8000) / 100; // Penalize >8m
    }
    
    // Reduce confidence in high ambient light (SNR degradation)
    if (ambient_lux > 10000) {
        base_confidence -= (ambient_lux - 10000) / 500; // Bright sunlight penalty
    }
    
    // Clamp to 0-100%
    if (base_confidence < 0) base_confidence = 0;
    
    return base_confidence;
}
```

---

## Calibration Procedures

### ToF Calibration

**1. Zero-Distance Calibration (System Delay):**

```c
// Measure internal propagation delays with no target
#define NUM_CAL_SAMPLES 100

uint32_t calibrate_zero_distance() {
    uint32_t sum = 0;
    uint8_t valid_samples = 0;
    
    for (int i = 0; i < NUM_CAL_SAMPLES; i++) {
        tdc_result_t result = tdc7200_measure();
        
        if (result.valid) {
            sum += result.time_ps;
            valid_samples++;
        }
        delay_ms(10);
    }
    
    if (valid_samples < NUM_CAL_SAMPLES / 2) {
        return 0; // Calibration failed
    }
    
    uint32_t avg_delay_ps = sum / valid_samples;
    
    // Store in non-volatile memory
    nvs_write_u32("cal_zero_ps", avg_delay_ps);
    
    return avg_delay_ps;
}
```

**2. Known-Distance Calibration (Scale Factor):**

```c
// Measure known distances to calibrate scale factor
#define KNOWN_DISTANCE_1_MM 500   // 50cm
#define KNOWN_DISTANCE_2_MM 2000  // 2m

float calibrate_scale_factor() {
    uint32_t zero_delay_ps = nvs_read_u32("cal_zero_ps");
    
    // Measure known distance 1
    uint32_t time_1_ps = measure_average_tof(NUM_CAL_SAMPLES) - zero_delay_ps;
    
    // Measure known distance 2
    uint32_t time_2_ps = measure_average_tof(NUM_CAL_SAMPLES) - zero_delay_ps;
    
    // Calculate scale factor (mm per picosecond)
    // Speed of light: 300 mm/ns = 0.3 mm/ps (round trip = 0.15 mm/ps)
    float scale_1 = (float)KNOWN_DISTANCE_1_MM / time_1_ps;
    float scale_2 = (float)KNOWN_DISTANCE_2_MM / time_2_ps;
    
    float avg_scale = (scale_1 + scale_2) / 2.0;
    
    // Store in non-volatile memory
    nvs_write_float("cal_scale", avg_scale);
    
    return avg_scale;
}
```

**3. Runtime Calibration (Temperature Compensation):**

```c
// Periodically recalibrate to compensate for temperature drift
void runtime_calibration() {
    static uint32_t last_cal_ms = 0;
    
    if ((millis() - last_cal_ms) > 600000) { // Every 10 minutes
        // Quick two-point calibration using internal reference
        uint32_t cal_1 = tdc7200_calibration_1();
        uint32_t cal_2 = tdc7200_calibration_2();
        
        // Update calibration coefficients
        tdc7200_update_cal(cal_1, cal_2);
        
        last_cal_ms = millis();
    }
}
```

### ALS Calibration

**1. Dark Calibration (Offset):**

```c
uint16_t calibrate_als_dark() {
    // Cover sensor completely, measure dark current offset
    uint16_t dark_reading = als_read_raw();
    
    nvs_write_u16("als_dark_offset", dark_reading);
    
    return dark_reading;
}
```

**2. Known-Light Calibration (Scale):**

```c
float calibrate_als_scale(uint16_t known_lux) {
    // Measure with calibrated lux meter in parallel
    uint16_t als_reading = als_read_raw();
    uint16_t dark_offset = nvs_read_u16("als_dark_offset");
    
    float scale = (float)known_lux / (als_reading - dark_offset);
    
    nvs_write_float("als_scale", scale);
    
    return scale;
}
```

---

## Testing Procedures

### Factory Acceptance Test (FAT)

**1. Power-On Test:**
- [ ] Verify 24V input current <50mA idle
- [ ] Verify 5V rail = 5.0V ±2%
- [ ] Verify 3.3V rail = 3.3V ±2%
- [ ] Verify no overcurrent or thermal faults

**2. Laser Safety Test:**
- [ ] Measure average optical power <1mW
- [ ] Verify pulse width 10-50ns
- [ ] Verify duty cycle <1%
- [ ] Check interlock function (if applicable)

**3. ToF Functionality Test:**
- [ ] Place target at 50cm, verify reading 500mm ±10mm
- [ ] Place target at 200cm, verify reading 2000mm ±20mm
- [ ] Verify measurement rate >50Hz
- [ ] Check confidence metric >80%

**4. ALS Test:**
- [ ] Illuminate with 500 lux, verify reading 500 ±50 lux
- [ ] Cover sensor, verify reading <10 lux
- [ ] Bright light (10000 lux), verify reading 10000 ±1000 lux

**5. DALI Test:**
- [ ] Send DALI OFF command, verify output = 0V
- [ ] Send DALI level 127, verify output = 5V ±0.1V (0-10V mode)
- [ ] Send DALI level 254, verify output = 10V ±0.1V
- [ ] Query status, verify response received

**6. Output Test:**
- [ ] 0-10V mode: Sweep 0-100%, verify linear output 0-10V
- [ ] 1-10V mode: Sweep 0-100%, verify linear output 1-10V
- [ ] PWM mode: Set 50%, verify duty cycle = 50% ±2%

**7. Energy Buffer Test:**
- [ ] Remove input power, verify runtime >10 seconds
- [ ] Verify brownout interrupt triggers at 2.7V
- [ ] Verify graceful shutdown (save state to NVS)

### Burn-In Test (24 hours)

**Procedure:**
1. Apply 24V input power
2. Enable ToF measurements at 50Hz
3. Enable WiFi connection
4. Cycle DALI commands every 60 seconds
5. Monitor for faults, thermal issues, crashes

**Pass Criteria:**
- No system resets or crashes
- Junction temperature <85°C
- ToF measurement drift <5%
- No communication errors

### Environmental Test

**Temperature Cycling:**
- -20°C for 1 hour → +70°C for 1 hour (10 cycles)
- Verify functionality at temperature extremes
- Check calibration drift <10%

**Humidity:**
- 85% RH, 40°C, 48 hours
- Verify no condensation or corrosion
- Verify functionality after drying

---

**Document Version:** 1.0  
**Last Updated:** 2024-12-14  

For related documentation, see:
- [Schematic Documentation](/docs/SCHEMATICS_LIDAR_COMPLETE.md)
- [Design Notes](/docs/DESIGN_NOTES_COMPLETE.md)
- [Firmware Structure](/docs/FIRMWARE_STRUCTURE.md)
