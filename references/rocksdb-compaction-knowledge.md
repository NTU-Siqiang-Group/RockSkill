---
name: rocksdb-compaction-knowledge
description: >-
  Use when implementing features that interact with flush, compaction,
  merge operators, or SST file lifecycle — reference for RocksDB's flush
  and compaction pipelines, partial compaction, subcompactions, and merge
  operator mechanism
---

# RocksDB Flush, Compaction & Merge Reference

## Contents

- [Data Flow Overview](#data-flow-overview)
- [Flush Pipeline](#flush-pipeline)
- [Compaction Pipeline](#compaction-pipeline)
- [Partial Compaction](#partial-compaction)
- [Trivial Moves](#trivial-moves)
- [Subcompactions](#subcompactions)
- [CompactionIterator: The Merge State Machine](#compactioniterator-the-merge-state-machine)
- [MergeOperator Mechanism](#mergeoperator-mechanism)
- [Guarantees](#guarantees)
- [Key Source Files](#key-source-files)

---

## Data Flow Overview

```
Write → MemTable (mutable)
         ↓ SwitchMemtable()
       MemTable (immutable)
         ↓ FlushJob
       L0 SSTs (overlapping)
         ↓ CompactionJob (partial, with subcompactions)
       L1 → L2 → ... → Ln (non-overlapping per level)
```

Key features that differ from textbook LSM:

| Feature | Description |
|---------|-------------|
| **Partial compaction** | Pick subset of files, not entire level |
| **Trivial moves** | Move files without rewriting when no overlap |
| **Subcompactions** | Split one compaction into parallel threads |
| **CompactionIterator** | State machine handling all key types (value, delete, merge, range delete) |
| **MergeOperator** | User-defined read-modify-write via accumulated operands |

---

## Flush Pipeline

### Flow

1. **MemTable full** → `SwitchMemtable()`: current memtable becomes immutable, new mutable created, writes continue immediately
2. **Schedule** → `FlushScheduler::ScheduleWork(cfd)`: lock-free atomic queue
3. **Execute** → `FlushJob::Run()`:
   - `PickMemTable()` — select immutables up to `max_memtable_id_`
   - `WriteLevel0Table()` — **mutex released** for I/O:
     - `MergingIterator` over all selected memtables
     - Wrapped in `CompactionIterator` (handles merges, deletes, timestamps)
     - Writes output SST via `TableBuilder`
   - Install via `LogAndApply()` — atomic Version update

### Key Options

| Option | Purpose |
|--------|---------|
| `write_buffer_size` | Memtable size before flush |
| `max_write_buffer_number` | Max memtables (mutable + immutable). Write stall when full |
| `atomic_flush` | Flush all CFs atomically for cross-CF consistency |

### What to Know When Implementing

- Flush uses `CompactionIterator` — same code path as compaction. If your feature modifies `CompactionIterator`, it affects flush too.
- `WriteLevel0Table()` releases the DB mutex. The output SST is written without holding any lock.
- Multiple immutable memtables are merged via `MergingIterator`, not flushed one-at-a-time.
- Source: `db/flush_job.h/.cc`, `db/db_impl/db_impl_write.cc` (`SwitchMemtable`).

---

## Compaction Pipeline

### Scheduling

```
MaybeScheduleFlushOrCompaction()   [mutex held]
  → Schedule HIGH-priority threads for flushes
  → Schedule LOW-priority threads for compactions
  → Respect max_background_flushes / max_background_compactions

BackgroundCompaction()             [mutex held → released → re-held]
  → Pick compaction (auto via CompactionPicker, or manual)
  → Special cases: deletion compaction (FIFO), trivial move
  → Regular: CompactionJob::Prepare() → Run() → Install()
```

### CompactionJob Flow

| Phase | Mutex | What Happens |
|-------|-------|-------------|
| `Prepare()` | Held | `GenSubcompactionBoundaries()` — partition key space |
| `Run()` | **Released** | `RunSubcompactions()` — parallel threads, I/O intensive |
| `Install()` | Held | `LogAndApply()` — add output files, remove input files |

### Per-Subcompaction Processing

Each subcompaction thread runs `ProcessKeyValueCompaction()`:

1. Create input iterator for its key range
2. Create `MergeHelper` for merge operand resolution
3. Create `CompactionIterator` — the core merge state machine
4. Iterate: for each output entry, add to current output SST file
5. When output file reaches `target_file_size`, finalize and start new file
6. Close all output files

### What to Know When Implementing

- `Run()` is the I/O-intensive phase — happens **without** the DB mutex. Do not assume mutex protection inside compaction processing.
- If your feature adds a new compaction picker, it must produce a `Compaction` object. The rest of the pipeline (`CompactionJob`) is reused.
- Source: `db/compaction/compaction_job.h/.cc`, `db/db_impl/db_impl_compaction_flush.cc`.

---

## Partial Compaction

### How It Works

Instead of merging entire levels, RocksDB picks a **subset of files**:

```
1. PickFileToCompact()         — select ONE file based on CompactionPri
2. ExpandInputsToCleanCut()    — extend to clean key boundaries
3. SetupOtherInputs()          — add ONLY overlapping output-level files
4. Create Compaction            — with just these files
```

### Clean Cut

A "clean cut" means no user key is split across the compaction boundary. If the selected file ends at key "foo" seq=5, but another file at the same level starts at "foo" seq=3, both must be included (same user key, different versions).

### max_compaction_bytes

Limits total input bytes per compaction. Prevents single compaction from being too large. If adding more files would exceed the limit, stop selecting.

### Concurrent Partial Compactions

| Scenario | Allowed? |
|----------|----------|
| Different CFs | Yes — fully independent |
| Same CF, different levels | Yes (e.g., L1→L2 and L3→L4) |
| Same level, different key ranges | Yes — non-overlapping ranges |
| Same level, overlapping key ranges | No — `being_compacted` flag prevents |

### What to Know When Implementing

- `ExpandInputsToCleanCut()` is essential for correctness. Never create a compaction that splits a user key.
- If your feature adds a new compaction picker: decide partial (like Level) or whole-level (like Tiering/FIFO). Partial is more complex but better for latency.
- Output-level files may be rewritten by multiple partial compactions — this is expected and correct.
- Source: `db/compaction/compaction_picker_level.cc`, `db/compaction/compaction_picker.cc`.

---

## Trivial Moves

When a file has **no overlap** with the output level, RocksDB moves it by metadata update only — zero I/O.

| Type | Method | Behavior |
|------|--------|----------|
| L0 → Lbase | `TryPickL0TrivialMove` | Scan oldest→newest, expand while no Lbase overlap |
| Non-L0 | `TryExtendNonL0TrivialMove` | Start with 1 file, extend ±, up to 4 files max |

**Conditions**: No compression change, single DB path, output level not empty (for L0).

Source: `db/compaction/compaction_picker_level.cc`.

---

## Subcompactions

### Purpose

Split one compaction into N parallel threads, each processing a disjoint key range.

### Boundary Generation (`GenSubcompactionBoundaries`)

1. Request ~128 **anchor points** from each input SST (key + estimated size)
2. Sort all anchors across all files by key
3. Walk anchors with cumulative size; emit boundary when `cumulative > total / max_subcompactions`
4. Result: partition keys dividing the key space into roughly equal-sized ranges

### Execution (`RunSubcompactions`)

- Subcompaction 0: runs in current thread
- Subcompactions 1..N-1: each in a new `port::Thread`
- All call `ProcessKeyValueCompaction(sub_compact)`
- Main thread `join()`s all workers
- `Install()` aggregates output files from all subcompactions into one `VersionEdit`

### SubcompactionState

Each subcompaction maintains: key range `[start, end)`, output files (`CompactionOutputs`), range deletion aggregator, status, and per-subcompaction stats.

### What to Know When Implementing

- `max_subcompactions` default is 1 (no parallelism). Set > 1 for parallel compaction.
- Subcompaction boundaries are approximate — based on SST anchor points, not exact key counts.
- If your feature adds per-compaction state, it may need to be per-subcompaction instead.
- Source: `db/compaction/compaction_job.cc` (`GenSubcompactionBoundaries`, `RunSubcompactions`), `db/compaction/subcompaction_state.h`.

---

## CompactionIterator: The Merge State Machine

Used by **both flush and compaction**. Wraps an input iterator and resolves key versions into output entries.

### Key Type Handling

| Type | Action |
|------|--------|
| `kTypeValue` | Output if first visible version in snapshot stripe. Drop if hidden by newer version in same stripe. At bottommost + no snapshot: reset seq to 0. |
| `kTypeDeletion` | Output if visible. Drop if obsolete (visible at earliest_snapshot + no key below output level). |
| `kTypeSingleDeletion` | Peek at next key. If matching Put in same snapshot stripe: drop both. Otherwise output SD. |
| `kTypeMerge` | Accumulate operands via `MergeHelper::MergeUntil()`. Stop at snapshot boundary, base value, or different key. Output merged result. |
| `kTypeRangeDeletion` | Handled by `CompactionRangeDelAggregator`, not output inline. |

### Snapshot Visibility

`findEarliestVisibleSnapshot(seq)`: binary search over sorted snapshot list.

- Two versions in **same snapshot stripe**: older is hidden → drop
- At **bottommost level** with no snapshot reference: can reset sequence to 0
- **Never drop** a version that any active snapshot might need

### Merge Operand Resolution

```
CompactionIterator encounters kTypeMerge
  → Calls MergeHelper::MergeUntil(input, stop_before_seq, at_bottom)
  → MergeUntil accumulates operands, stops at:
      - Different user key
      - Sequence number crosses snapshot boundary
      - Put/Delete found (base value)
      - Range tombstone covers key
      - End of input
  → If base value found: FullMergeV3(base, operands) → single merged value
  → If no base + bottommost: FullMergeV3(nullptr, operands)
  → If no base + NOT bottommost: output operands individually (cannot fully resolve)
  → PartialMerge may combine adjacent operands within a snapshot stripe
```

### What to Know When Implementing

- If your feature adds a new key type, `NextFromInput()` in `compaction_iterator.cc` must handle it.
- CompactionIterator is used during flush too — not just compaction. Changes affect both paths.
- Snapshot boundaries are hard barriers — merge operands cannot be combined across them.
- Source: `db/compaction/compaction_iterator.h/.cc`, `db/merge_helper.h/.cc`.

---

## MergeOperator Mechanism

### Interface

```cpp
class MergeOperator {
  // Full merge: base value (or nullptr) + all operands → result
  virtual bool FullMergeV3(const MergeOperationInputV3& in,
                           MergeOperationOutputV3* out) const;

  // Partial merge: combine two operands into one (optimization)
  virtual bool PartialMerge(const Slice& key,
                            const Slice& left, const Slice& right,
                            std::string* result, Logger*) const;
};

// Simplified interface for associative operations (e.g., counter increment)
class AssociativeMergeOperator : public MergeOperator {
  virtual bool Merge(const Slice& key, const Slice* existing_value,
                     const Slice& operand, std::string* new_value,
                     Logger*) const = 0;
  // FullMergeV2 and PartialMerge auto-implemented via repeated Merge() calls
};
```

### Where Merge Runs

| Context | Trigger | Operator Called |
|---------|---------|----------------|
| `Get()` | Accumulate operands across levels until base found | `FullMergeV3()` |
| Flush | All operands from one memtable | `FullMergeV3()` if base exists |
| Compaction | Operands at snapshot boundaries | `FullMergeV3()` at base, `PartialMerge()` between operands |
| Iterator | Lazily as iterator advances | `FullMergeV3()` |

### What to Know When Implementing

- If your feature uses `kTypeMerge`, you must provide a `MergeOperator` via `Options::merge_operator`.
- `PartialMerge` is optional but important for compaction efficiency — without it, operands accumulate across levels.
- `AllowSingleOperand()`: if true, a single Merge without base value is valid. If false (default), operands are kept until a base Put is found.
- Snapshot boundaries prevent merge resolution — operands in different stripes cannot be combined.
- Source: `include/rocksdb/merge_operator.h`, `db/merge_helper.h/.cc`.

---

## Guarantees

### Flush Guarantees

| Guarantee | Mechanism |
|-----------|-----------|
| Writes not blocked during flush | Memtable pipeline: new mutable created before flush starts |
| Flush output is atomic | `LogAndApply()` atomically installs L0 file |
| Cross-CF consistency (optional) | `atomic_flush` flushes all CFs together |
| WAL files cleaned up | Flush records log number; older WALs deleted after install |

### Compaction Guarantees

| Guarantee | Mechanism |
|-----------|-----------|
| Reads not blocked during compaction | SuperVersion ref counting; old files not deleted until unreferenced |
| Writes not blocked during compaction | Compaction releases DB mutex during I/O |
| No data loss from partial compaction | `ExpandInputsToCleanCut` ensures no user key is split |
| Snapshot isolation preserved | `CompactionIterator` never drops versions needed by active snapshots |
| Atomic output installation | `LogAndApply()` atomically adds output + removes input files |
| Concurrent compaction safety | `being_compacted` flag prevents overlapping compactions on same files |

### Merge Guarantees

| Guarantee | Mechanism |
|-----------|-----------|
| Merge operands applied in order | Accumulated oldest-to-newest, applied in sequence |
| No operand loss across snapshots | Operands not combined across snapshot boundaries |
| Merge resolved at bottommost level | `FullMergeV3` called when no older data can exist below |

---

## Key Source Files

| File | What It Contains |
|------|-----------------|
| `db/flush_job.h/.cc` | `FlushJob`, `PickMemTable()`, `WriteLevel0Table()` |
| `db/flush_scheduler.h` | Lock-free flush scheduling queue |
| `db/db_impl/db_impl_write.cc` | `SwitchMemtable()` |
| `db/db_impl/db_impl_compaction_flush.cc` | `MaybeScheduleFlushOrCompaction()`, `BackgroundCompaction()` |
| `db/compaction/compaction_job.h/.cc` | `CompactionJob`, `Prepare()`, `Run()`, `Install()`, `GenSubcompactionBoundaries()` |
| `db/compaction/subcompaction_state.h` | `SubcompactionState`, per-subcompaction key range and outputs |
| `db/compaction/compaction_iterator.h/.cc` | `CompactionIterator`, `NextFromInput()` state machine |
| `db/compaction/compaction_picker.cc` | `ExpandInputsToCleanCut()`, `SetupOtherInputs()` |
| `db/compaction/compaction_picker_level.cc` | `PickFileToCompact()`, trivial moves, intra-L0 |
| `db/merge_helper.h/.cc` | `MergeHelper`, `MergeUntil()`, operand accumulation |
| `include/rocksdb/merge_operator.h` | `MergeOperator`, `FullMergeV3()`, `AssociativeMergeOperator` |
| `db/compaction/compaction_outputs.h` | Output file management, target size splitting |
