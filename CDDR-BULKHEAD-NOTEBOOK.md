fetch logs, from: now()-30m, scanLimitGBytes: -1
| filter matchesPhrase(content, "bpp-accounts-summary")
| filter matchesPhrase(content, "502 Bad Gateway")
| parse content, "LD 'requestId=' LD:request_id ',' LD"
| parse content, "LD 'relationships/' INT:relationship_id '/bpp-accounts-summary' LD"
| fields timestamp, request_id, relationship_id, pod = k8s.pod.name, cluster = k8s.cluster.name
| sort timestamp desc
| limit 10
