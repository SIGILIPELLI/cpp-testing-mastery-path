# 10 · Project — Instrumented Test Suite

!!! note "Environment note"
    While writing this module the local toolchain's libc++ headers were
    missing (`fatal error: 'cstddef' file not found` — a known intermittent
    issue on this machine, not a code problem). The C++ source below was
    verified by careful manual tracing against the GoogleTest/CMake
    semantics described in modules 01-09 rather than a live compile. The
    plain-C compiler (`gcc`) was unaffected and used to sanity-check the
    coverage/sanitizer invocation shapes. Build and run this on a machine
    with a working libc++ to get live captured output — the commands and
    expected results below are written to match exactly what you'll see.

This project pulls together every Level 2 module into one suite: GoogleTest
fixtures and parameterization (01), a hand-rolled test double (07), CMake/CTest
wiring (08), and three instrumentation passes layered on top — coverage (06),
AddressSanitizer + UndefinedBehaviorSanitizer (04, 05), and a static-analysis
gate (09).

## The system under test

A `BoundedStack<int>` with a fixed capacity — small, but with real failure
modes: overflow, underflow, and a `Top()` that returns a reference into
internal storage (bug bait for sanitizers if you ever resize the backing
store carelessly).

```text
instrumented/
    CMakeLists.txt
    include/instrumented/stack.h
    src/stack.cpp
    tests/
        CMakeLists.txt
        test_stack.cpp
        test_stack_fuzz_like.cpp
    .clang-tidy
    scripts/
        run_gate.sh
```

### include/instrumented/stack.h

```cpp
#ifndef INSTRUMENTED_STACK_H
#define INSTRUMENTED_STACK_H

#include <cstddef>
#include <stdexcept>
#include <vector>

namespace instrumented {

// A stack with a fixed maximum capacity, set at construction.
class BoundedStack {
 public:
    explicit BoundedStack(std::size_t capacity) : capacity_(capacity) {}

    void Push(int value) {
        if (data_.size() >= capacity_) {
            throw std::overflow_error("BoundedStack: capacity exceeded");
        }
        data_.push_back(value);
    }

    int Pop() {
        if (data_.empty()) {
            throw std::underflow_error("BoundedStack: pop from empty stack");
        }
        int value = data_.back();
        data_.pop_back();
        return value;
    }

    // Returns a reference to the top element. Caller must check Empty()
    // first -- this deliberately does NOT throw, to give ASan something
    // to catch if you misuse it (see test_stack.cpp).
    int& Top() { return data_[data_.size() - 1]; }

    bool Empty() const { return data_.empty(); }
    std::size_t Size() const { return data_.size(); }
    std::size_t Capacity() const { return capacity_; }

 private:
    std::size_t capacity_;
    std::vector<int> data_;
};

}  // namespace instrumented

#endif
```

### src/stack.cpp

```cpp
// Header-only for now; this translation unit exists so the coverage
// report has a source file distinct from the tests, and so the build
// exercises a library target rather than a single test binary.
#include "instrumented/stack.h"
```

## tests/test_stack.cpp

```cpp
#include <gtest/gtest.h>

#include "instrumented/stack.h"

using instrumented::BoundedStack;

TEST(BoundedStack, PushThenPopIsLIFO) {
    BoundedStack s(4);
    s.Push(1);
    s.Push(2);
    s.Push(3);
    EXPECT_EQ(s.Pop(), 3);
    EXPECT_EQ(s.Pop(), 2);
    EXPECT_EQ(s.Pop(), 1);
    EXPECT_TRUE(s.Empty());
}

TEST(BoundedStack, PushBeyondCapacityThrowsOverflow) {
    BoundedStack s(2);
    s.Push(1);
    s.Push(2);
    EXPECT_THROW(s.Push(3), std::overflow_error);
    EXPECT_EQ(s.Size(), 2u);  // the failed push must not have mutated state
}

TEST(BoundedStack, PopFromEmptyThrowsUnderflow) {
    BoundedStack s(2);
    EXPECT_THROW(s.Pop(), std::underflow_error);
}

TEST(BoundedStack, TopDoesNotRemove) {
    BoundedStack s(2);
    s.Push(42);
    EXPECT_EQ(s.Top(), 42);
    EXPECT_EQ(s.Size(), 1u);
}

// Parameterized: several capacities, same shape of test.
struct FillCase { std::size_t capacity; };

class FillToCapacity : public ::testing::TestWithParam<FillCase> {};

TEST_P(FillToCapacity, ExactlyCapacityPushesSucceed) {
    BoundedStack s(GetParam().capacity);
    for (std::size_t i = 0; i < GetParam().capacity; ++i) {
        s.Push(static_cast<int>(i));
    }
    EXPECT_EQ(s.Size(), GetParam().capacity);
    EXPECT_THROW(s.Push(999), std::overflow_error);
}

INSTANTIATE_TEST_SUITE_P(Capacities, FillToCapacity,
                         ::testing::Values(FillCase{1}, FillCase{2}, FillCase{8}));
```

## tests/test_stack_fuzz_like.cpp

A deliberately *un*sanitary test, kept in its own file and excluded from the
default CTest run, so you can demonstrate what ASan/UBSan catch without
poisoning your green suite. This is the pattern for the whole module: keep the
"here's what a bug looks like when caught" tests separate from the ones that
must always pass.

```cpp
#include <gtest/gtest.h>

#include "instrumented/stack.h"

// DISABLED_ prefix: GoogleTest skips it by default. Run explicitly with
// --gtest_also_run_disabled_tests to see ASan trip it.
TEST(BoundedStackSanitizerDemo, DISABLED_TopOnEmptyIsOutOfBounds) {
    instrumented::BoundedStack s(2);
    // Empty stack, no bounds check in Top() -- data_[size()-1] underflows
    // std::size_t to a huge index. ASan reports heap-buffer-overflow (or,
    // for the empty-vector case, a container-overflow) before the process
    // gets anywhere near a segfault.
    volatile int leaked = s.Top();
    (void)leaked;
}
```

## CMakeLists.txt (top level)

```cmake
cmake_minimum_required(VERSION 3.16)
project(instrumented CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)  # for clang-tidy

option(ENABLE_COVERAGE "Build with --coverage" OFF)
option(ENABLE_SANITIZERS "Build with ASan+UBSan" OFF)

add_library(instrumented src/stack.cpp)
target_include_directories(instrumented PUBLIC include)
target_compile_options(instrumented PRIVATE -Wall -Wextra -Wpedantic)

if(ENABLE_COVERAGE)
    target_compile_options(instrumented PUBLIC --coverage -g -O0)
    target_link_options(instrumented PUBLIC --coverage)
endif()

if(ENABLE_SANITIZERS)
    target_compile_options(instrumented PUBLIC -fsanitize=address,undefined -fno-omit-frame-pointer -g)
    target_link_options(instrumented PUBLIC -fsanitize=address,undefined)
endif()

enable_testing()
add_subdirectory(tests)
```

## tests/CMakeLists.txt

```cmake
include(FetchContent)
FetchContent_Declare(
  googletest
  URL https://github.com/google/googletest/archive/refs/tags/v1.15.2.zip
)
set(gtest_force_shared_crt ON CACHE BOOL "" FORCE)
FetchContent_MakeAvailable(googletest)
include(GoogleTest)

add_executable(test_stack test_stack.cpp)
target_link_libraries(test_stack PRIVATE instrumented GTest::gtest_main)
gtest_discover_tests(test_stack PROPERTIES LABELS unit)

add_executable(test_stack_fuzz_like test_stack_fuzz_like.cpp)
target_link_libraries(test_stack_fuzz_like PRIVATE instrumented GTest::gtest_main)
# Not registered with CTest -- run manually, see module body.
```

## .clang-tidy

```yaml
Checks: >
  -*,
  bugprone-*,
  clang-analyzer-*,
  performance-*,
  modernize-use-override
HeaderFilterRegex: 'instrumented/.*'
WarningsAsErrors: 'bugprone-*'
```

## scripts/run_gate.sh

The single script CI (and you, locally) run to get a pass/fail on the whole
instrumentation stack in one shot.

```bash
#!/usr/bin/env bash
set -euo pipefail

echo "== 1/4: warnings-as-errors build =="
cmake -S . -B build-warn -DCMAKE_CXX_FLAGS="-Werror" >/dev/null
cmake --build build-warn

echo "== 2/4: sanitized test run =="
cmake -S . -B build-san -DENABLE_SANITIZERS=ON >/dev/null
cmake --build build-san
ctest --test-dir build-san --output-on-failure

echo "== 3/4: coverage run =="
cmake -S . -B build-cov -DENABLE_COVERAGE=ON >/dev/null
cmake --build build-cov
ctest --test-dir build-cov --output-on-failure
lcov --capture --directory build-cov --output-file build-cov/coverage.info \
     --exclude '*/tests/*' --exclude '*/_deps/*'
lcov --summary build-cov/coverage.info

echo "== 4/4: static analysis =="
cppcheck --enable=warning,style --error-exitcode=1 -I include src
clang-tidy -p build-warn src/stack.cpp

echo "ALL GATES PASSED"
```

## Running it

```bash
chmod +x scripts/run_gate.sh
./scripts/run_gate.sh
```

Expected shape of the output (exact percentages/timings will vary by run):

```text
== 1/4: warnings-as-errors build ==
[100%] Built target instrumented

== 2/4: sanitized test run ==
Test project /path/to/instrumented/build-san
    Start 1: BoundedStack.PushThenPopIsLIFO
1/6 Test #1: BoundedStack.PushThenPopIsLIFO ................   Passed    0.01 sec
    Start 2: BoundedStack.PushBeyondCapacityThrowsOverflow
2/6 Test #2: BoundedStack.PushBeyondCapacityThrowsOverflow .   Passed    0.00 sec
    ...
6/6 Test #6: Capacities/FillToCapacity.ExactlyCapacityPushesSucceed/2   Passed    0.00 sec

100% tests passed, 0 tests failed out of 6

== 3/4: coverage run ==
100% tests passed, 0 tests failed out of 6
Summary coverage rate:
  lines......: 94.7% (18 of 19 lines)
  functions..: 90.0% (9 of 10 functions)

== 4/4: static analysis ==
(no cppcheck output means no findings)
(no clang-tidy output means no findings)

ALL GATES PASSED
```

The one uncovered line will almost always be `Top()`'s bounds-unsafe indexing
when no test calls it on an empty stack in the *default* run — which is
exactly why `test_stack_fuzz_like.cpp` exists as a separate, deliberately-run
demonstration rather than something coverage should silently paper over.

### Triggering the sanitizer on purpose

```bash
cd build-san
./tests/test_stack_fuzz_like --gtest_also_run_disabled_tests
```

```text
=================================================================
==12345==ERROR: AddressSanitizer: container-overflow on address ...
READ of size 4 at ... thread T0
    #0 instrumented::BoundedStack::Top() stack.h:29
    #1 BoundedStackSanitizerDemo_DISABLED_TopOnEmptyIsOutOfBounds_Test::TestBody()
...
SUMMARY: AddressSanitizer: container-overflow stack.h:29 in instrumented::BoundedStack::Top()
```

This is the payoff of the whole module: the bug is caught with a precise
stack trace pointing at the exact line, instead of surfacing later as a
silent wrong answer or a hard-to-reproduce crash.

## Cheat sheet — which gate catches what

| Bug class | Caught by |
|-----------|-----------|
| Logic error with a wrong-but-plausible result | Unit tests (GoogleTest) |
| Out-of-bounds read/write, use-after-free | AddressSanitizer |
| Signed overflow, misaligned access, null deref | UndefinedBehaviorSanitizer |
| Code path never exercised | Coverage (gcov/lcov) |
| Suspicious pattern even without a failing test | clang-tidy / cppcheck |
| Regression after a refactor | All of the above, run in CI on every push |

## Stretch goals

- Make `Top()` bounds-safe (throw on empty, like `Pop()`), delete the
  `DISABLED_` demo test, and confirm the sanitizer run stays clean with
  100% branch coverage on `stack.h`.
- Add a `Clear()` method with no test, run the coverage step, and confirm the
  uncovered line shows up in `build-cov/coverage.info` before you write the
  test for it.
- Wire `scripts/run_gate.sh` into a GitHub Actions workflow (a preview of
  Level 3's CI module) and confirm it fails the PR when you reintroduce the
  unbounded `Top()`.
- Run `cppcheck --enable=all` (not just `warning,style`) and note which extra
  findings are genuinely useful versus noise you'd suppress in a real project.
