# Flow Research: Checkpoint and Backup Path

## Scope

This flow covers checkpoint creation, hard links/copies, BackupEngine,
incremental backup, restore, and consistency guarantees.

## Source Evidence

| Step | Source |
|---|---|
| Checkpoint API | `include/rocksdb/utilities/checkpoint.h` |
| Checkpoint implementation | `utilities/checkpoint/checkpoint_impl.cc` |
| Live files | `db/db_filesnapshot.cc` |
| Backup API | `include/rocksdb/utilities/backup_engine.h` |
| Backup implementation | `utilities/backup/backup_engine.cc` |
| Tests | `utilities/checkpoint/checkpoint_test.cc`, `utilities/backup/backup_engine_test.cc` |

## Checkpoint Call Chain

`Checkpoint::Create`
-> `CheckpointImpl::CreateCheckpoint`
-> create temp directory
-> `DB::DisableFileDeletions`
-> `CheckpointImpl::CreateCustomCheckpoint`
-> gather live files and needed WALs
-> hard link SSTs when supported
-> copy WAL tail / unsupported links
-> create CURRENT/options/metadata files
-> enable file deletions
-> rename temp directory to final checkpoint.

## Backup Call Chain

`BackupEngine::Open`
-> `BackupEngineImpl::Initialize`
-> `BackupEngineImpl::CreateNewBackupWithMetadata`
-> disable file deletions
-> use `CheckpointImpl::CreateCustomCheckpoint`
-> copy/share files into backup directories
-> write backup metadata
-> garbage collect incomplete or obsolete files.

## Consistency Guarantees

The checkpoint/backup code stabilizes the live file set by disabling file
deletions. WAL handling is required for data not yet flushed. Atomic flush plus
`backup_log_files=false` is a special configuration covered by checkpoint tests.

## Operational Signals

Use BackupEngine logs, backup statistics, IOStatsContext bytes read/written,
rate limiter settings (`backup_rate_limiter`, `restore_rate_limiter`), and
restore verification tests.
