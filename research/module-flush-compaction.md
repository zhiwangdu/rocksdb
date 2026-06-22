# Module Research: Flush, Compaction, Pickers, Filters, Amplification Control

## Module Responsibilities

Flush converts immutable memtables into L0 SST files and commits the result to
MANIFEST. Compaction rewrites existing SSTs to enforce LSM shape, reduce read
amplification, reclaim obsolete versions/tombstones, and manage space
amplification. These modules do not assign foreground write sequence numbers or
serve user reads directly.

## Core Source

| Path | Class / method | Role |
|---|---|---|
| `db/db_impl/db_impl_compaction_flush.cc` | `DBImpl::Flush`, `FlushMemTable`, `AtomicFlushMemTables`, `BackgroundFlush`, `BackgroundCompaction`, `MaybeScheduleFlushOrCompaction` | Scheduling, manual APIs, background workers. |
| `db/flush_job.h`, `db/flush_job.cc` | `FlushJob::PickMemTable`, `FlushJob::Run`, `FlushJob::WriteLevel0Table` | Build and install L0 output from immutable memtables. |
| `db/compaction/compaction.h`, `compaction.cc` | `Compaction` | Compaction plan and input/output metadata. |
| `db/compaction/compaction_picker.h`, `compaction_picker.cc` | `CompactionPicker` | Shared picking helpers and in-progress file registration. |
| `db/compaction/compaction_picker_level.cc` | `LevelCompactionPicker::NeedsCompaction`, `LevelCompactionBuilder::PickCompaction` | Leveled compaction picker. |
| `db/compaction/compaction_picker_universal.cc` | `UniversalCompactionPicker::NeedsCompaction`, `PickCompaction` | Universal/tiered sorted-run picker. |
| `db/compaction/compaction_picker_fifo.cc` | `FIFOCompactionPicker::NeedsCompaction`, `PickTTLCompaction` | FIFO size/TTL deletion and optional intra-L0 compaction. |
| `db/compaction/compaction_job.h`, `compaction_job.cc` | `CompactionJob::Run`, `ProcessKeyValueCompaction`, `InstallCompactionResults` | Executes compaction, writes outputs, commits metadata. |
| `db/compaction/compaction_iterator.h`, `compaction_iterator.cc` | `CompactionIterator::Next`, `InvokeFilterIfNeeded`, `PrepareOutput` | Drops obsolete keys, applies compaction filters, resolves merges. |
| `include/rocksdb/compaction_filter.h` | `CompactionFilter`, `CompactionFilterFactory` | User-defined key rewrite/drop during compaction. |

## Data Structures

| Structure | Role |
|---|---|
| Flush request | Queue entry in DBImpl containing CFs and max memtable IDs. Atomic flush may include several CFs. |
| `FlushJob` | Holds memtables, output file metadata, table builder options, job context, log buffer, and manifest write flag. |
| `Compaction` | Input levels/files, output level, reason, snapshots, deletion compaction flags, output file size policies. |
| `SubcompactionState` | Split compaction range, input iterator, outputs, job stats, status, blob metadata. |
| `CompactionIterator` | Internal-key stream transformer that enforces snapshot, tombstone, merge, compaction-filter, and blob-GC rules. |
| `VersionEdit` | Adds new output files and deletes input files during manifest commit. |

## Key Flows

### Flush

Write path marks memtable flush needed
-> `DBImpl::SwitchMemtable`
-> immutable memtable list
-> `DBImpl::MaybeScheduleFlushOrCompaction`
-> `DBImpl::BackgroundFlush`
-> `FlushJob::PickMemTable`
-> `FlushJob::Run`
-> `FlushJob::WriteLevel0Table`
-> `BuildTable` / `TableBuilder`
-> `MemTableList::TryInstallMemtableFlushResults`
-> `VersionSet::LogAndApply`
-> new SuperVersion.

### Compaction

`MaybeScheduleFlushOrCompaction`
-> `BackgroundCompaction`
-> picker (`LevelCompactionPicker`, `UniversalCompactionPicker`, or `FIFOCompactionPicker`)
-> `CompactionJob::Prepare`
-> `CompactionJob::Run`
-> `RunSubcompactions`
-> `CompactionJob::ProcessKeyValueCompaction`
-> `CreateInputIterator`
-> `CompactionIterator`
-> output table builders
-> `CompactionJob::Install`
-> `CompactionJob::InstallCompactionResults`
-> `VersionSet::LogAndApply`
-> new SuperVersion.

## Concurrency Model

| Area | Behavior |
|---|---|
| Scheduling | DB mutex protects queues and in-progress compaction state. |
| File IO | Flush/compaction release DB mutex while reading inputs and writing outputs. |
| Manifest install | Reacquires DB mutex and serializes through `VersionSet::LogAndApply`. |
| Subcompaction | One compaction can split into multiple ranges; each subcompaction writes its own outputs and status. |
| Backpressure | WriteController reacts to immutable memtables, L0 file counts, and pending compaction bytes. |
| Listeners | Flush/compaction begin/completed/precommit callbacks must be cheap and may block background progress. |

## Configuration

| Option | Default | Effect |
|---|---:|---|
| `max_background_jobs` | `2` | Worker capacity for background flush/compaction. |
| `level0_file_num_compaction_trigger` | `4` | L0 compaction pressure starts. |
| `level0_slowdown_writes_trigger` | `20` | Foreground slowdown. |
| `level0_stop_writes_trigger` | `36` | Foreground stop. |
| `soft_pending_compaction_bytes_limit` | `64GB` | Slow writes due to compaction debt. |
| `hard_pending_compaction_bytes_limit` | `256GB` | Stop writes due to compaction debt. |
| `target_file_size_base` | `64MB` | Output file target. |
| `max_bytes_for_level_base` | `256MB` | L1 target bytes. |
| `compaction_style` | Level | Level, universal, FIFO, or none. |
| `compaction_options_universal` | default struct | Sorted-run picking and size ratio policy. |
| `compaction_options_fifo` | `max_table_files_size=1GB` | FIFO deletion/intra-L0 policy. |
| `max_subcompactions` | option-dependent | Parallelizes large compactions. |
| `report_bg_io_stats` | `false` | Enables background IO stats collection. |

## Metrics, PerfContext, Logs

| Signal | Use |
|---|---|
| `COMPACTION_TIME`, `COMPACTION_CPU_TIME`, `SUBCOMPACTION_SETUP_TIME` | Compaction runtime and CPU pressure. |
| `COMPACT_READ_BYTES`, `COMPACT_WRITE_BYTES`, `FLUSH_WRITE_BYTES` | Write amplification and IO volume. |
| `COMPACTION_KEY_DROP_*`, `COMPACTION_RANGE_DEL_DROP_OBSOLETE` | Obsolete version/tombstone cleanup. |
| `FLUSH_TIME`, `FILE_WRITE_FLUSH_MICROS`, `FILE_WRITE_COMPACTION_MICROS` | Flush/compaction write latency. |
| `EventListener::OnCompactionPreCommit` | Observes compaction output before manifest commit. |
| LOG lines from `FlushJob::WriteLevel0Table` and `CompactionJob::Install` | Input/output files, bytes, job IDs, status. |

## Tests

| Test | Coverage |
|---|---|
| `db/db_flush_test.cc`, `db/flush_job_test.cc` | Flush triggers, manual flush, atomic flush, errors. |
| `db/db_compaction_test.cc`, `db/manual_compaction_test.cc` | General compaction behavior and manual ranges. |
| `db/compaction/compaction_job_test.cc` | Job execution and output verification. |
| `db/compaction/compaction_picker_test.cc` | Picker decisions and overlap constraints. |
| `db/db_universal_compaction_test.cc`, `db/compaction/tiered_compaction_test.cc` | Universal/tiered behavior. |
| `db/db_compaction_filter_test.cc` | User compaction filter behavior. |

## Operational and Development Concerns

- If flush cannot keep up, immutable memtables accumulate, WAL files are
  retained, and foreground writes eventually stall.
- If compaction cannot keep up, L0 grows, read amplification rises, space
  amplification rises, and pending compaction byte limits throttle writes.
- Trivial moves avoid rewriting when possible, but code must preserve manifest
  atomicity and file-level overlap invariants.
- Compaction iterator changes are high risk because they affect snapshots,
  tombstones, merges, blob GC, and user compaction filters in one hot path.
- Validate with compaction picker tests, compaction iterator tests, DB stress,
  and targeted recovery tests for output-file/manifest failure windows.
