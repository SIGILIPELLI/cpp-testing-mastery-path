---
title: "C/C++ Testing Cheat Sheet: Quick Reference, Beginner to Advanced"
description: "A free C/C++ Testing cheat sheet: 19 quick-reference tables from every level of the C/C++ Testing Mastery Path, each linked to its full lesson."
---

# C/C++ Testing Cheat Sheet

Quick-reference tables from **19 lessons** of the free C/C++ Testing Mastery Path, in course order from beginner to advanced. Each section links to the full lesson with explanations and runnable examples.

## Level 2 · Intermediate

### [10 · Project — Instrumented Test Suite](level-2/10-project-instrumented-suite.md)

| Bug class | Caught by |
|-----------|-----------|
| Logic error with a wrong-but-plausible result | Unit tests (GoogleTest) |
| Out-of-bounds read/write, use-after-free | AddressSanitizer |
| Signed overflow, misaligned access, null deref | UndefinedBehaviorSanitizer |
| Code path never exercised | Coverage (gcov/lcov) |
| Suspicious pattern even without a failing test | clang-tidy / cppcheck |
| Regression after a refactor | All of the above, run in CI on every push |

## Level 3 · Advanced

### [01 · Integration Testing Strategies](level-3/01-integration-testing-strategies.md)

| Question | Answer |
|---|---|
| What's real at this boundary? | Only the components you're actually verifying interact correctly |
| What's still faked? | Everything outside your control: network, third-party APIs, wall clock |
| How do I catch fake drift? | Contract tests run against both the fake and the real implementation |
| How do I keep it fast? | Shared suite-level fixtures, polling instead of sleeping, parallel partitions |
| How do I catch state leaks? | Run with `--gtest_shuffle` in CI |

### [02 · Embedded Target Testing](level-3/02-embedded-target-testing.md)

| Code characteristic | Where it's tested |
|---|---|
| Pure logic, parsing, math, state machines | Host, every commit |
| Thin register-access wrapper | Manual review + minimal on-target smoke test |
| Timing-dependent (DMA, interrupt latency) | On-target or HIL only |
| Wire format / protocol framing | Host, with explicit byte-level (not struct-layout) code |
| Anything using UB-sensitive constructs | Host build compiled with `-fsanitize=undefined` |

### [03 · Hardware-in-the-Loop Basics](level-3/03-hardware-in-the-loop-basics.md)

| Question | Answer |
|---|---|
| What does HIL add over host tests? | Catches failures that only exist in real electrical/timing behavior |
| What should still be a host test? | Anything the fake HAL can express faithfully — most logic |
| How do I avoid HIL flakiness? | Poll with a timeout and a stability check, never a bare `sleep` |
| How often should HIL run? | Smoke suite per merge, full suite nightly/pre-release |
| What must a HIL fixture guarantee? | A truly clean starting state — power cycle, not just reset, if persistence exists |

### [04 · Fuzz Testing](level-3/04-fuzz-testing.md)

| Question | Answer |
|---|---|
| What functions should have fuzz targets? | Anything parsing untrusted/external bytes |
| What sanitizers should be compiled in? | ASan + UBSan, always, alongside `-fsanitize=fuzzer` |
| What do I do with a crash? | Save it to a regression corpus, replay it, fix, re-replay |
| How do I speed up discovery? | Seed corpus with real valid/edge-case inputs, use a dictionary |
| How do I fuzz structured formats? | Interpret raw bytes as your grammar inside the target |
| How do I know it's actually working? | Watch `cov:`/`ft:` climb, not just "no crash yet" |

### [05 · Property-Based Testing](level-3/05-property-based-testing.md)

| Property shape | Example |
|---|---|
| Round-trip | `decode(encode(x)) == x` |
| Invariant preserved | Sorting preserves the multiset of elements |
| Postcondition | Sorted output is actually ordered |
| Oracle comparison | Fast implementation matches a trusted slow one |
| Metamorphic | A known relationship holds between two related calls |

### [06 · CI/CD for C/C++ Test Suites](level-3/06-ci-cd-for-test-suites.md)

| Concern | CI job | Blocking? |
|---|---|---|
| Correctness regressions | `unit-tests` | Yes, from day one |
| Memory/UB bugs | `sanitized-tests` | Yes, from day one |
| Style/pattern issues | `static-analysis` | Usually yes, tune noise first |
| Coverage trend | `coverage` | Often non-blocking initially |
| New crashing inputs | `fuzz-smoke` | Yes, once corpus/timeouts are stable |
| Cross-compiler/standard drift | Matrix build | Yes |

### [07 · Performance &amp; Benchmark Testing](level-3/07-performance-benchmark-testing.md)

| Question | Tool/technique |
|---|---|
| Is this fast enough in isolation? | A single `BENCHMARK` with a realistic input size |
| Is A faster than B? | Two benchmarks over the same input, same fixture |
| Did the last commit regress performance? | JSON output + `compare.py` in CI against a stored baseline |
| Is the measurement trustworthy? | `--benchmark_repetitions`, aggregate reporting, watch for throttling |
| Does the growth rate match the algorithm's claimed complexity? | `->Complexity()` with multiple range points |

### [08 · Concurrency Testing](level-3/08-concurrency-testing.md)

| Symptom | Likely cause | Tool |
|---|---|---|
| Wrong result, no crash, varies run to run | Unsynchronized read-modify-write | TSan |
| Rare, unreproducible hang | Lock-order inversion | TSan's deadlock detection |
| "Volatile should have fixed this" | Volatile isn't a memory-model tool | Use `_Atomic`/`std::atomic` instead |
| Passes locally, fails under load | Race only exercised at higher concurrency | Stress-run the same TSan binary many times, at higher thread counts |
| Slow to detect | TSan needs the racy access to actually execute | Combine with fuzzing/property tests generating varied thread interleavings via delays |

### [09 · Legacy Code: Getting Untestable Code Under Test](level-3/09-legacy-code-under-test.md)

| Situation | Move |
|---|---|
| No tests exist yet | Characterization tests first, bugs and all |
| Hard-wired C dependency | Link seam (Level 2, module 07) |
| Hard-wired call inside a C++ method | Extract-and-override (protected virtual) |
| Found a bug while characterizing | Record it as-is; fix it in a separate, tested change |
| Extraction changed a characterization result | The extraction has a bug — stop and investigate |

## Level 4 · Master

### [01 · Test Architecture for Large Codebases](level-4/01-test-architecture-large-codebases.md)

| Growing pain | Structural fix |
|---|---|
| CI takes too long | Dependency-graph-based affected-test selection |
| Unrelated teams break each other's tests | Declared test ownership + contract tests at public boundaries |
| Nobody trusts red builds anymore | A formal flake-detection and quarantine process with an SLA |
| One test binary is a bottleneck | Sharding + parallel execution |
| Shared fixtures drift silently | Version fixtures with the code, assign an owner |

### [02 · Safety-Critical Testing Standards](level-4/02-safety-critical-standards.md)

| Question | Where the answer lives |
|---|---|
| Is this C construct even allowed? | MISRA C/C++ rule set, checked by a MISRA-aware static analyzer |
| How much structural coverage is required? | DO-178C DAL / ISO 26262 ASIL table (section 3) |
| Does every requirement have a test? | The requirements traceability matrix (module 03) |
| Can I trust this analysis tool's output as evidence? | Only if it (or its use) is qualified (section 5) |
| What's actually different at a higher assurance level? | A specific, enumerated evidence bar — not "more effort" in the abstract |

### [03 · Requirements-Based Testing &amp; Traceability](level-4/03-requirements-based-testing.md)

| Question | Practice |
|---|---|
| Does every requirement have a test? | Generate a traceability matrix, don't maintain it by hand |
| How do I keep the link from rotting? | Encode requirement IDs in test names/annotations at the source |
| What does an orphan requirement mean? | A genuine coverage gap, invisible from the test suite alone |
| What does an orphan test mean? | Possibly testing a stale or undocumented requirement — worth review |
| Does this replace other testing techniques? | No — it's one more source of test cases, layered on the same infrastructure |

### [04 · Formal Methods &amp; Static Verification Basics](level-4/04-formal-methods-static-verification.md)

| Question | Tool/technique |
|---|---|
| Does this small C function ever overrun a buffer, for any input? | Bounded model checking (CBMC) |
| Can I state pre/postconditions machine-checkably, for both tests and proofs? | Design-by-contract (ACSL, or a lighter in-code assert-based convention) |
| Is there a bad interleaving in this concurrent algorithm, across all schedules? | A model checker (TLA+, or a concurrency-specific tool) |
| Is this the right component to formally verify at all? | Small, stable, high-consequence, precisely-statable property (section 4) |
| Does a passing proof mean the code is "correct"? | Only correct relative to the stated specification — see traps |

### [05 · Certification Evidence &amp; Test Reporting](level-4/05-certification-evidence-reporting.md)

| Evidence need | Artifact | Generated from |
|---|---|---|
| Non-programmer-readable test intent | Test procedures | Written once per test, tied to a requirement ID |
| Proof a specific build was tested | Test results record | Generated from CI's structured test output, never hand-transcribed |
| Proof required coverage was hit | Structural coverage report | `lcov`/DO-178C-qualified coverage tool output |
| Proof every failure was resolved | Problem reports, each with re-verification evidence | Opened per FAIL, closed only on a new PASS record |
| Everything ties to one software version | Configuration management (commit hash in every artifact) | Version control, referenced everywhere |

### [06 · Cross-Compilation Test Strategies](level-4/06-cross-compilation-test-strategies.md)

| Risk | Host-only mitigation | Full mitigation |
|---|---|---|
| Integer width assumptions | `<stdint.h>` types everywhere; `-Wconversion` | Cross-compile + run under QEMU/HIL |
| Endianness | Byte-explicit code (never rely on struct layout); host tests with known-endianness input | Same code, verified again under QEMU/HIL for defense in depth |
| Struct padding differences | `-Wpadded`, explicit byte-level (de)serialization | Cross-compiled binary inspection (`nm`, `objdump`) |
| Alignment faults | `-Wcast-align` | QEMU/HIL — genuinely needs the target's actual fault behavior |
| Minimal-libc gaps | Review target libc docs for missing functions | Attempt an actual cross-compile; link errors surface gaps immediately |

### [07 · Simulation &amp; Emulation (QEMU-based Test Rigs)](level-4/07-simulation-emulation-qemu.md)

| Question | Answer |
|---|---|
| Bare-metal firmware, need pass/fail from CI? | `qemu-system-*` + semihosting exit codes |
| Board has a UART model? | Route it via `-nographic`, capture stdout in CI logs like a normal test |
| Vendor peripheral has no QEMU model? | Falls back to HIL (Level 3, module 03) — no shortcut |
| Target runs embedded Linux, not bare metal? | `qemu-user` (instruction-level, no board model) instead |
| CI job hangs on a firmware bug? | Wrap the QEMU invocation in `timeout` — no OS supervision otherwise |

### [08 · Quality Gates &amp; Metrics](level-4/08-quality-gates-metrics.md)

| Question | Guidance |
|---|---|
| Coverage floor or delta? | Prefer delta-on-changed-lines once tooling supports it; floor is a reasonable starting gate |
| What should be zero-tolerance? | New static analysis findings on changed code, sanitizer failures |
| What should be a warning, not a block? | Complexity ceilings, coverage on pre-existing legacy files |
| How many blocking checks? | Roll them into one `merge-gate` job dependent on the real checks, for a single clear signal |
| How do I introduce a new gate without backlash? | Set the threshold near current reality, explain the "why," ratchet over time |

### [09 · Building an In-House C/C++ Test Framework](level-4/09-in-house-test-framework.md)

| Feature | This module's framework | GoogleTest |
|---|---|---|
| Auto-registration | Yes (constructor attribute) | Yes (more elaborate internally) |
| Assertion continues test on failure | Yes | Yes (`EXPECT_*`; `ASSERT_*` aborts the test) |
| Fixtures | Basic (section 5) | Full (`SetUp`/`TearDown`, fixture inheritance) |
| Parameterized tests | Not built here | Yes |
| Mocking | Not built here | Via GoogleMock |
| Death tests, typed tests | Not built here | Yes |
| Footprint | Two small `.c` files | Full library, larger binary |

---

Go deeper with the full [C/C++ Testing Mastery Path](index.md) curriculum or test yourself with the [interview questions](interview-questions.md).
