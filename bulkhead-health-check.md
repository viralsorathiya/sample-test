# CDDR bulkhead health check

Run query 1. If nothing recent comes back, CDDR is clean on the bulkhead side.

Last known state (checked 2026-09-02): last exception was 2026-08-27. Six clean days.
Before the 20 Aug deploy there were 45,083 exceptions, including 42,819 on 10 Aug
alone.

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

## 2. Only if query 1 shows something recent

Which pods, and how long they had been running when it happened.

```
fetch logs, from: now()-3d, to: now(), scanLimitGBytes: -1, samplingRatio: 1, bucket:{"cddr"}
| filter k8s.namespace.name == "cddr-ns"
| filter contains(k8s.pod.name, "cddr-main-subgraph")
| filter matchesPhrase(content, "BulkheadFullException")
| fields exception_ts = timestamp, pod = k8s.pod.name, cluster = k8s.cluster.name
| lookup [
    fetch logs, from: now()-3d, to: now(), scanLimitGBytes: -1, samplingRatio: 1, bucket:{"cddr"}
    | filter k8s.namespace.name == "cddr-ns"
    | filter contains(k8s.pod.name, "cddr-main-subgraph")
    | filter matchesPhrase(content, "Started CddrMainSubgraphApplication")
    | summarize startup_ts = max(timestamp), by: {pod = k8s.pod.name}
  ], sourceField: pod, lookupField: pod, prefix: "lu."
| fieldsAdd age_seconds = if(isNull(lu.startup_ts), null,
    else: toLong(exception_ts - lu.startup_ts) / 1000000000)
| summarize exceptions = count(),
            min_age_s  = min(age_seconds),
            max_age_s  = max(age_seconds),
            first_exc  = min(exception_ts),
            by: {cluster, pod}
| sort first_exc desc
```

```
age under 60s     cold start - happens on scale-up, small numbers
age hours/days    a downstream service was timing out - the serious kind
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
