# wattfrosch

wattfrosch is a modular, self-hosted data platform for
weather and energy data: fetch, store, transform, forecast, and serve it
through an API and dashboard — built from small, independently usable modules.

This repository is the documentation hub. It links all submodules and tracks
their status. Each module is its own repo with its own README and ROADMAP.

## Module status

| Module | Status | Description |
|---|---|---|
| [wattfrosch-contracts](https://github.com/florianquintes/wattfrosch-contracts) | v0.1.0 released | Shared contracts: `MeasurementRecord`, METRICS/REGIONS/SOURCES catalogs, `SOURCE_METRIC_MAP`/`SOURCE_REGION_MAP`, `CONTRACT_VERSION` |
| [wattfrosch-fetch](https://github.com/florianquintes/wattfrosch-fetch) | v0.1.0 (Dockerfile open) | REST clients (SMARD, Energy-Charts, Open-Meteo) + CLI, normalizes into `MeasurementRecord` |
| [wattfrosch-store](https://github.com/florianquintes/wattfrosch-store) | planned | DB schema, migrations, idempotent loader (real SQL, no ORM) |
| [wattfrosch-transform](https://github.com/florianquintes/wattfrosch-transform) | planned | dbt: staging + analytics marts |
| [wattfrosch-pipeline](https://github.com/florianquintes/wattfrosch-pipeline) | planned | Orchestration (Prefect) + backfill |
| [wattfrosch-forecast](https://github.com/florianquintes/wattfrosch-forecast) | planned | ML forecasting + optimization, FastAPI |
| [wattfrosch-dashboard](https://github.com/florianquintes/wattfrosch-dashboard) | planned | BI as Code |
| [wattfrosch-platform](https://github.com/florianquintes/wattfrosch-platform) | planned | docker-compose composition, proxy/tunnel, deployment |

## Architecture

```
External sources --REST--> fetch --> store --> (DB raw.*)
(DB raw.*) --> transform --> marts.*
pipeline orchestrates fetch/store/transform and does backfill
marts.* --> forecast (HTTP API)
marts.* --> dashboard (BI)
platform (docker-compose) composes everything.
```

## Design principles

- **No monolith** — every module is its own repo, usable and testable standalone.
- **Clear contracts** — modules communicate through [wattfrosch-contracts](https://github.com/florianquintes/wattfrosch-contracts); directed dependencies exist between modules (e.g. store → fetch, transform → store).
- **Real SQL** — analytics in hand-written SQL; no ORM.
- **Docker as the delivery layer** — grows with each module.
- **uv** for dependency management.

## wattfrosch-contracts

The shared contract package: defines `MeasurementRecord` and the catalogs
(METRICS, REGIONS, SOURCES, source maps). Definitions only — no logic, no I/O.

```sh
uv add wattfrosch-contracts
```

```python
from datetime import datetime, UTC

from wattfrosch_contracts import (
    MeasurementRecord, METRICS, REGIONS, SOURCES, CONTRACT_VERSION,
)

record = MeasurementRecord(
    ts_utc=datetime(2026, 10, 10, 12, 0, tzinfo=UTC),
    region="DE-LU",
    metric="consumption.load",
    value=42150.0,
    source="smard",
    resolution="PT1H",
)
```

Constructing a record validates everything: the metric, source and region keys
must exist in the catalogs, the region must be valid at `ts_utc`, the value
must be finite, and `resolution` is fixed to `"PT1H"` in v1.

## wattfrosch-fetch

Fetches data from SMARD, Energy-Charts and Open-Meteo and normalizes it into
`MeasurementRecord` — via CLI or as a library.

CLI (exports parquet by default; `--metric` is repeatable):

```sh
git clone https://github.com/florianquintes/wattfrosch-fetch
cd wattfrosch-fetch
uv sync
uv run wattfrosch-fetch --source smard --metric consumption.load \
    --region DE-LU --from 2026-10-10 --to 2026-10-12 \
    --out load.parquet
```

Library:

```python
from datetime import datetime, UTC

from wattfrosch_fetch import create_client

client = create_client("smard")
records = client.fetch_series(
    "consumption.load", "DE-LU",
    datetime(2026, 10, 10, tzinfo=UTC), datetime(2026, 10, 12, tzinfo=UTC),
)
for record in records:
    print(record.ts_utc, record.metric, record.value)

client.close()
```

An empty result is not an error (a warning is emitted instead). Failed runs
exit with code 1 (configuration), 2 (HTTP) or 3 (response validation).
