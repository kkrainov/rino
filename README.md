# Rino

Rino ("Trino without the T; R for Rust") is a working name for a planned open-source,
Rust-native, drop-in replacement for [Trino](https://trino.io), built on
[Apache DataFusion](https://datafusion.apache.org). The goal is one engine for heavy ETL
and interactive BI-style queries on Iceberg, compatible with Trino's client protocol,
SQL, functions and results.

> **Status: research.** There is no engine code yet. The first milestone, M0, is a public
> benchmark of the foundation (DataFusion + datafusion-distributed) against Trino's default
> in-memory execution: [iceberg-etl-bench](https://github.com/kkrainov/iceberg-etl-bench).
> If the results don't justify the project, it stops.

## Documents

- [v1 decisions](docs/decisions/v1-decisions.md): the scope, architecture and dependency
  choices settled during the research phase. They are inputs to the design doc and ADRs.

## Feedback

Questions, objections and pointers to prior work are welcome as GitHub issues.

## Licence

Apache-2.0.
