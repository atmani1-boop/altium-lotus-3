# LiDAR ToF 905nm System - Firmware Structure

## Table of Contents

1. [Task Architecture (FreeRTOS)](#task-architecture-freertos)
2. [Driver Interfaces](#driver-interfaces)
3. [Power Management](#power-management)
4. [ISR Handlers](#isr-handlers)
5. [Calibration Routines](#calibration-routines)

---

## Task Architecture (FreeRTOS)

### Overview

The firmware is built on **ESP-IDF** (Espressif IoT Development Framework) using **FreeRTOS** for real-time multitasking.

### Task Hierarchy

```
┌─────────────────────────────────────────────────────────────┐
│                     FreeRTOS Scheduler                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌────────────────┐  ┌────────────────┐  ┌──────────────┐  │
│  │ ToF Measurement│  │ Sensor Reading │  │ DALI Handler │  │
│  │ Task           │  │ Task           │  │ Task         │  │
│  │ Priority: 3    │  │ Priority: 2    │  │ Priority: 2  │  │
│  │ Stack: 4KB     │  │ Stack: 3KB     │  │ Stack: 3KB   │  │
│  └───────┬────────┘  └───────┬────────┘  └──────┬───────┘  │
│          │                   │                   │          │
│          └───────────┬───────┴───────────────────┘          │
│                      │                                      │
│            ┌─────────▼──────────┐                           │
│            │ Control Logic Task │                           │
│            │ Priority: 2        │                           │
│            │ Stack: 4KB         │                           │
│            └─────────┬──────────┘                           │
│                      │                                      │
│          ┌───────────┴────────────┐                         │
│          │                        │                         │
│  ┌───────▼────────┐      ┌────────▼──────┐                 │
│  │ WiFi/BLE Task  │      │ Logging Task  │                 │
│  │ Priority: 1    │      │ Priority: 0   │                 │
│  │ Stack: 8KB     │      │ Stack: 3KB    │                 │
│  └────────────────┘      └───────────────┘                 │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### Task Definitions

**1. ToF Measurement Task (Priority 3 - Highest)**

```c
#define TOF_TASK_STACK_SIZE 4096
#define TOF_TASK_PRIORITY   3

void tof_measurement_task(void *pvParameters) {
    TickType_t last_wake_time = xTaskGetTickCount();
    const TickType_t period_ticks = pdMS_TO_TICKS(20); // 50Hz measurement rate
    
    tof_result_t result;
    
    while (1) {
        // Wait for next measurement cycle
        vTaskDelayUntil(&last_wake_time, period_ticks);
        
        // Trigger laser pulse
        gpio_set_level(GPIO_LASER_TRIG, 1);
        esp_rom_delay_us(5); // 5µs pulse width (hardware will shape to 50ns)
        gpio_set_level(GPIO_LASER_TRIG, 0);
        
        // Wait for TDC7200 to complete measurement
        if (xSemaphoreTake(tdc_ready_sem, pdMS_TO_TICKS(10)) == pdTRUE) {
            // Read measurement from TDC7200
            result = tdc7200_read_result();
            
            if (result.valid) {
                // Apply calibration
                uint16_t distance_mm = tof_calculate_distance(&result);
                
                // Update presence detection
                presence_state_t presence = update_presence_detection(distance_mm);
                
                // Publish result to message queue
                tof_publish_result(distance_mm, presence, result.confidence);
            }
        } else {
            ESP_LOGW(TAG, "TDC timeout");
        }
        
        // Periodic calibration
        runtime_calibration_check();
    }
}
```

**2. Sensor Reading Task (Priority 2)**

```c
#define SENSOR_TASK_STACK_SIZE 3072
#define SENSOR_TASK_PRIORITY   2

void sensor_reading_task(void *pvParameters) {
    TickType_t last_wake_time = xTaskGetTickCount();
    const TickType_t period_ticks = pdMS_TO_TICKS(1000); // 1Hz update rate
    
    sensor_data_t sensor_data;
    
    while (1) {
        vTaskDelayUntil(&last_wake_time, period_ticks);
        
        // Read ambient light sensor
        sensor_data.lux = bh1750_read_lux();
        
        // Read internal temperature (from ESP32)
        sensor_data.temperature_c = esp_temp_sensor_read();
        
        // Read supercap voltage (via ADC)
        sensor_data.supercap_mv = adc_read_supercap_voltage();
        
        // Read input voltage (via ADC)
        sensor_data.vin_mv = adc_read_input_voltage();
        
        // Publish to message queue
        sensor_publish_data(&sensor_data);
        
        // Update DALI memory bank 2 (sensor data)
        dali_update_sensor_data(&sensor_data);
    }
}
```

**3. DALI Handler Task (Priority 2)**

```c
#define DALI_TASK_STACK_SIZE 3072
#define DALI_TASK_PRIORITY   2

void dali_handler_task(void *pvParameters) {
    dali_message_t rx_msg;
    
    while (1) {
        // Wait for DALI message from UART ISR
        if (xQueueReceive(dali_rx_queue, &rx_msg, portMAX_DELAY) == pdTRUE) {
            // Process DALI command
            handle_dali_command(&rx_msg);
            
            // Update operating time counter (D4i requirement)
            dali_update_operating_time();
        }
    }
}
```

**4. Control Logic Task (Priority 2)**

```c
#define CONTROL_TASK_STACK_SIZE 4096
#define CONTROL_TASK_PRIORITY   2

void control_logic_task(void *pvParameters) {
    lighting_input_t input = {0};
    lighting_output_t output = {0};
    
    TickType_t last_wake_time = xTaskGetTickCount();
    const TickType_t period_ticks = pdMS_TO_TICKS(100); // 10Hz control loop
    
    while (1) {
        vTaskDelayUntil(&last_wake_time, period_ticks);
        
        // Gather inputs from message queues
        tof_get_latest_result(&input.presence, &input.distance_mm);
        sensor_get_latest_data(&input.ambient_lux);
        dali_get_override(&input.dali_override);
        
        // Calculate desired output
        output = calculate_lighting_output(&input);
        
        // Apply fade
        uint8_t current_level = update_fade();
        
        // Update hardware outputs
        set_dac_output(output.voltage_mv);
        set_pwm_output(output.pwm_duty);
        
        // Log state change
        if (input.presence != prev_presence) {
            ESP_LOGI(TAG, "Presence: %s", presence_to_string(input.presence));
            event_log_write(EVENT_PRESENCE_CHANGE, input.presence);
        }
    }
}
```

**5. WiFi/BLE Task (Priority 1)**

```c
#define WIFI_TASK_STACK_SIZE 8192
#define WIFI_TASK_PRIORITY   1

void wifi_ble_task(void *pvParameters) {
    // Initialize WiFi
    wifi_init_sta();
    
    // Initialize BLE (for commissioning)
    ble_init_server();
    
    while (1) {
        // Handle WiFi events
        wifi_event_handler();
        
        // Handle BLE events
        ble_event_handler();
        
        // Publish telemetry to cloud (every 60 seconds)
        static uint32_t last_telemetry_ms = 0;
        if ((millis() - last_telemetry_ms) > 60000) {
            publish_telemetry_to_cloud();
            last_telemetry_ms = millis();
        }
        
        vTaskDelay(pdMS_TO_TICKS(100));
    }
}
```

**6. Logging Task (Priority 0 - Lowest)**

```c
#define LOG_TASK_STACK_SIZE 3072
#define LOG_TASK_PRIORITY   0

void logging_task(void *pvParameters) {
    log_entry_t entry;
    
    while (1) {
        // Wait for log entries
        if (xQueueReceive(log_queue, &entry, portMAX_DELAY) == pdTRUE) {
            // Write to NVS (non-volatile storage)
            nvs_write_log_entry(&entry);
            
            // Optionally send to cloud
            if (wifi_is_connected()) {
                cloud_send_log_entry(&entry);
            }
        }
    }
}
```

### Inter-Task Communication

**Message Queues:**

```c
QueueHandle_t tof_result_queue;      // ToF task → Control task
QueueHandle_t sensor_data_queue;     // Sensor task → Control task
QueueHandle_t dali_rx_queue;         // UART ISR → DALI task
QueueHandle_t log_queue;             // All tasks → Logging task

void queues_init() {
    tof_result_queue = xQueueCreate(10, sizeof(tof_result_t));
    sensor_data_queue = xQueueCreate(5, sizeof(sensor_data_t));
    dali_rx_queue = xQueueCreate(16, sizeof(dali_message_t));
    log_queue = xQueueCreate(32, sizeof(log_entry_t));
}
```

**Semaphores:**

```c
SemaphoreHandle_t tdc_ready_sem;     // TDC ISR → ToF task
SemaphoreHandle_t brownout_sem;      // Brownout ISR → Power management

void semaphores_init() {
    tdc_ready_sem = xSemaphoreCreateBinary();
    brownout_sem = xSemaphoreCreateBinary();
}
```

---

## Driver Interfaces

### TDC7200 Driver (Time-to-Digital Converter)

**Initialization:**

```c
typedef struct {
    spi_device_handle_t spi_handle;
    gpio_num_t cs_pin;
    gpio_num_t enable_pin;
    gpio_num_t int_pin;
    uint16_t clock_freq_mhz;
} tdc7200_config_t;

esp_err_t tdc7200_init(tdc7200_config_t *config) {
    // Configure SPI
    spi_device_interface_config_t dev_cfg = {
        .clock_speed_hz = 20000000,  // 20MHz SPI clock
        .mode = 0,                   // CPOL=0, CPHA=0
        .spics_io_num = config->cs_pin,
        .queue_size = 3,
    };
    ESP_ERROR_CHECK(spi_bus_add_device(VSPI_HOST, &dev_cfg, &config->spi_handle));
    
    // Configure control pins
    gpio_set_direction(config->enable_pin, GPIO_MODE_OUTPUT);
    gpio_set_direction(config->int_pin, GPIO_MODE_INPUT);
    
    // Enable TDC7200
    gpio_set_level(config->enable_pin, 1);
    vTaskDelay(pdMS_TO_TICKS(10));
    
    // Configure registers
    tdc7200_write_reg(TDC_CONFIG1, 0x01);  // Measurement mode 1
    tdc7200_write_reg(TDC_CONFIG2, 0x40);  // 3 stop measurements
    tdc7200_write_reg(TDC_INT_MASK, 0x07); // Enable all interrupts
    
    return ESP_OK;
}
```

**Measurement:**

```c
typedef struct {
    uint32_t time_ps;       // Time in picoseconds
    uint8_t confidence;     // 0-100%
    bool valid;             // Measurement valid flag
} tdc_result_t;

tdc_result_t tdc7200_read_result() {
    tdc_result_t result = {0};
    
    // Read status register
    uint8_t status = tdc7200_read_reg(TDC_INT_STATUS);
    
    if (status & 0x01) { // Measurement complete
        // Read TIME1 (24-bit)
        uint32_t time1 = tdc7200_read_reg24(TDC_TIME1);
        
        // Read CLOCK_COUNT1 (24-bit)
        uint32_t clock_count = tdc7200_read_reg24(TDC_CLOCK_COUNT1);
        
        // Read CALIBRATION1 and CALIBRATION2
        uint32_t cal1 = tdc7200_read_reg24(TDC_CALIBRATION1);
        uint32_t cal2 = tdc7200_read_reg24(TDC_CALIBRATION2);
        
        // Calculate time with calibration
        // normLSB = (CALIBRATION2 - CALIBRATION1) / (calCount × (CLK_FREQ / 1000000))
        float norm_lsb = (cal2 - cal1) / (10.0 * 16.0); // 16MHz clock, 10 cal periods
        
        // TOF = ((TIME1 / normLSB) + (CLOCK_COUNT1 × CLK_PERIOD)) 
        float tof_ns = (time1 / norm_lsb) + (clock_count * 62.5); // 62.5ns @ 16MHz
        
        result.time_ps = (uint32_t)(tof_ns * 1000);
        result.valid = true;
        result.confidence = (status & 0x80) ? 50 : 100; // Check coarse overflow
    }
    
    return result;
}
```

### DALI Driver

**UART Configuration (Manchester Encoding):**

```c
#define DALI_UART_NUM UART_NUM_1
#define DALI_BAUD_RATE 1200
#define DALI_TX_PIN GPIO_NUM_8
#define DALI_RX_PIN GPIO_NUM_9

esp_err_t dali_init() {
    uart_config_t uart_config = {
        .baud_rate = DALI_BAUD_RATE * 2, // 2400 for Manchester
        .data_bits = UART_DATA_8_BITS,
        .parity = UART_PARITY_DISABLE,
        .stop_bits = UART_STOP_BITS_1,
        .flow_ctrl = UART_HW_FLOWCTRL_DISABLE,
    };
    
    ESP_ERROR_CHECK(uart_param_config(DALI_UART_NUM, &uart_config));
    ESP_ERROR_CHECK(uart_set_pin(DALI_UART_NUM, DALI_TX_PIN, DALI_RX_PIN, 
                                  UART_PIN_NO_CHANGE, UART_PIN_NO_CHANGE));
    ESP_ERROR_CHECK(uart_driver_install(DALI_UART_NUM, 1024, 1024, 10, 
                                        &uart_queue, 0));
    
    return ESP_OK;
}
```

**Manchester Encoding/Decoding:**

```c
void dali_send_byte(uint8_t data) {
    uint8_t manchester[2];
    
    // Manchester encode: 0 = 01, 1 = 10
    for (int i = 0; i < 8; i++) {
        uint8_t bit = (data >> (7 - i)) & 0x01;
        
        if (bit) {
            manchester[i / 4] |= (0x02 << ((i % 4) * 2)); // 1 → 10
        } else {
            manchester[i / 4] |= (0x01 << ((i % 4) * 2)); // 0 → 01
        }
    }
    
    uart_write_bytes(DALI_UART_NUM, (const char *)manchester, 2);
}

uint8_t dali_receive_byte(uint32_t timeout_ms) {
    uint8_t manchester[2];
    uint8_t data = 0;
    
    int len = uart_read_bytes(DALI_UART_NUM, manchester, 2, 
                             pdMS_TO_TICKS(timeout_ms));
    
    if (len != 2) return 0xFF; // Timeout or error
    
    // Decode Manchester
    for (int i = 0; i < 8; i++) {
        uint8_t bits = (manchester[i / 4] >> ((i % 4) * 2)) & 0x03;
        
        if (bits == 0x02) { // 10 → 1
            data |= (1 << (7 - i));
        } else if (bits == 0x01) { // 01 → 0
            // data bit already 0
        } else {
            return 0xFF; // Invalid Manchester encoding
        }
    }
    
    return data;
}
```

### I2C Sensor Driver (BH1750 ALS)

```c
#define I2C_MASTER_NUM I2C_NUM_0
#define I2C_MASTER_SDA_IO GPIO_NUM_5
#define I2C_MASTER_SCL_IO GPIO_NUM_11
#define I2C_MASTER_FREQ_HZ 400000

#define BH1750_ADDR 0x23
#define BH1750_CONTINUOUS_H_RES_MODE 0x10

esp_err_t bh1750_init() {
    i2c_config_t conf = {
        .mode = I2C_MODE_MASTER,
        .sda_io_num = I2C_MASTER_SDA_IO,
        .scl_io_num = I2C_MASTER_SCL_IO,
        .sda_pullup_en = GPIO_PULLUP_ENABLE,
        .scl_pullup_en = GPIO_PULLUP_ENABLE,
        .master.clk_speed = I2C_MASTER_FREQ_HZ,
    };
    
    ESP_ERROR_CHECK(i2c_param_config(I2C_MASTER_NUM, &conf));
    ESP_ERROR_CHECK(i2c_driver_install(I2C_MASTER_NUM, conf.mode, 0, 0, 0));
    
    // Start continuous measurement
    i2c_cmd_handle_t cmd = i2c_cmd_link_create();
    i2c_master_start(cmd);
    i2c_master_write_byte(cmd, (BH1750_ADDR << 1) | I2C_MASTER_WRITE, true);
    i2c_master_write_byte(cmd, BH1750_CONTINUOUS_H_RES_MODE, true);
    i2c_master_stop(cmd);
    esp_err_t ret = i2c_master_cmd_begin(I2C_MASTER_NUM, cmd, pdMS_TO_TICKS(1000));
    i2c_cmd_link_delete(cmd);
    
    return ret;
}

uint16_t bh1750_read_lux() {
    uint8_t data[2];
    
    i2c_cmd_handle_t cmd = i2c_cmd_link_create();
    i2c_master_start(cmd);
    i2c_master_write_byte(cmd, (BH1750_ADDR << 1) | I2C_MASTER_READ, true);
    i2c_master_read_byte(cmd, &data[0], I2C_MASTER_ACK);
    i2c_master_read_byte(cmd, &data[1], I2C_MASTER_NACK);
    i2c_master_stop(cmd);
    i2c_master_cmd_begin(I2C_MASTER_NUM, cmd, pdMS_TO_TICKS(1000));
    i2c_cmd_link_delete(cmd);
    
    uint16_t raw = (data[0] << 8) | data[1];
    uint16_t lux = raw / 1.2; // Conversion factor for H-res mode
    
    return lux;
}
```

---

## Power Management

### Power Modes

**Active Mode:**
```c
void power_mode_active() {
    // Enable all peripherals
    esp_pm_lock_acquire(&pm_lock_active);
    
    // Set CPU frequency to 160MHz
    esp_pm_configure(&pm_config_active);
    
    // Enable WiFi modem
    esp_wifi_set_ps(WIFI_PS_NONE);
}
```

**Idle Mode:**
```c
void power_mode_idle() {
    // Reduce CPU frequency to 80MHz
    esp_pm_configure(&pm_config_idle);
    
    // Enable light sleep when idle
    esp_wifi_set_ps(WIFI_PS_MIN_MODEM);
    
    // Stop ToF measurements
    tof_measurement_stop();
}
```

**Deep Sleep Mode:**
```c
void power_mode_deep_sleep(uint32_t sleep_time_sec) {
    // Save state to NVS
    nvs_commit_all();
    
    // Configure wake-up sources
    esp_sleep_enable_timer_wakeup(sleep_time_sec * 1000000ULL);
    esp_sleep_enable_ext0_wakeup(GPIO_BROWNOUT, 0); // Wake on brownout recovery
    
    // Enter deep sleep
    ESP_LOGI(TAG, "Entering deep sleep for %d seconds", sleep_time_sec);
    esp_deep_sleep_start();
}
```

---

## ISR Handlers

### TDC7200 Interrupt (Measurement Complete)

```c
static void IRAM_ATTR tdc_int_isr(void *arg) {
    BaseType_t higher_priority_task_woken = pdFALSE;
    
    // Signal ToF task that measurement is ready
    xSemaphoreGiveFromISR(tdc_ready_sem, &higher_priority_task_woken);
    
    // Yield if higher priority task woken
    if (higher_priority_task_woken) {
        portYIELD_FROM_ISR();
    }
}

void tdc_int_init() {
    gpio_config_t io_conf = {
        .pin_bit_mask = (1ULL << GPIO_TDC_INT),
        .mode = GPIO_MODE_INPUT,
        .pull_up_en = GPIO_PULLUP_ENABLE,
        .intr_type = GPIO_INTR_NEGEDGE,
    };
    gpio_config(&io_conf);
    
    gpio_install_isr_service(0);
    gpio_isr_handler_add(GPIO_TDC_INT, tdc_int_isr, NULL);
}
```

### Brownout Interrupt

```c
static void IRAM_ATTR brownout_int_isr(void *arg) {
    BaseType_t higher_priority_task_woken = pdFALSE;
    
    // Immediately disable laser
    gpio_set_level(GPIO_LASER_TRIG, 0);
    
    // Signal power management task
    xSemaphoreGiveFromISR(brownout_sem, &higher_priority_task_woken);
    
    if (higher_priority_task_woken) {
        portYIELD_FROM_ISR();
    }
}

void brownout_int_init() {
    gpio_config_t io_conf = {
        .pin_bit_mask = (1ULL << GPIO_BROWNOUT),
        .mode = GPIO_MODE_INPUT,
        .pull_up_en = GPIO_PULLUP_ENABLE,
        .intr_type = GPIO_INTR_NEGEDGE,
    };
    gpio_config(&io_conf);
    
    gpio_isr_handler_add(GPIO_BROWNOUT, brownout_int_isr, NULL);
}
```

---

## Calibration Routines

### Factory Calibration (One-time)

```c
typedef struct {
    uint32_t tdc_zero_delay_ps;    // Internal propagation delay
    float tdc_scale_factor;        // mm per picosecond
    uint16_t als_dark_offset;      // Dark current offset
    float als_scale_factor;        // Lux per count
    uint16_t dac_offset_mv;        // DAC zero offset
    float dac_scale_factor;        // mV per LSB
} factory_cal_t;

esp_err_t factory_calibration() {
    factory_cal_t cal = {0};
    
    ESP_LOGI(TAG, "Starting factory calibration...");
    
    // 1. TDC zero-distance calibration
    ESP_LOGI(TAG, "Place no target in front of sensor");
    vTaskDelay(pdMS_TO_TICKS(5000));
    cal.tdc_zero_delay_ps = calibrate_tdc_zero_distance();
    
    // 2. TDC scale calibration
    ESP_LOGI(TAG, "Place target at 500mm");
    vTaskDelay(pdMS_TO_TICKS(5000));
    uint32_t time_500mm = measure_average_tof(100);
    
    ESP_LOGI(TAG, "Place target at 2000mm");
    vTaskDelay(pdMS_TO_TICKS(5000));
    uint32_t time_2000mm = measure_average_tof(100);
    
    cal.tdc_scale_factor = 1500.0 / (time_2000mm - time_500mm); // mm/ps
    
    // 3. ALS calibration
    ESP_LOGI(TAG, "Cover ALS sensor completely");
    vTaskDelay(pdMS_TO_TICKS(5000));
    cal.als_dark_offset = als_read_raw();
    
    ESP_LOGI(TAG, "Illuminate with 500 lux reference");
    vTaskDelay(pdMS_TO_TICKS(5000));
    uint16_t als_500lux = als_read_raw();
    cal.als_scale_factor = 500.0 / (als_500lux - cal.als_dark_offset);
    
    // 4. DAC calibration
    ESP_LOGI(TAG, "Measuring DAC offset and gain");
    cal.dac_offset_mv = calibrate_dac_offset();
    cal.dac_scale_factor = calibrate_dac_scale();
    
    // Save to NVS
    nvs_handle_t nvs_handle;
    ESP_ERROR_CHECK(nvs_open("factory_cal", NVS_READWRITE, &nvs_handle));
    ESP_ERROR_CHECK(nvs_set_blob(nvs_handle, "cal_data", &cal, sizeof(cal)));
    ESP_ERROR_CHECK(nvs_commit(nvs_handle));
    nvs_close(nvs_handle);
    
    ESP_LOGI(TAG, "Factory calibration complete!");
    
    return ESP_OK;
}
```

### Runtime Calibration (Periodic)

```c
void runtime_calibration_check() {
    static uint32_t last_cal_time_ms = 0;
    uint32_t now_ms = millis();
    
    // Calibrate every 10 minutes
    if ((now_ms - last_cal_time_ms) > 600000) {
        ESP_LOGI(TAG, "Runtime calibration");
        
        // TDC internal calibration (compensates temperature drift)
        tdc7200_trigger_cal();
        vTaskDelay(pdMS_TO_TICKS(100));
        uint32_t cal1 = tdc7200_read_reg24(TDC_CALIBRATION1);
        uint32_t cal2 = tdc7200_read_reg24(TDC_CALIBRATION2);
        
        // Update calibration coefficients
        tdc7200_update_cal_coeff(cal1, cal2);
        
        last_cal_time_ms = now_ms;
    }
}
```

---

**Document Version:** 1.0  
**Last Updated:** 2024-12-14  

For related documentation, see:
- [Schematic Documentation](/docs/SCHEMATICS_LIDAR_COMPLETE.md)
- [Design Notes](/docs/DESIGN_NOTES_COMPLETE.md)
- [Integration Guide](/docs/INTEGRATION_GUIDE.md)
- [Energy Buffer Design](/docs/ENERGY_BUFFER.md)
