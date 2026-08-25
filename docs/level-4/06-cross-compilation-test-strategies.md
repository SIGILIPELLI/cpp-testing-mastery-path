# 06 · Cross-Compilation Test Strategies

Level 3 module 02 split embedded code into host-testable logic and
target-only hardware access. This module goes one level deeper: even the
host-testable logic can behave differently once cross-compiled for the
actual target — different endianness, integer widths, ABI, and compiler
quirks. A test suite that only ever runs on the host compiler has not
verified the binary that actually ships.

!!! note "Environment note"
    No cross-compiler (`arm-none-eabi-gcc`, or any non-native target
    toolchain) is installed on this machine, so nothing cross-compiled was
    actually built or run here — every command and its "expected output"
    in this module is a template, marked as such. The one native-host
    comparison in section 3 (running the same test compiled two different
    ways) uses only tools already confirmed working earlier in this path.

## 1. What actually changes when you cross-compile

| Difference | Host (e.g. x86_64 macOS/Linux) | Common embedded target |
|---|---|---|
| Endianness | Little-endian | Often little-endian too, but not guaranteed (some ARM configs, PowerPC) |
| `int`/`long` width | Usually 32/64-bit | Can be 16-bit `int` on small MCUs |
| Floating point | Hardware FPU, IEEE 754 | Sometimes software-emulated, sometimes single-precision only |
| Alignment requirements | Often forgiving of unaligned access | Frequently faults or is drastically slower on unaligned access |
| Standard library | Full glibc/libc++ | Often a minimal libc (newlib, picolibc) with gaps |
| Undefined behavior | Compiler-specific, but observable | Can differ silently — code "working" on host, broken on target |

None of these differences show up by running the same test suite compiled
for the host, over and over — they only appear when the *same source* is
compiled for the target and exercised there, or symbolically reasoned about
in a way that's sensitive to them.

## 2. Testing on an emulator/QEMU as a middle ground

Full HIL (Level 3, module 03) is the ground truth, but it's scarce and
slow. An instruction-set emulator (module 07 covers this in depth) runs the
*actual cross-compiled binary*, catching ABI/width/alignment issues,
without needing physical hardware for every test run.

```bash
# Cross-compile for a Cortex-M target
arm-none-eabi-gcc -mcpu=cortex-m4 -mthumb -std=c11 \
    -Iinclude src/protocol.c tests/test_protocol.c -o test_protocol.elf

# Run under QEMU's system emulation instead of real hardware
qemu-system-arm -M lm3s6965evb -nographic -kernel test_protocol.elf
```

This is the pattern module 07 develops fully; the point here is where it
sits in the overall strategy — a second tier between "host, fast, but
doesn't reflect the target" and "real hardware, accurate, but scarce."

## 3. What you CAN check on the host: compiler-flag-driven divergence checks

Even without a cross-compiler, deliberately compiling the same source
multiple ways on the host surfaces some of the same class of bug —
specifically, code that silently depends on assumptions a stricter build
would violate.

```c
/* width_assumption.c -- a bug that "works" under one assumption */
#include <stdio.h>

/* BUG: assumes int is exactly 32 bits -- true on this host, NOT
   guaranteed, and false on some 16-bit-int embedded targets. */
unsigned pack_two_fields(unsigned short high, unsigned short low) {
    return ((unsigned)high << 16) | low;   /* if `unsigned` were 16-bit,
                                               this shift would be UB
                                               (shifting by the full
                                               width or more) */
}

int main(void) {
    printf("sizeof(unsigned) = %zu bytes (%zu bits)\n",
           sizeof(unsigned), sizeof(unsigned) * 8);
    printf("result = %u\n", pack_two_fields(0x1234, 0x5678));
    return 0;
}
```

```bash
gcc -std=c11 -Wall -Wextra width_assumption.c -o width_assumption
./width_assumption
```

```text
sizeof(unsigned) = 4 bytes (32 bits)
result = 305419896
```

This is real, run output confirming the assumption holds on *this* host —
which is exactly the trap: it tells you nothing about whether it holds on
your actual embedded target, where `unsigned` may be 16 bits. The fix,
independent of any cross-compiler, is to stop depending on the assumption:

```c
#include <stdint.h>
uint32_t pack_two_fields(uint16_t high, uint16_t low) {
    return ((uint32_t)high << 16) | low;   /* explicit width, correct on
                                               every platform */
}
```

`<stdint.h>` types remove the ambiguity entirely — this single change is
correct regardless of what `int` happens to be on any given target, and
it's checkable and fixable on the host, with no cross-compiler required.

## 4. Static analysis flags to catch cross-platform assumptions early

Several `-W` flags catch exactly this class of latent, platform-dependent
bug during a normal host build, without needing the target compiler at all:

```bash
gcc -std=c11 -Wall -Wextra -Wconversion -Wsign-conversion \
    -Wpadded -Wpacked width_assumption.c -o /dev/null -c
```

| Flag | Catches |
|---|---|
| `-Wconversion` | Implicit narrowing that could lose data on a platform with different widths |
| `-Wsign-conversion` | Implicit signed/unsigned conversions, a frequent source of platform-dependent bugs |
| `-Wpadded` | Structs padded differently than expected — a real risk for wire formats read on multiple platforms |
| `-Wcast-align` | Casts that could produce misaligned pointers, fine on x86, a hard fault on many embedded targets |

## 5. Endianness: testable on the host, without a cross-compiler

Unlike integer width, endianness bugs in wire-format code (Level 3, module
02) can be tested for correctness on the host *if* the code is written
byte-explicitly, and the test itself constructs known-endianness input —
exactly the technique that module recommended, verified here to actually
matter:

```c
/* This function is correct on ANY platform BECAUSE it never relies on
   the host's native integer layout -- it does explicit shift-and-mask. */
uint32_t read_be32(const uint8_t *buf) {
    return ((uint32_t)buf[0] << 24) | ((uint32_t)buf[1] << 16) |
           ((uint32_t)buf[2] << 8)  |  (uint32_t)buf[3];
}
```

```c
#include <assert.h>
static void test_read_be32_known_bytes(void) {
    uint8_t buf[4] = {0x12, 0x34, 0x56, 0x78};
    assert(read_be32(buf) == 0x12345678u);
}
```

This test passes identically on a little-endian host and a big-endian
target *because* the function never depends on host byte order in the
first place — the test doesn't prove endianness-independence by running on
multiple platforms, it proves it by construction, which is why Level 3
module 02 insisted on this style for wire formats specifically.

## 6. Building a cross-compilation CI matrix

The natural endpoint, extending Level 3 module 06's compiler matrix
concept across architectures rather than just compilers:

```yaml
  cross-compile-check:
    strategy:
      matrix:
        target: [x86_64-linux, arm-none-eabi, riscv32-unknown-elf]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: sudo apt-get install -y gcc-${{ matrix.target }} || true
      - run: |
          if [ "${{ matrix.target }}" = "x86_64-linux" ]; then
            gcc -Wall -Wextra -Wconversion -c src/*.c
          else
            ${{ matrix.target }}-gcc -Wall -Wextra -Wconversion -c src/*.c
          fi
```

Even a compile-only check (no execution) across multiple target triples
catches a meaningful class of bug — a `-Wconversion` warning that's silent
on the host toolchain's default widths but fires under the target's
narrower `int`.

## Cheat sheet

| Risk | Host-only mitigation | Full mitigation |
|---|---|---|
| Integer width assumptions | `<stdint.h>` types everywhere; `-Wconversion` | Cross-compile + run under QEMU/HIL |
| Endianness | Byte-explicit code (never rely on struct layout); host tests with known-endianness input | Same code, verified again under QEMU/HIL for defense in depth |
| Struct padding differences | `-Wpadded`, explicit byte-level (de)serialization | Cross-compiled binary inspection (`nm`, `objdump`) |
| Alignment faults | `-Wcast-align` | QEMU/HIL — genuinely needs the target's actual fault behavior |
| Minimal-libc gaps | Review target libc docs for missing functions | Attempt an actual cross-compile; link errors surface gaps immediately |

## Exercise

1. Take `pack_two_fields` from section 3, compile and run it as shown, then
   rewrite it using `<stdint.h>` types and re-verify the same output.
2. Compile your own project's wire-format code (or `frame_parse` from
   Level 3) with `-Wconversion -Wsign-conversion` and record every warning
   — for each, decide if it's a real cross-platform risk or a false
   positive, and justify the call.
3. Write `read_be32` and its test as shown in section 5, run it on this
   host, and explain in two sentences why passing here is meaningful
   evidence about behavior on a big-endian target, unlike most other tests
   in this path.
4. Sketch the CI matrix from section 6 for your own project's actual
   target triples, and identify which of your source files would need the
   fewest changes to even compile-check cleanly across all of them.
5. Write two sentences on one piece of your own code (real or
   hypothetical) that implicitly assumes something about `int` width,
   struct layout, or endianness that you are not currently confident holds
   on every platform you ship to.
