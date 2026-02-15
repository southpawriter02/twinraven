# TwinRaven — Tech Stack Specification

> Comprehensive dependency and tooling manifest for the TwinRaven framework.

---

## Table of Contents

- [Language and Runtime](#language-and-runtime)
- [Project Structure and Build](#project-structure-and-build)
- [Core Dependencies](#core-dependencies)
  - [Muninn — Memory Layer](#muninn--memory-layer)
  - [Huginn — Thought Layer](#huginn--thought-layer)
  - [Tool Registry](#tool-registry)
  - [Integrations](#integrations)
  - [CLI](#cli)
- [Testing](#testing)
- [Logging and Observability](#logging-and-observability)
- [Code Quality and Linting](#code-quality-and-linting)
- [Documentation](#documentation)
- [Containerization and Deployment](#containerization-and-deployment)
- [Dependency Summary Table](#dependency-summary-table)
- [Python Version Compatibility Matrix](#python-version-compatibility-matrix)

---

## Language and Runtime

| Property             | Value                                                         |
| -------------------- | ------------------------------------------------------------- |
| **Language**         | Python 3.11+                                                  |
| **Minimum version**  | 3.11 (for `tomllib`, `StrEnum`, `ExceptionGroup`)             |
| **Target version**   | 3.12                                                          |
| **Package manager**  | [uv](https://github.com/astral-sh/uv) (primary), pip fallback |
| **Build backend**    | [hatchling](https://hatch.pypa.io/)                           |
| **Project metadata** | `pyproject.toml` (PEP 621)                                    |

### Rationale

Python 3.11 is the floor because:

- `tomllib` is in the standard library (no third-party TOML dependency for config parsing).
- `StrEnum` enables cleaner lifecycle state enums (`draft | testing | promoted | retired`).
- `ExceptionGroup` supports structured error aggregation during validation replay.
- Task groups via `asyncio.TaskGroup` simplify parallel step execution in synthesized tools.

---

## Project Structure and Build

```
twinraven/
├── pyproject.toml              # PEP 621 metadata, dependencies, tool config
├── uv.lock                     # Reproducible lock file
├── src/
│   └── twinraven/
│       ├── __init__.py
│       ├── py.typed             # PEP 561 marker
│       ├── config.py
│       ├── muninn/
│       ├── huginn/
│       ├── tools/
│       ├── integrations/
│       └── cli/
├── tests/
│   ├── conftest.py
│   ├── unit/
│   ├── integration/
│   └── fixtures/
├── docs/
├── Dockerfile
├── docker-compose.yml
└── .github/
    └── workflows/
```

The project uses the **src layout** (`src/twinraven/`) to prevent accidental imports from the working directory and ensure that tests always run against the installed package.

---

## Core Dependencies

### Muninn — Memory Layer

The memory layer captures, stores, and exports tool invocation telemetry.

| Package               | Version | Purpose                                                               |
| --------------------- | ------- | --------------------------------------------------------------------- |
| **pydantic**          | `^2.6`  | Schema definitions for `MuninnEvent`, input validation, serialization |
| **sqlalchemy**        | `^2.0`  | ORM and query builder for the append-only event log                   |
| **alembic**           | `^1.13` | Database schema migrations                                            |
| **aiosqlite**         | `^0.20` | Async SQLite driver (default backend)                                 |
| **asyncpg**           | `^0.29` | Async PostgreSQL driver (production backend)                          |
| **pyarrow**           | `^15.0` | Parquet export format for event log bulk analysis                     |
| **opentelemetry-api** | `^1.22` | OTLP export interface for distributed tracing integration             |
| **opentelemetry-sdk** | `^1.22` | SDK implementation for trace/span export                              |
| **xxhash**            | `^3.4`  | Fast, deterministic hashing for `input_hash` field                    |

#### Storage Backends

TwinRaven supports pluggable storage via SQLAlchemy's async engine:

| Backend    | URI Pattern                     | Use Case                                   |
| ---------- | ------------------------------- | ------------------------------------------ |
| SQLite     | `sqlite+aiosqlite:///muninn.db` | Local development, single-agent            |
| PostgreSQL | `postgresql+asyncpg://...`      | Production, multi-agent, concurrent writes |

#### Schema and Migration

- **Alembic** manages the event log schema.
- Migrations are stored in `src/twinraven/muninn/migrations/`.
- The initial migration creates the `muninn_events` table with indexes on `session_id`, `tool_id`, `timestamp`, and `predecessor`/`successor` for chain reconstruction.

---

### Huginn — Thought Layer

The thought layer mines patterns, synthesizes tools, and validates candidates.

| Package          | Version | Purpose                                                                    |
| ---------------- | ------- | -------------------------------------------------------------------------- |
| **prefixspan**   | `^0.5`  | PrefixSpan sequential pattern mining algorithm                             |
| **numpy**        | `^1.26` | Numerical operations for support/confidence calculations                   |
| **pandas**       | `^2.2`  | Data manipulation for session-level aggregation and analytics              |
| **anthropic**    | `^0.45` | Anthropic API client for LLM-based tool generation                         |
| **openai**       | `^1.12` | OpenAI API client (alternative LLM provider)                               |
| **jinja2**       | `^3.1`  | Template engine for synthesized tool definition rendering                  |
| **jsonschema**   | `^4.21` | Validation of generated tool parameter schemas (JSON Schema Draft 2020-12) |
| **networkx**     | `^3.2`  | Graph-based analysis of tool chain dependencies and parallelism detection  |
| **scikit-learn** | `^1.4`  | Output similarity scoring during validation (cosine similarity, TF-IDF)    |

#### LLM Provider Abstraction

Huginn's synthesis engine uses an `LLMProvider` protocol to abstract over LLM backends. The default is Anthropic Claude (via the `anthropic` SDK), but OpenAI is supported as a drop-in alternative:

```python
class LLMProvider(Protocol):
    async def generate(self, prompt: str, schema: dict) -> str: ...
```

The provider is configured in `twinraven.yaml` under `huginn.synthesis.llm_model`.

#### GSP Implementation

The Generalized Sequential Patterns algorithm is implemented internally (no external library) using the `prefixspan` output as a base, augmented with timestamp-based time-window filtering. This avoids a dependency on less-maintained GSP libraries.

---

### Tool Registry

| Package        | Version | Purpose                                                          |
| -------------- | ------- | ---------------------------------------------------------------- |
| **pydantic**   | `^2.6`  | `SynthesizedTool` and `CandidateChain` schema definitions        |
| **filelock**   | `^3.13` | File-level locking for concurrent tool registry writes           |
| **watchfiles** | `^0.21` | Hot-reload of generated tool definitions for live agent sessions |

The registry stores synthesized tools as versioned JSON files in `tools/generated/`. Each tool definition includes full provenance linking back to the `CandidateChain` and original `MuninnEvent` sample IDs.

---

### Integrations

| Package            | Version  | Purpose                      | Install Extra          |
| ------------------ | -------- | ---------------------------- | ---------------------- |
| **langchain-core** | `^0.3`   | LangChain `BaseTool` adapter | `twinraven[langchain]` |
| **crewai**         | `^0.108` | CrewAI tool adapter          | `twinraven[crewai]`    |

Integrations are **optional dependencies** installed via extras:

```bash
uv pip install twinraven[langchain]
uv pip install twinraven[crewai]
uv pip install twinraven[all]    # everything
```

The `base.py` module provides a framework-agnostic `AgentToolWrapper` protocol that both adapters implement.

---

### CLI

| Package   | Version | Purpose                                                                |
| --------- | ------- | ---------------------------------------------------------------------- |
| **typer** | `^0.12` | CLI framework with auto-generated help and shell completion            |
| **rich**  | `^13.7` | Terminal formatting — tables, progress bars, syntax-highlighted output |

CLI entrypoint is registered in `pyproject.toml`:

```toml
[project.scripts]
twinraven = "twinraven.cli.main:app"
```

Commands: `analyze`, `inspect`, `approve`, `retire`, `export`.

---

## Testing

### Framework and Plugins

| Package            | Version | Purpose                                                                    |
| ------------------ | ------- | -------------------------------------------------------------------------- |
| **pytest**         | `^8.0`  | Test runner                                                                |
| **pytest-asyncio** | `^0.23` | Async test support (`@pytest.mark.asyncio`)                                |
| **pytest-cov**     | `^5.0`  | Coverage measurement and reporting                                         |
| **pytest-xdist**   | `^3.5`  | Parallel test execution                                                    |
| **pytest-mock**    | `^3.12` | `mocker` fixture (thin `unittest.mock` wrapper)                            |
| **hypothesis**     | `^6.98` | Property-based testing for schema validation and mining edge cases         |
| **freezegun**      | `^1.4`  | Deterministic time control for timestamp-dependent logic                   |
| **factory-boy**    | `^3.3`  | Test data factories for `MuninnEvent`, `CandidateChain`, `SynthesizedTool` |
| **respx**          | `^0.21` | Mock HTTP transport for `httpx`-based LLM API calls                        |
| **testcontainers** | `^4.0`  | Ephemeral PostgreSQL containers for integration tests                      |

### Test Structure

```
tests/
├── conftest.py                  # Shared fixtures, factories, database sessions
├── unit/
│   ├── muninn/
│   │   ├── test_collector.py    # Telemetry capture hooks
│   │   ├── test_event_log.py    # Append-only store operations
│   │   ├── test_schemas.py      # Pydantic model validation
│   │   └── test_exporters.py    # JSON, Parquet, OTLP export
│   ├── huginn/
│   │   ├── test_prefixspan.py   # PrefixSpan algorithm correctness
│   │   ├── test_gsp.py          # GSP with time-window constraints
│   │   ├── test_candidates.py   # Chain ranking and filtering
│   │   ├── test_generator.py    # LLM-based tool generation (mocked)
│   │   ├── test_merger.py       # Parameter merging logic
│   │   ├── test_replay.py       # Historical session replay
│   │   ├── test_equivalence.py  # Output comparison scoring
│   │   └── test_promotion.py    # Lifecycle state transitions
│   ├── tools/
│   │   ├── test_registry.py     # Tool storage and lookup
│   │   └── test_lifecycle.py    # draft → testing → promoted → retired
│   ├── integrations/
│   │   ├── test_langchain.py    # LangChain adapter
│   │   └── test_crewai.py       # CrewAI adapter
│   └── cli/
│       ├── test_analyze.py      # CLI analyze command
│       ├── test_approve.py      # CLI approve/promote workflow
│       └── test_inspect.py      # CLI inspect command
├── integration/
│   ├── test_event_log_pg.py     # PostgreSQL event log (testcontainers)
│   ├── test_full_loop.py        # End-to-end Muninn → Huginn → Registry
│   └── test_export_roundtrip.py # Write events → export → re-ingest
└── fixtures/
    ├── sample_events.json       # Canonical event log samples
    ├── sample_chains.json       # Pre-computed candidate chains
    └── sample_tools.json        # Reference synthesized tool definitions
```

### Coverage Targets

| Scope                | Target                     |
| -------------------- | -------------------------- |
| Overall              | ≥ 90% line coverage        |
| `muninn/`            | ≥ 95% (critical data path) |
| `huginn/mining/`     | ≥ 90%                      |
| `huginn/synthesis/`  | ≥ 85% (LLM calls mocked)   |
| `huginn/validation/` | ≥ 90%                      |
| `tools/`             | ≥ 95%                      |
| `integrations/`      | ≥ 80%                      |
| `cli/`               | ≥ 75%                      |

### Running Tests

```bash
# Unit tests (fast, no external dependencies)
uv run pytest tests/unit -v --tb=short

# Integration tests (requires Docker for testcontainers)
uv run pytest tests/integration -v --tb=short

# Full suite with coverage
uv run pytest --cov=twinraven --cov-report=html --cov-report=term-missing

# Parallel execution
uv run pytest -n auto
```

---

## Logging and Observability

### Structured Logging

| Package       | Version | Purpose                                                        |
| ------------- | ------- | -------------------------------------------------------------- |
| **structlog** | `^24.1` | Structured, contextual logging with JSON and console renderers |

TwinRaven uses `structlog` throughout, bound to Python's standard `logging` module. Every log entry carries structured context:

```python
import structlog

logger = structlog.get_logger(__name__)

logger.info(
    "chain_detected",
    chain_length=3,
    support=0.87,
    confidence=0.94,
    tools=["search", "read", "summarize"],
)
```

### Log Levels by Component

| Component           | Level     | What's logged                                                  |
| ------------------- | --------- | -------------------------------------------------------------- |
| `muninn.collector`  | `DEBUG`   | Every tool invocation capture                                  |
| `muninn.event_log`  | `INFO`    | Event writes, batch flushes                                    |
| `muninn.exporters`  | `INFO`    | Export start/complete, row counts                              |
| `huginn.mining`     | `INFO`    | Mining run start, candidates found, thresholds applied         |
| `huginn.synthesis`  | `INFO`    | Generation requests, LLM token usage, tool definitions created |
| `huginn.validation` | `INFO`    | Replay sessions, equivalence scores, promotion decisions       |
| `huginn.validation` | `WARNING` | Validation failures, latency regressions                       |
| `tools.registry`    | `INFO`    | Tool registered, promoted, retired                             |
| `tools.lifecycle`   | `WARNING` | Auto-retirement triggers, drift detection                      |
| `cli`               | `INFO`    | Command invocations, user actions                              |

### OpenTelemetry Integration

For production deployments, TwinRaven exports traces via OTLP:

| Signal      | Exporter              | Purpose                                                            |
| ----------- | --------------------- | ------------------------------------------------------------------ |
| **Traces**  | OTLP/gRPC             | Span-per-tool-invocation for distributed tracing                   |
| **Metrics** | Prometheus (optional) | Chain detection rates, synthesis throughput, tool lifecycle counts |

The OTLP exporter is enabled via configuration:

```yaml
observability:
  otlp_endpoint: "http://localhost:4317"
  service_name: "twinraven"
  trace_sample_rate: 1.0
```

---

## Code Quality and Linting

| Tool           | Version | Purpose                                                | Config Location           |
| -------------- | ------- | ------------------------------------------------------ | ------------------------- |
| **ruff**       | `^0.3`  | Linting and formatting (replaces flake8, isort, black) | `pyproject.toml`          |
| **mypy**       | `^1.8`  | Static type checking                                   | `pyproject.toml`          |
| **pre-commit** | `^3.6`  | Git hook management                                    | `.pre-commit-config.yaml` |
| **bandit**     | `^1.7`  | Security-focused static analysis                       | `pyproject.toml`          |

### Ruff Configuration

```toml
[tool.ruff]
target-version = "py311"
line-length = 100

[tool.ruff.lint]
select = [
    "E",    # pycodestyle errors
    "W",    # pycodestyle warnings
    "F",    # pyflakes
    "I",    # isort
    "N",    # pep8-naming
    "UP",   # pyupgrade
    "ANN",  # flake8-annotations
    "S",    # flake8-bandit
    "B",    # flake8-bugbear
    "A",    # flake8-builtins
    "C4",   # flake8-comprehensions
    "DTZ",  # flake8-datetimez
    "T20",  # flake8-print
    "PT",   # flake8-pytest-style
    "SIM",  # flake8-simplify
    "TCH",  # flake8-type-checking
    "RUF",  # ruff-specific
]
```

### mypy Configuration

```toml
[tool.mypy]
python_version = "3.11"
strict = true
warn_return_any = true
warn_unused_configs = true
disallow_untyped_defs = true

[[tool.mypy.overrides]]
module = "prefixspan.*"
ignore_missing_imports = true
```

### Pre-commit Hooks

```yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.3.0
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format
  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.8.0
    hooks:
      - id: mypy
        additional_dependencies: [pydantic, sqlalchemy[mypy]]
```

---

## Documentation

| Tool                     | Version | Purpose                                     |
| ------------------------ | ------- | ------------------------------------------- |
| **mkdocs**               | `^1.5`  | Documentation site generator                |
| **mkdocs-material**      | `^9.5`  | Material theme for MkDocs                   |
| **mkdocstrings[python]** | `^0.24` | Auto-generate API reference from docstrings |

Documentation lives in `docs/` and is built with:

```bash
uv run mkdocs serve    # local preview
uv run mkdocs build    # static site
```

---

## Containerization and Deployment

### Dockerfile

```dockerfile
FROM python:3.12-slim AS base
WORKDIR /app
COPY --from=ghcr.io/astral-sh/uv:latest /uv /usr/local/bin/uv
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev
COPY src/ src/
```

### Docker Compose (Development)

```yaml
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: muninn
      POSTGRES_USER: twinraven
      POSTGRES_PASSWORD: dev
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data

  jaeger:
    image: jaegertracing/all-in-one:latest
    ports:
      - "16686:16686" # UI
      - "4317:4317" # OTLP gRPC

volumes:
  pgdata:
```

### CI/CD (GitHub Actions)

```yaml
# .github/workflows/ci.yml
jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: ["3.11", "3.12", "3.13"]
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v5
      - run: uv sync --all-extras
      - run: uv run pytest --cov=twinraven --cov-report=xml
      - run: uv run mypy src/
      - run: uv run ruff check src/ tests/
```

---

## Dependency Summary Table

All dependencies organized by installation group.

### Core (always installed)

| Package      | Version | Layer                 |
| ------------ | ------- | --------------------- |
| pydantic     | ^2.6    | Muninn, Huginn, Tools |
| sqlalchemy   | ^2.0    | Muninn                |
| alembic      | ^1.13   | Muninn                |
| aiosqlite    | ^0.20   | Muninn                |
| xxhash       | ^3.4    | Muninn                |
| structlog    | ^24.1   | All                   |
| prefixspan   | ^0.5    | Huginn                |
| numpy        | ^1.26   | Huginn                |
| pandas       | ^2.2    | Huginn                |
| jinja2       | ^3.1    | Huginn                |
| jsonschema   | ^4.21   | Huginn                |
| networkx     | ^3.2    | Huginn                |
| scikit-learn | ^1.4    | Huginn                |
| filelock     | ^3.13   | Tools                 |
| watchfiles   | ^0.21   | Tools                 |
| typer        | ^0.12   | CLI                   |
| rich         | ^13.7   | CLI                   |
| pyyaml       | ^6.0    | Config                |

### Optional Extras

| Extra           | Packages                                                        |
| --------------- | --------------------------------------------------------------- |
| `langchain`     | langchain-core ^0.3                                             |
| `crewai`        | crewai ^0.108                                                   |
| `llm-anthropic` | anthropic ^0.45                                                 |
| `llm-openai`    | openai ^1.12                                                    |
| `postgres`      | asyncpg ^0.29                                                   |
| `export`        | pyarrow ^15.0, opentelemetry-api ^1.22, opentelemetry-sdk ^1.22 |
| `all`           | All of the above                                                |

### Development

| Package              | Version | Purpose                |
| -------------------- | ------- | ---------------------- |
| pytest               | ^8.0    | Test runner            |
| pytest-asyncio       | ^0.23   | Async test support     |
| pytest-cov           | ^5.0    | Coverage               |
| pytest-xdist         | ^3.5    | Parallel tests         |
| pytest-mock          | ^3.12   | Mocking                |
| hypothesis           | ^6.98   | Property-based testing |
| freezegun            | ^1.4    | Time control           |
| factory-boy          | ^3.3    | Test data factories    |
| respx                | ^0.21   | HTTP mocking           |
| testcontainers       | ^4.0    | Ephemeral containers   |
| ruff                 | ^0.3    | Lint and format        |
| mypy                 | ^1.8    | Type checking          |
| pre-commit           | ^3.6    | Git hooks              |
| bandit               | ^1.7    | Security analysis      |
| mkdocs               | ^1.5    | Documentation          |
| mkdocs-material      | ^9.5    | Documentation theme    |
| mkdocstrings[python] | ^0.24   | API reference          |

---

## Python Version Compatibility Matrix

| Python | Status           | Notes                                           |
| ------ | ---------------- | ----------------------------------------------- |
| 3.10   | ❌ Not supported | Missing `tomllib`, `StrEnum`, `TaskGroup`       |
| 3.11   | ✅ Minimum       | Full feature support                            |
| 3.12   | ✅ Target        | Performance improvements, better error messages |
| 3.13   | ✅ Supported     | Tested in CI                                    |

---

## License

All dependencies have been selected for GPL-3.0 compatibility. The core dependency licenses are:

| License    | Packages                                                                                |
| ---------- | --------------------------------------------------------------------------------------- |
| MIT        | pydantic, sqlalchemy, structlog, typer, rich, jinja2, filelock, xxhash, most test tools |
| BSD-3      | numpy, pandas, scikit-learn, networkx                                                   |
| Apache 2.0 | opentelemetry-\*, anthropic, openai, prefixspan                                         |

All listed licenses are compatible with GPL-3.0 distribution.
