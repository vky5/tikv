# TiKV Storage Internals: WAL, Snapshots, LSN, and MVCC

> This document covers TiKV's storage engine internals — specifically the
> Write-Ahead Log (WAL), engine-level snapshots, Log Sequence Numbers (LSN /
> sequence numbers), the MVCC layer, and the end-to-end write path. Raft
> consensus is deliberately excluded; the focus is purely on storage.

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Write-Ahead Log (WAL)](#2-write-ahead-log-wal)
   - 2.1 [What is the WAL?](#21-what-is-the-wal)
   - 2.2 [WAL Configuration](#22-wal-configuration)
   - 2.3 [Per-Write WAL Control (`WriteOptions`)](#23-per-write-wal-control-writeoptions)
   - 2.4 [WAL Sync and Flush](#24-wal-sync-and-flush)
   - 2.5 [When WAL is Disabled](#25-when-wal-is-disabled)
   - 2.6 [WAL Recovery Modes](#26-wal-recovery-modes)
3. [Log Sequence Number (LSN / Sequence Number)](#3-log-sequence-number-lsn--sequence-number)
   - 3.1 [Definition and Purpose](#31-definition-and-purpose)
   - 3.2 [Retrieving the Latest Sequence Number](#32-retrieving-the-latest-sequence-number)
   - 3.3 [Sequence Number per Snapshot](#33-sequence-number-per-snapshot)
   - 3.4 [Oldest Snapshot Sequence Number](#34-oldest-snapshot-sequence-number)
4. [Engine-Level Snapshots](#4-engine-level-snapshots)
   - 4.1 [Snapshot Trait (Interface)](#41-snapshot-trait-interface)
   - 4.2 [RocksSnapshot: Implementation Details](#42-rockssnapshot-implementation-details)
   - 4.3 [Snapshot Creation](#43-snapshot-creation)
   - 4.4 [Reads From a Snapshot](#44-reads-from-a-snapshot)
   - 4.5 [Snapshot Lifecycle and Cleanup](#45-snapshot-lifecycle-and-cleanup)
5. [Column Families](#5-column-families)
6. [MVCC (Multi-Version Concurrency Control)](#6-mvcc-multi-version-concurrency-control)
   - 6.1 [MvccTxn: Buffering Writes](#61-mvvctxn-buffering-writes)
   - 6.2 [How MVCC Keys Are Encoded](#62-how-mvcc-keys-are-encoded)
   - 6.3 [MVCC Reads via Snapshots](#63-mvcc-reads-via-snapshots)
   - 6.4 [MVCC Write Record Types](#64-mvcc-write-record-types)
7. [WriteBatch: Atomic Engine Writes](#7-writebatch-atomic-engine-writes)
   - 7.1 [WriteBatch Trait](#71-writebatch-trait)
   - 7.2 [RocksWriteBatchVec: Multi-Batch Write](#72-rockswritebatchvec-multi-batch-write)
8. [End-to-End Write Path](#8-end-to-end-write-path)
   - 8.1 [Phase 1 – Client Command Arrives at the Scheduler](#81-phase-1--client-command-arrives-at-the-scheduler)
   - 8.2 [Phase 2 – Acquire Snapshot and Latches](#82-phase-2--acquire-snapshot-and-latches)
   - 8.3 [Phase 3 – MVCC Processing](#83-phase-3--mvcc-processing)
   - 8.4 [Phase 4 – Convert to WriteBatch and Write to RocksDB](#84-phase-4--convert-to-writebatch-and-write-to-rocksdb)
   - 8.5 [Data Flow Diagram](#85-data-flow-diagram)
9. [Memtable Flush and Checkpointing](#9-memtable-flush-and-checkpointing)
   - 9.1 [Flush Operations](#91-flush-operations)
   - 9.2 [Checkpointing (Point-in-Time Backup)](#92-checkpointing-point-in-time-backup)
10. [Key Source Files Reference](#10-key-source-files-reference)

---

## 1. Architecture Overview

TiKV stores data using **RocksDB** as its embedded storage engine. The storage
stack (bottom to top) looks like this:

```
┌─────────────────────────────────────────────────────┐
│                  gRPC / TiDB Client                 │
├─────────────────────────────────────────────────────┤
│           TxnScheduler  (src/storage/txn/)          │
│  - Latches, command dispatch, snapshot acquisition  │
├─────────────────────────────────────────────────────┤
│         MVCC Layer  (src/storage/mvcc/)             │
│  - MvccTxn, SnapshotReader, conflict detection      │
├─────────────────────────────────────────────────────┤
│     Engine Traits  (components/engine_traits/)      │
│  - KvEngine, Snapshot, WriteBatch, MiscExt …        │
├─────────────────────────────────────────────────────┤
│   RocksDB Engine  (components/engine_rocks/)        │
│  - RocksEngine, RocksSnapshot, RocksWriteBatchVec   │
├─────────────────────────────────────────────────────┤
│           RocksDB  (WAL + Memtable + SSTs)          │
│  - WAL files (.log), MANIFEST, SST level files      │
└─────────────────────────────────────────────────────┘
```

The **engine_traits** crate defines abstract interfaces (Rust traits) so that
the MVCC and transaction layers never depend directly on RocksDB. The
**engine_rocks** crate provides concrete RocksDB implementations of those
traits.

---

## 2. Write-Ahead Log (WAL)

### 2.1 What is the WAL?

The WAL is a sequential, append-only log file (`.log` suffix in the RocksDB
data directory). Every write that reaches RocksDB is first appended to the WAL
**before** it is applied to the in-memory memtable. On restart after a crash,
RocksDB replays the WAL to recover writes that were in memory but not yet
flushed to on-disk SST files.

This gives TiKV **durability**: even if the process dies after a write is
acknowledged, the WAL ensures the write survives.

### 2.2 WAL Configuration

WAL behaviour is configured in `DbConfig` inside `src/config/mod.rs`. The
relevant fields are applied to RocksDB via `DbConfig::build_opt()`:

| Config field | Type | Default | What it controls |
|---|---|---|---|
| `wal_recovery_mode` | `DBRecoveryMode` | `PointInTime` | How RocksDB replays the WAL on startup (see §2.6) |
| `wal_dir` | `String` | `""` (same as DB dir) | Directory for WAL files (separate from SST files) |
| `wal_ttl_seconds` | `u64` | `0` (disabled) | How long to keep old WAL files after they are no longer needed |
| `wal_size_limit` | `ReadableSize` | `0` (no limit) | Maximum size of a single WAL file |
| `max_total_wal_size` | `Option<ReadableSize>` | `4 GiB` (for `RaftKv`) | Total size of all WAL files before flushing is forced |
| `wal_bytes_per_sync` | `ReadableSize` | `512 KiB` | How often OS-level `fdatasync` is called while writing the WAL |
| `track_and_verify_wals_in_manifest` | `bool` | `true` | Verifies WAL integrity via the MANIFEST file |

**Code reference** (`src/config/mod.rs`, `DbConfig::build_opt`):

```rust
opts.set_wal_recovery_mode(self.wal_recovery_mode);
if !self.wal_dir.is_empty() {
    opts.set_wal_dir(&self.wal_dir);
}
opts.set_wal_ttl_seconds(self.wal_ttl_seconds);
opts.set_wal_size_limit_mb(self.wal_size_limit.as_mb());
opts.set_max_total_wal_size(self.max_total_wal_size.unwrap_or(ReadableSize(0)).0);
opts.set_wal_bytes_per_sync(self.wal_bytes_per_sync.0);
opts.set_track_and_verify_wals_in_manifest(self.track_and_verify_wals_in_manifest);
```

### 2.3 Per-Write WAL Control (`WriteOptions`)

Every write operation in TiKV carries a `WriteOptions` value that can
individually override WAL behaviour for that write:

**Source:** `components/engine_traits/src/options.rs`

```rust
#[derive(Clone, Default)]
pub struct WriteOptions {
    sync: bool,        // force fsync after write
    no_slowdown: bool, // fail immediately if write is slowed down
    disable_wal: bool, // skip the WAL entirely for this write
}
```

These map directly to RocksDB's `WriteOptions` in
`components/engine_rocks/src/options.rs`:

```rust
impl From<engine_traits::WriteOptions> for RocksWriteOptions {
    fn from(opts: engine_traits::WriteOptions) -> Self {
        let mut r = RawWriteOptions::default();
        r.set_sync(opts.sync());
        r.set_no_slowdown(opts.no_slowdown());
        r.disable_wal(opts.disable_wal());
        RocksWriteOptions(r)
    }
}
```

**Meaning of each field:**

- **`sync = true`** — after the WAL append, call `fsync` so the data is
  guaranteed on physical disk before the write is considered complete. This is
  the safest mode but the slowest.
- **`sync = false`** (default) — rely on the OS page cache; the WAL is
  written to the kernel buffer but not immediately synced to disk. The WAL is
  flushed periodically via `wal_bytes_per_sync`.
- **`disable_wal = true`** — skip the WAL entirely. Data goes only into the
  memtable. If the process crashes before a memtable flush, **the data is
  lost**. Used in specific controlled scenarios (see §2.5).
- **`no_slowdown = true`** — if RocksDB is under write pressure (e.g., too
  many level-0 SSTs), fail immediately rather than waiting.

### 2.4 WAL Sync and Flush

TiKV exposes an explicit WAL sync operation through the `MiscExt` trait:

**Source:** `components/engine_traits/src/misc.rs`

```rust
pub trait MiscExt: CfNamesExt + FlowControlFactorsExt + WriteBatchExt {
    fn sync_wal(&self) -> Result<()>;
    fn flush_cfs(&self, cfs: &[&str], wait: bool) -> Result<()>;
    fn flush_cf(&self, cf: &str, wait: bool) -> Result<()>;
    fn flush_oldest_cf(&self, wait: bool, threshold: Option<SystemTime>) -> Result<bool>;
    // ...
}
```

The RocksDB implementation delegates straight to RocksDB's own WAL sync:

**Source:** `components/engine_rocks/src/misc.rs`

```rust
fn sync_wal(&self) -> Result<()> {
    self.as_inner().sync_wal().map_err(r2e)
}
```

And in `components/engine_rocks/src/engine.rs` the engine-level `sync()` calls this:

```rust
impl KvEngine for RocksEngine {
    fn sync(&self) -> Result<()> {
        self.db.sync_wal().map_err(r2e)
    }
}
```

After a batch write that modifies keys by key-scanning (e.g., `delete_all_in_range_cf_by_key`), the engine explicitly syncs the WAL when WAL is enabled:

```rust
if wb.count() > 0 {
    wb.write_opt(wopts)?;
    if !wopts.disable_wal() {
        self.sync_wal()?;  // explicit sync after key-level deletions
    }
}
```

**Flush vs Sync:**

- **Flush** (`flush_cf`, `flush_cfs`) moves data from the memtable to an SST
  file. Once flushed, the WAL records for that data can be discarded.
- **Sync** (`sync_wal`) calls `fsync` on the WAL file to ensure the kernel
  buffer is written to disk. It does **not** flush the memtable.

### 2.5 When WAL is Disabled

WAL is selectively disabled in performance-sensitive or controlled-safety paths:

1. **Apply FSM (Raft apply)** — `components/raftstore/src/store/fsm/apply.rs`:
   When `disable_wal` is set on the `ApplyContext`, writes skip the WAL. This
   is safe because Raft's own log provides durability; if a crash occurs the
   Raft log is replayed. The relevant write path in `write_to_db()`:

   ```rust
   let mut write_opts = engine_traits::WriteOptions::new();
   write_opts.set_sync(need_sync);
   write_opts.set_disable_wal(self.disable_wal);
   self.kv_wb_mut().write_opt(&write_opts).unwrap_or_else(|e| {
       panic!("failed to write to engine: {:?}", e);
   });
   ```

2. **Raftstore-v2 tablet worker** — `components/raftstore-v2/src/worker/tablet.rs`:
   WAL is disabled for certain tablet operations where Raft durability is
   sufficient:

   ```rust
   wopts.set_disable_wal(true);
   ```

### 2.6 WAL Recovery Modes

On startup, RocksDB uses the `wal_recovery_mode` to decide how aggressively to
replay and validate the WAL. TiKV defaults to **`PointInTime`**:

| Mode | Meaning |
|---|---|
| `TolerateCorruptedTailRecords` | Ignore errors at the end of the WAL (tail records may be incomplete after a crash) |
| `AbsoluteConsistency` | Fail on any WAL inconsistency; safest but may refuse to start after certain crashes |
| `PointInTime` | **(TiKV default)** Recover all consistent records; stop at the first inconsistency; safe and recovers as much data as possible |
| `SkipAnyCorruptedRecords` | Skip all corrupted records; maximises data recovery but may lose records between corruptions |

---

## 3. Log Sequence Number (LSN / Sequence Number)

### 3.1 Definition and Purpose

RocksDB (and by extension TiKV) uses the term **sequence number** rather than
LSN. It is a monotonically increasing `u64` that increments with every
committed write batch. It serves as the logical clock of the storage engine:

- It uniquely orders all write batches.
- It is stamped on every internal key in the SST files (`user_key + seqno + type`).
- It is captured in a snapshot to define which writes are visible to that snapshot.
- It is used by the compaction process to remove old versions of keys (those
  with a sequence number below the oldest active snapshot).

### 3.2 Retrieving the Latest Sequence Number

**Source:** `components/engine_traits/src/misc.rs`

```rust
fn get_latest_sequence_number(&self) -> u64;
```

**Source:** `components/engine_rocks/src/misc.rs`

```rust
fn get_latest_sequence_number(&self) -> u64 {
    self.as_inner().get_latest_sequence_number()
}
```

This gives the sequence number of the most recently committed write batch.
Every new `WriteBatch::write()` call atomically increments this counter and
returns the new sequence number as its return value (`Result<u64>`).

### 3.3 Sequence Number per Snapshot

Every `RocksSnapshot` captures the current sequence number at the instant it
is created:

**Source:** `components/engine_traits/src/snapshot_misc.rs`

```rust
pub trait SnapshotMiscExt {
    fn sequence_number(&self) -> u64;
}
```

**Source:** `components/engine_rocks/src/snapshot.rs`

```rust
impl SnapshotMiscExt for RocksSnapshot {
    fn sequence_number(&self) -> u64 {
        unsafe { self.snap.get_sequence_number() }
    }
}
```

All reads from a snapshot see exactly the state of the database at that
sequence number. Writes with a higher sequence number are invisible to the
snapshot.

### 3.4 Oldest Snapshot Sequence Number

When multiple snapshots exist simultaneously (e.g., long-running reads
concurrent with writes), RocksDB must keep all SST data that any live snapshot
could need. The engine exposes the minimum sequence number across all live
snapshots:

**Source:** `components/engine_rocks/src/misc.rs`

```rust
fn get_oldest_snapshot_sequence_number(&self) -> Option<u64> {
    match self.as_inner()
        .get_property_int(ROCKSDB_OLDEST_SNAPSHOT_SEQUENCE)
    {
        Some(0) => None,  // no active snapshots
        s => s,
    }
}
```

Compaction uses this value to decide which old key versions can be safely
removed: a version is garbage-collectable only if its sequence number is below
the oldest snapshot sequence number and a newer version exists.

---

## 4. Engine-Level Snapshots

### 4.1 Snapshot Trait (Interface)

**Source:** `components/engine_traits/src/snapshot.rs`

```rust
pub trait Snapshot
where
    Self: 'static
        + Peekable      // point reads (get_value, get_value_cf)
        + Iterable      // range scans (iterator_opt)
        + CfNamesExt    // list column families
        + SnapshotMiscExt  // sequence_number()
        + Send + Sync + Sized + Debug,
{
    fn in_memory_engine_hit(&self) -> bool { false }
}
```

The snapshot is the primary read abstraction in TiKV's storage layer. All MVCC
reads and transaction conflict checks operate on a snapshot, never directly on
the live mutable state of the engine.

### 4.2 RocksSnapshot: Implementation Details

**Source:** `components/engine_rocks/src/snapshot.rs`

```rust
pub struct RocksSnapshot {
    db: Arc<DB>,       // shared reference to the RocksDB instance
    snap: UnsafeSnap,  // RocksDB-internal snapshot handle
}

unsafe impl Send for RocksSnapshot {}
unsafe impl Sync for RocksSnapshot {}
```

`UnsafeSnap` is a raw pointer wrapper around RocksDB's C++ snapshot object.
The `Send + Sync` bounds are safe here because TiKV guarantees that the
underlying `DB` outlives any snapshot taken from it (enforced by the `Arc<DB>`
keeping the DB alive).

### 4.3 Snapshot Creation

```rust
impl RocksSnapshot {
    pub fn new(db: Arc<DB>) -> Self {
        unsafe {
            RocksSnapshot {
                snap: db.unsafe_snap(),  // atomically captures current seqno
                db,
            }
        }
    }
}
```

The `KvEngine::snapshot()` method creates a snapshot from the engine:

```rust
impl KvEngine for RocksEngine {
    type Snapshot = RocksSnapshot;

    fn snapshot(&self) -> RocksSnapshot {
        RocksSnapshot::new(self.db.clone())
    }
}
```

**What `unsafe_snap()` does internally:**
RocksDB atomically records the current sequence number and adds the snapshot to
its internal snapshot list. From this point, any SST file compaction will
refuse to discard data that the snapshot's sequence number depends on.

### 4.4 Reads From a Snapshot

All reads bind the snapshot handle to the read option, ensuring they see only
data at or before the snapshot's sequence number.

**Point read (`get_value`):**

```rust
impl Peekable for RocksSnapshot {
    fn get_value_opt(&self, opts: &ReadOptions, key: &[u8]) -> Result<Option<RocksDbVector>> {
        let opt: RocksReadOptions = opts.into();
        let mut opt = opt.into_raw();
        unsafe { opt.set_snapshot(&self.snap); }  // bind snapshot
        let v = self.db.get_opt(key, &opt).map_err(r2e)?;
        Ok(v.map(RocksDbVector::from_raw))
    }
    // get_value_cf_opt: same but with a column family handle
}
```

**Range scan (iterator):**

```rust
impl Iterable for RocksSnapshot {
    fn iterator_opt(&self, cf: &str, opts: IterOptions) -> Result<Self::Iterator> {
        let mut opt: RocksReadOptions = opts.into();
        let mut opt = opt.into_raw();
        unsafe { opt.set_snapshot(&self.snap); }  // bind snapshot
        let handle = get_cf_handle(self.db.as_ref(), cf)?;
        Ok(RocksEngineIterator::from_raw(
            DBIterator::new_cf(self.db.clone(), handle, opt)
        ))
    }
}
```

**Hint timestamps for MVCC reads:**

When TiKV knows the timestamp range of the MVCC data it needs (e.g., during
`prewrite` or `commit`), it passes `hint_min_ts` and `hint_max_ts` in
`IterOptions`. These hints allow RocksDB's Titan blob engine to skip data
outside the timestamp range:

```rust
pub struct IterOptions {
    hint_min_ts: Option<u64>,  // skip data with commit_ts < hint_min_ts
    hint_max_ts: Option<u64>,  // skip data with commit_ts > hint_max_ts
    // ...
}
```

### 4.5 Snapshot Lifecycle and Cleanup

Snapshots are managed by Rust's RAII: when a `RocksSnapshot` is dropped, its
destructor calls `release_snap`, telling RocksDB it is safe to GC the data the
snapshot was holding back.

```rust
impl Drop for RocksSnapshot {
    fn drop(&mut self) {
        unsafe {
            self.db.release_snap(&self.snap);
        }
    }
}
```

**Effect on compaction:** While a snapshot is alive, RocksDB's compaction
process cannot delete any key version with a sequence number ≥ that snapshot's
sequence number (if a newer version also exists). Long-lived snapshots can
therefore cause SST bloat. TiKV avoids this by keeping snapshots only as long
as the current read operation needs them.

---

## 5. Column Families

TiKV divides its data across four RocksDB column families (CFs), defined in
`components/engine_traits/src/cf_defs.rs`:

```rust
pub const CF_DEFAULT: CfName = "default";   // large values (size > 255 bytes)
pub const CF_LOCK:    CfName = "lock";      // pessimistic & optimistic locks
pub const CF_WRITE:   CfName = "write";     // write records (commit / rollback pointers)
pub const CF_RAFT:    CfName = "raft";      // Raft state machine metadata
```

The three data CFs together implement MVCC:

| CF | Key format | Value content | Purpose |
|---|---|---|---|
| `default` | `raw_key + start_ts` | User value bytes | Stores values larger than the inline threshold (255 B) |
| `write` | `raw_key + commit_ts` | `WriteRecord` (type + start_ts + optional short value) | Records commit or rollback events; short values are inlined here |
| `lock` | `raw_key` | `Lock` struct (type, start_ts, ttl, …) | Marks an uncommitted lock on the key; deleted on commit or rollback |

---

## 6. MVCC (Multi-Version Concurrency Control)

### 6.1 MvccTxn: Buffering Writes

`MvccTxn` is the central struct for a transaction's write operations. It
buffers all mutations in memory as `Modify` operations and does **not** write
to the engine until the transaction is committed.

**Source:** `src/storage/mvcc/txn.rs`

```rust
pub struct MvccTxn {
    pub(crate) start_ts: TimeStamp,      // transaction's start timestamp (from PD)
    pub(crate) write_size: usize,        // accumulated byte size of buffered writes
    pub(crate) modifies: Vec<Modify>,    // buffered Put/Delete operations

    // One-phase commit (1PC) optimisation: locks are kept in memory
    pub(crate) locks_for_1pc: Vec<(Key, Lock, bool)>,

    // Used to update deadlock detection when new locks are acquired
    pub(crate) new_locks: Vec<LockInfo>,

    // Async commit: in-memory lock visibility before engine write
    pub(crate) concurrency_manager: ConcurrencyManager,
    pub(crate) guards: Vec<KeyHandleGuard>,
}
```

Key constants:
```rust
// If a transaction's write size exceeds this, it is split into smaller batches
pub const MAX_TXN_WRITE_SIZE: usize = 32 * 1024;  // 32 KiB
```

### 6.2 How MVCC Keys Are Encoded

Every MVCC operation appends a timestamp to the user key:

```rust
// In CF_DEFAULT (large value storage)
pub(crate) fn put_value(&mut self, key: Key, ts: TimeStamp, value: Value) {
    let write = Modify::Put(CF_DEFAULT, key.append_ts(ts), value);
    // key stored as: raw_key + big-endian(start_ts)
}

// In CF_WRITE (commit/rollback marker)
pub(crate) fn put_write(&mut self, key: Key, ts: TimeStamp, value: Value) {
    let write = Modify::Put(CF_WRITE, key.append_ts(ts), value);
    // key stored as: raw_key + big-endian(commit_ts)
}

// In CF_LOCK (uncommitted lock)
pub(crate) fn put_lock(&mut self, key: Key, lock: &Lock, is_new: bool) {
    let write = Modify::Put(CF_LOCK, key, lock.to_bytes());
    // key stored as: raw_key (no timestamp; only one lock per key)
}
```

Timestamps are stored in **big-endian** byte order so that lexicographic key
ordering corresponds to descending timestamp order. This means a scan from a
key prefix naturally visits the most recent version first.

### 6.3 MVCC Reads via Snapshots

`SnapshotReader` wraps a `RocksSnapshot` to provide MVCC-aware reads:

**Source:** `src/storage/mvcc/reader/reader.rs`

```rust
pub struct SnapshotReader<S> {
    snapshot: S,           // underlying engine snapshot (RocksSnapshot)
    scan_mode: Option<ScanMode>,
    fill_cache: bool,
}
```

A typical MVCC read resolves the visible version of a key at `read_ts`:

1. Seek `CF_LOCK` at `raw_key` → if a lock with `lock.start_ts <= read_ts` is
   found and not rolled back, the read may need to wait for the lock to resolve.
2. Seek `CF_WRITE` backwards from `raw_key + read_ts` → find the most recent
   committed write with `commit_ts <= read_ts`.
3. If the `WriteRecord` has an inline short value, return it. Otherwise look up
   `CF_DEFAULT` at `raw_key + write.start_ts`.

All of steps 1–3 happen against the same immutable snapshot, providing
**consistent point-in-time reads**.

### 6.4 MVCC Write Record Types

The `CF_WRITE` record carries a type field that determines what happened:

| Type | Meaning |
|---|---|
| `Put` | Committed a regular value write |
| `Delete` | Committed a deletion (MVCC tombstone) |
| `Lock` | A lock-only write (no value change, used for read locks in TiDB) |
| `Rollback` | This version was rolled back; prevents re-use of the `start_ts` |

---

## 7. WriteBatch: Atomic Engine Writes

### 7.1 WriteBatch Trait

`WriteBatch` provides atomicity: either all of its accumulated operations are
persisted or none of them are.

**Source:** `components/engine_traits/src/write_batch.rs`

```rust
pub trait WriteBatch: Mutable {
    /// Commit all pending operations to disk.
    /// Returns the sequence number assigned to this batch.
    fn write_opt(&mut self, opts: &WriteOptions) -> Result<u64>;

    fn write(&mut self) -> Result<u64> {
        self.write_opt(&WriteOptions::default())
    }

    fn data_size(&self) -> usize;
    fn count(&self) -> usize;
    fn is_empty(&self) -> bool;

    /// True when the batch is large enough that it should be flushed
    /// (exceeds WRITE_BATCH_MAX_KEYS entries).
    fn should_write_to_engine(&self) -> bool;

    fn clear(&mut self);

    // Save-point mechanism for conditional rollback
    fn set_save_point(&mut self);
    fn pop_save_point(&mut self) -> Result<()>;
    fn rollback_to_save_point(&mut self) -> Result<()>;

    fn merge(&mut self, src: Self) -> Result<()> where Self: Sized;
}
```

The `write_opt()` call hands the batch atomically to RocksDB, which appends
all operations to the WAL (unless disabled) and then inserts them into the
memtable. The returned `u64` is the sequence number of this batch.

### 7.2 RocksWriteBatchVec: Multi-Batch Write

**Source:** `components/engine_rocks/src/write_batch.rs`

To avoid a single large `WriteBatch` monopolising the RocksDB write thread for
too long, TiKV uses `RocksWriteBatchVec` which splits the batch into chunks of
`WRITE_BATCH_MAX_KEY_NUM` (16) keys:

```rust
pub struct RocksWriteBatchVec {
    db: Arc<DB>,
    wbs: Vec<RawWriteBatch>,  // one inner WriteBatch per chunk
    save_points: Vec<usize>,
    index: usize,
    batch_size_limit: usize,
    support_write_batch_vec: bool,
}

const WRITE_BATCH_MAX_KEY_NUM: usize = 16;
// RocksEngine::WRITE_BATCH_MAX_KEYS = 256
```

When `MultiBatchWrite` is enabled (`support_write_batch_vec = true`), the
chunks can be consumed by multiple helper threads, each helping the leader
thread commit them. This is RocksDB's **pipelined group commit**.

The write implementation:

```rust
fn write_impl(&mut self, opts: &WriteOptions, mut cb: impl FnMut(u64)) -> Result<u64> {
    let opt: RocksWriteOptions = opts.into();
    if self.support_write_batch_vec {
        // Multi-batch: helper threads assist with sub-batches
        self.get_db().multi_batch_write_callback(
            self.as_inner(), &opt.into_raw(), |s| { cb(s); }
        ).map_err(r2e)?;
    } else {
        // Single batch: traditional write
        self.get_db().write_callback(
            &self.wbs[0], &opt.into_raw(), |s| { cb(s); }
        ).map_err(r2e)?;
    }
}
```

---

## 8. End-to-End Write Path

### 8.1 Phase 1 – Client Command Arrives at the Scheduler

**Source:** `src/storage/txn/scheduler.rs`

The `TxnScheduler` receives commands (e.g., `Prewrite`, `Commit`, `Rollback`)
from TiDB via gRPC. It:

1. Assigns a task ID.
2. Acquires **latches** on the keys involved. Latches serialize access to
   overlapping key sets among concurrent commands; they are not held across
   the network round-trip, only during local execution.
3. Dispatches the command to a thread pool worker.

The scheduler's comment describes the overall picture well:

> *TxnScheduler runs in a single-thread event loop, but command executions are
> delegated to a pool of worker threads. TxnScheduler keeps track of all the
> running commands and uses latches to ensure serialized access to the
> overlapping rows involved in concurrent commands.*

### 8.2 Phase 2 – Acquire Snapshot and Latches

Before a command can execute it needs a consistent view of the database:

```rust
// Pseudocode from scheduler
let snapshot = engine.snapshot(snap_ctx).await;
```

The snapshot is taken **once** per command execution. All reads within the
command (lock checks, version lookups, conflict detection) use this single
snapshot to ensure they see a consistent database state.

### 8.3 Phase 3 – MVCC Processing

Command handlers (e.g., `Prewrite::process_write`) create an `MvccTxn` and a
`SnapshotReader`, then call MVCC action functions:

```rust
// src/storage/txn/commands/prewrite.rs (simplified)
fn process_write(self, snapshot: S, context: WriteContext<'_, L>) -> Result<WriteResult> {
    let mut txn = MvccTxn::new(self.start_ts, context.concurrency_manager);
    let mut reader = SnapshotReader::new(self.start_ts, snapshot, true);

    for mutation in self.mutations {
        prewrite(&mut txn, &mut reader, &props, mutation, ...)?;
    }

    // txn.modifies now contains all Modify operations
    let write_data = WriteData::from_modifies(txn.into_modifies());
    Ok(WriteResult { to_be_write: write_data, ... })
}
```

Inside `prewrite()`:

- `reader` checks `CF_LOCK` (via snapshot) for conflicting locks.
- `reader` checks `CF_WRITE` (via snapshot) for write-write conflicts.
- `txn.put_lock(key, lock)` adds `Modify::Put(CF_LOCK, ...)` to the buffer.
- `txn.put_value(key, start_ts, value)` adds `Modify::Put(CF_DEFAULT, ...)`.
- `txn.put_write(key, commit_ts, write_record)` adds `Modify::Put(CF_WRITE, ...)`.

### 8.4 Phase 4 – Convert to WriteBatch and Write to RocksDB

**Source:** `components/tikv_kv/src/lib.rs` (`write_modifies`)

```rust
pub fn write_modifies(kv_engine: &impl LocalEngine, modifies: Vec<Modify>) -> Result<()> {
    let mut wb = kv_engine.write_batch();

    for modify in modifies {
        match modify {
            Modify::Put(cf, k, v)    => wb.put_cf(cf, k.as_encoded(), &v)?,
            Modify::Delete(cf, k)    => wb.delete_cf(cf, k.as_encoded())?,
            Modify::PessimisticLock(k, lock) => {
                wb.put_cf(CF_LOCK, k.as_encoded(), &lock.into_lock().to_bytes())?
            }
            Modify::DeleteRange(cf, s, e, notify_only) => {
                if !notify_only { wb.delete_range_cf(cf, s.as_encoded(), e.as_encoded())? }
            }
            Modify::Ingest(sst)      => { /* SST file ingestion */ }
        }
    }

    wb.write()?;  // atomic commit to RocksDB + WAL
    Ok(())
}
```

`wb.write()` calls `write_opt` with default `WriteOptions`, which means:

- WAL is **enabled** (default).
- Sync is **disabled** (rely on OS buffering + periodic `wal_bytes_per_sync`).

The returned sequence number from `wb.write()` identifies exactly which write
in the WAL corresponds to this transaction.

### 8.5 Data Flow Diagram

```
TiDB Client
    │
    │  gRPC (e.g., KvPrewrite)
    ▼
TxnScheduler
    ├── acquire latches on affected keys
    ├── spawn worker task
    │
    ▼
Command Handler (e.g., Prewrite::process_write)
    ├── snapshot = engine.snapshot()   ← captures current sequence number
    ├── MvccTxn::new(start_ts)
    │
    ├── SnapshotReader reads (all through snapshot):
    │     CF_LOCK  → detect conflicting locks
    │     CF_WRITE → detect write-write conflicts
    │
    ├── MvccTxn accumulates Modify operations:
    │     put_lock(key)          → Modify::Put(CF_LOCK, key, lock_bytes)
    │     put_value(key, ts, v)  → Modify::Put(CF_DEFAULT, key+ts, value)
    │     put_write(key, ts, wr) → Modify::Put(CF_WRITE, key+ts, write_record)
    │
    ▼
WriteData::from_modifies(txn.into_modifies())
    │
    ▼
write_modifies(engine, modifies)
    ├── engine.write_batch()          → RocksWriteBatchVec
    ├── wb.put_cf / delete_cf …       → buffer operations
    ├── wb.write()                    → write_opt(default WriteOptions)
    │
    ▼
RocksDB WriteBatch
    ├── Append to WAL (.log file)     ← durability
    ├── Insert into Memtable          ← read visibility
    │
    ▼
Background threads (when memtable is full):
    ├── Flush memtable → SST file (Level 0)
    ├── Compaction merges SST levels, GCs old MVCC versions
    └── Old WAL segments deleted after all their data is flushed
```

---

## 9. Memtable Flush and Checkpointing

### 9.1 Flush Operations

Once a memtable fills up, RocksDB flushes it to an immutable SST file in the
background. TiKV provides explicit flush control through `MiscExt`:

```rust
// Flush specific column families (wait=true blocks until flush completes)
fn flush_cfs(&self, cfs: &[&str], wait: bool) -> Result<()>;

// Flush a single CF
fn flush_cf(&self, cf: &str, wait: bool) -> Result<()>;

// Flush the oldest memtable (useful for time-bounded durability)
fn flush_oldest_cf(&self, wait: bool, threshold: Option<SystemTime>) -> Result<bool>;
```

The `flush_oldest_cf` implementation finds the memtable with the earliest
creation time and flushes only that one, using RocksDB's
`set_expected_oldest_key_time` to guard against races:

```rust
fn flush_oldest_cf(&self, wait: bool, age_threshold: Option<SystemTime>) -> Result<bool> {
    // find oldest memtable across all CFs
    let oldest = handles.into_iter()
        .filter_map(|h| {
            self.as_inner()
                .get_approximate_active_memtable_stats_cf(h)
                .map(|(_, time)| (h, time))
        })
        .min_by(|(_, a), (_, b)| a.cmp(b));

    if let Some((handle, time)) = oldest
        && age_threshold.is_none_or(|t| time <= t)
    {
        let mut fopts = FlushOptions::default();
        fopts.set_wait(wait);
        fopts.set_allow_write_stall(true);
        fopts.set_expected_oldest_key_time(time);
        self.as_inner().flush_cf(handle, &fopts).map_err(r2e)?;
        return Ok(true);
    }
    Ok(false)
}
```

**Relationship between flush and WAL:** After a memtable is flushed to an SST
file, all WAL records that correspond to writes in that memtable are no longer
needed for crash recovery. RocksDB tracks this internally and recycles (deletes
or reuses) those WAL segments automatically, subject to `wal_ttl_seconds` and
`wal_size_limit`.

### 9.2 Checkpointing (Point-in-Time Backup)

A **checkpoint** is a consistent, point-in-time snapshot of the entire
RocksDB database that can be taken without stopping writes. It works by:

1. Hard-linking all current SST files into the checkpoint directory.
2. If the current WAL is small enough (below `log_size_for_flush`), copying it;
   otherwise flushing the memtable first so only SST files are needed.

**Source:** `components/engine_traits/src/checkpoint.rs`

```rust
pub trait Checkpointable {
    type Checkpointer: Checkpointer;
    fn new_checkpointer(&self) -> Result<Self::Checkpointer>;
    fn merge(&self, dbs: &[&Self]) -> Result<()>;
}

pub trait Checkpointer {
    fn create_at(
        &mut self,
        db_out_dir: &Path,
        titan_out_dir: Option<&Path>,
        log_size_for_flush: u64,  // flush if WAL > this size
    ) -> Result<()>;
}
```

**Source:** `components/engine_rocks/src/checkpoint.rs`

```rust
impl Checkpointable for RocksEngine {
    fn new_checkpointer(&self) -> Result<Self::Checkpointer> {
        Ok(RocksEngineCheckpointer(self.as_inner().new_checkpointer()?))
    }
}

impl Checkpointer for RocksEngineCheckpointer {
    fn create_at(&mut self, db_out_dir: &Path, titan_out_dir: Option<&Path>,
                 log_size_for_flush: u64) -> Result<()> {
        self.0.create_at(db_out_dir, titan_out_dir, log_size_for_flush)
            .map_err(r2e)
    }
}
```

Checkpoints are used by the **backup** subsystem
(`components/backup/`) and by **Raftstore snapshot generation** to produce
consistent data snapshots without blocking reads or writes.

**Important:** Background work (compaction) must not be paused during a
checkpoint (the test in checkpoint.rs actually verifies that `create_at` fails
when background work is paused):

```rust
engine.pause_background_work().unwrap();
check_pointer.create_at(path2.as_path(), None, 0).unwrap_err(); // expected failure
engine.continue_background_work().unwrap();
check_pointer.create_at(path2.as_path(), None, 0).unwrap();     // succeeds
```

---

## 10. Key Source Files Reference

### Engine Traits (Abstract Interfaces)

| File | What it defines |
|---|---|
| `components/engine_traits/src/snapshot.rs` | `Snapshot` trait |
| `components/engine_traits/src/snapshot_misc.rs` | `SnapshotMiscExt` – `sequence_number()` |
| `components/engine_traits/src/misc.rs` | `MiscExt` – flush, sync_wal, get_latest/oldest_sequence_number |
| `components/engine_traits/src/checkpoint.rs` | `Checkpointable`, `Checkpointer` |
| `components/engine_traits/src/write_batch.rs` | `WriteBatch`, `Mutable`, `WriteBatchExt` |
| `components/engine_traits/src/options.rs` | `WriteOptions`, `ReadOptions`, `IterOptions` |
| `components/engine_traits/src/cf_defs.rs` | Column family name constants |

### RocksDB Implementation

| File | What it implements |
|---|---|
| `components/engine_rocks/src/snapshot.rs` | `RocksSnapshot` |
| `components/engine_rocks/src/engine.rs` | `RocksEngine`, `KvEngine::snapshot()`, `KvEngine::sync()` |
| `components/engine_rocks/src/write_batch.rs` | `RocksWriteBatchVec` |
| `components/engine_rocks/src/checkpoint.rs` | `RocksEngineCheckpointer` |
| `components/engine_rocks/src/misc.rs` | `MiscExt` for RocksEngine (flush, sync_wal, sequence numbers) |
| `components/engine_rocks/src/options.rs` | `WriteOptions` → `RawWriteOptions` conversion (WAL disable) |

### Write Path & MVCC

| File | What it contains |
|---|---|
| `src/storage/txn/scheduler.rs` | `TxnScheduler` – command dispatch, latches |
| `src/storage/mvcc/txn.rs` | `MvccTxn` – buffering writes as `Modify` operations |
| `src/storage/mvcc/reader/reader.rs` | `SnapshotReader` – MVCC reads from snapshots |
| `src/storage/txn/commands/prewrite.rs` | `Prewrite` command handler |
| `src/storage/txn/commands/commit.rs` | `Commit` command handler |
| `components/tikv_kv/src/lib.rs` | `Modify` enum, `write_modifies()` |

### Configuration

| File | What it configures |
|---|---|
| `src/config/mod.rs` | `DbConfig` – all WAL and RocksDB options |

---

*Raft internals are intentionally excluded; this document focuses solely on the storage engine layer.*
