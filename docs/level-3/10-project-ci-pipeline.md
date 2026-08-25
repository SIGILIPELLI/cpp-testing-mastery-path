# 10 · Project — CI Pipeline with Sanitizers, Coverage and Fuzzing

This capstone assembles every technique from Level 3 into one real,
buildable project with a complete GitHub Actions pipeline: a small
length-prefixed message parser (the kind of code most worth fuzzing),
tested at the unit level, sanitized, measured for coverage, fuzzed, and
gated in CI — with the C portions actually compiled and run on this
machine.

!!! note "Environment note"
    The C parser, its unit tests, and the ThreadSanitizer-checked
    concurrent cache in section 4 **were compiled and run on this host** —
    their output below is real. The GoogleTest/CMake/GitHub Actions
    scaffolding follows the same broken-libc++/no-CI-runner limitations
    noted throughout this level and was checked structurally, not executed.

## The system under test

```text
msgparser/
    include/msgparser/parser.h
    src/parser.c
    src/cache.c
    include/msgparser/cache.h
    tests/
        test_parser.c
        test_cache_concurrency.c
    fuzz/
        fuzz_parser.c
        corpus/
        regressions/
    CMakeLists.txt
    .github/workflows/ci.yml
```

A length-prefixed message: `[1-byte type][2-byte big-endian length][payload]`.
Plus a small thread-safe LRU-ish cache (size-bounded, not full LRU) that
stores recently parsed message types, to give the concurrency section
something real to test.

### include/msgparser/parser.h

```c
#ifndef MSGPARSER_PARSER_H
#define MSGPARSER_PARSER_H
#include <stdint.h>
#include <stddef.h>

typedef struct {
    uint8_t type;
    uint16_t len;
    const uint8_t *payload;
} message_t;

/* Returns 0 and fills *out on success; -1 on any malformed input.
   Never reads past buf[0..buf_len). */
int parse_message(const uint8_t *buf, size_t buf_len, message_t *out);

#endif
```

### src/parser.c

```c
#include "msgparser/parser.h"

int parse_message(const uint8_t *buf, size_t buf_len, message_t *out) {
    if (buf == NULL || out == NULL) return -1;
    if (buf_len < 3) return -1;                       /* type + 2-byte length */
    uint16_t len = (uint16_t)((uint16_t)buf[1] << 8 | buf[2]);
    /* Guard against len+3 overflowing size_t on exotic platforms, and
       against a declared length that overruns the actual buffer. */
    if (len > buf_len - 3) return -1;
    out->type = buf[0];
    out->len = len;
    out->payload = buf + 3;
    return 0;
}
```

### tests/test_parser.c

```c
#include <assert.h>
#include <string.h>
#include "msgparser/parser.h"

static void test_rejects_short_buffer(void) {
    uint8_t buf[2] = {0x01, 0x00};
    message_t m;
    assert(parse_message(buf, sizeof(buf), &m) == -1);
}

static void test_rejects_overrunning_length(void) {
    uint8_t buf[3] = {0x01, 0xFF, 0xFF};   /* claims 65535-byte payload */
    message_t m;
    assert(parse_message(buf, sizeof(buf), &m) == -1);
}

static void test_parses_valid_message(void) {
    uint8_t buf[6] = {0x02, 0x00, 0x03, 'a', 'b', 'c'};
    message_t m;
    assert(parse_message(buf, sizeof(buf), &m) == 0);
    assert(m.type == 0x02);
    assert(m.len == 3);
    assert(memcmp(m.payload, "abc", 3) == 0);
}

static void test_rejects_null_inputs(void) {
    message_t m;
    assert(parse_message(NULL, 10, &m) == -1);
    uint8_t buf[3] = {0, 0, 0};
    assert(parse_message(buf, sizeof(buf), NULL) == -1);
}

static void test_accepts_zero_length_payload(void) {
    uint8_t buf[3] = {0x05, 0x00, 0x00};
    message_t m;
    assert(parse_message(buf, sizeof(buf), &m) == 0);
    assert(m.len == 0);
}

int main(void) {
    test_rejects_short_buffer();
    test_rejects_overrunning_length();
    test_parses_valid_message();
    test_rejects_null_inputs();
    test_accepts_zero_length_payload();
    return 0;
}
```

## Running the unit tests, with sanitizers, on this host

```bash
gcc -std=c11 -Wall -Wextra -Iinclude \
    -fsanitize=address,undefined -g \
    src/parser.c tests/test_parser.c -o test_parser
./test_parser && echo "ALL UNIT TESTS PASSED (clean under ASan+UBSan)"
```

```text
ALL UNIT TESTS PASSED (clean under ASan+UBSan)
```

## src/cache.c and include/msgparser/cache.h — the concurrency piece

```c
#ifndef MSGPARSER_CACHE_H
#define MSGPARSER_CACHE_H
#include <stdint.h>
#include <pthread.h>

#define CACHE_SIZE 8

typedef struct {
    uint8_t types[CACHE_SIZE];
    size_t next_slot;
    pthread_mutex_t lock;
} type_cache_t;

void cache_init(type_cache_t *c);
void cache_record(type_cache_t *c, uint8_t type);
int  cache_contains(type_cache_t *c, uint8_t type);

#endif
```

```c
#include "msgparser/cache.h"
#include <string.h>

void cache_init(type_cache_t *c) {
    memset(c->types, 0xFF, sizeof(c->types));
    c->next_slot = 0;
    pthread_mutex_init(&c->lock, NULL);
}

void cache_record(type_cache_t *c, uint8_t type) {
    pthread_mutex_lock(&c->lock);
    c->types[c->next_slot] = type;
    c->next_slot = (c->next_slot + 1) % CACHE_SIZE;
    pthread_mutex_unlock(&c->lock);
}

int cache_contains(type_cache_t *c, uint8_t type) {
    pthread_mutex_lock(&c->lock);
    int found = 0;
    for (size_t i = 0; i < CACHE_SIZE; ++i) {
        if (c->types[i] == type) { found = 1; break; }
    }
    pthread_mutex_unlock(&c->lock);
    return found;
}
```

### tests/test_cache_concurrency.c — TSan-checked, real output

```c
#include <assert.h>
#include <pthread.h>
#include "msgparser/cache.h"

static type_cache_t cache;

static void *recorder(void *arg) {
    uint8_t base = *(uint8_t *)arg;
    for (int i = 0; i < 1000; ++i) {
        cache_record(&cache, (uint8_t)(base + (i % 4)));
        cache_contains(&cache, base);
    }
    return NULL;
}

int main(void) {
    cache_init(&cache);
    pthread_t t1, t2;
    uint8_t a = 1, b = 100;
    pthread_create(&t1, NULL, recorder, &a);
    pthread_create(&t2, NULL, recorder, &b);
    pthread_join(t1, NULL);
    pthread_join(t2, NULL);
    /* No assertion on final contents -- it's a ring buffer, order is
       nondeterministic under concurrency by design. The property under
       test is the absence of a data race, which TSan checks, not a
       specific final state. */
    return 0;
}
```

```bash
gcc -fsanitize=thread -Iinclude src/cache.c tests/test_cache_concurrency.c -o test_cache_tsan -lpthread
./test_cache_tsan && echo "PASS: cache is race-free under TSan"
```

```text
PASS: cache is race-free under TSan
```

## fuzz/fuzz_parser.c

```c
#include <stdint.h>
#include <stddef.h>
#include "msgparser/parser.h"

int LLVMFuzzerTestOneInput(const uint8_t *data, size_t size) {
    message_t m;
    parse_message(data, size, &m);
    return 0;
}
```

As with Level 3 module 04, this host's Command Line Tools install is
missing the libFuzzer runtime (`libclang_rt.fuzzer_osx.a`), so this target
was not linked or run here — the CI workflow below runs it on a Linux
runner where the dependency is present via the distro's clang package.

## CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.16)
project(msgparser C)
set(CMAKE_C_STANDARD 11)

option(ENABLE_SANITIZERS "" OFF)
option(ENABLE_COVERAGE "" OFF)

add_library(msgparser src/parser.c src/cache.c)
target_include_directories(msgparser PUBLIC include)
target_compile_options(msgparser PRIVATE -Wall -Wextra)
find_package(Threads REQUIRED)
target_link_libraries(msgparser PUBLIC Threads::Threads)

if(ENABLE_SANITIZERS)
    target_compile_options(msgparser PUBLIC -fsanitize=address,undefined -g)
    target_link_options(msgparser PUBLIC -fsanitize=address,undefined)
endif()
if(ENABLE_COVERAGE)
    target_compile_options(msgparser PUBLIC --coverage -g -O0)
    target_link_options(msgparser PUBLIC --coverage)
endif()

enable_testing()
add_executable(test_parser tests/test_parser.c)
target_link_libraries(test_parser PRIVATE msgparser)
add_test(NAME parser COMMAND test_parser)

add_executable(test_cache_concurrency tests/test_cache_concurrency.c)
target_link_libraries(test_cache_concurrency PRIVATE msgparser)
add_test(NAME cache_concurrency COMMAND test_cache_concurrency)
```

## .github/workflows/ci.yml

```yaml
name: CI

on: [push, pull_request]

jobs:
  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: cmake -S . -B build
      - run: cmake --build build
      - run: ctest --test-dir build --output-on-failure

  sanitized-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: cmake -S . -B build-san -DENABLE_SANITIZERS=ON
      - run: cmake --build build-san
      - run: ctest --test-dir build-san --output-on-failure

  thread-sanitizer:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: gcc -fsanitize=thread -Iinclude src/cache.c tests/test_cache_concurrency.c -o test_cache_tsan -lpthread
      - run: ./test_cache_tsan

  coverage:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: sudo apt-get install -y lcov
      - run: cmake -S . -B build-cov -DENABLE_COVERAGE=ON
      - run: cmake --build build-cov
      - run: ctest --test-dir build-cov --output-on-failure
      - run: |
          lcov --capture --directory build-cov --output-file coverage.info --exclude '*/tests/*'
          lcov --summary coverage.info

  fuzz-smoke:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: clang -std=c11 -Iinclude -g -fsanitize=fuzzer,address,undefined src/parser.c fuzz/fuzz_parser.c -o fuzz_parser
      - run: ./fuzz_parser -max_total_time=60 fuzz/corpus/
      - run: for f in fuzz/regressions/*; do [ -e "$f" ] && ./fuzz_parser "$f"; done
```

## Running it end to end (what's real vs. structural)

```bash
# Real, run on this host:
gcc -std=c11 -Wall -Wextra -Iinclude -fsanitize=address,undefined -g \
    src/parser.c tests/test_parser.c -o test_parser
./test_parser
# -> ALL UNIT TESTS PASSED (clean under ASan+UBSan)

gcc -fsanitize=thread -Iinclude src/cache.c tests/test_cache_concurrency.c \
    -o test_cache_tsan -lpthread
./test_cache_tsan
# -> PASS: cache is race-free under TSan

# Structural only (needs libFuzzer runtime / a working CI runner):
# ./fuzz_parser -max_total_time=60 fuzz/corpus/
```

## Stretch goals

- Add a checksum byte to the message format (as suggested in module 04's
  exercise), extend `test_parser.c` with cases for a corrupted checksum,
  and confirm the ASan/UBSan run stays clean.
- Replace the ring-buffer cache with a real bounded LRU (needs a doubly
  linked list or an index array) and re-run the TSan concurrency test —
  a more complex data structure is exactly where a subtle race is more
  likely to hide.
- Wire the four jobs in `ci.yml` into GitHub branch protection as required
  checks for `unit-tests`, `sanitized-tests`, and `thread-sanitizer`,
  leaving `coverage` and `fuzz-smoke` non-blocking initially, per module
  06's guidance.
- Write a property-based test (module 05's hand-rolled style, since
  RapidCheck needs libc++) asserting that for any buffer, `parse_message`
  either returns -1 or returns a `message_t` whose `payload` pointer plus
  `len` never reads past `buf + buf_len`.

Completing this project means you're ready for **Level 4 · Master**.
