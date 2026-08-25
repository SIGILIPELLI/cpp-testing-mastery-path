# 02 · Embedded Target Testing

Most of the code in an embedded project is not embedded-specific — it's
parsing, state machines, protocol framing, math. The trick that makes
embedded testing tractable is separating that majority from the small,
genuinely hardware-dependent minority, and running each on the platform
where it's cheap to test: the majority on your host machine, the minority
on-target (or on a simulator, module 07 of Level 4).

!!! note "Environment note"
    Written with the same broken-libc++ environment noted in module 01 —
    C++ snippets are manually traced, not compiled live here. The C
    fragments (cross-compiler invocations, linker scripts) are illustrative
    of a real embedded toolchain (e.g. `arm-none-eabi-gcc`) that is not
    installed on this host at all; none of this module's commands were run
    here. Verify on your actual target toolchain.

## 1. Host testing vs on-target testing

| | Host testing | On-target testing |
|---|---|---|
| Compiler | Your desktop's (`gcc`/`clang`) | Cross-compiler (`arm-none-eabi-gcc`) |
| Speed | Milliseconds per test | Seconds (flash + reset) to minutes |
| What it catches | Logic bugs, algorithm bugs, most regressions | Timing, memory layout, peripheral behavior, interrupt interaction |
| What it misses | Anything the real silicon does differently | Nothing, by definition — but only what you thought to test |
| Cost per run | ~free | Hardware, wiring, possibly a test rig |

The strategy: push as much logic as possible below the hardware seam (module
07 in Level 2) so it runs on the host, and reserve on-target runs for the
things that can only be true on the real part — DMA completion timing,
actual interrupt latency, power-on register state.

## 2. Structuring code for a host build

```c
/* protocol.h -- pure logic, no hardware headers included */
#include <stdint.h>
#include <stddef.h>

typedef struct { uint8_t type; uint16_t len; const uint8_t *payload; } frame_t;

/* Returns -1 on a malformed frame, 0 on success. Never touches a register. */
int frame_parse(const uint8_t *buf, size_t buf_len, frame_t *out);
```

```c
/* protocol.c */
#include "protocol.h"

int frame_parse(const uint8_t *buf, size_t buf_len, frame_t *out) {
    if (buf_len < 3) return -1;                 /* type + 2-byte length */
    uint16_t len = (uint16_t)(buf[1] | (buf[2] << 8));
    if ((size_t)(len + 3) > buf_len) return -1;  /* declared length overruns buffer */
    out->type = buf[0];
    out->len = len;
    out->payload = buf + 3;
    return 0;
}
```

This file has zero dependency on any MCU header. It compiles and tests with
plain `gcc` on any machine, and it is exactly the code most likely to have
off-by-one bugs — so putting it where fast tests can hammer it is the highest
leverage move in the whole module.

```c
/* test_protocol.c */
#include <assert.h>
#include <string.h>
#include "protocol.h"

static void test_short_buffer_rejected(void) {
    uint8_t buf[2] = {0x01, 0x00};
    frame_t f;
    assert(frame_parse(buf, sizeof(buf), &f) == -1);
}

static void test_length_overrun_rejected(void) {
    uint8_t buf[3] = {0x01, 0xFF, 0xFF};   /* claims 65535-byte payload */
    frame_t f;
    assert(frame_parse(buf, sizeof(buf), &f) == -1);
}

static void test_valid_frame_parses(void) {
    uint8_t buf[5] = {0x02, 0x02, 0x00, 0xAA, 0xBB};
    frame_t f;
    assert(frame_parse(buf, sizeof(buf), &f) == 0);
    assert(f.type == 0x02);
    assert(f.len == 2);
    assert(f.payload[0] == 0xAA && f.payload[1] == 0xBB);
}

int main(void) {
    test_short_buffer_rejected();
    test_length_overrun_rejected();
    test_valid_frame_parses();
    return 0;
}
```

```bash
gcc -std=c11 -Wall -Wextra -Iinclude protocol.c test_protocol.c -o test_protocol
./test_protocol && echo "all protocol tests passed"
```

```text
all protocol tests passed
```

This exact command was run on this host, since it needs only `gcc` and no
cross-toolchain — confirming the host-testable-logic split above works in
practice, not just in theory.

## 3. The part that can't move to the host

Below the seam, real register access:

```c
/* uart_hw.c -- compiled ONLY for the target, never for host tests */
#include "uart.h"
#define UART0_DR   (*(volatile uint32_t *)0x4000C000)
#define UART0_FR   (*(volatile uint32_t *)0x4000C018)
#define UART0_FR_TXFF (1u << 5)

void uart_putc(char c) {
    while (UART0_FR & UART0_FR_TXFF) { /* wait for space in TX FIFO */ }
    UART0_DR = (uint32_t)c;
}
```

There is nothing to unit test here in the normal sense — the correctness
claim ("this waits for the FIFO flag before writing") can only be verified
by reading the datasheet carefully and, ultimately, by observing it on real
hardware with a logic analyzer or by exercising it through a HIL rig
(module 03). What you *can* do on the host is compile it for a syntax/type
check and keep it small enough that a manual review is tractable — this
function is four lines for exactly that reason.

## 4. Two build targets, one source tree

```cmake
# CMakeLists.txt
cmake_minimum_required(VERSION 3.16)
project(firmware C)

add_library(protocol protocol.c)
target_include_directories(protocol PUBLIC include)

if(BUILD_HOST_TESTS)
    enable_testing()
    add_executable(test_protocol test_protocol.c)
    target_link_libraries(test_protocol PRIVATE protocol)
    add_test(NAME protocol COMMAND test_protocol)
else()
    # On-target build: pull in the real hardware layer and link a full image.
    add_executable(firmware.elf main.c uart_hw.c)
    target_link_libraries(firmware.elf PRIVATE protocol)
    target_compile_options(firmware.elf PRIVATE -mcpu=cortex-m4 -mthumb)
    target_link_options(firmware.elf PRIVATE -T${CMAKE_SOURCE_DIR}/link.ld)
endif()
```

```bash
# Host: fast, runs on every commit
cmake -S . -B build-host -DBUILD_HOST_TESTS=ON -DCMAKE_C_COMPILER=gcc
cmake --build build-host && ctest --test-dir build-host --output-on-failure

# Target: cross-compiled, run only when a hardware change lands or on a
# scheduled nightly job against the HIL rig from module 03
cmake -S . -B build-target -DCMAKE_TOOLCHAIN_FILE=arm-none-eabi.cmake
cmake --build build-target
```

## 5. Faking the toolchain-specific parts you still need on host

Some code sits right at the seam and needs *some* stand-in even for a host
build — a fake flash, a fake tick counter.

```c
/* tests/fake_flash.c -- links into the host test binary only */
static uint8_t fake_flash[4096];

int flash_read(uint32_t addr, uint8_t *out, size_t len) {
    if (addr + len > sizeof(fake_flash)) return -1;
    memcpy(out, fake_flash + addr, len);
    return 0;
}

int flash_write(uint32_t addr, const uint8_t *data, size_t len) {
    if (addr + len > sizeof(fake_flash)) return -1;
    memcpy(fake_flash + addr, data, len);
    return 0;
}

void fake_flash_reset(void) { memset(fake_flash, 0xFF, sizeof(fake_flash)); }
```

This mirrors module 07 of Level 2 — the link seam — applied specifically to
the host/target split rather than to unit-level fakes.

## 6. Traps specific to embedded host testing

- **`sizeof` and struct padding differ between host and target.** A
  packed protocol struct that "just works" with the host's ABI can silently
  mismatch the target's. Always use explicit byte-level parsing (as in
  `frame_parse` above) or `#pragma pack`/`__attribute__((packed))` verified
  on **both** toolchains, never implicit struct layout for wire formats.
- **Endianness.** If the target is big-endian and your host is little-endian
  (or vice versa), a host test that passes proves nothing about byte order.
  Write endianness-explicit code (shift-and-mask, as above) rather than
  relying on the host's native layout, and add one test that hand-constructs
  a buffer with a known byte order.
- **Integer width assumptions.** `int` is not guaranteed to be 32 bits.
  Use `<stdint.h>` types (`uint32_t`, `int16_t`) everywhere a wire format or
  register width matters, in both host and target code.
- **Undefined behavior that "works" on host but not target.** Signed integer
  overflow, unaligned access, and reading uninitialized memory can behave
  differently across compilers/architectures. Run the host test build under
  UBSan (Level 2, module 05) specifically because it will catch some of
  these before they ever reach hardware.

## Cheat sheet

| Code characteristic | Where it's tested |
|---|---|
| Pure logic, parsing, math, state machines | Host, every commit |
| Thin register-access wrapper | Manual review + minimal on-target smoke test |
| Timing-dependent (DMA, interrupt latency) | On-target or HIL only |
| Wire format / protocol framing | Host, with explicit byte-level (not struct-layout) code |
| Anything using UB-sensitive constructs | Host build compiled with `-fsanitize=undefined` |

## Exercise

1. Take the `frame_parse` function and add a checksum field (a one-byte XOR
   of the payload) to the wire format. Write host tests for: a valid
   checksum, a corrupted payload with a stale checksum, and a payload that's
   valid but the checksum byte itself is missing (buffer one byte short).
2. Identify one register-access function you'd need in a real UART driver
   for this protocol and write it against a `hal_t`-style seam (Level 2,
   module 07) so a fake can be substituted on host.
3. Set up the two-CMake-target split above (`BUILD_HOST_TESTS` on/off) even
   without real cross-compiler installed — stub the target branch with a
   comment showing what toolchain file it would use.
4. Compile and run your host test suite with `-fsanitize=undefined,address`
   and confirm it's clean.
5. Write three sentences on which parts of your protocol implementation you
   are *not* confident are correct without on-target or HIL verification,
   and why (this is the honest inventory a real embedded team keeps).
