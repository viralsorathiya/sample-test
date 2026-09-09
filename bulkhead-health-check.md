# CDDR bulkhead health check

Query 1 tells you which days had exceptions. Query 2 tells you whether they were cold
start (minor) or established pods (serious). Query 3 names what was timing out.

State as at 2026-09-08:

```
day (UTC)   exceptions   pods   window (local)
Sep 8               12      2   08:39 - 11:47
Sep 3            1,165     10   09:17 - 12:17
Sep 2              687      6   08:56 - 11:28
Aug 27             412     10   07:38 - 10:37
Aug 26             310      4   07:24 - 07:25
Aug 21               3      1   07:28
Aug 20              38      5   12:04 - 12:52
```

All on `aks04074prscu01`. `prncu01` has had zero throughout.

Every event falls in the morning, roughly 07:00 to 12:30. Sep 2 and Sep 3 are the
largest since the 20 Aug deploy, and Sep 3 affected all 10 pods for three hours.

For scale: before the deploy there were 45,083 exceptions in 30 days, including
42,819 on 10 Aug alone.

---

## 1. Which days had bulkhead exceptions

```
fetch logs, from: now()-21d, to: now(), scanLimitGBytes: -1, samplingRatio: 1, bucket:{"cddr"}
| filter k8s.namespace.name == "cddr-ns"
| filter matchesPhrase(content, "BulkheadFullException")
| summarize exceptions = count(),
            pods       = countDistinct(k8s.pod.name),
            first_exc  = min(timestamp),
            last_exc   = max(timestamp),
            by: {day = bin(timestamp, 1d), cluster = k8s.cluster.name}
| sort day desc
```

Days with no exceptions do not appear. So:

```
no rows at all           clean for 21 days
newest row is 27 Aug     clean since then
a recent row             still happening - run query 2
```

---

## 2. Cold start or not

This is the question that decides whether it matters. Change the two dates in the
FIRST fetch to the day you are investigating. Leave the lookup at `now()-21d` - it
needs a wide window to find the startup line for pods that have been running a while.

### 2a. The direct answer

```
fetch logs, from: "2026-09-03T00:00:00Z", to: "2026-09-04T00:00:00Z", scanLimitGBytes: -1, samplingRatio: 1, bucket:{"cddr"}
| filter k8s.namespace.name == "cddr-ns"
| filter contains(k8s.pod.name, "cddr-main-subgraph")
| filter matchesPhrase(content, "BulkheadFullException")
| fields exception_ts = timestamp, pod = k8s.pod.name, cluster = k8s.cluster.name
| lookup [
    fetch logs, from: now()-21d, to: now(), scanLimitGBytes: -1, samplingRatio: 1, bucket:{"cddr"}
    | filter k8s.namespace.name == "cddr-ns"
    | filter contains(k8s.pod.name, "cddr-main-subgraph")
    | filter matchesPhrase(content, "Started CddrMainSubgraphApplication")
    | summarize startup_ts = max(timestamp), by: {pod = k8s.pod.name}
  ], sourceField: pod, lookupField: pod, prefix: "lu."
| fieldsAdd age_seconds = if(isNull(lu.startup_ts), null,
    else: toLong(exception_ts - lu.startup_ts) / 1000000000)
| fieldsAdd age_band = if(isNull(age_seconds), "no_startup_line_found",
    else: if(age_seconds < 0,    "no_startup_line_found",
    else: if(age_seconds < 60,   "COLD START (under 60s)",
    else: if(age_seconds < 3600, "under 1 hour",
    else: "ESTABLISHED (over 1 hour)"))))
| summarize exceptions = count(), pods = countDistinct(pod), by: {age_band}
| sort exceptions desc
```

How to read it:

```
COLD START             pods rejecting traffic seconds after starting.
                       Happens on scale-up. Usually about 10 per pod.
                       Istio warmup is the fix. Low priority.

ESTABLISHED            pods that had been serving fine for hours or days.
                       A downstream service was timing out.
                       This is the one that matters - go to query 3.

no_startup_line_found  the pod started before the 21 day window, so it had
                       been running at least that long. Treat as ESTABLISHED.
```

### 2b. The seconds, per pod

Same query, but showing the actual ages so you can see how tight the clustering is
instead of trusting the band. Replace the last two lines of 2a with:

```
| summarize exceptions   = count(),
            earliest_age = min(age_seconds),
            latest_age   = max(age_seconds),
            first_exc    = min(exception_ts),
            last_exc     = max(exception_ts),
            startup      = takeAny(lu.startup_ts),
            by: {cluster, pod}
| sort first_exc asc
```

Cold start shows ages clustered in a narrow band, typically 7 to 16 seconds, with
roughly 10 exceptions per pod. Established shows ages in the thousands or hundreds of
thousands of seconds, and usually far more exceptions per pod.

For the raw list, one row per exception:

```
| fields cluster, pod, startup = lu.startup_ts, exception_ts, age_seconds
| sort exception_ts asc
```

---

## 3. What was timing out (only if query 2 shows ages in hours or days)

```
fetch logs, from: now()-3d, to: now(), scanLimitGBytes: -1, samplingRatio: 1, bucket:{"cddr"}
| filter k8s.namespace.name == "cddr-ns"
| filter contains(content, "Got retryable IO")
| parse content, "LD 'https://' LD:target_host '/' LD"
| summarize timeouts = count(), by: {hour = bin(timestamp, 1h), target_host}
| sort hour desc, timeouts desc
```

Match the hour against query 2. The host with the spike is the cause.

`rms-rltshp-svc` has been the recurring one. It times out at 4 seconds and CDDR calls
it once per request, so at volume it holds around 30 of the 50 bulkhead slots.

---

## Reporting it

If query 1 comes back clean, the accurate line is:

```
No bulkhead exceptions since 27 Aug.
```

Not "the problem is fixed". The exceptions were driven by rms-rltshp-svc timing out.
If that service has simply had a good fortnight, the quiet belongs to them, and CDDR
will fail the same way next time it goes bad.
