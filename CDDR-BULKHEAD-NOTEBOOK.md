fetch logs, from: now()-6h, scanLimitGBytes: -1
| filter matchesPhrase(content, "bpp-account-svc")
| filter matchesPhrase(content, "502")
| summarize errors = count(), by: {minute = bin(timestamp, 1m)}
| sort minute desc

fetch logs, from: now()-6h, scanLimitGBytes: -1
| filter matchesPhrase(content, "bpp-accounts-summary")
| fieldsAdd failed = if(matchesPhrase(content, "502"), 1, else: 0)
| summarize total = count(), failures = sum(failed), by: {hour = bin(timestamp, 1h)}
| sort hour desc
