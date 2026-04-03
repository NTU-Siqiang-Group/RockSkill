---
name: rocksdb-implement
description: Use when writing RocksDB feature code after exploration is complete — contains implementation patterns for all feature types including options, APIs, compaction, memtable, table format, cache, env, and filter policy
---

# RocksDB Implementation Stage

## Overview

This skill takes the exploration report produced by `rocksdb-explore.md` and produces the implementation. It contains a general workflow that applies to all feature types, followed by feature-type-specific sections with detailed patterns, files to touch, and reference implementations.

---

## General Workflow

These steps apply to every feature type. Follow them in order.

1. **Read the exploration report** — Verify the file list and interface contract. Confirm the feature type, base class, required methods, reference implementations, and files to create/modify all match expectations. If anything is missing or unclear, re-run exploration before proceeding.

2. **Implement core logic first** — Create new `.h` and `.cc` files following the reference implementations identified during exploration. Start with the class declaration in the header, then implement methods in the `.cc` file. Follow the same constructor patterns, error handling, and naming conventions as the reference.

3. **Register the feature** — Wire the new implementation into the factory/registration system. This varies by feature type (enum addition, factory method update, switch statement, or user-facing option). See the feature-type section below for specifics.

4. **Update build system** — If you created new `.cc` files: update `Makefile`, `CMakeLists.txt`, and `src.mk` to include them. Then regenerate the BUCK file:
   ```
   /usr/local/bin/python3 buckifier/buckify_rocksdb.py
   ```
   Do NOT manually edit BUCK files.

5. **Add metrics** — If the exploration report identified metrics opportunities, add tickers, histograms, or PerfContext counters following `references/rocksdb-stats-knowledge.md`.

6. **Run `make format-auto`** — Auto-format all changed files before testing. This ensures code matches the project `.clang-format` style.

---

## Feature-Type Sections

### Option (CF/DB/BlockBased)

**Reference:** `claude_md/add_option.md` for the full step-by-step guide.

**Key decision:** Determine whether the option is mutable (can change at runtime via `SetOptions()` — goes in `MutableCFOptions`) or immutable (set at DB open time — goes in `ImmutableCFOptions`).

**Files to touch (in order):**

1. **`include/rocksdb/advanced_options.h`** (for CF options) or **`include/rocksdb/options.h`** (for DB options) or **`include/rocksdb/table.h`** (for BlockBasedTableOptions) — Define the option field with documentation comment.
2. **`options/cf_options.h`** — Add the field to the internal struct. Update both constructors (default and from public options).
3. **`options/cf_options.cc`** — Register the option in the type info map for serialization/deserialization. Add to `Dump()` for logging.
4. **`options/options_helper.cc`** — Add to `UpdateColumnFamilyOptions()` (mutable options only).
5. **`options/options_settable_test.cc`** — Add the option to the test string so the settable test covers it.

**Pattern:** Find an existing option of the same C++ type (bool, int, uint64_t, size_t) and copy its pattern across all five files. This ensures correct serialization handling.

**Registration:** Options are registered through the type info map in `cf_options.cc` (or `db_options.cc`). The map entry controls how the option is serialized, deserialized, and validated.

**Reference implementations:** Look at any recently added option of the same type in `advanced_options.h`.

---

### Remove Deprecated Option

**Reference:** `claude_md/remove_option.md` for the full step-by-step guide.

**Critical rule:** KEEP the entry in the type info map (`options/cf_options.cc` or `options/db_options.cc`) with `OptionVerificationType::kDeprecated`. This allows old options files to load without errors.

**KEEP or add:**
- Test in `options/options_test.cc` under `OptionsOldApiTest::GetOptionsFromMapTest` — verifies deprecated options still parse.

**REMOVE from:**
- Public header (`include/rocksdb/advanced_options.h`, `options.h`, or `table.h`)
- Internal option structs (`options/cf_options.h`)
- `options/options_helper.cc` (mutable option handling)
- `options/options_settable_test.cc`
- Stress/bench tools (`db_stress_tool/`, `tools/db_bench_tool.cc`) — remove any usage

**Base class:** N/A (operates on existing option infrastructure).

**Key methods:** N/A.

**Registration pattern:** Change the existing type info entry from its current verification type to `OptionVerificationType::kDeprecated`.

**Files to create/modify:** No new files. Only modifications to existing files as listed above.

**Reference implementations:** Search git history for recent option deprecation commits.

---

### Public API

**Reference:** `claude_md/add_public_api.md` for the full step-by-step guide.

**Files (in order):**

1. **`include/rocksdb/db.h`** — Add a virtual method to the `DB` class. Add a convenience overload that takes no `ColumnFamilyHandle*` and delegates to the default column family.
2. **Options struct in `include/rocksdb/options.h`** — If the API needs new configuration, add an options struct or extend an existing one.
3. **`db/db_impl/db_impl.h`** — Add the override declaration. Add `using DB::YourMethod;` to pull in the convenience overload.
4. **`db/db_impl/db_impl_*.cc`** — Write the implementation. Choose the file by category:
   - `db_impl_compaction_flush.cc` — compaction/flush related
   - `db_impl_write.cc` — write path
   - `db_impl_open.cc` — open/close
   - `db_impl_files.cc` — file management
   - `db_impl.cc` — general / does not fit elsewhere
5. **`include/rocksdb/utilities/stackable_db.h`** — Add a delegation method: `return db_->YourMethod(args);`
6. **`db/db_impl/db_impl_secondary.h`** — Return `Status::NotSupported("Not supported in secondary mode")`.
7. **`db/db_impl/compacted_db_impl.h`** — Return `Status::NotSupported("Not supported in compacted mode")`.

**Pattern:** Use `Status` as the return type. Accept `ColumnFamilyHandle*` for CF-aware operations. Validate inputs first, then perform the operation.

**Base class:** `DB` in `include/rocksdb/db.h`.

**Key methods:** The virtual method you are adding, plus its convenience overload.

**Registration pattern:** Virtual method override in `DBImpl`. Delegation in `StackableDB`. `NotSupported` stubs in secondary and compacted DB.

**Reference implementations:** Look at existing methods in `db.h` that follow the same pattern (e.g., `GetProperty`, `GetApproximateSizes`).

---

### Compaction Strategy

**Base class:** `CompactionPicker` in `db/compaction/compaction_picker.h`.

**Key methods to implement:**
- `PickCompaction()` — Select files for compaction. Return a `Compaction*` object describing the input/output levels and files.
- `NeedsCompaction()` — Return true if compaction should be scheduled. Called frequently, must be fast.
- `SanitizeCompactionInputLevel()` — Validate that the compaction input level is valid for this strategy.

**Registration:**
1. Add a new value to the `CompactionStyle` enum in `include/rocksdb/advanced_options.h`.
2. Wire into `CreateCompactionPicker()` in `db/version_set.cc` — add a case to the switch statement that instantiates your picker.

**Files to create/modify:**
- Create: `db/compaction/compaction_picker_<name>.h` and `db/compaction/compaction_picker_<name>.cc`
- Modify: `include/rocksdb/advanced_options.h` (enum), `db/version_set.cc` (factory switch), build files

**Reference implementations:** `LevelCompactionPicker` in `db/compaction/compaction_picker_level.cc`, `FIFOCompactionPicker` in `db/compaction/compaction_picker_fifo.cc`.

---

### Memtable Implementation

**Base classes:** `MemTableRep` in `include/rocksdb/memtablerep.h` (the data structure), `MemTableRepFactory` (the factory).

**Key methods:**

- `MemTableRep::Allocate(len)` — Allocate memory for a key using the arena allocator.
- `MemTableRep::Insert(handle)` — Insert a key handle into the data structure.
- `MemTableRep::Get(key, callback)` — Point lookup. Call the callback for each matching key.
- `MemTableRep::GetIterator()` — Return an iterator for range scans. Must support `Seek()`, `SeekForPrev()`, `Next()`, `Prev()`.
- `MemTableRepFactory::CreateMemTableRep()` — Factory method that creates the `MemTableRep` instance.
- `MemTableRepFactory::Name()` — Return a unique string identifier for the memtable type.

**Registration:** User sets `Options::memtable_factory` to an instance of your factory. No enum or switch statement needed.

**Files to create/modify:**
- Create: `memtable/<name>rep.h` and `memtable/<name>rep.cc`
- Modify: build files (`Makefile`, `CMakeLists.txt`, `src.mk`)

**Reference implementations:** `SkipListRep` in `memtable/skiplistrep.cc`, `HashSkipListRep` in `memtable/hash_skiplist_rep.cc`.

---

### Table Format

**Base classes:** `TableFactory`, `TableReader`, `TableBuilder` in `include/rocksdb/table.h`.

**Key methods:**

- `TableFactory::NewTableReader()` — Open an existing table file for reading.
- `TableFactory::NewTableBuilder()` — Create a new table builder for writing.
- `TableFactory::Name()` — Return a unique string identifier.
- `TableReader::Get()` — Point lookup in the table.
- `TableReader::NewIterator()` — Create an iterator for range scans.
- `TableBuilder::Add(key, value)` — Add a key-value pair to the table being built.
- `TableBuilder::Finish()` — Finalize the table and flush to disk.
- `TableBuilder::Abandon()` — Discard the partially built table.

**Registration:** User sets `Options::table_factory` to an instance of your factory. No enum or switch statement needed.

**Files to create/modify:**
- Create: a new directory `table/<name>/` containing reader, builder, and factory files
- Modify: build files (`Makefile`, `CMakeLists.txt`, `src.mk`)

**Reference implementation:** `BlockBasedTableFactory` in `table/block_based/` — study the reader (`block_based_table_reader.cc`), builder (`block_based_table_builder.cc`), and factory (`block_based_table_factory.cc`).

---

### Cache Policy

**Base class:** `Cache` in `include/rocksdb/cache.h`. Alternatively, wrap an existing cache via `CacheWrapper`.

**Key methods:**
- `Insert(key, value, charge, deleter)` — Add an entry with a memory charge and a deleter callback.
- `Lookup(key)` — Find an entry by key. Return a handle, or nullptr if not found.
- `Release(handle)` — Release a handle obtained from `Lookup()`.
- `Erase(key)` — Remove an entry by key.
- `GetCapacity()` — Return the maximum capacity.
- `GetUsage()` — Return the current memory usage.
- `Name()` — Return a unique string identifier.

**Registration:** Created via a factory function. User sets `BlockBasedTableOptions::block_cache` to the created instance.

**Files to create/modify:**
- Create: `cache/<name>_cache.h` and `cache/<name>_cache.cc`
- Modify: build files (`Makefile`, `CMakeLists.txt`, `src.mk`)

**Reference implementations:** `LRUCache` in `cache/lru_cache.h` and `cache/lru_cache.cc`, `ClockCache` in `cache/clock_cache.h`.

---

### File System / Env Extension

**Base class:** `FileSystem` in `include/rocksdb/file_system.h` (preferred over the legacy `Env` interface).

**Key classes to implement:**
- `FileSystem` — Factory methods for creating file objects (`NewSequentialFile`, `NewRandomAccessFile`, `NewWritableFile`, `NewDirectory`).
- `FSSequentialFile` — Sequential read operations.
- `FSRandomAccessFile` — Random read operations (pread-style).
- `FSWritableFile` — Write operations (append, sync, close).
- `FSDirectory` — Directory operations (fsync directory).

**Registration:** User sets `DBOptions::file_system` to an instance of your filesystem.

**Files to create/modify:**
- Create: `env/<name>_file_system.h` and `env/<name>_file_system.cc`
- Modify: build files (`Makefile`, `CMakeLists.txt`, `src.mk`)

**Reference implementation:** `PosixFileSystem` in `env/` — study `env/fs_posix.cc` and related files.

---

### Filter Policy

**Base class:** `FilterPolicy` in `include/rocksdb/filter_policy.h`.

**Key methods:**
- `CreateFilter(keys, n, dst)` — Build a filter from an array of keys. Append the filter to `dst`.
- `KeyMayMatch(key, filter)` — Check if a key might exist in the filter. False positives allowed, false negatives are not.
- `GetFilterBitsBuilder()` — Return a `FilterBitsBuilder` for partitioned filters.
- `GetFilterBitsReader(contents)` — Return a `FilterBitsReader` for partitioned filters.
- `Name()` — Return a unique string identifier.

**Registration:** User sets `BlockBasedTableOptions::filter_policy` to an instance of your filter policy.

**Files to create/modify:**
- Create: `table/<name>_filter.h` and `table/<name>_filter.cc`
- Modify: build files (`Makefile`, `CMakeLists.txt`, `src.mk`)

**Reference implementations:** `BloomFilterPolicy` and `RibbonFilterPolicy` in `table/block_based/filter_policy.cc`.

---

## Implementation Report Requirements

The Implementation Report (generated in pipeline step 4) must be detailed enough that a **fresh agent with zero RocksDB context** can execute it step-by-step and produce correct, compiling code. The executing agent has not seen the Exploration Report or codebase — the plan must be self-contained.

### Document Header

Every Implementation Report must start with:

```markdown
# [Feature Name] Implementation Report

**Goal:** [One sentence — what this builds]

**Feature Type:** [From exploration report]

**Architecture:** [2-3 sentences — approach and key design decisions]

---
```

### Reference Divergence Analysis

Place this immediately after the header — it frames all subsequent decisions.

For each reference implementation identified in the Exploration Report:
- List the key behaviors/patterns in that reference.
- For each behavior, mark: **KEEP** (reuse as-is) or **CHANGE** (different for this feature), with reason.
- This prevents blind copy-paste of patterns that don't apply.

Example (compaction strategy referencing `LevelCompactionPicker`):

| Reference Behavior | Keep/Change | Reason |
|--------------------|-------------|--------|
| `ExpandInputsToCleanCut` on input level | KEEP | Need clean key boundaries |
| `GetOverlappingInputs` on output level | CHANGE → remove | New strategy does not merge with output level |
| `RegisterCompaction` after picking | KEEP | Standard bookkeeping |

### File Structure

Before defining tasks, list every file to create or modify with its responsibility:

```markdown
## File Structure

**New files:**
- `path/to/new_feature.h` — class declaration, public interface
- `path/to/new_feature.cc` — implementation

**Files to modify:**
- `include/rocksdb/advanced_options.h:42` — add enum value or option field
- `db/relevant_file.cc:1234` — add factory case or registration
- `options/options_helper.cc:567` — add serialization entry (if new option/enum)
- `src.mk:89` — add new .cc to source list
- `CMakeLists.txt:345` — add new .cc to cmake
```

Line numbers must be approximate but present — they tell the executing agent where to look.

### Task Decomposition

Break the implementation into bite-sized tasks. Each task targets one file or one logical unit. Each step within a task is a **single action**.

#### Task Structure

````markdown
### Task N: [Component Name]

**Files:**
- Create: `exact/path/to/file.h`
- Create: `exact/path/to/file.cc`
- Modify: `path/to/existing.cc:123` — [what change]

**Behavioral spec:**
- **Purpose**: [One sentence]
- **Does**: [What this component does — concrete behaviors]
- **Does NOT**: [What this component must NOT do — deviations from reference]

- [ ] **Step 1: Create header with class declaration**

```cpp
// Show the complete header file content.
// The executing agent copies this verbatim.
```

- [ ] **Step 2: Implement core method**

```cpp
// Show the complete method implementation.
// No pseudocode, no placeholders, no ellipsis.
// For long methods (>80 lines), may use:
//   // [lines N-M identical to reference: ClassName::MethodName]
// to indicate a verbatim copy the agent can look up.
```

- [ ] **Step 3: Register in factory / enum / options**

```cpp
// Show the exact code to insert and where (file:line).
```
````

### Step Granularity Rules

Each step must be **one action** that takes 2-5 minutes:

| Good (single action) | Bad (multiple actions) |
|----------------------|----------------------|
| "Create header with class declaration" | "Create header and implement all methods" |
| "Add enum value to CompactionStyle" | "Update all option files" |
| "Add serialization entry in options_helper.cc" | "Wire up options" |

### No Placeholders

Every step must contain the **actual code** the executing agent will write. These are plan failures — never write them:

- `// TODO`, `// implement later`, `// fill in details`
- `// Add appropriate error handling`
- `// Similar to Task N` (repeat the code — the agent may read tasks out of order)
- Pseudocode where real code is needed (e.g., `// loop through levels and pick files`)
- Steps that describe what to do without showing the code
- References to functions, types, or variables not defined or shown in the plan
- `...` or ellipsis in code blocks (show the full function body)

**Exception:** For methods longer than ~80 lines, show the complete code but you may use a comment like `// [lines 45-60 identical to reference: LevelCompactionPicker::SetupOtherInputs]` to indicate a verbatim copy from a named reference. The executing agent can then read that reference.

### Self-Review

After writing the complete report, scan for these before presenting to the critic agent:

1. **Divergence coverage**: Every CHANGE in the divergence table has a corresponding task step. Every KEEP has code that preserves it.
2. **Placeholder scan**: No `TODO`, `...`, `// similar to`, or `// implement` in code blocks.
3. **Type consistency**: Function signatures and enum values in later tasks match their definitions in earlier tasks.

---

## Coding Conventions

These conventions are enforced for all feature types:

- **Use `Status` for error handling** — Always return `Status` from functions that can fail. Propagate errors up the call stack; do not silently swallow them.
- **Use `Slice` / `std::string_view` to avoid copies** — Pass string data by reference using `Slice` (RocksDB's string view type) rather than copying into `std::string`.
- **Use `LIKELY`/`UNLIKELY` macros on hot paths** — Annotate branch conditions on performance-critical paths to help branch prediction.
- **Stack allocation over heap allocation on hot paths** — Avoid `new`/`malloc` in tight loops. Use stack-allocated buffers or arena allocation.
- **Prefer `enum class` over unscoped enums** — Use scoped enumerations for type safety.
- **Document thread-safety assumptions in comments** — State whether a class or method is thread-safe, requires external synchronization, or is single-threaded only.
- **Follow existing naming conventions and `.clang-format`** — Match the naming style of surrounding code. Run `make format-auto` to apply the project's clang-format configuration.

---

## MVCC, Timestamp, and Transaction Considerations

Every new feature must preserve RocksDB's existing guarantees around versioning, timestamps, and transactions. Review this checklist during implementation. If any item applies, read the corresponding reference for details.

### Multi-Version Control (MVCC)

Reference: `references/rocksdb-mvcc-knowledge.md`

- **Key format**: InternalKeys contain `[user_key | (timestamp) | seqno + type]`. If your feature encodes or decodes keys, account for this layout.
- **Snapshot visibility**: Reads filter by sequence number. If you add a new read path, it must respect `ReadOptions::snapshot` — only return entries with `seq <= snapshot_seq`.
- **Compaction correctness**: If you add a new entry type or modify compaction logic, ensure `CompactionIterator` never drops a version that an active snapshot needs.
- **Write path sequencing**: New write types must flow through `WriteBatch` → sequence allocation → MemTable insertion. Do not bypass this path or assign sequence numbers manually.

### User-Defined Timestamps (UDT)

Reference: `references/rocksdb-mvcc-knowledge.md` (UDT section)

- **Timestamp in keys**: When UDT is enabled (`Comparator::timestamp_size() > 0`), timestamp bytes sit between user key and internal footer. Use `ExtractTimestampFromKey()` / `StripTimestampFromUserKey()` — do not hard-code key offsets.
- **Read filtering**: If your feature adds a read path, check `ReadOptions::timestamp`. If set, filter results to `ts <= read_timestamp`.
- **Compaction GC**: Respect `full_history_ts_low` — never drop versions with `ts >= full_history_ts_low`.
- **Deletion type**: Use `kTypeDeletionWithTimestamp` (0x14) instead of `kTypeDeletion` when UDT is enabled.

### Concurrency and Transactions

Reference: `references/rocksdb-concurrency-knowledge.md`

- **Thread safety**: Document thread-safety assumptions for new classes. If your feature is accessed from read path, it must work without holding the DB mutex (readers use SuperVersion refs, not locks).
- **Write group compatibility**: If your feature adds a new write option flag, consider whether it affects write group batching — writers with incompatible flags cannot be grouped.
- **Transaction visibility**: If your feature interacts with `WritePrepared` or `WriteUnprepared` transactions, data may be in the DB but uncommitted. Use `SnapshotChecker` to determine visibility, not raw sequence comparison.
- **2PC markers**: If your feature touches WAL replay or recovery, account for 2PC markers (`kTypeBeginPrepareXID`, `kTypeEndPrepareXID`, `kTypeCommitXID`, `kTypeRollbackXID`).
- **Lock interaction**: If your feature performs reads inside a transaction path (e.g., `GetForUpdate`), ensure it acquires appropriate locks via `LockManager`.

---

## Cross-References

- **References:** `references/rocksdb-knowledge.md` for interfaces and build system, `references/rocksdb-stats-knowledge.md` for metrics patterns, `references/rocksdb-mvcc-knowledge.md` for MVCC and timestamps, `references/rocksdb-concurrency-knowledge.md` for concurrency and transactions.
- **Previous stage:** `rocksdb-explore.md` (produces the exploration report consumed here).
- **Next stage:** `rocksdb-test.md` (writes and runs tests for the implementation).
