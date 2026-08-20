# 01 - Measure: latency baseline

Model `Qwen3.5 0.8B`  host `Windows-AMD64`  llama.cpp `b10488`
Settings: `threads=4` `ngl=99` `ctx=2048`
`max_tokens=64`  warm-up discarded
Completed requests: `Q4_K_M` 10/10  `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| Q4_K_M | 0.50 | 8629 | 1419 / 1517 | 43.0 / 44.6 | 4105 / 4242 / 4242 | 23.3 |
| UD-Q2_K_XL | 0.39 | 12244 | 1859 / 1986 | 617.6 / 621.1 | 40807 / 40985 / 40985 | 1.6 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` decodes **14.56x SLOWER** than `Q4_K_M` here, despite being 0.11 GB smaller. That is a real result, not a mistake: fewer bits only buys speed when decode is limited by memory bandwidth. On a machine that is compute-limited instead � few cores, no GPU offload  the extra dequantization work of a heavily-quantized format can cost more than the bytes it saves. Say which case yours is.

## Your observation

Based on the measurements on my machine (11th Gen Intel i5-1135G7 with Vulkan integrated GPU):
1. The Q4_K_M quantization (0.50 GB) runs at 23.3 tok/s (TPOT = 43.0 ms).
2. The UD-Q2_K_XL quantization (0.39 GB) runs at 1.6 tok/s (TPOT = 617.6 ms), which is 14.56x slower.
3. Why this happens: My system uses an integrated GPU (Intel Iris Xe) under Vulkan, which is highly compute-limited and lacks optimized tensor or hardware support for 2-bit mixed dequantization. The massive dequantization arithmetic overhead of UD-Q2_K_XL completely offsets the small memory-saving (0.11 GB or 22% size reduction).
4. Conclusion: The smaller quantization (UD-Q2_K_XL) is absolutely NOT worth it on my machine. Q4_K_M is much faster and provides higher response quality, while UD-Q2_K_XL is unusable due to the heavy compute bottleneck.
