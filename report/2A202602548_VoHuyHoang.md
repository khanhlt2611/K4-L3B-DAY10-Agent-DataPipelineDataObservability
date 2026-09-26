# Member Role Report — Day 10: Data Pipeline & Data Observability

## 1. Thông tin cá nhân

| Thông tin | Nội dung |
|---|---|
| Họ và tên | Võ Huy Hoàng |
| MSSV | 2A202602548 |
| Khóa/Lớp | K4-L3B |
| Tên nhóm | Agent |
| Vai trò chính | Thành viên 3 — Observability Owner (phụ trách khả năng quan sát dữ liệu) |
| Repository | `K4-L3B-DAY10-Agent-DataPipelineDataObservability` |
| Ngày hoàn thành | 2026-09-26 |

## 2. Vai trò và phạm vi công việc

### Phần việc sở hữu

| Module/deliverable | File/hàm phụ trách | Input | Output bàn giao | Trạng thái |
|---|---|---|---|---|
| Data quality (chất lượng dữ liệu) | `src/observability/quality.py` — `run_data_quality_checks()` | DataFrame sạch/bị lỗi/phục hồi và `Settings` | `data/quality/*_quality_report.json` | Hoàn thành |
| Freshness SLA (cam kết mức độ mới dữ liệu) | `src/observability/quality.py` — `build_freshness_report()` | `published`, `age_days`, ngưỡng 180 ngày | Các freshness report (báo cáo độ mới) dạng JSON | Hoàn thành |
| Báo cáo pipeline | `src/observability/reporting.py` — `generate_phase1_report()`, `generate_corruption_report()` | Metrics (chỉ số), quality và freshness | `phase1_report.md`, `corruption_report.md` | Hoàn thành |

### Việc hỗ trợ ngoài phạm vi chính

| Hoạt động | Module được hỗ trợ | Kết quả |
|---|---|---|
| Thống nhất data contract (hợp đồng dữ liệu) với TV2 | `ingestion/cleaning.py` | Chốt các cột bắt buộc: `paper_id`, `title`, `summary`, `published`, `age_days` |
| Hỗ trợ tích hợp với TV4 | `pipelines/phase1.py`, `pipelines/corruption_flow.py` | Quality/freshness và hàm sinh báo cáo có thể được gọi ở cả ba trạng thái |

## 3. Kết quả theo vai trò

| Nhiệm vụ | File/artifact liên quan | Kết quả | Cách xác minh |
|---|---|---|---|
| Xây dựng Quality Gate (cổng kiểm soát chất lượng) bằng Great Expectations 1.x | `quality.py`, `baseline_quality_report.json` | Baseline có 24 dòng, đủ cột và pass toàn bộ 5 lần kiểm tra thuộc 4 lớp expectation | Đọc `success=true` và từng `expectations[].success=true` |
| Xây dựng Freshness SLA | `freshness_report.json` | Baseline có 0/24 dòng stale (quá hạn), `stale_ratio=0`, `is_fresh=true` | Đối chiếu JSON |
| Phát hiện dữ liệu corruption | `corrupted_quality_report.json`, `corrupted_freshness_report.json` | Gate fail: còn 21 dòng, 4 giá trị ID thuộc nhóm trùng, 2 summary rỗng; 8/21 dòng stale | Đối chiếu artifact trên nhánh tích hợp |
| Xác minh repair | `repaired_quality_report.json`, `repaired_freshness_report.json` | Trở lại 24 dòng, không còn expectation fail, 0 dòng stale | Đối chiếu artifact trên nhánh tích hợp |
| Sinh báo cáo Markdown tự động | `reporting.py` | Tổng hợp baseline và bảng so sánh Baseline/Corrupted/Repaired, có delta (độ chênh lệch) | `data/reports/*.md` |

Commit chính của tôi là `dd58e4c` — `feat: implement and validate data observability`, gồm `quality.py`, `reporting.py` và hai artifact baseline.

## 4. Giải thích phần kỹ thuật đã thực hiện

### Vấn đề cần giải quyết

Pipeline RAG có thể vẫn chạy dù dữ liệu bị thiếu, trùng, quá ngắn hoặc quá cũ. Đây là silent failure (lỗi âm thầm): hệ thống không crash nhưng câu trả lời suy giảm. Phần việc của tôi tạo các tín hiệu kiểm soát để phát hiện lỗi trước khi dữ liệu đi vào phục vụ truy vấn.

### Cách triển khai

- Dùng `gx.get_context(mode="ephemeral")` của Great Expectations 1.x để tạo context tạm thời, Pandas data source (nguồn dữ liệu Pandas), asset (tài sản dữ liệu), batch definition (định nghĩa lô dữ liệu) và batch (lô kiểm tra).
- Kiểm tra số dòng phải bằng `settings.max_results`; `paper_id` và `title` không được null (rỗng); `paper_id` phải duy nhất; `summary` dài tối thiểu 20 ký tự.
- Kiểm tra trước các cột bắt buộc. Nếu thiếu cột, báo cáo fail có kiểm soát thay vì để GX phát sinh lỗi khó hiểu.
- Tính `stale_rows` khi `age_days > 180`. Dataset chỉ fresh (đủ mới) khi không có tuổi dữ liệu sai và tỷ lệ stale không vượt 25%.
- Chuẩn hóa tên file báo cáo và giới hạn đường dẫn trong `data/quality/`, tránh report name (tên báo cáo) tạo đường dẫn ngoài phạm vi.
- Sinh Markdown từ artifact bằng code, không sửa tay số liệu, qua đó giữ tính reproducibility (khả năng tái lập).

### Input, output và contract

| Thành phần | Mô tả |
|---|---|
| Input | DataFrame có `paper_id`, `title`, `summary`, `published`, `age_days`; `Settings` chứa ngưỡng và đường dẫn |
| Output | Dictionary kết quả và file JSON/Markdown tương ứng |
| Module phụ thuộc | `core.config`, `core.utils`, Pandas, Great Expectations 1.x |
| Module sử dụng output | `pipelines.phase1`, `pipelines.corruption_flow`, phần báo cáo và demo |
| Trường hợp lỗi | Thiếu cột, tuổi dữ liệu không hợp lệ, DataFrame rỗng, trùng ID, summary quá ngắn, tỷ lệ stale vượt 25% |

### Cách xác minh

```powershell
python -c "import pandas as pd; from core.config import load_settings; from observability.quality import run_data_quality_checks, build_freshness_report; s=load_settings(); df=pd.read_csv(s.paths.clean_csv); q=run_data_quality_checks(df,s,'baseline'); f=build_freshness_report(df,s,s.paths.freshness_report); print(q['success'], f['is_fresh'], q['row_count'])"
```

- Kết quả mong đợi và thực tế: `True True 24`.
- Artifact: `data/quality/baseline_quality_report.json`, `data/quality/freshness_report.json`.

## 5. Một quyết định kỹ thuật quan trọng

- **Bối cảnh:** Cần xác định freshness theo từng dòng nhưng vẫn phải có kết luận ở cấp toàn bộ dataset (tập dữ liệu).
- **Phương án 1:** Chỉ cần tồn tại một dòng cũ là fail. Cách này quá nhạy và dễ báo động giả.
- **Phương án 2:** Chỉ dùng ngày mới nhất. Cách này có thể che giấu phần lớn dữ liệu cũ.
- **Phương án chọn:** Đếm tất cả dòng có `age_days > 180`, tính `stale_ratio`, fail khi tỷ lệ vượt 25% hoặc có tuổi dữ liệu không hợp lệ.
- **Lý do:** Phản ánh phân bố độ cũ của toàn bộ dataset thay vì một giá trị cực trị.
- **Bằng chứng:** Baseline `0/24` stale nên pass; corrupted `8/21 = 38,10%` nên fail; repaired `0/24` nên pass.

## 6. Một lỗi hoặc blocker đã xử lý

- **Triệu chứng:** `ModuleNotFoundError: No module named 'core'` khi chạy lệnh kiểm tra từ thư mục gốc.
- **Nguyên nhân gốc:** Dự án dùng src layout (cấu trúc mã nguồn trong thư mục `src`) nhưng package chưa được cài vào `.venv`.
- **Cách xử lý:** Cài dự án ở editable mode (chế độ liên kết mã nguồn) bằng `python -m pip install --no-deps --no-build-isolation -e .`.
- **Cách xác minh:** `python -c "from core.config import load_settings; print(load_settings().source_api)"` in `Crossref REST API`.
- **Điều học được:** Kích hoạt virtual environment (môi trường ảo) chưa đủ; với src layout cần cài package hoặc cấu hình `PYTHONPATH` đúng.

## 7. Hiểu biết về luồng end-to-end

1. Crossref trả raw response (phản hồi thô), được ánh xạ thành `PaperRecord`, làm sạch thành DataFrame, tạo `text_for_embedding`, sinh vector bằng MiniLM và nạp vào ChromaDB.
2. Evaluation set (tập đánh giá) chứa câu hỏi, ground truth (đáp án chuẩn) và document ID (mã tài liệu chuẩn). Kết quả truy xuất dùng để tính Hit Rate; câu trả lời dùng để tính Token F1 và điểm judge (bộ chấm).
3. Quality checks kiểm tra tính đầy đủ, duy nhất, kích thước và độ dài nội dung; freshness monitoring (giám sát độ mới) đo tuổi dữ liệu theo thời gian. Hai nhóm tín hiệu liên quan nhưng không thay thế nhau.
4. Baseline, corrupted và repaired phải dùng chung test set để biến độc lập duy nhất là trạng thái dữ liệu; nếu đổi câu hỏi thì không thể quy kết chênh lệch metric cho corruption.
5. Repair thành công khi quality/freshness trở lại pass và metrics phục hồi gần baseline. Kết quả thực tế phục hồi hoàn toàn trong lần chạy này.

## 8. Phân tích kết quả

| Metric/signal | Baseline | Corrupted | Repaired | Nhận xét |
|---|---:|---:|---:|---|
| `retrieval_hit_rate` | 1.0000 | 0.8000 | 1.0000 | Giảm 0,20 rồi phục hồi hoàn toàn |
| `mean_token_f1` | 1.0000 | 0.8108 | 1.0000 | Giảm 0,1892 rồi phục hồi hoàn toàn |
| `judge_accuracy` | 1.0000 | 0.8000 | 1.0000 | Giảm 0,20 rồi phục hồi hoàn toàn |
| `mean_judge_score` | 5.0000 | 4.2000 | 5.0000 | Giảm 0,80 rồi phục hồi hoàn toàn |
| Quality Gate | PASS, 0 lỗi | FAIL, 3 lỗi | PASS, 0 lỗi | Gate phát hiện đúng row count, duplicate và summary ngắn |
| Freshness SLA | PASS, 0/24 stale | FAIL, 8/21 stale | PASS, 0/24 stale | Corrupted đạt tỷ lệ stale 38,10%, vượt ngưỡng 25% |

Chuỗi nguyên nhân–bằng chứng:

1. Corruption làm mất dòng, tạo ID trùng, summary rỗng và đẩy ngày về quá khứ → Quality Gate/Freshness SLA fail → Hit Rate giảm từ `1,0` xuống `0,8`, Token F1 giảm xuống `0,8108`.
2. Repair tái tạo dữ liệu từ raw snapshot (ảnh chụp dữ liệu thô đáng tin cậy) → quality và freshness trở lại pass → toàn bộ metrics trở về mức baseline.

Không đủ bằng chứng để nói một loại corruption riêng lẻ ảnh hưởng mạnh nhất, vì artifact chỉ lưu kết quả của lần tiêm sáu lỗi đồng thời. Muốn kết luận nhân quả riêng từng lỗi phải chạy ablation test (thử nghiệm loại bỏ từng yếu tố), mỗi lần chỉ bật một corruption và giữ nguyên test set.

## 9. Điều học được và hướng cải thiện

### Ba điều quan trọng nhất

1. Pipeline chạy hết không đồng nghĩa dữ liệu đúng; cần Quality Gate trước bước indexing (đánh chỉ mục).
2. Một chỉ số freshness đơn lẻ như `latest_published` không đủ; cần tỷ lệ stale trên toàn bộ dữ liệu.
3. Data quality tác động trực tiếp đến retrieval và câu trả lời RAG: khi dữ liệu giảm chất lượng, cả Hit Rate và Token F1 đều giảm.

### Nếu có thêm thời gian

Tôi sẽ bổ sung pytest (kiểm thử tự động) cho DataFrame rỗng, thiếu cột, ngày sai, ID trùng và ngưỡng stale đúng 25%; đồng thời chạy ablation test để định lượng tác động riêng của từng corruption.

## 10. Cam kết của thành viên

- [x] Nội dung phản ánh đúng vai trò Thành viên 3 và commit của tôi.
- [x] Tôi có thể giải thích luồng end-to-end, không chỉ module mình phụ trách.
- [x] Các kết luận số liệu đều có artifact để đối chiếu.
- [x] Tôi không ghi thành công cho phần chưa được kiểm chứng.
- [x] Báo cáo không chứa `.env`, API key, token hoặc secret.
- [x] Báo cáo không sao chép nguyên văn báo cáo của thành viên khác.

**Họ và tên:** Võ Huy Hoàng  
**Ngày xác nhận:** 2026-09-26
