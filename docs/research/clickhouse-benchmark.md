# ClickHouse vs TimescaleDB Benchmark

This document records an experimental comparison for the StellarView indexer. It does not recommend adoption, hybrid deployment, or migration.

## Reproduction details

- Git commit: `fe1f01a135e4b69dca35fc85b4a8db415a2bf85`
- Date: 2026-08-22
- Fixed range: pubnet ledgers 100 through 104, 5 ledgers
- Source: `s3://aws-public-blockchain/v1.1/stellar/ledgers/pubnet`, `us-east-2`
- Manifest: [`benchmark/fixtures/pubnet-range-manifest.json`](../../benchmark/fixtures/pubnet-range-manifest.json)
- Command: `./benchmark/scripts/run.sh run`
- Host: Linux x86_64, 4 CPUs, 7.6 GiB RAM, Go 1.18.1
- PostgreSQL/TimescaleDB: `timescale/timescaledb:2.17.2-pg16`
- ClickHouse: `clickhouse/clickhouse-server:25.3`
- Resource limits: not set; local Docker defaults
- Batch size: S3 path processes one ledger per worker; worker count defaults to 1
- Concurrency: 1 worker
- Cache state: ingestion is a fresh database run; query cache state is not controlled
- Repetitions: 1 until a Docker-capable host produces repeated raw results

## Scope and schema mapping

The active Go writer populates `ledgers`, `transactions`, `operations`, `token_events`, and `contract_events`. Accounts are defined by migrations but are not populated by the current pipeline, so account-by-ID is recorded as unavailable rather than fabricated.

| Logical entity | PostgreSQL/TimescaleDB | ClickHouse | Primary/order key | Dedup strategy |
|---|---|---|---|---|
| Ledgers | `ledgers` hypertable | `stellar_benchmark.ledgers` | `(sequence, closed_at)` | `ReplacingMergeTree(version)` |
| Transactions | `transactions` hypertable | `stellar_benchmark.transactions` | `(hash, created_at, id)` | `ReplacingMergeTree(version)`, raw and `FINAL` |
| Operations | `operations` hypertable | `stellar_benchmark.operations` | `(transaction_hash, application_order, created_at, id)` | `ReplacingMergeTree(version)`, raw and `FINAL` |
| Token events | `token_events` hypertable | `stellar_benchmark.token_events` | `(transaction_hash, ledger_sequence, created_at)` | `MergeTree` |
| Contract events | `contract_events` hypertable | `stellar_benchmark.contract_events` | `(contract_id, created_at, transaction_hash)` | `MergeTree` |

## Results

Measured values are intentionally not estimated. The checked-in environment did not provide Docker, so no database measurement has been generated in this checkout. The runner writes these values to `benchmark/results/` when executed on a host with the listed prerequisites.

| Metric | PostgreSQL/TimescaleDB | ClickHouse | Notes |
|---|---:|---:|---|
| Stored bytes | pending raw result | pending raw result | PostgreSQL `pg_database_size`; ClickHouse active parts |
| Uncompressed bytes | pending raw result | pending raw result | Definitions differ and are reported separately |
| Compression ratio | pending raw result | pending raw result | `uncompressed_size / stored_size` |
| Backfill ledgers/sec | pending raw result | pending raw result | Fresh run; raw per-run JSON |
| Transaction lookup p50/p95 | pending raw result | pending raw result | Direct SQL workload definition |
| Account lookup p50/p95 | unavailable | unavailable | No populated account table in current writer |
| Account history p50/p95 | pending fixture | pending fixture | Requires a populated account fixture; not present in active path |
| Transaction aggregate p50/p95 | pending raw result | pending raw result | Hour buckets over loaded range |
| Operation aggregate p50/p95 | pending raw result | pending raw result | Hour buckets over loaded range |
| Top assets p50/p95 | pending raw result | pending raw result | Asset ranking over operation rows |

## Deduplication results

The ClickHouse transaction and operation tables use `ReplacingMergeTree(version)`. Replaying rows is expected to increase raw row counts before background merges. The workload definitions include both ordinary reads and `FINAL`; the harness must record raw count, `FINAL` count, and post-`OPTIMIZE ... FINAL` count and latency. No result is claimed here until that experiment is run.

## Observed trade-offs

### ClickHouse observations

- The benchmark schema requires explicit `ORDER BY` choices for hash lookup, account history, and time ordering.
- Correctness-preserving reads using `FINAL` are a separate workload and must not be hidden behind raw-query timings.
- Local setup requires a second database service and a separate schema lifecycle.

### PostgreSQL/TimescaleDB observations

- The production path already has migration definitions, hypertables, indexes, and a reusable S3 backfill command.
- The current insert path uses separate transactions for ledger, transaction, operation, and event batches; this is measured behavior, not changed by the harness.
- Explorer-facing query handlers are not present in this repository, so production query semantics could not be reused directly.

### Operational observations

- Docker was unavailable in the recording environment, preventing measured database output.
- Reproduction requires public S3 access, Docker Compose, `psql`, and `curl`.

## Limitations

- Five ledgers are a small smoke-scale range and may not represent sustained pubnet backfill behavior.
- The experiment is single-node and has no production concurrency model.
- Network transfer and source preparation are recorded separately from database loading only where the runner can observe them.
- Cache state and OS-level resource contention are not fully controlled.
- Accounts and other migration-only tables are not populated by the current indexer, so those requested workloads remain unavailable rather than using synthetic data.
- Query timings and storage numbers must be generated on a Docker-capable host before they can be interpreted.

There is no go/no-go conclusion in this research note.
