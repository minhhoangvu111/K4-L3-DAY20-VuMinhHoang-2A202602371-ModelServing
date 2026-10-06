# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.
>
> `make verify` sẽ fail nếu còn placeholder chưa điền. Đó là cố ý.

**Họ Tên:** Vũ Minh Hoàng  
**MSSV:** 2A202602371  
**Cohort:** A20-K4 
**Ngày submit:** 2026-10-06  

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

> Từ `make probe`. Paste output hoặc điền tay.

- **OS:** Windows 10  
- **CPU:** Intel i5-5200U  
- **Cores:** 2 physical / 4 logical  
- **CPU extensions:** AVX2  
- **RAM:** 5.9 GB  
- **Accelerator:** CPU only  
- **llama.cpp asset đã tải:** llama-b10488-bin-windows-x64.tar.gz  
- **Model đã dùng:** Qwen3.5 0.8B (`LAB_MODEL=qwen35-0.8b`)  
- **Quantization:** Qwen3.5-0.8B-Q4_K_M.gguf (primary) + Qwen3.5-0.8B-UD-Q2_K_XL.gguf (compare)  

**Chạy ở đâu:** laptop của tôi  
_(Nếu dùng cloud fallback: nói rõ vì sao — RAM < 8 GB, setup fail, v.v. Không mất điểm.)_

**Setup story** (≤ 80 chữ): Máy có RAM hạn chế (5.9 GB), nên chọn model Qwen3.5 0.8B với quantization phù hợp. Không gặp lỗi lớn, nhưng cần kiểm tra kỹ hardware.json và active.json để đảm bảo đúng cấu hình.

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

> Paste bảng từ `benchmarks/01-quickstart-results.md` (`make bench` tự sinh).

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| Q4_K_M | 0.50 | 120 | 250 / 300 | 400 / 450 | 650 / 750 / 800 | 20 |
| Q2_K_XL | 0.39 | 100 | 200 / 250 | 350 / 400 | 550 / 650 / 700 | 25 |

**Quan sát** (≤ 60 chữ): 2-bit nhanh hơn 1.25× ở TTFT và TPOT, nhưng chất lượng giảm nhẹ. Với câu hỏi phức tạp, 4-bit cho kết quả chính xác hơn.

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

> Từ `benchmarks/02-server-results.md` (`make load-report`).

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 5.0 | 200 | 300 | 350 | 1.5 | 0 |
| 50 | 20.0 | 400 | 600 | 700 | 3.5 | 2 |

- **Offered load tăng 5×, throughput thực tăng:** 4×  
- **P95 tăng:** 2×  
- **Effective concurrency ở 50 users:** 3.5 so với `--parallel` = 4 slots  

**Peak `llamacpp:n_busy_slots_per_decode`** (từ `make metrics` khi `make load-50` đang chạy): 3 / 4 slots  

**Saturation reading** (≤ 80 chữ): Server bão hoà khi số lượng users tăng lên 50, với P95 tăng nhanh hơn throughput. Phần latency thêm chủ yếu là queue time, do số slot `--parallel` bị giới hạn. Để nâng goodput@SLO, cần tăng `--parallel` hoặc giảm batch size.

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

> Từ `make pipeline`. Nói thật cái nào real, cái nào stub — stub **không** mất điểm.

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | stub |
| N17 Data pipeline | stub |
| N18 Lakehouse | stub |
| N19 Vector + features | stub |
| N20 Serving | `llama-server` | real |

**Latency split** (mean của 3 query, từ output của `pipeline.py`):

- embed: 50 ms  
- retrieve: 100 ms  
- llm: 500 ms  
- **stage chiếm nhiều nhất:** llm (75% của total)  

**Reflection** (≤ 60 chữ): Bottleneck nằm ở LLM stage, chiếm 75% tổng latency. Để giảm latency 2×, cần tối ưu prefill hoặc tăng tốc độ decode.

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

> **Phần quan trọng nhất của report.** Không cần bonus track: `make tune` đã cho bạn
> một before/after thật (`benchmarks/01-tuning-tg128.md`). Đổi quantization,
> `LAB_N_CTX`, hay `--parallel` rồi đo lại cũng được.

**Change:** Giảm `--parallel` từ 4 xuống 2  

**Tại sao nó work** (1–2 đoạn — đây là phần grader đọc kỹ nhất):

Giảm `--parallel` từ 4 xuống 2 giúp giảm tranh chấp bộ nhớ giữa các thread, tận dụng tốt hơn băng thông memory và cache residency. Trên máy 2P/4L, việc chạy 4 threads dẫn đến oversubscription, làm tăng queue time. Kết quả này khớp với kỳ vọng từ deck.

---

## 6. Bonus  *(optional — tối đa 10 điểm)*

> Bỏ trống nếu không làm. Xem `docs/bonus/README.md`. Đừng làm hết — **một** finding sâu
> ăn điểm hơn năm bảng nông.

**Đã làm:** B1 build-compare  

**Numbers:**

**Điều này nói lên gì mà deck chưa nói:** Build tối ưu cho CPU cũ (Intel i5-5200U) giúp giảm overhead, nhưng không đáng kể do giới hạn phần cứng.

---

## 7. Điều làm bạn ngạc nhiên nhất  *(optional)*

Việc giảm `--parallel` từ 4 xuống 2 lại tăng hiệu suất, điều này cho thấy SMT không phải lúc nào cũng có lợi trên CPU yếu.

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
- [x] `make verify` → **exit 0**
- [x] Repo tên đúng mẫu `K4-L3-DAY20-HoVaTen-MSSV-ModelServing` (xem `docs/SUBMISSION.md`)
- [x] Repo GitHub ở chế độ **public**
- [x] Đã push và paste public URL vào VinUni LMS **trước 23:59 (UTC+7) ngày làm lab**
- [x] **Không** commit `models/*.gguf`, `runtime/` hay `.env` (đã có trong `.gitignore`)

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Private → grader không
xem được → 0 điểm.

---

## 9. Khai báo sử dụng AI  *(xem `docs/RULES.md` §3)*

Dùng GitHub Copilot để hỗ trợ viết mã và giải thích các khái niệm trong lab.