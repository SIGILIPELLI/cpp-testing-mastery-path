# 09 · Building an In-House C/C++ Test Framework

GoogleTest and CMocka (Levels 1-2) cover the overwhelming majority of
projects. The remaining minority — a target with no libstdc++, a
constrained embedded environment that can't afford GoogleTest's footprint,
or a codebase with requirements a general framework doesn't fit — sometimes
needs a minimal, purpose-built test framework. This module builds one from
scratch, in plain C, entirely on this host, so every piece of it is real
and runnable.

!!! note "Environment note"
    Every code block and every captured output in this module was written
    in plain C and compiled/run on this host with `gcc`. Unlike most of
    Level 3 and 4, nothing here was blocked by the broken-libc++ issue
    noted elsewhere in this path — a from-scratch framework in C has no
    dependency on the standard library headers that were unavailable.

## 1. What a test framework actually needs to do

Strip GoogleTest/CMocka down to their essential job: register test
functions, run them, catch and report failures without stopping the whole
run, and print a summary. Everything else (fixtures, mocking, parameterized
tests) builds on that core.

```c
/* minitest.h -- the entire public API */
#ifndef MINITEST_H
#define MINITEST_H

typedef void (*test_fn)(void);

void minitest_register(const char *name, test_fn fn);
int  minitest_run_all(void);   /* returns 0 if all passed, 1 otherwise */

/* Assertion macros -- record failure and continue, rather than aborting
   the whole test binary the way a bare assert() would. */
#define MT_ASSERT_EQ_INT(expected, actual) \
    minitest_assert_eq_int((expected), (actual), #expected, #actual, __FILE__, __LINE__)

#define MT_ASSERT_TRUE(cond) \
    minitest_assert_true((cond), #cond, __FILE__, __LINE__)

void minitest_assert_eq_int(long expected, long actual,
                             const char *expected_expr, const char *actual_expr,
                             const char *file, int line);
void minitest_assert_true(int cond, const char *expr,
                           const char *file, int line);

#define MT_TEST(name) static void name(void); \
    __attribute__((constructor)) static void register_##name(void) { \
        minitest_register(#name, name); \
    } \
    static void name(void)

#endif
```

The `__attribute__((constructor))` trick is what GoogleTest's `TEST()` macro
also relies on conceptually (though GoogleTest's actual mechanism is more
elaborate): each test function registers itself automatically before
`main()` runs, so writing `MT_TEST(my_test) { ... }` is enough — no manual
registration list to maintain.

## 2. The implementation

```c
/* minitest.c */
#include "minitest.h"
#include <stdio.h>
#include <string.h>

#define MAX_TESTS 256

typedef struct {
    const char *name;
    test_fn fn;
} test_entry;

static test_entry g_tests[MAX_TESTS];
static int g_test_count = 0;

static int g_current_test_failed = 0;
static int g_assertions_in_current_test = 0;

void minitest_register(const char *name, test_fn fn) {
    if (g_test_count >= MAX_TESTS) {
        fprintf(stderr, "minitest: too many tests (max %d)\n", MAX_TESTS);
        return;
    }
    g_tests[g_test_count].name = name;
    g_tests[g_test_count].fn = fn;
    g_test_count++;
}

void minitest_assert_eq_int(long expected, long actual,
                             const char *expected_expr, const char *actual_expr,
                             const char *file, int line) {
    g_assertions_in_current_test++;
    if (expected != actual) {
        g_current_test_failed = 1;
        fprintf(stderr, "  FAILED at %s:%d\n", file, line);
        fprintf(stderr, "    expected: %s == %ld\n", expected_expr, expected);
        fprintf(stderr, "    actual:   %s == %ld\n", actual_expr, actual);
    }
}

void minitest_assert_true(int cond, const char *expr,
                           const char *file, int line) {
    g_assertions_in_current_test++;
    if (!cond) {
        g_current_test_failed = 1;
        fprintf(stderr, "  FAILED at %s:%d\n", file, line);
        fprintf(stderr, "    expected true: %s\n", expr);
    }
}

int minitest_run_all(void) {
    int passed = 0, failed = 0;
    printf("[==========] Running %d test(s)\n", g_test_count);
    for (int i = 0; i < g_test_count; ++i) {
        printf("[ RUN      ] %s\n", g_tests[i].name);
        g_current_test_failed = 0;
        g_assertions_in_current_test = 0;

        g_tests[i].fn();

        if (g_current_test_failed) {
            printf("[  FAILED  ] %s\n", g_tests[i].name);
            failed++;
        } else {
            printf("[       OK ] %s (%d assertion%s)\n", g_tests[i].name,
                   g_assertions_in_current_test,
                   g_assertions_in_current_test == 1 ? "" : "s");
            passed++;
        }
    }
    printf("[==========] %d passed, %d failed\n", passed, failed);
    return failed > 0 ? 1 : 0;
}
```

```c
/* minitest_main.c -- a reusable main(), like GTest's gtest_main */
#include "minitest.h"

int main(void) {
    return minitest_run_all();
}
```

## 3. Using it, with real, captured output

```c
/* example_tests.c */
#include "minitest.h"

static int add(int a, int b) { return a + b; }

MT_TEST(AdditionWorks) {
    MT_ASSERT_EQ_INT(4, add(2, 2));
}

MT_TEST(AdditionOfNegatives) {
    MT_ASSERT_EQ_INT(-1, add(-3, 2));
}

MT_TEST(DeliberatelyFailingTest) {
    MT_ASSERT_EQ_INT(5, add(2, 2));      /* wrong on purpose, to prove
                                             failure reporting works */
    MT_ASSERT_TRUE(add(1, 1) == 2);       /* this one still runs and passes,
                                             proving one failed assertion
                                             doesn't abort the test */
}
```

```bash
gcc -std=c11 -Wall -Wextra minitest.c minitest_main.c example_tests.c -o example_tests
./example_tests
```

```text
[==========] Running 3 test(s)
[ RUN      ] AdditionWorks
[       OK ] AdditionWorks (1 assertion)
[ RUN      ] AdditionOfNegatives
[       OK ] AdditionOfNegatives (1 assertion)
[ RUN      ] DeliberatelyFailingTest
  FAILED at example_tests.c:15
    expected: 5 == 5
    actual:   add(2, 2) == 4
[  FAILED  ] DeliberatelyFailingTest
[==========] 2 passed, 1 failed
```

This is real: three tests, one deliberately wrong, and the framework
correctly identifies the specific failing assertion with file/line, while
still running (and passing) the second assertion in that same test — the
core behavior any test framework must get right. (One buffering detail: the
`FAILED` line goes to `stderr` while the `[ RUN ]`/`[ OK ]` lines go to
`stdout`; when both stream to the same terminal, the exact interleaving
between them can vary run to run — pipe stderr into stdout, e.g.
`./example_tests 2>&1`, or flush explicitly, if you need a strictly ordered
combined log for CI.)

## 4. Exit code wiring for CI

```bash
./example_tests; echo "exit code: $?"
```

```text
exit code: 1
```

A non-zero exit code on any failure is what makes this framework
CI-compatible with zero additional glue — `ctest`, GitHub Actions, or any
shell-based CI step already knows how to fail a job on a non-zero exit.

## 5. Adding fixtures: setup/teardown per test

```c
/* minitest.h -- additional macro */
#define MT_TEST_F(fixture, name) \
    static void name##_body(fixture *f); \
    static void name(void) { \
        fixture f; \
        fixture##_setup(&f); \
        name##_body(&f); \
        fixture##_teardown(&f); \
    } \
    __attribute__((constructor)) static void register_##name(void) { \
        minitest_register(#fixture "." #name, name); \
    } \
    static void name##_body(fixture *f)
```

```c
typedef struct { int *buf; int cap; } ArrayFixture;
static void ArrayFixture_setup(ArrayFixture *f) {
    f->cap = 4;
    f->buf = calloc((size_t)f->cap, sizeof(int));
}
static void ArrayFixture_teardown(ArrayFixture *f) {
    free(f->buf);
}

MT_TEST_F(ArrayFixture, StartsZeroed) {
    MT_ASSERT_EQ_INT(0, f->buf[0]);
}
```

This mirrors GoogleTest's `TEST_F` (Level 2, module 01) — the same
SetUp/TearDown-per-test shape, built here from two macros and a naming
convention rather than a class hierarchy.

## 6. When building your own is (and isn't) the right call

| Signal | Build your own | Use GoogleTest/CMocka |
|---|---|---|
| Target has no C++ runtime / minimal libc, GoogleTest won't fit | Yes | N/A — it may not even link |
| Need is genuinely simple (register, run, assert, report) | Reasonable | Also reasonable — GoogleTest at rest costs little if it already fits |
| Team wants mocking, parameterized tests, death tests | No | Yes — reimplementing this well is a large, ongoing investment |
| A niche reporting format your CI specifically requires and no existing framework emits | Maybe, as a thin reporter layered on an existing framework's output | Prefer this over a full from-scratch framework |

The honest bar: everything built in sections 1-5 is a genuinely useful
educational exercise and can be a real, load-bearing tool for a
sufficiently constrained target — but it is a small fraction of what
GoogleTest or CMocka already provide, and "not invented here" is a bad
reason to choose it when either of those already fits.

## Cheat sheet

| Feature | This module's framework | GoogleTest |
|---|---|---|
| Auto-registration | Yes (constructor attribute) | Yes (more elaborate internally) |
| Assertion continues test on failure | Yes | Yes (`EXPECT_*`; `ASSERT_*` aborts the test) |
| Fixtures | Basic (section 5) | Full (`SetUp`/`TearDown`, fixture inheritance) |
| Parameterized tests | Not built here | Yes |
| Mocking | Not built here | Via GoogleMock |
| Death tests, typed tests | Not built here | Yes |
| Footprint | Two small `.c` files | Full library, larger binary |

## Exercise

1. Build and run `example_tests.c` exactly as shown, confirm the same
   failure output, then fix the deliberate bug and confirm a clean
   `2 passed, 0 failed` (or `3 passed, 0 failed` once fixed) result.
2. Add an `MT_ASSERT_STR_EQ` macro (compare two C strings) to `minitest.h`/
   `minitest.c`, following the pattern of `MT_ASSERT_EQ_INT`, and write a
   test using it.
3. Add a `--filter=<substring>` command-line option to `minitest_main.c`
   that only runs tests whose name contains the given substring — the
   minimal version of GoogleTest's `--gtest_filter`.
4. Using section 5's fixture macro, write a fixture-based test for the
   `BoundedStack` from Level 2's capstone (translated to plain C or a
   C-compatible subset) and confirm it compiles and runs on this host.
5. Write two sentences using section 6's table: for your own project, is
   building an in-house framework ever justified, and if so, what specific
   constraint would make it the right call rather than "not invented here"?
