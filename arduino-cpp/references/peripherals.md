# Peripherals: the portable subset, and where it stops

Everything here is core Arduino API unless flagged. Cores differ in *details*;
the rules below are the portable form.

## GPIO

```cpp
constexpr uint8_t kButton = 4;
pinMode(kButton, INPUT_PULLUP);
const bool pressed = digitalRead(kButton) == LOW;   // active LOW with pull-up
```

- `INPUT_PULLUP` is the portable default for buttons. It is only ~45 kΩ on some
  cores and **absent** on ESP32 GPIO 34-39, so add an external pull-up there.
- `pinMode(pin, OUTPUT); digitalWrite(pin, LOW);` before configuring a pin as an
  input — the output latch defaults to LOW on some cores, which pops a spike on
  the pin while it is still an output.
- `OUTPUT_OPEN_DRAIN` does not exist in the Arduino API. Wire an external
  MOSFET for open-drain/bus or multi-drop buses.
- Never drive more than ~20 mA per ESP32/STM32 pin, or ~40 mA total on ESP32
  (3.3 V rail). LEDs and relays go through a driver.
- `digitalWrite` in a tight loop is wasteful; batch changes and use a port
  register only if you accept the arch guard cost.

## PWM / LEDC

Portable baseline — plain 8-bit PWM, roughly the same behaviour everywhere:

```cpp
analogWrite(pin, level);        // 0..255
```

ESP32 needs explicit frequency/resolution or you get a 1 kHz default that buzzes
audibly. Arduino-ESP32 **2.x** (what is installed here):

```cpp
#if defined(ARDUINO_ARCH_ESP32)
// core 2.x
ledcSetup(0, 5000, 8);         // channel 0, 5 kHz, 8-bit
ledcAttachPin(pin, 0);
ledcWrite(0, duty);            // 0..255
#endif
```

Arduino-ESP32 **3.x** replaced that with `ledcAttach(pin, freq, res)` returning
`bool`, plus `ledcWrite(pin, duty)`. Check the core version before writing the
call — `ARDUINO_ESP32_MAJOR` does not exist, the real macros are in
`<esp_arduino_version.h>`:

```cpp
#include <esp_arduino_version.h>
static_assert(ESP_ARDUINO_VERSION_MAJOR < 3, "update LEDC code for Arduino-ESP32 3.x");
```

(There is no `ESP.getCoreVersion()` in this core — for the Arduino core version
use the `ESP_ARDUINO_VERSION` macros, or `ESP.getSdkVersion()` for the underlying
ESP-IDF version.) If you must support both, put the PWM setup in one
`board_pwm.cpp` with two arch branches rather than sprinkling `ledcSetup`
through drivers.

Notes:

- Resolution above 8 bits needs `ledcSetup`/channel count care — ESP32 has 16
  LEDC channels shared by all timers, and 20-bit modes consume several.
- **Never** change a pin's LEDC/PWM timer away from `tone()`'s pin (or vice
  versa) on the same channel; they fight and produce a constant DC level.
- `tone()` is AVR-focused and 8-bit-fixed; it steals Timer2 (which also owns
  `pin 3`/`11` PWM) on Uno. Use it sparingly and never while using `millis`-grade
  accuracy timing.
- **No PWM in ISRs** on AVR (it reconfigures timers). On ESP32 an LEDC write
  from an ISR-attached flag processed in `loop()` is fine.

## ADC

```cpp
const int raw = analogRead(kSensorAdc);            // common to all cores
```

- Resolution differs: AVR 10-bit, ESP8266 10-bit, ESP32/STM32 12-bit. Never
  hard-code a scale. Use `analogReadResolution(bits)` on 12-bit cores, or
  store the raw count and convert per board.
- ESP8266 has exactly **one** ADC pin (`A0`), 10-bit, 0–1.0 V at the chip.
  NodeMCU/Wemos boards add a divider for 0–3.3 V — feeding 5 V still kills it.
  There is no second channel to switch to; multiplex externally if needed.
- For calibrated voltage on ESP32: `analogReadMilliVolts(pin)` (≈ 11 dB
  attenuation, needs `analogSetAttenuation(ADC_11db)`). Its raw equivalent is
  **not** a linear 0-3.3 V map — calibrate with known references if accuracy
  matters.
- AVR: ADC 0-15, and reads are slow (~104 µs). Sample at most ~9 kHz, and do not
  read more than one channel inside a tight timing loop without accounting for it.
- ESP32: use **ADC1** pins when WiFi/BLE is on. Halve the sample rate when the
  WiFi driver is active.
- Always take N samples and discard min/max, or a moving average, before
  thresholding. Single reads are noise.

```cpp
uint16_t readAdcAveraged(uint8_t pin, uint8_t n = 8) {
  uint16_t lo = UINT16_MAX, hi = 0, sum = 0;
  for (uint8_t i = 0; i < n; ++i) {
    const uint16_t v = analogRead(pin);
    if (v < lo) lo = v;
    if (v > hi) hi = v;
    sum += v;
  }
  return static_cast<uint16_t>((sum - lo - hi) / (n >= 2 ? n - 2 : 1));
}
```

## I2C (`Wire`)

```cpp
#include <Wire.h>

// The two-argument Wire.begin(sda, scl) exists on ESP32 AND ESP8266 — it does
// NOT compile on AVR, and STM32 configures pins differently. "Two Wire" (TWI)
// is the same bus as I2C: same pins, same library, same rules.
// Init per core, pins from Board.h:
void beginI2c(uint8_t sda, uint8_t scl, uint32_t hz = 100000) {
#if defined(ARDUINO_ARCH_ESP32) || defined(ARDUINO_ARCH_ESP8266)
  Wire.begin(sda, scl);
#elif defined(ARDUINO_ARCH_STM32)
  Wire.setSDA(sda);
  Wire.setSCL(scl);
  Wire.begin();
#else
  (void)sda; (void)scl;
  Wire.begin();   // AVR: fixed hardware pins (Uno/Nano A4/A5, Mega 20/21)
#endif
  Wire.setClock(hz);   // 100 kHz first, 400 kHz only once the bus is proven
}

// Returns true if a device ACKs at addr. Scan 0x08..0x77 at startup to prove
// the wiring before writing a single driver line.
bool i2cProbe(uint8_t addr) {
  Wire.beginTransmission(addr);
  return Wire.endTransmission() == 0;   // 0 == ACK; nonzero == NACK/error
}
```

- 100 kHz first, then 400 kHz. Long wires, pull-ups under ~2.2 kΩ, or a
  starved 3.3 V rail all produce "works on my board" NACKs.
- Bus capacitance limit: 400 pF, which caps a bus at roughly 1 m at 400 kHz.
- Every I2C transaction should be timeout-bounded. A hung bus on a 2-wire
  topology can freeze the loop indefinitely.
- Pull-ups: 4.7 kΩ typical. Verify the module does not already have them (most
  breakout boards do, often 10 kΩ, which is marginal at 400 kHz).
- `Wire` on ESP32 uses the Wire0/I2C0 peripheral; Arduino-ESP32 2.x exposes one
  or two buses depending on core config. Treat "two buses" as non-portable.

## SPI

```cpp
#include <SPI.h>

// Pin selection is per core — the 4-argument begin() is ESP32-only.
void beginSpi(uint8_t sck, uint8_t miso, uint8_t mosi, uint8_t cs) {
#if defined(ARDUINO_ARCH_ESP32)
  SPI.begin(sck, miso, mosi, cs);
#elif defined(ARDUINO_ARCH_ESP8266)
  (void)sck; (void)miso; (void)mosi; (void)cs;
  SPI.begin();   // ESP8266: fixed HSPI pins (SCK 14, MISO 12, MOSI 13)
#elif defined(ARDUINO_ARCH_STM32)
  SPI.setSCLK(sck);
  SPI.setMISO(miso);
  SPI.setMOSI(mosi);
  SPI.setSSEL(cs);
  SPI.begin();
#else
  (void)sck; (void)miso; (void)mosi; (void)cs;
  SPI.begin();   // AVR: fixed hardware pins (Uno/Nano 11/12/13, SS on 10)
#endif
}

SPI.beginTransaction(SPISettings(4000000, MSBFIRST, SPI_MODE0));
const uint8_t v = SPI.transfer(0x9F);
SPI.endTransaction();
```

- **Always** wrap transfers in `beginTransaction`/`endTransaction`: it sets the
  mode/clock deterministically and is what makes the bus correct when another
  library (SD, Ethernet, TFT) has reconfigured it.
- `SPISettings` must match the device datasheet. Wrong mode looks like "returns
  0xFF forever".
- Use the explicit `SPI.begin(sclk, miso, mosi, ss)` overload so pins live in
  `Board.h`.
- SPI is a separate bus from I2C; the ESP32 shares one peripheral set across
  several pins, so pins are interchangeable but not simultaneous.

## UART

```cpp
Serial.begin(115200);   // before ANY print, always
```

One hardware UART (Uno/Nano, ESP8266 UART0) or several (ESP32: UART0/1/2,
Mega: 4x). A second serial device means a second port — plan it in `Board.h`:

```cpp
#if defined(ARDUINO_ARCH_ESP32)
// Any GPIO can be a UART: RX, TX order. UART0 stays on USB for logs.
HardwareSerial extSerial(1);
extSerial.begin(9600, SERIAL_8N1, pins::kExtRx, pins::kExtTx);
#elif defined(ARDUINO_ARCH_ESP8266)
// UART0 only. swap() moves it to GPIO13(RX)/GPIO15(TX) to free the USB pins;
// Serial1 is TX-only on GPIO2 (logs without an RX line).
Serial.swap();
#elif defined(ARDUINO_ARCH_AVR)
// Uno/Nano: SoftwareSerial for a second port — 9600 baud max for sanity,
// and it disables interrupts while sending (jitters millis/PWM/Servo).
#include <SoftwareSerial.h>
SoftwareSerial extSerial(pins::kExtRx, pins::kExtTx);
extSerial.begin(9600);
#endif
```

RS-485 half-duplex (MAX485/MAX3485) on any UART — drive DE/RE, flush, release:

```cpp
digitalWrite(pins::kRs485De, HIGH);   // transmit mode
extSerial.write(frame, len);
extSerial.flush();                     // wait until the last stop bit leaves
digitalWrite(pins::kRs485De, LOW);     // back to receive mode
```

- ESP32 on USB-CDC: `Serial.begin(115200)` triggers the enumeration handshake and
  can take ~2 s. Prints before it are lost. `while (!Serial) {}` blocks — do it
  only in `setup()` behind a timeout, or use `millis()`-based logic.
- 115200 8N1 is the safe default everywhere. Uno's USB-UART chip cannot do
  reliable 921600; 57600/115200 only.
- Serial RX is a 64-byte buffer on AVR (256 on ESP32/ESP8266). A
  `while (Serial.available())` drain loop is mandatory; a `for` loop of fixed
  length drops bytes under load.
- Never `Serial.print` in an ISR, and never `print` a float on AVR — use
  `dtostrf(value, 6, 2, buf)` then `Serial.print(buf)`. ESP32/ESP8266 print
  floats natively.

## EEPROM / NVS

```cpp
#if defined(ARDUINO_ARCH_ESP32)
// Prefer Preferences (NVS flash store). There is no commit() step.
#include <Preferences.h>
Preferences prefs;
prefs.begin("app", false);              // false == read/write
prefs.putShort("offset", 0);
const int16_t offset = prefs.getShort("offset", 0);
prefs.end();
#elif defined(ARDUINO_ARCH_ESP8266)
#include <EEPROM.h>
// ESP8266: flash emulation — begin(size) reserves RAM, commit() writes to flash.
// Forgetting commit() is the classic "settings never stick" bug.
int16_t offset = 0;
EEPROM.begin(512);
EEPROM.get(0, offset);
EEPROM.put(0, offset);
EEPROM.commit();
#elif defined(ARDUINO_ARCH_AVR) || defined(ARDUINO_ARCH_STM32)
#include <EEPROM.h>
// AVR/STM32 EEPROM API: no begin(), no commit() — writes apply immediately.
// update() writes only when the byte changed (saves the ~100k write cycles).
EEPROM.update(0, 1);                    // byte 0: schema version
int16_t offset = 0;
EEPROM.get(1, offset);                  // multi-byte get/put, AVR + STM32
#endif
```

- EEPROM endurance is ~100 000 write cycles per cell (AVR real EEPROM, ESP8266
  flash sector). Use `update()`/`put()`, never `write()`, in a periodic save
  path, and checksum the record before trusting it.
- On ESP32 core 2.x (installed here) there is no usable `EEPROM` emulation —
  use `Preferences`. Core 3.x added one, but `Preferences` works on both.
- Keep a schema/version byte and a CRC so a firmware update cannot brick a device
  that reads garbage settings.

## Other serial buses

- **One-wire** (`OneWire`/`DallasTemperature`): only 4.7-10 kΩ pull-up, 400 µs
  slot, parasitally powered sensors need a strong driver and long settle
  delays. Not reliable without careful bus design.
- **CAN**: there is **no portable CAN API** and, in this installed core
  (Arduino-ESP32 2.0.17), no `TWAI` class at all — core 3.x added `TWAI.h`, AVR
  and STM32 have their own `CAN`/`Canvn` classes. Isolate it in one driver
  behind an arch guard, and verify with the controller's own error counters
  (`TWAI.getStatusInfo()` on 3.x, `CAN.getErrorCounts()` elsewhere) rather than
  by "it seemed to work". A wrong bit-timing setup makes the error counter
  explode long before frames stop.
- **SoftwareSerial** on AVR: UART eats ~1.2 KB of flash and conflicts with PWM
  timers; only use it for low baud on spare timers.

## Common sensors and displays

Library IDs and versions below were verified against the PlatformIO registry
(`pio pkg search "header:X.h"`), then installed and compiled for `uno`,
`esp32dev`, and `bluepill_f103c8` on 2026-10-08. Pin the version in `lib_deps`;
never paste an ID from memory without checking it first.

| Part | `lib_deps` ID (verified) | Header | Gotcha |
| ---- | ------------------------ | ------ | ------ |
| DHT11/DHT22 temp+humidity | `adafruit/DHT sensor library@^1.4.7` (auto-pulls `Adafruit Unified Sensor`) | `DHT.h` | DHT22 needs ≥2 s between reads; both return `NaN` on failure — check with `isnan()` |
| DS18B20 temp (one-wire) | `paulstoffregen/OneWire@^2.3.8` + `milesburton/DallasTemperature@^4.0.6` | `OneWire.h`, `DallasTemperature.h` | 4.7 kΩ pull-up mandatory; parasite power is unreliable — power VCC properly |
| BME280 temp/hum/pressure | `adafruit/Adafruit BME280 Library@^2.3.0` | `Adafruit_BME280.h` | I2C addr is 0x76 or 0x77 by module — `i2cProbe` both; needs the Unified Sensor lib |
| SSD1306/SH1106 OLED | `olikraus/U8g2@^2.36.18` | `U8g2lib.h` | full-buffer (`_F_`) constructors eat 1 KB — on AVR use page mode (`_1_`/`_2_`, see below) |
| I2C LCD 1602/2004 (PCF8574) | `fmalpartida/LiquidCrystal@^1.5.0` | `LiquidCrystal_I2C.h` | contrast pot + I2C addr (usually 0x27/0x3F) — probe before assuming |
| WS2812/NeoPixel strip | `adafruit/Adafruit NeoPixel@^1.15.5` (simple) or `fastled/FastLED@^3.10.6` (effects) | `Adafruit_NeoPixel.h` / `FastLED.h` | level-shift data to 5 V, big bulk cap on the strip supply, never power a strip from the board's 5 V pin |
| JSON encode/decode | `bblanchon/ArduinoJson@^7.4.3` | `ArduinoJson.h` | size the document with the ArduinoJson Assistant; on AVR keep documents to a few hundred bytes |
| WiFi captive portal | `tzapu/WiFiManager@^2.0.17` | `WiFiManager.h` | ESP32/ESP8266 only; `autoConnect` blocks up to the portal timeout (see `connectivity.md`) |
| MQTT messaging | `knolleary/PubSubClient@^2.8.0` | `PubSubClient.h` | default 256-byte buffer — `setBufferSize()` to fit; `loop()` every iteration (see `connectivity.md`) |

Non-blocking DS18B20 (12-bit conversion takes ~750 ms — never `delay` for it):

```cpp
#include <OneWire.h>
#include <DallasTemperature.h>

OneWire oneWire(pins::kOneWire);
DallasTemperature sensors(&oneWire);
static uint32_t lastRequest_ = 0;
static float lastTempC_ = 0;
static bool haveTemp_ = false;

void tempBegin(uint32_t now) {
  sensors.begin();
  sensors.setWaitForConversion(false);   // requestTemperatures() returns at once
  sensors.requestTemperatures();
  lastRequest_ = now;
}

// Call from loop(). Returns true once per fresh reading (~800 ms cadence).
bool tempUpdate(uint32_t now) {
  if (static_cast<uint32_t>(now - lastRequest_) < 800) return false;
  const float t = sensors.getTempCByIndex(0);
  haveTemp_ = (t != DEVICE_DISCONNECTED_C);
  if (haveTemp_) lastTempC_ = t;
  sensors.requestTemperatures();         // start the next conversion
  lastRequest_ = now;
  return haveTemp_;
}
```

DHT22 minimum pattern (≥2000 ms between reads, NaN check):

```cpp
#include <DHT.h>

DHT dht(pins::kDht, DHT22);

void climateBegin() { dht.begin(); }

bool climateRead(float& tempC, float& hum) {   // call at most every 2 s
  hum = dht.readHumidity();
  tempC = dht.readTemperature();
  return !isnan(hum) && !isnan(tempC);
}
```

SSD1306 OLED via U8g2 — page mode on AVR (a `_F_` full buffer is 1024 bytes,
half of an Uno's SRAM), full buffer elsewhere:

```cpp
#include <U8g2lib.h>

#if defined(ARDUINO_ARCH_AVR)
U8G2_SSD1306_128X64_NONAME_1_HW_I2C u8g2(U8G2_R0, U8X8_PIN_NONE);
#else
U8G2_SSD1306_128X64_NONAME_F_HW_I2C u8g2(U8G2_R0, U8X8_PIN_NONE);
#endif

void displayBegin() { u8g2.begin(); }

void displayText(const char* line) {
#if defined(ARDUINO_ARCH_AVR)
  u8g2.firstPage();
  do {
    u8g2.setFont(u8g2_font_6x10_tf);
    u8g2.drawStr(0, 10, line);
  } while (u8g2.nextPage());
#else
  u8g2.clearBuffer();
  u8g2.setFont(u8g2_font_6x10_tf);
  u8g2.drawStr(0, 10, line);
  u8g2.sendBuffer();
#endif
}
```

## Analog output and DAC

There is no portable `analogWrite`-to-voltage. `analogWrite` is PWM; low-pass
filter it if you need DC, and note that the ESP32's real DAC exists only on
GPIO 25/26 (`dacWrite(pin, 0..255)`, 8-bit). Use an external DAC (MCP4725 over
I2C) for anything cross-board.
