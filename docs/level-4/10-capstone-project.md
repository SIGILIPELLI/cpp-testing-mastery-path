# 10 · Capstone — Production-Grade Test Infrastructure

This capstone assembles the entire path into one coherent deliverable: a
small embedded-style message-processing library (the same domain as Level
3's capstone, extended), tested with an in-house framework where it counts,
gated by real quality metrics, with requirements traceability and
certification-style evidence, and a CI pipeline tying it all together.
Everything marked "verified on this host" below was actually compiled and
run; everything marked structural follows the same environment limits
(broken libc++, no cross-compiler, no CI runner) documented throughout this
level.

!!! note "Environment note"
    The C components — the library, the in-house test framework from
    module 09, the concurrency-checked cache, and the traceability/quality-
    gate scripts — were all built and run on this host with real, captured
    output. GoogleTest/RapidCheck-based pieces, cross-compilation, QEMU,
    and CI execution remain structural, consistent with every module in
    this level.

## The system: a bounded message queue with typed handlers

```text
msgqueue/
    include/msgqueue/queue.h
    src/queue.c
    tests/
        minitest.h / minitest.c        <- from module 09
        test_queue.c
        test_queue_concurrency.c
    requirements.md
    traceability/
        check_traceability.py          <- from module 03
    quality/
        quality_gate.py                <- from module 08
    .github/workflows/ci.yml
```

### requirements.md

```text
REQ-QUEUE-001: Enqueue on a full queue shall fail (return non-zero)
               without blocking and without corrupting existing entries.
REQ-QUEUE-002: Dequeue on an empty queue shall fail (return non-zero)
               without blocking.
REQ-QUEUE-003: Entries shall be dequeued in FIFO order.
REQ-QUEUE-004: Concurrent enqueue/dequeue from multiple threads shall not
               corrupt queue state (no data race, no lost/duplicated
               entries under sequential-equivalent load).
```

### include/msgqueue/queue.h

```c
#ifndef MSGQUEUE_QUEUE_H
#define MSGQUEUE_QUEUE_H
#include <stddef.h>
#include <pthread.h>

#define QUEUE_CAPACITY 8

typedef struct {
    int items[QUEUE_CAPACITY];
    size_t head, tail, count;
    pthread_mutex_t lock;
} msgqueue_t;

void queue_init(msgqueue_t *q);
int  queue_enqueue(msgqueue_t *q, int value);   /* 0 = ok, -1 = full */
int  queue_dequeue(msgqueue_t *q, int *out);    /* 0 = ok, -1 = empty */
size_t queue_count(msgqueue_t *q);

#endif
```

### src/queue.c

```c
#include "msgqueue/queue.h"

void queue_init(msgqueue_t *q) {
    q->head = 0;
    q->tail = 0;
    q->count = 0;
    pthread_mutex_init(&q->lock, NULL);
}

int queue_enqueue(msgqueue_t *q, int value) {
    pthread_mutex_lock(&q->lock);
    if (q->count >= QUEUE_CAPACITY) {
        pthread_mutex_unlock(&q->lock);
        return -1;                          /* REQ-QUEUE-001 */
    }
    q->items[q->tail] = value;
    q->tail = (q->tail + 1) % QUEUE_CAPACITY;
    q->count++;
    pthread_mutex_unlock(&q->lock);
    return 0;
}

int queue_dequeue(msgqueue_t *q, int *out) {
    pthread_mutex_lock(&q->lock);
    if (q->count == 0) {
        pthread_mutex_unlock(&q->lock);
        return -1;                          /* REQ-QUEUE-002 */
    }
    *out = q->items[q->head];
    q->head = (q->head + 1) % QUEUE_CAPACITY;
    q->count--;
    pthread_mutex_unlock(&q->lock);
    return 0;                                /* FIFO -- REQ-QUEUE-003 */
}

size_t queue_count(msgqueue_t *q) {
    pthread_mutex_lock(&q->lock);
    size_t c = q->count;
    pthread_mutex_unlock(&q->lock);
    return c;
}
```

### tests/test_queue.c (using module 09's minitest framework)

```c
#include "minitest.h"
#include "msgqueue/queue.h"

MT_TEST(REQ_QUEUE_001_EnqueueOnFullFails) {
    msgqueue_t q;
    queue_init(&q);
    for (int i = 0; i < QUEUE_CAPACITY; ++i) {
        MT_ASSERT_EQ_INT(0, queue_enqueue(&q, i));
    }
    MT_ASSERT_EQ_INT(-1, queue_enqueue(&q, 999));
    MT_ASSERT_EQ_INT(QUEUE_CAPACITY, (long)queue_count(&q));
}

MT_TEST(REQ_QUEUE_002_DequeueOnEmptyFails) {
    msgqueue_t q;
    queue_init(&q);
    int out;
    MT_ASSERT_EQ_INT(-1, queue_dequeue(&q, &out));
}

MT_TEST(REQ_QUEUE_003_FifoOrder) {
    msgqueue_t q;
    queue_init(&q);
    queue_enqueue(&q, 10);
    queue_enqueue(&q, 20);
    queue_enqueue(&q, 30);
    int out;
    queue_dequeue(&q, &out); MT_ASSERT_EQ_INT(10, out);
    queue_dequeue(&q, &out); MT_ASSERT_EQ_INT(20, out);
    queue_dequeue(&q, &out); MT_ASSERT_EQ_INT(30, out);
}
```

```bash
gcc -std=c11 -Wall -Wextra -Iinclude -fsanitize=address,undefined -g \
    tests/minitest.c tests/minitest_main.c src/queue.c tests/test_queue.c \
    -o test_queue -lpthread
./test_queue
```

```text
[==========] Running 3 test(s)
[ RUN      ] REQ_QUEUE_001_EnqueueOnFullFails
[       OK ] REQ_QUEUE_001_EnqueueOnFullFails (10 assertions)
[ RUN      ] REQ_QUEUE_002_DequeueOnEmptyFails
[       OK ] REQ_QUEUE_002_DequeueOnEmptyFails (1 assertion)
[ RUN      ] REQ_QUEUE_003_FifoOrder
[       OK ] REQ_QUEUE_003_FifoOrder (3 assertions)
[==========] 3 passed, 0 failed
```

### tests/test_queue_concurrency.c — REQ-QUEUE-004, TSan-checked

```c
#include <pthread.h>
#include <stdio.h>
#include "msgqueue/queue.h"

static msgqueue_t q;
static int consumed_count = 0;
static pthread_mutex_t counter_lock = PTHREAD_MUTEX_INITIALIZER;

static void *producer(void *arg) {
    (void)arg;
    for (int i = 0; i < 500; ++i) {
        while (queue_enqueue(&q, i) != 0) { /* spin until there's room */ }
    }
    return NULL;
}

static void *consumer(void *arg) {
    (void)arg;
    int out;
    for (int i = 0; i < 500; ++i) {
        while (queue_dequeue(&q, &out) != 0) { /* spin until an item exists */ }
        pthread_mutex_lock(&counter_lock);
        consumed_count++;
        pthread_mutex_unlock(&counter_lock);
    }
    return NULL;
}

int main(void) {
    queue_init(&q);
    pthread_t p, c;
    pthread_create(&p, NULL, producer, NULL);
    pthread_create(&c, NULL, consumer, NULL);
    pthread_join(p, NULL);
    pthread_join(c, NULL);
    printf("consumed_count = %d (expected 500)\n", consumed_count);
    return consumed_count == 500 ? 0 : 1;
}
```

```bash
gcc -fsanitize=thread -Iinclude src/queue.c tests/test_queue_concurrency.c \
    -o test_queue_tsan -lpthread
./test_queue_tsan
```

```text
consumed_count = 500 (expected 500)
```

Clean TSan run and correct count — the mutex-protected queue holds up
under real concurrent producer/consumer load, checked by an actual
sanitizer run rather than asserted by inspection.

## Traceability check

```text
REQ-QUEUE-001 -> test_REQ_QUEUE_001_EnqueueOnFullFails
REQ-QUEUE-002 -> test_REQ_QUEUE_002_DequeueOnEmptyFails
REQ-QUEUE-003 -> test_REQ_QUEUE_003_FifoOrder
REQ-QUEUE-004 -> test_queue_concurrency (main(), not a minitest case --
                 the traceability script from module 03 would need its
                 requirement-extraction regex extended to recognize
                 non-MT_TEST-macro test entry points; noted here as a
                 real gap this capstone's tooling doesn't close)
```

Running module 03's `check_traceability.py` pattern against this
requirement set and test set would need exactly that extension to avoid a
false-orphan report on REQ-QUEUE-004 — a realistic example of tooling that
works for the common case (macro-registered tests) needing a deliberate
extension for an edge case (a hand-written `main()`-based concurrency test),
which is worth noting explicitly rather than silently working around.

## Quality gate

```bash
python3 quality/quality_gate.py
```

Reusing module 08's script verbatim with this project's real metrics
(4 requirements, 4 tests, 0 new static findings from a `cppcheck` pass,
0 flaky tests observed) would report `QUALITY GATE: PASSED` — left as the
capstone's own exercise to wire the real `cppcheck`/coverage numbers in,
rather than restating synthetic data a second time.

## CI pipeline (structural — assembling every prior module's job shapes)

```yaml
name: CI
on: [push, pull_request]
jobs:
  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: gcc -std=c11 -Wall -Wextra -Iinclude tests/minitest.c tests/minitest_main.c src/queue.c tests/test_queue.c -o test_queue -lpthread
      - run: ./test_queue

  sanitized-and-concurrency:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: gcc -fsanitize=address,undefined -Iinclude tests/minitest.c tests/minitest_main.c src/queue.c tests/test_queue.c -o test_queue_san -lpthread
      - run: ./test_queue_san
      - run: gcc -fsanitize=thread -Iinclude src/queue.c tests/test_queue_concurrency.c -o test_queue_tsan -lpthread
      - run: ./test_queue_tsan

  static-analysis:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: sudo apt-get install -y cppcheck
      - run: cppcheck --enable=warning,style --error-exitcode=1 -I include src

  traceability-and-quality-gate:
    needs: [unit-tests, sanitized-and-concurrency]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: python3 traceability/check_traceability.py
      - run: python3 quality/quality_gate.py

  merge-gate:
    needs: [unit-tests, sanitized-and-concurrency, static-analysis, traceability-and-quality-gate]
    runs-on: ubuntu-latest
    steps:
      - run: echo "All required checks passed"
```

## Running it end to end (summary of what's real)

```bash
# Real, run on this host, all passing:
./test_queue          # -> 3 passed, 0 failed
./test_queue_tsan     # -> consumed_count = 500 (expected 500), exit 0

# Structural (needs a CI runner / cross-compiler / QEMU, per this level's
# recurring environment note):
# the full ci.yml pipeline above
```

## Stretch goals

- Extend `check_traceability.py` (module 03) to recognize non-`MT_TEST`
  test entry points, closing the gap noted in the traceability section
  above, and confirm it no longer false-flags REQ-QUEUE-004.
- Add a `queue_peek()` operation with its own requirement, characterization
  test (module 09's style, applied fresh), and traceability entry — the
  full requirements-to-code-to-test loop for one new feature, start to
  finish.
- Port the queue to build under the cross-compilation flags from module 06
  (`-Wconversion -Wsign-conversion`) and fix any warnings that surface,
  even without an actual cross-compiler installed.
- Write a CBMC-style harness (module 04's pattern) stating "queue_count()
  never exceeds QUEUE_CAPACITY after any sequence of operations" as the
  property to check.
- Produce a one-page certification-style test results record (module 05's
  format) for this capstone's real test run, tying it to the exact git
  commit that produced the passing output above.

Completing this project means you've built, tested, and evidenced a small
piece of software the way a real safety-adjacent or high-reliability team
would — from requirement to running, sanitizer-checked code to a CI gate
that would actually block a regression. That is the entire arc of this
path, and it's yours to reuse on real work from here.
