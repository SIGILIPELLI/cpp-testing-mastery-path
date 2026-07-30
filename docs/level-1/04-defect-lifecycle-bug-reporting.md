# 04 · Defect Lifecycle & Bug Reporting

Finding a bug is half the job. The other half is communicating it so precisely
that a developer can reproduce and fix it without ever talking to you. A bug
report is a technical document with a job to do, and in C/C++ it has to carry
information — build flags, sanitizer output, exact input bytes — that reports in
other languages can skip.

## 1. The defect lifecycle

Every defect moves through a state machine. Names vary by tool (Jira, Bugzilla,
GitHub Issues, Polarion), but the shape is standard:

```
    New ──► Assigned ──► In Progress ──► Fixed ──► Retest ──► Closed
     │          │                                     │
     │          │                                     └──► Reopened ──► Assigned
     │          │
     ├──► Duplicate (closed)
     ├──► Rejected / Not a Bug (closed)
     ├──► Deferred (backlog)
     └──► Cannot Reproduce ──► (needs more info from reporter)
```

| State | Meaning | Who owns it |
|---|---|---|
| **New** | Just filed, not yet triaged | Reporter (until triage) |
| **Assigned** | Triaged, accepted, given an owner | Developer |
| **In Progress** | Being worked on | Developer |
| **Fixed** | Developer believes it's resolved; a change is committed | Tester (awaiting verification) |
| **Retest** | Available in a build for the tester to verify | Tester |
| **Closed** | Verified fixed in a specific build | — |
| **Reopened** | Verification failed; back to the developer | Developer |
| **Duplicate** | Same underlying defect as an existing report | — |
| **Rejected / Not a Bug** | Behaviour is per specification | — |
| **Deferred** | Real, accepted, not fixing now | Product owner |
| **Cannot Reproduce** | Developer couldn't make it happen | Reporter |

Two rules that prevent most process pain:

- **Only the reporter (or a tester) closes a defect.** A developer marks it
  *Fixed*; verification is a separate act by a different person. Self-closing
  destroys the audit trail and lets unverified fixes ship.
- **Always record the build.** "Fixed" is meaningless without "fixed in build
  1.4.2-rc3" — otherwise nobody knows which builds still contain it.

!!! warning "'Cannot Reproduce' is usually a report-quality defect"
    In C/C++ it is *also* frequently a genuine environment difference: the bug
    reproduces at `-O2` but not `-O0`, on GCC but not Clang, on ARM but not
    x86-64, or only on the fifth run because it's a race. Before you accept a
    "cannot reproduce", check that the developer used your exact build
    configuration. Section 5 covers this.

## 2. Severity vs. priority

The most commonly confused pair in the whole discipline. They are **independent
axes** set by **different people** for **different reasons**.

| | Severity | Priority |
|---|---|---|
| Measures | Technical impact on the system | Urgency of fixing it |
| Set by | The tester (objective, from observed behaviour) | Product owner / manager (business decision) |
| Changes when | Rarely — impact is what it is | Often — as release dates and customers shift |
| Question | "How badly does it break?" | "How soon must we fix it?" |

### Severity scale

| Level | Name | Definition | C/C++ examples |
|---|---|---|---|
| S1 | **Critical / Blocker** | System unusable; no workaround; data loss or corruption | Segfault on startup; heap corruption; a buffer overflow reachable from untrusted input |
| S2 | **Major** | Major function broken; awkward workaround exists | A parser rejects all valid input over 100 chars; a memory leak that exhausts RAM in 6 hours |
| S3 | **Minor** | Function works but incorrectly in a limited case | Off-by-one in a displayed count; error message names the wrong field |
| S4 | **Trivial / Cosmetic** | No functional impact | Typo in a log message; misaligned column in `--help` |

### Priority scale

| Level | Name | Meaning |
|---|---|---|
| P1 | Urgent | Fix now; blocks the release or other people's work |
| P2 | High | Fix in this release |
| P3 | Medium | Fix when convenient |
| P4 | Low | Fix if there's ever spare time |

### All four combinations occur

| Combination | Realistic example |
|---|---|
| **High severity, high priority** | Firmware segfaults on boot with a common sensor attached. Obvious P1S1. |
| **High severity, low priority** | The program crashes if you pass `--legacy-mode`, a flag deprecated three years ago that one internal script uses. Severe (crash) but nobody urgent is affected. |
| **Low severity, high priority** | The company name is misspelled on the CLI splash screen shown in tomorrow's investor demo. Cosmetic, but fix it today. |
| **Low severity, low priority** | Trailing whitespace in a debug log line. |

Your job as a tester is to assign **severity accurately and defend it**, and to
*propose* a priority while accepting that someone else decides it. Inflating
severity to get attention is a short-term tactic that destroys the credibility
of your entire defect database.

## 3. The bug report template

| Field | Content |
|---|---|
| **ID** | Assigned by the tracker |
| **Title** | One line: *what* is wrong, *where*, and *when*. See section 4. |
| **Severity** | S1–S4, from the definitions above |
| **Priority (proposed)** | P1–P4 |
| **Reported by / Date** | You / today |
| **Build / Version** | Exact version, commit SHA, or build number |
| **Environment** | OS, architecture, compiler + version, build flags, target hardware |
| **Component** | The module/file, if known |
| **Preconditions** | What state must exist first |
| **Steps to reproduce** | Numbered, minimal, complete |
| **Expected result** | What should happen, with the requirement ID if there is one |
| **Actual result** | What did happen, verbatim |
| **Reproducibility** | Always / Intermittent (n of m runs) / Once |
| **Attachments** | Input files, core dump, sanitizer output, stack trace, log excerpt, screenshot |
| **Regression?** | Did this work in an earlier build? Which one? |
| **Workaround** | If one exists |

## 4. Writing a title that does its job

The title is read by dozens of people who will never open the report. It should
be searchable and self-explanatory.

| Poor title | Problem | Better |
|---|---|---|
| "Crash" | Says nothing | "Segfault in `parse_frame()` when input frame is truncated at byte 3" |
| "Doesn't work" | Says nothing | "`get_int()` returns 0 instead of the fallback for non-numeric values" |
| "Memory issue" | Which kind? Where? | "Heap-use-after-free in `SensorBuffer::flush()` reported by ASan on repeated flush" |
| "Bug in config parsing" | Too broad | "Config keys with trailing spaces are silently ignored" |
| "URGENT!!! FIX NOW" | Priority belongs in the priority field | "Firmware fails to boot when EEPROM checksum is 0xFFFF" |

The formula: **`<symptom>` in `<location>` when `<condition>`.**

## 5. Reporting C/C++ crashes and undefined behaviour

This is where C/C++ bug reporting genuinely differs from other languages. A
crash report without the following is usually unactionable.

### Always include the exact build configuration

```
Compiler:  gcc 13.2.0 (Ubuntu 13.2.0-23ubuntu4)
Flags:     -std=c++17 -O2 -Wall -Wextra -g
Arch:      x86-64
OS:        Ubuntu 24.04.1 LTS, kernel 6.8.0-45
Libc:      glibc 2.39
Build:     cmake -DCMAKE_BUILD_TYPE=Release ..
```

Optimization level is not a detail. Undefined behaviour frequently manifests at
`-O2` and vanishes at `-O0`, because the optimizer is permitted to assume UB
never happens (see Module 1). "Works in Debug, crashes in Release" is a *symptom
worth reporting in the title*, not an inconvenience to hide.

### Include the signal and the stack trace

```
$ ./tempconv --input malformed.csv
Segmentation fault (core dumped)
$ echo $?
139        # 128 + 11 (SIGSEGV)
```

Get a backtrace with gdb:

```bash
# Enable core dumps for this shell, then reproduce
ulimit -c unlimited
./tempconv --input malformed.csv

gdb ./tempconv core
(gdb) bt
#0  0x0000555555555 in parse_row (row=0x0, out=0x7fff...) at parser.c:88
#1  0x0000555555777 in read_file (path=0x7fff... "malformed.csv") at reader.c:41
#2  0x0000555555999 in main (argc=3, argv=0x7fff...) at main.c:22
```

Or run it under gdb directly:

```bash
gdb --args ./tempconv --input malformed.csv
(gdb) run
(gdb) bt full        # backtrace with local variables -- attach the whole thing
```

Attach the **full** backtrace, not just the top frame. Frame 0 is where it died;
frames 1 and 2 are usually where the actual bug is.

### Include sanitizer output verbatim

If you can rebuild with sanitizers (Level 2 covers them properly), a two-minute
rebuild converts a vague "it crashes sometimes" into a precise diagnosis:

```bash
g++ -std=c++17 -g -fsanitize=address -fno-omit-frame-pointer -o tempconv_asan *.cpp
./tempconv_asan --input malformed.csv
```

```
=================================================================
==48211==ERROR: AddressSanitizer: heap-buffer-overflow on address 0x60200000ef35
READ of size 1 at 0x60200000ef35 thread T0
    #0 0x4f8a12 in parse_row(char const*, Row*) parser.cpp:88:17
    #1 0x4f9c04 in read_file(char const*) reader.cpp:41:9
    #2 0x4fa118 in main main.cpp:22:5

0x60200000ef35 is located 0 bytes to the right of 5-byte region
allocated by thread T0 here:
    #0 0x4c1b3d in operator new(unsigned long)
    #1 0x4f8901 in parse_row(char const*, Row*) parser.cpp:74:20
```

Paste this **whole block** into the report. It names the file, line, the exact
kind of violation, and the allocation site. A report containing this costs the
developer minutes; the same bug reported as "crashes on some CSV files" costs
days.

### Reduce the input to a minimal reproducer

A 4 MB CSV file that triggers the bug is a starting point, not a report. Bisect
it: halve the file, re-run, keep the half that still fails, repeat. You will
usually land on a handful of bytes.

```
# minimal_repro.csv -- 11 bytes, still segfaults
id,temp
1,
```

Then state it exactly, including invisible characters:

> Reproduces with any CSV where a row has a header but an empty final field and
> **no trailing newline**. Hex dump of the minimal file:
> `69 64 2c 74 65 6d 70 0a 31 2c` (note: no `0a` at end of file).

A hex dump beats a description whenever whitespace, line endings, or encoding
could matter — which for a C parser is nearly always.

### Report intermittent failures honestly

For a race or a UB-dependent bug, reproducibility is data:

> **Reproducibility:** Intermittent — 3 of 50 runs at `-O2`; 0 of 50 runs at
> `-O0`; 47 of 50 runs under `-fsanitize=thread`. Reproduction script attached
> (`loop_repro.sh` runs the binary 50 times and counts non-zero exits).

Never write "sometimes crashes" when you can write "3 of 50 runs". Attach the
loop script so the developer can measure the same way — otherwise "I ran it
twice and it was fine" closes your report as Cannot Reproduce.

## 6. A complete worked bug report

> **ID:** BUG-1174
> **Title:** Heap-buffer-overflow read in `parse_row()` when final CSV field is empty and file has no trailing newline
> **Severity:** S1 — Critical (out-of-bounds read; crashes at `-O2`; input is user-supplied, so potentially a security issue)
> **Priority (proposed):** P1
> **Reported by:** A. Tester, 2026-07-30
> **Build:** `tempconv` v1.4.2-rc3, commit `a91f3d7`
> **Environment:** Ubuntu 24.04.1, x86-64, gcc 13.2.0, `-std=c++17 -O2 -g`, glibc 2.39. Also reproduced with clang 18 at `-O2`.
> **Component:** `parser.cpp`, `parse_row()`
>
> **Preconditions:** None. Fresh build, no configuration.
>
> **Steps to reproduce:**
> 1. Create `minimal_repro.csv` containing exactly the bytes `69 64 2c 74 65 6d 70 0a 31 2c` (`id,temp\n1,` — no trailing newline). Attached.
> 2. Run `./tempconv --input minimal_repro.csv --output /tmp/out.csv`
>
> **Expected result:** Per REQ-CSV-007, a row with a missing final field is reported as a malformed row: the tool prints `error: malformed row 1` to stderr and exits with code 2. No crash.
>
> **Actual result:** Process terminates with SIGSEGV (exit code 139). No output file is created. Under `-fsanitize=address` the run aborts with a heap-buffer-overflow READ of size 1, 0 bytes to the right of a 5-byte allocation made at `parser.cpp:74`, read at `parser.cpp:88`. Full ASan report attached (`asan.log`); gdb backtrace attached (`bt.txt`).
>
> **Reproducibility:** Always at `-O2` (20/20 runs). At `-O0`, the process does *not* crash and silently writes a row containing garbage — arguably worse, since it produces wrong output with no error. Both behaviours trace to the same out-of-bounds read.
>
> **Regression?** Yes. v1.3.8 correctly printed `error: malformed row 1`. Bisect points to commit `c40b12e` ("optimize row splitting").
>
> **Workaround:** Ensure input files end with a newline.
>
> **Attachments:** `minimal_repro.csv`, `asan.log`, `bt.txt`, `hexdump.txt`

Notice how much of that report's value comes from things the tester *did* rather
than *observed*: minimized the input, rebuilt with ASan, tried a second
compiler, tried a second optimization level, bisected to a commit, and checked
an older release. That work is the difference between a report that gets fixed
this week and one that sits in triage.

## 7. Defect metrics worth tracking

| Metric | Formula | What it tells you |
|---|---|---|
| **Defect density** | Defects ÷ KLOC | Which modules are the problem children (defect clustering, Module 1) |
| **Defect Removal Efficiency (DRE)** | Defects found before release ÷ (found before + found after) × 100 | How effective your process is. 95% is good; below 80% means production is your test environment. |
| **Defect age** | Time from New to Closed | Whether fixes are actually happening |
| **Reopen rate** | Reopened ÷ Fixed | High rate means fixes aren't being verified properly, or reports are ambiguous |
| **Cannot-Reproduce rate** | CNR ÷ total filed | High rate usually means report quality — or environment-dependent UB |

## 8. Common bug-reporting mistakes

| Mistake | Effect |
|---|---|
| Multiple bugs in one report | Half gets fixed, the report gets closed, the rest survives |
| No build/environment info | Cannot Reproduce, especially in C/C++ |
| "Expected: it should work" | Nobody knows what correct looks like; developer guesses |
| Non-minimal reproducer | Developer spends the first day doing your minimization |
| Assigning the root cause in the title ("null pointer in malloc") | Anchors the developer on a wrong theory; report *symptoms*, hypothesize separately |
| Emotional language | Reduces your credibility and the report's shelf life |
| Filing without searching for duplicates | Noise; splits the discussion across two threads |

## Exercise

Return to the `average()` function from Module 1's exercise:

```c
int average(int *values, int count) {
    int sum = 0;
    for (int i = 0; i <= count; i++) {
        sum += values[i];
    }
    return sum / count;
}
```

1. **File three separate bug reports** — one per defect you identified — using
   the full template from section 3. Do not combine them. For each:
   - assign and *justify* a severity from the scale in section 2,
   - propose a priority and say who would really decide it,
   - write a title using the `<symptom> in <location> when <condition>` formula,
   - give minimal, numbered reproduction steps including the exact array
     contents and `count` value you would pass.

2. For the out-of-bounds read specifically, write the **Actual result** field
   twice: once describing what a plain `-O0` run would most likely show, and
   once describing what an ASan-instrumented run would show. Note in one
   sentence which version a developer can act on faster, and why.

3. Construct a **severity/priority grid** (a 2×2 table: high/low severity ×
   high/low priority) and place each of your three defects in it, assuming this
   function is used in (a) an internal reporting script, then (b) an automotive
   ECU. Note which placements move between the two contexts and why — this is
   testing principle 6, "testing is context dependent", made concrete.
