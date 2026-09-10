# DAX Measures

## Current-state (SCD-aware)

```dax
Current Rank =
CALCULATE(
    MAX(dim_coin[market_cap_rank]),
    dim_coin[IsCurrent] = TRUE
)
```
Filters dim_coin down to only the active version of each coin — ignores
historical rows even though the table can hold multiple versions per coin.

## Latest-value pattern (used 3x below)

```dax
Latest Price =
CALCULATE(
    MAX(fact_price_snapshot[CurrentPrice]),
    FILTER(
        fact_price_snapshot,
        fact_price_snapshot[LastUpdated] = MAX(fact_price_snapshot[LastUpdated])
    )
)

Latest Volume =
CALCULATE(
    MAX(fact_price_snapshot[TotalVolume]),
    FILTER(
        fact_price_snapshot,
        fact_price_snapshot[LastUpdated] = MAX(fact_price_snapshot[LastUpdated])
    )
)

Latest 24h Change =
CALCULATE(
    MAX(fact_price_snapshot[PriceChangePct24h]),
    FILTER(
        fact_price_snapshot,
        fact_price_snapshot[LastUpdated] = MAX(fact_price_snapshot[LastUpdated])
    )
)
```
Restricts a snapshot-history fact table down to just each coin's most
recent observation — the pattern to reuse any time you want "current
value" out of a table that stores full history rather than one row per entity.

## Volatility

```dax
Price Volatility (StdDev) =
STDEV.P(fact_price_snapshot[CurrentPrice])
```
Standard deviation of price across all captured snapshots per coin.
Note: currently data-limited — coins with only one snapshot captured so
far show 0, not because they're stable, but because there's nothing yet
to compute variance against. Becomes more meaningful as more snapshots
accumulate over time.

## All-time-high comparison

```dax
Pct From ATH =
DIVIDE(
    AVERAGE(fact_price_snapshot[CurrentPrice]) - AVERAGE(fact_price_snapshot[AllTimeHigh]),
    AVERAGE(fact_price_snapshot[AllTimeHigh])
)
```
How far current price sits below (or above) each coin's all-time high.

## Ranking / leaderboard

```dax
Top Gainer =
VAR Top10Coins =
    TOPN(10, ALL(dim_coin[name]), [Latest Volume], DESC)
RETURN
    CALCULATE(
        SELECTEDVALUE(dim_coin[name]),
        TOPN(1, Top10Coins, [Latest 24h Change], DESC)
    )

Top Loser =
VAR Top10Coins =
    TOPN(10, ALL(dim_coin[name]), [Latest Volume], DESC)
RETURN
    CALCULATE(
        SELECTEDVALUE(dim_coin[name]),
        TOPN(1, Top10Coins, [Latest 24h Change], ASC)
    )
```
Deliberately scoped to the same top-10-by-volume universe used
everywhere else on the report page, so the headline cards always name a
coin the viewer can find in the table/chart below — an earlier version
searched all 100 coins and could surface a name invisible elsewhere on
the page, which was confusing even though technically correct.

Total Market Volume card:
```dax
Total Latest Volume =
SUMX(
    VALUES(dim_coin[name]),
    [Latest Volume]
)
```
