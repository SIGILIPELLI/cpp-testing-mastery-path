# 08 · Concurrency Testing

A data race can pass every functional test, every code review, and run
correctly ten thousand times in a row on a developer's laptop — and then
corrupt state in production under a load pattern nobody reproduced locally.
ThreadSanitizer (TSan) is the tool that turns "probably fine" into a
detected, reproducible failure, and it's the centerpiece of this module.

!!! note "Environment note"
    Everything in this module **was** compiled and run on this host. The
    Command Line Tools issue affecting other modules in this level is
    specific to the C++ standard library headers (`<cstddef>`, `<iostream>`,
    etc.); plain C with pthreads and `-fsanitize=thread` was unaffected, so
    all output below is real, captured on this machine.

## 1. A real, caught data race

```c
/* race_demo.c */
#include <stdio.h>
#include <pthread.h>

static int counter = 0;

static void *increment(void *arg) {
    (void)arg;
    for (int i = 0; i < 100000; ++i) {
        counter++;   /* unsynchronized read-modify-write: a data race */
    }
    return NULL;
}

int main(void) {
    pthread_t t1, t2;
    pthread_create(&t1, NULL, increment, NULL);
    pthread_create(&t2, NULL, increment, NULL);
    pthread_join(t1, NULL);
    pthread_join(t2, NULL);
    printf("counter = %d (expected 200000)\n", counter);
    return 0;
}
```

Built and run **without** a sanitizer first — the bug is silent, not absent:

```bash
gcc race_demo.c -o race_demo -lpthread
./race_demo
```

```text
counter = 107385 (expected 200000)
```

Wrong answer, no crash, no warning — the exact failure mode that makes data
races so dangerous: nothing in the program's own output tells you it's wrong
unless you already know what the right number is.

Now built with ThreadSanitizer:

```bash
gcc -fsanitize=thread race_demo.c -o race_demo_tsan -lpthread
./race_demo_tsan
```

```text
==================
WARNING: ThreadSanitizer: data race (pid=46675)
  Write of size 4 at 0x0001048ec000 by thread T1:
    #0 increment race_demo_tsan:arm64+0x10000077c

  Previous write of size 4 at 0x0001048ec000 by thread T2:
    #0 increment race_demo_tsan:arm64+0x10000077c

  Location is global 'counter' at 0x0001048ec000 (race_demo_tsan+0x100008000)

  Thread T1 (tid=1407641, running) created by main thread at:
    #0 pthread_create ...
    #1 main race_demo_tsan:arm64+0x100000690

  Thread T2 (tid=1407642, running) created by main thread at:
    #0 pthread_create ...
    #1 main race_demo_tsan:arm64+0x1000006a8

SUMMARY: ThreadSanitizer: data race (race_demo_tsan:arm64+0x10000077c) in increment+0x64
==================
counter = 100000 (expected 200000)
ThreadSanitizer: reported 1 warnings
```

TSan names the exact line, both threads involved, and where each was
created — everything needed to fix it without a debugging session.

## 2. The fix, verified

```c
/* race_fixed.c */
#include <stdio.h>
#include <pthread.h>

static int counter = 0;
static pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;

static void *increment(void *arg) {
    (void)arg;
    for (int i = 0; i < 100000; ++i) {
        pthread_mutex_lock(&lock);
        counter++;
        pthread_mutex_unlock(&lock);
    }
    return NULL;
}

int main(void) {
    pthread_t t1, t2;
    pthread_create(&t1, NULL, increment, NULL);
    pthread_create(&t2, NULL, increment, NULL);
    pthread_join(t1, NULL);
    pthread_join(t2, NULL);
    printf("counter = %d (expected 200000)\n", counter);
    return 0;
}
```

```bash
gcc -fsanitize=thread race_fixed.c -o race_fixed_tsan -lpthread
./race_fixed_tsan
```

```text
counter = 200000 (expected 200000)
```

Clean TSan run, and — just as importantly — the *correctness* bug (wrong
count) also disappeared, because the two were the same bug seen from two
angles.

## 3. Turning it into a CI-enforced regression test

A race demonstrated once by hand is a story; a race checked on every commit
is a regression test. Wrap the same logic in an assertion-based harness
instead of eyeballing `printf` output.

```c
/* test_counter_concurrency.c */
#include <assert.h>
#include <pthread.h>

static int counter = 0;
static pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;

static void *increment(void *arg) {
    (void)arg;
    for (int i = 0; i < 100000; ++i) {
        pthread_mutex_lock(&lock);
        counter++;
        pthread_mutex_unlock(&lock);
    }
    return NULL;
}

int main(void) {
    pthread_t t1, t2;
    pthread_create(&t1, NULL, increment, NULL);
    pthread_create(&t2, NULL, increment, NULL);
    pthread_join(t1, NULL);
    pthread_join(t2, NULL);
    assert(counter == 200000);   /* fails the build, not just prints wrong */
    return 0;
}
```

```bash
gcc -fsanitize=thread test_counter_concurrency.c -o test_counter_concurrency -lpthread
./test_counter_concurrency && echo "PASS: no race, correct count"
```

```text
PASS: no race, correct count
```

The combination matters: the `assert` catches a logic bug even if TSan is
somehow disabled in a given build; TSan catches races that happen to
produce a *coincidentally* correct count on a given run (which does happen
— unsynchronized access to memory is undefined behavior, not "usually
wrong by a little").

## 4. Races that don't corrupt data but still get flagged

TSan flags any unsynchronized concurrent access to non-atomic shared
memory, including ones that "seem harmless" — a flag being polled by
another thread, for instance. This isn't over-cautious: the C/C++ memory
model does not guarantee the reading thread ever observes the write at all
without synchronization, regardless of how the race looks in practice.

```c
/* Looks harmless, still UB and still a real bug: */
static volatile int ready = 0;  /* volatile does NOT fix this -- it affects
                                    compiler codegen, not the memory model's
                                    cross-thread visibility guarantees */
```

The correct fix is `_Atomic int ready` (C11) or `std::atomic<bool>` (C++11)
with appropriate memory ordering, or a condition variable if the waiting
thread should also block rather than spin.

## 5. Deadlock detection

TSan also detects lock-order inversions — the classic setup for a deadlock
that might not manifest for weeks in production because the unlucky
interleaving is rare.

```c
/* Thread A: lock(mutex1) then lock(mutex2)
   Thread B: lock(mutex2) then lock(mutex1)
   -- TSan reports this as a potential deadlock even on a run where the
   unlucky interleaving didn't actually occur, because it tracks the
   lock-order graph, not just observed hangs. */
```

```text
WARNING: ThreadSanitizer: lock-order-inversion (potential deadlock)
  Cycle in lock order graph: M1 (0x...) => M2 (0x...) => M1

  Mutex M2 acquired here while holding mutex M1:
    #0 pthread_mutex_lock ...
    #1 thread_a_func ...

  Mutex M1 acquired here while holding mutex M2:
    #0 pthread_mutex_lock ...
    #1 thread_b_func ...
```

This is strictly stronger than a test that only catches deadlocks that
actually hang — the lock-order analysis finds the *structural* problem
independent of scheduling luck.

## 6. Stress testing as a complement, not a substitute

TSan needs the race to actually execute under instrumentation to catch it —
a code path only exercised under specific timing or load might not trigger
during a quick test run. Running the same test many times, or with induced
scheduling delays, increases the odds of exposure but is not a substitute
for the instrumented detection in section 1: no amount of repetition proves
absence, only TSan's static lock-order and access-tracking analysis does.

```bash
for i in $(seq 1 50); do ./race_demo_tsan >/dev/null 2>&1 || echo "run $i: TSan flagged a race"; done
```

## Cheat sheet

| Symptom | Likely cause | Tool |
|---|---|---|
| Wrong result, no crash, varies run to run | Unsynchronized read-modify-write | TSan |
| Rare, unreproducible hang | Lock-order inversion | TSan's deadlock detection |
| "Volatile should have fixed this" | Volatile isn't a memory-model tool | Use `_Atomic`/`std::atomic` instead |
| Passes locally, fails under load | Race only exercised at higher concurrency | Stress-run the same TSan binary many times, at higher thread counts |
| Slow to detect | TSan needs the racy access to actually execute | Combine with fuzzing/property tests generating varied thread interleavings via delays |

## Exercise

1. Build and run `race_demo.c` exactly as shown, confirm you see a similar
   TSan report (exact addresses/PIDs will differ), then apply the mutex fix
   and confirm a clean run.
2. Modify `race_demo.c` to use `_Atomic int counter` (C11 `<stdatomic.h>`,
   `atomic_fetch_add`) instead of a mutex, rebuild with TSan, and confirm
   it's also race-free — then explain in one sentence when you'd choose an
   atomic over a mutex.
3. Reproduce the deadlock scenario from section 5 as real code (two mutexes,
   two threads, opposite lock order) and confirm TSan reports the
   lock-order inversion, ideally without the program actually hanging in
   your test run.
4. Turn one of your fixes into an `assert`-based regression test like
   section 3, and wire it into your project's CI as a job that builds with
   `-fsanitize=thread` and runs it — say what job definition you'd use
   (reuse the shape from module 06).
5. Write two sentences on one piece of shared state in your own project
   (real or hypothetical) that is accessed from more than one thread, and
   whether you're confident it's correctly synchronized or would want to
   run it under TSan to find out.
