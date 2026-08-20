# Bonus - GPU offload sweep

Host `Windows-AMD64` · backend(s) `nvidia_cuda, vulkan` ·
llama.cpp `b10488` · `threads=8` · metric `tg128`

| -ngl | tg128 (tok/s) | vs -ngl 0 | vs best |
|:--|--:|--:|--:|
| 0 | 10.5 | 1.00x | 10% |
| 8 | 12.5 | 1.19x | 12% |
| 16 | 20.7 | 1.97x | 20% |
| 24 | 36.6 | 3.49x | 35% |
| 32 | 58.1 | 5.54x | 55% |
| 99 | 104.8 | 10.00x | 100% |

Best: `-ngl 99` at 104.8 tok/s
-- 10.00x faster than CPU-only.

Where the curve flattens tells you the model ran out of layers to move. Where it
*peaks below* full offload tells you something did not fit and the accelerator
started paying to fetch weights it could not hold.

## Your finding

Full offload (`-ngl 99`) là tốt nhất trên máy này: 104.8 tok/s, nhanh **10×** so với
CPU-only (`-ngl 0`, 10.5 tok/s). Đường cong tăng **đơn điệu** qua mọi mốc
(10.5 → 12.5 → 20.7 → 36.6 → 58.1 → 104.8) và peak đúng ở giá trị cao nhất — **không**
có knee ở giá trị partial.

Điều đó cho biết **không có gì hết trước**: toàn bộ Gemma 4 E2B (UD-Q4_K_XL, ~2.97 GB)
cộng KV cache vẫn vừa trong 6 GB VRAM của RTX 3060, nên mỗi layer đẩy lên GPU đều được
trả công tuyến tính. Nếu VRAM cạn thì curve đã peak ở một `-ngl` partial rồi tụt, vì
phần layer tràn buộc device phải fetch weight qua PCIe (băng thông host↔device thấp hơn
VRAM nhiều) — ở đây không xảy ra. Kết luận: knob quyết định trên máy này là mức offload,
và vì model đủ nhỏ để nằm trọn trong VRAM nên câu trả lời tối ưu đơn giản là đẩy hết.
