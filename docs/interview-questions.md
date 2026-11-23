---
title: "C/C++ Testing Interview Questions and Answers (180 Questions)"
description: "180 C/C++ Testing interview questions with answers and explanations, from beginner to advanced, organised by level. Free, from the C/C++ Testing Mastery Path."
---

# C/C++ Testing Interview Questions and Answers

**180 questions** with the correct answer and a short explanation, organised from beginner (Level 1) to advanced (Level 4). Every question comes from a lesson in the free C/C++ Testing Mastery Path, so you can open the lesson to go deeper.

!!! tip "Practise, don't just read"
    Each lesson also ends with an interactive quiz, and the [final quiz](quiz.md) tests all four levels. Try to answer each question below before reading the answer.

## Level 1 · Entry

### [01 · What Is Software Testing?](level-1/01-what-is-software-testing.md)

**Q1.** What is software testing, in one sentence?

**Answer:** Evaluating a program against expectations to find defects and provide information about quality

It's disciplined, planned and repeatable.

**Q2.** Why can't you test without expected results?

**Answer:** Without knowing the correct behavior you can't tell a defect from a feature; you've only observed output

An oracle is required.

**Q3.** What does 'testing shows the presence, not the absence, of defects' imply?

**Answer:** Passing tests can't prove a program is bug-free

Exhaustive testing is impossible.

**Q4.** Why is a test that lives only in your head not a test?

**Answer:** It isn't repeatable or reviewable by anyone else

Turn intent into artifacts.

### [02 · Test Case Design &amp; Documentation](level-1/02-test-case-design-documentation.md)

**Q5.** What is the difference between a test scenario and a test case?

**Answer:** A scenario is a high-level thing to test; a case has precise inputs, steps and expected results

Scenario, case and step differ in size and precision.

**Q6.** What does traceability link?

**Answer:** Tests back to the requirements they verify

It shows what's covered and what isn't.

**Q7.** What should a good test case's expected result be?

**Answer:** Specific and observable, so pass/fail is unambiguous

Avoid 'works correctly'.

**Q8.** What belongs in test data design?

**Answer:** Representative, boundary and invalid values, kept under control and documented

Data is part of the test.

### [03 · Test Types &amp; Levels](level-1/03-test-types-and-levels.md)

**Q9.** Which are the test levels from smallest to largest?

**Answer:** Unit, integration, system, acceptance

Levels describe how much of the system is under test.

**Q10.** Which is a non-functional test type?

**Answer:** Performance or security testing

Functional checks what; non-functional checks how well.

**Q11.** What does regression testing check?

**Answer:** That previously working behavior still works after a change

Automate them.

**Q12.** Why say precisely 'which tests at which level'?

**Answer:** 'We tested it' hides what wasn't tested

Notice gaps.

### [04 · Defect Lifecycle &amp; Bug Reporting](level-1/04-defect-lifecycle-bug-reporting.md)

**Q13.** What must a C/C++ bug report include beyond steps?

**Answer:** Build flags, sanitizer output, version and the exact input bytes

Developers must reproduce without asking.

**Q14.** Which is the best bug title?

**Answer:** 'Crash in parse_header() on 0-length input (ASan heap-buffer-overflow)'

Specific, searchable and precise.

**Q15.** What's the typical defect lifecycle order?

**Answer:** New, assigned/open, fixed, verified (retested), closed

Names vary by tool.

**Q16.** A developer says 'cannot reproduce'. What helps?

**Answer:** A minimal reproducer with exact environment and inputs

Reproducibility is everything.

### [05 · Manual Testing in Practice](level-1/05-manual-testing-in-practice.md)

**Q17.** When is exploratory testing valuable?

**Answer:** To find unexpected problems by learning and testing at once, guided by a charter

Time-boxed and noted.

**Q18.** Why note what you did during exploratory sessions?

**Answer:** So findings can be reproduced and coverage understood

Session notes.

**Q19.** Which task suits manual testing?

**Answer:** Usability and one-off investigation that isn't worth automating

Automate the repetitive.

**Q20.** What's a smoke test?

**Answer:** A quick check that the build's basic functions work before deeper testing

Fail fast.

### [06 · Toolchain Setup for Testers](level-1/06-toolchain-setup-for-testers.md)

**Q21.** Why enable -Wall -Wextra?

**Answer:** Warnings find bugs at compile time for free

The cheapest defect detector.

**Q22.** What does -g do?

**Answer:** Adds debug symbols so debuggers and sanitizers show source lines

Pair with -O0 for debugging.

**Q23.** What does -Werror do?

**Answer:** Treats warnings as errors so they can't be ignored

Use in CI.

**Q24.** Why record compiler and version in a bug report?

**Answer:** Behavior and diagnostics differ across compilers and versions

Reproducibility.

### [07 · Unit Testing with GoogleTest](level-1/07-unit-testing-googletest.md)

**Q25.** What does TEST(Suite, Name) do?

**Answer:** Registers a test case automatically with GoogleTest

No manual registration.

**Q26.** What's the difference between EXPECT_EQ and ASSERT_EQ?

**Answer:** EXPECT_* records a failure and continues; ASSERT_* stops the current test

Use ASSERT when continuing is meaningless.

**Q27.** Why add a descriptive failure message?

**Answer:** So a failure is understandable without opening the source

'false is not true' isn't enough.

**Q28.** What do fixtures share?

**Answer:** Setup and teardown for related tests via TEST_F

SetUp runs per test.

### [08 · Unit Testing C with Unity/CMocka](level-1/08-unit-testing-c-unity-cmocka.md)

**Q29.** When is Unity a good choice?

**Answer:** Tiny embedded targets: it is small with zero dependencies

CMocka has more features.

**Q30.** What does CMocka offer that Unity lacks out of the box?

**Answer:** Mocking support and leak detection

Choose by need.

**Q31.** How do you test C code with GoogleTest?

**Answer:** Wrap the C headers in extern "C" and compile tests as C++

Name mangling otherwise breaks linking.

**Q32.** Why do plain C tests need a runner?

**Answer:** C has no automatic test registration; tests are listed explicitly or via macros

Unity uses RUN_TEST.

### [09 · Test Builds &amp; Running Suites](level-1/09-test-builds-running-suites.md)

**Q33.** What does ctest do?

**Answer:** Runs the tests registered with CMake and reports each separately

Supports filtering with -R.

**Q34.** Why use the same tests in debug, release and sanitizer builds?

**Answer:** Different configurations expose different bugs

Configure with options.

**Q35.** What does `ctest --output-on-failure` do?

**Answer:** Shows test output only for failing tests

Keeps logs readable.

**Q36.** How should a growing suite be organized?

**Answer:** Separate test targets per module

Keeps navigation manageable.

### [10 · Project — Test Plan &amp; GoogleTest Suite](level-1/10-project-test-plan-and-suite.md)

**Q37.** Why pair a manual test plan with an automated suite?

**Answer:** The plan says what to verify and why; automation locks in what's cheap to rerun

Neither replaces the other.

**Q38.** What should a test plan include?

**Answer:** Scope, approach, risks, cases, environment and exit criteria

It guides testing.

**Q39.** What's the right candidate for automation?

**Answer:** Stable, repetitive, high-value checks

ROI matters.

**Q40.** How do you keep the plan useful?

**Answer:** Link cases to requirements and update as the product changes

Living documents.

## Level 2 · Intermediate

### [01 · GoogleTest Advanced](level-2/01-googletest-advanced.md)

**Q41.** What do parameterized tests (TEST_P) avoid?

**Answer:** Copy-pasting the same test logic for many inputs

INSTANTIATE_TEST_SUITE_P supplies values.

**Q42.** When do SetUpTestSuite/TearDownTestSuite run?

**Answer:** Once per suite rather than per test, e.g. for expensive shared setup

Beware shared state coupling tests.

**Q43.** What do typed tests verify?

**Answer:** One contract against several types

TYPED_TEST_SUITE.

**Q44.** How do you make failure messages useful?

**Answer:** Stream context with &lt;&lt; or use matchers with descriptive output

EXPECT_THAT with matchers.

### [02 · Mocking with GoogleMock](level-2/02-mocking-with-googlemock.md)

**Q45.** What does a mock let you assert?

**Answer:** The interaction itself: whether a call happened, with what args, in what order

EXPECT_CALL.

**Q46.** What makes a collaborator mockable in C++?

**Answer:** An interface (virtual functions) to substitute

Seam by abstraction.

**Q47.** Why do mocks make tests faster and more reliable?

**Answer:** They remove slow, non-deterministic dependencies like networks

Unit tests fail for one reason.

**Q48.** What is over-mocking?

**Answer:** Mocking so much that tests check implementation details instead of behavior

Brittle tests.

### [03 · Testing C Code with CMocka Advanced](level-2/03-cmocka-advanced.md)

**Q49.** How do you control a mocked C function's return?

**Answer:** will_return() queues values that mock() returns

A queue of return values.

**Q50.** What does expect_value / check_expected do?

**Answer:** Verifies the arguments passed to the mocked function

Argument checking.

**Q51.** Why use linker wrapping (--wrap) for mocking C?

**Answer:** It substitutes functions you don't own without changing source

Portability traps exist.

**Q52.** What does CMocka's test_malloc detect?

**Answer:** Leaks and heap corruption in code under test

Built-in memory checks.

### [04 · Memory Error Detection](level-2/04-memory-error-detection.md)

**Q53.** What does AddressSanitizer detect?

**Answer:** Heap/stack buffer overflows, use-after-free and leaks

Compile with -fsanitize=address.

**Q54.** Why can a program with a buffer overflow still pass tests?

**Answer:** The write may hit unused memory and the answer is right until the layout shifts

Instrument accesses.

**Q55.** How does valgrind differ from ASan?

**Answer:** It runs the unmodified binary under emulation: slower, no recompile

Both find memory errors.

**Q56.** What does LeakSanitizer report?

**Answer:** Heap allocations never freed at exit

Included with ASan on Linux.

### [05 · Undefined Behavior &amp; UBSan](level-2/05-undefined-behavior-ubsan.md)

**Q57.** Why can UB pass at -O0 and fail at -O2?

**Answer:** The optimizer may assume UB never happens and transform the code

UB isn't just 'garbage value'.

**Q58.** Which is undefined behavior?

**Answer:** Signed integer overflow

Unsigned overflow is defined.

**Q59.** What does UBSan do?

**Answer:** Instruments code to report UB like overflow and invalid shifts at runtime

-fsanitize=undefined.

**Q60.** Why run UBSan in tests?

**Answer:** UB bugs are invisible to ordinary assertions

Turn the invisible into diagnostics.

### [06 · Code Coverage (gcov/lcov)](level-2/06-code-coverage.md)

**Q61.** What does line coverage tell you?

**Answer:** Which lines executed during tests, not whether assertions were meaningful

It's narrow.

**Q62.** Why can 100% line coverage hide untested logic?

**Answer:** A line with multiple conditions can run without exercising every outcome

Use branch coverage and strong assertions.

**Q63.** Which flag enables gcov instrumentation?

**Answer:** --coverage

lcov builds reports.

**Q64.** Branch coverage measures...

**Answer:** Whether each decision's true and false outcomes were taken

Stronger than line coverage.

### [07 · Test Doubles for Hardware/OS](level-2/07-test-doubles-hardware-os.md)

**Q65.** What is a seam?

**Answer:** A place where behavior can be substituted for testing

Code lacking seams is untestable as written.

**Q66.** Which double returns canned answers?

**Answer:** A stub

Mocks verify interactions.

**Q67.** What's a fake?

**Answer:** A working but simplified implementation, like an in-memory filesystem

Useful for OS and hardware.

**Q68.** How do you test code reading a register at 0x40?

**Answer:** Abstract the register access behind an interface or function you can replace

Create a seam.

### [08 · CMake/CTest Deep Dive](level-2/08-cmake-ctest-deep-dive.md)

**Q69.** Why prefer targets with PUBLIC/PRIVATE properties over global variables?

**Answer:** Requirements propagate correctly through dependencies

Modern CMake.

**Q70.** What does add_test register?

**Answer:** A test for ctest to run, reported individually

Enables selection and reporting.

**Q71.** How do you run a sanitizer build alongside normal one?

**Answer:** Separate build directories or presets with different flags

Topology.

**Q72.** What does `ctest -R name` do?

**Answer:** Runs tests whose names match the regex

Selection.

### [09 · Static Analysis](level-2/09-static-analysis.md)

**Q73.** How does static analysis differ from dynamic testing?

**Answer:** It reasons about source paths without running them

Finds issues on unexecuted paths.

**Q74.** Why do analyzers get a bad reputation?

**Answer:** Poor configuration buries real findings under noise

Tune the rule set.

**Q75.** Which tool is a C/C++ static analyzer/linter?

**Answer:** clang-tidy

Also cppcheck.

**Q76.** How should findings be triaged?

**Answer:** Fix real bugs, suppress with justification and tune noisy checks

Keep signal high.

### [10 · Project — Instrumented Test Suite](level-2/10-project-instrumented-suite.md)

**Q77.** What does the instrumented suite combine?

**Answer:** Sanitizers, coverage and static analysis on one test suite

Layered defenses.

**Q78.** Why build each configuration separately?

**Answer:** Sanitizers and coverage change codegen and shouldn't mix

Clean topology.

**Q79.** What is the purpose of coverage reports in CI?

**Answer:** Spot untested code and watch trends, not as a goal by itself

Goodhart's law applies.

**Q80.** What should fail the CI job?

**Answer:** Failing tests, sanitizer reports and static-analysis errors

Automatic gates.

## Level 3 · Advanced

### [01 · Integration Testing Strategies](level-3/01-integration-testing-strategies.md)

**Q81.** What is a big-bang integration approach's weakness?

**Answer:** Failures are hard to localize because everything is integrated at once

Incremental is easier.

**Q82.** What does top-down integration use for missing lower modules?

**Answer:** Stubs

Bottom-up uses drivers.

**Q83.** What does integration testing verify?

**Answer:** Interfaces and interactions between components

Units are tested before.

**Q84.** What's a contract test?

**Answer:** Verifies both sides of an interface agree on its behavior

Reduces integration surprises.

### [02 · Embedded Target Testing](level-3/02-embedded-target-testing.md)

**Q85.** Why run some tests on the real target?

**Answer:** Compiler, word size and hardware behavior can differ from the host

Host tests aren't enough.

**Q86.** What is a common technique for on-target results?

**Answer:** Report results over UART/semihosting to the host

Collect pass/fail.

**Q87.** What limits embedded tests?

**Answer:** Small RAM/flash, so frameworks must be tiny

Unity fits.

**Q88.** What does a host-based test with a hardware abstraction layer give you?

**Answer:** Fast feedback on logic without the device

Then confirm on target.

### [03 · Hardware-in-the-Loop Basics](level-3/03-hardware-in-the-loop-basics.md)

**Q89.** What is HIL testing?

**Answer:** Running real firmware against simulated or real hardware inputs under automation

Real device, controlled environment.

**Q90.** What makes HIL tests slow and costly?

**Answer:** Physical devices, setup and limited parallelism

Use for what only hardware can show.

**Q91.** Which belongs in HIL rather than host tests?

**Answer:** Timing, interrupts and peripheral behavior

Hardware-dependent behavior.

**Q92.** What helps HIL reliability?

**Answer:** Automated reset/power control and known initial state

Determinism.

### [04 · Fuzz Testing](level-3/04-fuzz-testing.md)

**Q93.** What does fuzzing do?

**Answer:** Feeds many generated/mutated inputs to find crashes and sanitizer errors

Finds what you didn't think of.

**Q94.** Why combine fuzzing with sanitizers?

**Answer:** Silent memory errors become visible failures

ASan finds hidden overflows.

**Q95.** What is a corpus?

**Answer:** A set of interesting inputs that guide and persist fuzzing progress

Seeds.

**Q96.** What should you do with a fuzzer crash?

**Answer:** Minimize it into a regression test

Keep it as a test.

### [05 · Property-Based Testing](level-3/05-property-based-testing.md)

**Q97.** What is a property-based test?

**Answer:** It checks an invariant over many generated inputs instead of fixed examples

Example: reverse(reverse(x)) == x.

**Q98.** What does shrinking do?

**Answer:** Reduces a failing input to a minimal counterexample

Easier debugging.

**Q99.** Which is a good property for sort()?

**Answer:** Output is ordered and a permutation of the input

Describe behavior, not implementation.

**Q100.** When do examples still matter?

**Answer:** For documenting key cases and regressions

Combine both.

### [06 · CI/CD for C/C++ Test Suites](level-3/06-ci-cd-for-test-suites.md)

**Q101.** What should a CI pipeline for C/C++ tests include?

**Answer:** Build matrix, tests, sanitizers, coverage and static analysis

Fast feedback first.

**Q102.** Why cache dependencies and build outputs in CI?

**Answer:** To cut run time and keep feedback fast

ccache.

**Q103.** Why run sanitizer builds as separate jobs?

**Answer:** They need different flags and are slower

Parallel jobs.

**Q104.** What's a flaky test's effect on CI?

**Answer:** It erodes trust and teaches people to ignore failures

Quarantine and fix.

### [07 · Performance &amp; Benchmark Testing](level-3/07-performance-benchmark-testing.md)

**Q105.** Why benchmark in release builds?

**Answer:** Debug builds have different performance characteristics

Use -O2 and warm up.

**Q106.** What can the optimizer do to a benchmark?

**Answer:** Eliminate work whose result is unused, so use sinks or DoNotOptimize

Check what's measured.

**Q107.** Why run benchmarks multiple times?

**Answer:** Noise: report distributions rather than one number

Use medians and variance.

**Q108.** What's a performance regression test?

**Answer:** Compares results against a baseline and fails on significant slowdown

Allow for noise.

### [08 · Concurrency Testing](level-3/08-concurrency-testing.md)

**Q109.** Which tool finds data races?

**Answer:** ThreadSanitizer (-fsanitize=thread)

TSan instruments memory accesses.

**Q110.** Why are concurrency bugs hard to test?

**Answer:** They depend on timing and may appear rarely

Stress and tools help.

**Q111.** What helps reproduce a race?

**Answer:** Running many iterations, forcing interleavings and using TSan

Repetition.

**Q112.** Which is a deadlock cause?

**Answer:** Inconsistent lock ordering across threads

Define lock hierarchies.

### [09 · Legacy Code: Getting Untestable Code Under Test](level-3/09-legacy-code-under-test.md)

**Q113.** What is a characterization test?

**Answer:** A test that records current behavior before changing legacy code

It protects against unintended change.

**Q114.** What does the 'seam' idea allow in legacy code?

**Answer:** Substituting behavior at a point without editing the code there

Preprocessor, link or object seams.

**Q115.** What's the first step before refactoring untested code?

**Answer:** Get it under test with characterization tests

Safety net first.

**Q116.** Why make small, safe dependency-breaking changes?

**Answer:** To create seams with minimal risk

Then add tests.

### [10 · Project — CI Pipeline with Sanitizers, Coverage and Fuzzing](level-3/10-project-ci-pipeline.md)

**Q117.** What does the pipeline combine?

**Answer:** Sanitizers, coverage and fuzzing in CI

Layered checks.

**Q118.** Why run fuzzing for a short time in CI and longer offline?

**Answer:** CI needs fast feedback; long fuzzing finds deeper bugs

Both have roles.

**Q119.** What should happen to a new fuzz crash?

**Answer:** Add the input as a regression test

Locks the fix in.

**Q120.** What should fail the pipeline?

**Answer:** Test failures, sanitizer findings and new crashes

Gates.

## Level 4 · Master

### [01 · Test Architecture for Large Codebases](level-4/01-test-architecture-large-codebases.md)

**Q121.** Why organize tests by layers (unit, integration, system)?

**Answer:** So fast tests run often and slow tests run where needed

The pyramid.

**Q122.** What does test isolation require?

**Answer:** No shared mutable state or order dependence between tests

Tests must run in any order.

**Q123.** Why share fixtures and helpers in a common library?

**Answer:** To reduce duplication and keep tests consistent

Maintainability.

**Q124.** What makes large suites slow?

**Answer:** Too many integration tests and expensive setup; parallelize and prune

Measure test time.

### [02 · Safety-Critical Testing Standards](level-4/02-safety-critical-standards.md)

**Q125.** What do standards like DO-178C or ISO 26262 demand of testing?

**Answer:** Rigor and evidence scaled to the criticality level

Higher integrity needs more.

**Q126.** Why is requirement-to-test traceability central?

**Answer:** Auditors must see each requirement verified

Evidence.

**Q127.** What does MC/DC coverage check?

**Answer:** That each condition independently affects a decision's outcome

Required at highest levels.

**Q128.** Does passing tests alone satisfy a safety standard?

**Answer:** No: process, documentation and independence are also required

Standards cover the process.

### [03 · Requirements-Based Testing &amp; Traceability](level-4/03-requirements-based-testing.md)

**Q129.** What is requirements-based testing?

**Answer:** Deriving tests from requirements so each is verified

Black-box focus.

**Q130.** What does a traceability matrix show?

**Answer:** Which tests cover which requirements and gaps

Coverage of requirements.

**Q131.** What should a testable requirement be?

**Answer:** Clear, unambiguous and verifiable

Otherwise it can't be tested.

**Q132.** What happens when a requirement changes?

**Answer:** Update linked tests and trace

Maintain trace.

### [04 · Formal Methods &amp; Static Verification Basics](level-4/04-formal-methods-static-verification.md)

**Q133.** How does formal verification differ from testing?

**Answer:** It proves properties over all inputs rather than sampling some

Within a model.

**Q134.** What can bounded model checking do?

**Answer:** Explore all executions up to a bound to find violations

Counterexamples.

**Q135.** What is a limitation of formal methods?

**Answer:** They need precise specifications and effort, and verify only the model

Use where risk justifies.

**Q136.** What is an assertion's role in verification?

**Answer:** States a property tools can check

Specifications.

### [05 · Certification Evidence &amp; Test Reporting](level-4/05-certification-evidence-reporting.md)

**Q137.** What does certification evidence include?

**Answer:** Plans, traceability, test results, coverage reports and reviews

Auditors check artifacts.

**Q138.** Why must reports be reproducible?

**Answer:** A reviewer must be able to regenerate results from the same inputs

Version everything.

**Q139.** What makes a test report credible?

**Answer:** Exact versions, configuration, raw results and unexplained failures investigated

Transparency.

**Q140.** What is test independence?

**Answer:** Someone other than the author verifies critical work

Reduces blind spots.

### [06 · Cross-Compilation Test Strategies](level-4/06-cross-compilation-test-strategies.md)

**Q141.** What is cross-compilation?

**Answer:** Building on one architecture for another

Host to target.

**Q142.** How can you run target binaries on the host?

**Answer:** With an emulator such as QEMU user mode

Slower but convenient.

**Q143.** Why still test on real hardware?

**Answer:** Emulators can miss timing and peripheral behavior

Confirm key behavior.

**Q144.** What does a toolchain file do in CMake?

**Answer:** Defines compiler and target settings for cross builds

CMAKE_TOOLCHAIN_FILE.

### [07 · Simulation &amp; Emulation (QEMU-based Test Rigs)](level-4/07-simulation-emulation-qemu.md)

**Q145.** What does QEMU provide for testing?

**Answer:** Emulated CPUs and boards to run firmware without hardware

Fast and parallelizable.

**Q146.** What's a limitation of emulation?

**Answer:** It may not match real timing or all peripherals

Know the gaps.

**Q147.** How do you capture results from a QEMU test run?

**Answer:** Semihosting or serial output parsed by the host

Automate.

**Q148.** What can emulation parallelize?

**Answer:** Many instances in CI without physical boards

Scales testing.

### [08 · Quality Gates &amp; Metrics](level-4/08-quality-gates-metrics.md)

**Q149.** What is a quality gate?

**Answer:** An automated pass/fail criterion that blocks release when unmet

Examples: tests, coverage floor, no new critical findings.

**Q150.** Why not chase a coverage number as a goal?

**Answer:** People game it; coverage doesn't measure assertion quality

Use trends.

**Q151.** Which metric is useful with care?

**Answer:** Escaped defects and mean time to detect/repair

Outcome metrics.

**Q152.** How should gates be introduced?

**Answer:** Ratchet: don't allow regressions, then improve gradually

Avoid blocking everything.

### [09 · Building an In-House C/C++ Test Framework](level-4/09-in-house-test-framework.md)

**Q153.** When might you build an in-house test framework?

**Answer:** Unusual constraints (tiny targets, safety needs) that off-the-shelf tools don't meet

Weigh maintenance cost.

**Q154.** What are core components of a minimal framework?

**Answer:** Test registration, assertions, a runner and result reporting

Small is fine.

**Q155.** What's the main cost of in-house frameworks?

**Answer:** Ongoing maintenance and missing community features

Prefer standard tools when possible.

**Q156.** How should assertion macros report failures?

**Answer:** With file, line and expression

Debuggability.

### [10 · Capstone — Production-Grade Test Infrastructure](level-4/10-capstone-project.md)

**Q157.** What does the capstone integrate?

**Answer:** Unit, integration, sanitizers, coverage, fuzzing, CI and reporting

Production-grade infrastructure.

**Q158.** What makes infrastructure production-grade?

**Answer:** Reproducible, automated, fast and maintainable with clear gates

Sustainable.

**Q159.** How do you prove the infrastructure works?

**Answer:** Inject known defects and confirm the gates catch them

Test the tests.

**Q160.** What should be documented for the team?

**Answer:** How to run, extend and interpret the suite

Onboarding.

## Mixed questions across all levels

**Q161.** A manager asks: 'Can you guarantee the release has no bugs if all tests pass?' What's the accurate answer?

**Answer:** No: testing shows the presence of defects, not their absence

Exhaustive testing is impossible.

**Q162.** A function takes two 32-bit ints. Why can't you test every combination?

**Answer:** About 1.8x10^19 combinations, roughly 585,000 years at a million tests per second

Testing is a selection problem.

**Q163.** A valid age range is 18 to 65. Which inputs best expose off-by-one bugs?

**Answer:** 17, 18, 19, 64, 65 and 66 (boundary values)

Boundary value analysis.

**Q164.** Test inputs 20, 30 and 40 are all valid ages. What does equivalence partitioning say?

**Answer:** They're in the same partition, so one representative is usually enough

Pick one per partition plus boundaries.

**Q165.** A defect report says only 'program crashes'. What's missing?

**Answer:** Steps, exact input, environment/build flags and sanitizer output to reproduce it

A bug report must let a developer reproduce it alone.

**Q166.** A line shows 100% line coverage yet an `if (a && b)` bug slipped through. Why?

**Answer:** Executing a line doesn't mean every condition outcome was tested or asserted

Use branch/MC/DC coverage and strong assertions.

**Q167.** A test passes at -O0 but the product misbehaves at -O2. What should you suspect?

**Answer:** Undefined behavior, which the optimizer may exploit; run UBSan

UB isn't just a wrong value.

**Q168.** Tests pass but production corrupts memory occasionally. Which tool finds it?

**Answer:** AddressSanitizer (or valgrind) on the test run

Memory errors can be invisible to assertions.

**Q169.** Why mock a payment gateway in a unit test?

**Answer:** Real networks are slow and non-deterministic; a mock isolates the unit and lets you assert on interactions

A unit test should fail for one reason.

**Q170.** Code reads a register at fixed address 0x40 and can't be unit tested. What's the fix?

**Answer:** Introduce a seam: put register access behind an interface you can fake or stub

Untestable as written means it lacks a seam.

**Q171.** Which pairing is right: stub, mock, fake?

**Answer:** Stub returns canned answers, mock verifies interactions, fake is a simplified working implementation

Choose the right double.

**Q172.** A parser crashes on one malformed packet found by a fuzzer. What's the right follow-up?

**Answer:** Minimize the input and add it as a regression test with the fix

Fuzz findings become permanent tests.

**Q173.** Which property fits property-based testing of sort()?

**Answer:** Output is ordered and a permutation of the input

Check invariants over many generated inputs.

**Q174.** A race appears once in 1000 runs. Which tool helps?

**Answer:** ThreadSanitizer plus repeated stress runs

Timing bugs need instrumentation.

**Q175.** You must change untested legacy code. What first?

**Answer:** Write characterization tests capturing current behavior, creating seams as needed

Get it under test first.

**Q176.** Which is the best pipeline order for C/C++ test CI?

**Answer:** Fast checks first (build, unit tests), then sanitizers, coverage, static analysis and longer fuzzing

Fast feedback with layered depth.

**Q177.** A safety standard requires MC/DC coverage. What does that mean?

**Answer:** Each condition must be shown to independently affect the decision outcome

A rigorous structural criterion.

**Q178.** What does a requirements traceability matrix reveal?

**Answer:** Which requirements have tests and which don't

Gaps in verification.

**Q179.** A team sets a 90% coverage gate and tests start asserting nothing. What's the lesson?

**Answer:** Metrics become targets and get gamed; pair coverage with assertion quality (e.g. mutation testing)

Goodhart's law.

**Q180.** When should you write an in-house test framework?

**Answer:** When constraints (tiny targets, certification) rule out existing ones, accepting the maintenance cost

Prefer off-the-shelf when possible.

## Practical coding questions

### Boundary value analysis

Write boundary_values(lo, hi) returning the inputs to test for a valid range [lo, hi]: just below, on and just above each boundary, plus a nominal middle value. Then run them through valid_age (18 to 65).

Starter code:

```
def boundary_values(lo, hi):
    # return [lo-1, lo, lo+1, a nominal middle value, hi-1, hi, hi+1]
    return []

def valid_age(n):
    return 18 <= n <= 65

cases = boundary_values(18, 65)
print(cases)
print([valid_age(n) for n in cases])
```

Expected output:

```text
[17, 18, 19, 41, 64, 65, 66]
[False, True, True, True, True, True, False]
```

??? success "Show a solution"

    ```
    def boundary_values(lo, hi):
        """Boundary value analysis for a valid range [lo, hi]:
        just below, on and just above each boundary, plus a nominal value."""
        mid = (lo + hi) // 2
        return [lo - 1, lo, lo + 1, mid, hi - 1, hi, hi + 1]

    def valid_age(n):
        return 18 <= n <= 65

    cases = boundary_values(18, 65)
    print(cases)
    print([valid_age(n) for n in cases])
    ```

    Bugs cluster at boundaries (off-by-one: &lt; versus &lt;=). Testing both sides of each edge detects them with just seven cases instead of the whole range. Verified by running the solution.

### Mutation score

Implement mutation_report(ages): check that the suite passes on the original is_adult, then report which mutants it kills (the suite fails on them) and which survive. Compare a weak suite that only tests age 40 with a strong suite that tests 17, 18 and 40.

Starter code:

```
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

Expected output:

```text
weak suite    score 1/4  survived: ['boundary >= to >', 'threshold 18 to 21', 'always true']
strong suite  score 4/4  survived: []
```

??? success "Show a solution"

    ```
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
        assert run_suite(is_adult, ages), "suite must pass on the original code"
        killed = [name for name, fn in MUTANTS.items() if not run_suite(fn, ages)]
        survived = [name for name in MUTANTS if name not in killed]
        return killed, survived

    for label, ages in [("weak suite   ", [40]), ("strong suite ", [17, 18, 40])]:
        killed, survived = mutation_report(ages)
        print("%s score %d/%d  survived: %s" % (label, len(killed), len(MUTANTS), survived))
    ```

    The weak suite kills only the 'negated' mutant yet still passes on the original, much like a high-coverage suite with weak assertions. Boundary tests (17 and 18) kill all four. Verified by running the solution.

---

More free practice: the [C/C++ Testing Mastery Path](index.md) curriculum, the [Interview Prep Pack](https://learnmasterypath.com/interview-pack/) and the [free resume ATS check](https://learnmasterypath.com/resume/).
