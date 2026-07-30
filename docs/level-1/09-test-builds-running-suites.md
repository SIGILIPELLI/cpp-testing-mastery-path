# 09 · Test Builds & Running Suites

You now have tests. This module is about running them well: organizing test
targets so a growing suite stays navigable, using CTest's selection and
reporting features, and structuring the build so the same tests run in debug,
release, and (in Level 2) sanitizer configurations.

## 1. Project layout for a testable project

```
myproject/
├── CMakeLists.txt              # top level -- project, options, subdirs
├── include/
│   └── myproject/              # public headers, namespaced by directory
│       ├── calculator.h
│       └── ring_buffer.h
├── src/
│   ├── CMakeLists.txt
│   ├── calculator.cpp
│   └── ring_buffer.cpp
├── tests/
│   ├── CMakeLists.txt
│   ├── unit/
│   │   ├── test_calculator.cpp
│   │   └── test_ring_buffer.cpp
│   ├── integration/
│   │   └── test_pipeline.cpp
│   └── data/                   # fixture files: sample inputs, golden outputs
│       ├── valid.csv
│       └── malformed.csv
└── build/                      # generated -- never committed
```

Two conventions worth adopting from the start:

- **Mirror the source tree in the test tree.** `src/calculator.cpp` →
  `tests/unit/test_calculator.cpp`. Finding the tests for a file becomes
  mechanical, and gaps become visible (a source file with no matching test file
  is a question worth asking).
- **Separate unit from integration directories.** They have different speeds and
  different failure meanings, and you'll want to run them separately (section 4).

## 2. A complete multi-target CMake setup

Top-level `CMakeLists.txt`:

```cmake
cmake_minimum_required(VERSION 3.14)
project(myproject VERSION 1.0.0 LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)   # generates compile_commands.json,
                                        # used by clang-tidy (Level 2) and IDEs

option(BUILD_TESTING "Build the test suite" ON)

if(CMAKE_CXX_COMPILER_ID MATCHES "GNU|Clang")
    add_compile_options(-Wall -Wextra)
endif()

add_subdirectory(src)

if(BUILD_TESTING)
    enable_testing()
    add_subdirectory(tests)
endif()
```

`src/CMakeLists.txt`:

```cmake
add_library(myproject
    calculator.cpp
    ring_buffer.cpp
)
target_include_directories(myproject PUBLIC ${CMAKE_SOURCE_DIR}/include)
```

`tests/CMakeLists.txt`:

```cmake
include(FetchContent)
FetchContent_Declare(
  googletest
  GIT_REPOSITORY https://github.com/google/googletest.git
  GIT_TAG        v1.15.2
)
set(gtest_force_shared_crt ON CACHE BOOL "" FORCE)
FetchContent_MakeAvailable(googletest)

include(GoogleTest)

# Helper so each new test file is one line instead of four
function(add_project_test name source)
    add_executable(${name} ${source})
    target_link_libraries(${name} PRIVATE myproject GTest::gtest_main)
    # Make the test data directory findable from the test binary
    target_compile_definitions(${name} PRIVATE
        TEST_DATA_DIR="${CMAKE_CURRENT_SOURCE_DIR}/data")
    gtest_discover_tests(${name}
        PROPERTIES LABELS "${ARGV2}")     # optional third arg = label
endfunction()

add_project_test(test_calculator  unit/test_calculator.cpp   unit)
add_project_test(test_ring_buffer unit/test_ring_buffer.cpp  unit)
add_project_test(test_pipeline    integration/test_pipeline.cpp integration)
```

The `TEST_DATA_DIR` define solves a problem that bites every test suite
eventually: **tests must not depend on the current working directory.**

```cpp
TEST(PipelineTest, ParsesValidSampleFile) {
    const std::string path = std::string(TEST_DATA_DIR) + "/valid.csv";
    auto rows = parse_file(path);
    EXPECT_EQ(rows.size(), 42u);
}
```

Without it, `ctest` from the build directory works and `ctest` from the project
root doesn't — an "it works on my machine" failure with no interesting cause.

## 3. Test labels — organizing a growing suite

Labels let you slice the suite by speed, level, or subsystem.

```cmake
# Via the helper above:
add_project_test(test_calculator unit/test_calculator.cpp unit)

# Or directly on an existing test:
set_tests_properties(SlowSoakTest PROPERTIES LABELS "slow;integration")
```

```bash
ctest --test-dir build -L unit                  # only unit-labelled tests
ctest --test-dir build -L integration           # only integration
ctest --test-dir build -LE slow                 # exclude slow tests
ctest --test-dir build --print-labels           # list all labels
```

A labelling scheme that scales:

| Label | Meaning | When it runs |
|---|---|---|
| `unit` | Fast, isolated | Every build, every commit |
| `integration` | Multiple components, may touch the filesystem | Every commit |
| `system` | Full binary, end to end | Pre-merge |
| `slow` | Anything over ~1 second | Nightly |
| `hardware` | Needs a physical device attached | Manually, on the bench |

## 4. Running tests — the CTest command set

```bash
# Everything, showing output only for failures (the everyday command)
ctest --test-dir build --output-on-failure

# List without running
ctest --test-dir build -N

# By name regex
ctest --test-dir build -R RingBuffer            # include
ctest --test-dir build -E Slow                  # exclude

# By label
ctest --test-dir build -L unit

# Parallel -- huge win once you have hundreds of tests
ctest --test-dir build -j $(nproc)

# Only what failed last time
ctest --test-dir build --rerun-failed --output-on-failure

# Stop on the first failure
ctest --test-dir build --stop-on-failure

# Hunt flakiness
ctest --test-dir build --repeat until-fail:100
ctest --test-dir build --repeat until-pass:3    # tolerate known-flaky (record why!)

# Show timing, slowest last
ctest --test-dir build --output-on-failure --test-output-size-passed 0 \
      --no-tests=error
```

`--no-tests=error` is worth adding in CI: without it, a misconfigured build that
registers *zero* tests reports success. A green pipeline that ran nothing is
the most dangerous state a test suite can be in.

### Reading CTest output

```
Test project /home/you/myproject/build
      Start  1: CalculatorTest.AddsTwoPositiveNumbers
 1/12 Test  #1: CalculatorTest.AddsTwoPositiveNumbers ......   Passed    0.01 sec
      Start  2: CalculatorTest.DivisionTruncatesTowardZero
 2/12 Test  #2: CalculatorTest.DivisionTruncatesTowardZero ..   Passed    0.01 sec
      Start  3: RingBufferTest.WriteBeyondCapacityIsTruncated
 3/12 Test  #3: RingBufferTest.WriteBeyondCapacityIsTruncated  ***Failed  0.01 sec
/home/you/myproject/tests/unit/test_ring_buffer.cpp:44: Failure
Expected equality of these values:
  buffer.write(big.data(), big.size())
    Which is: 100
  64

...

92% tests passed, 1 tests failed out of 12

Total Test time (real) =   0.18 sec

The following tests FAILED:
      3 - RingBufferTest.WriteBeyondCapacityIsTruncated (Failed)
Errors while running CTest
```

The exit code is non-zero, which is what CI checks. The `--rerun-failed` flag
then re-runs exactly test 3 while you fix it.

### Timeouts

A hung test blocks the whole pipeline forever. Set a timeout.

```cmake
# Per test
set_tests_properties(SlowSoakTest PROPERTIES TIMEOUT 300)

# Default for everything in this directory
set(CTEST_TEST_TIMEOUT 60)
```

```bash
ctest --test-dir build --timeout 30    # command-line override
```

Deadlocks in concurrent C++ code are exactly the failure mode this catches, and
"the CI job ran for six hours and was killed" is a much worse diagnostic than
"test X timed out after 30 seconds".

## 5. Multiple build configurations

The same source, built and tested several ways. This is the mechanical core of
C/C++ testing.

```bash
# Debug -- unoptimized, asserts active
cmake -S . -B build-debug -DCMAKE_BUILD_TYPE=Debug
cmake --build build-debug -j
ctest --test-dir build-debug --output-on-failure

# Release -- optimized, NDEBUG
cmake -S . -B build-release -DCMAKE_BUILD_TYPE=Release
cmake --build build-release -j
ctest --test-dir build-release --output-on-failure

# A different compiler
cmake -S . -B build-clang -DCMAKE_CXX_COMPILER=clang++ -DCMAKE_BUILD_TYPE=Release
cmake --build build-clang -j
ctest --test-dir build-clang --output-on-failure
```

Three build directories, one source tree, three independent verdicts. A test
that passes in `build-debug` and fails in `build-release` is a strong signal of
undefined behaviour (Module 1) — and finding that difference *before* release is
the whole point of maintaining separate configurations.

A shell script that runs the matrix:

```bash
#!/usr/bin/env bash
# scripts/run_all_configs.sh
set -euo pipefail

configs=(
    "build-debug:Debug:g++"
    "build-release:Release:g++"
    "build-clang-release:Release:clang++"
)

failed=0
for cfg in "${configs[@]}"; do
    IFS=: read -r dir type compiler <<< "$cfg"
    echo "=== $dir ($type, $compiler) ==="
    cmake -S . -B "$dir" -DCMAKE_BUILD_TYPE="$type" \
          -DCMAKE_CXX_COMPILER="$compiler" > /dev/null
    cmake --build "$dir" -j
    if ! ctest --test-dir "$dir" --output-on-failure --no-tests=error; then
        echo "FAILED in $dir"
        failed=1
    fi
done

exit $failed
```

`set -euo pipefail` matters here: without `-e`, a failing build would be
followed by ctest running the *previous* binaries and reporting success.

## 6. A test build preset

CMake presets (3.19+) capture these configurations in a file so nobody has to
remember the flags.

`CMakePresets.json`:

```json
{
  "version": 3,
  "configurePresets": [
    {
      "name": "debug",
      "binaryDir": "${sourceDir}/build-debug",
      "cacheVariables": { "CMAKE_BUILD_TYPE": "Debug" }
    },
    {
      "name": "release",
      "binaryDir": "${sourceDir}/build-release",
      "cacheVariables": { "CMAKE_BUILD_TYPE": "Release" }
    }
  ],
  "testPresets": [
    {
      "name": "debug",
      "configurePreset": "debug",
      "output": { "outputOnFailure": true },
      "execution": { "noTestsAction": "error", "stopOnFailure": false }
    },
    {
      "name": "unit-only",
      "configurePreset": "debug",
      "filter": { "include": { "label": "unit" } },
      "output": { "outputOnFailure": true }
    }
  ]
}
```

```bash
cmake --preset debug
cmake --build build-debug
ctest --preset debug
ctest --preset unit-only
```

## 7. Machine-readable results

CI systems want structured output, not console text.

```bash
# GoogleTest's own XML (JUnit-compatible -- most CI systems parse this)
./build/tests/test_calculator --gtest_output=xml:results/calculator.xml

# CTest's XML, plus a JUnit conversion
ctest --test-dir build --output-junit results/ctest-junit.xml
```

```xml
<?xml version="1.0" encoding="UTF-8"?>
<testsuites tests="12" failures="1" disabled="0" errors="0" time="0.18">
  <testsuite name="RingBufferTest" tests="6" failures="1" time="0.05">
    <testcase name="StartsEmpty" status="run" time="0.01"
              classname="RingBufferTest" />
    <testcase name="WriteBeyondCapacityIsTruncated" status="run" time="0.01"
              classname="RingBufferTest">
      <failure message="Expected equality of these values:
  buffer.write(big.data(), big.size())
    Which is: 100
  64" type="" />
    </testcase>
  </testsuite>
</testsuites>
```

This is what produces the "1 test failed" annotation on a pull request rather
than requiring someone to read 4,000 lines of build log. Level 3's CI module
wires it up properly.

## 8. Keeping a suite fast

A slow suite stops being run, and a suite that isn't run has no value. Some
discipline:

| Symptom | Cause | Fix |
|---|---|---|
| Unit tests take minutes | They're touching disk, network, or `sleep()` | Fake the dependency (Level 2 M7); replace `sleep` with an injectable clock |
| Suite time grows linearly with test count | Tests run serially | `ctest -j $(nproc)`; ensure tests don't share files |
| One test dominates the runtime | It's an integration test mislabelled as a unit test | Label it, move it out of the fast set |
| Rebuilds are slow | Everything includes everything | Reduce header coupling; that's a design finding worth reporting |

Measure before optimizing:

```bash
ctest --test-dir build -j 8 2>&1 | tail -20
# Total Test time (real) =  12.44 sec

# Which tests are slowest? gtest reports per-test timing:
./build/tests/test_pipeline --gtest_print_time=1
```

Rough budgets that keep a suite usable:

| Suite | Target | Consequence of exceeding it |
|---|---|---|
| Smoke | < 60 s | Nobody runs it before pushing |
| Unit | < 2 min | Developers stop running it locally |
| Full pre-merge | < 15 min | Pull requests queue up |
| Nightly | Hours are fine | — |

## 9. Handling tests that must be skipped

Sometimes a test can't run — no hardware attached, a feature disabled at
compile time. Skip it *visibly*; never delete it or comment it out.

```cpp
TEST(SerialPortTest, ReadsFromDevice) {
    if (!device_present("/dev/ttyUSB0")) {
        GTEST_SKIP() << "no serial device attached";
    }
    // ... real test ...
}
```

```
[ RUN      ] SerialPortTest.ReadsFromDevice
tests/integration/test_serial.cpp:14: Skipped
no serial device attached
[  SKIPPED ] SerialPortTest.ReadsFromDevice (0 ms)
```

CTest equivalent, driven by the exit code:

```cmake
set_tests_properties(HardwareTest PROPERTIES SKIP_RETURN_CODE 77)
```

The visibility is the point. A commented-out test is invisible; a skipped test
appears in every report with its reason, so "we haven't been running the
hardware tests for four months" is a fact somebody notices.

!!! warning "Disabled tests rot"
    GoogleTest lets you prefix a test with `DISABLED_` to exclude it. Run
    `--gtest_also_run_disabled_tests` periodically, and treat a long-disabled
    test as a defect in its own right. A test suite's credibility depends on
    green meaning "everything we claim to test, passes."

## 10. Suite health checklist

Run this against any C/C++ project's test setup:

- [ ] `ctest` runs from a clean clone with documented commands
- [ ] Zero tests registered is treated as an error (`--no-tests=error`)
- [ ] Tests are independent — the suite passes with `--gtest_shuffle` and `-j`
- [ ] No test depends on the current working directory
- [ ] Every test has a timeout
- [ ] Tests are labelled by level and speed
- [ ] Unit suite runs in under two minutes
- [ ] The suite runs in Debug **and** Release
- [ ] Results are exported as JUnit XML for CI
- [ ] Skipped and disabled tests are listed and reviewed
- [ ] Adding a new test file is a one-line change

## Exercise

Take the project you built across Modules 6–8 and turn it into a properly
organized suite.

1. **Restructure** it to the layout in section 1: `include/`, `src/`,
   `tests/unit/`, `tests/integration/`, `tests/data/`. Use the
   `add_project_test()` helper from section 2 so each test file costs one line.

2. **Add a data-driven integration test.** Create `tests/data/valid.csv` and
   `tests/data/malformed.csv`, add a function to your library that reads a CSV
   of integers and returns their mean, and write a test in
   `tests/integration/` that reads both files via `TEST_DATA_DIR`. Confirm the
   test passes when run from **three different working directories** (the
   project root, `build/`, and `/tmp`).

3. **Label everything.** Give unit tests the `unit` label and integration tests
   `integration`. Verify with `ctest --test-dir build --print-labels`, then show
   that `ctest -L unit` and `ctest -L integration` each run the right subset.

4. **Build the matrix.** Write `scripts/run_all_configs.sh` based on section 5,
   covering Debug/g++, Release/g++, and Release/clang++ (drop the clang entry if
   clang isn't installed and note that in a comment). Run it and record the
   total time and results for each configuration.

5. **Add a timeout and prove it fires.** Write a test that deliberately loops
   forever, set `TIMEOUT 5` on it, and confirm CTest reports it as
   `***Timeout` rather than hanging. Then remove it.

6. **Prove independence.** Run `ctest -j 8 --repeat until-fail:20`. If anything
   fails, find the shared state — a shared temp filename between two tests is
   the usual culprit — and fix it.

7. **Export results.** Produce `results/ctest-junit.xml` with
   `--output-junit` and confirm it contains one `<testcase>` element per test.

8. **Complete the checklist** in section 10 for your project, and write one
   sentence per unticked box explaining what would need to change.
