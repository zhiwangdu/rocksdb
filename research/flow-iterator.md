# Flow Research: Iterator and Range Scan Path

## Scope

This flow covers `NewIterator`, `InternalIterator`, `MergingIterator`,
`LevelIterator`/two-level iterators, `BlockBasedTableIterator`, seek/next/prev,
prefix seek, snapshot visibility, tombstones, and range deletions.

## Source Evidence

| Step | Source |
|---|---|
| User iterator creation | `db/db_impl/db_impl.cc`, `db/arena_wrapped_db_iter.cc` |
| User-visible iterator | `db/db_iter.h`, `db/db_iter.cc` |
| Internal iterator interface | `table/internal_iterator.h` |
| Merge iterator | `table/merging_iterator.cc`, `table/merging_iterator.h` |
| File iterators | `table/two_level_iterator.cc`, `db/version_set.cc` |
| Block iterator | `table/block_based/block_based_table_iterator.cc` |
| Range deletion | `db/range_del_aggregator.cc`, `db/range_tombstone_fragmenter.cc` |

## Call Chain

`DB::NewIterator`
-> `DBImpl::NewIterator`
-> `ColumnFamilyData::GetReferencedSuperVersion`
-> choose snapshot sequence
-> `DBImpl::NewIteratorImpl`
-> mutable memtable iterator
-> immutable memtable iterators
-> `Version::AddIterators`
-> table/two-level child iterators
-> `NewMergingIterator`
-> `DBIter::NewIter`
-> user `Seek` / `Next` / `Prev`
-> internal heap navigation
-> `DBIter` filters by snapshot, tombstones, range deletions, and merge state.

## Detailed Behavior

- `DBImpl::NewIterator` rejects incompatible timestamp/read-tier inputs and
  references a SuperVersion before choosing an implicit snapshot sequence.
- `MergingIterator` maintains min/max heaps and has direction switching logic
  for forward/backward iteration.
- `BlockBasedTableIterator::SeekImpl` can check prefix filters before opening a
  data block. It then seeks index and data blocks and supports async second-pass
  initialization.
- `DBIter` turns internal-key order into user-key order. It skips overwritten
  versions, newer-than-snapshot records, tombstones, and range-deleted keys.
- Prefix seek depends on `prefix_extractor`, `prefix_same_as_start`, and table
  filter/index compatibility.
- Range deletion handling can reseek child iterators to skip covered ranges.

## Metrics

Use histogram `DB_SEEK`, PerfContext `seek_child_seek_time`,
`seek_min_heap_time`, `seek_max_heap_time`, `find_next_user_entry_time`,
`internal_key_skipped_count`, `internal_delete_skipped_count`,
`internal_recent_skipped_count`, and `internal_range_del_reseek_count`.

## Tests

Use `db/db_iter_test.cc`, `db/db_iterator_test.cc`,
`db/db_iter_stress_test.cc`, `db/multi_cf_iterator_test.cc`,
`table/merger_test.cc`, `table/block_based/block_based_table_iterator.cc`
indirect tests, `db/db_range_del_test.cc`, and `db/range_del_aggregator_test.cc`.
