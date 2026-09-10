# Fabric Crypto Medallion Pipeline

An expert-level Microsoft Fabric project implementing true medallion
architecture (Bronze → Silver → Gold) on live cryptocurrency market data
from the CoinGecko API, with real incremental loading, a scheduled
pipeline, and a Slowly Changing Dimension (SCD Type 2) tracking coin
rank history over time.

## Architecture

Bronze (raw JSON snapshots) → Silver (cleaned, typed, deduplicated,
incrementally appended using a data-driven watermark) → Gold
(`fact_price_snapshot` + `dim_coin`, the latter a true SCD Type 2 table
resolved via surrogate keys) → Direct Lake semantic model → Power BI report.

Full write-up, including three real bugs debugged from first principles,
is in [`docs/architecture.md`](docs/architecture.md).

## Tech stack
- Microsoft Fabric (Data Pipeline with scheduled trigger, Lakehouse, Notebook, Power BI semantic model)
- CoinGecko public API (no auth required)
- PySpark, Delta Lake (`DeltaTable.merge`-style update/insert logic)
- DAX (including RANKX, TOPN, time-filtered "latest value" patterns)

## Status
✅ Complete

## Contents
- `notebooks/` — Bronze/Silver/Gold transformation notebook export
- `dax/` — documented DAX measures
- `screenshots/` — report screenshots
- `docs/` — architecture notes, debugging log, and design decisions
