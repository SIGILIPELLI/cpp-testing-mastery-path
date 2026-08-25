# 09 · Legacy Code: Getting Untestable Code Under Test

"Legacy code" here means, per Michael Feathers' definition, code without
tests — regardless of age. The hard part is never writing the test; it's
that the code has no seam to insert one, and safely creating a seam requires
changing code you don't yet have tests to protect. This module is the
escape route from that circle.

!!! note "Environment note"
    The C example in section 2 (characterization test) **was** compiled and
    run with plain `gcc` on this host and its output is real. The C++
    example in section 4 (extract-and-override seam) was verified by manual
    trace only, since this host's libc++ headers are broken (see module 01's
    note) — noted inline again there.

## 1. The problem, precisely

```c
/* pricing.c -- no seam, no tests, and everyone is afraid to touch it */
double calculate_total(int item_count, double unit_price, const char *region) {
    double subtotal = item_count * unit_price;
    double tax_rate = 0.0;
    if (strcmp(region, "CA") == 0) tax_rate = 0.0725;
    else if (strcmp(region, "NY") == 0) tax_rate = 0.08;
    else if (strcmp(region, "OR") == 0) tax_rate = 0.0;
    double discount = 0.0;
    if (item_count >= 10) discount = subtotal * 0.05;
    return (subtotal - discount) * (1 + tax_rate);
}
```

Not a horror story by legacy-code standards — no globals, no hidden I/O —
but it has zero tests, and nobody currently knows whether the discount is
supposed to apply before or after tax without reading it that closely
(it's before, here).

## 2. Step one: a characterization test, not a correctness test

Before changing anything, pin down *current* behavior exactly, bugs
included. This is not the same activity as writing a correct test — you are
recording what the code does, not what it should do, so a refactor has
something to diff against.

```c
/* test_pricing_characterization.c */
#include <assert.h>
#include <math.h>
#include <string.h>

extern double calculate_total(int item_count, double unit_price, const char *region);

static int close_enough(double a, double b) { return fabs(a - b) < 1e-9; }

int main(void) {
    /* Every case below was RUN against the existing function and its
       result recorded here -- these are observations, not requirements. */
    assert(close_enough(calculate_total(1, 10.0, "CA"), 10.725));
    assert(close_enough(calculate_total(1, 10.0, "NY"), 10.8));
    assert(close_enough(calculate_total(1, 10.0, "OR"), 10.0));
    assert(close_enough(calculate_total(1, 10.0, "TX"), 10.0));   /* unknown
        region silently gets 0% tax -- almost certainly a bug, but the
        characterization test records it AS-IS; fixing it is a separate,
        deliberate, tested change, not a side effect of a refactor */
    assert(close_enough(calculate_total(10, 10.0, "CA"),
                         (100.0 - 5.0) * 1.0725));   /* discount tier boundary */
    assert(close_enough(calculate_total(9, 10.0, "CA"), 90.0 * 1.0725)); /* just below it */

    return 0;
}
```

```bash
gcc test_pricing_characterization.c pricing.c -o test_char -lm
./test_char && echo "characterization: all current behavior confirmed"
```

```text
characterization: all current behavior confirmed
```

This was actually run against the `pricing.c` above; every asserted number
is the real computed result, not a guess — which is the entire point of a
characterization test.

!!! warning "Do not fix bugs while characterizing"
    The unknown-region-gets-0%-tax behavior above is very likely wrong. A
    characterization test still asserts it, because its job is to make the
    *next* change (a refactor, or eventually a deliberate bug fix) safe —
    conflating "record current behavior" with "fix behavior" removes your
    only safety net for the refactor itself.

## 3. Step two: find a seam without changing behavior

With characterization tests as a safety net, the next move is a seam that
doesn't require solving the whole design problem at once. The **link seam**
(Level 2, module 07) is often the least invasive: extract the
region-to-tax-rate logic into its own translation unit, callable and
therefore substitutable, with no change to `calculate_total`'s observable
behavior.

```c
/* tax_rate.h -- new file, the seam */
double tax_rate_for_region(const char *region);

/* tax_rate.c -- new file, logic moved verbatim */
double tax_rate_for_region(const char *region) {
    if (strcmp(region, "CA") == 0) return 0.0725;
    if (strcmp(region, "NY") == 0) return 0.08;
    if (strcmp(region, "OR") == 0) return 0.0;
    return 0.0;   /* still a bug -- still not being fixed here */
}
```

```c
/* pricing.c -- the only line that changed in the original function */
double calculate_total(int item_count, double unit_price, const char *region) {
    double subtotal = item_count * unit_price;
    double tax_rate = tax_rate_for_region(region);
    double discount = 0.0;
    if (item_count >= 10) discount = subtotal * 0.05;
    return (subtotal - discount) * (1 + tax_rate);
}
```

Re-run the *same* characterization tests, unchanged, after this extraction:

```bash
gcc test_pricing_characterization.c pricing.c tax_rate.c -o test_char -lm
./test_char && echo "characterization: still passes after extraction"
```

```text
characterization: still passes after extraction
```

If any characterization assertion had failed here, the extraction — not the
original function — introduced a behavior change, caught immediately rather
than discovered downstream.

Now `tax_rate_for_region` is independently, directly testable, and the bug
(unknown region → 0% tax) can be fixed as its own small, reviewed,
*tested* change:

```c
static void test_unknown_region_defaults_documented(void) {
    /* Once this is deliberately decided (say, unknown region should be
       an error, not silent 0%), this test replaces the characterization
       assertion above -- a conscious, reviewed behavior change, not an
       accidental one. */
}
```

## 4. Seams for object-oriented legacy code (C++)

The equivalent move in C++ when the dependency is a concrete class with no
interface: **extract-and-override**. Pull the hard-to-test call into a
protected virtual method, then subclass it in tests to substitute a fake —
without touching the class's public contract.

```cpp
// Before: untestable because it reaches directly into a real network call.
class PaymentProcessor {
public:
    bool Charge(double amount) {
        auto response = HttpPost("https://payments.example/charge", amount);
        return response.status == 200;
    }
};
```

```cpp
// After: one seam added, zero behavior change for production callers.
class PaymentProcessor {
public:
    bool Charge(double amount) {
        auto response = DoHttpPost(amount);   // now goes through the seam
        return response.status == 200;
    }
protected:
    virtual HttpResponse DoHttpPost(double amount) {
        return HttpPost("https://payments.example/charge", amount);
    }
};

// tests/fake_payment_processor.h
class TestablePaymentProcessor : public PaymentProcessor {
public:
    HttpResponse next_response{200, ""};
    double last_amount_charged = 0.0;
protected:
    HttpResponse DoHttpPost(double amount) override {
        last_amount_charged = amount;
        return next_response;
    }
};
```

```cpp
TEST(PaymentProcessor, ChargeReturnsTrueOn200) {
    TestablePaymentProcessor p;
    p.next_response = {200, ""};
    EXPECT_TRUE(p.Charge(19.99));
    EXPECT_DOUBLE_EQ(p.last_amount_charged, 19.99);
}

TEST(PaymentProcessor, ChargeReturnsFalseOnNon200) {
    TestablePaymentProcessor p;
    p.next_response = {500, "server error"};
    EXPECT_FALSE(p.Charge(19.99));
}
```

`DoHttpPost` being `virtual` and `protected` (not `private`) is the whole
trick — it costs one vtable indirection in production and buys a
substitution point in tests, with the public `Charge` API completely
unchanged.

## 5. The order of operations, always

1. **Characterize** current behavior (section 2) — no behavior change yet.
2. **Extract a seam** (sections 3-4) — verify characterization tests still
   pass; if they don't, the extraction itself has a bug.
3. **Write real unit tests** against the newly seamed, isolated piece.
4. **Only then** fix any bugs the characterization step surfaced, as their
   own deliberate, reviewed, tested changes.

Skipping straight to step 4 — "I'll just fix the tax bug while I'm in
here" — is how a legacy-code cleanup turns into an untested behavior change
indistinguishable, in a diff, from a refactor.

## 6. Traps

- **Characterizing too little.** A characterization suite with three happy-
  path cases gives false confidence; deliberately include boundary and
  weird-input cases (the discount tier boundary above) precisely because
  those are where a refactor is most likely to introduce a subtle change.
- **Golden-master over-reliance.** For complex outputs, capturing full
  output as a "golden file" and diffing against it can substitute for
  hand-written assertions — but a golden master that's never reviewed by a
  human is just as capable of encoding a bug as a hand-written assertion
  is, and is harder to read when it breaks.
- **Refactoring and behavior-fixing in the same commit.** Keep them
  separate commits (ideally separate PRs) so a regression can be bisected
  to "the extraction" or "the intentional fix," not both at once.
- **Adding a seam that itself needs elaborate testing to trust.** The
  `virtual`/`override` seam in section 4 is minimal specifically to avoid
  this — resist the urge to build a whole abstract interface hierarchy
  before you have a single test protecting the class.

## Cheat sheet

| Situation | Move |
|---|---|
| No tests exist yet | Characterization tests first, bugs and all |
| Hard-wired C dependency | Link seam (Level 2, module 07) |
| Hard-wired call inside a C++ method | Extract-and-override (protected virtual) |
| Found a bug while characterizing | Record it as-is; fix it in a separate, tested change |
| Extraction changed a characterization result | The extraction has a bug — stop and investigate |

## Exercise

1. Take the `calculate_total` function, add a characterization test for a
   `region_count == 0` edge case (zero items), run it against the original
   function, and record whatever it actually returns — even if it looks
   wrong.
2. Perform the same link-seam extraction shown in section 3 for the
   discount calculation (`>= 10 items -> 5% off`), re-run your full
   characterization suite, and confirm nothing changed.
3. Once `tax_rate_for_region` is isolated, decide what unknown regions
   *should* do (error? a documented default rate?) and write that as a new,
   named, deliberate test — replacing the old characterization assertion
   for that case.
4. Apply the extract-and-override pattern (section 4) to one method in a
   class you own that currently reaches out to something hard to test
   (a file, a socket, a global), and write two unit tests against the
   resulting seam.
5. Write two sentences on a piece of code in your own project you'd
   currently be afraid to refactor, and name the first characterization
   test you'd write before touching it.
