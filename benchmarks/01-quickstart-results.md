# 01 - Measure: latency baseline

Model `Gemma 4 E2B` · host `Windows-AMD64` · llama.cpp `b10488`
Settings: `threads=4` `ngl=99` `ctx=2048`
`max_tokens=64` · warm-up discarded
Completed requests: `UD-Q4_K_XL` 10/10 · `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 19861 | 402 / 1293 | 19.9 / 21.3 | 1644 / 2535 / 2535 | 50.3 |
| UD-Q2_K_XL | 2.24 | 8748 | 461 / 3928 | 23.2 / 24.5 | 1915 / 5404 / 5404 | 43.2 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` decodes **1.16x SLOWER** than `UD-Q4_K_XL` here, despite being 0.73 GB smaller. That is a real result, not a mistake: fewer bits only buys speed when decode is limited by memory bandwidth. On a machine that is compute-limited instead — few cores, no GPU offload — the extra dequantization work of a heavily-quantized format can cost more than the bytes it saves. Say which case yours is.

## My observation

`UD-Q2_K_XL` nhỏ hơn `UD-Q4_K_XL` 0.73 GB, tương đương khoảng 25%, nhưng decode chỉ đạt 43.2 tok/s so với 50.3 tok/s của Q4; Q4 nhanh hơn khoảng 1.16×. Tôi đã hỏi cả hai model cùng một prompt về continuous batching với `temperature=0` và `max_tokens=128`. Cả hai đều làm đúng yêu cầu gồm ba ý và một hạn chế, đồng thời đưa ra nội dung tương đương; Q2 chỉ dài hơn (91 so với 77 completion token), nên chưa thấy suy giảm chất lượng rõ trong mẫu này. Với RAM 15.8 GB, tôi chọn Q4 vì nhanh hơn và vẫn vừa bộ nhớ; Q2 chỉ đáng cân nhắc khi ưu tiên dung lượng.
