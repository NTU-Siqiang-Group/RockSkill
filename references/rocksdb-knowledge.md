---
name: rocksdb-knowledge
description: >-
  Use when exploring RocksDB codebase structure, finding interfaces, or
  understanding the build system — shared reference for all RocksDB skills
---

# RocksDB Structural Knowledge Reference

## Directory Layout

| Directory | Purpose |
|---|---|
| `db/` | Core DB implementation (write path, read path, compaction, flush, recovery) |
| `db/compaction/` | Compaction strategies and pickers |
| `include/rocksdb/` | Public headers (API surface) |
| `memtable/` | Memtable implementations |
| `table/` | Table formats (block-based, plain, cuckoo) |
| `table/block_based/` | Block-based table reader/writer/builder |
| `cache/` | Cache implementations (LRU, clock) |
| `env/` | File system and environment abstractions |
| `util/` | Internal utilities (hash, coding, random, etc.) |
| `options/` | Option structs, serialization, validation |
| `tools/` | db_bench, ldb, sst_dump |
| `db_stress_tool/` | Stress test infrastructure |
| `utilities/` | Optional features (transactions, backup engine, checkpoint) |
| `monitoring/` | Statistics, perf context, histograms |

## Key Interfaces per Feature Type

| Feature Type | Base Class / Interface | Header Location |
|---|---|---|
| Compaction strategy | `CompactionPicker` | `db/compaction/compaction_picker.h` |
| Memtable | `MemTableRep`, `MemTableRepFactory` | `include/rocksdb/memtablerep.h` |
| Table format | `TableReader`, `TableBuilder`, `TableFactory` | `include/rocksdb/table.h` |
| Cache policy | `Cache`, `CacheWrapper` | `include/rocksdb/cache.h` |
| File system / Env | `FileSystem`, `Env` | `include/rocksdb/file_system.h`, `include/rocksdb/env.h` |
| Filter policy | `FilterPolicy` | `include/rocksdb/filter_policy.h` |
| Public API | `DB` | `include/rocksdb/db.h` |
| Option (CF) | `AdvancedColumnFamilyOptions` | `include/rocksdb/advanced_options.h` |
| Option (DB) | `DBOptions` | `include/rocksdb/options.h` |

## Build System

- `make dbg` — debug build for all unit tests
- `make check` — build and run all tests
- `make check-progress` — JSON progress for long builds
- `make format-auto` — auto-format code
- `make clean && DEBUG_LEVEL=0 make db_bench` — release build for benchmarking
- New `.cc` files must be added to `Makefile`, `CMakeLists.txt`, and `src.mk`. Then run `/usr/local/bin/python3 buckifier/buckify_rocksdb.py` to regenerate BUCK. Do NOT manually edit BUCK files.
- Use `-j$(sysctl -n hw.ncpu)` on macOS or `-j$(nproc)` on Linux for parallel builds. Always pass `-j` to avoid single-threaded compilation.

## Testing Conventions

- Use `ASSERT_OK(status)` for Status checks, `ASSERT_EQ(actual, expected)` for equality, `ASSERT_TRUE(condition)` for boolean assertions.
- Use sync points for thread synchronization. Never use `sleep` to coordinate threads in tests.
- Cap each unit test at 60 seconds.
- Use table-driven tests (struct array with input/expected pairs plus a loop) when multiple cases share the same logic.
- Use `Random` from `util/random.h` with a time-based seed, not `std::mt19937`. Pair with `SCOPED_TRACE("seed=" + std::to_string(seed))` so failures are reproducible.

## Coding Conventions

- Use `Status` for all error handling. Always check and propagate errors; do not silently ignore a returned `Status`.
- Use `Slice` or `std::string_view` to pass string data without copying.
- Use `LIKELY(cond)` and `UNLIKELY(cond)` macros on hot-path branches to hint the branch predictor.
- Prefer stack allocation over heap allocation on hot paths to reduce allocator pressure.
- Prefer `enum class` over unscoped enums for type safety.
- Document thread-safety assumptions in comments (e.g., "caller must hold mutex", "thread-safe after construction").
- Follow existing naming conventions. Run `make format-auto` or rely on `.clang-format` to enforce formatting before submitting code.
