# Flow Research: Write Path

## Scope

This flow covers `DB::Put`, `DB::Write`, `WriteBatch`, `WriteThread`, WAL
append/sync, memtable insertion, sequence number publication, snapshots,
write stall, atomic flush, and column families.

## Source Evidence

| Step | Source |
|---|---|
| Public API | `include/rocksdb/db.h`, `include/rocksdb/write_batch.h` |
| Implementation | `db/db_impl/db_impl_write.cc` |
| Write grouping | `db/write_thread.h`, `db/write_thread.cc` |
| Batch replay | `db/write_batch.cc`, `db/write_batch_internal.h` |
| WAL | `db/log_writer.cc`, `db/log_format.h` |
| Memtable | `db/memtable.cc`, `db/memtable_list.cc` |
| Sequence numbers | `db/version_set.h`, `db/version_set.cc`, `db/dbformat.h` |

## Call Chain

`DB::Put`
-> `DB::Write`
-> `DBImpl::Write`
-> `DBImpl::WriteImpl`
-> `WriteThread::JoinBatchGroup`
-> leader `DBImpl::PreprocessWrite`
-> `WriteThread::EnterAsBatchGroupLeader`
-> `DBImpl::WriteGroupToWAL` or `ConcurrentWriteGroupToWAL`
-> `log::Writer::AddRecord`
-> assign sequence range from `VersionSet`
-> optional WAL sync / manifest WAL edit
-> `WriteBatchInternal::InsertInto`
-> `MemTableInserter`
-> `MemTable::Add`
-> `versions_->SetLastSequence`
-> `WriteThread::ExitAsBatchGroupLeader`.

## Detailed Behavior

- `DBImpl::Write` updates per-key protection info when requested, then calls
  `WriteImpl`.
- `WriteImpl` rejects invalid combinations such as `sync && disableWAL`,
  pipelined write with incompatible options, `disableWAL` with recycled WALs,
  and DeleteRange with row cache.
- The write group leader reserves a sequence range after WAL write. The range is
  per key by default, or per batch when `seq_per_batch_` is active.
- WAL is written before memtable insertion in the normal path. If sync is
  requested, WAL state can be applied to MANIFEST through `ApplyWALToManifest`.
- `WriteBatchInternal::InsertInto` replays user operations through
  `MemTableInserter`. `MemTable::Add` encodes internal key plus value into the
  memtable arena and updates bloom filters/counters.
- `versions_->SetLastSequence` publishes visibility only after successful
  memtable insertion, preventing readers from seeing a sequence that has no data.
- Column families are carried in batch records; `ColumnFamilyMemTablesImpl`
  maps column family IDs to the right memtable during insertion.

## Snapshot Visibility

Snapshots compare their sequence number to internal key sequence numbers. A
snapshot cannot see keys with a sequence greater than the snapshot. The write
path matters because publishing `LastSequence` is the point at which future
implicit snapshots can include the newly written sequence range.

## Write Stalls

Foreground writes can be delayed or stopped by:

- too many immutable memtables waiting for flush;
- too many L0 files (`level0_slowdown_writes_trigger`,
  `level0_stop_writes_trigger`);
- soft or hard pending compaction bytes;
- write buffer manager or DB-wide write buffer limits;
- low-priority write throttling.

Evidence is in `DBImpl::PreprocessWrite`, write controller code, and option
defaults in `include/rocksdb/options.h` and `include/rocksdb/advanced_options.h`.

## Sync Write

`WriteOptions::sync=true` requires WAL. The leader writes the WAL record and
syncs according to WAL writer state. Relevant diagnostics are `WAL_FILE_SYNCED`,
`WAL_FILE_SYNC_MICROS`, `PerfContext::write_wal_time`, and filesystem
`IOStatsContext::fsync_nanos`.

## Atomic Flush and Column Families

Atomic flush does not make a write batch atomic across CFs; it makes flushing
memtables across CFs publish atomically. The write path stores CF IDs in batch
records, while `Flush`/`AtomicFlushMemTables` later commit file metadata for
selected CF memtables together.

## Tests

Use `db/db_write_test.cc`, `db/write_batch_test.cc`, `db/db_wal_test.cc`,
`db/write_callback_test.cc`, `db/db_kv_checksum_test.cc`,
`db/write_controller_test.cc`, and transaction tests for two-queue and
WritePrepared/WriteUnprepared behavior.
