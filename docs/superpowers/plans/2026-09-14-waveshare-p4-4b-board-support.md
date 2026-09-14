# Waveshare ESP32-P4-WIFI6-Touch-LCD-4B Board Support Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a new board target so WLED Touch Remote boots on the Waveshare ESP32-P4-WIFI6-Touch-LCD-4B (720x720 ST7703 MIPI-DSI panel, GT911 touch, ESP32-C6 Wi-Fi), reusing the existing UI as-is.

**Architecture:** Add `WLED_BOARD_WAVESHARE_P4B` alongside the existing `WLED_BOARD_JC4880P443` DSI board. Both share the `WLED_PANEL_DSI` capability flag and the same native `esp_lcd_mipi_dsi` bring-up code in `src/display.cpp`, branching only where the two panels genuinely differ (vendor init command table, timings, backlight wiring). New PlatformIO environments (`waveshare-p4-4b` for hardware, `macos-waveshare-p4-4b` for the SDL simulator) mirror the existing `jc4880p443` ones.

**Tech Stack:** PlatformIO, Arduino-ESP32 (pioarduino fork) for ESP32-P4, LVGL 8, native ESP-IDF `esp_lcd_mipi_dsi` API, ESP-Hosted Wi-Fi on ESP32-C6.

**Spec:** `docs/superpowers/specs/2026-09-14-waveshare-p4-4b-board-support-design.md`

---

## Verification approach

This is embedded firmware with no unit-test harness in this repo (confirmed: no `tests/` directory, no PlatformIO `test/` folder). The fastest automatable feedback loop is compilation:

- The **macOS SDL simulator** (`WLED_TOUCH_SIMULATOR=1`) compiles and runs the full UI at 720x720 without touching any ESP32-P4-specific code, so it's available as a real, running check after just the config/environment tasks (Task 1-3) — before any hardware-specific display code is written.
- The **real hardware environment** (`waveshare-p4-4b`) is checked by compiling it (`pio run -e waveshare-p4-4b`), which catches API misuse, typos, and missing symbols against the real ESP-IDF headers, but cannot verify the panel actually lights up correctly (timings, init table correctness) — that requires the physical board.
- Final physical verification (display image correct, touch registers, Wi-Fi connects, OTA works) is a manual checklist for you to run once you flash real hardware, listed at the end of this plan.

---

### Task 1: Add the new board id and its config macros

**Files:**
- Modify: `include/app_config.h`

- [x] **Step 1: Add the board id**

In `include/app_config.h`, change:

```c
#define WLED_BOARD_CYD 0
#define WLED_BOARD_JC4880P443 1
#define WLED_BOARD_JC8048W550C 2
```

to:

```c
#define WLED_BOARD_CYD 0
#define WLED_BOARD_JC4880P443 1
#define WLED_BOARD_JC8048W550C 2
#define WLED_BOARD_WAVESHARE_P4B 3
```

- [x] **Step 2: Fold the new board into the DSI capability flag**

Change:

```c
#define WLED_PANEL_DSI (WLED_BOARD == WLED_BOARD_JC4880P443)
```

to:

```c
#define WLED_PANEL_DSI (WLED_BOARD == WLED_BOARD_JC4880P443 || WLED_BOARD == WLED_BOARD_WAVESHARE_P4B)
```

(`WLED_TOUCH_GT911` and `WLED_BOARD_HAS_PSRAM` already derive from `WLED_PANEL_DSI`, so both automatically cover the new board — no changes needed to those two lines.)

- [x] **Step 3: Add the screen-size block**

After the existing `#if WLED_BOARD == WLED_BOARD_JC4880P443 ... #endif` block (the one setting `WLED_SCREEN_WIDTH 480` / `WLED_SCREEN_HEIGHT 800`), insert a new block for the square panel:

```c
#if WLED_BOARD == WLED_BOARD_WAVESHARE_P4B
// Waveshare ESP32-P4-WIFI6-Touch-LCD-4B: ESP32-P4 with a 720x720 ST7703
// MIPI-DSI panel and GT911 touch. Square panel, so there is no
// portrait/landscape distinction to make here.
#define WLED_SCREEN_WIDTH 720
#define WLED_SCREEN_HEIGHT 720
#define WLED_LVGL_BUFFER_LINES 40
#define WLED_DISPLAY_ROTATION 0
#define WLED_DISPLAY_ROTATION_FLIPPED 2
#ifndef WLED_CYD_ENABLE_BATTERY
#define WLED_CYD_ENABLE_BATTERY 0
#endif
#endif
```

- [x] **Step 4: Add it to `CYD_HARDWARE_PROFILE` selection**

Change:

```c
#ifndef CYD_HARDWARE_PROFILE
#if WLED_BOARD == WLED_BOARD_JC4880P443
#define CYD_HARDWARE_PROFILE CYD_PROFILE_ST7701_GT911
#elif WLED_BOARD == WLED_BOARD_JC8048W550C
#define CYD_HARDWARE_PROFILE CYD_PROFILE_ST7262_GT911
#else
#define CYD_HARDWARE_PROFILE CYD_PROFILE_AUTO
#endif
#endif
```

to:

```c
#ifndef CYD_HARDWARE_PROFILE
#if WLED_BOARD == WLED_BOARD_JC4880P443
#define CYD_HARDWARE_PROFILE CYD_PROFILE_ST7701_GT911
#elif WLED_BOARD == WLED_BOARD_JC8048W550C
#define CYD_HARDWARE_PROFILE CYD_PROFILE_ST7262_GT911
#elif WLED_BOARD == WLED_BOARD_WAVESHARE_P4B
#define CYD_HARDWARE_PROFILE CYD_PROFILE_ST7701_GT911
#else
#define CYD_HARDWARE_PROFILE CYD_PROFILE_AUTO
#endif
#endif
```

(Reusing `CYD_PROFILE_ST7701_GT911` is fine — this board never goes through CYD auto-detection, the profile constant is only read by code paths this board doesn't take.)

- [x] **Step 5: Add the pin/timing macro block**

Find the existing chain that ends with the JC4880P443 pin block and the CYD fallback:

```c
#define CYD_BOARD_CAPACITIVE 1

#else

#define CYD_TFT_SCLK 14
```

Insert a new `#elif` branch between them, so it reads:

```c
#define CYD_BOARD_CAPACITIVE 1

#elif WLED_BOARD == WLED_BOARD_WAVESHARE_P4B

// Waveshare ESP32-P4-WIFI6-Touch-LCD-4B: ESP32-P4 + ESP32-C6, 720x720 ST7703
// MIPI-DSI panel. Pin numbers and timings are from Waveshare's own
// examples/arduino/libraries/displays/displays_config.h (SCREEN_DEFAULT).
#define WSP4B_TFT_RST 27
#define WSP4B_TFT_BL 26            // active-low PWM
#define WSP4B_TFT_BL_ENABLE 33     // gate; assert only after PWM is attached
#define WSP4B_TFT_BL_FREQ 5000
#define WSP4B_TFT_BL_RES 10
#define WSP4B_PANEL_WIDTH 720
#define WSP4B_PANEL_HEIGHT 720
#define WSP4B_DSI_LANES 2
#define WSP4B_DSI_LANE_MBPS 480
#define WSP4B_DSI_LDO_CHANNEL 3
#define WSP4B_DSI_LDO_MV 2500
#define WSP4B_DPI_CLOCK_MHZ 38
#define WSP4B_HSYNC_BACK_PORCH 50
#define WSP4B_HSYNC_PULSE_WIDTH 20
#define WSP4B_HSYNC_FRONT_PORCH 50
#define WSP4B_VSYNC_BACK_PORCH 20
#define WSP4B_VSYNC_PULSE_WIDTH 4
#define WSP4B_VSYNC_FRONT_PORCH 20

#define WSP4B_TOUCH_SDA 7
#define WSP4B_TOUCH_SCL 8
#define WSP4B_TOUCH_RST -1
#define WSP4B_TOUCH_INT -1
#define WSP4B_TOUCH_ADDR 0x5D
#define WSP4B_TOUCH_I2C_PORT 0

// The GT911 driver is shared with the JC4880P443/JC8048W550C, which wire it
// elsewhere.
#define GT911_TOUCH_SDA WSP4B_TOUCH_SDA
#define GT911_TOUCH_SCL WSP4B_TOUCH_SCL
#define GT911_TOUCH_RST WSP4B_TOUCH_RST
#define GT911_TOUCH_INT WSP4B_TOUCH_INT
#define GT911_TOUCH_ADDR WSP4B_TOUCH_ADDR

#define CYD_BOARD_CAPACITIVE 1

#else

#define CYD_TFT_SCLK 14
```

- [x] **Step 6: Commit**

```bash
git add include/app_config.h
git commit -m "Add Waveshare ESP32-P4-WIFI6-Touch-LCD-4B board config"
```

---

### Task 2: Add the PlatformIO environments

**Files:**
- Modify: `platformio.ini`

- [x] **Step 1: Add the hardware environment**

After the `[env:jc4880p443]` block (ends just before `[env:jc8048w550c]`), insert:

```ini
[env:waveshare-p4-4b]
; Waveshare ESP32-P4-WIFI6-Touch-LCD-4B (ESP32-P4 + ESP32-C6, 720x720 ST7703
; MIPI-DSI panel, GT911 touch). Wi-Fi runs on the on-board C6 via ESP-Hosted,
; same as jc4880p443.
; This board defaults to rev3/post-v3 silicon (Waveshare's own Arduino IDE
; instructions specify "Chip Variant: v3.00 or newer" as the default for
; current-production boards). If the boot log's ROM banner reports
; "esp32p4-eco2", this is pre-rev3 silicon -- switch board to
; esp32-p4-evboard (the profile jc4880p443 uses) instead.
; Serial runs over this board's CH343P USB-UART bridge on UART0, not a
; native USB-C CDC port, so this omits the ARDUINO_USB_* flags jc4880p443 needs.
platform = https://github.com/pioarduino/platform-espressif32/releases/download/55.03.311/platform-espressif32.zip
board = esp32-p4_r3-evboard
framework = arduino
monitor_speed = 115200
upload_speed = 1500000
monitor_filters = esp32_exception_decoder

lib_deps =
    lvgl/lvgl@^8.3.11
    bblanchon/ArduinoJson@^7.0.4

build_flags =
    -D LV_CONF_INCLUDE_SIMPLE
    -I include
    -D CORE_DEBUG_LEVEL=1
    -D WLED_BOARD=WLED_BOARD_WAVESHARE_P4B

board_build.partitions = default_16MB.csv
extra_scripts =
    pre:scripts/embed_logo.py
    pre:scripts/git_version.py
    pre:scripts/embed_hosted_firmware.py
```

- [x] **Step 2: Add the macOS simulator environment**

After the `[env:macos-jc4880p443]` block, insert:

```ini
[env:macos-waveshare-p4-4b]
; Native SDL simulator using the Waveshare board's 720x720 square layout.
extends = env:macos
build_unflags =
    -D WLED_CYD_ENABLE_BATTERY=1
build_flags =
    ${env:macos.build_flags}
    -D WLED_BOARD=WLED_BOARD_WAVESHARE_P4B
    -D WLED_CYD_ENABLE_BATTERY=0
```

- [x] **Step 3: Commit**

```bash
git add platformio.ini
git commit -m "Add waveshare-p4-4b and macos-waveshare-p4-4b PlatformIO environments"
```

---

### Task 3: Verify the simulator builds and runs at 720x720

This is the first real, running checkpoint: the simulator doesn't touch any of the ESP32-P4-specific display code (it's all guarded by `!WLED_TOUCH_SIMULATOR`), so it should build cleanly right after Tasks 1-2 and show the existing UI reflowed into a 720x720 square window.

- [x] **Step 1: Build the simulator**

Run: `pio run -e macos-waveshare-p4-4b`

Expected: `SUCCESS` (requires `sdl2` installed — `brew install sdl2` per the README if not already present).

- [x] **Step 2: Run it and look at the UI**

Run: `.pio/build/macos-waveshare-p4-4b/program`

Expected: a 720x720 window opens showing the splash screen and then the main tab UI. Click/drag to exercise a couple of screens (main tab, presets, settings) to confirm nothing crashes at this aspect ratio. Layout will very likely look imperfect (this board's UI polish is explicitly out of scope for this pass per the spec) — the goal here is just "renders and doesn't crash," not "looks good."

No commit for this task — it's a verification checkpoint. If it crashes or fails to build, stop and fix before continuing (that would indicate a mistake in Task 1/2, not something to defer).

---

### Task 4: Generalize the DSI-board branches in `display.cpp` from JC4880P443 to WLED_PANEL_DSI

Six spots in `src/display.cpp` currently hardcode `WLED_BOARD == WLED_BOARD_JC4880P443` where the logic is identical for any DSI board (both are full-frame `dsi_framebuffer`-backed panels with no special mirroring beyond the existing 180-degree flip support). All six are unreachable in simulator builds (each sits behind a `#if WLED_TOUCH_SIMULATOR` branch that's checked first), so Task 3's simulator check does not exercise this code — this task is required for the **real hardware** environment to even compile, since `initDisplay()` (already generic on `WLED_PANEL_DSI`) references buffer variables that are currently only declared under the JC4880P443-specific condition.

**Files:**
- Modify: `src/display.cpp`

- [x] **Step 1: Buffer variable declarations (~line 52)**

Change:

```cpp
#elif WLED_BOARD == WLED_BOARD_JC4880P443
// A full-frame LVGL buffer lets the renderer work without 40-line tiles.  Keep
// the small pair only as a safe fallback if external RAM is unavailable.
lv_color_t* p4_full_draw_buf = nullptr;
```

to:

```cpp
#elif WLED_PANEL_DSI
// A full-frame LVGL buffer lets the renderer work without 40-line tiles.  Keep
// the small pair only as a safe fallback if external RAM is unavailable.
lv_color_t* p4_full_draw_buf = nullptr;
```

- [x] **Step 2: DSI handle globals (~line 101)**

Change:

```cpp
#elif WLED_BOARD == WLED_BOARD_JC4880P443
esp_lcd_dsi_bus_handle_t dsi_bus = nullptr;
```

to:

```cpp
#elif WLED_PANEL_DSI
esp_lcd_dsi_bus_handle_t dsi_bus = nullptr;
```

- [x] **Step 3: `drawSplashTextRow` (~line 685)**

Change:

```cpp
#elif WLED_BOARD == WLED_BOARD_JC4880P443
  if (!display_flipped) {
    memcpy(dsi_framebuffer + y * kScreenWidth + x, splash_text_row, width * sizeof(uint16_t));
```

to:

```cpp
#elif WLED_PANEL_DSI
  if (!display_flipped) {
    memcpy(dsi_framebuffer + y * kScreenWidth + x, splash_text_row, width * sizeof(uint16_t));
```

- [x] **Step 4: `drawDisplaySplash` (~line 747)**

Change:

```cpp
#elif WLED_BOARD == WLED_BOARD_JC4880P443
  if (!display_flipped) {
    for (uint16_t y = 0; y < kWledLogoHeight; ++y) {
      memcpy(dsi_framebuffer + (y0 + y) * kScreenWidth + x0,
```

to:

```cpp
#elif WLED_PANEL_DSI
  if (!display_flipped) {
    for (uint16_t y = 0; y < kWledLogoHeight; ++y) {
      memcpy(dsi_framebuffer + (y0 + y) * kScreenWidth + x0,
```

- [x] **Step 5: `flushDisplay` (~line 782)**

Change:

```cpp
#elif WLED_BOARD == WLED_BOARD_JC4880P443
  const bool flip = display_flipped;
```

to:

```cpp
#elif WLED_PANEL_DSI
  const bool flip = display_flipped;
```

- [x] **Step 6: `displayClear` (~line 980)**

Change:

```cpp
#elif WLED_BOARD == WLED_BOARD_JC4880P443
  for (uint32_t i = 0; i < uint32_t(kScreenWidth) * kScreenHeight; ++i) dsi_framebuffer[i] = rgb565;
```

to:

```cpp
#elif WLED_PANEL_DSI
  for (uint32_t i = 0; i < uint32_t(kScreenWidth) * kScreenHeight; ++i) dsi_framebuffer[i] = rgb565;
```

- [x] **Step 7: Verify JC4880P443 still compiles unchanged**

Run: `pio run -e jc4880p443`
Expected: `SUCCESS` — this is a pure rename of the condition (JC4880P443 was the only board satisfying `WLED_PANEL_DSI` before this change added a second one), so JC4880P443 must build identically. This is the regression check for this task.

- [x] **Step 8: Verify the simulator still builds**

Run: `pio run -e macos-waveshare-p4-4b`
Expected: `SUCCESS` (these branches are unreachable in sim builds, so this should be unaffected — confirms nothing else broke).

- [x] **Step 9: Commit**

```bash
git add src/display.cpp
git commit -m "Generalize DSI framebuffer code paths to any WLED_PANEL_DSI board"
```

---

### Task 5: Add ST7703 panel bring-up to `initP4Panel()`

**Files:**
- Modify: `src/display.cpp`

- [x] **Step 1: Replace the hardcoded JC4880-only bring-up with a board-selected version**

Find `initP4Panel()` (currently ~line 510-542):

```cpp
bool initP4Panel() {
  esp_ldo_channel_config_t ldo = {};
  ldo.chan_id = JC4880_DSI_LDO_CHANNEL; ldo.voltage_mv = JC4880_DSI_LDO_MV;
  esp_lcd_dsi_bus_config_t bus = {};
  bus.bus_id = 0; bus.num_data_lanes = JC4880_DSI_LANES; bus.phy_clk_src = MIPI_DSI_PHY_CLK_SRC_DEFAULT; bus.lane_bit_rate_mbps = JC4880_DSI_LANE_MBPS;
  esp_lcd_dbi_io_config_t dbi = {};
  dbi.virtual_channel = 0; dbi.lcd_cmd_bits = 8; dbi.lcd_param_bits = 8;
  if (esp_ldo_acquire_channel(&ldo, &dsi_ldo) != ESP_OK || esp_lcd_new_dsi_bus(&bus, &dsi_bus) != ESP_OK ||
      esp_lcd_new_panel_io_dbi(dsi_bus, &dbi, &dsi_io) != ESP_OK) return false;
  esp_lcd_dpi_panel_config_t dpi = {};
  dpi.virtual_channel = 0; dpi.dpi_clk_src = MIPI_DSI_DPI_CLK_SRC_DEFAULT; dpi.dpi_clock_freq_mhz = JC4880_DPI_CLOCK_MHZ;
  dpi.pixel_format = LCD_COLOR_PIXEL_FORMAT_RGB565; dpi.num_fbs = 1;
  dpi.video_timing.h_size = JC4880_PANEL_WIDTH; dpi.video_timing.v_size = JC4880_PANEL_HEIGHT;
  dpi.video_timing.hsync_back_porch = JC4880_HSYNC_BACK_PORCH; dpi.video_timing.hsync_pulse_width = JC4880_HSYNC_PULSE_WIDTH; dpi.video_timing.hsync_front_porch = JC4880_HSYNC_FRONT_PORCH;
  dpi.video_timing.vsync_back_porch = JC4880_VSYNC_BACK_PORCH; dpi.video_timing.vsync_pulse_width = JC4880_VSYNC_PULSE_WIDTH; dpi.video_timing.vsync_front_porch = JC4880_VSYNC_FRONT_PORCH;
  if (esp_lcd_new_panel_dpi(dsi_bus, &dpi, &dpi_panel) != ESP_OK) return false;
  pinMode(JC4880_TFT_RST, OUTPUT); digitalWrite(JC4880_TFT_RST, LOW); delay(20); digitalWrite(JC4880_TFT_RST, HIGH); delay(120);
  // JC4880P443's ST7701S vendor sequence. This is intentionally kept here
  // beside the native DSI setup instead of hidden in a graphics library.
  auto cmd = [](uint8_t c, std::initializer_list<uint8_t> p) { return p4Write(c, p.begin(), p.size()); };
  cmd(0xFF,{0x77,0x01,0x00,0x00,0x13}); cmd(0xEF,{0x08});
  cmd(0xFF,{0x77,0x01,0x00,0x00,0x10}); cmd(0xC0,{0x63,0x00}); cmd(0xC1,{0x0D,0x02}); cmd(0xC2,{0x10,0x08}); cmd(0xCC,{0x10});
  cmd(0xB0,{0x80,0x09,0x53,0x0C,0xD0,0x07,0x0C,0x09,0x09,0x28,0x06,0xD4,0x13,0x69,0x2B,0x71});
  cmd(0xB1,{0x80,0x94,0x5A,0x10,0xD3,0x06,0x0A,0x08,0x08,0x25,0x03,0xD3,0x12,0x66,0x6A,0x0D});
  cmd(0xFF,{0x77,0x01,0x00,0x00,0x11}); cmd(0xB0,{0x5D}); cmd(0xB1,{0x58}); cmd(0xB2,{0x87}); cmd(0xB3,{0x80}); cmd(0xB5,{0x4E}); cmd(0xB7,{0x85}); cmd(0xB8,{0x21}); cmd(0xB9,{0x10,0x1F}); cmd(0xBB,{0x03}); cmd(0xBC,{0x00}); cmd(0xC1,{0x78}); cmd(0xC2,{0x78}); cmd(0xD0,{0x88});
  cmd(0xE0,{0x00,0x3A,0x02}); cmd(0xE1,{0x04,0xA0,0x00,0xA0,0x05,0xA0,0x00,0xA0,0x00,0x40,0x40}); cmd(0xE2,{0x30,0x00,0x40,0x40,0x32,0xA0,0x00,0xA0,0x00,0xA0,0x00,0xA0,0x00}); cmd(0xE3,{0x00,0x00,0x33,0x33}); cmd(0xE4,{0x44,0x44});
  cmd(0xE5,{0x09,0x2E,0xA0,0xA0,0x0B,0x30,0xA0,0xA0,0x05,0x2A,0xA0,0xA0,0x07,0x2C,0xA0,0xA0}); cmd(0xE6,{0x00,0x00,0x33,0x33}); cmd(0xE7,{0x44,0x44}); cmd(0xE8,{0x08,0x2D,0xA0,0xA0,0x0A,0x2F,0xA0,0xA0,0x04,0x29,0xA0,0xA0,0x06,0x2B,0xA0,0xA0}); cmd(0xEB,{0x00,0x00,0x4E,0x4E,0x00,0x00,0x00}); cmd(0xEC,{0x08,0x01});
  cmd(0xED,{0xB0,0x2B,0x98,0xA4,0x56,0x7F,0xFF,0xFF,0xFF,0xFF,0xF7,0x65,0x4A,0x89,0xB2,0x0B}); cmd(0xEF,{0x08,0x08,0x08,0x45,0x3F,0x54}); cmd(0xFF,{0x77,0x01,0x00,0x00,0x00});
  const uint8_t colmod[] = {0x55}; p4Write(0x3A,colmod,1); p4Write(0x11,nullptr,0); delay(120); p4Write(0x29,nullptr,0);
  if (esp_lcd_panel_init(dpi_panel) != ESP_OK || esp_lcd_dpi_panel_get_frame_buffer(dpi_panel, 1, reinterpret_cast<void**>(&dsi_framebuffer)) != ESP_OK) return false;
  initGt911Touch();
  return dsi_framebuffer != nullptr;
}
```

Replace it with:

```cpp
#if WLED_BOARD == WLED_BOARD_WAVESHARE_P4B
constexpr int8_t kDsiTftRst = WSP4B_TFT_RST;
constexpr uint8_t kDsiLdoChannel = WSP4B_DSI_LDO_CHANNEL;
constexpr int kDsiLdoMv = WSP4B_DSI_LDO_MV;
constexpr int kDsiLanes = WSP4B_DSI_LANES;
constexpr int kDsiLaneMbps = WSP4B_DSI_LANE_MBPS;
constexpr int kDsiDpiClockMhz = WSP4B_DPI_CLOCK_MHZ;
constexpr int kDsiPanelWidth = WSP4B_PANEL_WIDTH;
constexpr int kDsiPanelHeight = WSP4B_PANEL_HEIGHT;
constexpr int kDsiHsyncBackPorch = WSP4B_HSYNC_BACK_PORCH;
constexpr int kDsiHsyncPulseWidth = WSP4B_HSYNC_PULSE_WIDTH;
constexpr int kDsiHsyncFrontPorch = WSP4B_HSYNC_FRONT_PORCH;
constexpr int kDsiVsyncBackPorch = WSP4B_VSYNC_BACK_PORCH;
constexpr int kDsiVsyncPulseWidth = WSP4B_VSYNC_PULSE_WIDTH;
constexpr int kDsiVsyncFrontPorch = WSP4B_VSYNC_FRONT_PORCH;
#else
constexpr int8_t kDsiTftRst = JC4880_TFT_RST;
constexpr uint8_t kDsiLdoChannel = JC4880_DSI_LDO_CHANNEL;
constexpr int kDsiLdoMv = JC4880_DSI_LDO_MV;
constexpr int kDsiLanes = JC4880_DSI_LANES;
constexpr int kDsiLaneMbps = JC4880_DSI_LANE_MBPS;
constexpr int kDsiDpiClockMhz = JC4880_DPI_CLOCK_MHZ;
constexpr int kDsiPanelWidth = JC4880_PANEL_WIDTH;
constexpr int kDsiPanelHeight = JC4880_PANEL_HEIGHT;
constexpr int kDsiHsyncBackPorch = JC4880_HSYNC_BACK_PORCH;
constexpr int kDsiHsyncPulseWidth = JC4880_HSYNC_PULSE_WIDTH;
constexpr int kDsiHsyncFrontPorch = JC4880_HSYNC_FRONT_PORCH;
constexpr int kDsiVsyncBackPorch = JC4880_VSYNC_BACK_PORCH;
constexpr int kDsiVsyncPulseWidth = JC4880_VSYNC_PULSE_WIDTH;
constexpr int kDsiVsyncFrontPorch = JC4880_VSYNC_FRONT_PORCH;
#endif

bool initP4Panel() {
  esp_ldo_channel_config_t ldo = {};
  ldo.chan_id = kDsiLdoChannel; ldo.voltage_mv = kDsiLdoMv;
  esp_lcd_dsi_bus_config_t bus = {};
  bus.bus_id = 0; bus.num_data_lanes = kDsiLanes; bus.phy_clk_src = MIPI_DSI_PHY_CLK_SRC_DEFAULT; bus.lane_bit_rate_mbps = kDsiLaneMbps;
  esp_lcd_dbi_io_config_t dbi = {};
  dbi.virtual_channel = 0; dbi.lcd_cmd_bits = 8; dbi.lcd_param_bits = 8;
  if (esp_ldo_acquire_channel(&ldo, &dsi_ldo) != ESP_OK || esp_lcd_new_dsi_bus(&bus, &dsi_bus) != ESP_OK ||
      esp_lcd_new_panel_io_dbi(dsi_bus, &dbi, &dsi_io) != ESP_OK) return false;
  esp_lcd_dpi_panel_config_t dpi = {};
  dpi.virtual_channel = 0; dpi.dpi_clk_src = MIPI_DSI_DPI_CLK_SRC_DEFAULT; dpi.dpi_clock_freq_mhz = kDsiDpiClockMhz;
  dpi.pixel_format = LCD_COLOR_PIXEL_FORMAT_RGB565; dpi.num_fbs = 1;
  dpi.video_timing.h_size = kDsiPanelWidth; dpi.video_timing.v_size = kDsiPanelHeight;
  dpi.video_timing.hsync_back_porch = kDsiHsyncBackPorch; dpi.video_timing.hsync_pulse_width = kDsiHsyncPulseWidth; dpi.video_timing.hsync_front_porch = kDsiHsyncFrontPorch;
  dpi.video_timing.vsync_back_porch = kDsiVsyncBackPorch; dpi.video_timing.vsync_pulse_width = kDsiVsyncPulseWidth; dpi.video_timing.vsync_front_porch = kDsiVsyncFrontPorch;
  if (esp_lcd_new_panel_dpi(dsi_bus, &dpi, &dpi_panel) != ESP_OK) return false;
  pinMode(kDsiTftRst, OUTPUT); digitalWrite(kDsiTftRst, LOW); delay(20); digitalWrite(kDsiTftRst, HIGH); delay(120);
  auto cmd = [](uint8_t c, std::initializer_list<uint8_t> p) { return p4Write(c, p.begin(), p.size()); };
#if WLED_BOARD == WLED_BOARD_WAVESHARE_P4B
  // Waveshare ESP32-P4-WIFI6-Touch-LCD-4B's ST7703 vendor sequence, from
  // examples/arduino/libraries/displays/displays_config.h in Waveshare's repo.
  cmd(0xB9, {0xF1, 0x12, 0x83});
  cmd(0xB1, {0x00, 0x00, 0x00, 0xDA, 0x80});
  cmd(0xB2, {0x3C, 0x12, 0x30});
  cmd(0xB3, {0x10, 0x10, 0x28, 0x28, 0x03, 0xFF, 0x00, 0x00, 0x00, 0x00});
  cmd(0xB4, {0x80});
  cmd(0xB5, {0x0A, 0x0A});
  cmd(0xB6, {0x97, 0x97});
  cmd(0xB8, {0x26, 0x22, 0xF0, 0x13});
  cmd(0xBA, {0x31, 0x81, 0x0F, 0xF9, 0x0E, 0x06, 0x20, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x44, 0x25, 0x00, 0x90, 0x0A, 0x00, 0x00, 0x01, 0x4F, 0x01, 0x00, 0x00, 0x37});
  cmd(0xBC, {0x47});
  cmd(0xBF, {0x02, 0x11, 0x00});
  cmd(0xC0, {0x73, 0x73, 0x50, 0x50, 0x00, 0x00, 0x12, 0x70, 0x00});
  cmd(0xC1, {0x25, 0x00, 0x32, 0x32, 0x77, 0xE4, 0xFF, 0xFF, 0xCC, 0xCC, 0x77, 0x77});
  cmd(0xC6, {0x82, 0x00, 0xBF, 0xFF, 0x00, 0xFF});
  cmd(0xC7, {0xB8, 0x00, 0x0A, 0x10, 0x01, 0x09});
  cmd(0xC8, {0x10, 0x40, 0x1E, 0x02});
  cmd(0xCC, {0x0B});
  cmd(0xE0, {0x00, 0x0B, 0x10, 0x2C, 0x3D, 0x3F, 0x42, 0x3A, 0x07, 0x0D, 0x0F, 0x13, 0x15, 0x13, 0x14, 0x0F, 0x16,
             0x00, 0x0B, 0x10, 0x2C, 0x3D, 0x3F, 0x42, 0x3A, 0x07, 0x0D, 0x0F, 0x13, 0x15, 0x13, 0x14, 0x0F, 0x16});
  cmd(0xE3, {0x07, 0x07, 0x0B, 0x0B, 0x0B, 0x0B, 0x00, 0x00, 0x00, 0x00, 0xFF, 0x00, 0xC0, 0x10});
  cmd(0xE9, {0xC8, 0x10, 0x0A, 0x00, 0x00, 0x80, 0x81, 0x12, 0x31, 0x23, 0x4F, 0x86, 0xA0, 0x00, 0x47, 0x08, 0x00,
             0x00, 0x0C, 0x00, 0x00, 0x00, 0x00, 0x00, 0x0C, 0x00, 0x00, 0x00, 0x98, 0x02, 0x8B, 0xAF, 0x46, 0x02,
             0x88, 0x88, 0x88, 0x88, 0x88, 0x98, 0x13, 0x8B, 0xAF, 0x57, 0x13, 0x88, 0x88, 0x88, 0x88, 0x88, 0x00,
             0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00});
  cmd(0xEA, {0x97, 0x0C, 0x09, 0x09, 0x09, 0x78, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x9F, 0x31, 0x8B, 0xA8, 0x31,
             0x75, 0x88, 0x88, 0x88, 0x88, 0x88, 0x9F, 0x20, 0x8B, 0xA8, 0x20, 0x64, 0x88, 0x88, 0x88, 0x88, 0x88,
             0x23, 0x00, 0x00, 0x02, 0x71, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00,
             0x00, 0x00, 0x00, 0x40, 0x80, 0x81, 0x00, 0x00, 0x00, 0x00});
  cmd(0xEF, {0xFF, 0xFF, 0x01});
  const uint8_t colmod[] = {0x55}; p4Write(0x3A, colmod, 1);
  p4Write(0x11, nullptr, 0); delay(250);
  p4Write(0x29, nullptr, 0); delay(50);
#else
  // JC4880P443's ST7701S vendor sequence. This is intentionally kept here
  // beside the native DSI setup instead of hidden in a graphics library.
  cmd(0xFF,{0x77,0x01,0x00,0x00,0x13}); cmd(0xEF,{0x08});
  cmd(0xFF,{0x77,0x01,0x00,0x00,0x10}); cmd(0xC0,{0x63,0x00}); cmd(0xC1,{0x0D,0x02}); cmd(0xC2,{0x10,0x08}); cmd(0xCC,{0x10});
  cmd(0xB0,{0x80,0x09,0x53,0x0C,0xD0,0x07,0x0C,0x09,0x09,0x28,0x06,0xD4,0x13,0x69,0x2B,0x71});
  cmd(0xB1,{0x80,0x94,0x5A,0x10,0xD3,0x06,0x0A,0x08,0x08,0x25,0x03,0xD3,0x12,0x66,0x6A,0x0D});
  cmd(0xFF,{0x77,0x01,0x00,0x00,0x11}); cmd(0xB0,{0x5D}); cmd(0xB1,{0x58}); cmd(0xB2,{0x87}); cmd(0xB3,{0x80}); cmd(0xB5,{0x4E}); cmd(0xB7,{0x85}); cmd(0xB8,{0x21}); cmd(0xB9,{0x10,0x1F}); cmd(0xBB,{0x03}); cmd(0xBC,{0x00}); cmd(0xC1,{0x78}); cmd(0xC2,{0x78}); cmd(0xD0,{0x88});
  cmd(0xE0,{0x00,0x3A,0x02}); cmd(0xE1,{0x04,0xA0,0x00,0xA0,0x05,0xA0,0x00,0xA0,0x00,0x40,0x40}); cmd(0xE2,{0x30,0x00,0x40,0x40,0x32,0xA0,0x00,0xA0,0x00,0xA0,0x00,0xA0,0x00}); cmd(0xE3,{0x00,0x00,0x33,0x33}); cmd(0xE4,{0x44,0x44});
  cmd(0xE5,{0x09,0x2E,0xA0,0xA0,0x0B,0x30,0xA0,0xA0,0x05,0x2A,0xA0,0xA0,0x07,0x2C,0xA0,0xA0}); cmd(0xE6,{0x00,0x00,0x33,0x33}); cmd(0xE7,{0x44,0x44}); cmd(0xE8,{0x08,0x2D,0xA0,0xA0,0x0A,0x2F,0xA0,0xA0,0x04,0x29,0xA0,0xA0,0x06,0x2B,0xA0,0xA0}); cmd(0xEB,{0x00,0x00,0x4E,0x4E,0x00,0x00,0x00}); cmd(0xEC,{0x08,0x01});
  cmd(0xED,{0xB0,0x2B,0x98,0xA4,0x56,0x7F,0xFF,0xFF,0xFF,0xFF,0xF7,0x65,0x4A,0x89,0xB2,0x0B}); cmd(0xEF,{0x08,0x08,0x08,0x45,0x3F,0x54}); cmd(0xFF,{0x77,0x01,0x00,0x00,0x00});
  const uint8_t colmod[] = {0x55}; p4Write(0x3A,colmod,1); p4Write(0x11,nullptr,0); delay(120); p4Write(0x29,nullptr,0);
#endif
  if (esp_lcd_panel_init(dpi_panel) != ESP_OK || esp_lcd_dpi_panel_get_frame_buffer(dpi_panel, 1, reinterpret_cast<void**>(&dsi_framebuffer)) != ESP_OK) return false;
  initGt911Touch();
  return dsi_framebuffer != nullptr;
}
```

- [x] **Step 2: Verify both hardware environments compile**

Run: `pio run -e jc4880p443`
Expected: `SUCCESS`

Run: `pio run -e waveshare-p4-4b`
Expected: `SUCCESS` on first attempt, or a first-run toolchain/package download followed by `SUCCESS`. If it fails with a compiler error, re-check the constants against this task's code before moving on — this is the step most likely to surface a typo given the size of the init table.

- [x] **Step 3: Commit**

```bash
git add src/display.cpp
git commit -m "Add ST7703 panel bring-up for Waveshare ESP32-P4-WIFI6-Touch-LCD-4B"
```

---

### Task 6: Add PWM backlight handling for the Waveshare board

Unlike JC4880P443's plain GPIO backlight, this board drives its backlight through active-low PWM gated by a separate enable pin (per Waveshare's `display_cfg_backlight()`).

**Files:**
- Modify: `src/display.cpp`

- [x] **Step 1: Add the backlight helper**

Directly above `void displaySetBrightness(uint8_t brightness) {` (currently ~line 938), add:

```cpp
#if !WLED_TOUCH_SIMULATOR && WLED_BOARD == WLED_BOARD_WAVESHARE_P4B
void setBacklightWsp4b(uint8_t brightness) {
  static bool attached = false;
  if (!attached) {
    attached = ledcAttach(WSP4B_TFT_BL, WSP4B_TFT_BL_FREQ, WSP4B_TFT_BL_RES) &&
               ledcOutputInvert(WSP4B_TFT_BL, true);
    if (attached) {
      pinMode(WSP4B_TFT_BL_ENABLE, OUTPUT);
      digitalWrite(WSP4B_TFT_BL_ENABLE, HIGH);
    }
  }
  if (!attached) return;
  const uint32_t duty = uint32_t(brightness) * ((1u << WSP4B_TFT_BL_RES) - 1) / 255;
  ledcWrite(WSP4B_TFT_BL, duty);
}
#endif

```

- [x] **Step 2: Branch `displaySetBrightness` on the new board**

Change:

```cpp
void displaySetBrightness(uint8_t brightness) {
#if !WLED_TOUCH_SIMULATOR && WLED_PANEL_DSI
  setBacklight(JC4880_TFT_BL, brightness);
#elif !WLED_TOUCH_SIMULATOR && WLED_PANEL_RGB
  setBacklight(JC8048_TFT_BL, brightness);
#elif !WLED_TOUCH_SIMULATOR
  setBacklight(cyd_backlight_pin, brightness);
#else
  (void)brightness;
#endif
}
```

to:

```cpp
void displaySetBrightness(uint8_t brightness) {
#if !WLED_TOUCH_SIMULATOR && WLED_BOARD == WLED_BOARD_WAVESHARE_P4B
  setBacklightWsp4b(brightness);
#elif !WLED_TOUCH_SIMULATOR && WLED_PANEL_DSI
  setBacklight(JC4880_TFT_BL, brightness);
#elif !WLED_TOUCH_SIMULATOR && WLED_PANEL_RGB
  setBacklight(JC8048_TFT_BL, brightness);
#elif !WLED_TOUCH_SIMULATOR
  setBacklight(cyd_backlight_pin, brightness);
#else
  (void)brightness;
#endif
}
```

- [x] **Step 3: Branch `displayPrepareForBoot` on the new board**

Change:

```cpp
void displayPrepareForBoot() {
#if !WLED_TOUCH_SIMULATOR && WLED_PANEL_DSI
  pinMode(JC4880_TFT_BL, OUTPUT);
  digitalWrite(JC4880_TFT_BL, LOW);
#elif !WLED_TOUCH_SIMULATOR && WLED_PANEL_RGB
```

to:

```cpp
void displayPrepareForBoot() {
#if !WLED_TOUCH_SIMULATOR && WLED_BOARD == WLED_BOARD_WAVESHARE_P4B
  // Hold the backlight gate low until setBacklightWsp4b() has PWM running,
  // so the panel never flashes full brightness with undefined contents.
  pinMode(WSP4B_TFT_BL_ENABLE, OUTPUT);
  digitalWrite(WSP4B_TFT_BL_ENABLE, LOW);
#elif !WLED_TOUCH_SIMULATOR && WLED_PANEL_DSI
  pinMode(JC4880_TFT_BL, OUTPUT);
  digitalWrite(JC4880_TFT_BL, LOW);
#elif !WLED_TOUCH_SIMULATOR && WLED_PANEL_RGB
```

- [x] **Step 4: Verify all three hardware environments compile**

Run: `pio run -e esp32-cyd`
Expected: `SUCCESS`

Run: `pio run -e jc4880p443`
Expected: `SUCCESS`

Run: `pio run -e waveshare-p4-4b`
Expected: `SUCCESS`

- [x] **Step 5: Commit**

```bash
git add src/display.cpp
git commit -m "Add active-low PWM backlight handling for Waveshare P4-4B"
```

---

### Task 7: Embed the C6 Wi-Fi firmware for the new environment

`scripts/embed_hosted_firmware.py` only embeds the ESP-Hosted C6 firmware blob when `$PIOENV == "jc4880p443"`; without extending this, the new environment builds "successfully" but Wi-Fi silently never comes up.

**Files:**
- Modify: `scripts/embed_hosted_firmware.py`

- [x] **Step 1: Extend the environment check**

Change:

```python
if env.subst("$PIOENV") == "jc4880p443":
```

to:

```python
if env.subst("$PIOENV") in ("jc4880p443", "waveshare-p4-4b"):
```

- [x] **Step 2: Verify the hardware environment still compiles**

Run: `pio run -e waveshare-p4-4b`
Expected: `SUCCESS`, and the build log should show the generated
`include/generated/hosted_c6_firmware_asm.h` being (re)written (same as it
already does for `jc4880p443`).

- [x] **Step 3: Commit**

```bash
git add scripts/embed_hosted_firmware.py
git commit -m "Embed ESP-Hosted C6 firmware for waveshare-p4-4b builds"
```

---

### Task 8: Add the OTA build-target string

**Files:**
- Modify: `src/update_manager.cpp`

- [x] **Step 1: Add the board branch**

Change:

```cpp
#if WLED_BOARD == WLED_BOARD_JC4880P443
constexpr const char* kBuildTarget = "jc4880p443";
#elif WLED_BOARD == WLED_BOARD_JC8048W550C
constexpr const char* kBuildTarget = "jc8048w550c";
#else
constexpr const char* kBuildTarget = "esp32-cyd";
#endif
```

to:

```cpp
#if WLED_BOARD == WLED_BOARD_JC4880P443
constexpr const char* kBuildTarget = "jc4880p443";
#elif WLED_BOARD == WLED_BOARD_JC8048W550C
constexpr const char* kBuildTarget = "jc8048w550c";
#elif WLED_BOARD == WLED_BOARD_WAVESHARE_P4B
constexpr const char* kBuildTarget = "waveshare-p4-4b";
#else
constexpr const char* kBuildTarget = "esp32-cyd";
#endif
```

- [x] **Step 2: Verify it compiles**

Run: `pio run -e waveshare-p4-4b`
Expected: `SUCCESS`

- [x] **Step 3: Commit**

```bash
git add src/update_manager.cpp
git commit -m "Add waveshare-p4-4b OTA build-target identifier"
```

---

### Task 9: Wire the new board into CI release and web-installer publishing

**Files:**
- Modify: `.github/workflows/build-release.yml`
- Modify: `.github/workflows/publish-web-installer.yml`

- [x] **Step 1: Add it to the release build step**

In `.github/workflows/build-release.yml`, change:

```yaml
          pio run -e jc4880p443
          cp -a .pio/build/jc4880p443 release-build/jc4880p443
          pio run -e jc8048w550c
          cp -a .pio/build/jc8048w550c release-build/jc8048w550c
```

to:

```yaml
          pio run -e jc4880p443
          cp -a .pio/build/jc4880p443 release-build/jc4880p443
          pio run -e jc8048w550c
          cp -a .pio/build/jc8048w550c release-build/jc8048w550c
          pio run -e waveshare-p4-4b
          cp -a .pio/build/waveshare-p4-4b release-build/waveshare-p4-4b
```

- [x] **Step 2: Add it to the packaging step**

In the same file, change:

```bash
          package_env esp32-cyd esp32 4MB 0x1000
          package_env jc4880p443 esp32p4 16MB 0x2000
          package_env jc8048w550c esp32s3 16MB 0x0
```

to:

```bash
          package_env esp32-cyd esp32 4MB 0x1000
          package_env jc4880p443 esp32p4 16MB 0x2000
          package_env jc8048w550c esp32s3 16MB 0x0
          package_env waveshare-p4-4b esp32p4 16MB 0x2000
```

(Same chip family and bootloader offset as `jc4880p443` — both are ESP32-P4.)

- [x] **Step 3: Add it to the web-installer download step**

In `.github/workflows/publish-web-installer.yml`, change:

```bash
          gh release download "$tag" --repo "$GITHUB_REPOSITORY" --dir web-installer/firmware \
            --pattern "wled-touch-remote-${tag}-esp32-cyd-merged.bin" \
            --pattern "wled-touch-remote-${tag}-jc4880p443-merged.bin" \
            --pattern "wled-touch-remote-${tag}-jc8048w550c-merged.bin"
          mv "web-installer/firmware/wled-touch-remote-${tag}-esp32-cyd-merged.bin" \
            web-installer/firmware/wled-touch-remote-wifi.bin
          mv "web-installer/firmware/wled-touch-remote-${tag}-jc4880p443-merged.bin" \
            web-installer/firmware/wled-touch-remote-jc4880p443.bin
          mv "web-installer/firmware/wled-touch-remote-${tag}-jc8048w550c-merged.bin" \
            web-installer/firmware/wled-touch-remote-jc8048w550c.bin
```

to:

```bash
          gh release download "$tag" --repo "$GITHUB_REPOSITORY" --dir web-installer/firmware \
            --pattern "wled-touch-remote-${tag}-esp32-cyd-merged.bin" \
            --pattern "wled-touch-remote-${tag}-jc4880p443-merged.bin" \
            --pattern "wled-touch-remote-${tag}-jc8048w550c-merged.bin" \
            --pattern "wled-touch-remote-${tag}-waveshare-p4-4b-merged.bin"
          mv "web-installer/firmware/wled-touch-remote-${tag}-esp32-cyd-merged.bin" \
            web-installer/firmware/wled-touch-remote-wifi.bin
          mv "web-installer/firmware/wled-touch-remote-${tag}-jc4880p443-merged.bin" \
            web-installer/firmware/wled-touch-remote-jc4880p443.bin
          mv "web-installer/firmware/wled-touch-remote-${tag}-jc8048w550c-merged.bin" \
            web-installer/firmware/wled-touch-remote-jc8048w550c.bin
          mv "web-installer/firmware/wled-touch-remote-${tag}-waveshare-p4-4b-merged.bin" \
            web-installer/firmware/wled-touch-remote-waveshare-p4-4b.bin
```

- [x] **Step 4: Add its manifest entry**

In the same file, change:

```bash
          printf '%s\n' '{"name":"WLED Touch Remote — ESP32-S3 (JC8048W550C)","version":"${{ steps.release.outputs.tag }}","new_install_improv":true,"builds":[{"chipFamily":"ESP32-S3","parts":[{"path":"firmware/wled-touch-remote-jc8048w550c.bin","offset":0}]}]}' > web-installer/manifest-jc8048w550c.json
```

to:

```bash
          printf '%s\n' '{"name":"WLED Touch Remote — ESP32-S3 (JC8048W550C)","version":"${{ steps.release.outputs.tag }}","new_install_improv":true,"builds":[{"chipFamily":"ESP32-S3","parts":[{"path":"firmware/wled-touch-remote-jc8048w550c.bin","offset":0}]}]}' > web-installer/manifest-jc8048w550c.json
          printf '%s\n' '{"name":"WLED Touch Remote — ESP32-P4 (Waveshare P4-4B)","version":"${{ steps.release.outputs.tag }}","new_install_improv":true,"builds":[{"chipFamily":"ESP32-P4","parts":[{"path":"firmware/wled-touch-remote-waveshare-p4-4b.bin","offset":0}]}]}' > web-installer/manifest-waveshare-p4-4b.json
```

- [x] **Step 5: Commit**

```bash
git add .github/workflows/build-release.yml .github/workflows/publish-web-installer.yml
git commit -m "Add waveshare-p4-4b to release build and web installer publishing"
```

(This task's correctness can only be fully checked by an actual CI run, which happens on the next real release — there is nothing more to verify locally beyond the YAML being well-formed, which the commit step implicitly does since these are plain strings, not templated YAML structures.)

---

### Task 10: Document the new board

**Files:**
- Modify: `README.md`

- [x] **Step 1: Add it to "Supported hardware"**

Change:

```markdown
## Supported hardware

Supported devices:

- **Guition ESP32-P4 JC4880P443, 4.3-inch display** - highly recommended
- **Guition JC2432W328C** - recommended capacitive CYD
- **Guition JC8048W550C, 5-inch 800x480 display** (untested)
- **ESP32-024 and ESP32-2432S028-style resistive CYDs** - largely supported but **not recommended**
```

to:

```markdown
## Supported hardware

Supported devices:

- **Guition ESP32-P4 JC4880P443, 4.3-inch display** - highly recommended
- **Guition JC2432W328C** - recommended capacitive CYD
- **Guition JC8048W550C, 5-inch 800x480 display** (untested)
- **Waveshare ESP32-P4-WIFI6-Touch-LCD-4B, 720x720 display** (preliminary; UI not yet tuned for the square screen)
- **ESP32-024 and ESP32-2432S028-style resistive CYDs** - largely supported but **not recommended**
```

- [x] **Step 2: Add it to the "Build locally" instructions**

Change:

```markdown
For the JC4880P443, use `pio run -e jc4880p443`; for the JC8048W550C, use `pio run -e jc8048w550c`. See `platformio.ini` for all available environments and `include/app_config.h` for board-specific options.
```

to:

```markdown
For the JC4880P443, use `pio run -e jc4880p443`; for the JC8048W550C, use `pio run -e jc8048w550c`; for the Waveshare ESP32-P4-WIFI6-Touch-LCD-4B, use `pio run -e waveshare-p4-4b`. See `platformio.ini` for all available environments and `include/app_config.h` for board-specific options.
```

- [x] **Step 3: Add it to the macOS simulator list**

Change:

```markdown
# 480x800 ESP32-P4 JC4880P443
pio run -e macos-jc4880p443
.pio/build/macos-jc4880p443/program

# 800x480 ESP32-S3 JC8048W550C
pio run -e macos-jc8048w550c
.pio/build/macos-jc8048w550c/program
```

to:

```markdown
# 480x800 ESP32-P4 JC4880P443
pio run -e macos-jc4880p443
.pio/build/macos-jc4880p443/program

# 800x480 ESP32-S3 JC8048W550C
pio run -e macos-jc8048w550c
.pio/build/macos-jc8048w550c/program

# 720x720 ESP32-P4 Waveshare P4-4B
pio run -e macos-waveshare-p4-4b
.pio/build/macos-waveshare-p4-4b/program
```

- [x] **Step 4: Commit**

```bash
git add README.md
git commit -m "Document Waveshare ESP32-P4-WIFI6-Touch-LCD-4B support"
```

---

## Manual hardware verification (you, on the physical board)

Nothing past this point can be verified from here — it needs the real board connected over USB.

- [x] Flash: `pio run -e waveshare-p4-4b -t upload` (or `pio run -e waveshare-p4-4b -t upload -t monitor` to also watch boot logs)
- [x] Confirm the ROM banner in the boot log — if it says `esp32p4-eco2`, switch `board = esp32-p4_r3-evboard` to `board = esp32-p4-evboard` in `platformio.ini` and re-flash (see Task 2, Step 1's comment) — **the tested unit reported `esp32p4-eco2`; switched profile, see commit d9550b1**
- [x] Confirm the panel lights up and shows the WLED Touch Remote splash/UI, not a blank or garbled screen (if blank/garbled: check the `WSP4B_DSI_LDO_CHANNEL`/`WSP4B_DSI_LDO_MV` assumption flagged as a risk in the spec) — confirmed working on real hardware
- [x] Confirm touch responds (tap through a couple of tabs) — confirmed working on real hardware
- [x] Confirm Wi-Fi setup works (Settings → Wi-Fi, join a 2.4 GHz network) and the device discovers or connects to a WLED controller — confirmed working on real hardware
- [x] Confirm Settings → Software Update → Check for Updates doesn't error out (validates the `kBuildTarget` string end to end once a release exists) — confirmed working on real hardware

---

## Self-review notes

- **Spec coverage:** every section of the design spec (board config, display bring-up, backlight, Wi-Fi/OTA, PlatformIO environment, CI/release/docs housekeeping) has a corresponding task above. The spec's "out of scope" items (square UI tuning, camera/audio/SD, touch calibration) are not addressed here, as intended.
- **Type/name consistency:** `WSP4B_*` macros introduced in Task 1 are the exact names referenced in Tasks 5 and 6; `kBuildTarget` string `"waveshare-p4-4b"` in Task 8 matches the PlatformIO environment name from Task 2 and the CI asset names in Task 9.
- **No placeholders:** every step has literal code or an exact command; the two genuine open questions (DSI LDO channel/voltage, silicon revision) are called out explicitly as manual-verification items rather than hidden as TODOs.
