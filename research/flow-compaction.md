# Flow Research: Compaction Path

## Scope

This flow covers compaction triggers, pickers, `CompactionJob`,
`CompactionIterator`, SST inputs/outputs, tombstone cleanup, range deletion,
trivial move, subcompaction, compaction filters, errors, and differences among
level, universal, and FIFO compaction.

## Source Evidence

| Step | Source |
|---|---|
| Scheduling | `db/db_impl/db_impl_compaction_flush.cc` |
| Shared picker logic | `db/compaction/compaction_picker.cc` |
| Leveled picker | `db/compaction/compaction_picker_level.cc` |
| Universal picker | `db/compaction/compaction_picker_universal.cc` |
| FIFO picker | `db/compaction/compaction_picker_fifo.cc` |
| Job execution | `db/compaction/compaction_job.cc` |
| Key processing | `db/compaction/compaction_iterator.cc` |
| Filters | `include/rocksdb/compaction_filter.h` |

## Call Chain

flush/manifest install updates compaction score
-> `DBImpl::MaybeScheduleFlushOrCompaction`
-> `DBImpl::BackgroundCompaction`
-> compaction picker
-> `CompactionJob::Prepare`
-> `CompactionJob::Run`
-> `CompactionJob::RunSubcompactions`
-> `CompactionJob::ProcessKeyValueCompaction`
-> input iterator over compacted files
-> `CompactionIterator::SeekToFirst` / `Next`
-> output table builder(s)
-> `CompactionJob::FinishCompactionOutputFile`
-> `CompactionJob::Install`
-> `CompactionJob::InstallCompactionResults`
-> `VersionEdit` deletes inputs and adds outputs
-> `VersionSet::LogAndApply`.

## Trigger Conditions

- `VersionStorageInfo::CompactionScore(level) >= 1`.
- Explicit files/ranges marked for compaction.
- Periodic/TTL compaction.
- Read-triggered compaction files.
- Bottommost or forced blob-GC files.
- Manual compaction.
- FIFO size/TTL thresholds.

## Strategy Differences

| Style | Picker evidence | Behavior |
|---|---|---|
| Level | `LevelCompactionPicker::NeedsCompaction`, `LevelCompactionBuilder` | Maintains non-overlap in L1+ and target bytes per level; controls read amp by moving data down levels. |
| Universal | `UniversalCompactionPicker::NeedsCompaction`, `UniversalCompactionBuilder::CalculateSortedRuns` | Treats L0 and levels as sorted runs; compacts runs by size/age/ratio. |
| FIFO | `FIFOCompactionPicker::NeedsCompaction`, `PickTTLCompaction` | Deletes old/excess files and optionally compacts L0 depending on FIFO options. |

## Tombstone and Range Deletion Cleanup

`CompactionIterator` can drop obsolete point versions, point tombstones, and
range tombstones when no snapshot or lower-level data requires them. Counters
`COMPACTION_KEY_DROP_*` and `COMPACTION_RANGE_DEL_DROP_OBSOLETE` explain why
data disappeared during compaction.

## Failure Handling

`CompactionJob::Run` collects subcompaction errors, cleans aborted outputs when
not resumable, syncs output directories, verifies outputs, and only then
installs results. Failure before `VersionSet::LogAndApply` means output files
are not live and are cleaned as obsolete.

## Metrics and Tests

Use `COMPACTION_TIME`, `COMPACTION_CPU_TIME`, `COMPACT_READ_BYTES`,
`COMPACT_WRITE_BYTES`, `NUM_FILES_IN_SINGLE_COMPACTION`,
`NUM_SUBCOMPACTIONS_SCHEDULED`, `COMPACTION_KEY_DROP_*`, and
`FILE_WRITE_COMPACTION_MICROS`. Tests include `db/db_compaction_test.cc`,
`db/compaction/compaction_job_test.cc`, `db/compaction/compaction_picker_test.cc`,
`db/db_universal_compaction_test.cc`, `db/db_compaction_filter_test.cc`, and
`db/compaction/compaction_iterator_test.cc`.
