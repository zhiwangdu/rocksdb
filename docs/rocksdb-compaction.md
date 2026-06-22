# RocksDB Compaction

Compaction is the background mechanism that controls LSM shape, read
amplification, write amplification, space amplification, tombstone cleanup, and
blob garbage. This document focuses on code entry points and operational
diagnostics.

## 1. Source Map

| Area | Source evidence |
|---|---|
| Scheduling | `db/db_impl/db_impl_compaction_flush.cc` |
| Common picker | `db/compaction/compaction_picker.h`, `db/compaction/compaction_picker.cc` |
| Level picker | `db/compaction/compaction_picker_level.cc` |
| Universal picker | `db/compaction/compaction_picker_universal.cc` |
| FIFO picker | `db/compaction/compaction_picker_fifo.cc` |
| Plan object | `db/compaction/compaction.h`, `db/compaction/compaction.cc` |
| Job execution | `db/compaction/compaction_job.h`, `db/compaction/compaction_job.cc` |
| Key transformation | `db/compaction/compaction_iterator.h`, `db/compaction/compaction_iterator.cc` |
| User filter | `include/rocksdb/compaction_filter.h` |
| Tests | `db/compaction/compaction_job_test.cc`, `db/compaction/compaction_picker_test.cc`, `db/db_compaction_test.cc` |

## 2. Triggering

Compaction is scheduled by `DBImpl::MaybeScheduleFlushOrCompaction`. Pickers are
driven by `VersionStorageInfo::ComputeCompactionScore` and by explicit file
markers. `LevelCompactionPicker::NeedsCompaction` checks expired TTL files,
periodic files, bottommost/forced/blob-GC markers, read-triggered files, and
per-level compaction scores. Universal and FIFO have their own
`NeedsCompaction` implementations.

Operational triggers:

- L0 file count reaches `level0_file_num_compaction_trigger`;
- pending compaction bytes exceed soft/hard limits;
- periodic/TTL compaction marks files;
- read-triggered compaction marks files;
- manual `CompactRange` or `CompactFiles`;
- FIFO size/TTL cleanup.

## 3. Execution

Call chain:

`BackgroundCompaction`
-> picker returns `Compaction`
-> `CompactionJob::Prepare`
-> `CompactionJob::Run`
-> `CompactionJob::RunSubcompactions`
-> `CompactionJob::ProcessKeyValueCompaction`
-> `CreateInputIterator`
-> `CompactionIterator`
-> output SST creation
-> `CompactionJob::Install`
-> `InstallCompactionResults`
-> `VersionSet::LogAndApply`.

`CompactionJob::Run` collects subcompaction status, cleans aborted outputs,
syncs output directories, verifies output files, aggregates job stats, and only
then returns status for install.

## 4. CompactionIterator Rules

`CompactionIterator` is where most correctness risk lives. It must:

- keep keys visible to active snapshots;
- drop older versions hidden by newer versions;
- drop point tombstones only when safe;
- handle range tombstones and range deletion blocks;
- resolve merge operands when safe;
- invoke `CompactionFilter` only in allowed cases;
- handle blob references and garbage collection;
- preserve internal key order for table builder output.

Relevant metrics are `COMPACTION_KEY_DROP_NEWER_ENTRY`,
`COMPACTION_KEY_DROP_OBSOLETE`, `COMPACTION_KEY_DROP_RANGE_DEL`,
`COMPACTION_KEY_DROP_USER`, and `COMPACTION_RANGE_DEL_DROP_OBSOLETE`.

## 5. Strategy Tradeoffs

| Strategy | Strength | Risk |
|---|---|---|
| Leveled | Predictable reads and space with non-overlapping L1+. | Higher write amplification when data churn is high. |
| Universal | Lower write amp for write-heavy workloads. | Higher temporary space amp and more sorted runs to search. |
| FIFO | Cheap time/size-window retention. | Deletes by policy, not by key-level merge semantics; limited use cases. |

## 6. Trivial Move and Subcompaction

Trivial move can commit metadata changes without rewriting files when key-range
and level constraints allow. Subcompaction splits a large compaction into
multiple key ranges. Both still commit through `VersionSet::LogAndApply`, so
MANIFEST remains the visibility boundary.

## 7. Operational Diagnostics

| Symptom | Likely cause | Signals | Source entry |
|---|---|---|---|
| L0 files grow | Compaction not scheduled or too slow | L0 file count, `level0_*` stalls, LOG compaction messages | `MaybeScheduleFlushOrCompaction`, level picker |
| Write stalls | Pending compaction debt or immutable memtables | `STALL_MICROS`, `WRITE_STALL`, pending bytes | write controller and picker score |
| Space amp high | Snapshots/tombstones/compaction backlog | live/obsolete file counts, compaction drops low | `CompactionIterator`, obsolete cleanup |
| Iterator slow | Hidden versions/range tombstones | PerfContext skipped counters | `DBIter`, `MergingIterator`, `CompactionIterator` |
| Compaction CPU high | Compression, filters, merge resolution | `COMPACTION_CPU_TIME`, `COMPACTION_CPU_TOTAL_TIME` | `CompactionJob::ProcessKeyValueCompaction` |

## 8. Development Checklist

- Check picker changes with overlap/in-progress invariants.
- Check `CompactionIterator` changes with snapshots, range deletion, merge,
  blob, and bottommost-level tests.
- Check manifest install failure windows.
- Check `EventListener` callback ordering: `OnCompactionBegin`,
  `OnSubcompactionBegin`, `OnCompactionPreCommit`, `OnCompactionCompleted`.
- Benchmark if changing picker strategy, output sizing, compression, or
  iterator hot loops.
