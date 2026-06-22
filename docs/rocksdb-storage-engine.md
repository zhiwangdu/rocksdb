# RocksDB Storage Engine

This document explains the LSM implementation: memtables, WAL, SSTs, VersionSet,
Manifest, compaction strategies, tombstones, amplification, filters, cache,
prefix extractors, merge operators, and column families.

## 1. LSM Shape

Per column family, RocksDB data flows through:

`mutable MemTable -> immutable MemTable list -> L0 SSTs -> L1+ SSTs -> optional blob files`.

Source evidence:

- `db/memtable.h`, `db/memtable.cc`
- `db/memtable_list.h`, `db/memtable_list.cc`
- `db/flush_job.cc`
- `db/version_set.h`, `db/version_set.cc`
- `table/block_based/block_based_table_builder.cc`
- `table/block_based/block_based_table_reader.cc`

L0 files can overlap. L1+ files are expected to be non-overlapping within each
level for leveled compaction, which lets `Version::Get` reduce lookup work with
file indexes and binary search.

## 2. MemTable, WAL, and Sequence Numbers

`DBImpl::WriteImpl` writes the WAL before inserting the memtable in the normal
path. `WriteBatchInternal::InsertInto` replays batch records into
`MemTable::Add`. `MemTable::Add` encodes internal key bytes as user key plus an
8-byte packed sequence/type suffix, followed by value bytes and optional
protection bytes.

The sequence number namespace is DB-wide, not per column family. Snapshots use
sequence comparisons to decide whether internal keys are visible.

## 3. SST Format

The default block-based SST contains:

- data blocks with internal key/value records;
- index blocks mapping separator keys to data block handles;
- optional full or partitioned filter blocks;
- properties and metaindex blocks;
- optional range deletion block;
- footer and checksums.

Builder evidence is in `BlockBasedTableBuilder::Add`, `WriteBlock`,
`WriteFilterBlock`, `WriteIndexBlock`, and `Finish` in
`table/block_based/block_based_table_builder.cc`. Reader evidence is in
`BlockBasedTable::Open`, `BlockBasedTable::Get`, and
`BlockBasedTableIterator::SeekImpl`.

## 4. Version and Manifest Management

`VersionSet` is the metadata owner. Every flush, compaction, ingestion, column
family add/drop, and WAL metadata change is represented as a `VersionEdit`.
`VersionSet::LogAndApply` serializes edit application, writes the encoded edit
to MANIFEST with `log::Writer::AddRecord`, and publishes a new `Version`.

Recovery reads CURRENT to find MANIFEST, replays `VersionEdit::DecodeFrom`, and
then replays WALs to rebuild unflushed memtables.

## 5. Compaction Strategies

| Strategy | Source | Core behavior |
|---|---|---|
| Level | `db/compaction/compaction_picker_level.cc` | Uses compaction scores per level and maintains non-overlap in L1+. Best default for bounded read and space amp. |
| Universal | `db/compaction/compaction_picker_universal.cc` | Treats files/levels as sorted runs and compacts by size/age/ratio. Useful for write-heavy workloads that tolerate higher temporary space amp. |
| FIFO | `db/compaction/compaction_picker_fifo.cc` | Deletes old/excess files by size/TTL, with optional intra-L0 compaction. Useful for cache/time-window workloads. |

`CompactionJob::ProcessKeyValueCompaction` builds an input iterator, runs
`CompactionIterator`, writes output SSTs, then installs results by deleting
inputs and adding outputs in one `VersionEdit`.

## 6. Tombstones and Range Deletions

Point deletes and single deletes are internal key types inserted through the
normal write path. Range deletions use `kTypeRangeDeletion`, are stored in a
range deletion memtable/table block, and are fragmented for lookup/iteration.

Source evidence:

- `MemTable::Add` in `db/memtable.cc`
- `db/range_del_aggregator.cc`
- `db/range_tombstone_fragmenter.cc`
- `CompactionIterator` in `db/compaction/compaction_iterator.cc`

Tombstones are not free. They improve delete latency but can increase read and
space amplification until compaction proves older data is obsolete.

## 7. Amplification Control

| Amplification | Main causes | Controls | Evidence / metrics |
|---|---|---|---|
| Read amp | L0 overlap, many levels/runs, bad filters, cache misses, tombstones | L0 triggers, compaction style, Bloom/Ribbon filters, block cache, prefix extractor | `Version::Get`, `BlockBasedTable::Get`, `BLOOM_FILTER_*`, `BLOCK_CACHE_*` |
| Write amp | Repeated rewriting through levels, small targets, universal run merges, blob GC | `target_file_size_base`, `max_bytes_for_level_base`, dynamic level bytes, compaction style | `CompactionJob`, `COMPACT_READ_BYTES`, `COMPACT_WRITE_BYTES` |
| Space amp | Snapshots, tombstones, compaction backlog, universal temporary overlap, backup/checkpoint retention | Snapshot discipline, compaction capacity, bottommost compaction, obsolete cleanup | `VersionStorageInfo`, `CompactionIterator`, live file/obsolete file logs |

## 8. Cache and Filters

Block cache sits below table readers and can cache data, index, filter, and
compression dictionary blocks. `BlockBasedTable::UpdateCacheHitMetrics` and
`UpdateCacheMissMetrics` record per-block-type counters. Bloom filters are
queried in `BlockBasedTable::FullFilterKeyMayMatch` before index/data block
lookup.

Operational signals:

- `BLOCK_CACHE_HIT`, `BLOCK_CACHE_MISS`, `BLOCK_CACHE_DATA_*`,
  `BLOCK_CACHE_INDEX_*`, `BLOCK_CACHE_FILTER_*`
- `BLOOM_FILTER_USEFUL`, `BLOOM_FILTER_FULL_POSITIVE`,
  `BLOOM_FILTER_PREFIX_USEFUL`
- PerfContext `block_read_time`, `read_filter_block_nanos`,
  `read_index_block_nanos`

## 9. Prefix and Merge Effects

`prefix_extractor` affects memtable bloom, SST filters, prefix seeks, and hash
index behavior. The extractor must be compatible with comparator ordering and
the workload's query shape.

Merge operands are accumulated through `MergeContext` and resolved by
`MergeHelper`. Deep merge stacks increase point-read and iterator cost until
compaction resolves them.

## 10. Column Family Effects

Column families isolate LSM state and options, but share the WAL, sequence
number namespace, DB thread pools, rate limiter, and filesystem. A single slow
column family can retain WAL files and increase DB open recovery time. Atomic
flush changes flush publication across column families but does not change the
global write sequence model.
