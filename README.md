# RocksDB Feature Implementation Skill

An AI-agent skill that guides coding assistants through implementing new features in RocksDB. Encodes structural knowledge of the RocksDB codebase and follows a pipeline: **explore** → **implement** → **test**.

## Skill Structure

```
rockskill/
├── SKILL.md                        # Entry point / orchestrator
├── rocksdb-explore.md              # Stage 1: exploration
├── rocksdb-implement.md            # Stage 2: implementation
├── rocksdb-test.md                 # Stage 3: testing
└── references/
    ├── rocksdb-knowledge.md        # Structural knowledge
    └── rocksdb-stats-knowledge.md  # Statistics & profiling
```

`SKILL.md` is the entry point. It orchestrates the pipeline and tells the agent when to read each sub-file. The `references/` directory contains shared knowledge files used by multiple stages.

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
