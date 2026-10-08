# OpenCode Skill Arduino ESP8266 ESP32

Skill `arduino-cpp` untuk OpenCode: menulis, refactor, debug, dan build firmware Arduino C/C++ yang **portabel antar board** (`.ino`, `.cpp/.h`, PlatformIO, ESP32, ESP8266, STM32, AVR).

Urutan target prioritas: **ESP32 → STM32 → AVR**, kecuali user mengunci satu board.

Sumber resmi: https://docs.arduino.cc/ dan https://docs.arduino.cc/language-reference/en/

Repo ini berisi:
```
.
├── arduino-cpp/
│   ├── SKILL.md            # definisi utama skill (kontrak, workflow, pola kode)
│   └── references/         # panduan detail per topik
│       ├── porting.md
│       ├── peripherals.md
│       ├── connectivity.md
│       ├── protocols.md
│       ├── concurrency.md
│       ├── patterns.md
│       ├── build-and-flash.md
│       └── troubleshooting.md
└── python-virtual-environment/
    └── SKILL.md            # skill pendamping: wajib pakai venv sementara di /tmp/opencode
```

## Fitur Skill

- Firmware Arduino-framework C++ multi-board dari satu `main.cpp` yang sama.
- Board didukung:
  - ESP32 (DevKit / S3 / C3) — WiFi/BLE, OTA, WebSocket/TLS
  - ESP8266 (NodeMCU v2 / Wemos) — WiFi murah, MQTT/HTTP
  - Uno / Nano / Mega (AVR) — 5V, shield, simpel
  - Blue Pill / Nucleo (STM32) — 32-bit murah, ADC/timer lebih banyak
- Peripheral: GPIO, PWM/LEDC, ADC, I2C, SPI, UART, EEPROM, 1-Wire, DHT, DS18B20, OLED SSD1306, LCD, NeoPixel, JSON, servo, relay.
- Konektivitas: WiFi, WiFiManager, MQTT, HTTP+JSON, HTTPS/TLS, WebSocket, UDP, TCP, Ethernet W5500, OTA, deep sleep, NTP.
- Aturan keras:
  1. Tanpa `delay()` blocking di `loop()` — pakai deadline `millis()`/`micros()` atau state machine.
  2. Waktu harus rollover-safe: `(uint32_t)(now - t0) >= period`.
  3. Tanpa heap di steady-state pada AVR (larang `String`, `new` di loop).
  4. ISR super pendek (flag/counter saja).
  5. Semua pin di satu header (`Board.h` / `pins.h`), tanpa angka magis.
  6. Kode arch-specific wajib `#if defined(ARDUINO_ARCH_*)`.
  7. Tipe integer fixed-width (`uint8_t`, `uint32_t`).
  8. Wajib build sukses untuk semua env target + lapor angka flash/SRAM.

## Prasyarat / Toolchain Terverifikasi

Yang dipakai skill ini (sudah dikompilasi nyata untuk `esp32dev`, `nodemcuv2`, `uno`, `nano`, `bluepill_f103c8`):

| Tool | Status |
| ---- | ------ |
| `~/.platformio/penv/bin/pio` (PlatformIO Core 6.2.0) | terinstal — **pakai path penuh ini, bukan `pio` biasa** |
| `espressif32` 7.1.1 (Arduino-ESP32 2.0.17, API `ledcSetup`/`ledcAttachPin`) | terinstal |
| `espressif8266` 4.2.1 (Arduino-ESP8266 3.1.2) | terinstal |
| `ststm32` 20.0.0 (STM32duino 2.10.x) | terinstal |
| `atmelavr` 5.3.0 | terinstal |
| `native` 1.2.1 + `g++` host | untuk `pio test -e native` |
| `arduino-cli` | tidak terinstal — hanya jika butuh workflow `.ino` + IDE |

> Selalu cek ulang versi dengan `pio pkg list --global`, jangan percaya angka di atas mentah-mentah — API `ledcSetup` vs `ledcAttach` pindah antara Arduino-ESP32 2.x dan 3.x.

## Cara Instalasi

### 1. Instal skill ke OpenCode

Skill ini `compatibility: opencode`.

Opsi A — global (semua project):
```bash
mkdir -p ~/.config/opencode/skills
cp -r arduino-cpp ~/.config/opencode/skills/
ls ~/.config/opencode/skills/arduino-cpp/SKILL.md
```

Opsi B — project saja:
```bash
mkdir -p .opencode/skills
cp -r arduino-cpp .opencode/skills/
```

Restart / reload OpenCode, lalu panggil dengan kata kunci: `arduino`, `sketch`, `.ino`, `firmware`, `ESP32`, `ESP8266`, `NodeMCU`, `platformio.ini`, `pio run -t upload`, dll.

### 2. Instal PlatformIO Core (jika belum ada)

`pio` di skill ini adalah install venv-lokal, jadi panggil langsung — jangan `pip install platformio` ke system Python:

```bash
~/.platformio/penv/bin/pio --version
~/.platformio/penv/bin/pio pkg list --global
```

Jika `~/.platformio/penv/bin/pio` belum ada, instal resmi PlatformIO Core lalu pastikan path itu muncul.

### 3. Instal platform yang dibutuhkan

```bash
PIO=~/.platformio/penv/bin/pio
$PIO pkg install -g -p atmelavr
$PIO pkg install -g -p espressif32
$PIO pkg install -g -p espressif8266
$PIO pkg install -g -p ststm32
$PIO pkg install -g -p native
```

### 4. Verifikasi instalasi

```bash
PIO=~/.platformio/penv/bin/pio
$PIO --version
$PIO pkg list --global
$PIO run --help
$PIO test --help
```

## Cara Menggunakan

### 1. Siapkan jawaban intake (jangan dikarang AI)

Sebelum coding, jawab 5 hal ini:

1. **Board?** Model persis (Uno, Nano-new/bootloader-lama, ESP32-DevKit/S3/C3, NodeMCU v2, Blue Pill, Nucleo-...).
2. **Periferal?** Nomor part + pin ("DHT22 di pin 4", bukan "sensor suhu").
3. **Power?** USB / baterai / mains. Motor/servo/relay wajib rel sendiri.
4. **Konektivitas?** None / WiFi / Ethernet W5500 / MQTT / HTTP/HTTPS / UDP / WebSocket / OTA. TLS mengharuskan ESP32/ESP8266, bukan AVR.
5. **Timing?** Sebut jika ada yang <100 ms atau safety (PID, RPM, alarm).

Jika board tidak disebut, default yang diusulkan skill: **ESP32**.

### 2. Buat project PlatformIO

Struktur baku:
```
project/
├── platformio.ini
├── include/
│   ├── Board.h
│   └── Logger.h
├── lib/DriverX/DriverX.{h,cpp}
├── src/main.cpp + App.{cpp,h}
├── test/test_app/ + stubs/Arduino.h
└── partitions.csv   # ESP32 saja bila perlu
```

Skeleton `platformio.ini` minimal:

```ini
[platformio]
default_envs = esp32dev

[env:esp32dev]
platform = espressif32
board = esp32dev
framework = arduino
monitor_speed = 115200
upload_speed = 921600
build_flags = -Wall -Wextra

[env:esp8266]
platform = espressif8266
board = nodemcuv2
framework = arduino
monitor_speed = 115200

[env:uno]
platform = atmelavr
board = uno
framework = arduino
monitor_speed = 9600
build_flags = -Wall -Wextra

[env:nano]
platform = atmelavr
board = nanoatmega328new
framework = arduino
monitor_speed = 115200

[env:bluepill_f103c8]
platform = ststm32
board = bluepill_f103c8
framework = arduino
monitor_speed = 115200
upload_protocol = stlink

[env:native]
platform = native
test_framework = unity
build_flags = -std=gnu++17 -Itest/stubs -Iinclude -Isrc -Wall -Wextra
```

### 3. Perintah build / flash / monitor / test

```bash
PIO=~/.platformio/penv/bin/pio

$PIO run -e esp32dev                  # build satu board
$PIO run -e esp32dev -e esp8266 -e uno -e nano -e bluepill_f103c8  # build semua
$PIO run -e esp32dev -t upload        # build + flash
$PIO run -e esp32dev -t clean         # bersih-bersih
$PIO device monitor -b 115200         # serial monitor
$PIO run -e esp32dev -t size           # tabel flash/SRAM
$PIO test -e native                   # unit test host
$PIO run                              # build default_envs
```

Tips flash:
- `ls /dev/ttyUSB* /dev/ttyACM*`, set `upload_port = /dev/ttyACM0` bila tidak kedetek.
- ESP32 USB-CDC: `Serial.begin(115200)` harus paling awal, dan bila perlu tambah `monitor_dtr = 0` + `monitor_rts = 0`.

### 4. Pola kode wajib

Non-blocking blink (dipakai untuk semua timing):

```cpp
bool update(uint32_t now) {
  if ((uint32_t)(now - last_) < period_) return false;
  last_ = now;
  // ... ganti state, tulis output
  return true;
}
```

- `loop()`: baca input → `update()` tiap objek → render output.
- Driver bebas `Serial`/`millis`/singleton agar bisa di-test di host (oper `now` sebagai parameter).
- Logging hemat RAM via `Logger.h` (`snprintf` ke `char buf[64]`, `F()`/`PSTR` di AVR, `dtostrf` untuk float di AVR).
- Contoh lengkap scheduler, debounce, state-machine, ring-buffer, PID ada di `arduino-cpp/references/patterns.md`.

### 5. Checklist "done"

- `pio run` sukses semua env, tanpa warning baru.
- `git diff | rg '\bdelay\(|String\b|digitalWrite\(\s*[0-9]'` bersih.
- Waktu rollover-safe, loop serial bounded.
- Test host mencakup overflow `millis()`, overflow buffer, edge debounce.
- Angka flash/SRAM dilaporkan.
- Kredensial tidak di source (pakai WiFiManager atau flag `-D`), ada kebijakan reconnect non-blocking.

## Panduan Referensi

Baca file yang relevan, jangan menebak:

| File | Kapan dibaca |
| ---- | ------------ |
| `references/porting.md` | beda pin, matriks API arch, bahaya 3.3V vs 5V, tambah board baru |
| `references/peripherals.md` | GPIO, PWM/LEDC, ADC, I2C, SPI, UART, EEPROM, tabel lib sensor/display terverifikasi |
| `references/connectivity.md` | WiFi reconnect, WiFiManager, MQTT, HTTP+JSON, OTA, deep sleep, NTP, budget daya |
| `references/protocols.md` | UDP, WebSocket, Ethernet W5500, HTTPS/TLS, TCP raw dengan deadline |
| `references/concurrency.md` | ISR, volatile/atomics, FreeRTOS ESP32, mutex/queue, triase reset-cause |
| `references/patterns.md` | scheduler non-blocking, debounce, state machine, ring buffer, PID |
| `references/build-and-flash.md` | referensi `platformio.ini` penuh, partisi, OTA, USB-CDC, native test |
| `references/troubleshooting.md` | brownout, upload gagal, serial sampah, linker overflow, timing flaky |

## Catatan Python venv

Skill pendamping `python-virtual-environment` mewajibkan semua `python`/`pip`/`pytest` lewat venv sementara:

```bash
VENV=/tmp/opencode/$(basename "$(pwd)")-venv
python3 -m venv "$VENV"
"$VENV/bin/python" -m pip install -r requirements.txt
"$VENV/bin/python" script.py
```
