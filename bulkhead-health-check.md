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

## 3. What was timing out

Run this when query 2 comes back mostly ESTABLISHED. Dates pinned to Sep 3.

```
fetch logs, from: "2026-09-03T00:00:00Z", to: "2026-09-04T00:00:00Z", scanLimitGBytes: -1, samplingRatio: 1, bucket:{"cddr"}
| filter k8s.namespace.name == "cddr-ns"
| filter contains(content, "Got retryable IO")
| parse content, "LD 'https://' LD:target_host '/' LD"
| summarize timeouts = count(), by: {hour = bin(timestamp, 1h), target_host}
| sort hour asc, timeouts desc
```

The exceptions ran 09:17 to 12:17 local. Look for a host spiking in those hours.

`rms-rltshp-svc` has been the driver every previous time. It times out at 4 seconds
and CDDR calls it once per request, so at volume it holds around 30 of the 50
bulkhead slots. If it appears again that is five occurrences, which is a pattern
rather than a one-off.

Sep 3 was also the day BPP could not authenticate with ForgeRock, so
`bpp-account-svc` may show up. Treat that with suspicion as a cause - those 502s
returned in 54ms, and a call failing that fast does not hold a bulkhead slot long
enough to matter.

### 3b. Minute by minute, if you need to line it up precisely

```
fetch logs, from: "2026-09-03T14:00:00Z", to: "2026-09-03T18:00:00Z", scanLimitGBytes: -1, samplingRatio: 1, bucket:{"cddr"}
| filter k8s.namespace.name == "cddr-ns"
| filter contains(content, "Got retryable IO") or matchesPhrase(content, "BulkheadFullException")
| fieldsAdd kind = if(matchesPhrase(content, "BulkheadFullException"), "bulkhead", else: "downstream_timeout")
| summarize count = count(), by: {minute = bin(timestamp, 1m), kind}
| sort minute asc
```

Shows whether the downstream timeouts start before the bulkhead fills. They should -
threads have to be held before the pool runs out.

---

## 4. Raw log line - does it record how long the call took

Run this next. Need to see whether the timeout duration is in the line.

```
fetch logs, from: "2026-09-03T15:00:00Z", to: "2026-09-03T17:00:00Z", scanLimitGBytes: -1, samplingRatio: 1, bucket:{"cddr"}
| filter k8s.namespace.name == "cddr-ns"
| filter contains(content, "Got retryable IO")
| filter contains(content, "mfd-vndr-accts-rest")
| fields timestamp, content
| limit 3
```

Open one row and read the full text. Looking for a duration, an elapsed time, or a
configured timeout value.

Why it matters: Sep 3 had 1,188 timeouts against `mfd-vndr-accts-rest` and filled the
bulkhead. Sep 7 had 970 against `rms-rltshp-svc` and did not. Similar volume, opposite
outcome - so how long each call holds a thread, or how many times CDDR calls it per
request, is the difference.

### 4b. Calls per request, from the callChart lines

```
fetch logs, from: "2026-09-03T15:00:00Z", to: "2026-09-03T17:00:00Z", scanLimitGBytes: -1, samplingRatio: 1, bucket:{"cddr"}
| filter k8s.namespace.name == "cddr-ns"
| filter matchesPhrase(content, "callChart")
| fields timestamp, content
| limit 5
```

The callChart lines list each downstream call with its duration, like
`Relationship call (183ms)`. That is where the per-call timing comes from.

---

## 5. Calls per request - the fan-out check

How many times does one inbound request call `mfd-vndr-accts-rest`? The URL is
per-account (`/accounts/{id}/mutual-fund-vendor-accts`), so a planning group with
several accounts makes several calls.

```
fetch logs, from: "2026-09-03T15:00:00Z", to: "2026-09-03T17:00:00Z", scanLimitGBytes: -1, samplingRatio: 1, bucket:{"cddr"}
| filter k8s.namespace.name == "cddr-ns"
| filter contains(content, "Got retryable IO")
| filter contains(content, "mfd-vndr-accts-rest")
| parse content, "LD 'requestId=' LD:req_id ',' LD"
| filter isNotNull(req_id)
| summarize calls = count(), by: {req_id}
| summarize requests      = count(),
            total_calls   = sum(calls),
            max_per_req   = max(calls),
            p50_per_req   = percentile(calls, 50),
            p95_per_req   = percentile(calls, 95)
```

Compare against `rms-rltshp-svc`, measured at 1.00 calls per request. If this comes
back at five or ten, that is the difference between Sep 3 filling the bulkhead and
Sep 7 not.

Note these counts include retry attempts - the log line says "going to retry for the
1 time", so one logical call can produce more than one line.

### 5b. Same, for rms-rltshp on Sep 7, as the comparison

```
fetch logs, from: "2026-09-07T17:00:00Z", to: "2026-09-07T19:00:00Z", scanLimitGBytes: -1, samplingRatio: 1, bucket:{"cddr"}
| filter k8s.namespace.name == "cddr-ns"
| filter contains(content, "Got retryable IO")
| filter contains(content, "rms-rltshp-svc")
| parse content, "LD 'requestId=' LD:req_id ',' LD"
| filter isNotNull(req_id)
| summarize calls = count(), by: {req_id}
| summarize requests      = count(),
            total_calls   = sum(calls),
            max_per_req   = max(calls),
            p50_per_req   = percentile(calls, 50),
            p95_per_req   = percentile(calls, 95)
```

---

## 6. RUN THIS NEXT - call durations during the peak

```
fetch logs, from: "2026-09-03T17:30:00Z", to: "2026-09-03T18:30:00Z", scanLimitGBytes: -1, samplingRatio: 1, bucket:{"cddr"}
| filter k8s.namespace.name == "cddr-ns"
| filter matchesPhrase(content, "callChart")
| fields timestamp, content
| limit 5
```

Open one row and read the full text.

The callChart line lists every downstream call with its duration, like
`Relationship call (183ms)`. Looking for whether calls that SUCCEEDED during the
incident were taking seconds rather than milliseconds.

Why this matters more than anything measured so far - see "Timeouts cannot fill the
bulkhead" below.

---

## 7. Is the bulkhead metric in Dynatrace

Verifies the claim that `resilience4j_bulkhead_*` is not ingested for CDDR. Last
checked 2026-08-27 - re-run before relying on it.

### 7a. Any bulkhead metric at all, from any app

```
fetch metric.series
| filter matchesPhrase(metric.key, "bulkhead")
| summarize series = count(), by: {metric.key}
| sort metric.key asc
```

Expect rows like `pps-ps-api.resilience4j.bulkhead.available.concurrent.calls`. Note
the app-name prefix and the dots - other teams get theirs in with a different naming
convention. Anything starting `cddr` would sort between `cache_gets_total` and
`fsd-annuity`.

### 7b. Every metric Dynatrace holds for CDDR

This is the definitive check.

```
fetch metric.series
| filter k8s.namespace.name == "cddr-ns"
| summarize series = count(), by: {metric.key}
| sort metric.key asc
```

If the list is only Kubernetes and OneAgent metrics with nothing from the actuator
endpoint, the gap is confirmed. If actuator metrics appear under names we have not
searched for, the gap is not real and the earlier conclusion was wrong.

### 7c. Broader search, both naming conventions

```
fetch metric.series
| filter matchesPhrase(metric.key, "resilience4j")
| summarize series = count(), by: {metric.key}
| sort metric.key asc
```

Searching for `resilience4j_bulkhead` with underscores alone will miss the dotted
form. CDDR's cache metrics arrive dotted (`cache.gets`, not `cache_gets_total`), so
the same may apply here.

---

## 8. The metric DOES exist - confirm and use it

7c found `resilience4j.bulkhead.available.concurrent.calls` with 16 series, unprefixed
and dotted. Earlier searches used `resilience4j_bulkhead` with underscores and missed
it. Same trap as the cache metrics, which arrive as `cache.gets` rather than
`cache_gets_total`.

### 8a. Does it belong to CDDR

```
fetch metric.series
| filter metric.key == "resilience4j.bulkhead.available.concurrent.calls"
| summarize series = count(), by: {k8s.namespace.name, k8s.deployment.name, k8s.cluster.name}
| sort series desc
```

If `cddr-ns` appears, saturation is directly measurable and the log-based guesswork
was unnecessary.

### 8b. Saturation on Sep 3 - the answer we have been chasing

Set the notebook timeframe to 2026-09-03, or leave the filter and adjust the
timeframe picker.

```
timeseries available = min(`resilience4j.bulkhead.available.concurrent.calls`, default: 50),
  filter: { k8s.namespace.name == "cddr-ns" },
  by: { k8s.pod.name },
  interval: 1m
```

`min` per minute is the low-water mark. Read it as:

```
available = 0     bulkhead completely full, requests being rejected
available = 5     nearly full
available = 48    normal, two calls in flight
```

The limit is 50, confirmed from `resilience4j.bulkhead.max.allowed.concurrent.calls`.

This tells you how close CDDR runs to the limit under normal load - which is the
question Adam raised on the call and nobody could answer.

### 8b-2. Lowest free slots per pod - one number, no chart needed

8b returns a series per pod, which is awkward to read as a table. This collapses each
series to its minimum, so you get one row per pod showing how empty the bulkhead got.

Set the timeframe picker to **Sep 3, 09:00 - 13:00**.

The picker uses LOCAL time, not UTC. The Sep 3 exceptions ran 09:17-12:17 local, so
those are the numbers to type in. Entering the UTC equivalent puts you hours past the
incident.

A narrow window also matters because over 7 days Dynatrace overrides `interval: 1m`
and uses 10 minutes instead, which averages away any saturation shorter than that.

```
timeseries available = min(`resilience4j.bulkhead.available.concurrent.calls`, default: 50),
  filter: { k8s.namespace.name == "cddr-ns" },
  by: { k8s.pod.name },
  interval: 1m
| fieldsAdd lowest_free = arrayMin(available)
| fields k8s.pod.name, lowest_free
| sort lowest_free asc
```

```
lowest_free = 0      bulkhead completely full at some point
lowest_free = 5      nearly full
lowest_free = 47-50  normal, never under pressure
```

The limit is 50. From the 7 day view, normal sits at 47-50, so anything in single
figures is real pressure.

### 8c. How close to the limit on a normal day

```
timeseries available = min(`resilience4j.bulkhead.available.concurrent.calls`, default: 50),
  filter: { k8s.namespace.name == "cddr-ns" },
  interval: 1h
```

Run over 7 days. If the daily low-water mark sits at 45+ on quiet days and drops to 0
only during incidents, 50 is a reasonable limit. If it regularly dips into single
figures, the limit is too tight and raising it is a legitimate option rather than
masking.

Caveat: this is a gauge, sampled per minute. Short bursts lasting under a minute may
not appear. Good for sustained saturation like Sep 3; will miss the 100ms cold-start
spikes.

---

## 9. Tracing upstream - where does the stall start

### 9a. One host or all of them, minute by minute

The decisive test. If a single host spikes at 09:17 it is that service. If every host
spikes together, the cause is the shared path - gateway, network or DNS - and no
individual service owner will find anything.

```
fetch logs, from: "2026-09-03T16:10:00Z", to: "2026-09-03T16:30:00Z", scanLimitGBytes: -1, samplingRatio: 1, bucket:{"cddr"}
| filter k8s.namespace.name == "cddr-ns"
| filter contains(content, "Got retryable IO")
| parse content, "LD 'https://' LD:target_host '/' LD"
| summarize timeouts = count(), by: {minute = bin(timestamp, 1m), target_host}
| sort minute asc, timeouts desc
```

```
one host spikes at 09:17     that service stalled - go to its owner
all hosts spike together     shared infrastructure in front of apps2
nothing spikes at 09:17      the stall did not produce timeouts, so calls
                             were slow but still completing - see 9c
```

### 9b. Are the upstream services monitored in Dynatrace

If they run in an observed cluster you can read their side directly instead of
inferring from CDDR's client view.

```
fetch logs, from: now()-1d, to: now(), scanLimitGBytes: -1, samplingRatio: 1
| filter matchesPhrase(k8s.namespace.name, "mfd")
      or matchesPhrase(k8s.namespace.name, "gna")
      or matchesPhrase(k8s.namespace.name, "rms")
      or matchesPhrase(k8s.namespace.name, "ins-account")
| summarize lines = count(), by: {k8s.namespace.name, k8s.cluster.name}
| sort lines desc
```

Also worth checking the Services list in the Dynatrace UI for `mfd-vndr`,
`gna-accounts`, `rms-rltshp`. A monitored service gives you their error rate and
response time; an unmonitored one appears only as an outbound HTTP destination.

### 9c. Were calls slow without timing out

The 09:17 dip came with a 29% drop in log volume, so requests stalled rather than
failed. Slow-but-successful calls hold bulkhead slots and log nothing, which is why
timeout counts have never lined up.

```
fetch logs, from: "2026-09-03T16:15:00Z", to: "2026-09-03T16:20:00Z", scanLimitGBytes: -1, samplingRatio: 1, bucket:{"cddr"}
| filter k8s.namespace.name == "cddr-ns"
| filter matchesPhrase(content, "callChart")
| fields timestamp, content
| limit 10
```

Read the durations in the callChart lines. Normal is 29-183ms. If calls during that
minute show seconds, that is the stall, and the host named alongside is where it
started.

---

## 10. It is not a downstream service - narrowing the shared cause

9a returned 16 hosts timing out in the same minute, across two domains
(`apps2` and `apps3`). Sixteen independent services do not stall together. The cause
is shared, and on CDDR's side of the connection.

Candidates: DNS resolution, the Istio sidecar, node-level network or CPU pressure, or
a stop-the-world GC pause.

### 10a. Which pods, and which nodes

If every affected pod sits on one node, it is node-level. If they are spread across
nodes, it is not.

```
fetch logs, from: "2026-09-03T16:16:00Z", to: "2026-09-03T16:19:00Z", scanLimitGBytes: -1, samplingRatio: 1, bucket:{"cddr"}
| filter k8s.namespace.name == "cddr-ns"
| filter contains(content, "Got retryable IO")
| summarize timeouts = count(), by: {k8s.pod.name, k8s.node.name}
| sort timeouts desc
```

### 10b. Was it CPU throttling or GC

First find what metrics exist:

```
fetch metric.series
| filter k8s.namespace.name == "cddr-ns"
| filter matchesPhrase(metric.key, "cpu")
      or matchesPhrase(metric.key, "throttl")
      or matchesPhrase(metric.key, "gc")
      or matchesPhrase(metric.key, "memory")
| summarize series = count(), by: {metric.key}
| sort metric.key asc
```

Then chart whichever throttling or GC pause metric appears, over
Sep 3 09:15-09:20 local, `by: {k8s.pod.name}`, `interval: 1m`.

A GC pause or CPU throttle on the pods would stall every outbound call at once and
would also explain the 29% drop in log volume during that minute.

### 10c. Was it DNS

Look for resolution failures or slow lookups in the same window.

```
fetch logs, from: "2026-09-03T16:15:00Z", to: "2026-09-03T16:20:00Z", scanLimitGBytes: -1, samplingRatio: 1
| filter matchesPhrase(content, "UnknownHostException")
      or matchesPhrase(content, "Temporary failure in name resolution")
      or matchesPhrase(content, "dns")
| summarize lines = count(), by: {minute = bin(timestamp, 1m), k8s.namespace.name}
| sort lines desc
```

Note this drops the bucket filter deliberately - if CoreDNS stalled, other namespaces
would see it too, and that would be the strongest possible evidence.

### 10d. Did other namespaces stall at the same minute

The decisive test for a cluster-wide cause.

```
fetch logs, from: "2026-09-03T16:15:00Z", to: "2026-09-03T16:20:00Z", scanLimitGBytes: -1, samplingRatio: 1
| filter matchesPhrase(content, "SocketTimeoutException")
      or matchesPhrase(content, "Read timed out")
| summarize timeouts = count(), by: {minute = bin(timestamp, 1m), k8s.namespace.name}
| sort timeouts desc
```

If several unrelated namespaces spike at 09:17, the problem is the cluster or its
network, not CDDR. That moves this to the platform team.

---

## 11. Is the cluster-wide stall chronic

10d showed 10+ unrelated namespaces timing out in the same minute on Sep 3. This
checks whether the same thing happened on the other incident mornings.

Run once per date, changing only the two timestamps. Incident days were Sep 8, Sep 3,
Sep 2, Aug 27, Aug 26, all between 07:00 and 12:30 local.

```
fetch logs, from: "2026-09-02T13:00:00Z", to: "2026-09-02T20:00:00Z", scanLimitGBytes: -1, samplingRatio: 1
| filter matchesPhrase(content, "SocketTimeoutException")
      or matchesPhrase(content, "Read timed out")
| summarize timeouts    = count(),
            namespaces  = countDistinct(k8s.namespace.name),
            by: {minute = bin(timestamp, 1m)}
| filter namespaces >= 5
| sort timeouts desc
| limit 20
```

Any minute where five or more namespaces time out together is a cluster-level stall,
not an application fault. If those minutes line up with the bulkhead incidents on
every date, this is chronic and recurring rather than a one-off.

### 11b. All incident days at once, if the scan cost allows

```
fetch logs, from: now()-14d, to: now(), scanLimitGBytes: -1, samplingRatio: 1
| filter matchesPhrase(content, "SocketTimeoutException")
      or matchesPhrase(content, "Read timed out")
| summarize timeouts   = count(),
            namespaces = countDistinct(k8s.namespace.name),
            by: {minute = bin(timestamp, 1m)}
| filter namespaces >= 5
| summarize stall_minutes = count(),
            worst         = max(timeouts),
            by: {day = bin(minute, 1d)}
| sort day asc
```

Gives a count of cluster-wide stall minutes per day. Expensive - run 11 on single
days first and only widen if the scan is affordable.

---

## 12. The control - is a correlated minute actually unusual

Section 11 shows the worst 20 minutes per day. It does not show whether those minutes
differ from any other minute. Without that, "29 namespaces timed out together" may
just be what a large cluster looks like every minute, and the correlation means
nothing.

Run this before escalating anything.

### 12a. Distribution across a full day

```
fetch logs, from: "2026-09-03T12:00:00Z", to: "2026-09-03T20:00:00Z", scanLimitGBytes: -1, samplingRatio: 1
| filter k8s.cluster.name == "aks04074prscu01"
| filter matchesPhrase(content, "SocketTimeoutException")
      or matchesPhrase(content, "Read timed out")
| summarize timeouts   = count(),
            namespaces = countDistinct(k8s.namespace.name),
            by: {minute = bin(timestamp, 1m)}
| summarize minutes    = count(),
            p50_ns     = percentile(namespaces, 50),
            p95_ns     = percentile(namespaces, 95),
            max_ns     = max(namespaces),
            p50_to     = percentile(timeouts, 50),
            p95_to     = percentile(timeouts, 95),
            max_to     = max(timeouts)
```

Note the cluster filter. Section 11 had none, so its counts spanned every cluster.

```
p50 namespaces around 5, spikes to 29    the correlation is real
p50 namespaces already 25-30             normal background, no finding
```

Same for timeouts: if p50 is 2,000 and the "spike" was 2,779, there is no spike.

### 12b. The incident minutes against that baseline

```
fetch logs, from: "2026-09-03T12:00:00Z", to: "2026-09-03T20:00:00Z", scanLimitGBytes: -1, samplingRatio: 1
| filter k8s.cluster.name == "aks04074prscu01"
| filter matchesPhrase(content, "SocketTimeoutException")
      or matchesPhrase(content, "Read timed out")
| summarize timeouts   = count(),
            namespaces = countDistinct(k8s.namespace.name),
            by: {minute = bin(timestamp, 1m)}
| sort minute asc
```

Full timeline, no limit, no sort by size. Read whether 09:17 stands out from the
minutes either side of it, or whether it sits in a continuous band.

That single chart decides whether there is anything to escalate.

---

## 13. Which clusters were affected at 09:17

12a returned `max_ns = 2` for `aks04074prscu01`, yet 10d showed ten namespaces
timing out at 09:17. 10d had no cluster filter, so those namespaces sit on other
clusters. This confirms whether the stall spanned clusters.

```
fetch logs, from: "2026-09-03T16:16:00Z", to: "2026-09-03T16:19:00Z", scanLimitGBytes: -1, samplingRatio: 1
| filter matchesPhrase(content, "SocketTimeoutException")
      or matchesPhrase(content, "Read timed out")
| summarize timeouts = count(), by: {k8s.cluster.name, k8s.namespace.name}
| sort timeouts desc
```

```
several clusters listed   the stall is above the cluster layer - shared
                          network path, DNS, or the route to apps2/apps3
only prscu01              it is that cluster, and 10d was picking up
                          unrelated background from elsewhere
```

### 13b. Same minute, one row per cluster

```
fetch logs, from: "2026-09-03T16:10:00Z", to: "2026-09-03T16:30:00Z", scanLimitGBytes: -1, samplingRatio: 1
| filter matchesPhrase(content, "SocketTimeoutException")
      or matchesPhrase(content, "Read timed out")
| summarize timeouts   = count(),
            namespaces = countDistinct(k8s.namespace.name),
            by: {minute = bin(timestamp, 1m), k8s.cluster.name}
| sort minute asc, timeouts desc
```

Shows whether every cluster spikes on the same minute or whether they are offset.
Simultaneous across clusters is close to proof of a shared network or DNS cause,
since nothing inside a cluster can affect another one.

---

## 14. Topology - what sits in front of apps2 and apps3

Ten-plus clusters across two platforms and two sites stalled in the same minute. The
shared element is the destination, not the source. This looks for what that is.

### 14a. Does Dynatrace know these hostnames as entities

```
fetch dt.entity.service
| filter matchesPhrase(entity.name, "apps2") or matchesPhrase(entity.name, "apps3")
| fields id, entity.name, serviceType, agentTechnologyType
| limit 50
```

If they appear as monitored services, open one and read its own error rate and
response time for 09:17. If they appear only as external destinations, Dynatrace sees
the call but not the far end.

### 14b. What CDDR actually calls

```
fetch dt.entity.service
| filter matchesPhrase(entity.name, "cddr")
| fields id, entity.name, serviceType
```

Take the `cddr-main-subgraph` service id, open it in the UI, and use the service flow
or Smartscape view to see its outbound dependencies. That shows how Dynatrace models
the hop to apps2 - as a service, a process group, or an unmonitored host.

### 14c. Hosts and process groups behind those names

```
fetch dt.entity.host
| filter matchesPhrase(entity.name, "apps2") or matchesPhrase(entity.name, "apps3")
| fields id, entity.name
| limit 50
```

```
fetch dt.entity.process_group
| filter matchesPhrase(entity.name, "apps2") or matchesPhrase(entity.name, "apps3")
| fields id, entity.name
| limit 50
```

### In the UI

Smartscape is the faster route if the queries come back thin:

```
Services -> search "apps2"        are they monitored at all
Services -> cddr-main-subgraph    open it, then Service flow, and follow
                                  the outbound calls
Smartscape -> Services            shows the dependency graph directly
```

What you are looking for is a single component every one of those clusters routes
through - a load balancer, an API gateway, a firewall, or a DNS resolver. If
Dynatrace does not monitor it, the topology stops at the hostname and the answer has
to come from the network team.

### The question to ask them

`apps2.edwardjones.com` and `apps3.edwardjones.com` both stalled at 09:17 on Sep 3,
seen simultaneously from AKS and DKP clusters in both STL and PHX. What do those two
names share - the same load balancer, the same firewall, the same DNS zone, the same
egress path? That shared component is where the stall started.

---

## 15. Are the apps2/apps3 backends monitored anywhere

14a searched service entity names for "apps2"/"apps3" and found only two unrelated
Salesforce proxies. But the DNS name is not necessarily the workload name - those
services may run as Kubernetes workloads in monitored clusters under their own names.

### 15a. Do the service names appear as workloads

```
fetch logs, from: now()-2d, to: now(), scanLimitGBytes: -1, samplingRatio: 1
| filter matchesPhrase(k8s.namespace.name, "rms")
      or matchesPhrase(k8s.namespace.name, "gna")
      or matchesPhrase(k8s.namespace.name, "mfd")
      or matchesPhrase(k8s.deployment.name, "rms-rltshp")
      or matchesPhrase(k8s.deployment.name, "gna-accounts")
      or matchesPhrase(k8s.deployment.name, "mfd-vndr")
| summarize lines = count(), by: {k8s.cluster.name, k8s.namespace.name, k8s.deployment.name}
| sort lines desc
| limit 30
```

If any of them appear, open that workload and look at 09:17 on Sep 3 directly.

### 15b. Service entities by the workload name rather than the DNS name

```
fetch dt.entity.service
| filter matchesPhrase(entity.name, "rms-rltshp")
      or matchesPhrase(entity.name, "gna-accounts")
      or matchesPhrase(entity.name, "mfd-vndr")
      or matchesPhrase(entity.name, "ins-account")
| fields id, entity.name, serviceType
| limit 30
```

### 15c. Hosts and process groups

```
fetch dt.entity.host
| filter matchesPhrase(entity.name, "rms") or matchesPhrase(entity.name, "gna")
| fields id, entity.name
| limit 30
```

```
fetch dt.entity.process_group
| filter matchesPhrase(entity.name, "rms-rltshp") or matchesPhrase(entity.name, "gna-accounts")
| fields id, entity.name
| limit 30
```

### 15d. Anything logging from the server side at 09:17

If these services are monitored at all, they would have logged something in that
minute. This searches every bucket and every namespace.

```
fetch logs, from: "2026-09-03T16:16:00Z", to: "2026-09-03T16:19:00Z", scanLimitGBytes: -1, samplingRatio: 1
| filter matchesPhrase(content, "rms-rltshp")
      or matchesPhrase(content, "gna-accounts")
      or matchesPhrase(content, "mfd-vndr")
| summarize lines = count(), by: {k8s.cluster.name, k8s.namespace.name}
| sort lines desc
| limit 30
```

Rows that are NOT `cddr-ns` are the interesting ones - that would be someone else's
view of the same services, or the services themselves.

### If all four come back empty

The backends are not in Dynatrace. Everything visible is the client side, and the
answer has to come from whoever operates apps2/apps3 or the network path to them.

---

## 16. The backends are on DKP - read their side

15a found them. `apps2.edwardjones.com` fronts workloads running on the DKP clusters:

```
dkp-prod-phx-general   rms-relationship-service   rms-rltshp-svc-*
dkp-prod-phx-general   mfd-networking             mfd-vndr-accts-rest-*
dkp-prod-phx-general   gna-accounts               gna-accounts-svc-*
dkp-prod-phx-general   gna-accounts               gna-restrictions-*
dkp-prod-stl-general   gna-accounts               gna-accounts-svc-*
```

And 13a showed `dkp-prod-phx-general` timing out at 09:17 as well, then spiking to
7,235 at 09:22. The clusters hosting the backends were stalling too.

Keep windows tight - 15a scanned 34 TiB.

### 16a. What the backends logged at 09:17

```
fetch logs, from: "2026-09-03T16:16:00Z", to: "2026-09-03T16:19:00Z", scanLimitGBytes: -1, samplingRatio: 1
| filter in(k8s.cluster.name, {"dkp-prod-phx-general", "dkp-prod-stl-general"})
| filter in(k8s.namespace.name, {"rms-relationship-service", "mfd-networking", "gna-accounts"})
| filter loglevel == "ERROR" or loglevel == "WARN"
| summarize lines = count(), by: {k8s.namespace.name, k8s.deployment.name, loglevel}
| sort lines desc
| limit 30
```

### 16b. What were THEY timing out on

If the backends were themselves blocked on something - a database, a mainframe, an
auth service - that is the origin rather than the network.

```
fetch logs, from: "2026-09-03T16:16:00Z", to: "2026-09-03T16:19:00Z", scanLimitGBytes: -1, samplingRatio: 1
| filter in(k8s.cluster.name, {"dkp-prod-phx-general", "dkp-prod-stl-general"})
| filter matchesPhrase(content, "SocketTimeoutException")
      or matchesPhrase(content, "Read timed out")
      or matchesPhrase(content, "timeout")
| summarize lines = count(), by: {k8s.namespace.name, k8s.deployment.name}
| sort lines desc
| limit 30
```

### 16c. Baseline for the DKP side

Same as 12a but for DKP - is 09:17 unusual there, or normal background?

```
fetch logs, from: "2026-09-03T12:00:00Z", to: "2026-09-03T20:00:00Z", scanLimitGBytes: -1, samplingRatio: 1
| filter k8s.cluster.name == "dkp-prod-phx-general"
| filter matchesPhrase(content, "SocketTimeoutException")
      or matchesPhrase(content, "Read timed out")
| summarize timeouts = count(), by: {minute = bin(timestamp, 1m)}
| sort minute asc
```

### 16d. Sample the actual error text

```
fetch logs, from: "2026-09-03T16:16:00Z", to: "2026-09-03T16:19:00Z", scanLimitGBytes: -1, samplingRatio: 1
| filter k8s.cluster.name == "dkp-prod-phx-general"
| filter k8s.namespace.name == "rms-relationship-service"
| filter loglevel == "ERROR"
| fields timestamp, k8s.deployment.name, content
| limit 5
```

Read one in full. Whatever `rms-rltshp-svc` was failing on at 09:17 is one hop closer
to the origin.

---

## 17. Read the backend errors

16a showed `gna-accounts-svc` logging 352 ERRORs at 09:17 - the only backend genuinely
failing rather than merely slow. `rms-rltshp-svc` logged 94 lines total while CDDR was
timing out on it 441 times, so it was not erroring, it just was not answering.

### 17a. What gna-accounts-svc was failing on

```
fetch logs, from: "2026-09-03T16:16:00Z", to: "2026-09-03T16:19:00Z", scanLimitGBytes: -1, samplingRatio: 1
| filter k8s.cluster.name == "dkp-prod-phx-general"
| filter k8s.namespace.name == "gna-accounts"
| filter contains(k8s.deployment.name, "gna-accounts-svc")
| filter loglevel == "ERROR"
| fields timestamp, content
| limit 5
```

### 17b. And the DKP management plane

`kommander/prometheus-adapter` logged 718 timeouts in the same window. Kommander is
DKP's own control plane - if that was stalling, the cause is below the applications.

```
fetch logs, from: "2026-09-03T16:16:00Z", to: "2026-09-03T16:19:00Z", scanLimitGBytes: -1, samplingRatio: 1
| filter k8s.cluster.name == "dkp-prod-phx-general"
| filter k8s.namespace.name == "kommander"
| filter matchesPhrase(content, "timeout") or matchesPhrase(content, "Read timed out")
| fields timestamp, k8s.deployment.name, content
| limit 5
```

### What the DKP side already tells us

```
rms-rltshp-svc     94 log lines    CDDR saw 441 timeouts to it
mfd-acct-profile  105 log lines    CDDR saw 87
gna-accounts-svc  352 ERROR        CDDR saw 124
```

The backends were not throwing errors in proportion to the timeouts CDDR saw. Calls
were not arriving, or not being answered in time - which is the path between the
clusters, not the applications at either end.

Meanwhile DKP's own control plane was timing out too, which no application can cause.

---

## 18. What is still investigable from here

CDDR-side analysis is finished. These three are the remaining questions that Dynatrace
can answer without network or platform access.

### 18a. Which cluster stalled first

If timeouts start in one place and spread, that is directional evidence for where the
origin is. If everything starts in the same second, it is simultaneous and points at
something all of them touch at once - DNS, or a shared network path.

```
fetch logs, from: "2026-09-03T16:16:00Z", to: "2026-09-03T16:19:00Z", scanLimitGBytes: -1, samplingRatio: 1
| filter matchesPhrase(content, "SocketTimeoutException")
      or matchesPhrase(content, "Read timed out")
| summarize first_timeout = min(timestamp),
            last_timeout  = max(timestamp),
            timeouts      = count(),
            by: {k8s.cluster.name}
| sort first_timeout asc
```

Compare the earliest against DKP's apiserver timeout at **16:17:11.679**. If
application timeouts start before that, the control plane was a victim too. If they
start after, the control plane may be closer to the cause.

This is the one that tells you whether DKP is the origin or another casualty.

### 18a-2. Widen the window - 18a was truncated

18a ran from 16:16:00 and the earliest cluster appeared at 16:16:04, four seconds in.
That is the window boundary, not necessarily the true start. Widen it.

```
fetch logs, from: "2026-09-03T16:05:00Z", to: "2026-09-03T16:22:00Z", scanLimitGBytes: -1, samplingRatio: 1
| filter matchesPhrase(content, "SocketTimeoutException")
      or matchesPhrase(content, "Read timed out")
| summarize first_timeout = min(timestamp),
            last_timeout  = max(timestamp),
            timeouts      = count(),
            by: {k8s.cluster.name}
| sort first_timeout asc
```

Sort by `first_timeout`, not by count - the UI defaults to sorting by the numeric
column and that hides the ordering.

From the truncated run, the sequence was:

```
16:16:04   dkp-prod-phx-external
16:16:12   dkp-prod-phx-general
16:16:37   dkp-prod-stl-general
16:17:11   DKP apiserver "http: Handler timeout"
16:17:21   dkp-prod-stl-external
16:17:34   aks04074prscu01 (CDDR)
16:17:40   aks03231prscu01
```

DKP clusters lead the AKS clusters by roughly 70 seconds, with DKP's own control
plane timing out between the two groups. If that holds with a wider window, the
stall started on DKP and propagated outward to everything calling into it.

If the wider window shows DKP starting even earlier, keep widening until the first
timeout is comfortably inside the window rather than at its edge.

### 18b. Does it happen at a consistent time

Every event so far falls between 07:00 and 12:30 local. If they cluster at a
particular minute past the hour, something scheduled is involved.

```
fetch logs, from: now()-30d, to: now(), scanLimitGBytes: -1, samplingRatio: 1
| filter k8s.namespace.name == "cddr-ns"
| filter matchesPhrase(content, "BulkheadFullException")
| summarize exceptions = count(), by: {minute = bin(timestamp, 1m)}
| fieldsAdd minute_of_hour = formatTimestamp(minute, format: "mm")
| summarize events = count(), total = sum(exceptions), by: {minute_of_hour}
| sort events desc
| limit 20
```

A flat spread means it is random. A concentration at particular minutes means a job,
a backup, a certificate refresh or a scan.

### 18c. Is it getting worse

```
fetch logs, from: now()-30d, to: now(), scanLimitGBytes: -1, samplingRatio: 1
| filter matchesPhrase(content, "SocketTimeoutException")
      or matchesPhrase(content, "Read timed out")
| summarize timeouts   = count(),
            namespaces = countDistinct(k8s.namespace.name),
            clusters   = countDistinct(k8s.cluster.name),
            by: {minute = bin(timestamp, 1m)}
| filter clusters >= 5
| summarize stall_minutes = count(),
            worst_minute  = max(timeouts),
            by: {day = bin(minute, 1d)}
| sort day asc
```

Counts the minutes per day where five or more clusters timed out together. Gives a
frequency trend and tells the platform team whether this is stable or degrading.

Expensive - run 18a and 18b first.

### What cannot be answered from here

Why multiple clusters across two platforms and two sites stall together needs network
telemetry, Azure or DKP platform logs, or DNS query logs. None of that is visible from
the application side.

---

## Results so far

### 2026-09-03 - pod age at exception

```
ESTABLISHED (over 1 hour)   1,120   6 pods
COLD START (under 60s)         41   5 pods
no_startup_line_found           4   1 pod
```

96% established. Not a scale-up problem. Six pods that had been running over an hour
stopped coping at the same time.

### 2026-09-03 - downstream timeouts by hour (local)

```
hour     mfd-vndr-accts-rest   gna-accounts   rms-rltshp   cos-taxlotdata
06:00                    611
07:00                    237
08:00                    630
09:00                    251                         442
10:00                    404
11:00                  1,188            496
12:00                    232            669
13:00                    560
14:00                                                                510
```

`mfd-vndr-accts-rest` is the dominant host, over 4,100 timeouts sustained across
eight hours. The bulkhead exceptions ran 09:17-12:17, through its peak.

### 2026-09-07 - the counter-example

```
rms-rltshp-svc           970
gna-restrictions         860
ins-account-svc          457
fpl-fin-goals-rest       330
con-network-rest         109
cln-loans-details-svc     70
```

Six services timing out in the same hour, and **zero bulkhead exceptions**. Similar
volume to Sep 3, opposite outcome.

### Raw timeout line, 2026-09-03

```
[WARN]~2026-09-03-16.30.07.543GMT [callerUserId=..., requestId=0a6700a8-5fa4-4fe4-be38-cbb5d98d0de4,
callerSysId=CDDR, correlationId=..., callerAppName=cddr-flex-gw, requestStartTime=1788453003382]
com.edwardjones.odi.ods.dgs.config.resttemplate.AppHttpRequestRetryStrategy odi-ods-dgs-svc-worker-0
Got retryable IOException of class java.net.SocketTimeoutException from HttpRequest GET
https://mfd-vndr-accts-rest.apps2.edwardjones.com/accounts/150171173/mutual-fund-vendor-accts,
going to retry for the 1 time, IOException message Read timed out
```

Three things in it:

- `SocketTimeoutException` / "Read timed out" - the connection was established and CDDR
  waited. A thread is held for the full timeout. Contrast the BPP 502s, which died at
  TLS handshake in 54ms and held nothing.
- "going to retry for the 1 time" - CDDR retries, so one logical call holds a thread
  for roughly double the timeout. Not accounted for in any earlier concurrency maths.
- The URL is per-account. A planning group with five accounts makes five calls.
  `rms-rltshp-svc` was measured at 1.00 calls per request; this one fans out.

`requestStartTime=1788453003382` against the 16:30:07.543 log timestamp gives 4,161ms.
That is time since the inbound request started, not this call alone, so treat it as an
upper bound near the 4s timeout rather than proof of it.

Together these explain why 1,188 timeouts filled the bulkhead on Sep 3 while 970 did
not on Sep 7: same timeout, multiplied by retries and by accounts per request.

### Calls per request - fan-out ruled out

```
mfd-vndr-accts-rest   828 requests     881 calls   p50 1.0   p95 1.0   max 4
rms-rltshp-svc      1,091 requests   1,091 calls   p50 1.0   p95 1.0   max 1
```

Both are one call per request. The fan-out theory was wrong.

The Sep 3 window used for this ran 08:00-10:00 local, which missed the 11:00 peak, so
828 is not the busy period. Does not change the conclusion - p50 and p95 are both 1.0.

### Timeouts cannot fill the bulkhead

Arithmetic on the Sep 3 peak hour:

```
1,188 timeouts / 3600 seconds        = 0.33 per second
0.33 per second x 4 seconds held     = 1.3 concurrent slots
```

The bulkhead is 50, spread across 10 pods. 1.3 slots is a rounding error.

Aug 10 worked out differently only because 8,127 timeouts landed in 18 minutes rather
than across an hour - 7.5 per second, which does reach roughly 30 slots. Sep 3 has
nothing like that shape.

So what fills the bulkhead is almost certainly the calls that DO NOT time out. If a
downstream is answering in 2-3 seconds instead of 200ms, thousands of successful calls
are each holding a thread for seconds, and none of them log anything. The timeouts are
the visible tail of that, not the cause.

This would explain why timeout counts have never predicted saturation across any of
these incidents. Section 6 tests it.

### What this changes

Earlier analysis named `rms-rltshp-svc` as the recurring cause based on the August
data. Sep 3 disproves that - a different service entirely, and Sep 7 shows rms-rltshp
at high volume causing nothing.

Timeout volume alone does not predict saturation.

What holds: CDDR has one shared bulkhead of 50 for all queries. Any slow service
behind `apps2.edwardjones.com` can fill it and take down every unrelated query with
it. The recurring pattern is not a particular dependency - it is that the bulkhead
does not isolate.

---

## Reporting it

If query 1 comes back clean, the accurate line is:

```
No bulkhead exceptions since 27 Aug.
```

Not "the problem is fixed". The exceptions were driven by rms-rltshp-svc timing out.
If that service has simply had a good fortnight, the quiet belongs to them, and CDDR
will fail the same way next time it goes bad.
