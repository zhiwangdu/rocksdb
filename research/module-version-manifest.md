# Module Research: VersionSet, Version, VersionEdit, Manifest, FileMetaData

## Module Responsibilities

The version subsystem is the metadata authority for the LSM. It records which
SST and blob files are live for each column family, persists those changes in
MANIFEST logs, reads CURRENT/MANIFEST during open, computes compaction scores,
and publishes new `Version` objects for readers. It does not read or write user
data blocks directly; table cache and table readers do that.

## Core Source

| Path | Class / method | Role |
|---|---|---|
| `db/version_set.h`, `db/version_set.cc` | `VersionSet`, `Version`, `VersionStorageInfo` | Metadata ownership, current versions, compaction scores, file search. |
| `db/version_edit.h`, `db/version_edit.cc` | `VersionEdit`, `FileMetaData` | Durable edit record format for MANIFEST. |
| `db/version_builder.h`, `db/version_builder.cc` | `VersionBuilder::Apply`, `VersionBuilder::SaveTo` | Builds a new version by applying edits to prior state. |
| `db/filename.h`, `db/filename.cc` | `CurrentFileName`, `DescriptorFileName`, `ParseFileName` | File naming for CURRENT, MANIFEST, WAL, SST. |
| `db/table_cache.h`, `db/table_cache.cc` | `TableCache` | Opens table readers referenced by versions. |

## Data Structures

| Structure | Meaning |
|---|---|
| `VersionSet` | DB-wide metadata manager containing column family set, next file number, last sequence, manifest writer queue, manifest file number and size. |
| `Version` | Immutable reader-visible view of one column family's files and blob metadata. Refcounted by SuperVersion, iterators, compactions, and jobs. |
| `VersionStorageInfo` | Per-level file arrays, compact cursors, compaction scores, file indexer, L0 ordering, and base-level sizing. |
| `VersionEdit` | Delta log record: comparator, log numbers, next file, last sequence, CF add/drop, deleted files, new files, blob additions/garbage, WAL additions/deletions. |
| `FileMetaData` | SST metadata: file number, size, smallest/largest internal keys, sequence bounds, table properties, compaction flags, temperature, checksums. |
| MANIFEST | Physical log file of `VersionEdit` records written by `VersionSet::LogAndApply` and read by `VersionSet::Recover`. |
| CURRENT | Small file pointing to the active MANIFEST. |

## Key Flows

### Manifest Commit

Flush/compaction/ingest creates `VersionEdit`
-> caller holds DB mutex
-> `VersionSet::LogAndApply`
-> `VersionSet::LogAndApplyHelper`
-> `VersionBuilder::Apply`
-> `VersionSet::ProcessManifestWrites`
-> `VersionEdit::EncodeTo`
-> `log::Writer::AddRecord`
-> manifest file sync
-> append new `Version`
-> install SuperVersion.

### Manifest Recovery

`DBImpl::Recover`
-> `VersionSet::Recover`
-> read CURRENT with `GetCurrentManifestPath`
-> create manifest `log::Reader`
-> `VersionEditHandler::Iterate`
-> `VersionEdit::DecodeFrom`
-> `VersionBuilder`
-> current `Version` per column family
-> last sequence and file numbers restored.

## Concurrency Model

| Area | Behavior |
|---|---|
| Manifest writers | `VersionSet::LogAndApply` queues `ManifestWriter` objects and serializes manifest mutation under DB mutex. |
| Readers | Read a `Version` through `SuperVersion`; old versions stay alive by refcount. |
| Compactions | Mark input files as `being_compacted`; manifest commit deletes inputs and adds outputs atomically. |
| Flush | Adds an L0 file and advances min log number in the same metadata transaction. |
| Recovery | Single-threaded under DB open mutex; may use table opening threads depending on `open_files_async`. |

## Configuration

| Option | Source | Effect |
|---|---|---|
| `max_manifest_file_size` | `include/rocksdb/options.h`, `VersionSet::TuneMaxManifestFileSize` | Controls manifest rollover. |
| `reuse_manifest_on_open` | `include/rocksdb/options.h`, `VersionSet::Recover` | Reopens manifest for append when safe. |
| `paranoid_checks` | `include/rocksdb/options.h` | Makes metadata/file validation stricter. |
| `max_open_files` | `include/rocksdb/options.h` | Determines table reader caching/opening behavior. |
| `level_compaction_dynamic_level_bytes` | `include/rocksdb/advanced_options.h`, `VersionStorageInfo::ComputeCompactionScore` | Changes base-level target sizing. |

## Metrics, Logs, and Diagnostics

| Signal | Source evidence | Use |
|---|---|---|
| `MANIFEST_FILE_SYNC_MICROS` | `include/rocksdb/statistics.h` | Detect manifest sync latency. |
| `VersionSet::Recover` LOG lines | `db/version_set.cc` | Current MANIFEST, next file number, last sequence, log numbers. |
| `VersionStorageInfo::ComputeCompactionScore` | `db/version_set.cc` | Explain compaction priority and base-level behavior. |
| LOG entries during `LogAndApply` | `db/version_set.cc` | Manifest write failures and metadata commit latency. |

## Tests

| Test file | Coverage |
|---|---|
| `db/version_set_test.cc` | Manifest recovery, version construction, compaction scores, file brief generation. |
| `db/version_edit_test.cc` | `VersionEdit` encode/decode compatibility. |
| `db/version_builder_test.cc` | Applying metadata edits. |
| `db/db_dynamic_level_test.cc` | Dynamic-level base bytes and compaction score behavior. |
| `db/db_basic_test.cc` | Manifest write counts and open behavior. |

## Operational and Development Concerns

- MANIFEST is the crash-consistency record for SST membership. Output files
  created by flush/compaction are not visible until the edit is committed.
- A large MANIFEST slows open and recovery; investigate frequent CF churn,
  high flush/compaction rate, or disabled manifest reuse.
- File metadata must use internal-key boundaries, not user-key-only boundaries,
  because sequence and type are part of search and snapshot correctness.
- L0 files can overlap, so version search must preserve L0 ordering. L1+ files
  are expected to be non-overlapping for binary search.
- Any change to `VersionEdit` encoding needs compatibility tests and recovery
  tests against older tags.
