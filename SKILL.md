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
| `rocksdb-critic.md` | Critic agent: reviews Implementation Report for correctness, guarantee preservation, and intent alignment |
| `rocksdb-test.md` | Stage 3: write and run tests, scaled to change size |
| `references/rocksdb-knowledge.md` | Shared reference: directory layout, interfaces, build system, conventions |
| `references/rocksdb-stats-knowledge.md` | Shared reference: statistics, profiling, metrics patterns |
| `references/rocksdb-mvcc-knowledge.md` | Shared reference: MVCC, sequence numbers, snapshots, user-defined timestamps |
| `references/rocksdb-concurrency-knowledge.md` | Shared reference: concurrency, locking, transactions |
| `references/rocksdb-lsm-knowledge.md` | Shared reference: LSM-tree structure, RocksDB accommodations vs textbook model |
| `references/rocksdb-compaction-knowledge.md` | Shared reference: flush/compaction pipelines, partial compaction, subcompactions, merge operators |
| `references/rocksdb-tiered-compaction-knowledge.md` | Shared reference: tiered sorted-run structure, tiered read path, bounded-run picking, run-id persistence and recovery |
| `references/rocksdb-table-knowledge.md` | Shared reference: SST file format, block encoding, index/filter, iterators, caching |

The pipeline stages read from the shared reference files as needed. Each stage produces output consumed by the next stage.

## Pipeline

1. **Parse the request** — Identify what the user wants to build. If the request is ambiguous, ask clarifying questions. Do not ask more than five question before starting exploration.

2. **Read `references/rocksdb-lsm-knowledge.md`** — Understand the textbook LSM-tree model and how RocksDB's implementation differs (L0 overlapping files, partial compaction, trivial moves, dynamic level sizing, etc.). This context is essential before exploring the codebase.

   If the request involves tiered compaction, overlapping sorted runs within a
   non-L0 level, tiered point/range lookup, or run-id recovery across reopen,
   also read `references/rocksdb-tiered-compaction-knowledge.md` before
   exploration.

3. **Read `rocksdb-explore.md`** — Run the exploration stage. Present a summary to the user:
   - Feature type identified
   - Reference implementations found
   - Files to create/modify
   - Metrics opportunities (if any)
   - **PAUSE and ask the user to confirm before proceeding.**

4. **Generate Implementation Report** — Based on the exploration report and `rocksdb-implement.md`, produce a self-contained implementation plan that a **fresh agent with zero codebase context** can execute step-by-step. See the "Implementation Report Requirements" section in `rocksdb-implement.md` for the full template. The report must include:
   - **Reference divergence analysis**: KEEP/CHANGE table for each reference implementation
   - **File structure**: every file to create/modify with approximate line numbers
   - **Bite-sized tasks**: each step is a single action (2-5 minutes) with complete code — no pseudocode, no placeholders, no `...`
   - **Behavioral specs**: for each component, what it does and what it does NOT do
   - **MVCC / timestamp / transaction considerations** (if applicable)
   - **Self-review**: quick scan for divergence coverage, placeholders, and type consistency before presenting

5. **Critic Review** — Launch an independent critic agent (following `rocksdb-critic.md`) to evaluate the Implementation Report. The critic checks:
   - **Intent alignment**: does the plan match what the user asked for?
   - **Guarantee preservation**: do modified modules still honor snapshot isolation, key ordering, write atomicity, reader-writer isolation, compaction correctness, and LSM invariants?
   - **Correctness**: error handling, thread safety, data races, deadlocks
   - **Performance**: unnecessary allocations on hot paths, missing optimizations
   - **Reference divergence validation**: are KEEP/CHANGE decisions in the divergence analysis sound?

   The critic outputs issues as **BLOCKER** (must fix) or **WARNING** (user should know). If BLOCKERs are found, revise the Implementation Report to address them before presenting to the user.

   Present the Implementation Report together with the Critic Review to the user.
   - **PAUSE and ask the user to confirm before proceeding.**

6. **Execute implementation** — Follow the confirmed Implementation Report. Write code per `rocksdb-implement.md`. Run `make format-auto` at the end.

7. **Read `rocksdb-test.md`** — Write tests, build, run, and perform dedup review. Fix any failures before proceeding.

8. **Final summary** — Report:
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
2. **After Implementation Report + Critic Review (step 5)** — User reviews the implementation plan and the critic's findings before any code is written.

Steps 6-8 proceed without interruption unless errors occur.
