---
name: rocksdb-explore
description: Use when starting a new RocksDB feature implementation — explores the codebase, classifies feature type, finds reference implementations, and produces an exploration report before any code is written
---

# RocksDB Explore — Codebase Exploration Stage

## Overview

This skill runs before any code is written. It explores the RocksDB codebase, classifies the requested feature, finds reference implementations, and produces a structured exploration report consumed by `rocksdb-implement.md`.

## Feature Type Classification

Match the user's request to one of the following feature types:

- **Option (CF/DB/BlockBased)** — Add a new configuration option to ColumnFamilyOptions, DBOptions, or BlockBasedTableOptions.
- **Remove deprecated option** — Remove an option that has been deprecated, preserving backward compatibility for option file loading.
- **Public API** — Add a new method to the public `DB` interface.
- **Compaction strategy** — Implement a new compaction picking algorithm.
- **Memtable implementation** — Implement a new in-memory data structure for the memtable.
- **Table format** — Implement a new on-disk table format (reader, builder, factory).
- **Cache policy** — Implement a new cache eviction or admission policy.
- **File system / Env extension** — Implement a new file system or environment abstraction.
- **Filter policy** — Implement a new filter (e.g., Bloom alternative) for key lookup acceleration.
- **If unclear, ask the user.** — If the request does not clearly match any of the above types, ask one clarifying question before proceeding.

## Steps

1. **Classify the feature type** — Match the user's request to the feature type list above. If ambiguous, ask one clarifying question.

2. **Read the relevant interface** — Look up the base class and header from `references/rocksdb-knowledge.md` "Key Interfaces per Feature Type" table. Read the header file. Note: required virtual methods, key data structures, thread-safety contracts, lifetime/ownership patterns.

3. **Find existing implementations** — Locate 1-2 existing implementations of the same feature type. Read them to understand: constructor patterns, method implementations, error handling patterns, how they interact with the rest of the system. These serve as templates.

4. **Trace the registration/factory pattern** — Find where existing implementations of this type are instantiated. Follow the factory method or enum-based dispatch. Note the exact registration point.

5. **Identify reusable utilities** — Search `util/`, `monitoring/`, and nearby code for 2-3 relevant helpers. Do not exhaustively explore. Look for: encoding/decoding helpers, hash functions, string utilities, existing test fixtures.

6. **Build the file modification list** — Enumerate every file to create or modify. Group into: (a) New files — header + implementation, (b) Existing files to modify — registration, build system, options.

7. **Assess metrics opportunity** — Reference `references/rocksdb-stats-knowledge.md`. Decide if the feature warrants: new tickers (for countable events), new histograms (for latency/size distributions), new PerfContext counters (for per-operation tracking). Note which macros to use and where.

## Output Format

Produce the following structured exploration report:

```
## Exploration Report

### Feature Type
[type name]

### Reference Implementations
- [file1]: [what it demonstrates]
- [file2]: [what it demonstrates]

### Interface Contract
- Base class: [name] in [header]
- Required methods: [list]
- Thread-safety: [notes]

### File List
**New files:**
- [path]: [purpose]

**Files to modify:**
- [path]: [what change]

### Metrics Plan
- Tickers: [list or "none needed"]
- Histograms: [list or "none needed"]
- PerfContext: [list or "none needed"]
```

---

References: `references/rocksdb-knowledge.md` for interfaces and build system, `references/rocksdb-stats-knowledge.md` for metrics patterns.

Next stage: pass exploration report to `rocksdb-implement.md`.
