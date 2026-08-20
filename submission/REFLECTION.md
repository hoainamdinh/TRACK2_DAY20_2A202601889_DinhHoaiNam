# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.
>
> `make verify` sẽ fail nếu còn placeholder chưa điền. Đó là cố ý.

**Họ Tên:** Đinh Hoài Nam
**Cohort:** Cohort 3 - Track 2 (3A202601889) - E403
**Ngày submit:** 20/08/2026

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

> Từ `make probe`. Paste output hoặc điền tay.

- **OS:** Windows 11
- **CPU:** 11th Gen Intel(R) Core(TM) i5-1135G7
- **Cores:** 4 physical / 8 logical
- **CPU extensions:** AVX2
- **RAM:** 16 GB
- **Accelerator:** Vulkan
- **llama.cpp asset đã tải:** llama-b10488-bin-win-vulkan-x64.zip
- **Model đã dùng:** Qwen3.5 0.8B (LAB_MODEL=qwen35-0.8b)
- **Quantization:** Q4_K_M + UD-Q2_K_XL

**Chạy ở đâu:** laptop của tôi

**Setup story** (≤ 80 chữ): điều gì cần thay đổi để lab chạy trên máy bạn? Có bước nào fail rồi phải workaround không?
Tôi sử dụng model Qwen3.5 0.8B. Khi chạy `.\lab.ps1 serve` ban đầu bị lỗi "invalid argument: at" do đường dẫn thư mục chứa khoảng trắng ("Codelabs at VinUni") làm hàm `os.execv` trên Windows phân tách sai đối số. Tôi đã workaround bằng cách sửa `serve.py` để sử dụng `subprocess.run` thay thế cho `os.execv` khi chạy trên hệ điều hành Windows.
---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

> Paste bảng từ `benchmarks/01-quickstart-results.md` (`make bench` tự sinh).

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| Q4_K_M | 0.50 | 8629 | 1419 / 1517 | 43.0 / 44.6 | 4105 / 4242 / 4242 | 23.3 |
| UD-Q2_K_XL | 0.39 | 12244 | 1859 / 1986 | 617.6 / 621.1 | 40807 / 40985 / 40985 | 1.6 |

**Quan sát** (≤ 60 chữ): Bản 2-bit chậm hơn 14.56 lần so với 4-bit do dequantization overhead trên iGPU rất lớn. Hoàn toàn không đáng để đánh đổi vì tiết kiệm RAM rất ít (0.11 GB) nhưng hiệu năng giảm sâu không thể dùng được.

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

> Từ `benchmarks/02-server-results.md` (`make load-report`).

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 0.13 | 44000 | 45000 | 45000 | 4.5 | 0.0% |
| 50 | 0.15 | 44000 | 46000 | 46000 | 5.4 | 0.0% |

- **Offered load tăng 5×, throughput thực tăng:** 1.15×
- **P95 tăng:** 1.02×
- **Effective concurrency ở 50 users:** 5.4 so với `--parallel` = 4 slots

**Peak `llamacpp:n_busy_slots_per_decode`** (từ `make metrics` khi `make load-50` đang chạy): 2.33 / 4 slots

**Saturation reading** (≤ 80 chữ): Server bão hòa dưới 50 users khi throughput chỉ tăng 1.15x khi load tăng 5x, và effective concurrency (5.4) vượt quá 4 slots gây nghẽn hàng đợi (deferred=46). Tôi sẽ tăng --parallel lên 8 để tận dụng xử lý song song của continuous batching.

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

> Từ `make pipeline`. Nói thật cái nào real, cái nào stub — stub **không** mất điểm.

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | GCP Project | stub |
| N17 Data pipeline | Airflow DAG | stub |
| N18 Lakehouse | BigQuery | stub |
| N19 Vector + features | Vector search | stub |
| N20 Serving | `llama-server` | real |

**Latency split** (mean của 3 query, từ output của `pipeline.py`):

- embed: 0.0 ms
- retrieve: 0.1 ms
- llm: 7621.6 ms
- **stage chiếm nhiều nhất:** llm (100% của total)

**Reflection** (≤ 60 chữ): Bottleneck ở LLM (100% latency), đúng kỳ vọng vì truy vấn từ khóa rất nhanh còn LLM sinh text cực kỳ tốn tính toán. Để giảm latency 2x, tôi sẽ giảm số thread CPU về 1 hoặc dùng speculative decoding.

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

> **Phần quan trọng nhất của report.** Không cần bonus track: `make tune` đã cho bạn
> một before/after thật (`benchmarks/01-tuning-tg128.md`). Đổi quantization,
> `LAB_N_CTX`, hay `--parallel` rồi đo lại cũng được.

**Change:** Hạ số thread (-t) từ 4 xuống 1 khi chạy mô hình hoàn toàn trên GPU (ngl=99)

```
before:  27.6 tok/s (t=4)
after:   29.2 tok/s (t=1)
speedup: 1.06x
```

**Tại sao nó work** (1–2 đoạn — đây là phần grader đọc kỹ nhất):

_Giải thích như đang nói với bạn ngồi cạnh. Bám vào **cơ chế**, không phải "vibes":
memory bandwidth? vector width? cache residency? scheduling? queueing? Nếu kết quả
**khác** với kỳ vọng từ deck — nói rõ, và giải thích vì sao. Grader thưởng điểm cho
lập luận đúng về một kết quả bất ngờ, hơn là một con số đẹp không được giải thích._

Khi chạy Qwen3.5 0.8B hoàn toàn trên iGPU (ngl=99), toàn bộ tính toán tensor đều do iGPU Intel Iris Xe xử lý. CPU chỉ đóng vai trò điều phối luồng và gọi (launch) các Vulkan GPU kernel.

Việc cấu hình nhiều thread CPU (-t 4 hoặc nhiều hơn) không tăng tốc độ tính toán (do GPU xử lý) mà ngược lại gây ra overhead đồng bộ hóa luồng giữa các nhân CPU và tranh chấp tài nguyên lập lịch gọi kernel. Do đó, giảm số thread CPU về tối thiểu (-t 1) là tối ưu nhất vì nó giảm tải CPU và tăng tốc độ kích hoạt kernel lên GPU.

---

## 6. Bonus  *(optional — tối đa 20 điểm)*

> Bỏ trống nếu không làm. Xem `bonus/README.md`. Đừng làm hết — **một** finding sâu
> ăn điểm hơn năm bảng nông.

**Đã làm:** B2 (sweep-batch) & B5 (semantic-cache-offline)

**Numbers:**

```
[B2: Prefill Batch Sweep]
before:  150.2 tok/s (-b 512 -ub 512)
after:   214.5 tok/s (-b 2048 -ub 512)
speedup: 1.43x

[B5: Semantic Cache (Offline)]
before:  8 LLM calls (no semantic cache, 0% hit rate)
after:   5 LLM calls (3 HITs saved, 38% hit rate)
speedup: Inf on hits (0ms vs ~7.5s total time skipped per hit)
```

**Điều này nói lên gì mà deck chưa nói:**

1. **Prefill Batch Size (B2):** Thí nghiệm chỉ ra rằng việc tăng kích thước logical batch (-b 2048) và micro-batch (-ub 512) giúp tăng tốc độ xử lý prefill lên 1.43x. Tuy nhiên, một micro-batch lớn sẽ giữ khóa tính toán GPU lâu hơn cho mỗi bước xử lý, khiến các request gửi đến sau phải chờ lâu hơn trong hàng đợi (tăng TTFT). Do đó, việc đo lường P95 TTFT dưới tải cao là bắt buộc để đảm bảo sự cân bằng giữ thông lượng và độ trễ.

2. **Semantic Cache (B5):** Việc sử dụng cache ngữ nghĩa (offline) giúp bắt được 38% số câu hỏi trùng lặp về ngữ nghĩa nhưng khác từ vựng (vd: "What is goodput at SLO?" và "Can you define goodput@SLO?"). Mỗi lần HIT tiết kiệm 100% chi phí tính toán (không prefill, không decode). Tuy nhiên, nếu threshold quá thấp sẽ dễ trả về câu trả lời sai ngữ cảnh, cần sử dụng các mô hình embedding chuyên dụng và kỹ thuật salting để bảo mật cache.

---

## 7. Điều làm bạn ngạc nhiên nhất  *(optional)*

Điều ngạc nhiên nhất là việc tăng số thread CPU khi chạy ngl=99 trên GPU lại làm giảm hiệu năng sinh token (TPOT) thay vì tăng lên, do phát sinh overhead đồng bộ hóa giữa các luồng CPU.

---

## 8. Self-check trước khi push

- [ ] `hardware.json` committed
- [ ] `models/active.json` committed
- [ ] `benchmarks/01-quickstart-results.md` committed (`make bench`)
- [ ] `benchmarks/01-tuning-tg128.md` committed (`make tune`)
- [ ] `benchmarks/02-server-results.md` committed (`make load-report`)
- [ ] `benchmarks/02-server-batching-u50.md` hoặc `-metrics-u50.csv` committed (`make metrics`)
- [ ] `benchmarks/locust-10_stats.csv` + `locust-50_stats.csv` committed (`make load-10` / `load-50`)
- [ ] `benchmarks/03-integration-results.md` committed (`make pipeline`)
- [ ] Mọi section **"required — replace this line"** trong các file `benchmarks/*.md`
      đã được thay bằng nhận xét của bạn
- [ ] 5 screenshots trong `submission/screenshots/`
- [ ] `make verify` → **exit 0**
- [ ] Repo GitHub ở chế độ **public**
- [ ] Đã paste public URL vào VinUni LMS
- [ ] **Không** commit `models/*.gguf` hay `runtime/` (đã có trong `.gitignore`)

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Private → grader không
xem được → 0 điểm.
