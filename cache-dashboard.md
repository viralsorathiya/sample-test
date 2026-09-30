a. Dashboard "CDDR Cache Invalidator" exists in Dynatrace.
b. Cluster dropdown lists all six CDDR clusters and defaults to ppd.
c. All tiles return data for ppd, except skip reasons and Redis failures, which are
   empty when nothing was skipped or failed.
d. Tiles only show cddr-cache-invalidator, not other apps using Redis or Kafka.
e. Prod shows data once the app is deployed there (check after the prod rollout).
