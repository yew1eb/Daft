# Daft - AI Coding Agent Guide

> **What is Daft?** Daft is a high-performance data engine for AI and multimodal workloads. It processes images, audio, video, and structured data at any scale with a Python-native API and Rust-powered execution engine.

- **Website**: https://www.daft.ai
- **Documentation**: https://docs.daft.ai
- **Repository**: https://github.com/Eventual-Inc/Daft
- **Issue Tracker**: https://github.com/Eventual-Inc/Daft/issues
- **Discussions**: https://github.com/Eventual-Inc/Daft/discussions

---

## Technology Stack

### Core Architecture
Daft is a **hybrid Rust/Python project**:

- **Rust (src/)**: Core execution engine, data structures, I/O, query planning/optimization
- **Python (daft/)**: User-facing API, UDF framework, integrations, catalog connectors
- **Binding**: PyO3 for Rust-Python interop, built as a Python extension module (`cdylib`)

### Key Dependencies

**Rust:**
- Arrow/Parquet: Custom forks (`src/arrow2`, `src/parquet2`) + Apache Arrow 57.1.0
- Async runtime: Tokio
- Python interop: PyO3 0.27.2
- Serialization: bincode, serde
- Storage: OpenDAL for object storage (S3, GCS, Azure)

**Python:**
- PyArrow (8.0.0 to 24.0.0 supported)
- fsspec for filesystem abstraction
- Ray for distributed execution (optional)
- Various catalog integrations: pyiceberg, deltalake, lance

### Build Tools
- **uv**: Python package manager and virtual environment (required)
- **maturin**: Build Rust extension for Python
- **cargo**: Rust package manager
- **npm**: For dashboard frontend (Node.js 22)

---

## Project Structure

```
.
├── src/                          # Rust source code
│   ├── common/                   # Shared utilities (logging, config, error handling)
│   ├── arrow2/                   # Fork of arrow2 crate
│   ├── parquet2/                 # Fork of parquet2 crate
│   ├── daft-core/                # Core data structures (Series, DataType, arrays)
│   ├── daft-dsl/                 # Domain-specific language (expressions)
│   ├── daft-logical-plan/        # Query logical planning
│   ├── daft-local-plan/          # Local execution planning
│   ├── daft-local-execution/     # Native execution engine
│   ├── daft-distributed/         # Distributed execution (Ray)
│   ├── daft-io/                  # I/O operations (S3, GCS, Azure, local)
│   ├── daft-parquet/             # Parquet reading/writing
│   ├── daft-csv/                 # CSV reading/writing
│   ├── daft-json/                # JSON reading/writing
│   ├── daft-sql/                 # SQL parser and execution
│   ├── daft-catalog/             # Catalog abstractions
│   ├── daft-scan/                # Data source scanning
│   ├── daft-functions*/          # Expression functions (UTF8, JSON, temporal, etc.)
│   ├── daft-image/               # Image processing
│   ├── daft-micropartition/      # Partition data structure
│   ├── daft-recordbatch/         # RecordBatch abstractions
│   ├── daft-runners/             # Runner abstractions (native, Ray)
│   ├── daft-context/             # Execution context
│   ├── daft-session/             # Session management
│   ├── daft-dashboard/           # Web dashboard (Rust backend + Next.js frontend)
│   ├── daft-cli/                 # Command-line interface
│   └── lib.rs                    # Main library entry point
│
├── daft/                         # Python source code
│   ├── __init__.py               # Public API exports
│   ├── dataframe/                # DataFrame implementation
│   ├── expressions/              # Expression API
│   ├── functions/                # Built-in functions
│   ├── io/                       # I/O connectors (Delta Lake, Iceberg, etc.)
│   ├── catalog/                  # Catalog implementations
│   ├── runners/                  # Runner implementations (native, Ray)
│   ├── udf/                      # UDF framework
│   ├── ai/                       # AI/LLM integrations
│   └── sql/                      # SQL interface
│
├── tests/                        # Test suite
│   ├── unit tests by module/
│   └── integration/              # Integration tests (Docker-based)
│
├── docs/                         # Documentation (MkDocs)
├── tutorials/                    # Jupyter notebook tutorials
├── benchmarking/                 # Performance benchmarks
└── tools/                        # Development utilities
```

---

## Development Setup

### Prerequisites
- **uv**: `curl -LsSf https://astral.sh/uv/install.sh | sh`
- **Rust**: Uses specific nightly toolchain (`nightly-2025-09-03`)
- **Python**: 3.10+ (3.10 and 3.13 tested in CI)

### Initial Setup

```bash
# 1. Set up Python virtual environment with all dependencies
make .venv

# 2. Install pre-commit hooks
make hooks

# 3. Build the Rust extension
make build
```

### Environment Variables

| Variable | Description |
|----------|-------------|
| `DAFT_RUNNER` | Execution backend: `native` or `ray` (required for tests) |
| `DAFT_LOG` | Rust log level (e.g., `debug`, `info`, `warn`) |
| `DAFT_ANALYTICS_ENABLED` | Set to `0` to disable telemetry |
| `DAFT_PROGRESS_BAR` | Set to `0` to disable progress bars |

---

## Build Commands

```bash
# Development build (fast, unoptimized)
make build

# Release build (optimized)
make build-release

# Build wheel only (no install)
make build-whl

# Clean build artifacts
make clean
```

### Manual Build (without Make)

```bash
# Ensure correct Rust toolchain
rustup show active-toolchain  # Should use rust-toolchain.toml

# Build with maturin
source .venv/bin/activate
maturin develop --uv          # Development
maturin develop --release --uv  # Release
```

---

## Testing

### Test Runners
Daft supports two execution backends:
- **native**: Local native execution (default for development)
- **ray**: Distributed execution via Ray

### Running Tests

```bash
# Run all tests (requires DAFT_RUNNER to be set)
export DAFT_RUNNER=native
make test

# Run specific test file
make test EXTRA_ARGS="-v tests/dataframe/test_select.py"

# Run specific test method
make test EXTRA_ARGS="-v tests/dataframe/test_select.py::test_select_dataframe"

# Run with Ray runner
export DAFT_RUNNER=ray
make test
```

### Test Categories

```bash
# Unit tests (default, -m excludes integration/benchmark/hypothesis)
pytest -n auto --ignore tests/integration

# Integration tests (requires Docker services)
pytest tests/integration/io -m integration

# Property-based tests (Hypothesis)
pytest -m hypothesis

# Doctests
make doctests
```

### Test Configuration

- **pytest config**: `pyproject.toml` under `[tool.pytest.ini_options]`
- **Default markers**: `-m 'not (integration or benchmark or hypothesis)'`
- **Parallel execution**: Uses `pytest-xdist` (`-n auto`)

---

## Code Quality & Linting

### Pre-commit Hooks
All code must pass pre-commit hooks before submission:

```bash
# Install hooks
make hooks

# Run all hooks manually
make precommit

# Or run individually
make check-format    # Check formatting
make format          # Auto-format code
make lint            # Run linters
```

### Python (Ruff)
- **Config**: `.ruff.toml`
- **Line length**: 120
- **Key rules**: UP (pyupgrade), I (isort), D (pydocstyle), RUF, T10
- **Required import**: `from __future__ import annotations`

### Rust (rustfmt + clippy)
- **Config**: `rustfmt.toml`
- **Key settings**: `group_imports = "StdExternalCrate"`, `imports_granularity = "Crate"`
- **Clippy**: Strict linting enabled in `Cargo.toml` workspace lints

### Type Checking
- **mypy**: Strict mode enabled, configured in `.pre-commit-config.yaml`
- **pyright**: Configured in `pyproject.toml` (typeCheckingMode = "off" for most files)

---

## CI/CD

### GitHub Actions Workflows

| Workflow | Purpose |
|----------|---------|
| `pr-test-suite.yml` | Main test suite (unit + integration tests) |
| `pr-labeller.yml` | PR title/commit message linting |
| `build-wheel.yml` | Wheel building for releases |
| `publish-pypi.yml` | PyPI publication |
| `build-docs.yml` | Documentation building |

### CI Matrix
- **Python versions**: 3.10, 3.13
- **Runners**: native, ray
- **OS**: Ubuntu, macOS (main branch only)
- **PyArrow versions**: 8.0.0 (compat), 22.0.0 (default)

### Test Skip Logic
CI skips trivial changes (docs only, config files, etc.). See `skipcheck` job in `pr-test-suite.yml`.

---

## Writing Code

### Adding a New Rust Module

1. Create crate in `src/daft-<name>/`
2. Add to workspace in root `Cargo.toml`
3. Add as dependency in main `Cargo.toml`
4. Register Python module in `src/lib.rs` (`register_modules`)
5. Add to `python` feature if Python bindings needed

### Adding a New Expression Function

1. Implement in appropriate `daft-functions-*` crate
2. Register in `src/lib.rs` function registry
3. Add Python binding if needed
4. Add tests in `tests/functions/`

### Adding an I/O Connector

1. Scan logic: `src/daft-scan/`
2. Python API: `daft/io/`
3. Integration tests: `tests/integration/`

### Python API Changes

Public API exports are in `daft/__init__.py`. Follow existing patterns:
- Use `from __future__ import annotations`
- Type hints required
- Docstrings in Google style

---

## Testing Guidelines

### Unit Tests
- Located alongside functionality in `tests/`
- Use `pytest` fixtures from `tests/conftest.py`
- Parametrize tests for multiple data sources (`arrow`, `parquet`)

### Integration Tests
- Located in `tests/integration/`
- Docker Compose for external services
- Marked with `@pytest.mark.integration`

### Running Integration Tests Locally

```bash
# IO tests
docker compose -f tests/integration/io/docker-compose/docker-compose.yml up -d
pytest tests/integration/io -m integration

# SQL tests
docker compose -f tests/integration/sql/docker-compose/docker-compose.yml up -d
pytest tests/integration/sql -m integration

# Catalog tests (Iceberg, Unity, etc.)
# See individual docker-compose files in tests/integration/
```

---

## Documentation

### Building Docs

```bash
# Install dependencies
make install-docs-deps

# Build documentation
make docs

# Serve locally
make docs-serve
```

### Docstrings
- **Style**: Google format
- **Examples**: Include doctests where applicable
- **Types**: Use Python type hints

### API Documentation
Generated from docstrings using MkDocs with mkdocstrings plugin.

---

## Debugging

### Rust Debugging
```bash
# Set log level
export DAFT_LOG=debug

# Or set Python log level before importing daft
import logging
logging.basicConfig(level=logging.DEBUG)
import daft
daft.refresh_logger()
```

### Python Debugging
```python
# Enable detailed tracebacks
import daft
from daft.context import set_execution_config
set_execution_config(enable_native_tracing=True)
```

---

## Common Issues

### "Failed to build: rust using incorrect toolchain"
Ensure you're using the toolchain specified in `rust-toolchain.toml`:
```bash
rustup show active-toolchain
# Should show: nightly-2025-09-03
```

### Import errors after build
The Rust extension is built as `daft/daft.abi3.so`. If imports fail:
1. Check the file exists
2. Try `make build` again
3. Check Python architecture matches Rust build

### Ray tests hanging
Ray tests can be resource-intensive. On macOS CI, workers are limited to 2 to avoid timeouts.

---

## Release Process

1. Version bump in `Cargo.toml` and `pyproject.toml`
2. Update `CHANGELOG.md`
3. Tag release: `git tag vX.Y.Z`
4. Push tag: `git push origin vX.Y.Z`
5. CI builds and publishes wheels to PyPI

---

## Security

- **Dependencies**: Scan with `cargo audit`
- **Private keys**: Pre-commit hooks detect private keys
- **Telemetry**: Can be disabled with `DO_NOT_TRACK=true`

---

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for detailed guidelines.

**Quick summary:**
1. Non-trivial PRs require an approved issue
2. Include tests with PRs
3. Keep PRs focused and small
4. Pass all CI checks
5. Follow Conventional Commits for PR titles

---

## Resources

- **API Docs**: https://docs.daft.ai/en/stable/api/
- **User Guide**: https://docs.daft.ai/en/stable/
- **Examples**: https://docs.daft.ai/en/stable/examples/
- **Slack Community**: https://www.daft.ai/slack
