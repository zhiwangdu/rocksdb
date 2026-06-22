# Flow Research: Manifest and VersionSet Path

## Scope

This flow covers `VersionEdit`, `VersionSet`, `Version`, `FileMetaData`,
MANIFEST writes, CURRENT, `LogAndApply`, Version lifecycle, SuperVersion switch,
and crash consistency.

## Source Evidence

| Step | Source |
|---|---|
| Edit format | `db/version_edit.h`, `db/version_edit.cc` |
| VersionSet | `db/version_set.h`, `db/version_set.cc` |
| Builder | `db/version_builder.h`, `db/version_builder.cc` |
| Current/manifest names | `db/filename.h`, `db/filename.cc` |
| SuperVersion | `db/column_family.h`, `db/column_family.cc` |

## Call Chain

metadata-producing job creates `VersionEdit`
-> `VersionSet::LogAndApply`
-> manifest writer queue
-> `VersionSet::LogAndApplyHelper`
-> `VersionBuilder::Apply`
-> `VersionEdit::EncodeTo`
-> `log::Writer::AddRecord`
-> manifest file flush/sync
-> optional CURRENT update when new descriptor log is created
-> append new `Version`
-> `ColumnFamilyData::InstallSuperVersion`.

## Crash Consistency

Flush/compaction output files can exist before they are live. They become live
only after the MANIFEST edit is durable. Conversely, old files remain live until
the edit deleting them is durable. On restart, `VersionSet::Recover` replays
only durable edits and rebuilds a consistent current version.

## Important Invariants

- `next_file_number` must not reuse numbers already present in MANIFEST or on
  disk.
- `last_sequence` must be monotonic and consistent with WAL replay.
- L1+ file metadata must preserve non-overlap assumptions for binary search.
- `FileMetaData::being_compacted` protects picker decisions while compaction is
  in progress.
- SuperVersion installation must happen after metadata commit, not before.

## Diagnostics

Manifest problems show up as slow open, missing-file errors, or corruption
during `VersionSet::Recover`. Check LOG lines for manifest path, file number,
next file number, last sequence, log number, and max column family.
