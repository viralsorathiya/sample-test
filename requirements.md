fetch logs, from: "2026-09-03T16:10:00Z", to: "2026-09-03T16:30:00Z", scanLimitGBytes: -1, samplingRatio: 1, bucket:{"cddr"}
| filter k8s.namespace.name == "cddr-ns"
| filter contains(k8s.pod.name, "cddr-main-subgraph")
| summarize lines = count(), by: {minute = bin(timestamp, 1m)}
| sort minute asc
