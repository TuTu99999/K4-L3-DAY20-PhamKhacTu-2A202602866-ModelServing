# 02 - Continuous batching under load (u50)

Host `Windows-AMD64` · `--parallel 4` · 14 samples over
60s at 2.0s intervals · raw CSV: `02-server-metrics-u50.csv`

| Gauge | Peak observed |
|:--|--:|
| `n_busy_slots_per_decode` (avg/decode) | 3.76 of 4 slots (94%) |
| `requests_processing` | 4 |
| `requests_deferred` | 46 |
| `kv_cache_usage_ratio` | n/a — not exported by llama.cpp `b10488` |
| `tokens_predicted_total` (final) | 1991 |

Highest sampled value was **3.76 of 4** slots. Note this gauge is llama.cpp's *average* busy slots per decode step, so the number below is the highest average we sampled, not an instantaneous maximum batch width. A peak near 1 means
requests were served one at a time -- either the load was too light to overlap, or
they arrived too far apart. A peak approaching `--parallel` means the scheduler was
genuinely packing concurrent requests into shared decode steps.
`requests_deferred` went above zero: more requests arrived than there were slots, so some waited. That wait is the queue time in your P95.

## Your observation

The highest sampled batch width was 3.76 of 4 slots (94%), while the server also
reported 4 processing requests and as many as 46 deferred requests. This clearly
shows that continuous batching kept almost every decode slot busy and that the
50-user workload exceeded the server's four-slot capacity. The 16.0 effective
concurrency in `02-server-results.md` is higher than the batch width because Little's
Law includes requests waiting in the queue as well as requests actively decoding.
For actual batching width I trust the server gauge, because it measures busy slots
inside llama.cpp directly; effective concurrency is instead useful evidence of the
combined in-service and queued workload.
