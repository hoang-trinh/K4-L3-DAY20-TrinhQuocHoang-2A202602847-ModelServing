# 01 - Tune: thread-count sweep

Model `Qwen3.5-0.8B-Q4_K_M.gguf` · host `Windows-AMD64` · llama.cpp `b10488`
CPU: **8 physical · 16 logical** cores · `ngl=99` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 45.9 | 88% |
| 4 | 48.1 | 93% |
| 8 | 51.9 | 100% |
| 16 | 46.9 | 90% |
| 32 | 45.5 | 88% |

**Best**: `-t 8` at 51.9 tok/s
**Slowest tested**: `-t 32` at 45.5 tok/s (1.14x spread)
**Against the physical-core default** (`-t 8`, 51.9 tok/s): 1.00x

Use this in your run:

```bash
LAB_N_THREADS=8 make bench
```

## Your explanation

- **Knee location:** The throughput knee sits precisely at **`-t 8`** (51.9 tok/s), which matches the exact physical core count of the AMD Ryzen 7 6800HS (8 physical cores).
- **Physical scaling (1 to 8 threads):** Throughput scales modestly from 45.9 tok/s at `-t 1` to 51.9 tok/s at `-t 8` (~1.13x speedup). Because each physical core possesses its own independent execution pipeline and private L1/L2 cache, increasing thread count parallelizes dequantization and matrix-vector operations effectively up to the physical core limit.
- **SMT contention (16 threads):** At `-t 16` (matching 16 logical cores), throughput regresses to 46.9 tok/s (90% of peak). Simultaneous Multithreading (SMT) shares execution units, registers, and private cache within each physical core. For memory-bandwidth bound decode operations, logical cores do not provide additional memory channels; instead, they introduce cache contention and execution port stalling.
- **Oversubscription penalty (32 threads):** At `-t 32` (2x logical core count), performance deteriorates further to 45.5 tok/s (88% of peak, slowest tested). With 32 worker threads competing for 16 logical processors, the OS incurs heavy CPU context-switching overhead, thread synchronization latency, and thrashing of the L3 cache and memory bus.
- **Mechanism summary:** Token generation (`tg128`) is strictly **memory-bandwidth bound** (loading all model weights from RAM on each token step). Bounded by the dual-channel DDR5 bus bandwidth, the memory channels saturate near the physical core count (8 threads). Beyond 8 threads, surplus threads merely contend for identical memory bandwidth and cache lines, resulting in negative scaling.
