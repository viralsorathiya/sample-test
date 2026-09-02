CDDR: Investigate per-cache metrics missing from WPaaG dashboard after Redis migration

Description

1. After the Hazelcast to Redis migration on 20 Aug, the per-cache panels on the
   WPaaG dashboard stopped showing data for CDDR. Caching itself is working -
   Redis is serving traffic normally - but we have lost visibility of hit and
   miss rates per cache.

2. Hazelcast publishes per-cache statistics by default. Spring Data Redis does
   not - they are disabled unless enableStatistics() is called on the
   RedisCacheManager, which is why the metrics stopped.

3. securityAndCurrentValue is the cache affected - 43 million lookups in the week
   before the migration and no metrics since. Three other active caches still
   report through custom counters in our own code, and the remaining seven are
   disabled so need no action.

4. So to restore this we can call enableStatistics() on the RedisCacheManager.
   That publishes the standard cache.gets metric with the result dimension, which
   WPaaG already reads, so no change is needed to the shared dashboard. The
   counters are held in the JVM and make no calls to Redis, so this cannot
   reintroduce the keyspace scans removed earlier for performance.

5. Cache size and eviction counts per cache cannot be restored - Redis has no way
   to count keys belonging to one cache without walking the whole keyspace.

Acceptance Criteria

a. enableStatistics() is enabled on the RedisCacheManager.
b. cache_gets_total with result=hit/miss is published for all enabled Redis
   caches, including securityAndCurrentValue.
c. WPaaG Hit Rates panel shows CDDR data with no change to the shared dashboard.
d. No increase in Redis operations attributable to metrics collection.
e. Decision recorded on whether the redundant custom counters are removed in the
   same change or a follow-up.


========
Investigation complete.

Caching was never affected - Redis was serving traffic throughout, with healthy
hits, misses, memory growth and connected clients. What we lost is the per-cache
metrics feeding the WPaaG panels.

Cause: Hazelcast publishes per-cache statistics by default. Spring Data Redis has
them disabled unless enableStatistics() is called on the RedisCacheManager, so
they stopped being published at the migration.

Confirmed in Dynatrace (metric cache.gets, namespace cddr-ns):
- securityAndCurrentValue: 43,131,019 lookups in the week before the migration,
  nothing since. Only active cache with no fallback.
- accountRestrictions, accountPriorDayTotalBalanceAmount and holding still report,
  but through custom counters in our own code using a non-standard metric name,
  which is why WPaaG cannot read them.
- The other seven Redis caches recorded zero lookups before the migration as well.
  Aiping Shi confirmed these are deliberately disabled - no action needed.

Also worth noting: Azure's Cache Hits and Cache Misses are per-second rates even
though the portal shows the unit as "Count". The 266 in the original report works
out at roughly 958,000 hits an hour, not 266.

Fix is enableStatistics() on the RedisCacheManager. Verified against Spring Boot's
RedisCacheMetrics source - the hit, miss and put counters are read from
cache.getStatistics(), held in the JVM, with no iteration or scanning of Redis
keys. It cannot reintroduce the keyspace SCANs removed earlier for performance.

Cache size and eviction counts per named cache are not recoverable. Spring returns
null for both on Redis, and Redis has no command to count keys by prefix - DBSIZE
is whole-database only. Azure provides a cache-wide key count instead, though on a
clustered cache the Total Keys metric reports the largest shard rather than the
true total.

Next step is the code change and a deploy. The three custom counters become
redundant once it ships.
