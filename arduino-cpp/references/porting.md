# Porting: what actually changes between AVR, ESP8266, ESP32, and STM32

## 1. Capability matrix

Treat this as the *decision* table. If a feature you are about to use is not in
the "common to all" row set, it needs an arch guard.

| Topic | AVR (Uno/Nano/Mega) | ESP8266 (Arduino core 3.1.x) | ESP32 (Arduino core 2.0.x) | STM32 (STM32duino 2.10.x) |
| ----- | ------------------- | ---------------------------- | -------------------------- | -------------------------- |
| Core | 8-bit AVR, 16 MHz | 32-bit Tensilica L106, 80 MHz (160 opt) | Dual Xtensa LX6, 240 MHz | ARM Cortex-M0/M3/M4, 32–180 MHz |
| Flash | 32 KB (0.5 KB bootloader) / 256 KB | 512 KB – 4 MB (board-dependent) | 4 MB (partitioned) | 32 KB – 2 MB |
| SRAM | 2 KB (Uno/Nano) / 8 KB (Mega) | ~80 KB usable | ~320 KB DRAM (+ PSRAM option) | 6–96 KB |
| Logic level | 5 V | **3.3 V, NOT 5 V tolerant** | **3.3 V, NOT 5 V tolerant** | 3.3 V (F1/F3 partly 5 V tolerant) |
| `analogRead` | 10-bit, ~4 reads / 100 µs | **one pin only** (`A0`), 10-bit, 0–1.0 V | 12-bit, ~**ADC2 unusable while WiFi/BT on** | 12-bit, many channels, oversampling |
| `analogWrite` | Real 8-bit timer PWM | software PWM ~1 kHz, 10-bit (0–1023) | LEDC peripheral, needs freq + resolution set up | 12-bit timer PWM on most pins |
| PWM-capable pins | 3,5,6,9,10,11 (Uno) | most GPIO (software PWM) | almost any GPIO via LEDC | most GPIO, but per-timer channel rules |
| `Serial` | 5 V UART, needs DTR to reset via USB-UART | UART0 + TX-only `Serial1`; `swap()` option | UART0 (or USB CDC) | USB-CDC or UART, board-dependent |
| `EEPROM` lib | real 1 KB / 4 KB, no begin/commit | flash emulation, `begin(size)` + `commit()` required | no EEPROM — use `Preferences` (NVS) | real Flash-backed emulation |
| `dtostrf` | required for float→string | not required (`Serial.print(float)` fine) | not required | not required |
| C++ standard | gnu++11 | gnu++11 | gnu++11 | gnu++11 (some board builds gnu++14) |
| Exceptions / RTTI | off | off | off by default | off by default |
| Full STL | no | partial, allocation-heavy | partial, allocation-heavy | partial |
| FPU | software float only | software float only | hardware single-precision | hardware single-precision |
| Interrupt model | tiny ISRs, shared vectors, `cli()` | ISR in IRAM (`ICACHE_RAM_ATTR`), `yield()` or WDT resets | ISR must live in IRAM, FreeRTOS-aware | standard Cortex-M NVIC, one handler per vector |
| FreeRTOS | no | no (WiFi runs on the same core) | yes (ESP32) | optional (some cores) |

Nano note: a Nano (ATmega328P) is Uno silicon in a small footprint — same
2 KB SRAM, same A4/A5 (I2C) and 11/12/13 (SPI) pins, LED on D13. PlatformIO
board IDs: `nanoatmega328` (old bootloader, 57600 baud) vs `nanoatmega328new`
(115200 baud); cheap clones usually need the old-bootloader variant. Nano
**Every** is a different chip (megaAVR, `atmelmegaavr` platform) — not covered
by the `uno`/`nano` envs.

## 2. Detect the arch, don't guess

Cores define these macros (all of them are in the toolchain's predefines, so no
extra cost):

```cpp
#if defined(ARDUINO_ARCH_AVR)
#elif defined(ARDUINO_ARCH_ESP8266)
#elif defined(ARDUINO_ARCH_ESP32)
#elif defined(ARDUINO_ARCH_STM32) || defined(ARDUINO_ARCH_STM32F1) || defined(ARDUINO_ARCH_STM32F4)
#elif defined(ARDUINO_ARCH_RP2040) || defined(ARDUINO_ARCH_MBED)
#elif defined(ARDUINO_ARCH_SAMD)
#endif
```

Test `ARDUINO_ARCH_ESP8266` before `ARDUINO_ARCH_ESP32`: the two never coincide,
but putting the smaller sibling first keeps the "cheap board" branch obvious.
ISR functions need an IRAM attribute that differs per core — define one alias:

```cpp
#if defined(ARDUINO_ARCH_ESP32)
#define ISR_ATTR IRAM_ATTR
#elif defined(ARDUINO_ARCH_ESP8266)
#define ISR_ATTR ICACHE_RAM_ATTR
#else
#define ISR_ATTR   // AVR/STM32: no placement attribute needed
#endif

void ISR_ATTR onPulse() { /* flag/counter only, see concurrency.md */ }
```

ESP32 note: `ARDUINO_ARCH_MBED` is also defined on some ESP32-S2/S3+C3 builds.
Test `ARDUINO_ARCH_ESP32` first, `ARDUINO_ARCH_MBED` last.

To dump what a build really sees, print it once and keep it in a scratch build.
Use only `print`/`println` here — `Serial.printf` exists on ESP32/STM32 but **not**
on AVR, so a `printf`-based dump does not compile for `uno`:

```cpp
void printArch(void) {
#if defined(ARDUINO_ARCH_ESP32)
  Serial.print(F("esp32 chip=")); Serial.print(ESP.getChipModel());
  Serial.print(F(" rev="));       Serial.print(ESP.getChipRevision());
  Serial.print(F(" cores="));     Serial.print(ESP.getChipCores());
  Serial.print(F(" sdk="));       Serial.println(ESP.getSdkVersion());
#elif defined(ARDUINO_ARCH_AVR)
  // RAMEND/RAMSTART come from <avr/io.h> via Arduino.h; FLASHEND likewise.
  Serial.print(F("avr ram="));
  Serial.print((unsigned)RAMEND - (unsigned)RAMSTART + 1u);
  Serial.print(F(" flash="));
  Serial.println((unsigned)FLASHEND + 1u);
#elif defined(ARDUINO_ARCH_STM32)
  Serial.println(F("stm32"));
#else
  Serial.println(F("unknown arch"));
#endif
}
```

Pin maps live in the variant headers — read them instead of reciting numbers:

```bash
grep -rn "define LED_BUILTIN\|static const uint8_t SDA\|SCL\|SS\b" \
  ~/.platformio/packages/framework-arduinoespressif32/variants/*/pins_arduino.h
ls ~/.platformio/packages/framework-arduinoststm32/variants/
```

## 3. Voltage hazards (the #1 field bug)

- ESP32 GPIO is **3.3 V only**. A 5 V signal on GPIO 34-39 is not just
  "unsupported", it can damage the pin. Power a 5 V sensor through a divider or
  a level shifter; drive 3.3 V logic into 5 V inputs.
- 5 V-tolerant ESP32 pins are only the ones the strapping list allows, and the
  set differs by module. Default to assuming **nothing** is 5 V tolerant.
- An ESP32/STM32 pin cannot source 5 V. Power sensors from `VIN`/`5V` and share
  a common ground; never tie two boards' `5V` rails together blindly.
- Powered-down peripherals with active pull-ups back-feed IO pins. Add a
  series resistor or a MOSFET gate control on anything that sleeps.

## 4. ESP32 strapping and reserved pins

- **Strapping pins** are sampled at reset: GPIO 0 (boot mode), 2, 5, 12 (MTDI —
  a **1.0 V** flash chip pull it *low* at boot, 3.3 V breaks boot), 15. Never
  hard-wire a strap pin low; never let an LED+resistor to ground decide boot.
- GPIO 34-39 are **input-only** and have **no internal pull-up/pull-down** and
  no output driver. `pinMode(pin, OUTPUT)` silently does nothing useful, and
  `INPUT_PULLUP` does not work. Add an external 10 kΩ pull-up if you need one.
- GPIO 6-11 are the SPI flash pins — do not touch them (a flying wire here
  bricks the boot loop).
- ADC1 = GPIO 32-39. ADC2 = GPIO 0,2,4,12-15,25-27. **ADC2 is unavailable while
  WiFi/BLE is active**; use ADC1 for anything that must work online.
- DAC is only on GPIO 25/26. GPIO 32/33 are unreliable on ESP32-WROVER modules
  (they are used by the PSPI flash on those variants) — avoid them if unsure.
- Deep-sleep wake source: `esp_sleep_enable_ext1_wakeup()` takes an
  `esp_ext1_wakeup_pin_t` bitmask, which is **uint64** on chips with >32 GPIOs.

## 5. AVR specifics

- Uno/Nano: 2 KB SRAM is the real budget. A single `Serial.println(String(...))`
  can eat 100+ bytes. If SRAM is over ~75 %, something is allocating.
- `F()` only matters on AVR. Writing `F("x")` on ESP32/STM32 is harmless (the
  macro degrades to the literal) — so use it unconditionally and stop thinking
  about it.
- `pgm_read_byte` / `pgm_read_word` / `pgm_read_dword`, `memcpy_P`,
  `strlen_P`, `strcpy_P`, `snprintf_P` (with `PSTR()`) for flash-resident strings
  and tables — all verified on `uno` with the installed avr core.
- There is **no** `Serial.print_P` / `println_P` in the Arduino AVR core. To
  print a `PROGMEM` string, either use `F("...")` for literals, or copy it out
  first: `strcpy_P(buf, kMsg); Serial.print(buf);`.
- Large lookup tables belong in `PROGMEM`:
  `const uint16_t lut[] PROGMEM = {...};` then `pgm_read_word(&lut[i])`.
- AVR has no `double` — it is the same as `float` (32-bit). Never assume 64-bit
  float math.
- `random()`/`randomSeed()` differ per core (ESP32 uses its own RNG). For
  reproducible sequences implement a tiny xorshift yourself.
- 16-bit `int` and 8-bit `char` are traps in shifts and index math; use
  `int32_t`/`uint32_t` for anything computed.

## 6. STM32 specifics

- The pin *number* is meaningless across STM32 boards; a "Blue Pill D13" is PB3
  on one and something else on another. The variant header is the only truth, so
  every number goes in `Board.h` and nothing else.
- Several timers are claimed by `millis`/`micros`, and some timers are reserved
  by the core for PWM. Touching `HAL` timers directly will break `millis()`.
- `Serial` on Blue Pill is hardware USART1 (PA9/PA10) or USB-CDC, depending on
  the variant menu; `Serial1`/`Serial2`/`Serial3` names follow the variant
  header, so read it instead of guessing.
- `analogRead` channels map to ADC instances; some ADC pins are also used by
  timers/FSMC. Prefer the Arduino-level API over direct HAL for portability.
- F103 is Cortex-M3 with **no FPU**: heavy `float` math is meaningfully slower
  than on an F4. Keep `double` usage at zero.

## 7. ESP8266 specifics

- **Strapping at reset:** GPIO0 low = flash mode, GPIO2 high, GPIO15 low,
  EN/CH_PD high. Wrong strapping → `ets Jan 8 2013, rst cause:2, boot
  mode:(x,y)` boot loop. Never hard-wire these pins to a sensor that drives
  them at power-up.
- **Flash mode** (`board_build.flash_mode`): most modules want `dio` or `dout`;
  a QIO image on a DIO-only flash chip boot-loops. If a known-good firmware
  suddenly boot-loops after a board change, the flash mode is suspect #1.
- **Single core + background WiFi:** a loop that never calls `yield()`,
  `delay()`, or a blocking network read ends in `wdt reset`. Sprinkle
  `yield()` in long loops, or `ESP.wdtFeed()` in tight ones.
- **Deep sleep:** `ESP.deepSleep(microseconds)` — timer wake needs GPIO16
  wired to RST, otherwise it sleeps exactly once. See `connectivity.md` §6.
- **WiFi API differs from ESP32:** `WiFi.hostname("x")` (not `setHostname`),
  `WiFi.setAutoReconnect(true)`, `WiFi.persistent(false)` (stops the core
  rewriting SSID/pass to flash on every connect), `ESP.getResetReason()`.
- **HTTP include differs:** `#include <ESP8266HTTPClient.h>` (not
  `<HTTPClient.h>`); the class name `HTTPClient` is the same.
- `analogWrite` range is 0–1023 (10-bit) at ~1 kHz; `analogRead` is the single
  A0 pin (§ADC in `peripherals.md`).
- ~80 KB heap is plenty until TLS + JSON + WebSocket stack up — BearSSL alone
  wants kilobytes (see `protocols.md` §HTTPS). Keep TLS buffers small
  (`setBufferSizes(512, 512)`) and JSON documents modest.

## 8. Adding a new target board

1. `pio pkg install -g -p <platform>` if the platform is missing (e.g.
   `atmelavr`, `atmelsam`, `nordicnrf52`).
2. Add a `[env:<board>]` block; copy an existing env, change `board = <id>`.
3. `pio boards <platform>` to list valid board IDs.
4. `pio run -e <board>` and read the flash/SRAM report.
5. Update `Board.h` with the pin constants and any new `#if` branch.
6. Add the new env to `default_envs` only when it is expected to stay green.

Keep the number of envs small and honest: an env nobody builds is a lie waiting
to be discovered as a broken release.
