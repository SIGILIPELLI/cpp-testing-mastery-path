# 02 · Safety-Critical Testing Standards

MISRA C/C++, DO-178C, and ISO 26262 are not testing frameworks — they're
process and coding standards that dictate *what evidence* a testing effort
must produce before software is allowed into an airplane, a car's braking
system, or medical equipment. This module is a working orientation: what
each standard actually asks for, how it changes day-to-day testing
practice, and where the tools from earlier in this path fit into that
evidence.

!!! note "Environment note"
    Purely a standards/process module — no code was expected to run, and
    none did. The one illustrative MISRA-violation snippet was checked by
    manual reading against the rule text, not by running a MISRA checker
    (none installed on this host).

## 1. What these standards actually are

| Standard | Domain | What it governs |
|---|---|---|
| MISRA C / MISRA C++ | Any safety-relevant embedded C/C++ | A restricted subset of the language — which constructs are banned or restricted, because they're common sources of undefined/implementation-defined behavior |
| DO-178C | Airborne software | A process standard: for a given criticality level, what verification evidence (requirements traceability, structural coverage, tool qualification) must exist |
| ISO 26262 | Automotive electrical/electronic systems | A process and risk-classification standard (ASIL A-D) analogous to DO-178C, for road vehicles |

None of them replace GoogleTest, sanitizers, or fuzzing — they specify a
minimum bar of *evidence and process rigor* around testing, at a criticality
level proportional to the consequence of failure.

## 2. MISRA: a coding subset, not a test technique

MISRA rules ban or restrict specific language constructs because they're
disproportionately associated with real bugs. A short, concrete example:

```c
/* MISRA C:2012 Rule 14.4 (required): the controlling expression of an
   if/while statement shall have essentially Boolean type. */

int x = get_value();
if (x) {           /* VIOLATION: int used where the rule requires bool */
    do_something();
}

if (x != 0) {       /* compliant: explicit, unambiguous comparison */
    do_something();
}
```

The rule isn't pedantry: `if (x)` reads identically whether `x` is meant as
a boolean flag or a count, and a reviewer (or the next maintainer) cannot
tell from the code alone which was intended — exactly the kind of ambiguity
that produces a real defect when the variable's meaning later changes.

```c
/* Rule 10.1/10.3 family: no implicit conversions between essentially
   different types in an expression. */
uint16_t len = 300;
uint8_t truncated = len;    /* VIOLATION: silent narrowing, MISRA flags it
                                even though C allows it -- this is exactly
                                the class of bug Level 3's fuzz testing
                                (module 04) and UBSan (Level 2, module 05)
                                also independently catch at runtime */
```

MISRA compliance is checked statically (a MISRA-aware static analyzer —
commercial tools like PC-lint, Polyspace, or Coverity; not the open-source
`cppcheck`/`clang-tidy` from Level 2 module 09, which don't implement full
MISRA rule sets), and is a **gate**, not a suggestion, in a MISRA-mandated
project: code that violates a "required" rule does not ship without a
documented, reviewed deviation.

## 3. Structural coverage requirements (DO-178C)

DO-178C assigns a Design Assurance Level (DAL A through E, A being
catastrophic-failure-consequence) to software, and each level mandates a
specific structural coverage criterion on top of functional testing:

| DAL | Consequence of failure | Required structural coverage |
|---|---|---|
| A | Catastrophic (loss of aircraft) | Modified Condition/Decision Coverage (MC/DC) |
| B | Hazardous | Decision coverage |
| C | Major | Statement coverage |
| D | Minor | None mandated beyond functional testing |
| E | No safety effect | None |

**MC/DC** is the standard's signature requirement and worth understanding
concretely: for a decision with multiple conditions, every condition must
be shown to *independently* affect the decision's outcome — not merely that
every condition has been both true and false somewhere in the suite (that's
condition coverage, a weaker bar).

```c
/* if (a && b) -- MC/DC requires test cases proving EACH of a, b
   independently flips the outcome, holding the other fixed: */
/* a=T, b=T -> true   (baseline)                                   */
/* a=F, b=T -> false  (shows 'a' independently controls the result) */
/* a=T, b=F -> false  (shows 'b' independently controls the result) */
/* Three cases, not four (a=F,b=F is redundant for MC/DC purposes,
   though a decent test suite would likely include it anyway). */
```

This is `gcov`/`lcov` line-and-branch coverage (Level 2, module 06) taken
one step further — decision coverage tells you every branch was taken both
ways, MC/DC tells you every *individual condition* was shown to matter, and
matters because `a && b` being reached both true and false says nothing
about whether `b`'s value alone was ever actually consequential.

```bash
# Ordinary lcov gets you line/branch coverage, not MC/DC directly.
# Specialized DO-178C coverage tools (e.g. LDRA, VectorCAST) compute and
# report MC/DC against a qualified toolchain -- part of what "tool
# qualification" (section 5) exists to certify.
lcov --capture --directory build --output-file coverage.info --rc branch_coverage=1
```

## 4. Requirements traceability, briefly (full depth in module 03)

Every requirement must trace to at least one test, and every test must
trace back to the requirement(s) it verifies — a bidirectional map that
DO-178C and ISO 26262 both require as certification evidence, and which
module 03 covers in depth, including how to build and maintain the matrix.

```text
REQ-BRAKE-014: "The system shall disengage cruise control within 200ms
                of brake pedal input exceeding 5% travel."
    -> TEST-BRAKE-014-01: unit test, mocked pedal sensor, asserts
       disengage() called within one control-loop tick after a 6%
       input value.
    -> TEST-BRAKE-014-02: HIL test (Level 3, module 03), real pedal
       hardware, measures actual disengagement latency.
```

An orphan requirement (no test) or an orphan test (traces to nothing) is
itself a finding in a certification audit — the traceability *matrix's
completeness* is evidence, independent of whether the underlying tests pass.

## 5. Tool qualification

If a tool's output is trusted as certification evidence (a compiler, a
coverage tool, a static analyzer) without a human independently verifying
every result, the tool itself typically needs **qualification** — documented
evidence that the tool reliably does what it claims, for the specific way
it's being used. This is why safety-critical projects often use qualified
commercial toolchains rather than plain `gcc`/`lcov`: not because the open
tools are worse, but because qualification evidence for them doesn't
already exist in a form auditors accept.

## 6. ISO 26262's ASIL and the same shape of argument

ISO 26262 mirrors DO-178C's structure for automotive: ASIL A (lowest) to
ASIL D (highest), each specifying a growing rigor bar across the full
lifecycle — hazard analysis, requirements traceability, coverage targets,
and process documentation — not just "write more tests." The core lesson
transfers directly: **higher assurance level does not mean "test harder" in
some vague sense, it means a specific, enumerable, auditable set of
additional evidence.**

## Cheat sheet

| Question | Where the answer lives |
|---|---|
| Is this C construct even allowed? | MISRA C/C++ rule set, checked by a MISRA-aware static analyzer |
| How much structural coverage is required? | DO-178C DAL / ISO 26262 ASIL table (section 3) |
| Does every requirement have a test? | The requirements traceability matrix (module 03) |
| Can I trust this analysis tool's output as evidence? | Only if it (or its use) is qualified (section 5) |
| What's actually different at a higher assurance level? | A specific, enumerated evidence bar — not "more effort" in the abstract |

## Exercise

1. Take one function from an earlier module in this path (e.g. `frame_parse`
   from Level 3 module 02) and identify one place its code, as written,
   would likely draw a MISRA finding (implicit conversion, non-boolean
   condition, or similar) — rewrite that line to be MISRA-compliant.
2. For a decision with three conditions (`if (a && (b || c))`), work out
   the minimal set of test cases that would satisfy MC/DC, and compare the
   count against what would satisfy only condition coverage.
3. Write one requirement (in the `REQ-...` style of section 4) for a
   feature in your own project, and list every test that currently traces
   to it — if none do, that's the finding a real audit would raise.
4. Explain in three sentences why a project targeting ASIL D would need
   qualification evidence for its coverage tool even if the tool is
   technically correct — what's the actual argument for qualification
   beyond correctness?
5. Write two sentences comparing DO-178C's DAL table to ISO 26262's ASIL
   scale: what's structurally the same about how they scale rigor with
   consequence, and what's different about the domains they apply to?
