# 01 · GoogleTest Advanced

Level 1 covered `TEST`, `EXPECT_EQ` and fixtures — enough to test a calculator.
Real suites hit walls that basic assertions don't solve: the same logic checked
against twenty inputs, one contract checked against five types, a failure message
that says `false is not true` and nothing else. This module covers the GoogleTest
features that keep a suite readable as it grows past a few hundred tests.

## 1. Fixture lifecycle — per-test vs per-suite

`SetUp`/`TearDown` run **once per test**. `SetUpTestSuite`/`TearDownTestSuite`
run once for the whole suite and are `static`. Use the per-suite hooks only for
expensive, genuinely read-only setup — a loaded reference table, a temp
directory. The moment tests mutate per-suite state, you have order-dependent
tests.

```cpp
class DatabaseTest : public ::testing::Test {
protected:
    static void SetUpTestSuite() {          // once, before the first test
        schema_ = new Schema("fixtures/schema.sql");
    }
    static void TearDownTestSuite() {       // once, after the last test
        delete schema_;
        schema_ = nullptr;
    }
    void SetUp() override { conn_.open(*schema_); }   // before every test
    void TearDown() override { conn_.close(); }       // after every test

    static Schema* schema_;
    Connection conn_;
};
Schema* DatabaseTest::schema_ = nullptr;
```

!!! warning "The per-suite trap"
    Anything a test writes into `schema_` leaks into every later test in the
    suite. If you cannot make the shared object `const`, it probably belongs in
    `SetUp()` instead. Verify with `--gtest_shuffle` (section 7).

## 2. Parameterized tests — one body, many inputs

When several tests differ only in their input and expected output, collapse them
into a `TEST_P`. Define a parameter struct, derive from
`TestWithParam<T>`, and instantiate.

```cpp
#include <gtest/gtest.h>
#include <stdexcept>
#include <string>

int clampi(int v, int lo, int hi) {
    if (lo > hi) throw std::invalid_argument("lo>hi");
    return v < lo ? lo : (v > hi ? hi : v);
}

struct ClampCase { int v, lo, hi, want; };

class ClampTest : public ::testing::TestWithParam<ClampCase> {};

TEST_P(ClampTest, Clamps) {
    const ClampCase& c = GetParam();
    EXPECT_EQ(clampi(c.v, c.lo, c.hi), c.want);
}

INSTANTIATE_TEST_SUITE_P(Boundaries, ClampTest, ::testing::Values(
    ClampCase{  5, 0, 10,  5},   // interior
    ClampCase{ -1, 0, 10,  0},   // below low
    ClampCase{ 11, 0, 10, 10},   // above high
    ClampCase{  0, 0, 10,  0},   // exactly low
    ClampCase{ 10, 0, 10, 10}    // exactly high
), [](const testing::TestParamInfo<ClampCase>& i) {
    return "case" + std::to_string(i.index);        // readable test names
});
```

Build and run (GoogleTest installed via Homebrew; adjust the prefix for your
system):

```bash
clang++ -std=c++17 -I/opt/homebrew/opt/googletest/include \
    clamp_test.cpp -L/opt/homebrew/opt/googletest/lib \
    -lgtest -lgtest_main -o clamp_test
./clamp_test
```

```
[----------] 5 tests from Boundaries/ClampTest
[ RUN      ] Boundaries/ClampTest.Clamps/case0
[       OK ] Boundaries/ClampTest.Clamps/case0 (0 ms)
[ RUN      ] Boundaries/ClampTest.Clamps/case1
[       OK ] Boundaries/ClampTest.Clamps/case1 (0 ms)
[ RUN      ] Boundaries/ClampTest.Clamps/case2
[       OK ] Boundaries/ClampTest.Clamps/case2 (0 ms)
[ RUN      ] Boundaries/ClampTest.Clamps/case3
[       OK ] Boundaries/ClampTest.Clamps/case3 (0 ms)
[ RUN      ] Boundaries/ClampTest.Clamps/case4
[       OK ] Boundaries/ClampTest.Clamps/case4 (0 ms)
[----------] 5 tests from Boundaries/ClampTest (0 ms total)
```

Each case is a **separate test**: one failure names `case2`, and the other four
still run. That is the whole point — a loop inside a single `TEST` would stop at
the first failure and report one result.

!!! tip "Name your cases"
    Without the name-generator lambda you get `Clamps/0`, `Clamps/1`, … . With a
    struct field you can do better still: return `i.param.label` and put a
    human-readable label in the case. Failure output is the product.

## 3. Typed tests — one contract, many types

Parameterized tests vary the *value*. Typed tests vary the *type*. Use them when
several implementations must satisfy the same contract.

```cpp
template <typename T>
class ContainerTest : public ::testing::Test {
protected:
    T c_;
};

using ContainerTypes = ::testing::Types<std::vector<int>, std::vector<long>>;
TYPED_TEST_SUITE(ContainerTest, ContainerTypes);

TYPED_TEST(ContainerTest, StartsEmpty) {
    EXPECT_TRUE(this->c_.empty());     // note: this-> is required
}
```

```
[----------] 1 test from ContainerTest/0, where TypeParam = std::vector<int, std::allocator<int>>
[ RUN      ] ContainerTest/0.StartsEmpty
[       OK ] ContainerTest/0.StartsEmpty (0 ms)
[----------] 1 test from ContainerTest/1, where TypeParam = std::vector<long, std::allocator<long>>
[ RUN      ] ContainerTest/1.StartsEmpty
[       OK ] ContainerTest/1.StartsEmpty (0 ms)
```

!!! warning "`this->` is not optional"
    In a template, members of a dependent base class are not visible to
    unqualified lookup. Writing `c_.empty()` fails to compile with a message
    about `c_` not being declared. Always `this->c_`.

## 4. Matchers — assertions that explain themselves

`EXPECT_THAT(value, matcher)` produces far better failure text than a boolean.
Matchers come from `<gmock/gmock.h>` but work in plain GoogleTest.

```cpp
#include <gmock/gmock.h>
using ::testing::ElementsAre;
using ::testing::HasSubstr;
using ::testing::AllOf, ::testing::Gt, ::testing::Lt;
using ::testing::UnorderedElementsAre;

EXPECT_THAT(name, HasSubstr("ell"));
EXPECT_THAT(v, ElementsAre(1, 2, 3));              // order matters
EXPECT_THAT(tags, UnorderedElementsAre("a", "b")); // order does not
EXPECT_THAT(score, AllOf(Gt(0), Lt(100)));
```

Compare the two failure messages for a vector mismatch:

```
# EXPECT_TRUE(v == expected)
  Value of: v == expected
    Actual: false
  Expected: true

# EXPECT_THAT(v, ElementsAre(1, 2, 3))
  Value of: v
  Expected: has 3 elements where element #0 is equal to 1, ...
    Actual: { 1, 2, 4 }, whose element #2 doesn't match
```

The second one tells you what to fix without opening a debugger.

## 5. Death tests

A death test asserts that code terminates the process — an `assert`, an
`abort()`, a fatal log. The statement runs in a forked child.

```cpp
TEST(ConfigDeathTest, RejectsNegativeCapacity) {
    EXPECT_DEATH(make_pool(-1), "capacity must be positive");
}
```

Naming matters: put death tests in a suite ending in `DeathTest` so GoogleTest
runs them before other suites, which avoids forking a process that already
holds threads.

!!! warning "Death tests and threads"
    Forking a multi-threaded process is only conditionally safe. If your fixture
    starts threads, GoogleTest will print a warning about the death test style.
    Prefer testing the *precondition check* on a single-threaded path, or switch
    the API to throw so you can use `EXPECT_THROW` instead.

For exceptions — the far more common case in C++ — use the throw family, which
needs no forking:

```cpp
TEST(Clamp, RejectsInvertedRange) {
    EXPECT_THROW(clampi(1, 10, 0), std::invalid_argument);
    EXPECT_NO_THROW(clampi(1, 0, 10));
}
```

## 6. Locating failures inside helpers

A failure reported inside a shared helper points at the helper's line, not the
call site. `SCOPED_TRACE` adds the context back.

```cpp
void ExpectValidUser(const User& u) {
    EXPECT_FALSE(u.name.empty());
    EXPECT_GT(u.id, 0);
}

TEST(UserTest, AllFixturesValid) {
    for (const auto& u : load_fixtures()) {
        SCOPED_TRACE("user id=" + std::to_string(u.id));
        ExpectValidUser(u);
    }
}
```

Helpers that use `ASSERT_*` must return `void` and be invoked through
`ASSERT_NO_FATAL_FAILURE(Helper())` — otherwise the `ASSERT` returns from the
*helper* and the test carries on believing everything passed.

## 7. Runner flags that catch real bugs

```bash
./clamp_test --gtest_filter='Boundaries/*'       # run a subset
./clamp_test --gtest_shuffle --gtest_repeat=20   # hunt order dependence
./clamp_test --gtest_brief=1                     # only failures
./clamp_test --gtest_output=xml:results.xml      # CI-consumable report
./clamp_test --gtest_list_tests                  # enumerate without running
```

`--gtest_shuffle --gtest_repeat=20` is the single most valuable command here. A
suite that passes in file order and fails when shuffled has shared state, and
that shared state will eventually produce a flaky failure in CI at the worst
possible time.

## 8. Choosing the right tool

| Situation | Use | Not |
|---|---|---|
| Same logic, many input/expected pairs | `TEST_P` + `INSTANTIATE_TEST_SUITE_P` | a loop inside one `TEST` |
| Same contract, several types | `TYPED_TEST_SUITE` | copy-pasted suites |
| Container contents | `EXPECT_THAT(v, ElementsAre(...))` | `EXPECT_TRUE(v == e)` |
| Order-insensitive contents | `UnorderedElementsAre` | sorting then comparing |
| Function throws | `EXPECT_THROW` | `try`/`catch` + `FAIL()` |
| Function calls `abort()` | `EXPECT_DEATH` (suite named `*DeathTest`) | `EXPECT_THROW` |
| Expensive read-only setup | `SetUpTestSuite` | a file-scope global |
| Assertion inside a helper | `SCOPED_TRACE` | guessing from the line number |
| Fatal check inside a helper | `ASSERT_NO_FATAL_FAILURE(Helper())` | bare `Helper()` |

## Exercise

Take the `Statistics` class you built at the end of Level 1 Module 7 (or write a
small `TextBuffer` class if you'd rather start fresh) and rework its suite.

1. **Convert to parameterized tests.** Replace your hand-written `mean` tests
   with a single `TEST_P` driven by at least **eight** cases, including the
   empty input, a single element, values that don't divide evenly, and negative
   values. Supply a name generator so the failure output reads
   `Mean/StatsTest.Computes/negatives` rather than `Computes/3`.

2. **Add a typed test.** Make `count_above` a template over the element type,
   then write a `TYPED_TEST_SUITE` covering `int`, `long`, and `double` that
   asserts the "strictly greater" boundary holds for every type. Confirm the
   runner reports one test per type.

3. **Replace three boolean assertions with matchers.** Find three
   `EXPECT_TRUE`/`EXPECT_FALSE` assertions and rewrite them with `EXPECT_THAT`.
   Deliberately break the code under test, capture both failure messages, and
   write two sentences on what the matcher told you that the boolean did not.

4. **Prove independence.** Add a `SetUpTestSuite` that loads a shared fixture
   vector, then deliberately have one test `push_back` onto it. Run
   `--gtest_shuffle --gtest_repeat=20` and record the failure. Fix it by moving
   the state into `SetUp()`, and re-run to confirm 20 clean passes.

5. **Trace a helper.** Write an `ExpectSaneStats()` helper used by three tests,
   break one input so it fails, and observe that the report names the helper.
   Add `SCOPED_TRACE` and show the improved output.
