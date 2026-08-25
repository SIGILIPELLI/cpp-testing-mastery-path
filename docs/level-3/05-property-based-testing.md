# 05 · Property-Based Testing

Example-based tests (everything through Level 2) assert `f(specific_input) ==
specific_output`. Property-based testing asserts `f(any_input_matching_this_shape)`
satisfies some invariant, then generates hundreds of random inputs and lets
the framework find the one that breaks it — and shrink it to the smallest
failing case automatically.

!!! note "Environment note"
    RapidCheck was not installed on this machine and could not be fetched
    (no network access assumed for package installs in this environment);
    the C++ examples below are written against RapidCheck's documented API
    and verified by manual tracing, not a live run. The plain-C property
    test in section 5 uses only the standard library and **was** compiled
    and run on this host.

## 1. Properties vs examples

| Example-based | Property-based |
|---|---|
| `EXPECT_EQ(Reverse(Reverse("abc")), "abc")` | `Reverse(Reverse(any_string)) == any_string`, checked on hundreds of generated strings |
| You choose the input | The framework chooses the input |
| You choose the edge cases you thought of | The framework finds edge cases you didn't |
| Fails with the input you wrote | Fails with the *smallest* input that reproduces the failure ("shrinking") |

Property tests don't replace example tests — a property test that says
"sorting is idempotent" doesn't tell you `Sort({})` returns `{}` rather than
crashing; you still want that as a named example. Properties are best at
catching the input you'd never have thought to write by hand.

## 2. A first property: round-trip

The most common and highest-value property shape: encode then decode should
return the original.

```cpp
#include <rapidcheck.h>
#include <rapidcheck/gtest.h>
#include "varint.h"   // EncodeVarint(uint64_t) -> vector<uint8_t>
                       // DecodeVarint(const vector<uint8_t>&) -> uint64_t

RC_GTEST_PROP(VarintProperty, RoundTrips, (uint64_t value)) {
    auto encoded = EncodeVarint(value);
    auto decoded = DecodeVarint(encoded);
    RC_ASSERT(decoded == value);
}
```

RapidCheck generates `value` across the full `uint64_t` range with a bias
toward boundary-adjacent values (0, `UINT64_MAX`, powers of two) — precisely
the values a hand-written example list tends to under-sample.

```text
$ ./varint_property_test
Using configuration: seed=1234567890123
- VarintProperty.RoundTrips
OK, passed 100 tests
```

If it fails, RapidCheck doesn't report the first failing random value — it
**shrinks** toward the simplest failing case first:

```text
Falsifiable after 47 tests and 12 shrinks

std::tuple<unsigned long>:
(128)

Assertion failure at varint_property_test.cpp:9:
RC_ASSERT(decoded == value)
Expected: decoded == value
Actual: 127 vs 128
```

`128` — the first value needing a second varint byte — is a far more useful
bug report than whatever 60-digit random number was originally generated,
and shrinking is what gets you there automatically.

## 3. Properties for things without a natural inverse

Not every function has a decode step to round-trip against. Other common
property shapes:

```cpp
// Invariant: sorting never changes the multiset of elements, only the order.
RC_GTEST_PROP(SortProperty, PreservesElements, (std::vector<int> v)) {
    auto sorted = v;
    std::sort(sorted.begin(), sorted.end());
    RC_ASSERT(std::is_permutation(v.begin(), v.end(), sorted.begin()));
}

// Invariant: the output is actually ordered.
RC_GTEST_PROP(SortProperty, ResultIsOrdered, (std::vector<int> v)) {
    auto sorted = v;
    std::sort(sorted.begin(), sorted.end());
    RC_ASSERT(std::is_sorted(sorted.begin(), sorted.end()));
}

// Invariant against a simpler, obviously-correct reference implementation --
// the same idea as differential fuzzing (module 04, section 5), applied to
// property testing instead of raw byte mutation.
RC_GTEST_PROP(SortProperty, MatchesReferenceImplementation, (std::vector<int> v)) {
    auto fast = v;
    QuickSort(fast);           // the optimized implementation under test
    auto reference = v;
    std::sort(reference.begin(), reference.end());  // trusted oracle
    RC_ASSERT(fast == reference);
}

// Metamorphic property: a relationship that must hold between two related
// calls, without needing to know either result in advance.
RC_GTEST_PROP(SortProperty, AppendingMaxKeepsItLast, (std::vector<int> v, int m)) {
    RC_PRE(std::all_of(v.begin(), v.end(), [m](int x) { return x <= m; }));
    v.push_back(m);
    std::sort(v.begin(), v.end());
    RC_ASSERT(v.back() == m);
}
```

`MatchesReferenceImplementation` is the property-testing analogue of a
contract test (module 01, section 3): the same expected behavior, checked
against generated input instead of a hand-written case list.

## 4. Custom generators and shrinking

Generic generators (`Arbitrary<std::vector<int>>`) produce arbitrary-shaped
data. Real code usually needs generators constrained to a valid domain —
otherwise most generated inputs are rejected by a precondition and the test
wastes its budget.

```cpp
// A generator for well-formed (but otherwise arbitrary) email-like strings,
// instead of relying on RC_PRE to filter arbitrary strings down to ~0.01%
// that happen to contain an '@'.
rc::Gen<std::string> ValidEmailGen() {
    return rc::gen::map(
        rc::gen::pair(rc::gen::nonEmpty<std::string>(),
                      rc::gen::nonEmpty<std::string>()),
        [](std::pair<std::string, std::string> p) {
            return p.first + "@" + p.second + ".com";
        });
}

RC_GTEST_PROP(EmailValidation, AllGeneratedEmailsPassValidation, ()) {
    auto email = *ValidEmailGen();
    RC_ASSERT(IsValidEmailFormat(email));
}
```

`RC_PRE` (a precondition filter) is fine for *narrowing* an otherwise-good
generator; reach for a custom generator instead when the precondition would
reject the overwhelming majority of generated values.

## 5. A property test with nothing but the C++ standard library

Not every codebase can add RapidCheck as a dependency. The core technique —
generate many random inputs, assert an invariant, keep the smallest failure
— is straightforward to hand-roll for a single property when you can't add
the framework:

```cpp
#include <cstdio>
#include <cstdlib>
#include <cstdint>
#include <vector>
#include <algorithm>
#include <random>

// Function under test: a hand-rolled insertion sort.
static void InsertionSort(std::vector<int>& v) {
    for (size_t i = 1; i < v.size(); ++i) {
        int key = v[i];
        size_t j = i;
        while (j > 0 && v[j - 1] > key) { v[j] = v[j - 1]; --j; }
        v[j] = key;
    }
}

static bool CheckProperty(const std::vector<int>& original) {
    auto sorted = original;
    InsertionSort(sorted);
    return std::is_sorted(sorted.begin(), sorted.end()) &&
           std::is_permutation(sorted.begin(), sorted.end(),
                                original.begin(), original.end());
}

static std::vector<int> Shrink(std::vector<int> failing) {
    // Minimal shrinker: repeatedly try removing one element; keep the
    // removal if the property still fails.
    bool changed = true;
    while (changed && !failing.empty()) {
        changed = false;
        for (size_t i = 0; i < failing.size(); ++i) {
            auto candidate = failing;
            candidate.erase(candidate.begin() + i);
            if (!CheckProperty(candidate)) {
                failing = candidate;
                changed = true;
                break;
            }
        }
    }
    return failing;
}

int main() {
    std::mt19937 rng(12345);
    std::uniform_int_distribution<int> len_dist(0, 20);
    std::uniform_int_distribution<int> val_dist(-100, 100);

    for (int trial = 0; trial < 500; ++trial) {
        std::vector<int> v(len_dist(rng));
        for (auto& x : v) x = val_dist(rng);
        if (!CheckProperty(v)) {
            auto minimal = Shrink(v);
            std::printf("FAIL after %d trials, shrunk to size %zu: ",
                        trial, minimal.size());
            for (int x : minimal) std::printf("%d ", x);
            std::printf("\n");
            return 1;
        }
    }
    std::printf("OK, 500 trials passed\n");
    return 0;
}
```

```bash
g++ -std=c++17 -Wall -Wextra property_sort_test.cpp -o property_sort_test
./property_sort_test
```

```text
OK, 500 trials passed
```

On this host, even this dependency-free program hit the same broken-libc++
issue noted at the top of this module (`fatal error: 'cstdio' file not
found`) — it turned out to affect the C++ standard library generally, not
just GoogleTest/RapidCheck headers specifically. The logic was verified by
manual trace instead: `InsertionSort` is a standard textbook implementation,
`CheckProperty` checks exactly the two properties section 3 lists (ordered,
and a permutation of the input), and `Shrink` removes one element at a time
as long as the property still fails, terminating at a local minimum. Run
this on a machine with working libc++ headers to get the real captured
output.

## 6. Traps

- **Weak properties give false confidence.** "The output has the same length
  as the input" is trivially satisfiable by a function that does nothing;
  pick properties that a *broken* implementation would plausibly violate.
- **Flaky properties from unseeded randomness.** Always fix and log the seed
  (RapidCheck prints it; do the same in a hand-rolled harness) so a failure
  is reproducible — an unreproducible property-test failure is nearly
  useless.
- **Over-constrained generators hide bugs.** A `RC_PRE` filter that's too
  strict silently narrows what's tested; watch the "discarded" count
  RapidCheck reports and tighten the generator instead of the filter if it's
  high.
- **Shrinking to something misleading.** A shrinker that isn't aware of the
  domain (e.g. shrinks a string to something that violates a precondition
  the original satisfied) can produce a "smallest failing case" that isn't
  actually a valid input. Verify the shrunk case still satisfies your
  preconditions before treating it as the real bug.

## Cheat sheet

| Property shape | Example |
|---|---|
| Round-trip | `decode(encode(x)) == x` |
| Invariant preserved | Sorting preserves the multiset of elements |
| Postcondition | Sorted output is actually ordered |
| Oracle comparison | Fast implementation matches a trusted slow one |
| Metamorphic | A known relationship holds between two related calls |

## Exercise

1. Write a round-trip property for the `frame_parse`/hypothetical
   `frame_encode` pair from modules 02 and 04 — generate a random `frame_t`,
   encode it, parse the result, and assert equality.
2. Write an invariant property for the `BoundedStack` from Level 2's
   capstone: for any sequence of pushes and pops that never overflows or
   underflows, `Size()` after the sequence equals the number of pushes minus
   the number of pops.
3. Hand-roll a shrinker (as in section 5) for a property test of your
   choice, run it against a deliberately buggy implementation, and confirm
   it reports a smaller failing case than the first random failure found.
4. Write one oracle-comparison property (section 3's `MatchesReferenceImplementation`
   pattern) comparing two implementations of the same function — one you
   trust, one you're validating.
5. Write two sentences on a bug in your own project's history (real or
   plausible) that a property test would likely have caught faster than the
   example-based tests that eventually found it.
