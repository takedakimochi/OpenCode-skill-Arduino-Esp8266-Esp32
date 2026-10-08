# Concurrency: interrupts, atomics, and the ESP32's real second CPU

## 1. ISR rules that hold on every core

An interrupt handler must be re-entrant-free, short, and free of anything that
can block. Concretely, **never** inside an ISR:

- `Serial.print` / `Serial.read` / any `Print` or `Stream` method
- `delay`, `delayMicroseconds`, `micros()` (on AVR `micros()` is unsafe in an ISR)
- `new`, `String`, `malloc`, `printf`
- I2C/SPI transactions, `Wire.requestFrom`, `SD` file access
- `log_i`/`ESP_LOG*` (string literals live in flash, which is not accessible from
  an ISR on ESP32)
- any loop that waits on hardware

**Do** inside an ISR: set a `volatile` flag, bump a counter, latch a timestamp,
toggle an output pin, or push to a lock-free/ISR-safe queue.

```cpp
// One alias for the core-specific IRAM attribute (see porting.md §2). Never
// use a bare empty fallback: on ESP8266 an ISR linked into flash crashes the
// moment it fires.
#if defined(ARDUINO_ARCH_ESP32)
#define ISR_ATTR IRAM_ATTR
#elif defined(ARDUINO_ARCH_ESP8266)
#define ISR_ATTR ICACHE_RAM_ATTR
#else
#define ISR_ATTR   // AVR/STM32: no placement attribute needed
#endif

volatile uint8_t g_rising_edges = 0;

void ISR_ATTR onPulse() {
  if (g_rising_edges < 255) ++g_rising_edges;
}
```

An ESP32 ISR handler must be linked into IRAM (otherwise the flash cache can be
disabled while the handler runs). A function marked `IRAM_ATTR` is still allowed
to call flash functions per the linker script on some cores — so treat
"crashes only when an interrupt fires" as a flash-access symptom and look for
`Serial`/`printf`/`ESP_LOG`/SPI inside the handler.

## 2. Attaching interrupts portably

```cpp
// AVR requires digitalPinToInterrupt(); ESP32/STM32 ignore the wrapper.
attachInterrupt(digitalPinToInterrupt(pins::kSensor), onPulse, RISING);
```

`attachInterrupt(0, ...)` works on ESP32/STM32 but is a compile error on AVR —
always use `digitalPinToInterrupt()`.

AVR specifics:

- External interrupts are `INT0` (D2) and `INT1` (D3) only. `attachInterrupt` on
  any other pin uses **PCINT** (pin-change), which fires on any change, so filter
  the edge yourself.
- Every variable an ISR touches **and** `loop()` reads/writes must be `volatile`.
  Without it the compiler hoists the load out of `loop()` and the ISR result is
  never seen.
- `volatile` does **not** make a 64-bit or multi-word read atomic. On AVR, 16-bit
  and 32-bit values shared with an ISR need interrupt masking:

```cpp
noInterrupts();
const uint32_t snapshot = g_counter;   // 32-bit on an 8-bit CPU = 4 byte moves
interrupts();
```

- Changing an ISR's variables (e.g. reloading a timer compare register) needs
  `noInterrupts()` around the write, or you can trap mid-update.

## 3. ESP32 is not a single-threaded Arduino

Core 2.x runs `loop()` on **core 1** while the WiFi/TCP/IP stack and the SDK
event loop run on **core 0**. That means:

- `loop()` can be starved by WiFi work; a long blocking loop in `setup()` can
  trigger the WiFi/task watchdog and reboot with
  `Guru Meditation Error: Task watchdog got triggered`.
- Do **not** call most `WiFi`, `WebServer`, `HTTPClient`, or `PubSubClient` calls
  from both cores without a mutex. They are not re-entrant.
- `analogRead` on ADC2 fails while WiFi is up (see `peripherals.md`).

Sharing a device (I2C bus, SPI bus, a log buffer) between the loop and other
tasks:

```cpp
#include <freertos/FreeRTOS.h>
#include <freertos/semphr.h>
static SemaphoreHandle_t g_i2c_mutex = nullptr;

// in setup()
g_i2c_mutex = xSemaphoreCreateMutex();

bool lockI2c(uint32_t timeoutMs) {
  return xSemaphoreTake(g_i2c_mutex, pdMS_TO_TICKS(timeoutMs)) == pdTRUE;
}
void unlockI2c() { xSemaphoreGive(g_i2c_mutex); }
```

Use a **mutex**, not a binary semaphore, for mutual exclusion, and always give it
back on every exit path. `portMAX_DELAY` for a `take` inside a task is usually
correct; inside an ISR use the `FromISR` variants.

Queue from ISR to task:

```cpp
void IRAM_ATTR onIsr() {
  BaseType_t woken = pdFALSE;
  xQueueSendFromISR(g_queue, &msg, &woken);
  if (woken == pdTRUE) portYIELD_FROM_ISR();
}
```

## 4. Running your own tasks on ESP32

```cpp
void samplerTask(void* arg) {
  static uint8_t buf[128];               // stack for locals: 8 KB default
  for (;;) {
    fillSample(buf, sizeof buf);
    publish(buf);
    vTaskDelay(pdMS_TO_TICKS(10));      // never busy-wait
  }
}
void setup() { xTaskCreatePinnedToCore(samplerTask, "sampler", 4096, nullptr, 1, nullptr, 0); }
```

- If a task is subscribed to the task WDT, call `esp_task_wdt_reset()` in its
  loop, or the board reboots.
- `vTaskDelay(pdMS_TO_TICKS(1))` is 1 tick; on the default config a tick is
  ~1 ms (Arduino core 2.x: 120 Hz on some, 1000 Hz with `CONFIG_FREERTOS_HZ`).
  Do not assume 1 ms — use `millis()`/`micros()` for real timing and ticks only
  for yielding.
- Every task's stack must be sized for its **deepest** call chain, not its
  average. Watch for `printf`/float formatting, which wants ~300+ bytes extra.
- Guard the task body with a flag or a queue; do not share raw globals with
  `loop()`.
- Tasks, `analogRead`, and `micros()` all work, but on 64-bit members of a
  shared struct are not atomic even on this 32-bit CPU — use a mutex or
  `std::atomic<uint32_t>`.

Guard any task-based code so it never compiles for a target that cannot do it:

```cpp
#if defined(ARDUINO_ARCH_ESP32)
  xTaskCreatePinnedToCore(...);
#endif
```

On AVR and bare STM32, `loop()` is your only thread, so the same design becomes
**cooperative scheduling**: a table of objects, each with `update(now)`, called
in a fixed order from `loop()`. That is why `patterns.md` is built around
`update(now)` rather than callbacks.

## 5. Watchdogs and reset-cause triage

When a board "reboots by itself", print the reason before doing anything else.
`esp_reset_reason()` is ESP32-only and `Serial.printf` does not exist on AVR —
guard both:

```cpp
#if defined(ARDUINO_ARCH_AVR)
#include <avr/io.h>   // MCUSR
#endif

void printResetCause(void) {
#if defined(ARDUINO_ARCH_ESP32)
  Serial.print(F("reset="));
  Serial.println((int)esp_reset_reason());
#elif defined(ARDUINO_ARCH_ESP8266)
  Serial.print(F("reset="));
  Serial.println(ESP.getResetReason());   // "External System", "Software/...", "Deep-Sleep Wake", ...
#elif defined(ARDUINO_ARCH_AVR)
  const uint8_t cause = MCUSR;
  MCUSR = 0;   // clear so the next reset reads fresh
  // bit3 WDRF watchdog, bit2 BORF brown-out, bit1 EXTRF external, bit0 PORF power-on
  Serial.print(F("mcusr=0x"));
  Serial.println(cause, HEX);
#endif
}
```

| Print | Meaning | Usual fix |
| ----- | ------- | ---------- |
| `ESP_RST_POWERON` | normal | — |
| `ESP_RST_TASK_WDT` / `SW_RESET` | task watchdog | blocking call in a task; add `vTaskDelay` / `esp_task_wdt_reset` |
| `ESP_RST_PANIC` | unhandled exception | check the backtrace printed at 115200 baud *before* the reboot |
| `ESP_RST_BROWNOUT` | voltage dip | see `troubleshooting.md` |
| `ESP_RST_INT_WDT` | ISR too long | move work out of the ISR |

On AVR, the reset cause is in `MCUSR` (`(void)MCUSR; MCUSR = 0;` at the top of
`setup()`); on STM32, `HAL_RCC_GetResetSource()` or the IWDG flags in
`HAL_RCC_GetStatus()`. Read the cause first, then fix — guessing wastes days.

A panic handler that prints before dying is worth 10 lines:

```cpp
void loop() { vTaskDelay(1); }   // never return to a clean loopTask
```

## 6. ESP8266: one core, background WiFi

No tasks, no second core — `loop()` is the only thread, and the WiFi stack
borrows it between your iterations. The contract is simple: never hog the CPU
for more than ~a few hundred ms without handing control back.

```cpp
for (uint32_t i = 0; i < 100000; ++i) {
  crunch(i);
  if ((i & 0xFF) == 0) yield();   // feed the WiFi stack + SW watchdog
}
```

- `yield()` (or any `delay()`) runs the WiFi stack and feeds the software
  watchdog. A loop without it ends in `wdt reset` — the single most common
  ESP8266 crash.
- For very tight loops that cannot yield, `ESP.wdtFeed()` feeds only the
  watchdog (WiFi still starves — bounded use only).
- `delay(ms)` on ESP8266 yields internally, so `delay(10)` in `loop()` is both
  a timing tool and good citizenship. This is the one core where a small
  `delay()` at the end of `loop()` is idiomatic, not a bug.
- ISRs use `ICACHE_RAM_ATTR` (via the `ISR_ATTR` alias in §1), stay tiny, and
  — like everywhere — never touch `Serial`, WiFi, or the heap.
