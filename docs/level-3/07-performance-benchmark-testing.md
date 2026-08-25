# 07 · Performance & Benchmark Testing

Correctness tests answer "does it give the right answer." Benchmarks answer
"does it give the right answer fast enough, and did the last commit make it
slower." Without a benchmark suite, performance regressions are invisible
until a user complains — by which point nobody remembers which of the last
fifty commits caused it.

!!! note "Environment note"
    Google Benchmark is not installed on this machine (not present via
    Homebrew here) and the C++ toolchain has the broken-libc++ issue noted
    throughout this level, so nothing in this module was compiled or run.
    Every listing was checked for correctness by manual trace against
    Google Benchmark's documented API and general algorithmic reasoning
    about complexity, not measurement. Treat the numbers in this module as
    illustrative shapes of output, not real measurements — install
    `google-benchmark` and build with a working toolchain to get real
    numbers on your machine.

## 1. A first benchmark

```cpp
#include <benchmark/benchmark.h>
#include <vector>
#include <algorithm>

static void BM_SortRandomVector(benchmark::State& state) {
    const size_t n = state.range(0);
    for (auto _ : state) {
        state.PauseTiming();
        std::vector<int> v(n);
        for (size_t i = 0; i < n; ++i) v[i] = static_cast<int>(n - i);
        state.ResumeTiming();

        std::sort(v.begin(), v.end());
        benchmark::DoNotOptimize(v.data());
    }
    state.SetComplexityN(n);
}
BENCHMARK(BM_SortRandomVector)->Range(1 << 8, 1 << 16)->Complexity();

BENCHMARK_MAIN();
```

```bash
g++ -std=c++17 -O2 bench_sort.cpp -lbenchmark -lpthread -o bench_sort
./bench_sort
```

Illustrative shape of the output (real numbers depend entirely on the
machine running it):

```text
------------------------------------------------------------------
Benchmark                        Time             CPU   Iterations
------------------------------------------------------------------
BM_SortRandomVector/256         1834 ns         1830 ns       380000
BM_SortRandomVector/1024        9102 ns         9080 ns        76000
BM_SortRandomVector/4096       46310 ns        46200 ns        15000
BM_SortRandomVector/16384     221400 ns       220900 ns         3100
BM_SortRandomVector/65536    1089000 ns      1086000 ns          640
BM_SortRandomVector_BigO       15.84 NlgN      15.80 NlgN
BM_SortRandomVector_RMS             3 %             3 %
```

`state.PauseTiming()`/`ResumeTiming()` matters as much as the measured code
itself: without excluding vector construction from the timed region, you'd
be benchmarking allocation and initialization, not the sort.

## 2. Comparing implementations head-to-head

The highest-value use of a benchmark suite is not an absolute number — it's
a comparison that survives being run on a laptop with the fan spinning at a
different speed than last time.

```cpp
static void BM_StdSort(benchmark::State& state) {
    std::vector<int> base(state.range(0));
    std::iota(base.rbegin(), base.rend(), 0);
    for (auto _ : state) {
        auto v = base;
        std::sort(v.begin(), v.end());
        benchmark::DoNotOptimize(v.data());
    }
}
BENCHMARK(BM_StdSort)->Arg(10000);

static void BM_InsertionSort(benchmark::State& state) {
    std::vector<int> base(state.range(0));
    std::iota(base.rbegin(), base.rend(), 0);
    for (auto _ : state) {
        auto v = base;
        InsertionSort(v);   // O(n^2) -- from module 05
        benchmark::DoNotOptimize(v.data());
    }
}
BENCHMARK(BM_InsertionSort)->Arg(10000);

BENCHMARK_MAIN();
```

```text
BM_StdSort/10000            118000 ns       117500 ns         5900
BM_InsertionSort/10000    41200000 ns     41100000 ns           17
```

A ~350x gap at n=10000 is exactly the kind of number that turns "insertion
sort is fine for small inputs" from a guess into a documented, re-checkable
claim — and the benchmark that produced it is the thing that catches a
regression if someone "optimizes" `InsertionSort` into something worse.

## 3. Fixtures for benchmarks that need setup

Same idea as GoogleTest fixtures (Level 2, module 01), applied to
benchmarks — setup that shouldn't be part of the timed region and shouldn't
be re-typed per benchmark.

```cpp
class SortFixture : public benchmark::Fixture {
public:
    void SetUp(const benchmark::State& state) override {
        data.resize(state.range(0));
        std::mt19937 rng(42);   // fixed seed: comparable runs, not fair coins
        std::uniform_int_distribution<int> dist(-1000000, 1000000);
        for (auto& x : data) x = dist(rng);
    }
    std::vector<int> data;
};

BENCHMARK_F(SortFixture, StdSort)(benchmark::State& state) {
    for (auto _ : state) {
        auto v = data;
        std::sort(v.begin(), v.end());
        benchmark::DoNotOptimize(v.data());
    }
}
```

The fixed RNG seed is deliberate: a benchmark comparing two runs of the same
code should vary only in the code, not in which random data each run
happened to draw.

## 4. Regression detection in CI

A benchmark number without a baseline to compare against is trivia. Google
Benchmark's JSON output plus a comparison tool turns it into a gate.

```bash
./bench_sort --benchmark_format=json --benchmark_out=current.json
# compare.py ships with Google Benchmark's tools/ directory
python3 compare.py benchmarks baseline.json current.json
```

```text
Comparing baseline.json to current.json
Benchmark                          Time             CPU      Time Old      Time New
----------------------------------------------------------------------------------
BM_SortRandomVector/16384        +18.2%          +18.0%        221400        261700
```

An 18% regression on a specific size, flagged in a PR comment, is the
performance-testing equivalent of a red unit test — except without a
benchmark step in CI, this would have shipped silently.

```yaml
  benchmark:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: sudo apt-get install -y libbenchmark-dev
      - run: g++ -O2 -std=c++17 bench_sort.cpp -lbenchmark -lpthread -o bench_sort
      - run: ./bench_sort --benchmark_format=json --benchmark_out=current.json
      - run: python3 tools/compare.py benchmarks baseline.json current.json
```

## 5. Noise, warmup, and what "statistically significant" means here

Benchmark numbers are noisier than test pass/fail, and treating a single run
as ground truth invites chasing phantom regressions caused by a busy CI
runner, not a real code change.

| Noise source | Mitigation |
|---|---|
| CPU frequency scaling / thermal throttling | Run on a dedicated or pinned-frequency runner; note variance, don't chase single-digit % swings |
| First-run cache-cold effects | `--benchmark_min_time` large enough for several iterations before trusting the number |
| Other processes on a shared CI runner | Use `--benchmark_repetitions=5 --benchmark_report_aggregates_only=true` and look at the median, not one sample |
| Compiler over-optimizing away the "work" | `benchmark::DoNotOptimize()` on results, `benchmark::ClobberMemory()` when memory-visible side effects matter |

```cpp
BENCHMARK(BM_SortRandomVector)
    ->Range(1 << 8, 1 << 16)
    ->Repetitions(5)
    ->ReportAggregatesOnly(true);
```

## 6. Complexity, not just speed

`->Complexity()` (section 1) fits your measured points against known growth
curves (O(N), O(N log N), O(N²), …) and reports the best fit plus the
residual mean-square error (RMS) — a low RMS on an O(N log N) fit is
evidence your algorithm's asymptotic behavior matches expectation, which a
single timing number at one size cannot show.

```cpp
BENCHMARK(BM_LinearSearch)->RangeMultiplier(2)->Range(1<<10, 1<<20)->Complexity(benchmark::oN);
BENCHMARK(BM_BinarySearch)->RangeMultiplier(2)->Range(1<<10, 1<<20)->Complexity(benchmark::oLogN);
```

If `BM_BinarySearch`'s fit comes back closer to O(N) than O(log N), that's a
strong signal the "binary" search has a linear-time bug hiding in it (e.g.
scanning instead of properly bisecting) — a correctness bug a functional
test might miss entirely if it only checks the returned value, not the
number of comparisons made to get there.

## Cheat sheet

| Question | Tool/technique |
|---|---|
| Is this fast enough in isolation? | A single `BENCHMARK` with a realistic input size |
| Is A faster than B? | Two benchmarks over the same input, same fixture |
| Did the last commit regress performance? | JSON output + `compare.py` in CI against a stored baseline |
| Is the measurement trustworthy? | `--benchmark_repetitions`, aggregate reporting, watch for throttling |
| Does the growth rate match the algorithm's claimed complexity? | `->Complexity()` with multiple range points |

## Exercise

1. Write a `BENCHMARK_F` fixture benchmarking three implementations of
   `LongestWord` from Level 1's capstone project at input sizes 100, 10000,
   and 1000000 words, and predict (before running, since your environment
   may also lack Google Benchmark) which will scale worst and why.
2. Add `->Complexity()` to a benchmark of a binary search you write, and
   describe what result would make you suspicious the implementation isn't
   actually O(log N).
3. Sketch the CI job from section 4 for your own project, including where
   `baseline.json` would be stored (committed to the repo? a separate
   artifact store?) and who/what updates it when a deliberate, accepted
   slowdown ships.
4. Write two sentences on one place in your own code where you *believe*
   an optimization helped, and what benchmark you'd need to write to turn
   that belief into a checked, re-verifiable fact.
5. Identify one function in your project where correctness tests would pass
   even if performance regressed catastrophically (e.g. from O(log N) to
   O(N)), and describe the benchmark that would catch it.
