# CDDR cache dashboard

Built from Maosheng's query. Same metrics, plus a hit-rate tile and a scope note.

## The thing to put on the dashboard first

Maosheng's query returns **3 caches**: `accountPriorDayTotalBalanceAmount`,
`accountRestrictions` and `holding`.

CDDR has **11** Redis caches. The other 8 publish nothing, because
`cache.gets.hit` / `cache.gets.miss` come from custom counters written by hand for
those three. The rest rely on Spring's built-in Redis statistics, which are disabled
(ODI-22529).

So this dashboard is accurate and incomplete at the same time. Without a note on it,
people will read three green lines as "caching is healthy" when two thirds of the
caches are invisible. Put the note in a markdown tile at the top, not in a ticket
nobody opens.

---

## Tile 1 - markdown, top of dashboard

```
Covers 3 of CDDR's 11 Redis caches: accountPriorDayTotalBalanceAmount,
accountRestrictions, holding.

The other 8 have no hit/miss metrics. Those three have custom counters in the
application; the rest depend on Spring Redis cache statistics, which are off.
Tracked under ODI-22529 - enabling them would put all 11 on this dashboard.

Cluster: aks04074prscu01 (SCU) only.
```

## Tile 2 - Hit rate by cache

The headline number. Raw hit counts track traffic volume; the rate tracks whether the
cache is doing its job.

```
timeseries {
  hits   = sum(`cache.gets.hit`),
  misses = sum(`cache.gets.miss`)
},
by: { cache },
filter: {
  entityName(dt.entity.kubernetes_cluster) == "aks04074prscu01" and
  entityName(dt.entity.cloud_application) == "cddr-main-subgraph"
},
union: true,
nonempty: true
| fieldsAdd hit_rate = 100 * hits[] / (hits[] + misses[])
| fields cache, hit_rate
```

Line chart, y-axis 0-100, unit percent.

`keyName` is dropped here on purpose - one line per cache reads better than one per
key. Keep it in tile 3 if the key-level split is wanted.

## Tile 3 - Hits and misses, by volume

Maosheng's original query, unchanged.

```
timeseries {
  hits   = sum(`cache.gets.hit`),
  misses = sum(`cache.gets.miss`)
},
by: { cache, keyName },
filter: {
  entityName(dt.entity.kubernetes_cluster) == "aks04074prscu01" and
  entityName(dt.entity.cloud_application) == "cddr-main-subgraph"
},
union: true,
nonempty: true
```

## Tile 4 - Total lookups per cache

Context for tile 2. A 40% hit rate on 10 lookups a minute does not matter; on 500k it
does.

```
timeseries {
  hits   = sum(`cache.gets.hit`),
  misses = sum(`cache.gets.miss`)
},
by: { cache },
filter: {
  entityName(dt.entity.kubernetes_cluster) == "aks04074prscu01" and
  entityName(dt.entity.cloud_application) == "cddr-main-subgraph"
},
union: true,
nonempty: true
| fieldsAdd lookups = hits[] + misses[]
| fields cache, lookups
```

---

## Two changes worth making

**Make the cluster a variable.** The query hardcodes `aks04074prscu01`, so NCU is
invisible. A dashboard variable lets one dashboard serve both, and works for non-prod
later.

```
variable   cluster
values     aks04074prscu01, aks04074prncu01
filter     entityName(dt.entity.kubernetes_cluster) == $cluster
```

**Set a rolling timeframe, not a pinned one.** Saving while a fixed window is selected
freezes the dashboard at that window.

---

## Reading the chart

The 7-day view shows a clear daily shape - high during business hours, near zero
overnight, and flat across Sep 12-13, which was the weekend. Expected. Worth saying so
on the dashboard, otherwise someone raises the weekend as an incident.

`sum` is right for these metrics: they are counters, so summing within each interval
gives lookups per interval.

---

# Part 2 - Account and contact usage (Maosheng, 22 Sep)

His ask: "do we have metrics for usage for account and contact queries? I am trying
to find out metrics before turning on cache for account and contact" ... "just group
downstream service call for" these two:

```
https://gna-accounts-svc.apps2.edwardjones.com/v2/accounts/{accountId}
https://con-contacts-rest.apps2.edwardjones.com/contacts/{contactId}?taxInfo=Y
```

Context: `account` and `contact` are two of the 8 Redis caches that are switched off
on purpose (Aiping confirmed, Aug 21). This is the data for deciding whether to switch
them on.

Two possible sources:

```
callChart log line   proven - already used in the BPP notebook and bulkhead work.
                     One line per request, lists every downstream call with its ms.
                     Gives volume and latency. No account/contact IDs.
spans                not yet tried on CDDR. Carries the full URL, so it has the IDs.
                     Only needed for the repeat-rate tile.
```

## Step 0 - how callChart names these two calls (run first)

```
fetch logs, from: now()-1h, scanLimitGBytes: -1, samplingRatio: 1, bucket:{"cddr"}
| filter k8s.namespace.name == "cddr-ns"
| filter matchesPhrase(content, "callChart")
| fields timestamp, content
| limit 5
```

Open one row. Known labels so far: `Relationship call (183ms)`,
`BankingPartnerPlatformAccountSummary call (54ms)`. Find the label for the
accounts call and the contacts call, and write them down exactly.

## Tile 5a - calls per 5 min, from callChart (use this one)

Put the two labels from Step 0 in place of ACCOUNT_LABEL and CONTACT_LABEL.

```
fetch logs, scanLimitGBytes: -1, samplingRatio: 1, bucket:{"cddr"}
| filter k8s.namespace.name == "cddr-ns"
| filter matchesPhrase(content, "callChart")
| fieldsAdd acct = if(contains(content, "ACCOUNT_LABEL call ("), 1, else: 0),
            cont = if(contains(content, "CONTACT_LABEL call ("), 1, else: 0)
| makeTimeseries account_calls = sum(acct), contact_calls = sum(cont), interval: 5m
```

Counts requests that made the call. If one request calls accounts twice, it counts
once. No `from:` so the tile follows the dashboard timeframe.

## Tile 5b - how slow those calls are

Slow calls are the other reason to cache. Same labels.

```
fetch logs, scanLimitGBytes: -1, samplingRatio: 1, bucket:{"cddr"}
| filter k8s.namespace.name == "cddr-ns"
| filter matchesPhrase(content, "callChart")
| parse content, "LD 'ACCOUNT_LABEL call (' INT:acct_ms 'ms)' LD"
| filter isNotNull(acct_ms)
| makeTimeseries p50 = percentile(acct_ms, 50), p95 = percentile(acct_ms, 95), interval: 15m
```

Copy it once more with CONTACT_LABEL for the contacts line.

---

The rest of Part 2 (spans) is only for the repeat-rate tile. Skip it if Maosheng
only wants volume.

Source: spans (outgoing HTTP calls). Field names checked against the Dynatrace
semantic dictionary: span.kind, server.address, url.path, url.full.

## Step 1 - check the data exists (run first, last 1 hour)

```
fetch spans, from: now()-1h
| filter span.kind == "client"
| filter server.address == "gna-accounts-svc.apps2.edwardjones.com"
      or server.address == "con-contacts-rest.apps2.edwardjones.com"
| fields start_time, server.address, url.path, http.response.status_code,
         k8s.namespace.name, k8s.pod.name, dt.entity.service,
         supportability.atm_sampling_ratio, aggregation.count
| limit 20
```

Look at three things:

```
rows at all?                    no rows -> CDDR's calls are not traced; stop and tell me
k8s.namespace.name              should say cddr-ns. If empty, tell me what IS filled
                                in (dt.entity.service or k8s.pod.name) - I'll swap the filter
supportability.atm_sampling_ratio   empty or 1 -> counts are exact
                                    above 1 -> spans are sampled, use the Step 2b version
```

## Tile 5 - Calls per minute, accounts vs contacts

```
fetch spans
| filter span.kind == "client"
| filter k8s.namespace.name == "cddr-ns"
| filter (server.address == "gna-accounts-svc.apps2.edwardjones.com" and startsWith(url.path, "/v2/accounts/"))
      or (server.address == "con-contacts-rest.apps2.edwardjones.com" and startsWith(url.path, "/contacts/"))
| fieldsAdd api = if(server.address == "gna-accounts-svc.apps2.edwardjones.com", "accounts", else: "contacts")
| makeTimeseries calls = count(), by: {api}, interval: 1m
```

Line chart. Title: "Account and contact calls from CDDR (per minute)".

## Tile 6 - Repeat rate (the number that decides the cache)

Volume alone does not say whether a cache helps. What matters is how often the same
ID is asked for again. If 1,000 calls hit 950 different accounts, a cache saves
almost nothing. If they hit 100 accounts, it saves 90%.

```
fetch spans
| filter span.kind == "client"
| filter k8s.namespace.name == "cddr-ns"
| filter (server.address == "gna-accounts-svc.apps2.edwardjones.com" and startsWith(url.path, "/v2/accounts/"))
      or (server.address == "con-contacts-rest.apps2.edwardjones.com" and startsWith(url.path, "/contacts/"))
| fieldsAdd api = if(server.address == "gna-accounts-svc.apps2.edwardjones.com", "accounts", else: "contacts")
| fieldsAdd id = if(api == "accounts", substring(url.path, from: 13), else: substring(url.path, from: 10))
| summarize calls = count(), unique_ids = countDistinct(id), by: {api, hour = bin(start_time, 1h)}
| fieldsAdd repeat_pct = round(100.0 * (calls - unique_ids) / calls, decimals: 1)
| sort hour asc
```

Table. repeat_pct is roughly the best hit rate a 1-hour cache could get.
The id is only used inside the query - the tile shows counts, not account numbers.

## Step 2b - only if spans are sampled

In tile 5, replace `calls = count()` with:

```
calls = sum(coalesce(supportability.atm_sampling_ratio, 1) * coalesce(aggregation.count, 1))
```

Not verified on your tenant - compare against Step 1 before trusting it. Tile 6's
repeat_pct is a ratio, so sampling affects it less; leave it as is.
