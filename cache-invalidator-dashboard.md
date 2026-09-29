# cddr-cache-invalidator dashboard

Maosheng: "can you create a dashboard for this new app in DT."

Queries are taken from the app's own `docs/dynatrace-metrics-queries.md`. One change
to each: a filter so the tile only shows this app in the chosen cluster. Without it,
`lettuce.*` and `kafka.consumer.*` also pick up every other app that uses Redis or
Kafka.

---

## Step 0 - run first, in a Notebook (Last 7 days)

Shows which clusters the app is sending data from, and proves the filter fields work.

```
timeseries received = sum(`cddr.kafka.records.received`),
  by: { dt.entity.kubernetes_cluster, dt.entity.cloud_application }
| fieldsAdd cluster = entityName(dt.entity.kubernetes_cluster),
            app     = entityName(dt.entity.cloud_application),
            total   = arraySum(received)
| fields cluster, app, total
```

```
rows with app = cddr-cache-invalidator    good - build the dashboard
rows but cluster/app empty                send a screenshot - filter needs changing
no rows                                   app not sending data yet - tell Maosheng
```

---

## Step 1 - dashboard variable

Dashboard -> settings (gear) -> Variables -> add:

```
name     cluster
type     list
values   aks04074dvscu01, aks04074tescu01, aks04074stscu01, aks04074stncu01, aks04074prscu01, aks04074prncu01
default  aks04074stscu01   (ppd. 28 Sep: data in dev, uat, ppd, stage-dr - not prod yet)
```

---

## Tile 1 - Kafka messages: received, skipped, dead-lettered (line)

```
timeseries {
  received     = sum(`cddr.kafka.records.received`),
  skipped      = sum(`cddr.kafka.records.skipped`),
  dead_letter  = sum(`cddr.kafka.records.dead_lettered`)
}, by: { topic }, interval: 1m,
filter: { entityName(dt.entity.cloud_application) == "cddr-cache-invalidator"
      and entityName(dt.entity.kubernetes_cluster) == $cluster },
union: true,
nonempty: true
```

`union: true` matters: without it, if one of the three metrics has no data (e.g. nothing
was ever dead-lettered), the whole tile comes back empty.

## Tile 2 - Invalidations by cache and outcome (line)

```
timeseries events = sum(`cddr.cache.invalidation`), by: { cache, outcome }, interval: 1m,
filter: { entityName(dt.entity.cloud_application) == "cddr-cache-invalidator"
      and entityName(dt.entity.kubernetes_cluster) == $cluster }
```

## Tile 3 - Failure ratio per cache (table or single value)

```
timeseries
  failures = sum(`cddr.cache.invalidation`, filter: { outcome == "failure" }),
  total    = sum(`cddr.cache.invalidation`),
  by: { cache }, interval: 5m,
filter: { entityName(dt.entity.cloud_application) == "cddr-cache-invalidator"
      and entityName(dt.entity.kubernetes_cluster) == $cluster }
| fieldsAdd failure_ratio = arraySum(failures) / arraySum(total)
| fields cache, failure_ratio
```

## Tile 4 - Keys invalidated per cache (line)

```
timeseries keys = sum(`cddr.cache.keys.invalidated`), by: { cache }, interval: 1m,
filter: { entityName(dt.entity.cloud_application) == "cddr-cache-invalidator"
      and entityName(dt.entity.kubernetes_cluster) == $cluster }
```

## Tile 5 - Message handling time p95, ms (line)

```
timeseries handling_p95_ms = percentile(`cddr.kafka.message.handling.time`, 95), by: { topic }, interval: 1m,
filter: { entityName(dt.entity.cloud_application) == "cddr-cache-invalidator"
      and entityName(dt.entity.kubernetes_cluster) == $cluster }
```

## Tile 6 - Redis invalidation time per cache, ms (line)

```
timeseries invalidate_ms = avg(`cddr.redis.invalidate.time`), by: { cache }, interval: 1m,
filter: { entityName(dt.entity.cloud_application) == "cddr-cache-invalidator"
      and entityName(dt.entity.kubernetes_cluster) == $cluster }
```

## Tile 7 - Consumer lag (line)

How far behind Kafka the app is. Rising and not coming back down = falling behind.

```
timeseries lag_max = max(`kafka.consumer.fetch.manager.records.lag.max`), by: { client.id }, interval: 1m,
filter: { entityName(dt.entity.cloud_application) == "cddr-cache-invalidator"
      and entityName(dt.entity.kubernetes_cluster) == $cluster }
```

## Tile 8 - Assigned partitions (line)

Drops to 0 = the consumer left the group.

```
timeseries assigned = avg(`kafka.consumer.coordinator.assigned.partitions`), by: { client.id }, interval: 1m,
filter: { entityName(dt.entity.cloud_application) == "cddr-cache-invalidator"
      and entityName(dt.entity.kubernetes_cluster) == $cluster }
```

## Tile 9 - Why messages were skipped (line)

```
timeseries skipped = sum(`cddr.kafka.records.skipped`), by: { topic, reason }, interval: 5m,
filter: { entityName(dt.entity.cloud_application) == "cddr-cache-invalidator"
      and entityName(dt.entity.kubernetes_cluster) == $cluster }
```

## Tile 10 - Redis failures by exception (line)

```
timeseries failures = sum(`cddr.redis.invalidation.failures`), by: { cache, exception }, interval: 5m,
filter: { entityName(dt.entity.cloud_application) == "cddr-cache-invalidator"
      and entityName(dt.entity.kubernetes_cluster) == $cluster }
```

---

Left out on purpose (in the app doc if wanted later): batch handling time, fetch
latency, commit rate, retry attempts, lettuce first-response, total Redis command
volume, and the metric-selector versions (those are for classic dashboards).
