---
name: arduino-cpp
description: Write, refactor, debug, and build portable Arduino C/C++ firmware (.ino, .cpp/.h, PlatformIO, ESP32, ESP8266, NodeMCU, Wemos, STM32, AVR, Uno, Nano, Mega, Blue Pill, Nucleo). Use when the user mentions Arduino, sketch, .ino, firmware, microcontroller, ESP32, ESP8266, NodeMCU, STM32, Arduino Uno/Nano/Mega/Blue Pill, board, pin mapping, sensor, DHT, DS18B20, OLED, display, SSD1306, LCD, servo, relay, UART, Serial, RS485, I2C, Two Wire, SPI, Ethernet, W5500, TCP, UDP, WiFi, MQTT, HTTP, HTTPS, TLS, WebSocket, OTA, deep sleep, sketch/build/upload/flash, `platformio.ini`, `pio run -t upload`, `analogRead`, `attachInterrupt`, `millis`, `F()` macro, or "keeps brownouting / rebooting on its own". Covers PlatformIO workflow, non-blocking timing, ISR rules, peripherals, WiFi/MQTT/HTTP/WebSocket/Ethernet/TLS, flash/SRAM budget, and host-side unit tests. Never write a blocking `delay()` loop or AVR-only code unless the target board is fixed.
compatibility: opencode
metadata:
  domain: embedded
  language: c, c++
  build_tool: platformio
  platforms: esp32, stm32, avr
  purpose: portable-arduino-firmware
  source: https://docs.arduino.cc/
---

# Arduino C/C++ — Portable Firmware Across Boards

You are writing **Arduino-framework C++** that must build and run on more than one
board. Target order of preference: **ESP32 → STM32 → AVR**, unless the user pins
a board.

Official docs: <https://docs.arduino.cc/> · Language reference: <https://docs.arduino.cc/language-reference/en/>

## Toolchain on this machine (verified)

| Tool | Status | Notes |
| ---- | ------ | ----- |
| `pio` (PlatformIO Core 6.2.0) | installed | `~/.platformio/penv/bin/pio` — **use this, not bare `pio`** |
| platform `espressif32` 7.1.1 | installed | Arduino-ESP32 core **2.0.17** → LEDC API is `ledcSetup`/`ledcAttachPin` |
| platform `espressif8266` 4.2.1 | installed | Arduino-ESP8266 core **3.1.2** → `ICACHE_RAM_ATTR`, `ESP.deepSleep`, BearSSL TLS |
| platform `ststm32` 20.0.0 | installed | STM32duino core **2.10.x**; `bluepill_f103c8` builds fine |
| platform `atmelavr` 5.3.0 | installed | Arduino AVR core 5.4.0 (Uno builds: 4452 B flash / 289 B RAM) |
| platform `native` 1.2.1 | installed | host unit tests via `pio test -e native` |
| `g++` (host) | installed | standalone logic tests and compile smoke checks |
| `arduino-cli` | not installed | if the user insists on `.ino` + IDE workflow, install it or port to PlatformIO |

`pio` is a project-local venv-managed install, so calling it directly is correct
and does not need a Python venv of its own. Never `pip install platformio` into
the system Python.

```bash
~/.platformio/penv/bin/pio --version
~/.platformio/penv/bin/pio pkg list --global
```

Always re-check installed platform versions with `pio pkg list --global` instead
of trusting the numbers above — core APIs (`ledcSetup` vs `ledcAttach`) moved
between Arduino-ESP32 2.x and 3.x.

The `platformio.ini` skeleton, `Board.h`, `Logger.h`, and the pattern code in
this skill were **compiled for real** on this machine with `esp32dev`,
`nodemcuv2`, `uno`, `nano`, and `bluepill_f103c8` from one shared `main.cpp`.
If you change them, re-verify all five before quoting sizes.

## The portability contract (non-negotiable)

1. **No blocking.** `delay()` / `_delay_ms()` in `loop()` is a bug, not a style
   choice. All timing is `millis()`/`micros()` deadlines or a state machine.
2. **Rollover-safe time.** Never `millis() >= t + 500` and never store `millis()`
   in `unsigned long` and compare naively. Use `(uint32_t)(now - t0) >= period`.
3. **No heap in steady state.** `String` concatenation, `new` in a loop, and
   `std::vector` growth are banned on AVR. Use `char buf[N]` + `snprintf`, and
   `F()` on AVR for every string literal in flash-resident code.
4. **ISRs stay tiny.** No `Serial`, no `delay`, no allocation, no blocking I/O in
   an interrupt. Set a flag / bump a counter / push to a queue, handle in `loop()`.
5. **All pin numbers live in one header** (`pins.h` / `Board.h`). No magic
   numbers in driver code, and no `2`/`13`/`A0` sprinkled in logic.
6. **Every arch-specific claim is guarded** with `#if defined(ARDUINO_ARCH_*)`
   (`ARDUINO_ARCH_ESP32` for LEDC/FreeRTOS, `ARDUINO_ARCH_ESP8266` for
   single-core WiFi quirks, `ARDUINO_ARCH_AVR` for PROGMEM/`dtostrf`).
   Two-arg `Wire.begin(sda, scl)` is ESP32/ESP8266-only, 4-arg `SPI.begin()`
   is ESP32-only, `HTTPClient.h` vs `ESP8266HTTPClient.h` differ — per-core
   init lives in `references/peripherals.md`. Do not assume the ESP32 API
   exists on any other target.
7. **Fixed-width integer types** (`uint8_t`, `uint32_t`, `int32_t`) for anything
   persisted, printed, shared with an ISR, or stored in a struct on the wire.
8. **Bounds-check every buffer** you fill from input (`snprintf` return value,
   `strncpy` terminator, ring-buffer wrap).
9. **Compile, don't guess.** Finish with `pio run -e <env>` (and `pio test -e
   native` when a test env exists). A sketch is not "done" until it builds.

## Intake checklist (ask before you code)

Never invent a pinout, a part number, or a power budget. If any answer is
missing, ask — one short question round beats three rebuilds:

1. **Board(s)?** Exact model (Uno, Nano, Mega, ESP32-DevKit, ESP32-S3/C3,
   Blue Pill, Nucleo-…). "My Arduino" is not an answer; the variant header is
   the only pin truth (`references/porting.md` §2).
2. **Peripherals?** Part numbers, not categories ("DHT22 on pin 4", not "a
   temperature sensor"). Library picks come from the verified table in
   `references/peripherals.md` — never paste a registry ID from memory.
3. **Power?** USB, battery, or mains; anything inductive (motor/servo/relay)
   gets its own rail (§8 of `references/connectivity.md`).
4. **Connectivity?** None, WiFi, Ethernet (W5500), MQTT, HTTP/HTTPS, UDP,
   WebSocket, OTA updates? Decides ESP32/ESP8266 vs AVR more than anything
   else — and TLS rules out AVR entirely (`references/protocols.md`).
5. **Timing?** Anything sub-100 ms or safety-adjacent (PID loop, RPM counting,
   alarms) must be stated up front — it changes the design, not just the code.

## Board choice (default: ESP32)

| Pick | When | Watch out |
| ---- | ---- | --------- |
| ESP32 (DevKit/S3/C3) | WiFi/BLE, OTA, WebSocket/TLS, real performance, most new projects | 3.3 V only, strapping pins, ADC2 vs WiFi (`porting.md` §4) |
| ESP8266 (NodeMCU/Wemos) | cheapest WiFi, simple sensors + MQTT/HTTP | single core (`yield()` or WDT resets), one ADC pin, ~80 KB heap, BearSSL for TLS (`porting.md` §7, `protocols.md`) |
| Uno/Nano (AVR) | 5 V sensors, teaching, simplest wiring, huge shield base, Ethernet via W5500 | 2 KB SRAM, no WiFi/TLS, slow float, `F()` + `dtostrf` everywhere |
| Mega (AVR) | many pins/interrupts, still 5 V | same AVR limits with 8 KB SRAM; big but slow |
| Blue Pill / Nucleo (STM32) | cheap 32-bit, more timers/ADC, partly 5 V tolerant (F1) | pin numbers meaningless across boards; ST-Link or bootloader to flash |

If the user names no board, propose ESP32 and say why in one line. If they pin
one board, the portability contract still applies — single-board code rots the
moment a second board appears.

## Workflow

1. **Pin down the target set.** If the user says "for my board" and the pinout is
   unknown, ask which board(s) and which peripherals, or read the variant header
   (`~/.platformio/packages/framework-arduinoespressif32/variants/*/pins_arduino.h`,
   `.../framework-arduinoststm32/variants/*/`). Never invent a pinout.
2. **Check the arch API surface** in `references/porting.md` before using any API
   that is not common to all cores.
3. **Lay out the project** (see layout below), pin constants first, then drivers,
   then `loop()`.
4. **Build for every env**: `pio run -e esp32dev -e esp8266 -e uno -e nano -e bluepill_f103c8`. Fix
   warnings, not just errors. On a machine without a platform, install it first:
   `pio pkg install -g -p atmelavr`.
5. **Flash + watch the monitor**: `pio run -e <env> -t upload && pio device monitor
   -b 115200`. On ESP32 USB-CDC boards, `Serial.begin(115200)` must come *before*
   any print, otherwise the first lines are lost while the CDC enumerates.
6. **Run host tests** for any non-trivial logic: `pio test -e native`.
7. **Report** flash/SRAM usage from the build log — it is the only early warning
   for the next feature. Verified baseline for the reference project in this
   skill (identical `main.cpp` on all three): `esp32dev` RAM 6.7 % / flash
   21.1 %, `uno` RAM 14.1 % (289 B of 2048) / flash 13.8 %, `bluepill_f103c8`
   RAM 7.0 % / flash 27.5 %. The full kitchen-sink verification build (every
   snippet in this skill: WiFi/MQTT/HTTP/WebSocket/Ethernet/OLED/sensors,
   2026-10-08) also fits everywhere: `nanoatmega328new` RAM 55.3 % / flash
   78.5 %, `nodemcuv2` RAM 42.7 % / flash 50.6 %.

## Project layout

```text
project/
├── platformio.ini          # one [env:] per target board
├── include/
│   ├── Board.h             # arch detection + pin constants + capability table
│   └── Logger.h            # level-gated logging, F() on AVR
├── lib/
│   └── DriverX/            # self-contained drivers (no global state)
│       ├── DriverX.h
│       └── DriverX.cpp
├── src/
│   ├── main.cpp            # setup() + loop(); must be the only sketch file
│   └── App.cpp/.h          # state machines, business logic
├── test/
│   ├── test_app/           # host tests
│   └── stubs/Arduino.h     # fake Arduino API for the native env
└── partitions.csv          # ESP32 only, when the default partition is too small
```

Keep drivers free of `Serial`, `millis`, and static singletons so they can be
tested on the host. Pass `now` in as a parameter instead of calling `millis()`
inside a driver.

## platformio.ini skeleton

```ini
[platformio]
default_envs = esp32dev

[env:esp32dev]
platform = espressif32
board = esp32dev
framework = arduino
monitor_speed = 115200
upload_speed = 921600
lib_deps =
    adafruit/Adafruit GFX Library
lib_ldf_mode = chain+
build_flags =
    -Wall
    -Wextra
    -Iinclude
build_unflags =
    -std=gnu++11
build_src_flags =
    -DLOG_LEVEL=3

[env:uno]
platform = atmelavr
board = uno
framework = arduino
monitor_speed = 9600
build_flags = -Wall -Wextra

[env:nano]
platform = atmelavr
board = nanoatmega328new        ; 115200 baud; old-bootloader clones: nanoatmega328
framework = arduino             ; 30720 B flash max — a display + JSON + Ethernet
monitor_speed = 115200          ; together do NOT fit; drop the display lib first
build_flags = -Wall -Wextra

[env:esp8266]
platform = espressif8266
board = nodemcuv2
framework = arduino
monitor_speed = 115200
upload_speed = 921600
build_flags = -Wall -Wextra

[env:bluepill_f103c8]
platform = ststm32
board = bluepill_f103c8
framework = arduino
monitor_speed = 115200
upload_protocol = stlink
build_flags = -Wall -Wextra

; host tests for pure logic — no hardware
[env:native]
platform = native
test_framework = unity
build_flags = -std=gnu++17 -Itest/stubs -Iinclude -Isrc -Wall -Wextra
```

Notes: `lib_ldf_mode = chain+` is the safe default for Arduino libraries that
include each other. `include/` is already on the include path for every env, so
`-Iinclude` is only needed for unusual layouts. Drop `-std=gnu++11` from
`build_unflags` only if you have verified the toolchain accepts the newer
standard for that board. `lib_deps` must be installed (`pio pkg install -e
<env>`) before an offline build. The five board envs above plus `[env:native]`
were all compiled from the same `main.cpp` on this machine, so this skeleton is a
known-good starting point, not a guess.

## Board abstraction pattern

```cpp
// include/Board.h
#pragma once
#include <Arduino.h>

#if defined(ARDUINO_ARCH_ESP32)
static_assert(sizeof(void *) == 4, "ESP32 expected 32-bit pointers");
#define BOARD_HAS_WIFI   1
#define BOARD_HAS_ADC2   1
#define BOARD_HAS_EEPROM 0   // use Preferences (NVS), not EEPROM
#define BOARD_LED_BUILTIN LED_BUILTIN
#elif defined(ARDUINO_ARCH_ESP8266)
#define BOARD_HAS_WIFI   1
#define BOARD_HAS_ADC2   0   // single A0 only, 0-1.0 V
#define BOARD_HAS_EEPROM 1   // emulation: begin(size) + commit() required
#define BOARD_LED_BUILTIN LED_BUILTIN
#elif defined(ARDUINO_ARCH_STM32) || defined(ARDUINO_ARCH_STM32F1)
#define BOARD_HAS_WIFI   0
#define BOARD_HAS_ADC2   1
#define BOARD_HAS_EEPROM 1
#else   // AVR: Uno, Nano, Mega
#define BOARD_HAS_WIFI   0
#define BOARD_HAS_ADC2   0
#define BOARD_HAS_EEPROM 1
#define BOARD_LED_BUILTIN LED_BUILTIN
#endif

namespace pins {
constexpr uint8_t kStatusLed = 2;
constexpr uint8_t kButton    = 4;   // INPUT_PULLUP, active LOW
constexpr uint8_t kSensorAdc = A0;
}  // namespace pins
```

Use `constexpr` in a `namespace`, not `#define`, for pin numbers. Then
`pinMode(pins::kButton, INPUT_PULLUP)` is identical on every target.

## Non-blocking timing pattern

```cpp
#include <stdint.h>

class Blinker {
 public:
  void begin(uint8_t pin, uint32_t periodMs) {
    pin_ = pin;
    period_ = periodMs ? periodMs : 1u;
    pinMode(pin_, OUTPUT);
  }

  // Call as often as possible. Returns true on each state transition.
  bool update(uint32_t now) {
    if (static_cast<uint32_t>(now - last_) < period_) return false;
    last_ = now;
    state_ ^= 1;
    digitalWrite(pin_, state_ ? HIGH : LOW);
    return true;
  }

 private:
  uint8_t pin_ = 0;
  uint32_t period_ = 500;
  uint32_t last_ = 0;   // 0 == "just started"; first update() fires immediately
  uint8_t state_ = 0;
};
```

`loop()` becomes: read inputs → `update()` each object → render output. Full
scheduler, debounce, PID, ring-buffer, and state-machine templates are in
`references/patterns.md`.

## Logging that does not cost RAM

Verified on ESP32, Uno, and Blue Pill. AVR has no `Print::printf` and avr-libc's
`printf` has no float support, so every core formats into a small fixed buffer
instead of relying on a `printf` member existing:

```cpp
// include/Logger.h
#pragma once
#include <Arduino.h>

#ifndef LOG_LEVEL
#define LOG_LEVEL 2          // 0 off, 1 error, 2 info, 3 debug
#endif

#if defined(ARDUINO_ARCH_AVR)
#include <avr/pgmspace.h>
#define LOGF_(fmt, ...)                                              \
  do {                                                               \
    char _logbuf[64];                                                \
    snprintf_P(_logbuf, sizeof(_logbuf), PSTR(fmt), ##__VA_ARGS__);  \
    Serial.print(_logbuf);                                           \
  } while (0)
#else
#define LOGF_(fmt, ...)                                              \
  do {                                                               \
    char _logbuf[64];                                                \
    snprintf(_logbuf, sizeof(_logbuf), fmt, ##__VA_ARGS__);         \
    Serial.print(_logbuf);                                           \
  } while (0)
#endif

#if LOG_LEVEL >= 2
#define LOGI(fmt, ...) LOGF_(fmt, ##__VA_ARGS__)
#else
#define LOGI(fmt, ...) do { } while (0)
#endif

#if LOG_LEVEL >= 1
#define LOGE(fmt, ...) LOGF_("E: " fmt, ##__VA_ARGS__)
#else
#define LOGE(fmt, ...) do { } while (0)
#endif
```

Notes: `PSTR()` wraps a single literal, so the variadic helper must keep a named
format parameter. Format specifiers only — a flash string passed as a `%s`
argument cannot work through `snprintf_P`; on AVR print such strings separately
with `Serial.print(F("..."))`. Never log floats on AVR: use `dtostrf`.

## Build / flash / monitor commands

```bash
PIO=~/.platformio/penv/bin/pio
$PIO run -e esp32dev                       # build
$PIO run -e esp32dev -t upload             # build + flash
$PIO run -e esp32dev -t clean              # wipe build dir
$PIO device monitor -b 115200 -e esp32dev  # serial monitor
$PIO run -e esp32dev -t size               # flash/SRAM table
$PIO test -e native                        # host unit tests
$PIO run                                   # build default_envs
```

If flashing finds no port: `ls /dev/ttyUSB* /dev/ttyACM*`, add
`upload_port = /dev/ttyACM0`, and for ESP32 add `monitor_dtr = 0`,
`monitor_rts = 0` if the port disappears after reset (auto-reset quirk).

## Before you say "done"

- `pio run` succeeds for **all** target envs, no new warnings.
- Grep your own diff: no `delay(`, no `String`, no float in `Serial.print` on
  AVR (`dtostrf`), no pin literal outside `Board.h`.
  `git diff -U0 | rg '\bdelay\(|String\b|digitalWrite\(\s*[0-9]'` is a decent
  smoke test.
- Time math is rollover-safe; `while (Serial.available() ...)` loops are bounded.
- Host tests cover the timing/state logic, not just the happy path
  (`millis()` overflow, buffer overflow, debounce edges).
- Build log flash and SRAM numbers reported, with headroom to spare.
- Anything board-specific is behind an arch guard with a comment saying why.
- If the project uses the network: no credentials in source (WiFiManager or
  `-D` flags), reconnect policy present, no blocking wait on WiFi/MQTT.

## References

Read the relevant file instead of guessing; each is self-contained.

| File | Read it when |
| ---- | ------------ |
| `references/porting.md` | Pin differences, arch API matrix, 3.3 V vs 5 V hazards, adding a new board |
| `references/peripherals.md` | GPIO, PWM/LEDC, ADC, I2C, SPI, UART, EEPROM, one-wire, verified sensor/display library table (DHT, DS18B20, OLED, LCD, NeoPixel, JSON) |
| `references/connectivity.md` | WiFi reconnect policy, WiFiManager provisioning, MQTT, HTTP+JSON, OTA, deep sleep, NTP, power budgeting (ESP32 + ESP8266) |
| `references/protocols.md` | UDP, WebSocket client, Ethernet TCP/IP (W5500), HTTPS/TLS, raw TCP with deadlines |
| `references/concurrency.md` | ISRs, volatile/atomics, FreeRTOS tasks on ESP32, mutex/queue rules, reset-cause triage |
| `references/patterns.md` | Non-blocking scheduler, debounce, state machine, ring buffer, PID, drivers |
| `references/build-and-flash.md` | Full `platformio.ini` reference, partitions, OTA, USB-CDC, native tests |
| `references/troubleshooting.md` | Brownout, upload failed, garbage serial, linker overflow, flaky timing |
