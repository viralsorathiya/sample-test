Step 1 — Confirm it is still failing

Dynatrace → Logs

fetch logs, from: now()-1h, scanLimitGBytes: -1
| filter matchesPhrase(content, "bpp-accounts-summary")
| fieldsAdd failed = if(matchesPhrase(content, "502 Bad Gateway"), 1, else: 0)
| summarize total = count(), failures = sum(failed), by: {minute = bin(timestamp, 1m)}
| fieldsAdd rate = round(failures * 100.0 / total, decimals: 1)
| sort minute desc

Looking for: does the most recent minute still show failures. That is your "is it live" answer.

---

Step 2 — Find out whether BPP is monitored at all

Dynatrace → Services → search box → type bpp

Looking for: a service named something like bpp-account-svc.

It appears        go to Step 3 - you can see their side directly
Nothing found     go to Step 4 - you only have our side

This single check decides whether you can prove the cause or only describe the symptom.

---

Step 3 — If BPP is in Dynatrace

Open the service, then:

a. Failure rate — does their failure rate match the ~18% we see? If theirs is near zero, the failure is happening between the gateway and the backend, and never reaches the application.

b. Response time — any spike that lines up with our errors.

c. Failure analysis tab — what exception are they throwing, if any.

d. Process/pod availability — restarts or instances dropping out. That is what makes a backend pool member unhealthy.

The key comparison: if BPP shows no errors while we see 502s, the requests are dying at the gateway. That is infrastructure, not their application code.

---

Step 4 — Establish what kind of failure it is

Get one complete failing log line:

fetch logs, from: now()-2h, scanLimitGBytes: -1
| filter matchesPhrase(content, "bpp-accounts-summary")
| filter matchesPhrase(content, "502 Bad Gateway")
| fields timestamp, content
| limit 3

Looking for: a duration or elapsed time in the line.

under 1 second     the gateway had no healthy backend to send to
around 20 seconds  a backend timeout

That distinction matters because of what the Azure documentation says.

---

Step 5 — Apply the Azure rule

Microsoft's troubleshooting doc for Application Gateway:

▎ "In Application Gateway v2, if the application gateway doesn't receive a response from the backend application within this interval, the request is tried against a second backend pool member. If the second request also fails, the user request gets a 504 error instead."

The error page in our log says Microsoft-Azure-Application-Gateway/v2.

So a slow backend would give us 504. We are getting 502. Backend slowness is ruled out by the status code itself.

What Microsoft lists as remaining causes of a 502:

NSG, UDR or DNS blocking the gateway from reaching the backend
health probe cannot reach the backend
backend pool empty or unconfigured
unhealthy instances in the backend pool
upstream TLS certificate does not match

---

Step 6 — Narrow it with the failure rate

We fail about 18% of calls, not all of them.

empty pool           would be 100% failure
whole pool unhealthy would be 100% failure
cert mismatch        would be 100% failure

None of those fit. What fits an intermittent rate:

- backend members flapping between healthy and unhealthy
- one member in the pool passing the health probe but failing real requests

18% is roughly 1 in 5.5 — the shape you would get from one bad member in a pool of five or six.

---

Step 7 — Rule ourselves out

fetch logs, from: now()-6h, scanLimitGBytes: -1
| filter matchesPhrase(content, "bpp-accounts-summary")
| fieldsAdd failed = if(matchesPhrase(content, "502 Bad Gateway"), 1, else: 0)
| summarize total = count(), failures = sum(failed), by: {pod = k8s.pod.name}
| fieldsAdd rate = round(failures * 100.0 / total, decimals: 1)
| sort rate desc

Looking for: whether every CDDR pod fails at a similar rate.

all similar     not us - the problem is on the far side
one much worse  it is us - that pod has a connection or DNS problem

---

Step 8 — Check whether it is data-specific

fetch logs, from: now()-6h, scanLimitGBytes: -1
| filter matchesPhrase(content, "bpp-accounts-summary")
| filter matchesPhrase(content, "502 Bad Gateway")
| parse content, "LD 'relationships/' LD:rel_id '/bpp-accounts-summary' LD"
| summarize failures = count(), by: {rel_id}
| sort failures desc
| limit 20

Looking for: whether a handful of relationship IDs account for most failures.

a few IDs dominate   specific accounts are triggering it - a BPP data problem
evenly spread        infrastructure, nothing to do with the request content

---

What you can then tell BPP

Once steps 4, 7 and 8 are done you can say, with evidence:

- it is still happening, at X% of calls
- the 502 comes from Azure Application Gateway, not from their application
- on v2 a slow backend returns 504, so this is not a timeout
- it is intermittent, which rules out an empty pool, a fully unhealthy pool, and a certificate mismatch
- it is not specific to any CDDR pod, and [is / is not] specific to particular relationship IDs

Which leaves them one place to look: backend pool health on their Application Gateway.
