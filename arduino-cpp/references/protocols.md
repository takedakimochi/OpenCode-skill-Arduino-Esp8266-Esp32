# Protocols: UART framing, UDP, WebSocket, Ethernet, HTTPS, raw TCP

UART/I2C/SPI pin-level setup lives in `peripherals.md`; WiFi association,
reconnect, and OTA live in `connectivity.md`. This file is the **socket layer**:
what goes over the wire once the link is up.

| Transport | ESP32 | ESP8266 | AVR (Uno/Nano) | Needs |
| --------- | ----- | ------- | -------------- | ----- |
| UDP | yes | yes | no (no net) | nothing — `WiFiUdp.h` ships with the core |
| WebSocket client | yes | yes (tight) | no | `links2004/WebSockets@^2.7.3` |
| Ethernet TCP/IP | yes | yes | yes | `arduino-libraries/Ethernet@^2.0.2` + W5500 module |
| HTTP | yes | yes (other header) | via Ethernet | `connectivity.md` §4 |
| HTTPS/TLS | yes (`WiFiClientSecure`) | yes, small (BearSSL) | **no** — 2 KB RAM cannot do TLS | NTP-synced clock for cert checks |
| Raw TCP | yes | yes | via Ethernet | `WiFiClient` / `EthernetClient` |

## 1. UDP (telemetry, discovery, NTP)

`WiFiUdp.h` ships with both cores — no library to install. UDP has no
connection: every datagram stands alone, may be lost, and must fit one packet
(keep payloads ≤ ~1400 bytes to avoid IP fragmentation).

```cpp
#if defined(ARDUINO_ARCH_ESP32) || defined(ARDUINO_ARCH_ESP8266)
#include <WiFiUdp.h>

WiFiUDP udp;
static constexpr uint16_t kLocalPort = 4444;

void udpBegin() { udp.begin(kLocalPort); }

bool udpSend(const char* host, uint16_t port, const uint8_t* buf, size_t len) {
  if (len > 1400) return false;
  return udp.beginPacket(host, port) == 1 &&
         udp.write(buf, len) == len &&
         udp.endPacket() == 1;
}

// Call from loop(). Returns bytes read, 0 when nothing arrived.
int udpReceive(uint8_t* buf, size_t cap) {
  const int n = udp.parsePacket();   // non-blocking; 0 == nothing
  if (n <= 0) return 0;
  const int m = udp.read(buf, cap - 1);
  return m < 0 ? 0 : m;
}
#endif
```

`udp.remoteIP()` / `udp.remotePort()` after a successful `parsePacket()` tell
you who sent it — answer discoveries there, never to a hardcoded IP.

## 2. WebSocket client

`links2004/WebSockets` (`WebSocketsClient.h`), verified `2.7.3`. One persistent
TCP connection, server can push, no polling. `loop()` must run every
`loop()` iteration, same discipline as MQTT.

```cpp
#if (defined(ARDUINO_ARCH_ESP32) || defined(ARDUINO_ARCH_ESP8266)) && \
    __has_include(<WebSocketsClient.h>)
#include <WebSocketsClient.h>

WebSocketsClient ws;

static char lastWsMsg[128]{};
static bool haveWsMsg_ = false;

void onWsEvent(WStype_t type, uint8_t* payload, size_t length) {
  // Runs inside ws.loop(): copy and return fast, never block here.
  if (type == WStype_TEXT && length < sizeof lastWsMsg) {
    memcpy(lastWsMsg, payload, length);
    lastWsMsg[length] = '\0';
    haveWsMsg_ = true;   // consumed by the app in loop()
  }
}

void wsBegin(const char* host, uint16_t port, const char* path) {
  ws.begin(host, port, path);          // ws:// — see WSS note below
  ws.onEvent(onWsEvent);
  ws.setReconnectInterval(5000);      // built-in backoff, keep it
}

void wsMaintain() { ws.loop(); }      // every loop() iteration, no gating
#endif
```

- ESP8266 + WSS (`wss://`) works but BearSSL eats a large slice of the 80 KB
  heap — plain `ws://` inside a trusted LAN, WSS only facing the internet, and
  test the heap with `ESP.getFreeHeap()` under load.
- Text frames are **not** NUL-terminated — always use the `length`.
- Reconnect storms on flaky WiFi: the 5 s interval above plus the `WifiLink`
  backoff in `connectivity.md` compose fine; do not add a third retry layer.

## 3. Ethernet TCP/IP (W5500)

For boards without WiFi (Uno/Nano) or where WiFi is forbidden: a W5500 module
over SPI. The chip runs the TCP/IP stack in hardware, so even a 2 KB AVR can
do TCP — prefer W5500 over ENC28J60 (whose software stack needs far more RAM).

```cpp
#if __has_include(<Ethernet.h>)
#include <SPI.h>
#include <Ethernet.h>

static const byte kMac[6] = {0xDE, 0xAD, 0xBE, 0xEF, 0xFE, 0xED};

bool ethBegin(uint32_t timeoutMs = 10000) {
#if defined(ARDUINO_ARCH_ESP32)
  Ethernet.init(pins::kEthCs);   // custom CS; AVR uses the default SS pin
#endif
  const uint32_t t0 = millis();
  while (Ethernet.begin((uint8_t*)kMac) == 0) {   // DHCP; 0 == failed
    if (static_cast<uint32_t>(millis() - t0) >= timeoutMs) return false;
    delay(1000);   // setup-time only; loop() stays non-blocking afterwards
  }
  return true;
}

bool ethGet(const char* host, const char* path) {
  EthernetClient client;
  if (!client.connect(host, 80)) return false;
  client.print("GET ");
  client.print(path);
  client.println(" HTTP/1.0");
  client.print("Host: ");
  client.println(host);
  client.println();
  // ...bounded read exactly like §5 below, then client.stop()
  client.stop();
  return true;
}
#endif
```

- `Ethernet.maintain()` in `loop()` renews the DHCP lease — without it the
  address silently expires after hours.
- The W5500 has 8 sockets; `stop()` every client you open or the 9th
  `connect()` fails forever until reset.
- Default CS: AVR uses pin 10 (SS). On ESP32/ESP8266 pass your CS to
  `Ethernet.init()` **before** `begin()`.

## 4. HTTPS / TLS

TLS needs three things AVR cannot offer (RAM for the session, CPU for the
handshake, a clock for cert expiry) — **HTTPS is ESP-only.** Even on ESP, sync
the clock first (`beginTime()` in `connectivity.md`); an unsynced clock fails
every certificate check with a confusing error.

```cpp
// ESP-only: there is no WiFiClientSecure.h on AVR. Keep all TLS code behind
// this guard so shared projects still build for Uno/Nano.
#if defined(ARDUINO_ARCH_ESP32)
#include <WiFiClientSecure.h>
WiFiClientSecure tls;
tls.setInsecure();   // LAB ONLY — pins nothing. Production: tls.setCACert(ca_pem).
#elif defined(ARDUINO_ARCH_ESP8266)
#include <WiFiClientSecure.h>
BearSSL::WiFiClientSecure tls;
tls.setBufferSizes(512, 512);   // default 16 KB+ breaks the 80 KB heap
tls.setInsecure();              // LAB ONLY — production: trust anchors / fingerprint
#endif
```

- Production ESP32: embed the server CA with `tls.setCACert(ca_pem)` and keep
  NTP synced. Production ESP8266: `tls.setTrustAnchors(&cert)` or
  `tls.setFingerprint(fp)` — fingerprints die on every server cert renewal,
  trust anchors survive it.
- ESP8266 + HTTPS + JSON + WebSocket at once can exhaust the heap at runtime
  (not at compile time). Check `ESP.getFreeHeap()` in the worst state, not in
  `setup()`.

## 5. Raw TCP with a deadline

When HTTP/MQTT/WebSocket don't fit (custom binary protocol, telnet-style
console), `WiFiClient` (or `EthernetClient` — same methods) with an explicit
deadline. `setTimeout()` bounds each `read()`; the outer deadline bounds the
whole transaction:

```cpp
#include <Client.h>   // explicit: some cores (STM32) don't pull it in via Arduino.h

bool tcpRequest(Client& client, const char* host, uint16_t port,
                const char* req, char* resp, size_t cap, uint32_t timeoutMs) {
  client.setTimeout(timeoutMs);
  if (!client.connect(host, port)) return false;
  client.print(req);
  size_t n = 0;
  const uint32_t t0 = millis();
  while (n + 1 < cap && static_cast<uint32_t>(millis() - t0) < timeoutMs) {
    const int c = client.read();   // -1 on timeout, no hanging
    if (c < 0) break;
    resp[n++] = (char)c;
  }
  resp[n] = '\0';
  client.stop();   // always — see the 8-socket rule above
  return n > 0;
}
```

`Client&` accepts `WiFiClient`, `EthernetClient`, and `WiFiClientSecure`
alike — write the protocol once, pass whichever transport the board has.
