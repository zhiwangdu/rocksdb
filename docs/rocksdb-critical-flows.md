# RocksDB Critical Flows

This document is a compact call-chain guide for the critical RocksDB flows.
Detailed research notes are in `research/flow-*.md`.

## 1. Write Path

`DB::Put / DB::Write`
-> `DBImpl::Write`
-> `DBImpl::WriteImpl` (`db/db_impl/db_impl_write.cc`)
-> `WriteThread::JoinBatchGroup` (`db/write_thread.cc`)
-> `DBImpl::PreprocessWrite`
-> `WriteThread::EnterAsBatchGroupLeader`
-> `DBImpl::WriteGroupToWAL` / `ConcurrentWriteGroupToWAL`
-> `log::Writer::AddRecord` (`db/log_writer.cc`)
-> sequence assignment through `VersionSet`
-> optional WAL sync
-> `WriteBatchInternal::InsertInto` (`db/write_batch.cc`)
-> `MemTable::Add` (`db/memtable.cc`)
-> `versions_->SetLastSequence`
-> `WriteThread::ExitAsBatchGroupLeader`.

Critical invariants: WAL-before-memtable, no sequence publication before
successful memtable insert, and sequence ranges consistent with WAL replay.

## 2. Get Path

`DB::Get`
-> `DBImpl::Get`
-> `DBImpl::GetImpl` (`db/db_impl/db_impl.cc`)
-> reference `SuperVersion`
-> mutable memtable lookup
-> immutable memtable lookup
-> `Version::Get` (`db/version_set.cc`)
-> `FilePicker`
-> `TableCache::Get`
-> `BlockBasedTable::Get` (`table/block_based/block_based_table_reader.cc`)
-> `FullFilterKeyMayMatch`
-> index iterator
-> data block iterator
-> `GetContext`
-> merge/tombstone/blob post-processing.

Critical invariants: snapshot sequence filtering, L0 newest-first search,
L1+ non-overlap, and tombstone/range tombstone visibility.

## 3. Iterator / Range Scan Path

`DB::NewIterator`
-> `DBImpl::NewIterator`
-> `ColumnFamilyData::GetReferencedSuperVersion`
-> `DBImpl::NewIteratorImpl`
-> memtable iterators
-> `Version::AddIterators`
-> `NewMergingIterator`
-> `DBIter::NewIter`
-> user `Seek` / `Next` / `Prev`
-> `MergingIterator` heap navigation
-> `BlockBasedTableIterator::SeekImpl` for SST children
-> `DBIter` snapshot/tombstone/merge filtering.

Operational focus: high `internal_key_skipped_count` or
`internal_range_del_reseek_count` means the iterator is paying for obsolete
versions, tombstones, or range deletions.

## 4. Flush Path

memtable fills
-> `DBImpl::SwitchMemtable`
-> immutable memtable list
-> `DBImpl::MaybeScheduleFlushOrCompaction`
-> `DBImpl::BackgroundFlush`
-> `FlushJob::PickMemTable`
-> `FlushJob::Run`
-> `FlushJob::WriteLevel0Table`
-> `BuildTable`
-> `BlockBasedTableBuilder::Finish`
-> `MemTableList::TryInstallMemtableFlushResults`
-> `VersionSet::LogAndApply`
-> install SuperVersion.

Failure before manifest install leaves the output file non-live and eligible for
cleanup. Failed memtables are rolled back with
`MemTableList::RollbackMemtableFlush`.

## 5. Compaction Path

compaction score or manual request
-> `DBImpl::MaybeScheduleFlushOrCompaction`
-> `DBImpl::BackgroundCompaction`
-> compaction picker
-> `CompactionJob::Prepare`
-> `CompactionJob::Run`
-> `RunSubcompactions`
-> `CompactionJob::ProcessKeyValueCompaction`
-> `CompactionIterator`
-> output table builders
-> `CompactionJob::Install`
-> `CompactionJob::InstallCompactionResults`
-> `VersionSet::LogAndApply`
-> install SuperVersion.

Critical invariants: input files marked in-compaction, output files not visible
before manifest commit, obsolete keys dropped only when snapshots/lower levels
allow.

## 6. Recovery Path

`DB::Open`
-> `DBImpl::Recover`
-> lock DB
-> CURRENT / MANIFEST discovery
-> `VersionSet::Recover`
-> `VersionEdit::DecodeFrom`
-> reconstruct current Versions
-> collect WALs
-> `DBImpl::RecoverLogFiles`
-> `DBImpl::ProcessLogFile`
-> `log::Reader::ReadRecord`
-> `WriteBatchInternal::InsertInto`
-> rebuild memtables
-> final flush or active WAL restore
-> set recovered sequence.

Critical diagnostics: open LOG, manifest path, WAL list, last sequence,
corruption/missing-file status, file read histograms.

## 7. Manifest / VersionSet Path

flush/compaction/ingest creates `VersionEdit`
-> `VersionSet::LogAndApply`
-> manifest writer queue
-> `VersionBuilder::Apply`
-> `VersionEdit::EncodeTo`
-> `log::Writer::AddRecord`
-> manifest sync
-> append new `Version`
-> install SuperVersion.

This is the metadata commit point for SST membership.

## 8. Cache / BloomFilter Path

`BlockBasedTable::Get`
-> table timestamp check
-> `FullFilterKeyMayMatch`
-> index block lookup
-> data block lookup in block cache
-> optional secondary/compressed cache
-> file read on miss
-> cache insert and `GetContext` counter report.

Metrics: `BLOCK_CACHE_*`, `BLOOM_FILTER_*`, `PerfContext::block_read_time`,
`read_filter_block_nanos`, `read_index_block_nanos`.

## 9. IngestExternalFile Path

`DB::IngestExternalFile`
-> `DBImpl::IngestExternalFile`
-> `ExternalSstFileIngestionJob::Prepare`
-> property/key-range validation
-> overlap flush if needed
-> `ExternalSstFileIngestionJob::Run`
-> assign level and global sequence
-> copy/move/link file
-> `VersionEdit::AddFile`
-> `VersionSet::LogAndApply`
-> listener `OnExternalFileIngested`.

## 10. Checkpoint / Backup Path

`Checkpoint::Create`
-> `CheckpointImpl::CreateCheckpoint`
-> disable file deletions
-> link/copy live SSTs and WALs
-> create CURRENT/MANIFEST/options files
-> rename temp directory.

`BackupEngineImpl::CreateNewBackupWithMetadata`
-> disable file deletions
-> use checkpoint custom callbacks
-> copy/share files
-> write backup metadata
-> garbage collect.
