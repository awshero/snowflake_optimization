# The cost model — bedrock for first-principles reasoning

Everything here is a fact you can reason *up* from. When a cost claim is disputed, check it against these identities before arguing about it.

Rates and multipliers are **list figures and they drift**. Verify against the current Snowflake Service Consumption Table and the user's contract before any number carries weight in a decision.

## Contents
- [The identities](#the-identities)
- [Warehouse credit rates](#warehouse-credit-rates)
- [Generation and type multipliers](#generation-and-type-multipliers)
- [Serverless multipliers](#serverless-multipliers)
- [Cloud services and the 10% rebate](#cloud-services-and-the-10-rebate)
- [Storage](#storage)
- [Credits to dollars](#credits-to-dollars)
- [Where the evidence lives](#where-the-evidence-lives)

---

## The identities

```
warehouse_credits  = size_rate(size, generation, type) × running_seconds × clusters_running
running_seconds    = query_seconds + idle_seconds
every resume       = max(actual_seconds, 60) billed
serverless_credits = compute_hours × feature_multiplier
storage_cost       = avg_daily_bytes(active + time_travel + failsafe + clone_retained) × $/TB
dollars            = credits × $/credit(edition, region, contract)
```

Consequences worth stating explicitly, because they are the ones people get wrong:

- Warehouse billing is **per-second with a 60-second minimum** on every start or resume. Suspending stops charges immediately.
- **Idle bills identically to work.** A running warehouse consumes credits whether or not a query is executing, which is why per-query cost views understate the bill by 20–40% in most estates.
- Resizing to a **larger** warehouse bills the 60-second minimum only for the incremental clusters, which makes scheduled resizing cheap.
- **Cost is neutral to size only if runtime scales inversely.** Doubling size and halving runtime is a wash; doubling size for a 20% gain is a 60% cost increase.

## Warehouse credit rates

Credits per hour, per cluster, standard Gen1:

| XS | S | M | L | XL | 2XL | 3XL | 4XL | 5XL | 6XL |
|---|---|---|---|---|---|---|---|---|---|
| 1 | 2 | 4 | 8 | 16 | 32 | 64 | 128 | 256 | 512 |

Multiply by `clusters_running` for multi-cluster warehouses. `MIN_CLUSTER_COUNT = 4` on an L warehouse is a 32-credit/hour warehouse, not an 8-credit one — this is the most commonly missed multiplier in the model.

## Generation and type multipliers

| Variant | Multiplier vs standard Gen1 | Note |
|---|---|---|
| Gen2 (AWS, GCP) | ~1.35× | Break-even needs ~26% faster runtime |
| Gen2 (Azure) | ~1.25× | Break-even needs ~20% faster runtime |
| Snowpark-optimized | ~1.5× | M = 6 credits/hr rather than 4 |
| Adaptive | Snowflake-managed | Positioned as throughput-per-dollar, **not** spend reduction |

Two Gen2-specific facts that change recommendations:
- **Query Acceleration is on by default** (scale factor 2) on new Gen2 warehouses, and it bills separately at 1×.
- **Snowflake Optima Indexing** builds and maintains hidden indexes for recurring selective point lookups on Gen2 standard and Adaptive warehouses at **no additional compute or storage charge**, on a best-effort basis. Check what it already covers before paying the 2× multiplier for Search Optimization on the same access pattern.

## Serverless multipliers

Serverless features bill in compute-hours times a per-feature multiplier. The charge is driven by **base-table churn, not query volume** — which is why a heavily-written, lightly-read table can generate large cost for no benefit.

| Feature | Multiplier | Note |
|---|---|---|
| Automatic clustering | 2× | Continuous, proportional to DML churn |
| Materialized views | 2× | Triggered by every base-table change |
| Search optimization | 2× | Plus storage for the access path |
| Snowpipe | 1.25× | **Plus ~0.06 credits per 1,000 files** |
| Query acceleration | 1× | Capped by scale factor |
| Snowpipe Streaming | 1× | Plus a small per-client-hour charge |

The Snowpipe per-file charge is fixed regardless of file size, so cost per GB explodes as files shrink. 1TB as 10,000 × 100MB files costs ~0.6 credits in file charges; the same 1TB as 10M × 100KB files costs ~600.

## Cloud services and the 10% rebate

Cloud services credits are rebated up to **10% of that day's virtual-warehouse credits**, computed daily in UTC. Only the excess bills.

Read a cloud-services bill as a **symptom**, not a line item. It means either enormous metadata traffic (catalog refreshes, `SHOW` loops, `INFORMATION_SCHEMA` polling, huge generated SQL with long compilation) or unusually low warehouse usage. Note the trap: your own successful compute optimization lowers the denominator and can push cloud services over the threshold. Always report it as a ratio alongside the absolute.

## Storage

Billed on **average daily on-disk compressed bytes**, roughly $23/TB/month on-demand in AWS US regions, higher elsewhere. Four components:

| Component | Driver | Notes |
|---|---|---|
| Active | Table size | Compressed |
| Time Travel | **Churn, not size** | Bills per 24h period since the change |
| Fail-safe | Permanent tables only | 7 days, **non-configurable**, cannot be disabled |
| Retained for clone | Divergence after cloning | Clones start free and accrue as source and clone diverge |

Table type determines exposure:

| Type | Time Travel | Fail-safe | Max recovery window |
|---|---|---|---|
| Permanent | 0–90 days | 7 days | 97 days |
| Transient | 0–1 day | none | 1 day |
| Temporary | 0–1 day | none | 1 day (session-scoped) |

Because Time Travel scales with churn, a 100GB table rewritten daily with 90-day retention holds roughly 9TB of history. High retention on high-churn tables is the worst possible combination and is usually inherited from an account-level default rather than chosen.

## Credits to dollars

Approximate list ranges per credit, lowest (US AWS) to highest (non-US):

| Edition | $/credit |
|---|---|
| Standard | $2.00 – $3.10 |
| Enterprise | $3.00 – $4.65 |
| Business Critical | $4.00 – $6.20 |
| VPS | $6.00 – $9.30 |

Storage ~$23/TB/month (AWS US). AI functions bill in a **separate AI-credit currency**, flat-priced and decoupled from edition, spanning roughly $0.12 to $5.10 per million tokens depending on model — about a 40× spread.

Never hard-code a dollar rate across an estate. Two accounts burning identical credits can differ ~2× in dollars because of edition and region. Derive effective $/credit per account by dividing invoiced currency by actual credits — that catches contract terms no configuration view exposes.

**Under a capacity commitment, reduced consumption burns the commit down more slowly rather than producing a refund.** Report *credits avoided* and *dollars realized* as two separate numbers.

## Where the evidence lives

| Question | View |
|---|---|
| Dollars, by account and service | `ORGANIZATION_USAGE.USAGE_IN_CURRENCY_DAILY` |
| Credits by service type, daily | `METERING_DAILY_HISTORY` (includes cloud-services rebate columns) |
| Warehouse credits, hourly, **and idle** | `WAREHOUSE_METERING_HISTORY` — idle = `credits_used_compute − credits_attributed_compute_queries` |
| Per-query credits | `QUERY_ATTRIBUTION_HISTORY` — excludes idle, omits sub-100ms queries, lags ~6h; use for *relative ranking*, not absolute truth |
| Query diagnostics | `QUERY_HISTORY` — `partitions_scanned`/`partitions_total`, `bytes_spilled_to_*`, `queued_overload_time`, `percentage_scanned_from_cache`, `compilation_time`, `query_parameterized_hash` |
| Who read what | `ACCESS_HISTORY` |
| Storage breakdown per table | `TABLE_STORAGE_METRICS` — `active_bytes`, `time_travel_bytes`, `failsafe_bytes`, `retained_for_clone_bytes` |
| Serverless spend | `AUTOMATIC_CLUSTERING_HISTORY`, `SEARCH_OPTIMIZATION_HISTORY`, `MATERIALIZED_VIEW_REFRESH_HISTORY`, `PIPE_USAGE_HISTORY`, `SERVERLESS_TASK_HISTORY`, `QUERY_ACCELERATION_HISTORY` |
| Current config | `SHOW WAREHOUSES` / `SHOW PARAMETERS` — **Snowflake keeps no config history**, so snapshot daily if you want to prove a change caused a saving |

Latency differs by view: `QUERY_HISTORY` up to ~45 min, `QUERY_ATTRIBUTION_HISTORY` up to ~6 h, `ORGANIZATION_USAGE` up to a day. Never present a partial day as complete.
