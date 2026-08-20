# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.
>
> `make verify` sẽ fail nếu còn placeholder chưa điền. Đó là cố ý.

**Họ Tên:** Trần Văn Toàn
**Cohort:** A20-K1
**Ngày submit:** 2026-08-20

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

> Từ `make probe`. Paste output hoặc điền tay.

- **OS:** Windows 10 (AMD64)
- **CPU:** AMD Ryzen 9 5900HS with Radeon Graphics
- **Cores:** 8 physical / 16 logical
- **CPU extensions:** AVX2 (Zen 3)
- **RAM:** 23.7 GB
- **Accelerator:** NVIDIA GeForce RTX 3060 Laptop GPU (6 GB) — CUDA backend, Vulkan cũng present
- **llama.cpp asset đã tải:** llama.cpp build `b10488` (backend CUDA, cờ `-DGGML_CUDA=ON`)
- **Model đã dùng:** Gemma 4 E2B (`LAB_MODEL=gemma4-e2b`)
- **Quantization:** UD-Q4_K_XL (primary) + UD-Q2_K_XL (compare) (từ `models/active.json`)

**Chạy ở đâu:** Laptop của tôi (RAM 23.7 GB, đủ cho model mặc định, không cần cloud fallback).

**Setup story** (≤ 80 chữ): Máy có 23.7 GB RAM và RTX 3060 nên `make setup` chọn thẳng
Gemma 4 E2B với backend CUDA (`ngl=99`), không phải đổi model. Không có bước nào fail;
llama.cpp prebuilt `b10488` nhận đúng kiến trúc `gemma4` nên không cần build lại. Chạy
các target bằng `.\lab.ps1` vì Windows không có `make`.

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

> Paste bảng từ `benchmarks/01-quickstart-results.md` (`make bench` tự sinh).

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 27357 | 70 / 415 | 11.8 / 12.4 | 808 / 1150 / 1150 | 85.0 |
| UD-Q2_K_XL | 2.24 | 4663 | 66 / 224 | 11.8 / 12.1 | 803 / 958 / 958 | 84.4 |

**Quan sát** (≤ 60 chữ): Trên GPU (ngl=99), 2-bit decode 84.4 tok/s so với 85.0 tok/s
của 4-bit — chênh <1%, TPOT gần như trùng nhau. Tiết kiệm được 0.73 GB đĩa nhưng không
nhanh hơn vì decode đã bound bởi bandwidth chứ không bởi kích thước weight. Hỏi cùng
một câu: 4-bit trả lời mạch lạc, đầy đủ; 2-bit đôi khi lặp/thiếu ý. Với máy đủ RAM+VRAM,
2-bit **không đáng** — mất chất lượng mà gần như không có lợi tốc độ.

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

> Từ `benchmarks/02-server-results.md` (`make load-report`).

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 3.29 | 2000 | 3400 | 4000 | 7.0 | 0.0% |
| 50 | 3.32 | 13000 | 15000 | 16000 | 41.1 | 0.0% |

- **Offered load tăng 5×, throughput thực tăng:** 1.01×
- **P95 tăng:** 4.41×
- **Effective concurrency ở 50 users:** 41.1 so với `--parallel` = 4 slots

**Peak `llamacpp:n_busy_slots_per_decode`** (từ `make metrics` khi `make load-50` đang
chạy): 3.97 / 4 slots (99%)

**Saturation reading** (≤ 80 chữ): Server bão hoà ngay dưới mức 50 users. Bằng chứng
mạnh nhất: offered load ×5 nhưng throughput chỉ ×1.01 trong khi P95 ×4.41 — con số 1.01×
throughput là thứ thuyết phục tôi. Phần latency thêm là **queue time**, không phải compute:
`n_busy_slots_per_decode` đã đạt 3.97/4 (slots kín) và `requests_deferred` lên tới 45, nên
request mới phải xếp hàng. Để nâng goodput@SLO tôi sẽ tăng `--parallel` (thêm decode slot)
**trước**, vì bottleneck là số slot chứ không phải tốc độ một token — chỉ khi VRAM/KV cache
hết mới chuyển sang giảm ctx hay dùng quant nhỏ hơn.

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

> Từ `make pipeline`. Nói thật cái nào real, cái nào stub — stub **không** mất điểm.

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | môi trường chạy local | stub |
| N17 Data pipeline | không nối | stub |
| N18 Lakehouse | TOY_DOCS in-memory (6 docs) | stub |
| N19 Vector + features | keyword overlap (không embedding server) | stub |
| N20 Serving | `llama-server` | real |

**Latency split** (mean của 3 query, từ output của `pipeline.py`):

- embed: 0.0 ms
- retrieve: 0.0 ms
- llm: 2579.2 ms
- **stage chiếm nhiều nhất:** llm (100% của total)

**Reflection** (≤ 60 chữ): Bottleneck nằm hoàn toàn ở llm (100%), đúng như kỳ vọng vì
retrieve chỉ là keyword overlap trên 6 doc in-memory (≈0 ms). Để giảm 2× latency tôi tấn
công stage llm: bật prompt/prefix caching để bỏ prefill lặp, giảm `max_tokens`, hoặc dùng
quant nhỏ hơn cho decode. Retrieve/embed không đáng tối ưu vì chúng gần như bằng 0.

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

> **Phần quan trọng nhất của report.** Không cần bonus track: `make tune` đã cho bạn
> một before/after thật (`benchmarks/01-tuning-tg128.md`). Đổi quantization,
> `LAB_N_CTX`, hay `--parallel` rồi đo lại cũng được.

**Change:** Sweep thread count `-t` từ 1 → 32 (`make tune`, metric tg128, ngl=99).

```
before:  -t 1   → 104.7 tok/s
after:   -t 16  → 108.2 tok/s  (best)
speedup: 1.03×  (spread toàn dải chỉ 3%)
```

**Tại sao nó work** (1–2 đoạn — đây là phần grader đọc kỹ nhất):

Đây là một kết quả **trái với kỳ vọng của deck** và chính điều đó mới đáng nói. Deck dự
đoán đường cong tg128 sẽ peak đúng ở physical-core count (8) rồi tụt khi oversubscribe
(16, 32) vì các thread tranh nhau ALU và cache. Trên máy tôi curve gần như **phẳng**:
1 thread đã đạt 104.7 tok/s và 32 thread cũng chỉ 108.1 — chênh vỏn vẹn 3%, không hề có
knee, thậm chí không tụt ở 32 thread.

Cơ chế: tôi chạy với `ngl=99` nên **toàn bộ layer decode nằm trên RTX 3060, không phải
CPU**. Decode là bước autoregressive bị chặn bởi **memory bandwidth của VRAM**, và phần
việc đó do CUDA kernel trên GPU đảm nhiệm. Thread count của llama.cpp chỉ điều phối phần
host-side (sampling, dispatch, một ít prefill) — vốn không phải bottleneck — nên tăng
thread gần như không đổi throughput. Nói cách khác, "single change that mattered most"
trên máy này **không phải** thread count: knob đó bị GPU vô hiệu hoá. Bài học ngược lại
với kỳ vọng CPU-only là: khi đã offload lên GPU thì phải tối ưu ở tầng GPU (batch/slot,
KV cache, quant) chứ chỉnh `-t` là công cốc.

---

## 6. Bonus  *(optional — tối đa 20 điểm)*

> Bỏ trống nếu không làm. Xem `bonus/README.md`. Đừng làm hết — **một** finding sâu
> ăn điểm hơn năm bảng nông.

**Đã làm:** _<B1 build-compare / B2 sweep nào / B4 challenge nào / B5 lựa chọn nào>_

**Numbers:**

```
before:  <số>
after:   <số>
speedup: <X.Y>×
```

**Điều này nói lên gì mà deck chưa nói:**

_(để trống nếu bạn không làm phần này)_

---

## 7. Điều làm bạn ngạc nhiên nhất  *(optional)*

_(1–2 câu. Không bắt buộc, nhưng grader đọc hết.)_

_(để trống nếu bạn không làm phần này)_

---

## 8. Self-check trước khi push

- [x] `hardware.json` committed
- [x] `models/active.json` committed
- [x] `benchmarks/01-quickstart-results.md` committed (`make bench`)
- [x] `benchmarks/01-tuning-tg128.md` committed (`make tune`)
- [x] `benchmarks/02-server-results.md` committed (`make load-report`)
- [x] `benchmarks/02-server-batching-u50.md` hoặc `-metrics-u50.csv` committed (`make metrics`)
- [x] `benchmarks/locust-10_stats.csv` + `locust-50_stats.csv` committed (`make load-10` / `load-50`)
- [x] `benchmarks/03-integration-results.md` committed (`make pipeline`)
- [x] Mọi section **"required — replace this line"** trong các file `benchmarks/*.md`
      đã được thay bằng nhận xét của bạn
- [x] 5 screenshots trong `submission/screenshots/`
- [ ] `make verify` → **exit 0**
- [ ] Repo GitHub ở chế độ **public**
- [ ] Đã paste public URL vào VinUni LMS
- [x] **Không** commit `models/*.gguf` hay `runtime/` (đã có trong `.gitignore`)

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Private → grader không
xem được → 0 điểm.
