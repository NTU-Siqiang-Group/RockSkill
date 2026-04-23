# RocksDB Feature Implementation Skill

An AI-agent skill that guides coding assistants through implementing new features in RocksDB. Encodes structural knowledge of the RocksDB codebase — including LSM-tree internals, MVCC, concurrency, compaction mechanics, and merge operators — and follows a 7-step pipeline with two user checkpoints.

## Workflow

```
1. Parse request
2. Load background knowledge           ← references/
3. Exploration Report                  ← rocksdb-explore.md
   ✓ CHECKPOINT: user confirms
4. Implementation Report               ← rocksdb-implement.md
   (per-method spec, divergence analysis, concrete implementation steps)
   ✓ CHECKPOINT: user confirms
5. Execute implementation
6. Test                                ← rocksdb-test.md
7. Final summary
```

The Implementation Report (step 4) requires:
- **Per-method behavioral spec**: what each method does AND does NOT do
- **Reference divergence analysis**: which patterns from reference implementations to keep vs change
- **Pseudocode** for core logic, reviewed before code is written

## Knowledge Base

The `references/` directory provides domain knowledge that the agent reads before and during implementation:

| Reference | Covers |
|-----------|--------|
| `rocksdb-lsm-knowledge.md` | LSM-tree model, RocksDB accommodations (L0 overlapping files, partial compaction, dynamic level sizing, etc.) |
| `rocksdb-compaction-knowledge.md` | Flush/compaction pipelines, partial compaction mechanics, subcompactions, CompactionIterator state machine, MergeOperator |
| `rocksdb-mvcc-knowledge.md` | Sequence numbers, InternalKey format, snapshots, user-defined timestamps, compaction GC rules |
| `rocksdb-concurrency-knowledge.md` | Write path (leader-follower batching), read path (SuperVersion), DB mutex scope, transactions (pessimistic/optimistic), 2PC |
| `rocksdb-table-knowledge.md` | SST file format, block encoding, index/filter blocks, iterator semantics, caching, prefetching |
| `rocksdb-knowledge.md` | Directory layout, key interfaces per feature type, build system, coding conventions |
| `rocksdb-stats-knowledge.md` | Statistics framework (tickers, histograms), PerfContext, IOStatsContext, adding new metrics |

## Skill Structure

```
rockskill/
├── SKILL.md                                    # Entry point / orchestrator
├── rocksdb-explore.md                          # Stage: exploration
├── rocksdb-implement.md                        # Stage: implementation (+ report requirements)
├── rocksdb-test.md                             # Stage: testing
└── references/
    ├── rocksdb-lsm-knowledge.md                # LSM-tree & RocksDB accommodations
    ├── rocksdb-compaction-knowledge.md          # Flush, compaction, merge operators
    ├── rocksdb-table-knowledge.md               # SST format, blocks, iterators, caching
    ├── rocksdb-mvcc-knowledge.md                # MVCC, snapshots, timestamps
    ├── rocksdb-concurrency-knowledge.md         # Concurrency, transactions
    ├── rocksdb-knowledge.md                     # Codebase structure, build system
    └── rocksdb-stats-knowledge.md               # Statistics & profiling
```

## Supported Feature Types

- **Options** (CF/DB/BlockBased) — add new configuration options
- **Public APIs** — add methods to the `DB` interface
- **Compaction strategies** — new compaction picking algorithms
- **Memtable implementations** — new in-memory data structures
- **Table formats** — new on-disk SST formats
- **Cache policies** — new eviction/admission policies
- **File system / Env extensions** — new storage backends
- **Filter policies** — new key lookup filters (Bloom alternatives)
- **Remove deprecated options** — safe option removal with backward compatibility

## Usage

### Claude Code

Copy the `rockskill/` directory into your skills location:

```bash
cp -r rockskill ~/.claude/skills/rockskill
```

Then invoke with `/rockskill` in any Claude Code session.

### Cursor

Add to `.cursor/rules/`:

```bash
cp rockskill/SKILL.md .cursor/rules/rocksdb-feature.mdc
```

Or reference in `.cursorrules`: "When implementing RocksDB features, follow the instructions in `rockskill/SKILL.md`"

### Windsurf

Reference `rockskill/SKILL.md` in `.windsurfrules` or your workspace instructions.

### GitHub Copilot

Reference in `.github/copilot-instructions.md`:

```markdown
When implementing new features in RocksDB, follow the workflow in rockskill/SKILL.md.
```

### Other Tools

Point your AI coding assistant at `rockskill/SKILL.md` as the entry point. It references all other files by relative path.

## Test Case

A test case of using rockskill to implement Tiering compaction in RocksDB is available at [RockSkill-cases](https://github.com/NTU-Siqiang-Group/RockSkill-cases). It includes the full pipeline output (exploration report, implementation report, final summary) and benchmark results.

## Scope

Covers **core C++ implementation and testing**. Intentionally excludes release notes, Java/C bindings, db_bench integration, and stress tests (mentioned as follow-up items in the orchestrator's summary).

## License

MIT License
