# 01 - Measure: latency baseline

Model `Qwen3.5 0.8B` · host `Windows-AMD64` · llama.cpp `b10488`
Settings: `threads=8` `ngl=99` `ctx=2048`
`max_tokens=64` · warm-up discarded
Completed requests: `Q4_K_M` 10/10 · `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| Q4_K_M | 0.50 | 10358 | 2271 / 2407 | 21.9 / 22.3 | 3618 / 3811 / 3811 | 45.7 |
| UD-Q2_K_XL | 0.39 | 8189 | 2263 / 2367 | 21.7 / 22.1 | 3647 / 3742 / 3742 | 46.0 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` and `Q4_K_M` decode within 2% of each other here, for 0.11 GB difference on disk.

## Your observation

- **Quantitative benchmark:** Both quantizations achieve virtually identical decode throughput (45.7 tok/s for `Q4_K_M` vs 46.0 tok/s for `UD-Q2_K_XL`, a difference < 1%) and comparable TTFT (2271 ms vs 2263 ms P50). The 2-bit ultra-quantized model saves only 0.11 GB of memory/disk space (0.39 GB vs 0.50 GB).
- **Qualitative serving comparison:** When serving both models with the same prompt (*"Explain why the sky is blue in 2 sentences."*):
  - **`Q4_K_M` (4-bit, port 8080):** Followed the instruction accurately, generated 57 tokens, and terminated cleanly with a stop token (`finish_reason: stop`, `truncated = 0`) in 2.01s total latency.
  - **`UD-Q2_K_XL` (2-bit, port 8090):** Suffered severe degradation in instruction adherence and failed to predict the end-of-sequence token (`<|im_end|>`). It looped continuously until reaching the 512-token context window limit (`completion_tokens: 488`, `finish_reason: length`, `truncated = 1`), causing latency to explode to nearly 10 seconds.
- **Verdict:** For small models like `Qwen3.5 0.8B`, the 2-bit quantization is **not worth it**. The minuscule 0.11 GB savings in memory does not justify the collapse in instruction compliance, which in practice makes responses both incoherent and substantially slower due to runaway generation.
