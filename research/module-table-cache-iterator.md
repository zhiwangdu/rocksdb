# Module Research: SST, BlockBasedTable, Cache, BloomFilter, Iterators

## Module Responsibilities

Table modules store immutable sorted runs on disk and provide point/range read
primitives. They own SST construction, block layout, filter/index metadata,
block cache integration, table readers, and internal iterators. They do not own
write batching, sequence assignment, or compaction policy.

## Core Source

| Path | Class / method | Role |
|---|---|---|
| `table/table_builder.h`, `table/table_reader.h` | `TableBuilder`, `TableReader` | Generic table format interfaces. |
| `table/block_based/block_based_table_builder.cc` | `BlockBasedTableBuilder::Add`, `Flush`, `WriteBlock`, `WriteFilterBlock`, `WriteIndexBlock`, `Finish` | Builds block-based SSTs. |
| `table/block_based/block_based_table_reader.cc` | `BlockBasedTable::Open`, `Get`, `NewIterator`, `FullFilterKeyMayMatch`, `UpdateCacheHitMetrics` | Reads block-based SSTs. |
| `table/block_based/block_based_table_iterator.cc` | `BlockBasedTableIterator::SeekImpl`, `Next`, `Prev`, `InitDataBlock` | Per-SST range iteration. |
| `table/block_based/block.h`, `block_builder.h` | `Block`, `BlockBuilder`, `DataBlockIter` | Data/index block encoding and iteration. |
| `table/block_based/filter_block.h`, `full_filter_block.cc`, `partitioned_filter_block.cc` | `FilterBlockReader`, full and partitioned filters | Bloom/Ribbon lookup gates. |
| `table/block_based/index_builder.cc`, `binary_search_index_reader.cc`, `hash_index_reader.cc`, `partitioned_index_reader.cc` | Index builders/readers | Maps internal keys to data block handles. |
| `table/block_based/block_cache.cc`, `include/rocksdb/cache.h`, `cache/lru_cache.cc`, `cache/clock_cache.cc` | `Cache`, `LRUCache`, `ClockCache` | Block cache implementations and block handles. |
| `db/table_cache.h`, `db/table_cache.cc` | `TableCache` | Caches table readers by file. |
| `db/db_iter.cc`, `db/arena_wrapped_db_iter.cc` | `DBIter`, `ArenaWrappedDBIter` | User-facing iterator semantics over internal entries. |
| `table/merging_iterator.cc`, `table/two_level_iterator.cc` | `MergingIterator`, `TwoLevelIterator` | Merge child iterators and lazily open per-file iterators. |

## Data Structures

| Structure | Role |
|---|---|
| Data block | Sorted internal key/value entries with restart points; read by `DataBlockIter`. |
| Index block | Maps separator keys to block handles; can be binary, hash, or partitioned index. |
| Filter block | Full, partitioned, or legacy filters; `BlockBasedTable::FullFilterKeyMayMatch` gates point lookups. |
| Metaindex/properties block | Table properties, compression dictionary, range deletion block, filter/index metadata. |
| Block cache key | Stable cache key derived from file identity plus block offset/type. |
| `GetContext` | Read-path state machine for found/not found/merge/deletion and cache metric aggregation. |
| `DBIter` | Converts internal key stream into user-visible keys by applying snapshot sequence, tombstones, merge operands, and range deletions. |
| `MergingIterator` | Heap over child internal iterators from memtables and files. |

## Key Flows

### Point Lookup in SST

`Version::Get`
-> `TableCache::Get`
-> `BlockBasedTable::Get`
-> `BlockBasedTable::FullFilterKeyMayMatch`
-> `BlockBasedTable::NewIndexIterator`
-> index seek
-> `NewDataBlockIterator`
-> block cache lookup or file read
-> `DataBlockIter::SeekForGet`
-> `GetContext::SaveValue`.

### Iterator / Range Scan

`DBImpl::NewIterator`
-> `DBImpl::NewIteratorImpl`
-> child iterators from mutable memtable, immutable memtables, and `Version::AddIterators`
-> `NewMergingIterator`
-> `DBIter::NewIter`
-> user `Seek` / `Next` / `Prev`
-> internal iterator heap navigation
-> snapshot and tombstone filtering in DBIter.

## Concurrency Model

| Area | Behavior |
|---|---|
| SST files | Immutable after manifest publication. |
| Table readers | Shared and refcounted through table cache/version metadata. |
| Block cache | Concurrent cache API; block cache metrics are recorded per block type in `BlockBasedTable`. |
| Iterators | Not thread-safe; pin referenced blocks/readers through cleanup chains and `PinnedIteratorsManager`. |
| Async IO | `BlockBasedTableIterator` and MultiGet can use prefetch/async file reader paths where configured. |

## Configuration

| Option | Default | Effect |
|---|---:|---|
| `BlockBasedTableOptions::block_cache` | `nullptr` meaning internal 32MB cache unless disabled | Main block cache. |
| `no_block_cache` | `false` | Disables block cache. |
| `cache_index_and_filter_blocks` | `false` | Uses block cache for index/filter blocks. |
| `pin_l0_filter_and_index_blocks_in_cache` | `false` | Pins L0 index/filter blocks when cached. |
| `filter_policy` | `nullptr` | Enables Bloom/Ribbon filters. |
| `whole_key_filtering` | `true` | Full-key point lookup filter. |
| `prefix_extractor` | `nullptr` | Prefix filters and prefix seek behavior. |
| `read_options.fill_cache` | `true` by API default | Whether user reads populate block cache. |
| `read_options.read_tier` | `kReadAllTier` by API default | `kBlockCacheTier` fails rather than doing disk IO. |

## Metrics, PerfContext, Logs

| Signal | Use |
|---|---|
| `BLOCK_CACHE_*` tickers | Hit/miss/add/bytes by block type. |
| `BLOOM_FILTER_*` tickers | Filter usefulness and false-positive pressure. |
| `PerfContext::block_read_count`, `block_read_byte`, `block_read_time` | Disk block read cost. |
| `PerfContext::read_index_block_nanos`, `read_filter_block_nanos`, `new_table_block_iter_nanos`, `block_seek_nanos` | Table-reader CPU/IO split. |
| `PerfContext::internal_key_skipped_count`, `internal_delete_skipped_count`, `internal_range_del_reseek_count` | Iterator tombstone/version cleanup pressure. |
| Histograms `SST_READ_MICROS`, `READ_BLOCK_GET_MICROS`, `FILE_READ_*` | Read latency by activity. |

## Tests

| Test | Coverage |
|---|---|
| `table/table_test.cc` | Table building/reading and ingest-related table behavior. |
| `table/block_based/block_based_table_reader_test.cc` | BlockBasedTable reader behavior. |
| `table/block_based/block_test.cc` | Block encoding and iterator behavior. |
| `table/block_based/full_filter_block_test.cc`, `partitioned_filter_block_test.cc` | Filter format and lookup behavior. |
| `db/db_block_cache_test.cc`, `cache/cache_test.cc`, `cache/lru_cache_test.cc` | Cache behavior and integration. |
| `db/db_iter_test.cc`, `db/db_iterator_test.cc`, `table/merger_test.cc` | Iterator correctness. |

## Operational and Development Concerns

- Poor block cache hit rate can come from too-small cache, wrong block size,
  scan pollution, disabled `fill_cache=false` patterns, or index/filter blocks
  outside cache.
- Bloom filters reduce read amplification only when the prefix/whole-key domain
  matches the workload. Bad prefix extractor choices can produce incorrect
  assumptions or ineffective filters.
- Iterators pay for all hidden internal keys. High skipped-key counters usually
  indicate old versions, tombstones, or merge operands waiting for compaction.
- Avoid adding allocations on `BlockBasedTable::Get`, `DataBlockIter`, and
  `DBIter` hot paths.
- Validate table changes with block tests, table tests, cache tests, iterator
  tests, prefix tests, and file checksum/corruption tests.
