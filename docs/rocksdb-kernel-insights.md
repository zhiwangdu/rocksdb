# RocksDB Kernel Insights

This document summarizes RocksDB from a database-kernel development viewpoint.

## 1. Why RocksDB Is Designed This Way

RocksDB optimizes for write-heavy storage with acceptable read cost by using an
LSM. Writes append to WAL and memtable first, then background flush/compaction
turn memory into immutable sorted files and maintain the LSM shape. This trades
random in-place updates for sequential writes and background rewrite work.

Source anchors:

- Write/WAL/memtable: `db/db_impl/db_impl_write.cc`, `db/write_batch.cc`,
  `db/memtable.cc`, `db/log_writer.cc`
- LSM metadata: `db/version_set.cc`, `db/version_edit.cc`
- Table format: `table/block_based/*`
- Compaction: `db/compaction/*`

## 2. Designs Worth Borrowing

| Design | Why it works | Source |
|---|---|---|
| SuperVersion | Readers avoid DB mutex while preserving object lifetime. | `db/column_family.h`, `db/column_family.cc` |
| WAL-before-memtable | Simple crash recovery invariant. | `db/db_impl/db_impl_write.cc`, `db/db_impl/db_impl_open.cc` |
| MANIFEST as metadata log | File membership changes are atomic and replayable. | `db/version_set.cc`, `db/version_edit.cc` |
| Internal keys | User key plus sequence/type gives MVCC-like snapshot visibility. | `db/dbformat.h` |
| Block cache below table reader | One cache interface serves data/index/filter blocks. | `include/rocksdb/cache.h`, `table/block_based/block_cache.cc` |
| Compaction iterator | Centralizes obsolete-version, tombstone, merge, filter, and blob-GC logic. | `db/compaction/compaction_iterator.cc` |
| EventListener and metrics | Operational hooks are first-class without coupling core logic to one monitoring system. | `include/rocksdb/listener.h`, `include/rocksdb/statistics.h` |

## 3. Highest-Complexity Areas

- `DBImpl::WriteImpl`: write modes, callbacks, WAL sync, sequence publication,
  blob direct write, memtable insertion, and error paths meet in one hot path.
- `VersionSet::LogAndApply`: metadata correctness, manifest IO, version
  builder, and writer queueing.
- `CompactionIterator`: snapshot correctness, tombstones, merges, blob GC, and
  user compaction filters.
- `DBIter` and `MergingIterator`: range scans must hide internal complexity
  while remaining fast.
- Recovery: must reconcile MANIFEST, WAL, column families, missing files,
  corruption policy, and transaction state.

## 4. Modules Most Likely to Introduce Bugs

| Module | Common bug class |
|---|---|
| Write path | Sequence publication before visibility, invalid option combinations, partial memtable insert after WAL success. |
| VersionSet/MANIFEST | File-number reuse, missing file deletion/addition, corrupt recovery compatibility. |
| CompactionIterator | Dropping data visible to snapshots, mishandling range tombstones or merge operands. |
| Iterator | Direction-switch bugs, bounds/prefix bugs, stale pinned resources. |
| Cache/filter | Incorrect cache keys, stale table options, filter false-negative bugs. |
| Recovery | WAL replay ordering, corrupt-tail policy mistakes, missing-file handling. |
| Utilities | Checkpoint/backup live-file races, ingestion overlap/sequence mistakes. |

## 5. Mini-LSM Minimum Closed Loop

For a small educational LSM, implement:

1. Internal key format: user key, sequence, type.
2. WriteBatch plus WAL append and replay.
3. Mutable skiplist memtable and immutable memtable list.
4. Flush to sorted SST with data block, index, filter, footer.
5. Manifest with add/delete file edits and CURRENT pointer.
6. Version object with L0 overlap and L1+ non-overlap search.
7. Point Get with snapshot sequence filtering.
8. Iterator over memtables and SSTs with merge heap.
9. Basic leveled compaction and tombstone cleanup.
10. Tests for crash recovery after every file/manifest/WAL boundary.

## 6. Time-Series Database Considerations

If using or adapting RocksDB for time-series workloads, focus on:

- key schema and comparator so time ranges are contiguous;
- prefix extractor for series ID to make filters useful;
- block size and compression for scan-heavy workloads;
- FIFO or leveled compaction depending on retention semantics;
- tombstone/range deletion volume for TTL deletes;
- cache partitioning to avoid scans evicting hot point-read blocks;
- ingest/external SST for bulk backfill;
- snapshot/backup retention so old data is not kept unexpectedly.

## 7. Cassandra / openGemini Comparison Dimensions

Compare along these axes:

- write path durability: WAL/commitlog, memtable model, batching;
- LSM metadata: MANIFEST/VersionSet versus system tables/metadata;
- compaction strategies: leveled/universal/FIFO versus size-tiered/time-window;
- query model: embedded KV engine versus distributed storage/query engine;
- replication and consensus: outside RocksDB, native in distributed systems;
- range scan/index model: key-ordered SST versus time-series partition/index;
- observability: RocksDB tickers/PerfContext/LOG versus cluster metrics;
- operational failure domains: local DB recovery versus cluster repair/rebalance.

## 8. Engineering Compromises

RocksDB often chooses practical engineering over theoretical simplicity:

- multiple write modes complicate `WriteImpl` but support different latency and
  transaction tradeoffs;
- universal/FIFO/leveled compaction coexist because no one strategy wins every
  workload;
- filters and cache policies have many options because workloads differ;
- MANIFEST append is simple and robust but can become large and slow to replay;
- snapshots are cheap but can retain old versions and space;
- EventListener callbacks are flexible but must be kept cheap by users;
- Env/FileSystem abstraction is more complex than direct POSIX calls but is
  necessary for portability and custom storage.

## 9. Operations Summary

For write issues, start with WAL sync time, write stall counters, immutable
memtable count, L0 file count, pending compaction bytes, and background jobs.

For read issues, start with block cache hit rate, Bloom usefulness, L0 hit
counts, PerfContext block read time, and iterator skipped-key counters.

For recovery issues, start with MANIFEST size/replay logs, WAL count, last
sequence, missing file messages, and corruption status.

For space issues, start with snapshots, obsolete files, compaction backlog,
range tombstones, backups/checkpoints, and blob garbage.
