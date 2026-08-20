# 02 - Serve: load test + saturation reading

Host `Darwin-arm64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=8` ·
`ngl=99`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 98 | 1.75 | 4500 | 6800 | 8100 | 8.1 | 0.0% |
| 50 | 86 | 1.45 | 29000 | 35000 | 36000 | 34.7 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **0.83x** (17% of linear) |
| P95 latency | **5.15x** |
| Effective concurrency at 50 users | 34.7 vs `--parallel 4` slots (occupancy/slot ratio 8.67) |

**Saturated.** Throughput delivered only 0.83x for 5x the offered load, and effective concurrency (34.7) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

Throughput moved 0.83x while P95 moved 5.15x. That gap is the goodput argument: past saturation you buy throughput by spending latency, and if your SLO is a P95 target then the requests you added are no longer being served within it. (This lab does not fix an SLO number for you -- pick one in your write-up and state how much goodput you keep at it.)

## Reading

Server bão hoà rõ rệt khi tăng tải từ 10 lên 50 users (offered load tăng 5×):
1. **Bằng chứng bão hoà**: Throughput thực tế không tăng mà hơi giảm nhẹ (1.75 RPS $\rightarrow$ 1.45 RPS, tương đương **0.83×**), trong khi P95 latency tăng vọt **5.15×** (từ 6,800 ms lên 35,000 ms). Effective Concurrency đạt **34.7**, vượt xa sức chứa 4 slot của hệ thống (tỉ lệ occupancy/slot = 8.67).
2. **Bản chất của Latency tăng**: Toàn bộ phần chênh lệch thời gian (~28 giây ở P95) là **Queue Time** (chờ lượt giải phóng slot trong scheduler) chứ không phải Compute Time, vì năng lực tính toán phần cứng đã bão hoà 100%.
3. **Knob ưu tiên để nâng Goodput@SLO**: Nếu đặt SLO là P95 ≤ 8,000 ms, để nâng Goodput ta cần tăng `--parallel` (ví dụ từ 4 lên 6 hoặc 8 slots) kết hợp với tối ưu số thread `LAB_N_THREADS=4` (đã chứng minh ở bài test tune tăng 1.19× tốc độ decode trên 4 nhân P-cores của M2), đồng thời áp dụng cơ chế Early Rejection / Backpressure khi hàng đợi vượt quá ngưỡng SLO thay vì nhận request vô hạn khiến P95 suy thoái.
