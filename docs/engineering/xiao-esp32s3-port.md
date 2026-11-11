# XIAO ESP32-S3 port — engineering baseline

Status: **design / not validated on hardware**  
Source baseline: smalltv-mod 3.6.0, fork commit `0da303ccbd2eb7e9dc7e7505e3f2c697f58629bd`  
Development branch: `feat/xiao-esp32s3`  
Scope: reuse the original GeekMagic SmallTV Ultra 1.54-inch display with a custom PCB and a Seeed Studio XIAO ESP32-S3.

## Executive decision

Port the existing firmware instead of rewriting the UI and networking stack. **No XIAO firmware image is available yet** and **no stock SmallTV Ultra device should be flashed with an ESP32 image**. The upstream ESP8266 firmware is preserved separately in `JohnCooper06/smalltv-ultra`.

## Verified software seams

- `src/Target.h` validates chip-family selections and defines the `SMALLTV_FAMILY_ESP32` shim.
- `src/Platform.h` and `src/Platform.cpp` abstract network, time, reset and update APIs by MCU family.
- `src/config.h` maps target-specific headers and defines `UPDATE_ASSET` / `FW_VARIANT`.
- `src/board_esp8266.h`, `src/board_esp32c2.h`, `src/board_esp32.h` and `src/board_esp32_pro.h` define existing LCD pin maps, colour order, backlight and other board features.
- `src/Gfx.cpp` uses explicit SPI GPIOs on the ESP32 family and drives a 240×240 ST7789 through Arduino_GFX in SPI mode 3.
- `platformio.ini` pins build toolchains and dependencies. `.github/workflows/build.yml` has firmware build and native tests.
- `tools/ci_checks.py` checks update asset names against the workflow's staged release binaries and enforces OTA partition size limits.

## Provisional hardware assumptions — NOT production-ready

| Item | Working assumption | Required verification |
| --- | --- | --- |
| MCU | Seeed Studio XIAO ESP32-S3 | Confirm exact board variant and PlatformIO board ID |
| LCD | GMT154-06, 1.54-inch, 240×240, ST7789-class SPI | Confirm LCD marking, datasheet and controller |
| FPC | 10 contacts, 0.5 mm pitch | Confirm pad orientation, contact side and connector part number |
| SPI SCK | XIAO D8 / GPIO7 | Verify against official Seeed pinout and schematic |
| SPI MOSI | XIAO D10 / GPIO9 | Verify against official Seeed pinout and schematic |
| LCD CS | XIAO D1 / GPIO2 | Verify LCD CS connection and correct boot constraints |
| LCD D/C | XIAO D3 / GPIO4 | Verify |
| LCD RESET | XIAO D0 / GPIO1 | Verify |
| Backlight control | XIAO D6 / GPIO43 | Verify pin availability and whether the PCB uses active-high or active-low driver input |
| PCB outline | 32 × 36 mm preliminary within a ~36 × 40 × 40 mm enclosure | Measure the original board, slide grooves, screw location and USB-C clearance |

**Electrical hold point:** do not connect the LCD backlight directly to a GPIO. The driver topology, LED current limit, supply rail and PWM logic must be validated from the actual LCD variant and intended PCB design. The stock SmallTV's active-low backlight behavior does **not** establish the correct polarity of a new custom PCB.

**Manufacturing hold point:** do not release Gerbers/BOM/assembly files before the FPC pinout, contact orientation, mechanical drawing and backlight current are confirmed.

## Porting hazards and implementation decisions

1. Give the new board its **own target macro** (proposed `SMALLTV_ESP32S3`) and header (`src/board_xiao_esp32s3.h`). Extend the mutually exclusive chip-selection gate in `src/Target.h`; reuse the ESP32-family platform branch where compatible.
2. Add a **dedicated** PlatformIO environment (proposed `xiao_esp32s3`) only after confirming the board ID, flash/PSRAM and pin availability. Never reuse an ESP8266/ESP32-C2/classic-ESP32 factory image.
3. Give the new target its own distinct `FW_VARIANT` and **disable automatic GitHub self-update** until a correctly staged, tested, XIAO-specific release asset exists. Never allow the new target to fall through to the ESP8266 `smalltv-mod-firmware.bin` asset in `src/config.h`.
4. Check `Gfx.cpp` SPI setup, panel row/column offsets, IPS inversion, RGB/BGR order, reset sequencing and PWM output against a hardware smoke-test; preserve settings compatibility where safe.
5. Ensure first-boot recovery over XIAO USB-C and safe boot with the LCD disconnected. The original Ultra USB-C may be power-only, but XIAO USB-C provides a separate development and recovery path.
6. Treat OTA, crash recovery, network authentication, TLS validation and credentials storage as separate security review gates; upstream TLS client paths currently use `setInsecure()` and should **not** be represented as certificate-validated.

## Increment plan and acceptance criteria

### A. Baseline and protection (this document)
- [x] Confirm development branch on GitHub.
- [x] Identify firmware target, graphics and CI integration points.
- [x] Record hardware uncertainties and safe stop conditions.
- [ ] Review this document in a draft PR; keep `main` unchanged.

### B. Compile-only target (next PR increment)
- [ ] Add target macro and board header without modifying existing boards' behavior.
- [ ] Introduce dedicated PlatformIO environment with verified board/partition settings.
- [ ] Add a **compile-time or equivalent guard** that prevents an unverified build from being flashed accidentally.
- [ ] Ensure XIAO self-update cannot download any stock ESP8266 or other incompatible image.
- [ ] Add checks that catch wrong target/asset mappings.
- [ ] Compile on CI and report exact commit SHA, toolchain and firmware size; do not claim a hardware test.

### C. USB smoke-test firmware
- [ ] Boot with USB serial logging and display disconnected.
- [ ] Verify power rails and current budget.
- [ ] Connect LCD via a **verified** FPC/pinout and show solid red/green/blue, orientation and text.
- [ ] Check backlight dimming (0%, 10%, 50%, 100%) with current measurement.
- [ ] Report observable evidence, failures and recovery steps.

### D. Integration and hardware release
- [ ] Exercise Wi-Fi setup, Web UI, modes, settings, reboot recovery and OTA with the correct partition scheme.
- [ ] Validate display, USB and mounting geometry of the PCB; run electrical-rule and design-rule checks.
- [ ] Release KiCad source, Gerbers, drill files, BOM, placement and assembly documentation **only after** the hardware hold points clear.

## Rules for development

Use isolated branches, small commits and draft PRs. Keep `main` unchanged until code review, automated checks and explicit hardware validation are complete. A successful compile is **not** evidence that the screen, backlight, power or enclosure works. Keep all work in this existing fork, never duplicate it into an unrelated repository.
