# 03 - Integrate: RAG pipeline run

Host `Windows-AMD64` · llama.cpp `b10488` ·
retrieval backend: **keyword overlap** · 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.2 | 7646.8 | 7647.1 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.1 | 4129.3 | 4129.5 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.1 | 6153.3 | 6153.5 |

Mean per stage (ms): embed **0.0** · retrieve **0.1** ·
llm **5976.5** · total **5976.7**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Based on the provided context, **Goodput** is more useful than raw throughput because it accounts for **SLOs** (Service Level Objectives).

While raw throughput measures the total requests per second (requests/sec) that pass through the system, Goodput specifically counts only the requests that met the **TTFT** (Throughput Target for Functionality) and **TPOT** (Throughput Target for Performance).

**What problem does PagedAttention actually solve?**

> PagedAttention solves the problem of **internal fragmentation in GPU memory** caused by storing KV cache in non-contiguous pages.

**When does splitting prefill and decode help?**

> Splitting prefill and decode helps when **prefill is compute-bound and decode is memory-bandwidth-bound**.

This is because the context explicitly states: "Disaggregated serving splits prefill and decode onto separate pools because prefill is compute-bound and decode is memory-bandwidth-bound." By separating these operations, the system can utilize different hardware resources (e.g., GPU for compu


## Which N16-N19 pieces are real

- **Components breakdown (Real vs Stubbed):**
  - **N16 Cloud/IaC:** Stubbed (running locally on Windows AMD64 host).
  - **N17 Data pipeline:** Stubbed (using in-memory sample corpus).
  - **N18 Lakehouse:** Stubbed (local in-memory document structures).
  - **N19 Vector + features:** Stubbed (in-memory toy keyword-overlap retrieval without dense embeddings; `embed = 0.0 ms`, `retrieve = 0.1 ms`).
  - **N20 Serving:** **Real** (`llama-server` running `Qwen3.5-0.8B-Q4_K_M.gguf` on `http://localhost:8080` with `--cont-batching` and `--parallel 4`).
- **Dominant stage analysis:** As expected, the **LLM stage** overwhelmingly dominates the pipeline, consuming **5976.5 ms** (virtually 100% of the total 5976.7 ms latency). In-memory keyword retrieval takes negligible time (0.1 ms), while autoregressive LLM decoding requires multiple sequential passes over model weights bound by memory bandwidth.
- **How to halve pipeline latency (Amdahl's Law):**
  - By Amdahl's Law, optimizing retrieval or embedding yields zero perceivable gain (even reducing 0.1 ms to 0 ms changes nothing). We must attack the **LLM generation phase**:
  - **Speculative Decoding / Draft Models:** Verify multiple draft tokens per memory read step, cutting TPOT significantly.
  - **Response token capping (`max_tokens`) & Early Stopping:** Enforce strict generation limits to prevent verbosity.
  - **Prefix / KV Caching:** Reusing KV states for repeated system prompts and common contexts to shrink prefill overhead.
