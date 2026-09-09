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
