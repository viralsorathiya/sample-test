# Dashboard tickets

## Ticket 1

Title

```
CDDR: Dynatrace dashboard for Redis cache hits/misses and account/contact call volume
```

Description

```
1. After the move from Hazelcast to Redis we lost the per-cache visibility the WPaaG
   panels used to give us. Maosheng asked (15 Sep) for a dashboard built from his
   cache hit/miss query.

2. He also asked (22 Sep) for usage metrics on the accounts and contacts downstream
   calls, grouped by URL, before turning on caching for account and contact.

3. Dashboard "CDDR Redis and Cache" has been created in Dynatrace with:
   - Cache hits and misses (count) - Maosheng's query
   - Cache hit rate % per cache
   - CDDR calls to accounts and contacts, grouped by URL
   - A note that only 3 of CDDR's 11 Redis caches report hit/miss metrics

4. The hit/miss tiles cover 3 caches only: accountPriorDayTotalBalanceAmount,
   accountRestrictions and holding. The other 8 need Spring Redis cache statistics
   enabled (ODI-22529).

5. The accounts/contacts tile reads spans, which scan about 28 GB per hour. Dynatrace
   stops a query at 500 GB by default, so this tile works for timeframes up to about
   12 hours.
```

Acceptance Criteria

```
a. Dashboard "CDDR Redis and Cache" exists in Dynatrace.
b. Cache hits and misses tile shows data per cache for cddr-main-subgraph.
c. Cache hit rate % tile shows hits / (hits + misses) per cache.
d. Accounts/contacts tile shows CDDR's calls to
   gna-accounts-svc /v2/accounts/{accountId} and con-contacts-rest /contacts/{contactId},
   one line per URL.
e. A note on the dashboard states which caches are covered and references ODI-22529.
f. Dashboard shared with Maosheng.
```

Links

```
Related to   ODI-22529   enabling Spring Redis cache statistics adds the other 8 caches
```

---

## Ticket 2

Title

```
CDDR: Dynatrace dashboard for cddr-cache-invalidator
```

Description

```
1. cddr-cache-invalidator is a new worker, split out of cddr-main-subgraph. It reads
   account and contact change messages from Kafka and deletes the matching Redis cache
   entries. Maosheng asked (28 Sep) for a Dynatrace dashboard for it.

2. Dashboard "CDDR Cache Invalidator" has been created, using the queries in the app's
   docs/dynatrace-metrics-queries.md, with a cluster dropdown so one dashboard covers
   every environment.

3. As at 28 Sep the app sends data from dev, uat, ppd and stage-dr. Nothing from prod
   yet. The dashboard defaults to ppd (aks04074stscu01).

4. Tiles: Kafka messages received/skipped/dead-lettered, invalidations by cache and
   outcome, failure ratio per cache, keys invalidated, message handling time, Redis
   invalidation time, consumer lag, assigned partitions, skip reasons, Redis failures.

5. Two findings while building it:
   - The p95 queries in the app doc fail in our tenant ("timeseries percentile function
     requires a rollup"). Handling time uses avg and max instead.
   - The cluster variable must be the List type. An empty Code variable makes every
     tile return 0 records with no error.
```

Acceptance Criteria

```
a. Dashboard "CDDR Cache Invalidator" exists in Dynatrace.
b. Cluster dropdown lists all six CDDR clusters and defaults to ppd.
c. All tiles return data for ppd, except skip reasons and Redis failures, which are
   empty when nothing was skipped or failed.
d. Tiles only show cddr-cache-invalidator, not other apps using Redis or Kafka.
e. Prod shows data once the app is deployed there (check after the prod rollout).
f. Dashboard shared with Maosheng.
g. Maosheng told that the p95 queries in the app doc do not work in our tenant.
```

Links

```
Related to   ODI-22613   the cddr-cache-invalidator work (from the repo's commits)
```

---

## Ticket 2 - Acceptance Criteria (what the dashboard shows), plain lines for Jira

```
Cluster dropdown covering dev, uat, ppd, stage-dr, prod and prod-dr, defaulting to ppd
Kafka messages received, skipped and dead-lettered, per topic
Cache invalidations by cache (account, contact) and outcome (success, failure, read_only)
Invalidation failure ratio per cache
Redis keys invalidated per cache
Kafka message handling time, average and max, in milliseconds
Redis invalidation time per cache, in milliseconds
Kafka consumer lag
Kafka assigned partitions per consumer
Reasons messages were skipped
Redis invalidation failures by exception type
All tiles show cddr-cache-invalidator only, not other apps using Redis or Kafka
```
