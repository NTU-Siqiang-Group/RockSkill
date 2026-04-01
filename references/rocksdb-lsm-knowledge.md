---
name: rocksdb-lsm-knowledge
description: >-
  Use when implementing features that interact with LSM-tree structure,
  compaction, level management, SST files, or memtables — reference for
  how RocksDB's LSM implementation differs from the textbook model
---

# RocksDB LSM-Tree Reference

## Contents

- [Textbook LSM-Tree](#textbook-lsm-tree)
- [RocksDB Accommodations Overview](#rocksdb-accommodations-overview)
- [Level 0: Overlapping Sorted Runs](#level-0-overlapping-sorted-runs)
- [File-Level Partial Compaction](#file-level-partial-compaction)
- [Trivial Moves](#trivial-moves)
- [Dynamic Level Sizing](#dynamic-level-sizing)
- [Compaction Priority Strategies](#compaction-priority-strategies)
- [Subcompactions](#subcompactions)
- [MemTable Pipeline and Flush](#memtable-pipeline-and-flush)
- [SST File Format](#sst-file-format)
- [Range Deletions](#range-deletions)
- [Bloom Filters](#bloom-filters)
- [Column Families](#column-families)
- [Alternative Compaction Styles](#alternative-compaction-styles)
- [Textbook vs RocksDB Summary](#textbook-vs-rocksdb-summary)
- [Key Source Files](#key-source-files)

---

## Textbook LSM-Tree

The modern LSM-tree (Luo & Carey 2020, Dayan & Idreos 2018) is a sorted-array-based structure:

- **L levels**, each T times larger than the previous. T = size ratio.
- `L = O(log_T(N/M))` where N = total data, M = memtable size.
- **Leveling**: 1 sorted run per level. When full, merge entirely into next level.
- **Tiering**: Up to T sorted runs per level. When full, merge all T runs and push to next level.

| Metric | Leveling | Tiering |
|--------|----------|---------|
| Write amplification | O(T * L) | O(L) |
| Point read (with Bloom) | O(L) | O(T * L) |
| Short range query | O(L) | O(T * L) |
| Space amplification | O(1/T) | O(T) |

**Textbook assumptions** (RocksDB departs from all of these):
1. Each level is one monolithic sorted run
2. Compaction merges an entire level into the next
3. L0 is not special
4. No range deletions, no column families
5. No concurrent or partial compaction
6. Fixed level sizes

---

## RocksDB Accommodations Overview

| Accommodation | Textbook | RocksDB | Why |
|---------------|----------|---------|-----|
| L0 files | 1 sorted run | Multiple overlapping SSTs | Avoid blocking writes during flush |
| Compaction scope | Entire level | Subset of files | Reduce latency, enable concurrency |
| File movement | Always rewrite | Trivial move if no overlap | Zero-cost when possible |
| Level sizing | Fixed T^i | Dynamic, adapts to data volume | Reduce space amplification |
| File selection | N/A | 5 priority strategies | Workload-specific optimization |
| Compaction threads | Single | Subcompactions (parallel) | Utilize multi-core |
| Memtable | Single, blocking | Pipeline of multiple | Decouple writes from flush |
| SST format | Abstract sorted array | Block-based with index/filter/compression | Production efficiency |
| Deletes | Point tombstones | Range deletions in meta block | Bulk delete support |
| Filters | Uniform | Level-aware Bloom/Ribbon | Optimize memory per level |
| Keyspaces | Single | Column families, shared WAL | Multiple logical databases |
| Compaction styles | Leveled only | Leveled, Universal, FIFO, Tiering | Workload flexibility |

---

## Level 0: Overlapping Sorted Runs

Textbook: L0 = just another level with one sorted run.

RocksDB L0 is fundamentally different:

- **Overlapping files**: Each memtable flush creates a new L0 SST. Files overlap in key ranges (like tiering at L0).
- **Ordering**: By sequence/epoch number, not key range. Newer files supersede older for same key.
- **Read impact**: Must check ALL L0 files (no binary search on boundaries). L1+ uses binary search.
- **Three-threshold write control**:
  - `level0_file_num_compaction_trigger` → start L0→L1 compaction
  - `level0_slowdown_writes_trigger` → throttle writes
  - `level0_stop_writes_trigger` → block writes
- **Intra-L0 compaction**: When L0→L1 is blocked, merge within L0 to reduce file count.

### What to Know When Implementing

- If your feature reads from L0, it must handle overlapping files (cannot assume sorted non-overlapping).
- L0 file count directly impacts write throughput via stall thresholds.
- Source: `db/version_set.cc` (ComputeCompactionScore L0 section), `db/compaction/compaction_picker_level.cc` (intra-L0).

---

## File-Level Partial Compaction

Textbook: Merge entire Level i into entire Level i+1.

RocksDB picks a **subset of files**:

1. `PickFileToCompact()` — select one file based on `CompactionPri` strategy
2. `ExpandInputsToCleanCut()` — extend to clean key boundaries (no split user keys)
3. `SetupOtherInputs()` — add only overlapping output-level files
4. Create `Compaction` with just these files

**Benefits**: smaller jobs, lower latency spikes, concurrent compactions on different key ranges, bounded by `max_compaction_bytes`.

**Trade-off**: Output-level files may be rewritten multiple times across partial compactions.

### What to Know When Implementing

- If your feature adds a new compaction picker, decide: partial (like Level) or whole-level (like FIFO/Tiering).
- `ExpandInputsToCleanCut` is essential for correctness — never split a user key across compaction boundaries.
- Source: `db/compaction/compaction_picker_level.cc`, `db/compaction/compaction_picker.cc`.

---

## Trivial Moves

Textbook: No concept. All compaction rewrites data.

RocksDB: When a file has **no overlap** with the output level, move it by updating metadata only — zero I/O.

- **L0 trivial moves**: Scan oldest→newest, expand while no Lbase overlap
- **Non-L0 trivial moves**: Start with one file, extend left/right (up to 4 files)
- Stop if overlap found or size exceeds `max_compaction_bytes`

Source: `db/compaction/compaction_picker_level.cc` (`TryPickL0TrivialMove`, `TryExtendNonL0TrivialMove`).

---

## Dynamic Level Sizing

Textbook: Fixed sizes — L_i = base * T^i. Half-empty levels waste capacity.

RocksDB (`level_compaction_dynamic_level_bytes`): Calculates sizes **backwards** from largest level.

1. Find actual data at largest non-empty level
2. Work backwards: each smaller level's target = larger / T
3. First level with target > 0 becomes `base_level_`
4. Levels between L0 and `base_level_` are skipped

**Benefit**: Eliminates wasted intermediate levels. Especially important during data growth/shrink.

Source: `db/version_set.cc` (`CalculateBaseBytes`).

---

## Compaction Priority Strategies

Textbook: No within-level file selection. RocksDB supports:

| Strategy | Behavior | Best For |
|----------|----------|----------|
| `kByCompensatedSize` | Largest file first | General |
| `kOldestLargestSeqFirst` | Oldest max-seq file | Data aging |
| `kOldestSmallestSeqFirst` | Longest since compacted | Preventing starvation |
| `kMinOverlappingRatio` | Least overlap with next level | Minimizing write amp |
| `kRoundRobin` | Cursor-based cycling | Even key range wear |

Data structure: `files_by_compaction_pri_[level]` — file indices sorted by strategy.

Source: `include/rocksdb/advanced_options.h`, `db/compaction/compaction_picker_level.cc`.

---

## Subcompactions

Textbook: Single-threaded merge. RocksDB: Split one compaction into parallel subcompactions.

- Input key range divided into disjoint subranges
- Each processed on separate thread
- `max_subcompactions` controls degree

Source: `db/compaction/subcompaction_state.h`.

---

## MemTable Pipeline and Flush

Textbook: Single memtable, blocks writes during flush.

RocksDB: Multiple memtables in a pipeline.

- **Mutable** (`mem_`): accepts writes
- **Immutable queue** (`imm_`): waiting for flush, writes continue to new mutable
- `max_write_buffer_number`: max total memtables
- `atomic_flush`: flush all CFs atomically
- Write stalls when immutable queue is full

Source: `db/flush_job.h`, `db/column_family.h`.

---

## SST File Format

Textbook: Abstract sorted array. RocksDB: Block-based table.

```
[ Data Blocks ] → sorted keys, delta-encoded, per-block compressed
[ Index Block ] → maps key range → data block offset
[ Filter Block ] → Bloom/Ribbon filter
[ Range Del Block ] → range tombstones
[ Properties Block ] → table stats
[ Meta Index ] → references to meta blocks
[ Footer ] → entry points
```

- **Delta encoding**: store only key differences from previous key
- **Restart points**: full key every `block_restart_interval` (default 16) for binary search
- **Per-block compression**: Snappy, ZSTD, LZ4 independently per block

---

## Range Deletions

Textbook: Point tombstones only. RocksDB: `DeleteRange(start, end)` in separate meta block.

- `ReadRangeDelAggregator`: combines tombstones across files during reads
- `CompactionRangeDelAggregator`: handles tombstone GC during compaction (respects snapshots)
- Non-bottommost levels: must preserve tombstones. Bottommost: can drop them.

Source: `db/range_del_aggregator.h`.

---

## Bloom Filters

Textbook: Uniform FPR across levels. RocksDB: Level-aware.

- **Full filter**: one Bloom per SST (default)
- **Partitioned filter**: per-index-partition Bloom (better cache locality)
- `bloom_before_level`: Bloom for small levels, Ribbon for large levels (Ribbon is more space-efficient)

Source: `include/rocksdb/filter_policy.h`.

---

## Column Families

Textbook: Single keyspace. RocksDB: Multiple CFs, each with its own LSM tree.

- All CFs share one WAL for crash recovery
- Each CF has separate memtable pipeline, level structure, compaction picker
- `SuperVersion` per CF: packages memtable + immutables + version into one ref-counted snapshot

Source: `db/column_family.h`.

---

## Alternative Compaction Styles

| Style | Sorted Runs/Level | Merge With Output? | Write Amp | Best For |
|-------|-------------------|---------------------|-----------|----------|
| Leveled | 1 | Yes (merge into output) | O(T*L) | General |
| Universal | Multiple (flat) | By size ratio heuristic | Lower | Write-heavy |
| FIFO | N/A (drop oldest) | No merge | O(1) | TTL data |
| Tiering | Up to T-1 | No (append to output) | O(L) | Write-optimized |

Source: `include/rocksdb/advanced_options.h`.

---

## Textbook vs RocksDB Summary

| Feature | Textbook | RocksDB |
|---------|----------|---------|
| L0 | 1 run, no overlap | Multiple overlapping SSTs |
| Compaction scope | Entire level | Partial (file-level) |
| File movement | Always rewrite | Trivial moves |
| Level sizing | Fixed T^i | Dynamic from largest level |
| File selection | Whole level | 5 priority strategies |
| Parallelism | Single-threaded | Subcompactions |
| Memtable | Single, blocking | Pipeline of multiple |
| SST format | Sorted array | Block-based with index/filter |
| Deletes | Point tombstones | Range deletions |
| Filters | Uniform | Level-aware Bloom/Ribbon |
| Keyspaces | Single | Column families |
| Compaction styles | Leveled only | Leveled, Universal, FIFO, Tiering |

---

## Key Source Files

| File | What It Contains |
|------|-----------------|
| `db/version_set.cc` | Level scoring, dynamic level sizing (`CalculateBaseBytes`), `ComputeCompactionScore` |
| `db/version_set.h` | `VersionStorageInfo` (files per level, scores, level max bytes) |
| `db/compaction/compaction_picker.h` | `CompactionPicker` base class, `ExpandInputsToCleanCut`, `SetupOtherInputs` |
| `db/compaction/compaction_picker_level.cc` | `LevelCompactionPicker`, partial file selection, trivial moves, intra-L0 |
| `db/compaction/compaction_picker_universal.cc` | `UniversalCompactionPicker`, sorted run model |
| `db/compaction/compaction.h` | `Compaction` class, `CompactionInputFiles` |
| `db/compaction/subcompaction_state.h` | Parallel subcompaction state |
| `db/column_family.h` | `ColumnFamilyData`, `SuperVersion`, memtable pipeline |
| `db/flush_job.h` | Flush pipeline, atomic flush |
| `db/range_del_aggregator.h` | Range deletion aggregation |
| `table/block_based/block_builder.h` | Block format, delta encoding, restart points |
| `include/rocksdb/filter_policy.h` | Bloom/Ribbon filter configuration |
| `include/rocksdb/advanced_options.h` | `CompactionStyle`, `CompactionPri` enums |
