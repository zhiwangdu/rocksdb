# RocksDB Recovery and Troubleshooting

This document covers DB open/recovery and operational failure analysis.

## 1. Recovery Call Chain

`DB::Open`
-> `DBImpl::Recover` (`db/db_impl/db_impl_open.cc`)
-> DB lock and directory setup
-> CURRENT/MANIFEST discovery
-> `VersionSet::Recover` (`db/version_set.cc`)
-> `VersionEdit::DecodeFrom` (`db/version_edit.cc`)
-> current Versions rebuilt
-> WAL files selected
-> `DBImpl::RecoverLogFiles`
-> `DBImpl::ProcessLogFile`
-> `log::Reader::ReadRecord`
-> `WriteBatchInternal::InsertInto`
-> recovered memtables
-> final flush or active WAL restore
-> publish recovered sequence.

## 2. Crash Consistency

WAL protects writes not yet flushed. MANIFEST protects SST membership. A flush
or compaction output file is not live until a durable `VersionEdit` says it is.
On restart, RocksDB trusts durable MANIFEST records and then replays WALs newer
than the minimum log number needed by unflushed data.

## 3. Corruption and Missing Files

Key source paths:

- `VersionSet::Recover` reads CURRENT and MANIFEST.
- `DBImpl::ProcessLogFile` reads WAL records and applies recovery mode.
- `VersionEdit::DecodeFrom` parses durable metadata edits.
- `db/corruption_test.cc`, `db/fault_injection_test.cc`, and `db/repair_test.cc`
  cover failure cases.

Options that change behavior:

- `wal_recovery_mode`
- `paranoid_checks`
- `best_efforts_recovery`
- `create_if_missing`
- `error_if_exists`

## 4. Troubleshooting Matrix

| Scenario | Symptoms | Likely causes | Config / metrics | Source evidence | Steps / optimization |
|---|---|---|---|---|---|
| Writes slow | High write latency | WAL sync, write group wait, memtable insert CPU, stalls | `DB_WRITE`, `PerfContext::write_wal_time`, `write_memtable_time`, `write_thread_wait_nanos` | `DBImpl::WriteImpl`, `WriteThread` | Separate WAL vs memtable time; inspect sync writes and stall counters. |
| Write stall | Writes delayed/stopped | L0 count, immutable memtables, pending compaction bytes | `STALL_MICROS`, `WRITE_STALL`, `level0_*`, pending bytes | write controller, `MaybeScheduleFlushOrCompaction` | Increase compaction/flush capacity or reduce write burst/LSM debt. |
| WAL sync slow | Sync writes slow | Filesystem fsync latency, WAL device contention, rate limiter | `WAL_FILE_SYNC_MICROS`, `IOStatsContext::fsync_nanos` | `log::Writer::AddRecord`, `DBImpl::SyncWAL` | Check storage latency, sync frequency, WAL dir placement. |
| MemTable buildup | Memory grows, WAL retained | Flush cannot keep up, high write buffers, blocked flush | `max_write_buffer_number`, immutable count, LOG flush queue | `MemTableList`, `BackgroundFlush` | Add flush workers, tune write buffers, inspect background errors. |
| Flush behind | L0 creation slow | Slow SST writes, compression, file sync, background error | `FLUSH_TIME`, `FILE_WRITE_FLUSH_MICROS`, `FLUSH_WRITE_BYTES` | `FlushJob::WriteLevel0Table` | Check IOStats, compression, direct IO, background errors. |
| Compaction behind | L0 grows, space/read amp high | Too few compaction threads, slow disk, bad picker pressure | `COMPACTION_TIME`, `COMPACT_*`, pending compaction bytes | `CompactionJob`, compaction pickers | Increase `max_background_jobs`, tune level sizes, inspect logs. |
| Too many L0 files | Reads slow, stalls | Flush faster than compaction | `level0_file_num_compaction_trigger`, `level0_slowdown_writes_trigger`, `level0_stop_writes_trigger` | `VersionStorageInfo::ComputeCompactionScore` | Tune compaction capacity and L0 thresholds. |
| Read amp high | Many files touched per Get | L0 overlap, weak filters, compaction backlog | `GET_HIT_L0`, `BLOOM_FILTER_*`, `BLOCK_CACHE_*` | `Version::Get`, `BlockBasedTable::Get` | Fix compaction debt, add filters, improve cache/index options. |
| Space amp high | Disk grows faster than live data | Snapshots, tombstones, universal overlap, backups | live/obsolete files, compaction drop counters | `CompactionIterator`, obsolete cleanup | Release snapshots, compact bottommost, inspect backups/checkpoints. |
| MANIFEST large | Slow open/recovery | Frequent metadata edits, no rollover/reuse, CF churn | open LOG, manifest size, `MANIFEST_FILE_SYNC_MICROS` | `VersionSet::LogAndApply`, `VersionSet::Recover` | Inspect edit rate, manifest options, CF lifecycle. |
| WAL files pile up | Disk usage/recovery grows | Unflushed CF, disabled flush, checkpoints/backup, 2PC prep | WAL file list, `max_total_wal_size` | `DBImpl::RecoverLogFiles`, `MinLogNumberToKeep` | Flush lagging CFs, check prepared txns and backups. |
| Block cache hit low | High disk reads | Small cache, scan pollution, disabled fill_cache, bad pinning | `BLOCK_CACHE_*`, `PerfContext::block_read_count` | `BlockBasedTable::UpdateCache*Metrics` | Size cache, cache index/filter, isolate scans. |
| Bloom poor | Many false positives | No filter, wrong prefix, high bits/key mismatch | `BLOOM_FILTER_USEFUL`, `BLOOM_FILTER_FULL_POSITIVE` | `FullFilterKeyMayMatch` | Use full-key or correct prefix filters; validate prefix domain. |
| Iterator slow | Seek/Next high latency | Tombstones, old versions, range deletions, many files | `DB_SEEK`, skipped PerfContext counters | `DBIter`, `MergingIterator` | Release snapshots, compact, tune prefix/bounds. |
| Range scan slow | High read IO/cache churn | Many files, small blocks, no readahead, tombstones | `iter_read_bytes`, block read metrics | `BlockBasedTableIterator`, `MergingIterator` | Use bounds/prefix, readahead, compact hot ranges. |
| Recovery slow | Slow `DB::Open` | Large MANIFEST, many WALs, many files, corruption retry | `FILE_READ_DB_OPEN_MICROS`, LOG recovery lines | `DBImpl::Recover`, `VersionSet::Recover` | Reduce WAL retention, compact/flush, inspect manifest. |
| Corruption | Open/read failure | Torn/lost writes, bad FS, missing files, checksum mismatch | status/log messages, checksum errors | `log::Reader`, `VersionEdit::DecodeFrom` | Preserve files, inspect with `ldb`/`sst_dump`, choose recovery mode carefully. |
| Too many open files | Open/read errors | `max_open_files=-1` with many files or OS limit too low | OS limits, LOG file open errors | `TableCache`, table reader open | Increase ulimit or use bounded table cache. |
| Background threads low | Flush/compaction lag | `max_background_jobs` too small | job queue symptoms, compaction/flush lag | Env scheduling, `MaybeScheduleFlushOrCompaction` | Increase jobs and storage bandwidth; separate priorities. |
| Rate limiter impact | IO throughput capped | Low limiter rate or wrong mode/priority | limiter config, IO latency | `RateLimiter`, file readers/writers | Check `rate_limiter` mode and foreground priorities. |

## 5. Recovery Validation Tests

Run targeted tests before broad test suites:

- `db/corruption_test`
- `db/fault_injection_test`
- `db/repair_test`
- `db/db_wal_test`
- `db/version_set_test`
- `utilities/transactions/write_prepared_transaction_seqno_test`

Use `db_stress` for mixed recovery, snapshot, transaction, and compaction
coverage after metadata or recovery changes.
