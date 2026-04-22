---
name: rocksdb-tiered-compaction-knowledge
description: >-
  Use when implementing or modifying tiered compaction, tiered reads, sorted
  run metadata, or reopen/recovery logic — reference for tiered structure,
  run-id persistence, bounded-run picking, and validation hazards
---

# RocksDB Tiered Compaction Reference

## Contents

- [Mental Model](#mental-model)
- [Tree Materialization](#tree-materialization)
- [Read Path](#read-path)
- [Iterator Path](#iterator-path)
- [Compaction Trigger and Picking](#compaction-trigger-and-picking)
- [Run ID Generation and Persistence](#run-id-generation-and-persistence)
- [Recovery After Reopen](#recovery-after-reopen)
- [Hot Spots and Hazards](#hot-spots-and-hazards)
- [Validation Checklist](#validation-checklist)
- [Key Source Files](#key-source-files)

## Mental Model

Tiered compaction changes the meaning of a non-L0 level:

- **Leveled**: one non-overlapping sorted run per non-L0 level
- **Tiered**: multiple sorted runs may exist in the same non-L0 level

Invariants:

- inside one sorted run, files are non-overlapping and sorted by smallest key
- different runs in the same level may overlap in key range
- runs are ordered by recency, newest first
- `L0` remains special and naturally tiered; each file is effectively its own run

Current policy:

- trigger compaction when run count in a level reaches or exceeds `T`
- `T` is taken from `max_bytes_for_level_multiplier`
- automatic compaction picks at most `T` runs, oldest first
- selected runs are merged into one new run in the next level
- manual compact-range for tiered still compacts the whole level if the range overlaps

## Tree Materialization

The persistent identity of a run is `sorted_run_id`.

The in-memory tiered structure is not stored as a separate on-disk tree object.
Instead, it is derived from per-file metadata:

- each file stores `sorted_run_id`
- files in a level are grouped by `sorted_run_id`
- files inside a run are sorted by smallest key
- runs are sorted by `sorted_run_id` descending, which represents recency

Main code:

- run metadata definitions:
  `db/version_set.h`
- run regrouping:
  `VersionStorageInfo::GenerateLevelSortedRunsBrief()` in `db/version_set.cc`

Implication:

- if `sorted_run_id` is lost or not persisted, the system cannot reconstruct run
  boundaries after reopen

## Read Path

Do **not** retrofit tiered behavior into the leveled read path by weakening its
assumptions globally. Keep explicit tiered branching.

Point lookup:

- use `TieredFilePicker` in `db/version_set.cc`
- walk levels in order
- for each non-L0 level, search runs in recency order
- inside a run, reuse non-overlapping-file search logic
- stop as soon as a newer visible result is found

L0 is special:

- do not treat L0 as one non-overlapping run
- check L0 files one by one in L0 order

Correctness rule:

- if the same user key exists in multiple runs in one level, the newest run must
  be searched first so the latest visible value wins

## Iterator Path

Tiered range iteration also needs explicit branching.

User iterators:

- `AddTieredIteratorsForLevel()` in `db/version_set.cc`
- `L0`: add one table iterator per file
- `L1+`: add one run iterator per run
- a run iterator may reuse `LevelIterator`, because the files **inside one run**
  are still non-overlapping

Compaction iterators:

- `VersionSet::MakeInputIterator()` in `db/version_set.cc`
- when tiered compaction selects only a subset of runs, the input iterator must
  iterate only those selected runs
- grouping compaction input files by `sorted_run_id` is required after switching
  from whole-level picking to bounded-run picking

Hazard:

- older code paths may assume “tiered compaction input level” means “entire
  source level”
- after bounded-run picking, that assumption is wrong

## Compaction Trigger and Picking

Tiered compaction score is based on **run count**, not bytes.

Main scoring code:

- `VersionStorageInfo::ComputeCompactionScore()` in `db/version_set.cc`

Main counting code:

- `VersionStorageInfo::NumTieredRunsForCompaction()` in `db/version_set.cc`

Current scoring behavior:

- `run_limit = max_bytes_for_level_multiplier`
- if `num_runs >= run_limit`, score is `num_runs / run_limit`
- otherwise score is `0`

Automatic picking:

- `TieredCompactionPicker::PickCompaction()` in
  `db/compaction/compaction_picker_tiered.cc`
- pick highest-score eligible level
- compact at most `T` runs, oldest first
- `L0`: pick oldest files
- `L1+`: pick oldest runs, which are at the back of the recency-sorted run list

Manual picking:

- `PickCompactionForCompactRange()` in the same file
- if a range overlaps a tiered level, current implementation picks the whole level

Important distinction:

- automatic tiered compaction is bounded by `T`
- manual tiered compact-range is whole-level by design

## Run ID Generation and Persistence

New runs need a new unique `sorted_run_id`.

Generation:

- `VersionSet::NewSortedRunId()` in `db/version_set.h`
- assigned in `Compaction::FinalizeInputInfo()` in `db/compaction/compaction.cc`

Persistence:

- every output file of the same tiered compaction gets the same
  `output_sorted_run_id`
- this is written into manifest edits through `VersionEdit`
- encoded/decoded in `db/version_edit.cc`

Recognition:

- files that share one `sorted_run_id` are reconstructed as one run after
  regrouping

## Recovery After Reopen

Recovery depends on manifest persistence of `sorted_run_id`.

On reopen:

1. replay manifest edits
2. recover each file’s `sorted_run_id`
3. regroup files by run id to rebuild run structure
4. compute `next_sorted_run_id_` as `max_recovered_run_id + 1`

Main code:

- manifest decode: `db/version_edit.cc`
- next run id recovery: `db/version_set.cc`

External observability:

- `LiveFileMetaData.sorted_run_id` is exposed in `include/rocksdb/metadata.h`
- `VersionSet::GetLiveFilesMetaData()` populates it

This lets integration tools reconstruct the tiered tree after reopen.

## Hot Spots and Hazards

These are the main places where tiered work can silently break correctness:

1. **Shared leveled assumptions**
   - many helpers assume one globally ordered non-L0 level
   - tiered levels violate that across runs

2. **L0 special cases**
   - L0 cannot reuse run-level binary search or run-level iterators

3. **Boundary computations**
   - helpers that summarize a level by first/last file are wrong for tiered
     multi-run input
   - scan all input files for tiered compactions

4. **Compaction input iteration**
   - after bounded-run picking, compaction input iteration must respect the
     selected subset of runs, not all files in the level

5. **Manifest persistence**
   - forgetting to encode/decode `sorted_run_id` breaks reopen semantics

6. **Recency order maintenance**
   - point lookup assumes runs are searched newest first
   - changing regrouping or run-id ordering can silently return stale values

## Validation Checklist

When changing tiered behavior, validate all of these:

- multiple runs can coexist in one non-L0 level
- point lookup over overlapping runs returns the newest value
- iterator merges multiple runs in sorted key order
- iterator handles multiple files inside one run
- auto compaction assigns distinct run IDs to new runs
- bounded picker chooses at most `T` oldest runs
- manual compact-range still behaves as expected
- reopen reconstructs the same run structure from manifest state
- integration run shows the intended multi-level run shape

Useful existing tests:

- `TieredVersionStorageInfoTest.*`
- `TieredVersionReadTest.*`
- `VersionSetTest.RecoverSortedRunIdFromManifest`
- `CompactionPickerTest.Tiered*`
- `TieredCompactionTest.*`

## Key Source Files

- `db/version_set.h`
- `db/version_set.cc`
- `db/compaction/compaction_picker_tiered.h`
- `db/compaction/compaction_picker_tiered.cc`
- `db/compaction/compaction.cc`
- `db/compaction/compaction_job.cc`
- `db/db_impl/db_impl_compaction_flush.cc`
- `db/version_edit.h`
- `db/version_edit.cc`
- `include/rocksdb/metadata.h`
