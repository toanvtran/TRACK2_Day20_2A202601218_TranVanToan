# 02 - Serve: load test + saturation reading

Host `Windows-AMD64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=8` ·
`ngl=99`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 195 | 3.29 | 2000 | 3400 | 4000 | 7.0 | 0.0% |
| 50 | 197 | 3.32 | 13000 | 15000 | 16000 | 41.1 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **1.01x** (20% of linear) |
| P95 latency | **4.41x** |
| Effective concurrency at 50 users | 41.1 vs `--parallel 4` slots (occupancy/slot ratio 10.27) |

**Saturated.** Throughput delivered only 1.01x for 5x the offered load, and effective concurrency (41.1) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

Throughput moved 1.01x while P95 moved 4.41x. That gap is the goodput argument: past saturation you buy throughput by spending latency, and if your SLO is a P95 target then the requests you added are no longer being served within it. (This lab does not fix an SLO number for you -- pick one in your write-up and state how much goodput you keep at it.)

## Your reading

The server saturates at or just below 50 concurrent users. The number that convinced me is
**throughput 1.01x for 5x offered load** — going from 10 to 50 users added essentially no
delivered RPS (3.29 → 3.32) while P95 blew up 4.41x (3400 → 15000 ms). That gap is pure
queue time, not compute: `make metrics` during load-50 showed `n_busy_slots_per_decode`
peaking at 3.97 of 4 slots (all decode slots full) with `requests_deferred` climbing to 45,
so extra requests waited in line rather than being served faster.

Picking an SLO of P95 ≤ 4 s: I keep it at 10 users (P95 3.4 s) but blow past it at 50
(P95 15 s), so goodput collapses beyond ~10 users. To raise goodput I would increase
`--parallel` (more decode slots) **first**, because the bottleneck is slot count — the
scheduler is packing 3.97/4 slots and deferring the rest, so more slots directly convert
queued requests into concurrent decode. Only if VRAM/KV cache runs out would I then fall
back to shrinking `ctx` or dropping to a smaller quant to free room for more slots.
