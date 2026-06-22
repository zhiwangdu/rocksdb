# Module Research Catalog

This catalog explicitly enumerates the modules requested in the objective. The
detailed analysis is split across `research/module-*.md` files; this file is the
coverage map for module-by-module lookup.

| Module | Responsibility boundary | Core source evidence | Key flow / references | Ops/dev focus |
|---|---|---|---|---|
| DBImpl | Owns DB lifecycle, read/write dispatch, background scheduling, errors. | `db/db_impl/db_impl.h`, `db/db_impl/db_impl.cc`, `db/db_impl/db_impl_write.cc`, `db/db_impl/db_impl_open.cc` | `research/module-db-core.md`, `research/flow-write.md`, `research/flow-get.md` | DB mutex, sequence publication, background errors. |
| DB Interface | Public API and compatibility contract. | `include/rocksdb/db.h` | API calls feed DBImpl. | Backward compatibility, documentation, ABI. |
| ColumnFamily | Per-CF memtables, versions, options, stats. | `db/column_family.h`, `db/column_family.cc` | SuperVersion and Version install. | CF isolation but shared WAL/sequence effects. |
| VersionSet | DB-wide metadata and MANIFEST owner. | `db/version_set.h`, `db/version_set.cc` | `VersionSet::Recover`, `LogAndApply` | Manifest correctness, file numbers, last sequence. |
| Version | Immutable CF LSM view. | `db/version_set.h`, `db/version_set.cc` | `Version::Get`, `Version::AddIterators` | Reader lifetime, file search, compaction scores. |
| VersionEdit | Durable metadata delta. | `db/version_edit.h`, `db/version_edit.cc` | `EncodeTo`, `DecodeFrom` | Compatibility and recovery. |
| Manifest | Append-only metadata log. | `db/version_set.cc`, `db/version_edit.cc`, `db/log_writer.cc` | `VersionSet::LogAndApply` | Open latency, crash consistency. |
| WAL | Write durability and recovery input. | `db/log_writer.cc`, `db/log_reader.cc`, `db/wal_manager.cc` | `log::Writer::AddRecord`, `DBImpl::RecoverLogFiles` | Sync latency, WAL retention, corruption. |
| WriteBatch | Encoded mutation batch. | `include/rocksdb/write_batch.h`, `db/write_batch.cc` | `WriteBatchInternal::InsertInto` | Record format, sequence assignment. |
| WriteThread | Group commit coordinator. | `db/write_thread.h`, `db/write_thread.cc` | `JoinBatchGroup`, `EnterAsBatchGroupLeader` | Contention, parallel memtable writes. |
| Recovery | Open-time manifest/WAL replay. | `db/db_impl/db_impl_open.cc`, `db/version_set.cc` | `research/flow-recovery.md` | Corruption modes, many WALs, large manifest. |
| MemTable | Mutable sorted write buffer. | `db/memtable.h`, `db/memtable.cc` | `MemTable::Add`, `MemTable::Get` | Memory, bloom, flush state, hot path. |
| SkipList | Default memtable ordered index. | `memtable/skiplist.h`, `memtable/inlineskiplist.h`, `memtable/skiplistrep.cc` | Memtable insert and lookup. | Concurrent insert correctness. |
| Arena | Fast append-style allocator. | `memory/arena.h`, `util/arena.h` | Memtable and iterator allocation. | Lifetime and fragmentation. |
| SSTable | Immutable sorted-run file. | `table/table_reader.h`, `table/table_builder.h`, `table/block_based/*` | Flush/compaction output and table read. | Format compatibility and checksums. |
| TableBuilder | SST writer interface. | `table/table_builder.h`, `table/block_based/block_based_table_builder.cc` | `Add`, `Finish` | Output order, block boundaries, compression. |
| TableReader | SST reader interface. | `table/table_reader.h`, `table/block_based/block_based_table_reader.cc` | `Get`, `NewIterator` | Table open, cache/filter integration. |
| BlockBasedTable | Default SST implementation. | `table/block_based/block_based_table_reader.cc`, `block_based_table_builder.cc` | Read and build block-based SST. | Hot read path, block cache, filters. |
| Block | Encoded data/index/filter block abstraction. | `table/block_based/block.h`, `block.cc` | Data block iterator. | Restart points and checksum/decompression. |
| DataBlock | User data block in SST. | `table/block_based/block.h`, `block_based_table_iterator.cc` | `DataBlockIter`, `SeekForGet` | Read IO and iterator CPU. |
| IndexBlock | Maps keys to data block handles. | `table/block_based/index_builder.cc`, `binary_search_index_reader.cc`, `hash_index_reader.cc` | Index seek before data block read. | Memory/cache tradeoffs. |
| FilterBlock | Bloom/Ribbon filter storage. | `table/block_based/filter_block.h`, `full_filter_block.cc`, `partitioned_filter_block.cc` | `FullFilterKeyMayMatch` | False positives, prefix compatibility. |
| BloomFilter | Negative lookup avoidance. | `include/rocksdb/filter_policy.h`, `table/block_based/filter_policy.cc` | Full and prefix key may-match checks. | Read amp, bits/key, workload shape. |
| Cache | Generic cache abstraction. | `include/rocksdb/cache.h`, `cache/*` | Block cache and secondary cache. | Capacity, priority, admission. |
| BlockCache | Table block cache integration. | `table/block_based/block_cache.cc`, `table/block_based/block_based_table_reader.cc` | `UpdateCacheHitMetrics`, `UpdateCacheMissMetrics` | Hit rate, scan pollution, pinning. |
| RowCache | Point-read row cache. | `include/rocksdb/options.h`, `db/db_impl/db_impl.cc` | DBImpl read/write integration. | DeleteRange incompatibility and memory. |
| Iterator | User-facing ordered scan. | `include/rocksdb/iterator.h`, `db/db_iter.cc` | `DBIter::NewIter` | Snapshot and tombstone semantics. |
| InternalIterator | Internal-key scan interface. | `table/internal_iterator.h` | Memtable/table/merge iterators. | Key ordering and status propagation. |
| MergingIterator | Heap over child internal iterators. | `table/merging_iterator.cc` | Seek/Next/Prev merge. | Direction switches and range-del reseek. |
| Snapshot | Sequence-based read view. | `db/snapshot_impl.h`, `db/dbformat.h` | ReadOptions snapshot comparisons. | Long snapshots retain old versions. |
| SequenceNumber | Global MVCC timestamp. | `db/dbformat.h`, `db/version_set.h` | Write publish and read visibility. | Monotonicity, recovery, transactions. |
| Transaction | Utility-level conflict/atomicity wrapper. | `include/rocksdb/utilities/transaction.h`, `utilities/transactions/*` | TransactionDB APIs. | Locks, write policies, recovery. |
| WritePrepared | Transaction policy with prepare/commit map. | `utilities/transactions/write_prepared_txn_db.cc`, `write_prepared_txn.h` | Commit map and read callbacks. | Snapshot visibility and seqno recovery. |
| WriteUnprepared | Transaction policy with unprepared writes. | `utilities/transactions/write_unprepared_txn.cc`, `write_unprepared_txn_db.cc` | Unprepared sequence tracking. | Rollback and visibility complexity. |
| Flush | Converts immutable memtables to L0. | `db/db_impl/db_impl_compaction_flush.cc` | `DBImpl::Flush`, `BackgroundFlush` | Flush lag, WAL retention. |
| FlushJob | Flush execution. | `db/flush_job.h`, `db/flush_job.cc` | `Run`, `WriteLevel0Table` | Output file failure, manifest install. |
| Compaction | LSM rewrite plan. | `db/compaction/compaction.h`, `compaction.cc` | Inputs/outputs/reason. | Overlap and output level correctness. |
| CompactionPicker | Chooses compaction inputs. | `db/compaction/compaction_picker.*` | Shared helper and in-progress tracking. | Avoid overlapping active compactions. |
| CompactionJob | Executes compaction. | `db/compaction/compaction_job.*` | `Run`, `InstallCompactionResults` | Output verification and manifest commit. |
| CompactionIterator | Drops/resolves internal records. | `db/compaction/compaction_iterator.*` | `Next`, `PrepareOutput`, filter invocation. | Snapshot/tombstone/merge correctness. |
| CompactionFilter | User drop/rewrite hook. | `include/rocksdb/compaction_filter.h` | Invoked by `CompactionIterator`. | User code in background hot path. |
| Universal Compaction | Sorted-run compaction. | `db/compaction/compaction_picker_universal.cc` | `UniversalCompactionPicker::PickCompaction` | Write amp vs temporary space amp. |
| Level Compaction | Leveled non-overlap compaction. | `db/compaction/compaction_picker_level.cc` | `LevelCompactionPicker::NeedsCompaction` | Read/space amp control. |
| FIFO Compaction | Size/TTL retention compaction. | `db/compaction/compaction_picker_fifo.cc` | `PickTTLCompaction` | Deletion policy and windowed workloads. |
| IngestExternalFile | Imports prebuilt SSTs. | `db/external_sst_file_ingestion_job.cc`, `table/sst_file_writer.cc` | `Prepare`, `Run`, `AssignLevelsForOneBatch` | Overlap, seqno, manifest atomicity. |
| Checkpoint | Consistent live-file snapshot. | `utilities/checkpoint/checkpoint_impl.cc` | `CreateCheckpoint`, `CreateCustomCheckpoint` | File deletion disable and WAL tail copy. |
| BackupEngine | Incremental backup/restore. | `utilities/backup/backup_engine.cc` | `CreateNewBackupWithMetadata`, `RestoreDBFromBackup` | Shared files, checksum, rate limit. |
| RateLimiter | IO token limiter. | `include/rocksdb/rate_limiter.h`, `util/rate_limiter.cc` | `Request`, `RequestToken` | Throughput caps and foreground priority. |
| Env | Portability and thread scheduling. | `include/rocksdb/env.h`, `env/env_posix.cc` | `Schedule`, `StartThread` | Platform boundaries. |
| FileSystem | File IO abstraction. | `include/rocksdb/file_system.h`, `file/*` | Random/sequential/writable file APIs. | Direct IO, fsync, object stores. |
| ThreadPool | Background worker pool. | `include/rocksdb/threadpool.h`, `util/threadpool_imp.cc` | Scheduled jobs. | Flush/compaction capacity. |
| Scheduler | Periodic and background scheduling. | `db/periodic_task_scheduler.cc`, `db/db_impl/db_impl_compaction_flush.cc` | Periodic tasks and BG work. | Starvation and shutdown. |
| EventListener | Operational callbacks. | `include/rocksdb/listener.h` | Flush/compaction/stall/error callbacks. | Callback cost and ordering. |
| Statistics | Tickers/histograms. | `include/rocksdb/statistics.h`, `monitoring/statistics.cc` | `StatisticsImpl` | Monitoring and debugging. |
| PerfContext | Per-thread perf counters. | `include/rocksdb/perf_context.h`, `monitoring/perf_context.cc` | `get_perf_context` | Hot path breakdown. |
| IOStatsContext | Per-thread IO counters. | `include/rocksdb/iostats_context.h`, `monitoring/iostats_context.cc` | `get_iostats_context` | IO latency/bytes. |
| Options | Config surface and validation. | `include/rocksdb/options.h`, `include/rocksdb/advanced_options.h`, `options/*` | Option parsing/dumping. | Misconfiguration and dynamic updates. |
| Tests | Regression evidence. | `db/*_test.cc`, `table/*_test.cc`, `utilities/*_test.cc` | See `research/notes/source-index.md`. | Coverage and failure reproduction. |
| Tools / utilities | Bench, inspect, stress. | `tools/db_bench.cc`, `tools/ldb.cc`, `tools/sst_dump.cc`, `db_stress_tool/*` | `db_bench`, `ldb`, `sst_dump`, `db_stress` | Benchmarking and operational inspection. |

## Shared Metrics and Tests

Use `research/notes/options-metrics-tests.md` for metrics and test coverage by
area. Most modules map to one or more of `Statistics`, `PerfContext`,
`IOStatsContext`, EventListener callbacks, LOG/event-log output, and focused
unit tests.
