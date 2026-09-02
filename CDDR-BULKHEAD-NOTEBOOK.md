# CDDR bulkhead exceptions - Dynatrace notebook

One section per claim. Paste each block into its own notebook cell so the numbers
can be reproduced independently.

Two things that will produce wrong answers if changed: the `bucket` argument, and
`k8s.namespace.name` as the filter. `k8s.workload.name` returns empty.

Bucket note: the `cddr` bucket currently covers 2026-08-21T15:35 onward. Sections
that need data before that date query `bucket:{"cddr","default"}` to pick it up
from the default bucket's longer-lived index.

Timeframes below use `now()-30d`. That is right for the sections meant to be re-run,
but wrong for the ones documenting a finding - Aug 10 leaves a rolling 30-day window
in mid September and takes the evidence with it. Pin absolute dates on sections 2 and
3 before saving this as a permanent reference, and export the result tables. The
`Got retryable IO` logs have already aged out once; the numbers are the record, not
the queries.

---

## 0. Has this happened lately

Run this first. It answers "when did we last see one, and how long have we been
clean" without needing to interpret an empty result.

```
fetch logs, from: now()-30d, to: now(), scanLimitGBytes: -1, samplingRatio: 1, bucket:{"cddr"}
| filter k8s.namespace.name == "cddr-ns"
| filter matchesPhrase(content, "BulkheadFullException")
| summarize total           = count(),
            last_exception  = max(timestamp),
            first_exception = min(timestamp),
            pods            = countDistinct(k8s.pod.name),
            clusters        = countDistinct(k8s.cluster.name)
| fieldsAdd days_since_last = round(toLong(now() - last_exception) / 86400000000000, decimals: 1)
```

**No rows returned means none in 30 days.** That is the answer, not a broken query -
confirm with the timeline below, which prints a row for every day whether or not
anything happened.

```
fetch logs, from: now()-30d, to: now(), scanLimitGBytes: -1, samplingRatio: 1, bucket:{"cddr"}
| filter k8s.namespace.name == "cddr-ns"
| filter matchesPhrase(content, "BulkheadFullException")
| makeTimeseries exceptions = count(default: 0), interval: 1d, by: {cluster = k8s.cluster.name}
```

`default: 0` is what makes quiet days show as zero instead of vanishing. A solid run
of zeros is a real finding and should be readable as one - "nothing since 27 Aug" is
worth as much as a spike, and without this you cannot tell it apart from a query that
returned nothing because it was wrong.

Widen `now()-30d` to `now()-90d` for a longer view, bearing in mind the `cddr` bucket
currently only reaches 2026-08-21 and older data comes from `bucket:{"cddr","default"}`.

## 0a. Which pods, which cluster, how long after startup

Answers "when did we last see one, on which cluster, and how old was the pod" in a
single result.

```
fetch logs, from: now()-7d, to: now(), scanLimitGBytes: -1, samplingRatio: 1, bucket:{"cddr"}
| filter k8s.namespace.name == "cddr-ns"
| filter matchesPhrase(content, "BulkheadFullException")
| summarize exceptions      = count(),
            first_exception = min(timestamp),
            last_exception  = max(timestamp),
            by: {cluster = k8s.cluster.name, pod = k8s.pod.name, workload = k8s.deployment.name.clean}
| lookup [
    fetch logs, from: now()-7d, to: now(), scanLimitGBytes: -1, samplingRatio: 1, bucket:{"cddr"}
    | filter k8s.namespace.name == "cddr-ns"
    | filter matchesPhrase(content, "Started CddrMainSubgraphApplication")
    | summarize startup_ts = max(timestamp), by: {pod = k8s.pod.name}
  ], sourceField: pod, lookupField: pod, prefix: "lu."
| fieldsAdd uptime_seconds = if(isNull(lu.startup_ts), null,
    else: toLong(first_exception - lu.startup_ts) / 1000000000)
| fieldsAdd uptime_minutes = if(isNull(uptime_seconds), null, else: round(uptime_seconds / 60, decimals: 1))
| fieldsAdd uptime_hours   = if(isNull(uptime_seconds), null, else: round(uptime_seconds / 3600, decimals: 2))
| fieldsAdd kind = if(isNull(uptime_seconds), "unknown",
    else: if(uptime_seconds < 60, "cold start", else: "established"))
| fields cluster, workload, pod, exceptions, kind,
         lu.startup_ts, first_exception, last_exception,
         uptime_seconds, uptime_minutes, uptime_hours
| sort last_exception desc
```

Empty result means none in seven days. Change `now()-7d` to widen.

`kind` is the column to read. `cold start` is expected on scale-up and the Istio
warmup change addresses it. `established` means a pod that had been serving fine
stopped coping, which warmup does not touch and which usually means something
downstream.

## 0b. When, by hour, and on which cluster

Shows whether it was one burst or something sustained, and whether both prod clusters
are involved.

```
fetch logs, from: now()-14d, to: now(), scanLimitGBytes: -1, samplingRatio: 1, bucket:{"cddr"}
| filter k8s.namespace.name == "cddr-ns"
| filter matchesPhrase(content, "BulkheadFullException")
| summarize exceptions = count(),
            pods       = countDistinct(k8s.pod.name),
            by: {hour = bin(timestamp, 1h), cluster = k8s.cluster.name}
| sort hour desc
```

Historically every incident has landed on one cluster at a time. If a row shows both,
that is new and worth saying so.

Swap `1h` for `1m` and narrow the timeframe to see the shape inside an incident. The
distinction that matters is a single sharp minute versus a rate sustained over many -
Aug 10 held above 1,000/minute for 17 minutes, which is a different problem from a
100ms burst at pod startup.

## 0c. Which downstream services were timing out

Only works while the retry logs are still in retention. They aged out of the older
window already, so run it soon after an incident rather than weeks later.

```
fetch logs, from: now()-7d, to: now(), scanLimitGBytes: -1, samplingRatio: 1, bucket:{"cddr"}
| filter k8s.namespace.name == "cddr-ns"
| filter contains(content, "Got retryable IO")
| parse content, "LD 'https://' LD:target_host '/' LD"
| summarize timeouts = count(), by: {target_host, hour = bin(timestamp, 1h)}
| sort hour desc, timeouts desc
```

`rms-rltshp-svc` has been the recurring one. Its timeouts cluster tightly at
4,001-4,420ms, and at volume that holds roughly 30 of the 50 bulkhead slots
continuously, which is what saturates it.

---

## 1. Daily exception count

Backs: the daily shape, and that Aug 10 dominates the period.

```
fetch logs, from: now()-30d, to: now(), scanLimitGBytes: -1, samplingRatio: 1, bucket:{"cddr","default"}
| filter k8s.namespace.name == "cddr-ns"
| filter matchesPhrase(content, "BulkheadFullException") or matchesPhrase(content, "Got retryable IO")
| fieldsAdd day = formatTimestamp(timestamp, format: "yyyy-MM-dd")
| fieldsAdd is_bulkhead = if(matchesPhrase(content, "BulkheadFullException"), 1, else: 0)
| fieldsAdd is_retry    = if(matchesPhrase(content, "Got retryable IO"), 1, else: 0)
| summarize bulkhead_count = sum(is_bulkhead), retry_count = sum(is_retry),
            by: {day, cluster = k8s.cluster.name}
| fieldsAdd ratio = if(retry_count > 0, round(bulkhead_count / retry_count, decimals: 3), else: null)
| sort day asc, cluster asc
```

Expected: Aug 10 at 42,819 against 10-1,013 on every other active day. `retry_count`
is 0 throughout - those logs have aged out of both buckets, so the ratio column
cannot be filled. Left in deliberately so nobody assumes it was never attempted.

Split by cluster so it is clear whether both prod clusters are affected or only one.
Drop `cluster` from the `by:` clause for the plain daily total.

---

## 2. Before / after 2026-08-21

Backs: the raw 98.4% drop, and the caveat that it is one outage.

```
fetch logs, from: now()-30d, to: now(), scanLimitGBytes: -1, samplingRatio: 1, bucket:{"cddr","default"}
| filter k8s.namespace.name == "cddr-ns"
| filter matchesPhrase(content, "BulkheadFullException")
| fieldsAdd period = if(timestamp < toTimestamp("2026-08-21T00:00:00Z"), "before", else: "after")
| summarize exceptions = count(), by: {period}
```

Expected: before 45,083, after 725.

Then the same excluding Aug 10, which is the number worth quoting:

```
fetch logs, from: now()-30d, to: now(), scanLimitGBytes: -1, samplingRatio: 1, bucket:{"cddr","default"}
| filter k8s.namespace.name == "cddr-ns"
| filter matchesPhrase(content, "BulkheadFullException")
| fieldsAdd day = formatTimestamp(timestamp, format: "yyyy-MM-dd")
| filter day != "2026-08-10"
| fieldsAdd period = if(timestamp < toTimestamp("2026-08-21T00:00:00Z"), "before", else: "after")
| summarize exceptions = count(), by: {period}
```

Expected: before 2,264 over 18 days (126/day), after 725 over 12 days (60/day).

---

## 3. Pod age at each exception

The main result. Backs: 99.6% of exceptions are on established pods, and the
bimodal distribution.

Age is computed per exception, not per pod. Classifying a pod by its first
exception and then attributing all of its exceptions to that class is what
produced the wrong answer on the first pass - one pod with an early cold-start
blip dragged an entire outage into the cold-start bucket.

```
fetch logs, from: now()-30d, to: now(), scanLimitGBytes: -1, samplingRatio: 1, bucket:{"cddr"}
| filter k8s.namespace.name == "cddr-ns"
| filter k8s.deployment.name.clean == "cddr-main-subgraph"
| filter matchesPhrase(content, "BulkheadFullException")
| fields exception_ts = timestamp, pod = k8s.pod.name, cluster = k8s.cluster.name
| lookup [
    fetch logs, from: now()-30d, to: now(), scanLimitGBytes: -1, samplingRatio: 1, bucket:{"cddr"}
    | filter k8s.namespace.name == "cddr-ns"
    | filter k8s.deployment.name.clean == "cddr-main-subgraph"
    | filter matchesPhrase(content, "Started CddrMainSubgraphApplication")
    | summarize startup_ts = max(timestamp), by: {pod = k8s.pod.name}
  ], sourceField: pod, lookupField: pod, prefix: "lu."
| fieldsAdd age_seconds = if(isNull(lu.startup_ts), null,
    else: toLong(exception_ts - lu.startup_ts) / 1000000000)
| fieldsAdd age_band = if(isNull(age_seconds), "age_unknown",
    else: if(age_seconds < 0,    "age_unknown",
    else: if(age_seconds < 60,   "under_60s",
    else: if(age_seconds < 600,  "1_to_10min",
    else: if(age_seconds < 3600, "10_to_60min",
    else: "over_1hour")))))
| summarize count = count(), by: {age_band}
```

Expected:

```
under_60s        195
1_to_10min         0
10_to_60min        0
over_1hour    44,512
age_unknown    1,101
```

Two caveats to state alongside this, because they are real:

- The lookup takes `max(timestamp)` per pod, which is the last restart in the
  window rather than the last restart before each exception. Where that produces a
  negative age the row falls into `age_unknown` rather than being silently dropped.
- The 1,101 `age_unknown` are Aug 17-20 exceptions on the `5f8ffc7cd7` replicaset,
  created 2026-08-07, before the cddr bucket's floor. Their startup is known from
  the default bucket to be Aug 7, making those pods 10-13 days old. They belong in
  `over_1hour`, which brings the corrected total to 44,938.

Split by period:

```
... same query up to age_band ...
| fieldsAdd period = if(exception_ts < toTimestamp("2026-08-21T00:00:00Z"), "before", else: "after")
| summarize count = count(), by: {period, age_band}
| sort period asc, age_band asc
```

Expected: before 145 under_60s / 43,837 over_1hour / 1,101 unknown;
after 50 under_60s / 675 over_1hour / 0 unknown.

And Aug 10 alone, which is the check that settles the cold-start question:

```
... same query up to age_band ...
| filter formatTimestamp(exception_ts, format: "yyyy-MM-dd") == "2026-08-10"
| summarize count = count(), by: {age_band}
```

Expected: 10 under_60s, 42,809 over_1hour. The 10 are pod `5f8ffc7cd7-gj574`,
which started 14:48:38 and fired at 14:48:50 - four hours before the 18:09 outage
and unrelated to it.

---

## 3b. Per-exception detail: cluster, pod, uptime at failure

The row-level view behind section 3. Use this when someone asks to see actual pods
and timings rather than buckets, or to check whether one prod cluster is carrying
the problem.

```
fetch logs, from: now()-30d, to: now(), scanLimitGBytes: -1, samplingRatio: 1, bucket:{"cddr"}
| filter k8s.namespace.name == "cddr-ns"
| filter k8s.deployment.name.clean == "cddr-main-subgraph"
| filter matchesPhrase(content, "BulkheadFullException")
| summarize exceptions      = count(),
            first_exception = min(timestamp),
            last_exception  = max(timestamp),
            by: {pod = k8s.pod.name, cluster = k8s.cluster.name}
| lookup [
    fetch logs, from: now()-30d, to: now(), scanLimitGBytes: -1, samplingRatio: 1, bucket:{"cddr"}
    | filter k8s.namespace.name == "cddr-ns"
    | filter k8s.deployment.name.clean == "cddr-main-subgraph"
    | filter matchesPhrase(content, "Started CddrMainSubgraphApplication")
    | summarize startup_ts = max(timestamp), by: {pod = k8s.pod.name}
  ], sourceField: pod, lookupField: pod, prefix: "lu."
| fieldsAdd uptime_seconds = if(isNull(lu.startup_ts), null,
    else: toLong(first_exception - lu.startup_ts) / 1000000000)
| fieldsAdd uptime_minutes = if(isNull(uptime_seconds), null,
    else: round(uptime_seconds / 60, decimals: 1))
| fieldsAdd uptime_hours   = if(isNull(uptime_seconds), null,
    else: round(uptime_seconds / 3600, decimals: 2))
| fields cluster, pod, exceptions, lu.startup_ts, first_exception, last_exception,
         uptime_seconds, uptime_minutes, uptime_hours
| sort first_exception asc
```

Read `uptime_seconds` as how long the pod had been running when it first rejected a
request. Seconds means cold start; hours or days means it had been serving fine and
something else changed.

`cluster` is the thing to check first - both prod clusters run CDDR, and if the
exceptions sit on one of them the cause is more likely local to that cluster than
to the application.

For the per-cluster totals rather than per-pod rows:

```
... same query up to age_band in section 3 ...
| summarize count = count(), by: {cluster, age_band}
| sort cluster asc, age_band asc
```

---

## 4. Replica floor per day

Backs: the minReplicas 4 to 6 change is not visible.

```
fetch logs, from: now()-30d, to: now(), scanLimitGBytes: -1, samplingRatio: 1, bucket:{"cddr","default"}
| filter k8s.namespace.name == "cddr-ns"
| filter k8s.deployment.name.clean == "cddr-main-subgraph"
| fieldsAdd hour = formatTimestamp(timestamp, format: "yyyy-MM-dd HH")
| fieldsAdd day  = formatTimestamp(timestamp, format: "yyyy-MM-dd")
| summarize distinct_pods_per_hour = countDistinct(k8s.pod.name), by: {day, hour}
| summarize min_pods = min(distinct_pods_per_hour), by: {day}
| sort day asc
```

Expected: 10 every day from 2026-08-18 onward, which is the earliest retained data.
The 4 to 6 transition predates it. Worth asking what `minReplicas` is actually set
to, since the floor never drops below 10.

---

## 5. wmp-migration-api volume per day

Backs: the recursive-call fix cannot be dated from available retention.

```
fetch logs, from: now()-30d, to: now(), scanLimitGBytes: -1, samplingRatio: 1, bucket:{"cddr"}
| filter k8s.namespace.name == "cddr-ns"
| filter contains(content, "wmp-migration-api")
| fieldsAdd day = formatTimestamp(timestamp, format: "yyyy-MM-dd")
| summarize count = count(), by: {day}
| sort day asc
```

Expected: first lines on 2026-08-25, then 33k-309k per day with no step-down. If
the fix landed before the 25th it is outside retention.

---

## 6. Current pod inventory and ages

Useful alongside section 0 - tells you which pods are new enough to still be warming
up, and therefore whether a cold-start exception was expected.

```
fetch logs, from: now()-3d, to: now(), scanLimitGBytes: -1, samplingRatio: 1, bucket:{"cddr"}
| filter k8s.namespace.name == "cddr-ns"
| filter matchesPhrase(content, "Started CddrMainSubgraphApplication")
| parse content, "LD 'in ' DOUBLE:boot_seconds ' seconds (process running for ' DOUBLE:process_seconds LD"
| summarize startup_ts     = max(timestamp),
            boot_seconds   = takeLast(boot_seconds),
            by: {cluster = k8s.cluster.name, pod = k8s.pod.name}
| fieldsAdd age_hours = round(toLong(now() - startup_ts) / 3600000000000, decimals: 2)
| fields cluster, pod, startup_ts, boot_seconds, age_hours
| sort startup_ts desc
```

`boot_seconds` is Spring Boot reporting its own startup time - typically 20-33
seconds. A pod restarting repeatedly shows up here as several recent rows for the
same replicaset.

The full startup line also carries `process running for`, which is JVM uptime at the
moment Spring finished. Subtracting it from the log timestamp reconstructs container
start to within a second, verified against the Kubernetes API. That is how pod ages
are derived anywhere the Kubernetes events have already aged out.

---

## What cannot be shown

`maxWaitDuration` 50ms to 0 is not detectable. No log line records whether a
request waited in the bulkhead queue before rejection - both settings emit the same
`BulkheadFullException` line. Nothing in the available data distinguishes them.

Per-request bulkhead saturation over time is also unavailable.
`resilience4j_bulkhead_available_concurrent_calls` is published on the pod's
`/actuator/prometheus` endpoint - 48 of 50 when checked - but is not ingested into
Dynatrace for CDDR. Other namespaces have theirs in Grail. Without it there is no way
to see how close the bulkhead runs to its limit under normal load, which is what
would answer whether 50 is the right number.
