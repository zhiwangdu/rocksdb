# LSM Research: Read, Write, and Space Amplification

## Read Amplification

Main causes:

- overlapping L0 files;
- too many levels or sorted runs;
- ineffective Bloom/prefix filters;
- block cache misses;
- tombstones/merge operands that force scanning hidden versions;
- range scans crossing many files.

Controls:

- `level0_file_num_compaction_trigger`, L0 slowdown/stop triggers;
- compaction style and compaction score;
- Bloom/Ribbon filters and prefix extractor;
- block cache size and index/filter caching;
- manual compaction for pathological debt.

Source evidence: `Version::Get`, `FilePicker`, `BlockBasedTable::Get`,
`BlockBasedTable::FullFilterKeyMayMatch`, `DBIter`, and
`VersionStorageInfo::ComputeCompactionScore`.

## Write Amplification

Main causes:

- repeated compaction through levels;
- too-small target file/level sizes;
- compaction filters or blob GC rewriting values;
- universal compaction sorted-run rewrite policy;
- high delete/update churn.

Controls:

- `target_file_size_base`, `max_bytes_for_level_base`,
  `max_bytes_for_level_multiplier`;
- `level_compaction_dynamic_level_bytes`;
- compaction style selection;
- `max_compaction_bytes`;
- write buffer size and flush merge behavior.

Signals are `COMPACT_READ_BYTES`, `COMPACT_WRITE_BYTES`, `FLUSH_WRITE_BYTES`,
`COMPACTION_TIME`, `COMPACTION_CPU_TIME`, and compaction LOG output.

## Space Amplification

Main causes:

- live old versions retained by snapshots;
- tombstones waiting for bottommost compaction;
- universal compaction temporary overlap;
- compaction backlog;
- blob files with garbage awaiting GC;
- backups/checkpoints keeping obsolete files alive.

Controls:

- compaction debt thresholds;
- bottommost/manual compaction;
- snapshot lifetime discipline;
- FIFO/universal options;
- blob GC options;
- file deletion re-enable after checkpoint/backup.

Relevant code is `CompactionIterator`, `VersionStorageInfo`,
`CompactionJob::InstallCompactionResults`, and obsolete-file handling in DBImpl.
