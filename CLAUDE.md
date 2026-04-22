# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Run Dagster UI
dagster dev

# Run all tests
pytest tests/

# Run single test file
pytest tests/test_lesson_9.py

# Lint and format
make ruff
```

Ruff is configured to allow ERA001 (comments), E501 (long lines), F401 (unused imports).

## Environment

Requires `.env` with `DUCKDB_DATABASE` pointing to the DuckDB file (e.g., `data/staging/data.duckdb`). Tests auto-set this via `conftest.py` fixture.

## Architecture

NYC taxi data pipeline built as a Dagster Essentials course project. Lessons 3–9 each have reference solutions in `src/dagster_essentials/completed/lesson_N/`; the active working code lives in `src/dagster_essentials/defs/`.

**Entry point**: `definitions.py` uses `load_from_defs_folder()` — Dagster auto-discovers all assets, jobs, schedules, and sensors from the `defs/` directory. No explicit imports needed when adding new definitions there.

**Asset pipeline flow**:
```
taxi_trips_file (download parquet) → taxi_trips (load to DuckDB, monthly partitioned)
taxi_zones_file (download CSV)     → taxi_zones (load to DuckDB)
                                     ↓
                               manhattan_stats (join + aggregate) → manhattan_map (PNG)
                                     ↓
                               weekly_stats → trips_by_week
```

**Key patterns from lesson 9 (most complete)**:
- `MonthlyPartitionsDefinition` / `WeeklyPartitionsDefinition` on assets and jobs
- `DuckDBResource` injected via `dg.EnvVar("DUCKDB_DATABASE")` — never hardcode DB path
- `adhoc_request_sensor` polls `data/requests/*.json`, triggers runs via `SensorResult(run_requests=[...])`
- `AdhocRequestConfig(dg.Config)` passes per-run parameters to the `adhoc_request` asset
- Asset groups: `raw_files`, `ingested`, `manhattan`
- Asset kinds: `{"python"}`, `{"duckdb"}` — set on `@dg.asset` decorator

**Test pattern** (`tests/`): each lesson test materializes assets with `materialize()` and asserts `result.success`. Fixtures in `fixtures.py` clean DuckDB tables between runs.
