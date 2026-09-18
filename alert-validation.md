# Validating the ODS GraphQL Errors detector (PR #1041)

Three tests. Test 1 says whether it fires at all. Test 2 says whether the threshold of
10 is sensible. Test 3 says whether it catches the other known incidents.

Each test uses the detector's own query, with `arrayMax` on the end so you get one
number instead of a series.

If a test returns nothing, first check the deployment name and bucket. The detector
targets `odi-ods-dgs-svc-deploy` in `data_enablement_services`; CDDR prod logs have
also been found as `cddr-main-subgraph` in the `cddr` bucket. A wrong bucket looks
exactly like "no errors".

---

## Test 1 - does it fire on a known incident

2026-09-03, 16:16-16:19 UTC. CDDR logged roughly 1,165 bulkhead exceptions that
morning.

```
fetch logs, from: "2026-09-03T16:00:00Z", to: "2026-09-03T16:40:00Z", scanLimitGBytes: -1, samplingRatio: 1, bucket:{"data_enablement_services"}
| filter matchesPhrase(k8s.deployment.name, "odi-ods-dgs-svc-deploy")
| filter startsWith(content, "[WARN]")
| filter matchesPhrase(content, "com.edwardjones.odi.ods.dgs.exceptions")
| filter not matchesPhrase(content, "401 Unauthenticated issue encountered for")
| filter not matchesPhrase(content, "DOWNSTREAM Data Integrity 404: Non-nullable was not found for")
| filter not matchesPhrase(content, "403 Permission denied for")
| makeTimeseries error_count = count(default: 0), interval: 1m, by: {dt.source_entity = dt.entity.cloud_application}
| fieldsAdd total_error_count = arrayMovingSum(error_count, 5)
| fieldsAdd peak = arrayMax(total_error_count)
| fields dt.source_entity, peak
| sort peak desc
```

```
peak well above 10    the detector fires - point proven
peak below 10         it still would not fire, filters need another look
no rows               wrong bucket or wrong deployment name
```

Add the region filter if the detector has one (`| filter region == "SCU"`).

---

## Test 2 - is 10 above normal background

Run over a quiet week. Pick one with no known incident - avoid 26/27 Aug and 2, 3 and
8 Sep.

```
fetch logs, from: "2026-08-28T00:00:00Z", to: "2026-09-02T00:00:00Z", scanLimitGBytes: -1, samplingRatio: 1, bucket:{"data_enablement_services"}
| filter matchesPhrase(k8s.deployment.name, "odi-ods-dgs-svc-deploy")
| filter startsWith(content, "[WARN]")
| filter matchesPhrase(content, "com.edwardjones.odi.ods.dgs.exceptions")
| filter not matchesPhrase(content, "401 Unauthenticated issue encountered for")
| filter not matchesPhrase(content, "DOWNSTREAM Data Integrity 404: Non-nullable was not found for")
| filter not matchesPhrase(content, "403 Permission denied for")
| makeTimeseries error_count = count(default: 0), interval: 1m
| fieldsAdd total_error_count = arrayMovingSum(error_count, 5)
| fieldsAdd p50 = arrayPercentile(total_error_count, 50),
            p95 = arrayPercentile(total_error_count, 95),
            peak = arrayMax(total_error_count)
| fields p50, p95, peak
```

If `arrayPercentile` is not available, drop those two lines and keep `arrayMax`, then
judge from the chart.

```
p95 below 10        threshold of 10 is fine
p95 above 10        it will fire on ordinary days - set the threshold above p95
peak near 10        borderline, expect noise
```

The number to aim for sits above normal peaks and below what the incidents produce.
Test 1 gives the upper bound, this gives the lower one.

---

## Test 3 - does it catch the other incidents

Same as test 1 with different dates. These are the mornings with confirmed bursts.

```
Aug 26   from "2026-08-26T12:00:00Z" to "2026-08-26T18:00:00Z"
Aug 27   from "2026-08-27T12:00:00Z" to "2026-08-27T18:00:00Z"
Sep 02   from "2026-09-02T13:00:00Z" to "2026-09-02T20:00:00Z"
Sep 08   from "2026-09-08T13:00:00Z" to "2026-09-08T20:00:00Z"
```

An incident where `peak` stays under the threshold is a miss. Four for four is a
detector worth merging.

---

## If test 1 returns no rows

Try the same query against the other naming:

```
bucket:{"cddr"}
| filter k8s.namespace.name == "cddr-ns"
| filter contains(k8s.pod.name, "cddr-main-subgraph")
```

If that works and the detector's version does not, the detector is pointed at the
wrong place and would never have fired regardless of threshold.
