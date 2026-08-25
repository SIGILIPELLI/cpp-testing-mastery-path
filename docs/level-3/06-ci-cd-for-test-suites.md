# 06 · CI/CD for C/C++ Test Suites

Every technique from Levels 1-3 — unit tests, sanitizers, coverage, static
analysis, fuzzing, integration tests — only compounds in value once it runs
automatically on every change, without anyone remembering to type the
command. This module wires all of it into GitHub Actions.

!!! note "Environment note"
    No GitHub Actions runner exists on this machine, so the workflow YAML
    below was not executed here — it's checked for structural correctness
    (valid YAML, correct action versions, matching the CMake targets
    actually built earlier in this path) rather than run. The `cmake`/`ctest`
    commands *inside* the workflow follow the same commands already used and
    verified (where the toolchain allowed) in Level 2 and this level's
    earlier modules.

## 1. The minimum viable workflow

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install dependencies
        run: sudo apt-get update && sudo apt-get install -y cmake ninja-build

      - name: Configure
        run: cmake -S . -B build -G Ninja -DCMAKE_BUILD_TYPE=Debug

      - name: Build
        run: cmake --build build

      - name: Test
        run: ctest --test-dir build --output-on-failure
```

This alone is worth more than most projects' first ten commits of CI
tuning: it fails a PR the moment a test breaks, on a clean machine, instead
of relying on "it worked on my laptop."

## 2. Layering in the Level 2/3 instrumentation as separate jobs

Running every check in one job means one flaky sanitizer run blocks the
whole pipeline and a coverage report competes with a fuzzer for wall-clock
time. Split by concern and run them in parallel.

```yaml
jobs:
  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: cmake -S . -B build -DCMAKE_BUILD_TYPE=Debug
      - run: cmake --build build
      - run: ctest --test-dir build --output-on-failure

  sanitized-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: cmake -S . -B build-san -DENABLE_SANITIZERS=ON
      - run: cmake --build build-san
      - run: ctest --test-dir build-san --output-on-failure

  coverage:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: sudo apt-get install -y lcov
      - run: cmake -S . -B build-cov -DENABLE_COVERAGE=ON
      - run: cmake --build build-cov
      - run: ctest --test-dir build-cov --output-on-failure
      - run: |
          lcov --capture --directory build-cov --output-file coverage.info \
               --exclude '*/tests/*' --exclude '*/_deps/*'
          lcov --summary coverage.info
      - uses: actions/upload-artifact@v4
        with:
          name: coverage-report
          path: coverage.info

  static-analysis:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: sudo apt-get install -y cppcheck clang-tidy
      - run: cmake -S . -B build -DCMAKE_EXPORT_COMPILE_COMMANDS=ON
      - run: cppcheck --enable=warning,style --error-exitcode=1 -I include src
      - run: clang-tidy -p build src/*.cpp

  fuzz-smoke:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: |
          clang -std=c17 -Iinclude -g -fsanitize=fuzzer,address,undefined \
                src/protocol.c fuzz/fuzz_frame_parse.c -o fuzz_frame_parse
      - name: Run fuzzer for a fixed budget
        run: ./fuzz_frame_parse -max_total_time=120 fuzz/corpus/
      - name: Regression corpus (must never crash)
        run: |
          for f in fuzz/regressions/*; do ./fuzz_frame_parse "$f"; done
      - uses: actions/upload-artifact@v4
        if: failure()
        with:
          name: fuzz-crash
          path: crash-*
```

This is the same set of gates from Level 2's capstone `run_gate.sh`, plus
this level's fuzz smoke test, mapped one-to-one onto CI jobs — the script
was the local rehearsal, this is the enforced version everyone's PR goes
through.

## 3. Caching to keep the pipeline fast

GoogleTest is fetched via `FetchContent` in earlier modules — rebuilding it
from source on every CI run wastes minutes per job. Cache the CMake build
directory's dependency subtree, keyed on the lockable inputs.

```yaml
      - uses: actions/cache@v4
        with:
          path: build/_deps
          key: ${{ runner.os }}-deps-${{ hashFiles('**/CMakeLists.txt') }}
          restore-keys: |
            ${{ runner.os }}-deps-
```

The cache key hashes `CMakeLists.txt` files specifically — bump the
googletest version pin in that file and the key changes, so a stale cached
dependency never silently lingers past the version that actually changed it.

## 4. Required checks and branch protection

A workflow that runs but doesn't block merges is a suggestion, not a gate.
In the repository's branch protection settings for `main`, mark
`unit-tests`, `sanitized-tests`, and `static-analysis` as **required status
checks**. Two calls worth making deliberately:

- **Coverage and fuzzing usually stay non-blocking at first.** Coverage
  percentage is a trend to watch, not a hard gate, until the team has
  agreed on a target; fuzz-smoke failing on a genuinely new crash should
  block, but a flaky fuzz timeout shouldn't hold every PR hostage while
  that's being tuned.
- **Sanitized tests should block from day one.** A sanitizer catching a
  real memory bug and not blocking merge defeats the entire point of
  running it in CI at all.

## 5. Matrix builds across compilers and standards

The C++ standard library difference that broke this very environment
(module 01's note) is exactly the kind of thing a matrix build catches
before it reaches a contributor's laptop:

```yaml
  unit-tests:
    strategy:
      matrix:
        compiler: [gcc, clang]
        std: [17, 20]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: |
          cmake -S . -B build \
            -DCMAKE_CXX_COMPILER=${{ matrix.compiler == 'gcc' && 'g++' || 'clang++' }} \
            -DCMAKE_CXX_STANDARD=${{ matrix.std }}
      - run: cmake --build build
      - run: ctest --test-dir build --output-on-failure
```

Four jobs from one definition — catches "works on clang, breaks on gcc" and
"works on C++20, breaks on C++17" without four separate workflow files.

## 6. Reporting back to the PR

Raw CI logs are the fallback; the pieces that most reduce back-and-forth are
inline annotations and a coverage delta comment.

```yaml
      - name: Annotate test failures
        if: failure()
        uses: mikepenz/action-junit-report@v4
        with:
          report_paths: 'build/**/*.xml'
```

```yaml
      - name: Coverage summary comment
        uses: romeovs/lcov-reporter-action@v0.4.0
        with:
          lcov-file: coverage.info
          github-token: ${{ secrets.GITHUB_TOKEN }}
```

A JUnit-format test report (`ctest` supports `--output-junit`) is the
integration point most CI reporting tools expect — generate it even if
nothing consumes it yet, because retrofitting it later means re-touching
every job.

## Cheat sheet

| Concern | CI job | Blocking? |
|---|---|---|
| Correctness regressions | `unit-tests` | Yes, from day one |
| Memory/UB bugs | `sanitized-tests` | Yes, from day one |
| Style/pattern issues | `static-analysis` | Usually yes, tune noise first |
| Coverage trend | `coverage` | Often non-blocking initially |
| New crashing inputs | `fuzz-smoke` | Yes, once corpus/timeouts are stable |
| Cross-compiler/standard drift | Matrix build | Yes |

## Exercise

1. Write a `ci.yml` for the Level 2 capstone (`instrumented/`) with separate
   `unit-tests`, `sanitized-tests`, and `coverage` jobs, each configuring
   its own build directory as shown in section 2.
2. Add the caching block from section 3, and explain in one sentence why
   the cache key should include the CMakeLists.txt hash rather than being a
   fixed string.
3. Turn the `run_gate.sh` static-analysis step from Level 2's capstone into
   its own `static-analysis` job here, and decide (with a one-sentence
   justification) whether it should be a required check immediately or
   only after a noise-reduction pass.
4. Add a two-entry compiler matrix (gcc/clang) to the `unit-tests` job and
   note, from this path's own experience with a broken libc++ install, what
   specific class of bug a compiler matrix is best positioned to catch that
   a single-compiler CI would miss.
5. Write two sentences on which of your project's current checks (real or
   hypothetical) are blocking merges today versus which are still only
   "run and reported" — and whether that split still makes sense.
