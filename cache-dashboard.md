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
