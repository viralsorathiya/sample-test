Q5 is wrong. You classified each pod by the age at its FIRST exception, then
attributed all of that pod's exceptions to that classification.

Pod 5f8ffc7cd7-gj574 had one exception at 14:48:50, twelve seconds after
startup. The Aug 10 outage was at 18:09, four hours later, on pods running
three days. If gj574 also fired during 18:09, those exceptions were counted
as cold start.

That matters: the Aug 10 18:00 hour alone was 42,809 exceptions across six
pods, all established. Your table reports established pods at 808 total
across the whole period. Both cannot be true.

Redo it per exception, not per pod:

For every BulkheadFullException, find the most recent
"Started CddrMainSubgraphApplication" line for THAT pod BEFORE THAT
exception, and compute the age at that individual exception. Then bucket
each exception, not each pod.

Give me:
  - exception count by age band: under 60s, 1-10 min, 10-60 min, over 1 hour
  - the same split for before and after 2026-08-21
  - for 2026-08-10 specifically, the count in each band

Numbers only. If a pod has no startup line in retention, put those exceptions
in a separate "age unknown" column rather than dropping them.
