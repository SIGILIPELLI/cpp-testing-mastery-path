# 04 · Fuzz Testing

Unit tests check the inputs you thought of. A fuzzer checks the inputs you
didn't — by mutating bytes, guided by code coverage, until it finds one that
crashes your parser, trips a sanitizer, or diverges from an invariant you
declared. For any function that consumes untrusted or externally-sourced
bytes (protocol parsers, file format readers, compression), fuzzing finds
bugs that careful manual test-case design reliably misses.

!!! note "Environment note"
    libFuzzer's runtime is not installed with this machine's Command Line
    Tools (`ld: library ... libclang_rt.fuzzer_osx.a not found`), on top of
    the broken libc++ headers noted in earlier modules. Every command below
    was attempted and is shown with its real failure where relevant; the
    fuzz target and harness code were verified by manual review against
    libFuzzer's documented API instead of a live fuzzing run. Run this on a
    Linux CI image or a properly reinstalled Xcode toolchain to get real
    corpus/crash output.

## 1. What a fuzz target is

A fuzz target is a function with a fixed signature that libFuzzer (or AFL,
or a `cargo fuzz`-style wrapper) calls repeatedly with mutated byte buffers.
It does one thing per call and must not crash on *valid* or *malformed*
input — only on a genuine bug.

```c
/* fuzz_frame_parse.c -- built with clang -fsanitize=fuzzer,address */
#include <stdint.h>
#include <stddef.h>
#include "protocol.h"

int LLVMFuzzerTestOneInput(const uint8_t *data, size_t size) {
    frame_t out;
    frame_parse(data, size, &out);   /* return value ignored -- we only
                                         care whether it crashes or trips
                                         a sanitizer, not what it returns */
    return 0;
}
```

```bash
clang -std=c11 -Iinclude -g -fsanitize=fuzzer,address,undefined \
      protocol.c fuzz_frame_parse.c -o fuzz_frame_parse
./fuzz_frame_parse -max_total_time=60 corpus/
```

This is exactly the `frame_parse` function from Level 3 module 02 — the
pure-logic, host-testable parser is also the ideal fuzz target, for the same
reason: no hardware dependency, deterministic, fast per-call.

On this host the link failed (`libclang_rt.fuzzer_osx.a not found`) because
the compiler runtime library that ships libFuzzer isn't present in this
Command Line Tools install:

```text
ld: library '/Library/Developer/CommandLineTools/usr/lib/clang/21/lib/darwin/libclang_rt.fuzzer_osx.a' not found
clang: error: linker command failed with exit code 1 (use -v to see invocation)
```

On a Linux CI runner (Ubuntu's `clang` package ships the runtime) or a
correctly provisioned Xcode install, the same command builds and running it
produces output shaped like this:

```text
INFO: Seed: 1234567890
INFO: Loaded 1 modules   (128 inline 8-bit counters): 128 [...]
INFO: A corpus is not provided, starting from an empty corpus
#2      INITED cov: 9 ft: 9 corpus: 1B exec/s: 0 rss: 24Mb
#16     NEW    cov: 12 ft: 13 corpus: 2B exec/s: 0 rss: 24Mb L: 3/3
#128    NEW    cov: 14 ft: 16 corpus: 5B exec/s: 0 rss: 25Mb L: 5/5
...
#65536  DONE   cov: 15 ft: 17 corpus: 8B exec/s: 1092 rss: 26Mb
```

`cov` (edges covered) climbing over time is the signal the fuzzer is
learning your parser's structure, not just throwing random bytes.

## 2. What a crash looks like, and how to preserve it

When the fuzzer finds an input that trips a sanitizer, it writes the input
to disk and halts:

```text
==31337==ERROR: AddressSanitizer: heap-buffer-overflow on address 0x602000000018
READ of size 2 at 0x602000000018 thread T0
    #0 frame_parse protocol.c:5
    #1 LLVMFuzzerTestOneInput fuzz_frame_parse.c:8
...
artifact_prefix='./'; Test unit written to ./crash-3f9a1c2b...
```

That `crash-3f9a1c2b...` file is a **minimized, deterministic reproducer**.
Two things must happen with it:

1. **Commit it to the regression corpus**, not just fix the bug. A fuzz
   crash you don't turn into a permanent regression test is a bug you might
   reintroduce.
2. **Re-run it against the fixed binary** to confirm the fix, using the same
   binary, not a fresh fuzzing session:

```bash
./fuzz_frame_parse crash-3f9a1c2b...   # single deterministic replay
```

```cpp
// regression_corpus_test.cpp -- turns every saved crash into a permanent,
// fast, deterministic unit test that runs in the normal CI suite.
#include <gtest/gtest.h>
#include <filesystem>
#include <fstream>
#include "protocol.h"

TEST(FrameParseRegressions, AllSavedCrashesNowHandleGracefully) {
    for (const auto& entry :
         std::filesystem::directory_iterator("regressions/")) {
        std::ifstream f(entry.path(), std::ios::binary);
        std::vector<uint8_t> data(std::istreambuf_iterator<char>(f), {});
        frame_t out;
        // The only requirement: no crash, no sanitizer trip. A -1 return
        // (rejected as malformed) is a perfectly fine outcome.
        frame_parse(data.data(), data.size(), &out);
    }
}
```

## 3. Seed corpus matters more than fuzzing duration

A fuzzer starting from an empty corpus wastes most of its early budget
rediscovering structure a good seed corpus hands it for free.

```bash
mkdir -p corpus
# Seed with real, valid frames -- gives the fuzzer a head start on
# the "this is well-formed" region of the input space.
printf '\x01\x02\x00\xAA\xBB' > corpus/valid_frame_1
printf '\x02\x00\x00'         > corpus/empty_payload
printf '\x01\xFF\xFF'         > corpus/max_length_claim  # the exact
                                                          # overflow-prone
                                                          # case from
                                                          # module 02
./fuzz_frame_parse -max_total_time=300 corpus/
```

`-max_len` and dictionary files (`-dict=protocol.dict`, listing meaningful
byte sequences like known frame type bytes) further steer mutation toward
inputs that look like real protocol traffic rather than pure noise.

## 4. Structure-aware fuzzing for non-trivial formats

For a byte-flat format like the frame parser above, raw byte mutation works.
For something with internal structure (a JSON-like grammar, TLV chains), a
libFuzzer target can build structured input from the raw bytes instead of
passing them straight through, so mutations still respect enough shape to
reach deep code paths:

```cpp
#include <cstdint>
#include <cstddef>
#include "tlv_parser.h"

// Interpret the fuzzer's raw bytes as a sequence of {tag, len, value}
// triples rather than one flat buffer -- keeps mutations "on-grammar"
// so the fuzzer spends its budget past the outermost length check.
extern "C" int LLVMFuzzerTestOneInput(const uint8_t *data, size_t size) {
    size_t pos = 0;
    while (pos + 2 <= size) {
        uint8_t tag = data[pos];
        uint8_t len = data[pos + 1];
        pos += 2;
        if (pos + len > size) break;
        tlv_parser_feed(tag, data + pos, len);
        pos += len;
    }
    return 0;
}
```

## 5. Differential fuzzing

When two implementations are supposed to agree (a reference parser and an
optimized one, or your parser against a known-good library), feed the same
input to both and assert they agree — a powerful bug-finder that needs no
hand-written oracle beyond "these two must match."

```cpp
extern "C" int LLVMFuzzerTestOneInput(const uint8_t *data, size_t size) {
    frame_t a{}, b{};
    int ra = frame_parse(data, size, &a);
    int rb = frame_parse_reference(data, size, &b);  // slower, simpler impl
    if (ra != rb) __builtin_trap();
    if (ra == 0 && (a.type != b.type || a.len != b.len)) __builtin_trap();
    return 0;
}
```

## 6. Traps

- **A fuzz target that allocates unboundedly on attacker input is itself a
  bug** (denial of service). If `len` in a length-prefixed format can claim
  gigabytes, bound it before allocating, and let the fuzzer prove the bound
  holds — this is a real vulnerability class, not fuzzer noise.
- **Non-determinism in the target defeats minimization.** If your function
  reads the wall clock, a random seed, or uninitialized memory, the fuzzer's
  crash-minimization step (which re-runs the input to shrink it) can flake.
  Keep fuzz targets pure functions of their input.
- **Forgetting `-fsanitize=fuzzer,address` together.** libFuzzer alone finds
  crashes; without ASan/UBSan compiled in, many memory bugs corrupt state
  quietly and are found later, farther from the actual bug, or not at all.
- **Treating "ran for 5 minutes, no crash" as proof of correctness.** It's
  evidence, not proof. Track coverage (`cov:` in the log) — a fuzzer stuck
  at low coverage because of an early bailout (e.g. a checksum check with no
  matching corpus seed) is not exploring the code you care about.

## Cheat sheet

| Question | Answer |
|---|---|
| What functions should have fuzz targets? | Anything parsing untrusted/external bytes |
| What sanitizers should be compiled in? | ASan + UBSan, always, alongside `-fsanitize=fuzzer` |
| What do I do with a crash? | Save it to a regression corpus, replay it, fix, re-replay |
| How do I speed up discovery? | Seed corpus with real valid/edge-case inputs, use a dictionary |
| How do I fuzz structured formats? | Interpret raw bytes as your grammar inside the target |
| How do I know it's actually working? | Watch `cov:`/`ft:` climb, not just "no crash yet" |

## Exercise

1. Write a `LLVMFuzzerTestOneInput` target for the checksum-extended
   `frame_parse` you built in module 02's exercise. Seed the corpus with one
   valid frame, one corrupted-payload frame, and one truncated frame.
2. Design (in comments, since the toolchain here can't link libFuzzer) what
   a crashing input would look like for an off-by-one in your checksum
   length calculation, and write it as a static regression test file the
   way section 2 describes, without needing a live fuzzer run to produce it.
3. Write the differential fuzzing target from section 5 comparing your
   `frame_parse` against a deliberately naive reference implementation you
   write from scratch, and identify one case they'd disagree on if you
   introduced a one-line bug into either.
4. Write two sentences on the allocation-bound trap from section 6: does
   your parser have any path where an attacker-controlled length reaches an
   allocation or a loop bound before being validated?
5. Sketch (do not need to run) the CI job that would run this fuzz target
   for a fixed budget (e.g. 5 minutes) on every merge to main, fail the
   build on any new crash, and upload the crash artifact.
