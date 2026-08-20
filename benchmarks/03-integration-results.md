# 03 - Integrate: RAG pipeline run

Host `Windows-AMD64` · llama.cpp `b10488` ·
retrieval backend: **keyword overlap** · 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.1 | 7549.9 | 7550.0 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.1 | 6634.1 | 6634.3 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.1 | 8680.9 | 8681.0 |

Mean per stage (ms): embed **0.0** · retrieve **0.1** ·
llm **7621.6** · total **7621.8**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Based on the context provided, **Goodput** is more useful than raw throughput because it specifically accounts for SLOs (Service Level Objectives) by counting only requests that met the targets.

Raw throughput ignores SLOs, whereas Goodput counts requests per second that met the TTFT and TPOT targets. This means Goodput provides a more accurate and realistic view of system performance relative to

**What problem does PagedAttention actually solve?**

> PagedAttention solves the problem of **internal fragmentation in GPU memory** by storing the KV cache in non-contiguous pages.

This allows the model to use less total memory than a contiguous array would, as it avoids the fragmentation that would occur if the KV cache were stored in a single, contiguous block of memory.

**When does splitting prefill and decode help?**

> Splitting prefill and decode helps when **prefill is compute-bound and decode is memory-bound**.

In this context, the context explicitly states that prefill is compute-bound (requiring significant CPU processing) and decode is memory-bound (requiring significant GPU memory bandwidth). By splitting these operations, the system can offload the compute-intensive prefill step to a separate pool, allo


## Which N16-N19 pieces are real

1. N16 Cloud/IaC: Stub
2. N17 Data pipeline: Stub
3. N18 Lakehouse: Stub
4. N19 Vector + features: Stub
5. N20 Serving: Real (llama-server)

The dominant stage is the LLM decode (100% of total latency, mean of 7621.6 ms), which matches expectations because LLM inference is highly compute-bound and memory-bound compared to the lightweight keyword search retrieval (0.1 ms). If I had to halve the pipeline's latency, I would attack the LLM stage, either by using a more aggressive quantization (e.g. Q4 instead of Q8 if applicable) or enabling speculative decoding (MTP/draft model) to speed up token generation.
