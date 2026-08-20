# 01 - Tune: thread-count sweep

Model `Qwen3.5-0.8B-Q4_K_M.gguf` · host `Windows-AMD64` · llama.cpp `b10488`
CPU: **4 physical · 8 logical** cores · `ngl=99` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 29.2 | 100% |
| 2 | 27.6 | 95% |
| 4 | 27.6 | 94% |
| 8 | 25.3 | 87% |
| 16 | 27.4 | 94% |

**Best**: `-t 1` at 29.2 tok/s
**Slowest tested**: `-t 8` at 25.3 tok/s (1.15x spread)
**Against the physical-core default** (`-t 4`, 27.6 tok/s): 1.06x

Use this in your run:

```bash
LAB_N_THREADS=1 make bench
```

## Your explanation

The measurements show that -t 1 (29.2 tok/s) is the best thread count, and as the thread count increases, the performance decreases slightly (with -t 8 being the slowest at 25.3 tok/s). 

This behavior is expected because we are using a small model (Qwen3.5 0.8B) and offloading all 99 layers to the GPU (ngl=99) via the Vulkan backend. Because the entire computation is executed on the integrated Vulkan GPU (Intel Iris Xe), the CPU's primary role is orchestrating and launching GPU kernels. Increasing thread count does not parallelize the tensor computations (since they run on the GPU) but instead introduces CPU thread synchronization overhead and scheduling contention. Thus, a single CPU thread (-t 1) is sufficient and optimal for kernel launch orchestration.
