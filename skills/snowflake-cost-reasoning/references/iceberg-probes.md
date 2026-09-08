# Iceberg probes — how to reach each layer

The iceberg only helps if each layer is supported by evidence. Speculating about mental models while skipping the pattern data produces something that sounds insightful and is unfalsifiable. Work down, and gather evidence at every level.

The probes below are the questions that move you from one layer to the next, and the queries that answer them.

## Contents
- [Layer 1 — Events](#layer-1--events)
- [Layer 2 — Patterns](#layer-2--patterns)
- [Layer 3 — Structures](#layer-3--structures)
- [Layer 4 — Mental models](#layer-4--mental-models)
- [Knowing when to stop](#knowing-when-to-stop)

---

## Layer 1 — Events

**What it is:** the thing that prompted the conversation. One invoice, one spike, one alarming query, one team's complaint.

**Probes:**
- What exactly happened, in credits and dollars, over what window?
- Which service type moved — warehouse, serverless, storage, transfer, AI?
- Which account, warehouse, or object concentrated the change?

```sql
-- Where did it move? Service-type decomposition, day by day
SELECT usage_date, service_type, SUM(credits_used) AS credits
FROM snowflake.account_usage.metering_daily_history
WHERE usage_date >= DATEADD(day, -90, CURRENT_DATE())
GROUP BY 1,2 ORDER BY 1 DESC, 3 DESC;

-- Which warehouses moved most, this period vs the one before
SELECT warehouse_name,
       SUM(IFF(start_time >= DATEADD(day,-30,CURRENT_TIMESTAMP()), credits_used_compute, 0)) AS recent,
       SUM(IFF(start_time <  DATEADD(day,-30,CURRENT_TIMESTAMP()), credits_used_compute, 0)) AS prior
FROM snowflake.account_usage.warehouse_metering_history
WHERE start_time >= DATEADD(day,-60,CURRENT_TIMESTAMP())
GROUP BY 1 QUALIFY recent - prior <> 0 ORDER BY recent - prior DESC LIMIT 25;
```

**The move to Layer 2:** ask *has this happened before?* If yes, you are looking at a pattern and the event is just its latest instance. Treating it as a one-off is how the same fix gets applied four times.

## Layer 2 — Patterns

**What it is:** the recurring shape. Patterns are what tell you a structure exists, because structures are what produce repetition.

**Probes:**
- Over 6–12 months, is this rising, cyclical, or a step change? A step change has a cause with a date — find the date.
- What is the hour-of-week profile? Bimodal demand means a scheduling structure, not a sizing problem.
- What fraction of warehouse credits is idle, and has that ratio been stable?
- Is spend concentrated in a few query patterns? Group by `query_parameterized_hash`, not by query text — the correct unit is the *pattern*, since fixing one fixes every execution.

```sql
-- Idle ratio per warehouse: the pattern that most often explains the event
SELECT warehouse_name,
       ROUND(SUM(credits_used_compute), 0) AS credits,
       ROUND(SUM(credits_used_compute - credits_attributed_compute_queries), 0) AS idle_credits,
       ROUND(100 * SUM(credits_used_compute - credits_attributed_compute_queries)
             / NULLIF(SUM(credits_used_compute), 0), 1) AS pct_idle
FROM snowflake.account_usage.warehouse_metering_history
WHERE start_time >= DATEADD(day, -90, CURRENT_TIMESTAMP())
GROUP BY 1 HAVING credits > 100 ORDER BY idle_credits DESC;

-- Hour-of-week demand shape: reveals bimodality (a scheduling structure)
SELECT warehouse_name,
       DAYOFWEEK(start_time) AS dow, HOUR(start_time) AS hr,
       AVG(credits_used_compute) AS avg_credits
FROM snowflake.account_usage.warehouse_metering_history
WHERE start_time >= DATEADD(day, -28, CURRENT_TIMESTAMP())
GROUP BY 1,2,3 ORDER BY 1,2,3;
```

**The move to Layer 3:** ask *what keeps producing this?* A pattern that persists through staff changes, incident reviews and good intentions is being held in place by something structural.

## Layer 3 — Structures

**What it is:** the configuration, schedules, permissions, ownership and contracts that generate the pattern. This is where most durable fixes live, and where most cost work should concentrate.

Structures in a Snowflake estate fall into four kinds, and it is worth checking all four because teams reliably look only at the first:

| Kind | Examples | How to see it |
|---|---|---|
| **Configuration** | `AUTO_SUSPEND`, size, `MIN_CLUSTER_COUNT`, `SCALING_POLICY`, statement timeouts, retention, table type | `SHOW WAREHOUSES`, `SHOW PARAMETERS`, table DDL |
| **Temporal** | Task schedules, dbt cadence, replication frequency, dynamic table lag, orchestrator thread counts | Task and pipe definitions, orchestrator config |
| **Organisational** | Who may `CREATE WAREHOUSE`, whether teams see their own spend, whether anyone owns a warehouse at all | RBAC grants, tag coverage, the unattributed share |
| **Contractual** | Edition per account, region, capacity commitment and its end date | Contract, `USAGE_IN_CURRENCY_DAILY` divided by credits |

**Probes:**
- What is the current value, and **who set it, when, and why?** Snowflake keeps no configuration history — if you are not snapshotting daily, you cannot answer this, and that gap is itself a structural finding.
- Is this value chosen or inherited? An account-level default reaching 40,000 tables was a decision made once for a handful of them.
- Does anyone own this object? Untagged spend is not an attribution problem, it is an ownership problem wearing a technical costume.
- Is there a control? A warehouse with no resource monitor and no statement timeout has no ceiling at all.

```sql
-- Configuration structures, all at once
SHOW WAREHOUSES;
SELECT "name", "size", "type", "min_cluster_count", "max_cluster_count",
       "scaling_policy", "auto_suspend", "auto_resume", "resource_monitor", "owner"
FROM TABLE(RESULT_SCAN(LAST_QUERY_ID())) ORDER BY "name";

-- Organisational structure: how much spend has no owner?
SELECT ROUND(100 * SUM(IFF(warehouse_name IS NULL OR query_tag = '', credits_attributed_compute, 0))
             / NULLIF(SUM(credits_attributed_compute), 0), 1) AS pct_unattributed
FROM snowflake.account_usage.query_attribution_history
WHERE start_time >= DATEADD(day, -30, CURRENT_TIMESTAMP());
```

**The move to Layer 4:** ask *why is the structure like that, and why has nobody changed it?* Structures persist because someone benefits, or because changing them carries a cost nobody wants to bear. That asymmetry is the mental model.

## Layer 4 — Mental models

**What it is:** the beliefs and incentives that make the structure feel normal. You cannot query this layer; you infer it from what the structures imply and confirm it by asking people.

**Probes:**
- What would have to be believed for this structure to be the obvious choice?
- What happens to the person who changes it? If downsizing means owning the outage risk while the savings accrue to a budget they do not control, the structure is rational for them and no amount of reporting will move it.
- What is measured, and what is rewarded? Estates optimize for what is visible. If nobody sees cost per unit of work, nobody manages it.
- What did someone say when the change was last proposed? The objection is the mental model, stated aloud.

**The recurring ones**, worth testing explicitly:

| Belief | What it produces | The counter-evidence |
|---|---|---|
| Bigger warehouse = faster = better | Peak-sized warehouses running 24×7; sizes only ever revised upward | Load percent, spill and queue data showing headroom |
| Idle is negligible | 600s auto-suspend inherited estate-wide | The idle ratio, in dollars per year |
| Serverless is managed, so it is cheap | Clustering and MVs enabled and never reviewed | The 2× multiplier, and read counts of zero |
| Storage is a rounding error | 90-day retention on high-churn tables | `(time_travel + failsafe) / active > 1.0` |
| Performance is the platform team's job; cost is finance's | No team owns a number | The unattributed share |
| Nobody is rewarded for downsizing | Drift back toward baseline within a year | Ask who was thanked last time someone did |

The last one is the most common and the most consequential. It explains why estates drift back 20–30% within a year of a successful optimization programme: the structural fixes were made, but the incentive that produced them was never touched.

## Knowing when to stop

Depth is not a virtue in itself. Two failure modes bracket this:

- **Stopping too shallow** — treating an event as a one-off, fixing it, and meeting it again next quarter. The tell is that the same fix has been applied before.
- **Going too deep for the question** — answering "should I lower auto-suspend on WH_ETL?" with an analysis of organisational incentives. It reads as profound and helps nobody.

The useful rule: **go down one layer past where the fix will hold.** If a config change stops the problem recurring, structure is deep enough; note the mental model in a sentence so the drift risk is on record, and move on. If the problem has recurred despite config fixes, the structure is not the cause and you must keep going.

State the layer you are operating at and why. "This is a symptom fix; the structural cause is X and I am not proposing to change it today because Y" is a legitimate and often correct position — it just has to be a choice rather than an oversight.
