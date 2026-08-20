# Bonus - Context-length sweep (prefill cost)

Host `Darwin-arm64` · llama.cpp `b10488` ·
`threads=8` `ngl=99` · RAM 8.0 GB

| Prompt tokens | Prefill (tok/s) | TTFT contribution (ms) | vs linear scaling |
|:--|--:|--:|--:|
| 256 | 1561.6 | 163.9 | 1.00x |
| 1024 | 1616.9 | 633.3 | 0.97x |
| 2048 | 1573.2 | 1301.8 | 0.99x |
| 4096 | 1532.4 | 2672.9 | 1.02x |

At 4096 tokens, prefill costs **2673 ms**, which is
**1.02x** linear scaling -- so on this hardware, over this range, prefill is
still growing **roughly linearly**, not quadratically.

That is the correct finding, not a failed experiment. Attention is O(N^2), but it is only
one term: the per-layer linear projections and MLP are O(N), and on a 2B-class model at
short prompts they dominate. The quadratic term only overtakes them once N gets large
enough. Your prefill cost is currently bounded by throughput, not by sequence length.

To find where it *does* bend, extend the grid:

```bash
.venv/bin/python bonus/sweeps/ctx-len-sweep.py --grid 1024,4096,8192,16384,32768
```

Watch the "vs linear" column: the first row that climbs meaningfully above 1.0 is where
attention starts to matter on your machine. Report that crossover point.

Either way, this is the number to remember when someone proposes stuffing more retrieved
context into a RAG prompt "because the context window allows it". Prefill is paid in full,
on every request, before the first token appears.

## Finding

1. **Điểm Prefill chiếm ưu thế**: Ở độ dài context ngắn (256 tokens), TTFT chỉ tốn 163.9 ms (~10-15% E2E latency). Nhưng khi tăng lên 4,096 tokens, TTFT tăng lên tới **2,672.9 ms** (2.67 giây), vượt qua cả thời gian decode trung bình và chiếm hơn 70% tổng thời gian phản hồi.
2. **Tuyến tính vs Phi tuyến ($O(N)$ vs $O(N^2)$)**: Trong dải 256 đến 4,096 tokens trên Metal GPU, prefill scaling duy trì mức gần như hoàn hảo tuyến tính (**1.02× linear scaling**) nhờ các phép tính Linear Projection và MLP $O(N)$ vẫn chiếm ưu thế tuyệt đối so với quadratic attention $O(N^2)$.
3. **Bài học cho RAG Pipeline**: Không nên nhồi nhét vô tội vạ các chunk context vào prompt chỉ vì mô hình hỗ trợ context window lớn. Mỗi 1,000 tokens context bổ sung sẽ đánh đổi trực tiếp thêm ~650 ms độ trễ TTFT mà người dùng phải chờ đợi trước khi nhận được token đầu tiên. Cần giới hạn top-k (3-5 chunks liên quan nhất) hoặc bật Prefix / Prompt Caching để triệt tiêu chi phí prefill này.
