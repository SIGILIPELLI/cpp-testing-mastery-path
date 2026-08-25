# 04 · Formal Methods & Static Verification Basics

Every technique so far in this path — unit tests, property tests, fuzzing —
shares one structural limit: they demonstrate correctness for the inputs
exercised, and generalize by sampling, however cleverly (Level 3's property
tests come closest to "many inputs," but still not "all inputs"). Formal
verification proves a property holds for **every possible input**,
mathematically, without running the program at all. This module is an
orientation to what that buys you, what it costs, and where it fits
alongside everything already covered.

!!! note "Environment note"
    No formal verification tool (CBMC, Frama-C, TLA+) is installed on this
    machine, and none were run for this module. The illustrative
    annotations and specifications below are written to match each tool's
    real, documented syntax and were checked by careful manual reasoning
    about what each proof obligation means — but none of the "verified"
    claims in this module were checked by an actual solver on this host.
    Install the relevant tool and run it yourself to get a real proof or
    counterexample.

## 1. Testing vs. formal verification: what changes

| | Testing (Levels 1-3) | Formal verification |
|---|---|---|
| Claim | "For these inputs, behavior X held" | "For all inputs (in the modeled domain), property X holds" |
| Confidence source | Empirical — you ran it | Mathematical — a proof or an exhaustive/symbolic search |
| Cost | Roughly linear in test count | Often exponential in program complexity; frequently infeasible on large code |
| Failure mode when it works | Bugs outside the tested inputs still possible | A genuine counterexample if the property is false, or a proof if true |
| Practical scope | Whole systems, routinely | Small, critical, well-bounded components |

The practical takeaway: formal methods complement testing precisely where
testing is weakest — proving the *absence* of a whole bug class, not just
its absence in the cases you thought to write.

## 2. Bounded model checking with CBMC

CBMC ("C Bounded Model Checker") explores all execution paths of a C
function up to a bound (loop unrolling depth), checking assertions,
array bounds, and other properties symbolically rather than by running
concrete inputs.

```c
/* frame_parse.c -- same function verified empirically in Level 3, module 02 */
int frame_parse(const uint8_t *buf, size_t buf_len, frame_t *out) {
    if (buf_len < 3) return -1;
    uint16_t len = (uint16_t)(buf[1] | (buf[2] << 8));
    if ((size_t)(len + 3) > buf_len) return -1;
    out->type = buf[0];
    out->len = len;
    out->payload = buf + 3;
    return 0;
}
```

```c
/* harness_frame_parse.c -- tells CBMC what to check, not what input to use */
#include "frame_parse.h"

void harness(void) {
    uint8_t buf[MAX_BUF];         /* CBMC treats this as a symbolic, unconstrained
                                      array -- effectively "for all possible
                                      byte contents" */
    size_t buf_len;
    __CPROVER_assume(buf_len <= MAX_BUF);   /* constrain the input domain */
    frame_t out;

    int result = frame_parse(buf, buf_len, &out);

    if (result == 0) {
        /* The property: if parsing succeeded, the payload pointer plus
           its declared length never reaches past the buffer. This is
           the exact invariant Level 3 module 04's exercise asked you to
           state as a property test -- here it's checked for ALL buf_len
           and ALL buffer contents up to MAX_BUF, not a sample. */
        __CPROVER_assert(out.payload + out.len <= buf + buf_len,
                          "payload never overruns buffer");
    }
}
```

```bash
cbmc harness_frame_parse.c frame_parse.c --function harness --unwind 10
```

The tool's output, if the property holds, states the property was proven
for every path up to the unwind bound; if it doesn't hold, CBMC returns a
concrete `buf`/`buf_len` counterexample — a specific failing input, derived
mathematically rather than found by chance (as a fuzzer would).

## 3. Design-by-contract as a bridge between testing and formal methods

Contracts (preconditions, postconditions, invariants) are a middle ground:
lighter weight than full formal proof, but structured enough that both
testing tools (assert at runtime) and formal tools (prove statically) can
consume the same specification.

```cpp
// Frama-C's ACSL annotation style for C, shown as a specification comment
// a formal tool consumes directly -- this is not compiled C++, it's C
// annotated for a specific verification tool's front end.
/*@
  requires buf_len >= 0;
  requires \valid_read(buf + (0 .. buf_len - 1));
  assigns \nothing;
  ensures \result == 0 ==> (\result == 0 && out->len + 3 <= buf_len);
  ensures \result == -1 ==> \true;
*/
int frame_parse(const uint8_t *buf, size_t buf_len, frame_t *out);
```

The same postcondition (`out->len + 3 <= buf_len` when parsing succeeds) is
exactly what a unit test in Level 3 module 02 checked for specific inputs
and what the CBMC harness in section 2 checks for all inputs up to a bound
— design-by-contract is the specification language that makes both
consumers of the same underlying claim explicit and machine-checkable.

## 4. Where full formal proof pays for itself, and where it doesn't

| Good fit | Poor fit |
|---|---|
| A small, security-critical parser (exactly the shape of `frame_parse`) | A large application with extensive I/O, UI, or third-party dependencies |
| A concurrency primitive (a lock-free queue, a mutex implementation) | Code that changes weekly — proof effort has to be redone or maintained per change |
| A cryptographic primitive's constant-time property | Anything whose correctness depends heavily on external system behavior |
| An algorithm with a small, well-defined state space (a protocol state machine) | Code where "correct" is subjective/business-logic-dependent rather than a hard invariant |

The unifying theme: formal methods are worth their cost when the property
is precisely statable, the code is small and stable, and the consequence of
a bug is severe enough to justify the proof effort — which is also, not
coincidentally, exactly the profile of code covered by the safety-critical
standards in module 02.

## 5. Model checking beyond single functions: TLA+ for protocols/algorithms

For concurrent or distributed algorithms, the interesting bugs are often in
the *interleaving* of steps, not in any single function's logic — closer to
what Level 3 module 08's TSan catches empirically, but TLA+ (a
specification language for modeling systems as state machines) can
exhaustively check all interleavings up to a bound, not just the ones that
happened to occur during a test run.

```text
---- MODULE BoundedCounter ----
EXTENDS Naturals

VARIABLES counter, thread1_local, thread2_local

Init == counter = 0 /\ thread1_local = 0 /\ thread2_local = 0

(* Models the EXACT race from Level 3 module 08's race_demo.c:
   read counter, increment locally, write back -- unsynchronized. *)
Thread1Step == /\ thread1_local' = counter
               /\ counter' = thread1_local' + 1
               /\ UNCHANGED thread2_local

Thread2Step == /\ thread2_local' = counter
               /\ counter' = thread2_local' + 1
               /\ UNCHANGED thread1_local

Next == Thread1Step \/ Thread2Step

(* The property module 08 demonstrated empirically with a wrong final
   count: TLC (TLA+'s model checker) would report a reachable state
   violating "counter increases by exactly 1 per step", exhaustively,
   rather than by running the racy program and observing one bad outcome. *)
====
```

This is the same bug demonstrated with real, captured TSan output in Level
3 module 08 — modeled here instead as an exhaustively-checkable state
machine, illustrating the relationship: TSan catches the race when the
racy interleaving actually executes under instrumentation; a model checker
proves the race exists (or doesn't) by exploring the interleaving space
directly, without needing the unlucky schedule to occur at all.

## 6. Traps and honest limits

- **A proof is only as good as its specification.** Proving `frame_parse`
  never overruns the buffer says nothing about whether it parses the
  *intended* protocol correctly — a formally verified function can still
  be formally verified to do the wrong thing.
- **Bounded model checking is bounded.** CBMC's `--unwind 10` proves the
  property for loops executing up to 10 times; a bug only reachable with 11
  iterations is invisible to that run. Increasing the bound increases
  confidence, never reaches "proven for all N" without a separate unwinding
  assertion.
- **State space explosion is real, not a strawman.** A model checker on a
  system with even a modest number of concurrent components can run out of
  memory or time before completing — this is why formal methods target
  small, isolated components (section 4), not whole systems.
- **Formal verification is not a substitute for the rest of this path.**
  It proves properties about a model or a bounded execution space; it does
  not replace integration testing, HIL testing (Level 3, module 03), or
  fuzzing's ability to find bugs in code too large to formally verify at
  all.

## Cheat sheet

| Question | Tool/technique |
|---|---|
| Does this small C function ever overrun a buffer, for any input? | Bounded model checking (CBMC) |
| Can I state pre/postconditions machine-checkably, for both tests and proofs? | Design-by-contract (ACSL, or a lighter in-code assert-based convention) |
| Is there a bad interleaving in this concurrent algorithm, across all schedules? | A model checker (TLA+, or a concurrency-specific tool) |
| Is this the right component to formally verify at all? | Small, stable, high-consequence, precisely-statable property (section 4) |
| Does a passing proof mean the code is "correct"? | Only correct relative to the stated specification — see traps |

## Exercise

1. Write a CBMC-style harness (as in section 2, even without running it)
   for the `BoundedStack::Push`/`Pop` pair from Level 2's capstone, stating
   the property "Size() never exceeds Capacity() after any sequence of
   operations."
2. Write an ACSL-style contract (section 3) for one function in your own
   project, stating at least one precondition and one postcondition in the
   annotation syntax shown.
3. Using section 4's table, classify three functions from your own project
   as good, poor, or borderline fits for full formal verification, with a
   one-sentence justification each.
4. Extend the TLA+ sketch in section 5 to model the *fixed* version of the
   counter (with a mutex, as in Level 3 module 08's `race_fixed.c`) in
   words: what would the corresponding `Next` action need to look like to
   rule out the interleaving that caused the original bug?
5. Write two sentences on the difference between "TSan found this race
   because the racy interleaving happened to execute" (Level 3, module 08)
   and "a model checker proved this race is impossible" — what does the
   second buy you that the first structurally cannot?
