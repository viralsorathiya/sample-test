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
