# Bonus Challenge C2 - KV Cache Quantization

Host `Windows-AMD64` · Intel Iris Xe UMA iGPU · Vulkan Backend · llama.cpp `b10488` · Model `Qwen3.5-0.8B-Q4_K_M.gguf`

This challenge benchmarks the performance impact of KV Cache Quantization on an integrated GPU (UMA architecture) where memory bandwidth is shared with system RAM.

## Measurement Table

| Key Cache | Value Cache | Prefill (pp512) | Decode (tg128) | Memory Savings |
| :--- | :--- | :--- | :--- | :--- |
| **f16** *(baseline)* | **f16** | 170.1 tok/s | 23.9 tok/s | Baseline (100% size) |
| **q8_0** | **q8_0** | 199.1 tok/s | 24.5 tok/s | ~50% KV cache VRAM savings |
| **q4_0** | **q4_0** | 197.6 tok/s | 24.3 tok/s | ~75% KV cache VRAM savings |

## Key Findings

1. **Memory Bandwidth vs. Compute Trade-off:**
   On an integrated GPU (Intel Iris Xe), memory bandwidth is the primary bottleneck. Quantizing the KV cache to `q8_0` or `q4_0` reduces the total bytes read and written to system memory. Even though the GPU must perform additional arithmetic to dequantize the cache states during computation, the reduced memory transfer overhead results in a net speedup:
   * **Prefill Speedup (Q8):** **1.17x** (170.1 -> 199.1 tok/s)
   * **Decode Speedup (Q8):** **1.03x** (23.9 -> 24.5 tok/s)

2. **Quality and Selection Recommendation:**
   While `q4_0` saves the most VRAM (75% savings), it introduces quantization noise that can degrade the model's accuracy on long-context tasks (such as code generation or mathematical reasoning). Therefore, in production, **`q8_0` is the recommended option** as it offers a 1.17x throughput speedup and 50% memory savings with virtually zero degradation in model output quality.
