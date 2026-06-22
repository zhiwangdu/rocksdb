# Module Research: DB Core, Column Families, Snapshots, Options, Metrics

## Module Responsibilities

DB core owns the process-wide database state for one opened DB: public API
dispatch, column family registry, SuperVersion lifetime, global sequence
numbers, foreground write/read coordination, background work scheduling, and
error state. It does not own the SST block format, low-level filesystem
implementation, or transaction conflict policy; those are delegated to table,
Env/FileSystem, and utilities/transactions modules.

## Core Source

| Path | Class / method | Role |
|---|---|---|
| `include/rocksdb/db.h` | `DB` | Public DB interface and API contracts. |
| `db/db_impl/db_impl.h` | `DBImpl` | Main DB implementation object. |
| `db/db_impl/db_impl.cc` | `DBImpl::GetImpl`, `DBImpl::NewIterator`, `DBImpl::IngestExternalFile` | Read path, iterator creation, ingestion entry points. |
| `db/db_impl/db_impl_write.cc` | `DBImpl::Write`, `DBImpl::WriteImpl` | Foreground write path and write group orchestration. |
| `db/db_impl/db_impl_open.cc` | `DBImpl::Recover`, `DBImpl::RecoverLogFiles` | Open and recovery. |
| `db/db_impl/db_impl_compaction_flush.cc` | `DBImpl::Flush`, `DBImpl::BackgroundFlush`, `DBImpl::BackgroundCompaction`, `DBImpl::MaybeScheduleFlushOrCompaction` | Manual and background flush/compaction scheduling. |
| `db/column_family.h`, `db/column_family.cc` | `ColumnFamilyData`, `SuperVersion`, `ColumnFamilySet` | CF-local mutable state and reader-visible state snapshots. |
| `db/dbformat.h` | `SequenceNumber`, `InternalKey`, `ValueType` | Internal key ordering and visibility metadata. |
| `include/rocksdb/options.h`, `include/rocksdb/advanced_options.h`, `include/rocksdb/table.h` | `Options`, `DBOptions`, `ColumnFamilyOptions`, `BlockBasedTableOptions` | Configuration surface. |
| `include/rocksdb/listener.h` | `EventListener` | Operational callback hooks. |
| `include/rocksdb/statistics.h`, `include/rocksdb/perf_context.h`, `include/rocksdb/iostats_context.h` | `Statistics`, `PerfContext`, `IOStatsContext` | Observability surface. |

## Core Data Structures

| Structure | In-memory state | Disk / metadata state | References |
|---|---|---|---|
| `DBImpl` | DB mutex, WAL writers, write threads, flush/compaction queues, `VersionSet`, `ColumnFamilyMemTables`, background error handler | DB directory, WAL directory, CURRENT/MANIFEST/SST files via helpers | Owns `VersionSet`; references `ColumnFamilyData`; uses Env/FileSystem and table cache. |
| `ColumnFamilyData` | Mutable memtable, immutable memtable list, current `Version`, options, compaction picker, internal stats | Column family ID and schema stored in MANIFEST via `VersionEdit` | Referenced by handles, SuperVersions, jobs, and VersionSet. |
| `SuperVersion` | Reader-visible tuple of mutable memtable, immutable memtable list, and current `Version` | None directly; points to current version metadata | Acquired by readers through `ColumnFamilyData::GetReferencedSuperVersion`; replaced by flush/compaction/open/config changes. |
| `SequenceNumber` | Global monotonic sequence in `VersionSet`; write groups reserve ranges | Persisted in WAL batch headers and MANIFEST last sequence | Read snapshots compare internal key seq against snapshot seq. |
| `Snapshot` | Snapshot sequence number and list links | Not a durable object | Held by read options and transaction layers. |

## Key Flows

### API Dispatch

`DB::Put` / `DB::Write` in `include/rocksdb/db.h`
-> `DBImpl::Write` in `db/db_impl/db_impl_write.cc`
-> `DBImpl::WriteImpl`
-> write thread, WAL, memtable.

`DB::Get` in `include/rocksdb/db.h`
-> `DBImpl::Get` in `db/db_impl/db_impl.cc`
-> `DBImpl::GetImpl`
-> SuperVersion acquisition
-> memtable / immutable memtable lookup
-> `Version::Get` in `db/version_set.cc`.

`DB::NewIterator`
-> `DBImpl::NewIterator`
-> `ColumnFamilyData::GetReferencedSuperVersion`
-> `DBImpl::NewIteratorImpl`
-> `DBIter::NewIter` over merged internal iterators.

### SuperVersion Swap

Flush, compaction, recovery, and config changes create or update a `Version`.
They then install a new `SuperVersion` through
`ColumnFamilyData::InstallSuperVersion` in `db/column_family.cc`. Readers keep
old memtables and versions alive through references until cleanup.

## Concurrency Model

| Component | Concurrency behavior |
|---|---|
| `DBImpl::mutex_` | Protects VersionSet mutation, CF state changes, flush/compaction queues, and most background scheduling decisions. |
| Write path | `WriteThread` serializes group leaders and optionally launches parallel memtable writers; WAL writes use WAL coordination and sequence publishing. |
| Read path | Avoids DB mutex in common path by referencing `SuperVersion`; table/memtable objects remain alive by refcounts. |
| Background work | `MaybeScheduleFlushOrCompaction` posts work to Env thread pools. Flush/compaction release DB mutex during file IO and reacquire before manifest install. |
| Event listeners | `include/rocksdb/listener.h` documents that callbacks must be cheap; many run on write, flush, compaction, or manifest-sensitive paths. |

## Configuration

See `research/notes/options-metrics-tests.md` for the option table. Core
options most likely to change DB-level behavior are `max_background_jobs`,
`max_open_files`, `max_total_wal_size`, `db_write_buffer_size`,
`write_buffer_manager`, `paranoid_checks`, `atomic_flush`,
`allow_concurrent_memtable_write`, `enable_pipelined_write`, and
`unordered_write`.

## Metrics, PerfContext, Logs

| Signal | Source evidence | Use |
|---|---|---|
| `DB_GET`, `DB_WRITE` histograms | `include/rocksdb/statistics.h`, `StopWatch` in DBImpl write/get paths | User-visible latency. |
| `STALL_MICROS`, `WRITE_STALL` | `include/rocksdb/statistics.h`, write controller code | Foreground backpressure. |
| `PerfContext::db_mutex_lock_nanos`, `write_thread_wait_nanos` | `include/rocksdb/perf_context.h` | Distinguish DB mutex contention from write group wait. |
| Event log entries | `db/flush_job.cc`, `db/compaction/compaction_job.cc`, `db/db_impl/db_impl_open.cc` | Recovery, flush, compaction, and open timeline. |
| `EventListener::OnStallConditionsChanged` | `include/rocksdb/listener.h` | External alerting for stall state changes. |

## Tests

| Test file | Coverage |
|---|---|
| `db/db_basic_test.cc` | DB API behavior, open/close, basic get/put/delete. |
| `db/column_family_test.cc` | Column family lifecycle, handles, SuperVersion side effects. |
| `db/db_options_test.cc`, `options/options_test.cc` | Option parsing, defaults, dynamic changes. |
| `db/db_statistics_test.cc`, `db/perf_context_test.cc` | Statistics and PerfContext accounting. |
| `db/listener_test.cc` | Listener callbacks and callback ordering. |

## Operational and Development Concerns

- The DB mutex is a correctness boundary. Do not perform slow IO or listener
  work while holding it unless existing code proves the path is expected.
- SuperVersion lifetime bugs usually manifest as stale reads, use-after-free, or
  missing newly flushed/compacted data. Validate with iterator/read tests plus
  stress tests.
- Sequence number publication is a database-wide invariant; write, recovery,
  and transaction code must agree on reservation and visibility.
- `max_background_jobs`, L0 triggers, and pending compaction byte limits are the
  main operators' levers when writes slow down.
- Adding core APIs requires updates across Make/CMake/BUCK if new files are
  added and must follow public API compatibility rules from `CLAUDE.md`.
