# 04 · Memory Error Detection

A passing test suite proves your assertions held. It does not prove the program
stayed inside its own memory. C and C++ will happily read a freed pointer, write
one byte past a buffer, and return the right answer anyway — until the day the
heap layout shifts and the same code corrupts something important. Memory error
detectors close that gap by instrumenting every access.

## 1. What goes wrong

| Error | What happens | Typical symptom |
|---|---|---|
| Heap buffer overflow | Read/write past an allocation | Corrupted neighbouring data |
| Stack buffer overflow | Past a local array | Smashed return address, wild jump |
| Use-after-free | Access to a freed block | Works until the allocator reuses it |
| Double free | `free()` twice | Allocator abort, or silent corruption |
| Invalid free | `free()` a non-heap pointer | Immediate abort |
| Memory leak | Never freed | Growth over hours; fine in a short test |
| Uninitialised read | Reading a variable never written | Nondeterministic behaviour |

Every one of these is undefined behaviour. None of them reliably crashes.

## 2. AddressSanitizer — the default choice

ASan is built into Clang and GCC. Add one flag to compile *and* link:

```bash
clang -fsanitize=address -fno-omit-frame-pointer -g -O1 prog.c -o prog
./prog
```

Here is a real off-by-one — the classic "forgot the NUL byte" bug:

```c
#include <stdlib.h>
#include <string.h>
#include <stdio.h>

char *duplicate_prefix(const char *src, size_t n) {
    char *out = malloc(n);        /* should be n + 1 */
    strncpy(out, src, n);
    out[n] = '\0';                /* writes 1 past the end */
    return out;
}

int main(void) {
    char *p = duplicate_prefix("hello world", 5);
    printf("%s\n", p);
    free(p);
    return 0;
}
```

```bash
clang -fsanitize=address -g -O0 obug.c -o obug && ./obug
```

```
=================================================================
==6214==ERROR: AddressSanitizer: heap-buffer-overflow on address 0x6020000000d5 at pc 0x0001002c4874 bp 0x00016fb3aa80 sp 0x00016fb3aa78
WRITE of size 1 at 0x6020000000d5 thread T0
    #0 0x0001002c4870 in duplicate_prefix obug.c:7
    #1 0x0001002c48bc in main obug.c:10

0x6020000000d5 is located 0 bytes after 5-byte region [0x6020000000d0,0x6020000000d5)
allocated by thread T0 here:
    #0 0x000100ae5164 in malloc+0x78 (libclang_rt.asan_osx_dynamic.dylib)
    #1 0x0001002c4808 in duplicate_prefix obug.c:5
```

Without ASan this program prints `hello` and exits 0. Every day. Until it
doesn't.

## 3. Reading the report

Four things carry all the information:

1. **The error class** — `heap-buffer-overflow`, `heap-use-after-free`, …
2. **The access** — `WRITE of size 1`. Read or write, and how wide.
3. **The offending stack** — frame `#0` is the exact line.
4. **The region description** — `0 bytes after 5-byte region`, plus *where it
   was allocated*. For a use-after-free you also get where it was freed.

`0 bytes after a 5-byte region` allocated at line 5 and written at line 7 tells
you the allocation is one byte too small. That is the whole diagnosis.

!!! warning "`-g` and `-fno-omit-frame-pointer` are not optional"
    Without them you get hex addresses instead of `obug.c:7`. Optimised builds
    (`-O2`) inline frames away; use `-O1` or `-O0` for sanitizer builds. The
    performance you save is worth nothing if the report is unreadable.

## 4. Tuning with `ASAN_OPTIONS`

```bash
ASAN_OPTIONS=detect_leaks=1 ./prog                 # leak checking (see §5)
ASAN_OPTIONS=abort_on_error=1 ./prog               # SIGABRT for a core dump
ASAN_OPTIONS=halt_on_error=0 ./prog                # keep going, report all
ASAN_OPTIONS=detect_stack_use_after_return=1 ./prog
ASAN_OPTIONS=log_path=/tmp/asan ./prog             # one file per process
ASAN_OPTIONS=strict_string_checks=1 ./prog         # stricter str* checking
```

In CI, set `halt_on_error=0` so one run surfaces every problem, and
`log_path` so the reports survive as build artifacts.

## 5. Leak detection — and a platform caveat

LeakSanitizer ships inside ASan. On Linux x86-64 it is **on by default**; a
program that exits with unreleased allocations prints a report and exits
non-zero:

```
=================================================================
==1==ERROR: LeakSanitizer: detected memory leaks

Direct leak of 32 byte(s) in 1 object(s) allocated from:
    #0 0x4a4c5d in malloc
    #1 0x4f1a2b in leaky leak.c:20

SUMMARY: AddressSanitizer: 32 byte(s) leaked in 1 allocation(s).
```

On Apple Silicon macOS it is not available. Asking for it explicitly says so:

```bash
ASAN_OPTIONS=detect_leaks=1 ./leak
```

```
==5300==AddressSanitizer: detect_leaks is not supported on this platform.
```

!!! warning "Leak coverage is platform-dependent"
    The above is real output from macOS on arm64. If your team develops on Macs,
    **leaks will only be caught in CI**, and only if CI runs Linux. Make that an
    explicit job rather than an accident. macOS developers can fall back on the
    `leaks(1)` command or CMocka's instrumented allocator (Module 3, §6) for
    local checking.

## 6. Valgrind Memcheck

Valgrind needs no recompilation — it interprets the binary — which makes it the
tool of choice for third-party or already-shipped executables, and the only one
that detects **uninitialised reads** well.

```bash
valgrind --leak-check=full --show-leak-kinds=all --track-origins=yes \
         --error-exitcode=1 ./prog
```

```
==3412== Invalid write of size 1
==3412==    at 0x1091AE: duplicate_prefix (obug.c:7)
==3412==    by 0x1091E9: main (obug.c:10)
==3412==  Address 0x4a4d045 is 0 bytes after a block of size 5 alloc'd
==3412==    at 0x484182F: malloc (vg_replace_malloc.c:431)
==3412==    by 0x109193: duplicate_prefix (obug.c:5)
==3412==
==3412== LEAK SUMMARY:
==3412==    definitely lost: 0 bytes in 0 blocks
```

!!! info "Not verified on this platform"
    Every other command in this module was run on the machine these notes were
    written on. Valgrind was not: it has no working macOS/arm64 port, so the
    output above is reference material from Linux rather than a local capture.
    Run it yourself on a Linux box or container before relying on the exact
    wording. `--error-exitcode=1` is the flag that makes it usable in CI —
    without it Valgrind reports errors and still exits 0.

## 7. Choosing between them

| | AddressSanitizer | Valgrind Memcheck |
|---|---|---|
| Needs recompilation | Yes (`-fsanitize=address`) | No |
| Slowdown | ~2× | 20–50× |
| Memory overhead | ~3× | ~2× |
| Heap overflow | Yes | Yes |
| **Stack** overflow | Yes | No |
| Global overflow | Yes | No |
| Use-after-free | Yes | Yes |
| Use-after-return | Optional flag | No |
| Uninitialised reads | No (that's MSan) | **Yes** |
| Leaks | Via LSan; platform-dependent | Yes |
| macOS arm64 | ASan yes, LSan no | Unsupported |
| Good for | Every CI run | Deep audits, binaries you can't rebuild |

The practical answer is **both, at different cadences**: ASan on every pull
request because it is cheap, Valgrind nightly because it is thorough. They are
not interchangeable — ASan cannot see uninitialised reads, Valgrind cannot see
stack overflows.

!!! warning "Never combine ASan and Valgrind"
    Running an ASan-instrumented binary under Valgrind produces a flood of false
    positives from ASan's own shadow-memory bookkeeping. Build two binaries.

## 8. Wiring sanitizers into the suite

Sanitizers find bugs only on code paths your tests execute. A sanitizer build is
therefore a **test-suite multiplier**, not a replacement for tests — coverage
(Module 6) tells you how much of the program the multiplier actually reached.

```cmake
option(ENABLE_SANITIZERS "Build with ASan + UBSan" OFF)
if(ENABLE_SANITIZERS)
  add_compile_options(-fsanitize=address,undefined -fno-omit-frame-pointer -g -O1)
  add_link_options(-fsanitize=address,undefined)
endif()
```

```bash
cmake -S . -B build-asan -DENABLE_SANITIZERS=ON
cmake --build build-asan
ctest --test-dir build-asan --output-on-failure
```

Keep it a **separate build directory**. Sanitizer binaries are slower and have
different ABI characteristics; you want the normal build for day-to-day work and
the instrumented one on demand and in CI.

## Exercise

Work from a deliberately broken string library.

1. **Write `strlib.c`** with four functions, each carrying exactly one of these
   bugs: a heap overflow (allocate one byte short), a use-after-free (free then
   read), a double free, and a leak (early return on the error path).

2. **Write a GoogleTest or CMocka suite** that exercises all four. Build it
   *without* sanitizers and record the result — expect most of them to pass.
   Write two sentences on what that says about assertion-based testing alone.

3. **Rebuild with ASan** and capture each report. For every one, name the four
   parts from section 3 (class, access, stack, region) and state the fix from
   the report alone, without reading your source.

4. **Test the leak boundary.** Run the leak case with
   `ASAN_OPTIONS=detect_leaks=1`. Record what your platform does. If it reports
   `not supported`, find a second way to catch it — a Linux container, `leaks(1)`,
   or CMocka's `UNIT_TESTING` allocator — and document which one you'd put in CI.

5. **Add a stack overflow** (write past a local `char buf[8]`). Confirm ASan
   catches it, then reason about why Valgrind would not.

6. **Add the CMake option** from section 8 to your project. Verify that
   `ctest --test-dir build-asan` runs green after all four bugs are fixed, and
   that reintroducing any one of them turns it red.
