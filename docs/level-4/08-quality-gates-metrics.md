# 08 · Quality Gates & Metrics

Every technique in this path produces a number: coverage percentage,
static analysis finding count, flake rate, mean time to fix a failing
build. A **quality gate** turns a chosen subset of those numbers into an
automated pass/fail decision on a merge — the mechanism that makes
"we require 80% coverage" an enforced fact rather than a wiki page nobody
reads. This module is about choosing metrics that drive the right behavior
and building the gates that enforce them.

!!! note "Environment note"
    The Python threshold-checking script in section 3 **was written and run
    on this host** against synthetic coverage data, and its output is real.
    The rest of the module is process/design guidance and GitHub Actions
    YAML, checked structurally rather than executed (no CI runner on this
    host, consistent with earlier modules).

## 1. Metrics worth gating on, and ones that backfire

| Metric | Good gate? | Why |
|---|---|---|
| Line/statement coverage, with a floor (not a rising ratchet on every file) | Yes, carefully | Prevents obvious regressions; a rigid 100% target on every file incentivizes low-value tests written purely to hit the number |
| Coverage *delta* on changed lines only | Yes, often better than a global floor | Directly targets "did this PR add untested code," without punishing a large legacy file's pre-existing gap |
| Static analysis finding count, at zero for new code | Yes | New code should not introduce new findings; a project-wide zero target on day one of adopting a linter is usually unrealistic |
| Cyclomatic complexity per function, with a ceiling | Yes, as a warning more than a hard block | Flags functions worth a second look; blocking merges on it outright can incentivize artificial function splitting that doesn't actually reduce complexity |
| Total test count | No | Trivially gameable (split one test into ten), and says nothing about what's actually verified |
| "100% of tests pass" | This is table stakes, not a quality metric | Necessary but doesn't measure quality — a suite with no assertions "passes" 100% |

The unifying principle: gate on metrics that are hard to game without doing
the actual work the metric is meant to proxy for, and prefer deltas on
changed code over absolute floors when a codebase has legacy gaps that
can't be closed in one PR.

## 2. Coverage gates: floor vs. delta, concretely

```yaml
# Floor: blocks if TOTAL coverage drops below a fixed percentage.
# Simple, but can block an unrelated PR because someone else's file
# has always been under-tested.
- name: Coverage floor check
  run: |
    COVERAGE=$(lcov --summary coverage.info | grep lines | grep -oE '[0-9]+\.[0-9]+')
    python3 -c "import sys; sys.exit(0 if float('$COVERAGE') >= 75.0 else 1)"
```

```yaml
# Delta: blocks if the LINES THIS PR TOUCHED are under-covered, regardless
# of the file's historical baseline. Better-targeted, more setup.
- name: Coverage on changed lines
  run: |
    diff-cover coverage.xml --compare-branch=origin/main --fail-under=90
```

`diff-cover` (or an equivalent tool) computes coverage specifically for the
lines a diff touches — the gate that actually answers "did this change add
untested code," which a global floor only answers as a noisy proxy.

## 3. A real, runnable threshold-gate script

```python
#!/usr/bin/env python3
# quality_gate.py -- combines multiple metrics into one pass/fail decision,
# the way a real CI quality-gate step would.
import sys

def check_gate(metrics: dict, thresholds: dict) -> list[str]:
    """Returns a list of failure messages; empty list means the gate passed."""
    failures = []
    if metrics["coverage_pct"] < thresholds["min_coverage_pct"]:
        failures.append(
            f"Coverage {metrics['coverage_pct']}% is below the "
            f"required {thresholds['min_coverage_pct']}%")
    if metrics["new_static_findings"] > thresholds["max_new_findings"]:
        failures.append(
            f"{metrics['new_static_findings']} new static analysis findings "
            f"exceed the limit of {thresholds['max_new_findings']}")
    if metrics["flaky_test_count"] > thresholds["max_flaky_tests"]:
        failures.append(
            f"{metrics['flaky_test_count']} flaky tests exceed the limit "
            f"of {thresholds['max_flaky_tests']}")
    return failures

def main():
    # Synthetic metrics standing in for real ctest/lcov/cppcheck/flake-
    # tracker output, to demonstrate the gate logic itself runs.
    metrics = {
        "coverage_pct": 82.5,
        "new_static_findings": 2,
        "flaky_test_count": 1,
    }
    thresholds = {
        "min_coverage_pct": 80.0,
        "max_new_findings": 0,
        "max_flaky_tests": 3,
    }

    failures = check_gate(metrics, thresholds)
    if failures:
        print("QUALITY GATE: FAILED")
        for f in failures:
            print(f"  - {f}")
        sys.exit(1)
    print("QUALITY GATE: PASSED")

if __name__ == "__main__":
    main()
```

```bash
python3 quality_gate.py
```

```text
QUALITY GATE: FAILED
  - 2 new static analysis findings exceed the limit of 0
```

This script and its output are real — run against the synthetic metrics
dict above, the coverage and flaky-test checks pass (82.5% ≥ 80%, 1 ≤ 3)
while the static-analysis check correctly fails (2 new findings > the
zero-tolerance threshold), demonstrating a gate that fails on exactly one
of several checked dimensions rather than an all-or-nothing signal.

## 4. Trend metrics, not just point-in-time gates

A single PR's numbers are a snapshot; the trend across many PRs is what
reveals whether a codebase's quality is improving, stable, or eroding
underneath gates that only check "not worse than the floor."

```text
Week    Coverage%   Static findings (open)   Median CI time   Flaky tests
2026-07  78.2            14                      6m 40s            5
2026-07  79.1            11                      6m 55s            4
2026-08  80.5             9                      7m 10s            4
2026-08  81.3             7                      9m 20s            6
```

The median CI time creeping up while coverage improves is a real trade-off
worth surfacing explicitly, not silently accepting — this is exactly the
tension Level 4 module 01's affected-test-selection and sharding techniques
exist to manage before it becomes a team-wide productivity problem.

## 5. Gate placement: where in the pipeline

```yaml
jobs:
  unit-tests:        # from Level 3, module 06
    # ...
  coverage:
    needs: unit-tests
    # ...
    steps:
      - run: lcov --summary coverage.info
      - run: python3 quality_gate.py --coverage-file coverage.info
  static-analysis:    # from Level 2, module 09 / Level 3, module 06
    # ...
  merge-gate:
    needs: [unit-tests, coverage, static-analysis]
    runs-on: ubuntu-latest
    steps:
      - run: echo "All required checks passed"
```

A single `merge-gate` job that depends on every individual check, marked as
the one required status check in branch protection, is a common pattern —
it gives one clear pass/fail signal on the PR regardless of how many
underlying jobs actually ran, and makes it easy to add a new check later
without re-touching branch protection settings each time.

## 6. Traps: gates that damage the culture they're meant to protect

- **A gate too strict for the codebase's current state gets bypassed
  informally** (admin merge, disabled check "just this once") — and an
  informally-bypassable gate is worse than no gate, because it creates the
  appearance of enforcement without the substance. Set initial thresholds
  at or slightly above current reality, then ratchet up deliberately.
- **Gating on a metric nobody understands the "why" of** breeds resentment
  and gaming. A coverage gate introduced without explaining *which* bugs
  it's meant to catch invites exactly the "write a test with no
  assertions to hit the number" behavior the gate was meant to prevent.
- **Too many blocking gates slow every PR down**, including trivial ones. A
  one-line comment fix shouldn't wait on a 20-minute fuzz-smoke run — scope
  gates to what's actually relevant to the change (module 01's affected-
  test-selection principle, applied to gates specifically).

## Cheat sheet

| Question | Guidance |
|---|---|
| Coverage floor or delta? | Prefer delta-on-changed-lines once tooling supports it; floor is a reasonable starting gate |
| What should be zero-tolerance? | New static analysis findings on changed code, sanitizer failures |
| What should be a warning, not a block? | Complexity ceilings, coverage on pre-existing legacy files |
| How many blocking checks? | Roll them into one `merge-gate` job dependent on the real checks, for a single clear signal |
| How do I introduce a new gate without backlash? | Set the threshold near current reality, explain the "why," ratchet over time |

## Exercise

1. Adapt the `quality_gate.py` script to read real coverage output (from
   `lcov --summary`) and real `cppcheck`/`clang-tidy` finding counts for
   one of your own projects, and run it.
2. Design a delta-coverage gate (section 2) for your own project: what
   tool would compute it, and what threshold would you set for newly
   touched lines versus the project's overall baseline?
3. Build a `merge-gate` job (section 5) that depends on your project's
   real CI jobs, and identify which existing jobs should feed it versus
   which should remain informational only.
4. Using the trend table format in section 4, sketch four weeks of
   plausible metrics for your own project (real numbers if you have them,
   reasonable estimates otherwise), and write two sentences on what trend,
   if it continued, would concern you most.
5. Write two sentences on a gate you've seen (in this project or another)
   that was too strict, too lax, or measuring the wrong thing — and what
   you'd change about it using this module's framework.
