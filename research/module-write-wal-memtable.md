# Module Research: WriteThread, WriteBatch, WAL, MemTable, SkipList, Arena

## Module Responsibilities

The write subsystem turns user mutations into a durable and visible sequence of
internal keys. It owns group commit, WAL append and optional sync, sequence
assignment, memtable insertion, memtable bloom updates, and triggering flush
pressure. It does not choose compaction inputs or table block format.

## Core Source

| Path | Class / method | Role |
|---|---|---|
| `db/db_impl/db_impl_write.cc` | `DBImpl::Write`, `DBImpl::WriteImpl` | Validates write options, joins write group, writes WAL, inserts memtable, publishes last sequence. |
| `db/write_thread.h`, `db/write_thread.cc` | `WriteThread`, `WriteThread::Writer`, `WriteThread::JoinBatchGroup`, `WriteThread::EnterAsBatchGroupLeader` | Group commit coordination and leader/follower state machine. |
| `include/rocksdb/write_batch.h`, `db/write_batch.cc`, `db/write_batch_internal.h` | `WriteBatch`, `WriteBatchInternal::InsertInto`, `MemTableInserter` | Write batch record format and replay into memtables. |
| `db/log_writer.h`, `db/log_writer.cc` | `log::Writer::AddRecord` | WAL physical record fragmentation, checksum, and file append. |
| `db/log_reader.h`, `db/log_reader.cc` | `log::Reader::ReadRecord` | WAL/manifest physical record reader used by recovery. |
| `db/memtable.h`, `db/memtable.cc` | `MemTable::Add`, `MemTable::Get`, `MemTable::ShouldFlushNow` | In-memory sorted write buffer. |
| `db/memtable_list.h`, `db/memtable_list.cc` | `MemTableList::PickMemtablesToFlush`, `TryInstallMemtableFlushResults` | Immutable memtable queue and flush commit. |
| `memtable/skiplist.h`, `memtable/inlineskiplist.h`, `memtable/skiplistrep.cc` | `SkipList`, `InlineSkipList`, `SkipListRepFactory` | Default sorted memtable representation. |
| `memory/arena.h`, `util/arena.h` | `Arena` | Append-style allocation for memtable entries and iterator scratch. |

## Data Structures

| Structure | Data layout and lifecycle |
|---|---|
| `WriteBatch` | Header stores sequence and count; records encode column family and operation. `WriteBatchInternal::SetSequence` assigns the base seq before `MemTableInserter` replay. |
| `WriteThread::Writer` | Per-writer state containing batch, callbacks, sequence, status, WAL metadata, and group linkage. |
| WAL record | `log::Writer::AddRecord` writes logical records into 32KB log blocks using physical record fragments with checksum and type. |
| `MemTable` entry | `MemTable::Add` encodes varint internal key size, user key, packed sequence/type, varint value size, value, and optional protection bytes. |
| Memtable bloom | `MemTable::Add` updates prefix and/or whole-key bloom filters when configured. |
| Immutable memtable list | `MemTableList` tracks not-flushed memtables, history, flush state flags, and install/rollback. |

## Key Flow

`DBImpl::Write`
-> `DBImpl::WriteImpl`
-> `WriteThread::JoinBatchGroup`
-> leader calls `PreprocessWrite`
-> `WriteThread::EnterAsBatchGroupLeader`
-> `DBImpl::WriteGroupToWAL` / `ConcurrentWriteGroupToWAL`
-> sequence range assignment
-> `WriteBatchInternal::InsertInto`
-> `MemTableInserter`
-> `MemTable::Add`
-> `versions_->SetLastSequence`
-> `WriteThread::ExitAsBatchGroupLeader`.

Important source evidence:

- `DBImpl::WriteImpl` validates `sync` with `disableWAL`, pipelined write
  compatibility, unordered write compatibility, and row cache/DeleteRange
  incompatibility in `db/db_impl/db_impl_write.cc`.
- Normal write group leader writes WAL before memtable insertion in
  `DBImpl::WriteImpl`, with perf timers `write_wal_time` and
  `write_memtable_time`.
- `WriteBatchInternal::InsertInto` iterates every writer's batch and uses
  `MemTableInserter` in `db/write_batch.cc`.
- `MemTable::Add` constructs the exact internal entry layout and updates flush
  state in `db/memtable.cc`.

## Concurrency Model

| Part | Model |
|---|---|
| Write group | `WriteThread` elects one leader; followers may be completed by the leader. |
| Parallel memtable write | Enabled when `allow_concurrent_memtable_write` is true, group size is greater than one, and batches do not contain unsupported operations such as merge. |
| WAL write | Serialized by write group leader and WAL synchronization logic; `sync` writes can force manifest WAL state updates. |
| Memtable insert | Default skiplist supports concurrent insert through concurrent APIs; non-concurrent path updates counters and flush state inline. |
| Flush pressure | `MemTable::UpdateFlushState` and write preprocessing can schedule memtable switch and background flush. |

## Configuration

| Option | Default | Effect |
|---|---:|---|
| `write_buffer_size` | `64MB` | Mutable memtable flush threshold. |
| `max_write_buffer_number` | `2` | Number of write buffers before write stall/stop. |
| `min_write_buffer_number_to_merge` | `1` | Number of immutable memtables to merge into a flush. |
| `allow_concurrent_memtable_write` | `true` | Allows parallel insert into compatible memtable reps. |
| `enable_pipelined_write` | `false` | Splits WAL and memtable stages into separate queues. |
| `unordered_write` | `false` | Relaxes snapshot immutability for throughput. |
| `manual_wal_flush` | `false` | User controls WAL flushing; affects write option validation. |
| `wal_bytes_per_sync` | `0` | Background WAL writeback interval. |
| `sync` in `WriteOptions` | `false` | Requests WAL sync before write completion. |
| `disableWAL` in `WriteOptions` | `false` | Bypasses WAL; invalid with `sync`. |

## Metrics, PerfContext, Logs

| Signal | Use |
|---|---|
| `NUMBER_KEYS_WRITTEN`, `BYTES_WRITTEN`, `WRITE_DONE_BY_SELF`, `WRITE_DONE_BY_OTHER` | Grouping and write volume. |
| `WRITE_WITH_WAL`, `WAL_FILE_BYTES`, `WAL_FILE_SYNCED`, `WAL_FILE_SYNC_MICROS` | WAL cost and sync behavior. |
| `PerfContext::write_wal_time`, `write_memtable_time`, `write_delay_time`, `write_thread_wait_nanos` | Break down write latency. |
| `STALL_MICROS`, `WRITE_STALL` | Backpressure. |
| LOG lines around memtable switch and WAL rotation | Explain flush pressure and WAL retention. |

## Tests

| Test | Coverage |
|---|---|
| `db/db_write_test.cc` | Write modes, group writes, failure cases. |
| `db/write_batch_test.cc` | WriteBatch encoding and replay. |
| `db/db_wal_test.cc`, `db/log_test.cc` | WAL record writing, reading, sync, archival. |
| `db/write_callback_test.cc` | Write callbacks and write-thread sync points. |
| `db/db_kv_checksum_test.cc` | WAL/memtable protection info corruption checks. |
| `db/db_memtable_test.cc`, `memtable/skiplist_test.cc` | Memtable representation and lookup behavior. |

## Operational and Development Concerns

- WAL-before-memtable ordering is a crash recovery invariant. Changing the
  order requires a recovery proof.
- `disableWAL` improves latency but moves durability responsibility to the
  application and increases data-loss risk.
- Write stalls commonly come from L0 file count, pending compaction bytes, too
  many immutable memtables, or shared write buffer manager pressure.
- Concurrent memtable writes are hot-path code; avoid allocations, virtual work,
  or heavy metrics in `MemTable::Add` and `MemTableInserter`.
- Validate write changes with `db_write_test`, `write_batch_test`, WAL tests,
  and `db_stress` configurations that exercise sync, no-WAL, transactions, and
  column families.
