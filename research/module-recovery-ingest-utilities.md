# Module Research: Recovery, IngestExternalFile, Checkpoint, Backup, Transactions, Env

## Module Responsibilities

This research file covers cross-cutting modules that sit around the storage
engine core: DB open/recovery, external SST ingestion, checkpoint/backup,
transactions, Env/FileSystem/thread pools, rate limiting, and tooling.

## Core Source

| Module | Path | Classes / methods |
|---|---|---|
| Recovery | `db/db_impl/db_impl_open.cc`, `db/version_set.cc` | `DBImpl::Recover`, `DBImpl::RecoverLogFiles`, `DBImpl::ProcessLogFile`, `VersionSet::Recover`, `VersionSet::TryRecover` |
| WAL reader | `db/log_reader.cc`, `db/log_format.h` | `log::Reader::ReadRecord`, WAL physical record format |
| External SST ingestion | `db/external_sst_file_ingestion_job.cc`, `include/rocksdb/sst_file_writer.h`, `table/sst_file_writer.cc` | `ExternalSstFileIngestionJob::Prepare`, `Run`, `AssignLevelsForOneBatch`, `AssignGlobalSeqnoForIngestedFile`, `SstFileWriter` |
| Checkpoint | `utilities/checkpoint/checkpoint_impl.cc`, `include/rocksdb/utilities/checkpoint.h` | `Checkpoint::Create`, `CheckpointImpl::CreateCheckpoint`, `CreateCustomCheckpoint` |
| Backup | `utilities/backup/backup_engine.cc`, `include/rocksdb/utilities/backup_engine.h` | `BackupEngine::Open`, `BackupEngineImpl::CreateNewBackupWithMetadata`, `RestoreDBFromBackup`, `GarbageCollect` |
| Transactions | `utilities/transactions/*`, `include/rocksdb/utilities/transaction*.h` | `TransactionDB`, `PessimisticTransactionDB`, `OptimisticTransactionDBImpl`, `WritePreparedTxnDB`, `WriteUnpreparedTxnDB` |
| Env / FileSystem | `include/rocksdb/env.h`, `include/rocksdb/file_system.h`, `env/env_posix.cc`, `file/*` | `Env::Schedule`, `Env::StartThread`, `FileSystem`, `FSRandomAccessFile`, `WritableFileWriter` |
| Rate limiter | `include/rocksdb/rate_limiter.h`, `util/rate_limiter.cc` | `RateLimiter::Request`, `RequestToken`, `NewGenericRateLimiter` |
| Tools | `tools/db_bench.cc`, `tools/ldb.cc`, `tools/sst_dump.cc`, `db_stress_tool/*` | Benchmarking, inspection, and stress testing entry points. |

## Data Structures

| Structure | Role |
|---|---|
| Recovery context | Tracks whether DB is new, recovered sequence, WAL replay status, and retry state. |
| `VersionEdit` during recovery | Recreates file metadata and CF state from MANIFEST, then WAL replay may flush recovered memtables. |
| Ingested file info | External file metadata, level, global sequence number, checksum, and copied/moved target path. |
| Checkpoint file set | Live SSTs, MANIFEST/CURRENT/options, and needed WALs; created through hard links where supported and copies otherwise. |
| Backup metadata | Backup ID, private/shared files, checksums, timestamps, app metadata, and shared-file naming scheme. |
| Transaction commit map | WritePrepared/WriteUnprepared visibility state mapping prepare and commit sequence numbers. |
| Env thread pools | Priority pools used by flush, compaction, high-priority work, bottom-priority work, and user scheduling. |

## Key Flows

### Recovery

`DB::Open`
-> `DBImpl::Recover`
-> lock DB directory
-> read CURRENT / MANIFEST through `VersionSet::Recover`
-> list WALs
-> `DBImpl::RecoverLogFiles`
-> `DBImpl::ProcessLogFile`
-> `log::Reader::ReadRecord`
-> `WriteBatchInternal::InsertInto`
-> recovered memtables
-> final flush or active WAL restore
-> publish sequence number and SuperVersions.

### External SST Ingestion

`DBImpl::IngestExternalFile`
-> `ExternalSstFileIngestionJob::Prepare`
-> validate table properties and key ranges
-> flush overlapping memtables if needed
-> `ExternalSstFileIngestionJob::Run`
-> assign levels and global seqnos
-> copy/move/link files into DB
-> `VersionEdit` adds files
-> `VersionSet::LogAndApply`
-> listener `OnExternalFileIngested`.

### Checkpoint / Backup

`Checkpoint::Create`
-> `CheckpointImpl::CreateCheckpoint`
-> disable file deletions
-> `CreateCustomCheckpoint`
-> hard link SSTs where possible
-> copy live WAL tail when needed
-> create CURRENT/MANIFEST/options files
-> rename temp checkpoint directory.

`BackupEngineImpl::CreateNewBackupWithMetadata`
-> disable file deletions
-> use `CheckpointImpl::CreateCustomCheckpoint`
-> copy or share files into backup private/shared dirs
-> write backup metadata
-> garbage collect partial/obsolete files.

## Concurrency Model

| Area | Behavior |
|---|---|
| Recovery | DB open is single-owner, under DB mutex for metadata. WAL replay inserts into memtables without foreground writers. |
| Ingestion | Foreground call coordinates with flush/compaction overlap checks and can block writes while installing files. |
| Checkpoint/backup | Disable file deletions to keep live files stable while linking/copying. |
| Transactions | Transaction DB wrappers use locks, write callbacks, and read callbacks to enforce conflict and visibility rules above DBImpl. |
| Env | `Env::Schedule` posts background work to priority thread pools; FileSystem APIs abstract POSIX/Windows/object-store details. |
| Rate limiter | IO paths request tokens by priority and operation type; incorrect priority can throttle foreground reads/writes. |

## Configuration

| Option | Effect |
|---|---|
| `wal_recovery_mode` | Corruption tolerance policy during WAL replay. |
| `paranoid_checks` | Stricter corruption/missing-file behavior. |
| `best_efforts_recovery` | Allows recovery by scanning manifests when CURRENT is bad/missing. |
| `create_if_missing`, `error_if_exists` | DB open creation behavior. |
| `IngestExternalFileOptions::move_files`, `snapshot_consistency`, `ingest_behind`, `fail_if_not_bottommost_level` | Ingestion placement and safety. |
| `BackupEngineOptions::share_table_files`, `backup_log_files`, `backup_rate_limiter`, `restore_rate_limiter` | Backup storage efficiency and IO throttling. |
| `TransactionDBOptions::write_policy` | WriteCommitted, WritePrepared, or WriteUnprepared visibility strategy. |
| `rate_limiter` and IO priorities | Global IO throttling behavior. |

## Metrics, Logs, and Diagnostics

| Signal | Use |
|---|---|
| Recovery LOG lines in `DBImpl::Recover` and `VersionSet::Recover` | Identify manifest, WALs, last sequence, missing files, corruption retry. |
| `WAL_FILE_BYTES`, `WAL_FILE_SYNCED`, WAL archive counts | WAL pressure and sync behavior. |
| `PerfContext::ingest_external_file_*` fields | Ingest latency and write blocking. |
| Backup statistics in `BackupEngine` | Backup/restore progress and corruption. |
| File IO histograms by activity | Distinguish DB open, flush, compaction, and user-read IO. |
| `EventListener::OnBackgroundError` | External handling of corruption/IO background failures. |

## Tests

| Test | Coverage |
|---|---|
| `db/corruption_test.cc`, `db/fault_injection_test.cc`, `db/repair_test.cc` | Recovery and corruption behavior. |
| `db/db_wal_test.cc`, `db/wal_manager_test.cc`, `db/log_test.cc` | WAL lifecycle and log reader/writer. |
| `db/external_sst_file_test.cc`, `db/external_sst_file_basic_test.cc` | External SST ingestion. |
| `utilities/checkpoint/checkpoint_test.cc` | Checkpoint consistency and WAL handling. |
| `utilities/backup/backup_engine_test.cc` | Incremental backup, restore, corruption, shared files. |
| `utilities/transactions/*_test.cc` | Transaction conflict and visibility. |
| `env/env_test.cc`, `file/random_access_file_reader_test.cc`, `db/db_rate_limiter_test.cc` | Env/FileSystem/thread/rate-limiter behavior. |

## Operational and Development Concerns

- Recovery slowness usually comes from large MANIFEST, many WALs retained by
  unflushed memtables/CFs, table opening cost, or corruption retry.
- Checkpoint and backup are consistency-sensitive around live WAL copying and
  file deletion disabling. Treat filesystem link/copy/fsync failures as
  first-class failure cases.
- External ingestion bypasses normal write path. It must validate table
  properties, sequence numbers, range overlap, and manifest atomicity.
- Transaction wrappers deliberately override read/write visibility. DBImpl
  changes that publish sequence numbers or reorder writes need transaction tests.
- Env/FileSystem abstractions are portability boundaries. Avoid direct POSIX
  calls in core logic unless guarded and justified.
