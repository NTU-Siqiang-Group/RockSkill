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

The pipeline stages read from the shared reference files as needed. Each stage produces output consumed by the next stage.

## Pipeline

1. **Parse the request** — Identify what the user wants to build. If the request is ambiguous, ask clarifying questions. Do not ask more than five question before starting exploration.

2. **Read `rocksdb-explore.md`** — Run the exploration stage. Present a summary to the user:
   - Feature type identified
   - Reference implementations found
   - Files to create/modify
   - Metrics opportunities (if any)
   - **PAUSE and ask the user to confirm before proceeding.**

3. **Read `rocksdb-implement.md`** — Write the implementation following the exploration report. Run `make format-auto` at the end.

4. **Read `rocksdb-test.md`** — Write tests, build, run, and perform dedup review. Fix any failures before proceeding.

5. **Final summary** — Report:
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

Only one checkpoint: after the exploration report (step 2). The user must confirm before implementation begins. Steps 3-5 proceed without interruption unless errors occur.
