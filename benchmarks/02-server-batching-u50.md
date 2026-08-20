# 02 - Continuous batching under load (u50)

Host `Windows-AMD64` · `--parallel 4` · 14 samples over
60s at 2.0s intervals · raw CSV: `02-server-metrics-u50.csv`

| Gauge | Peak observed |
|:--|--:|
| `n_busy_slots_per_decode` (avg/decode) | 2.33 of 4 slots (58%) |
| `requests_processing` | 4 |
| `requests_deferred` | 46 |
| `kv_cache_usage_ratio` | n/a — not exported by llama.cpp `b10488` |
| `tokens_predicted_total` (final) | 1447 |

Highest sampled value was **2.33 of 4** slots. Note this gauge is llama.cpp's *average* busy slots per decode step, so the number below is the highest average we sampled, not an instantaneous maximum batch width. A peak near 1 means
requests were served one at a time -- either the load was too light to overlap, or
they arrived too far apart. A peak approaching `--parallel` means the scheduler was
genuinely packing concurrent requests into shared decode steps.
`requests_deferred` went above zero: more requests arrived than there were slots, so some waited. That wait is the queue time in your P95.

## Your observation

1. Peak batch width was 2.33 out of 4 slots.
2. Comparison: The peak batch width (2.33) is lower than the effective concurrency (5.4) reported in 02-server-results.md.
3. Why they disagree: The peak batch width represents the average number of slots actively decoding at any single step. The effective concurrency (5.4) is calculated via Littles Law, which counts all in-flight requests including those queued (deferred = 46) waiting for a slot. Since many requests were queued, the total occupancy (effective concurrency) was higher than active slot utilization.
4. Trust: I trust peak batch width for measuring actual GPU/server compute density and parallel slot utilization, while effective concurrency is useful for understanding total queue queueing backlog.
