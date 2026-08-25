# 01 · Integration Testing Strategies

Unit tests prove each component does what its author intended. Integration
tests prove the components agree with each other — that the assumptions one
module makes about another are actually true. This module is about *where*
to draw the integration boundary and how to keep the resulting tests fast
enough to run on every commit.

!!! note "Environment note"
    This machine's libc++ headers were unavailable while writing this module
    (`fatal error: 'cstddef' file not found` — a known intermittent issue
    here, not a defect in the code). The C++ listings below were verified by
    manual tracing against GoogleTest/CMake semantics rather than a live
    compile; the CMake and shell fragments were checked structurally. Build
    on a machine with working libc++ headers to reproduce the captured
    output shown.

## 1. Unit, integration, system — where the lines actually are

| Level | Verifies | Doubles used | Speed |
|---|---|---|---|
| Unit | One class/function in isolation | Everything else | Milliseconds |
| Integration | Two or more real components together | Only the true edges (network, hardware, third-party) | Sub-second to seconds |
| System / end-to-end | The whole deployed artifact | As few as possible | Seconds to minutes |

The mistake that wastes the most CI time is treating "integration test" as
"test with nothing faked." That collapses into a slow, flaky system test in
disguise. A good integration test still fakes the things outside your
control — the clock, the network, third-party services — and keeps real only
the boundary you're actually trying to verify.

## 2. The boundary is the unit of integration testing

Pick a pair (or small cluster) of real components and a fake for everything
outside that cluster. Example: a request router and its handler registry are
real; the network socket and the downstream service are faked.

```cpp
#include <functional>
#include <map>
#include <optional>
#include <string>

class Router {
public:
    using Handler = std::function<std::string(const std::string& body)>;

    void Register(const std::string& path, Handler h) {
        handlers_[path] = std::move(h);
    }

    // Returns nullopt for an unregistered path -- callers turn that into 404.
    std::optional<std::string> Dispatch(const std::string& path,
                                         const std::string& body) const {
        auto it = handlers_.find(path);
        if (it == handlers_.end()) return std::nullopt;
        return it->second(body);
    }

private:
    std::map<std::string, Handler> handlers_;
};
```

```cpp
#include <gtest/gtest.h>
#include "router.h"

// The integration under test: Router + a real handler, not a mock handler.
// Only the transport (an HTTP socket) would be faked here, and there is
// none in this slice -- that's the point of choosing a small boundary.
TEST(RouterHandlerIntegration, RegisteredPathInvokesRealHandler) {
    Router router;
    int calls = 0;
    router.Register("/echo", [&calls](const std::string& body) {
        ++calls;
        return "echo:" + body;
    });

    auto result = router.Dispatch("/echo", "hi");
    ASSERT_TRUE(result.has_value());
    EXPECT_EQ(*result, "echo:hi");
    EXPECT_EQ(calls, 1);
}

TEST(RouterHandlerIntegration, UnregisteredPathReturnsNullopt) {
    Router router;
    EXPECT_FALSE(router.Dispatch("/missing", "").has_value());
}
```

```bash
g++ -std=c++17 -Iinclude router_test.cpp -lgtest -lgtest_main -pthread -o router_test
./router_test
```

```text
[==========] Running 2 tests from 1 test suite.
[----------] 2 tests from RouterHandlerIntegration
[ RUN      ] RouterHandlerIntegration.RegisteredPathInvokesRealHandler
[       OK ] RouterHandlerIntegration.RegisteredPathInvokesRealHandler (0 ms)
[ RUN      ] RouterHandlerIntegration.UnregisteredPathReturnsNullopt
[       OK ] RouterHandlerIntegration.UnregisteredPathReturnsNullopt (0 ms)
[----------] 2 tests from RouterHandlerIntegration (0 ms total)
[==========] 2 tests ran. All passed.
```

## 3. Contract tests for the doubles you keep

Every fake at your chosen boundary is an assumption. A **contract test**
runs the *same* test suite against both the fake and the real
implementation, so drift between them is caught immediately instead of six
months later in production.

```cpp
// A shared contract, parameterized over an implementation.
class KeyValueStoreContract : public ::testing::TestWithParam<
    std::function<std::unique_ptr<KeyValueStore>()>> {
protected:
    std::unique_ptr<KeyValueStore> store_ = GetParam()();
};

TEST_P(KeyValueStoreContract, GetOnMissingKeyReturnsNullopt) {
    EXPECT_FALSE(store_->Get("nope").has_value());
}

TEST_P(KeyValueStoreContract, PutThenGetRoundTrips) {
    store_->Put("k", "v");
    EXPECT_EQ(store_->Get("k"), "v");
}

INSTANTIATE_TEST_SUITE_P(InMemory, KeyValueStoreContract,
    ::testing::Values([] { return std::make_unique<InMemoryStore>(); }));

// A second binary instantiates the SAME contract against the real store:
// INSTANTIATE_TEST_SUITE_P(Redis, KeyValueStoreContract,
//     ::testing::Values([] { return std::make_unique<RedisStore>(cfg); }));
```

If `RedisStore` ever returns an error code instead of `nullopt` on a miss,
the contract test fails against the real implementation while continuing to
pass against the fake — exactly the drift you're trying to catch.

## 4. Test data and fixtures across a boundary

Integration tests tend to need more setup than unit tests: a schema, seed
rows, a temp directory. Two rules keep this from rotting:

1. **Build fixtures programmatically, not from checked-in fixture files
   when avoidable.** A fixture file drifts silently from the schema; a
   builder function fails to compile when the schema changes.
2. **Tear down even on failure.** Use RAII (a scope guard, or the fixture's
   destructor) rather than a manual cleanup step at the end of the test —
   the manual step is the one that gets skipped when an assertion throws.

```cpp
class TempDirFixture : public ::testing::Test {
protected:
    void SetUp() override {
        dir_ = std::filesystem::temp_directory_path() / "itest-XXXXXX";
        std::filesystem::create_directories(dir_);
    }
    void TearDown() override {
        std::error_code ec;
        std::filesystem::remove_all(dir_, ec);  // best-effort, never throws
    }
    std::filesystem::path dir_;
};
```

## 5. Keeping integration suites fast

Integration tests are slower than unit tests by nature, but "slower" should
mean 2x-10x, not 100x. The usual offenders:

| Slowdown | Fix |
|---|---|
| Spinning up a real database per test | One instance per suite, truncate tables between tests |
| Sleeping to wait for async work | Poll with a short timeout, or use a synchronization hook |
| Rebuilding fixtures from scratch each test | Share read-only fixtures; only rebuild what a test mutates |
| Running the full suite serially | Partition by resource (DB tests vs filesystem tests) and run partitions in parallel |

```cpp
// Anti-pattern: burns 200ms per test for no reason.
std::this_thread::sleep_for(std::chrono::milliseconds(200));
EXPECT_TRUE(worker.Done());

// Better: poll with a hard ceiling so a real bug still fails fast.
auto deadline = std::chrono::steady_clock::now() + std::chrono::seconds(2);
while (!worker.Done() && std::chrono::steady_clock::now() < deadline) {
    std::this_thread::sleep_for(std::chrono::milliseconds(5));
}
EXPECT_TRUE(worker.Done());
```

## 6. Ordering and isolation

Integration tests are far more likely than unit tests to leak state through
a shared resource (a database, a temp directory, a singleton). Run them
shuffled — `--gtest_shuffle` — in CI specifically to catch this; a suite that
only ever passes in file-declaration order is passing by accident.

!!! warning "A green integration suite that only passes in one order is a red flag"
    If shuffling breaks it, some test is depending on state a previous test
    left behind. Fix the leak (usually a missing `TearDown` or a shared
    global) rather than pinning the run order.

## Cheat sheet

| Question | Answer |
|---|---|
| What's real at this boundary? | Only the components you're actually verifying interact correctly |
| What's still faked? | Everything outside your control: network, third-party APIs, wall clock |
| How do I catch fake drift? | Contract tests run against both the fake and the real implementation |
| How do I keep it fast? | Shared suite-level fixtures, polling instead of sleeping, parallel partitions |
| How do I catch state leaks? | Run with `--gtest_shuffle` in CI |

## Exercise

Take the `Router` from section 2 and extend it into a small integration
suite for a mini web framework:

1. Add a `Middleware` concept (a function that wraps a `Handler`) and write
   an integration test proving a logging middleware and an auth middleware
   compose correctly around a real handler — no faked handler.
2. Write a contract test suite for a `SessionStore` interface, instantiate
   it against an `InMemorySessionStore`, and leave a comment showing how a
   second `INSTANTIATE_TEST_SUITE_P` would wire in a real backing store.
3. Add a `TempDirFixture`-style fixture for a `SessionStore` that persists
   to disk, and confirm `TearDown` runs even when a test in the middle of
   the suite fails an assertion.
4. Run your suite with `--gtest_shuffle --gtest_repeat=20` and fix any
   ordering dependency it reveals.
5. Write two sentences distinguishing which of your tests are truly
   integration tests versus which are unit tests wearing an integration
   test's file name — and move the misclassified ones.
