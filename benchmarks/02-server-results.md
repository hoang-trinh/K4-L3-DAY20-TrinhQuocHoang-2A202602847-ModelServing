# 02 - Serve: load test + saturation reading

Host `Windows-AMD64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=8` ·
`ngl=99`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 36 | 0.63 | 13000 | 21000 | 22000 | 8.5 | 0.0% |
| 50 | 42 | 0.74 | 35000 | 55000 | 56000 | 23.0 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **1.17x** (23% of linear) |
| P95 latency | **2.62x** |
| Effective concurrency at 50 users | 23.0 vs `--parallel 4` slots (occupancy/slot ratio 5.74) |

**Saturated.** Throughput delivered only 1.17x for 5x the offered load, and effective concurrency (23.0) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

Throughput moved 1.17x while P95 moved 2.62x. That gap is the goodput argument: past saturation you buy throughput by spending latency, and if your SLO is a P95 target then the requests you added are no longer being served within it. (This lab does not fix an SLO number for you -- pick one in your write-up and state how much goodput you keep at it.)

## Your reading

- **Saturation point and evidence:** The server is already deeply saturated before 50 users. The key numbers that prove saturation are:
  - **Sub-linear throughput:** A **5x increase in offered load** (10 to 50 simulated users) only yielded a **1.17x increase in throughput** (0.63 -> 0.74 RPS), achieving merely 23% of linear scaling.
  - **Severe latency degradation:** P95 latency exploded from 21s to 55s (**2.62x increase**), indicating that additional traffic became unserviceable queue delay.
  - **Effective concurrency (23.0 vs 4 slots):** An occupancy-to-slot ratio of 5.74x proves that requests spent the majority of their lifecycle waiting in line, directly corroborated by `requests_deferred = 46` during peak load.
- **Goodput under SLO:** Assuming an SLO target of $P95 \le 25\text{s}$, the system maintained healthy goodput at 10 users ($P95 = 21\text{s}$), but completely collapsed at 50 users ($P95 = 55\text{s}$), where virtually all requests violated the SLO.
- **Knob to raise goodput at SLO:** 
  - The first knob to adjust is **Admission Control / Max Queue Length** (dropping or fast-failing excess requests rather than letting them queue for 40+ seconds). Queued requests that exceed the 25s SLO become waste (badput).
  - Alternatively, if hardware permits, increase `--parallel` slots (e.g., from 4 to 6 or 8) along with KV cache allocation. Widening slots directly leverages continuous batching to serve more concurrent requests per decode step.
  - We would *not* increase thread count (`-t`), because our sweep in CP2 proved that threads above 8 saturate dual-channel memory bandwidth and trigger negative scaling. Queue mitigation—not compute threading—is the correct architectural lever.
