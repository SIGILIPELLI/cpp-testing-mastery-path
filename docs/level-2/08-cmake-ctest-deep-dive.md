# 08 · CMake/CTest Deep Dive

Level 1 used CMake as a build script that happened to produce a test binary.
That stops scaling around the point you want a sanitizer build, a coverage
build, tests that must run in a fixed order, and a CI job that reports each test
separately. This module treats CMake as the thing that defines your test
*topology*, and CTest as the runner that exploits it.

## 1. Targets, not variables

Modern CMake attaches requirements to targets and propagates them. `PUBLIC`
means "me and anyone who links me"; `PRIVATE` means "me only"; `INTERFACE`
means "consumers only".

```cmake
cmake_minimum_required(VERSION 3.20)
project(textkit LANGUAGES CXX)
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

add_library(textkit src/textkit.cpp)
target_include_directories(textkit PUBLIC include)   # consumers get include/
target_compile_options(textkit PRIVATE -Wall -Wextra)
```

Because the include directory is `PUBLIC`, the test target gets it for free by
linking. Because the warning flags are `PRIVATE`, they don't leak into consumers
who may have different standards. Global `include_directories()` and
`CMAKE_CXX_FLAGS` do neither, which is why they cause trouble at scale.

## 2. Registering tests properly

```cmake
find_package(GTest REQUIRED)
enable_testing()             # must be in the TOP-LEVEL CMakeLists
add_subdirectory(tests)
```

```cmake
# tests/CMakeLists.txt
add_executable(test_textkit test_textkit.cpp)
target_link_libraries(test_textkit PRIVATE textkit GTest::gtest_main)

include(GoogleTest)
gtest_discover_tests(test_textkit PROPERTIES LABELS unit)
```

```bash
cmake -S . -B build -DCMAKE_PREFIX_PATH=/opt/homebrew/opt/googletest
cmake --build build
ctest --test-dir build --output-on-failure
```

```
Test project /path/to/cmproj/build
    Start 1: WordCount.CountsSimpleSentence
1/3 Test #1: WordCount.CountsSimpleSentence ...   Passed    0.00 sec
    Start 2: WordCount.EmptyStringIsZero
2/3 Test #2: WordCount.EmptyStringIsZero ......   Passed    0.00 sec
    Start 3: LongestWord.PicksFirstOnTie
3/3 Test #3: LongestWord.PicksFirstOnTie ......   Passed    0.00 sec

100% tests passed out of 3

Label Time Summary:
unit    =   0.01 sec*proc (3 tests)

Total Test time (real) =   0.01 sec
```

Three CTest tests from one binary. Compare with `add_test(NAME all COMMAND
test_textkit)`, which gives CTest a single opaque test: no per-test timing, no
`ctest -R WordCount`, and a failure report that names the binary rather than the
case.

!!! warning "`enable_testing()` must be top-level"
    Calling it only inside `tests/CMakeLists.txt` produces a `CTestTestfile.cmake`
    in the subdirectory but not at the build root, and `ctest --test-dir build`
    cheerfully reports **"No tests were found!!!"** while exiting **0**. A CI job
    that just runs ctest will go green with zero tests executed. Guard against it
    with `ctest --test-dir build -N` and a check that the count is non-zero.

`gtest_discover_tests` runs the binary at build time with `--gtest_list_tests`.
That means cross-compiling breaks it — the host cannot run the target binary. Use
`DISCOVERY_MODE PRE_TEST` to defer discovery to test time on the target, or fall
back to `add_test` for cross builds.

## 3. Labels and selection

Labels are how you carve a suite into "fast enough for every save" and
"everything".

```cmake
gtest_discover_tests(test_textkit PROPERTIES LABELS unit)
add_test(NAME integration.round_trip COMMAND round_trip_test)
set_tests_properties(integration.round_trip PROPERTIES LABELS "integration;slow")
```

```bash
ctest --test-dir build -L unit                 # only unit
ctest --test-dir build -LE slow                # exclude slow
ctest --test-dir build -R 'WordCount'          # regex on test name
ctest --test-dir build -E 'Flaky'              # exclude by name
ctest --test-dir build -N                      # list, don't run
```

```
Test project /path/to/cmproj/build
  Test #1: WordCount.CountsSimpleSentence
  Test #2: WordCount.EmptyStringIsZero
  Test #3: LongestWord.PicksFirstOnTie

Total Tests: 3
```

## 4. Test properties that earn their keep

```cmake
set_tests_properties(integration.round_trip PROPERTIES
    TIMEOUT 30                       # kill and fail after 30s
    LABELS "integration"
    WORKING_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}/data"
    ENVIRONMENT "TZ=UTC;LC_ALL=C"    # kill locale/timezone flakiness
    RUN_SERIAL TRUE                  # never parallel with anything else
)

set_tests_properties(cli.rejects_bad_flag PROPERTIES WILL_FAIL TRUE)
set_tests_properties(cli.reports_usage    PROPERTIES
    PASS_REGULAR_EXPRESSION "usage: textkit")
set_tests_properties(parser.no_warnings   PROPERTIES
    FAIL_REGULAR_EXPRESSION "warning:")
```

`ENVIRONMENT "TZ=UTC;LC_ALL=C"` deserves special mention: date formatting and
string collation are the two most common sources of "passes on my machine".

Fixtures express setup/teardown *between* tests:

```cmake
add_test(NAME db.setup    COMMAND db_tool --create)
add_test(NAME db.teardown COMMAND db_tool --drop)
set_tests_properties(db.setup    PROPERTIES FIXTURES_SETUP    DB)
set_tests_properties(db.teardown PROPERTIES FIXTURES_CLEANUP  DB)
set_tests_properties(query.finds_user PROPERTIES FIXTURES_REQUIRED DB)
```

CTest now runs `db.setup` before any `DB` test, `db.teardown` after — and skips
both if you filter to tests that don't need `DB`.

!!! warning "`WILL_FAIL` inverts, it does not validate"
    `WILL_FAIL TRUE` passes if the process exits non-zero — including a segfault,
    a missing shared library, or a typo in the command. Prefer
    `PASS_REGULAR_EXPRESSION` so you assert *why* it failed.

## 5. Build variants for instrumentation

One source tree, several build directories. This is the pattern Modules 4, 5 and
6 all plug into.

```cmake
option(ENABLE_SANITIZERS "Build with ASan + UBSan" OFF)
option(ENABLE_COVERAGE   "Build with gcov instrumentation" OFF)

if(ENABLE_SANITIZERS)
  add_compile_options(-fsanitize=address,undefined -fno-sanitize-recover=all
                      -fno-omit-frame-pointer -g -O1)
  add_link_options(-fsanitize=address,undefined)
endif()
if(ENABLE_COVERAGE)
  add_compile_options(--coverage -O0 -g)
  add_link_options(--coverage)
endif()
```

```bash
cmake -S . -B build              -DCMAKE_BUILD_TYPE=Debug
cmake -S . -B build-asan         -DENABLE_SANITIZERS=ON
cmake -S . -B build-cov          -DENABLE_COVERAGE=ON
cmake -S . -B build-release      -DCMAKE_BUILD_TYPE=Release

cmake --build build-asan && ctest --test-dir build-asan --output-on-failure
```

```
100% tests passed out of 3

Label Time Summary:
unit    =   0.17 sec*proc (3 tests)

Total Test time (real) =   0.18 sec
```

Note the timing: the same three tests took 0.01 s unsanitized and 0.17 s under
ASan — the expected order-of-magnitude cost, and the reason these live in
separate directories rather than in your default build.

!!! warning "`add_compile_options` must precede the targets"
    Directory-level commands apply to targets created *after* them. Put the
    `option()`/`if()` blocks above your `add_library`/`add_subdirectory` calls or
    the flags silently do nothing. A sanitizer build that finds no bugs and runs
    at full speed is a build with no sanitizer in it — check with
    `cmake --build build-asan --verbose | head`.

## 6. Presets — stop retyping the flags

`CMakePresets.json` puts the matrix in version control:

```json
{
  "version": 6,
  "configurePresets": [
    { "name": "debug", "binaryDir": "build",
      "cacheVariables": { "CMAKE_BUILD_TYPE": "Debug" } },
    { "name": "asan", "inherits": "debug", "binaryDir": "build-asan",
      "cacheVariables": { "ENABLE_SANITIZERS": "ON" } },
    { "name": "cov", "inherits": "debug", "binaryDir": "build-cov",
      "cacheVariables": { "ENABLE_COVERAGE": "ON" } }
  ],
  "buildPresets": [ { "name": "asan", "configurePreset": "asan" } ],
  "testPresets": [
    { "name": "asan", "configurePreset": "asan",
      "output": { "outputOnFailure": true } }
  ]
}
```

```bash
cmake --preset asan && cmake --build --preset asan && ctest --preset asan
```

## 7. Running the suite

```bash
ctest --test-dir build -j8                     # parallel
ctest --test-dir build --output-on-failure     # logs only for failures
ctest --test-dir build --rerun-failed          # just what broke last time
ctest --test-dir build --repeat until-fail:20  # hunt flakes
ctest --test-dir build --repeat after-timeout:3
ctest --test-dir build --timeout 60            # global cap
ctest --test-dir build --schedule-random       # shake out inter-test order deps
ctest --test-dir build --output-junit res.xml  # CI-consumable
```

`-j8` plus `--schedule-random` is the combination that surfaces hidden coupling:
tests sharing a temp file path or a fixed port will start failing intermittently,
which is a bug you want to find locally rather than in CI. Mark genuinely
exclusive tests `RUN_SERIAL TRUE` rather than dropping back to `-j1`.

## 8. Reference

| Goal | Command or property |
|---|---|
| List tests without running | `ctest -N` |
| Run a subset by name | `ctest -R <regex>` |
| Run a subset by label | `ctest -L <label>` |
| Exclude by label | `ctest -LE <label>` |
| Show output for failures only | `ctest --output-on-failure` |
| Re-run only failures | `ctest --rerun-failed` |
| Hunt flaky tests | `ctest --repeat until-fail:20 --schedule-random` |
| Parallelise | `ctest -j$(nproc)` |
| Per-test time limit | `TIMEOUT` property |
| Expect a non-zero exit | `WILL_FAIL TRUE` |
| Assert on output text | `PASS_REGULAR_EXPRESSION` |
| Pin locale/timezone | `ENVIRONMENT "TZ=UTC;LC_ALL=C"` |
| Order-dependent setup | `FIXTURES_SETUP` / `FIXTURES_REQUIRED` |
| Never run in parallel | `RUN_SERIAL TRUE` |
| Machine-readable report | `--output-junit res.xml` |

## Exercise

Take a small library of your own — the `textkit` above, or your Level 1 project.

1. **Convert `add_test` to `gtest_discover_tests`.** Confirm with `ctest -N` that
   the count goes from 1 to the number of `TEST()` cases, and that
   `ctest -R <one test name>` runs exactly one.

2. **Reproduce the silent-zero-tests failure.** Move `enable_testing()` out of
   the top-level file into `tests/CMakeLists.txt`. Run `ctest --test-dir build`
   and record both the message and `echo $?`. Then write the one-line guard that
   would have caught it in CI.

3. **Label and split.** Add a deliberately slow test (`sleep 3` or a large
   input). Label it `slow`, and show that `ctest -LE slow` finishes in well under
   a second while the full run does not. Report both wall times.

4. **Add all four build variants** from section 5 — debug, release, asan, cov —
   and confirm each configures, builds, and passes. Record the `ctest` wall time
   for debug versus asan and comment on the ratio you observe.

5. **Prove the ordering trap.** Verify that moving `add_compile_options(-fsanitize=...)`
   *below* `add_subdirectory(tests)` makes the sanitizer build silently normal.
   Show the evidence from `cmake --build build-asan --verbose`.

6. **Add properties that matter.** Give your integration test a `TIMEOUT`, pin
   `TZ=UTC` and `LC_ALL=C`, and add one `PASS_REGULAR_EXPRESSION` test for a CLI
   error message. Confirm the timeout fires by temporarily sleeping past it.

7. **Hunt a flake.** Introduce shared state between two tests (a fixed temp file
   path). Show that `ctest -j8 --schedule-random --repeat until-fail:20` catches
   it and a plain serial `ctest` does not. Fix it, then write a
   `CMakePresets.json` that makes the whole matrix a one-liner.
