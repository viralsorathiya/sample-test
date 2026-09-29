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

Dashboard -> settings (gear) -> Variables -> add. Choose the **List** type (not Code) -
an empty Code variable makes every tile return 0 records with no error.

```
name     cluster
type     list
values   aks04074dvscu01, aks04074tescu01, aks04074stscu01, aks04074stncu01, aks04074prscu01, aks04074prncu01
default  aks04074stscu01   (ppd. 28 Sep: data in dev, uat, ppd, stage-dr - not prod yet)
```

---

## How every tile is built (changed 28 Sep)

Same shape as Step 0, which is proven to return data: split by the cluster and app
entities, then filter on their names as a pipeline step (`| filter ...`). The first
version put `entityName()` inside `timeseries filter:`, which the Dynatrace docs do not
document, and the tiles came back empty.

`$cluster` in a pipeline filter is the documented form (docs example:
`| filter host.name == $Host`). The value is inserted with double quotes.

## Tile 1 - Kafka messages: received, skipped, dead-lettered (line)

```
timeseries {
  received    = sum(`cddr.kafka.records.received`),
  skipped     = sum(`cddr.kafka.records.skipped`),
  dead_letter = sum(`cddr.kafka.records.dead_lettered`)
}, by: { topic, dt.entity.kubernetes_cluster, dt.entity.cloud_application }, interval: 1m,
union: true
| filter entityName(dt.entity.cloud_application) == "cddr-cache-invalidator"
     and entityName(dt.entity.kubernetes_cluster) == $cluster
```

`union: true`: per the docs, without it only series present in all three metrics are
returned (like an INNER JOIN), so no dead-letters would empty the tile.

## Tile 2 - Invalidations by cache and outcome (line)

```
timeseries events = sum(`cddr.cache.invalidation`), by: { cache, outcome, dt.entity.kubernetes_cluster, dt.entity.cloud_application }, interval: 1m
| filter entityName(dt.entity.cloud_application) == "cddr-cache-invalidator"
     and entityName(dt.entity.kubernetes_cluster) == $cluster
```

## Tile 3 - Failure ratio per cache (table)

```
timeseries
  failures = sum(`cddr.cache.invalidation`, filter: { outcome == "failure" }),
  total    = sum(`cddr.cache.invalidation`),
  by: { cache, dt.entity.kubernetes_cluster, dt.entity.cloud_application }, interval: 5m,
union: true
| filter entityName(dt.entity.cloud_application) == "cddr-cache-invalidator"
     and entityName(dt.entity.kubernetes_cluster) == $cluster
| fieldsAdd failure_ratio = arraySum(failures) / arraySum(total)
| fields cache, failure_ratio
```

## Tile 4 - Keys invalidated per cache (line)

```
timeseries keys = sum(`cddr.cache.keys.invalidated`), by: { cache, dt.entity.kubernetes_cluster, dt.entity.cloud_application }, interval: 1m
| filter entityName(dt.entity.cloud_application) == "cddr-cache-invalidator"
     and entityName(dt.entity.kubernetes_cluster) == $cluster
```

## Tile 5 - Message handling time, avg and max, ms (line)

The app doc's p95 version fails in Dynatrace: "timeseries percentile function requires
a rollup with the given metric key(s)". The metric is not stored in a form Dynatrace
can take a true p95 from, so avg and max are used instead.

```
timeseries {
  avg_ms = avg(`cddr.kafka.message.handling.time`),
  max_ms = max(`cddr.kafka.message.handling.time`)
}, by: { topic, dt.entity.kubernetes_cluster, dt.entity.cloud_application }, interval: 1m,
union: true
| filter entityName(dt.entity.cloud_application) == "cddr-cache-invalidator"
     and entityName(dt.entity.kubernetes_cluster) == $cluster
```

## Tile 6 - Redis invalidation time per cache, ms (line)

```
timeseries invalidate_ms = avg(`cddr.redis.invalidate.time`), by: { cache, dt.entity.kubernetes_cluster, dt.entity.cloud_application }, interval: 1m
| filter entityName(dt.entity.cloud_application) == "cddr-cache-invalidator"
     and entityName(dt.entity.kubernetes_cluster) == $cluster
```

## Tile 7 - Consumer lag (line)

How far behind Kafka the app is. Rising and not coming back down = falling behind.

```
timeseries lag_max = max(`kafka.consumer.fetch.manager.records.lag.max`), by: { client.id, dt.entity.kubernetes_cluster, dt.entity.cloud_application }, interval: 1m
| filter entityName(dt.entity.cloud_application) == "cddr-cache-invalidator"
     and entityName(dt.entity.kubernetes_cluster) == $cluster
```

## Tile 8 - Assigned partitions (line)

Drops to 0 = the consumer left the group.

```
timeseries assigned = avg(`kafka.consumer.coordinator.assigned.partitions`), by: { client.id, dt.entity.kubernetes_cluster, dt.entity.cloud_application }, interval: 1m
| filter entityName(dt.entity.cloud_application) == "cddr-cache-invalidator"
     and entityName(dt.entity.kubernetes_cluster) == $cluster
```

## Tile 9 - Why messages were skipped (line)

Empty is normal if nothing was skipped.

```
timeseries skipped = sum(`cddr.kafka.records.skipped`), by: { topic, reason, dt.entity.kubernetes_cluster, dt.entity.cloud_application }, interval: 5m
| filter entityName(dt.entity.cloud_application) == "cddr-cache-invalidator"
     and entityName(dt.entity.kubernetes_cluster) == $cluster
```

## Tile 10 - Redis failures by exception (line)

Empty is normal if nothing failed.

```
timeseries failures = sum(`cddr.redis.invalidation.failures`), by: { cache, exception, dt.entity.kubernetes_cluster, dt.entity.cloud_application }, interval: 5m
| filter entityName(dt.entity.cloud_application) == "cddr-cache-invalidator"
     and entityName(dt.entity.kubernetes_cluster) == $cluster
```

---

Left out on purpose (in the app doc if wanted later): batch handling time, fetch
latency, commit rate, retry attempts, lettuce first-response, total Redis command
volume, and the metric-selector versions (those are for classic dashboards).
