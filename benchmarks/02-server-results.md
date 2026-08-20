# 02 - Serve: load test + saturation reading

Host `Windows-AMD64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=4` ·
`ngl=99`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 6 | 0.13 | 44000 | 45000 | 45000 | 4.5 | 0.0% |
| 50 | 7 | 0.15 | 44000 | 46000 | 46000 | 5.4 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **1.15x** (23% of linear) |
| P95 latency | **1.02x** |
| Effective concurrency at 50 users | 5.4 vs `--parallel 4` slots (occupancy/slot ratio 1.35) |

**Saturated.** Throughput delivered only 1.15x for 5x the offered load, and effective concurrency (5.4) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

P95 grew no faster than throughput (1.02x vs 1.15x), so this server still has headroom at 50 users.

> **Small sample.** Only 6 requests completed in the
> shorter run, so these percentiles are indicative rather than solid. Note also that
> locust averages only *completed* requests: when the run ends with requests still
> queued, effective concurrency is an **under**-estimate. Trust the throughput-scaling
> row over the concurrency row here, and run longer (`-t 3m`) if you want firmer numbers.

## Your reading

Our server saturates at or below 50 users. The primary evidence is that the throughput delivered only increased by 1.15x (from 0.13 RPS to 0.15 RPS) when the offered load increased by 5x (from 10 to 50 users). The effective concurrency at 50 users (5.4) exceeds our parallel slot count (4), indicating that requests are being queued (deferred).

To raise goodput at our SLO under heavy load, the first knob I would change is the `--parallel` slots parameter (increasing it from 4 to 8, or scaling up to match the active queue). This would allow the server to utilize continuous batching more effectively for concurrent requests, reducing queue wait times, provided the system has enough memory and compute capacity.
