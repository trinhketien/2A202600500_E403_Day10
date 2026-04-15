# Báo Cáo Nhóm — Lab Day 10: Data Pipeline & Data Observability

**Tên nhóm:** Nhóm Antigravity (Demo Group)  
**Thành viên:**
| Tên | Vai trò (Day 10) | Email |
|-----|------------------|-------|
| Trịnh Kế Tiến | Chạy Pipeline End-to-end + Leader | trinhketien@email.com |
| Member 2 | Thêm 3 Rule Mới (Cleaning Rules) | m2@example.com |
| Member 3 | Viết 2 Expectations (Data Quality) | m3@example.com |
| Member 4 | Inject Corruption & Eval | m4@example.com |
| Member 5 | Freshness Check & Báo cáo | m5@example.com |

**Ngày nộp:** 15/04/2026  
**Độ dài khuyến nghị:** 600–1000 từ

---

## 1. Pipeline tổng quan

**Tóm tắt luồng:**
Pipeline xuất phát từ file export raw dạng CSV chứa các snippet tài liệu chính sách công ty (Refund Policy, IT Helpdesk, HR leave). Dữ liệu này được đưa qua hàm `clean_rows` để chuẩn hóa (ngày tháng, lọc rác thẻ BOM, loại câu quá ngắn, quy đổi cửa sổ hoàn tiền 14 ngày thành 7 ngày cho hợp version v4). Sau đó, dữ liệu sạch được nạp qua module **Great Expectations siêu nhẹ** (`run_expectations`) để có cơ chế báo động (WARN) và ngắt (HALT) nếu vi phạm nghiêm trọng (ví dụ: mất ID, sai format ngày ISO). Cuối cùng mới được nhúng (embed) vào **ChromaDB** dùng làm Knowledge Base cho các Agent ngày 09, và nhả ra một file `manifest.json` ghi nhận metadata cho Freshness Check.

**Lệnh chạy một dòng (End-to-End Baseline Sprint 1):**
```bash
python etl_pipeline.py run --run-id sprint1-clean
```

---

## 2. Cleaning & expectation

Nhóm đã thêm **3 rule mới (R7, R8, R9)** và **2 expectation mới (E7, E8)** vào baseline có sẵn để bắt lỗi các dữ liệu cực đoan từ data export.

### 2a. Bảng metric_impact

| Rule / Expectation mới (tên ngắn) | Trước (chạy clean) | Khi inject (Sprint 3) | Chứng cứ (log / metric) |
|-----------------------------------|------------------|-----------------------------|-------------------------------|
| **R7:** `strip_bom_control_chars` | `quarantine: 4` | Cách ly dữ liệu ẩn ký tự BOM | File pipeline `etl_pipeline.py` |
| **R8:** `min_chunk_content_length`| `quarantine: 4` | Cách ly khi inject chunk < 20 ký tự | Chạy `clean_rows` loại rác hiệu quả|
| **R9:** `reject_missing_export_at`| `quarantine: 4` | Bị rớt (quarantine) nếu rỗng \`exported_at\` | Freshness check tránh văng lỗi None |
| **E7:** `no_bom_in_chunk_text` (HALT)| `OK` (halt) | FAIL (halt) khi rule làm sạch tắt bỏ | Expectation log `E7 -> FAIL` |
| **E8:** `all_rows_have_export_at` (WARN)| `OK` (warn) | FAIL (warn) khi không có time tracker | Expectation log `E8 -> FAIL` |

**Ví dụ 1 lần expectation fail và cách xử lý:**
Khi chạy lệnh inject lỗi (`python etl_pipeline.py run --run-id inject-bad --no-refund-fix --skip-validate`), lỗi chính sách refund 14 ngày đã lọt vào. Log báo `expectation[refund_no_stale_14d_window] FAIL (halt) :: violations=1`. Do ta bật cờ `--skip-validate` (bỏ qua HALT) nên nó vẫn nạp lên vectorDB. Hệ quả là model sinh ra file `eval_bad.csv` chứa `hits_forbidden = yes`. Nếu không bỏ qua, pipeline đã bị block không làm ô nhiễm DB!

---

## 3. Before / after ảnh hưởng retrieval hoặc agent

**Kịch bản inject (Sprint 3):**
Nhóm đã cố ý tắt mất luật fix Refund Policy 14 ngày (Flag `--no-refund-fix` và `--skip-validate`). Lúc này, tài liệu cũ với cú pháp "Yêu cầu hoàn tiền trong 14 ngày làm việc" bị trộn vào ChromaDB thay vì được fix về 7 ngày.

**Kết quả định lượng (từ CSV):**

🔹 *Trước khi Fix (Dữ liệu bẩn `eval_bad.csv`):*
```csv
q_refund_window, Khách hàng có bao nhiêu ngày để yêu cầu hoàn tiền?, Yêu cầu được gửi trong vòng 7 ngày (và 14 ngày), hits_forbidden=yes
```
➡️ Retrieval Engine tìm trúng câu trả lời *chứa từ khóa "14 ngày làm việc"*. Điều này khiến AI Agent tư vấn sai cho khách hàng về quyền lợi trả hàng (Sự cố pháp lý / tài chính nghiêm trọng xảy ra).

🔹 *Sau chạy Clean Run (Dữ liệu sạch `eval_clean.csv`):*
```csv
q_refund_window, Khách hàng có bao nhiêu ngày để yêu cầu hoàn tiền?, Yêu cầu được gửi trong vòng 7 ngày làm việc..., hits_forbidden=no
```
➡️ Lệnh `expectation[refund_no_stale_14d_window] OK (halt)` xuất hiện! Khách hàng được AI Agent trả lời chính xác, tránh tổn thất lớn.

---

## 4. Freshness & monitoring

Module `monitoring/freshness_check.py` phân tích JSON manifest. SLA freshness thiết lập mặc định ở ngưỡng `24.0 giờ`.
Trong thực tập lab, do timestamp ghi trong manifest mồi là `2026-04-10T08:00:00`, nhưng thời gian hiện tại là 15/04/2026 (cách 120 giờ) > 24 giờ, nên trạng thái manifest bị flag đỏ là:
`freshness_check=FAIL {"latest_exported_at": "2026-04-10T08:00:00", "age_hours": 120.239, "sla_hours": 24.0, "reason": "freshness_sla_exceeded"}` 
Nếu hệ thống đi vào vận hành thực tế, luồng cronjob sẽ đảm bảo age_hours này quanh quẩn ở mức 0-2 (khi job chạy mỗi 2 giờ).

---

## 5. Liên hệ Day 09

Dữ liệu sau embed được insert vào trực tiếp `ChromaDB` qua `collection=day10_kb`. Data Source này không bị rời rạc mà chính là "tim đập" nuôi các Agent (Retrieval Worker / Synthesis Worker) của bài Lab Day 09.
Pipeline ETL này bảo vệ tính nguyên vẹn cho Agent. Dữ liệu từ API của công ty sẽ được "Lọc → Rửa (Clean) → Check Nhãn Cấm (Expectation) → Vectorize". Ai đó chèn rác trên CSDL nguồn sẽ bị pipeline chặn lại không cho lọt xuống RAG Multi-Agent.

---

## 6. Rủi ro còn lại & việc chưa làm
- Expectation Pydantic: Script vẫn dùng code chay cơ bản để filter. Lý tưởng nên dùng Great Expectations thư viện chuẩn hoặc Pandera.
- Versioning VectorDB chưa mạnh tay: Khi Upsert theo `chunk_id`, nếu chunk nguồn bị delete thì ta chỉ đang prune rất thủ công, dễ gây dính vector ma trong kiến trúc HA lớn.
