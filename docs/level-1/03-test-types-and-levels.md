# 03 · Test Types & Levels

"We tested it" is not a useful sentence. *Which* tests, at *which* level, of
*which* type? This module gives you the vocabulary to say precisely what was
tested and — more usefully — to notice what wasn't.

Two independent axes organize all testing:

- **Levels** — *how much of the system* is under test (unit → integration →
  system → acceptance)
- **Types** — *what property* you are checking (functional vs. non-functional,
  and within that: smoke, regression, performance, security, …)

They are orthogonal. You can have a functional unit test and a performance unit
test; a smoke system test and a regression system test.

## 1. The four test levels

| Level | Scope | Who usually writes it | Typical C/C++ tooling | Runtime |
|---|---|---|---|---|
| **Unit** | One function or class, isolated | Developer | GoogleTest, CMocka, Unity | Milliseconds |
| **Integration** | Two or more components together | Developer / tester | GoogleTest + real or fake collaborators | Seconds |
| **System** | The whole built product, end to end | Tester | Scripts, CLI drivers, hardware rigs | Minutes to hours |
| **Acceptance** | The product against user/customer needs | Customer, product owner, certification body | Scripted manual runs, formal protocols | Days |

### Unit testing

A **unit test** exercises the smallest independently testable piece of code —
in C, usually one function; in C++, one function or one class — with its
dependencies replaced or absent.

```cpp
// Unit test: only Calculator::divide is under test. Nothing else runs.
TEST(CalculatorTest, DivideTruncatesTowardZero) {
    Calculator c;
    EXPECT_EQ(c.divide(7, 2), 3);
}
```

Properties that define a unit test:

- **Fast** — a suite of thousands runs in seconds
- **Isolated** — no filesystem, no network, no hardware, no other components
- **Deterministic** — same result every run, in any order
- **Precise** — when it fails, you know exactly which function is broken

If a "unit test" opens a socket or reads a file, it is an integration test that
has been mislabelled — and it will eventually be slow and flaky.

### Integration testing

An **integration test** checks that components work *together* — that the
interfaces between them are compatible in reality, not just on paper. Most real
bugs in mature systems live at these seams: mismatched units (metres vs. feet),
mismatched error conventions (0-on-success vs. 0-on-failure), lifetime and
ownership mistakes, byte order, struct padding.

Two classic strategies:

| Strategy | How it works | Trade-off |
|---|---|---|
| **Big bang** | Assemble everything, test the whole thing | Cheap to set up; when it fails you have no idea where |
| **Incremental** (top-down, bottom-up, or sandwich) | Add one component at a time | More scaffolding (stubs/drivers) but failures are localized |

Incremental integration needs two kinds of scaffolding, and the words are worth
knowing precisely:

- A **stub** stands in for a component *below* the one under test (the thing it
  calls). Testing a display module top-down, you stub the sensor driver.
- A **driver** stands in for a component *above* (the thing that calls it).
  Testing a sensor driver bottom-up, you write a driver that calls into it.

In embedded C/C++ these are constant companions, because the "component below"
is frequently a physical peripheral that does not exist on your desk. Level 2
Module 7 covers test doubles for hardware in depth.

### System testing

**System testing** exercises the complete, integrated product in an environment
as close to production as you can manage: the real binary, real configuration,
real (or realistically simulated) hardware. Nothing is stubbed if it can be
avoided.

For a C/C++ command-line tool, a system test looks like running the actual
executable and checking its behaviour:

```bash
# System test: the real binary, real files, checking real observable output
./build/tempconv --input samples/valid.csv --output /tmp/out.csv
echo "exit code: $?"            # expect 0
diff /tmp/out.csv expected/valid_out.csv   # expect no differences
```

For embedded firmware, a system test means the real image flashed onto the real
board with the real sensors attached — which is why Level 3 has a whole module
on hardware-in-the-loop.

### Acceptance testing

**Acceptance testing** answers "will the customer accept this?" — validation
rather than verification (see Module 1). Varieties:

- **UAT (User Acceptance Testing)** — real users, real workflows
- **Contract acceptance** — the criteria written into the purchase contract
- **Regulatory acceptance** — a certification body reviews evidence against a
  standard (DO-178C, IEC 62304, ISO 26262)
- **Alpha / beta** — internal then external field testing

In safety-critical C/C++ work, this level generates the heaviest paperwork of
the project, and the traceability you built in Module 2 is what makes it
survivable.

### The test pyramid

The standard guidance on how many of each to have:

```
        /\          Acceptance   — few, slow, expensive, high confidence
       /  \         System       — some
      /    \        Integration  — more
     /______\       Unit         — many, fast, cheap, precise
```

Aim for a broad base of fast unit tests and a narrow tip of slow end-to-end
ones. The failure mode when this inverts — few unit tests, mostly slow system
tests — is called the *ice cream cone*: the suite takes hours, is flaky, and
when it fails nobody can tell which of 300 functions is at fault.

!!! note "Embedded projects deform the pyramid — deliberately"
    In embedded C/C++ you often cannot unit-test hardware-touching code on the
    target cheaply. The standard answer is **host-based unit testing**: compile
    the *logic* for your PC with fake hardware layers underneath, run thousands
    of fast unit tests there, and reserve the physical target for a much smaller
    number of integration and system tests. That keeps the pyramid shape while
    respecting the hardware. Level 3 Module 2 covers on-target vs. host testing.

## 2. Functional vs. non-functional testing

**Functional testing** asks: *does it do the right thing?* — inputs produce the
specified outputs, error paths behave as specified.

**Non-functional testing** asks: *how well does it do it?* — the qualities, not
the features. These are where C and C++ projects most often get chosen as the
implementation language in the first place, so they matter disproportionately
here.

| Non-functional type | Question | C/C++ tooling |
|---|---|---|
| **Performance** | How fast? What throughput? | Google Benchmark, `perf` (Level 3 M7) |
| **Load / stress** | What happens under sustained or excessive demand? | Custom drivers, soak scripts |
| **Memory / resource** | Does it leak? Does it fit in 64 KB of RAM? | Valgrind, ASan, map file analysis (Level 2 M4) |
| **Reliability / soak** | Does it survive 72 hours of continuous running? | Long-run harnesses; catches slow leaks |
| **Security** | Can input corrupt memory or escalate privilege? | Fuzzing, ASan, static analysis (Level 3 M4) |
| **Portability** | Does it behave the same on ARM as on x86-64? | Cross-compilation matrices (Level 4 M6) |
| **Maintainability** | Can the team safely change it? | Static analysis, complexity metrics, coverage |
| **Usability** | Can a person use it? | Manual, human judgement |

!!! warning "Non-functional failures are the ones that reach production"
    A functional bug usually shows up in the first hour of testing. A memory
    leak of 40 bytes per request shows up after nine days of uptime, in the
    field, on a customer's device. In C/C++ specifically, *resource* and
    *memory-safety* non-functional testing is not optional polish — it is the
    core of the discipline.

## 3. Testing by knowledge of internals

A third way of slicing the same activity — how much of the implementation the
tester can see.

| Approach | Tester sees | Derives tests from | Strength | Weakness |
|---|---|---|---|---|
| **Black-box** | Nothing but the interface | Requirements, the public API | Tests what users actually depend on; survives refactoring | Can miss internal paths entirely |
| **White-box** (glass-box) | The source code | Code structure, branches, paths | Can target every branch; measurable via coverage | Tests can "agree with" the bug they were written from |
| **Grey-box** | Some internals (architecture, data structures) | Both | Practical middle ground; the common real-world mode | Requires discipline not to over-couple |

Black-box techniques (boundary value analysis, equivalence partitioning) are
Module 5. White-box coverage measurement is Level 2 Module 6. The healthy
practice is to **design black-box first, then measure with white-box coverage
and fill the holes** — that way your tests express requirements, and coverage
merely tells you where you forgot to look.

## 4. Test types by purpose

These names describe *why* a particular run is happening. Same test cases,
different selections and different moments.

### Smoke testing

A tiny, fast subset that answers "is this build worth testing at all?" Named
after hardware bring-up: power it on, see if smoke comes out. Run it first,
every time; if it fails, reject the build and don't waste the day.

A C/C++ smoke suite is typically: the binary builds, it starts, `--version`
works, the three most critical functions return correct values on nominal input.
Target: **under 60 seconds.**

### Sanity testing

A narrow check after a small change or a bug fix — "did the specific thing that
was supposed to be fixed actually get fixed?" Narrow and deep, where smoke is
broad and shallow.

### Regression testing

Re-running existing tests to confirm that a change did not break something that
previously worked. This is the single largest justification for test automation:
regression is repetitive, unavoidable, and grows monotonically forever.

**Every bug you fix should gain a regression test.** The workflow:

1. A defect is reported.
2. Write a test that reproduces it — it **fails**. (If it passes, you haven't
   reproduced the bug and your fix would be a guess.)
3. Fix the code.
4. The test now **passes**.
5. It stays in the suite forever, so the bug cannot silently return.

### Re-testing (confirmation testing)

Distinct from regression, and often confused with it:

| | Re-test | Regression |
|---|---|---|
| Question | Is *this specific defect* fixed? | Did the fix break *anything else*? |
| Scope | The failing case only | A broad selection or the whole suite |
| Timing | Immediately after the fix | After the fix, usually automated in CI |

### Exploratory testing

Simultaneous learning, test design, and execution — unscripted, guided by
skill and curiosity rather than a document. This is where genuinely unexpected
bugs are found, and it is the strongest argument for keeping humans in the loop
in an automated world. Timebox it (a 90-minute "session"), state a charter
("explore the CSV import path looking for input the parser mishandles"), and
take notes as you go; anything interesting becomes a permanent test case.

### Ad-hoc testing

Unstructured, undocumented poking. Legitimate as a quick sniff test; dangerous
as a strategy, because it is unrepeatable and its coverage is unknown. If ad-hoc
testing finds something, convert it into a documented case immediately.

### Others worth knowing

| Type | Purpose |
|---|---|
| **Alpha / beta** | Field testing before general release |
| **Compatibility** | Works with other versions, compilers, libraries, hardware |
| **Recovery** | Behaves correctly after a crash, power loss, or watchdog reset — crucial in embedded |
| **Installation** | Deploys and upgrades cleanly (for firmware: does the OTA update path work?) |
| **Localization** | Correct across locales — in C, note that `printf("%f")` decimal separators change with the locale, a classic parsing bug |

## 5. Putting it together — a C/C++ example

Say you're testing a C++ library that parses sensor readings from a serial
stream and publishes averaged values.

| Level | What you'd test |
|---|---|
| Unit | `parse_frame()` handles a valid frame, a truncated frame, a bad checksum. `RollingAverage::add()` computes correctly, including on overflow. |
| Integration | `SerialReader` + `FrameParser` together: bytes arriving split across two reads still produce one correct frame. |
| System | Real binary, real (or emulated) serial port, 10,000 frames in: correct output file, exit code 0, no leaks under Valgrind. |
| Acceptance | The customer's field data file produces the values their existing system produces, within tolerance. |

| Type | What you'd run |
|---|---|
| Smoke | Build succeeds, binary starts, one known-good frame parses correctly. 30 seconds. |
| Functional | The full unit + integration suite against every requirement. |
| Regression | That same suite, on every commit, in CI. |
| Performance | Frames/second at `-O2` versus the documented budget. |
| Memory | 72-hour soak with a repeating stream; RSS must be flat. |
| Security | Fuzz `parse_frame()` with random bytes under ASan — the highest-value single test in this whole table for C/C++ code that parses untrusted input. |

## Exercise

You are handed a small C++ library, `libconfig`, which:

- reads a `key=value` text configuration file from disk,
- exposes `std::optional<std::string> get(const std::string& key)` and
  `int get_int(const std::string& key, int fallback)`,
- and is used by a long-running daemon that reloads the file on `SIGHUP`.

Produce a one-page **test strategy table** with these columns: *Level*, *Type*,
*What is tested*, *Automated or manual*, *Tooling*. Fill in at least:

1. Three **unit**-level entries (name the specific functions and the specific
   properties).
2. Two **integration**-level entries — and for each, name what would have to be
   stubbed or faked, and why.
3. Two **system**-level entries, one of which must exercise the `SIGHUP` reload
   path.
4. Three **non-functional** entries. One must address the fact that this is a
   *long-running* process, and one must address the fact that it parses a file
   a user can edit (think: what does `get_int` do with `"99999999999999"`?).
5. A **smoke** subset — list exactly which of your entries you would run first
   on a new build, and justify why the total stays under a minute.

Then write two or three sentences on where your table sits relative to the test
pyramid, and whether that's appropriate for this component.
