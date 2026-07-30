# 05 · Manual Testing in Practice

Exhaustive testing is impossible (Module 1, principle 2). A single `int`
parameter has 4,294,967,296 possible values; a function taking two of them has
about 1.8×10¹⁹ combinations. At a million tests per second that is roughly
585,000 years.

So testing is fundamentally a **selection problem**: choose the small set of
inputs most likely to expose defects. This module covers the classical
techniques that do that selection systematically — and the specific edge cases
that matter in C and C++ because of integer overflow and fixed-size buffers.

## 1. Equivalence partitioning (EP)

**Idea:** divide the input domain into partitions where every value in a
partition should be handled *the same way* by the program. Then test one
representative value from each partition — because if the program is right for
one member, it's very likely right for the rest, and if it's wrong, one value
finds it as well as a million would.

### Worked example

Requirement:

> **REQ-TEMP-004** — `parse_temperature()` accepts a decimal integer in the
> inclusive range −40 to 125. Valid → return 0 and write the value.
> Out of range → return −2. Non-numeric text → return −1.

Partitions:

| # | Partition | Valid? | Representative | Expected |
|---|---|---|---|---|
| P1 | Integers below −40 | Invalid | `"-100"` | rc = −2 |
| P2 | Integers −40 … 125 | **Valid** | `"25"` | rc = 0, out = 25 |
| P3 | Integers above 125 | Invalid | `"500"` | rc = −2 |
| P4 | Non-numeric text | Invalid | `"abc"` | rc = −1 |
| P5 | Empty string | Invalid | `""` | rc = −1 |
| P6 | Numeric with junk | Invalid | `"25x"` | rc = −1 |
| P7 | `NULL` pointer | Invalid | `NULL` | (spec silent — **requirements gap**) |

Six partitions, six tests instead of billions. And note P7: the act of
partitioning *found a requirements defect*, because there was a class of input
with no defined behaviour. That's routine — EP is as much a specification
review technique as a test design technique.

!!! tip "Partition the outputs too"
    Testers instinctively partition inputs. Also ask: what distinct *outputs*
    exist, and do I have a case producing each one? Here the outputs are
    {0, −1, −2} and "out written" vs. "out unmodified" — that's six output
    combinations to account for. Output partitioning catches paths that input
    partitioning misses.

### Rules of thumb

- Exactly **one** representative per partition; more is waste.
- Every partition needs at least one test — including all the invalid ones.
- Test invalid partitions **one at a time**. If you pass `NULL` *and* an
  out-of-range value in the same case and it returns −1, you don't know which
  input triggered it. This is called *masking*.

## 2. Boundary value analysis (BVA)

**Idea:** defects cluster at the edges of partitions, not in the middle. `<`
versus `<=`, `count` versus `count - 1`, `i <= n` versus `i < n` — off-by-one is
the most common logic error in existence. So test *the boundaries*, not just
the interiors.

### The 3-value method

For each boundary, test: **just below, exactly at, just above.**

For the valid range −40 … 125:

| Boundary | Values | Expected |
|---|---|---|
| Lower | `-41` | rc = −2 (invalid) |
| | `-40` | rc = 0, out = −40 (valid — the minimum) |
| | `-39` | rc = 0, out = −39 (valid) |
| Upper | `124` | rc = 0, out = 124 (valid) |
| | `125` | rc = 0, out = 125 (valid — the maximum) |
| | `126` | rc = −2 (invalid) |

### The 2-value method

Many teams use just the two values straddling each boundary — `-41/-40` and
`125/126` — arguing that the interior values are already covered by EP. That's
four tests instead of six and catches the same off-by-one errors. Both are
defensible; be consistent within a project.

### Why BVA works — the bug it catches

```c
/* Intended: accept -40 to 125 inclusive. */
if (value < -40 || value > 124) {     /* BUG: should be > 125 */
    return -2;
}
```

- An EP-only test with the representative `"25"` — **passes**. Bug survives.
- A BVA test with `"125"` — **fails**. Bug found.

This one-character class of error is why BVA is the highest value-per-test
technique in manual testing.

## 3. C/C++-specific boundaries

Everything above is language-neutral. These next boundaries exist because of
how C and C++ represent data, and they are where the genuinely dangerous defects
live.

### Integer type limits

| Type (typical 64-bit Linux) | Min | Max | Notes |
|---|---|---|---|
| `signed char` | −128 | 127 | |
| `unsigned char` | 0 | 255 | |
| `short` | −32,768 | 32,767 | |
| `unsigned short` | 0 | 65,535 | |
| `int` | −2,147,483,648 | 2,147,483,647 | 16-bit on many small MCUs — check your target |
| `unsigned int` | 0 | 4,294,967,295 | |
| `long` | −2⁶³ | 2⁶³−1 | 32-bit on Windows, 64-bit on Linux |
| `size_t` | 0 | SIZE_MAX | **unsigned** — see below |

Any function taking an integer should be tested at `INT_MAX`, `INT_MIN`,
`INT_MAX - 1`, `INT_MIN + 1`, `0`, `-1`, and `1`. These are always in your
boundary set even when the requirement says nothing about them.

```c
#include <limits.h>
/* Boundary test data for any int parameter */
INT_MIN      /* -2147483648 */
INT_MIN + 1
-1
0
1
INT_MAX - 1
INT_MAX      /* 2147483647 */
```

### Signed integer overflow is undefined behaviour

```c
int a = INT_MAX;
int b = a + 1;      /* UB -- NOT guaranteed to wrap to INT_MIN */
```

The standard does not define this. In practice at `-O0` you'll usually see the
wrap; at `-O2` the compiler may assume it can't happen and delete your check:

```c
/* A common, broken overflow check. */
int sum = a + b;
if (sum < a) {           /* compiler: "a + b can't overflow (UB), so
                            sum < a is impossible" -- check deleted */
    return -1;
}
```

The correct check tests *before* overflowing:

```c
#include <limits.h>
if (b > 0 && a > INT_MAX - b) return -1;   /* would overflow positive */
if (b < 0 && a < INT_MIN - b) return -1;   /* would overflow negative */
int sum = a + b;                            /* now provably safe */
```

**As a tester:** any arithmetic on user-controlled integers gets boundary cases
at `INT_MAX` and `INT_MIN`, and you run them under UBSan (Level 2 Module 5),
because a passing result at `-O0` proves nothing.

### Unsigned wraparound — the `size_t` trap

Unsigned arithmetic is *defined* to wrap modulo 2ⁿ. That makes it not-UB, but
just as dangerous:

```c
size_t len = strlen(s);        /* if s is "" then len == 0 */
for (size_t i = 0; i <= len - 1; i++) {   /* 0 - 1 == SIZE_MAX! */
    process(s[i]);                        /* iterates ~1.8e19 times */
}
```

Test data implication: **for any function taking a length or count, always test
0.** The empty case is where unsigned underflow lives.

```c
/* Another classic -- allocation size overflow */
void *buf = malloc(count * sizeof(Item));   /* count * 24 can wrap to a
                                               tiny number; the malloc
                                               succeeds and every write
                                               overflows the heap */
```

Boundary values for a `count` parameter therefore include `0`, `1`,
`SIZE_MAX / sizeof(Item)`, and `SIZE_MAX / sizeof(Item) + 1`.

### Buffer boundaries

For anything writing into a fixed-size buffer of size N, the standard set is:

| Input length | Why |
|---|---|
| 0 | Empty input; does it still null-terminate? |
| 1 | Minimum non-empty |
| N − 2 | Comfortably inside |
| N − 1 | Exactly fills the buffer *with* room for the terminator |
| N | Exactly fills the buffer *without* room for the terminator — the classic off-by-one |
| N + 1 | One past — must be rejected or truncated |
| 10 × N | Grossly oversized |

```c
char dest[16];
/* Boundary test lengths for a copy into dest: 0, 1, 14, 15, 16, 17, 160 */
```

`N − 1` versus `N` is where `strcpy`-family bugs live, and it is the single most
important boundary pair in C testing.

### Other C/C++ edge inputs worth a case

| Category | Test values |
|---|---|
| Pointers | valid, `NULL`, dangling (freed), unaligned, one-past-the-end |
| Strings | `""`, no null terminator, embedded `\0`, very long, non-ASCII/UTF-8, `\r\n` vs `\n` |
| Floating point | `0.0`, `-0.0`, `NaN`, `+inf`, `-inf`, denormals, `0.1 + 0.2 != 0.3` |
| Arrays | empty (n=0), single element, exactly full, index −1, index n |
| Division | divisor = 0 (UB for integers), `INT_MIN / -1` (also UB — overflows) |
| Shifts | shift by 0, by width−1, by width, by width+1, by a negative amount (last three are UB) |
| Files | missing, empty, permission denied, a directory, a symlink loop, no trailing newline |

!!! warning "`INT_MIN / -1` and `INT_MIN % -1`"
    `-2147483648 / -1` mathematically equals `2147483648`, which does not fit in
    an `int`. It is undefined behaviour and on x86 it raises SIGFPE — the same
    signal as divide-by-zero. Any function doing integer division on
    user-supplied values needs *two* division boundary cases, not one.

## 4. Decision tables

When behaviour depends on a *combination* of conditions, a decision table
enumerates the combinations so none is forgotten.

Requirement: a firmware `should_transmit()` returns true only when the device is
powered, has a valid sensor reading, and either the reading changed or the
periodic timer has elapsed.

| Rule | Powered | Valid reading | Changed | Timer elapsed | → Transmit? |
|---|---|---|---|---|---|
| R1 | F | – | – | – | **No** |
| R2 | T | F | – | – | **No** |
| R3 | T | T | F | F | **No** |
| R4 | T | T | F | T | **Yes** |
| R5 | T | T | T | F | **Yes** |
| R6 | T | T | T | T | **Yes** |

Six rules, six test cases, complete coverage of the logic — versus 2⁴ = 16 naive
combinations, most of which are collapsed by the "–" (don't care) entries.
Decision tables are also excellent *specification review* tools: filling one in
forces the question "what should happen in this combination?" for every row, and
that's where undefined requirements surface.

## 5. State transition testing

For anything with modes — a connection, a state machine, a device driver — model
the states and legal transitions, then test both the legal ones and the
**illegal** ones.

```
        ┌──────────┐  open()   ┌──────────┐  start()  ┌──────────┐
        │  CLOSED  │ ────────► │   IDLE   │ ────────► │ STREAMING│
        └──────────┘           └──────────┘           └──────────┘
             ▲                      ▲   │                  │
             │      close()         │   └──── stop() ◄─────┘
             └──────────────────────┴──────────────────────┘
```

| Test | Sequence | Expected |
|---|---|---|
| Happy path | open → start → stop → close | All succeed |
| Illegal: start before open | start | Error, no crash |
| Illegal: double open | open → open | Error or idempotent — but **specified**, and no leak |
| Illegal: close while streaming | open → start → close | Either rejected, or stops cleanly — never a leaked buffer |
| Illegal: use after close | open → close → start | Error, **not** a use-after-free |
| Resource check | 1000 × (open → close) | Handle count and RSS flat at the end |

That last two rows are why state testing matters so much in C/C++: illegal
transitions in a managed language throw an exception, while in C/C++ they
frequently produce a use-after-free or a leak, and the program keeps running as
if nothing happened.

## 6. Error guessing and checklists

**Error guessing** is the experience-driven technique: you've seen this bug
before, so you go looking for it. It's unsystematic by definition, which is why
it should be *captured as a checklist* so it doesn't depend on who's testing.

A starter C/C++ error-guessing checklist:

- [ ] Pass `NULL` for every pointer parameter, one at a time
- [ ] Pass 0 for every length/count/size parameter
- [ ] Pass the maximum for every length/count/size parameter
- [ ] Pass an empty string where a string is expected
- [ ] Pass a string with no null terminator (via a non-terminated char array)
- [ ] Pass negative values where the code expects positive
- [ ] Pass `INT_MAX` / `INT_MIN` to any arithmetic
- [ ] Call the "cleanup" function twice
- [ ] Call the "cleanup" function without calling "init"
- [ ] Call functions out of their documented order
- [ ] Run the same operation 10,000 times and watch memory
- [ ] Run the same input at `-O0` and `-O2` and compare results
- [ ] Run with a different compiler (gcc vs clang)
- [ ] Fill output buffers with a canary byte and verify what was really written
- [ ] Interrupt the operation midway (signal, timeout, disconnect)

## 7. Combining the techniques — a worked pass

Function under test:

```c
/* REQ-CPY-001: copies at most dest_size-1 chars from src into dest,
   always null-terminates dest, returns chars copied, or -1 if
   dest is NULL, src is NULL, or dest_size is 0. */
int safe_copy(char *dest, size_t dest_size, const char *src);
```

**Step 1 — Equivalence partitions:**

| # | Partition | Representative |
|---|---|---|
| P1 | `dest` NULL | `NULL, 16, "hi"` |
| P2 | `src` NULL | `buf, 16, NULL` |
| P3 | `dest_size` == 0 | `buf, 0, "hi"` |
| P4 | src shorter than dest_size−1 | `buf[16], 16, "hi"` |
| P5 | src longer than dest_size−1 (truncation) | `buf[16], 16, "<40 chars>"` |
| P6 | src empty | `buf[16], 16, ""` |

**Step 2 — Boundaries** on `strlen(src)` against `dest_size` = 16:

| `strlen(src)` | Expected | Why this value |
|---|---|---|
| 0 | returns 0, `dest[0] == '\0'` | empty-input boundary |
| 1 | returns 1 | minimum non-empty |
| 14 | returns 14 | one inside the limit |
| **15** | returns 15, `dest[15] == '\0'` | exactly fills the buffer including terminator |
| **16** | returns 15 (truncated), `dest[15] == '\0'` | the off-by-one boundary — where `strcpy` bugs live |
| 17 | returns 15 (truncated) | one past |
| 200 | returns 15 (truncated) | grossly oversized |

**Step 3 — `dest_size` boundaries** independently: 0 (→ −1), 1 (→ copies
nothing, writes only `'\0'`, returns 0), 2, and `SIZE_MAX`.

**Step 4 — Canary verification.** For every copy case, pre-fill `dest` with
`'X'` and assert that bytes *after* the terminator are still `'X'`. Without
this, a function that writes one byte too many passes every functional check.

```c
char dest[16];
memset(dest, 'X', sizeof(dest));
int n = safe_copy(dest, 16, "hello");
/* check: n == 5, dest[5] == '\0', and dest[6..15] are still 'X' */
```

That's roughly 6 + 7 + 4 = 17 well-chosen cases for a three-parameter function.
Compare that with the ~10³⁰ inputs that technically exist — and note that these
17 would find essentially every realistic defect in an implementation of this
signature.

## 8. Manual test execution discipline

When you actually run the cases:

1. **Record the environment first** — compiler, version, flags, target. Before
   the first case, not after the first failure.
2. **Run in a defined order and note deviations.** If case 7 requires state left
   by case 6, that's a design flaw in your cases (Module 2), but if you're stuck
   with it, document it.
3. **Record actual results verbatim.** Copy-paste the output; don't paraphrase.
   "Returned an error" loses the information that it returned −1 rather than −2.
4. **Log a defect the moment a case fails** — then continue. Don't stop the run
   to debug; you'll lose the rest of the day and the remaining coverage.
5. **Distinguish Fail from Blocked.** *Fail* = the software did the wrong thing.
   *Blocked* = you couldn't run the case (build broken, hardware unavailable).
   Reporting blocked cases as passed is how coverage silently evaporates.
6. **Re-run the whole affected area after a fix** — the fix is a change, and
   changes cause regressions (Module 3).

## Exercise

You are testing this C function:

```c
/* REQ-BUF-002: ring_write() copies n bytes from src into the ring buffer rb.
   Returns the number of bytes written. If the buffer has less than n bytes
   of free space, it writes as many as fit and returns that count.
   Returns -1 if rb is NULL, src is NULL, or rb has not been initialized.
   The buffer capacity is 64 bytes. */
int ring_write(RingBuffer *rb, const uint8_t *src, size_t n);
```

Produce:

1. **An equivalence partition table** for all three parameters, with one
   representative value each. Include at least one partition that reveals a
   *requirements gap* — a class of input the requirement does not define — and
   say what question you would ask the developer.

2. **A boundary value table** for `n` against the 64-byte capacity, using the
   3-value method, for two starting conditions: an empty buffer and a buffer
   already holding 60 bytes. State the expected return value for every row.

3. **A C/C++-specific edge case list** of at least six entries drawn from
   sections 3 and 6, each with the specific value you'd pass and the specific
   failure mode you are hunting. At least one must target unsigned wraparound on
   `n`, and one must target the fact that `src` is a byte pointer with no
   terminator.

4. **A decision table** for a proposed extension where the function also takes a
   `bool overwrite` flag (overwrite the oldest data when full) and a
   `bool blocking` flag. Enumerate the rules for the combinations of
   {buffer full?, overwrite?, blocking?} and give the expected behaviour for
   each — marking any row where the requirement as stated is ambiguous.

5. Finally, pick the **five highest-value cases** from everything above — the
   five you'd run if you had ten minutes — and justify each in one sentence.
   Keep this list; it becomes your smoke suite in Module 10.
