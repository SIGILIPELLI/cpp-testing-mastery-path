# 03 · Testing C Code with CMocka Advanced

C has no virtual functions, so the mocking approach from Module 2 does not
transfer. CMocka solves the problem differently: it gives you a queue of return
values (`will_return`/`mock()`), a queue of expected arguments
(`expect_value`/`check_expected`), and a set of linker tricks for substituting
functions you don't own. This module covers those mechanisms and the portability
traps that come with them.

## 1. Installing and building

```bash
brew install cmocka                     # macOS
sudo apt install libcmocka-dev          # Debian/Ubuntu
```

CMocka's headers have a strict include order — `<stdarg.h>`, `<stddef.h>`,
`<setjmp.h>` must precede `<cmocka.h>`, because the header uses those types
without including them itself.

```c
#include <stdarg.h>
#include <stddef.h>
#include <setjmp.h>
#include <cmocka.h>     /* must come after the four above */
```

!!! warning "Wrong include order = a wall of errors"
    Putting `<cmocka.h>` first produces dozens of errors about `va_list`,
    `size_t` and `jmp_buf` being undeclared. The error text never mentions
    include order. If a fresh CMocka file explodes on compile, check this first.

## 2. The mock queue — `will_return` and `mock()`

CMocka's core idea: the test pushes values onto a per-function queue, and the
test double pops them.

```c
/* the seam: declared here, defined by production code OR by the test */
int hw_read_register(unsigned addr);

/* code under test */
int temperature_c(void) {
    int raw = hw_read_register(0x40);
    return (raw * 5) / 16 - 20;
}

/* the test double */
int hw_read_register(unsigned addr) {
    check_expected(addr);        /* pops the expected-argument queue */
    return (int) mock();         /* pops the return-value queue */
}
```

```c
static void test_temperature_converts_raw(void **state) {
    (void) state;
    expect_value(hw_read_register, addr, 0x40);
    will_return(hw_read_register, 128);
    assert_int_equal(temperature_c(), 20);
}

static void test_temperature_handles_zero(void **state) {
    (void) state;
    expect_value(hw_read_register, addr, 0x40);
    will_return(hw_read_register, 0);
    assert_int_equal(temperature_c(), -20);
}

int main(void) {
    const struct CMUnitTest tests[] = {
        cmocka_unit_test(test_temperature_converts_raw),
        cmocka_unit_test(test_temperature_handles_zero),
    };
    return cmocka_run_group_tests(tests, NULL, NULL);
}
```

Build and run:

```bash
clang -I/opt/homebrew/opt/cmocka/include test_temp.c \
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

The queues are **strict in both directions**. Queue two values and consume one,
and the test fails at teardown. Consume a value that was never queued, and the
test fails immediately. That strictness is the feature — it is what makes these
doubles mocks rather than stubs.

!!! tip "Deprecation in CMocka 1.1.8+"
    Recent CMocka deprecates the type-agnostic `check_expected` in favour of
    `check_expected_int`, `check_expected_uint` and `check_expected_ptr`. The
    old macro still works but emits `-Wdeprecated-declarations`. Prefer the
    typed forms in new code.

## 3. Argument expectations

```c
expect_value(func, param, 42);                    /* == 42 */
expect_not_value(func, param, 0);                 /* != 0 */
expect_string(func, param, "hello");              /* strcmp == 0 */
expect_memory(func, param, &blob, sizeof blob);   /* memcmp == 0 */
expect_in_range(func, param, 1, 100);             /* inclusive */
expect_any(func, param);                          /* accept anything */
expect_check(func, param, my_checker, cast_ptr);  /* custom predicate */
```

Every one of these is consumed by a single `check_expected(param)` in the
double. To expect three calls, queue three expectations.

## 4. Group and per-test fixtures

```c
static int group_setup(void **state) {
    struct ctx *c = calloc(1, sizeof *c);
    if (c == NULL) return -1;         /* non-zero aborts the group */
    c->table = load_lookup_table();
    *state = c;
    return 0;
}
static int group_teardown(void **state) {
    struct ctx *c = *state;
    free_lookup_table(c->table);
    free(c);
    return 0;
}
static int test_setup(void **state)    { ((struct ctx *)*state)->calls = 0; return 0; }

int main(void) {
    const struct CMUnitTest tests[] = {
        cmocka_unit_test_setup(test_a, test_setup),
        cmocka_unit_test_setup_teardown(test_b, test_setup, NULL),
        cmocka_unit_test(test_c),
    };
    return cmocka_run_group_tests(tests, group_setup, group_teardown);
}
```

Group setup runs once for the whole binary; per-test setup runs before each
test. Note that the per-test `*state` starts as the group's state pointer — a
per-test setup that overwrites `*state` loses the group context unless it saves
it first.

## 5. Substituting functions you don't own

The seam in section 2 worked because `hw_read_register` had no production
definition in the test binary. That is the **link seam**, and it is the portable
approach. When you must intercept a function that *is* linked in — `malloc`,
`open`, a third-party API — you need interposition.

On GNU toolchains, the linker can redirect calls:

```c
void *__wrap_malloc(size_t n) {
    if (mock_type(int) == 0) return NULL;    /* simulate OOM */
    return __real_malloc(n);
}
extern void *__real_malloc(size_t n);
```

```bash
gcc test_alloc.c -Wl,--wrap=malloc -lcmocka -o test_alloc
```

!!! warning "`--wrap` is GNU ld / lld only"
    Apple's linker does not implement `--wrap`; the flag is rejected outright.
    macOS offers `-Wl,-alias` and `DYLD_INSERT_LIBRARIES`, but neither is a drop-in
    replacement, and both are blocked for hardened binaries. **If you need your C
    tests to build on macOS and Linux, design a link seam instead of relying on
    `--wrap`.** Route allocations through your own `mem_alloc()` and give the
    test binary a different implementation — it costs one indirection and buys
    portability.

## 6. CMocka's allocator — leak and overflow detection for free

CMocka ships instrumented allocators. Swap them in and every test gains leak and
bounds checking with no extra tooling.

```c
#define UNIT_TESTING 1     /* before including cmocka.h */
```

With `UNIT_TESTING` defined, `malloc`/`calloc`/`realloc`/`free` in the code under
test are redirected to `test_malloc` and friends. A block still allocated when
the test ends is reported:

```
[  ERROR   ] --- Blocks allocated...
0x600000f04030: allocated by parser.c:88
ERROR: test_parse_header leaked 1 block(s)
```

This catches leaks that only occur on error paths — exactly the paths production
rarely exercises and reviewers rarely read.

## 7. Assertion vocabulary

| Assertion | Checks |
|---|---|
| `assert_true(x)` / `assert_false(x)` | truthiness |
| `assert_int_equal(a, b)` | signed integer equality |
| `assert_uint_equal(a, b)` | unsigned integer equality |
| `assert_ptr_equal(a, b)` | pointer identity |
| `assert_null(p)` / `assert_non_null(p)` | NULL-ness |
| `assert_string_equal(a, b)` | `strcmp` |
| `assert_memory_equal(a, b, n)` | `memcmp` |
| `assert_float_equal(a, b, eps)` | float within epsilon |
| `assert_in_range(v, lo, hi)` | inclusive bounds |
| `assert_return_code(rc, errno)` | `rc >= 0`, else report errno |

Every CMocka assertion is **fatal** — it longjmps out of the test. There is no
`EXPECT`-style non-fatal variant as in GoogleTest, so a single test reports at
most one failure. Prefer many small tests over one long one.

## 8. Traps worth memorising

| Trap | Symptom | Fix |
|---|---|---|
| `<cmocka.h>` included first | Errors about `va_list`, `size_t` | Include the four prerequisite headers first |
| Queued more values than consumed | Failure at teardown, not at the call | One `will_return` per expected call |
| `check_expected` with no `expect_*` | "No entries for symbol" | Queue an expectation per call |
| Relying on `--wrap` | Build breaks on macOS/Clang | Use a link seam |
| Per-test setup overwrites `*state` | Group context lost, segfault | Save the group pointer first |
| Non-zero return from group setup | Whole group silently skipped | Return 0 on success |
| Assertions after the first failure | Never run (longjmp) | Split into separate tests |
| `mock()` cast to the wrong width | Garbage values on 64-bit | Use `mock_type(T)` / `mock_ptr_type(T)` |

## Exercise

Test a small ring buffer that allocates its storage and reads from a hardware
seam.

```c
/* ring.h */
typedef struct ring ring;
ring  *ring_create(size_t capacity);   /* NULL on bad capacity or OOM */
void   ring_destroy(ring *r);
int    ring_push(ring *r, int value);  /* 0 ok, -1 full */
int    ring_pop(ring *r, int *out);    /* 0 ok, -1 empty */
int    ring_fill_from_sensor(ring *r, size_t n);  /* uses hw_read_register */
```

1. **Implement `ring.c`** against that contract, with `ring_fill_from_sensor`
   calling `hw_read_register(0x40)` once per sample and pushing each result.

2. **Build the link seam.** Put `hw_read_register` in its own file for
   production, and give the test binary a CMocka double instead. Write tests for
   `n = 0`, `n = 1`, `n = capacity`, and `n = capacity + 1`, queueing exactly the
   right number of `will_return` values each time.

3. **Prove the queue is strict.** Queue one value too many in a passing test and
   record the failure message. Then queue one too few and record that message.
   Write one sentence on why both are failures rather than warnings.

4. **Add argument expectations.** Use `expect_value` to assert the register
   address, then change the production code to read `0x41` and confirm the test
   catches it.

5. **Turn on the instrumented allocator.** Define `UNIT_TESTING`, then
   deliberately return early from `ring_create` after the first `malloc` but
   before storing the pointer. Capture the leak report. Fix it and confirm the
   report goes away.

6. **Fixtures.** Convert your tests to use `cmocka_unit_test_setup_teardown` so
   each test gets a fresh `ring` of capacity 4 and destroys it afterwards. Verify
   with the instrumented allocator that no test leaks.
