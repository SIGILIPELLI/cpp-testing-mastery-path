---
description: "Test Architecture for Large Codebases — Everything so far in this path — unit tests, fixtures, doubles, sanitizers, CI — works cleanly at the scale of one…"
---

# 01 · Test Architecture for Large Codebases

Everything so far in this path — unit tests, fixtures, doubles, sanitizers,
CI — works cleanly at the scale of one module or one repository. At the
scale of a million-line codebase with dozens of teams, the failure mode
shifts: it's no longer "how do I test this function," it's "why does the
test suite take 40 minutes, why did my unrelated change break someone
else's tests, and why does nobody trust the red build anymore." This module
is about the structural decisions that keep a test suite healthy as it
grows by two orders of magnitude.

!!! note "Environment note"
    This module is architectural rather than code-heavy by nature — most
    listings are directory layouts, build-graph fragments, and policy, not
    runnable programs. Where C/C++ snippets appear, they follow the same
    environment constraints noted throughout this path (broken libc++
    headers on this host; plain C examples were compiled and run where
    shown, and are marked as such).

## 1. The core problem: test suite growth is superlinear

A codebase that grows 10x in size tends to grow its test suite by more than
10x, because:

- Integration surface grows combinatorially, not linearly, with the number
  of components.
- Flaky tests accumulate and are rarely deleted, only "quarantined."
- Build/test infrastructure that was fine at N files degrades non-linearly
  (a full rebuild-and-retest strategy is fine at 50 files, catastrophic at
  50,000).

The architectural answer is not "write fewer tests" — it's structuring the
suite so most changes only trigger a small, relevant slice of it.

## 2. Test pyramid, at scale, is a dependency graph problem

The classic pyramid (many unit tests, fewer integration tests, few
end-to-end tests) is really a statement about **dependency graph depth**:
a unit test depends on one module, an end-to-end test depends on the whole
system's dependency closure. The bigger the codebase, the more this
distinction determines what's affordable to run per-commit.

```text
apps/
  checkout-service/
    src/
    tests/            <- unit tests, depend only on this module + its libs
  BUILD (declares: deps = [//libs/pricing, //libs/inventory])

libs/
  pricing/
    src/
    tests/            <- unit tests, depend only on //libs/pricing
  inventory/
    ...

integration_tests/
  checkout_pricing_integration/
    <- depends on BOTH //apps/checkout-service and //libs/pricing,
       explicitly, so a build system can compute "which tests does
       this change affect" from the dependency graph alone.
```

A build system that understands this graph (Bazel, or a hand-rolled
dependency map in CMake/CI) can answer "which tests must re-run for this
diff" precisely — the single highest-leverage investment for keeping CI
fast as the codebase grows.

## 3. Affected-test selection

```bash
# Bazel-style: compute the transitive set of tests depending on changed files
bazel query "rdeps(//..., set($(git diff --name-only main | tr '\n' ' ')))" \
  --output=label | grep "_test$" > affected_tests.txt
bazel test $(cat affected_tests.txt)
```

Without dependency-graph-based selection, the two fallback strategies are
both bad at scale: run everything (CI time grows with codebase size,
unboundedly), or run "tests near the changed files" by convention (misses
real cross-module breakage, the exact failure mode integration tests exist
to catch).

```yaml
# A coarser, CMake/CI-native approximation when a full graph query tool
# isn't available: tag tests with the modules they touch, and only run
# tags matching changed paths.
jobs:
  test-changed:
    steps:
      - run: |
          CHANGED=$(git diff --name-only origin/main...HEAD)
          TAGS=$(echo "$CHANGED" | sed -n 's|^libs/\([a-z_]*\)/.*|\1|p' | sort -u)
          for tag in $TAGS; do
            ctest --test-dir build -L "$tag"
          done
```

## 4. Owning boundaries: who is allowed to break whose tests

At small scale, "run the whole suite before merging" is sufficient social
contract. At large scale, a change in library A breaking a test in
unrelated team B's integration suite, discovered only after merge, is a
recurring source of friction. Two structural fixes:

- **Every integration test has a declared owner (a team, in CODEOWNERS or
  equivalent) who is notified — and ideally blocks the merge — when their
  test starts failing**, not just whoever happened to touch the code last.
- **Public interfaces are versioned or contract-tested** (Level 3, module
  01) so a library team can evolve internals freely while a contract test
  catches the rare case where the *public* behavior actually changed.

```text
# CODEOWNERS
/integration_tests/checkout_pricing_integration/  @checkout-team @pricing-team
/libs/pricing/                                    @pricing-team
```

## 5. Flake management as a first-class process, not an afterthought

At scale, a 0.1% flake rate per test times ten thousand tests means most CI
runs have at least one spurious failure — and the moment "just re-run it"
becomes the default response, the suite has stopped meaning anything.

| Practice | What it buys |
|---|---|
| Automatic flake detection (track pass/fail history per test) | Distinguishes "this test is flaky" from "this change broke something," without a human noticing by feel |
| A quarantine mechanism (skip in the blocking gate, still run and tracked) | Keeps a known-flaky test from blocking unrelated merges while it's fixed, without silently deleting coverage |
| A hard SLA on quarantine ("fixed or deleted within N days") | Prevents quarantine from becoming a permanent graveyard of ignored red tests |
| Retries **only** at the infra level (network blips), never as a policy for logic flakiness | A retry that papers over a genuine race (Level 3, module 08) hides the bug instead of fixing it |

```python
# A minimal flake tracker: record every test result, flag tests whose
# pass rate over the last N runs is inconsistent (neither ~100% nor ~0%).
def is_flaky(history: list[bool], window: int = 50) -> bool:
    recent = history[-window:]
    if len(recent) < window:
        return False
    pass_rate = sum(recent) / len(recent)
    return 0.02 < pass_rate < 0.98   # neither reliably green nor reliably red
```

## 6. Build/test infrastructure that scales

| At small scale | Breaks down at large scale | Replacement |
|---|---|---|
| Full rebuild every CI run | Minutes become hours | Incremental builds with correct dependency tracking (ccache, Bazel remote cache) |
| One monolithic test binary | Minutes to link, one crash takes down all results | Sharded test binaries, parallel execution, per-shard reporting |
| Serial `ctest` run | Doesn't use available CI parallelism | `ctest -j$(nproc)`, or matrix-parallel CI jobs (Level 3, module 06) |
| Local developer machine as the only fast feedback loop | Inconsistent hardware, "works on my machine" | A remote build cache/execution service shared across the team |

```cmake
# Sharding a CTest suite roughly by directory, runnable in parallel CI jobs.
add_test(NAME shard_a COMMAND test_runner --gtest_filter=Pricing*:Inventory*)
add_test(NAME shard_b COMMAND test_runner --gtest_filter=Checkout*:Shipping*)
```

## 7. Test data ownership and versioning

Large codebases accumulate shared fixture data (a schema snapshot, a sample
config, a golden dataset) that many tests depend on. Treat it like any
other dependency:

- Version it alongside the code it describes, not as a separately-updated
  "test fixtures" repo that drifts.
- Give it an owner and a changelog — a fixture update that silently changes
  fifty tests' expected values is a hidden mass edit, not a small change.

## Cheat sheet

| Growing pain | Structural fix |
|---|---|
| CI takes too long | Dependency-graph-based affected-test selection |
| Unrelated teams break each other's tests | Declared test ownership + contract tests at public boundaries |
| Nobody trusts red builds anymore | A formal flake-detection and quarantine process with an SLA |
| One test binary is a bottleneck | Sharding + parallel execution |
| Shared fixtures drift silently | Version fixtures with the code, assign an owner |

## How It Actually Works: affected-test selection as a reachability query

"Dependency-graph-based affected-test selection" is a concrete graph
algorithm, not a vague architectural principle — build systems that
implement it (Bazel's `bazel test //...` with query, or a custom mapping
layer over CMake) all reduce to the same computation.

- **The build graph already encodes the dependency data — it's reused, not
  rebuilt.** A build system that tracks header/library dependencies for
  incremental compilation (so it knows "recompile `foo.cpp` if `bar.h`
  changed") has, as a side effect, a complete directed graph from source
  files to the object files, libraries, and ultimately test binaries that
  transitively depend on them. Affected-test selection is a **reverse
  reachability query** on that same graph: given the set of changed files in
  a commit, find every test-binary node reachable by following dependency
  edges backward from those files.
- **This is why the graph must be accurate at file granularity, not just
  target granularity, to be worth anything.** If the dependency data only
  says "this test target depends on this library target" without tracking
  which specific headers/sources within the library the test binary actually
  transitively includes, then any change to any file in a large shared
  library marks every test that links it as "affected" — collapsing the
  optimization back to "run everything," which is the actual failure mode
  behind vague dependency tracking in a large codebase.
- **Sharding is a partition of the *same* registry from Level 1 Module 7, not
  a different execution model.** `--gtest_shard_index=k --gtest_total_shards=N`
  tells one GoogleTest process to only construct and run the subset of its
  registered `TestInfo` entries whose position modulo `N` equals `k` — the
  registry, construction, and per-test lifecycle are completely unchanged;
  sharding just distributes iteration over that one array across N separate
  OS processes (often on N separate machines) instead of one process running
  the whole array serially.

## 🔀 Related lessons on other tracks

- [Java Testing — 01 · Test Architecture at Scale](https://sigilipelli.github.io/java-testing-mastery-path/level-4/01-test-architecture-at-scale/)
- [Python Testing — 01 · Test Architecture at Scale](https://sigilipelli.github.io/python-testing-mastery-path/level-4/01-test-architecture/)

## Exercise

1. Sketch a dependency graph (as a diagram or a nested list) for a
   three-service system of your choosing, and identify which tests would
   need to re-run for a change to just one shared library versus a change
   to one service's internals only.
2. Design an affected-test-selection script (bash or Python, pseudocode is
   fine) for a CMake-based project that doesn't have a full build-graph
   query tool, using file-path-to-tag mapping as in section 3.
3. Write a CODEOWNERS-style ownership map for a hypothetical five-team,
   fifteen-module codebase, and identify the two or three integration
   tests you'd expect to have the most contentious ownership questions.
4. Implement the `is_flaky` function from section 5 against a small
   synthetic pass/fail history, and design the quarantine SLA policy
   (in words) you'd propose for your own project.
5. Write two sentences on which specific practice in this module your
   current project (or one you've worked on) most needs, and what the
   concrete first step toward adopting it would be.
