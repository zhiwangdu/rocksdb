# Flow Research: Cache and BloomFilter Path

## Scope

This flow covers block cache, row cache, filter policy, full and partitioned
filters, cache key construction, block pinning, compressed/secondary cache,
cache misses, and read amplification control.

## Source Evidence

| Step | Source |
|---|---|
| Cache API | `include/rocksdb/cache.h`, `include/rocksdb/advanced_cache.h` |
| Cache implementations | `cache/lru_cache.cc`, `cache/clock_cache.cc`, `cache/secondary_cache*.cc` |
| Block cache integration | `table/block_based/block_cache.cc`, `table/block_based/block_based_table_reader.cc` |
| Filters | `include/rocksdb/filter_policy.h`, `table/block_based/filter_policy.cc`, `full_filter_block.cc`, `partitioned_filter_block.cc` |
| Row cache | `db/db_impl/db_impl.cc`, row cache option in `DBOptions` |

## Call Chain

`Version::Get`
-> `BlockBasedTable::Get`
-> full/prefix filter check through `FilterBlockReader`
-> if maybe match, index block lookup
-> data block lookup in block cache
-> secondary/compressed cache fallback if configured
-> file read on miss
-> insert block into cache according to `ReadOptions` and table options.

## Amplification Control

- Bloom filters avoid index/data block lookup for negative point reads.
- Block cache avoids repeated disk reads for data, index, filter, and
  compression dictionary blocks.
- Pinning index/filter blocks reduces lookup CPU/IO at the cost of cache
  residency.
- Partitioned filters/indexes reduce memory overhead for large SSTs.
- Row cache can bypass lower-level lookup for repeated point reads but is not
  compatible with DeleteRange in `DBImpl::WriteImpl`.

## Metrics

Use `BLOCK_CACHE_MISS`, `BLOCK_CACHE_HIT`, `BLOCK_CACHE_DATA_*`,
`BLOCK_CACHE_INDEX_*`, `BLOCK_CACHE_FILTER_*`, `BLOCK_CACHE_BYTES_*`,
`BLOOM_FILTER_USEFUL`, `BLOOM_FILTER_FULL_POSITIVE`,
`BLOOM_FILTER_PREFIX_USEFUL`, `PerfContext::block_cache_hit_count`,
`block_read_count`, `block_read_time`, and `read_filter_block_nanos`.

## Tests

Use `db/db_block_cache_test.cc`, `cache/cache_test.cc`, `cache/lru_cache_test.cc`,
`cache/compressed_secondary_cache_test.cc`, `db/db_bloom_filter_test.cc`,
`table/block_based/full_filter_block_test.cc`, and
`table/block_based/partitioned_filter_block_test.cc`.
