# RocksDB Feature Implementation Skills

A set of AI-agent skills that guide coding assistants through implementing new features in RocksDB. The skills encode structural knowledge of the RocksDB codebase and follow a pipeline: **explore** → **implement** → **test**.

## What's Included

### Pipeline Skills

| Skill | File | Purpose |
|---|---|---|
| **rocksdb-feature** | `skills/rocksdb-feature.md` | Orchestrator — entry point that runs the full pipeline |
| **rocksdb-explore** | `skills/rocksdb-explore.md` | Stage 1: classify feature type, find reference implementations, build file list |
| **rocksdb-implement** | `skills/rocksdb-implement.md` | Stage 2: write code following feature-type-specific patterns |
| **rocksdb-test** | `skills/rocksdb-test.md` | Stage 3: write and run tests, scaled to change size |

### Reference Files

| File | Purpose |
|---|---|
| `references/rocksdb-knowledge.md` | RocksDB directory layout, key interfaces, build system, coding conventions |
| `references/rocksdb-stats-knowledge.md` | Statistics framework, PerfContext, IOStatsContext, how to add metrics |

### Supported Feature Types

The skills cover these RocksDB extension points:

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

Start by pointing your AI coding assistant at `skills/rocksdb-feature.md` (the orchestrator), or invoke a specific stage directly. The pipeline skills reference the files in `references/` for structural knowledge.

### Claude Code

Copy skill directories into `~/.claude/skills/` (personal) or `.claude/skills/` (project-level). Each skill needs to be a directory with a `SKILL.md` file:

```bash
# Create skill directories
for skill in rocksdb-feature rocksdb-explore rocksdb-implement rocksdb-test; do
  mkdir -p ~/.claude/skills/$skill
  cp skills/$skill.md ~/.claude/skills/$skill/SKILL.md
done

# Copy reference files into skills that need them
for skill in rocksdb-explore rocksdb-implement rocksdb-test; do
  cp references/rocksdb-knowledge.md ~/.claude/skills/$skill/
done
for skill in rocksdb-explore rocksdb-implement; do
  cp references/rocksdb-stats-knowledge.md ~/.claude/skills/$skill/
done
```

Then invoke with `/rocksdb-feature` in any Claude Code session.

### Cursor

Add the skill files to your project's context. In `.cursor/rules/`, create a rule file that includes the skill content:

```bash
# Option 1: Copy as a rule file
cp skills/rocksdb-feature.md .cursor/rules/rocksdb-feature.mdc

# Option 2: Reference in your existing rules
# Add to .cursorrules: "When implementing RocksDB features, follow the instructions in skills/rocksdb-feature.md"
```

### Windsurf

Add the skill content to your Windsurf rules in `.windsurfrules` or reference the files in your workspace instructions.

### GitHub Copilot

Reference the skills in `.github/copilot-instructions.md`:

```markdown
When implementing new features in RocksDB, follow the workflow described in skills/rocksdb-feature.md.
Reference files for codebase structure: references/rocksdb-knowledge.md, references/rocksdb-stats-knowledge.md.
```

### Other Tools

Any AI coding assistant that supports custom instructions or context files can use these skills. Point the tool at the relevant `.md` files:

1. For the full workflow: `skills/rocksdb-feature.md`
2. For specific stages: `skills/rocksdb-explore.md`, `skills/rocksdb-implement.md`, `skills/rocksdb-test.md`
3. For reference: `references/rocksdb-knowledge.md`, `references/rocksdb-stats-knowledge.md`

## Scope

The skills cover **core C++ implementation and testing**. They intentionally exclude:

- Release notes
- Java/C API bindings
- db_bench integration
- Stress test code

These are mentioned as follow-up items in the orchestrator's final summary.

## File Structure

```
rockskill/
├── README.md
├── skills/
│   ├── rocksdb-feature.md          # Orchestrator
│   ├── rocksdb-explore.md          # Stage 1: Exploration
│   ├── rocksdb-implement.md        # Stage 2: Implementation
│   └── rocksdb-test.md             # Stage 3: Testing
├── references/
│   ├── rocksdb-knowledge.md        # Structural knowledge
│   └── rocksdb-stats-knowledge.md  # Statistics & profiling knowledge
└── docs/
    ├── design.md                   # Design spec
    └── plan.md                     # Implementation plan
```

## License

[Add your license here]
