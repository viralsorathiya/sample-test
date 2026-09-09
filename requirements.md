timeseries available = min(`resilience4j.bulkhead.available.concurrent.calls`, default: 50),
  filter: { k8s.namespace.name == "cddr-ns" },
  by: { k8s.pod.name },
  interval: 1m
| fieldsAdd lowest_free = arrayMin(available)
| fields k8s.pod.name, lowest_free
| sort lowest_free asc
