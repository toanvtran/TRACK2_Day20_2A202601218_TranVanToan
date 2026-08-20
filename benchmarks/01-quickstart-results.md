# 01 - Measure: latency baseline

Model `Gemma 4 E2B` · host `Windows-AMD64` · llama.cpp `b10488`
Settings: `threads=8` `ngl=99` `ctx=2048`
`max_tokens=64` · warm-up discarded
Completed requests: `UD-Q4_K_XL` 10/10 · `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 27357 | 70 / 415 | 11.8 / 12.4 | 808 / 1150 / 1150 | 85.0 |
| UD-Q2_K_XL | 2.24 | 4663 | 66 / 224 | 11.8 / 12.1 | 803 / 958 / 958 | 84.4 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` and `UD-Q4_K_XL` decode within 2% of each other here, for 0.73 GB difference on disk.

## Your observation

On this machine (RTX 3060, `ngl=99`), UD-Q2_K_XL decodes at 84.4 tok/s versus 85.0 tok/s
for UD-Q4_K_XL — under 1% apart, with TPOT P50 identical at 11.8 ms. The 2-bit build saves
0.73 GB on disk but buys no real speed, because decode is bound by VRAM memory bandwidth,
not by weight size. Asking both the same question, the 4-bit answer was coherent and
complete while the 2-bit one occasionally repeated itself or dropped detail. With enough
RAM and VRAM, the 2-bit quant is **not worth it** here: it trades quality for a size win
I don't need and gives no latency benefit.
