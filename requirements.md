Using Dynatrace, analyse how CDDR's BulkheadFullExceptions are behaving now and
whether three recent changes altered that behaviour.

The changes: bulkhead `maxWaitDuration` 50ms -> 0, HPA `minReplicas` 4 -> 6, and a
fix to the `wmp-migration-api` recursive call cycle. Exact dates unknown - find them
from the data. CDDR moved from Hazelcast to Redis the night of 2026-08-20, so use
`2026-08-21T00:00:00Z` as the candidate split point.

## Environment

```
namespace     cddr-ns
clusters      aks04074prscu01, aks04074prncu01
log bucket    "cddr"
workload      cddr-main-subgraph
```

Two things that will waste your time: always pass `bucket: "cddr"` (the default
bucket holds about a day and will make you think retention is short), and filter on
`k8s.namespace.name == "cddr-ns"` because `k8s.workload.name` returns empty. Use
`scanLimitGBytes: -1` on wide queries.

## Context

`BulkheadFullException` fires when outbound HTTP calls time out, threads are held,
and the Resilience4j `topLevelBulkhead` (50 concurrent) fills. The
`AppHttpRequestRetryStrategy - Got retryable IO` WARN lines are those timeouts and
name the target host.

Raw exception counts measure how often downstream broke, not the effect of the
changes - one hour produced 42,809 exceptions and another produced zero with
identical config. So report the ratio of `BulkheadFullException` to `Got retryable
IO` alongside the counts.

The three changes also push in opposite directions: `maxWaitDuration` 0 causes more
exceptions (requests fail instead of waiting), while more replicas and fewer
recursive calls cause fewer. A flat count could mean two effects cancelling.

## Questions

1. Daily `BulkheadFullException` count and daily `Got retryable IO` count for the
   last 30 days, side by side, with the ratio.

2. The same, aggregated before and after `2026-08-21T00:00:00Z`.

3. Per day, the minimum number of distinct `cddr-main-subgraph` pods logging in any
   hour. That approximates the replica floor - identify the day it moves 4 to 6.

4. Per day, count of log lines referencing `wmp-migration-api`. If a recursive cycle
   was removed the volume should step down on one day.

5. For every `BulkheadFullException` in the period, the pod's age when it fired,
   derived from the most recent `Started CddrMainSubgraphApplication` line for that
   pod beforehand. ~40 seconds means cold start, hours or days means an established
   pod. Report the split per period.

`maxWaitDuration` is not detectable from logs - nothing records whether a request
waited before failing. Say so rather than inferring it.

## Rules

Show the query used for each answer. Tables and numbers only.

If something is not available, write "not available" and move on. Do not infer, do
not fill gaps, do not build a causal narrative around missing data.
