# Guition JC1060P470C_I_W_Y: building firmware from scratch (ESP-IDF)

All coded using claude, trail and error and referencing https://github.com/wegi1/ESP32P4-JC1060P470C-I_W_Y 

Everything needed to bring up this 7" ESP32-P4 panel with plain ESP-IDF + LVGL. It covers the hardware map, toolchain,
component versions, `sdkconfig.defaults`, partition table, the display init table and timings that actually work,
touch, backlight, Wi-Fi / Bluetooth through the ESP32-C6, and the OV02C10 camera. All of it is taken from a
working build, and every setting that cost a debugging session is called out.

No credentials are in this file. Put yours in `main/secrets.h` (template at the end) and keep that file out of git.

---

## 1. Hardware

| Part | Detail |
|---|---|
| SoC | ESP32-P4, **revision 1.3** (pre-v3 silicon), 16 MB flash, 32 MB PSRAM (hex mode) |
| Wireless | ESP32-C6 co-processor running ESP-Hosted, SDIO 4-bit to the P4 (Wi-Fi 6 + BLE 5) |
| Display | 7" 1024x600 IPS, **JD9165** driver, MIPI-DSI 2 lanes. Recent units carry the **"V2" panel** (see section 5) |
| Touch | GT911 capacitive, I2C |
| Camera | 15-pin 0.3 mm FPC (Raspberry-Pi pinout), MIPI-CSI 2 lanes, OV02C10 module in the vendor kit |

### Pins

| Function | GPIO | Notes |
|---|---|---|
| LCD reset | **0** | some docs say GPIO5; this unit needs GPIO0 |
| Backlight PWM | 23 | LEDC, 20 kHz, 10-bit |
| Touch SDA / SCL | 7 / 8 | I2C0. **Shared with the camera connector's SCCB** |
| Touch RST / INT | 22 / 21 | reset done by hand, see section 6 |
| C6 SDIO CMD / CLK | 19 / 18 | |
| C6 SDIO D0..D3 | 14, 15, 16, 17 | |
| C6 reset | 54 | |
| DSI PHY supply | internal LDO channel 3 at 2500 mV | `esp_ldo_acquire_channel` before creating the DSI bus |
| Camera power enable | pulled high (10 k to 3V3) | always on, no GPIO |
| Camera XCLK | none | the module has its own oscillator |

---

## 2. Toolchain

- **ESP-IDF 5.5.5.** On this rev-1.3 P4 with IDF 5.5.1, DSI commands sent before the video stream started never left the command FIFO.
- Required for rev-1.3 silicon: `CONFIG_ESP32P4_SELECTS_REV_LESS_V3=y`, `CONFIG_ESP32P4_REV_MIN_100=y`.
- PSRAM at 200 MHz needs `CONFIG_IDF_EXPERIMENTAL_FEATURES=y`.
- **Windows:** keep the project path short and free of spaces if you can. If you can't:
  - set `IDF_COMPONENT_CACHE_PATH` to a short folder such as `C:\idfcc`. `esp_video`'s bundled examples exceed the 260-character path limit.
  - the camera IPA generator splits its input paths on spaces, so point the OV02C10 IPA JSON option at a copy in a folder without spaces (section 9).

### `main/idf_component.yml`

```yaml
dependencies:
  idf: ">=5.5.0"
  lvgl/lvgl: "~9.3.0"
  espressif/esp_lvgl_port: "~2.6.0"
  espressif/esp_lcd_touch_gt911: "^1.1.3"
  espressif/esp_hosted: "~2.12.0"
  espressif/esp_wifi_remote: "*"
  espressif/esp_websocket_client: "^1.4.0"
  espressif/mdns: "^1.8.0"
```

Version pins that matter:
- `esp_lvgl_port` **~2.6**. 2.9 needs a newer IDF than 5.5.
- `lvgl` **~9.3**. 9.2 lacks `RGB565_SWAPPED`.
- `esp_hosted` ~2.12 with `esp_wifi_remote` for Wi-Fi through the C6.

For the camera, add `esp_video` 1.1 and the vendor's `esp_cam_sensor` as local components (section 9).

---

## 3. `sdkconfig.defaults`

```ini
# ---- Chip: ESP32-P4 revision 1.3 (pre-v3 silicon, "engineering sample") ----
CONFIG_IDF_TARGET="esp32p4"
CONFIG_ESP32P4_SELECTS_REV_LESS_V3=y
CONFIG_ESP32P4_REV_MIN_100=y
CONFIG_IDF_EXPERIMENTAL_FEATURES=y
CONFIG_ESP_DEFAULT_CPU_FREQ_MHZ_360=y

# ---- Flash / partitions ----
CONFIG_ESPTOOLPY_FLASHSIZE_16MB=y
CONFIG_ESPTOOLPY_FLASHMODE_DIO=y
CONFIG_ESPTOOLPY_FLASHFREQ_80M=y
CONFIG_PARTITION_TABLE_CUSTOM=y
CONFIG_PARTITION_TABLE_CUSTOM_FILENAME="partitions.csv"
CONFIG_BOOTLOADER_APP_ROLLBACK_ENABLE=y

# ---- PSRAM (32 MB, hex mode, 200 MHz) ----
CONFIG_SPIRAM=y
CONFIG_SPIRAM_MODE_HEX=y
CONFIG_SPIRAM_SPEED_200M=y
CONFIG_SPIRAM_USE_MALLOC=y
CONFIG_SPIRAM_MALLOC_ALWAYSINTERNAL=4096
CONFIG_SPIRAM_TRY_ALLOCATE_WIFI_LWIP=y
CONFIG_SPIRAM_XIP_FROM_PSRAM=y

# ---- FreeRTOS / misc ----
CONFIG_FREERTOS_HZ=1000
CONFIG_ESP_MAIN_TASK_STACK_SIZE=8192
CONFIG_FREERTOS_THREAD_LOCAL_STORAGE_POINTERS=1
CONFIG_ESP_CONSOLE_USB_SERIAL_JTAG=y
CONFIG_COMPILER_OPTIMIZATION_PERF=y

# ---- Wi-Fi through the ESP32-C6 co-processor (ESP-Hosted over SDIO, 4-bit) ----
CONFIG_ESP_HOSTED_ENABLED=y
CONFIG_ESP_HOSTED_P4_DEV_BOARD_FUNC_BOARD=y
CONFIG_SLAVE_IDF_TARGET_ESP32C6=y
CONFIG_ESP_HOSTED_SDIO_HOST_INTERFACE=y
CONFIG_ESP_HOSTED_SDIO_4_BIT_BUS=y
CONFIG_ESP_HOSTED_SDIO_CLOCK_FREQ_KHZ=10000
CONFIG_ESP_HOSTED_SDIO_PIN_CMD=19
CONFIG_ESP_HOSTED_SDIO_PIN_CLK=18
CONFIG_ESP_HOSTED_SDIO_PIN_D0=14
CONFIG_ESP_HOSTED_SDIO_PIN_D1=15
CONFIG_ESP_HOSTED_SDIO_PIN_D2=16
CONFIG_ESP_HOSTED_SDIO_PIN_D3=17
CONFIG_ESP_HOSTED_SDIO_GPIO_RESET_SLAVE=54
CONFIG_ESP_HOSTED_USE_MEMPOOL=n
CONFIG_ESP_HOSTED_DFLT_TASK_FROM_SPIRAM=y

# ---- Networking ----
CONFIG_LWIP_SNTP_MAX_SERVERS=2
CONFIG_HTTPD_MAX_REQ_HDR_LEN=1024
CONFIG_ESP_HTTP_CLIENT_ENABLE_HTTPS=n

# ---- LVGL 9 ----
CONFIG_LV_COLOR_DEPTH_16=y
CONFIG_LV_DRAW_SW_DRAW_UNIT_CNT=1
CONFIG_LV_USE_DRAW_PPA=n
CONFIG_LV_FONT_MONTSERRAT_14=y
CONFIG_LV_FONT_MONTSERRAT_16=y
CONFIG_LV_FONT_MONTSERRAT_18=y
CONFIG_LV_FONT_MONTSERRAT_20=y
CONFIG_LV_FONT_MONTSERRAT_24=y
CONFIG_LV_FONT_MONTSERRAT_28=y
CONFIG_LV_FONT_MONTSERRAT_32=y
CONFIG_LV_FONT_MONTSERRAT_40=y
CONFIG_LV_FONT_MONTSERRAT_48=y
CONFIG_LV_FONT_DEFAULT_MONTSERRAT_20=y
CONFIG_LV_USE_PERF_MONITOR=n
CONFIG_LV_USE_LOG=n
CONFIG_LV_USE_CLIB_MALLOC=y
CONFIG_LV_USE_CLIB_STRING=y
CONFIG_LV_USE_CLIB_SPRINTF=y
CONFIG_LV_USE_SNAPSHOT=y
CONFIG_LV_USE_LODEPNG=y

# ---- Bluetooth: NimBLE host on the P4, controller on the ESP32-C6 via ESP-Hosted (VHCI over SDIO) ----
CONFIG_BT_ENABLED=y
CONFIG_BT_NIMBLE_ENABLED=y
CONFIG_BT_CONTROLLER_DISABLED=y
CONFIG_BT_NIMBLE_TRANSPORT_UART=n
CONFIG_ESP_HOSTED_ENABLE_BT_NIMBLE=y
CONFIG_ESP_HOSTED_NIMBLE_HCI_VHCI=y
CONFIG_BT_NIMBLE_ROLE_CENTRAL=y
CONFIG_BT_NIMBLE_ROLE_OBSERVER=y
CONFIG_BT_NIMBLE_ROLE_PERIPHERAL=n
CONFIG_BT_NIMBLE_ROLE_BROADCASTER=n
CONFIG_BT_NIMBLE_MAX_CONNECTIONS=3
CONFIG_BT_NIMBLE_ATT_PREFERRED_MTU=517
CONFIG_BT_NIMBLE_MEM_ALLOC_MODE_EXTERNAL=y
CONFIG_BT_NIMBLE_EXT_ADV=n
CONFIG_BT_NIMBLE_LOG_LEVEL_WARNING=y
CONFIG_BT_NIMBLE_HOST_TASK_STACK_SIZE=6144
CONFIG_BT_NIMBLE_GATT_MAX_PROCS=8
CONFIG_BT_NIMBLE_MSYS_1_BLOCK_COUNT=24

# ---- ESPHome API encryption (Noise_NNpsk0_25519_ChaChaPoly_SHA256)
CONFIG_MBEDTLS_CHACHA20_C=y
CONFIG_MBEDTLS_POLY1305_C=y
CONFIG_MBEDTLS_CHACHAPOLY_C=y

# Camera: OV02C10 on the 15-pin MIPI-CSI connector (vendor esp_cam_sensor in components/)
CONFIG_CAMERA_OV02C10=y
CONFIG_CAMERA_OV02C10_MIPI_RAW10_1288x728_30FPS=y
CONFIG_CAMERA_OV02C10_ISP_AF_ENABLE=n
CONFIG_ESP_VIDEO_ENABLE_MIPI_CSI_VIDEO_DEVICE=y
CONFIG_ESP_VIDEO_ENABLE_ISP_PIPELINE_CONTROLLER=y
CONFIG_ISP_PIPELINE_CONTROLLER_TASK_STACK_USE_PSRAM=y
CONFIG_ESP_VIDEO_ENABLE_HW_H264_VIDEO_DEVICE=n
CONFIG_ESP_VIDEO_ENABLE_HW_JPEG_VIDEO_DEVICE=n
# the IPA generator splits paths on spaces: keep a copy of components/esp_cam_sensor/sensors/ov02c10/cfg/ov02c10_default.json here
CONFIG_CAMERA_OV02C10_CUSTOMIZED_IPA_JSON_CONFIGURATION_FILE=y
CONFIG_CAMERA_OV02C10_CUSTOMIZED_IPA_JSON_CONFIGURATION_FILE_PATH="<PATH-WITHOUT-SPACES>/ov02c10_default.json"

# two web servers (status/OTA + camera stream), HA websocket, ESPHome API, FTP radar, mDNS: 10 sockets is not enough
CONFIG_LWIP_MAX_SOCKETS=32
CONFIG_LWIP_MAX_ACTIVE_TCP=32
```

---

## 4. `partitions.csv` (two 6 MB OTA slots)

```csv
# Name,   Type, SubType, Offset,   Size
nvs,      data, nvs,     0x9000,   0x6000
otadata,  data, ota,     0xf000,   0x2000
phy_init, data, phy,     0x11000,  0x1000
ota_0,    app,  ota_0,   0x20000,  0x600000
ota_1,    app,  ota_1,   0x620000, 0x600000
```

---

## 5. Display (JD9165, MIPI-DSI)

### Bus and timing

| Setting | Value |
|---|---|
| DSI lanes / lane rate | 2 lanes, 750 Mbps |
| DBI (command) IO | virtual channel 0, 8-bit commands, 8-bit params |
| Pixel format | RGB565 |
| **Pixel clock** | **48 MHz** |
| H sync / back porch / front porch | 24 / 136 / 160 |
| V sync / back porch / front porch | 2 / 21 / 12 |
| Frame buffers | 1 |
| Reset | GPIO0, then **wait 120 ms** before the init table |

The pixel clock is the setting that bites:
- The DPI clock is divided from a 240 MHz PLL.
- **52 MHz** is the vendor's and ESPHome's value, but it isn't an exact divider of 240. The host and bridge timing drift, which shows as washed-out vertical bands.
- **40 MHz** divides exactly, but the panel then shows **every hue as its opposite** (red as cyan, green as magenta, blue as yellow) while greys stay correct. It looks like a broken panel or a swapped colour order, but neither MADCTL BGR, INVON nor re-sending the table fixes it.
- **48 MHz** (240 / 5) gives correct colours and no bands.

The framebuffer screenshot can't show either problem, because it reads LVGL's buffer, not the glass. Check colours with a test pattern of red, green, blue, yellow, cyan, magenta, white and grey bars plus a grey ramp, and photograph it.

### Order of operations

1. Acquire LDO channel 3 at 2500 mV (DSI PHY). Wait 120 ms.
2. Create the DSI bus and the DBI panel IO.
3. Create the JD9165 panel and pulse reset on GPIO0. Wait 120 ms.
4. **Start the DPI video stream, then send the init table** (ESPHome's order). Skip the driver's panel-ID read: if the panel doesn't answer a DSI read, every later command hangs.

### Init table: the "V2 New Panel" sequence

**Guition's original JD9165 table does not initialise the V2 panel.** You get a flat lavender field with stripes. Use this one, the same as ESPControl's V2 device. ESPHome appends PIXFMT / MADCTL / INVOFF / SLPOUT / DISPON after it; they're included at the end.

```c
static const jd9165_lcd_init_cmd_t s_jd9165_init[] = {
    {0x30, (uint8_t[]){0x00}, 1, 0},
    {0xF7, (uint8_t[]){0x49, 0x61, 0x02, 0x00}, 4, 0},
    {0x30, (uint8_t[]){0x01}, 1, 0},
    {0x04, (uint8_t[]){0x0C}, 1, 0},
    {0x05, (uint8_t[]){0x08}, 1, 0},
    {0x0B, (uint8_t[]){0x11}, 1, 0},
    {0x20, (uint8_t[]){0x04}, 1, 0},
    {0x1F, (uint8_t[]){0x05}, 1, 0},
    {0x23, (uint8_t[]){0x38}, 1, 0},
    {0x28, (uint8_t[]){0x18}, 1, 0},
    {0x29, (uint8_t[]){0x29}, 1, 0},
    {0x2A, (uint8_t[]){0x01}, 1, 0},
    {0x2B, (uint8_t[]){0x29}, 1, 0},
    {0x2C, (uint8_t[]){0x01}, 1, 0},
    {0x30, (uint8_t[]){0x02}, 1, 0},
    {0x00, (uint8_t[]){0x05}, 1, 0},
    {0x01, (uint8_t[]){0x22}, 1, 0},
    {0x02, (uint8_t[]){0x08}, 1, 0},
    {0x03, (uint8_t[]){0x12}, 1, 0},
    {0x04, (uint8_t[]){0x16}, 1, 0},
    {0x05, (uint8_t[]){0x64}, 1, 0},
    {0x06, (uint8_t[]){0x00}, 1, 0},
    {0x07, (uint8_t[]){0x00}, 1, 0},
    {0x08, (uint8_t[]){0x78}, 1, 0},
    {0x09, (uint8_t[]){0x00}, 1, 0},
    {0x0A, (uint8_t[]){0x04}, 1, 0},
    {0x0B, (uint8_t[]){0x16, 0x17, 0x0B, 0x0D, 0x0D, 0x0D, 0x11, 0x10, 0x07, 0x07, 0x09}, 11, 0},
    {0x0C, (uint8_t[]){0x09, 0x1E, 0x1E, 0x1C, 0x1C, 0x0D, 0x0D, 0x0D, 0x0D, 0x0D, 0x0D}, 11, 0},
    {0x0D, (uint8_t[]){0x0A, 0x05, 0x0B, 0x0D, 0x0D, 0x0D, 0x11, 0x10, 0x06, 0x06, 0x08}, 11, 0},
    {0x0E, (uint8_t[]){0x08, 0x1F, 0x1F, 0x1D, 0x1D, 0x0D, 0x0D, 0x0D, 0x0D, 0x0D, 0x0D}, 11, 0},
    {0x0F, (uint8_t[]){0x0A, 0x05, 0x0D, 0x0B, 0x0D, 0x0D, 0x11, 0x10, 0x1D, 0x1D, 0x1F}, 11, 0},
    {0x10, (uint8_t[]){0x1F, 0x08, 0x08, 0x06, 0x06, 0x0D, 0x0D, 0x0D, 0x0D, 0x0D, 0x0D}, 11, 0},
    {0x11, (uint8_t[]){0x16, 0x17, 0x0D, 0x0B, 0x0D, 0x0D, 0x11, 0x10, 0x1C, 0x1C, 0x1E}, 11, 0},
    {0x12, (uint8_t[]){0x1E, 0x09, 0x09, 0x07, 0x07, 0x0D, 0x0D, 0x0D, 0x0D, 0x0D, 0x0D}, 11, 0},
    {0x13, (uint8_t[]){0x00, 0x00, 0x00, 0x00}, 4, 0},
    {0x14, (uint8_t[]){0x00, 0x00, 0x41, 0x41}, 4, 0},
    {0x15, (uint8_t[]){0x00, 0x00, 0x00, 0x00}, 4, 0},
    {0x17, (uint8_t[]){0x00}, 1, 0},
    {0x18, (uint8_t[]){0x85}, 1, 0},
    {0x19, (uint8_t[]){0x06, 0x09}, 2, 0},
    {0x1A, (uint8_t[]){0x05, 0x08}, 2, 0},
    {0x1B, (uint8_t[]){0x0A, 0x04}, 2, 0},
    {0x26, (uint8_t[]){0x00}, 1, 0},
    {0x27, (uint8_t[]){0x00}, 1, 0},
    {0x30, (uint8_t[]){0x06}, 1, 0},
    {0x12, (uint8_t[]){0x3F, 0x25, 0x27, 0x35, 0x1D, 0x1B, 0x1B, 0x1A, 0x18, 0x0A, 0x2A, 0x21, 0x19, 0x30}, 14, 0},
    {0x13, (uint8_t[]){0x3F, 0x26, 0x27, 0x35, 0x1E, 0x1C, 0x1C, 0x1A, 0x18, 0x0B, 0x2A, 0x21, 0x19, 0x30}, 14, 0},
    {0x30, (uint8_t[]){0x0A}, 1, 0},
    {0x02, (uint8_t[]){0x4F}, 1, 0},
    {0x0B, (uint8_t[]){0x40}, 1, 0},
    {0x30, (uint8_t[]){0x0D}, 1, 0},
    {0x0D, (uint8_t[]){0x04}, 1, 0},
    {0x10, (uint8_t[]){0x05}, 1, 0},
    {0x11, (uint8_t[]){0x0C}, 1, 0},
    {0x12, (uint8_t[]){0x05}, 1, 0},
    {0x13, (uint8_t[]){0x0C}, 1, 0},
    {0x30, (uint8_t[]){0x00}, 1, 0},
    {0x11, NULL, 0, 120},
    {0x3A, (uint8_t[]){0x55}, 1, 0},   // PIXFMT: RGB565
    {0x36, (uint8_t[]){0x00}, 1, 0},   // MADCTL
    {0x20, NULL, 0, 0},                // INVOFF
    {0x11, NULL, 0, 120},              // SLPOUT
    {0x29, NULL, 0, 20},               // DISPON
};
```

### LVGL (`esp_lvgl_port` 2.6, `lvgl_port_add_disp_dsi`)

- Draw buffer: `1024 * 120` pixels, double-buffered, in PSRAM.
- `sw_rotate = true`, then `lv_display_set_rotation(disp, LV_DISPLAY_ROTATION_180)`. The panel is mounted upside down relative to its scan direction, and the P4's PPA does the rotation.
- Port task: priority 4, 12 KB stack, timer period 5 ms, max sleep 500 ms.
- Faint vertical banding in very light areas remains even at 48 MHz on some units; ESPControl shows the same.

---

## 6. Touch (GT911)

The driver checks the chip only 10 ms after releasing reset, and this GT911 isn't answering yet. Reset it yourself and pass `rst_gpio_num` / `int_gpio_num` as `GPIO_NUM_NC`:

1. Drive INT (21) and RST (22) low. INT low during reset selects address 0x5D.
2. Wait 20 ms, release RST, wait 10 ms.
3. Switch INT to input, then **wait 100 ms**.
4. Probe 0x5D, then 0x14, with a few retries 300 ms apart.

I2C0 on 7/8 at 400 kHz with internal pull-ups. Keep the bus handle: the camera's SCCB uses the same bus.

---

## 7. Backlight

LEDC low-speed timer 0, 10-bit, 20 kHz on GPIO23, channel 0. Duty 0..1023.

---

## 8. Wi-Fi and Bluetooth through the ESP32-C6

- `esp_hosted` + `esp_wifi_remote`: the normal `esp_wifi_*` API works on the P4. SDIO 4-bit at **10 MHz** (pins in section 3) is reliable.
- BLE: NimBLE host on the P4, controller on the C6 over ESP-Hosted VHCI (`CONFIG_ESP_HOSTED_NIMBLE_HCI_VHCI`). Central + observer, 3 connections, MTU 517 and NimBLE memory in PSRAM is enough for an ESPHome-style active Bluetooth proxy.
- Sockets: the default `LWIP_MAX_SOCKETS=10` runs out once you have two HTTP servers, a WebSocket client and an API server. When it does, servers accept and immediately drop connections. Use 32.

---

## 9. Camera (OV02C10 on the CSI connector)

### Connector (15-pin, Raspberry-Pi order)

| Pin | Signal |
|---|---|
| 2 / 3 | DATA0 N / P |
| 5 / 6 | DATA1 N / P |
| 8 / 9 | CLK N / P |
| 11 | power enable, pulled high via 10 k |
| 12 | IO1 (not connected to the P4) |
| 13 / 14 | SCL / SDA: the touch bus, GPIO8 / GPIO7, through 0 R |
| 15 | 3V3 |
| 1, 4, 7, 10, 16, 17 | GND |

### Software

- Sensor at SCCB address **0x36**. Probe it on the touch bus before starting video, so the firmware runs fine without a camera.
- **Components:** `espressif/esp_video` **1.1.0**, which pulls in `esp_ipa`, `esp_sccb_intf` and `esp_h264`. Remove its `examples/` folder if you vendor it on Windows. The registry's `esp_cam_sensor` (1.2.x up to 2.x) has **no OV02C10 driver**. Use the vendor's copy from the board's demo package (`1-Demo/Demo_IDF/components/espressif__esp_cam_sensor`): registry 1.2.1 plus `sensors/ov02c10`. Put it in `components/esp_cam_sensor` so it overrides the managed one.
- `esp_video_init` with a CSI config: `init_sccb = false`, `i2c_handle` = the touch bus, `freq` = 100 kHz, `reset_pin` = `pwdn_pin` = -1.
- `/dev/video0` default format: 1288x728 **RGB565** with the 1288x728 RAW10 sensor mode. Don't call `VIDIOC_S_FMT` with the format it already has: it returns EINVAL.
- Encode with the P4's hardware JPEG engine (`esp_driver_jpeg`). RGB565 in, YUV420, quality 75: about 30 KB per frame, encoding at 15 fps or more.
- **Stopping and restarting capture:** `VIDIOC_STREAMOFF` returns every buffer to you. `VIDIOC_QBUF` all of them again before `STREAMON`, or `DQBUF` blocks forever.
- The ISP pipeline controller (`CONFIG_ESP_VIDEO_ENABLE_ISP_PIPELINE_CONTROLLER`) runs auto-exposure and auto white balance from `ov02c10_default.json`. Autofocus (`CAMERA_OV02C10_ISP_AF_ENABLE`) is off here.
- Only capture while someone is watching. A continuous 1288x728 stream costs PSRAM bandwidth the display also needs.

An MJPEG stream (`multipart/x-mixed-replace`) on its own small HTTP server plus a `/snapshot.jpg` works directly with Home Assistant's **MJPEG IP Camera** integration.

---

## 10. Secrets template (`main/secrets.h`, never commit it)

```c
#pragma once
#define SECRET_WIFI_SSID      "your-ssid"
#define SECRET_WIFI_PASSWORD  "your-wifi-password"
#define SECRET_HA_HOST        "homeassistant.local"   // Home Assistant host
#define SECRET_HA_PORT        8123
#define SECRET_HA_TOKEN       "long-lived-access-token"
#define SECRET_OTA_PASSWORD   "choose-one"
#define SECRET_API_KEY        "base64-32-byte-key"    // ESPHome API Noise key, if you implement the API
```

`.gitignore`:

```
build/
managed_components/
sdkconfig
sdkconfig.old
dependencies.lock
main/secrets.h
```

---

## 11. Build and flash

```bash
idf.py set-target esp32p4
idf.py build
idf.py -p <PORT> flash monitor
```

With the partition table above and `CONFIG_BOOTLOADER_APP_ROLLBACK_ENABLE`:
- Call `esp_ota_mark_app_valid_cancel_rollback()` once the screen is up.
- A bad over-the-air image then rolls back by itself on the next reset.

Back up the factory firmware first:

```bash
esptool.py --chip esp32p4 -p <PORT> read_flash 0 0x1000000 backup.bin
```

---

## 12. Quick troubleshooting

| Symptom | Cause / fix |
|---|---|
| Flat lavender field, stripes | Original JD9165 table on a V2 panel: use the V2 table above |
| Washed-out vertical bands | Pixel clock not an exact divider of 240 MHz (for example 52): use 48 MHz |
| Every colour inverted, greys fine | Pixel clock 40 MHz: use 48 MHz |
| Picture upside down | `sw_rotate` + LVGL rotation 180 |
| Watchdog / hang during display init | IDF 5.5.1 command FIFO: use 5.5.5, start DPI before sending commands, skip the ID read |
| GT911 not found | Manual reset with a 100 ms wait, try 0x5D and 0x14 |
| Web servers accept then drop connections | `LWIP_MAX_SOCKETS` too low: 32 |
| Camera frames stop after an idle period | Re-queue all buffers after `STREAMOFF` |
| `esp_video` won't unpack on Windows | Short `IDF_COMPONENT_CACHE_PATH`, or vendor it without `examples/` |
| `esp_video_ipa_config.c` generation fails | IPA JSON path contains a space |
