# 01 · What Is Software Testing?

Software testing is the disciplined activity of **evaluating a program against
expectations** in order to find defects and provide information about quality.
That's the whole job in one sentence — but every word in it matters.

- *Disciplined* — testing is not "clicking around until something breaks." It
  is planned, documented, and repeatable.
- *Against expectations* — you cannot test without knowing what the correct
  behaviour is. If nobody can tell you the expected result, you have found a
  requirements defect before you've run a single test.
- *Find defects* — the purpose is to find problems, not to confirm the program
  works. A test run that finds nothing is a *less* informative test run.
- *Provide information* — you don't decide whether to ship; you give the people
  who do decide an accurate picture of the risk.

!!! note "Testing shows the presence of bugs, never their absence"
    Dijkstra's line is the foundational idea of the whole field. A test that
    passes proves that one specific path, with one specific set of inputs, on
    one specific build, behaved as expected. It says nothing about the paths you
    did not run. This is why coverage, sanitizers and fuzzing (Levels 2 and 3)
    exist — they widen the set of behaviours you actually exercise.

## 1. Manual testing vs. test automation

These are not rival camps; they are two tools with different strengths, and a
competent C/C++ tester uses both.

| | Manual testing | Test automation |
|---|---|---|
| Who runs it | A person, following (or improvising around) a plan | A machine, on every commit |
| Best at | New features, exploratory work, usability, judging "does this feel wrong?" | Repetition — regression, boundary sweeps, thousands of inputs |
| Cost shape | Cheap to start, expensive to repeat | Expensive to build, nearly free to repeat |
| Feedback speed | Minutes to days | Seconds to minutes |
| Finds | Unexpected problems the author never considered | Regressions in things you already thought about |
| Fails at | Doing the same 400 checks reliably at 5pm on a Friday | Noticing that a correct-looking output is nonsense |

The practical rule: **anything you will check more than about three times should
be automated; anything requiring judgement should be manual.** A boundary sweep
over an integer parser belongs in GoogleTest. Deciding whether a crash dump is
"the same bug we saw last week" belongs to a person.

In C and C++ specifically, automation carries extra weight because the failure
modes are so often invisible. A buffer overrun by one byte usually produces no
visible symptom at all when a human runs the program by hand — but under
AddressSanitizer (Level 2) it aborts immediately with a precise diagnosis. Some
categories of C/C++ bug are effectively *undetectable* by manual testing.

## 2. QA vs. QC vs. testing

These three terms get used interchangeably in job ads and are genuinely
different things:

| Term | Full name | Scope | Nature | Example |
|---|---|---|---|---|
| **QA** | Quality Assurance | The whole process | **Preventive** — improve how work is done | Introducing a code review checklist, enforcing a coding standard, mandating a static analysis gate |
| **QC** | Quality Control | The product | **Detective** — inspect what was produced | Running the regression suite on release candidate 3, reviewing test results |
| **Testing** | — | A specific activity | An *implementation technique* of QC | Executing test case `TC-014` and recording pass/fail |

A memorable framing: **QA is about the recipe; QC is about tasting the soup;
testing is the act of picking up the spoon.** In small teams one person does all
three, which is exactly why the distinction is worth knowing — you should be
able to say which hat you're wearing when you propose work.

## 3. The SDLC and where testing lives

The **SDLC** (Software Development Life Cycle) is the end-to-end process of
building software. A conventional breakdown:

1. **Requirements** — what should it do?
2. **Design** — how will it be structured?
3. **Implementation** — write the code
4. **Testing** — verify it
5. **Deployment** — ship it
6. **Maintenance** — fix and extend it

The naive reading — "testing is step 4" — is the single most expensive mistake
a team can make. The reason is the **cost-of-defect curve**: the price of fixing
a defect grows by roughly an order of magnitude at each stage it survives.

| Defect found during | Relative cost to fix | Why |
|---|---|---|
| Requirements | 1× | Edit a sentence in a document |
| Design | ~5× | Redraw a module boundary |
| Implementation | ~10× | Rewrite a function, re-review |
| System test | ~50× | Diagnose, fix, re-test, re-integrate |
| Production | ~100×+ | All of the above, plus recall/patch/support/reputation — and for safety-critical embedded firmware, possibly a physical field service call per unit |

The conclusion the industry drew is **shift-left testing**: move test activity as
early as possible. A tester who reads the requirements document and asks "what
should `parse_temperature()` do if the string is empty?" has just prevented a
defect at 1× cost that would otherwise have been found at 50×.

## 4. The STLC — the testing life cycle

Inside the SDLC, testing has its own life cycle, the **STLC** (Software Testing
Life Cycle). Each phase has entry criteria (what must be true to start), the
activity itself, and a deliverable.

| # | Phase | Activity | Deliverable |
|---|---|---|---|
| 1 | **Requirement analysis** | Read requirements, identify what is testable, ask clarifying questions | List of testable requirements, list of ambiguities |
| 2 | **Test planning** | Decide scope, approach, resources, schedule, risks | Test plan |
| 3 | **Test case design** | Write the individual test cases and prepare test data | Test cases, test data, traceability matrix |
| 4 | **Environment setup** | Build the target, install the toolchain, prepare hardware/emulator | A working, documented test environment |
| 5 | **Test execution** | Run the cases, record actual results, log defects | Execution log, defect reports |
| 6 | **Test closure** | Summarize results, capture lessons learned | Test summary report |

Modules 2, 4 and 5 of this level cover phases 3 and 5 in depth; module 10 has
you produce a real deliverable for phases 2 through 5.

!!! tip "Verification vs. validation"
    **Verification** asks "are we building the product *right*?" — does the code
    match the specification. **Validation** asks "are we building the *right*
    product?" — does the specification match what the user actually needs. A
    unit test is verification. A user acceptance test is validation. You can
    pass every verification test and still ship something useless.

## 5. The seven principles of testing

These are the standard ISTQB principles, and they are genuinely useful as a
checklist when a project's testing feels wrong:

1. **Testing shows the presence of defects**, not their absence.
2. **Exhaustive testing is impossible.** A single `int` parameter has ~4.3
   billion values; two of them have 1.8×10¹⁹ combinations. You must *select*.
3. **Early testing saves time and money** (the cost curve above).
4. **Defects cluster.** A small number of modules typically contain most of the
   bugs — usually the most complex or most recently changed ones. Follow the
   clusters.
5. **Tests wear out** (the "pesticide paradox"). Running the same suite forever
   stops finding new bugs; the suite must keep growing and changing.
6. **Testing is context dependent.** A pacemaker's firmware and a hobby LED
   blinker do not get the same testing.
7. **Absence-of-errors is a fallacy.** A bug-free implementation of the wrong
   requirement is still a failed product.

## 6. Why testing C and C++ is different

Everything above applies to any language. This section is why this course exists
separately from a general QA course.

### No garbage collector — memory is your problem

In Java, Python, or C#, a forgotten object is eventually collected. In C and
C++, every `malloc`/`new` needs a matching `free`/`delete`. This creates whole
bug categories that simply do not exist in managed languages:

| Bug | What happens | Typical visible symptom |
|---|---|---|
| **Memory leak** | Allocation never freed | Nothing at all — until a long-running process exhausts RAM days later |
| **Double free** | Same pointer freed twice | Heap corruption, crash somewhere *else* entirely |
| **Use-after-free** | Pointer dereferenced after `free` | Usually "works fine", occasionally garbage data, occasionally a crash |
| **Buffer overflow** | Write past the end of an array | Silently corrupts an adjacent variable; the security exploit of choice |
| **Uninitialized read** | Reading a variable never assigned | Works in debug builds, fails in release builds |

Note the pattern in that last column: **the symptom is usually nothing.** A test
suite of hand-run functional checks will pass on all five of these. This is the
central practical fact of C/C++ testing.

### Undefined behaviour

The C and C++ standards define certain operations as **undefined behaviour**
(UB): signed integer overflow, dereferencing a null pointer, reading an
uninitialized variable, out-of-bounds array access, and dozens more. "Undefined"
does not mean "unpredictable but bounded" — it means the standard imposes *no
requirements whatsoever*. The compiler is entitled to assume UB never occurs and
optimize accordingly.

The practical consequence for a tester:

```c
/* This "null check" can be deleted entirely by an optimizing compiler.
   Dereferencing p on the first line is UB, so the compiler is allowed to
   assume p is not null -- which makes the check below provably dead code. */
int value = *p;
if (p == NULL) {
    return -1;      /* may never execute in an -O2 build */
}
```

This means a UB bug can **behave differently between a debug build and a release
build, between GCC and Clang, and between compiler versions**. "It passed on my
machine" is a much weaker statement in C/C++ than elsewhere. Testing only the
debug build is a real and common mistake.

### Platform and toolchain variance

| Dimension | Why it changes results |
|---|---|
| Compiler (GCC / Clang / MSVC) | Different optimizations, different UB exploitation, different warnings |
| Optimization level (`-O0` vs `-O2`) | Changes timing, inlining, and how UB manifests |
| Architecture (x86-64 / ARM) | Different `int` sizes on some targets, different alignment rules, different endianness |
| Standard library implementation | libstdc++ vs libc++ vs newlib behave differently at the edges |
| Target (host PC vs embedded MCU) | Limited RAM, no filesystem, no `printf`, no OS |

A C/C++ test matrix therefore frequently has multiple *build configurations* as
an axis, not just multiple test cases. Level 3 covers this properly.

### The tester's compensating tools

Because manual observation cannot see these bugs, C/C++ testing leans on
tooling that makes the invisible visible:

| Tool | Catches | Covered in |
|---|---|---|
| Compiler warnings (`-Wall -Wextra`) | Suspicious constructs, at zero runtime cost | Module 6 |
| GoogleTest / CMocka | Functional regressions | Modules 7–8 |
| AddressSanitizer, Valgrind | Leaks, overflows, use-after-free | Level 2 |
| UndefinedBehaviorSanitizer | Signed overflow, bad shifts, null derefs | Level 2 |
| gcov/lcov | Which lines your tests never touched | Level 2 |
| clang-tidy, cppcheck | Bugs found without running the code | Level 2 |
| libFuzzer / AFL | Inputs no human would think to try | Level 3 |
| ThreadSanitizer | Data races | Level 3 |

You do not need any of these on day one. But you should understand *now* why
the automation half of this course is not optional: in C and C++, tooling is not
a convenience, it is the only way to observe an entire class of defects.

## Key terms

| Term | Meaning |
|---|---|
| **Error / mistake** | A human action that produces an incorrect result (a developer's typo) |
| **Defect / bug / fault** | The flaw in the code produced by that error |
| **Failure** | The observable wrong behaviour when that defect is executed |
| **Test case** | A documented set of inputs, steps, and expected results |
| **Test suite** | A collection of related test cases |
| **Regression** | Something that used to work and no longer does |
| **UB** | Undefined behaviour — code whose meaning the standard does not define |
| **Shift-left** | Moving test activity earlier in the life cycle |
| **SUT / UUT** | System Under Test / Unit Under Test |

A defect that is never executed never causes a failure — which is exactly why
untested code paths are where bugs live.

## Exercise

No code for this one — it's an analysis exercise, and the reasoning is the
point.

Take the following C function, which is supposed to return the average of an
array of integers:

```c
int average(int *values, int count) {
    int sum = 0;
    for (int i = 0; i <= count; i++) {
        sum += values[i];
    }
    return sum / count;
}
```

Write down, in a short document:

1. **Three distinct defects** in this function. For each, state whether it is a
   memory-safety defect, an undefined-behaviour defect, or a logic defect.
2. For each defect, describe **what a manual tester running the program by hand
   would most likely observe** — and be honest where the answer is "nothing
   unusual."
3. Which of the tools in the table in section 6 would reliably catch each one?
4. Finally: name one thing about this function that is a **requirements**
   problem rather than a code problem (hint: what *should* it return for an
   empty array?) — and note which SDLC stage that should have been caught in.

Keep your answers; you will turn them into formal test cases in Module 2 and a
bug report in Module 4.
