# RocksDB Code Map

## 1. Directory Structure

| Directory | Purpose |
|---|---|
| `include/rocksdb/` | Public C++ API, options, statistics, listeners, utilities APIs. |
| `db/` | Core DB implementation, WAL, memtable glue, VersionSet, recovery, tests. |
| `db/db_impl/` | Split `DBImpl` implementation: open, write, flush/compaction, secondary/follower/read-only modes. |
| `db/compaction/` | Compaction plan, picker, job, iterator, subcompaction, tests. |
| `memtable/` | Memtable representation implementations and skiplist code. |
| `table/` | Table format interfaces and generic iterators. |
| `table/block_based/` | Default SST format implementation: blocks, indexes, filters, readers, builders, cache hooks. |
| `cache/` | Cache implementations and secondary/compressed cache support. |
| `env/`, `file/` | Env/FileSystem implementations and file readers/writers. |
| `monitoring/` | Statistics, PerfContext, IOStatsContext, thread status. |
| `options/` | Option parsing, dumping, validation, settable option maps. |
| `utilities/` | Optional modules: transactions, backup, checkpoint, TTL, blob DB, persistent cache, etc. |
| `tools/` | `db_bench`, `ldb`, `sst_dump`, stress/support tools. |
| `docs/components/` | Existing focused read/write flow docs. |

## 2. Core Source Entrances

| Topic | Start here |
|---|---|
| DB API | `include/rocksdb/db.h` |
| Write path | `db/db_impl/db_impl_write.cc` |
| Point reads / iterators | `db/db_impl/db_impl.cc`, `db/version_set.cc`, `db/db_iter.cc` |
| Open/recovery | `db/db_impl/db_impl_open.cc`, `db/version_set.cc` |
| Memtable | `db/memtable.cc`, `memtable/skiplist.h`, `memtable/inlineskiplist.h` |
| WAL | `db/log_writer.cc`, `db/log_reader.cc`, `db/wal_manager.cc` |
| Manifest / versions | `db/version_set.cc`, `db/version_edit.cc`, `db/version_builder.cc` |
| Flush | `db/flush_job.cc`, `db/db_impl/db_impl_compaction_flush.cc` |
| Compaction | `db/compaction/compaction_job.cc`, `db/compaction/compaction_iterator.cc`, picker files |
| SST read/write | `table/block_based/block_based_table_reader.cc`, `table/block_based/block_based_table_builder.cc` |
| Cache/filter | `table/block_based/block_cache.cc`, `cache/lru_cache.cc`, `table/block_based/full_filter_block.cc` |
| Transactions | `utilities/transactions/` |
| Backup/checkpoint | `utilities/backup/backup_engine.cc`, `utilities/checkpoint/checkpoint_impl.cc` |

## 3. Core Class Index

| Class | Path | Why it matters |
|---|---|---|
| `DBImpl` | `db/db_impl/db_impl.h` | Main database implementation and scheduler. |
| `WriteThread` | `db/write_thread.h` | Group commit and write concurrency. |
| `WriteBatchInternal` | `db/write_batch_internal.h` | Batch sequence and replay internals. |
| `MemTable` | `db/memtable.h` | Mutable sorted write buffer. |
| `MemTableList` | `db/memtable_list.h` | Immutable memtable flush queue. |
| `ColumnFamilyData` | `db/column_family.h` | CF-local state and options. |
| `SuperVersion` | `db/column_family.h` | Reader-visible state tuple. |
| `VersionSet` | `db/version_set.h` | Metadata authority and MANIFEST owner. |
| `Version` | `db/version_set.h` | One immutable LSM file view. |
| `VersionEdit` | `db/version_edit.h` | Durable metadata delta. |
| `FileMetaData` | `db/version_edit.h` | SST metadata. |
| `FlushJob` | `db/flush_job.h` | Memtable-to-SST job. |
| `Compaction` | `db/compaction/compaction.h` | Compaction plan. |
| `CompactionPicker` | `db/compaction/compaction_picker.h` | Selects compaction inputs. |
| `CompactionJob` | `db/compaction/compaction_job.h` | Executes compaction. |
| `CompactionIterator` | `db/compaction/compaction_iterator.h` | Drops/resolves internal records during compaction. |
| `TableBuilder` | `table/table_builder.h` | SST writer interface. |
| `TableReader` | `table/table_reader.h` | SST reader interface. |
| `BlockBasedTable` | `table/block_based/block_based_table_reader.h` | Default SST reader. |
| `BlockBasedTableBuilder` | `table/block_based/block_based_table_builder.h` | Default SST builder. |
| `DBIter` | `db/db_iter.h` | User iterator over internal keys. |
| `MergingIterator` | `table/merging_iterator.cc` | Merges memtable/file iterators. |
| `Cache` | `include/rocksdb/cache.h` | Block cache abstraction. |
| `BackupEngine` | `include/rocksdb/utilities/backup_engine.h` | Backup/restore API. |
| `Checkpoint` | `include/rocksdb/utilities/checkpoint.h` | Consistent live-file snapshot API. |

## 4. Core Test Index

| Area | Tests |
|---|---|
| DB basics | `db/db_basic_test.cc`, `db/db_test.cc` |
| Write/WAL | `db/db_write_test.cc`, `db/write_batch_test.cc`, `db/db_wal_test.cc`, `db/log_test.cc` |
| Memtable | `db/db_memtable_test.cc`, `db/memtable_list_test.cc`, `memtable/skiplist_test.cc` |
| Read/iterator | `db/db_iter_test.cc`, `db/db_iterator_test.cc`, `db/db_block_cache_test.cc`, `db/db_bloom_filter_test.cc` |
| Version/recovery | `db/version_set_test.cc`, `db/version_edit_test.cc`, `db/corruption_test.cc`, `db/fault_injection_test.cc` |
| Flush/compaction | `db/db_flush_test.cc`, `db/flush_job_test.cc`, `db/db_compaction_test.cc`, `db/compaction/compaction_job_test.cc` |
| Utilities | `db/external_sst_file_test.cc`, `utilities/checkpoint/checkpoint_test.cc`, `utilities/backup/backup_engine_test.cc`, `utilities/transactions/*_test.cc` |
| Metrics/platform | `db/db_statistics_test.cc`, `db/perf_context_test.cc`, `env/env_test.cc`, `db/db_rate_limiter_test.cc` |

## 5. Recommended Reading Order

1. `DBImpl`: `db/db_impl/db_impl.h`, `db/db_impl/db_impl.cc`
2. Write path: `db/db_impl/db_impl_write.cc`, `db/write_thread.*`
3. MemTable: `db/memtable.*`, `memtable/skiplist.h`
4. WAL: `db/log_writer.*`, `db/log_reader.*`
5. VersionSet / Manifest: `db/version_set.*`, `db/version_edit.*`
6. SST / BlockBasedTable: `table/block_based/block_based_table_reader.cc`,
   `table/block_based/block_based_table_builder.cc`
7. Iterator: `db/db_iter.*`, `table/merging_iterator.cc`
8. Flush: `db/flush_job.cc`, flush scheduling in `db_impl_compaction_flush.cc`
9. Compaction: `db/compaction/compaction_job.cc`,
   `db/compaction/compaction_iterator.cc`, picker files
10. Recovery: `db/db_impl/db_impl_open.cc`, `db/version_set.cc`
11. Cache / BloomFilter: `table/block_based/block_cache.cc`,
    `table/block_based/full_filter_block.cc`, `cache/*`
12. Options / Statistics: `include/rocksdb/options.h`,
    `include/rocksdb/statistics.h`, `monitoring/*`
13. Backup / Checkpoint: `utilities/backup/backup_engine.cc`,
    `utilities/checkpoint/checkpoint_impl.cc`
