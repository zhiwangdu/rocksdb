# Flow Research: Flush Path

## Scope

This flow covers flush triggers, mutable memtable switch, immutable memtables,
`FlushJob`, `TableBuilder`, SST creation, MANIFEST/VersionSet updates, and
flush failure handling.

## Source Evidence

| Step | Source |
|---|---|
| Scheduling | `db/db_impl/db_impl_compaction_flush.cc` |
| Flush job | `db/flush_job.cc`, `db/flush_job.h` |
| Memtable list | `db/memtable_list.cc`, `db/memtable_list.h` |
| Table build | `db/builder.cc`, `table/block_based/block_based_table_builder.cc` |
| Manifest commit | `db/version_set.cc`, `db/version_edit.cc` |

## Call Chain

write path detects full memtable
-> `DBImpl::SwitchMemtable`
-> mutable memtable becomes immutable
-> `DBImpl::MaybeScheduleFlushOrCompaction`
-> Env background thread
-> `DBImpl::BackgroundFlush`
-> `FlushJob::PickMemTable`
-> `FlushJob::Run`
-> optional `FlushJob::MemPurge`
-> `FlushJob::WriteLevel0Table`
-> `NewMergingIterator` over picked memtables
-> `BuildTable` / `TableBuilder::Add`
-> `BlockBasedTableBuilder::Finish`
-> `MemTableList::TryInstallMemtableFlushResults`
-> `VersionSet::LogAndApply`
-> install SuperVersion and schedule more work.

## Trigger Conditions

- Mutable memtable reaches `write_buffer_size`.
- Too many WAL bytes retained across CFs (`max_total_wal_size`) force a flush.
- Manual `DB::Flush`.
- Atomic flush request across CFs.
- Error recovery can schedule recovery flushes.
- WriteBufferManager / DB-wide write buffer pressure.

## Failure Handling

`FlushJob::Run` rolls back memtable flush state on failure through
`MemTableList::RollbackMemtableFlush`. If an output file was created but not
committed to MANIFEST, cleanup is driven by job context/obsolete file handling.
Background errors can stop non-recovery flushes and propagate through the
error handler.

## Metrics and Logs

Use `FLUSH_TIME`, `FLUSH_WRITE_BYTES`, `FILE_WRITE_FLUSH_MICROS`,
`MEMTABLE_PAYLOAD_BYTES_AT_FLUSH`, `MEMTABLE_GARBAGE_BYTES_AT_FLUSH`,
`IOStatsContext::write_nanos`, `fsync_nanos`, listener events
`OnFlushBegin`/`OnFlushCompleted`, and LOG/event-log entries from
`FlushJob::WriteLevel0Table`.

## Tests

Use `db/db_flush_test.cc`, `db/flush_job_test.cc`,
`db/memtable_list_test.cc`, `db/db_wal_test.cc`, and checkpoint/backup tests
for live-file and WAL consistency interactions.
