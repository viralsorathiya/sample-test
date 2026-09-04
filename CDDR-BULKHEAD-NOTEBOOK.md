fetch logs, from: now()-30m, scanLimitGBytes: -1
| filter matchesPhrase(content, "bpp-accounts-summary")
| filter matchesPhrase(content, "502 Bad Gateway")
| summarize failures = count(), last_failure = max(timestamp)
| fieldsAdd minutes_ago = round(toLong(now() - last_failure) / 60000000000, decimals: 1)
