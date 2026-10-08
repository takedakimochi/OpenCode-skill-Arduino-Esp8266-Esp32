# Build, flash, and test with PlatformIO

`pio` lives in a PlatformIO-managed venv on this machine:

```bash
PIO=~/.platformio/penv/bin/pio
$PIO --version      # PlatformIO Core 6.2.0
```

Add `export PATH="$HOME/.platformio/penv/bin:$PATH"` to your shell if you want
bare `pio`. Do not `pip install platformio` into the system Python.

## platformio.ini reference

```ini
[platformio]
default_envs = esp32dev
; src_dir = src            ; default
; lib_dir = lib
; data_dir = data
extra_configs =

[env:esp32dev]
platform = espressif32
board = esp32dev
framework = arduino
monitor_speed = 115200
upload_speed = 921600
upload_port = /dev/ttyACM0
monitor_dtr = 0
monitor_rts = 0
lib_deps =
    bblanchon/ArduinoJson@^7.0.4
    adafruit/Adafruit SSD1306
    adafruit/Adafruit GFX Library
lib_ldf_mode = chain+
lib_compat_mode = strict
build_flags =
    -Wall
    -Wextra
    -Iinclude
    -DLOG_LEVEL=3
build_src_flags =
    -DAPP_VERSION='"1.4.0"'
build_unflags =
    -std=gnu++11
; board_build.partitions = partitions.csv
; board_build.filesystem = littlefs
; upload_protocol = espota
; upload_port = 192.168.1.50
; upload_flags =
;     --port=3232
```

| Key | Why it matters |
| --- | -------------- |
| `lib_ldf_mode` | `chain+` is the safest for Arduino libraries that `#include` each other; `deep+` finds more but slows the build; `off` is fastest and needs explicit `lib_deps`. |
| `lib_compat_mode = strict` | Stops a library that claims to support every core from being linked on an incompatible one. |
| `build_src_flags` | Applies to `src/` only — use it for per-sketch `-D`s so they do not leak into libraries. |
| `build_unflags` | Removing a core-injected `-std=` is how you opt into a newer standard; verify per toolchain. |
| `monitor_dtr/rts = 0` | Stops the monitor from toggling DTR/RTS, which resets an ESP32 board right after opening. |
| `board_build.partitions` | Required once app+libs exceed the default 1.2 MB app slot. |

### Per-arch `build_flags` you will actually need

All of these macro names were checked against the installed
`framework-arduinoespressif32` 2.0.17 core.

```ini
; ESP32: make the native-USB port the "Serial" (S3/C3 boards)
build_flags = -DARDUINO_USB_CDC_ON_BOOT=1        ; otherwise Serial == UART0
build_flags = -DARDUINO_USB_MODE=1               ; 1 = hardware CDC/JTAG, bit 0
build_flags = -DCORE_DEBUG_LEVEL=1               ; 0..5, costs flash
build_flags = -DARDUINO_RUNNING_CORE=1           ; 0 or 1: which core runs loop()
build_flags = -DARDUINO_EVENT_RUNNING_CORE=1     # or, in sdkconfig.defaults
build_flags = -DARDUINO_ADC2_0                   ; use ADC2 despite the WiFi caveat

; AVR: no stdlib float print, no exceptions
build_flags = -fno-exceptions -fno-rtti
build_flags = -Wl,--gc-sections                  ; already on by default
```

`-DPIO_FRAMEWORK_ARDUINO_*` flags from PlatformIO 5.x / espressif32 6.x are
**dead** on this espressif32 7.1.1 platform: its builder has no reference to
them. For anything that is really an ESP-IDF `CONFIG_*` option (CPU frequency,
flash size, PSRAM, main stack size, FreeRTOS tick rate), drop an
`sdkconfig.defaults` file in the project root — that is the mechanism the builder
actually reads:

```ini
# sdkconfig.defaults
CONFIG_ESPTOOLPY_FLASHSIZE_4MB=y
CONFIG_FREERTOS_HZ=1000
CONFIG_COMPILER_OPTIMIZATION_SIZE=y
```

`-DARDUINO_ADC2_0` is the one exception worth remembering: it re-enables the ADC2
pins that the core otherwise refuses while WiFi is up.

### ESP32 partitions

Default table gives ~1.25 MB to the app. When a build reports
`Partition Table overflow` or the app just barely fits, ship a custom
`partitions.csv` at the project root and point the env at it with
`board_build.partitions = partitions.csv` (verified working on this platform —
the build log then says `Generating partitions .pio/build/esp32dev/partitions.bin`):

```csv
# Name,   Type, SubType, Offset,   Size,  Flags
nvs,      data, nvs,     0x9000,   0x5000,
otadata,  data, ota,     0xe000,   0x2000,
app0,     app,  ota_0,   0x10000,  0x140000,
app1,     app,  ota_1,   0x150000, 0x140000,
spiffs,   data, spiffs,  0x290000, 0x160000,
coredump, data, coredump,0x3F0000, 0x10000,
```

The `from 1310720 bytes` figure in the build log comes from this table, not from
the board JSON (`esp32dev.json` claims 4 MB), so a mismatch between the two is
normal and worth checking when a flash operation starts failing near the end of
the app. Pick 1.25 MB × 2 (OTA, simple), 1.9 MB × 2 (OTA), or one big app with no
OTA. The layout must match the flash size of the actual module — a 4 MB module
cannot use a table that addresses 8 MB.

## Commands

```bash
$PIO run                                  # build default_envs
$PIO run -e esp32dev                      # build one env
$PIO run -e esp32dev -e uno              # several at once — do this before "done"
$PIO run -e esp32dev -t upload            # build + flash
$PIO run -e esp32dev -t upload -t monitor # flash then monitor
$PIO run -e esp32dev -t size              # flash/SRAM table
$PIO run -e esp32dev -t size-data         # .data/.bss breakdown
$PIO run -e esp32dev -t clean             # wipe build products for this env
$PIO run -t clean                         # wipe everything
$PIO device list                          # serial ports (also lists DFU/CMake/UDC)
$PIO device monitor -b 115200 -e esp32dev
$PIO test -e native                       # host unit tests
$PIO check -e esp32dev --skip-packages    # static check (needs pio check tool)
$PIO pkg install -e esp32dev              # install lib_deps for an env
$PIO boards ststm32 | less                # valid board ids
$PIO run -e esp32dev -t compiledb         # clangd/IDE compile database
```

Always quote the build verdict from the log. Verified real output from this
machine for the reference project in this skill (same `main.cpp` on all three):

```text
; esp32dev
HARDWARE: ESP32 240MHz, 320KB RAM, 4MB Flash
RAM:   [=         ]   6.7% (used 21800 bytes from 327680 bytes)
Flash: [==        ]  21.1% (used 276393 bytes from 1310720 bytes)

; bluepill_f103c8
HARDWARE: STM32F103C8T6 72MHz, 20KB RAM, 64KB Flash
RAM:   [=         ]   7.0% (used 1432 bytes from 20480 bytes)
Flash: [===       ]  27.5% (used 18016 bytes from 65536 bytes)

; uno
HARDWARE: ATMEGA328P 16MHz, 2KB RAM, 31.50KB Flash
RAM:   [=         ]  14.1% (used 289 bytes from 2048 bytes)
Flash: [=         ]  13.8% (used 4452 bytes from 32256 bytes)
```

Note the AVR line says "RAM" while `pio run -e uno -t size` reports the same
number as `Data:  289 bytes (14.1% Full)`. On AVR that 2048 bytes is the *whole*
budget — the stack lives there and PlatformIO's percentage leaves no room for a
local array. Above ~75 % data on an Uno is a latent reset bug; above ~90 % it is
a guaranteed one.

## Flashing recipes per board

| Board | Command | Gotchas |
| ----- | ------- | ------- |
| ESP32 (UART) | `$PIO run -e esp32dev -t upload` | if the port vanishes on reset, add `monitor_dtr = 0`, `monitor_rts = 0`; hold BOOT and press EN if it waits for a download |
| ESP32 (native USB, S3/C3) | add `-DARDUINO_USB_CDC_ON_BOOT=1` | use the USB port, not the UART port; `upload_speed = 460800` is plenty |
| ESP32 OTA | `upload_protocol = espota`, `upload_port = <ip>` | needs the device already running an OTA-capable firmware and reachable |
| STM32 Blue Pill | `$PIO run -e bluepill_f103c8 -t upload` | needs a ST-Link or a bootloader on the board; 8 KB flash (F103C8) is tiny |
| STM32 Nucleo/Discovery | usually just works | the on-board ST-Link handles reset |
| AVR Uno/Nano | `$PIO run -e uno -t upload` | install platform first: `$PIO pkg install -g -p atmelavr`; Linux may need the user in `dialout` for `/dev/ttyACM*` |
| Native (host) | `$PIO test -e native` | no hardware at all |

If the port is missing: `ls /dev/ttyUSB* /dev/ttyACM*`, `dialout` membership,
and a different USB cable (charge-only cables are a classic).

## Host / native tests

`[env:native]` compiles the same `src/` on the host, so pure logic can be tested
without a board. The trick is a stub `Arduino.h` in `test/stubs/`.

`test/stubs/Arduino.h` (verified — the `inline` global avoids a second `.cpp`,
and the settable input lets tests drive `Button`-style debounce classes):

```cpp
#pragma once
#include <stdint.h>
#include <stddef.h>
#include <stdio.h>
#include <string.h>

#define HIGH 1
#define LOW  0
#define INPUT        0
#define OUTPUT       1
#define INPUT_PULLUP 2
#define RISING  3

// Analog pin numbers are board-specific (A0=14 on UNO, 34 on many ESP32 boards),
// so the stub only needs *a* number. Production code must use pins::kSensorAdc.
#define A0 14

inline void pinMode(uint8_t, uint8_t) {}
inline void digitalWrite(uint8_t, uint8_t) {}
inline uint8_t g_digital_read = HIGH;    // tests drive inputs through this
inline void setDigitalRead(uint8_t v) { g_digital_read = v; }
inline int  digitalRead(uint8_t) { return g_digital_read; }
inline int  analogRead(uint8_t) { return 512; }
inline void analogWrite(uint8_t, int) {}
inline void delay(uint32_t) {}
inline void delayMicroseconds(uint32_t) {}
inline uint32_t millis() { return 0; }
inline uint32_t micros() { return 0; }
inline int digitalPinToInterrupt(uint8_t p) { return p; }
inline void attachInterrupt(int, void (*)(), int) {}
inline void noInterrupts() {}
inline void interrupts() {}
inline void yield() {}

class SerialStub {
 public:
  void begin(unsigned long) {}
  int available() { return (int)(len_ - pos_); }
  int read() { return pos_ < len_ ? (int)(unsigned char)buf_[pos_++] : -1; }
  void print(const char*) {}
  void print(int) {}
  void println() {}
  void println(const char*) {}
  size_t write(int c) { if (len_ < sizeof buf_) buf_[len_++] = (char)c; return 1; }
  void feed(const char* s) { while (*s) write(*s++); }   // test helper
  void clear() { len_ = pos_ = 0; }
 private:
  char buf_[128]{};
  size_t len_ = 0, pos_ = 0;
};
inline SerialStub Serial;
```

`test/test_app/test_main.cpp`:

```cpp
#include <unity.h>
#include "Scheduler.h"

void setUp(void) {}
void tearDown(void) {}

static int calls = 0;
static void bump(uint32_t, void*) { ++calls; }

void test_scheduler_runs_all_tasks(void) {
  calls = 0;
  Scheduler s;
  TEST_ASSERT_TRUE(s.add(Task{bump, nullptr}));
  s.run(0);
  TEST_ASSERT_EQUAL_INT(1, calls);
}

int main(int, char**) {
  UNITY_BEGIN();
  RUN_TEST(test_scheduler_runs_all_tasks);
  return UNITY_END();
}
```

This exact layout builds and runs here (7 test cases, all passing, < 1 s).

Run it: `pio test -e native`. The first run auto-installs Unity into
`.pio/libdeps/native/Unity/`. Keep the stub intentionally thin — if production
code needs a richer Arduino API in tests, that is a signal the logic should be
pushed behind an interface instead of the stub growing.

Quick compile check without PlatformIO (Unity must already exist from a previous
`pio test`, or install it once with `pio pkg install -g -t tool-unity`):

```bash
g++ -std=gnu++17 -Wall -Wextra -I.pio/libdeps/native/Unity/src \
    -Itest/stubs -Iinclude -Isrc \
    -o /tmp/opencode/t test/test_app/test_main.cpp && /tmp/opencode/t
```

On AVR the same logic file is a genuinely useful compile gate, because it catches
"this uses a 64-bit type / an STL container / a 4 KB local" long before the
toolchain would have told you:

```bash
~/.platformio/penv/bin/pio run -e uno -t size   # Data: NNN bytes — the SRAM budget
```

## Adding libraries

```bash
$PIO pkg install -e esp32dev -l "adafruit/Adafruit SSD1306"   # ad-hoc
```

Prefer pinning in `platformio.ini` (`lib_deps = author/name@^x.y.z`) over ad-hoc
installs: the version in the manifest is the version CI and other machines get.

- `lib/` folders you commit are first in the search order, then
  `lib_deps`, then the framework.
- One `.cpp` per library folder named after the folder keeps the Library
  Dependency Finder fast and predictable.
- `extra_scripts = pre:scripts/patch_library.py` (Python, run by PlatformIO's
  own venv) is the supported way to patch a third-party library instead of
  forking it.

## If the user insists on the Arduino IDE / `.ino` workflow

`arduino-cli` is **not** installed here. Two honest options:

1. Convert the sketch into a PlatformIO project (`src/main.cpp` holds `setup()`
   and `loop()`; `#include <Arduino.h>` at the top; the IDE's function prototypes
   are unnecessary and should be deleted) — recommended.
2. Install the CLI and the core, then use
   `arduino-cli compile --fqbn esp32:esp32:esp32 <sketch>` and
   `arduino-cli upload -p /dev/ttyACM0 --fqbn esp32:esp32:esp32 <sketch>`.

Do not fake IDE behaviour by leaving `*.ino` files around: IDE-only automatic
prototype generation hides real C++ errors and breaks PlatformIO builds.
