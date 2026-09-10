# Architecture

## Pipeline layers

### Bronze
- Data Pipeline (`ingest_crypto_bronze`) calls CoinGecko's `/coins/markets`
  endpoint (no auth) for the top 100 coins by market cap.
- Each run lands a new, uniquely timestamped JSON file
  (`crypto_yyyyMMdd_HHmmss.json`) into Lakehouse **Files** — never
  overwritten, preserving a full raw history for reprocessing if needed.

### Silver
- Reads all Bronze files, flattens the flat JSON array, extracts the
  inconsistently-nested `roi.percentage` field via dot notation, casts
  date fields to real timestamps.
- **Incremental loading**: rather than rebuilding from scratch each run,
  Silver computes a watermark (`MAX(LastUpdated)` already in the table)
  and only appends rows newer than that watermark — verified end-to-end
  by triggering a real new Bronze pull and confirming exactly the new
  rows (and only the new rows) were appended.
- An `audit_silver_runs` table logs every run's watermark before/after
  and row count appended, giving a queryable history of pipeline
  behavior instead of relying on memory or notebook cell order.

### Gold
- `dim_coin` — a true **SCD Type 2** dimension. `market_cap_rank` (and
  other tracked attributes) can change over time; rather than overwrite,
  changed rows are closed out (`IsCurrent = false`, `EffectiveEndDate`
  stamped) and a new version is inserted. Verified with a controlled
  test (artificially changing Bitcoin's stored rank and confirming the
  merge logic correctly detected and versioned it).
- `fact_price_snapshot` — every price observation, resolved to the
  **specific dim_coin version** active at the time of each snapshot via
  a `SurrogateKey`, not the natural coin ID. This was a deliberate fix
  for a real modeling flaw: relating fact to dim on the natural key
  alone would create a many-to-many relationship once dim_coin holds
  multiple historical rows per coin.

## Semantic model
- Direct Lake, only Gold tables included (Bronze/Silver/audit tables are
  pipeline infrastructure, not reporting data).
- Relationship: `fact_price_snapshot[SurrogateKey]` → `dim_coin[SurrogateKey]`,
  many-to-one — the surrogate key relationship is what makes the model
  unambiguous despite dim_coin's historical versioning.

## Real bugs hit and resolved
1. **Nested JSON schema-inference timeout** — Fabric's Copy Data schema
   mapping couldn't handle the weather project's nested arrays; same
   risk considered here, avoided by landing raw JSON to Files and
   parsing explicitly in-notebook (see prior project's precedent).
2. **Notebook cell execution order** — re-running an old full-overwrite
   Silver cell after building the incremental version silently produced
   a 299-row state that looked correct but wasn't reached via the
   intended path. Root-caused by systematically checking Silver's actual
   row count and watermark rather than trusting assumed state — resolved
   by consolidating to one linear, clearly labeled notebook with no
   duplicate/conflicting cells.
3. **SCD Type 2 join returning zero rows** — dim_coin's initial
   `EffectiveStartDate` was stamped with `current_timestamp()` (when the
   cell happened to run) rather than each coin's true earliest observed
   date, causing every fact row's `LastUpdated` to fall before the dim
   row's start date and fail the join. Fixed by seeding
   `EffectiveStartDate` from `MIN(LastUpdated)` per coin in Silver
   instead, and verified via a join-count check (489/489 fact rows
   matched a dim version) before writing.

## Data quality note
`Figure Heloc`'s `PriceChangePct24h` field is populated in its earliest
snapshot but null in all subsequent ones — a source-data inconsistency
from CoinGecko itself (likely reflecting limited trading history for
this asset), not a pipeline defect. Surfaced rather than papered over
with a default value.
