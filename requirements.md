You have Dynatrace access for Edward Jones production. Analyse how CDDR's
BulkheadFullExceptions are behaving now, and whether three recent changes altered
that behaviour.

## The changes to assess

1. Resilience4j bulkhead `maxWaitDuration` 50ms -> 0
2. HPA `minReplicas` 4 -> 6
3. A fix to the `wmp-migration-api` recursive call cycle

Exact dates are unknown. Part of the task is to find them from the data.

Known boundary: CDDR moved from Hazelcast to Redis on the night of 2026-08-20,
so use `2026-08-21T00:00:00Z` as a candidate split point. Other config changes
may have shipped with that release.

## Environment

```
namespace       cddr-ns
clusters        aks04074prscu01, aks04074prncu01
log bucket      "cddr"
main workload   cddr-main-subgraph
```

Two things that will waste your time if you miss them:

- Always pass `bucket: "cddr"`. The default bucket holds about one day and will
  make you conclude that retention is short. The cddr bucket holds 30+ days.
- Filtering by `k8s.workload.name` returns empty. Use
  `k8s.namespace.name == "cddr-ns"`.

Use `scanLimitGBytes: -1` on wide queries.

## What is already established

CDDR uses a Resilience4j semaphore bulkhead named `topLevelBulkhead`,
`maxConcurrentCalls` 50, `maxWaitDuration` 0. When outbound HTTP calls time out,
threads are held, the bulkhead fills, and new requests are rejected with
`BulkheadFullException`, surfacing as `CustomDataFetchingException`.

`AppHttpRequestRetryStrategy - Got retryable IO` WARN lines are outbound HTTP
timeouts and name the target host.

Eight spike hours in the previous 30 days, every other hour at or below 584:

```
Hour (UTC)      retryable IO   gna-accounts   rms-rltshp   BulkheadFull
Aug 10 18:00           9,355            115        8,127         42,809
Aug 13 17:00           5,904          1,597        1,261          1,013
Aug 18 15:00           1,991            810          523            805
Aug 27 17:00           2,091            836          570            362
Aug 26 14:00           1,510            920          264            310
Aug 17 20:00           2,442            876          366            171
Aug 14 01:00           1,809            619          256              0
Aug 14 17:00           1,125             70            1              0
```

Bulkhead counts track `rms-rltshp-svc`, not `gna-accounts-svc`. The two hours with
zero bulkhead exceptions are the two lowest `rms-rltshp` counts.

Measured over Aug 10 18:09-18:26:

```
rms-rltshp-svc     8,122 timeouts   avg 4,019ms   range 4,001-4,420ms   1.00 calls/request
gna-accounts-svc     107 timeouts   avg 5,351ms   range 4,005-11,653ms  1.11 calls/request
```

8,127 timeouts over 18 minutes is roughly 7.5 per second. At 4 seconds held each,
that is about 30 of the 50 bulkhead slots occupied continuously, which explains the
saturation.

Aug 10 18:09-18:26 was the largest incident: 42,809 BulkheadFullExceptions across
6 pods, sustained above 1,000 per minute for 17 minutes. No alert fired on it.

## The problem with a naive before/after comparison

Raw exception counts measure how often downstream services broke, not the effect of
the three changes. Aug 10 produced 42,809 because `rms-rltshp-svc` timed out 8,127
times; Aug 14 produced zero because it timed out 256 times. The bulkhead
configuration was identical on both days.

The three changes also push in opposite directions:

```
maxWaitDuration 50ms -> 0    MORE exceptions   requests fail instead of waiting
minReplicas 4 -> 6           FEWER exceptions  more pods, more total slots
wmp cycle fix                FEWER exceptions  fewer concurrent calls per request
```

So a flat count after the change could mean nothing changed, or two effects
cancelling. Report the ratio of BulkheadFullException to `Got retryable IO`
alongside the raw counts, because that controls for the trigger.

## Questions

1. Daily `BulkheadFullException` count and daily `Got retryable IO` count for the
   last 30 days, side by side, with the ratio between them.

2. The same figures aggregated for the two periods: before `2026-08-21T00:00:00Z`
   and after.

3. Confirm the Redis release happened when expected. Every `cddr-main-subgraph` pod
   name with its first log timestamp between 2026-08-20 12:00 and 2026-08-21 12:00
   UTC. A full rollout means all pods restarting together.

4. Date the `minReplicas` change. For each day in the last 30 days, the minimum
   number of distinct `cddr-main-subgraph` pods logging in any hour. That
   approximates the replica floor. Identify the day it moves from 4 to 6.

5. Date the wmp cycle fix. Count log lines referencing `wmp-migration-api` per day
   for the last 30 days. If a recursive call cycle was removed, the volume should
   step down on one specific day.

6. Deployment boundaries. For each `cddr-main-subgraph` pod, its first log
   timestamp; grouped by day, how many new pod names appeared. Days where several
   appear at once are deployments. List those dates and times.

7. Has the shape changed? For every `BulkheadFullException` in the last 30 days,
   the pod's age at the time of the exception, derived from the most recent
   `Started CddrMainSubgraphApplication` log line for that pod before the exception.
   Roughly 40 seconds means cold start; hours or days means an established pod.
   Report the split per period.

`maxWaitDuration` is not detectable from logs, since nothing records whether a
request waited 50ms before failing. Say so rather than inferring it.

## Rules

Show the query used for each answer. Tables and numbers only.

If something is not available, write "not available" and move on. Do not infer, do
not fill gaps, do not construct a causal narrative to cover missing data. Label any
claim that is not directly measured.
