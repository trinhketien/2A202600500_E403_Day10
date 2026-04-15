# Báo Cáo Nhóm — Lab Day 10: Data Pipeline & Data Observability

**Tên nhóm:** Nhóm Antigravity  
**Thành viên:**
| Tên | Vai trò (Day 10) | Email |
|-----|------------------|-------|
| Trịnh Kế Tiến | Pipeline End-to-end + Leader | trinhketiennic@email.com |
| Thành viên 2 | Cleaning Rules (R7–R9) | member2@example.com |
| Thành viên 3 | Expectations (E7–E8) + Quality | member3@example.com |
| Thành viên 4 | Inject Corruption + Eval Retrieval | member4@example.com |
| Thành viên 5 | Freshness Check + Docs + Báo cáo | member5@example.com |

**Ngày nộp:** 15/04/2026  
**Repo:** https://github.com/trinhketien/2A202600500_E403_Day10  
**Độ dài khuyến nghị:** 600–1000 từ

---

> **Nộp tại:** `reports/group_report.md`  
> Phải có **run_id**, **đường dẫn artifact**, và **bằng chứng before/after** (CSV eval hoặc screenshot).

---

## 1. Pipeline tổng quan (150–200 từ)

**Tóm tắt luồng:**

Pipeline ETL xử lý file export raw dạng CSV (`data/raw/policy_export_dirty.csv`) chứa 10 bản ghi chính sách công ty (Refund Policy v4, SLA P1 2026, IT Helpdesk FAQ, HR Leave Policy). Luồng xử lý end-to-end gồm 4 bước:

1. **Ingest**: Đọc CSV raw qua `load_raw_csv()` — thu được 10 raw records.
2. **Clean**: Hàm `clean_rows()` áp dụng 9 quy tắc (6 baseline + 3 mới R7–R9) để chuẩn hóa ngày ISO, loại doc_id lạ, fix refund 14→7 ngày, strip BOM, chặn chunk ngắn, reject thiếu exported_at. Kết quả: **6 cleaned + 4 quarantine**.
3. **Validate**: Module `run_expectations()` chạy 8 expectations (6 baseline + 2 mới E7–E8). Nếu có halt → pipeline dừng, không embed dữ liệu lỗi.
4. **Embed**: Upsert 6 chunks sạch vào ChromaDB collection `day10_kb` dùng model `all-MiniLM-L6-v2`. Strategy idempotent: upsert theo `chunk_id` + prune vector cũ không còn trong batch.

`run_id` được ghi trong mọi log và manifest JSON tại `artifacts/manifests/`.

**Lệnh chạy một dòng:**

```bash
python etl_pipeline.py run --run-id sprint4-final
```

---

## 2. Cleaning & expectation (150–200 từ)

Baseline đã có 6 rule (allowlist doc_id, normalize effective_date, HR stale <2026, missing chunk_text, dedupe, refund 14→7). Nhóm thêm **3 rule mới** + **2 expectation mới**:

### 2a. Bảng metric_impact (bắt buộc — chống trivial)

| Rule / Expectation mới | Trước (sprint1-clean) | Sau inject (inject-bad) | Chứng cứ |
|-------------------------|:-----:|:-----:|------|
| **R7** `strip_bom_control_chars` | quarantine=4 | Quarantine tăng khi inject BOM | `transform/cleaning_rules.py` L117–L128 |
| **R8** `min_chunk_content_length(<20)` | quarantine=4 | Quarantine tăng khi inject chunk ngắn | `transform/cleaning_rules.py` L130–L137 |
| **R9** `reject_missing_exported_at` | quarantine=4 | Quarantine tăng khi exported_at rỗng | `transform/cleaning_rules.py` L139–L143 |
| **E7** `no_bom_control_in_chunk_text` (halt) | OK | FAIL khi bypass R7 | `quality/expectations.py` L115–L129 |
| **E8** `all_rows_have_exported_at` (warn) | OK | FAIL khi bypass R9 | `quality/expectations.py` L131–L143 |

**Rule chính (baseline + mở rộng):**

- R1: allowlist doc_id → quarantine unknown_doc_id
- R2–R3: normalize effective_date ISO + quarantine invalid format
- R4: HR leave policy effective_date < 2026-01-01 → quarantine stale
- R5: missing chunk_text → quarantine
- R6: deduplicate chunk_text
- R7–R9: BOM strip, min length 20, reject missing exported_at **(MỚI)**

**Ví dụ 1 lần expectation fail và cách xử lý:**

Khi chạy `python etl_pipeline.py run --run-id inject-bad --no-refund-fix --skip-validate`:
```
expectation[refund_no_stale_14d_window] FAIL (halt) :: violations=1
WARN: expectation failed but --skip-validate -> continuing embed (only for Sprint 3 demo).
```
Expectation E3 phát hiện 1 chunk refund còn chứa "14 ngày làm việc". Nếu không có `--skip-validate`, pipeline sẽ HALT — bảo vệ ChromaDB khỏi dữ liệu sai.

---

## 3. Before / after ảnh hưởng retrieval hoặc agent (200–250 từ)

> Bắt buộc: inject corruption (Sprint 3) — mô tả + dẫn `artifacts/eval/…`

**Kịch bản inject:**

Nhóm cố ý tắt rule fix refund 14→7 ngày bằng flag `--no-refund-fix` và bỏ qua halt bằng `--skip-validate`. Lệnh: `python etl_pipeline.py run --run-id inject-bad --no-refund-fix --skip-validate`. Dữ liệu bẩn (chunk chứa "14 ngày làm việc") được embed trực tiếp vào ChromaDB.

**Kết quả định lượng (từ CSV):**

| Câu hỏi | Metric | eval_bad.csv (inject-bad) | eval_clean.csv (sprint4-final) |
|----------|--------|:---:|:---:|
| q_refund_window | contains_expected | yes | yes |
| q_refund_window | **hits_forbidden** | **yes** ❌ | **no** ✅ |
| q_p1_sla | contains_expected | yes | yes |
| q_lockout | contains_expected | yes | yes |
| q_leave_version | contains_expected | yes | yes |
| q_leave_version | top1_doc_expected | yes | yes |

**Phân tích:** Khi inject dữ liệu bẩn, câu `q_refund_window` vẫn tìm thấy từ khóa đúng ("7 ngày") NHƯNG đồng thời cũng hit forbidden keyword ("14 ngày làm việc") → `hits_forbidden=yes`. Agent sẽ nhận được thông tin mâu thuẫn: vừa nói 7 ngày vừa nói 14 ngày. Sau khi chạy clean pipeline (`sprint4-final`), `hits_forbidden=no` — Agent nhận thông tin thuần nhất.

**Grading run cuối cùng** (`artifacts/eval/grading_run.jsonl`, `run_id=sprint4-final`):

| ID chính thức | Câu hỏi | contains_expected | hits_forbidden | top1_doc_matches | Điểm (SCORING.md) |
|---------------|---------|:-:|:-:|:-:|:---:|
| `gq_d10_01` | Hoàn tiền bao nhiêu ngày? | ✅ true | ✅ false | ✅ true | 4đ (Pass) |
| `gq_d10_02` | SLA P1 bao lâu? | ✅ true | — | ✅ true | 3đ (Pass) |
| `gq_d10_03` | Phép năm 2026 bao nhiêu ngày? | ✅ true | ✅ false | ✅ true | 3đ (Merit) |

**3/3 câu PASS** — đạt hạng **MERIT**. Pipeline clean bảo vệ thành công dữ liệu cho AI Agent.

**Đường dẫn artifact:**
- `artifacts/eval/eval_bad.csv` — eval trên dữ liệu inject
- `artifacts/eval/eval_clean.csv` — eval trên dữ liệu sạch
- `artifacts/eval/grading_run.jsonl` — kết quả grading chính thức (3 dòng `gq_d10_01..03`)

---

## 4. Freshness & monitoring (100–150 từ)

Module `monitoring/freshness_check.py` đọc manifest JSON và so sánh `latest_exported_at` với thời gian hiện tại. SLA mặc định: **24 giờ**.

Kết quả chạy trên manifest `sprint4-final`:
```
freshness_check=FAIL {
  "latest_exported_at": "2026-04-10T08:00:00",
  "age_hours": 120.239,
  "sla_hours": 24.0,
  "reason": "freshness_sla_exceeded"
}
```

Dữ liệu export từ ngày 10/04, kiểm tra ngày 15/04 → cách 120 giờ > SLA 24 giờ → **FAIL**. Trong production, cronjob chạy pipeline mỗi 2–4 giờ sẽ giữ `age_hours` dưới ngưỡng SLA. Trạng thái PASS/WARN/FAIL cho phép team ops thiết lập alert (Slack/email) khi freshness vượt ngưỡng.

---

## 5. Liên hệ Day 09 (50–100 từ)

Dữ liệu sau embed nằm trong ChromaDB collection `day10_kb`. Collection này chính là Knowledge Base phục vụ cho Retrieval Worker và Synthesis Worker trong kiến trúc multi-agent Day 09. Pipeline ETL Day 10 đóng vai trò "upstream guardian": mọi dữ liệu từ hệ thống nguồn phải qua Clean → Validate → Embed trước khi Agent truy vấn. Nếu dữ liệu bẩn lọt qua, toàn bộ chuỗi Agent phía sau sẽ trả lời sai — đây là bài học cốt lõi "Garbage In, Garbage Out".

---

## 6. Rủi ro còn lại & việc chưa làm

- **Freshness SLA**: Dữ liệu lab mẫu cách 120 giờ -> luôn FAIL. Production cần cronjob tự động refresh.
- **Versioning VectorDB**: Upsert + prune thủ công. Cần snapshot/rollback mechanism cho collection Chroma.
- **Expectation framework**: Dùng code Python thuần. Nên chuyển sang Great Expectations hoặc Pandera cho reporting HTML.
- **Distinction target**: Chưa đạt 2 boundary freshness (ingest + publish) và chưa tích hợp GE/Pydantic thật.
