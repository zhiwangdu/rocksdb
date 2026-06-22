# Flow Research: Get Path

## Scope

This flow covers `DB::Get`, SuperVersion, mutable/immutable memtable lookup,
Version/SST lookup, Bloom filters, index/data blocks, block cache, merge
operands, tombstones, and snapshot sequence visibility.

## Source Evidence

| Step | Source |
|---|---|
| API and implementation | `include/rocksdb/db.h`, `db/db_impl/db_impl.cc` |
| Memtable lookup | `db/memtable.cc` |
| SST file selection | `db/version_set.cc` |
| Table cache | `db/table_cache.cc` |
| Block-based table read | `table/block_based/block_based_table_reader.cc` |
| Get state machine | `table/get_context.cc`, `table/get_context.h` |
| Merge and tombstones | `db/merge_helper.cc`, `db/range_del_aggregator.cc` |

## Call Chain

`DB::Get`
-> `DBImpl::Get`
-> `DBImpl::GetImpl`
-> acquire `SuperVersion`
-> choose snapshot sequence
-> `MemTable::Get` on mutable memtable
-> immutable memtable lookup, newest first
-> `Version::Get`
-> `FilePicker::GetNextFile`
-> `TableCache::Get`
-> `BlockBasedTable::Get`
-> `BlockBasedTable::FullFilterKeyMayMatch`
-> index iterator seek
-> data block cache lookup/read
-> `DataBlockIter::SeekForGet`
-> `GetContext::SaveValue`
-> merge/blob/tombstone post-processing
-> release `SuperVersion`.

## Detailed Behavior

- `DBImpl::Get` normalizes `ReadOptions::io_activity` to `Env::IOActivity::kGet`.
- The read path references a `SuperVersion` so mutable memtable, immutable
  memtables, and current Version stay alive without holding the DB mutex.
- The read sequence comes from explicit snapshot if provided, otherwise latest
  visible sequence. Internal keys newer than the read sequence are ignored.
- `MemTable::Get` can use memtable bloom filters before skiplist lookup.
- `Version::Get` uses `FilePicker` to search files. L0 can contain overlapping
  files and must be searched by newest ordering. L1+ files are non-overlapping
  and binary-searchable.
- `BlockBasedTable::Get` checks table timestamp bounds, then full/prefix filter,
  then index block, then data block. With `read_tier=kBlockCacheTier`, missing
  blocks return incomplete rather than doing disk IO.
- `GetContext` collects cache counters and tracks found/not found/merge/delete
  states. Merge operands are accumulated until a base value or tombstone resolves
  them.
- Range tombstone sequence is compared with candidate keys so covered point
  keys are hidden.

## Metrics

Key signals are `MEMTABLE_HIT`, `MEMTABLE_MISS`, `GET_HIT_L0`, `GET_HIT_L1`,
`GET_HIT_L2_AND_UP`, `BLOCK_CACHE_*`, `BLOOM_FILTER_*`, histogram `DB_GET`,
and PerfContext fields `get_from_memtable_time`,
`get_from_output_files_time`, `block_read_time`, `read_index_block_nanos`,
and `read_filter_block_nanos`.

## Tests

Use `db/db_basic_test.cc`, `db/db_block_cache_test.cc`,
`db/db_bloom_filter_test.cc`, `db/merge_test.cc`, `db/db_merge_operator_test.cc`,
`db/db_range_del_test.cc`, `table/block_based/block_based_table_reader_test.cc`,
and `table/table_test.cc`.
