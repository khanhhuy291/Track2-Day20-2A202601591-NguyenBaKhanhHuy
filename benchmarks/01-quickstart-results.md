# 01 - Measure: latency baseline

Model `Qwen3.5 0.8B` · host `Darwin-arm64` · llama.cpp `b10488`
Settings: `threads=8` `ngl=99` `ctx=2048`
`max_tokens=64` · warm-up discarded
Completed requests: `Q4_K_M` 10/10 · `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| Q4_K_M | 0.50 | 2099 | 79 / 261 | 13.6 / 16.1 | 906 / 1027 / 1027 | 73.4 |
| UD-Q2_K_XL | 0.39 | 2062 | 74 / 76 | 12.0 / 12.8 | 828 / 878 / 878 | 83.5 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` decodes **1.14x faster** than `Q4_K_M` here, for 0.11 GB less on disk.

## Observation

Bản 2-bit `UD-Q2_K_XL` (0.39 GB) mang lại tốc độ decode nhanh hơn khoảng **1.14×** so với bản 4-bit `Q4_K_M` (0.50 GB), cụ thể là 83.5 tok/s (TPOT 12.0 ms) so với 73.4 tok/s (TPOT 13.6 ms). TTFT P50 cũng giảm nhẹ từ 79 ms xuống 74 ms. Điều này hoàn toàn phù hợp với lý thuyết Decode bound bởi memory bandwidth: mô hình 2-bit giảm ~22% dung lượng cần đọc từ RAM mỗi token decode. Tuy nhiên, với model nhỏ 0.8B, bản 2-bit bị suy giảm độ mạch lạc và suy luận ngữ nghĩa đáng kể; với tài nguyên RAM 8.0 GB của máy Apple M2, bản `Q4_K_M` vẫn là lựa chọn cân bằng tối ưu hơn nhiều giữa chất lượng phản hồi và tốc độ.
