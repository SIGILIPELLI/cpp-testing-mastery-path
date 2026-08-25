# 07 · Simulation & Emulation (QEMU-based Test Rigs)

Module 06 introduced QEMU as a middle ground between host testing and full
hardware-in-the-loop (Level 3, module 03): it runs your actual
cross-compiled binary, on an emulated instruction set and peripheral set,
without needing physical hardware. This module builds that middle ground
into a real, CI-runnable test rig.

!!! note "Environment note"
    QEMU is not installed on this machine and no cross-compiled binary
    exists to run under it, so nothing in this module was executed here —
    every command and captured-output block is a template, checked for
    correctness against QEMU's and semihosting's documented behavior, not
    a live run. Install `qemu-system-arm` (or the relevant target) and a
    matching cross-compiler to reproduce this for real.

## 1. What QEMU system emulation actually gives you

`qemu-system-arm` (and equivalents for other architectures) emulates an
entire target board — CPU, memory map, and a subset of peripherals — well
enough to run real firmware binaries and observe real behavior, including
the classes of bug module 06 identified as invisible to host testing:
integer width, alignment faults, and (for the peripherals QEMU models)
actual register-level behavior.

```bash
qemu-system-arm -M lm3s6965evb -nographic -kernel firmware.elf
```

`-M lm3s6965evb` selects an emulated board model (a Stellaris/TI
Cortex-M3-based eval board that QEMU has long supported); `-nographic`
routes the emulated UART to your terminal instead of opening a display
window — the standard way to get firmware's console output directly into
a CI log.

## 2. Getting test results out: semihosting

Embedded test binaries don't have a filesystem or a normal process exit
code to report pass/fail with. **Semihosting** is the mechanism QEMU (and
real debug probes) provide for target code to make host-like system calls
— including, critically, an exit code.

```c
/* test_main.c -- compiled for the target, run under QEMU */
#include <stdio.h>

extern void exit(int status);   /* semihosting-backed on this target config */

static void test_frame_parse_rejects_overrun(void) {
    uint8_t buf[3] = {0x01, 0xFF, 0xFF};
    frame_t f;
    if (frame_parse(buf, sizeof(buf), &f) != -1) {
        printf("FAIL: test_frame_parse_rejects_overrun\n");
        exit(1);        /* semihosting SYS_EXIT -- QEMU translates this to
                            its own process exit code, which the CI job
                            checks like any other command's exit status */
    }
}

int main(void) {
    test_frame_parse_rejects_overrun();
    printf("ALL TESTS PASSED\n");
    exit(0);
    return 0;
}
```

```bash
arm-none-eabi-gcc -mcpu=cortex-m3 -mthumb --specs=rdimon.specs \
    -T link.ld src/protocol.c test_main.c -o test_main.elf

qemu-system-arm -M lm3s6965evb -nographic -semihosting \
    -kernel test_main.elf
echo "QEMU exit code: $?"
```

```text
ALL TESTS PASSED
QEMU exit code: 0
```

`--specs=rdimon.specs` links against ARM's semihosting-backed C library
variant, and `-semihosting` tells QEMU to honor those calls — together they
turn `printf` and `exit` on a bare-metal target into something a CI
pipeline can capture and check exactly like a normal host test binary's
output and exit code.

## 3. Modeling peripherals: what QEMU can and can't stand in for

| Peripheral class | QEMU support | Fidelity |
|---|---|---|
| UART, timers, interrupt controller | Usually modeled for supported boards | High — these are what QEMU's board models are built around |
| GPIO | Often modeled, but simplistic | Medium — logical level toggling works, real electrical characteristics don't exist to model |
| Vendor-specific peripherals (a specific sensor's I2C interface) | Rarely modeled unless someone wrote a QEMU device model for it | Low to none — usually needs a custom QEMU device model or falls back to HIL |
| DMA, cache behavior, precise timing | Partially modeled at best | Low — timing-sensitive bugs (Level 3, module 03's table) are exactly what QEMU cannot substitute for |

This table is the practical version of Level 3 module 03's core argument,
one layer more specific: QEMU extends what's testable-without-hardware
significantly (anything the emulated board models faithfully) but does not
eliminate the need for HIL — timing-critical and vendor-peripheral-specific
behavior remain HIL's job.

## 4. A three-tier strategy, assembled from modules 02, 03, 06, and this one

```text
Tier 1 -- Host tests (Level 3, module 02)
    Pure logic, no hardware dependency.
    Fastest, runs on every commit, unlimited parallelism.

Tier 2 -- QEMU emulation (this module)
    Real cross-compiled binary, emulated CPU/peripherals.
    Catches: ABI/width/alignment bugs, logic bugs in code that touches
    modeled peripherals (UART, timers), boot/startup code bugs.
    Runs on every commit or every merge -- fast enough (seconds), no
    physical rig needed, no rig contention between parallel CI jobs.

Tier 3 -- HIL (Level 3, module 03)
    Real hardware.
    Catches: timing, unmodeled peripherals, real electrical behavior.
    Runs on a slower cadence (merge to main, nightly) due to rig scarcity.
```

QEMU's sweet spot is specifically that it removes the rig-scarcity
constraint from Tier 3 for the subset of bugs it CAN catch — letting that
subset run on every commit, in parallel, in ordinary CI infrastructure,
while genuinely irreplaceable HIL testing stays reserved for what only real
hardware can show.

## 5. CI integration

```yaml
  qemu-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: sudo apt-get install -y gcc-arm-none-eabi qemu-system-arm
      - run: |
          arm-none-eabi-gcc -mcpu=cortex-m3 -mthumb --specs=rdimon.specs \
              -T link.ld src/protocol.c test_main.c -o test_main.elf
      - name: Run under QEMU with a timeout
        run: |
          timeout 30s qemu-system-arm -M lm3s6965evb -nographic \
              -semihosting -kernel test_main.elf
```

The `timeout` wrapper matters specifically for embedded test binaries: a
firmware bug that hangs (rather than crashing or returning) would otherwise
hang the CI job indefinitely, since there's no OS-level process supervision
the way a host test binary has.

## 6. Full-system emulation vs instruction-level emulation

A distinction worth being precise about: `qemu-system-*` emulates a whole
board (what this module has covered); `qemu-user` (or `qemu-arm`, without
`-system`) emulates just the CPU instruction set for a single
statically-linked user-space binary, translating syscalls to the host OS
directly. The latter is useful for a narrower purpose — running a
cross-compiled Linux userspace binary's test suite without a board model
at all, when your target actually runs Linux rather than bare metal.

```bash
# qemu-user: no board model, just instruction-set translation for a
# statically-linked binary, syscalls passed through to the host kernel.
qemu-arm -L /usr/arm-linux-gnueabihf ./test_binary_arm
```

This is the right tool specifically when the target *is* running a full
OS (embedded Linux) rather than bare-metal firmware — bare-metal code with
no OS underneath needs `qemu-system-*` and semihosting as covered above.

## Cheat sheet

| Question | Answer |
|---|---|
| Bare-metal firmware, need pass/fail from CI? | `qemu-system-*` + semihosting exit codes |
| Board has a UART model? | Route it via `-nographic`, capture stdout in CI logs like a normal test |
| Vendor peripheral has no QEMU model? | Falls back to HIL (Level 3, module 03) — no shortcut |
| Target runs embedded Linux, not bare metal? | `qemu-user` (instruction-level, no board model) instead |
| CI job hangs on a firmware bug? | Wrap the QEMU invocation in `timeout` — no OS supervision otherwise |

## Exercise

1. Sketch the semihosting-based test harness (section 2's shape) for one
   test from Level 3's `msgparser` capstone, including what the pass/fail
   exit code convention would be.
2. Using section 3's table, classify three peripherals your own project
   (real or hypothetical) depends on as QEMU-modelable, partially
   modelable, or HIL-only, and justify each classification.
3. Write the three-tier CI trigger table (as in Level 3 module 03's
   section 6, extended with the QEMU tier from section 4 here) for your
   own project's cadence: what runs per-commit, per-merge, and nightly.
4. Add the `timeout` wrapper from section 5 to a hypothetical QEMU CI step
   and explain in two sentences what class of bug this specifically
   protects the CI pipeline against, distinct from the bug the test itself
   is checking for.
5. Write two sentences on whether your own project's embedded target
   (real or hypothetical) runs bare metal or a full OS, and which QEMU
   invocation style (`qemu-system-*` vs `qemu-user`) that implies.
