# Bonus B1 - Prebuilt vs source build

Host `Windows-AMD64` · CPU `AMD Ryzen 9 5900HS with Radeon Graphics`
Vector extensions detected: none
llama.cpp `b10488` both sides · `threads=8` ·
**both pinned to `ngl=0`** so this isolates the compiler ·
metric `tg128`, 3 repetitions

> **Backend mismatch, handled.** The prebuilt binary sees
> `['CUDA0: NVIDIA GeForce RTX 3060 Laptop GPU (6143 MiB, 5130 MiB free)']` and your source build sees `(no devices)`.
> Left at `-ngl 99` this comparison would have measured the accelerator and printed
> it under a compiler headline, so both sides were pinned to `-ngl 0`.

| Binary | Built for | tg128 (tok/s) | Relative |
|:--|--:|--:|--:|
| prebuilt release | runtime CPU dispatch | 10.8 | 1.00x |
| your source build | this CPU (`-DGGML_NATIVE=ON`) | 11.0 | 1.02x |

On this machine, **they are within 3% -- no meaningful difference**.

before: 10.8 tok/s (prebuilt release)
after:  11.0 tok/s (source build, -DGGML_NATIVE=ON)
speedup: 1.02x

Same source revision, same model, same backend, same `-ngl` -- the only difference
is what the compiler was allowed to assume about the CPU.
A gap this small usually means the prebuilt binary already dispatches to the right kernels at runtime (releases ship one libggml-cpu-*.so per microarchitecture and pick via CPUID), or that this workload is bandwidth-bound rather than instruction-bound. Both are real findings -- say which one you think it is.


## Your explanation

Khoảng cách chỉ 2% (10.8 → 11.0 tok/s) — thực chất là hoà. Hai lý do cụ thể:

1. **Không có extension mới để bật.** Probe báo `Vector extensions detected: none`
   cho CPU này, nên `-DGGML_NATIVE=ON` gần như không mở khoá thêm AVX2/AVX-512 nào
   mà bản prebuilt chưa dùng. Prebuilt release của llama.cpp ship nhiều
   `libggml-cpu-*` theo microarchitecture và chọn qua CPUID lúc chạy, nên nó vốn đã
   dispatch đúng kernel cho Zen 3 — build lại từ nguồn không cho thêm gì.

2. **Workload này bound bởi memory bandwidth, không phải instruction throughput.**
   tg128 (decode) là bước autoregressive: mỗi token phải đọc lại toàn bộ weight từ
   RAM. Bottleneck là băng thông bộ nhớ, không phải số lệnh SIMD/giây. Vì vậy dù
   compiler có phát ra lệnh vector tối ưu hơn thì cũng không rút ngắn được thời gian
   chờ dữ liệu — 2% chênh lệch nằm trong nhiễu đo.

Kết luận: trên máy này compiler flag là knob vô nghĩa; nút thắt thật nằm ở bandwidth
(và ở tầng GPU khi bật offload — xem sweep `-ngl`).
