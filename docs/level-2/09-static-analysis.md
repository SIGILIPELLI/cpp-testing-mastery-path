# 09 · Static Analysis

Every technique so far has been dynamic: it finds bugs on code paths your tests
actually execute. Static analysis reads the source instead and reasons about
paths nobody ran. That makes it the cheapest defect-finding tool you own — no
test case required — and also the one with the worst reputation, because a badly
configured analyser buries three real findings under four hundred style
complaints.

## 1. Static vs dynamic, precisely

| | Static analysis | Dynamic (sanitizers, tests) |
|---|---|---|
| Runs the code | No | Yes |
| Needs a test case | No | Yes |
| Covers unexecuted paths | Yes | No |
| False positives | Common | Essentially none |
| False negatives | Common | Only for unexecuted paths |
| Finds leaks on error paths | Often | Only if a test hits the path |
| Finds data-dependent overruns | Rarely | Reliably |
| Cost per run | Seconds to minutes | Build + test time |

They are complements, not alternatives, and section 5 demonstrates that with a
bug that only one of them sees.

## 2. Tier zero: the compiler you already have

Before installing anything, turn on the analyser bundled with your compiler.

```bash
clang -Wall -Wextra -Wpedantic -Wshadow -Wconversion \
      -Wcast-qual -Wold-style-cast -Wnull-dereference \
      -Wdouble-promotion -Wformat=2 -Werror -c src/*.c
```

Worth singling out:

- `-Wconversion` — silent narrowing, the source of a large fraction of embedded
  bugs.
- `-Wshadow` — an inner variable hiding an outer one.
- `-Wformat=2` — `printf` format/argument mismatches, which are UB.
- `-Werror` — the one that makes the rest matter. Warnings that don't fail the
  build get ignored within a week.

GCC adds `-fanalyzer`, a real interprocedural path-sensitive checker for C:

```bash
gcc-16 -fanalyzer -Wall -c src/parser.c
```

!!! tip "Adopt `-Werror` incrementally"
    Turning it on across a legacy codebase produces thousands of errors and gets
    reverted. Enable `-Werror` for **new** targets and for the specific warnings
    you have already cleaned; expand the list one warning at a time.

## 3. cppcheck

cppcheck does path-sensitive analysis without needing your build to succeed,
which makes it easy to adopt.

```bash
brew install cppcheck            # or: sudo apt install cppcheck
```

Here is real output against deliberately broken code:

```c
#include <stdlib.h>
#include <string.h>

char *duplicate_prefix(const char *src, size_t n) {
    char *out = malloc(n);
    strncpy(out, src, n);
    out[n] = '\0';
    return out;
}

int sum_upto(int n) {
    int total;                       /* never initialised */
    for (int i = 0; i <= n; i++) total += i;
    return total;
}

void leaky(void) {
    char *buf = malloc(32);
    if (buf == NULL) return;
    strcpy(buf, "hello");            /* never freed */
}
```

```bash
cppcheck --enable=all --inconclusive --std=c11 \
         --suppress=missingIncludeSystem buggy.c
```

```
Checking buggy.c ...
buggy.c:23:1: error: Memory leak: buf [memleak]
buggy.c:6:13: warning: If memory allocation fails, then there is a possible null pointer dereference: out [nullPointerOutOfMemory]
buggy.c:5:23: note: Assuming allocation function fails
buggy.c:5:23: note: Assignment 'out=malloc(n)', assigned value is 0
buggy.c:6:13: note: Null pointer dereference
buggy.c:16:12: warning: Uninitialized variable: total [uninitvar]
buggy.c:13:23: note: Assuming condition is false
buggy.c:16:12: note: Uninitialized variable: total
```

Three real defects, with the reasoning chain shown — `Assuming allocation
function fails` tells you exactly which hypothetical path it explored. No test
case was written, and no test would have found the leak without deliberately
exercising the early return.

For projects with a compilation database, feed it the real flags:

```bash
cmake -S . -B build -DCMAKE_EXPORT_COMPILE_COMMANDS=ON
cppcheck --project=build/compile_commands.json \
         --enable=warning,style,performance,portability \
         --inline-suppr --error-exitcode=1 -j8
```

`--error-exitcode=1` is what makes it a gate rather than a report. `--enable=all`
includes `information` and `unusedFunction`, both noisy on libraries — prefer the
explicit list above for CI.

## 4. clang-tidy

clang-tidy uses Clang's own AST, so it understands C++ far better than a
standalone parser can, and many checks auto-fix.

```yaml
# .clang-tidy
Checks: >
  -*,
  bugprone-*,
  cert-*,
  clang-analyzer-*,
  cppcoreguidelines-*,
  performance-*,
  readability-*,
  -readability-magic-numbers,
  -cppcoreguidelines-avoid-magic-numbers,
  -readability-identifier-length
WarningsAsErrors: 'bugprone-*,clang-analyzer-*'
HeaderFilterRegex: '^(src|include)/'
```

```bash
cmake -S . -B build -DCMAKE_EXPORT_COMPILE_COMMANDS=ON
clang-tidy -p build src/parser.cpp
run-clang-tidy -p build -j8            # whole project
clang-tidy -p build --fix src/parser.cpp
```

Typical output:

```
src/parser.cpp:42:9: warning: 'auto ptr' can be declared as 'const auto *ptr' [readability-qualified-auto]
src/parser.cpp:57:5: warning: use range-based for loop instead [modernize-loop-convert]
src/parser.cpp:88:13: warning: Potential leak of memory pointed to by 'buf' [clang-analyzer-unix.Malloc]
```

!!! info "Not verified on this platform"
    Unlike the cppcheck and compiler output above, the clang-tidy examples were
    not run on the machine these notes were written on — Apple's Command Line
    Tools do not ship `clang-tidy`. Install it via `brew install llvm` (then use
    `/opt/homebrew/opt/llvm/bin/clang-tidy`) or `apt install clang-tidy`, and
    expect the exact wording to vary by version.

Two configuration notes that decide whether the tool survives contact with a
team: start from `-*` and add check groups explicitly — the default set is
enormous. And set `HeaderFilterRegex`, or you will get warnings from every
system header your translation unit pulls in.

## 5. What static analysis misses

Look again at `duplicate_prefix`. cppcheck flagged the possible null dereference
and the leak in `leaky()`. It did **not** flag this:

```c
char *out = malloc(n);
strncpy(out, src, n);
out[n] = '\0';          /* writes at index n in an n-byte buffer */
```

That is a straightforward one-byte heap overflow, and the analyser walked right
past it. AddressSanitizer, in contrast, catches it the first time the line runs:

```
==6214==ERROR: AddressSanitizer: heap-buffer-overflow on address 0x6020000000d5
WRITE of size 1 at 0x6020000000d5 thread T0
    #0 0x0001002c4870 in duplicate_prefix obug.c:7
0x6020000000d5 is located 0 bytes after 5-byte region [0x6020000000d0,0x6020000000d5)
```

The lesson is not that cppcheck is bad. It is that the two tools have
**disjoint blind spots**, and a pipeline with only one of them has a hole in it:

- Static analysis found the leak on an error path no test exercised.
- ASan found the overflow that static analysis could not prove.

Run both.

## 6. Suppressions and baselines

A finding you have judged to be a false positive should be suppressed **at the
line, with a reason** — never globally.

```c
/* cppcheck-suppress nullPointerOutOfMemory ; allocator is checked in caller */
strncpy(out, src, n);
```

```cpp
// NOLINTNEXTLINE(cppcoreguidelines-pro-type-reinterpret-cast) -- ABI boundary
auto* hdr = reinterpret_cast<Header*>(buffer);
```

```bash
cppcheck --inline-suppr --project=build/compile_commands.json
```

For an existing codebase with hundreds of findings, generate a **baseline** of
current results, gate CI on *new* findings only, and burn the baseline down over
time. A gate that fails on day one gets disabled on day two.

!!! warning "`NOLINT` without a reason is technical debt"
    A bare `// NOLINT` is indistinguishable from a genuine bug that someone
    silenced to get a build green. Require the check name and a justification in
    review, the same way you would for any other exception.

## 7. Putting it in the pipeline

| Stage | Tool | Gate |
|---|---|---|
| Editor / save | clang-tidy (IDE integration) | none |
| Pre-commit | `clang-format`, changed-file clang-tidy | block commit |
| Pull request | Compiler `-Werror`, cppcheck `--error-exitcode=1` | block merge |
| Pull request | Unit tests + ASan/UBSan build | block merge |
| Nightly | Full clang-tidy, coverage report, Valgrind | report + trend |
| Release | Everything, plus MISRA/CERT if regulated | block release |

Order matters: put the fast tools early. A compiler warning that surfaces in two
seconds is worth more than the same finding from a forty-minute nightly job.

## 8. Traps

| Trap | Consequence | Fix |
|---|---|---|
| `--enable=all` in CI | Noise buries real findings | Explicit check list |
| No `--error-exitcode=1` | Findings reported, build green | Add the flag |
| clang-tidy without `-p` | Wrong flags, phantom errors | Export `compile_commands.json` |
| No `HeaderFilterRegex` | Warnings from system headers | Scope to your source |
| Bare `// NOLINT` | Real bugs hidden forever | Name the check, give a reason |
| Warnings without `-Werror` | Ignored within a week | Fail the build |
| Static analysis only | Misses the overflow in §5 | Pair with sanitizers |
| Sanitizers only | Misses the unexecuted error path | Pair with static analysis |
| Gating a legacy repo from day one | Gate gets disabled | Baseline, then ratchet |

## Exercise

1. **Write `buggy.c`** from section 3 and reproduce the cppcheck output exactly.
   For each of the three findings, state whether a unit test would plausibly have
   caught it, and why.

2. **Confirm the blind spot.** Show that cppcheck does not report the `out[n]`
   overflow. Then build the same file with `-fsanitize=address`, run it, and
   capture the report. Write three sentences on what this means for a pipeline
   that runs only one of the two.

3. **Turn on the compiler.** Build your Level 1 project with the full warning
   set from section 2 plus `-Werror`. Count the findings. Fix them, or suppress
   with a written justification, and record how many were genuine.

4. **Try `-fanalyzer`.** Run `gcc -fanalyzer` over the same C code and compare
   its findings with cppcheck's. Note at least one thing each found that the
   other did not.

5. **Configure clang-tidy.** Write a `.clang-tidy` starting from `-*`, enabling
   `bugprone-*` and `clang-analyzer-*`, with `HeaderFilterRegex` scoped to your
   source. Run it via `compile_commands.json`. Then enable `readability-*` as
   well and record how many findings the count jumps by — this is the noise
   argument, quantified.

6. **Suppress honestly.** Find one true false positive. Suppress it inline with
   the check name and a one-line reason. Then find one finding you are tempted to
   suppress but shouldn't, and fix the code instead. Explain the difference in
   two sentences.

7. **Build the gate.** Add a CI script that runs compiler `-Werror`, cppcheck
   with `--error-exitcode=1`, and the ASan test build. Verify the script exits
   non-zero when you reintroduce any one of the three bugs from step 1, and zero
   when the tree is clean.
