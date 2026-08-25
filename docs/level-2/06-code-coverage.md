# 06 · Code Coverage (gcov/lcov)

Coverage answers one question: which lines did the test suite execute? That is
useful and it is narrow. Coverage cannot tell you whether the assertions were
meaningful, only whether the code ran. This module shows how to measure it, how
to read the numbers honestly, and — with a worked example — why 100% line
coverage routinely hides untested logic.

## 1. Instrumenting a build

GCC and Clang both implement gcov-style instrumentation behind `--coverage`,
which is shorthand for `-fprofile-arcs -ftest-coverage` at compile time and
`-lgcov` at link time. Pass it to **both** steps.

```c
/* grade.c */
#include "grade.h"
char grade_for(int score) {
    if (score < 0 || score > 100) return '?';
    if (score >= 90) return 'A';
    if (score >= 80) return 'B';
    if (score >= 70) return 'C';
    return 'F';
}
```

```c
/* test_grade.c */
#include <assert.h>
#include <stdio.h>
#include "grade.h"
int main(void) {
    assert(grade_for(95) == 'A');
    assert(grade_for(85) == 'B');
    assert(grade_for(50) == 'F');
    printf("all assertions passed\n");
    return 0;
}
```

```bash
gcc-16 --coverage -O0 -g -c grade.c      -o grade.o
gcc-16 --coverage -O0 -g -c test_grade.c -o test_grade.o
gcc-16 --coverage grade.o test_grade.o   -o test_grade
./test_grade
```

```
all assertions passed
```

Compiling produces `grade.gcno` (the control-flow graph). Running produces
`grade.gcda` (the execution counts). Both are needed.

!!! warning "Compile and link as separate steps"
    Doing it in one command — `gcc --coverage -O0 -g grade.c test_grade.c -o t` —
    names the notes files after the *output binary*, e.g. `t-grade.gcno`, and
    plain `gcov grade.c` then reports `grade.gcno: No such file or directory`.
    This is a real and confusing failure. Compile to `.o` files first, which is
    what CMake and Make do anyway.

Also non-negotiable: **`-O0`**. Optimisation merges, reorders and deletes lines,
so counts stop mapping to your source.

## 2. Reading a gcov report

```bash
gcov-16 grade.c
```

```
File 'grade.c'
Lines executed:100.00% of 6
Creating 'grade.c.gcov'
```

100% line coverage, three test cases, five possible return values. Something is
clearly missing — and line coverage cannot see it. Ask for branches:

```bash
gcov-16 --branch-probabilities grade.c
```

```
File 'grade.c'
Lines executed:100.00% of 6
Branches executed:100.00% of 10
Taken at least once:70.00% of 10
No calls
```

There it is. **100% of lines, 70% of branches.** The annotated source
(`grade.c.gcov`) names the gaps exactly:

```
function grade_for called 3 returned 100% blocks executed 82%
        3:    2:char grade_for(int score) {
       3*:    3:    if (score < 0 || score > 100) return '?';
branch  0 taken 100% (fallthrough)
branch  1 taken 0%
branch  2 taken 0% (fallthrough)
branch  3 taken 100%
        3:    4:    if (score >= 90) return 'A';
branch  0 taken 33% (fallthrough)
branch  1 taken 67%
        2:    5:    if (score >= 80) return 'B';
branch  0 taken 50% (fallthrough)
branch  1 taken 50%
       1*:    6:    if (score >= 70) return 'C';
branch  0 taken 0% (fallthrough)
branch  1 taken 100%
        1:    7:    return 'F';
```

Two findings you could not get from the line number:

- Line 3: `branch 1 taken 0%` and `branch 2 taken 0%` — **the validation path
  never ran.** Nothing tested `grade_for(-5)` or `grade_for(150)`. The `'?'`
  return is dead as far as the suite is concerned.
- Line 6: `branch 0 taken 0%` — **`'C'` is never returned.** No input in the
  70–79 range was tested.

The `*` suffix on `3*:` and `1*:` is gcov's own hint that the line contains at
least one branch that was never taken.

Adding `grade_for(-1)`, `grade_for(101)` and `grade_for(75)` takes branch
coverage to 100% and, more to the point, actually tests two whole behaviours the
original suite ignored.

## 3. What the coverage types measure

| Type | Question | Catches | Misses |
|---|---|---|---|
| Line / statement | Did this line run? | Wholly untested code | Untaken branches, as above |
| Branch / decision | Did each `if` go both ways? | The `'?'` and `'C'` gaps | Which sub-condition mattered |
| Condition | Did each sub-expression take both values? | `score < 0` never true in `a \|\| b` | Combinations |
| MC/DC | Did each condition independently affect the outcome? | Redundant conditions | — |
| Function | Was each function called? | Dead functions | Everything inside them |
| Path | Was each route through the function taken? | Interaction bugs | Impractical: exponential |

For most projects, **branch coverage is the right target**. MC/DC is mandated
for safety-critical work (DO-178C level A, ISO 26262 ASIL D) and is expensive
everywhere else.

## 4. HTML reports with lcov

`gcov` reports one file at a time. `lcov` aggregates, and `genhtml` renders.

```bash
lcov --gcov-tool "$(which gcov-16)" --capture --directory . --output-file cov.info
```

```
Found 2 data files in .
Finished processing 2 GCDA files
Summary coverage rate:
  source files: 2
  lines.......: 100.0% (12 of 12 lines)
  functions...: 100.0% (2 of 2 functions)
```

Filter out noise, then render:

```bash
lcov --remove cov.info '/usr/*' '*/_deps/*' '*/tests/*' --output-file cov.info
genhtml cov.info --branch-coverage --output-directory coverage-html
open coverage-html/index.html      # xdg-open on Linux
```

!!! warning "Match your gcov to your compiler"
    `gcov` files are versioned. Running Apple's `/usr/bin/gcov` against data
    produced by Homebrew GCC 16 gives `version mismatch` or garbage. If you
    compiled with `gcc-16`, pass `--gcov-tool $(which gcov-16)` to lcov. For
    Clang builds, use `llvm-cov gcov` via a small wrapper script, or switch to
    Clang's native `-fprofile-instr-generate -fcoverage-mapping` and
    `llvm-profdata` / `llvm-cov show`.

Also always `--remove` third-party and test code. Test files are executed by
definition, so leaving them in inflates the number and hides gaps in the code
you care about.

## 5. Wiring it into CMake

```cmake
option(ENABLE_COVERAGE "Build with gcov instrumentation" OFF)
if(ENABLE_COVERAGE)
  add_compile_options(--coverage -O0 -g)
  add_link_options(--coverage)
endif()
```

```bash
cmake -S . -B build-cov -DENABLE_COVERAGE=ON
cmake --build build-cov
ctest --test-dir build-cov
lcov --capture --directory build-cov --output-file cov.info
genhtml cov.info --branch-coverage --output-directory coverage-html
```

Use a **separate build directory**: `.gcda` files accumulate counts across runs,
so a stale directory silently reports coverage from tests you deleted. Delete
the `.gcda` files (`lcov --zerocounters --directory build-cov`) before each
measured run.

## 6. Using the number honestly

Coverage is a **gap-finding tool, not a quality score**. Two properties make it
easy to abuse:

- It is trivially gameable. A test that calls every function and asserts nothing
  reaches 100%. Some teams have discovered this by accident and then defended
  the number.
- It measures execution, not verification. A line can run inside a test that
  never checks its effect.

Practical policy:

1. Track **branch** coverage, not line coverage.
2. Enforce a ratchet — coverage may not drop below its current value — rather
   than an absolute target. Absolute targets get met by writing assertion-free
   tests for trivial getters.
3. Read the annotated source for changed files in review; that is where the
   signal is. The percentage is for trend lines, not for decisions.
4. Treat uncovered **error-handling** paths as the top priority. They are the
   least exercised in production and the most likely to be wrong.
5. Never let coverage substitute for the mutation question: *if I broke this
   line, would a test fail?*

## Exercise

1. **Reproduce section 2 exactly.** Build `grade.c` with `--coverage`, run the
   three-case suite, and confirm you get 100% lines and 70% branches. If your
   numbers differ, check that you compiled with `-O0` and in two steps.

2. **Close the gap.** Add test cases until `Taken at least once` reaches 100%.
   List which behaviours the new cases test that the old suite did not, and
   write two sentences on what that means for "100% line coverage" as a target.

3. **Demonstrate the gameable property.** Write a test that calls `grade_for`
   with all five categories of input and contains **no assertions at all**.
   Measure coverage. Record the number, then explain in three sentences why it
   is indistinguishable from the good suite's number.

4. **Compare condition coverage.** Line 3 is `score < 0 || score > 100`. Design
   the minimum set of inputs that gives 100% branch coverage but leaves one
   sub-condition never independently decisive, and explain what MC/DC would
   additionally require.

5. **Generate an HTML report** with lcov and genhtml for a multi-file project.
   Use `--remove` to exclude system headers and your test directory, and record
   the before/after percentages. Explain which of the two is the honest number.

6. **Build the ratchet.** Add a CMake `ENABLE_COVERAGE` option, then write a
   short shell script that fails with a non-zero exit code if branch coverage
   falls below a value stored in a `coverage-baseline.txt` file. Verify it fails
   when you delete a test, and passes when you restore it.
