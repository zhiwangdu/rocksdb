# Recovery and Operations Research

## Slow Recovery

Symptoms: long `DB::Open`, long MANIFEST replay, many WALs replayed, high file
open/read IO.

Likely causes:

- large MANIFEST from frequent metadata edits;
- many archived/live WALs retained by unflushed column families;
- large memtables replayed from WAL;
- many SST table readers opened with `max_open_files=-1`;
- corruption retry or best-efforts recovery scan.

Evidence paths: `DBImpl::Recover`, `VersionSet::Recover`,
`DBImpl::RecoverLogFiles`, `DBImpl::ProcessLogFile`,
`VersionEdit::DecodeFrom`.

Diagnostics: LOG manifest lines, recovery event log, WAL file list,
`FILE_READ_DB_OPEN_MICROS`, IOStats read/open counters, `ldb manifest_dump`.

## Corruption

Symptoms: open failure, checksum mismatch, WAL reader corruption, missing SST.

Likely causes: torn writes, lost fsync, storage errors, external file deletion,
incorrect FileSystem implementation, or application using no-WAL writes without
external durability.

Relevant options: `paranoid_checks`, `wal_recovery_mode`,
`best_efforts_recovery`, checksum options, direct IO options.

Validation: `db/corruption_test.cc`, `db/fault_injection_test.cc`,
`db/repair_test.cc`, and table checksum tests.

## Write Slowdown / Stall

Symptoms: high write latency, `STALL_MICROS`, logs mentioning delayed/stopped
writes.

Likely causes: L0 file buildup, immutable memtable backlog, pending compaction
bytes, low `max_background_jobs`, slow WAL sync, rate limiter too tight.

Evidence paths: `DBImpl::PreprocessWrite`, `WriteController`, `FlushJob`,
`CompactionJob`, `MaybeScheduleFlushOrCompaction`.

## Read / Iterator Slowdown

Symptoms: high `DB_GET`, `DB_SEEK`, low block cache hit rate, high skipped-key
PerfContext counters, excessive L0 reads.

Likely causes: poor cache sizing, filters disabled/ineffective, prefix mismatch,
long snapshots retaining tombstones, compaction backlog, range scans over many
files.

Evidence paths: `Version::Get`, `BlockBasedTable::Get`, `DBIter`,
`MergingIterator`, `RangeDelAggregator`.
