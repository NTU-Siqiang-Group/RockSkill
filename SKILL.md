---
name: rocksdb-feature
description: Use when implementing a new feature in RocksDB — orchestrates the full pipeline from exploration through implementation and testing
---

# rocksdb-feature

## Overview

Entry point for RocksDB feature work. Manages the explore → implement → test pipeline, coordinating the three stage skills to take a user request from initial analysis through working, tested code.

## Skill Structure

This skill directory contains the following files:

| File | Purpose |
|------|---------|
| `SKILL.md` | This file — orchestrator and entry point |
| `rocksdb-explore.md` | Stage 1: classify feature type, find references, produce exploration report |
| `rocksdb-implement.md` | Stage 2: write code following feature-type-specific patterns |
| `rocksdb-test.md` | Stage 3: write and run tests, scaled to change size |
| `references/rocksdb-knowledge.md` | Shared reference: directory layout, interfaces, build system, conventions |
| `references/rocksdb-stats-knowledge.md` | Shared reference: statistics, profiling, metrics patterns |
| `references/rocksdb-mvcc-knowledge.md` | Shared reference: MVCC, sequence numbers, snapshots, user-defined timestamps |
| `references/rocksdb-concurrency-knowledge.md` | Shared reference: concurrency, locking, transactions |
| `references/rocksdb-lsm-knowledge.md` | Shared reference: LSM-tree structure, RocksDB accommodations vs textbook model |
| `references/rocksdb-compaction-knowledge.md` | Shared reference: flush/compaction pipelines, partial compaction, subcompactions, merge operators |

The pipeline stages read from the shared reference files as needed. Each stage produces output consumed by the next stage.

## Pipeline

1. **Parse the request** — Identify what the user wants to build. If the request is ambiguous, ask clarifying questions. Do not ask more than five question before starting exploration.

2. **Read `references/rocksdb-lsm-knowledge.md`** — Understand the textbook LSM-tree model and how RocksDB's implementation differs (L0 overlapping files, partial compaction, trivial moves, dynamic level sizing, etc.). This context is essential before exploring the codebase.

3. **Read `rocksdb-explore.md`** — Run the exploration stage. Present a summary to the user:
   - Feature type identified
   - Reference implementations found
   - Files to create/modify
   - Metrics opportunities (if any)
   - **PAUSE and ask the user to confirm before proceeding.**

4. **Generate Implementation Report** — Based on the exploration report and `rocksdb-implement.md`, produce a detailed implementation plan. See the "Implementation Report Requirements" section in `rocksdb-implement.md` for the full template. Present it to the user:
   - Step-by-step implementation order (which files to modify in what sequence)
   - Per-method behavioral spec: for each method, what it does, what it does NOT do, inputs/outputs
   - Reference divergence analysis: which behaviors from reference implementations to keep vs change, and why
   - Pseudocode for core logic (non-trivial methods)
   - Key design decisions (e.g., mutable vs immutable option, which base class to extend)
   - MVCC / timestamp / transaction considerations (if applicable, referencing `references/rocksdb-mvcc-knowledge.md` and `references/rocksdb-concurrency-knowledge.md`)
   - Metrics to add (if any)
   - Potential risks or open questions
   - **PAUSE and ask the user to confirm before proceeding.**

5. **Execute implementation** — Follow the confirmed Implementation Report. Write code per `rocksdb-implement.md`. Run `make format-auto` at the end.

6. **Read `rocksdb-test.md`** — Write tests, build, run, and perform dedup review. Fix any failures before proceeding.

7. **Final summary** — Report:
   - What was built (files created and modified)
   - What was tested (test names, pass/fail status)
   - Follow-up items for the user to handle manually:
     - "Consider adding stress test coverage" (if new option/feature)
     - "Consider adding db_bench support" (if performance-related)
     - "Consider adding a release note to `unreleased_history/`"
     - "Consider adding Java/C API bindings"

## Error Handling

- If exploration finds no matching feature type → stop and ask the user for guidance.
- If implementation hits a compilation error → fix before moving to the test stage.
- If tests fail → diagnose and fix before reporting completion.

## User Checkpoints

Two checkpoints:
1. **After Exploration Report (step 3)** — User confirms the feature type, reference implementations, and file list before planning.
2. **After Implementation Report (step 4)** — User confirms the concrete implementation plan before any code is written.

Steps 5-7 proceed without interruption unless errors occur.
