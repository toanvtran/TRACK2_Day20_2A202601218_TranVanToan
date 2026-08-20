# 02 - Continuous batching under load (u50)

Host `Windows-AMD64` · `--parallel 4` · 15 samples over
60s at 2.0s intervals · raw CSV: `02-server-metrics-u50.csv`

| Gauge | Peak observed |
|:--|--:|
| `n_busy_slots_per_decode` (avg/decode) | 3.97 of 4 slots (99%) |
| `requests_processing` | 4 |
| `requests_deferred` | 45 |
| `kv_cache_usage_ratio` | n/a — not exported by llama.cpp `b10488` |
| `tokens_predicted_total` (final) | 27187 |

Highest sampled value was **3.97 of 4** slots. Note this gauge is llama.cpp's *average* busy slots per decode step, so the number below is the highest average we sampled, not an instantaneous maximum batch width. A peak near 1 means
requests were served one at a time -- either the load was too light to overlap, or
they arrived too far apart. A peak approaching `--parallel` means the scheduler was
genuinely packing concurrent requests into shared decode steps.
`requests_deferred` went above zero: more requests arrived than there were slots, so some waited. That wait is the queue time in your P95.

## Your observation

Peak batch width was **3.97 of 4 slots** — the scheduler genuinely packed almost 4
concurrent requests into shared decode steps, proving continuous batching worked. This does
**not** match the effective concurrency of 41.1 from `02-server-results.md`, and that is
expected: the two measure different things. `n_busy_slots_per_decode` is a server-side gauge
of actual decode slot occupancy, hard-capped at `--parallel = 4`. Effective concurrency is
Little's Law (RPS × avg latency) and counts *queued* requests too, so it can far exceed the
slot count — 41.1 vs 4 means about 37 requests were waiting in the queue at any moment.

I trust **3.97/4** as the measure of real parallel compute, and I trust **41.1** as the
measure of how overloaded the queue was. Together they tell the whole story: 4 slots busy,
everything else deferred — the 41.1 − 3.97 gap is the queue time showing up in P95.
