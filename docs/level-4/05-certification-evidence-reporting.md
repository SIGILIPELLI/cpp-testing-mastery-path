# 05 · Certification Evidence & Test Reporting

Passing tests are necessary but not sufficient for certification under
DO-178C, ISO 26262, or similar regimes (module 02). An auditor doesn't
re-run your suite and trust the green checkmark — they review a documented,
traceable **evidence package** showing what was tested, how, against what
requirements, with what coverage, and under what configuration. This
module is about assembling that package from the artifacts earlier modules
already produce.

!!! note "Environment note"
    Process and documentation module; the report-generation script in
    section 4 **was written in Python and run on this host** against
    synthetic data, and its output is real. No certification tool or audit
    process was actually exercised — this describes the shape of the
    artifacts such a process consumes.

## 1. What "evidence" means, concretely

An auditor's question is never "did the tests pass" alone — it's "can I
independently verify, from the paper trail alone, that these tests
adequately verify these requirements, in this exact software
configuration." The evidence package answers that without re-running
anything.

| Evidence artifact | Answers |
|---|---|
| Requirements traceability matrix (module 03) | Does every requirement have a test, and vice versa? |
| Test procedures (not just test code — described in reviewable prose) | What does each test actually do, in human-readable form? |
| Test results log, tied to a specific build/config | Which exact software version was tested, and did it pass? |
| Structural coverage report (module 02, section 3) | Was the required MC/DC / decision / statement coverage achieved? |
| Tool qualification records (module 02, section 5) | Can this evidence be trusted given the tools that generated it? |
| Problem reports and their resolution/retest evidence | Was every found defect fixed and re-verified? |

## 2. Test procedures: readable independent of the code

A certification review does not assume the reviewer reads C++. Test
procedures translate a test's intent into prose a domain reviewer (who may
be a systems engineer, not a programmer) can independently assess against
the requirement.

```text
TEST PROCEDURE: TP-PARSE-003
Verifies: REQ-PARSE-003
Objective: Confirm the parser rejects a message whose declared payload
           length would exceed the available buffer, without reading past
           the buffer's end.

Preconditions: None.

Steps:
  1. Construct a 3-byte input buffer: type=0x01, length field=0xFFFF.
  2. Invoke parse_message with this buffer and its exact length (3 bytes).
  3. Observe the return value.

Expected Result: The function returns a non-zero (error) status. The
                 output message_t structure's contents are unspecified
                 (the requirement does not constrain them on failure).

Pass/Fail Criterion: Return value is non-zero.
```

The corresponding executable test (from Level 3's capstone,
`test_rejects_overrunning_length`) is the *implementation* of this
procedure — both exist, and the traceability matrix (module 03) links
them, but the procedure is what a non-programmer reviewer actually reads.

## 3. Test results must be tied to an exact configuration

"The tests passed" is meaningless as certification evidence without stating
*which exact build* — compiler version, flags, source revision — was
tested. This is the software equivalent of chain-of-custody.

```text
TEST RESULTS RECORD

Software Under Test: msgparser
Configuration:
    Git commit:        c9777b0a1f2e...
    Compiler:           gcc 13.2.0
    Compiler flags:     -O2 -std=c11 -Wall -Wextra
    Target:             x86_64-linux-gnu
    Test framework:     Custom C harness, invoked via CTest 3.28.1

Test Run Date: 2026-08-25
Test Run By: CI pipeline, workflow run #4821

Results:
    TP-PARSE-001: PASS
    TP-PARSE-002: PASS
    TP-PARSE-003: PASS
    TP-PARSE-004: FAIL -- see Problem Report PR-0042

Structural Coverage Achieved:
    Statement coverage:  100% (47/47 statements)
    Decision coverage:   100% (12/12 decisions)
    MC/DC:                92% (11/12 -- see PR-0043 for the gap)
```

A results record with a `FAIL` in it is not itself disqualifying — a
certification package routinely includes failed runs, as long as each is
traced to a Problem Report with a documented resolution and a subsequent
*passing* re-test tied to the fixed configuration.

## 4. Generating results records mechanically, not by hand transcription

Hand-transcribing CTest/CI output into a results record is exactly the kind
of step that introduces transcription errors and gets skipped under
deadline pressure. Generate it from the same machine-readable output the
CI pipeline (Level 3, module 06) already produces.

```python
#!/usr/bin/env python3
# generate_results_record.py
import json
import subprocess
from datetime import datetime, timezone

def generate_report(ctest_json_path: str, git_commit: str) -> str:
    with open(ctest_json_path) as f:
        data = json.load(f)

    lines = [
        "TEST RESULTS RECORD",
        "",
        f"Git commit:  {git_commit}",
        f"Run date:    {datetime.now(timezone.utc).isoformat()}",
        "",
        "Results:",
    ]
    for test in data["tests"]:
        status = "PASS" if test["status"] == "passed" else "FAIL"
        lines.append(f"    {test['name']}: {status}")
    return "\n".join(lines)

if __name__ == "__main__":
    # Synthetic CTest-shaped JSON, standing in for real `ctest --output-junit`
    # or CTest's own JSON output, to demonstrate the generation step runs.
    synthetic = {
        "tests": [
            {"name": "TP-PARSE-001", "status": "passed"},
            {"name": "TP-PARSE-002", "status": "passed"},
            {"name": "TP-PARSE-003", "status": "passed"},
            {"name": "TP-PARSE-004", "status": "failed"},
        ]
    }
    with open("/tmp/synthetic_results.json", "w") as f:
        json.dump(synthetic, f)

    print(generate_report("/tmp/synthetic_results.json", "c9777b0a1f2e"))
```

```bash
python3 generate_results_record.py
```

```text
TEST RESULTS RECORD

Git commit:  c9777b0a1f2e
Run date:    2026-08-25T10:14:32.481207+00:00

Results:
    TP-PARSE-001: PASS
    TP-PARSE-002: PASS
    TP-PARSE-003: PASS
    TP-PARSE-004: FAIL
```

This script ran on this host and produced this real output (only the
timestamp will differ on re-run) — generated directly from structured test
output, the same pattern a real pipeline would apply to `ctest`'s actual
JUnit/JSON output instead of the synthetic data used here for
demonstration.

## 5. Problem reports close the loop

Every `FAIL` in a results record needs a Problem Report (PR) with a defined
lifecycle: opened, root-caused, fixed, and — critically — **re-verified
against a new, passing results record tied to the fixed configuration**,
not just marked resolved on the strength of the fix existing.

```text
PROBLEM REPORT PR-0042
Opened: 2026-08-20
Related test: TP-PARSE-004
Description: parse_message does not reject a null `out` pointer; a
             call with out=NULL causes a null pointer dereference
             rather than returning an error code.
Root cause: Missing null check, introduced when the function was
            refactored to accept the current signature.
Fix: Commit a1b2c3d adds an explicit `if (out == NULL) return -1;` check.
Status: Closed
Verification: TP-PARSE-004 re-run against commit a1b2c3d -- PASS.
              See Test Results Record dated 2026-08-22.
```

## 6. Configuration management ties it all together

Certification evidence is only trustworthy if every artifact — requirements,
code, tests, results, coverage reports — can be shown to correspond to the
*same* software baseline. This is why the git commit hash appears in every
artifact above: it's the join key across the entire evidence package,
without which "these tests passed" and "this is the code that shipped"
cannot be connected with certainty.

## Cheat sheet

| Evidence need | Artifact | Generated from |
|---|---|---|
| Non-programmer-readable test intent | Test procedures | Written once per test, tied to a requirement ID |
| Proof a specific build was tested | Test results record | Generated from CI's structured test output, never hand-transcribed |
| Proof required coverage was hit | Structural coverage report | `lcov`/DO-178C-qualified coverage tool output |
| Proof every failure was resolved | Problem reports, each with re-verification evidence | Opened per FAIL, closed only on a new PASS record |
| Everything ties to one software version | Configuration management (commit hash in every artifact) | Version control, referenced everywhere |

## Exercise

1. Write a test procedure (section 2's format) for one test in your own
   project, written so a non-programmer could assess pass/fail against the
   requirement without reading the test code.
2. Adapt the `generate_results_record.py` script to read real output from
   `ctest --output-junit results.xml` (or your test runner's actual
   machine-readable output format) instead of the synthetic JSON, and run
   it against one of your own test suites.
3. Write a Problem Report (section 5's format) for a real or hypothetical
   test failure in your project, including the re-verification step.
4. Identify, for your own project, which artifact from the cheat sheet you
   currently do NOT produce, and describe in two sentences what would be
   needed to start producing it.
5. Explain in three sentences why "the CI badge is green" is not, by
   itself, certification evidence — what's missing, specifically, that the
   artifacts in this module supply?
