# Snowflake Optimization

Design and delivery plan for an in-house Snowflake cost-optimization platform,
targeting a ~$40M/year Snowflake estate.

## Contents

| File | What it is |
|---|---|
| `snowflake-credit-control.html` | The full program document — open it in a browser. |

## What the document covers

1. **Billing mechanics** — the six rules that govern every optimization
   (warehouse-seconds vs query-seconds, size/generation multipliers, serverless
   multipliers, the cloud-services 10% rebate, storage as a daily average).
2. **Tunable parameter catalog** — 62 levers across 8 cost domains, each with its
   credit mechanic, typical waste signature, savings potential, effort and
   reversibility.
3. **Deep dives** — per-lever anatomy: how cost accrues, how teams actually use
   it, the failure modes, the fix, and the detection SQL.
4. **Query performance as cost** — the diagnostic ladder (pruning → spill →
   queueing → cache → compilation → row explosion → repetition), a SQL
   anti-pattern catalog, and the acceleration decision matrix
   (clustering / search optimization / materialized views / QAS / Optima).
5. **Waste taxonomy** — 20 ranked recurring patterns with detection signals.
6. **Tool architecture** — metering warehouse, attribution engine, detector
   library, action layer.
7. **Analyser catalog** — 40 analysers specified in full: method, thresholds,
   veto dependencies, false-positive guards, and emitted findings.
8. **Phased roadmap** — six phases over twelve months, with a coverage matrix
   proving all 62 levers have a phase.
9. **Operating model, risks, and a week-one SQL pack.**

## Caveats

- Credit rates, serverless multipliers and per-credit prices are **list or
  published figures as of September 2026**. Verify against your own contract and
  the current Snowflake Service Consumption Table before committing to a number.
- Every SQL snippet uses a **$3.20/credit placeholder**. Replace it with a join
  to your own rate card before any figure leaves the team.
- The spend decomposition in section 00 is a **planning estimate**, not measured.
  Phase 0 exists to replace it with reconciled actuals.
