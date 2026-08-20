# 01 - Tune: thread-count sweep

Model `gemma-4-E2B-it-UD-Q4_K_XL.gguf` · host `Windows-AMD64` · llama.cpp `b10488`
CPU: **8 physical · 16 logical** cores · `ngl=99` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 104.7 | 97% |
| 4 | 107.5 | 99% |
| 8 | 107.8 | 100% |
| 16 | 108.2 | 100% |
| 32 | 108.1 | 100% |

**Best**: `-t 16` at 108.2 tok/s
**Slowest tested**: `-t 1` at 104.7 tok/s (1.03x spread)
**Against the physical-core default** (`-t 8`, 107.8 tok/s): 1.00x

Use this in your run:

```bash
LAB_N_THREADS=16 make bench
```

## Your explanation

There is effectively **no knee**. The curve is flat: `-t 1` already gives 104.7 tok/s and
`-t 32` gives 108.1 tok/s — a 3% spread across the whole range, and it never drops when I
oversubscribe past 8 physical cores. This contradicts the expected CPU shape (peak at
physical-core count, decline above it).

The reason is that I run with `ngl=99`, so every decode layer lives on the RTX 3060, not
the CPU. tg128 decode is autoregressive and bound by GPU VRAM bandwidth, executed by CUDA
kernels. The llama.cpp `-t` value only sizes the host-side thread pool (sampling, dispatch,
a little prefill orchestration), which is never the bottleneck once layers are offloaded.
So adding threads changes almost nothing — the flat curve is exactly what a GPU-bound
decode should look like. On this machine the lever that would move throughput is at the GPU
tier (batch width / slots, KV cache, quant), not `-t`.
