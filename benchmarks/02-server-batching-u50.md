# 02 - Continuous batching under load (u50)

Host `Windows-AMD64` · `--parallel 4` · 13 samples over
60s at 2.0s intervals · raw CSV: `02-server-metrics-u50.csv`

| Gauge | Peak observed |
|:--|--:|
| `n_busy_slots_per_decode` (avg/decode) | 3.80 of 4 slots (95%) |
| `requests_processing` | 4 |
| `requests_deferred` | 46 |
| `kv_cache_usage_ratio` | n/a — not exported by llama.cpp `b10488` |
| `tokens_predicted_total` (final) | 4909 |

Highest sampled value was **3.80 of 4** slots. Note this gauge is llama.cpp's *average* busy slots per decode step, so the number below is the highest average we sampled, not an instantaneous maximum batch width. A peak near 1 means
requests were served one at a time -- either the load was too light to overlap, or
they arrived too far apart. A peak approaching `--parallel` means the scheduler was
genuinely packing concurrent requests into shared decode steps.
`requests_deferred` went above zero: more requests arrived than there were slots, so some waited. That wait is the queue time in your P95.

## Your observation

- **Peak batch width vs effective concurrency:** The peak sampled batch width from internal server telemetry (`n_busy_slots_per_decode`) was **3.80 of 4 slots (95%)**, whereas the client-side effective concurrency calculated from Little's Law in `02-server-results.md` was **23.0**.
- **Why they differ:** They measure fundamentally different boundaries:
  - `n_busy_slots_per_decode` measures **actual active decode slots** executing concurrently in hardware, which is physically bounded by `--parallel 4`.
  - Little's Law ($L = \lambda \cdot W = 23.0$) measures **total in-system concurrency**, capturing both actively processing requests and queued requests. The discrepancy is fully accounted for by `requests_deferred = 46`, indicating ~19 concurrent requests were stuck waiting in queue rather than executing.
- **Trust evaluation:** Both metrics are valid for their specific purpose. For evaluating **continuous batching efficiency**, we trust `n_busy_slots_per_decode` (3.80/4), proving that the scheduler successfully saturated 95% of available batch capacity. For understanding **user-perceived latency degradation**, Little's Law and `requests_deferred` reveal that queueing delay—not decode compute time—dominated client P95 latency.
