# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

# Resources

- https://docs.daft.ai for the user-facing API docs
- CONTRIBUTING.md for detailed development process
- https://github.com/Eventual-Inc/Daft for issues, discussions, and PRs

# Dev Workflow

1. [Once] Set up Python environment and install dependencies: `make .venv`
   - Requires `uv` and `rustup` to be installed. On M1 Macs, set `IS_M1=1`.
2. [Optional] Activate .venv: `source .venv/bin/activate`. Not necessary with Makefile commands.
3. If Rust code is modified, rebuild: `make build` (debug) or `make build-release` (optimized).
4. Run tests. See [Testing Details](#testing-details).

# Testing Details

- `make test` runs tests in `tests/` directory. Uses `pytest` under the hood.
  - Must set `DAFT_RUNNER` environment variable to `ray` or `native` to run the tests with the corresponding runner.
    - Start with `DAFT_RUNNER=native` unless testing Ray or distributed code.
  - `make test EXTRA_ARGS="..."` passes additional arguments to `pytest`.
    - `make test EXTRA_ARGS="-v tests/dataframe/test_select.py"` runs the test in the given file.
    - `make test EXTRA_ARGS="-v tests/dataframe/test_select.py::test_select_dataframe"` runs the given test method.
  - Default `integration`, `benchmark`, and `hypothesis` tests are disabled. Best to run on CI.
- `make doctests` runs doctests in `daft/` directory. Tests docstrings in Daft APIs.

# Linting and Formatting

- `make lint` — runs `ruff` (Python) and `clippy` (Rust) via pre-commit.
- `make format` — runs `ruff-format` (Python) and `rustfmt` (Rust) via pre-commit.
- `make check-format` — checks without modifying files.
- `make precommit` — runs all pre-commit hooks.

# PR Conventions

- Titles: Conventional Commits format; enforced by `.github/workflows/pr-labeller.yml`.
- Descriptions: follow `.github/pull_request_template.md`.
- Non-trivial PRs must link an approved GitHub issue.

# Architecture Overview

Daft is a high-performance distributed dataframe engine. The codebase is a Python library with a Rust core, bridged via [PyO3](https://pyo3.rs/) and compiled with [maturin](https://github.com/PyO3/maturin). The compiled Rust extension is exposed as `daft.daft` (the `daft.abi3.so` shared library).

## Query Execution Pipeline

A Daft query goes through these stages:

1. **Python API** (`daft/dataframe/dataframe.py`) — User-facing `DataFrame` class. Operations are lazy; they build a logical plan without executing.
2. **Logical Plan** (`src/daft-logical-plan/`) — Rust tree of logical operators (Filter, Project, Join, Agg, Scan, Sink, etc.) in `src/daft-logical-plan/src/ops/`. The `LogicalPlanBuilder` is the primary construction API.
3. **Optimizer** (`src/daft-logical-plan/src/optimization/`) — Rule-based optimizer rewrites the logical plan (predicate pushdown, projection pruning, join reordering, etc.).
4. **Physical Plan Translation** — Translates the optimized logical plan to either:
   - `LocalPhysicalPlan` (`src/daft-local-plan/`) for the **native runner**
   - A distributed pipeline (`src/daft-distributed/`) for the **Ray runner**
5. **Execution**:
   - **Native runner**: `src/daft-local-execution/` — async Rust pipeline executor with sources, intermediate operators (streaming), and sinks (blocking). Runs locally with threads.
   - **Ray runner**: `src/daft-distributed/` — schedules tasks on Ray workers via Python callbacks; uses the same local execution engine on each worker.

## Key Rust Crates

| Crate | Purpose |
|---|---|
| `daft-core` | Core data types, `DataArray`, `Series`, Arrow-backed column arrays |
| `daft-schema` | Schema and `DataType` definitions |
| `daft-dsl` | Expression DSL (`Expr` enum, expression evaluation) |
| `daft-recordbatch` | `RecordBatch` — a collection of named `Series` (single partition) |
| `daft-micropartition` | `MicroPartition` — the runtime unit of data (wraps record batches) |
| `daft-logical-plan` | Logical plan tree + optimizer |
| `daft-local-plan` | Physical plan for native execution |
| `daft-local-execution` | Native async streaming execution engine |
| `daft-distributed` | Distributed execution engine (Ray integration) |
| `daft-scan` | Scan infrastructure: `ScanTask`, file listing, pushdowns |
| `daft-io` | I/O layer: S3, GCS, Azure, HTTP, local file reads |
| `daft-parquet`, `daft-csv`, `daft-json` | Format-specific readers/writers |
| `daft-sql` | SQL parser and planner (builds logical plans from SQL strings) |
| `daft-catalog` | Catalog and session management |
| `daft-functions` / `daft-functions-*` | Built-in scalar functions (utf8, json, list, temporal, etc.) |
| `daft-runners` | Runner abstraction and Python bindings for runner selection |
| `common/` | Shared utilities: error types, runtime, IO config, display, tracing |

## Python Package Structure

```
daft/
  __init__.py         # Top-level public API re-exports
  dataframe/          # DataFrame and GroupedDataFrame classes
  expressions/        # Python Expression wrappers
  series.py           # Series (single column) Python wrapper
  datatype.py         # DataType Python wrapper
  io/                 # Read/write functions (parquet, csv, json, iceberg, delta, etc.)
  udf/                # UDF decorators (@udf, @func, @cls, @method)
  sql/                # sql() and sql_expr() entry points
  catalog/            # Catalog, Table, Identifier Python types
  session.py          # Session management
  runners/            # Runner configuration (native/ray)
  functions/          # Python-callable built-in functions
  ai/                 # AI integrations (OpenAI, Google, Transformers, vLLM)
  daft.abi3.so        # Compiled Rust extension (generated by maturin)
```

## Adding a New Built-in Function

The typical pattern for adding a Rust-backed function:
1. Implement the core logic in a `daft-functions-*` crate (or `daft-functions` for simpler cases).
2. Register the function's `ScalarUDF` implementation.
3. Add a Python-callable wrapper in `daft/functions/` or as an `Expression` method.
4. Expose via `register_modules` in the relevant Rust crate's `python.rs`.

## Extension System

`src/daft-ext/` provides a stable ABI for building external Daft extensions (plugins) in Rust without recompiling Daft itself. Uses `daft-ext-abi`, `daft-ext-core`, and `daft-ext-macros`.
