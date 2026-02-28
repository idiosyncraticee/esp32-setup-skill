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

### 3. esp-tls PRIV_REQUIRES naming
- The component name is `esp-tls` (hyphen), NOT `esp_tls` (underscore) — the underscore variant fails to resolve
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
    esp-tls
```

### 5. Component Dependencies (known working versions)
```yaml
# idf_component.yml
dependencies:
  idf: ">=5.1"
  espressif/es8311: "^1.0.0"
  espressif/esp_io_expander_tca9554: "^1.0.1"
  espressif/esp_lcd_sh8601: "^2.0.1"
  espressif/esp_jpeg: "^1.3.1"
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
CONFIG_MBEDTLS_EXTERNAL_MEM_ALLOC=y
CONFIG_MBEDTLS_DYNAMIC_BUFFER=y
CONFIG_MBEDTLS_DYNAMIC_FREE_PEER_CERT=y
CONFIG_MBEDTLS_DYNAMIC_FREE_CONFIG_DATA=y
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

### 8b. LVGL canvas for image display
Map an RGB565 buffer directly to an LVGL canvas — zero-copy, just invalidate on update:
```c
// Create once:
canvas = lv_canvas_create(lv_scr_act());
lv_canvas_set_buffer(canvas, rgb565_buf,
                     DISPLAY_WIDTH, DISPLAY_HEIGHT, LV_IMG_CF_TRUE_COLOR);
lv_obj_center(canvas);

// Update after decoding a new image:
lv_obj_invalidate(canvas);  // triggers LVGL to re-flush the buffer
```
Always wrap LVGL calls in `display_lock()` / `display_unlock()` (mutex) since LVGL is not thread-safe and the poller task runs on a different core than the LVGL task.

### 8c. LVGL animations
```c
lv_anim_t a;
lv_anim_init(&a);
lv_anim_set_var(&a, obj);
lv_anim_set_exec_cb(&a, (lv_anim_exec_xcb_t)lv_obj_set_y);
lv_anim_set_values(&a, start_y, end_y);
lv_anim_set_time(&a, 2000);
lv_anim_set_playback_time(&a, 2000);
lv_anim_set_repeat_count(&a, LV_ANIM_REPEAT_INFINITE);
lv_anim_set_path_cb(&a, lv_anim_path_ease_in_out);
lv_anim_start(&a);
```
- Cast `lv_obj_set_y` (or `_x`, `_opa`, etc.) to `lv_anim_exec_xcb_t`
- `lv_anim_set_playback_time` makes it ping-pong
- Must be called from within `display_lock()` context

### 9. Audio init sequence — CRITICAL ORDER
The ES8311 codec needs MCLK running during initialization to lock its PLL. **I2S channels must be enabled BEFORE ES8311 init.** Without this, `i2s_channel_read()` returns timeout and records 0 bytes.

Correct init order:
1. Init legacy I2C bus (400kHz, internal pullups)
2. Enable PA via GPIO46
3. Init I2S std mode (stereo, Philips format) with `auto_clear = true` and **MCLK multiple = 384**
4. **Enable BOTH TX and RX channels immediately** — this starts MCLK output
5. THEN init ES8311 (MCLK is now running):
```c
es8311_clock_config_t clk_cfg = {
    .mclk_from_mclk_pin = true,
    .mclk_frequency = SAMPLE_RATE * 384,   // MUST match I2S mclk_multiple
    .sample_frequency = SAMPLE_RATE,        // MUST set — 0 causes init failure
};
es8311_init(handle, &clk_cfg, ES8311_RESOLUTION_16, ES8311_RESOLUTION_16);
// 4th param is resolution NOT sample rate — ES8311_RESOLUTION_16, not 16000
```
6. Call `es8311_sample_frequency_config(handle, SAMPLE_RATE * 384, SAMPLE_RATE)` — explicitly sets codec internal dividers
7. I2S config must set `std_cfg.clk_cfg.mclk_multiple = 384` to match the ES8311 clock config
8. **After ES8311 init, DISABLE both channels** — prevents DMA queue overflow during idle period before first recording
9. I2S reads are stereo — extract left channel only for mono recording
10. Recording task needs **8192 bytes stack** (960-byte I2S buffer + ESP-IDF overhead)
11. Use `volatile bool recording` flag — written by main task, read by recording task
12. Race condition: when stopping recording, wait 300ms after setting flag before reading `record_pos`

### 9b. Recording session lifecycle — CRITICAL DMA PATTERN
The I2S DMA queue stalls if channels are left enabled without reading. Channels must be enabled per-session and reading must start IMMEDIATELY.

```
Enable TX+RX → read loop (NO delay) → set recording=false → loop exits → disable RX+TX
```

Key rules:
1. **Enable both TX and RX** for recording — TX drives MCLK/BCLK/WS clocks, RX captures data
2. **Start reading IMMEDIATELY after enable** — any delay lets DMA buffers fill, queue overflows, and all subsequent reads timeout (0x107 ESP_ERR_TIMEOUT)
3. **Do NOT add a delay** between channel enable and first read — PLL lock time (~1ms) is covered by the first read's 100ms timeout
4. **Read buffer should match DMA frame size**: 240 frames × 4 bytes (stereo 16-bit) = 960 bytes
5. **Disable BOTH channels after recording ends** — prevents DMA queue stall before next session
6. **Do NOT disable only RX while TX stays enabled** — this kills shared clocks and causes permanent read timeout
7. For playback: enable TX only → write → disable TX

### 10. mbedTLS memory — MUST use PSRAM
The default `CONFIG_MBEDTLS_INTERNAL_MEM_ALLOC=y` allocates TLS buffers from scarce internal SRAM. With large image buffers in PSRAM, internal RAM runs out and TLS fails with `mbedtls_ssl_setup returned -0x7F00` (`MBEDTLS_ERR_SSL_ALLOC_FAILED`).

**Fix**: Add to `sdkconfig.defaults`:
```
CONFIG_MBEDTLS_EXTERNAL_MEM_ALLOC=y
CONFIG_MBEDTLS_DYNAMIC_BUFFER=y
CONFIG_MBEDTLS_DYNAMIC_FREE_PEER_CERT=y
CONFIG_MBEDTLS_DYNAMIC_FREE_CONFIG_DATA=y
```

**Important**: After changing sdkconfig, run `idf.py fullclean` then rebuild. Cached builds do NOT pick up sdkconfig changes reliably.

### 11. JPEG decoding — component, API, and limitations
Add `espressif/esp_jpeg: "^1.3.1"` to `idf_component.yml`. The ESP32 JPEG decoder (tjpgd) only supports **baseline JPEG**:
- No progressive JPEG
- No PNG (header `89 50 4E 47`)
- No WebP

If fetching images from external APIs, ensure the server returns baseline JPEG. Many services (including picsum.photos) serve PNG even with `.jpg` extension. Use `Accept: image/jpeg` header and/or convert server-side.

To verify format, log the first 4 bytes of downloaded data:
- `FF D8 FF E0` or `FF D8 FF E1` = JPEG (good)
- `89 50 4E 47` = PNG (won't decode)
- `52 49 46 46` = WebP (won't decode)

**Decode JPEG → RGB565 for display:**
```c
#include "jpeg_decoder.h"

esp_jpeg_image_cfg_t jpeg_cfg = {
    .indata = jpeg_buf,
    .indata_size = jpeg_len,
    .outbuf = rgb565_buf,
    .outbuf_size = RGB565_BUF_SIZE,
    .out_format = JPEG_IMAGE_FORMAT_RGB565,
    .out_scale = JPEG_IMAGE_SCALE_0,
    .flags = { .swap_color_bytes = 1 },  // Required for SH8601 display
};
esp_jpeg_image_output_t out_info;
esp_jpeg_decode(&jpeg_cfg, &out_info);
// out_info.width, out_info.height available after decode
```
- `swap_color_bytes = 1` is **critical** for the SH8601 — without it colors are garbled
- Allocate `jpeg_buf` in PSRAM (`MALLOC_CAP_SPIRAM`) — images can be 100KB-512KB
- Allocate `rgb565_buf` in PSRAM too — 368×448×2 = ~322KB

### 12. LVGL canvas animation (bounce, zoom)
To animate a canvas (e.g., gentle bounce), use LVGL's animation system:
- **Callback type mismatch**: `lv_obj_set_y` takes `lv_coord_t` (int16_t) but `lv_anim_exec_xcb_t` expects `void (*)(void*, int32_t)`. Always use a wrapper:
```c
static void bounce_anim_cb(void *obj, int32_t v) {
    lv_obj_set_y((lv_obj_t *)obj, (lv_coord_t)v);
}
```
- **Canvas created in two places**: If the canvas is created early (e.g., in the init/task function) and `show_on_display` checks `if (!canvas)`, any setup code inside that block (zoom, animation start) will never execute. Use a separate `static bool` flag or apply settings where the canvas is **first** created.
- **Zoom to hide bounce edges**: A full-screen canvas bouncing ±N pixels reveals the background at edges. Use `lv_img_set_zoom(canvas, val)` where 256 = 1.0x. For ±15px bounce on 448px height: `lv_img_set_zoom(canvas, 280)` (~1.09x). Apply zoom where the canvas is **first created**, not in a conditional branch that may be skipped.
- **Bounce recipe**:
```c
lv_anim_t a;
lv_anim_init(&a);
lv_anim_set_var(&a, canvas);
lv_anim_set_exec_cb(&a, bounce_anim_cb);
lv_coord_t center_y = (LV_VER_RES - DISPLAY_HEIGHT) / 2;
lv_anim_set_values(&a, center_y - 15, center_y + 15);  // ±15px amplitude
lv_anim_set_time(&a, 2000);              // 2s per direction
lv_anim_set_playback_time(&a, 2000);     // 2s return
lv_anim_set_repeat_count(&a, LV_ANIM_REPEAT_INFINITE);
lv_anim_set_path_cb(&a, lv_anim_path_ease_in_out);
lv_anim_start(&a);
```

### 13. Network diagnostics pattern
University/guest WiFi often blocks port 443 while allowing port 80. Add startup diagnostics before image polling:
1. DNS resolution via `getaddrinfo()` — confirms DNS works
2. HTTP GET to `http://example.com` (port 80) — confirms internet access, detects captive portals (302 redirect)
3. Raw TCP connect to target host port 443 — confirms HTTPS port is open
4. Full HTTPS request to `https://example.com` — confirms TLS works end-to-end

Requires: `#include "lwip/netdb.h"`, `#include "lwip/sockets.h"`, `#include <fcntl.h>`

### 14. HTTPS API calls
- Use `esp_crt_bundle_attach` in `esp_http_client_config_t` for TLS cert validation
- Set `timeout_ms` to at least 20000 (20s) for slow/guest networks
- Groq API supports both STT (`whisper-large-v3-turbo`) and TTS (`canopylabs/orpheus-v1-english`)
- Single API key works for both: `https://api.groq.com/openai/v1/audio/transcriptions` and `https://api.groq.com/openai/v1/audio/speech`
- Multipart POST for STT: manual boundary construction with `esp_http_client_open()`/`_write()` for streaming large audio
- TTS returns WAV — parse the 44-byte header for sample rate before playback

### 15. Local secrets via sdkconfig.defaults.local
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
