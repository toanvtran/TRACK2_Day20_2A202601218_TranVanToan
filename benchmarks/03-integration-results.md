# 03 - Integrate: RAG pipeline run

Host `Windows-AMD64` · llama.cpp `b10488` ·
retrieval backend: **keyword overlap** · 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.0 | 2734.8 | 2734.9 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.0 | 2479.2 | 2479.3 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.0 | 2523.5 | 2523.6 |

Mean per stage (ms): embed **0.0** · retrieve **0.0** ·
llm **2579.2** · total **2579.3**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Goodput@SLO counts only the requests per second that met the TTFT and TPOT targets. Throughput at saturation ignores SLOs.

**What problem does PagedAttention actually solve?**

> PagedAttention stores the KV cache in non-contiguous pages, removing the internal fragmentation that wasted most GPU memory.

**When does splitting prefill and decode help?**

> Splitting prefill and decode helps because prefill is compute-bound and decode is memory-bandwidth-bound.


## Which N16-N19 pieces are real

- **N16 Cloud/IaC** — stubbed. Everything runs on my local laptop, no provisioned cloud infra.
- **N17 Data pipeline** — stubbed. No ingestion pipeline is wired in.
- **N18 Lakehouse** — stubbed. The corpus is the 6-doc `TOY_DOCS` list held in memory, not a real lakehouse table.
- **N19 Vector + features** — stubbed. Retrieval is keyword overlap; no embedding server is running, so no real vector search.
- **N20 Serving** — real. `llama-server` (Gemma 4 E2B, CUDA) serves the completions.

The dominant stage is **llm at 100%** (2579 ms mean; embed and retrieve both ≈0 ms). This
is exactly what I expected: with only 6 in-memory docs and keyword matching, retrieval is
free, so all latency sits in decode. To halve pipeline latency I would attack the **llm
stage**, since it is 100% of the budget — enable prompt/prefix caching so the shared system
prompt and context are not re-prefilled every query, cap `max_tokens`, and raise
`--parallel` if queries run concurrently. Optimizing retrieve or embed would be pointless
here; they are already near zero.
