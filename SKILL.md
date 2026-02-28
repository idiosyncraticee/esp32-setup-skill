---
name: esp32-setup
description: Scaffold an ESP-IDF project for the Waveshare ESP32-S3-Touch-AMOLED-1.8 board with audio, display, WiFi, and API integration
argument-hint: "[optional: description of what the project should do]"
---

# ESP32-S3 Waveshare AMOLED 1.8" Project Setup

Set up a complete ESP-IDF project for the **Waveshare ESP32-S3-Touch-AMOLED-1.8** board.

User's project description: $ARGUMENTS

## Board Specs

- **MCU**: ESP32-S3R8, 16MB flash, 8MB PSRAM
- **Audio**: ES8311 codec (I2C addr 0x18), I2S interface
- **Display**: SH8601 1.8" AMOLED (368x448), QSPI interface
- **I/O Expander**: TCA9554 (I2C addr 0x20) — controls display reset
- **Touch**: FT3168 (INT GPIO21)
- **Speaker PA**: GPIO46

## Verified Pin Configuration

| Function | GPIO |
|----------|------|
| I2C SDA / SCL (shared bus) | 15 / 14 |
| I2S MCLK / BCLK / WS / DOUT / DIN | 16 / 9 / 45 / 8 / 10 |
| Speaker PA enable | 46 |
| Display QSPI CS / CLK / D0-D3 | 12 / 11 / 4 / 5 / 6 / 7 |
| Touch INT | 21 |
| Boot button | 0 |

## ESP-IDF Installation

If `~/esp/esp-idf/` does not exist, install it:

```bash
mkdir -p ~/esp && cd ~/esp
git clone -b v5.3.2 --recursive https://github.com/espressif/esp-idf.git
cd esp-idf && ./install.sh esp32s3
```

The "detached HEAD" warning on clone is normal — ESP-IDF releases are tags, not branches.

To activate in any shell session: `. ~/esp/esp-idf/export.sh`

**Important**: Use dot-sourcing (`. file`) not `source file` — the latter may not persist PATH in all shell contexts.

## Critical Lessons Learned (from debugging real builds)

### 1. Legacy I2C Driver Required
The ES8311 (`espressif/es8311 ^1.0.0`) and TCA9554 (`espressif/esp_io_expander_tca9554 ^1.0.1`) components use the **legacy I2C driver API** (`driver/i2c.h`), NOT the new I2C master driver (`driver/i2c_master.h`).

- Use `i2c_param_config()` + `i2c_driver_install()`, NOT `i2c_new_master_bus()`
- Pass `i2c_port_t` (e.g., `I2C_NUM_0`) to `es8311_create()` and `esp_io_expander_new_i2c_tca9554()`, NOT a bus handle

### 2. Required includes (commonly missed)
- `ESP_RETURN_ON_ERROR` macro requires `#include "esp_check.h"` — add this to ANY .c file that uses it (audio, wifi, display, etc.)
- `esp_timer_create()`, `esp_timer_start_periodic()`, `esp_timer_handle_t`, `esp_timer_create_args_t` require `#include "esp_timer.h"`
- These are NOT pulled in transitively by other ESP-IDF headers — you must include them explicitly

### 3. esp_tls is NOT a PRIV_REQUIRES component
- Do NOT add `esp_tls` to `PRIV_REQUIRES` — it will fail to resolve
- For `esp_crt_bundle_attach`, include `"esp_crt_bundle.h"` and add `mbedtls` to `PRIV_REQUIRES`

### 4. CMakeLists PRIV_REQUIRES (known working set)
```cmake
PRIV_REQUIRES
    driver
    esp_wifi
    esp_http_client
    esp_lcd
    nvs_flash
    esp_psram
    json
    mbedtls
```

### 5. Component Dependencies (known working versions)
```yaml
# idf_component.yml
dependencies:
  idf: ">=5.1"
  espressif/es8311: "^1.0.0"
  espressif/esp_io_expander_tca9554: "^1.0.1"
  espressif/esp_lcd_sh8601: "^2.0.1"
  esp_lcd_touch_ft5x06: "*"
  lvgl/lvgl: "8.4.*"
```

### 6. sdkconfig.defaults essentials
```
CONFIG_IDF_TARGET="esp32s3"
CONFIG_ESPTOOLPY_FLASHSIZE_16MB=y
CONFIG_ESPTOOLPY_FLASHMODE_DIO=y
CONFIG_ESPTOOLPY_FLASHFREQ_80M=y
CONFIG_SPIRAM=y
CONFIG_SPIRAM_MODE_OCT=y
CONFIG_SPIRAM_SPEED_80M=y
CONFIG_PARTITION_TABLE_CUSTOM=y
CONFIG_PARTITION_TABLE_CUSTOM_FILENAME="partitions.csv"
CONFIG_MBEDTLS_CERTIFICATE_BUNDLE=y
CONFIG_MBEDTLS_CERTIFICATE_BUNDLE_DEFAULT_FULL=y
CONFIG_LV_COLOR_16_SWAP=y
CONFIG_LV_MEM_CUSTOM=y
CONFIG_LV_MEMCPY_MEMSET_STD=y
CONFIG_LV_FONT_MONTSERRAT_16=y
CONFIG_LV_FONT_MONTSERRAT_20=y
CONFIG_LV_FONT_MONTSERRAT_24=y
```

### 7. Partition table for 16MB flash
```csv
# Name,   Type, SubType, Offset,   Size,    Flags
nvs,      data, nvs,     ,         0x6000,
phy_init, data, phy,     ,         0x1000,
factory,  app,  factory, ,         0xF00000,
```

### 8. Display init sequence (verified working from Waveshare reference)
1. Init TCA9554 via legacy I2C
2. Set EXIO0/1/2 as outputs
3. Reset: set ALL three pins (EXIO0/1/2) LOW → wait 200ms → set ALL three HIGH
4. Init SPI bus using `SH8601_PANEL_BUS_QSPI_CONFIG()` macro
5. Init panel IO using `SH8601_PANEL_IO_QSPI_CONFIG()` macro with async flush callback
6. Provide **custom init commands** (the driver defaults do NOT work for this panel):
```c
static const sh8601_lcd_init_cmd_t lcd_init_cmds[] = {
    {0x11, (uint8_t[]){0x00}, 0, 120},                          // Sleep out
    {0x44, (uint8_t[]){0x01, 0xD1}, 2, 0},                      // Tear effect line
    {0x35, (uint8_t[]){0x00}, 1, 0},                             // Tear effect on
    {0x53, (uint8_t[]){0x20}, 1, 10},                            // Write control display
    {0x2A, (uint8_t[]){0x00, 0x00, 0x01, 0x6F}, 4, 0},          // Column address set
    {0x2B, (uint8_t[]){0x00, 0x00, 0x01, 0xBF}, 4, 0},          // Row address set
    {0x51, (uint8_t[]){0x00}, 1, 10},                            // Brightness off initially
    {0x29, (uint8_t[]){0x00}, 0, 10},                            // Display on
    {0x51, (uint8_t[]){0xFF}, 1, 0},                             // Brightness max
};
```
7. Set `vendor_config.init_cmds` and `vendor_config.init_cmds_size` — do NOT use `NULL`
8. Use `rgb_ele_order = LCD_RGB_ELEMENT_ORDER_RGB` and `bits_per_pixel = 16`
9. LVGL flush callback must be **async**: call `lv_disp_flush_ready()` from the SPI completion callback, NOT synchronously in the flush function
10. Set `disp_drv.rounder_cb` — SH8601 requires coordinates rounded to even boundaries:
```c
area->x1 = (area->x1 >> 1) << 1;
area->y1 = (area->y1 >> 1) << 1;
area->x2 = ((area->x2 >> 1) << 1) + 1;
area->y2 = ((area->y2 >> 1) << 1) + 1;
```
11. LVGL buffers: 1/4 screen double buffer, `MALLOC_CAP_DMA` with PSRAM fallback
12. LVGL task: 8192 stack on core 1

### 9. Audio init sequence
1. Init legacy I2C bus (400kHz, internal pullups)
2. `es8311_create(I2C_NUM_0, 0x18)` → init with clock config:
```c
es8311_clock_config_t clk_cfg = {
    .mclk_from_mclk_pin = true,
    .mclk_frequency = SAMPLE_RATE * 256,   // MUST set — 0 causes init failure
    .sample_frequency = SAMPLE_RATE,        // MUST set — 0 causes init failure
};
es8311_init(handle, &clk_cfg, ES8311_RESOLUTION_16, ES8311_RESOLUTION_16);
// 4th param is resolution NOT sample rate — ES8311_RESOLUTION_16, not 16000
```
3. Init I2S std mode (stereo, Philips format) — do NOT enable channels at init
4. Enable PA via GPIO46
5. For recording: enable I2S RX channel, read in loop, disable when done
6. For playback: enable I2S TX channel, write data, disable when done
7. I2S reads are stereo — extract left channel only for mono recording
8. Recording task needs **8192 bytes stack** (1024-byte I2S buffer + ESP-IDF overhead)
9. Race condition: when stopping recording, wait 200ms after setting flag before reading `record_pos`

### 10. HTTPS API calls
- Use `esp_crt_bundle_attach` in `esp_http_client_config_t` for TLS cert validation
- Groq API supports both STT (`whisper-large-v3-turbo`) and TTS (`canopylabs/orpheus-v1-english`)
- Single API key works for both: `https://api.groq.com/openai/v1/audio/transcriptions` and `https://api.groq.com/openai/v1/audio/speech`
- Multipart POST for STT: manual boundary construction with `esp_http_client_open()`/`_write()` for streaming large audio
- TTS returns WAV — parse the 44-byte header for sample rate before playback

### 11. Local secrets via sdkconfig.defaults.local
ESP-IDF does NOT natively load `sdkconfig.defaults.local` — you must configure it in `CMakeLists.txt`:

```cmake
# In top-level CMakeLists.txt, BEFORE include($ENV{IDF_PATH}/tools/cmake/project.cmake)
if(EXISTS "${CMAKE_SOURCE_DIR}/sdkconfig.defaults.local")
    set(SDKCONFIG_DEFAULTS "sdkconfig.defaults;sdkconfig.defaults.local")
endif()
```

This loads `sdkconfig.defaults` first, then overlays `sdkconfig.defaults.local` (local values win). Use the local file for credentials — never commit it to git.

```
# sdkconfig.defaults.local — gitignored
CONFIG_WIFI_SSID="MyNetwork"
CONFIG_WIFI_PASSWORD="MyPassword"
CONFIG_GROQ_API_KEY="gsk_xxxx"
```

After changing, delete `sdkconfig` and rebuild so the new defaults are picked up.

Always add to `.gitignore`:
```
build/
sdkconfig
sdkconfig.old
sdkconfig.defaults.local
managed_components/
dependencies.lock
```

## Reference Documentation

When adding new features or peripherals, fetch these resources for up-to-date API details:

### Board Documentation
- **Waveshare Wiki**: https://www.waveshare.com/wiki/ESP32-S3-Touch-AMOLED-1.8 — schematic, pinout, example code links
- **Waveshare Demo Code**: https://github.com/waveshareteam/Waveshare-ESP32-components — reference implementations for all onboard peripherals
- **Waveshare AMOLED 1.8 Reference**: https://github.com/waveshareteam/ESP32-S3-Touch-AMOLED-1.8 — engineering sample with verified display init commands, LVGL setup, and I2S codec example

### ESP-IDF Component Registry
Search and browse components at https://components.espressif.com/ — use `idf.py add-dependency` to add new ones.

Key components already in use:
- https://components.espressif.com/components/espressif/es8311
- https://components.espressif.com/components/espressif/esp_io_expander_tca9554
- https://components.espressif.com/components/espressif/esp_lcd_sh8601
- https://components.espressif.com/components/lvgl/lvgl

### ESP-IDF Programming Guides
- **I2C**: https://docs.espressif.com/projects/esp-idf/en/v5.3.2/esp32s3/api-reference/peripherals/i2c.html
- **I2S**: https://docs.espressif.com/projects/esp-idf/en/v5.3.2/esp32s3/api-reference/peripherals/i2s.html
- **SPI**: https://docs.espressif.com/projects/esp-idf/en/v5.3.2/esp32s3/api-reference/peripherals/spi_master.html
- **GPIO**: https://docs.espressif.com/projects/esp-idf/en/v5.3.2/esp32s3/api-reference/peripherals/gpio.html
- **LCD**: https://docs.espressif.com/projects/esp-idf/en/v5.3.2/esp32s3/api-reference/peripherals/lcd.html
- **HTTP Client**: https://docs.espressif.com/projects/esp-idf/en/v5.3.2/esp32s3/api-reference/protocols/esp_http_client.html
- **WiFi**: https://docs.espressif.com/projects/esp-idf/en/v5.3.2/esp32s3/api-reference/network/esp_wifi.html

### API Documentation
- **Groq STT**: https://console.groq.com/docs/speech-to-text
- **Groq TTS**: https://console.groq.com/docs/text-to-speech

### Adding New Peripherals
When the user wants to add a peripheral (accelerometer, GPS, etc.):
1. Fetch the Waveshare wiki page above to check if the board has it and get the GPIO/I2C address
2. Search the ESP-IDF component registry for a driver
3. If no component exists, fetch the ESP-IDF I2C/SPI docs and write a driver using the legacy I2C API
4. Add the component to `idf_component.yml` and any new `PRIV_REQUIRES` to `main/CMakeLists.txt`

## Instructions

Based on the user's project description (or a general voice assistant if none given):

1. Check if ESP-IDF is installed at `~/esp/esp-idf/`; if not, install it
2. Create the project scaffold: `CMakeLists.txt`, `sdkconfig.defaults`, `partitions.csv`, `main/CMakeLists.txt`, `main/idf_component.yml`, `main/Kconfig.projbuild`, `main/config.h`
3. Create `.gitignore` and `sdkconfig.defaults.local` (with placeholder credentials)
4. Create source modules as needed for the project (audio, display, wifi, etc.)
5. Create `main.c` with the application logic
6. Build with `. ~/esp/esp-idf/export.sh && idf.py build` and fix any errors
7. Report the build status and next steps (fill in `sdkconfig.defaults.local`, then `rm sdkconfig && idf.py build && idf.py flash monitor`)
