# 10 · Project — Test Plan & GoogleTest Suite

This project ties Level 1 together. You'll take a small C++ library, write a
**manual test plan** for it using the documentation techniques from modules
02-05, then **automate** the parts worth automating with GoogleTest and CTest
from modules 06-09.

The point is the pairing: a test plan tells you *what* to verify and why, and
the automated suite locks in the subset that's cheap to run on every commit.
Neither replaces the other.

## The system under test

A `TextStats` library that analyses a block of text. Small enough to finish,
big enough to have real edge cases.

```text
textstats/
    CMakeLists.txt
    include/textstats/textstats.h
    src/textstats.cpp
    tests/
        CMakeLists.txt
        unit/test_word_count.cpp
        unit/test_longest_word.cpp
        unit/test_average_length.cpp
    docs/
        test-plan.md
```

### include/textstats/textstats.h

```cpp
#ifndef TEXTSTATS_H
#define TEXTSTATS_H

#include <string>
#include <vector>

namespace textstats {

// Splits on whitespace. Consecutive separators do not produce empty words.
std::vector<std::string> Tokenize(const std::string& text);

// Number of whitespace-separated words.
std::size_t WordCount(const std::string& text);

// Longest word. On a tie, returns the FIRST one encountered.
// Returns "" for text with no words.
std::string LongestWord(const std::string& text);

// Mean word length. Returns 0.0 for text with no words -- deliberately
// NOT a division-by-zero or a throw. This is a documented contract.
double AverageWordLength(const std::string& text);

}  // namespace textstats

#endif
```

### src/textstats.cpp

```cpp
#include "textstats/textstats.h"

#include <cctype>
#include <sstream>

namespace textstats {

std::vector<std::string> Tokenize(const std::string& text) {
    std::vector<std::string> words;
    std::istringstream stream(text);
    std::string word;
    // operator>> already collapses runs of whitespace for us.
    while (stream >> word) {
        words.push_back(word);
    }
    return words;
}

std::size_t WordCount(const std::string& text) {
    return Tokenize(text).size();
}

std::string LongestWord(const std::string& text) {
    std::string longest;
    for (const auto& word : Tokenize(text)) {
        if (word.size() > longest.size()) {  // strict > keeps the FIRST on a tie
            longest = word;
        }
    }
    return longest;
}

double AverageWordLength(const std::string& text) {
    const auto words = Tokenize(text);
    if (words.empty()) {
        return 0.0;  // documented: empty input is 0.0, not UB
    }
    std::size_t total = 0;
    for (const auto& word : words) {
        total += word.size();
    }
    return static_cast<double>(total) / static_cast<double>(words.size());
}

}  // namespace textstats
```

## Part 1 — the manual test plan

Write `docs/test-plan.md`. This is the deliverable that proves you can *think*
about testing, not just type `EXPECT_EQ`.

### 1.1 Scope and approach

| Field | Entry |
|-------|-------|
| Feature under test | `TextStats` library v0.1 |
| In scope | `Tokenize`, `WordCount`, `LongestWord`, `AverageWordLength` |
| Out of scope | Unicode/multi-byte handling (documented as unsupported in v0.1) |
| Test levels | Unit (automated), exploratory (manual) |
| Entry criteria | Library compiles clean with `-Wall -Wextra` |
| Exit criteria | All planned unit tests pass; no Sev-1/Sev-2 defects open |

Stating "out of scope" explicitly matters. If nobody wrote down that Unicode
isn't supported, the first multi-byte bug report becomes an argument instead of
a known limitation.

### 1.2 Equivalence partitions and boundaries

Apply the techniques from module 05 to `WordCount`:

| Partition | Representative input | Expected |
|-----------|---------------------|----------|
| Empty text | `""` | 0 |
| Whitespace only | `"   \t\n "` | 0 |
| Single word | `"hello"` | 1 |
| Multiple words | `"the quick brown fox"` | 4 |
| Runs of separators | `"a\t\tb  c"` | 3 |
| Leading/trailing space | `"  hi  "` | 1 |

The boundary that actually bites here is **zero words** — it's the input that
drives `AverageWordLength` toward a division by zero. Boundary analysis is what
surfaces that before the crash does.

### 1.3 Test case documentation

Use the template from module 02. One worked example:

| Field | Entry |
|-------|-------|
| ID | TC-TS-011 |
| Title | `AverageWordLength` returns 0.0 for whitespace-only input |
| Precondition | Library built |
| Steps | 1. Call `AverageWordLength("   \t  ")` |
| Expected | Returns `0.0`; no crash, no exception |
| Actual | *(fill during execution)* |
| Status | *(Pass/Fail)* |
| Severity if failed | Sev-2 — likely a divide-by-zero / UB path |

Write at least **12** such cases across the four functions, including the tie
case for `LongestWord` and the runs-of-separators case for `Tokenize`.

### 1.4 An exploratory charter

Manual testing isn't only scripted cases. Write one charter in the
session-based format:

> **Explore** `TextStats` with pathological inputs
> **With** very long strings, embedded `\0`, punctuation-only tokens, mixed
> line endings
> **To discover** crashes, silent truncation, or surprising word counts
>
> Timebox: 30 minutes. Record anything surprising as a defect or a question.

## Part 2 — the automated suite

Now automate the deterministic cases. Exploratory findings stay manual.

### tests/unit/test_word_count.cpp

```cpp
#include <gtest/gtest.h>

#include "textstats/textstats.h"

TEST(WordCount, EmptyStringHasNoWords) {
    EXPECT_EQ(textstats::WordCount(""), 0u);
}

TEST(WordCount, WhitespaceOnlyHasNoWords) {
    EXPECT_EQ(textstats::WordCount("   \t\n  "), 0u);
}

TEST(WordCount, SingleWord) {
    EXPECT_EQ(textstats::WordCount("hello"), 1u);
}

TEST(WordCount, MultipleWords) {
    EXPECT_EQ(textstats::WordCount("the quick brown fox"), 4u);
}

TEST(WordCount, CollapsesRunsOfSeparators) {
    EXPECT_EQ(textstats::WordCount("a\t\tb  c"), 3u);
}

TEST(WordCount, IgnoresLeadingAndTrailingWhitespace) {
    EXPECT_EQ(textstats::WordCount("  hi  "), 1u);
}
```

### tests/unit/test_longest_word.cpp

```cpp
#include <gtest/gtest.h>

#include "textstats/textstats.h"

TEST(LongestWord, EmptyInputReturnsEmptyString) {
    EXPECT_EQ(textstats::LongestWord(""), "");
}

TEST(LongestWord, FindsTheLongest) {
    EXPECT_EQ(textstats::LongestWord("a bb ccc dd"), "ccc");
}

// This is the case the test PLAN surfaced -- the header documents
// "first one wins", so the test pins that contract down.
TEST(LongestWord, OnTieReturnsTheFirstOccurrence) {
    EXPECT_EQ(textstats::LongestWord("aaa bbb"), "aaa");
}

TEST(LongestWord, PunctuationCountsAsPartOfTheWord) {
    EXPECT_EQ(textstats::LongestWord("hi world!"), "world!");
}
```

### tests/unit/test_average_length.cpp

```cpp
#include <gtest/gtest.h>

#include "textstats/textstats.h"

// The boundary case from section 1.2 -- the one that would be a
// division by zero if the contract weren't handled explicitly.
TEST(AverageWordLength, EmptyInputReturnsZeroNotUB) {
    EXPECT_DOUBLE_EQ(textstats::AverageWordLength(""), 0.0);
}

TEST(AverageWordLength, WhitespaceOnlyReturnsZero) {
    EXPECT_DOUBLE_EQ(textstats::AverageWordLength("  \t "), 0.0);
}

TEST(AverageWordLength, SingleWord) {
    EXPECT_DOUBLE_EQ(textstats::AverageWordLength("hello"), 5.0);
}

// Use EXPECT_DOUBLE_EQ / EXPECT_NEAR for floating point -- never EXPECT_EQ.
TEST(AverageWordLength, MixedLengths) {
    // "a bb ccc" -> (1 + 2 + 3) / 3 == 2.0
    EXPECT_DOUBLE_EQ(textstats::AverageWordLength("a bb ccc"), 2.0);
}

TEST(AverageWordLength, NonTerminatingDecimal) {
    // "a bb" -> 3 / 2 == 1.5 exactly, but "a b cc" -> 4/3 is not exact
    EXPECT_NEAR(textstats::AverageWordLength("a b cc"), 1.3333333, 1e-6);
}
```

### tests/CMakeLists.txt

```cmake
include(FetchContent)
FetchContent_Declare(
  googletest
  URL https://github.com/google/googletest/archive/refs/tags/v1.15.2.zip
)
set(gtest_force_shared_crt ON CACHE BOOL "" FORCE)
FetchContent_MakeAvailable(googletest)

include(GoogleTest)

function(add_project_test name)
  add_executable(${name} ${ARGN})
  target_link_libraries(${name} PRIVATE textstats GTest::gtest_main)
  gtest_discover_tests(${name} PROPERTIES LABELS unit)
endfunction()

add_project_test(test_word_count      unit/test_word_count.cpp)
add_project_test(test_longest_word    unit/test_longest_word.cpp)
add_project_test(test_average_length  unit/test_average_length.cpp)
```

### CMakeLists.txt (top level)

```cmake
cmake_minimum_required(VERSION 3.14)
project(textstats CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

add_library(textstats src/textstats.cpp)
target_include_directories(textstats PUBLIC include)
target_compile_options(textstats PRIVATE -Wall -Wextra)

enable_testing()
add_subdirectory(tests)
```

## Running it

```bash
cmake -S . -B build -DCMAKE_BUILD_TYPE=Debug
cmake --build build
ctest --test-dir build --output-on-failure
```

```text
Test project /path/to/textstats/build
      Start  1: WordCount.EmptyStringHasNoWords
 1/15 Test  #1: WordCount.EmptyStringHasNoWords ..............   Passed    0.00 sec
      ...
15/15 Test #15: AverageWordLength.NonTerminatingDecimal ......   Passed    0.00 sec

100% tests passed, 0 tests failed out of 15
```

## Part 3 — close the loop

A test plan that never gets executed is decoration. Fill in the **Actual** and
**Status** columns of your test cases, then produce a short summary:

| Metric | Value |
|--------|-------|
| Cases planned | 12 |
| Cases automated | 15 |
| Cases executed manually | *(the exploratory charter findings)* |
| Defects found | *(list IDs)* |
| Exit criteria met? | Yes / No — with justification |

Note that "cases automated" can legitimately exceed "cases planned": writing
the code surfaces cases the plan missed. **Feed those back into the plan** —
that back-and-forth is the actual job.

## Stretch goals

- Introduce a deliberate bug (change `>` to `>=` in `LongestWord`) and confirm
  exactly one test fails, and that its name tells you what broke without
  opening the file. If it doesn't, your test names are too vague.
- Add a `Tokenize` test suite of its own — it's the shared dependency, so a bug
  there fails everything downstream. Which is why testing it directly is worth
  it.
- Run `ctest --repeat until-fail:20` to prove none of your tests depend on
  execution order.
- Add `-fsanitize=address,undefined` to a Debug build and re-run. Level 2
  covers sanitizers properly, but seeing a clean run now is a good baseline.

Completing this project means you're ready for **Level 2 · Intermediate**.
