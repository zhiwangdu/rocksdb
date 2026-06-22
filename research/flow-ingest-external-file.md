# Flow Research: IngestExternalFile Path

## Scope

This flow covers external SST import, file validation, level selection,
MANIFEST update, and interaction with compaction/flush.

## Source Evidence

| Step | Source |
|---|---|
| Public API | `include/rocksdb/db.h`, `include/rocksdb/sst_file_writer.h` |
| DB entry | `db/db_impl/db_impl.cc` |
| Ingestion job | `db/external_sst_file_ingestion_job.cc` |
| SST writer | `table/sst_file_writer.cc`, `table/sst_file_reader.cc` |
| Tests | `db/external_sst_file_test.cc`, `db/external_sst_file_basic_test.cc`, `table/table_test.cc` |

## Call Chain

`DB::IngestExternalFile`
-> `DBImpl::IngestExternalFile`
-> `ExternalSstFileIngestionJob::Prepare`
-> load table properties and boundaries
-> `NeedsFlush` for overlapping memtables
-> flush if needed
-> `ExternalSstFileIngestionJob::Run`
-> `AssignLevelsForOneBatch`
-> `AssignGlobalSeqnoForIngestedFile`
-> copy/move/link file into DB directory
-> `VersionEdit::AddFile`
-> `VersionSet::LogAndApply`
-> SuperVersion install
-> `EventListener::OnExternalFileIngested`.

## Key Rules

- If snapshots exist and `snapshot_consistency` requires it, ingestion assigns a
  global sequence number even for non-overlapping files.
- Ingested files must not overlap mutable/immutable memtables unless those
  memtables are flushed before install.
- Atomic replace range requires overlap checks against existing files and
  compactions.
- `ingest_behind` and bottommost-level constraints change allowed placement.

## Metrics and Tests

Use PerfContext ingest fields, listener events, LOG lines, and tests in
`db/external_sst_file_test.cc`, `db/external_sst_file_basic_test.cc`, and
`table/table_test.cc`.
