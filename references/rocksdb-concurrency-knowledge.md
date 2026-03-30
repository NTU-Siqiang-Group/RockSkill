---
name: rocksdb-concurrency-knowledge
description: >-
  Use when implementing features that interact with write path, read path,
  locking, background jobs, or transactions — reference for RocksDB concurrency
  mechanisms and guarantees
---

# RocksDB Concurrency & Transaction Reference

## Core Concurrency Model

Two fundamental principles:

1. **Readers never block writers, writers never block readers** — via SuperVersion ref counting and lock-free data structures
2. **Concurrent writes are serialized into groups** — a leader-follower pattern batches writes for efficiency

---

## Write Path: Leader-Follower Batching

**Source**: `db/write_thread.h`, `db/write_thread.cc`

Multiple concurrent writers are grouped into a **WriteGroup**. One leader writes on behalf of all followers.

### Flow

1. Writer joins via lock-free CAS on `newest_writer_` (atomic linked list)
2. First writer becomes leader; others wait via adaptive spin → yield → block
3. Leader groups compatible writers (matching sync, WAL, rate_limiter settings)
4. Leader writes WAL + memtable for entire group
5. Leader wakes followers with results, next pending writer becomes new leader

### Write Modes

| Mode | Option | Behavior |
|------|--------|----------|
| Standard | (default) | Leader does WAL + memtable sequentially |
| Pipelined | `enable_pipelined_write` | WAL and memtable stages overlap across groups |
| Two queues | `two_write_queues` | Separate queue for WAL-only writes (used by WritePrepared txns) |
| Concurrent memtable | `allow_concurrent_memtable_write` | Multiple writers insert into memtable in parallel via SkipList CAS |

### What to Know When Implementing

- New write types must flow through `WriteBatch` → WriteThread → sequence allocation → MemTable insertion. Do not bypass this path.
- If your feature adds a new write option flag, consider whether it affects write group compatibility (writers with different flags cannot be grouped).
- The adaptive waiting (spin → yield → block) is performance-critical. Do not add blocking operations inside the write group path.

---

## Read Path: Lock-Free via SuperVersion

**Source**: `db/column_family.h`

### SuperVersion

```cpp
struct SuperVersion {
  ReadOnlyMemTable* mem;      // Active memtable
  MemTableListVersion* imm;   // Immutable memtables
  Version* current;           // SST file structure
  std::atomic<uint32_t> refs; // Reference count
};
```

- Readers acquire a SuperVersion reference (via thread-local cache for low contention)
- Background jobs (flush/compaction) install a new SuperVersion atomically
- Old SuperVersion freed when all readers release their references

### What to Know When Implementing

- **Reads never hold the DB mutex.** If your feature adds a new read path, use `GetThreadLocalSuperVersion()` / `ReturnThreadLocalSuperVersion()`.
- SuperVersion is acquired BEFORE the snapshot sequence number is read — this ordering prevents flush races.
- If your feature creates long-lived reads, hold a snapshot to prevent data from being garbage collected.

---

## MemTable Concurrency

**Source**: `memtable/inlineskiplist.h`

- `InsertConcurrently()` uses CAS-based node linking — safe to call concurrently with reads and other inserts
- Allocated nodes are never deleted until skiplist destruction
- Enabled via `allow_concurrent_memtable_write = true`

---

## DB Mutex Scope

**Source**: `db/db_impl/db_impl.h`

RocksDB has **one global DB mutex** (`CacheAlignedInstrumentedMutex mutex_`).

### Holds the DB Mutex

| Operation | Why |
|-----------|-----|
| SuperVersion installation | Atomically swap Version pointers |
| File metadata updates | Add/remove SST files from Version |
| Memtable management | Create new memtable after flush |
| Background job scheduling | Coordinate flush/compaction |
| Write thread leader transitions | Brief, during group exit |

### Does NOT Hold the DB Mutex

| Operation | Mechanism Instead |
|-----------|------------------|
| Reads (Get, Iterator) | SuperVersion ref counting |
| Concurrent memtable inserts | SkipList CAS |
| Compaction work (after file selection) | Version ref counting |
| WAL writes | Write group leader handles |
| Block cache operations | Block cache has own locks |

### What to Know When Implementing

- Minimize time holding the DB mutex — it is the main bottleneck for metadata operations.
- Flush and compaction release the mutex during I/O-intensive work, only re-acquiring for `LogAndApply()`.
- All column families share the same global mutex. Per-CF state uses atomics (`refs_`, `dropped_`, `super_version_number_`).

---

## Flush and Compaction Concurrency

**Source**: `db/version_set.h`

- `Version` represents a snapshot of the SST file layout
- `LogAndApply(edit, mutex_)` atomically installs a new Version and SuperVersion
- Old files are deleted only after all readers release their references

| Job | Mutex Held | Mutex Released |
|-----|------------|---------------|
| Flush | Pick memtable, install new Version | Write SST file (I/O) |
| Compaction | PickCompaction, install output Version | Read/write SSTs (CPU + I/O) |

---

## Write Stall and Rate Limiting

**Source**: `db/write_controller.h`

| Condition | Action |
|-----------|--------|
| L0 files > `level0_slowdown_writes_trigger` | Delay writes |
| L0 files > `level0_stop_writes_trigger` | Stop writes |
| Pending compaction bytes too high | Delay writes |
| Too many memtables | Stop/delay writes |

`WriteController` issues tokens: `GetStopToken()`, `GetDelayToken(rate)`, `GetCompactionPressureToken()`. Tokens are released when the stall condition resolves.

---

## Transaction Architecture

### Two Strategies

| Type | Locking | Conflict Detection | Best For |
|------|---------|-------------------|----------|
| `TransactionDB` (pessimistic) | Row-level locks at write time | Deadlock detection + timeout | Write-heavy, short txns |
| `OptimisticTransactionDB` | None during execution | Validation at commit | Read-heavy, rare conflicts |

### Three Pessimistic Write Policies

| Policy | When Data Hits DB | Memory | Key Trade-off |
|--------|-------------------|--------|---------------|
| WriteCommitted | At commit | High | Simple, baseline |
| WritePrepared | At prepare | Low | Uses CommitCache for visibility |
| WriteUnprepared | Incrementally | Lowest | Best for large txns |

### Inheritance

```
Transaction → TransactionBaseImpl
  ├── OptimisticTransaction
  └── PessimisticTransaction
      ├── WriteCommittedTxn
      └── WritePreparedTxn → WriteUnpreparedTxn
```

---

## Pessimistic Transactions

### Row-Level Locking

**Source**: `utilities/transactions/lock/point/point_lock_manager.h`

- Locks organized as **striped hash table** per column family (default 16 stripes)
- Each stripe has its own mutex — reduces contention
- `LockInfo` per key tracks: exclusive flag, holder txn IDs, waiter queue

### Deadlock Detection

- Traverses wait-for graph during lock acquisition
- Depth limited by `deadlock_detect_depth` (default 50)
- Returns `Status::Busy()` when cycle found

### Key APIs

| API | Purpose |
|-----|---------|
| `txn->GetForUpdate(opts, cf, key, &val)` | Read + acquire lock |
| `txn->Put(cf, key, val)` | Buffered write + acquire lock |
| `txn->SetSnapshot()` | Capture read snapshot |
| `txn->Commit()` | Validate + write to DB + release locks |
| `txn->Rollback()` | Discard writes + release locks |
| `txn->SetSavePoint()` / `RollbackToSavePoint()` | Nested rollback |

### Error Statuses

| Status | Meaning |
|--------|---------|
| `Status::Busy()` | Write conflict or deadlock |
| `Status::TimedOut()` | Lock acquisition timeout |
| `Status::Expired()` | Transaction lifetime exceeded |
| `Status::TryAgain()` | Memtable history too old for validation |

---

## WritePrepared Transactions

**Source**: `utilities/transactions/write_prepared_txn_db.h`

- Data written to DB at **prepare** (durable but invisible)
- Commit writes only a WAL marker and updates **CommitCache** (`prepare_seq → commit_seq`)
- Visibility via `IsInSnapshot(prep_seq, snapshot_seq)`: checks CommitCache + PreparedHeap
- Uses `SnapshotChecker` (`db/snapshot_checker.h`) for compaction visibility decisions
- Two-write-queues: main queue for prepare (WAL + memtable), second queue for commit (WAL only)

## WriteUnprepared Transactions

**Source**: `utilities/transactions/write_unprepared_txn.h`

- Flushes writes to DB **incrementally** before prepare (threshold-based)
- Tracks unprepared sequence numbers in `unprep_seqs_` map
- Lowest memory usage for large transactions
- Transaction can read its own unprepared writes via `WriteUnpreparedTxnReadCallback`

---

## Optimistic Transactions

**Source**: `utilities/transactions/optimistic_transaction.h`

- No locks during execution — all writes buffered in `WriteBatchWithIndex`
- Conflict detected at commit: checks if any tracked key's seq changed since snapshot
- Two validation modes: `kValidateSerial` (in write group) or `kValidateParallel` (before write group, using `OccLockBuckets`)
- Returns `Status::Busy()` on conflict — application must retry

### When to Use

- Read-heavy workloads with rare write conflicts
- Short transactions where retry is cheap
- Scenarios where deadlock avoidance matters

---

## Two-Phase Commit (2PC)

### WAL Markers

| Marker | Value | Purpose |
|--------|-------|---------|
| `kTypeBeginPrepareXID` | 0x9 | Start of prepare |
| `kTypeEndPrepareXID` | 0xA | End of prepare |
| `kTypeCommitXID` | 0xB | Transaction committed |
| `kTypeRollbackXID` | 0xC | Transaction rolled back |

### Protocol

```cpp
txn->SetName("txn1");    // Required for 2PC
txn->Prepare();           // Phase 1: write data + prepare marker
txn->Commit();            // Phase 2: write commit marker
// OR txn->Rollback();
```

### Crash Recovery

1. On restart, scan WAL for `kTypeEndPrepareXID` without matching `kTypeCommitXID`
2. Recovered transactions re-acquire locks in PREPARED state
3. Application decides: `Commit()` or `Rollback()`

### What to Know When Implementing

- If your feature interacts with WAL replay or recovery, account for 2PC markers.
- WritePrepared/WriteUnprepared have different visibility rules — data is in the DB but may not be committed. Use `SnapshotChecker` to determine visibility.

---

## Guarantees

### Base RocksDB (No Transactions)

| Guarantee | Mechanism |
|-----------|-----------|
| Readers never block writers | SuperVersion ref counting |
| Writers never block readers | Atomic SuperVersion installation |
| Atomic batch writes | WriteBatch with consecutive sequence numbers |
| Crash recovery | WAL replay restores state |
| Snapshot isolation | Immutable point-in-time view |

### Pessimistic Transactions

| Guarantee | Mechanism |
|-----------|-----------|
| Serializable isolation | Row-level locks + snapshot validation |
| No dirty reads | Writes buffered until commit |
| Repeatable reads | `GetForUpdate` prevents concurrent modification |
| Deadlock handling | Wait-for graph traversal + configurable timeout |
| Crash atomicity | 2PC WAL markers |

### Optimistic Transactions

| Guarantee | Mechanism |
|-----------|-----------|
| Snapshot isolation | Commit-time validation |
| No deadlocks | No locks acquired |
| Atomic commit-or-abort | Validation fails → entire txn rejected |

### What RocksDB Does NOT Guarantee

- **No cross-CF atomicity** without explicit transaction use
- **No automatic retry** on conflict — application handles `Status::Busy()`
- **No distributed transactions** — 2PC is local only
- **WriteUnprepared may leave partial writes** on crash before prepare
- **Optimistic transactions may starve** under high contention

---

## Key Source Files

### Concurrency

| File | What It Contains |
|------|-----------------|
| `db/write_thread.h` | WriteThread, Writer, WriteGroup, state machine |
| `db/write_controller.h` | Write stall rate limiting |
| `db/column_family.h` | SuperVersion, thread-local caching |
| `db/db_impl/db_impl.h` | DB mutex, background job coordination |
| `db/version_set.h` | Version, VersionSet, LogAndApply |
| `memtable/inlineskiplist.h` | Lock-free concurrent SkipList |
| `port/port_posix.h` | Mutex, RWMutex, CondVar |
| `monitoring/instrumented_mutex.h` | Stats-wrapped mutexes, RAII locks |

### Transactions

| File | What It Contains |
|------|-----------------|
| `include/rocksdb/utilities/transaction.h` | Public Transaction API |
| `include/rocksdb/utilities/transaction_db.h` | TransactionDB, options |
| `include/rocksdb/utilities/optimistic_transaction_db.h` | OptimisticTransactionDB |
| `utilities/transactions/transaction_base.h` | TransactionBaseImpl, SavePoint |
| `utilities/transactions/pessimistic_transaction.h` | PessimisticTransaction |
| `utilities/transactions/write_prepared_txn_db.h` | CommitCache, PreparedHeap |
| `utilities/transactions/write_unprepared_txn.h` | Incremental writes, unprep_seqs_ |
| `utilities/transactions/optimistic_transaction.h` | OCC validation |
| `utilities/transactions/lock/point/point_lock_manager.h` | Striped lock manager |
| `db/snapshot_checker.h` | SnapshotChecker for WritePrepared visibility |
| `db/dbformat.h` | 2PC WAL marker types |
