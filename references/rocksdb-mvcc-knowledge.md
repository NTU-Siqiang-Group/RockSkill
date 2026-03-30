---
name: rocksdb-mvcc-knowledge
description: >-
  Use when implementing features that interact with versioning, snapshots,
  sequence numbers, or user-defined timestamps — reference for RocksDB MVCC
  mechanisms and guarantees
---

# RocksDB MVCC & Timestamp Reference

## Two Versioning Mechanisms

| Mechanism | Assigned By | Purpose | GC Control |
|-----------|------------|---------|------------|
| Sequence Numbers | RocksDB (automatic) | Internal write ordering, snapshot isolation | Releasing snapshots |
| User-Defined Timestamps (UDT) | Application (explicit) | Application-level time-travel reads | `full_history_ts_low` |

Both are independent. When both are active, a version is visible only if **both** conditions pass: `seq <= snapshot_seq AND ts <= read_timestamp`.

---

## InternalKey Format

```
Without UDT:  [ user_key | seqno(56b) + type(8b) ]
With UDT:     [ user_key | timestamp  | seqno(56b) + type(8b) ]
```

- Sequence number: 56-bit, monotonically increasing, packed with 8-bit `ValueType` into 8 bytes
- Timestamp: variable size (determined by `Comparator::timestamp_size()`), placed between user key and internal footer
- Sort order: user key ascending, then sequence number **descending** (newest first)
- Source: `db/dbformat.h`, `include/rocksdb/types.h`

## Key Value Types

| Type | When to Use |
|------|------------|
| `kTypeValue` (0x1) | Regular put |
| `kTypeDeletion` (0x0) | Delete (tombstone) |
| `kTypeMerge` (0x2) | Merge operand |
| `kTypeSingleDeletion` (0x7) | Delete with exactly one preceding Put |
| `kTypeRangeDeletion` (0xF) | Range tombstone (meta block) |
| `kTypeDeletionWithTimestamp` (0x14) | Delete with UDT |
| `kTypeWideColumnEntity` (0x16) | Wide-column data |

When adding a new feature, use the existing type that matches. Do not invent new types unless extending the format itself.

---

## Snapshots

### Guarantees

- **Point-in-time consistency**: all reads on a snapshot see the same DB state
- **Immutable**: concurrent writes never affect a snapshot's view
- **Lightweight**: creation is O(1) — just captures a sequence number
- **No dirty / non-repeatable / phantom reads** within a snapshot

### Key APIs

| API | Purpose |
|-----|---------|
| `db->GetSnapshot()` | Capture current sequence number as a snapshot |
| `db->ReleaseSnapshot(snap)` | Release snapshot, allow GC of versions only it pins |
| `ManagedSnapshot(db)` | RAII wrapper — auto-releases on destruction |
| `ReadOptions::snapshot` | Read as of this snapshot; if null, uses implicit snapshot at read time |

### Internal Implementation

- `SnapshotImpl` in `db/snapshot_impl.h`: doubly-linked circular list ordered by sequence number
- `SnapshotList::oldest()` returns the earliest active snapshot — this is the GC watermark
- Releasing the oldest snapshot triggers `UpdateOldestSnapshot()`, enabling compaction to reclaim old versions

### What to Know When Implementing

- If your feature creates long-lived reads (e.g., backup, export), **use a snapshot** to get a consistent view.
- Hold snapshots for the **shortest time possible** — each held snapshot prevents compaction from reclaiming space.
- When writing tests involving snapshots: create snapshot, write more data, verify reads through snapshot see only pre-snapshot data.

---

## Sequence Numbers in Read/Write Paths

### Write Path

1. `WriteBatch` collects operations (Put, Delete, Merge, etc.)
2. `FetchAddLastAllocatedSequence(N)` atomically assigns N consecutive sequence numbers
3. All entries in one `WriteBatch` get consecutive numbers — this enables atomic batch writes
4. Entries inserted into MemTable with their sequence numbers; multiple versions of same key coexist
5. WAL records the batch with its starting sequence number for crash recovery
6. Source: `db/db_impl/db_impl_write.cc`, `db/write_batch_internal.h`

### Read Path

1. Determine snapshot sequence: explicit (`ReadOptions::snapshot`) or implicit (`GetLastPublishedSequence()`)
2. **Critical**: SuperVersion acquired BEFORE sequence number — prevents flush race
3. `LookupKey` encodes user_key + snapshot_seq for MemTable/SST search (`db/lookup_key.h`)
4. `DBIter` filters by snapshot: skips entries with seq > snapshot, resolves deletions, collapses merges
5. Returns one entry per user key (the latest visible version)
6. Source: `db/db_impl/db_impl.cc`, `db/db_iter.h`

### What to Know When Implementing

- Every write **must** get a sequence number. If you add a new write type, ensure it flows through the standard `WriteBatch` → sequence allocation → MemTable insertion path.
- Reads without an explicit snapshot use an implicit one. This means each `Get()` call is isolated but two calls may see different states.
- If your feature needs multi-read consistency, take a snapshot.

---

## Compaction Version Cleanup

Compaction decides which versions to keep or discard via `CompactionIterator` (`db/compaction/compaction_iterator.cc`).

### Drop Rules

| Rule | Condition | Action |
|------|-----------|--------|
| Hidden entry | Two versions of same key visible in same snapshot stripe | Drop older |
| Obsolete deletion | Deletion visible at `earliest_snapshot`, key doesn't exist in deeper levels | Drop deletion marker |
| SingleDelete | Cancels exactly one preceding Put in same snapshot stripe | Drop both |
| Bottommost cleanup | Deletion marker at bottommost level, no version below | Drop marker |

### Key Invariants

- **Never drop a version that any active snapshot might need** — `findEarliestVisibleSnapshot()` checks this via binary search
- **Oldest snapshot = GC watermark** — versions with seq < oldest_snapshot AND not the latest visible version can be dropped
- If your feature adds new entry types, ensure `CompactionIterator` handles them correctly (check `NextFromInput()` logic)

---

## User-Defined Timestamps (UDT)

### Setup

```cpp
// Enable UDT: set a timestamp-aware comparator
cf_options.comparator = BytewiseComparatorWithU64Ts();  // 8-byte uint64_t

// Optional: don't persist timestamps to SST (memtable only)
cf_options.persist_user_defined_timestamps = false;  // default: true
```

Constraints:
- Timestamp size is fixed for the lifetime of a column family
- `persist_user_defined_timestamps` is immutable after CF creation
- When not persisting: requires `full_history_ts_low` set, `max_write_buffer_number > 2`

### Comparator Interface

| Method | Purpose |
|--------|---------|
| `timestamp_size()` | Byte size of timestamp (0 = UDT disabled) |
| `Compare(a, b)` | Full comparison including timestamps |
| `CompareWithoutTimestamp(a, b)` | Compare user keys only |
| `CompareTimestamp(ts1, ts2)` | Compare timestamp portions only |

Built-in: `BytewiseComparatorWithU64Ts()`, `ReverseBytewiseComparatorWithU64Ts()`

### Write with Timestamps

```cpp
// Option A: separate timestamp parameter
db->Put(write_opts, cf, key, ts, value);
db->Delete(write_opts, cf, key, ts);

// Option B: timestamp appended to key
std::string key_with_ts = user_key + ts_bytes;
db->Put(write_opts, cf, key_with_ts, value);

// Option C: batch with deferred timestamp
WriteBatch batch;
batch.Put(cf, key, value);
batch.UpdateTimestamps(ts, ts_sz_func);  // apply timestamp later
```

### Read with Timestamps

| ReadOptions Field | Purpose |
|-------------------|---------|
| `timestamp` | Upper bound — returns latest version with `ts <= timestamp` |
| `iter_start_ts` | Lower bound for iterators — returns all versions where `iter_start_ts <= ts <= timestamp` |

- Point lookup (`Get`): returns single latest version within timestamp bound
- Iterator without `iter_start_ts`: one version per key (latest within bound)
- Iterator with `iter_start_ts`: **multiple versions** per key (all within timestamp range)
- `Iterator::timestamp()` returns the timestamp of the current entry

### Timestamp GC via `full_history_ts_low`

| API | Purpose |
|-----|---------|
| `db->IncreaseFullHistoryTsLow(cf, ts)` | Advance GC cutoff (can only increase) |
| `db->GetFullHistoryTsLow(cf, &ts)` | Query current cutoff |

- Data with `ts < full_history_ts_low` **may be garbage collected** during compaction
- Data with `ts >= full_history_ts_low` is **never** garbage collected
- Reads at timestamps below cutoff may return incomplete results

Compaction GC logic (`compaction_iterator.cc`):
- `is_timestamp_eligible_for_gc = (ts_size == 0) || (full_history_ts_low && ts < full_history_ts_low)`
- Latest version of a key is always kept even if below cutoff
- Older versions below cutoff are dropped

### What to Know When Implementing

- If your feature touches key encoding/decoding: account for optional timestamp bytes between user key and internal footer. Use `ExtractTimestampFromKey()`, `StripTimestampFromUserKey()` from `db/dbformat.h`.
- If your feature adds a new read path: check `Comparator::timestamp_size()` — if non-zero, filtering by timestamp is required.
- If your feature modifies compaction logic: respect `full_history_ts_low` and timestamp-based GC rules.
- `kTypeDeletionWithTimestamp` (0x14) is used instead of `kTypeDeletion` when UDT is enabled.

---

## Key Source Files

| File | What It Contains |
|------|-----------------|
| `db/dbformat.h` | InternalKey format, timestamp extraction functions, ValueType enum |
| `include/rocksdb/types.h` | `SequenceNumber` type |
| `include/rocksdb/snapshot.h` | Public `Snapshot`, `ManagedSnapshot` API |
| `db/snapshot_impl.h` | `SnapshotImpl`, `SnapshotList` internals |
| `db/lookup_key.h` | `LookupKey` (read path key construction) |
| `db/db_iter.h` | `DBIter` (version filtering, timestamp bounds) |
| `db/db_impl/db_impl.cc` | `GetSnapshot()`, `ReleaseSnapshot()`, `Get()` |
| `db/db_impl/db_impl_write.cc` | Write path, sequence allocation |
| `db/compaction/compaction_iterator.cc` | Version drop rules, snapshot visibility, timestamp GC |
| `include/rocksdb/comparator.h` | Timestamp-aware comparator interface |
| `include/rocksdb/options.h` | `ReadOptions::timestamp`, `iter_start_ts` |
| `include/rocksdb/advanced_options.h` | `persist_user_defined_timestamps` |
| `include/rocksdb/db.h` | `IncreaseFullHistoryTsLow()`, `GetFullHistoryTsLow()` |
| `util/udt_util.h` | UDT utility functions, timestamp recovery |
