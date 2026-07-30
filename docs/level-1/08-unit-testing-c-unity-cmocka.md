# 08 · Unit Testing C with Unity/CMocka

GoogleTest is C++ — it uses classes, templates, and exceptions. Plenty of code
you'll test is plain C, sometimes compiled by a C-only toolchain for a
microcontroller with 32 KB of RAM. This module covers the two dominant C test
frameworks: **Unity** (tiny, embedded-friendly, zero dependencies) and
**CMocka** (fuller featured, with mocking and leak detection built in).

## 1. Choosing a framework

| | Unity | CMocka | GoogleTest |
|---|---|---|---|
| Language | C | C | C++ |
| Size | ~3 source files, a few KB | Small library | Large |
| Dependencies | None | libc (+ setjmp) | C++ runtime, STL |
| Runs on bare metal | **Yes** | Usually | Rarely |
| Built-in mocking | Via CMock (separate) | **Yes** | Via GoogleMock |
| Leak detection | No | **Yes** | No |
| Best for | Firmware, MCU targets, minimal deps | Host-side C testing, C with mocks | C++ code |

A common real-world split: **Unity on the target, CMocka or GoogleTest on the
host.** If your C code is portable enough to compile for your PC, you can also
just test it with GoogleTest — C++ can call C directly, and that's covered in
section 5.

## 2. Unity — the minimal C framework

Unity is three files: `unity.c`, `unity.h`, `unity_internals.h`. You can
literally copy them into your project.

```bash
git clone --depth 1 https://github.com/ThrowTheSwitch/Unity.git external/Unity
```

### The code under test

`src/temperature.h`:

```c
#ifndef TEMPERATURE_H
#define TEMPERATURE_H

/* Parses a decimal integer temperature in the inclusive range -40..125.
   On success writes the value to *out_celsius and returns 0.
   Returns -1 if text is NULL, out_celsius is NULL, or text is not a
   valid decimal integer. Returns -2 if the value is out of range.
   On any error, *out_celsius is left unmodified. */
int parse_temperature(const char *text, int *out_celsius);

#endif /* TEMPERATURE_H */
```

`src/temperature.c`:

```c
#include "temperature.h"
#include <stdlib.h>
#include <errno.h>
#include <limits.h>

int parse_temperature(const char *text, int *out_celsius) {
    if (text == NULL || out_celsius == NULL) {
        return -1;
    }
    if (text[0] == '\0') {
        return -1;
    }

    char *end = NULL;
    errno = 0;
    long value = strtol(text, &end, 10);   /* strtol, not atoi -- atoi
                                              has no error reporting and
                                              overflow is UB */
    if (errno == ERANGE || *end != '\0') {
        return -1;                          /* trailing junk or out of long */
    }
    if (value < -40 || value > 125) {
        return -2;
    }

    *out_celsius = (int)value;
    return 0;
}
```

!!! tip "`strtol` over `atoi` is itself a testability decision"
    `atoi("99999999999999")` is undefined behaviour and reports nothing.
    `strtol` sets `errno` and tells you where parsing stopped. When you review a
    C codebase as a tester, `atoi`/`atof` calls on untrusted input are a
    reportable finding — they make correct error handling impossible.

### The Unity test file

`tests/test_temperature.c`:

```c
#include "unity.h"
#include "temperature.h"

/* Unity requires these two, even if empty. They run before/after EACH test. */
void setUp(void)    { }
void tearDown(void) { }

static void test_parses_nominal_value(void) {
    int out = 0;
    TEST_ASSERT_EQUAL_INT(0, parse_temperature("25", &out));
    TEST_ASSERT_EQUAL_INT(25, out);
}

static void test_parses_negative_value(void) {
    int out = 0;
    TEST_ASSERT_EQUAL_INT(0, parse_temperature("-15", &out));
    TEST_ASSERT_EQUAL_INT(-15, out);
}

static void test_accepts_lower_boundary(void) {
    int out = 0;
    TEST_ASSERT_EQUAL_INT(0, parse_temperature("-40", &out));
    TEST_ASSERT_EQUAL_INT(-40, out);
}

static void test_accepts_upper_boundary(void) {
    int out = 0;
    TEST_ASSERT_EQUAL_INT(0, parse_temperature("125", &out));
    TEST_ASSERT_EQUAL_INT(125, out);
}

static void test_rejects_below_lower_boundary(void) {
    int out = 0x5A5A5A5A;                       /* sentinel */
    TEST_ASSERT_EQUAL_INT(-2, parse_temperature("-41", &out));
    TEST_ASSERT_EQUAL_INT(0x5A5A5A5A, out);     /* must be unmodified */
}

static void test_rejects_above_upper_boundary(void) {
    int out = 0x5A5A5A5A;
    TEST_ASSERT_EQUAL_INT(-2, parse_temperature("126", &out));
    TEST_ASSERT_EQUAL_INT(0x5A5A5A5A, out);
}

static void test_rejects_non_numeric_text(void) {
    int out = 0x5A5A5A5A;
    TEST_ASSERT_EQUAL_INT(-1, parse_temperature("abc", &out));
    TEST_ASSERT_EQUAL_INT(0x5A5A5A5A, out);
}

static void test_rejects_empty_string(void) {
    int out = 0x5A5A5A5A;
    TEST_ASSERT_EQUAL_INT(-1, parse_temperature("", &out));
    TEST_ASSERT_EQUAL_INT(0x5A5A5A5A, out);
}

static void test_rejects_null_text(void) {
    int out = 0x5A5A5A5A;
    TEST_ASSERT_EQUAL_INT(-1, parse_temperature(NULL, &out));
    TEST_ASSERT_EQUAL_INT(0x5A5A5A5A, out);
}

static void test_rejects_null_output_pointer(void) {
    TEST_ASSERT_EQUAL_INT(-1, parse_temperature("25", NULL));
}

static void test_rejects_trailing_junk(void) {
    int out = 0x5A5A5A5A;
    TEST_ASSERT_EQUAL_INT(-1, parse_temperature("25x", &out));
    TEST_ASSERT_EQUAL_INT(0x5A5A5A5A, out);
}

int main(void) {
    UNITY_BEGIN();
    RUN_TEST(test_parses_nominal_value);
    RUN_TEST(test_parses_negative_value);
    RUN_TEST(test_accepts_lower_boundary);
    RUN_TEST(test_accepts_upper_boundary);
    RUN_TEST(test_rejects_below_lower_boundary);
    RUN_TEST(test_rejects_above_upper_boundary);
    RUN_TEST(test_rejects_non_numeric_text);
    RUN_TEST(test_rejects_empty_string);
    RUN_TEST(test_rejects_null_text);
    RUN_TEST(test_rejects_null_output_pointer);
    RUN_TEST(test_rejects_trailing_junk);
    return UNITY_END();     /* returns the failure count -- CTest reads it */
}
```

The manual `RUN_TEST` list is Unity's main ergonomic cost — C has no way to
auto-register functions portably. (Unity ships a Ruby script,
`generate_test_runner.rb`, that generates this `main()` for you; on embedded
projects that's usually wired into the build.)

### Build and run

```bash
gcc -std=c11 -Wall -Wextra -g \
    -I src -I external/Unity/src \
    -o test_temperature \
    tests/test_temperature.c src/temperature.c external/Unity/src/unity.c

./test_temperature
```

```
tests/test_temperature.c:12:test_parses_nominal_value:PASS
tests/test_temperature.c:18:test_parses_negative_value:PASS
tests/test_temperature.c:24:test_accepts_lower_boundary:PASS
tests/test_temperature.c:30:test_accepts_upper_boundary:PASS
tests/test_temperature.c:36:test_rejects_below_lower_boundary:PASS
tests/test_temperature.c:43:test_rejects_above_upper_boundary:PASS
tests/test_temperature.c:50:test_rejects_non_numeric_text:PASS
tests/test_temperature.c:57:test_rejects_empty_string:PASS
tests/test_temperature.c:64:test_rejects_null_text:PASS
tests/test_temperature.c:71:test_rejects_null_output_pointer:PASS
tests/test_temperature.c:76:test_rejects_trailing_junk:PASS

-----------------------
11 Tests 0 Failures 0 Ignored
OK
```

A failure looks like:

```
tests/test_temperature.c:30:test_accepts_upper_boundary:FAIL: Expected 0 Was -2

-----------------------
11 Tests 1 Failures 0 Ignored
FAIL
```

### Unity assertion macros

| Macro | Checks |
|---|---|
| `TEST_ASSERT_TRUE(c)` / `TEST_ASSERT_FALSE(c)` | Boolean |
| `TEST_ASSERT_EQUAL_INT(exp, act)` | Signed integers |
| `TEST_ASSERT_EQUAL_UINT(exp, act)` | Unsigned integers |
| `TEST_ASSERT_EQUAL_INT8/16/32/64` | Fixed-width integers |
| `TEST_ASSERT_EQUAL_HEX8/16/32` | Integers, printed as hex on failure |
| `TEST_ASSERT_EQUAL_PTR(exp, act)` | Pointers |
| `TEST_ASSERT_NULL(p)` / `TEST_ASSERT_NOT_NULL(p)` | Null checks |
| `TEST_ASSERT_EQUAL_STRING(exp, act)` | C strings by content |
| `TEST_ASSERT_EQUAL_MEMORY(exp, act, len)` | Raw byte comparison |
| `TEST_ASSERT_EQUAL_INT_ARRAY(exp, act, n)` | Element-wise array comparison |
| `TEST_ASSERT_FLOAT_WITHIN(delta, exp, act)` | Float with tolerance |
| `TEST_ASSERT_BIT_HIGH(bit, value)` / `_BIT_LOW` | Individual bits — very handy for register testing |
| `TEST_FAIL_MESSAGE("...")` | Unconditional failure |
| `TEST_IGNORE_MESSAGE("...")` | Skip with a reason |

Note the fixed-width and hex variants. In embedded C you're constantly comparing
`uint8_t` register values, and `TEST_ASSERT_EQUAL_HEX8(0xA5, reg)` prints
`Expected 0xA5 Was 0x25` — far more readable than decimal.

!!! warning "Every Unity assertion is an ASSERT, not an EXPECT"
    Unity uses `longjmp` to abort the test function on the first failed
    assertion. There is no "record and continue" mode. So each test reports at
    most one failure, and you should keep tests small and focused.

## 3. CMocka — richer C testing

CMocka adds fixtures, grouped test registration, and — its headline feature —
**built-in memory leak and buffer-overflow detection**.

```bash
# Ubuntu/Debian
sudo apt install libcmocka-dev
# macOS
brew install cmocka
# Fedora
sudo dnf install libcmocka-devel
```

### A CMocka test file

`tests/test_temperature_cmocka.c`:

```c
#include <stdarg.h>     /* these four includes must come BEFORE cmocka.h */
#include <stddef.h>
#include <setjmp.h>
#include <stdint.h>
#include <cmocka.h>

#include "temperature.h"

static void test_parses_nominal_value(void **state) {
    (void)state;                       /* unused -- silences -Wunused-parameter */
    int out = 0;
    assert_int_equal(parse_temperature("25", &out), 0);
    assert_int_equal(out, 25);
}

static void test_accepts_upper_boundary(void **state) {
    (void)state;
    int out = 0;
    assert_int_equal(parse_temperature("125", &out), 0);
    assert_int_equal(out, 125);
}

static void test_rejects_above_upper_boundary(void **state) {
    (void)state;
    int out = 0x5A5A5A5A;
    assert_int_equal(parse_temperature("126", &out), -2);
    assert_int_equal(out, 0x5A5A5A5A);
}

static void test_rejects_null_text(void **state) {
    (void)state;
    int out = 0x5A5A5A5A;
    assert_int_equal(parse_temperature(NULL, &out), -1);
    assert_int_equal(out, 0x5A5A5A5A);
}

int main(void) {
    const struct CMUnitTest tests[] = {
        cmocka_unit_test(test_parses_nominal_value),
        cmocka_unit_test(test_accepts_upper_boundary),
        cmocka_unit_test(test_rejects_above_upper_boundary),
        cmocka_unit_test(test_rejects_null_text),
    };
    return cmocka_run_group_tests(tests, NULL, NULL);
}
```

```bash
gcc -std=c11 -Wall -Wextra -g -I src -o test_temp_cmocka \
    tests/test_temperature_cmocka.c src/temperature.c -lcmocka

./test_temp_cmocka
```

```
[==========] tests: Running 4 test(s).
[ RUN      ] test_parses_nominal_value
[       OK ] test_parses_nominal_value
[ RUN      ] test_accepts_upper_boundary
[       OK ] test_accepts_upper_boundary
[ RUN      ] test_rejects_above_upper_boundary
[       OK ] test_rejects_above_upper_boundary
[ RUN      ] test_rejects_null_text
[       OK ] test_rejects_null_text
[==========] tests: 4 test(s) run.
[  PASSED  ] 4 test(s).
```

!!! warning "The include order is mandatory"
    `cmocka.h` depends on `stdarg.h`, `stddef.h`, `setjmp.h` and `stdint.h`
    being included first and will not compile otherwise. This trips up everyone
    on their first CMocka file.

### CMocka assertion macros

| Macro | Checks |
|---|---|
| `assert_true(c)` / `assert_false(c)` | Boolean |
| `assert_int_equal(a, b)` / `assert_int_not_equal` | Integers |
| `assert_string_equal(a, b)` / `assert_string_not_equal` | C strings |
| `assert_memory_equal(a, b, size)` | Raw bytes |
| `assert_ptr_equal(a, b)` | Pointers |
| `assert_null(p)` / `assert_non_null(p)` | Null checks |
| `assert_in_range(v, min, max)` | Inclusive range |
| `assert_float_equal(a, b, epsilon)` | Float with tolerance |
| `fail()` / `fail_msg("...")` | Unconditional |
| `skip()` | Skip the current test |

### Fixtures with per-test state

CMocka's `void **state` parameter is a slot for per-test data, managed by setup
and teardown functions:

```c
typedef struct {
    RingBuffer *buffer;
} TestState;

static int setup_ring(void **state) {
    TestState *s = malloc(sizeof(TestState));
    if (s == NULL) return -1;              /* non-zero aborts this test */
    s->buffer = ring_create(64);
    *state = s;
    return 0;
}

static int teardown_ring(void **state) {
    TestState *s = *state;
    ring_destroy(s->buffer);
    free(s);
    return 0;
}

static void test_new_buffer_is_empty(void **state) {
    TestState *s = *state;
    assert_int_equal(ring_size(s->buffer), 0);
}

int main(void) {
    const struct CMUnitTest tests[] = {
        cmocka_unit_test_setup_teardown(test_new_buffer_is_empty,
                                        setup_ring, teardown_ring),
    };
    return cmocka_run_group_tests(tests, NULL, NULL);
}
```

### Built-in leak detection

This is the reason to prefer CMocka over Unity for host-side C testing. Replace
`malloc`/`free`/`calloc`/`realloc` with CMocka's instrumented versions:

```c
/* Put these in the code under test, or define them project-wide when
   building for tests. CMocka provides _test_malloc / _test_free. */
#ifdef UNIT_TESTING
extern void* _test_malloc(size_t size, const char* file, int line);
extern void  _test_free(void* ptr, const char* file, int line);
#define malloc(size) _test_malloc(size, __FILE__, __LINE__)
#define free(ptr)    _test_free(ptr, __FILE__, __LINE__)
#endif
```

Now a test whose code under test leaks fails automatically:

```
[  ERROR   ] --- Blocks allocated...
  0x5601a2c0 : src/ring_buffer.c:23
  ERROR: test_create_buffer leaked 1 block(s)
[  FAILED  ] test_create_buffer
```

CMocka also plants guard bytes around each allocation, so writing past the end
of a `malloc`'d block is caught at `free` time. It is not a replacement for
AddressSanitizer (Level 2), but it is free, works everywhere, and catches the
two most common C memory bugs with no extra build configuration.

## 4. Wiring C tests into CMake and CTest

```cmake
cmake_minimum_required(VERSION 3.14)
project(temperature LANGUAGES C)

set(CMAKE_C_STANDARD 11)
set(CMAKE_C_STANDARD_REQUIRED ON)
add_compile_options(-Wall -Wextra)

add_library(temperature src/temperature.c)
target_include_directories(temperature PUBLIC src)

enable_testing()

# --- Unity ---
add_library(unity external/Unity/src/unity.c)
target_include_directories(unity PUBLIC external/Unity/src)

add_executable(test_temperature_unity tests/test_temperature.c)
target_link_libraries(test_temperature_unity PRIVATE temperature unity)
add_test(NAME UnityTemperatureTests COMMAND test_temperature_unity)

# --- CMocka (optional -- only if installed) ---
find_library(CMOCKA_LIB cmocka)
if(CMOCKA_LIB)
    add_executable(test_temperature_cmocka tests/test_temperature_cmocka.c)
    target_link_libraries(test_temperature_cmocka PRIVATE temperature ${CMOCKA_LIB})
    add_test(NAME CMockaTemperatureTests COMMAND test_temperature_cmocka)
else()
    message(STATUS "cmocka not found -- skipping CMocka tests")
endif()
```

```bash
cmake -S . -B build && cmake --build build
ctest --test-dir build --output-on-failure
```

```
    Start 1: UnityTemperatureTests
1/2 Test #1: UnityTemperatureTests ............   Passed    0.00 sec
    Start 2: CMockaTemperatureTests
2/2 Test #2: CMockaTemperatureTests ...........   Passed    0.00 sec

100% tests passed, 0 tests failed out of 2
```

Both frameworks return a non-zero exit code on failure (`UNITY_END()` returns
the failure count; `cmocka_run_group_tests` returns the number of failed tests),
so the CTest contract from Module 6 works unchanged.

## 5. Testing C code from GoogleTest

If your C code compiles for the host, you can skip the C frameworks entirely and
use GoogleTest. The one thing you must get right is `extern "C"`:

```cpp
#include <gtest/gtest.h>

extern "C" {
#include "temperature.h"      // C header -- prevents C++ name mangling
}

TEST(ParseTemperatureTest, AcceptsUpperBoundary) {
    int out = 0;
    EXPECT_EQ(parse_temperature("125", &out), 0);
    EXPECT_EQ(out, 125);
}
```

Without `extern "C"`, the C++ compiler mangles `parse_temperature` into a C++
symbol and the linker cannot find the C implementation — producing an
`undefined reference to parse_temperature(char const*, int*)` error that
confuses everyone the first time.

A more robust habit is to put the guard in the header itself, so callers don't
have to remember:

```c
#ifndef TEMPERATURE_H
#define TEMPERATURE_H

#ifdef __cplusplus
extern "C" {
#endif

int parse_temperature(const char *text, int *out_celsius);

#ifdef __cplusplus
}
#endif

#endif /* TEMPERATURE_H */
```

| Approach | Choose when |
|---|---|
| Unity | Target is an MCU; no C++ runtime; minimal footprint required |
| CMocka | Host-side C testing; you want leak detection and mocking |
| GoogleTest + `extern "C"` | The C code builds for the host and the project already uses gtest for C++ |

Mixing is fine and common: Unity for the on-target smoke suite, GoogleTest for
the host-side bulk.

## 6. Static functions — the C testability problem

C's `static` keyword gives a function internal linkage, so a test file cannot
call it:

```c
/* parser.c */
static int validate_checksum(const uint8_t *frame, size_t len) { ... }
```

Three standard workarounds:

| Approach | How | Trade-off |
|---|---|---|
| **`#include` the .c file** | Test file does `#include "parser.c"` instead of the header | Works everywhere, no production change; but you must not also link `parser.c`, or you get duplicate symbols |
| **Conditional `static`** | `#ifdef UNIT_TESTING #define STATIC #else #define STATIC static #endif`, then declare `STATIC int validate_checksum(...)` | Production keeps internal linkage; but the test build is not byte-identical to production |
| **Don't test it directly** | Reach it through the public function that calls it | Purest black-box approach; but hard to cover error paths |

```c
/* testable_static.h -- the conditional-static idiom */
#ifdef UNIT_TESTING
#define STATIC
#else
#define STATIC static
#endif
```

The third option is the right default. Reach for the first two when a static
helper contains genuinely intricate logic (a checksum, a state transition table)
whose error paths you cannot reach from outside. If you find yourself needing
them everywhere, that's a design finding worth raising: the file is doing too
much and wants splitting.

## 7. Frameworks compared on the same test

The same boundary case, three ways:

=== "Unity"

    ```c
    static void test_accepts_upper_boundary(void) {
        int out = 0;
        TEST_ASSERT_EQUAL_INT(0, parse_temperature("125", &out));
        TEST_ASSERT_EQUAL_INT(125, out);
    }
    /* plus: RUN_TEST(test_accepts_upper_boundary); in main() */
    ```

=== "CMocka"

    ```c
    static void test_accepts_upper_boundary(void **state) {
        (void)state;
        int out = 0;
        assert_int_equal(parse_temperature("125", &out), 0);
        assert_int_equal(out, 125);
    }
    /* plus: cmocka_unit_test(test_accepts_upper_boundary), in the array */
    ```

=== "GoogleTest"

    ```cpp
    TEST(ParseTemperatureTest, AcceptsUpperBoundary) {
        int out = 0;
        EXPECT_EQ(parse_temperature("125", &out), 0);
        EXPECT_EQ(out, 125);
    }
    /* no registration needed */
    ```

Note the argument order difference: Unity and CMocka put **expected first**
(`TEST_ASSERT_EQUAL_INT(0, actual)`), GoogleTest puts **actual first**
(`EXPECT_EQ(actual, 0)`). Getting this backwards doesn't break the test, but it
inverts every failure message — "Expected 3 Was 0" when you meant the opposite.
Check the convention before you write a hundred tests.

## Exercise

Build a complete C unit test suite for a ring buffer.

1. **Implement `src/ring_buffer.c` / `.h`** with this API:

    ```c
    typedef struct RingBuffer RingBuffer;

    RingBuffer *ring_create(size_t capacity);   /* NULL on failure */
    void        ring_destroy(RingBuffer *rb);   /* safe on NULL */
    /* Writes up to n bytes; returns bytes written, or -1 on NULL args. */
    int         ring_write(RingBuffer *rb, const uint8_t *src, size_t n);
    /* Reads up to n bytes; returns bytes read, or -1 on NULL args. */
    int         ring_read(RingBuffer *rb, uint8_t *dest, size_t n);
    size_t      ring_size(const RingBuffer *rb);
    size_t      ring_capacity(const RingBuffer *rb);
    ```

2. **Write a Unity suite** with at least 15 tests, driven by the boundary table
   you built in Module 5's exercise. It must include:
   - `ring_write` of 0 bytes, 1 byte, capacity−1, capacity, capacity+1
   - a write-then-read round trip verified with
     `TEST_ASSERT_EQUAL_MEMORY`
   - wraparound: fill, read half, write more so the data wraps the internal
     index, then read everything and verify the bytes and order
   - `NULL` for each pointer parameter, one at a time
   - `ring_destroy(NULL)` must not crash

3. **Write a CMocka version** of at least five of those tests, and enable
   CMocka's leak detection by defining `UNIT_TESTING` and the `malloc`/`free`
   macros in `ring_buffer.c`. Then **deliberately remove the `free()`** inside
   `ring_destroy` and confirm CMocka reports the leak with a file and line
   number. Restore it afterwards.

4. **Register both suites with CTest** as in section 4 and confirm
   `ctest --test-dir build -N` lists two tests.

5. **Add a static helper** — `static size_t bytes_free(const RingBuffer *rb)` —
   and test it using the conditional-`STATIC` idiom from section 6. Then write
   two sentences arguing either for or against doing this, given that you could
   instead infer free space from `ring_capacity() - ring_size()`.

6. **Cross-check with GoogleTest.** Add `extern "C"` guards to `ring_buffer.h`
   and write three of the same tests in a GoogleTest file. Confirm all three
   suites pass, and note in one sentence which framework gave you the most
   useful failure message when you broke `ring_write` on purpose.
