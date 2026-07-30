# 07 · Unit Testing with GoogleTest

GoogleTest (gtest) is the most widely used C++ unit testing framework. It gives
you test registration, a rich assertion vocabulary, fixtures for shared setup,
and a runner with filtering and reporting — replacing the hand-rolled harness
from Module 6 with something you can grow to thousands of tests.

## 1. Getting GoogleTest into your project

The cleanest approach is CMake's `FetchContent`, which downloads and builds
GoogleTest as part of your configure step. No system install, and every
developer gets the same version.

`CMakeLists.txt` (top level):

```cmake
cmake_minimum_required(VERSION 3.14)
project(myproject LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
add_compile_options(-Wall -Wextra)

add_library(calculator src/calculator.cpp)
target_include_directories(calculator PUBLIC src)

# --- GoogleTest ---------------------------------------------------------
include(FetchContent)
FetchContent_Declare(
  googletest
  GIT_REPOSITORY https://github.com/google/googletest.git
  GIT_TAG        v1.15.2          # pin a version -- never track a branch
)
# Windows: keep gtest from overriding the parent project's CRT settings
set(gtest_force_shared_crt ON CACHE BOOL "" FORCE)
FetchContent_MakeAvailable(googletest)
# ------------------------------------------------------------------------

enable_testing()
add_subdirectory(tests)
```

`tests/CMakeLists.txt`:

```cmake
add_executable(test_calculator test_calculator.cpp)
target_link_libraries(test_calculator PRIVATE calculator GTest::gtest_main)

# Auto-discover every TEST() in the binary and register each with CTest
include(GoogleTest)
gtest_discover_tests(test_calculator)
```

Two details worth understanding:

- `GTest::gtest_main` provides a `main()` that runs all registered tests, so
  your test file doesn't need one.
- `gtest_discover_tests()` runs the binary at build time with
  `--gtest_list_tests` and registers **each test individually** with CTest — so
  `ctest -R Divide` works, and a failure names the specific test rather than
  "the test binary". Prefer it over `add_test(NAME ... COMMAND ...)` for gtest
  binaries.

!!! tip "Pin the tag"
    `GIT_TAG main` means your test suite's behaviour can change without you
    changing anything — the worst possible property for a test framework. Always
    pin a release tag.

## 2. Your first test

`tests/test_calculator.cpp`:

```cpp
#include <gtest/gtest.h>
#include "calculator.h"

// TEST(TestSuiteName, TestName)
TEST(CalculatorTest, AddsTwoPositiveNumbers) {
    Calculator c;
    EXPECT_EQ(c.add(2, 3), 5);
}

TEST(CalculatorTest, AddsNegativeAndPositive) {
    Calculator c;
    EXPECT_EQ(c.add(-1, 1), 0);
}

TEST(CalculatorTest, DivisionTruncatesTowardZero) {
    Calculator c;
    EXPECT_EQ(c.divide(7, 2), 3);
}
```

Build and run:

```bash
cmake -S . -B build
cmake --build build
./build/tests/test_calculator
```

```
[==========] Running 3 tests from 1 test suite.
[----------] Global test environment set-up.
[----------] 3 tests from CalculatorTest
[ RUN      ] CalculatorTest.AddsTwoPositiveNumbers
[       OK ] CalculatorTest.AddsTwoPositiveNumbers (0 ms)
[ RUN      ] CalculatorTest.AddsNegativeAndPositive
[       OK ] CalculatorTest.AddsNegativeAndPositive (0 ms)
[ RUN      ] CalculatorTest.DivisionTruncatesTowardZero
[       OK ] CalculatorTest.DivisionTruncatesTowardZero (0 ms)
[----------] 3 tests from CalculatorTest (0 ms total)

[==========] 3 tests from 1 test suite ran. (0 ms total)
[  PASSED  ] 3 tests.
```

Or through CTest:

```bash
ctest --test-dir build --output-on-failure
```

```
    Start 1: CalculatorTest.AddsTwoPositiveNumbers
1/3 Test #1: CalculatorTest.AddsTwoPositiveNumbers ...   Passed    0.00 sec
    Start 2: CalculatorTest.AddsNegativeAndPositive
2/3 Test #2: CalculatorTest.AddsNegativeAndPositive ...   Passed    0.00 sec
    Start 3: CalculatorTest.DivisionTruncatesTowardZero
3/3 Test #3: CalculatorTest.DivisionTruncatesTowardZero   Passed    0.00 sec

100% tests passed, 0 tests failed out of 3
```

### What a failure looks like

Change one expectation to something wrong and rebuild:

```cpp
TEST(CalculatorTest, DivisionTruncatesTowardZero) {
    Calculator c;
    EXPECT_EQ(c.divide(7, 2), 4);      // wrong on purpose
}
```

```
[ RUN      ] CalculatorTest.DivisionTruncatesTowardZero
/home/you/myproject/tests/test_calculator.cpp:16: Failure
Expected equality of these values:
  c.divide(7, 2)
    Which is: 3
  4
[  FAILED  ] CalculatorTest.DivisionTruncatesTowardZero (0 ms)
...
[  FAILED  ] 1 test, listed below:
[  FAILED  ] CalculatorTest.DivisionTruncatesTowardZero

 1 FAILED TEST
```

File, line, both values, and which test. That diagnostic quality is most of
what a framework buys you over `if (x != y) printf("fail")`.

## 3. Naming tests

The test name is documentation that runs. Since `gtest_discover_tests` surfaces
these names in CTest output and CI dashboards, a good name means a failure is
diagnosable from a build log alone.

| Poor | Better |
|---|---|
| `TEST(Calc, Test1)` | `TEST(CalculatorTest, AddsTwoPositiveNumbers)` |
| `TEST(Calc, DivideWorks)` | `TEST(CalculatorTest, DivisionTruncatesTowardZero)` |
| `TEST(Calc, Error)` | `TEST(CalculatorTest, ThrowsInvalidArgumentWhenDivisorIsZero)` |

Conventions that work well:

- **Suite name** = the class or unit under test, plus `Test`.
- **Test name** = the behaviour being asserted, in the affirmative, no
  underscores (gtest reserves `_` for internal use — `TEST(Foo, Bar_Baz)` can
  actually break in edge cases).
- Read it as a sentence: *"CalculatorTest: division truncates toward zero."*

## 4. Assertions

### EXPECT vs. ASSERT

| Macro family | On failure | Use when |
|---|---|---|
| `EXPECT_*` | Records the failure, **continues** the test | Default — you want all failures reported at once |
| `ASSERT_*` | Records the failure, **returns from the function** immediately | Continuing would crash or be meaningless |

```cpp
TEST(VectorTest, SecondElementIsCorrect) {
    std::vector<int> v = build_vector();

    ASSERT_GE(v.size(), 2u);    // if this fails, v[1] would be UB -- must abort
    EXPECT_EQ(v[1], 42);        // safe to continue past this one
}
```

The rule for C/C++ specifically: **use `ASSERT` for any precondition whose
violation would make the following lines undefined behaviour.** A null pointer
check before dereferencing, a size check before indexing. `EXPECT` on those
turns a clean test failure into a segfault that takes the whole test binary
down with it — losing the results of every test after it.

!!! warning "`ASSERT_*` only returns from the *current function*"
    Because `ASSERT` expands to `return;`, it cannot be used in a function
    returning non-void, and if you put it in a helper function it returns from
    the *helper*, not the test — execution continues in the test body. Either
    keep asserts in the test body, or have helpers return a value the test
    checks.

### The assertion vocabulary

| Assertion | Checks |
|---|---|
| `EXPECT_TRUE(cond)` / `EXPECT_FALSE(cond)` | Boolean condition |
| `EXPECT_EQ(a, b)` / `EXPECT_NE(a, b)` | `a == b` / `a != b` |
| `EXPECT_LT` `EXPECT_LE` `EXPECT_GT` `EXPECT_GE` | `<` `<=` `>` `>=` |
| `EXPECT_STREQ(a, b)` / `EXPECT_STRNE` | **C strings** (`char*`) by content |
| `EXPECT_STRCASEEQ` / `EXPECT_STRCASENE` | C strings, case-insensitive |
| `EXPECT_FLOAT_EQ(a, b)` / `EXPECT_DOUBLE_EQ` | Floats within 4 ULPs |
| `EXPECT_NEAR(a, b, tol)` | Floats within an explicit tolerance |
| `EXPECT_THROW(stmt, ExType)` | Statement throws that exception type |
| `EXPECT_ANY_THROW(stmt)` | Statement throws anything |
| `EXPECT_NO_THROW(stmt)` | Statement throws nothing |
| `EXPECT_DEATH(stmt, regex)` | Statement crashes/aborts with matching stderr |
| `FAIL()` / `SUCCEED()` | Unconditional |
| `ADD_FAILURE() << "msg"` | Record a failure and continue |

Every `ASSERT_` variant exists for every `EXPECT_` listed.

!!! warning "`EXPECT_EQ` on `char*` compares pointers, not text"
    ```cpp
    const char* a = get_name();       // returns "hello"
    EXPECT_EQ(a, "hello");            // WRONG -- compares addresses; may fail
    EXPECT_STREQ(a, "hello");         // right -- compares contents
    ```
    This bites nearly everyone once. `std::string` compares correctly with
    `EXPECT_EQ` because it has `operator==`; raw `char*` does not.

### Floating point

```cpp
TEST(MathTest, FloatingPointComparison) {
    double result = 0.1 + 0.2;

    // EXPECT_EQ(result, 0.3);      // FAILS -- result is 0.30000000000000004
    EXPECT_DOUBLE_EQ(result, 0.3);  // passes -- within 4 units in the last place
    EXPECT_NEAR(result, 0.3, 1e-9); // passes -- explicit tolerance
}
```

Use `EXPECT_NEAR` with a tolerance derived from the domain (measurement
precision, accumulated error budget) whenever you can justify one; fall back to
`EXPECT_DOUBLE_EQ` only for values that should be bit-identical modulo rounding.

### Custom failure messages

Every assertion accepts streamed context:

```cpp
TEST(ParserTest, AllSamplesParse) {
    for (size_t i = 0; i < samples.size(); ++i) {
        EXPECT_EQ(parse(samples[i]), expected[i])
            << "sample index " << i << ", input was: '" << samples[i] << "'";
    }
}
```

Without the message, a failure in a loop tells you only that "one of them was
wrong". With it, you know which. Always add context inside loops.

## 5. Testing error paths

```cpp
TEST(CalculatorTest, ThrowsInvalidArgumentWhenDivisorIsZero) {
    Calculator c;
    EXPECT_THROW(c.divide(1, 0), std::invalid_argument);
}

TEST(CalculatorTest, DoesNotThrowForValidDivisor) {
    Calculator c;
    EXPECT_NO_THROW(c.divide(1, 2));
}
```

To inspect the exception's contents, catch it manually:

```cpp
TEST(CalculatorTest, DivideByZeroMessageNamesTheProblem) {
    Calculator c;
    try {
        c.divide(1, 0);
        FAIL() << "expected std::invalid_argument to be thrown";
    } catch (const std::invalid_argument& e) {
        EXPECT_STREQ(e.what(), "division by zero");
    }
}
```

The `FAIL()` after the call is essential — without it, a version of `divide()`
that stops throwing would silently pass this test.

## 6. Test fixtures

When several tests need the same setup, a **fixture** removes the duplication
and guarantees each test gets a *fresh* instance.

```cpp
#include <gtest/gtest.h>
#include "ring_buffer.h"

class RingBufferTest : public ::testing::Test {
protected:
    void SetUp() override {
        buffer = new RingBuffer(64);      // runs before EACH test
    }

    void TearDown() override {
        delete buffer;                    // runs after EACH test
    }

    RingBuffer* buffer;                   // accessible in every TEST_F below
};

// Note TEST_F (fixture), not TEST. First argument is the fixture class name.
TEST_F(RingBufferTest, StartsEmpty) {
    EXPECT_EQ(buffer->size(), 0u);
    EXPECT_TRUE(buffer->empty());
}

TEST_F(RingBufferTest, WriteIncreasesSize) {
    const uint8_t data[] = {1, 2, 3};
    EXPECT_EQ(buffer->write(data, 3), 3);
    EXPECT_EQ(buffer->size(), 3u);
}

TEST_F(RingBufferTest, WriteBeyondCapacityIsTruncated) {
    std::vector<uint8_t> big(100, 0xAB);
    EXPECT_EQ(buffer->write(big.data(), big.size()), 64);   // capacity is 64
    EXPECT_TRUE(buffer->full());
}
```

The critical property: **gtest constructs a brand-new fixture object for every
single test.** `StartsEmpty` cannot be affected by what `WriteIncreasesSize`
did, and the tests can run in any order. That independence (Module 2, section 8)
is not something you have to arrange — the framework enforces it.

### Prefer the constructor/destructor when you can

```cpp
class RingBufferTest : public ::testing::Test {
protected:
    RingBufferTest() : buffer(64) {}      // constructor instead of SetUp
    RingBuffer buffer;                     // value member -- no new/delete at all
};
```

This version has no manual memory management, so the fixture itself cannot leak.
Use `SetUp()`/`TearDown()` when setup can fail in a way you want to assert on
(you cannot use `ASSERT_*` in a constructor) or when teardown must run even if
setup threw.

### Per-suite setup

For expensive one-time setup shared by all tests in a suite:

```cpp
class DatabaseTest : public ::testing::Test {
protected:
    static void SetUpTestSuite() {        // once, before the first test
        shared_db = open_test_database();
    }
    static void TearDownTestSuite() {     // once, after the last test
        close_database(shared_db);
        shared_db = nullptr;
    }
    static Database* shared_db;
};
Database* DatabaseTest::shared_db = nullptr;
```

Use sparingly — shared state between tests reintroduces exactly the coupling
fixtures exist to prevent.

## 7. Running and filtering

```bash
# Everything
./build/tests/test_calculator

# List without running
./build/tests/test_calculator --gtest_list_tests

# One suite
./build/tests/test_calculator --gtest_filter='CalculatorTest.*'

# One test
./build/tests/test_calculator --gtest_filter='CalculatorTest.AddsTwoPositiveNumbers'

# Wildcards, and '-' to exclude
./build/tests/test_calculator --gtest_filter='*Divide*'
./build/tests/test_calculator --gtest_filter='-*Slow*'

# Stop at the first failure
./build/tests/test_calculator --gtest_break_on_failure

# Repeat -- the flaky test hunter
./build/tests/test_calculator --gtest_repeat=100

# Randomize order -- exposes hidden inter-test dependencies
./build/tests/test_calculator --gtest_shuffle

# Machine-readable output for CI
./build/tests/test_calculator --gtest_output=xml:results.xml
```

`--gtest_shuffle` is worth running periodically even when everything is green:
if your suite passes in order but fails shuffled, you have hidden shared state,
which will eventually produce a failure nobody can reproduce.

## 8. Turning manual test cases into gtest

This is the bridge between the first half of this level and the second. Take
TC-TEMP-012 from Module 2:

| Field | Value |
|---|---|
| Title | Above-maximum temperature is rejected without modifying the output parameter |
| Test data | `text = "126"`, `out` pre-set to `0x5A5A5A5A` |
| Expected | `rc == -2` and `out == 0x5A5A5A5A` |

Becomes:

```cpp
TEST(ParseTemperatureTest, RejectsAboveMaximumWithoutModifyingOutput) {
    int out = 0x5A5A5A5A;                    // sentinel from the test case
    int rc = parse_temperature("126", &out);

    EXPECT_EQ(rc, -2) << "126 is above the maximum of 125";
    EXPECT_EQ(out, 0x5A5A5A5A) << "output must be left unmodified on error";
}
```

The mapping is mechanical:

| Manual test case field | GoogleTest equivalent |
|---|---|
| Test Case ID + Title | The `TEST(Suite, Name)` identifier |
| Preconditions | Fixture `SetUp()` or the first lines of the test |
| Test data | Local variables / literals |
| Steps | The code in the test body |
| Expected result | The `EXPECT_*` / `ASSERT_*` assertions |
| Requirement ID | A comment, or streamed into the failure message |
| Actual result / Status | Produced by the runner |

And the boundary set from Module 5 becomes a run of tests:

```cpp
TEST(ParseTemperatureTest, AcceptsLowerBoundary) {
    int out = 0;
    EXPECT_EQ(parse_temperature("-40", &out), 0);
    EXPECT_EQ(out, -40);
}

TEST(ParseTemperatureTest, RejectsJustBelowLowerBoundary) {
    int out = 0x5A5A5A5A;
    EXPECT_EQ(parse_temperature("-41", &out), -2);
    EXPECT_EQ(out, 0x5A5A5A5A);
}

TEST(ParseTemperatureTest, AcceptsUpperBoundary) {
    int out = 0;
    EXPECT_EQ(parse_temperature("125", &out), 0);
    EXPECT_EQ(out, 125);
}

TEST(ParseTemperatureTest, RejectsJustAboveUpperBoundary) {
    int out = 0x5A5A5A5A;
    EXPECT_EQ(parse_temperature("126", &out), -2);
    EXPECT_EQ(out, 0x5A5A5A5A);
}
```

Repetitive — deliberately so at this stage. Level 2's parameterized tests
collapse this into a single test driven by a data table, which is exactly the
shape your Module 5 boundary table already has.

## 9. Test structure — Arrange, Act, Assert

Keep every test in three visually distinct phases:

```cpp
TEST(RingBufferTest, ReadReturnsWhatWasWritten) {
    // Arrange -- set up the world
    RingBuffer buffer(64);
    const uint8_t input[] = {0xDE, 0xAD, 0xBE, 0xEF};
    buffer.write(input, 4);

    // Act -- perform the one operation under test
    uint8_t output[4] = {0};
    int n = buffer.read(output, 4);

    // Assert -- check the observable outcome
    ASSERT_EQ(n, 4);
    EXPECT_EQ(std::memcmp(input, output, 4), 0);
    EXPECT_TRUE(buffer.empty());
}
```

If a test has two "Act" phases, it is probably two tests. The value of splitting
them is that a failure names which behaviour broke.

## 10. Common mistakes

| Mistake | Why it hurts | Fix |
|---|---|---|
| `EXPECT_EQ` on `char*` | Compares pointers | `EXPECT_STREQ` |
| `EXPECT_EQ` on floats | Fails on representable-value rounding | `EXPECT_NEAR` / `EXPECT_DOUBLE_EQ` |
| `EXPECT` where the next line dereferences | Segfault kills the whole binary | `ASSERT` for preconditions |
| Logic in the test (`if`/`while` deciding what to assert) | The test can silently assert nothing | Straight-line tests; separate tests per case |
| Tests sharing state via globals or statics | Order-dependent, flaky | Fixtures; verify with `--gtest_shuffle` |
| Signed/unsigned comparisons in assertions | `-Wsign-compare` warnings, surprising results | `EXPECT_EQ(v.size(), 3u)` — note the `u` |
| Asserting on implementation details | Test breaks on every refactor | Assert on the public contract |
| Writing the test after seeing the output | Certifies the bug (Module 2) | Derive expectations from requirements |

## Exercise

Extend the project from Module 6 with a real GoogleTest suite.

1. **Wire up GoogleTest** via `FetchContent` as in section 1, and convert your
   three hand-rolled checks to `TEST()` cases with `gtest_discover_tests`.
   Confirm `ctest --test-dir build -N` lists them as three separate tests.

2. **Add a `Statistics` class** to your library:

    ```cpp
    class Statistics {
    public:
        // Throws std::invalid_argument if values is empty.
        double mean(const std::vector<int>& values) const;
        // Returns the largest value. Throws std::invalid_argument if empty.
        int max(const std::vector<int>& values) const;
        // Returns count of values strictly greater than threshold.
        size_t count_above(const std::vector<int>& values, int threshold) const;
    };
    ```

    Implement it, then write a `StatisticsTest` fixture and at least **twelve**
    tests covering:
    - a nominal case for each method
    - the empty-vector case for all three methods (two throw, one returns 0)
    - a single-element vector for each method
    - `mean` of values that don't divide evenly — assert with `EXPECT_NEAR` and
      justify your tolerance in a comment
    - `count_above` where the threshold equals an element exactly (a boundary:
      "strictly greater" means it must not be counted)
    - `max` where the maximum is the first element, and again where it is the
      last

3. **Deliberately break one method** (e.g. make `count_above` use `>=`) and
   record which of your tests fail. If only one test catches it, add the missing
   boundary case. Restore the method afterwards.

4. **Add an overflow test.** Write a test for `mean({INT_MAX, INT_MAX})`. Run it
   in both `build-debug` and `build-release`. Record what happens in each.
   Then write three sentences explaining the result in terms of undefined
   behaviour (Module 1 and Module 5, section 3) and what the *implementation*
   should do to make this testable.

5. **Prove test independence.** Run your suite with `--gtest_shuffle
   --gtest_repeat=20` and confirm it passes every time. If it doesn't, find the
   shared state.
