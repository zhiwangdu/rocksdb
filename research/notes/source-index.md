# RocksDB Source Evidence Index

This file is the research index used by the generated RocksDB kernel
documentation. It intentionally points to implementation files, method names,
test files, metrics, and option declarations rather than high-level API pages.

## Core Entry Points

| Area | Source evidence | Classes / methods |
|---|---|---|
| Public DB API | `include/rocksdb/db.h` | `DB::Open`, `DB::Put`, `DB::Write`, `DB::Get`, `DB::NewIterator`, `DB::Flush`, `DB::CompactRange`, `DB::IngestExternalFile` |
| DB implementation | `db/db_impl/db_impl.h`, `db/db_impl/db_impl.cc` | `DBImpl`, `DBImpl::GetImpl`, `DBImpl::NewIterator`, `DBImpl::IngestExternalFile`, `DBImpl::SyncWAL`, `DBImpl::FlushWAL` |
| Write path | `db/db_impl/db_impl_write.cc` | `DBImpl::Write`, `DBImpl::WriteImpl`, `DBImpl::PipelinedWriteImpl`, `DBImpl::UnorderedWriteMemtable` |
| Read path | `db/db_impl/db_impl.cc`, `db/version_set.cc` | `DBImpl::GetImpl`, `Version::Get`, `Version::MultiGet`, `TableCache::Get` |
| Open / recovery | `db/db_impl/db_impl_open.cc`, `db/version_set.cc` | `DBImpl::Recover`, `DBImpl::RecoverLogFiles`, `DBImpl::ProcessLogFile`, `VersionSet::Recover`, `VersionSet::TryRecover` |
| Flush scheduling | `db/db_impl/db_impl_compaction_flush.cc`, `db/flush_job.cc` | `DBImpl::Flush`, `DBImpl::BackgroundFlush`, `FlushJob::Run`, `FlushJob::WriteLevel0Table` |
| Compaction scheduling | `db/db_impl/db_impl_compaction_flush.cc`, `db/compaction/*` | `DBImpl::BackgroundCompaction`, `LevelCompactionPicker::NeedsCompaction`, `UniversalCompactionPicker::PickCompaction`, `FIFOCompactionPicker::PickTTLCompaction`, `CompactionJob::Run` |
| Version metadata | `db/version_set.h`, `db/version_set.cc`, `db/version_edit.h`, `db/version_edit.cc` | `VersionSet`, `Version`, `VersionStorageInfo`, `VersionEdit`, `FileMetaData`, `VersionSet::LogAndApply`, `VersionEdit::EncodeTo`, `VersionEdit::DecodeFrom` |
| Column family state | `db/column_family.h`, `db/column_family.cc` | `ColumnFamilyData`, `SuperVersion`, `ColumnFamilySet`, `ColumnFamilyData::GetReferencedSuperVersion`, `ColumnFamilyData::InstallSuperVersion` |
| WAL | `db/log_writer.h`, `db/log_writer.cc`, `db/log_reader.h`, `db/log_reader.cc`, `db/wal_manager.cc` | `log::Writer::AddRecord`, `log::Reader::ReadRecord`, `WalManager` |
| WriteBatch | `include/rocksdb/write_batch.h`, `db/write_batch.cc`, `db/write_batch_internal.h` | `WriteBatch`, `WriteBatchInternal::SetSequence`, `WriteBatchInternal::InsertInto` |
| Write thread | `db/write_thread.h`, `db/write_thread.cc` | `WriteThread`, `WriteThread::JoinBatchGroup`, `WriteThread::EnterAsBatchGroupLeader`, `WriteThread::LaunchParallelMemTableWriters` |
| MemTable | `db/memtable.h`, `db/memtable.cc`, `db/memtable_list.h`, `db/memtable_list.cc` | `MemTable::Add`, `MemTable::Get`, `MemTableList::PickMemtablesToFlush`, `MemTableList::TryInstallMemtableFlushResults` |
| SkipList / Arena | `memtable/skiplist.h`, `memtable/inlineskiplist.h`, `memory/arena.h`, `util/arena.h` | `SkipList`, `InlineSkipList`, `Arena` |
| SST build | `db/builder.h`, `db/builder.cc`, `table/table_builder.h`, `table/block_based/block_based_table_builder.cc` | `BuildTable`, `TableBuilder`, `BlockBasedTableBuilder::Add`, `BlockBasedTableBuilder::Finish` |
| SST read | `table/table_reader.h`, `table/block_based/block_based_table_reader.cc`, `table/block_based/block_based_table_iterator.cc` | `TableReader`, `BlockBasedTable::Get`, `BlockBasedTable::NewIterator`, `BlockBasedTableIterator::SeekImpl` |
| Blocks / filters | `table/block_based/block.h`, `table/block_based/block_builder.h`, `table/block_based/filter_block.h`, `table/block_based/full_filter_block.cc`, `table/block_based/partitioned_filter_block.cc` | `Block`, `BlockBuilder`, `FilterBlockReader`, `FullFilterBlockReader`, `PartitionedFilterBlockReader` |
| Block cache | `include/rocksdb/cache.h`, `cache/lru_cache.cc`, `cache/clock_cache.cc`, `table/block_based/block_cache.cc` | `Cache`, `LRUCache`, `ClockCache`, `BlockBasedTable::UpdateCacheHitMetrics`, `BlockBasedTable::UpdateCacheMissMetrics` |
| Iterators | `db/db_iter.h`, `db/db_iter.cc`, `db/arena_wrapped_db_iter.cc`, `table/merging_iterator.cc`, `table/two_level_iterator.cc` | `DBIter`, `ArenaWrappedDBIter`, `MergingIterator`, `TwoLevelIterator` |
| Range deletion | `db/range_del_aggregator.h`, `db/range_del_aggregator.cc`, `db/range_tombstone_fragmenter.h`, `db/range_tombstone_fragmenter.cc` | `RangeDelAggregator`, `FragmentedRangeTombstoneIterator` |
| Merge | `db/merge_helper.h`, `db/merge_helper.cc`, `include/rocksdb/merge_operator.h` | `MergeHelper`, `MergeOperator`, `AssociativeMergeOperator` |
| Ingestion | `db/external_sst_file_ingestion_job.cc`, `table/sst_file_writer.cc`, `include/rocksdb/sst_file_writer.h` | `ExternalSstFileIngestionJob::Prepare`, `ExternalSstFileIngestionJob::Run`, `SstFileWriter` |
| Checkpoint | `utilities/checkpoint/checkpoint_impl.cc`, `include/rocksdb/utilities/checkpoint.h` | `Checkpoint::Create`, `CheckpointImpl::CreateCheckpoint`, `CheckpointImpl::CreateCustomCheckpoint` |
| Backup | `utilities/backup/backup_engine.cc`, `include/rocksdb/utilities/backup_engine.h` | `BackupEngine::Open`, `BackupEngineImpl::CreateNewBackupWithMetadata`, `BackupEngineImpl::RestoreDBFromBackup` |
| Transactions | `utilities/transactions/*`, `include/rocksdb/utilities/transaction*.h` | `PessimisticTransactionDB`, `OptimisticTransactionDBImpl`, `WritePreparedTxnDB`, `WriteUnpreparedTxnDB`, `TransactionUtil` |
| Env / FileSystem | `include/rocksdb/env.h`, `include/rocksdb/file_system.h`, `env/env_posix.cc`, `file/*` | `Env::Schedule`, `Env::StartThread`, `FileSystem`, `RandomAccessFileReader`, `WritableFileWriter` |
| Thread pools / scheduler | `include/rocksdb/threadpool.h`, `env/env_posix.cc`, `util/threadpool_imp.cc`, `db/periodic_task_scheduler.cc` | `ThreadPool`, `PosixEnv::Schedule`, `PeriodicTaskScheduler` |
| Rate limiter | `include/rocksdb/rate_limiter.h`, `util/rate_limiter.cc` | `RateLimiter::Request`, `RateLimiter::RequestToken`, `NewGenericRateLimiter` |
| Metrics | `include/rocksdb/statistics.h`, `include/rocksdb/perf_context.h`, `include/rocksdb/iostats_context.h`, `monitoring/*` | `Tickers`, `Histograms`, `PerfContext`, `IOStatsContext`, `StatisticsImpl` |
| Tools | `tools/db_bench.cc`, `tools/db_bench_tool.cc`, `tools/ldb.cc`, `tools/sst_dump.cc`, `db_stress_tool/*` | `db_bench`, `ldb`, `sst_dump`, `db_stress` |

## Tests Worth Reading

| Concern | Test evidence |
|---|---|
| Write path, WAL, write controller | `db/db_write_test.cc`, `db/write_batch_test.cc`, `db/db_wal_test.cc`, `db/log_test.cc`, `db/write_controller_test.cc`, `db/write_callback_test.cc` |
| Memtable and write buffers | `db/db_memtable_test.cc`, `db/memtable_list_test.cc`, `db/db_write_buffer_manager_test.cc`, `memtable/skiplist_test.cc`, `memtable/inlineskiplist_test.cc`, `memtable/write_buffer_manager_test.cc` |
| Read path and iterators | `db/db_basic_test.cc`, `db/db_iter_test.cc`, `db/db_iterator_test.cc`, `db/db_block_cache_test.cc`, `db/db_bloom_filter_test.cc`, `db/prefix_test.cc`, `table/block_based/block_based_table_reader_test.cc` |
| Version / manifest / recovery | `db/version_set_test.cc`, `db/version_edit_test.cc`, `db/db_log_iter_test.cc`, `db/corruption_test.cc`, `db/repair_test.cc`, `db/fault_injection_test.cc`, `db/wal_manager_test.cc` |
| Flush / compaction | `db/db_flush_test.cc`, `db/flush_job_test.cc`, `db/db_compaction_test.cc`, `db/compaction/compaction_job_test.cc`, `db/compaction/compaction_picker_test.cc`, `db/db_universal_compaction_test.cc`, `db/db_dynamic_level_test.cc` |
| Range deletion and tombstones | `db/db_range_del_test.cc`, `db/range_del_aggregator_test.cc`, `db/range_tombstone_fragmenter_test.cc`, `db/compaction/compaction_iterator_test.cc` |
| External ingest | `db/external_sst_file_test.cc`, `db/external_sst_file_basic_test.cc`, `table/table_test.cc` |
| Checkpoint / backup | `utilities/checkpoint/checkpoint_test.cc`, `utilities/backup/backup_engine_test.cc` |
| Transactions | `utilities/transactions/transaction_test.cc`, `utilities/transactions/optimistic_transaction_test.cc`, `utilities/transactions/write_prepared_transaction_test.cc`, `utilities/transactions/write_unprepared_transaction_test.cc`, `utilities/transactions/write_prepared_transaction_seqno_test.cc` |
| Env, rate limiter, metrics | `env/env_test.cc`, `env/io_posix_test.cc`, `db/db_rate_limiter_test.cc`, `db/db_statistics_test.cc`, `db/perf_context_test.cc`, `monitoring/statistics_test.cc`, `monitoring/iostats_context_test.cc` |

## Existing Component Docs

This checkout already contains focused component walkthroughs under
`docs/components/read_flow/` and `docs/components/write_flow/`. The new final
docs in `docs/rocksdb-*.md` cite source files directly and use those component
docs as supporting structure, not as a substitute for source evidence.
