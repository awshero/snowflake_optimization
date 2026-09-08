---
name: snowflake-cost-reasoning
description: >-
  Applies first-principles, second-order, and iceberg (systems) thinking to Snowflake cost and
  credit-consumption problems. Use this whenever someone asks why Snowflake spend is high or rising,
  whether to change a warehouse setting, whether a proposed optimization will actually save money,
  how to diagnose a bill spike, or how to build a cost-reduction plan — and also whenever they
  mention credits, warehouse sizing, auto-suspend, multi-cluster settings, serverless features,
  clustering, Time Travel, storage growth, or a Snowflake invoice, even if they never use the word
  "analysis". Especially valuable when a proposed fix looks obviously correct, because this skill
  exists to catch the Snowflake optimizations that backfire.
---

# Snowflake cost reasoning

## Why this skill exists

Snowflake cost problems punish shallow reasoning more than most domains, for three specific reasons:

1. **The obvious fix is often wrong, and wrong in the expensive direction.** Downsizing a warehouse that is spilling to remote storage makes it slower by more than the rate falls, so the bill goes *up*. Tightening auto-suspend past the arrival cadence causes resume churn against a 60-second minimum, so the bill goes *up*. These are not exotic edge cases; they are the two most common first moves.

2. **The visible number is rarely the real problem.** A March invoice spike is an event. The thing generating it is usually a structure — a config default inherited by 400 warehouses, a schedule nobody revisits, an ownership gap — and underneath that, a belief, like "bigger is faster" or "nobody gets promoted for downsizing."

3. **Credits and dollars are different currencies.** Under a capacity commitment, cutting consumption 20% can save exactly zero dollars this year. A recommendation that ignores this is not merely incomplete; it will be judged a failure by finance in month nine.

Three frameworks address these three failures, and they compose: **diagnose downward, rebuild from bedrock, project forward.**

```
ICEBERG        ↓   Where does this problem actually live?
                   event → pattern → structure → mental model

FIRST PRINCIPLES ↓ At that level, what is irreducibly true
                   once inherited assumptions are stripped?

SECOND ORDER   →   If we act on that: and then what?
                   who adapts, what breaks, what does finance see?
```

Run them in that order. The order matters: rebuilding before you have diagnosed produces an elegant answer to the wrong problem, and projecting forward from an unexamined premise just makes a bad plan more confident.

---

## Pass 1 — Iceberg: find the level the problem lives on

Most cost conversations happen at the event level, which is why most cost fixes do not hold. Work downward until you reach a level where a change would actually stop the problem recurring. Each layer needs **evidence**, not speculation — if you have query access, go get it.

| Layer | In Snowflake terms | Example | Where the evidence lives |
|---|---|---|---|
| **Event** | One bill, one spike, one expensive query | "March was $4.1M, up 18%" | `METERING_DAILY_HISTORY`, the invoice |
| **Pattern** | The recurring shape over time | "It spikes every quarter-end"; "idle has been ~35% for a year" | `WAREHOUSE_METERING_HISTORY` trended, hour-of-week profiles, `QUERY_HISTORY` by `query_parameterized_hash` |
| **Structure** | Config, schedules, ownership, incentives, contract | "`AUTO_SUSPEND=600` inherited estate-wide"; "`CREATE WAREHOUSE` granted broadly"; "no team sees its own number"; "capacity commitment runs to March" | `SHOW WAREHOUSES`, task schedules, RBAC grants, tag coverage, the contract |
| **Mental model** | The belief that keeps producing the structure | "Bigger is faster"; "storage is cheap"; "the person who downsizes owns the outage and none of the savings" | What gets rewarded, what people say when you propose a change |

Two things to hold onto:

**Leverage increases with depth, and so does difficulty.** Changing `AUTO_SUSPEND` on one warehouse is a structural fix worth real money and takes seconds. Changing "nobody is rewarded for downsizing" is a mental-model fix worth far more and takes a year. Name the level you are operating at, and be honest when you are treating a symptom because the deeper fix is out of scope — that is often the right call, but it should be a stated choice rather than an unexamined one.

**The most common mental models in Snowflake estates**, worth testing explicitly because they are rarely said out loud:
- *Bigger warehouse = faster = better.* Survives because upsizing is never punished and downsizing carries outage risk.
- *Idle is free, or too small to matter.* It is typically the single largest recoverable line.
- *Serverless means managed, and managed means cheap.* Clustering, search optimization and materialized views bill at a **2× multiplier**.
- *Storage is a rounding error.* Time Travel plus Fail-safe can exceed the table itself on high-churn data.
- *Our team's spend is someone else's problem.* Produced by the absence of attribution, not by apathy.

---

## Pass 2 — First principles: strip to what is irreducibly true

Snowflake cost has an unusually solid bedrock. Almost every claim in a cost conversation can be checked against these identities, and most disagreements dissolve when you do:

```
warehouse_credits  = size_rate(size, generation) × running_seconds × clusters_running
running_seconds    = query_seconds + idle_seconds          ← idle bills identically to work
every resume       = max(actual_seconds, 60) billed        ← the 60-second minimum
serverless_credits = compute_hours × feature_multiplier    ← 2× clustering/SOS/MV,
                                                             1.25× Snowpipe (+ per-file),
                                                             1× QAS / Snowpipe Streaming
storage_cost       = avg_daily_bytes(active + time_travel + failsafe + clone_retained) × $/TB
dollars            = credits × $/credit(edition, region, contract)
```

Reason *up* from these rather than across from convention. Some things that follow immediately, and that people get wrong constantly:

- **Halving a query's runtime at a fixed size halves its cost. Halving runtime by doubling the size saves nothing.** Every performance recommendation must say which of the two it is.
- **Idle is billed identically to work**, so the auto-suspend tail is pure loss — and it is invisible in per-query views, which is why per-query cost tools understate the bill.
- **A 5-second task on a warehouse bills 60 seconds.** Frequency multiplies that: 1,440 runs a day at 12× overhead.
- **Serverless is convenient, not cheap.** An hour of reclustering costs two hours of equivalent warehouse compute.
- **Dropping a table does not stop the bill.** Fail-safe keeps charging for 7 days on permanent tables and cannot be disabled.
- **Credits are not dollars.** Edition, region and contract sit between them, and under a commitment the conversion this year may be zero.

**How to strip an assumption.** When someone says "we need a 2XL for this," the assumption is doing the work, not the evidence. Ask what the query actually requires: is it spilling (memory-bound), queueing (concurrency-bound), or neither (over-provisioned)? Ask what changed since the 2XL was chosen, and whether anyone has tested a smaller size since. The useful question is rarely "is 2XL right?" but "what evidence would tell us, and do we have it?"

Reasoning by analogy — "the other team uses an XL, so we should" — is the specific failure mode here. It is how sizing decisions propagate through an estate without anyone ever measuring anything.

---

## Pass 3 — Second order: and then what?

This is where the skill earns its keep. For every proposed change, trace the consequences of the consequences before recommending it. Four questions, in this order:

1. **Mechanically, and then what?** If auto-suspend drops to 30s and queries arrive every 45s, the warehouse resumes constantly, each resume bills a 60-second minimum, and the cache is thrown away. Cost rises and performance falls.
2. **Who adapts, and how?** Systems contain people. Put ECONOMY scaling on an interactive warehouse and analysts wait; some will rerun queries, some will escalate, some will build a private extract elsewhere. The credits saved may reappear as salary spent.
3. **What does this look like in two years?** A clustering key added to fix pruning that a `WHERE`-clause rewrite would have fixed becomes a permanent 2× reclustering charge that nobody remembers approving.
4. **What does finance actually see?** Credits avoided and dollars realized are different numbers under a commitment. Say which one you are claiming.

**The traps worth knowing by heart** — read `references/second-order-traps.md` for the full catalog with mechanisms and the correct move. The most costly recurring ones:

| The obvious move | And then what |
|---|---|
| Downsize a warehouse that is spilling | It spills harder and runs more than 2× longer. Cost rises, and jobs may now fail. Spilling is a *query* problem. |
| Tighten auto-suspend aggressively | Resume churn against the 60-second minimum plus cache loss. Cost can rise. |
| Enable Query Acceleration for performance | It bills separately at 1×. Unless you *also* downsize the warehouse it made safe to shrink, you have added a line item. |
| Migrate the fleet to Gen2 | A flat 25–35% rate increase on every workload that does not get ~26% faster. |
| Cut consumption under a capacity commitment | The commit burns down slower. Cash saving this year: zero, unless renegotiated or reallocated. |
| Consolidate warehouses to cut idle | The merged warehouse must be sized for its worst query and kept warm for its most impatient consumer. |
| Add a materialized view to speed a query | Maintenance is triggered by base-table churn at 2×. On a fast-changing table it exceeds the saving. |
| Suspend a warehouse via resource monitor | If it lands on month-end close, you lose the program's mandate. The second-order effect is political, and it is fatal. |

Note the last one. **Second-order effects on the cost programme itself are as real as effects on the bill.** One wrongful suspension buys a year of "no, you may not automate that."

---

## Scaling effort to the question

Match the depth of the work to the stakes. A skill that produces a 2,000-word framework analysis for "should I lower auto-suspend on WH_ETL?" is worse than useless — people stop asking.

**Small questions** — one setting, one warehouse, a yes/no. Answer in a few tight paragraphs. Run all three passes in your head; show only what changes the answer. The frameworks should be *invisible but present*: no headers, no framework names, just reasoning that happens to have checked the right things. Always surface the second-order check, even here — it is usually the part the asker had not done.

> *Yes, but not to 30s. Your queries arrive about every 90 seconds during the batch window, so a 30s suspend would resume roughly 40 extra times an hour and each resume bills a full minute — you would pay more and lose the cache. 120s is the number: it covers the common gaps and still suspends on the long tail. Expect ~$18k/year, and check it against 7 days of actual metering before you trust the figure.*

**Big questions** — a spike investigation, a programme, a multi-lever decision. Use the structure below, and make the framework passes explicit so a reader can follow and challenge the reasoning:

```markdown
## What's actually happening
(iceberg — work down to the level the problem lives on, with evidence per layer)

## What we know to be true
(first principles — the identities that apply, assumptions stripped, what the
 evidence actually supports)

## And then what
(second order — consequence chains for each candidate move, including the ones
 that make the situation worse)

## What this means you should do
(ranked, with dollars, reversibility, and what would falsify the recommendation)
```

---

## What good output looks like

**Quantify or say you cannot.** "This is wasteful" is not usable. "~$41k/year at $3.20/credit, which you should replace with your contract rate" is. When you lack data, state the estimate's basis and what query would settle it.

**Name reversibility.** `AUTO_SUSPEND` reverts in seconds. Reducing `DATA_RETENTION_TIME_IN_DAYS` destroys history immediately and is a one-way door. Treating those as comparable recommendations is how cost programmes cause incidents.

**State what would prove you wrong.** A recommendation with no falsifier is an opinion. "Downsize to L; if p95 runtime rises more than 30% or any remote spill appears, revert" is a recommendation someone can safely act on.

**Separate credits avoided from dollars realized** whenever a commitment exists.

**Do not manufacture depth.** If a question genuinely lives at the event level and the fix is one setting, say so. Dragging every question down to mental models is its own failure mode — it reads as profound and helps nobody.

---

## Reference files

Read these when the question warrants the detail:

- **`references/cost-model.md`** — the billing identities in full, credit rates by size and generation, serverless multipliers, storage mechanics, and the `ACCOUNT_USAGE` views that supply evidence for each. Read this whenever you need to compute or check a number.
- **`references/second-order-traps.md`** — the full catalogue of Snowflake optimizations that backfire, each with the mechanism, the signal that distinguishes it, and the correct move. Read this before recommending any change.
- **`references/iceberg-probes.md`** — the specific questions and queries that surface each iceberg layer, including how to tell a structural cause from a pattern. Read this when diagnosing a spike or a recurring problem.

Rates and multipliers are list figures and drift. Treat them as approximations to be checked against the current Snowflake Service Consumption Table and the user's own contract, and say so when a number carries weight.
