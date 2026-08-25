# 03 · Requirements-Based Testing & Traceability

Every prior module in this path starts from "here is some code, let's test
it." Requirements-based testing inverts the order: start from "here is what
the system must do," derive tests directly from that statement before (or
independent of) looking at the implementation, and maintain an explicit,
auditable link between the two. This is both a testing technique and — as
module 02 covered — a certification requirement in safety-critical domains.

!!! note "Environment note"
    This module is primarily process and documentation technique. The one
    C++ traceability-tooling sketch was checked by manual reading, not
    compiled, given the broken-libc++ state noted throughout this path; a
    small Python matrix-validation script in section 5 **was** run on this
    host and its output is real.

## 1. Requirements-based vs implementation-based tests

| | Implementation-based | Requirements-based |
|---|---|---|
| Derived from | Reading the code | Reading the specification, independent of the code |
| Catches | Bugs where the code does the wrong thing relative to itself | Bugs where the code correctly implements a misunderstanding of the requirement |
| Risk if the developer writes both code and tests | Tests can encode the same misunderstanding as the code | Lower — a second, requirement-first pass is a genuinely independent check |

The distinction matters most exactly when a developer writes both the
implementation and its tests: a requirements-based test derived by someone
(or some process) reading the spec fresh, without looking at the code
first, is structurally more likely to catch "I implemented the wrong
thing" — as opposed to "I implemented what I intended, correctly."

## 2. A worked example

```text
REQ-PARSE-003: "The message parser shall reject any input where the
                declared payload length would cause the total message
                to exceed the buffer bounds, returning an error code
                rather than reading beyond the buffer."
```

A requirements-based test is derived from that sentence alone:

```c
/* Derived directly from REQ-PARSE-003's wording, before/independent of
   reading parse_message's implementation. */
static void test_REQ_PARSE_003_declared_length_exceeds_buffer(void) {
    uint8_t buf[3] = {0x01, 0xFF, 0xFF};  /* declares a 65535-byte payload
                                              in a 3-byte buffer */
    message_t m;
    int result = parse_message(buf, sizeof(buf), &m);
    assert(result != 0);   /* "return an error code" -- exact code isn't
                               specified by the requirement, so the test
                               only asserts what the requirement asserts */
}
```

This is the exact `parse_message` from Level 3's capstone — the point is
that `test_rejects_overrunning_length` in that module was written by
looking at the code and thinking of an edge case; this version is written
by reading the requirement and deriving the same case independently. In a
codebase with a real traceability process, the requirement ID appears in
the test name or a companion annotation specifically so the link survives
refactors and renames.

## 3. The traceability matrix

A traceability matrix is a table (however it's stored — a spreadsheet, a
generated report, comments in code) mapping every requirement to the tests
that verify it, and vice versa.

```text
| Requirement    | Tests                                          | Status  |
|----------------|--------------------------------------------------|---------|
| REQ-PARSE-001  | test_REQ_PARSE_001_rejects_short_buffer          | Pass    |
| REQ-PARSE-002  | test_REQ_PARSE_002_parses_valid_message          | Pass    |
| REQ-PARSE-003  | test_REQ_PARSE_003_declared_length_exceeds_buffer | Pass    |
| REQ-PARSE-004  | (none)                                            | ORPHAN  |
```

The `ORPHAN` row is the matrix doing its job: `REQ-PARSE-004` exists in the
specification but has no verifying test — a gap that's invisible by reading
the test suite alone (nothing in the test suite tells you what's *missing*)
and only visible by reading the matrix.

## 4. Generating a matrix from annotated tests, not maintaining it by hand

A hand-maintained spreadsheet drifts from reality the first time someone
adds a test and forgets to update it. Tie the requirement ID to the test at
the source, and generate the matrix mechanically.

```cpp
// A lightweight convention: encode the requirement ID in the test name
// (GoogleTest) so a script can extract the mapping without a separate
// annotation system to keep in sync.
TEST(ParserRequirements, REQ_PARSE_003_DeclaredLengthExceedsBuffer) {
    uint8_t buf[3] = {0x01, 0xFF, 0xFF};
    message_t m;
    EXPECT_NE(parse_message(buf, sizeof(buf), &m), 0);
}
```

```bash
# Extract requirement IDs from test names, cross-reference against the
# requirements list, and flag orphans in both directions.
./test_binary --gtest_list_tests | grep -oE 'REQ_[A-Z]+_[0-9]+' | sort -u > tested.txt
grep -oE 'REQ-[A-Z]+-[0-9]+' requirements.md | tr '-' '_' | sort -u > required.txt
comm -23 required.txt tested.txt   # requirements with no test
comm -13 required.txt tested.txt   # tests referencing a requirement that doesn't exist
```

## 5. A real, runnable matrix-completeness check

```python
#!/usr/bin/env python3
# check_traceability.py
import re
import sys

def extract_requirement_ids(text, pattern):
    return set(re.findall(pattern, text))

def main():
    requirements_doc = """
    REQ-PARSE-001: Reject buffers shorter than the minimum header size.
    REQ-PARSE-002: Parse a well-formed message correctly.
    REQ-PARSE-003: Reject a declared length that exceeds the buffer.
    REQ-PARSE-004: Reject a null buffer or null output pointer.
    """
    test_source = """
    TEST(Parser, REQ_PARSE_001_ShortBufferRejected) { }
    TEST(Parser, REQ_PARSE_002_ValidMessageParses) { }
    TEST(Parser, REQ_PARSE_003_OverrunRejected) { }
    """

    required = {r.replace('-', '_') for r in
                extract_requirement_ids(requirements_doc, r'REQ-[A-Z]+-\d+')}
    tested = extract_requirement_ids(test_source, r'REQ_[A-Z]+_\d+')

    orphan_requirements = required - tested
    orphan_tests = tested - required

    if orphan_requirements:
        print(f"FAIL: requirements with no test: {sorted(orphan_requirements)}")
    if orphan_tests:
        print(f"WARN: tests referencing unknown requirements: {sorted(orphan_tests)}")

    if orphan_requirements:
        sys.exit(1)
    print("OK: every requirement has at least one test" if not orphan_requirements else "")

if __name__ == "__main__":
    main()
```

```bash
python3 check_traceability.py
```

```text
FAIL: requirements with no test: ['REQ_PARSE_004']
```

This exact script was run on this host and produced this exact output —
`REQ-PARSE-004` ("reject a null buffer") is genuinely present in the mock
requirements doc but has no matching test in the mock source, so the script
correctly flags it and exits non-zero, precisely the behavior you'd wire
into a CI gate.

## 6. Bidirectional traceability catches two different failure modes

- **Requirement → test (forward)**: catches untested requirements — the
  more commonly discussed direction, and the one most audits focus on.
- **Test → requirement (backward)**: catches tests that exist for no
  documented reason, or that were written against a since-changed or
  since-deleted requirement. A stale backward-trace is a signal the test
  may be testing an implementation detail rather than a real, current
  requirement — worth a second look, not automatic deletion.

## 7. Where this fits with earlier testing techniques

Requirements-based tests are not a replacement for the rest of this path —
they're an additional *source* of test cases, layered onto the same
GoogleTest/CMake/CI infrastructure from Levels 1-3. A mature suite has
tests from multiple origins: requirements-derived, boundary-analysis-derived
(Level 1), property-derived (Level 3, module 05), and fuzz-discovered
(Level 3, module 04) — each catching a different class of gap, and the
traceability matrix specifically only accounts for the first.

## Cheat sheet

| Question | Practice |
|---|---|
| Does every requirement have a test? | Generate a traceability matrix, don't maintain it by hand |
| How do I keep the link from rotting? | Encode requirement IDs in test names/annotations at the source |
| What does an orphan requirement mean? | A genuine coverage gap, invisible from the test suite alone |
| What does an orphan test mean? | Possibly testing a stale or undocumented requirement — worth review |
| Does this replace other testing techniques? | No — it's one more source of test cases, layered on the same infrastructure |

## Exercise

1. Write three requirements (`REQ-...` style) for the `BoundedStack` from
   Level 2's capstone project, covering push, pop, and the overflow
   behavior, then derive one test per requirement without re-reading the
   implementation first.
2. Run (or adapt) the `check_traceability.py` script from section 5 against
   your own three requirements and tests, and confirm it reports no
   orphans in either direction.
3. Deliberately delete one test, re-run the script, and confirm it
   correctly flags the now-orphaned requirement.
4. Deliberately rename a requirement ID in your requirements list without
   updating the corresponding test name, re-run the script, and confirm it
   flags both an orphan requirement and an orphan test.
5. Write two sentences on one requirement in your own project (real or
   hypothetical) that you suspect has no directly-traceable test today,
   and what the requirements-based test for it would look like.
