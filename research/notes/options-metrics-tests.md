# Options, Metrics, and Test Research

## High-Impact Options

| Option | Default | Source evidence | Operational impact |
|---|---:|---|---|
| `write_buffer_size` | `64 << 20` | `include/rocksdb/options.h` | Per-column-family mutable memtable budget; controls flush frequency and L0 file production. |
| `max_write_buffer_number` | `2` | `include/rocksdb/advanced_options.h` | Upper bound on mutable plus immutable write buffers before stalls; higher values absorb bursts but increase memory/WAL retention. |
| `min_write_buffer_number_to_merge` | `1` | `include/rocksdb/advanced_options.h` | Allows merging multiple immutable memtables into one flush output. |
| `db_write_buffer_size` | `0` | `include/rocksdb/options.h` | Cross-CF DB-wide memtable budget; disabled by default. |
| `write_buffer_manager` | `nullptr` | `include/rocksdb/options.h`, `memtable/write_buffer_manager.cc` | Cross-DB memory accounting and flush pressure. |
| `level0_file_num_compaction_trigger` | `4` | `include/rocksdb/options.h` | L0 compaction trigger. |
| `level0_slowdown_writes_trigger` | `20` | `include/rocksdb/advanced_options.h` | Foreground write slowdown trigger. |
| `level0_stop_writes_trigger` | `36` | `include/rocksdb/advanced_options.h` | Foreground write stop trigger. |
| `target_file_size_base` | `64 * 1048576` | `include/rocksdb/advanced_options.h` | L1 output file target size; higher levels scale by multiplier. |
| `max_bytes_for_level_base` | `256 * 1048576` | `include/rocksdb/options.h` | L1 total target size; drives leveled compaction pressure and space amplification. |
| `level_compaction_dynamic_level_bytes` | `true` | `include/rocksdb/advanced_options.h` | Dynamically picks base level to reduce space amp; `VersionStorageInfo::ComputeCompactionScore` is the implementation reference. |
| `compaction_style` | `kCompactionStyleLevel` | `include/rocksdb/advanced_options.h` | Chooses leveled, universal, FIFO, or none. |
| `soft_pending_compaction_bytes_limit` | `64GB` | `include/rocksdb/advanced_options.h` | Write slowdown based on compaction debt. |
| `hard_pending_compaction_bytes_limit` | `256GB` | `include/rocksdb/advanced_options.h` | Write stop based on compaction debt. |
| `max_background_jobs` | `2` | `include/rocksdb/options.h` | Shared flush/compaction worker budget. |
| `max_background_compactions` | `-1` | `include/rocksdb/options.h` | Legacy override for compaction workers. |
| `max_background_flushes` | `-1` | `include/rocksdb/options.h` | Legacy override for flush workers. |
| `max_open_files` | `-1` | `include/rocksdb/options.h` | `-1` opens all table readers at open; bounded values use table cache. |
| `WAL_ttl_seconds` | `0` | `include/rocksdb/options.h` | Archived WAL deletion by age. |
| `WAL_size_limit_MB` | `0` | `include/rocksdb/options.h` | Archived WAL deletion by size. |
| `max_total_wal_size` | `0` | `include/rocksdb/options.h` | Forces flushes to limit live WAL retention across CFs. |
| `manual_wal_flush` | `false` | `include/rocksdb/options.h` | User controls WAL flush; affects sync and rate limiter validation in `DBImpl::WriteImpl`. |
| `atomic_flush` | `false` | `include/rocksdb/options.h` | All selected CF memtables are flushed atomically with a single manifest install. |
| `allow_concurrent_memtable_write` | `true` | `include/rocksdb/options.h` | Enables parallel memtable insertion when memtable factory supports it. |
| `enable_pipelined_write` | `false` | `include/rocksdb/options.h` | Separates WAL and memtable write queues. |
| `unordered_write` | `false` | `include/rocksdb/options.h` | Higher write throughput with relaxed snapshot immutability unless paired with transaction visibility logic. |
| `rate_limiter` | `nullptr` | `include/rocksdb/options.h`, `include/rocksdb/rate_limiter.h` | Limits reads/writes depending on mode and IO priority. |
| `block_cache` | `nullptr`, creates internal 32MB cache if not disabled | `include/rocksdb/table.h` | Main uncompressed block cache for data/index/filter blocks. |
| `cache_index_and_filter_blocks` | `false` | `include/rocksdb/table.h` | Moves index/filter blocks into block cache instead of pinning them in table readers. |
| `pin_l0_filter_and_index_blocks_in_cache` | `false` | `include/rocksdb/table.h` | Pins L0 index/filter blocks when caching them. |
| `filter_policy` | `nullptr` | `include/rocksdb/table.h` | Enables Bloom or Ribbon filters. |
| `whole_key_filtering` | `true` | `include/rocksdb/table.h` | Full-key filters for point lookups. |
| `prefix_extractor` | `nullptr` | `include/rocksdb/options.h` | Enables prefix bloom/index behavior; must match comparator domain assumptions. |
| `memtable_whole_key_filtering` | `false` | `include/rocksdb/advanced_options.h` | Adds whole keys to the memtable bloom filter. |
| `paranoid_checks` | `true` | `include/rocksdb/options.h` | Stricter open/recovery and file validation behavior. |

## Metrics and Counters

| Metric family | Source evidence | Useful fields |
|---|---|---|
| Block cache tickers | `include/rocksdb/statistics.h`, `table/block_based/block_based_table_reader.cc` | `BLOCK_CACHE_HIT`, `BLOCK_CACHE_MISS`, `BLOCK_CACHE_DATA_HIT`, `BLOCK_CACHE_DATA_MISS`, `BLOCK_CACHE_INDEX_*`, `BLOCK_CACHE_FILTER_*`, `BLOCK_CACHE_BYTES_READ`, `BLOCK_CACHE_BYTES_WRITE` |
| Bloom filter tickers | `include/rocksdb/statistics.h`, `BlockBasedTable::FullFilterKeyMayMatch` | `BLOOM_FILTER_USEFUL`, `BLOOM_FILTER_FULL_POSITIVE`, `BLOOM_FILTER_PREFIX_CHECKED`, `BLOOM_FILTER_PREFIX_USEFUL` |
| Memtable / Get tickers | `include/rocksdb/statistics.h`, `db/memtable.cc`, `db/version_set.cc` | `MEMTABLE_HIT`, `MEMTABLE_MISS`, `GET_HIT_L0`, `GET_HIT_L1`, `GET_HIT_L2_AND_UP` |
| Write / WAL tickers | `include/rocksdb/statistics.h`, `db/db_impl/db_impl_write.cc` | `NUMBER_KEYS_WRITTEN`, `BYTES_WRITTEN`, `WRITE_DONE_BY_SELF`, `WRITE_DONE_BY_OTHER`, `WRITE_WITH_WAL`, `WAL_FILE_SYNCED`, `WAL_FILE_BYTES` |
| Flush / compaction tickers | `include/rocksdb/statistics.h`, `db/flush_job.cc`, `db/compaction/compaction_job.cc` | `FLUSH_WRITE_BYTES`, `COMPACT_READ_BYTES`, `COMPACT_WRITE_BYTES`, `COMPACTION_KEY_DROP_*`, `COMPACTION_RANGE_DEL_DROP_OBSOLETE` |
| Stall metrics | `include/rocksdb/statistics.h`, `db/write_controller.*` | `STALL_MICROS`, histogram `WRITE_STALL` |
| Histograms | `include/rocksdb/statistics.h` | `DB_GET`, `DB_WRITE`, `COMPACTION_TIME`, `COMPACTION_CPU_TIME`, `WAL_FILE_SYNC_MICROS`, `MANIFEST_FILE_SYNC_MICROS`, `SST_READ_MICROS`, `SST_WRITE_MICROS`, `FLUSH_TIME` |
| PerfContext | `include/rocksdb/perf_context.h`, `monitoring/perf_context.cc` | `write_wal_time`, `write_memtable_time`, `write_delay_time`, `get_from_memtable_time`, `get_from_output_files_time`, `block_cache_hit_count`, `block_read_time`, `internal_key_skipped_count`, `internal_range_del_reseek_count` |
| IOStatsContext | `include/rocksdb/iostats_context.h`, `monitoring/iostats_context.cc` | `bytes_read`, `bytes_written`, `open_nanos`, `allocate_nanos`, `write_nanos`, `read_nanos`, `fsync_nanos`, `range_sync_nanos`, per-temperature file IO counters |
| EventListener | `include/rocksdb/listener.h` | `OnFlushBegin`, `OnFlushCompleted`, `OnCompactionBegin`, `OnCompactionPreCommit`, `OnCompactionCompleted`, `OnSubcompactionBegin`, `OnExternalFileIngested`, `OnBackgroundError`, `OnStallConditionsChanged` |

## Tests as Coverage Evidence

| Topic | Tests |
|---|---|
| WAL and recovery | `db/db_wal_test.cc`, `db/log_test.cc`, `db/wal_manager_test.cc`, `db/corruption_test.cc`, `db/fault_injection_test.cc`, `db/repair_test.cc` |
| Write batching and callbacks | `db/db_write_test.cc`, `db/write_batch_test.cc`, `db/write_callback_test.cc`, `db/db_kv_checksum_test.cc` |
| Memtable and skiplist | `db/db_memtable_test.cc`, `db/memtable_list_test.cc`, `memtable/skiplist_test.cc`, `memtable/inlineskiplist_test.cc` |
| Block cache and filters | `db/db_block_cache_test.cc`, `db/db_bloom_filter_test.cc`, `table/block_based/full_filter_block_test.cc`, `table/block_based/partitioned_filter_block_test.cc`, `table/block_based/data_block_hash_index_test.cc` |
| Iteration and range deletion | `db/db_iter_test.cc`, `db/db_iterator_test.cc`, `db/db_iter_stress_test.cc`, `db/db_range_del_test.cc`, `db/range_del_aggregator_test.cc` |
| Flush and compaction | `db/db_flush_test.cc`, `db/flush_job_test.cc`, `db/db_compaction_test.cc`, `db/db_universal_compaction_test.cc`, `db/compaction/compaction_job_test.cc`, `db/compaction/compaction_picker_test.cc` |
| Version metadata | `db/version_set_test.cc`, `db/version_builder_test.cc`, `db/version_edit_test.cc`, `db/db_dynamic_level_test.cc` |
| Ingestion / utilities | `db/external_sst_file_test.cc`, `utilities/checkpoint/checkpoint_test.cc`, `utilities/backup/backup_engine_test.cc`, `utilities/sorted_run_builder/sorted_run_builder_test.cc` |
| Transactions | `utilities/transactions/transaction_test.cc`, `utilities/transactions/optimistic_transaction_test.cc`, `utilities/transactions/write_prepared_transaction_test.cc`, `utilities/transactions/write_unprepared_transaction_test.cc`, `utilities/transactions/write_prepared_transaction_seqno_test.cc` |
| Env, rate limiter, stats | `env/env_test.cc`, `file/random_access_file_reader_test.cc`, `db/db_rate_limiter_test.cc`, `db/db_statistics_test.cc`, `db/perf_context_test.cc`, `monitoring/statistics_test.cc` |
