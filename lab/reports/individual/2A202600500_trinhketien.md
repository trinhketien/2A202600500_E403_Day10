# Báo Cáo Cá Nhân — Lab Day 10: Data Pipeline & Observability

**Họ và tên:** Trịnh Kế Tiến
**Vai trò:** Pipeline Engineer / Observability / Team Leader
**Ngày nộp:** 15/04/2026
**Độ dài yêu cầu:** 400–650 từ

---

## 1. Tôi phụ trách phần nào? (80–120 từ)

**File / module:**
- `etl_pipeline.py`: Tôi fix triệt để lỗi Unicode encoding ở dòng console logger, giúp tool chạy trơn tru trên môi trường console của cá nhân. Thực hiện thao tác chay bash file và chạy run args.
- `transform/cleaning_rules.py`: Tối ưu hàm clean_rows với 3 quy tắc mới (Rule R7: Lọc ký tự trắng BOM (\ufeff) / R8: Chặn độ dài / R9: Ép field "Exported at").
- `quality/expectations.py`: Tích hợp Expectation 7 và 8.

**Kết nối với thành viên khác:**
Tôi tạo base source vững chắc, build lại bash pipeline `run_id=sprint1-clean` để team có metrics rõ ràng đo đếm trước khi chuyển qua Inject lỗi Bad Data cho team Report. Đồng thời thiết kế script ghi tự động ra CSV.

**Bằng chứng (commit / comment trong code):**
Đoạn code Strip bom (Commit mới push trên dev branch):
```python
# ── R7: strip_bom_control_chars
cleaned_text = re.sub(r"[\x00-\x08\ufeff]", "", text)
```

---

## 2. Một quyết định kỹ thuật (100–150 từ)

Khi viết script đánh giá chất lượng Data (*Expectation* module), một quyết định lớn là phân cấp cấu hình **`SEVERITY` (Mức độ nghiêm trọng của lỗi Data)** thành 2 nhãn rõ ràng: `WARN` và `HALT`, thay vì một trạng thái fail chung chung. 
- Nhãn **`WARN`** tôi ứng dụng vào luật *E8 (Thiếu exported_at)*: Nếu thiếu metadata tracking chỉ báo cáo về log để review chứ không chặn luồng chạy.
- Nhãn **`HALT`** được quy định nghiêm ngặt cho *E7 (Ký tự BOM lẫn trong văn bản)* và *Stale 14-day policy*: Bất kì file raw nào dính lỗi này tôi can thiệp trả kết quả Halt. Trong `etl_pipeline.py` tôi bắt cờ nếu `halt = True` thì văng `SystemExit` (không embed lên ChromaDB). Quyết định cô lập 100% Vector Database khỏi dữ liệu lỗi lầm này tiết kiệm vô số tiền bạc cho công ty để không giải quyết claim pháp lý sai.

---

## 3. Một lỗi hoặc anomaly đã xử lý (100–150 từ)

**Symptom:**
Khi thực hiện test Sprint 3 "Inject Data Có Nguy Cơ" (`python etl_pipeline.py run --run-id inject-bad --no-refund-fix --skip-validate`), console Terminal chết cứng liên quan đến quá trình encoding log: `UnicodeEncodeError: 'charmap' codec can't encode character...`

**Diagnosis:**
Hàm `log(msg)` báo cáo thất bại vì thông báo `log("WARN: expectation failed but --skip-validate → tiếp tục embed (chỉ dùng cho demo Sprint 3).")` có chứa ký tự tiếng Việt với dấu Unicode và đặc biệt là dấu mũi tên `→`. Console trên local không config encode cp1258 nên Crash luôn script Python trước khi lưu Manifest file vào ổ cứng.

**Mitigation:**
Tôi lập tức can thiệp thay nội dung log trên source `etl_pipeline.py` (Line 91) từ text Tiếng Việt + icon utf8 sang 100% dạng mã ASCII:
`log("WARN: expectation failed but --skip-validate -> continuing embed (only for Sprint 3 demo).")`
Sau fix này, script chạy xuyên suốt và tạo thành công `eval_bad.csv`.

---

## 4. Bằng chứng trước / sau (80–120 từ)

Quá trình Eval Data Vector tạo ra 2 CSV trước và sau minh chứng cực kỳ sắc nét: (Extracting the specific cell records)

**Sự cố khi Inject Data dính lỗi bị nạp vào nhầm (Eval Bad - `run_id=inject-bad`):**
Cột `hits_forbidden` chỉ thẳng kết quả **`yes`** khi Rất nhiều Policy hoàn tiền cũ chui được lên DB!
`q_refund_window, ..., policy_refund_v4, Yêu cầu được gửi trong vòng 7 ngày làm việc... và 14 ngày làm việc... ,yes,yes,,3`

**Khi ETL Clean Guardrail của tôi được kích hoạt (Eval Clean - `run_id=sprint4-final`):**
Nhờ chặn từ vòng gửi xe các lỗi trên pipeline, Agent Retrieval đã ko truy vấn ra kết quả lỗi:
`q_refund_window, ..., policy_refund_v4, Yêu cầu được gửi trong vòng 7 ngày làm việc kể từ thời điểm...,yes,no,,3` -> Cột `hits_forbidden=no` (Bằng chứng thép!).

---

## 5. Cải tiến tiếp theo (40–80 từ)

Nếu tôi có dư dả 2 giờ làm việc nữa, tôi sẽ cài đặt Pandas-Profiling (ydata-profiling) hoặc Great Expectation Dashboard chính hiệu để sinh HTML report tự động sau mỗi lần extract CSV thay vì việc print log CLI thô sơ. Điều này giúp Data Engineer quan sát phổ phân bố dữ liệu Outlier rất dễ dàng trước khi nhấn nút "Build Model".
