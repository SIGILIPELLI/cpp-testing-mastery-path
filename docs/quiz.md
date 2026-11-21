---
title: "Final Quiz"
description: "Test what you have learned across all four levels of C/C++ Testing Mastery Path with a 20-question final quiz."
---

# Final Quiz

20 questions drawn from all four levels of this course. You need **70% (14 out of 20)** to pass —
you can retake it as many times as you like. Your best result is saved in this browser.

<div id="mp-quiz-root"></div>

<script type="application/json" id="mp-quiz-data">
[
  {
    "question": "A manager asks: 'Can you guarantee the release has no bugs if all tests pass?' What's the accurate answer?",
    "options": [
      "Yes, if reviewed",
      "Yes, if coverage is 100%",
      "Yes, if CI is green",
      "No: testing shows the presence of defects, not their absence"
    ],
    "correctIndex": 3,
    "explanation": "Exhaustive testing is impossible."
  },
  {
    "question": "A function takes two 32-bit ints. Why can't you test every combination?",
    "options": [
      "It's only 4 billion",
      "It's easy",
      "About 1.8x10^19 combinations, roughly 585,000 years at a million tests per second",
      "Ints are small"
    ],
    "correctIndex": 2,
    "explanation": "Testing is a selection problem."
  },
  {
    "question": "A valid age range is 18 to 65. Which inputs best expose off-by-one bugs?",
    "options": [
      "1, 2, 3",
      "17, 18, 19, 64, 65 and 66 (boundary values)",
      "100, 200",
      "Random only"
    ],
    "correctIndex": 1,
    "explanation": "Boundary value analysis."
  },
  {
    "question": "Test inputs 20, 30 and 40 are all valid ages. What does equivalence partitioning say?",
    "options": [
      "Test all three always",
      "They're invalid",
      "They're in the same partition, so one representative is usually enough",
      "They're boundaries"
    ],
    "correctIndex": 2,
    "explanation": "Pick one per partition plus boundaries."
  },
  {
    "question": "A defect report says only 'program crashes'. What's missing?",
    "options": [
      "A nicer title",
      "Steps, exact input, environment/build flags and sanitizer output to reproduce it",
      "A screenshot",
      "A priority only"
    ],
    "correctIndex": 1,
    "explanation": "A bug report must let a developer reproduce it alone."
  },
  {
    "question": "A line shows 100% line coverage yet an `if (a && b)` bug slipped through. Why?",
    "options": [
      "gcov is wrong",
      "Executing a line doesn't mean every condition outcome was tested or asserted",
      "Coverage is infallible",
      "The bug isn't real"
    ],
    "correctIndex": 1,
    "explanation": "Use branch/MC/DC coverage and strong assertions."
  },
  {
    "question": "A test passes at -O0 but the product misbehaves at -O2. What should you suspect?",
    "options": [
      "Slow hardware",
      "A flaky CI",
      "A broken test runner",
      "Undefined behavior, which the optimizer may exploit; run UBSan"
    ],
    "correctIndex": 3,
    "explanation": "UB isn't just a wrong value."
  },
  {
    "question": "Tests pass but production corrupts memory occasionally. Which tool finds it?",
    "options": [
      "ctest -R",
      "AddressSanitizer (or valgrind) on the test run",
      "gcov",
      "clang-format"
    ],
    "correctIndex": 1,
    "explanation": "Memory errors can be invisible to assertions."
  },
  {
    "question": "Why mock a payment gateway in a unit test?",
    "options": [
      "Mocks are required",
      "Real networks are slow and non-deterministic; a mock isolates the unit and lets you assert on interactions",
      "The gateway is free",
      "Mocks are prettier"
    ],
    "correctIndex": 1,
    "explanation": "A unit test should fail for one reason."
  },
  {
    "question": "Code reads a register at fixed address 0x40 and can't be unit tested. What's the fix?",
    "options": [
      "Introduce a seam: put register access behind an interface you can fake or stub",
      "Test only on hardware",
      "Remove the register",
      "Skip testing"
    ],
    "correctIndex": 0,
    "explanation": "Untestable as written means it lacks a seam."
  },
  {
    "question": "Which pairing is right: stub, mock, fake?",
    "options": [
      "Mock returns canned only",
      "Fake verifies calls",
      "Stub returns canned answers, mock verifies interactions, fake is a simplified working implementation",
      "All identical"
    ],
    "correctIndex": 2,
    "explanation": "Choose the right double."
  },
  {
    "question": "A parser crashes on one malformed packet found by a fuzzer. What's the right follow-up?",
    "options": [
      "Disable the fuzzer",
      "Ignore",
      "Delete the corpus",
      "Minimize the input and add it as a regression test with the fix"
    ],
    "correctIndex": 3,
    "explanation": "Fuzz findings become permanent tests."
  },
  {
    "question": "Which property fits property-based testing of sort()?",
    "options": [
      "Output equals input",
      "Output is ordered and a permutation of the input",
      "Output is empty",
      "Output has fixed values"
    ],
    "correctIndex": 1,
    "explanation": "Check invariants over many generated inputs."
  },
  {
    "question": "A race appears once in 1000 runs. Which tool helps?",
    "options": [
      "gcov",
      "AddressSanitizer only",
      "ThreadSanitizer plus repeated stress runs",
      "clang-format"
    ],
    "correctIndex": 2,
    "explanation": "Timing bugs need instrumentation."
  },
  {
    "question": "You must change untested legacy code. What first?",
    "options": [
      "Refactor immediately",
      "Delete it",
      "Write characterization tests capturing current behavior, creating seams as needed",
      "Rewrite"
    ],
    "correctIndex": 2,
    "explanation": "Get it under test first."
  },
  {
    "question": "Which is the best pipeline order for C/C++ test CI?",
    "options": [
      "Manual only",
      "Everything serially at the end",
      "Fast checks first (build, unit tests), then sanitizers, coverage, static analysis and longer fuzzing",
      "Fuzz only"
    ],
    "correctIndex": 2,
    "explanation": "Fast feedback with layered depth."
  },
  {
    "question": "A safety standard requires MC/DC coverage. What does that mean?",
    "options": [
      "Every line ran",
      "Every function was called",
      "Each condition must be shown to independently affect the decision outcome",
      "Every file compiled"
    ],
    "correctIndex": 2,
    "explanation": "A rigorous structural criterion."
  },
  {
    "question": "What does a requirements traceability matrix reveal?",
    "options": [
      "Test speed",
      "Code size",
      "Which requirements have tests and which don't",
      "Bug count"
    ],
    "correctIndex": 2,
    "explanation": "Gaps in verification."
  },
  {
    "question": "A team sets a 90% coverage gate and tests start asserting nothing. What's the lesson?",
    "options": [
      "Metrics become targets and get gamed; pair coverage with assertion quality (e.g. mutation testing)",
      "Remove tests",
      "Raise to 100%",
      "Ignore"
    ],
    "correctIndex": 0,
    "explanation": "Goodhart's law."
  },
  {
    "question": "When should you write an in-house test framework?",
    "options": [
      "Always",
      "To save time",
      "When constraints (tiny targets, certification) rule out existing ones, accepting the maintenance cost",
      "Never"
    ],
    "correctIndex": 2,
    "explanation": "Prefer off-the-shelf when possible."
  }
]
</script>

## Coding challenges

Try these on your own computer, then compare your output. Nothing you write is sent anywhere.

### Boundary value analysis

Write boundary_values(lo, hi) returning the inputs to test for a valid range [lo, hi]: just below, on and just above each boundary, plus a nominal middle value. Then run them through valid_age (18 to 65).

**Starter code**

```python
def boundary_values(lo, hi):
    # return [lo-1, lo, lo+1, a nominal middle value, hi-1, hi, hi+1]
    return []

def valid_age(n):
    return 18 <= n <= 65

cases = boundary_values(18, 65)
print(cases)
print([valid_age(n) for n in cases])
```

**Expected output**

```text
[17, 18, 19, 41, 64, 65, 66]
[False, True, True, True, True, True, False]
```

<details><summary>Show solution and explanation</summary>

<pre><code>def boundary_values(lo, hi):
    &quot;&quot;&quot;Boundary value analysis for a valid range [lo, hi]:
    just below, on and just above each boundary, plus a nominal value.&quot;&quot;&quot;
    mid = (lo + hi) // 2
    return [lo - 1, lo, lo + 1, mid, hi - 1, hi, hi + 1]

def valid_age(n):
    return 18 &lt;= n &lt;= 65

cases = boundary_values(18, 65)
print(cases)
print([valid_age(n) for n in cases])
</code></pre>

<p>Bugs cluster at boundaries (off-by-one: &lt; versus &lt;=). Testing both sides of each edge detects them with just seven cases instead of the whole range. Verified by running the solution.</p>
</details>

### Mutation score

Implement mutation_report(ages): check that the suite passes on the original is_adult, then report which mutants it kills (the suite fails on them) and which survive. Compare a weak suite that only tests age 40 with a strong suite that tests 17, 18 and 40.

**Starter code**

```python
def is_adult(age):
    return age >= 18

# Mutants: small deliberate changes a good test suite must notice (kill).
MUTANTS = {
    "boundary >= to >": lambda age: age > 18,
    "threshold 18 to 21": lambda age: age >= 21,
    "negated": lambda age: not (age >= 18),
    "always true": lambda age: True,
}

def run_suite(fn, ages):
    """The suite passes only if every expected result matches."""
    expected = {17: False, 18: True, 40: True}
    return all(fn(a) == expected[a] for a in ages)

def mutation_report(ages):
    # return (killed, survived): names of mutants the suite detects (suite fails on them)
    # and those it misses. The suite must pass on the original is_adult first.
    return [], list(MUTANTS)

for label, ages in [("weak suite   ", [40]), ("strong suite ", [17, 18, 40])]:
    killed, survived = mutation_report(ages)
    print("%s score %d/%d  survived: %s" % (label, len(killed), len(MUTANTS), survived))
```

**Expected output**

```text
weak suite    score 1/4  survived: ['boundary >= to >', 'threshold 18 to 21', 'always true']
strong suite  score 4/4  survived: []
```

<details><summary>Show solution and explanation</summary>

<pre><code>def is_adult(age):
    return age &gt;= 18

# Mutants: small deliberate changes a good test suite must notice (kill).
MUTANTS = {
    &quot;boundary &gt;= to &gt;&quot;: lambda age: age &gt; 18,
    &quot;threshold 18 to 21&quot;: lambda age: age &gt;= 21,
    &quot;negated&quot;: lambda age: not (age &gt;= 18),
    &quot;always true&quot;: lambda age: True,
}

def run_suite(fn, ages):
    &quot;&quot;&quot;The suite passes only if every expected result matches.&quot;&quot;&quot;
    expected = {17: False, 18: True, 40: True}
    return all(fn(a) == expected[a] for a in ages)

def mutation_report(ages):
    assert run_suite(is_adult, ages), &quot;suite must pass on the original code&quot;
    killed = [name for name, fn in MUTANTS.items() if not run_suite(fn, ages)]
    survived = [name for name in MUTANTS if name not in killed]
    return killed, survived

for label, ages in [(&quot;weak suite   &quot;, [40]), (&quot;strong suite &quot;, [17, 18, 40])]:
    killed, survived = mutation_report(ages)
    print(&quot;%s score %d/%d  survived: %s&quot; % (label, len(killed), len(MUTANTS), survived))
</code></pre>

<p>The weak suite kills only the &#x27;negated&#x27; mutant yet still passes on the original, much like a high-coverage suite with weak assertions. Boundary tests (17 and 18) kill all four. Verified by running the solution.</p>
</details>
