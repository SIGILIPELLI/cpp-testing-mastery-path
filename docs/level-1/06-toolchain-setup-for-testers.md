# 06 · Toolchain Setup for Testers

From here on, this course writes code. This module gets a working C/C++ build
and test environment onto your machine, and — just as importantly — teaches the
compiler flags that are themselves a testing tool. Warnings are the cheapest
defect detector in existence: they cost nothing at runtime and find bugs before
the program ever runs.

## 1. Install a compiler

You need a compiler supporting at least C11 and C++17.

```bash
# macOS -- Xcode command line tools (provides clang and clang++)
xcode-select --install

# Ubuntu/Debian
sudo apt update
sudo apt install build-essential      # gcc, g++, make

# Fedora/RHEL
sudo dnf install gcc gcc-c++ make

# Windows -- MSYS2 (recommended for following this course), then in the
# MSYS2 UCRT64 shell:
#   pacman -S mingw-w64-ucrt-x86_64-gcc mingw-w64-ucrt-x86_64-cmake
```

Verify:

```bash
gcc --version
# gcc (Ubuntu 13.2.0-23ubuntu4) 13.2.0

g++ --version
# g++ (Ubuntu 13.2.0-23ubuntu4) 13.2.0

clang --version
# Ubuntu clang version 18.1.3
```

!!! tip "Install both GCC and Clang if you can"
    They disagree. Code that is undefined behaviour may work under one and break
    under the other, and their warning sets only partially overlap — each finds
    things the other misses. Building your test suite under both compilers is
    one of the cheapest quality wins available in C/C++, and Level 3's CI module
    automates exactly that.

## 2. Install CMake and a test runner

```bash
# Ubuntu/Debian
sudo apt install cmake

# macOS (Homebrew)
brew install cmake

# Fedora
sudo dnf install cmake
```

```bash
cmake --version
# cmake version 3.28.3   -- 3.14 or newer is required for this course

ctest --version
# ctest version 3.28.3   -- ships with CMake
```

CMake 3.14+ matters because that's when `FetchContent` became convenient, which
is how Module 7 pulls in GoogleTest without you having to install it manually.

## 3. Warnings are your first test suite

A compiler warning is a test that runs at build time. Turn on as many as you can
tolerate and treat them as failures.

| Flag | What it does |
|---|---|
| `-Wall` | The common, high-value warnings. Despite the name, *not* all warnings. |
| `-Wextra` | A second batch — unused parameters, sign comparisons, missing field initializers |
| `-Wpedantic` | Warn about anything not strictly conforming to the standard |
| `-Werror` | Turn every warning into a hard error. Use in CI. |
| `-Wshadow` | A local variable shadows another — a genuine bug source |
| `-Wconversion` | Implicit conversions that may lose value (noisy but finds real bugs) |
| `-Wsign-compare` | Comparing signed and unsigned — the classic `i < vec.size()` trap |
| `-Wformat=2` | `printf` format string doesn't match its arguments |
| `-Wnull-dereference` | A pointer that can provably be null is dereferenced |
| `-Wuninitialized` | Use of a variable before assignment (needs `-O1`+ to be effective) |
| `-g` | Emit debug symbols — required for readable stack traces and sanitizer output |

The baseline this course uses:

```bash
# C
gcc -std=c11 -Wall -Wextra -g -O0 -o prog prog.c

# C++
g++ -std=c++17 -Wall -Wextra -g -O0 -o prog prog.cpp
```

### See it catch a real bug

`warn_demo.c`:

```c
#include <stdio.h>
#include <string.h>

int main(void) {
    char buf[8];
    int  n;                                  /* never initialized */
    strcpy(buf, "this string is far too long");   /* overflows buf */
    printf("%d\n", n);                       /* reads uninitialized n */
    printf("%s\n", buf);
    return 0;
}
```

Compile with warnings on:

```bash
gcc -std=c11 -Wall -Wextra -O1 -g -o warn_demo warn_demo.c
```

```
warn_demo.c: In function 'main':
warn_demo.c:7:5: warning: 'strcpy' writing 28 bytes into a region of size 8
                 overflows the destination [-Wstringop-overflow=]
    7 |     strcpy(buf, "this string is far too long");
      |     ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
warn_demo.c:8:5: warning: 'n' is used uninitialized [-Wuninitialized]
    8 |     printf("%d\n", n);
      |     ^~~~~~~~~~~~~~~~~
```

Two serious bugs found in under a second, without running anything. Now compile
the same file with warnings off:

```bash
gcc -std=c11 -o warn_demo_quiet warn_demo.c    # silence
./warn_demo_quiet
# 0
# this string is far too long
```

It *appears* to work. It has already corrupted the stack. This is the C/C++
testing problem in miniature, and it's why the first thing a tester on a C/C++
project should check is whether the build is warning-clean.

!!! warning "`-Wuninitialized` needs optimization enabled"
    GCC's uninitialized-variable analysis runs as part of the optimizer, so at
    `-O0` it finds almost nothing. Build your warning-check pass at `-O1` or
    `-O2` even if you debug at `-O0`.

## 4. Debug builds vs. release builds

You must test both. They are different programs.

| | Debug (`-O0 -g`) | Release (`-O2` / `-O3`) |
|---|---|---|
| Optimization | None | Aggressive |
| Debuggability | Variables inspectable, line numbers exact | Variables optimized away, lines reordered |
| Speed | Slow | Fast |
| UB behaviour | Often "does the naive thing" | Optimizer exploits UB; behaviour changes |
| `assert()` | Active | Active unless `NDEBUG` is defined |
| Uninitialized memory | Often happens to be zero | Often garbage |

```bash
# Debug
g++ -std=c++17 -Wall -Wextra -g -O0 -o prog_debug prog.cpp

# Release
g++ -std=c++17 -Wall -Wextra -O2 -DNDEBUG -o prog_release prog.cpp
```

A test plan that only exercises the debug build has tested a program you do not
ship.

!!! warning "`NDEBUG` disables every `assert()`"
    `-DNDEBUG` compiles all `assert()` calls out entirely. If a developer put
    logic inside an assert — `assert(init_hardware() == 0);` — that logic
    **disappears in release builds**. Grepping for `assert(` calls containing
    function calls with side effects is a genuinely productive five-minute
    review task on any C/C++ codebase.

## 5. CMake basics for test targets

Hand-typing `g++` commands stops scaling around file number three. CMake
generates the build for you and — crucially for testers — gives you `ctest`.

### Minimal project

```
myproject/
├── CMakeLists.txt
├── src/
│   ├── calculator.h
│   └── calculator.cpp
└── tests/
    └── test_calculator.cpp
```

`src/calculator.h`:

```cpp
#pragma once

class Calculator {
public:
    int add(int a, int b) const;
    int divide(int a, int b) const;   // throws std::invalid_argument on b == 0
};
```

`src/calculator.cpp`:

```cpp
#include "calculator.h"
#include <stdexcept>

int Calculator::add(int a, int b) const {
    return a + b;
}

int Calculator::divide(int a, int b) const {
    if (b == 0) {
        throw std::invalid_argument("division by zero");
    }
    return a / b;
}
```

`CMakeLists.txt`:

```cmake
cmake_minimum_required(VERSION 3.14)
project(myproject LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Warnings for every target in this project
add_compile_options(-Wall -Wextra)

# The library under test
add_library(calculator src/calculator.cpp)
target_include_directories(calculator PUBLIC src)

# Enables the `ctest` command and the add_test() function below
enable_testing()

add_subdirectory(tests)
```

`tests/CMakeLists.txt`:

```cmake
add_executable(test_calculator test_calculator.cpp)
target_link_libraries(test_calculator PRIVATE calculator)

# Register the executable with CTest under the name "CalculatorTests"
add_test(NAME CalculatorTests COMMAND test_calculator)
```

`tests/test_calculator.cpp` — a hand-rolled test for now; Module 7 replaces this
with GoogleTest:

```cpp
#include "calculator.h"
#include <cstdio>
#include <stdexcept>

static int failures = 0;

static void check(bool condition, const char* what) {
    if (condition) {
        std::printf("[ PASS ] %s\n", what);
    } else {
        std::printf("[ FAIL ] %s\n", what);
        ++failures;
    }
}

int main() {
    Calculator c;

    check(c.add(2, 3) == 5, "add(2,3) == 5");
    check(c.add(-1, 1) == 0, "add(-1,1) == 0");
    check(c.divide(7, 2) == 3, "divide(7,2) truncates to 3");

    bool threw = false;
    try {
        c.divide(1, 0);
    } catch (const std::invalid_argument&) {
        threw = true;
    }
    check(threw, "divide(1,0) throws invalid_argument");

    std::printf("\n%d failure(s)\n", failures);
    return failures == 0 ? 0 : 1;   // non-zero exit == test failure
}
```

### Configure, build, run

```bash
cmake -S . -B build              # configure: generate the build system
cmake --build build              # compile
ctest --test-dir build --output-on-failure
```

```
Test project /home/you/myproject/build
    Start 1: CalculatorTests
1/1 Test #1: CalculatorTests ..................   Passed    0.00 sec

100% tests passed, 0 tests failed out of 1

Total Test time (real) =   0.01 sec
```

Three things just happened that matter for the rest of the course:

- `cmake -S . -B build` keeps all generated files in `build/` — an
  **out-of-source build**. Never build in your source tree; `rm -rf build` must
  be a safe, complete reset.
- `add_test()` registered the executable with CTest under a name.
- **CTest decides pass/fail from the process exit code.** Zero = pass, non-zero
  = pass fails. That's the entire contract, which is why the hand-rolled harness
  above returns `failures == 0 ? 0 : 1`.

### Build types

```bash
cmake -S . -B build-debug   -DCMAKE_BUILD_TYPE=Debug     # -g -O0
cmake -S . -B build-release -DCMAKE_BUILD_TYPE=Release   # -O3 -DNDEBUG
cmake --build build-release
ctest --test-dir build-release --output-on-failure
```

Two build directories, two configurations, same test suite. This is the
mechanical form of "test both builds" from section 4.

## 6. Essential CTest commands

| Command | Purpose |
|---|---|
| `ctest --test-dir build` | Run all registered tests |
| `ctest --test-dir build --output-on-failure` | Show output for failing tests (use this by default) |
| `ctest --test-dir build -V` | Verbose — show all output always |
| `ctest --test-dir build -N` | List tests without running them |
| `ctest --test-dir build -R Calc` | Run only tests whose name matches the regex `Calc` |
| `ctest --test-dir build -E Slow` | Exclude tests matching `Slow` |
| `ctest --test-dir build --rerun-failed` | Re-run only what failed last time |
| `ctest --test-dir build -j 8` | Run 8 tests in parallel |
| `ctest --test-dir build --repeat until-fail:50` | Run 50 times — the flaky-test hunter |

That last one is genuinely valuable for C/C++: races and uninitialized-memory
bugs are intermittent, and `--repeat until-fail:100` converts "it failed once
last Tuesday" into a reproducible defect report (Module 4).

## 7. Debugging tools a tester should have

You do not need to be a debugger expert, but you do need to produce a stack
trace for a crash report.

```bash
# Ubuntu/Debian
sudo apt install gdb valgrind

# macOS -- lldb ships with Xcode command line tools
```

Minimum competence:

```bash
gdb --args ./build/test_calculator
(gdb) run          # runs until it exits or crashes
(gdb) bt           # backtrace -- the call stack at the crash
(gdb) bt full      # backtrace plus local variables
(gdb) quit
```

That's enough to fill in the "Attachments" field of a crash report. Valgrind
and the sanitizers get their own modules in Level 2.

## 8. A tester's project checklist

When you join a C/C++ project, work through this before writing a single test
case:

- [ ] Does the project build from a clean checkout with documented commands?
- [ ] Is the build **warning-clean** with `-Wall -Wextra`? If not, how many
      warnings? (Baseline it — a growing count is a quality signal.)
- [ ] Are both Debug and Release configurations buildable and tested?
- [ ] Is there a test target, and does `ctest` find it?
- [ ] Which compiler(s) and version(s) are officially supported?
- [ ] Which target architectures ship?
- [ ] Is there a documented way to run a single test?
- [ ] Do tests run in CI on every commit? On which configurations?
- [ ] Can you produce a stack trace from a crash?
- [ ] Are sanitizer builds available (`-fsanitize=address`)?

Every "no" on that list is a finding worth raising — several of them are more
valuable than any individual test case you could write that week.

## Exercise

Build the project from section 5 on your own machine, then extend it:

1. **Set it up.** Create the directory structure, the three source files, and
   both `CMakeLists.txt` files. Confirm that
   `ctest --test-dir build --output-on-failure` reports 1 test passing.

2. **Make it fail deliberately.** Add a check to `test_calculator.cpp` asserting
   that `c.divide(7, 2) == 4` (wrong — integer division truncates). Rebuild, run
   ctest, and confirm two things: that the harness prints `[ FAIL ]`, and that
   **ctest reports the test as failed**. Then verify *why* — run
   `./build/tests/test_calculator; echo $?` and confirm the exit code is 1.
   Fix the check afterwards.

3. **Add a warnings finding.** Introduce a deliberate bug into
   `calculator.cpp` — declare `int unused_total;` and return it from a new
   method without assigning it. Rebuild and record the exact warning text. Then
   add `-Werror` to `add_compile_options()` in the top-level `CMakeLists.txt`
   and confirm the build now *fails* rather than warns. Remove the bug, keep
   `-Werror`.

4. **Two build types.** Configure `build-debug` and `build-release` as in
   section 5. Run the suite in both. Then add a `Calculator::overflow_add(int,
   int)` that simply returns `a + b`, and write a check asserting
   `overflow_add(INT_MAX, 1) == INT_MIN`. Run it in both build directories and
   record the results. Write two sentences explaining what you observed and why
   this test case is a bad test regardless of whether it passed (refer back to
   Module 5, section 3).

5. **Complete the checklist** in section 8 against your own project and write
   down which boxes you cannot tick yet.
