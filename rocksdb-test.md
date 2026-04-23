---
name: rocksdb-test
description: >-
  Use when writing and running tests for a new RocksDB feature — scales testing
  effort based on change size, covers unit tests, integration tests, dedup
  review, and metrics verification
---

# RocksDB Test Skill

## Overview

This skill writes and runs tests for a newly implemented RocksDB feature. It scales testing effort proportionally to the size and risk of the change, from lightweight unit tests for small tweaks to full `make check` runs for new subsystems.

## Scope Assessment

Before writing tests, classify the change size and apply the corresponding testing level:

| Change Size | Criteria | Testing Level |
|---|---|---|
| Small | Single file change, option addition, simple behavior tweak | Unit tests only |
| Medium | New class implementing existing interface, multi-file change | Unit tests + integration tests |
| Large | New subsystem, new table format, new compaction strategy | Unit tests + integration tests + `make check` |

**Examples:**
- **Small:** Adding a new boolean option to `ColumnFamilyOptions`, fixing a boundary check in an existing method.
- **Medium:** Implementing a new `MemTableRep` backed by an existing data structure, adding a new public API method to `DB`.
- **Large:** Adding an entirely new compaction strategy, implementing a new table format, building a new cache policy.

## Step 1: Write Unit Tests

**Placement:** Place tests adjacent to the feature code:
- `db/db_*_test.cc` for DB-level features
- `table/table_test.cc` for table format features
- `cache/cache_test.cc` for cache features
- `memtable/*_test.cc` for memtable features
- Match the feature's directory — find an existing `*_test.cc` file nearby

**Table-driven test pattern:** When testing multiple input/expected pairs that share the same logic, use a struct array with a loop:

```cpp
struct TestCase {
  std::string name;
  int input;
  int expected;
};
TestCase cases[] = {
    {"zero", 0, 0},
    {"positive", 5, 10},
    {"boundary", INT_MAX, INT_MAX},
};
for (const auto& tc : cases) {
  SCOPED_TRACE(tc.name);
  ASSERT_EQ(MyFunction(tc.input), tc.expected);
}
```

**Coverage requirements:**
- Basic functionality (happy path) — the feature works as documented
- Edge cases — empty input, max values, boundary conditions, zero-length slices
- Error paths — invalid input, resource exhaustion, `Status` error propagation
- Configuration variations — different option combinations that affect the feature

**Assertions:**
- `ASSERT_OK(status)` for `Status` checks
- `ASSERT_EQ(actual, expected)` for value equality
- `ASSERT_TRUE(condition)` for boolean conditions
- Prefer `ASSERT_*` over `EXPECT_*` unless the test should continue after failure

**Thread synchronization:** Use sync points for coordinating threads in tests. Never use `sleep` — it makes tests slow and flaky.

**Timeout:** Cap each individual test at 60 seconds. If a test needs more time, it is too large — split it.

**Randomized tests:** Use `Random` from `util/random.h` with a time-based seed, and always log the seed for reproducibility:

```cpp
uint32_t seed = static_cast<uint32_t>(time(nullptr));
SCOPED_TRACE("seed=" + std::to_string(seed));
Random rnd(seed);
```

Do NOT use `std::mt19937` or other standard library RNGs — use RocksDB's `Random` utility.

**Deterministic vs randomized:** Keep deterministic edge-case tests separate from randomized fuzz-style tests. Deterministic tests should always produce the same result. Randomized tests explore broader input spaces but must log their seed so failures can be reproduced.

## Step 2: Write Integration Tests (medium/large only)

Integration tests verify the feature works correctly within the full DB lifecycle. Use the `DBTestBase` fixture, which handles DB creation, destruction, and temporary directory management.

**Test patterns:**
- **Flush verification:** Write data -> flush -> verify data is correct in SST files
- **Compaction verification:** Write data -> trigger compaction -> verify data survives compaction correctly
- **Recovery verification:** Write data -> close DB -> reopen DB -> verify data persists (tests WAL recovery and SST integrity)
- **Concurrent operations:** Use sync points to coordinate multiple threads performing operations simultaneously — verify no data corruption or crashes
- **Multiple column families:** If the feature is CF-aware, test with multiple column families to verify isolation and correct per-CF behavior

**Persistence test template:**

```cpp
TEST_F(MyFeatureTest, PersistAfterReopen) {
  Options options = CurrentOptions();
  // configure feature-specific options
  DestroyAndReopen(options);
  // write data
  ASSERT_OK(Put("key", "value"));
  ASSERT_OK(Flush());
  // reopen
  Reopen(options);
  // verify
  ASSERT_EQ(Get("key"), "value");
}
```

## Step 3: Build and Run

**Parallel builds:** Always use parallel compilation:
- macOS: `-j$(sysctl -n hw.ncpu)`
- Linux: `-j$(nproc)`

**Small/medium changes:** Build and run only the relevant test binary. To find the test binary name, grep the test file name in `Makefile`:

```bash
# Find the binary name
grep 'your_test_file' Makefile

# Build and run
JOBS=${JOBS:-$(nproc 2>/dev/null || sysctl -n hw.ncpu)}
make -j${JOBS} <test_binary_name> && ./<test_binary_name>
```

**Multiple test binaries:** When multiple test binaries need to be run, use `gtest_parallel.py` if available for faster execution:

```bash
python3 ${GTEST_PARALLEL}/gtest_parallel.py ./<test_binary>
```

**Large changes:** For large changes, run the full test suite. Use `make clean` before `make check` to ensure a clean build state:

```bash
JOBS=${JOBS:-$(nproc 2>/dev/null || sysctl -n hw.ncpu)}
make clean && make -j${JOBS} check
```

Run this in the background and monitor progress:

```bash
make check-progress
```

Report results when complete.

**If compilation fails:** Fix errors and re-run. Do not proceed to the dedup review step until all tests compile and pass.

## Step 4: Dedup Review (after tests pass)

Once all tests pass, review the test code for redundancy and clean it up:

- **Extract helper functions** for repeated object construction, round-trip patterns (write -> read -> verify), and common setup/teardown sequences
- **Consolidate similar test cases** into table-driven tests — if multiple tests differ only in input/expected values, merge them into a single parameterized test
- **Keep edge-case tests separate** from randomized tests — do not merge deterministic boundary tests into randomized loops
- **Test-only methods:** If tests need access to internal state, mark the method `private` and use the `friend class` pattern with `TEST_F` fixture wrappers. Fully qualify the target method to avoid infinite recursion:

```cpp
// In the class header:
class MyClass {
 private:
  int InternalState() const;
  friend class MyClassTest;
};

// In the test file:
class MyClassTest : public testing::Test {
  // ...
};

TEST_F(MyClassTest, CheckInternalState) {
  MyClass obj;
  // Call fully qualified to avoid recursion
  ASSERT_EQ(obj.InternalState(), 42);
}
```

- **Re-run tests after dedup** to confirm nothing broke during cleanup

## Step 5: Verify Metrics (if applicable)

Only perform this step if the feature added new tickers, histograms, or PerfContext counters (as identified in the exploration report's metrics plan).

**Pattern:**

1. Create a DB with statistics enabled
2. Reset counters
3. Perform the operation that should increment the counter
4. Assert the counter was incremented

```cpp
Options options;
options.statistics = CreateDBStatistics();
// ... open DB with these options ...

options.statistics->Reset();
// or for PerfContext:
// get_perf_context()->Reset();

// Perform the operation that triggers the metric
ASSERT_OK(Put("key", "value"));
ASSERT_OK(Flush());

// Verify the counter incremented
ASSERT_GT(options.statistics->getTickerCount(YOUR_TICKER), 0);
```

For histograms, verify with:

```cpp
ASSERT_GT(options.statistics->getHistogramCount(YOUR_HISTOGRAM), 0);
```

For PerfContext counters, verify with:

```cpp
ASSERT_GT(get_perf_context()->your_counter, 0);
```

---

## Cross-References

- **References:** `references/rocksdb-knowledge.md` for test conventions and build commands
- **Previous stage:** `rocksdb-implement.md`
