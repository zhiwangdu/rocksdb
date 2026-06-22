# LSM Research: Tombstones, Range Deletion, Merge, Prefix, Column Family

## Tombstone Lifecycle

Point deletes and single deletes are written as internal key types through the
normal write path. They hide older values in reads and are eventually dropped by
`CompactionIterator` when snapshot and lower-level constraints allow.

## Range Deletion

`DeleteRange` writes range tombstones into a separate memtable table and SST
range deletion block. `MemTable::Add` uses `range_del_table_` for
`kTypeRangeDeletion`. Range tombstones are fragmented by
`RangeTombstoneFragmenter` and queried by `RangeDelAggregator`.

Point lookups compare max covering tombstone sequence. Iterators use
range-deletion-aware reseek/skip logic. Compaction can drop obsolete range
tombstones and records `COMPACTION_RANGE_DEL_DROP_OBSOLETE`.

## Merge Operator

Merge operands are stored as internal key records. Reads accumulate operands in
`MergeContext` and call merge operators through `MergeHelper`. Compaction can
resolve merges when safe. High merge operand depth increases read and iterator
latency and appears in PerfContext merge counters.

## Prefix Extractor

Prefix extractor affects memtable bloom, SST prefix filters, hash index, and
prefix seek. It must be compatible with the comparator and workload. Mismatched
prefix assumptions can make filters ineffective or force total-order fallback.

## Column Family Effects

Column families isolate memtables, versions, options, and compaction pickers,
but share global WAL and sequence numbers. A slow-flushing CF can retain WALs
and affect DB-wide recovery/open time. Atomic flush can commit several CF flush
results together.
