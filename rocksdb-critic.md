---
name: rocksdb-critic
description: Use after generating an Implementation Report to review it for correctness, guarantee preservation, and alignment with the user's intent before code is written
---

# RocksDB Implementation Report Critic

## Overview

This skill reviews an Implementation Report before the user sees it. It catches semantic errors, guarantee violations, and misalignment with the user's intent — issues that are cheap to fix in a plan but expensive to fix in code.

## Inputs

The critic receives three artifacts from the main pipeline:

1. **User's original request** — what they asked to build
2. **Exploration Report** — feature type, reference implementations, file list
3. **Implementation Report** — per-method specs, divergence analysis, and concrete implementation steps

## Evaluation Axes

### 1. Intent Alignment

- Does the Implementation Report address what the user actually asked for?
- Are there gaps between the user's request and what the plan delivers?
- Does the plan introduce unnecessary scope (features the user didn't ask for)?
- Are the key design decisions consistent with the user's stated goals?

### 2. Guarantee Preservation

For each module being modified, verify that existing guarantees are preserved. Reference the relevant knowledge file for each check:

| Guarantee Area | Reference | Key Questions |
|----------------|-----------|---------------|
| Snapshot isolation | `references/rocksdb-mvcc-knowledge.md` | Do reads still respect snapshot sequence numbers? Can compaction still safely drop versions? |
| Key ordering | `references/rocksdb-mvcc-knowledge.md` | Is InternalKey sort order (user key asc, seqno desc) preserved? Are timestamp bytes accounted for? |
| Write atomicity | `references/rocksdb-concurrency-knowledge.md` | Do writes still flow through WriteBatch → sequence allocation → MemTable? |
| Reader-writer isolation | `references/rocksdb-concurrency-knowledge.md` | Do readers still use SuperVersion without holding DB mutex? |
| Compaction correctness | `references/rocksdb-compaction-knowledge.md` | Are CompactionIterator drop rules still valid? Are active snapshots protected? |
| LSM invariants | `references/rocksdb-lsm-knowledge.md` | Are level size ratios, file overlap rules, and sorted run properties maintained (or intentionally changed with justification)? |

### 3. Correctness

- **Error handling**: Does every method that can fail return `Status`? Are errors propagated, not silently ignored?
- **Thread safety**: Are new classes/methods documented for thread safety? Are synchronization primitives (mutex, atomics, CAS) used correctly?
- **Data races**: Can concurrent readers and writers access the same state without proper synchronization?
- **Deadlocks**: Are locks acquired in a consistent order? Could the change introduce a cycle in the lock ordering?
- **Edge cases**: Are boundary conditions handled (empty input, single element, maximum values)?

### 4. Performance

- **Hot path allocations**: Does the plan introduce heap allocations (`new`, `malloc`, `std::string` copies) on hot paths (read, write, compaction inner loops)?
- **Unnecessary copies**: Are `Slice` / `std::string_view` used where possible instead of `std::string`?
- **Branch hints**: Are `LIKELY`/`UNLIKELY` macros used on hot-path branches?
- **Benchmark flag**: If the change is performance-sensitive (new compaction strategy, new memtable, new cache policy), flag that benchmark results should be collected.

### 5. Reference Divergence Validation

This is the most critical check — it catches blind copy-paste errors.

For each **CHANGE** in the divergence analysis:
- Is the reason for diverging from the reference valid?
- Does the change actually achieve its stated purpose?
- Could the change introduce a semantic error? (e.g., removing output-level overlap gathering in a tiering compaction is correct; removing input-level cleanup is not)

For each **KEEP** in the divergence analysis:
- Is the kept behavior actually appropriate for the new feature?
- Could keeping this behavior silently make the new feature behave like the reference instead of like the intended design? (This is exactly what happened in v1 tiering: keeping `GetOverlappingInputs` on the output level made tiering behave like leveling)

## Output Format

```
## Critic Review

### Intent Alignment
[Assessment — aligned / gaps identified]

### Guarantee Preservation
[For each relevant guarantee: PASS or ISSUE with explanation]

### Correctness
[Issues found or "No correctness issues found"]

### Performance
[Issues found or "No performance issues found"]

### Reference Divergence Validation
[For each CHANGE/KEEP entry: validated or flagged with explanation]

### Summary
- **BLOCKERs**: [count] — must be fixed before proceeding
  - [BLOCKER description and suggested fix]
- **WARNINGs**: [count] — user should be aware
  - [WARNING description]

[If no issues: "No issues found. Implementation Report is ready for user review."]
```

## Issue Severity

| Severity | Definition | Action |
|----------|-----------|--------|
| **BLOCKER** | Would cause incorrect behavior, data loss, guarantee violation, or semantic error if implemented as-is | Pipeline auto-revises the Implementation Report before presenting to user |
| **WARNING** | Potential concern that may or may not be an issue depending on context — missing optimization, unclear edge case, debatable design choice | Included in the report for the user to evaluate |

## What This Critic Does NOT Do

- Does not review actual code (only the Implementation Report)
- Does not run tests or benchmarks
- Does not second-guess the user's feature choice or overall design direction
- Does not flag style issues (that's what `make format-auto` is for)
- Does not expand scope — only validates what the report claims to do
