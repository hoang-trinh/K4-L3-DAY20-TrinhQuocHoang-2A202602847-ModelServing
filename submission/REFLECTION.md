# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.
>
> `make verify` sẽ fail nếu còn placeholder chưa điền. Đó là cố ý.

**Họ Tên:** Trịnh Quốc Hoàng
**MSSV:** 2A202602847
**Cohort:** K4
**Ngày submit:** 2026-10-06

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

> Từ `make probe`. Paste output hoặc điền tay.

- **OS:** Windows 10 (AMD64)
- **CPU:** AMD Ryzen 7 6800HS Creator Edition
- **Cores:** 8 physical / 16 logical
- **CPU extensions:** AVX2
- **RAM:** 13.7 GB
- **Accelerator:** Vulkan
- **llama.cpp asset đã tải:** llama-b10488-bin-win-vulkan-x64.zip
- **Model đã dùng:** Qwen3.5 0.8B (`LAB_MODEL=qwen35-0.8b`)
- **Quantization:** Q4_K_M + UD-Q2_K_XL (từ `models/active.json`)

**Chạy ở đâu:** laptop của tôi
_(Nếu dùng cloud fallback: nói rõ vì sao — RAM < 8 GB, setup fail, v.v. Không mất điểm.)_

**Setup story** (≤ 80 chữ): Trên Windows PowerShell 5.1, lab.ps1 gặp lỗi parse do ký tự em-dash không có BOM, và detect-hardware.py gặp UnicodeEncodeError trên console CP1252. Tôi đã sửa em-dash thành ASCII và cấu hình UTF-8 cho Python stdout.

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

> Paste bảng từ `benchmarks/01-quickstart-results.md` (`make bench` tự sinh).

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| Q4_K_M | 0.50 | 10358 | 2271 / 2407 | 21.9 / 22.3 | 3618 / 3811 / 3811 | 45.7 |
| UD-Q2_K_XL | 0.39 | 8189 | 2263 / 2367 | 21.7 / 22.1 | 3647 / 3742 / 3742 | 46.0 |

**Quan sát** (≤ 60 chữ): 2-bit chỉ nhanh hơn 0.3 tok/s (<1%) và nhỏ hơn 0.11 GB, không đáng đổi. Thử cùng câu hỏi trên serve, bản 4-bit trả lời chuẩn 57 token (2s), còn bản 2-bit mất stop token nên lặp 488 token đụng trần context (10s).

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

> Từ `benchmarks/02-server-results.md` (`make load-report`).

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 0.63 | 13000 | 21000 | 22000 | 8.5 | 0.0% |
| 50 | 0.74 | 35000 | 55000 | 56000 | 23.0 | 0.0% |

- **Offered load tăng 5×, throughput thực tăng:** 1.17×
- **P95 tăng:** 2.62×
- **Effective concurrency ở 50 users:** 23.0 so với `--parallel` = 4 slots

**Peak `llamacpp:n_busy_slots_per_decode`** (từ `make metrics` khi `make load-50` đang chạy): 3.80 / 4 slots

**Saturation reading** (≤ 80 chữ): Server bão hoà sâu ở 50 user: RPS chỉ tăng 1.17× (0.63 lên 0.74), P95 tăng 2.62× (21s lên 55s). Độ trễ tăng thêm là queue time vì effective concurrency (23.0) vượt xa 4 slot và requests_deferred = 46. Để nâng goodput tại SLO P95 ≤ 25s, tôi sẽ dùng Admission Control cắt queue quá hạn và tăng --parallel lên 6-8 slot.

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

> Từ `make pipeline`. Nói thật cái nào real, cái nào stub — stub **không** mất điểm.

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | local Windows | stub |
| N17 Data pipeline | sample corpus | stub |
| N18 Lakehouse | in-memory list | stub |
| N19 Vector + features | keyword overlap | stub |
| N20 Serving | `llama-server` | real |

**Latency split** (mean của 3 query, từ output của `pipeline.py`):

- embed: 0.0 ms
- retrieve: 0.1 ms
- llm: 5976.5 ms
- **stage chiếm nhiều nhất:** llm (100% của total)

**Reflection** (≤ 60 chữ): Bottleneck là llm (100% latency), đúng kỳ vọng vì autoregressive decode bị nghẽn băng thông nhớ. Để giảm latency 2×, tôi sẽ tấn công vào stage LLM bằng Speculative Decoding, hạ max_tokens hoặc dùng Prefix/KV Caching. Tối ưu retrieval không có tác dụng theo luật Amdahl.

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

> **Phần quan trọng nhất của report.** Không cần bonus track: `make tune` đã cho bạn
> một before/after thật (`benchmarks/01-tuning-tg128.md`). Đổi quantization,
> `LAB_N_CTX`, hay `--parallel` rồi đo lại cũng được.

**Change:** Tối ưu số thread decode -t từ 1 lên 8 (đúng số physical core của CPU AMD Ryzen 7 6800HS)

```
before:  45.9 tok/s
after:   51.9 tok/s
speedup: 1.13×
```

**Tại sao nó work** (1–2 đoạn — đây là phần grader đọc kỹ nhất):

Quá trình sinh token (tg128) bị giới hạn chủ yếu bởi băng thông bộ nhớ (memory bandwidth) do mỗi bước decode phải nạp lại toàn bộ trọng số mô hình từ RAM. Việc tăng từ 1 lên 8 thread tận dụng tối đa 8 core vật lý độc lập có pipeline và cache L1/L2 riêng để tính toán song song ma trận và dequantization, mang lại speedup 1.13x trước khi bão hòa kênh DDR5.

Vượt qua 8 thread (16 thread logical qua SMT hoặc 32 thread oversubscription), hiệu năng bị tụt ngược (từ 51.9 xuống 46.9 và 45.5 tok/s). Cơ chế là SMT chia sẻ chung ALU/cache của cùng core vật lý mà không tăng thêm băng thông bộ nhớ, dẫn đến cache thrashing và overhead chuyển ngữ cảnh (context switching) của hệ điều hành.

---

## 6. Bonus  *(optional — tối đa 10 điểm)*

> Bỏ trống nếu không làm. Xem `docs/bonus/README.md`. Đừng làm hết — **một** finding sâu
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

- [ ] `hardware.json` committed
- [ ] `models/active.json` committed
- [ ] `benchmarks/01-quickstart-results.md` committed (`make bench`)
- [ ] `benchmarks/01-tuning-tg128.md` committed (`make tune`)
- [ ] `benchmarks/02-server-results.md` committed (`make load-report`)
- [ ] `benchmarks/02-server-batching-u50.md` hoặc `-metrics-u50.csv` committed (`make metrics`)
- [ ] `benchmarks/locust-10_stats.csv` + `locust-50_stats.csv` committed (`make load-10` / `load-50`)
- [ ] `benchmarks/03-integration-results.md` committed (`make pipeline`)
- [ ] Mọi section **"required — replace this line"** trong các file `benchmarks/*.md`
      đã được thay bằng nhận xét của bạn
- [ ] 5 screenshots trong `submission/screenshots/`
- [ ] `make verify` → **exit 0**
- [ ] Repo tên đúng mẫu `K4-L3-DAY20-HoVaTen-MSSV-ModelServing` (xem `docs/SUBMISSION.md`)
- [ ] Repo GitHub ở chế độ **public**
- [ ] Đã push và paste public URL vào VinUni LMS **trước 23:59 (UTC+7) ngày làm lab**
- [ ] **Không** commit `models/*.gguf`, `runtime/` hay `.env` (đã có trong `.gitignore`)

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Private → grader không
xem được → 0 điểm.

---

## 9. Khai báo sử dụng AI  *(xem `docs/RULES.md` §3)*

_(Công cụ nào, dùng vào việc gì. Ghi "Không dùng" nếu không dùng.)_
