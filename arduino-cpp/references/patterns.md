# Patterns: portable, testable, non-blocking

Every pattern here takes time as a **parameter**. That single rule is what makes
firmware testable on the host and portable across cores, because no logic
depends on `millis()` being a real clock.

## 1. Non-blocking scheduler

```cpp
// include/Scheduler.h
#pragma once
#include <stdint.h>

struct Task {
  void (*update)(uint32_t now, void* ctx);
  void* ctx;
};

class Scheduler {
 public:
  bool add(Task t) {
    if (count_ >= kMax) return false;
    tasks_[count_++] = t;
    return true;
  }
  void run(uint32_t now) {
    for (uint8_t i = 0; i < count_; ++i) tasks_[i].update(now, tasks_[i].ctx);
  }

 private:
  static constexpr uint8_t kMax = 12;
  Task tasks_[kMax]{};
  uint8_t count_ = 0;
};
```

`loop()` becomes exactly three lines:

```cpp
void loop() { scheduler.run(millis()); }
```

Rules that keep this honest:

- The whole `run()` must finish well inside one loop period (target < 5 ms of
  work per iteration, measured).
- A task that needs to run faster than `loop()` (sub-100 ms) belongs in a timer
  ISR that only sets a flag, or in an ESP32 task.
- No task may ever block. A blocking call in one task stalls every other task.

## 2. Rollover-safe deadlines

```cpp
// True once per period, forever, even across millis() overflow.
inline bool due(uint32_t now, uint32_t last, uint32_t period) {
  return static_cast<uint32_t>(now - last) >= period;
}

// Advance a deadline without drift; caps catch-up after a long stall.
inline uint32_t nextDeadline(uint32_t last, uint32_t period, uint32_t cap) {
  return last + (period > cap ? cap : period);
}
```

Unsigned subtraction is the trick: it is correct across the 49.7-day `millis()`
wrap. `micros()` wraps every ~71 minutes, `millis()` every ~49.7 days, and
`micros()` on AVR is only accurate to ±4 µs.

```cpp
static uint32_t last = 0;
if (due(now, last, 1000)) { last = now; doWork(); }   // no drift
```

## 3. Button debounce (state + time, not `delay`) 

```cpp
class Button {
 public:
  explicit Button(uint8_t pin) : pin_(pin) {}

  void begin() {
    pinMode(pin_, INPUT_PULLUP);
    stable_ = raw_ = (digitalRead(pin_) == LOW);   // true == pressed
    changedAt_ = 0;
  }

  void update(uint32_t now) {
    const bool r = (digitalRead(pin_) == LOW);
    if (r != raw_) { raw_ = r; changedAt_ = now; }
    if (static_cast<uint32_t>(now - changedAt_) >= kDebounceMs && raw_ != stable_) {
      stable_ = raw_;
      pressed_ = true;   // consumed by the app via takePress()
    }
  }

  bool takePress() { const bool p = pressed_; pressed_ = false; return p; }
  bool held() const { return stable_; }   // debounced level, not the raw pin

 private:
  static constexpr uint32_t kDebounceMs = 25;
  uint8_t pin_;
  bool raw_ = false, stable_ = false, pressed_ = false;
  uint32_t changedAt_ = 0;
};
```

`raw_`/`stable_` hold a *pressed* boolean, not a pin level, so `held()` is just
`stable_` — comparing it to `LOW` is a silent bug. Test this class; a
debounce/state pair is exactly the logic that is easy to get wrong and hard to
eyeball on hardware.

On ESP32 a 25 ms poll interval can miss short presses because the loop can
stall; if presses must not be missed, count edges in the ISR and read the
counter here.

## 4. Explicit state machine (no `goto`, no `delay`)

```cpp
enum class PumpState : uint8_t { Idle, Priming, Running, Draining, Fault };

class Pump {
 public:
  void start(uint32_t now) {
    now_ = now;
    if (state_ == PumpState::Idle) enter(PumpState::Priming);
  }
  void update(uint32_t now) {
    now_ = now;
    switch (state_) {
      case PumpState::Idle:     break;
      case PumpState::Priming:  if (elapsed() >= 2000)  enter(PumpState::Running); break;
      case PumpState::Running:  if (elapsed() >= 30000) enter(PumpState::Draining); break;
      case PumpState::Draining: if (dryDetected())       enter(PumpState::Fault);
                                else if (elapsed() >= 5000) enter(PumpState::Idle);
                                break;
      case PumpState::Fault:    if (elapsed() >= 60000) enter(PumpState::Idle); break;
    }
    applyOutputs();
  }
  PumpState state() const { return state_; }

 private:
  void enter(PumpState s) { state_ = s; entered_ = now_; }
  uint32_t elapsed() const { return static_cast<uint32_t>(now_ - entered_); }
  void applyOutputs() { digitalWrite(kPumpPin, state_ == PumpState::Running ? HIGH : LOW); }
  static constexpr uint8_t kPumpPin = 7;
  PumpState state_ = PumpState::Idle;
  uint32_t entered_ = 0, now_ = 0;
};
```

Each state gets its own timeout, so there is exactly one place to reason about
timing and no nested `delay()` ever appears.

## 5. Fixed-capacity ring buffer (no malloc, usable from an ISR)

```cpp
template <typename T, uint8_t N>
class Ring {
 public:
  bool push(const T& v) {
    const uint8_t next = static_cast<uint8_t>((head_ + 1u) % N);
    if (next == tail_) return false;              // full: drop, never block
    buf_[head_] = v;
    head_ = next;
    return true;
  }
  bool pop(T& out) {
    if (head_ == tail_) return false;
    out = buf_[tail_];
    tail_ = static_cast<uint8_t>((tail_ + 1u) % N);
    return true;
  }
  uint8_t size() const { return static_cast<uint8_t>((head_ + N - tail_) % N); }
  void clear() { head_ = tail_ = 0; }

 private:
  T buf_[N]{};
  uint8_t head_ = 0, tail_ = 0;
};
```

Always choose `N+1` semantics (as above) so "full" and "empty" are
distinguishable. A full buffer that silently overwrites loses the newest sample;
decide which one you want and log when it happens.

## 6. PID with anti-windup (derivative on measurement)

Floats are fast enough for control-loop rates (10–100 Hz) on all three targets,
including AVR — just never run this in an ISR. No Arduino dependency, so it is
host-testable as-is.

```cpp
class Pid {
 public:
  Pid(float kp, float ki, float kd, float iLimit, float outMax = 255.0f)
      : kp_(kp), ki_(ki), kd_(kd), iLimit_(iLimit), outMax_(outMax) {}

  void reset() { integral_ = 0; prevMeasured_ = 0; first_ = true; output_ = 0; }

  // dt in seconds; output is clamped so the caller can apply it directly.
  float update(float setpoint, float measured, float dt) {
    if (dt <= 0) return output_;
    const float error = setpoint - measured;
    integral_ += error * dt;
    integral_ = clamp(integral_, -iLimit_, iLimit_);   // anti-windup
    float derivative = 0;
    if (!first_) derivative = -(measured - prevMeasured_) / dt;  // on measurement:
    first_ = false;                                              // no setpoint kick
    prevMeasured_ = measured;
    output_ = clamp(kp_ * error + ki_ * integral_ + kd_ * derivative,
                    -outMax_, outMax_);
    return output_;
  }
  float output() const { return output_; }

 private:
  static float clamp(float v, float lo, float hi) {
    return v < lo ? lo : (v > hi ? hi : v);
  }
  float kp_, ki_, kd_, iLimit_, outMax_;
  float integral_ = 0, prevMeasured_ = 0, output_ = 0;
  bool first_ = true;
};
```

Integral windup, derivative kick, and output clamping are the three things that
make a hand-rolled PID misbehave; the clamp on `integral_` (anti-windup) matters
most during startup. Tuning order: P until it oscillates, back off 30–50%, then
a little I, then a little D. Skip the `i` term entirely if tuning time is
scarce.

## 7. Driver shape that survives a port

```cpp
// lib/TempSensor/TempSensor.h
#pragma once
#include <stdint.h>

class ITempSensor {
 public:
  virtual ~ITempSensor() = default;
  virtual bool begin() = 0;
  virtual bool readCelsius(float& out) = 0;     // false = bus/CRC failure
};

class Bme280 final : public ITempSensor {
 public:
  explicit BmeSensor(uint8_t addr) : addr_(addr) {}
  bool begin() override;
  bool readCelsius(float& out) override;
 private:
  uint8_t addr_;
};

// src/App.cpp — the only place that knows the concrete type
ITempSensor* sensor = new Bme280(0x76);
```

Rules: no `Serial`, no `millis`, no globals, no `delay` inside a driver. That
makes it host-testable with a fake transport and swappable between an I2C part
and, say, an SPI part later.

## 8. Parsing serial commands safely

Keep the parser free of `Serial` so it is testable, and drain the port in
`loop()`:

```cpp
// Transport side (src/main.cpp) — the only place that touches Serial.
inline void pumpParser(LineParser& p) {
  while (Serial.available() > 0) p.feed(Serial.read());   // drain: required everywhere
}
```

```cpp
// Logic side (lib/LineParser/LineParser.h) — pure, host-testable.
class LineParser {
 public:
  using Handler = void (*)(const char*, void*);
  LineParser(Handler h, void* ctx) : h_(h), ctx_(ctx) {}

  void feed(int c) {
    if (c < 0) return;                        // Serial::read() returns -1 on EOF
    if (c == '\n' || c == '\r') {
      if (len_ && !overflowed_) { line_[len_] = '\0'; h_(line_, ctx_); }
      len_ = 0; overflowed_ = false;
    } else if (!overflowed_) {
      if (len_ + 1u < sizeof line_) {         // bound the write
        line_[len_++] = static_cast<char>(c);
      } else {
        overflowed_ = true; len_ = 0;         // overflow: drop the whole line
      }
    }
  }

 private:
  Handler h_;
  void* ctx_;
  char line_[64]{};
  uint8_t len_ = 0;
  bool overflowed_ = false;
};
```

- `Serial.read()` returns `int` (`-1` on timeout), not `char`. Comparing it to a
  `char` and getting EOF wrong is a classic bug.
- `while (Serial.available())` is mandatory, not optional: the RX buffer is
  64 bytes on AVR and overflows silently.
- Use `strncmp`/`strtoul` rather than `sscanf` on AVR — `sscanf` pulls in a lot
  of flash and `%s` is easy to overflow. `strtok` is acceptable if the buffer is
  only used once.
- If the command changes hardware state, act on a copy and commit after full
  validation, so a truncated line cannot half-configure the device.

## 9. Testable timing without a board

The `update(uint32_t now)` shape means the host test is a plain loop:

```cpp
#include <unity.h>
#include "Pump.h"
void setUp(void) {} void tearDown(void) {}

void test_pump_runs_for_thirty_seconds(void) {
  Pump p; p.start(0);
  for (uint32_t t = 0; t < 30000; t += 10) p.update(t);
  TEST_ASSERT_TRUE(p.state() == PumpState::Running);
  p.update(35000);
  TEST_ASSERT_TRUE(p.state() == PumpState::Draining);
}

int main(int, char**) {
  UNITY_BEGIN();
  RUN_TEST(test_pump_runs_for_thirty_seconds);
  return UNITY_END();
}
```

See `build-and-flash.md` for the `native` env and the `Arduino.h` stub that makes
this compile without hardware.
