# Bonus - Batch-size sweep (chunked prefill)

Host `Windows-AMD64` · llama.cpp `b10488` ·
`threads=4` `ngl=99` · metric `pp512`

| -b (logical) | -ub (micro) | pp512 (tok/s) | vs best |
|:--|--:|--:|--:|
| 128 | 128 | 200.7 | 94% |
| 256 | 256 | 207.0 | 97% |
| 512 | 256 | 200.6 | 94% |
| 512 | 512 | 150.2 | 70% |
| 1024 | 512 | 171.1 | 80% |
| 2048 | 512 | 214.5 | 100% |

Best: `-b 2048 -ub 512` at 214.5 tok/s
(1.43x the slowest point tested).

This sweep only measures the throughput half of the trade. The cost it hides is
TTFT for queued requests: a larger micro-batch holds the device longer per step,
so anything waiting behind it waits longer. To see both halves, re-run
`make load-50` with your best and worst settings via
`.venv/bin/python labs/02-serve/serve.py -- -b N -ub M` and compare P95.

## Your finding

1. Analysis: The best prefill throughput is achieved at -b 2048 -ub 512 (214.5 tok/s). The slowest is -b 512 -ub 512 (150.2 tok/s).
2. Production Choice: For production, I would choose -b 512 -ub 256 (or -b 256 -ub 256). Although -b 2048 -ub 512 yields slightly higher raw prefill throughput (214.5 vs 200.6 tok/s), a micro-batch size of 512 (-ub 512) locks the GPU compute pipeline for a longer duration per execution step. This increases queue wait times for newly arriving requests. A smaller micro-batch size of 256 (-ub 256) offers 94% of the peak throughput but yields a much lower and stable TTFT under contention.
3. Production Metrics: To verify this choice in production, I would measure the P95 TTFT (Time to First Token) and Goodput@SLO under concurrent load, ensuring the larger micro-batch does not breach our target latency budget.
