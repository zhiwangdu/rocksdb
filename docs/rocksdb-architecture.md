# RocksDB Architecture

This document summarizes RocksDB as an LSM storage engine implementation, with
source evidence for the development and operations boundaries. Research notes
used for this synthesis live under `research/`.

## 1. Architecture at a Glance

RocksDB is organized around one `DBImpl` object per opened database, one
`ColumnFamilyData` object per column family, and one `Version` per current
reader-visible LSM view. Foreground writes enter through `DBImpl::WriteImpl`,
foreground reads enter through `DBImpl::GetImpl` or `DBImpl::NewIterator`, and
background work is scheduled from `DBImpl::MaybeScheduleFlushOrCompaction`.

Source evidence:

- API surface: `include/rocksdb/db.h`
- Main implementation: `db/db_impl/db_impl.h`, `db/db_impl/db_impl.cc`,
  `db/db_impl/db_impl_write.cc`
- Background scheduling: `db/db_impl/db_impl_compaction_flush.cc`
- Column families and SuperVersion: `db/column_family.h`,
  `db/column_family.cc`
- Version metadata: `db/version_set.h`, `db/version_set.cc`

## 2. Main Layers

| Layer | Main files | Core classes | Responsibilities |
|---|---|---|---|
| Public API | `include/rocksdb/db.h`, `include/rocksdb/options.h` | `DB`, `Options`, `ReadOptions`, `WriteOptions` | API contract and configuration. |
| DB core | `db/db_impl/*` | `DBImpl` | Open, read/write dispatch, WAL ownership, background scheduling, errors. |
| Column family state | `db/column_family.*` | `ColumnFamilyData`, `SuperVersion` | Mutable/immutable memtables, current Version, CF options, reader-visible state. |
| Write buffer | `db/memtable.*`, `memtable/*` | `MemTable`, `MemTableList`, `SkipList`, `InlineSkipList`, `Arena` | In-memory sorted writes and flush queue. |
| WAL | `db/log_writer.*`, `db/log_reader.*`, `db/wal_manager.*` | `log::Writer`, `log::Reader`, `WalManager` | Durability before memtable visibility and recovery replay. |
| Metadata | `db/version_set.*`, `db/version_edit.*` | `VersionSet`, `Version`, `VersionEdit`, `FileMetaData` | MANIFEST/CURRENT, live file metadata, compaction scores. |
| SST/table | `table/*`, `table/block_based/*` | `TableBuilder`, `TableReader`, `BlockBasedTable`, `Block`, `FilterBlockReader` | Immutable sorted-run format, filters, indexes, block IO. |
| Cache | `include/rocksdb/cache.h`, `cache/*`, `table/block_based/block_cache.cc` | `Cache`, `LRUCache`, `ClockCache` | Block and secondary cache. |
| Compaction | `db/compaction/*` | `CompactionPicker`, `CompactionJob`, `CompactionIterator` | Rewrite SSTs to control amplification and reclaim obsolete data. |
| Utilities | `utilities/*` | `BackupEngine`, `Checkpoint`, `TransactionDB` | Optional features layered around core DB. |
| Platform | `include/rocksdb/env.h`, `include/rocksdb/file_system.h`, `env/*`, `file/*` | `Env`, `FileSystem`, file readers/writers | Thread pools, time, filesystem, rate-limited IO. |

## 3. Ownership Boundaries

`DBImpl` owns policy and scheduling. It decides when to switch memtables, write
WALs, flush, compact, and enter error states. `MemTable` owns in-memory record
layout and lookup. `VersionSet` owns durable metadata and sequence/file-number
state. `BlockBasedTable` owns SST block lookup, filters, and cache metrics.
`Env`/`FileSystem` owns portability and IO implementation.

The most important boundary is SuperVersion. Readers get a referenced
`SuperVersion` from `ColumnFamilyData::GetReferencedSuperVersion`, then read
without holding the DB mutex. Flush/compaction install a replacement through
`ColumnFamilyData::InstallSuperVersion` only after metadata is safe.

## 4. Concurrency Model

| Path | Model | Source evidence |
|---|---|---|
| Foreground write | `WriteThread` groups writers, elects a leader, writes WAL, inserts memtables, and publishes sequence. | `db/write_thread.*`, `db/db_impl/db_impl_write.cc` |
| Foreground point read | References SuperVersion and searches memtables/SSTs without DB mutex in common path. | `db/db_impl/db_impl.cc`, `db/version_set.cc` |
| Iterator | Holds SuperVersion/table references through cleanup chains; not thread-safe per iterator. | `db/db_iter.*`, `db/arena_wrapped_db_iter.cc`, `table/merging_iterator.cc` |
| Flush | Background thread writes L0 file without DB mutex, then reacquires mutex for manifest install. | `db/flush_job.cc`, `db/db_impl/db_impl_compaction_flush.cc` |
| Compaction | Background worker can split into subcompactions, writes outputs, then serializes manifest commit. | `db/compaction/compaction_job.cc` |
| Manifest | `VersionSet::LogAndApply` queues manifest writers and serializes metadata mutation. | `db/version_set.cc` |

## 5. Crash Consistency Model

RocksDB relies on two durable logs:

- WAL stores user writes before they become memtable-visible.
- MANIFEST stores file metadata edits before SST membership changes become
  reader-visible after restart.

Normal write ordering is WAL append/sync first, memtable insert second, sequence
publication last. Flush/compaction writes output files first, commits a
`VersionEdit` to MANIFEST second, and only then publishes a new SuperVersion.
Recovery uses `VersionSet::Recover` for MANIFEST and `DBImpl::RecoverLogFiles`
for WAL replay.

## 6. Observability Surface

| Surface | Source | Use |
|---|---|---|
| Tickers/histograms | `include/rocksdb/statistics.h` | Cache, WAL, write, read, flush, compaction, stall counters. |
| PerfContext | `include/rocksdb/perf_context.h` | Per-thread latency breakdown for reads, writes, iterators, block IO. |
| IOStatsContext | `include/rocksdb/iostats_context.h` | Filesystem read/write/open/fsync timing and bytes. |
| EventListener | `include/rocksdb/listener.h` | Flush, compaction, external ingest, background error, stall callbacks. |
| LOG/event logger | `db/*`, `db/compaction/*`, `db/flush_job.cc` | Open/recovery, flush, compaction, file lifecycle, errors. |

## 7. Development Guidance

- Start core behavior changes at `DBImpl`, but push format-specific behavior
  down to table/memtable modules and metadata behavior into VersionSet.
- Treat `VersionSet::LogAndApply`, `MemTable::Add`, `BlockBasedTable::Get`, and
  `CompactionIterator` as high-risk hot paths.
- Any change that touches sequence assignment must be checked against
  snapshots, recovery, and transactions.
- Any change that touches file membership must be checked against MANIFEST
  recovery, obsolete file cleanup, backup/checkpoint, and ingestion.
- Run focused tests first, then broader `db_stress` or `make check` when the
  blast radius reaches DBImpl, VersionSet, compaction, or recovery.
