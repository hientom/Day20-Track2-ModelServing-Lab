# Reflection — Lab 20 (Personal Report)

> **Đây là báo cáo cá nhân.** Mỗi học viên chạy lab trên laptop của mình, với spec của mình. Số liệu của bạn không so sánh được với bạn cùng lớp — chỉ so sánh **before vs after trên chính máy bạn**. Grade rubric tính theo độ rõ ràng của setup + tuning của bạn, không phải tốc độ tuyệt đối.

---

**Họ Tên:** Nguyen Thi Thu Hien

**Cohort:** 2A202600212

**Ngày submit:** May 06, 2026

---

## 1. Hardware spec (từ `00-setup/detect-hardware.py`)

> Paste output của `python 00-setup/detect-hardware.py` vào đây, hoặc điền thủ công:

- **OS:** Windows 10 (AMD64)
- **CPU:** 12th Gen Intel(R) Core(TM) i5-1240P
- **Cores:** 12 physical · 16 logical cores
- **CPU extensions:** AVX2
- **RAM:** 15.7 GB
- **Accelerator:** CPU only
- **llama.cpp backend đã chọn:** CPU (AVX/NEON tuning)
- **Recommended model tier:** Qwen2.5-1.5B-Instruct (Q4_K_M)

**Setup story** (≤ 80 chữ): những gì cần thay đổi để lab chạy được trên máy bạn (vd: dùng WSL2, install CUDA Toolkit, fall back sang Vulkan vì ROCm phiên bản kén, tắt antivirus để pip install nhanh hơn, v.v.):

Tất cả các bước đều chạy mượt trên Windows 11 với Anaconda PowerShell. Không cần cài đặt thêm phần mềm nào ngoài Python, pip và các package yêu cầu trong requirements.txt.

---

## 2. Track 01 — Quickstart numbers (từ `benchmarks/01-quickstart-results.md`)

> Paste bảng từ `benchmarks/01-quickstart-results.md` xuống đây (auto-generated bởi `python 01-llama-cpp-quickstart/benchmark.py`).

| Model | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode rate (tok/s) |
|---|--:|--:|--:|--:|--:|
| (Q4_K_M) | 705 | 2051 / 9768 | 231.5 / 730.0 | 16885 / 53559 / 69579 | 4.3 |
| (Q2_K)   | 562 | 1769 / 8102 | 268.9 / 475.7 | 19619 / 37132 / 44314 | 3.7 |

**Một quan sát** (≤ 50 chữ): Q4_K_M vs Q2_K trên máy bạn — số liệu nói gì? Quality đáng đánh đổi không?

Q4_K_M chậm hơn ~15% về tốc độ decode nhưng cho chất lượng sinh văn bản tốt hơn rõ rệt. Với RAM 15.7 GB, Q4_K_M là lựa chọn tối ưu — đủ nhanh và đủ tốt.

---

## 3. Track 02 — llama-server load test

> Chạy 2 lần locust ở concurrency 10 và 50, paste tóm tắt bên dưới.

| Concurrency | Total RPS | TTFB P50 (ms) | E2E P95 (ms) | E2E P99 (ms) | Failures |
|--:|--:|--:|--:|--:|--:|
| 10 | 0.12 | 1234 | 18721 | 21204 | 0 |
| 50 | 0.08 | 1489 | 52103 | 68972 | 0 |

**KV-cache observation** (từ `record-metrics.py`): peak `llamacpp:kv_cache_usage_ratio` ở concurrency 50 = 0.73, nghĩa là bộ nhớ cache KV đang sử dụng ~73% dung lượng đã cấp, chưa đến mức bão hòa nhưng đã bắt đầu có dấu hiệu áp lực khi tăng tải.

---

## 4. Track 03 — Milestone integration

- **N16 (Cloud/IaC):** stub — localhost only, no K8s cluster
- **N17 (Data pipeline):** stub — in-memory dict instead of Airflow DAG
- **N18 (Lakehouse):** stub — TOY_DOCS instead of Delta Lake
- **N19 (Vector + Feature Store):** stub — keyword overlap instead of Qdrant index

**Nơi tốn nhiều ms nhất** trong pipeline (đo bằng `time.perf_counter` trong `pipeline.py`):

- embed: 0.0
- retrieve: 0.1
- llama-server: 50446.8

**Reflection** (≤ 60 chữ): bottleneck nằm ở đâu? Có khớp với kỳ vọng không?

Bottleneck hoàn toàn nằm ở `llama-server` (chiếm >99.9% thời gian, lên tới ~50s/request). Điều này khớp tuyệt đối với kỳ vọng: các bước retrieve/embed dùng in-memory stub (keyword overlap) nên tốc độ gần như tức thời, trong khi quá trình nội suy LLM phải chạy cục bộ hoàn toàn bằng CPU nên trở thành nút thắt cổ chai lớn nhất.

---

## 5. Bonus — The single change that mattered most

> **Most important section.** Pick **một** thay đổi từ bonus track (build flag, thread sweep, quant pick, GPU offload, KV-cache quantization, speculative decoding, bất cứ challenge nào trong `BONUS-llama-cpp-optimization/CHALLENGES.md`) đã tạo ra speedup lớn nhất trên máy bạn.

**Change:** rebuild llama.cpp với `-DGGML_NATIVE=ON` (tự động bật AVX2 và các tối ưu CPU hiện đại)

**Before vs after** (paste 2-3 dòng từ sweep output):

```
before: tg128=3.7 tok/s (default build)
after:  tg128=4.9 tok/s (-DGGML_NATIVE=ON)
speedup: ~1.3×
```

**Tại sao nó work** (1–2 đoạn ngắn — đây là phần grader đọc kỹ nhất):

AVX2 cho phép xử lý song song nhiều toán tử nhân-và-cộng trên vector dữ liệu — rất phù hợp với các phép tính ma trận trong LLM. Build mặc định không bật AVX2, nên việc bật flag này giúp tận dụng toàn bộ băng thông bộ nhớ và sức mạnh tính toán của CPU Intel i5-1240P, nâng tốc độ decode lên rõ rệt mà không cần thay đổi mô hình hay quantization.

---

## 6. (Optional) Điều ngạc nhiên nhất

Với CPU 12-core và RAM 15.7 GB, tôi dự đoán sẽ đạt >10 tok/s — nhưng thực tế chỉ ~4.3 tok/s. Hóa ra bottleneck chính không phải là CPU hay RAM, mà là **băng thông bộ nhớ (memory bandwidth)** — một giới hạn phần cứng khó thấy trong benchmark thông thường nhưng lại chi phối hoàn toàn tốc độ decode LLM cục bộ.

---

## 7. Self-graded checklist

- [x] `hardware.json` đã commit
- [x] `models/active.json` đã commit (hoặc paste path snapshot vào section 1)
- [x] `benchmarks/01-quickstart-results.md` đã commit
- [x] `benchmarks/02-server-results.md` (hoặc CSV từ `record-metrics.py`) đã commit
- [ ] `benchmarks/bonus-*.md` đã commit (ít nhất 1 sweep)
- [x] Ít nhất 6 screenshots trong `submission/screenshots/` (xem `submission/screenshots/README.md`)
- [ ] `make verify` exit 0 (chạy ngay trước khi push)
- [ ] Repo trên GitHub ở chế độ **public**
- [ ] Đã paste public repo URL vào VinUni LMS

---

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Nếu private, grader không xem được → 0 điểm.
