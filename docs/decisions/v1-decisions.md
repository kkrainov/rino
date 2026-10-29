# v1 decisions

Status: settled on 2026-10-23, before the design doc is drafted. These are inputs to the future ADRs. Revisit one only if M0 results or upstream changes invalidate it.

## Context

This is an open-source, Rust-native, drop-in replacement for Trino, built on Apache DataFusion.

- **Target users:** teams running Trino on Iceberg/S3.
- **Goal: replace the JVM engine with a native one.** Trino is a mature engine, but some of its limits come from the JVM or from its single-coordinator architecture. Presto (Prestissimo) and Spark (Photon, Gluten, Comet) have already moved execution to native code. Rino goes further and makes coordination native too. The hypotheses, which M0 and later milestones measure:
  - *Lower cost:* more work per core and no JVM heap overhead.
  - *No GC pauses.*
  - *No coordinator bottleneck:* symmetric nodes (see Topology), so losing one node doesn't take the cluster down.
- **Interactive workloads, one of the main goals.** The same engine serves BI dashboards and AI-agent workloads, both of which send many small concurrent queries, so teams can retire the second engine they add for them (ClickHouse, StarRocks and similar). Symmetric nodes spread planning and coordination across the cluster, native execution keeps tail latency predictable under concurrency, and DataFusion is close to ClickHouse when both read the same Parquet files.
- **Implications for the design:**
  - *Workload isolation:* ETL and dashboard queries share nodes, so we need priorities, queues and memory limits per workload class, similar to Trino's resource groups.
  - *Caching:* interactive latency on Iceberg/S3 is bounded by object-storage reads, so we need to cache Iceberg metadata, Parquet footers and data. See "Iceberg integration and caching" below.
  - *"One engine" doesn't have to mean one cluster:* separate clusters of the same engine still remove the second engine.
- **Drop-in priorities, in order:** client protocol, SQL syntax, function coverage, identical results.
- **First gate:** M0, a public benchmark of DataFusion and datafusion-distributed against Trino's default in-memory execution. If the results don't justify the project, it stops. The benchmark lives in its own neutral repo, [iceberg-etl-bench](https://github.com/kkrainov/iceberg-etl-bench).

## Scope

- **v1 workloads:** interactive BI-style queries and light ETL. Heavy ETL stays on Trino with fault-tolerant execution (FTE) until v1.x. After v1.x, the engine covers both workloads.
- **Security:** enthusiast-level only. That means a trusted `X-Trino-User` header on private networks, plus optional TLS with a password file or JWT. Trino clients require TLS for password authentication.

## Execution and fault tolerance

- **Execution model:** in-memory and pipelined, built on datafusion-distributed.
- **Spilling:** wherever upstream DataFusion supports spilling to disk and the node has a local disk, we use it. This is the main way the in-memory pipelined mode handles memory pressure. On diskless nodes, and for operators that can't spill yet, the retry and memory-safety measures below apply. The fault-tolerant mode planned for v1.x and v2 will handle memory pressure separately.
- **Retries:** a failed query is retried once, transparently, as a whole.
  - A `SELECT` is retried only if no rows have reached the client yet.
  - DML is retried only through an idempotent commit keyed by query ID.
  - The retry can use a safer plan, for example a sort-merge join instead of a hash join after an out-of-memory error.
- **Memory safety:**
  - A memory limit per query.
  - Admission control per node.
  - A memory pool set below container memory. M0 measures how much headroom is needed.
  - A watchdog on jemalloc's real allocation numbers. It cancels the largest query before the kernel kills the whole process.
  - Recovery when DataFusion fails cleanly with `ResourcesExhausted`.
- **Spilling stays upstream:** we don't build our own operator spilling. We adopt DataFusion's spilling as it lands, and watch EPICs apache/datafusion#24768 (hash join) and #25637 (window, cross join, TopK). Our part is to configure spill directories when a node has a local disk, and to turn spilling off cleanly when it doesn't.

## Topology

- **Symmetric nodes:** every node accepts queries, and each query is owned by one node. This closes Trino's single-coordinator gap for new queries.
- **Query state:** kept in memory on the owning node. `nextUri` encodes the owner, and any other node proxies requests to it.
- **Worker discovery:** a pluggable `WorkerResolver`, with static, DNS or Kubernetes implementations. Redis isn't required.
- **Invariants we keep now so later versions don't force a rewrite:**
  - query IDs that don't depend on any node,
  - an attempt ID on every task,
  - an `Exchange` trait at stage boundaries,
  - query control state that can be serialized.

## Dependencies: what we use, contribute, or own

| Component | Decision |
|---|---|
| DataFusion | Depend on it and never fork it. Contribute fixes for generic SQL gaps and memory accounting. |
| datafusion-distributed | Adopt it for pipelined execution. |
| sqlparser-rs | Start `TrinoDialect` in our repo, then contribute it upstream. Fix generic syntax gaps upstream. |
| iceberg-rust | Depend on it for everything **except reading data files**: catalog clients, the metadata model, scan planning (manifest pruning into file tasks with their delete files), delete-file loading, data-file writers and transactions. Its Parquet data reader (`ArrowReader`) isn't used. Keep a **thin fork** that adds open PRs apache/iceberg-rust#2203 (RowDelta), #3300 (Overwrite) and #3046 (RewriteFiles), and review those PRs upstream so the fork can be dropped. |
| Iceberg table provider | **Our own**, with DataFusion-native reads. See "Iceberg integration and caching" below. datafusion-iceberg, datafusion-distributed-iceberg and JanKaul/iceberg-rust serve as code sources, not dependencies. |
| Row-level DML | Our own crate for `MERGE`, `DELETE` and `UPDATE`, built on iceberg-rust's writers and our fork's transaction actions; we don't write Parquet files or manifests from scratch. v1 uses **copy-on-write only**, but still reads the delete files Trino writes. Merge-on-read writes come in v1.x. Commits follow Iceberg's optimistic concurrency spec, including conflict validation for row-level operations. That validation lives in this crate or the fork, because upstream iceberg-rust doesn't have it yet. |
| Trino protocol | Our own. |
| Trino-only statements | Our own front-end layer before DataFusion: `SHOW`, `USE`, `SET/RESET SESSION`, `PREPARE … FROM`, `EXECUTE … USING`, `CALL`, `ALTER TABLE EXECUTE`, `CREATE TABLE … WITH`. |
| Trino functions | A **standalone crate** modeled on `datafusion-spark`, with every function checked against real Trino. We propose it upstream once it's mature. |
| Join ordering | Rely on upstream (PR apache/datafusion#24456) and revisit after M0. |

**Out of v1:** `MATCH_RECOGNIZE`, polymorphic table functions and SQL/JSON path clauses.

## Iceberg integration and caching

**Reads go through DataFusion's Parquet reader** (`DataSourceExec` with `ParquetSource`), not iceberg-rust's `ArrowReader`. Each step is owned by one component:

| Step | Component |
|---|---|
| Planning | iceberg-rust resolves the snapshot, prunes manifests and returns file tasks, each with its delete files. |
| Execution | DataFusion scans those files as file groups. |
| Writes | Our DML crate, then iceberg-rust's writers, then our fork's transaction actions and conflict validation, then the catalog commit. |

Why DataFusion's reader (verified in DataFusion 55.1 and iceberg-rust main source):

| Capability | DataFusion reader | iceberg-rust reader |
|---|---|---|
| Runtime (dynamic) filters from join build sides, TopK and aggregates | Prune files and row groups mid-query | Not supported |
| Page-index pruning and bloom filters | On by default | Off by default |
| Parquet footer cache | Built in | None; the footer is re-read on every file open |
| Parallelism | File groups, byte-range splits, work-stealing scans | A single stream per scan in datafusion-iceberg |
| Distribution in datafusion-distributed | Scans split across workers natively | Needs a custom leaf |
| Future improvements | DataFusion's Parquet reader is where its performance work lands | — |

Precedent: JanKaul/iceberg-rust already reads Iceberg this way on DataFusion 55.

**What we own on the read path**, all through public DataFusion extension points with no upstream changes:

- Column mapping by Iceberg field ID, plus name mapping for files without IDs, via `PhysicalExprAdapterFactory`.
- Position deletes and deletion vectors as per-file row selections (`ParquetAccessPlan`).
- Equality deletes as an anti-join.
- Partition values, missing columns and default values.
- A plan serializer for datafusion-distributed. A file scan that uses a custom column adapter can't be serialized by default.

These are correctness risks. We test them against iceberg-rust's own reader and against Trino.

**Caching** integrates existing components through public interfaces. There is no custom storage and no upstream change:

| Cache layer | Component |
|---|---|
| Data bytes | A `CachingObjectStore` wrapper backed by foyer: a memory tier, plus an SSD tier when the node has a local disk. Registered with DataFusion's `ObjectStore` registry. |
| Parquet footers and page indexes | DataFusion's `FileMetadataCache`, which can be replaced to share the same memory budget. |
| Iceberg metadata (manifests and manifest lists) | A custom iceberg-rust `StorageFactory`, built on the same `object_store` and cache. |

The result is one I/O stack, one cache and one set of credentials. Iceberg files never change once written, so file path plus byte range is a permanent cache key. Nothing needs invalidating except the catalog's pointer to the current metadata, and that pointer is never cached.

**Upgrades:** pinned release trains. We move one DataFusion major at a time, in a single coordinated upgrade with datafusion-distributed and the Iceberg crates.

## Deferred

- **v1.x:**
  - Materialized fault-tolerant mode, with the exchange on S3, task retries, and the engine choosing the mode per query from history and Iceberg statistics.
  - Where that mode's scheduler lives: contributed upstream to datafusion-distributed, built on Ballista, or our own.
  - Merge-on-read writes.
- **v2:** coordinator takeover for materialized queries, and selective materialization.
- **Watch item:** datafusion-distributed hard-codes which plan nodes count as network boundaries (`network_boundary.rs`). Materialized mode needs that to be pluggable, so raise it with the maintainers early.
