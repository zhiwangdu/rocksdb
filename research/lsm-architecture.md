# LSM Research: Overall Structure and File Lifecycle

## Source Evidence

Core sources are `db/db_impl/*`, `db/version_set.*`, `db/version_edit.*`,
`db/memtable.*`, `db/flush_job.cc`, `db/compaction/*`,
`table/block_based/*`, and `db/table_cache.*`.

## LSM Structure

RocksDB organizes each column family as:

mutable memtable
-> immutable memtable list
-> L0 overlapping SST files
-> L1+ non-overlapping leveled SST files
-> optional blob files for separated values.

`SuperVersion` is the read-side handle to that complete state. `Version` owns
the SST/blobs view. `VersionSet` owns all current versions and MANIFEST state.

## MemTable to SST

Foreground writes insert into the mutable memtable. When flush is needed,
`DBImpl::SwitchMemtable` makes it immutable. `FlushJob::WriteLevel0Table`
builds an L0 SST from one or more immutable memtables using `TableBuilder`.
`MemTableList::TryInstallMemtableFlushResults` commits the new file through
`VersionSet::LogAndApply`.

## SST Lifecycle

1. Created by flush, compaction, ingestion, or repair.
2. Validated and fsynced according to file options.
3. Added to MANIFEST with `VersionEdit`.
4. Read by table cache/table readers through a `Version`.
5. Selected as compaction input or deleted by FIFO/obsolete cleanup.
6. Removed from live metadata by a later `VersionEdit`.
7. Physically deleted after no snapshots/iterators/jobs require it.

## Manifest and Version Management

MANIFEST is an append-only log of `VersionEdit`. CURRENT points to the active
MANIFEST. Recovery reads CURRENT then replays the MANIFEST to reconstruct
`VersionSet`, file metadata, and sequence/file-number state.

## Column Families

Each column family has independent memtables, versions, options, statistics,
and compaction picker, but shares DB-level WAL, sequence number namespace,
thread pools, rate limiter, and Env/FileSystem.
