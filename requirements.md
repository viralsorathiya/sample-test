Hi Adam, 

I'd like to give you an update on the BulkheadFull exceptions. 

Other than the cold start ones we already knew about, here's what I found.

Most of them come from short stalls where our calls out to apps2/apps3 stop getting answers for a minute or so. Threads sit waiting and keep hold of their bulkhead slots, all 50 fill up, and new requests get rejected. During these the traffic actually drops rather than rises, so it's calls hanging rather than extra load.

It isn't only us when it happens. Apps on 10+ other clusters see timeouts in the same minute, across AKS and DKP and both STL and PHX, and DKP's own API server logs a timeout at the same second. 

Cold start is only a small slice. Taking 3 Sep as an example, 1,120 of the 1,165 exceptions were on pods that had been up for hours, and only 41 were on pods that had just started. The Istio warmup change still helps but covers about 3% of these.

The pattern has repeated on 26 and 27 Aug and on 2, 3 and 8 Sep, always in the morning and always short-lived.

The bulkhead does its job through all of it: the probes still get through and the pods stay up.
