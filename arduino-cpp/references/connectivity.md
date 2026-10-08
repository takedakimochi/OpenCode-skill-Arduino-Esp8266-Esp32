# Connectivity: WiFi, MQTT, HTTP, OTA, and sleep (ESP32 + ESP8266)

Rule zero: the radio is the hungriest, flakiest peripheral on the board. Every
snippet below is non-blocking, bounded in time, and reconnects by itself —
`loop()` must keep running even when the network is gone. AVR/Blue Pill have no
WiFi; WiFi/MQTT/OTA/deep-sleep code always lives behind
`#if defined(ARDUINO_ARCH_ESP32) || defined(ARDUINO_ARCH_ESP8266)` so the same
project still builds for `uno`. ESP8266 API differences from ESP32 are marked
`// 8266:` inline — they are small but compile-breaking if ignored
(`setHostname` vs `hostname()`, `HTTPClient.h` vs `ESP8266HTTPClient.h`,
`esp_deep_sleep_start` vs `ESP.deepSleep`).

## 1. WiFi: connect fast, reconnect forever

```cpp
#if defined(ARDUINO_ARCH_ESP32) || defined(ARDUINO_ARCH_ESP8266)
#if defined(ARDUINO_ARCH_ESP32)
#include <WiFi.h>
#else
#include <ESP8266WiFi.h>
#endif

class WifiLink {
 public:
  void begin(const char* ssid, const char* pass, const char* hostname) {
    WiFi.mode(WIFI_STA);
#if defined(ARDUINO_ARCH_ESP32)
    WiFi.setHostname(hostname);
#else
    WiFi.hostname(hostname);          // 8266: hostname(), not setHostname()
    WiFi.setAutoReconnect(true);
    WiFi.persistent(false);           // 8266: don't rewrite flash on connect
#endif
    WiFi.begin(ssid, pass);
  }

  // Call from loop(). Returns true when associated AND holding an IP.
  // Never blocks: a dead AP just yields false until the retry timer fires.
  bool update(uint32_t now) {
    if (WiFi.status() == WL_CONNECTED) return true;
    if (static_cast<uint32_t>(now - lastAttempt_) >= kRetryMs) {
      lastAttempt_ = now;
      WiFi.disconnect();
      WiFi.reconnect();   // reuses the stored credentials
    }
    return false;
  }

  int32_t rssi() const { return WiFi.RSSI(); }   // < -80 dBm == marginal link

 private:
  static constexpr uint32_t kRetryMs = 10000;
  uint32_t lastAttempt_ = 0;
};
#endif
```

- Never `while (WiFi.status() != WL_CONNECTED) {}` — an unreachable AP then
  wedges `setup()` forever and starves the task watchdog.
- Never commit passwords: pass them via `build_src_flags = -DWIFI_SSID="..."`
  `-DWIFI_PASS="..."`, or provision with WiFiManager (§2).
- `WiFi.RSSI()` below ~-80 dBm means the link, not your code, is the bug.
- ADC2 pins are unusable while WiFi is up — use ADC1 (see `peripherals.md`).

## 2. Provisioning without hardcoded credentials

`tzapu/WiFiManager` (`WiFiManager.h`) serves a captive portal from the ESP32 or
ESP8266 so credentials are entered on a phone, not baked into firmware:

```cpp
#if (defined(ARDUINO_ARCH_ESP32) || defined(ARDUINO_ARCH_ESP8266)) && \
    __has_include(<WiFiManager.h>)
#include <WiFiManager.h>

bool provisionWifi(uint32_t timeoutSec = 120) {
  WiFiManager wm;
  wm.setConnectTimeout(10);              // seconds per connection attempt
  wm.setConfigPortalTimeout(timeoutSec); // bounded: portal gives up, code runs on
  return wm.autoConnect("setup-ap");     // false == timed out, stay offline
}
#endif
```

`autoConnect` blocks only up to the timeout. A `false` return is a normal
outcome (no known AP around) — run the offline code path instead of rebooting.

## 3. MQTT that survives the real world

`knolleary/PubSubClient` (`PubSubClient.h`). Three disciplines: `loop()` runs
every `loop()` iteration (it pumps keep-alive; default keep-alive is 15 s),
reconnects use a backoff timer, and the buffer is sized for your payload
(default is only 256 bytes):

```cpp
#if (defined(ARDUINO_ARCH_ESP32) || defined(ARDUINO_ARCH_ESP8266)) && \
    __has_include(<PubSubClient.h>)
#include <PubSubClient.h>

extern WiFiClient net;   // one client shared by MQTT (and HTTP if needed)
PubSubClient mqtt(net);

void mqttMaintain(uint32_t now, const char* host, uint16_t port,
                  const char* clientId, const char* subTopic) {
  if (mqtt.connected()) {
    mqtt.loop();   // every iteration — never gate this behind a timer
    return;
  }
  static uint32_t lastTry = 0;
  if (static_cast<uint32_t>(now - lastTry) < 5000) return;   // backoff
  lastTry = now;
  mqtt.setServer(host, port);
  mqtt.setBufferSize(1024);   // match your largest publish/subscribe payload
  if (mqtt.connect(clientId)) mqtt.subscribe(subTopic);
}
#endif
```

- Publish small, publish rarely; QoS 0 is the default and is enough for sensor
  telemetry. Check the `publish()` return — a full buffer fails silently.
- The callback runs inside `mqtt.loop()`: copy the payload and return. No
  `delay`, no publish-from-callback storms.

## 4. HTTP + JSON without leaking sockets

```cpp
#if (defined(ARDUINO_ARCH_ESP32) || defined(ARDUINO_ARCH_ESP8266)) && \
    __has_include(<ArduinoJson.h>)
#if defined(ARDUINO_ARCH_ESP32)
#include <HTTPClient.h>
#else
#include <ESP8266HTTPClient.h>   // 8266: different header, same class name
#endif
#include <ArduinoJson.h>

bool fetchConfig(const char* url, JsonDocument& doc) {
  HTTPClient http;
  http.setTimeout(5000);
#if defined(ARDUINO_ARCH_ESP8266)
  WiFiClient net;          // 8266: begin(url) without a client is a
  http.begin(net, url);    // compile error — the client is mandatory
#else
  http.begin(url);
#endif
  const int code = http.GET();
  const bool ok = (code == HTTP_CODE_OK) &&
                  (deserializeJson(doc, http.getStream()) == DeserializationError::Ok);
  http.end();   // always — or sockets leak until the next reset
  return ok;
}
#endif
```

- `bblanchon/ArduinoJson` (`ArduinoJson.h`): size the document with the
  ArduinoJson Assistant; on AVR keep documents to a few hundred bytes and never
  pretty-print to `Serial` in production.
- Prefer `http.getStream()` into the document over `getString()` + parse — one
  copy instead of two.

## 5. OTA updates

Flash the first firmware over USB, then update over the air. Two parts, both
required:

1. Firmware side — `ArduinoOTA.h` ships with both the ESP32 and the ESP8266
   core, no library to install:

```cpp
#if defined(ARDUINO_ARCH_ESP32) || defined(ARDUINO_ARCH_ESP8266)
#include <ArduinoOTA.h>

void beginOta(const char* hostname) {
  ArduinoOTA.setHostname(hostname);
  ArduinoOTA.begin();
}
// ArduinoOTA.handle() on every loop() — costs ~nothing when idle.
#endif
```

2. Upload side — in `platformio.ini` (see `build-and-flash.md`):

```ini
upload_protocol = espota
upload_port = 192.168.1.50
```

OTA needs a partition table with two app slots (`app0`/`app1`) once the
firmware grows — the default table already has them; see `build-and-flash.md`.

## 6. Deep sleep for battery devices

```cpp
#if defined(ARDUINO_ARCH_ESP32)
#include <esp_sleep.h>

// Survives deep sleep (not power loss). Counters, calibration, last readings.
RTC_DATA_ATTR uint32_t bootCount = 0;

void sleepSeconds(uint64_t s) {
  esp_sleep_enable_timer_wakeup(s * 1000000ULL);
  esp_deep_sleep_start();   // never returns; setup() reruns on wake
}
#elif defined(ARDUINO_ARCH_ESP8266)
// 8266: timer wake needs GPIO16 wired to RST, or it sleeps exactly once.
// RF_DEFAULT recalibrates radio on wake (slower, robust); RF_DISABLED skips it.
#define SLEEP_RF_MODE RF_DEFAULT
void sleepSeconds(uint64_t s) {
  ESP.deepSleep(s * 1000000ULL, SLEEP_RF_MODE);   // never returns
}
#endif
```

- Everything resets except `RTC_DATA_ATTR` memory. Re-`begin()` every
  peripheral in `setup()` — sleep is a reboot with amnesia, not a pause.
- GPIO wake uses `esp_sleep_enable_ext1_wakeup()` with a **uint64** bitmask on
  chips with >32 GPIOs (see `porting.md` §4).
- Deep sleep current is ~10 µA for the chip, but the board's regulator and USB
  chip usually dominate — measure the whole board, and prefer a low-quiescent
  regulator over trusting the chip number.

## 7. Timestamps via NTP

```cpp
#if defined(ARDUINO_ARCH_ESP32) || defined(ARDUINO_ARCH_ESP8266)
#include <time.h>

void beginTime() {
  configTime(0, 0, "pool.ntp.org", "time.google.com");   // UTC; add offsets as needed
}

bool nowTm(struct tm& out) {
#if defined(ARDUINO_ARCH_ESP32)
  return getLocalTime(&out);   // false until the first successful sync
#else
  // 8266: no getLocalTime() — read the clock and reject pre-sync garbage.
  const time_t t = time(nullptr);
  if (t < 1000000000L) return false;   // before Sep 2001 == not synced
  localtime_r(&t, &out);
  return true;
#endif
}
#endif
```

Time is garbage before the first sync — check the return instead of printing
`1970-01-01` to the log and confusing everyone.

## 8. Power budgeting for radio boards

| Load | Draw | Rule |
| ---- | ---- | ---- |
| ESP32 WiFi TX burst | 170–250 mA peaks | ≥470 µF bulk electrolytic + 100 nF ceramic at the module; a thin USB cable alone browns out (see `troubleshooting.md`) |
| USB 2.0 port | 500 mA max | one ESP32 + sensors is fine; ESP32 + motors/servos is not |
| Servo / motor / relay / solenoid | 100 mA – several A, plus back-EMF | separate rail, common ground, flyback diode on every inductive load, never switch from a GPIO directly |
| OLED / sensors | a few mA – 50 mA | board rail is fine; share ground, mind 3.3 V vs 5 V logic (see `porting.md` §3) |

When the symptom is "works on USB, resets on battery" or "resets when WiFi
transmits", it is the supply until proven otherwise — print the reset cause
(`concurrency.md` §5) and scope the 3.3 V rail during a TX burst.
