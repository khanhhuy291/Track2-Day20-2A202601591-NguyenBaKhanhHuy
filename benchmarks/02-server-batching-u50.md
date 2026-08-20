# 02 - Continuous batching under load (u50)

Host `Darwin-arm64` · `--parallel 4` · 30 samples over
60s at 2.0s intervals · raw CSV: `02-server-metrics-u50.csv`

| Gauge | Peak observed |
|:--|--:|
| `n_busy_slots_per_decode` (avg/decode) | 3.97 of 4 slots (99%) |
| `requests_processing` | 4 |
| `requests_deferred` | 46 |
| `kv_cache_usage_ratio` | n/a — not exported by llama.cpp `b10488` |
| `tokens_predicted_total` (final) | 17305 |

Highest sampled value was **3.97 of 4** slots. Note this gauge is llama.cpp's *average* busy slots per decode step, so the number below is the highest average we sampled, not an instantaneous maximum batch width. A peak near 1 means
requests were served one at a time -- either the load was too light to overlap, or
they arrived too far apart. A peak approaching `--parallel` means the scheduler was
genuinely packing concurrent requests into shared decode steps.
`requests_deferred` went above zero: more requests arrived than there were slots, so some waited. That wait is the queue time in your P95.

## Observation

Peak batch width đo được là **3.97 / 4 slots** (đạt 99.2% capacity tối đa của scheduler `--parallel 4`), và số lượng `requests_processing` luôn duy trì ở mức tối đa là 4 trong khi `requests_deferred` (hàng đợi) lên tới 46. Con số 3.97 phản ánh đúng năng lực tính toán thực tế tại mỗi decode step (slot utilisation), trong khi Effective Concurrency theo Little's Law trong `02-server-results.md` là **34.7** (bao gồm cả 4 request đang tính toán và ~30 request đang xếp hàng chờ slot). Cả hai số liệu hoàn toàn nhất quán: server đã khai thác triệt để continuous batching trên toàn bộ các slot được cấp phát, và toàn bộ áp lực tải vượt quá 4 slot đều được đẩy vào hàng đợi chờ xử lý.
