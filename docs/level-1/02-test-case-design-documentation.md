# 02 · Test Case Design & Documentation

A test that lives only in your head is not a test — it's a memory. This module
is about turning intent into artifacts other people can execute, review, and
trust: test scenarios, test cases, test data, and the traceability that ties
them back to requirements.

## 1. Scenario vs. case vs. step

These three levels of granularity are constantly confused. The difference is
size and precision.

| Level | Question it answers | Granularity | Example |
|---|---|---|---|
| **Test scenario** | *What* should we test? | One sentence, high level | "Verify the temperature parser handles out-of-range input" |
| **Test case** | *How* exactly do we test one specific thing, with what data and what expected result? | Precise, repeatable | "Given input string `\"999\"`, `parse_temperature()` returns `PARSE_RANGE_ERROR` and leaves `*out` unmodified" |
| **Test step** | What do I physically do next? | One action | "3. Call `parse_temperature(\"999\", &out)`" |

A single scenario typically explodes into 5–15 test cases. Start by
brainstorming scenarios (fast, broad, no detail), then expand the risky ones
into cases (slow, precise). Doing it the other way round — writing detailed
cases immediately — means you burn your energy on the first feature and never
map the rest.

## 2. The test case template

This is the workhorse artifact of manual testing. Use this template — or your
team's variant of it — for everything.

| Field | Purpose |
|---|---|
| **Test Case ID** | Unique, stable identifier. Never reuse a retired ID. |
| **Title** | One line, describes the *check*, not the feature |
| **Module / Component** | Which part of the system |
| **Requirement ID** | The requirement this verifies (for traceability, section 5) |
| **Priority** | P1 (critical path) / P2 (important) / P3 (nice to have) |
| **Type** | Positive / Negative / Boundary / Error-handling |
| **Preconditions** | State that must exist before step 1 |
| **Test data** | Exact inputs, written out literally |
| **Steps** | Numbered, imperative, one action each |
| **Expected result** | Exactly what should happen — observable and unambiguous |
| **Actual result** | Filled in at execution time |
| **Status** | Not Run / Pass / Fail / Blocked / Skipped |
| **Environment** | Compiler, flags, OS, target — critical for C/C++ |
| **Author / Date** | Who wrote it, when |
| **Notes** | Anything the executor needs to know |

### A worked example

Suppose the requirement is:

> **REQ-TEMP-004** — `int parse_temperature(const char *text, int *out_celsius)`
> shall parse a decimal integer temperature in the inclusive range −40 to 125.
> On success it shall write the value to `*out_celsius` and return `0`. On a
> value outside that range it shall return `-2` and leave `*out_celsius`
> unmodified. On text that is not a valid decimal integer it shall return `-1`
> and leave `*out_celsius` unmodified.

A test case against it:

| Field | Value |
|---|---|
| **Test Case ID** | TC-TEMP-012 |
| **Title** | Above-maximum temperature is rejected without modifying the output parameter |
| **Module** | `temperature.c` / `parse_temperature()` |
| **Requirement ID** | REQ-TEMP-004 |
| **Priority** | P1 |
| **Type** | Boundary / Negative |
| **Preconditions** | Test harness built with `gcc -std=c11 -Wall -Wextra -g`; no global state required |
| **Test data** | `text = "126"`, `out_celsius` pre-set to sentinel `0x5A5A5A5A` |
| **Steps** | 1. Set `int out = 0x5A5A5A5A;`<br>2. Call `int rc = parse_temperature("126", &out);`<br>3. Record `rc`<br>4. Record `out` |
| **Expected result** | `rc == -2` **and** `out == 0x5A5A5A5A` (unchanged) |
| **Actual result** | _(filled at execution)_ |
| **Status** | Not Run |
| **Environment** | Ubuntu 24.04, gcc 13.2, x86-64, `-O0` |
| **Author / Date** | A. Tester / 2026-07-30 |
| **Notes** | 126 is max+1. Pair with TC-TEMP-011 (125, the max, must pass). The sentinel value checks the "leave unmodified" half of the requirement, which is easy to forget. |

Three things in that example are worth copying into your own habits:

- **The sentinel.** The requirement says the output is left unmodified. Most
  testers check only the return code and miss half the requirement. Pre-loading
  a recognizable value (`0x5A5A5A5A`, `0xDEADBEEF`) makes "unmodified" testable.
- **The paired case.** Boundary cases come in pairs — max and max+1. A case that
  tests only the rejection side doesn't prove the boundary is in the right place.
- **The environment field.** In C/C++ this is not bureaucracy. The same test can
  pass at `-O0` and fail at `-O2`.

## 3. Writing expected results that actually work

The most common defect *in test cases themselves* is a vague expected result.

| Bad expected result | Why it fails | Better |
|---|---|---|
| "Works correctly" | Two executors will disagree | "Returns 0 and writes 25 to `*out_celsius`" |
| "Doesn't crash" | Doesn't say what it *should* do | "Returns -1; process exit code 0; no output on stderr" |
| "Handles the error" | Which error, handled how? | "Returns `-1`; `errno` untouched; `*out` unchanged" |
| "Value is correct" | Correct per what? | "Value equals 3 (= 7/2 truncated toward zero, per REQ-DIV-002)" |
| "Fast enough" | Unmeasurable | "Completes within 5 ms on the reference build" |

The test for a good expected result: **could two different people execute this
case and disagree on whether it passed?** If yes, rewrite it.

For C and C++ specifically, a complete expected result often has to name
several observable channels:

- return value
- output parameters (and which ones must be *unchanged*)
- `errno` or global state
- stdout / stderr content
- process exit code (a crash is exit-by-signal, not a normal return)
- memory state — "no leaks", "no invalid accesses" (verified by tooling, Level 2)

## 4. Positive, negative, and boundary cases

Every function you test deserves cases from all three families. Testers new to
the job write almost exclusively positive cases; that is where most missed bugs
come from.

| Family | Intent | For `parse_temperature` |
|---|---|---|
| **Positive** | Valid input produces the correct result | `"25"` → 0, out = 25 |
| **Negative** | Invalid input is rejected correctly | `"abc"` → −1; `""` → −1; `NULL` text → −1 |
| **Boundary** | Behaviour right at the edges of valid ranges | `"-40"`, `"-41"`, `"125"`, `"126"` |
| **Error-handling** | The system behaves sanely when something external fails | out-of-memory, file missing, device not responding |

A useful heuristic for a C function signature: **every parameter is an attack
surface.** For `int parse_temperature(const char *text, int *out_celsius)`:

- `text` — valid, empty, `NULL`, whitespace-only, very long (buffer risk),
  leading `+`, leading zeros, embedded null, non-ASCII
- `out_celsius` — valid pointer, `NULL` pointer (what should happen? if the
  requirement doesn't say, that's a requirements defect — go ask)

## 5. Test data design

Test data is a first-class artifact, not something invented at execution time.
Write it down literally, including the exact bytes for anything tricky.

| Data class | Examples for a C string parser |
|---|---|
| Typical / nominal | `"25"`, `"0"`, `"-10"` |
| Boundary | `"-40"`, `"125"`, `"-41"`, `"126"` |
| Malformed | `"abc"`, `"2 5"`, `"25.5"`, `"--25"`, `"0x19"` |
| Empty / absent | `""`, `NULL` |
| Extreme size | a 10,000-character digit string (does it overflow an internal buffer?) |
| Numeric extremes | `"2147483647"`, `"2147483648"`, `"-2147483648"` (does `atoi` overflow? that's UB) |
| Whitespace | `" 25"`, `"25 "`, `"\t25\n"` |
| Encoding | embedded `\0`, UTF-8 bytes, `\r\n` line ending |

!!! tip "Keep test data in files, not in prose"
    Once you automate (Modules 7–9), your literal test data becomes the argument
    list of a parameterized test. Data written as a clean table converts
    directly; data buried in sentences has to be re-derived. Design it as a
    table from day one.

## 6. Traceability

A **requirements traceability matrix (RTM)** maps requirements to the test cases
that verify them, in both directions. It answers two questions that come up on
every real project:

1. *Forward:* is every requirement covered by at least one test?
2. *Backward:* does every test exist for a reason, or are we running cases that
   verify nothing anyone asked for?

A minimal RTM:

| Req ID | Requirement summary | Test cases | Status |
|---|---|---|---|
| REQ-TEMP-001 | Parses valid decimal integers | TC-TEMP-001, TC-TEMP-002, TC-TEMP-003 | Covered |
| REQ-TEMP-002 | Returns 0 on success and writes the value | TC-TEMP-001, TC-TEMP-004 | Covered |
| REQ-TEMP-003 | Returns −1 on malformed text, leaves output unchanged | TC-TEMP-005 … TC-TEMP-009 | Covered |
| REQ-TEMP-004 | Enforces the −40…125 range | TC-TEMP-010 … TC-TEMP-013 | Covered |
| REQ-TEMP-005 | Handles a `NULL` output pointer safely | — | **GAP** |

That last row is the entire value of the document. The gap is visible, dated,
and assignable. Without an RTM, "we forgot to test the null-pointer path" is
discovered in production.

Traceability is a nice-to-have on a web app and a **hard regulatory requirement**
in safety-critical C/C++ — DO-178C for avionics and ISO 26262 for automotive
both require demonstrable requirement-to-test traceability as certification
evidence. Level 4 covers that in depth; for now, build the habit.

## 7. How much documentation is enough?

Full formal test cases are expensive. Match the weight to the risk:

| Context | Documentation level |
|---|---|
| Safety-critical firmware (medical, automotive, avionics) | Full formal cases, full RTM, reviewed and version-controlled |
| Commercial product, regulated-ish | Formal cases for P1 paths; checklists for the rest |
| Internal tool | Scenario checklists plus an automated suite |
| Throwaway prototype | Exploratory notes |

The one thing that is *always* worth writing down, at every level of formality:
**the expected result.** Steps can be terse. Data can be abbreviated. But an
undocumented expectation is not a test.

## 8. Common mistakes in test case design

| Mistake | Consequence | Fix |
|---|---|---|
| Test case depends on the previous case's leftover state | One failure cascades into ten; cases can't be reordered or run in isolation | Each case sets up and tears down its own state |
| Multiple checks bundled into one case | When it fails you don't know which check failed | One case, one logical assertion (related asserts about the same call are fine) |
| Expected result copied from observed behaviour | You have documented the bug as correct | Derive expected results from the *requirement*, before running anything |
| No environment recorded | Irreproducible in C/C++ | Always record compiler, flags, target |
| Steps written as narrative | Executor improvises, results diverge | Numbered imperative steps |
| ID reuse after deletion | Old defect reports point at the wrong test | IDs are permanent; retire, never recycle |

That third row deserves emphasis. Writing the expected result **before** you run
the code is the discipline that separates testing from observing. If you run
first and record what happened, you will faithfully certify every bug as
intended behaviour.

## Exercise

Take this C function, which is intended to copy a source string into a
fixed-size destination buffer and always null-terminate:

```c
/* Requirement REQ-CPY-001: safe_copy() shall copy at most dest_size-1
   characters from src into dest, and shall always null-terminate dest.
   It shall return the number of characters copied (excluding the
   terminator), or -1 if dest is NULL, src is NULL, or dest_size is 0. */
int safe_copy(char *dest, size_t dest_size, const char *src);
```

Produce three deliverables:

1. **A scenario list** — at least six one-line test scenarios covering the
   requirement.
2. **Three fully documented test cases** using the template in section 2, filled
   in completely (every field). Make one positive, one negative, and one
   boundary case. At least one must use a sentinel/canary value to verify that
   something is *not* modified — for example, filling `dest` with `'X'` before
   the call and checking that bytes past the terminator are untouched.
3. **A traceability matrix** with REQ-CPY-001 broken into its four sub-clauses
   (copy limit, always null-terminate, return count, the three −1 conditions) as
   separate rows, showing which of your cases cover which clause — and marking
   any clause you have not yet covered as a **GAP**.

Keep these; Module 5 will extend them with boundary value analysis, and Module 7
will turn them into GoogleTest code.
