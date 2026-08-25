# 03 · Hardware-in-the-Loop Basics

Host tests (module 01-02) prove your logic is correct assuming the hardware
behaves the way your fakes say it does. A **hardware-in-the-loop (HIL)** rig
closes that gap: it runs your real firmware on real (or close-to-real)
hardware, driven and observed automatically, so the assumption itself gets
checked on every run instead of once by hand.

!!! note "Environment note"
    No physical test rig exists on this machine, and none of the toolchain
    invocations below (cross-compilers, flashing tools, rig control scripts)
    were run here — this is a conceptual and structural walkthrough, marked
    as such throughout. Treat every command block as a template to adapt to
    your actual rig, not as verified output.

## 1. What a HIL rig actually is

A minimal rig has four parts:

| Part | Role | Example |
|---|---|---|
| Target | The real board running your firmware | An STM32 dev board |
| Harness | Drives inputs and reads outputs programmatically | A second MCU or a PC with GPIO/relay/DAC hardware |
| Flasher | Loads a fresh build before each run | `openocd`, `st-flash`, a JTAG probe |
| Orchestrator | Sequences flash → drive → observe → assert, and reports pass/fail | A Python or shell script, wired into CI |

The orchestrator is the piece that turns "I plugged in a board and poked at
it" into a repeatable, CI-triggerable test — that's the entire point of HIL
over ad hoc bench testing.

## 2. A HIL test looks like a host test, one layer removed

Same shape as GoogleTest — setup, act, assert — but "act" happens over a
wire to real hardware and "assert" reads back real signals.

```python
# hil/test_thermostat_relay.py -- orchestrator-side, drives the real board
import time
from hil_harness import Harness  # wraps serial + GPIO + relay control

def test_relay_turns_on_below_threshold(harness: Harness):
    harness.flash("build-target/firmware.elf")
    harness.reset()
    harness.set_simulated_temperature_c(15)   # via a DAC feeding the ADC pin
    time.sleep(0.5)                            # let the control loop run a few ticks
    assert harness.read_relay_state() == "ON"

def test_relay_turns_off_above_threshold(harness: Harness):
    harness.flash("build-target/firmware.elf")
    harness.reset()
    harness.set_simulated_temperature_c(30)
    time.sleep(0.5)
    assert harness.read_relay_state() == "OFF"

def test_relay_respects_minimum_cycle_time(harness: Harness):
    harness.flash("build-target/firmware.elf")
    harness.reset()
    harness.set_simulated_temperature_c(15)
    time.sleep(0.5)
    assert harness.read_relay_state() == "ON"
    harness.set_simulated_temperature_c(30)     # would flip it off immediately...
    time.sleep(0.1)                             # ...but this is under min_cycle_ms
    assert harness.read_relay_state() == "ON"   # still on -- the debounce held
```

This is deliberately the same thermostat controller from Level 2 module 07's
exercise. That exercise built the fake HAL and unit-tested the logic; this is
the second half of the discipline from that module's section 8 — verifying
the fakes against the real thing.

## 3. What HIL catches that host tests structurally cannot

| Failure class | Host test (fake HAL) | HIL |
|---|---|---|
| Off-by-one in hysteresis math | Catches it | Catches it, redundantly |
| ADC noise causing spurious readings | Cannot see it — the fake has no noise | Catches it |
| Relay physically too slow to switch within assumed timing | Cannot see it | Catches it |
| Wrong pin mapping in the board support package | Cannot see it — fake doesn't know about pins | Catches it |
| Power-on register state differing from assumed reset value | Cannot see it | Catches it |
| Logic error in the debounce algorithm itself | Catches it (fast, cheap) | Would catch it too, but slower and more expensive |

The table is the argument for the layered strategy: keep the algorithm bugs
in the cheap, fast layer (host tests) and reserve the expensive layer (HIL)
for the failure classes that are structurally invisible without real
silicon.

## 4. Determinism is the hard part

A HIL rig introduces real-world nondeterminism a host test never has: cable
noise, USB timing jitter, a relay that sometimes bounces. Two disciplines
keep this from turning into a permanently-flaky suite:

- **Settle time, not a fixed sleep.** Poll the observed signal with a
  timeout rather than sleeping a fixed duration and hoping — same principle
  as module 01's async-test guidance, more important here because hardware
  settle times vary run to run.
- **Debounce the *test's* reading**, separately from any debounce in the
  firmware under test. Read a GPIO three times over 30ms and require
  agreement before trusting it, so line noise on the harness side doesn't
  masquerade as a firmware bug.

```python
def read_stable_relay_state(harness: Harness, timeout_s=2.0) -> str:
    deadline = time.monotonic() + timeout_s
    last = None
    stable_count = 0
    while time.monotonic() < deadline:
        state = harness.read_relay_state()
        if state == last:
            stable_count += 1
            if stable_count >= 3:
                return state
        else:
            stable_count = 0
        last = state
        time.sleep(0.01)
    raise TimeoutError(f"relay state never stabilized, last={last}")
```

## 5. Fixture lifecycle: flash, reset, isolate

Every HIL test should start from a known-good state, exactly like a unit
test fixture — except "known-good state" now means a freshly flashed board,
not a freshly constructed object.

```python
import pytest

@pytest.fixture
def harness():
    h = Harness(port="/dev/ttyUSB0")
    h.power_cycle()          # clears any residual state a soft reset might miss
    h.flash("build-target/firmware.elf")
    h.reset()
    yield h
    h.power_off()            # always leave the rig in a safe, known state
```

!!! warning "A soft reset is not always a clean slate"
    Some peripherals (EEPROM, external flash, RTC-backed state) survive a
    reset line toggle. If your firmware persists anything, a HIL fixture
    needs to explicitly wipe that persistent state too, or tests will leak
    state into each other exactly like the shared-global trap in Level 2
    module 07.

## 6. Where HIL fits in CI

HIL rigs are scarce (one or two boards, not a fleet), slow (seconds per
flash cycle), and physical (a bad build can, in principle, leave a board in
a bad state). The usual compromise:

| Trigger | Runs |
|---|---|
| Every commit / PR | Host tests only (modules 01-02) — fast, unlimited parallelism |
| Merge to main | HIL smoke suite — a handful of critical-path tests |
| Nightly | Full HIL suite |
| Before a release tag | Full HIL suite + manual bench verification |

This mirrors the general principle from Level 2 module 07 section 8: fakes
for speed on every commit, the real thing on a slower cadence to catch what
the fakes cannot see.

## Cheat sheet

| Question | Answer |
|---|---|
| What does HIL add over host tests? | Catches failures that only exist in real electrical/timing behavior |
| What should still be a host test? | Anything the fake HAL can express faithfully — most logic |
| How do I avoid HIL flakiness? | Poll with a timeout and a stability check, never a bare `sleep` |
| How often should HIL run? | Smoke suite per merge, full suite nightly/pre-release |
| What must a HIL fixture guarantee? | A truly clean starting state — power cycle, not just reset, if persistence exists |

## Exercise

1. Sketch a `Harness` interface (method signatures only) for a rig that
   tests a UART-based sensor board: needs to flash, reset, inject a
   simulated sensor value, and read back a UART response. You don't need
   real hardware — this is a design exercise in what the seam should look
   like.
2. Write the three thermostat HIL tests from section 2 as pytest functions
   using your `Harness` interface, including the fixture from section 5.
3. Add `read_stable_relay_state`-style debouncing to your test's read path
   and explain in two sentences why a naive single read would be more
   flaky specifically on a HIL rig than in a host test.
4. Design the CI trigger table from section 6 for your specific project —
   name what you'd put in "smoke suite" versus "full suite" and justify
   the split by cost (rig-seconds) versus risk (what a regression there
   would cost you).
5. Write two sentences on one failure class from section 3 that your
   current test suite (real or hypothetical) has no coverage for at any
   level, host or HIL, and what it would take to add it.
