# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.
>
> `make verify` sẽ fail nếu còn placeholder chưa điền. Đó là cố ý.

**Họ Tên:** Phạm Khắc Tú
**MSSV:** 2A202602866
**Cohort:** A20-K4
**Ngày submit:** 2026-10-06

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

> Từ `make probe`. Paste output hoặc điền tay.

- **OS:** Windows 10 AMD64
- **CPU:** Intel Core i5-10300H @ 2.50 GHz
- **Cores:** 4 physical / 8 logical
- **CPU extensions:** AVX2 / FMA
- **RAM:** 15.8 GB
- **Accelerator:** NVIDIA GeForce GTX 1650 4096 MiB, Vulkan
- **llama.cpp asset đã tải:** `llama-b10488-bin-win-vulkan-x64.zip`
- **Model đã dùng:** Gemma 4 E2B (`LAB_MODEL=gemma4-e2b`)
- **Quantization:** `UD-Q4_K_XL` + `UD-Q2_K_XL` (từ `models/active.json`)

**Chạy ở đâu:** laptop của tôi, không dùng Colab/Kaggle
_(Nếu dùng cloud fallback: nói rõ vì sao — RAM < 8 GB, setup fail, v.v. Không mất điểm.)_

**Setup story** (≤ 80 chữ): điều gì cần thay đổi để lab chạy trên máy bạn? Có bước
nào fail rồi phải workaround không?

Windows PowerShell 5.1 gặp lỗi mã hóa UTF-8 nên tôi bật `PYTHONUTF8=1` và sửa launcher để giữ `llama-server` chạy trên Windows. Tải model bằng Hugging Face Python bị treo, vì vậy tôi dùng liên kết chính thức với `curl` có resume. Port 8080 bị Apache chiếm nên các bước serving dùng port 8090.

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

> Paste bảng từ `benchmarks/01-quickstart-results.md` (`make bench` tự sinh).

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 19861 | 402 / 1293 | 19.9 / 21.3 | 1644 / 2535 / 2535 | 50.3 |
| UD-Q2_K_XL | 2.24 | 8748 | 461 / 3928 | 23.2 / 24.5 | 1915 / 5404 / 5404 | 43.2 |

**Quan sát** (≤ 60 chữ): 2-bit nhanh hơn bao nhiêu, và **có đáng không**? Bạn đã thử
hỏi cùng một câu trên cả hai (`make serve` vs `.venv/bin/python labs/02-serve/serve.py --compare`)
chưa? Chất lượng khác nhau thế nào?

Q2 nhỏ hơn 0.73 GB (25%) nhưng decode 43.2 thay vì 50.3 tok/s; Q4 nhanh hơn 1.16×. Với cùng prompt và ngân sách 128 token, cả hai đều đúng yêu cầu và nêu cùng ý; Q2 dài hơn (91 so với 77 token), chưa thấy giảm chất lượng rõ. Vì RAM đủ, tôi chọn Q4; Q2 chỉ đáng dùng khi thiếu bộ nhớ.

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

> Từ `benchmarks/02-server-results.md` (`make load-report`).

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 0.50 | 16000 | 24000 | 24000 | 8.4 | 0.0% |
| 50 | 0.53 | 30000 | 53000 | 57000 | 16.0 | 0.0% |

- **Offered load tăng 5×, throughput thực tăng:** 1.06×
- **P95 tăng:** 2.21×
- **Effective concurrency ở 50 users:** 16.0 so với `--parallel` = 4 slots

**Peak `llamacpp:n_busy_slots_per_decode`** (từ `make metrics` khi `make load-50` đang
chạy): 3.76 / 4 slots

**Saturation reading** (≤ 80 chữ): server của bạn bão hoà ở đâu, và **bằng chứng nào**
thuyết phục bạn? Nếu P95 tăng nhanh hơn RPS thì phần latency thêm đó là queue time hay
compute time — bạn biết bằng cách nào? Nếu bạn phải nâng goodput@SLO, bạn sẽ đổi knob
nào **trước**, và vì sao knob đó?

Server gần bão hòa từ 10 users và bão hòa rõ trước 50 users: tải tăng 5× nhưng RPS chỉ tăng 1.06×, trong khi P95 tăng 2.21×. Effective concurrency 16.0 vượt 4 slot, peak busy 3.76/4 và 46 request deferred chứng minh latency thêm là queue time. Với SLO 30 giây, tôi sẽ thử tăng `--parallel` trước rồi đo lại RPS/P95, vì hàng đợi slot là giới hạn trực tiếp.

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

> Từ `make pipeline`. Nói thật cái nào real, cái nào stub — stub **không** mất điểm.

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | localhost | stub |
| N17 Data pipeline | in-memory list | stub |
| N18 Lakehouse | `TOY_DOCS` dictionary | stub |
| N19 Vector + features | keyword overlap | stub |
| N20 Serving | `llama-server` | real |

**Latency split** (mean của 3 query, từ output của `pipeline.py`):

- embed: 0.0 ms
- retrieve: 0.2 ms
- llm: 7283.3 ms
- **stage chiếm nhiều nhất:** LLM (100% của total)

**Reflection** (≤ 60 chữ): bottleneck ở đâu? Có khớp với kỳ vọng của bạn không? Nếu
phải giảm latency của pipeline này 2×, bạn sẽ tấn công vào đâu?

LLM là bottleneck như kỳ vọng, chiếm gần 100% tổng thời gian; keyword retrieval chỉ mất 0.2 ms và embedding đang tắt. Muốn giảm latency 2×, tôi sẽ tối ưu LLM decode bằng cách giảm ngân sách output token và thử accelerator/runtime nhanh hơn. Tối ưu retrieval gần như không thay đổi tổng latency; Q2 cũng không phù hợp vì đã đo chậm hơn Q4.

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

> **Phần quan trọng nhất của report.** Không cần bonus track: `make tune` đã cho bạn
> một before/after thật (`benchmarks/01-tuning-tg128.md`). Đổi quantization,
> `LAB_N_CTX`, hay `--parallel` rồi đo lại cũng được.

**Change:** giảm số CPU thread từ mặc định `-t 4` xuống `-t 1`

```
before:  50.8 tok/s (-t 4)
after:   52.0 tok/s (-t 1)
speedup: 1.02×
```

**Tại sao nó work** (1–2 đoạn — đây là phần grader đọc kỹ nhất):

_Giải thích như đang nói với bạn ngồi cạnh. Bám vào **cơ chế**, không phải "vibes":
memory bandwidth? vector width? cache residency? scheduling? queueing? Nếu kết quả
**khác** với kỳ vọng từ deck — nói rõ, và giải thích vì sao. Grader thưởng điểm cho
lập luận đúng về một kết quả bất ngờ, hơn là một con số đẹp không được giải thích._

Đường cong gần như phẳng và đạt knee ngay tại một thread: từ 1 đến 16 thread, throughput chỉ dao động 50.5–52.0 tok/s. Kết quả này khác trực giác “nhiều core hơn sẽ nhanh hơn” vì phép đo dùng `ngl=99`, nên phần lớn layer đã được offload lên GPU. Decode bị giới hạn chủ yếu bởi việc thực thi và di chuyển bộ nhớ phía GPU, không phải số core CPU.

Các thread CPU bổ sung không tạo thêm throughput hữu ích; chúng chỉ thêm scheduling và synchronization overhead trong khi cùng chia sẻ tài nguyên GPU/bộ nhớ. Mức tăng 1.02× nhỏ và có thể một phần là nhiễu giữa các lần chạy, nhưng chính độ phẳng của toàn bộ sweep là bằng chứng rằng tăng CPU thread không phải knob hiệu quả trên cấu hình này.

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
- [X] Repo GitHub ở chế độ **public**
- [X] Đã push và paste public URL vào VinUni LMS **trước 23:59 (UTC+7) ngày làm lab**
- [x] **Không** commit `models/*.gguf`, `runtime/` hay `.env` (đã có trong `.gitignore`)

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Private → grader không
xem được → 0 điểm.

---

## 9. Khai báo sử dụng AI  *(xem `docs/RULES.md` §3)*

Tôi sử dụng OpenAI Codex để giải thích yêu cầu lab, debug lỗi Windows
PowerShell/UTF-8 và launcher server, hỗ trợ chạy các lệnh đo, đối chiếu số liệu thật
từ `benchmarks/`, và biên tập phần trình bày. Tôi không dùng AI để tạo số liệu hoặc
screenshot giả; toàn bộ số đo và ảnh đều lấy từ các lần chạy trên laptop đã khai báo.
