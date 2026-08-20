# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.
>
> `make verify` sẽ fail nếu còn placeholder chưa điền. Đó là cố ý.

**Họ Tên:** Nguyễn Bá Khánh Huy (2A202601591)
**Cohort:** AICB-P2T2 / A20-K1
**Ngày submit:** 2026-08-20

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

> Từ `make probe`. Paste output hoặc điền tay.

- **OS:** macOS 15.0 (Darwin 25.0.0 arm64)
- **CPU:** Apple M2
- **Cores:** 8 physical / 8 logical (4 P-cores + 4 E-cores)
- **CPU extensions:** NEON
- **RAM:** 8.0 GB
- **Accelerator:** Apple Metal (Metal built into the release binary)
- **llama.cpp asset đã tải:** llama-b10488-bin-macos-arm64.tar.gz
- **Model đã dùng:** Qwen3.5 0.8B (`LAB_MODEL=qwen35-0.8b`)
- **Quantization:** Q4_K_M (primary) + UD-Q2_K_XL (compare)

**Chạy ở đâu:** Laptop cá nhân (MacBook Air M2 8GB)

**Setup story** (≤ 80 chữ): Lựa chọn model Qwen3.5 0.8B giúp tối ưu hóa thời gian tải và bộ nhớ RAM. Bộ binary native llama.cpp b10488 tích hợp sẵn Metal hoạt động mượt mà, không gặp lỗi tương thích.

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

> Paste bảng từ `benchmarks/01-quickstart-results.md` (`make bench` tự sinh).

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| Q4_K_M | 0.50 | 2099 | 79 / 261 | 13.6 / 16.1 | 906 / 1027 / 1027 | 73.4 |
| UD-Q2_K_XL | 0.39 | 2062 | 74 / 76 | 12.0 / 12.8 | 828 / 878 / 878 | 83.5 |

**Quan sát** (≤ 60 chữ): Bản 2-bit decode nhanh hơn 1.14× (83.5 vs 73.4 tok/s) nhờ giảm 22% băng thông bộ nhớ. Tuy nhiên chất lượng câu trả lời bị suy giảm ngữ nghĩa; trên máy 8GB RAM, bản Q4_K_M vẫn là lựa chọn cân bằng tối ưu.

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

> Từ `benchmarks/02-server-results.md` (`make load-report`).

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 1.75 | 4500 | 6800 | 8100 | 8.1 | 0.0% |
| 50 | 1.45 | 29000 | 35000 | 36000 | 34.7 | 0.0% |

- **Offered load tăng 5×, throughput thực tăng:** 0.83×
- **P95 tăng:** 5.15×
- **Effective concurrency ở 50 users:** 34.7 so với `--parallel` = 4 slots

**Peak `llamacpp:n_busy_slots_per_decode`** (từ `make metrics` khi `make load-50` đang chạy): 3.97 / 4 slots

**Saturation reading** (≤ 80 chữ): Server bão hoà ở 50 users vì Throughput đi ngang (1.45 RPS) trong khi P95 tăng vọt 5.15× do Queue time (Effective concurrency 34.7 >> 4 slots). Để nâng Goodput@SLO, cần tăng `--parallel` và áp dụng backpressure reject request khi hàng đợi vượt ngưỡng SLO.

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

> Từ `make pipeline`. Nói thật cái nào real, cái nào stub — stub **không** mất điểm.

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | Local macOS | stub |
| N17 Data pipeline | In-memory docs | stub |
| N18 Lakehouse | In-memory | stub |
| N19 Vector + features | Keyword overlap | stub |
| N20 Serving | `llama-server` | real |

**Latency split** (mean của 3 query, từ output của `pipeline.py`):

- embed: 0.0 ms
- retrieve: 0.0 ms
- llm: 1846.1 ms
- **stage chiếm nhiều nhất:** llm (100% của total)

**Reflection** (≤ 60 chữ): LLM Generation chiếm trọn 100% latency. Muốn giảm 2× độ trễ, cần tối ưu khâu Decode của LLM bằng thread tuning, Prompt Caching và giới hạn max_tokens.

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

> **Phần quan trọng nhất của report.** Không cần bonus track: `make tune` đã cho bạn
> một before/after thật (`benchmarks/01-tuning-tg128.md`). Đổi quantization,
> `LAB_N_CTX`, hay `--parallel` rồi đo lại cũng được.

**Change:** Điều chỉnh số thread `-t` từ mặc định 8 xuống 4 threads (khớp số Performance Cores trên Apple M2)

```
before:  75.2 tok/s
after:   89.7 tok/s
speedup: 1.19x
```

*(So với cấu hình oversubscription `-t 16` ở 59.6 tok/s, mức speedup lên tới **1.50×**)*

**Tại sao nó work** (1–2 đoạn — đây là phần grader đọc kỹ nhất):

Kết quả ban đầu có thể gây bất ngờ nếu giả định rằng tăng số luồng lên tối đa số core (8 cores) sẽ giúp tính toán nhanh hơn. Tuy nhiên, chip Apple M2 có kiến trúc bất đối xứng gồm **4 nhân hiệu năng cao (P-cores) và 4 nhân tiết kiệm điện (E-cores)**. 

Giai đoạn sinh token (Decode phase) là tác vụ cực kỳ nhạy cảm với băng thông bộ nhớ (memory bandwidth) và đồng bộ luồng (thread synchronization). Khi chạy `-t 4`, llama.cpp gán toàn bộ 4 thread vào 4 nhân P-cores có xung nhịp cao và bộ nhớ đệm L2 độc lập rộng lớn. Khi tăng lên `-t 8`, việc kéo thêm 4 nhân E-cores có IPC thấp hơn khiến các P-cores liên tục phải dừng chờ (barrier synchronization latency) ở mỗi bước decode ma trận. Khi đẩy lên `-t 16`, overhead do chuyển ngữ cảnh (context switching) và tranh chấp cache L1/L2 làm tốc độ sụt giảm nghiêm trọng xuống chỉ còn 59.6 tok/s. Do đó, `-t 4` chính là điểm ngọt (sweet spot) hoàn hảo nhất cho phần cứng máy.

---

## 6. Bonus  *(optional — tối đa 20 điểm)*

> Bỏ trống nếu không làm. Xem `bonus/README.md`. Đừng làm hết — **một** finding sâu
> ăn điểm hơn năm bảng nông.

**Đã làm:** B2 (Context-length sweep `make sweep-ctx` đo chi phí Prefill / TTFT) & B5/C8 (Semantic Cache threshold analysis `make semantic-cache-offline`)

**Numbers:**

```
before:  163.9 ms (TTFT ở 256 tokens context)
after:   2672.9 ms (TTFT ở 4096 tokens context)
scaling: 1.02x linear scaling (tương đương ~650 ms thêm cho mỗi 1000 tokens context)
```

**Điều này nói lên gì mà deck chưa nói:**

1. **Prefill scaling trong thực tế**: Lý thuyết Attention là $O(N^2)$, nhưng ở dải context phổ thông (256 đến 4,096 tokens) trên model cỡ nhỏ, các lớp Linear Projection và MLP $O(N)$ vẫn chiếm ưu thế áp đảo. Do đó đường chi phí TTFT tăng gần như tuyến tính ($1.02\times$ linear), đạt 2.67s ở 4k tokens.
2. **Chi phí ẩn của RAG context stuffing**: Nhồi nhét thêm 1,000 tokens vào prompt RAG đồng nghĩa với việc người dùng phải chờ thêm 650 ms trước khi token đầu tiên xuất hiện. Vì vậy, trong kiến trúc RAG thực tế, việc kết hợp **Semantic Cache** ở tầng ngoài cùng (đạt hit rate 38% trong bài test C8 giúp giảm 100% compute) và **Prefix Caching** ở tầng LLM là bắt buộc để duy trì TTFT thấp dưới SLO.

---

## 7. Điều làm bạn ngạc nhiên nhất  *(optional)*

Điểm thú vị nhất là cơ chế Continuous Batching của `llama-server` hoạt động cực kỳ hiệu quả khi duy trì mức utilisation 3.97 / 4 slots (99.2%) dưới tải cao, giúp hệ thống không bị crash mà xử lý hàng đợi mượt mà.

---

## 8. Self-check trước khi push

- [x] `hardware.json` committed
- [x] `models/active.json` committed
- [x] `benchmarks/01-quickstart-results.md` committed (`make bench`)
- [x] `benchmarks/01-tuning-tg128.md` committed (`make tune`)
- [x] `benchmarks/02-server-results.md` committed (`make load-report`)
- [x] `benchmarks/02-server-batching-u50.md` hoặc `-metrics-u50.csv` committed (`make metrics`)
- [x] `benchmarks/locust-10_stats.csv` + `locust-50_stats.csv` committed (`make load-10` / `load-50`)
- [x] `benchmarks/03-integration-results.md` committed (`make pipeline`)
- [x] Mọi section **"required — replace this line"** trong các file `benchmarks/*.md`
      đã được thay bằng nhận xét của bạn
- [x] 5 screenshots trong `submission/screenshots/`
- [x] `make verify` → **exit 0**
- [ ] Repo GitHub ở chế độ **public**
- [ ] Đã paste public URL vào VinUni LMS
- [x] **Không** commit `models/*.gguf` hay `runtime/` (đã có trong `.gitignore`)

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Private → grader không
xem được → 0 điểm.
