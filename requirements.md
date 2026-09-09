What happens: CDDR takes one of 50 bulkhead slots per request and holds it until the request finishes, including all the calls out to apps2/apps3. Normally those answer in about 200ms so we sit at 5-10 slots used. On 3 Sep at 09:17 those services stopped answering, our threads sat waiting still holding their slots, all 50 filled, and anything new got rejected. That's the BulkheadFullException. It cleared on its own after a couple of minutes.

It wasn't a traffic spike - our request volume actually dropped 29% in that minute. Calls were stalling, not arriving faster.

It also isn't only us. Sixteen different downstream services stopped answering in the same minute, and ten other clusters saw the same thing simultaneously across both AKS and DKP and both STL and PHX. DKP's own Kubernetes API server logged a timeout at the exact same second, which no application can cause.

Cold start isn't the driver either. 1,120 of 1,165 exceptions that day were on pods that had been running over an hour. Only 41 were within a minute of a pod starting. The Istio warmup change still has a place but it covers about 3% of this, so it shouldn't be recorded as the fix.

The bulkhead behaved correctly throughout - it capped at 50 of our 60 Tomcat threads, kept headroom for the health probes, and the pods stayed up. I'd leave the settings as they are. Raising the limit would let a stall eat that headroom and start causing restarts.

Same pattern on 26 and 27 Aug, 2 and 8 Sep. Always mornings, always on prscu01, never prncu01.
