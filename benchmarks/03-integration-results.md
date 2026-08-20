# 03 - Integrate: RAG pipeline run

Host `Darwin-arm64` · llama.cpp `b10488` ·
retrieval backend: **keyword overlap** · 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.0 | 2020.1 | 2020.3 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.0 | 1247.7 | 1247.8 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.0 | 2270.4 | 2270.5 |

Mean per stage (ms): embed **0.0** · retrieve **0.0** ·
llm **1846.1** · total **1846.2**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Based on the provided context, **Goodput** is more useful than raw throughput because it focuses on **SLOs (Service Level Objectives)** by only counting requests that met the Target Time-to-Fill (TTFT) and Target Time-to-Poll (TPOT) targets.

Raw throughput ignores SLOs, whereas Goodput specifically counts only the requests per second that met these targets. This makes Goodput more useful for moni

**What problem does PagedAttention actually solve?**

> PagedAttention solves the problem of **internal fragmentation in GPU memory**.

Specifically, it addresses the issue where the KV cache is stored in non-contiguous pages, which causes wasted memory space. By storing the cache in non-contiguous pages, PagedAttention removes this internal fragmentation, allowing for more efficient use of GPU memory.

**When does splitting prefill and decode help?**

> Based on the provided context, splitting prefill and decode helps when **prefill is compute-bound and decode is memory-bandwidth-bound**.

The context states that prefilling the model requires significant computation (likely GPU memory), while decoding requires significant memory bandwidth. By splitting these operations into separate pools (prefill and decode), the system can utilize different har


## Pipeline Analysis & Real/Stub Declaration

- **N16 (Cloud/IaC)**: Stub (chạy local trên Darwin-arm64)
- **N17 (Data pipeline)**: Stub
- **N18 (Lakehouse)**: Stub
- **N19 (Vector + features)**: Stub (sử dụng in-memory keyword overlap fallback thay vì external vector database)
- **N20 (Model Serving)**: **Real** (`llama-server` chạy local trên port 8080)

**Phân tích độ trễ:**
Giai đoạn `llm` chiếm trọn **100%** thời gian tổng thể (~1,846 ms / 1,846 ms), trong đó phần lớn độ trễ đến từ phase Decode các token câu trả lời. Điều này hoàn toàn khớp với kỳ vọng vì phần retrieve in-memory chỉ tốn < 0.1 ms.
Để giảm độ trễ của pipeline đi 2×:
1. **Tấn công vào LLM Decode**: Chạy mô hình với số thread tối ưu (`LAB_N_THREADS=4`), bật Prompt Caching (để prefix context trong prompt RAG không phải prefill lại), hoặc áp dụng Speculative Decoding (MTP head).
2. **Giới hạn Output Length / Early Stopping**: Điều chỉnh `max_tokens` ngắn gọn phù hợp với câu trả lời cần thiết.
