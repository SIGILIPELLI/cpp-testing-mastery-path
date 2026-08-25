# 07 · Test Doubles for Hardware/OS

Embedded and systems code is full of things a unit test cannot have: a
temperature register at address `0x40`, a socket, the wall clock, a filesystem
that may be full. The instinct is to declare such code untestable. The reality
is that it is untestable *as written* — it lacks a seam. This module is about
where to put seams, and which kind of double to hang on each one.

## 1. The five doubles

The terms get used loosely; the distinctions matter when you're choosing.

| Double | Behaviour | Use when |
|---|---|---|
| **Dummy** | Passed but never used | Filling a required parameter |
| **Stub** | Returns canned answers | The unit needs an input it can't compute |
| **Spy** | A stub that records calls | You want to assert afterwards |
| **Mock** | Records **and** asserts expectations | The interaction *is* the behaviour |
| **Fake** | A real, simplified implementation | Multi-call state — in-memory store, fake clock |

The important pairing is **stub vs mock**. A stub answers a question; a mock
also insists it was asked. Use a mock only when "was this called, with what, in
what order" is the requirement — otherwise you are asserting on implementation
and your tests will break at every refactor.

Fakes are the most under-used of the five. A fake clock or an in-memory
key-value store gives you realistic multi-call behaviour with none of the
expectation bookkeeping.

## 2. A seam is a place to change behaviour without editing the code

Untestable, because the dependency is hard-wired:

```c
int temperature_c(void) {
    volatile uint32_t *reg = (uint32_t *) 0x40001000;   /* no seam */
    return ((int) *reg * 5) / 16 - 20;
}
```

Testable, because the read is behind a function you can substitute:

```c
/* hw.h */
uint32_t hw_read_register(unsigned addr);

/* temp.c */
int temperature_c(void) {
    uint32_t raw = hw_read_register(0x40001000);
    return ((int) raw * 5) / 16 - 20;
}
```

Nothing about the production behaviour changed. The conversion arithmetic — the
part that actually has bugs — is now testable on your laptop.

## 3. The link seam (C, portable)

Compile the unit under test against the *declaration* only, and link a different
*definition* into the test binary.

```
src/temp.c        + src/hw_stm32.c     -> firmware
src/temp.c        + tests/hw_fake.c    -> test_temp
```

```c
/* tests/hw_fake.c */
#include <stdarg.h>
#include <stddef.h>
#include <setjmp.h>
#include <cmocka.h>

uint32_t hw_read_register(unsigned addr) {
    check_expected(addr);
    return (uint32_t) mock();
}
```

```bash
clang -I include -I/opt/homebrew/opt/cmocka/include \
      src/temp.c tests/hw_fake.c tests/test_temp.c \
      -L/opt/homebrew/opt/cmocka/lib -lcmocka -o test_temp
./test_temp
```

```
[==========] tests: Running 2 test(s).
[ RUN      ] test_temperature_converts_raw
[       OK ] test_temperature_converts_raw
[ RUN      ] test_temperature_handles_zero
[       OK ] test_temperature_handles_zero
[==========] tests: 2 test(s) run.
[  PASSED  ] 2 test(s).
```

This is the seam to reach for first in C. It needs no linker extensions, no
preprocessor tricks, and it works identically on Linux and macOS — unlike
`--wrap` (Module 3, §5).

## 4. The function-pointer seam (C, runtime switchable)

When one binary must swap implementations at runtime — a hardware-in-the-loop
mode, a simulator build — put the dependency in a struct of function pointers.

```c
/* hal.h */
typedef struct {
    uint32_t (*read_register)(unsigned addr);
    void     (*write_register)(unsigned addr, uint32_t value);
    uint64_t (*millis)(void);
} hal_t;

extern const hal_t *hal;     /* the seam */
void hal_set(const hal_t *impl);
```

```c
/* tests/fake_hal.c */
static uint32_t fake_regs[256];
static uint64_t fake_now;

static uint32_t fake_read(unsigned a)              { return fake_regs[a & 0xFF]; }
static void     fake_write(unsigned a, uint32_t v) { fake_regs[a & 0xFF] = v; }
static uint64_t fake_millis(void)                  { return fake_now; }

const hal_t fake_hal = { fake_read, fake_write, fake_millis };

void fake_hal_set_register(unsigned a, uint32_t v) { fake_regs[a & 0xFF] = v; }
void fake_hal_advance_ms(uint64_t d)               { fake_now += d; }
```

```c
static void test_debounce_ignores_short_press(void **state) {
    (void) state;
    hal_set(&fake_hal);
    fake_hal_set_register(BUTTON_REG, 1);
    fake_hal_advance_ms(5);                 /* below the 20 ms threshold */
    assert_false(button_pressed());
}
```

This is a **fake**, not a mock: it holds real state across calls, so a test can
write a register and read it back. That makes multi-step scenarios readable
without a wall of expectations.

!!! warning "Reset global fakes between tests"
    `fake_regs` and `fake_now` are file-scope state that survives from one test
    to the next. Without a `setup` that zeroes them, tests pass in file order and
    fail when shuffled. Give every global fake an explicit `reset()` and call it
    from the fixture.

## 5. Dependency injection (C++)

In C++ the seam is usually an abstract interface taken by reference or
`unique_ptr`. Module 2 covered mocking it; here is the same seam with a **fake**
instead, which is often the better tool.

```cpp
class Clock {
public:
    virtual ~Clock() = default;
    virtual std::chrono::steady_clock::time_point now() const = 0;
};

class SystemClock : public Clock {
public:
    std::chrono::steady_clock::time_point now() const override {
        return std::chrono::steady_clock::now();
    }
};

class FakeClock : public Clock {
public:
    std::chrono::steady_clock::time_point now() const override { return now_; }
    void advance(std::chrono::milliseconds d) { now_ += d; }
private:
    std::chrono::steady_clock::time_point now_{};
};
```

```cpp
TEST(RateLimiter, AllowsAgainAfterWindow) {
    FakeClock clock;
    RateLimiter limiter(clock, /*max*/ 2, std::chrono::seconds(60));

    EXPECT_TRUE (limiter.allow("user-1"));
    EXPECT_TRUE (limiter.allow("user-1"));
    EXPECT_FALSE(limiter.allow("user-1"));   // over the limit

    clock.advance(std::chrono::seconds(61)); // instant, not a real minute
    EXPECT_TRUE (limiter.allow("user-1"));
}
```

The alternative is `std::this_thread::sleep_for(61s)` in a test. That is 61
seconds of CI time per run, and it is still flaky under load. Faking time is the
single highest-value double in most codebases.

## 6. Faking the OS

The same pattern generalises. Wrap the syscall surface you use — not all of
POSIX, just what you touch — and fake it.

```cpp
class FileSystem {
public:
    virtual ~FileSystem() = default;
    virtual bool   exists(const std::string& path) const = 0;
    virtual std::string read(const std::string& path) const = 0;   // throws
    virtual void   write(const std::string& path, const std::string& data) = 0;
};

class InMemoryFileSystem : public FileSystem {
public:
    bool exists(const std::string& p) const override { return files_.count(p) > 0; }
    std::string read(const std::string& p) const override {
        auto it = files_.find(p);
        if (it == files_.end()) throw std::runtime_error("ENOENT: " + p);
        return it->second;
    }
    void write(const std::string& p, const std::string& d) override {
        if (full_) throw std::runtime_error("ENOSPC");
        files_[p] = d;
    }
    void set_full(bool v) { full_ = v; }          // test-only knob
private:
    std::map<std::string, std::string> files_;
    bool full_ = false;
};
```

`set_full(true)` is the point of the exercise. A disk-full error path is nearly
impossible to trigger against a real filesystem and trivial against a fake — and
error paths are where the bugs live.

The same trick covers the rest of the OS surface:

| Dependency | Fake gives you |
|---|---|
| Clock | Instant timeouts, deterministic scheduling |
| Filesystem | `ENOSPC`, `EACCES`, missing files, on demand |
| Network | Timeouts, partial reads, connection reset |
| RNG | Reproducible sequences from a fixed seed |
| Environment / config | No process-global mutation between tests |
| `malloc` | Allocation failure at the *n*th call |
| UART / SPI / I²C | Injected framing errors and NAKs |

## 7. Choosing a seam

| Language | Seam | Cost | Portable | Runtime-switchable |
|---|---|---|---|---|
| C | Link seam (separate `.c` per target) | Build config | Yes | No |
| C | Function-pointer struct | One indirection | Yes | Yes |
| C | `--wrap` linker interposition | None | **No** (GNU only) | No |
| C | `#define` over the symbol | None | Yes | No (and hides real code) |
| C++ | Virtual interface + injection | vtable dispatch | Yes | Yes |
| C++ | Template policy parameter | None (compile-time) | Yes | No |
| C++ | Link seam | Build config | Yes | No |

Preprocessor substitution deserves its "hides real code" note: `#define read_reg
fake_read_reg` means the test compiles something different from what ships, so a
bug in the real path can hide behind a passing suite.

## 8. How much to fake

Every double is a claim that the real thing behaves a certain way, and the claim
can be wrong. A fake filesystem that never returns `EINTR` will let an `EINTR`
bug through, and your suite will be green the whole time.

The discipline is layered: unit tests against fakes for logic and error paths;
**integration tests against the real dependency** for the assumptions the fakes
encode. If you never run against real hardware, you have tested your fake.

## Exercise

Build a thermostat controller with every dependency faked, then verify the fakes.

```c
/* Turns heating on below (target - hysteresis), off above (target + hysteresis).
   Refuses to change state more often than once per min_cycle_ms. */
typedef struct { int target_c; int hysteresis_c; uint64_t min_cycle_ms; } config_t;
void thermostat_init(const config_t *cfg);
void thermostat_tick(void);          /* reads sensor, may drive the relay */
```

1. **Define a `hal_t`** with `read_temperature_c()`, `set_relay(bool)`, and
   `millis()`. Implement the controller against it and nothing else.

2. **Write a fake HAL** with a settable temperature, a recorded relay history
   (a spy), and an advanceable clock. Give it a `reset()` and call it from your
   per-test setup.

3. **Test the hysteresis band** with at least six cases: well below target, just
   below the lower threshold, exactly at the lower threshold, inside the band,
   exactly at the upper threshold, and well above. Assert the relay state after
   each.

4. **Test the minimum cycle time** — the case that is impossible without a fake
   clock. Drive a temperature swing that would toggle the relay, advance the
   clock by less than `min_cycle_ms`, and assert nothing changed. Advance past
   it and assert it did. Record how long this test takes to run; compare with
   what a real 5-minute cycle time would cost.

5. **Inject a sensor fault.** Have the fake return a sentinel error value and
   assert the controller leaves the relay in its last safe state rather than
   toggling. This is the path that matters and the one you cannot trigger with
   real hardware on your desk.

6. **Classify your doubles.** For each part of your fake HAL, name which of the
   five kinds from section 1 it is, and justify it in one line.

7. **Verify the fake.** Write one integration test that runs the same controller
   against the real HAL (or a loopback/simulator build) and confirms at least the
   basic on/off behaviour. Write three sentences on which assumptions in your
   fake this test does *not* verify, and how you would catch those.
