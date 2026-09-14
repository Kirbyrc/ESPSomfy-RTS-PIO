# PlatformIO Port Changes

The goal of this fork/port is to build and run ESPSomfy-RTS inside the
PlatformIO environment in VSCode, rather than the Arduino IDE. This file
documents the changes made on top of the ported ESPSomfy-RTS-PIO project
during debugging of a watchdog-reboot crash on "Save Radio Settings"
(Waveshare ESP32-S3-Zero + CC1101). 

Original project: [rstrouse/ESPSomfy-RTS](https://github.com/rstrouse/ESPSomfy-RTS)
(forked from version 2.4.7)

## Porting to PlatformIO

The original project is an Arduino IDE sketch (all sources in the repo
root). Getting it building and running as a PlatformIO project took:

1. **Moved every source file into `src/`** (`Somfy.cpp`/`.h`, `Web.cpp`/`.h`,
   `MQTT.cpp`/`.h`, `SomfyController.ino`, etc.) — a pure file move with no
   content changes, since PlatformIO expects sources under `src/` rather
   than flat in the repo root.
2. **Renamed the `Network` class to `SomfyNetwork`** (`Network.cpp`/`.h` →
   `SomfyNetwork.cpp`/`.h`, plus every `#include "Network.h"` and
   `extern Network net;` reference across the codebase) to avoid a name
   collision with `Network.h`/`NetworkClient` types that ship as part of
   the Arduino-ESP32 core PlatformIO pulls in.
3. **Added `platformio.ini`**, targeting `board = waveshare_esp32_s3_zero`
   with `framework = arduino`, and declared the project's dependencies as
   PlatformIO `lib_deps` (`ArduinoJson`, `PubSubClient`, `WebSockets`, and
   at the time, the registry copy of `SmartRC-CC1101-Driver-Lib`) instead
   of relying on the Arduino IDE's Library Manager.
4. **Updated `.gitignore`** to exclude PlatformIO's `.pio/` build directory.
5. **Fixed ESP-IDF/Arduino-ESP32 core API changes** needed to compile
   against the newer core PlatformIO resolved (the original project
   targeted an older core version):
   - `esp_task_wdt_init(7, true)` (old two-argument signature) became the
     `esp_task_wdt_config_t` struct + `esp_task_wdt_init(&wdtConfig)` form
     the current core requires.
   - `tcpip_adapter_get_ip_info(...)` (a removed legacy IDF API used in
     `SSDPClass::localIP()`) was replaced with the portable
     `WiFi.localIP()` / `ETH.localIP()` accessors.
   - `esp_chip_info()`/`esp_chip_model_t` usage (`Somfy.cpp`, `GitOTA.cpp`)
     needed an explicit `#include <esp_chip_info.h>`, since that API moved
     out of the core headers implicitly pulled in before.
   - `EthernetSettings` stopped storing `eth_phy_type_t`/`eth_clock_mode_t`
     enum values directly, since the RMII/EMAC-specific Ethernet PHY types
     those enums come from only exist on chips with EMAC hardware
     (original ESP32), not the ESP32-S3 this port targets — those fields
     now just hold raw `uint8_t` values.
6. **Remapped the CC1101 SPI/GDO default pins** for ESP32-S3, since the
   original project's classic-ESP32 VSPI pins (18/19/23/5) don't carry
   over: GPIO23 doesn't exist on the S3, and GPIO19/20 are the chip's
   native USB D-/D+ lines. (These defaults were tuned further afterward,
   in a separate commit, to match the Waveshare ESP32-S3-Zero's specific
   wiring — see the pin values in `Somfy.cpp`.)

After the code was ported and building cleanly, a bug turned up during
testing on the actual hardware.

## Bug behavior

Saving radio settings (from the web UI, "Save Radio Settings") reliably
crashed the device: the task watchdog would fire and the ESP32 would reboot
mid-save, every time, inside the CC1101 driver's SPI transfer code.

## Root cause

Two independent bugs in the vendored `SmartRC-CC1101-Driver-Lib` (v2.5.7)
were causing the crash above on current ESP32 Arduino cores.

## Changes

### 1. Vendored and patched `SmartRC-CC1101-Driver-Lib`

The library dependency `lsatan/SmartRC-CC1101-Driver-Lib @ 2.5.7` was
removed from `platformio.ini` (`lib_deps`) and replaced with a local,
patched copy at `lib/SmartRC-CC1101-Driver-Lib/`, so the fixes below persist
across library reinstalls instead of being silently overwritten by the
unpatched registry version.

**a. Removed invalid `digitalWrite()` calls in `Init()`**
(`ELECHOUSE_CC1101_SRC_DRV.cpp`, `Init()`)

The driver called `digitalWrite(SCK_PIN, HIGH)` and
`digitalWrite(MOSI_PIN, LOW)` immediately after `SPI.begin()` had already
bound those two pins to the ESP32's hardware SPI peripheral. On the current
Arduino-ESP32 core this is rejected by the pin-ownership check
(`IO 7 is not set as GPIO. Execute digitalMode(7, OUTPUT) first.`) and
corrupts the SPI peripheral's internal state, causing the next
`SPI.transfer()` in `RegConfigSettings()` to hang forever and trip the task
watchdog. `SPI.begin()` already drives these lines to the correct idle
levels, so the calls were simply removed (with a comment explaining why).

**b. Made `SpiStart()` / `SpiEnd()` idempotent instead of
begin/end-per-transfer**
(`ELECHOUSE_CC1101_SRC_DRV.cpp`, `SpiStart()`, `SpiEnd()`, `setSpiPin()`)

The original driver called `SPI.begin()` and `SPI.end()` around *every
single* register read/write/strobe (`SpiWriteReg`, `SpiReadReg`,
`SpiStrobe`, etc. each call `SpiStart()`/`SpiEnd()` individually). On the
current ESP32-S3 core, this rapid begin/end churn on a shared bus hangs the
SPI hardware (`spiTransferByte` spins forever waiting on the `cmd.usr`
completion flag), again tripping the watchdog and rebooting the device.

Fixed by adding a static `spiBusStarted` flag: `SpiStart()` now only calls
`pinMode()`/`SPI.begin()` once, and `SpiEnd()` is a no-op (the bus stays
open for reuse). `setSpiPin()` was updated to explicitly call `SPI.end()`
and reset the flag only when the pin assignment actually changes, so
runtime radio pin reconfiguration (changing pins from the web UI) still
works correctly.

### 2. Blue LED indicator for WiFi hotspot (SoftAP) mode

(`src/SomfyNetwork.cpp`, `SomfyNetwork::networkEvent()`)

The onboard RGB LED already turned red at boot and green once WiFi
connects (pre-existing behavior). Added:
- `ARDUINO_EVENT_WIFI_AP_START` → LED turns **blue** while the device's
  SoftAP/hotspot is open (i.e. it couldn't join WiFi and is waiting for
  configuration).
- `ARDUINO_EVENT_WIFI_AP_STOP` → LED reverts to **red** once the hotspot
  closes, so it doesn't stay stuck blue.

## Building the project

This project targets the `esp32s3zero` environment defined in
`platformio.ini` (Waveshare ESP32-S3-Zero, `framework = arduino`).

**Prerequisites:** either the [PlatformIO IDE extension for
VSCode](https://platformio.org/install/ide?install=vscode), or PlatformIO
Core (the `pio` CLI) installed standalone.

**Using VSCode:** open this folder in VSCode with the PlatformIO extension
installed — it will pick up `platformio.ini` automatically. Use the
PlatformIO toolbar/status bar icons (or the PlatformIO sidebar's Project
Tasks) to Build, Upload, Upload Filesystem Image, and Monitor.

**Using the CLI**, from the project root:

```sh
# Build the firmware
pio run -e esp32s3zero

# Build and flash the firmware to a connected board
pio run -e esp32s3zero -t upload

# Build and flash the web UI (everything under data/) to the board's
# LittleFS partition - required for the web interface to work, and
# needs to be done separately from flashing the firmware
pio run -e esp32s3zero -t uploadfs

# Open a serial monitor (115200 baud, per platformio.ini)
pio device monitor
```

The compiled firmware lands at `.pio/build/esp32s3zero/firmware.bin`.
Both `upload` and `uploadfs` need the board connected over USB; if more
than one serial device is attached, add `--upload-port <COMx>` to target
the right one.
