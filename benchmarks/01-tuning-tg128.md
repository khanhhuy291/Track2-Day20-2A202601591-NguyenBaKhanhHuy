# 01 - Tune: thread-count sweep

Model `Qwen3.5-0.8B-Q4_K_M.gguf` · host `Darwin-arm64` · llama.cpp `b10488`
CPU: **8 physical · 8 logical** cores · `ngl=99` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 82.7 | 92% |
| 4 | 89.7 | 100% |
| 8 | 75.2 | 84% |
| 16 | 59.6 | 66% |

**Best**: `-t 4` at 89.7 tok/s
**Slowest tested**: `-t 16` at 59.6 tok/s (1.50x spread)
**Against the physical-core default** (`-t 8`, 75.2 tok/s): 1.19x

Use this in your run:

```bash
LAB_N_THREADS=4 make bench
```

## Explanation

Điểm tối ưu (knee / peak) đạt được chính xác tại `-t 4` với 89.7 tok/s (nhanh hơn 1.19× so với mặc định `-t 8` là 75.2 tok/s). Nguyên nhân nằm ở kiến trúc vi xử lý Apple M2 gồm **4 nhân hiệu năng cao (P-cores) và 4 nhân tiết kiệm điện (E-cores)**:
1. Khi chạy `-t 4`, llama.cpp điều phối vừa vặn trên 4 nhân P-cores với xung nhịp cao nhất và băng thông L2 cache độc lập lớn nhất.
2. Khi tăng lên `-t 8`, việc kéo thêm 4 nhân E-cores có IPC và xung nhịp thấp hơn vào quá trình tính toán gây ra hiện tượng barrier synchronization skew (các thread nhanh trên P-core phải chờ thread chậm trên E-core hoàn thành step decode), đồng thời gây nghẽn băng thông memory bus chia sẻ giữa 2 cluster.
3. Khi đẩy lên `-t 16`, tốc độ giảm nghiêm trọng còn 59.6 tok/s (giảm 34% so với peak) do oversubscription phần cứng, overhead context switching liên tục giữa các luồng và cache thrashing.
