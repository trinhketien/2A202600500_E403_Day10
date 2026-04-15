# Báo Cáo Cá Nhân — Lab Day 10: Data Pipeline & Observability

**Họ và tên:** Trịnh Kế Tiến  
**Vai trò:** Pipeline Engineer / Observability Lead  
**Ngày nộp:** 15/04/2026  
**Độ dài yêu cầu:** 400–650 từ

---

> Viết **"tôi"**, đính kèm **run_id**, **tên file**, **đoạn log** hoặc **dòng CSV** thật.  
> Lưu: `reports/individual/2A202600500_trinhketien.md`

---

## 1. Tôi phụ trách phần nào? (80–120 từ)

**File / module:**

- `etl_pipeline.py`: Tôi chạy pipeline end-to-end với 3 run_id khác nhau (`sprint1-clean`, `inject-bad`, `sprint4-final`). Fix lỗi UnicodeEncodeError tại dòng 91 (console cp1258 không encode được ký tự tiếng Việt).
- `transform/cleaning_rules.py`: Thêm 3 quy tắc mới R7 (strip BOM/control chars), R8 (min chunk length 20), R9 (reject missing exported_at).
- `quality/expectations.py`: Thêm 2 expectations mới E7 (`no_bom_control_in_chunk_text`, halt) và E8 (`all_rows_have_exported_at`, warn).
- `data/grading_questions.json`: Tạo bộ câu hỏi grading 3 câu (`gq_d10_01`, `gq_d10_02`, `gq_d10_03`) theo đúng rubric SCORING.md để chạy `grading_run.py`.

**Kết nối với thành viên khác:**

Tôi build baseline pipeline vững, tạo `run_id=sprint1-clean` (10 raw → 6 cleaned, 4 quarantine) làm chuẩn đo cho nhóm. Sau đó inject lỗi (`inject-bad`) và clean lại (`sprint4-final`) để nhóm có bằng chứng before/after viết báo cáo.

**Bằng chứng (commit / comment trong code):**

Commit `2209980` trên GitHub: `docs: update all reports + docs + grading with official gq_d10_01-03 IDs (3/3 PASS MERIT)`

---

## 2. Một quyết định kỹ thuật (100–150 từ)

Quyết định quan trọng nhất là phân cấp **severity** cho expectations thành `halt` vs `warn`:

- **E7** (`no_bom_control_in_chunk_text`) → `halt`: Ký tự BOM/control trong chunk_text sẽ phá hỏng embedding vector. Nếu lọt vào ChromaDB, retrieval sẽ trả kết quả sai âm thầm — rất khó debug. Do đó tôi đặt `halt` để pipeline dừng ngay, không embed.

- **E8** (`all_rows_have_exported_at`) → `warn`: Thiếu `exported_at` không ảnh hưởng chất lượng retrieval trực tiếp, nhưng khiến freshness check không đo được SLA. Đặt `warn` để ghi nhận vào log mà không chặn pipeline — team monitor sẽ review định kỳ.

Triết lý: **halt chỉ dành cho lỗi ảnh hưởng trực tiếp đến end-user (Agent trả lời sai)**. Lỗi metadata/tracking dùng `warn` để không gây downtime không cần thiết.

---

## 3. Một lỗi hoặc anomaly đã xử lý (100–150 từ)

**Triệu chứng:**

Khi chạy Sprint 3 inject (`python etl_pipeline.py run --run-id inject-bad --no-refund-fix --skip-validate`), pipeline crash tại dòng 91 với lỗi:
```
UnicodeEncodeError: 'charmap' codec can't encode character '\u1ebf' 
in position 50: character maps to <undefined>
```

**Metric/check phát hiện:**

Lỗi xảy ra ở hàm `log()` khi `print()` cố ghi tiếng Việt có dấu ra console cp1258 (Windows). Pipeline đã chạy xong validation (ghi nhận `refund_no_stale_14d_window FAIL`) nhưng crash trước khi embed → manifest không được tạo → freshness check không có dữ liệu.

**Fix:**

Thay log message tiếng Việt bằng ASCII thuần:
```python
# Trước (crash):
log("WARN: expectation failed but --skip-validate → tiếp tục embed...")
# Sau (fix):
log("WARN: expectation failed but --skip-validate -> continuing embed (only for Sprint 3 demo).")
```

Sau fix, pipeline chạy xuyên suốt: embed thành công, manifest được ghi, eval_bad.csv được tạo với `hits_forbidden=yes`.

---

## 4. Bằng chứng trước / sau (80–120 từ)

**Trước — Dữ liệu inject bẩn** (`run_id=inject-bad`, file `artifacts/eval/eval_bad.csv`):
```csv
q_refund_window,...,contains_expected=yes,hits_forbidden=yes
```
→ Agent nhận thông tin **mâu thuẫn**: vừa "7 ngày" vừa "14 ngày làm việc". `hits_forbidden=yes` chứng minh chunk stale đã lọt vào ChromaDB.

**Sau — Dữ liệu clean** (`run_id=sprint4-final`, file `artifacts/eval/eval_clean.csv`):
```csv
q_refund_window,...,contains_expected=yes,hits_forbidden=no
```
→ Agent chỉ nhận "7 ngày làm việc". `hits_forbidden=no` — không còn thông tin sai lệch.

**Grading cuối** (`artifacts/eval/grading_run.jsonl`): **3/3 câu PASS** (`gq_d10_01`, `gq_d10_02`, `gq_d10_03`) — `contains_expected=true`, `hits_forbidden=false`, `top1_doc_matches=true` → đạt hạng **MERIT**.

---

## 5. Cải tiến tiếp theo (40–80 từ)

Nếu có thêm 2 giờ, tôi sẽ tích hợp **ydata-profiling** (Pandas Profiling) để sinh HTML report tự động sau mỗi lần clean — hiển thị phân bố dữ liệu, missing values, và correlation matrix. Ngoài ra, sẽ thêm **Slack webhook** vào freshness_check.py để gửi alert tự động khi SLA vượt ngưỡng, thay vì chỉ ghi log thụ động như hiện tại.
