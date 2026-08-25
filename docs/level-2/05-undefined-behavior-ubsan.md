# 05 · Undefined Behavior & UBSan

Undefined behaviour is the part of C and C++ where the standard stops making
promises. It is not "the program crashes" and not "you get a garbage value" —
it is "the compiler may assume this never happens and optimise accordingly."
That is why UB bugs pass every test at `-O0`, ship at `-O2`, and fail in
production. UndefinedBehaviorSanitizer turns the invisible into a runtime
diagnostic.

## 1. Why UB is worse than a wrong answer

Consider the classic overflow check:

```c
int add_checked(int a, int b) {
    int sum = a + b;
    if (sum < a) return -1;    /* "overflow happened" */
    return sum;
}
```

Signed overflow is UB, so the compiler is entitled to assume `a + b` never
overflows. If it never overflows, `sum < a` implies `b < 0`, and with `b >= 0`
the check is provably false — so the optimiser **deletes it**. The test at `-O0`
passes because the arithmetic wraps in practice. The release build silently
loses the guard.

This is the shape of every serious UB bug: the code is not merely wrong, it
means something different at different optimisation levels.

## 2. UBSan in one flag

```bash
clang -fsanitize=undefined -fno-omit-frame-pointer -g prog.c -o prog
./prog
```

A real signed-overflow catch:

```c
#include <stdio.h>
#include <limits.h>

int main(void) {
    int x = INT_MAX;
    int y = x + 1;              /* signed overflow: UB */
    printf("%d\n", y);
    return 0;
}
```

```bash
clang -fsanitize=undefined -g ub.c -o ub && ./ub
```

```
ub.c:3:38: runtime error: signed integer overflow: 2147483647 + 1 cannot be represented in type 'int'
SUMMARY: UndefinedBehaviorSanitizer: undefined-behavior ub.c:3:38
```

And an out-of-range shift:

```c
int s = 1;
int r = s << 31;        /* 1 << 31 overflows int */
```

```
ub2.c:2:32: runtime error: left shift of 1 by 31 places cannot be represented in type 'int'
SUMMARY: UndefinedBehaviorSanitizer: undefined-behavior ub2.c:2:32
-2147483648
```

Note the last line. The program **kept running** and printed a value.

## 3. The recovery trap

By default most UBSan checks are *recoverable*: they print a diagnostic and
continue. In a test suite that is exactly wrong — the diagnostic scrolls past,
the assertions still pass, and the suite exits 0. CI stays green over a real
defect.

```bash
# Diagnostic only; exit code unchanged. Tests still "pass".
clang -fsanitize=undefined prog.c -o prog

# Abort on the first violation. Tests fail.
clang -fsanitize=undefined -fno-sanitize-recover=all prog.c -o prog
```

!!! warning "Always pair UBSan with `-fno-sanitize-recover=all` in tests"
    This is the most commonly missed UBSan flag, and skipping it turns the
    sanitizer into a log-noise generator. If you prefer a report of *every*
    violation rather than just the first, keep recovery on but add
    `UBSAN_OPTIONS=print_stacktrace=1:halt_on_error=1`, or grep the log and fail
    the build yourself. What you must not do is let a violation exit 0.

Useful runtime options:

```bash
UBSAN_OPTIONS=print_stacktrace=1 ./prog      # stack traces, not just one line
UBSAN_OPTIONS=halt_on_error=1 ./prog
UBSAN_OPTIONS=log_path=/tmp/ubsan ./prog
```

`print_stacktrace=1` needs `-g` and, on some platforms, `llvm-symbolizer` on
`PATH`. Without it you get the source line of the violation but not the call
chain that reached it.

## 4. What UBSan checks

`-fsanitize=undefined` is a group. These are the members worth knowing:

| Check | Catches | Flag |
|---|---|---|
| Signed overflow | `INT_MAX + 1` | `signed-integer-overflow` |
| Shift | `x << 33`, negative shift counts | `shift` |
| Division | `INT_MIN / -1`, `x / 0` | `integer-divide-by-zero` |
| Null | Dereference / member call on `nullptr` | `null` |
| Alignment | Misaligned load or store | `alignment` |
| Array bounds | Index past a **fixed-size** array | `bounds` |
| Object size | Access past a known-size object | `object-size` |
| Enum | Value outside the enumerator range | `enum` |
| Bool | A `bool` holding neither 0 nor 1 | `bool` |
| Float cast | `(int)1e20` | `float-cast-overflow` |
| Return | Falling off a non-`void` function | `return` |
| VLA bound | Negative or zero VLA size | `vla-bound` |
| Vptr (C++) | Wrong-type dynamic dispatch | `vptr` (needs RTTI) |
| Unreachable | Reaching `__builtin_unreachable()` | `unreachable` |

Two are **not** in the default group and must be requested:

```bash
-fsanitize=undefined,integer,implicit-conversion
```

`integer` includes **unsigned** overflow, which is well-defined by the standard
but is a bug in most programs that hit it. `implicit-conversion` catches silent
narrowing. Both are noisy on legacy code; turn them on for new modules.

## 5. Selecting and suppressing

```bash
clang -fsanitize=undefined -fno-sanitize=alignment prog.c -o prog
clang -fsanitize=signed-integer-overflow,shift,bounds prog.c -o prog
```

For a genuinely intentional violation — a hash function that relies on wrapping —
annotate the function rather than disabling the check globally:

```c
__attribute__((no_sanitize("signed-integer-overflow")))
uint32_t legacy_hash(const char *s) { /* ... */ }
```

Better still, make the wrap explicit by using unsigned arithmetic, which is
defined to wrap. Then no suppression is needed and the intent is in the code.

## 6. Combining sanitizers

ASan and UBSan compose, and together they are the standard instrumented build:

```bash
clang -fsanitize=address,undefined -fno-sanitize-recover=all \
      -fno-omit-frame-pointer -g -O1 prog.c -o prog
```

ThreadSanitizer (`-fsanitize=thread`) and MemorySanitizer (`-fsanitize=memory`)
are each **incompatible with ASan** and with each other — separate build
directories, separate CI jobs.

```cmake
option(ENABLE_SANITIZERS "Build with ASan + UBSan" OFF)
if(ENABLE_SANITIZERS)
  add_compile_options(-fsanitize=address,undefined -fno-sanitize-recover=all
                      -fno-omit-frame-pointer -g -O1)
  add_link_options(-fsanitize=address,undefined)
endif()
```

```bash
cmake -S . -B build-asan -DENABLE_SANITIZERS=ON
cmake --build build-asan
ctest --test-dir build-asan --output-on-failure
```

## 7. Traps

| Trap | Consequence | Fix |
|---|---|---|
| Omitting `-fno-sanitize-recover=all` | Violations logged, suite exits 0 | Add the flag in test builds |
| Sanitizing at `-O0` only | Optimiser-dependent UB never appears | Also run an `-O2` sanitizer build |
| Forgetting the flag at link time | Link errors or silently no instrumentation | Pass `-fsanitize=` to compile **and** link |
| Expecting `bounds` to catch heap overruns | It only handles fixed-size arrays | ASan for heap (Module 4) |
| Combining `-fsanitize=thread,address` | Refused or broken at runtime | Separate builds |
| No `-g` | Addresses instead of file:line | Always `-g` |
| Assuming unsigned overflow is caught | It is not UB, so not in the default set | Add `-fsanitize=integer` |
| Suppressing globally for one hot function | Whole-program blind spot | `__attribute__((no_sanitize(...)))` |

## 8. UB you will actually meet

```c
int  a[4];  int i = 5;  a[i];          /* bounds */
int  x = INT_MIN / -1;                 /* overflow, not just divide */
char *p = NULL; *p;                    /* null */
int  n = 33; unsigned u = 1u << n;     /* shift >= width */
double d = 1e20; int k = (int) d;      /* float-cast-overflow */
bool b; memcpy(&b, &junk, 1);          /* bool holding 2 */
int f(int x) { if (x) return 1; }      /* missing return */
```

The last one is especially insidious: GCC and Clang both warn about it, and both
also *optimise on the assumption it never happens*, so ignoring the warning can
change control flow elsewhere in the function.

## Exercise

Build a UB museum and then close it down.

1. **Write `ub_demo.c`** containing six functions, one per UB category: signed
   overflow, out-of-range shift, `INT_MIN / -1`, a fixed-array out-of-bounds
   read, a float-to-int cast overflow, and a non-`void` function that falls off
   the end. Each should be reachable from a test.

2. **Establish the baseline.** Build with plain `clang -O0`, run, and record the
   output. Then build with `-O2` and record it again. Note every function whose
   observable behaviour differs between the two, and explain one of them in
   terms of what the optimiser was permitted to assume.

3. **Add UBSan** and capture the diagnostic for all six. Confirm the reported
   file:line matches the function you expected.

4. **Prove the recovery trap.** Wrap the six calls in a test binary that returns
   0. Build with `-fsanitize=undefined` alone and record the exit code
   (`echo $?`). Rebuild with `-fno-sanitize-recover=all` and record it again.
   Write two sentences on why the first configuration is dangerous in CI.

5. **Extend the check set.** Add a function with unsigned wraparound and one
   with an implicit narrowing conversion. Show that the default group misses
   both, and that `-fsanitize=undefined,integer,implicit-conversion` catches
   them.

6. **Fix, don't suppress.** Rewrite all six functions to be well-defined —
   overflow-checked addition, unsigned shift arithmetic, an explicit
   `INT_MIN`/`-1` guard, a bounds check, a range check before the cast, and a
   final `return`. Confirm the UBSan build is silent and exits 0. Where you
   genuinely wanted wrapping, use unsigned types rather than
   `__attribute__((no_sanitize))`, and say in a comment why.
