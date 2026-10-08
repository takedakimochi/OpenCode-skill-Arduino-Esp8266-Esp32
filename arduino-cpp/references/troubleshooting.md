# Troubleshooting: symptoms → cause → fix

Work top-down: reproduce, then read the build/boot log, then change one thing.
Never "fix" firmware by adding delays; that hides a race and adds latency.

## Build-time

| Error | Cause | Fix |
| ----- | ----- | ---- |
| `'X' was not declared in this scope` (a plain function like `pinMode`) | Missing `#include <Arduino.h>` — PlatformIO does not auto-generate IDE prototypes | Add the include; delete IDE-style forward declarations |
| `Serial was not declared` | Compiling outside the Arduino env (host editor, or `test/` on a board env) | Add the `test/stubs/Arduino.h` stub, or fix the include |
| `undefined reference to __dso_handle` | Using `new`/`delete` or a `String` on AVR (`-fno-exceptions -fno-use-cxa-atexit`) | Remove heap usage; use a fixed buffer |
| `no matching function for call to attachInterrupt(int, ..., int)` | AVR needs `digitalPinToInterrupt()` | `attachInterrupt(digitalPinToInterrupt(pin), fn, mode)` |
| `region `iram0_0_seg' overflowed` / `section will not fit in region text` | Flash full | `-Os` (default), drop `-Og`, cut libraries, enable `-ffunction-sections`/`gc-sections`, log with `F()` |
| `region `dram0_0_seg' overflowed` | SRAM full (common: big `String`, buffer arrays, `Serial` buffers) | Cut buffers, use `F()`, `update()` instead of `write()`, check with `-t size-data` |
| `ld returned 1 exit status` + `multiple definition of setup` | Two files define `setup`/`loop` | Only `src/main.cpp` may. Verified: a hard link error on ESP32, AVR, and STM32 alike — the ESP32 link line also force-includes them with `-u _Z5setupv -u _Z4loopv`, so a missing `loop()` is an undefined-reference error, not a silent no-op |
| `Nothing to build` / `warning: Skipping ...` | `srcFilter` excluded everything, or files are in the wrong dir | `build_src_filter = +<*> -<.git/>` |
| `PlatformIOPackageException: Could not find the package` | Missing platform/lib | `pio pkg install -e <env>` |
| `...flash has insufficient space` on ESP32 | App > partition slot | Ship `partitions.csv` (see `build-and-flash.md`) |
| `error: invalid conversion from 'const char*' to 'char*'` | `Serial.println(String)` overload drift, or writing a literal through a non-const API | `const_cast` is a smell; use `print(F("..."))` or fix the signature |

## Linker / memory sanity

- Run `pio run -e <env> -t size` after *every* meaningful feature. RAM is the
  constraint, not flash.
- On AVR, `Serial.begin(115200)` costs ~2 KB flash and the print machinery costs
  more; keeping logs behind a level that compiles to nothing is the cheapest win.
- `PROGMEM` tables plus `pgm_read_*` for anything > ~100 bytes of constants.

## Flashing

| Symptom | Cause | Fix |
| ------- | ----- | --- |
| `Failed to connect to ESP32: Timed out waiting for packet header` | Auto-reset not triggered (DTR/RTS wiring, wrong port, USB-UART vs native USB) | Hold BOOT, tap EN; try the other port; `upload_speed = 115200`; unplug peripherals from the UART pins |
| `A fatal error occurred: Failed to connect` | Another program holds the port, or ESP is in deep sleep | Close the monitor first (`pio device monitor` must be stopped); remove `GPIO0` straps |
| Port disappears right after flash | Monitor toggling DTR/RTS | `monitor_dtr = 0`, `monitor_rts = 0` |
| `avrdude: verification error` / `st-flash write error` | Brownout, bad cable, USB power | Check 3.3 V rail, use a powered hub, lower `upload_speed`, shorter wires |
| `avrdude: stk500v2_getsync(): timeout` | Board's auto-reset circuit is broken (cheap clones) | Press RESET manually at the right moment, or `upload_protocol = arduino:avrdude` with manual reset |
| Same on a Nano, but Uno works | Wrong bootloader variant: old-bootloader clone on `nanoatmega328new` (or vice versa) | Switch `board = nanoatmega328` ↔ `nanoatmega328new`; old bootloader also needs `upload_speed = 57600` |
| ESP8266 `espcomm_sync failed` / `timed out waiting for packet header` | GPIO0 not low at reset, EN floating, or wrong USB-serial driver (CH340/CP2102) | Pull GPIO0 low + tap RST/EN; EN needs a 10 kΩ pull-up on bare modules; install the right USB driver; `upload_speed = 115200` |
| ESP8266 flashes fine but boot-loops (`ets Jan 8 2013, rst cause:2`) | Strapping pins driven at reset, or wrong flash mode (QIO image on DIO flash) | Free GPIO0/2/15 at boot; set `board_build.flash_mode = dout` (or `dio`) and re-flash |
| `/dev/ttyACM*` missing | User not in `dialout` | `sudo usermod -aG dialout $USER` then re-login (or a new session) |
| `Could not open port, Permission denied` | Same | Same |
| Blue Pill flashes but does nothing | No bootloader, wrong flash size, or `option bytes` locked | Flash the STM32duino bootloader once; set `board_build.flash_size`/`upload.maximum_size` to the real chip |

## Runtime

### Board resets by itself

1. **Print the reset reason first** (see `concurrency.md` §5) and read the serial
   log at the exact baud for the board.
2. `ESP_RST_BROWNOUT` / AVR BOD reset: the 3.3 V rail sagged. Usually one of:
   - WiFi TX bursts on a marginal USB supply → bigger bulk cap (≥470 µF) plus
     100 nF at the module, or a better supply.
   - A motor/relay/solenoid on the same rail → separate rail, common ground,
     flyback diodes, and never switch an inductive load from a GPIO directly.
    - Brownout detector level too high (AVR: the `BODLEVEL` fuses — read them
      with `avrdude -U lfuse:r:-:h -U hfuse:r:-:h -U efuse:r:-:h` and decode
      with a fuse calculator; ESP32: no user setting — the chip's own
      threshold is fixed) → fix the supply. Disabling the BOD hides a hardware
      fault and lets the core run at an undefined voltage.
   - On STM32, lowering the voltage scale to `LOWPOWER` trades brownout
     tolerance for current — usually the wrong direction.
3. WDT resets: a blocking call in a task (`vTaskDelay`), or a task that never
   yields. On ESP32, `yield()` inside long loops is better than nothing but a
   `vTaskDelay` is correct. On ESP8266, `wdt reset` means a loop ran too long
   without `yield()`/`delay()` — sprinkle `yield()` (see `concurrency.md` §6).
   On AVR, `wdt_enable(WDTO_2S)` (note the names are
   `WDTO_1S`/`WDTO_2S`/`WDTO_4S`/`WDTO_8S`, not `WDTO_2000MS`) plus
   `wdt_reset()` in `loop()`, or the board reboots forever.
4. ESP8266 `rst cause:2` boot loop with `boot mode:(x,y)`: the chip never
   reached your code — strapping or flash mode (see Flashing table above and
   `porting.md` §7). `rst cause:4` after running = software WDT (see step 3).
5. Stack overflow: bump `xTaskCreatePinnedToCore`'s stack size, or reduce
   recursion/large locals. `CONFIG_ESP32_TASK_WDT` aborts with
   `abort() was called` plus a backtrace — read the backtrace.
6. AVR stack overflow from large locals/arrays: move them to `static`/globals
   (2 KB stack!). AVR `int` is 16-bit — a `long` array of 128 entries is 512 bytes.

### Serial prints garbage or nothing

- Wrong baud (Uno/Nano 9600 default in the IDE, but `Serial.begin(115200)` in
  code is what counts), wrong USB port (native USB vs UART on ESP32-S3/C3).
- ESP32 native USB: `Serial` is only valid with
  `-DARDUINO_USB_CDC_ON_BOOT=1`; otherwise use `USBSerial` from
  `USB.h`, or the UART port.
- Prints before the CDC handshake are lost — call `Serial.begin()` first, and
  hold output until the port is up (bounded by a timeout).
- AVR float printing: `Serial.print(1.5)` prints `?` — use `dtostrf`.
- Garbage mid-stream: an ISR is printing, or two writers share `Serial`.

### Timing is wrong / jittery

- `delay()` chains add up and drift. Use deadline comparisons
  (`patterns.md` §2) and let `loop()` run free.
- `micros()` on AVR is ±4 µs and cannot be called from an ISR.
- ESP32: WiFi interrupts preempt your code; keep timing-critical work in a task
  pinned to the other core, and never disable interrupts for long.
- `tone()`/`noTone()` steals a timer on AVR and breaks `analogWrite` on the
  shared pins.
- First-call jitter: a `static` initialiser, lazy library setup, or a first
  `analogRead`. Warm the hardware in `setup()` before the timing-critical phase.

### I2C / SPI device not responding

- Confirm the pin numbers used are the ones you wired (see `Board.h` /
  `pins_arduino.h` for the variant — the default `SDA`/`SCL` differ per board).
- Logic level: a 5 V I2C pull-up rail into an ESP32 pin is a hardware risk.
- Scope it: 4.7 kΩ pull-ups, 100 kHz first, address bit (7-bit vs 8-bit, `<< 1`),
  and a 4.7 kΩ–10 kΩ pull-up on INT/open-drain lines.
- `Wire.requestFrom` returning fewer bytes than expected → device needs a
  settling delay after power-up, or its clock stretches.
- SPI returning `0xFF` → MISO is pulled up, device not selected, or the SPI mode
  is wrong. Verify with `SPI.beginTransaction(SPISettings(...))` every time.

## When a bug resists analysis

- Turn on everything that was compiled out: `-DLOG_LEVEL=3 -DCORE_DEBUG_LEVEL=3`.
  Then turn them back off and confirm the fix still holds.
- Slow time down: measure only what you need, print a heartbeat with a counter
  (not a timestamp you trust) so you can see how many iterations happened.
- Bisect the build: `build_src_filter` to compile half the app at a time.
- Re-read the datasheet for the specific part. Arduino abstracts the chip; the
  errata are where the real answers live.
- Do not add `delay()` to make it "stop happening". Find the shared resource.
