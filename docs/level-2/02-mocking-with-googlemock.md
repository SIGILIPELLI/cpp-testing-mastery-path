# 02 · Mocking with GoogleMock

A unit test should fail for one reason: the unit is wrong. That breaks down as
soon as the unit talks to a database, a socket, or a sensor — now the test can
fail because the network is slow, and it takes seconds instead of microseconds.
GoogleMock replaces those collaborators with objects you control, and lets you
assert on the *interaction* itself: was the upload attempted, with what payload,
in what order.

## 1. What a mock actually is

A mock is a test double that records calls and verifies expectations. That last
part is what separates it from a stub. A stub answers questions; a mock also
insists it was asked.

Use a mock when the *interaction* is the behaviour under test — "on failure, the
error is logged" is a statement about a call, not a return value. When you only
need a canned answer, a stub or a fake is simpler and less brittle.

## 2. Declaring a mock

GoogleMock mocks a **virtual interface**. Define the seam first, then mock it.

```cpp
#include <gmock/gmock.h>
#include <gtest/gtest.h>
#include <string>

class Logger {
public:
    virtual ~Logger() = default;
    virtual void info(const std::string&) = 0;
    virtual void error(const std::string&) = 0;
};

class Uploader {
public:
    virtual ~Uploader() = default;
    virtual bool put(const std::string& body) = 0;
};

class MockLogger : public Logger {
public:
    MOCK_METHOD(void, info,  (const std::string&), (override));
    MOCK_METHOD(void, error, (const std::string&), (override));
};

class MockUploader : public Uploader {
public:
    MOCK_METHOD(bool, put, (const std::string&), (override));
};
```

`MOCK_METHOD(return_type, name, (args), (specs))`. The specs list carries
`override`, `const`, `noexcept`, and `Calling Convention`. Always include
`override` — it is the only thing that catches a signature drift between the
interface and the mock.

!!! warning "Commas inside the argument list"
    `MOCK_METHOD(void, put, (std::map<int, std::string> m), (override))` fails to
    compile: the preprocessor splits on the comma inside `map<int, string>`.
    Wrap the type in an extra pair of parentheses —
    `MOCK_METHOD(void, put, ((std::map<int, std::string>) m), (override))` — or
    typedef it first. This is the single most common GoogleMock compile error.

## 3. The system under test

```cpp
bool sync(Uploader& u, Logger& log, const std::string& body) {
    log.info("start");
    if (!u.put(body)) {
        log.error("upload failed");
        return false;
    }
    log.info("done");
    return true;
}
```

Note that `sync` takes its collaborators as references. That is the seam — it is
what makes the function testable at all. A `sync` that constructed its own
`HttpUploader` internally could not be unit tested without a network.

## 4. Setting expectations

```cpp
using ::testing::Return;
using ::testing::_;
using ::testing::NiceMock;
using ::testing::InSequence;
using ::testing::SaveArg;
using ::testing::DoAll;

TEST(Sync, LogsInOrderOnSuccess) {
    NiceMock<MockLogger> log;
    MockUploader up;
    {
        InSequence s;                                  // order is asserted
        EXPECT_CALL(log, info("start"));
        EXPECT_CALL(up, put(_)).WillOnce(Return(true));
        EXPECT_CALL(log, info("done"));
    }
    EXPECT_TRUE(sync(up, log, "payload"));
}

TEST(Sync, ReportsFailure) {
    MockLogger log;
    NiceMock<MockUploader> up;
    EXPECT_CALL(up, put(_)).WillOnce(Return(false));
    EXPECT_CALL(log, info("start"));
    EXPECT_CALL(log, error("upload failed"));
    EXPECT_FALSE(sync(up, log, "x"));
}
```

Build and run:

```bash
clang++ -std=c++17 -I/opt/homebrew/opt/googletest/include \
    sync_test.cpp -L/opt/homebrew/opt/googletest/lib \
    -lgtest -lgmock -lgtest_main -o sync_test
./sync_test
```

```
[==========] Running 3 tests from 1 test suite.
[----------] 3 tests from Sync
[ RUN      ] Sync.LogsInOrderOnSuccess
[       OK ] Sync.LogsInOrderOnSuccess (0 ms)
[ RUN      ] Sync.CapturesBodyArgument
[       OK ] Sync.CapturesBodyArgument (0 ms)
[ RUN      ] Sync.ReportsFailure
[       OK ] Sync.ReportsFailure (0 ms)
[----------] 3 tests from Sync (0 ms total)

[==========] 3 tests from 1 test suite ran. (0 ms total)
[  PASSED  ] 3 tests.
```

Expectations are verified when the mock is **destroyed**, not at the end of the
statement. A missing call is reported as the test ends.

## 5. Cardinality and actions

```cpp
EXPECT_CALL(m, read())                     // implicitly Times(1) here
EXPECT_CALL(m, read()).Times(3);
EXPECT_CALL(m, read()).Times(::testing::AtLeast(1));
EXPECT_CALL(m, read()).Times(0);           // must never be called

EXPECT_CALL(m, read())
    .WillOnce(Return(10))
    .WillOnce(Return(20))
    .WillRepeatedly(Return(0));            // every call after the first two

EXPECT_CALL(m, put(_)).WillOnce([](const std::string& s) { return !s.empty(); });
EXPECT_CALL(m, load()).WillOnce(::testing::Throw(std::runtime_error("io")));
```

!!! warning "Implicit cardinality"
    If you supply `WillOnce` clauses and no `Times`, the cardinality becomes
    exactly the number of `WillOnce` clauses. Adding a `WillRepeatedly` changes
    it to "n or more". Writing `.Times(2).WillOnce(Return(1))` means the second
    call returns a **default-constructed** value, not `1` — a silent source of
    zeros in your assertions.

## 6. `ON_CALL` vs `EXPECT_CALL`

`ON_CALL` sets behaviour without asserting anything. `EXPECT_CALL` sets
behaviour **and** demands the call happen.

```cpp
ON_CALL(up, put(_)).WillByDefault(Return(true));   // background, no assertion
EXPECT_CALL(up, put("payload"));                   // this call must happen
```

The rule: state your default behaviour with `ON_CALL` in the fixture's `SetUp`,
and write `EXPECT_CALL` only for the one or two interactions the test is actually
about. A test with eight `EXPECT_CALL`s is asserting on the implementation, and
will break on every refactor.

## 7. Nice, naggy and strict

An "uninteresting call" is a call to a mock method with no matching
`EXPECT_CALL`. The default mock is **naggy**: it allows the call but prints a
warning. This is why real suites are noisy.

```cpp
MockLogger log;               // naggy  — warns on uninteresting calls
NiceMock<MockLogger> log;     // nice   — silent
StrictMock<MockLogger> log;   // strict — uninteresting call fails the test
```

Use `NiceMock` for collaborators that aren't the subject of the test (a logger
you don't care about), and `StrictMock` sparingly, where any unexpected traffic
is genuinely a bug. Defaulting everything to `StrictMock` produces a suite that
fails whenever anyone adds a log line.

## 8. Capturing arguments

When you need to inspect a complex argument rather than match it, capture it.

```cpp
TEST(Sync, CapturesBodyArgument) {
    NiceMock<MockLogger> log;
    NiceMock<MockUploader> up;
    std::string seen;
    EXPECT_CALL(up, put(_))
        .WillOnce(DoAll(SaveArg<0>(&seen), Return(true)));

    sync(up, log, "hello-world");
    EXPECT_EQ(seen, "hello-world");
}
```

`SaveArg<N>` stores argument `N`; `DoAll` chains it with the return action.
Capturing then asserting is usually clearer than a hand-written matcher, and the
failure message shows the actual value.

## 9. Matchers on arguments

```cpp
using ::testing::HasSubstr, ::testing::Field, ::testing::Property;
using ::testing::AllOf, ::testing::Not, ::testing::Truly;

EXPECT_CALL(up, put(HasSubstr("\"id\":")));
EXPECT_CALL(sink, write(Field(&Packet::length, 64)));
EXPECT_CALL(sink, write(Property(&Packet::valid, true)));
EXPECT_CALL(up, put(AllOf(Not(HasSubstr("password")), HasSubstr("user"))));
EXPECT_CALL(up, put(Truly([](const std::string& s){ return s.size() < 1024; })));
```

## 10. Reference

| Need | Write |
|---|---|
| Any argument | `EXPECT_CALL(m, f(_))` |
| Specific argument | `EXPECT_CALL(m, f("abc"))` |
| Argument predicate | `EXPECT_CALL(m, f(Truly(pred)))` |
| Field of a struct argument | `Field(&S::member, matcher)` |
| Return a value | `.WillOnce(Return(v))` |
| Return different values in turn | `.WillOnce(Return(a)).WillOnce(Return(b))` |
| Throw | `.WillOnce(Throw(std::runtime_error("x")))` |
| Capture an argument | `.WillOnce(DoAll(SaveArg<0>(&dest), Return(v)))` |
| Default behaviour, no assertion | `ON_CALL(m, f(_)).WillByDefault(...)` |
| Must never be called | `EXPECT_CALL(m, f(_)).Times(0)` |
| Enforce call order | `{ InSequence s; ... }` |
| Silence uninteresting calls | `NiceMock<M>` |
| Fail on uninteresting calls | `StrictMock<M>` |
| Verify early | `Mock::VerifyAndClearExpectations(&m)` |

## Exercise

Build a small `BackupService` and test it entirely with mocks.

```cpp
class Clock      { public: virtual ~Clock() = default;
                   virtual long now() = 0; };
class Storage    { public: virtual ~Storage() = default;
                   virtual bool write(const std::string& key,
                                      const std::string& data) = 0;
                   virtual bool remove(const std::string& key) = 0; };
class Notifier   { public: virtual ~Notifier() = default;
                   virtual void notify(const std::string& msg) = 0; };

// Writes data under key "backup-<now>". On write failure, notifies once and
// returns false. On success, removes any key passed in `supersedes` (if
// non-empty) and returns true.
bool run_backup(Clock&, Storage&, Notifier&,
                const std::string& data, const std::string& supersedes);
```

1. **Implement `run_backup`** to that contract, then mock all three
   collaborators.

2. **Write the success test.** Use `InSequence` to assert `now()` is called
   before `write()`, and `write()` before `remove()`. Use `SaveArg<0>` to capture
   the key and assert it matches `backup-` plus the timestamp your mock clock
   returned.

3. **Write the failure test.** Make `write` return `false` and assert
   `notify` is called exactly once and `remove` is **never** called
   (`Times(0)`). Confirm the test fails if you delete the `Times(0)` guard and
   change the implementation to always remove.

4. **Exercise the empty-`supersedes` boundary.** Assert `remove` is not called
   when `supersedes` is empty.

5. **Feel the three strictness modes.** Add a stray `notify("debug")` call to
   your implementation. Run the suite with `MockNotifier`, then
   `NiceMock<MockNotifier>`, then `StrictMock<MockNotifier>`. Record what each
   prints, and write two sentences on which one you'd want in CI and why.

6. **Refactor away over-specification.** Move the "always succeed" behaviour of
   `Storage::write` into an `ON_CALL` in a fixture `SetUp()`, and delete every
   `EXPECT_CALL` that isn't the point of its test. Count how many expectations
   you removed.
