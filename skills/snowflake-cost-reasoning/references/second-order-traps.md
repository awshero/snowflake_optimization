# Second-order traps — Snowflake optimizations that backfire

Each entry is a move that looks obviously correct at first order. Read the **signal** column first: it is what distinguishes the case where the move is right from the case where it is a mistake. The point is never "don't do this" — most of these are correct moves in the right circumstance. The point is that the circumstance has to be checked.

## Contents
- [Compute](#compute)
- [Concurrency and scaling](#concurrency-and-scaling)
- [Serverless and data layout](#serverless-and-data-layout)
- [Storage](#storage)
- [Contract and organisation](#contract-and-organisation)
- [Traps against the programme itself](#traps-against-the-programme-itself)

---

## Compute

### Downsizing a warehouse that is spilling
**And then what:** A smaller warehouse has less memory, so it spills more. Runtime grows by more than the rate falls — remote spill can slow a query by roughly an order of magnitude — so cost rises, and memory-bound jobs may now fail outright rather than merely slow down.
**Signal:** `bytes_spilled_to_remote_storage > 0` on any material query. Spilling is a *query* problem: an exploding join key, an unnecessary global sort, missing projection.
**Correct move:** Fix the query. Only downsize once remote spill is zero and load, queueing and runtime all show headroom.

### Tightening auto-suspend past the arrival cadence
**And then what:** Each resume bills a 60-second minimum and discards the local cache. If gaps are 45s and the suspend is 30s, you pay a fresh minute repeatedly and every query re-reads from cold storage. Cost rises and latency worsens simultaneously.
**Signal:** Compare the setting against the observed inter-query gap distribution, not against a policy. If the p70 gap exceeds the proposed value, expect churn.
**Correct move:** Set just above the p60–p75 gap. On warehouses with high `percentage_scanned_from_cache`, warm cache is worth real money — be more conservative there.

### Upsizing to fix queueing
**And then what:** Queueing is a concurrency constraint, not a throughput one. A bigger warehouse runs each query faster but does not raise how many run concurrently, so the queue persists at double the rate.
**Signal:** `queued_overload_time` high while `query_load_percent` is low and spill is zero.
**Correct move:** Add clusters, raise `MAX_CONCURRENCY_LEVEL`, or reschedule. Distinguish `queued_overload_time` (saturation) from `queued_provisioning_time` (the warehouse was resuming — an auto-suspend matter, and often an argument *against* tightening it).

### Migrating the fleet to Gen2
**And then what:** Gen2 bills ~1.35× (AWS/GCP) or ~1.25× (Azure). Workloads that do not get ~26%/~20% faster simply pay more. Sub-second point queries, metadata operations and short statements gain nothing and pay the premium on every 60-second minimum.
**Signal:** Measure credits per GB scanned and credits per 1,000 executions of the same `query_parameterized_hash` across the migration boundary, restricted to hashes present on both sides.
**Correct move:** Per-warehouse A/B with `USE_CACHED_RESULT=FALSE` and a suspend between runs. Rollback is a legitimate outcome.

### Defaulting to Snowpark-optimized warehouses for Python work
**And then what:** ~1.5× the standard rate, permanently, for memory headroom the job may never touch.
**Signal:** Zero remote spill and negligible local spill across a full cycle means the headroom is unused.
**Correct move:** Revert to standard. Where memory pressure is real but intermittent, split: memory-hungry jobs on a small Snowpark-optimized warehouse, the rest on standard. Be conservative — insufficient memory produces hard failures, not graceful degradation.

## Concurrency and scaling

### Enabling Query Acceleration "for performance"
**And then what:** QAS bills separately at a 1× multiplier. Added to an already-oversized warehouse, you now pay twice for the same outlier query.
**Signal:** QAS credits rising while warehouse credits stay flat or rise. That is additive cost, not substituted cost.
**Correct move:** QAS is a **substitute** for permanent upsizing, not a supplement. Enable it *and* downsize the base warehouse a step, as one paired change. The saving lives entirely in the downsize.

### Setting ECONOMY scaling policy everywhere
**And then what:** ECONOMY only starts a cluster when it estimates ~6 minutes of work to keep it busy, deliberately trading queue time for utilization. On an interactive warehouse, analysts wait — then rerun queries, escalate, or build private extracts elsewhere. Credits saved reappear as salary spent and shadow infrastructure.
**Signal:** Does a human wait on this warehouse? Check whether queries originate from a BI tool or a human-facing role.
**Correct move:** ECONOMY for batch, ELT, ingest, scheduled reporting, ML training. STANDARD for interactive BI and ad-hoc.

### Lowering MAX_CONCURRENCY_LEVEL to make dashboards faster
**And then what:** Fewer queries share each cluster, so queries spill onto *new* clusters. You have converted a latency complaint into a multi-cluster bill without anyone deciding to.
**Signal:** Average running clusters above 1.2 with a concurrency level below default and no spill evidence justifying it.
**Correct move:** Treat it as a memory-pressure lever, not a latency lever. If latency is the goal, look at the queries first.

### Consolidating warehouses to eliminate idle tails
**And then what:** The merged warehouse must be sized for its worst query and kept warm for its most impatient consumer, so the cheap workloads subsidize the expensive one. Attribution also collapses, and you lose the ability to tell whose spend it is.
**Signal:** Compute a tenancy-mixing score — the variance of query shapes. Sub-second interactive queries sharing with multi-hour batch is the shape to avoid.
**Correct move:** Consolidate only warehouses in the same workload class with non-overlapping active hours. The saving is the eliminated idle tail, not the compute.

## Serverless and data layout

### Adding a clustering key to fix poor pruning
**And then what:** Reclustering bills continuously at 2×, driven by DML churn. If the real cause was a non-prunable predicate — a function or cast wrapping the filtered column — you are paying perpetually for something a `WHERE`-clause rewrite would have fixed for free. On a table with heavy random DML, no key survives and you pay forever.
**Signal:** Classify the cause first. Function/cast on the filter column, or a type-mismatched join key, means it is a query problem.
**Correct move:** Fix the predicate. Cluster only where predicates are already prunable, the table is large, and 3–4 low-cardinality-first key columns match what queries actually filter on.

### Adding a materialized view to speed a repeated query
**And then what:** Maintenance is triggered by every base-table change at a 2× multiplier. On a fast-changing base, maintenance exceeds the query saving, and the MV also constrains base-table DDL.
**Signal:** Compare base-table churn against read frequency.
**Correct move:** MVs suit expensive aggregation over *slowly changing* bases. Net-meter it after 30 days: serverless credits consumed versus query credits avoided.

### Enabling Search Optimization for "faster lookups"
**And then what:** 2× build and maintenance plus access-path storage — and it does not help range scans or aggregations at all, only selective point lookups.
**Signal:** Are the queries point lookups or range scans? And is the warehouse Gen2?
**Correct move:** Column scope, never whole-table. On Gen2, check what **Optima Indexing** already covers at zero charge before paying the 2× multiplier for the same access pattern.

### Tightening a dynamic table's TARGET_LAG for freshness
**And then what:** Lag propagates *upstream* through the DAG. One tight leaf forces every ancestor to refresh at least that often, multiplying the cost of an entire pipeline from a three-word DDL change.
**Signal:** Walk the DAG. Also check `refresh_mode_reason` — a construct in the query can silently force FULL refresh, an order-of-magnitude cost difference nobody is alerted to.
**Correct move:** Set lag from documented consumer need; use `DOWNSTREAM` for intermediate tables so freshness is pulled by demand. Attribute cost to the leaf that set the tight lag.

### Switching ingestion to Snowpipe for convenience
**And then what:** 1.25× plus a **per-file charge that is fixed regardless of file size**. With small files, cost per GB explodes — the same terabyte can cost a thousand times more as 100KB files than as 100MB files.
**Signal:** Average bytes per file below ~10MB.
**Correct move:** Match method to real latency need, inferred from when data is first read rather than from the stated requirement. Predictable batches → `COPY` on a small warehouse. Negotiate 100–250MB compressed files with producers.

## Storage

### Reducing DATA_RETENTION_TIME_IN_DAYS across the estate
**And then what:** History beyond the new window is discarded **immediately**. This is a one-way door: reversible as a setting, not as data.
**Signal:** Which tables are genuinely regulated, and which merely inherited an account-level default?
**Correct move:** Set per layer with named owner sign-off and a notice period. Never automate. Convert reproducible layers to transient instead — that also removes Fail-safe, which cannot otherwise be disabled.

### Dropping accumulated clones to reclaim storage
**And then what:** If the nightly job that creates them is untouched, they come back within a month.
**Signal:** A regular cadence of dated clones with no matching drop.
**Correct move:** The primary fix is the job's lifecycle, not the artifacts.

### Dropping tables nobody has read
**And then what:** Annual and regulatory reporting reads once a year. A 90-day window will confidently propose dropping it. A view referencing a dropped table fails at read time, possibly months later.
**Signal:** 365-day read window, plus an `OBJECT_DEPENDENCIES` walk.
**Correct move:** Stage it — notify owner, revoke access or rename, wait 30 days, then drop with the Time Travel window still open.

## Contract and organisation

### Cutting consumption under a capacity commitment
**And then what:** You burn the commit down more slowly. There is no refund. Cash saving this year may be exactly zero, and the programme gets judged a failure in month nine by a finance partner who was never told.
**Signal:** Does a commitment exist, and when does it end?
**Correct move:** Agree the realization mechanism up front — renegotiate at renewal, reduce growth against a rising baseline, or reallocate freed capacity to work that would otherwise need new spend. Report credits avoided and dollars realized separately, always.

### Introducing chargeback to create accountability
**And then what:** Teams optimize for the metric rather than the outcome — hiding workloads in someone else's warehouse, disputing allocations, and turning an engineering exercise into accounting theatre. Attribution quality degrades precisely because it now has consequences.
**Signal:** Is attribution trusted yet? Is the unattributed share still material?
**Correct move:** Showback first. Revisit chargeback only after two quarters of numbers nobody disputes.

## Traps against the programme itself

These are second-order effects on your ability to keep working, and they are as real as effects on the bill.

### A resource monitor suspends a warehouse during month-end close
**And then what:** One wrongful suspension of a real workload costs more political capital than the guardrail saves. You lose the automation mandate for a year.
**Correct move:** Run every monitor notify-only for 30 days to calibrate thresholds before any suspend action is armed.

### Announcing an aggressive savings target
**And then what:** Promising 28% and delivering 15% destroys credibility even though 15% is excellent work. Promising 18% and delivering 24% funds year two.
**Correct move:** Commit to the low end publicly; report the full range internally.

### Optimizing other teams' workloads centrally
**And then what:** A central team runs out of context and goodwill at roughly the same moment, usually about four months in. The programme becomes the team that says no, and engineers route around it.
**Correct move:** Federated ownership. Central team owns the tooling, standards and guardrails; product teams own their own spend and reductions. Track cost per unit of work rather than absolute spend, so a team that doubles its workload at flat unit cost is visibly succeeding.

### Reporting a cost regression that your own success created
**And then what:** Cloud-services credits are rebated up to 10% of warehouse credits. Cut warehouse credits and the same metadata traffic can breach the threshold, appearing as a new bill.
**Correct move:** Report cloud services as a ratio alongside the absolute figure.
