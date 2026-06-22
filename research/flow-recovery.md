# Flow Research: Recovery Path

## Scope

This flow covers DB open, CURRENT, MANIFEST, VersionSet recovery, WAL replay,
memtable rebuild, sequence restoration, corruption handling, missing files, and
`paranoid_checks`.

## Source Evidence

| Step | Source |
|---|---|
| Open/recovery | `db/db_impl/db_impl_open.cc` |
| Manifest recovery | `db/version_set.cc`, `db/version_edit.cc`, `db/version_edit_handler.cc` |
| WAL reader | `db/log_reader.cc`, `db/log_format.h` |
| Batch replay | `db/write_batch.cc` |
| Memtable | `db/memtable.cc`, `db/memtable_list.cc` |
| Tests | `db/corruption_test.cc`, `db/fault_injection_test.cc`, `db/repair_test.cc`, `db/db_wal_test.cc` |

## Call Chain

`DB::Open`
-> `DBImpl::Recover`
-> lock DB directory
-> find CURRENT or manifest in best-efforts mode
-> `VersionSet::Recover`
-> read MANIFEST with `log::Reader`
-> `VersionEdit::DecodeFrom`
-> load current `Version`
-> collect WAL numbers
-> `DBImpl::RecoverLogFiles`
-> `DBImpl::ProcessLogFile`
-> `log::Reader::ReadRecord`
-> `WriteBatchInternal::InsertInto`
-> rebuild memtables
-> flush recovered memtables if needed
-> set last sequence and resume normal operation.

## Corruption and Missing Files

- `wal_recovery_mode` controls tolerance for corrupt tail records or skipped
  records.
- `best_efforts_recovery` can scan manifest files rather than relying only on
  CURRENT.
- `VersionSet::Recover` can retry manifest reads using filesystem reconstruct
  support when corruption is suspected.
- Missing SST files generally fail recovery unless best-efforts/no-error modes
  are explicitly in use.
- `paranoid_checks` increases strictness around detected corruption.

## Sequence Number Recovery

MANIFEST stores last sequence, next file number, log number, previous log
number, and min log number to keep. WAL replay observes batch sequence numbers
and advances `next_sequence`. The final recovered sequence must be at least the
highest observed committed write.

## Metrics and Logs

Use DB open LOG lines, recovery event-log entries (`recovery_started`), WAL
corruption retry tickers, file read histograms under `FILE_READ_DB_OPEN_MICROS`,
and `IOStatsContext` read/open counters.

## Tests

Use `db/corruption_test.cc`, `db/fault_injection_test.cc`,
`db/repair_test.cc`, `db/db_wal_test.cc`, `db/db_log_iter_test.cc`,
`db/version_set_test.cc`, and `utilities/transactions/write_prepared_transaction_seqno_test.cc`.
