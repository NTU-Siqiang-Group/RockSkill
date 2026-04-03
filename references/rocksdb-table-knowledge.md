---
name: rocksdb-table-knowledge
description: >-
  Use when implementing features that interact with SST files, block-based
  tables, table readers/builders, block cache, filters, or iterators —
  reference for RocksDB's table format, caching, and iterator guarantees
---

# RocksDB Table (SST File) Reference

## Contents

- [High-Level Structure](#high-level-structure)
- [Key Code Locations](#key-code-locations)
- [Guarantees](#guarantees)
- [Table Interfaces](#table-interfaces)
- [SST File Layout](#sst-file-layout)
- [Block Format and Encoding](#block-format-and-encoding)
- [Checksums](#checksums)
- [Index Block](#index-block)
- [Filter Block](#filter-block)
- [Iterator Semantics](#iterator-semantics)
- [Caching and Prefetching](#caching-and-prefetching)
- [Meta Blocks and Table Properties](#meta-blocks-and-table-properties)
- [Key Configuration Options](#key-configuration-options)
- [Key Source Files](#key-source-files)

---

## High-Level Structure

SST files are the immutable, sorted data files at each LSM level. The default format is the **Block-Based Table**:

```
SST File = [ Data Blocks | Filter Block | Index Block | Meta Blocks | Footer ]
```

| Component | What It Does |
|-----------|-------------|
| **Data Blocks** | Sorted KV pairs, delta-encoded, per-block compressed |
| **Index Block** | Maps key ranges → data block offsets (binary/hash/partitioned) |
| **Filter Block** | Bloom/Ribbon filter for fast "key not here" checks |
| **Meta Blocks** | Table properties, range tombstones, compression dictionary |
| **Footer** | Entry point: metaindex handle, checksum type, format version |

Three core interfaces:
- `TableFactory` — creates readers and builders
- `TableBuilder` — writes sorted KV pairs → SST file (used by flush + compaction)
- `TableReader` — reads SST file via `Get()` and `NewIterator()` (used by all read paths)

---

## Key Code Locations

| Area | Primary Files |
|------|--------------|
| Interfaces | `include/rocksdb/table.h` |
| Factory + Options | `table/block_based/block_based_table_factory.h` |
| Builder | `table/block_based/block_based_table_builder.h` |
| Reader | `table/block_based/block_based_table_reader.h` |
| Block encoding | `table/block_based/block_builder.h`, `table/block_based/block.h` |
| Iterator | `table/block_based/block_based_table_iterator.h` |
| Index | `table/block_based/index_builder.h` |
| Filter | `table/block_based/filter_block.h`, `full_filter_block.h`, `partitioned_filter_block.h` |
| Cache | `table/block_based/block_cache.h`, `table/block_based/block_prefetcher.h` |
| Format | `table/format.h` (BlockHandle, Footer, ChecksumType) |
| Meta blocks | `table/meta_blocks.h` |
| Properties | `include/rocksdb/table_properties.h` |
| Filter policy | `include/rocksdb/filter_policy.h` |

---

## Guarantees

### Data Integrity

| Guarantee | Mechanism |
|-----------|-----------|
| Per-block checksum | Every block verified on read (CRC32c / xxHash / XXH3) |
| Position-aware checksums (v6+) | Context checksums detect misplaced blocks |
| Sorted key invariant | `TableBuilder::Add()` asserts key ordering |
| Compression correctness | Decompressed size validated against original |

### Iterator Correctness

| Guarantee | Mechanism |
|-----------|-----------|
| Seek returns first key >= target | Binary search on index + within block |
| Next/Prev cross block boundaries correctly | Index iterator advances to adjacent blocks |
| No key skipping | Restart points ensure all keys reachable |
| Consistent snapshot | Pinned blocks not freed while iterator alive |

### Cache Correctness

| Guarantee | Mechanism |
|-----------|-----------|
| Unique cache keys | Session ID + file number + block offset |
| No stale entries | Cache keys change on file replacement |
| Thread-safe | Block cache internally synchronized |
| Memory-bounded | LRU/Clock eviction enforces capacity |

### Performance

| Property | Mechanism |
|----------|-----------|
| O(1) filter check per block | Bloom/Ribbon avoids unnecessary block reads |
| O(log N) seek within block | Binary search across restart points |
| Adaptive prefetching | Sequential access detected, readahead grows |
| Per-block compression | Reduces I/O without affecting in-memory layout |

---

## Table Interfaces

### TableBuilder — Write Path

```cpp
class TableBuilder {
  void Add(const Slice& key, const Slice& value);  // Must be sorted order
  Status Finish();    // Write index, filter, meta, footer
  void Abandon();     // Discard partially built file
  uint64_t FileSize() const;
  TableProperties GetTableProperties() const;
};
```

**Contract**: Keys added in InternalKey sorted order. Violation = corruption.

### TableReader — Read Path

```cpp
class TableReader {
  InternalIterator* NewIterator(const ReadOptions&, ...);
  Status Get(const ReadOptions&, const Slice& key, GetContext*, ...);
  Status VerifyChecksum(const ReadOptions&, ...);
  uint64_t ApproximateOffsetOf(const ReadOptions&, const Slice& key, ...);
};
```

### What to Know When Implementing

- If your feature adds a new table format: implement `TableFactory`, `TableReader`, `TableBuilder`. Register via `Options::table_factory`.
- If your feature modifies the block-based table: the reader/builder are large classes. Focus on the specific block type you're changing.
- `TableBuilder::Add()` is called by both flush (`FlushJob`) and compaction (`CompactionJob`). Changes affect both paths.

---

## SST File Layout

```
┌───────────────────────────┐
│  Data Block 1             │  Sorted keys, delta-encoded, compressed
│  Data Block 2             │  Each block: data + 1-byte compression type + 4-byte checksum
│  ...                      │
│  Data Block N             │
├───────────────────────────┤
│  Filter Block             │  Bloom/Ribbon (full or partitioned)
├───────────────────────────┤
│  Index Block              │  Key → BlockHandle (binary search / partitioned)
├───────────────────────────┤
│  Compression Dict         │  (optional) Shared ZSTD dictionary
├───────────────────────────┤
│  Range Deletion Block     │  (optional) Range tombstones
├───────────────────────────┤
│  Properties Block         │  Table statistics
├───────────────────────────┤
│  Metaindex Block          │  Maps block names → BlockHandle
├───────────────────────────┤
│  Footer (48 bytes)        │  metaindex_handle, checksum, format_version, magic
└───────────────────────────┘
```

`BlockHandle` = `(offset: varint64, size: varint64)` — locates any block in the file.

---

## Block Format and Encoding

### Delta Encoding

Within a data block, keys are delta-encoded against the previous key:

```
Entry = [shared_bytes: varint32] [unshared_bytes: varint32] [value_len: varint32]
        [key_delta: char[unshared_bytes]] [value: char[value_len]]
```

- `shared_bytes` = prefix length shared with previous key
- Only the non-shared portion is stored

### Restart Points

Every `block_restart_interval` keys (default 16), a **full key** is stored (`shared_bytes = 0`).

**Purpose**: Enable binary search within a block:
1. Binary search across restart point offsets
2. Linear scan within the segment (at most 16 keys)

Index blocks use `index_block_restart_interval = 1` — every entry is a restart point.

### Per-Block Compression

Each data block independently compressed. 5-byte block trailer:
- 1 byte: compression type
- 4 bytes: checksum

---

## Checksums

| Type | Enum | Notes |
|------|------|-------|
| None | `kNoChecksum` | No verification |
| CRC32c | `kCRC32c` | Hardware-accelerated on modern CPUs |
| xxHash | `kxxHash` | Fast general-purpose |
| xxHash64 | `kxxHash64` | 64-bit variant |
| XXH3 | `kXXH3` | Default since RocksDB 6.27, fastest |

**Context checksums** (format v6+): Combine checksum with file offset to detect blocks read from wrong positions.

---

## Index Block

| Type | Description | Best For |
|------|-------------|----------|
| `kBinarySearch` | One entry per data block: separator key + BlockHandle | General (default) |
| `kHashSearch` | Prefix-based hash index | Prefix query workloads |
| `kTwoLevelIndexSearch` | Top-level index → partition blocks | Large SSTs (reduces memory) |
| `kBinarySearchWithFirstKey` | Stores first key per block | Avoids some data block reads |

**Index key shortening**: Separator keys between blocks are shortened to reduce index size. Controlled by `IndexShorteningMode`.

**Partitioned index**: Index split into blocks cached independently. Top-level index fits in memory; partitions loaded on demand.

---

## Filter Block

| Type | Description |
|------|-------------|
| Full filter | Single Bloom/Ribbon for entire SST |
| Partitioned filter | One filter per index partition, cached independently |

**Build**: During `TableBuilder::Add()`, each key passed to `FilterBlockBuilder::Add()`. On `Finish()`, filter serialized as block.

**Read**: Before reading a data block, `FilterBlockReader::KeyMayMatch(key)` is called. False → skip block (no I/O). True → read block (may be false positive).

**`bloom_before_level`**: Bloom for small levels (L0, L1), Ribbon for large levels (L2+). Ribbon is more space-efficient.

### What to Know When Implementing

- If your feature changes key format: filters operate on user keys (without sequence number). Ensure `ExtractUserKey` is correct.
- Filter checks happen before data block reads. If your feature adds a new read path, call the filter first.

---

## Iterator Semantics

### BlockBasedTableIterator

Bridges index iterator and data block iterators:

- **Seek(target)**: Binary search index for target block → read block → seek within block
- **Next()**: If at end of block → advance index to next entry → read next block → position at first key
- **Prev()**: If at start of block → move index to previous entry → read block → position at last key

### Block Boundary Invariant

Iterators transparently cross block boundaries. The user sees a single sorted stream of keys regardless of how many blocks the data spans.

### Pinning

Iterators can pin blocks in the block cache:
- `IsKeyPinned()` / `IsValuePinned()` — whether memory remains valid
- `PinnedIteratorsManager` tracks pinned blocks for batch release
- Memory-mapped (immortal) tables are always pinned

### PrepareValue

Supports lazy value materialization — avoids reading data blocks when only keys are needed.

### What to Know When Implementing

- If your feature adds a new iterator: implement `InternalIterator` interface (`Seek`, `SeekForPrev`, `SeekToFirst`, `SeekToLast`, `Next`, `Prev`, `Valid`, `key`, `value`, `status`).
- Block boundary crossing must be correct. Test with small block sizes to force frequent boundary crossings.
- Test Prev() especially — it's more complex than Next() (must restart from restart point and scan forward).

---

## Caching and Prefetching

### Block Cache

- `BlockBasedTableOptions::block_cache` — shared cache for data/index/filter blocks
- Cache key = session ID + file number + block offset (unique across files and DB reopens)
- Lookup flow: compute key → cache lookup → if miss: read from file → decompress → insert → return

### Prefetching

**Auto-readahead** (sequential detection):
- After `num_file_reads_for_auto_readahead` sequential reads (default: 2), start prefetching
- Readahead starts at `initial_auto_readahead_size` (8 KB), doubles up to `max_auto_readahead_size` (256 KB)

**Cache prepopulation**:
- `kDisable` — no prepopulation (default)
- `kFlushOnly` — warm cache on flush
- `kFlushAndCompaction` — warm cache on flush and compaction

### Cache Pinning

`MetadataCacheOptions` controls what stays pinned:
- `top_level_index_pinning`, `partition_pinning`, `unpartitioned_pinning`
- Tiers: `kNone`, `kFlushedAndSimilar`, `kAll`

### What to Know When Implementing

- If your feature adds cached data: generate unique cache keys using `OffsetableCacheKey`. Never reuse keys across files.
- Prefetching is transparent to readers — `BlockPrefetcher` handles it. If your feature does large sequential reads, it benefits automatically.

---

## Meta Blocks and Table Properties

### Meta Block Types

| Name | Content |
|------|---------|
| `rocksdb.properties` | Table statistics (num entries, sizes, times) |
| `rocksdb.compression_dict` | Shared compression dictionary |
| `rocksdb.range_del` | Range deletion tombstones |

### Table Properties (Key Fields)

| Category | Fields |
|----------|--------|
| Data | `data_size`, `raw_key_size`, `raw_value_size`, `num_entries`, `num_data_blocks` |
| Deletions | `num_deletions`, `num_merge_operands`, `num_range_deletions` |
| Index | `index_size`, `index_partitions`, `top_level_index_size` |
| Filter | `filter_size`, `num_filter_entries` |
| Sequence | `key_largest_seqno`, `key_smallest_seqno` |

### Custom Property Collectors

```cpp
class TablePropertiesCollector {
  virtual Status AddUserKey(const Slice& key, const Slice& value,
                            EntryType type, SequenceNumber seq, uint64_t file_size);
  virtual Status Finish(UserCollectedProperties* properties) = 0;
  virtual bool NeedCompact() const { return false; }
};
```

Register via `ColumnFamilyOptions::table_properties_collector_factories`.

---

## Key Configuration Options

| Option | Default | Purpose |
|--------|---------|---------|
| `block_size` | 4 KB | Target data block size |
| `block_restart_interval` | 16 | Keys between restart points |
| `format_version` | 7 | SST format version (supports v2–v7) |
| `checksum` | kXXH3 | Checksum algorithm |
| `index_type` | kBinarySearch | Index structure |
| `filter_policy` | nullptr | Bloom/Ribbon filter (nullptr = none) |
| `partition_filters` | false | Partitioned filters |
| `block_cache` | 32 MB LRU | Shared block cache |
| `cache_index_and_filter_blocks` | false | Cache index/filter in block cache |
| `initial_auto_readahead_size` | 8 KB | Starting readahead |
| `max_auto_readahead_size` | 256 KB | Max readahead |
| `prepopulate_block_cache` | kDisable | Warm cache on flush/compaction |
| `whole_key_filtering` | true | Filter on whole key vs prefix |
| `use_delta_encoding` | true | Delta-encode keys in data blocks |

---

## Key Source Files

| File | What It Contains |
|------|-----------------|
| `include/rocksdb/table.h` | `TableFactory`, `TableReader`, `TableBuilder` interfaces |
| `include/rocksdb/table_properties.h` | `TableProperties`, `TablePropertiesCollector` |
| `include/rocksdb/filter_policy.h` | `FilterPolicy`, `FilterBitsBuilder/Reader` |
| `table/format.h` | `BlockHandle`, `Footer`, `ChecksumType` |
| `table/meta_blocks.h` | Meta block reading/writing |
| `table/block_based/block_based_table_factory.h` | Factory + `BlockBasedTableOptions` |
| `table/block_based/block_based_table_reader.h` | Reader: `Get`, `NewIterator`, `VerifyChecksum` |
| `table/block_based/block_based_table_builder.h` | Builder: `Add`, `Finish`, `Abandon` |
| `table/block_based/block.h` | `Block`, `DataBlockIter`, `IndexBlockIter` |
| `table/block_based/block_builder.h` | Delta encoding, restart points |
| `table/block_based/block_based_table_iterator.h` | Cross-block iteration |
| `table/block_based/index_builder.h` | Index builders (binary, hash, partitioned) |
| `table/block_based/filter_block.h` | Filter builder/reader interfaces |
| `table/block_based/block_cache.h` | Cache key generation |
| `table/block_based/block_prefetcher.h` | Auto-readahead logic |
