# Waveshare ESP32-P4-WIFI6-Touch-LCD-4B board support

Date: 2026-09-14
Status: Approved, not yet implemented

## Summary

Add a new board target so WLED Touch Remote runs on the Waveshare
ESP32-P4-WIFI6-Touch-LCD-4B: ESP32-P4 + on-board ESP32-C6, 720x720 square
ST7703 MIPI-DSI panel, GT911 capacitive touch. This is the same overall
architecture as the already-supported Guition JC4880P443 (ESP32-P4 + C6,
MIPI-DSI panel, GT911, ESP-Hosted Wi-Fi) but with a different panel
controller, different resolution/shape, and different pin wiring.

Reference: <https://github.com/waveshareteam/ESP32-P4-WIFI6-Touch-LCD-4B/tree/main/examples/arduino>
(pin numbers and the ST7703 init sequence below were read from that repo's
`examples/arduino/libraries/displays/displays_config.h`.)

Scope for this pass: get the board booting with display, touch, Wi-Fi and
OTA working, reusing the existing UI screens as-is. This mirrors how
JC8048W550C was added ("preliminary" support) — layout tuning for the square
screen is an explicit follow-up once it can be seen running on real hardware,
not part of this change.

## New board id

`include/app_config.h`:

```c
#define WLED_BOARD_CYD 0
#define WLED_BOARD_JC4880P443 1
#define WLED_BOARD_JC8048W550C 2
#define WLED_BOARD_WAVESHARE_P4B 3
```

Folded into the existing capability flags (no new flags needed — it's DSI +
GT911 + PSRAM, same as JC4880P443):

```c
#define WLED_PANEL_DSI (WLED_BOARD == WLED_BOARD_JC4880P443 || WLED_BOARD == WLED_BOARD_WAVESHARE_P4B)
```

New screen-size block:

```c
#if WLED_BOARD == WLED_BOARD_WAVESHARE_P4B
#define WLED_SCREEN_WIDTH 720
#define WLED_SCREEN_HEIGHT 720
#define WLED_LVGL_BUFFER_LINES 40   // starting point; tune against JC4880P443's value once running
#define WLED_DISPLAY_ROTATION 0
#define WLED_DISPLAY_ROTATION_FLIPPED 2
#ifndef WLED_CYD_ENABLE_BATTERY
#define WLED_CYD_ENABLE_BATTERY 0
#endif
#endif
```

Add `WLED_BOARD_WAVESHARE_P4B` to the `CYD_HARDWARE_PROFILE` selection
(`CYD_PROFILE_ST7701_GT911` is fine to reuse, since it's only used to
disambiguate CYD auto-detection profiles and this board never goes through
CYD auto-detect):

```c
#elif WLED_BOARD == WLED_BOARD_WAVESHARE_P4B
#define CYD_HARDWARE_PROFILE CYD_PROFILE_ST7701_GT911
```

New board-specific pin/timing block, values from Waveshare's
`displays_config.h` (`SCREEN_DEFAULT`) and `i2c.h`:

```c
#elif WLED_BOARD == WLED_BOARD_WAVESHARE_P4B

// Panel is driven over MIPI-DSI; reset, backlight PWM and backlight enable
// are the only GPIOs. Timings match the panel's 720x720 ST7703 module.
#define WSP4B_TFT_RST 27
#define WSP4B_TFT_BL 26            // active-low PWM
#define WSP4B_TFT_BL_ENABLE 33     // gate; assert only after PWM is running
#define WSP4B_TFT_BL_FREQ 5000
#define WSP4B_TFT_BL_RES 10
#define WSP4B_PANEL_WIDTH 720
#define WSP4B_PANEL_HEIGHT 720
#define WSP4B_DSI_LANES 2
#define WSP4B_DSI_LANE_MBPS 480
#define WSP4B_DSI_LDO_CHANNEL 3    // same LDO channel as JC4880P443; confirm on hardware
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
#define WSP4B_TOUCH_RST -1   // not driven on this board
#define WSP4B_TOUCH_INT -1   // not driven on this board
#define WSP4B_TOUCH_ADDR 0x5D
#define WSP4B_TOUCH_I2C_PORT 0

#define GT911_TOUCH_SDA WSP4B_TOUCH_SDA
#define GT911_TOUCH_SCL WSP4B_TOUCH_SCL
#define GT911_TOUCH_RST WSP4B_TOUCH_RST
#define GT911_TOUCH_INT WSP4B_TOUCH_INT
#define GT911_TOUCH_ADDR WSP4B_TOUCH_ADDR

#define CYD_BOARD_CAPACITIVE 1
```

Note: `WSP4B_DSI_LDO_CHANNEL`/`WSP4B_DSI_LDO_MV` are carried over from
JC4880P443 as a starting assumption — Waveshare's example doesn't set these
explicitly (they're an ESP-IDF-level DSI PHY power concern, not something
their Arduino_GFX-based example needs to configure the same way this
project's native `esp_lcd_mipi_dsi` path does). Verify against the P4's DSI
PHY LDO requirements during bring-up if the panel doesn't come up.

## Display bring-up (`src/display.cpp`)

`initP4Panel()` and the handful of `#elif WLED_BOARD == WLED_BOARD_JC4880P443`
spots (splash text blit, rotation handling, etc. — currently ~4 locations)
need to branch on the active DSI board rather than being hardcoded to
JC4880P443. Two ways to do this, pick whichever reads cleaner once in the code:

- Switch those spots to check `WLED_PANEL_DSI` generically where the logic is
  identical for both boards (it should be, since both are plain top-left-origin
  portrait/square DSI framebuffers with no special mirroring), or
- Add explicit `WLED_BOARD_WAVESHARE_P4B` branches alongside the existing ones
  where behavior needs to differ.

Add a second vendor init-command table (ST7703) next to the existing ST7701S
one, selected by board. Transcribed from Waveshare's
`vendor_specific_init_default[]`:

```c
auto cmd = [](uint8_t c, std::initializer_list<uint8_t> p) { return p4Write(c, p.begin(), p.size()); };
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
```

(Note the longer `delay(250)`/`delay(50)` after sleep-out/display-on vs
JC4880P443's `delay(120)` — Waveshare's example uses 250ms/50ms; keep those
since they're panel-specific timing, not arbitrary.)

Reset sequence: same shape as JC4880P443 (`WSP4B_TFT_RST` low 20ms, high,
then 120ms settle) before sending the init table — Waveshare's example
doesn't reset before init (`rst_pin = -1` in their config, they rely on
power-on reset), but keeping an explicit reset pulse matches this project's
existing pattern and is safer across warm restarts/OTA.

Backlight: unlike JC4880P443's plain GPIO backlight, this board needs
PWM (`ledcAttach`/`ledcWrite`, active-low output, 5kHz/10-bit) on
`WSP4B_TFT_BL`, gated by driving `WSP4B_TFT_BL_ENABLE` high only once PWM is
attached. Add this alongside (not replacing) the existing GPIO backlight
path, selected by board the same way the DSI init table is.

## Wi-Fi / OTA

- No code changes expected for the ESP-Hosted C6 link itself — the embedded
  C6 firmware blob (`esp32c6-v2.12.11.bin`) ships with the Arduino platform
  package and is not board-specific; both boards use the same hosted
  protocol and the same reference SDIO wiring.
- `scripts/embed_hosted_firmware.py`: extend the `env.subst("$PIOENV") ==
  "jc4880p443"` check to also match the new environment name, or generalize
  it to "any DSI P4 board" — otherwise the new environment silently builds
  without the C6 firmware embedded and Wi-Fi never comes up.
- `src/update_manager.cpp`: add the new board's OTA asset-name string
  (matching the `esp32-cyd` / `jc4880p443` / `jc8048w550c` pattern) so
  Settings → Software Update accepts the right release asset.

## New PlatformIO environment

`[env:waveshare-p4-4b]`, cloned from `[env:jc4880p443]` with:

- `board = esp32-p4_r3-evboard` (rev3/post-v3 silicon profile — Waveshare's
  own Arduino IDE instructions default to `Chip Variant: v3.00 or newer` for
  current-production boards). If the board turns out to be older silicon,
  the boot log's ROM banner reports `esp32p4-eco2`; switch to
  `esp32-p4-evboard` (the pre-rev3 profile, same one JC4880P443 uses) in
  that case.
- Serial: drop `ARDUINO_USB_CDC_ON_BOOT=1` / `ARDUINO_USB_MODE=1` — this
  board's serial runs over a CH343P USB-UART bridge on UART0, not the P4's
  native USB-C CDC port that JC4880P443 uses.
- `monitor_speed = 115200`, `upload_speed` default (no native-USB fast
  upload path on this board).
- `-D WLED_BOARD=WLED_BOARD_WAVESHARE_P4B`
- `board_build.partitions = default_16MB.csv` — reused as-is even though
  this board has 32MB flash. Under-using the extra flash is harmless and
  avoids introducing a new partition scheme in this pass.
- Same `lib_deps`, `extra_scripts` (embed_logo, git_version,
  embed_hosted_firmware) as `jc4880p443`.

## CI / release / docs housekeeping

Add the new board id everywhere JC8048W550C was added:

- `.github/workflows/build-release.yml` — build matrix entry
- `.github/workflows/publish-web-installer.yml` — web installer manifest entry
- `README.md` — "Supported hardware" list, marked preliminary/untested like
  JC8048W550C currently is
- `include/app_config.h`'s "auto-detect vs one-time setup" comment at the top
  of the file, if it enumerates boards by name

## Out of scope for this pass

- UI/layout tuning for the square 720x720 screen (explicit follow-up once
  visible on hardware)
- Camera (OV5647), audio (ES8311/ES7210), microSD — present on the Waveshare
  board but unused by this application
- Touch coordinate calibration beyond wiring in the existing GT911 driver
  already shared by JC4880P443/JC8048W550C

## Risks / unknowns to resolve during bring-up

1. **DSI PHY LDO channel/voltage** (`WSP4B_DSI_LDO_CHANNEL`/`_MV`) — carried
   over from JC4880P443 as a guess; Waveshare's Arduino_GFX-based example
   doesn't need to configure this explicitly the way this project's native
   `esp_lcd_mipi_dsi` path does. If the panel doesn't light up, this is the
   first thing to check against the P4's DSI PHY requirements.
2. **Silicon revision** — defaulting to rev3 (`esp32-p4_r3-evboard`); one
   boot-log check away from being wrong, and simple to flip if so.
3. **`WLED_LVGL_BUFFER_LINES = 40`** is a starting guess (matches
   JC4880P443); 720x720 is a larger framebuffer than either existing DSI/RGB
   board, so this may need tuning against available internal RAM.
