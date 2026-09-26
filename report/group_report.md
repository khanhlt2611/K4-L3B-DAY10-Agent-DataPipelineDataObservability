# Group Report — Day 10: Data Pipeline & Data Observability

## 1. Thông tin bài nộp

| Thông tin         | Nội dung                  |
| ------------------ | -------------------------- |
| Khóa/Lớp         | K4-L3B-DAY10              |
| Tên nhóm         | Agent                      |
| Repository         | https://github.com/khanhlt2611/K4-L3B-DAY10-Agent-DataPipelineDataObservability |
| Ngày hoàn thành | 2026-09-26               |

### Thành viên và phân công

| STT | Họ và tên | MSSV | Vai trò chính | Module/deliverable sở hữu |
| --: | --- | --- | --- | --- |
| 1 | Trần Cao Quốc Dinh | 2A202602939 | Source & Environment Owner | `src/ingestion/crossref.py`, setup `.env` & `pyproject.toml`, `data/raw/crossref_response.json`, `data/raw/crossref_records.json` |
| 2 | Lê Trọng Khánh | 2A202602941 | Data Model & Evaluation-Set Owner | `src/ingestion/cleaning.py`, `src/evaluation/testset.py`, `data/clean/papers_clean.*`, `data/eval/test_set.json` |
| 3 | (Thành viên 3) | — | Observability Owner | `src/observability/quality.py`, `src/observability/reporting.py`, `data/quality/*` |
| 4 | (Thành viên 4) | — | Corruption & Integration Owner | `src/ingestion/corruption.py`, `src/pipelines/phase1.py`, `src/pipelines/corruption_flow.py`, `data/reports/*`, `data/results/*` |

## 2. Tóm tắt kết quả

Nhóm đã hoàn thành toàn bộ pipeline từ ingestion đến corruption/repair. Baseline pipeline tải và parse thành công 24 bài báo từ Crossref API (với fallback snapshot offline), làm sạch thành 24 records chuẩn hóa, sinh embedding bằng `sentence-transformers/all-MiniLM-L6-v2` và nạp vào ChromaDB collection `papers-baseline`. Bộ đánh giá 10 câu hỏi (3 summary, 3 authors, 2 date, 2 categories) được sinh ra deterministic và dùng chung cho cả ba trạng thái.

Baseline đạt kết quả hoàn hảo: `retrieval_hit_rate = 1.0`, `mean_token_f1 = 1.0`, `judge_accuracy = 1.0`, `mean_judge_score = 5`. Quality Gate (Great Expectations 1.x) pass toàn bộ 5 expectations; Freshness SLA PASS với 0 stale rows.

Corruption suite thực thi 6 kịch bản: xóa 5 records mới nhất, làm trắng 2 abstract, inject noise vào 2 records, truncate 2 title, set stale date cho 6 records, thêm 2 duplicate rows — đưa dataset từ 24 về 21 rows. Kết quả: `retrieval_hit_rate` sụt xuống 0.8, Quality Gate FAIL (3 expectations thất bại), Freshness FAIL (stale_ratio 38.1%). Sau khi repair từ raw snapshot, toàn bộ metrics phục hồi về mức baseline. Blocker quan trọng nhất đã gặp: Crossref không trả `subject` cho 24 records — xử lý bằng fallback `Uncategorized`.

## 3. Kiến trúc và luồng dữ liệu

### Luồng end-to-end

```text
Crossref API (hoặc snapshot offline data/raw/crossref_response.json)
    -> parse_crossref_payload → list[PaperRecord] → data/raw/crossref_records.json
    -> build_clean_dataframe → DataFrame 16 cột → data/clean/papers_clean.{csv,json}
    -> build_test_set → 10 câu hỏi → data/eval/test_set.json
    -> LocalEmbeddingIndex.build (all-MiniLM-L6-v2) → ChromaDB papers-baseline
    -> run_quality_checks (GX 1.x) → data/quality/baseline_quality_report.json
    -> check_freshness_sla → data/quality/freshness_report.json
    -> evaluate_pipeline → data/results/baseline_metrics.json
    -> data/reports/phase1_report.md
    -> apply_corruption_suite → data/clean/papers_clean_corrupted.{csv,json}
    -> re-index → papers-corrupted → re-evaluate → corrupted_metrics.json
    -> quality/freshness on corrupted → corrupted_quality_report.json / corrupted_freshness_report.json
    -> repair từ raw snapshot → data/clean/papers_clean_repaired.{csv,json}
    -> re-index → papers-repaired → re-evaluate → repaired_metrics.json
    -> data/reports/corruption_report.md
```

### Trách nhiệm của từng khối

| Khối             | Input          | Xử lý chính             | Output/artifact          | Owner          |
| ----------------- | -------------- | -------------------------- | ------------------------ | -------------- |
| Ingestion         | Crossref REST API / snapshot offline | Fetch với retry 3 lần (backoff 2ⁿ giây) khi 429/503; parse DOI/title/abstract/authors/dates; fallback offline | `data/raw/crossref_response.json`, `data/raw/crossref_records.json` | Trần Cao Quốc Dinh |
| Cleaning          | `list[PaperRecord]` | Loại JATS/XML, chuẩn hóa khoảng trắng, parse ngày → ISO, tính `age_days`, khử trùng lặp, sinh `text_for_embedding` 5 phần | `data/clean/papers_clean.csv`, `data/clean/papers_clean.json` | Lê Trọng Khánh |
| Embedding/index   | Clean DataFrame | Embed `text_for_embedding` bằng `all-MiniLM-L6-v2`; nạp vào ChromaDB (cosine similarity) | `data/chroma/` (3 collections), `data/embeddings/*.json` | (Thành viên 3) |
| Evaluation        | Clean DataFrame + ChromaDB index | Sinh 10 câu hỏi 4 loại; đo Hit Rate, Token F1, Judge Score bằng mock LLM | `data/eval/test_set.json`, `data/results/*_metrics.json`, `data/results/*_answers.json` | Lê Trọng Khánh + (Thành viên 4) |
| Observability     | Clean DataFrame | GX 1.x: 5 expectations (row count, null, unique, summary length); Freshness SLA: age_days > 180 | `data/quality/*_quality_report.json`, `data/quality/*_freshness_report.json` | (Thành viên 3) |
| Corruption/repair | Clean DataFrame | 6 kịch bản làm bẩn; repair bằng cách tái tạo từ raw snapshot | `data/clean/papers_clean_corrupted.*`, `data/clean/papers_clean_repaired.*`, `data/results/corruption_log.json` | (Thành viên 4) |
| Orchestration     | Settings | Chạy các bước theo thứ tự phụ thuộc; xuất báo cáo | `data/reports/phase1_report.md`, `data/reports/corruption_report.md` | (Thành viên 4) |

## 4. Cách tái hiện kết quả

### Cấu hình không chứa secret

| Biến/cấu hình             | Giá trị sử dụng |
| ---------------------------- | ------------------- |
| `LLM_PROVIDER`             | `mock`              |
| `LLM_MODEL`                | `mock`              |
| Embedding model              | `sentence-transformers/all-MiniLM-L6-v2` |
| Số lượng Crossref records | 24                  |
| Retrieval `top_k`           | 5                   |
| Freshness threshold          | 180 days            |
| Random seed, nếu có        | N/A — test set chọn theo sort + stride, không dùng random |

Không dán nội dung API key hoặc file `.env` vào báo cáo.

### Lệnh cài đặt

```bash
uv sync
```

Hoặc:

```bash
python -m pip install -e .
```

### Lệnh chạy

Baseline:

```bash
uv run python script/run_phase1.py
```

Hoặc với môi trường `pip` đã kích hoạt:

```bash
python script/run_phase1.py
```

Corruption flow:

```bash
uv run python script/run_corruption_flow.py
```

Hoặc với môi trường `pip` đã kích hoạt:

```bash
python script/run_corruption_flow.py
```

### Kết quả tái hiện

| Lệnh             | Trạng thái | Thời điểm chạy gần nhất | Bằng chứng                         |
| ----------------- | ------------ | ----------------------------- | ------------------------------------ |
| Baseline pipeline | Thành công | 2026-09-26T05:41:08Z | `data/results/baseline_metrics.json`, `data/reports/phase1_report.md` |
| Corruption flow   | Thành công | 2026-09-26T05:41:09Z | `data/results/corrupted_metrics.json`, `data/results/repaired_metrics.json`, `data/reports/corruption_report.md` |

## 5. Ingestion, cleaning và data contract

### Nguồn dữ liệu

| Thuộc tính                | Giá trị                             |
| --------------------------- | ------------------------------------- |
| Source                      | Crossref REST API — `https://api.crossref.org/works` |
| Query/filter                | `agentic retrieval augmented generation large language model` |
| Thời điểm lấy dữ liệu | 2026-09-26T05:41:08Z (hoặc fallback snapshot) |
| Số record nhận được    | 24 raw records, 24 clean records |
| Cơ chế retry/backoff      | Tối đa 3 lần, backoff `2**attempt` giây khi gặp HTTP 429/503 hoặc lỗi mạng; fallback đọc snapshot `data/raw/crossref_response.json` nếu API thất bại hẳn |

### Raw và clean schema

| Trường        | Kiểu dữ liệu | Bắt buộc?  | Ý nghĩa   | Xử lý khi thiếu/sai |
| --------------- | --------------- | ------------ | ----------- | ---------------------- |
| `paper_id`    | `str`           | Có          | DOI bài báo | Bỏ record nếu thiếu |
| `title`       | `str`           | Có          | Tiêu đề bài báo | Bỏ record nếu thiếu |
| `summary`     | `str`           | Có          | Abstract (đã loại JATS/XML) | Bỏ record nếu thiếu |
| `authors`     | `list[str]`     | Không       | Danh sách tác giả | Dùng list rỗng nếu thiếu |
| `categories`  | `list[str]`     | Không       | Chủ đề / subject | Dùng `["Uncategorized"]` nếu rỗng |
| `published`   | `str` (ISO date) | Có         | Ngày xuất bản `YYYY-MM-DD` | Bỏ record nếu không parse được |
| `updated`     | `str` (ISO date) | Không      | Ngày cập nhật | Dùng `published` nếu thiếu |
| `age_days`    | `int`           | Derived     | Số ngày từ `published` đến `run_date` | Tính lúc clean |
| `text_for_embedding` | `str`  | Derived     | Chuỗi 5 phần cho embedding | Sinh trong `build_clean_dataframe` |

### Quy tắc cleaning

| Quy tắc                                 | Quality dimension liên quan | Số record bị tác động | Cách xác minh      |
| ---------------------------------------- | ---------------------------- | -------------------------: | -------------------- |
| Loại JATS/XML tags khỏi abstract bằng regex `<[^>]+>` | Validity | 24 records có abstract JATS | Kiểm tra clean JSON |
| Chuẩn hóa khoảng trắng (normalize_whitespace) | Consistency | Tất cả fields text | So sánh raw và clean |
| Khử trùng lặp theo `paper_id` (case-insensitive) | Uniqueness | 0 duplicate trong baseline | GX: expect_column_values_to_be_unique |
| Bỏ record thiếu `paper_id`/`title`/`published` hợp lệ | Completeness | 0 record bị loại trong baseline | Đếm clean records = 24 |
| Fallback category → `Uncategorized` | Completeness | 24/24 records (Crossref không trả `subject`) | Kiểm tra `categories_joined` |

`text_for_embedding` được sinh theo template 5 phần:
```
Title: {title} | Authors: {authors_joined} | Categories: {categories_joined} | Published: {published} | Summary: {summary}
```
`paper_id` lấy trực tiếp từ DOI Crossref (lowercase). `age_days` = `run_date - published` tính theo ngày UTC.

## 6. Evaluation setup

| Thành phần                             | Cấu hình thực tế          |
| ---------------------------------------- | ----------------------------- |
| Số câu hỏi                            | 10                            |
| Các `question_type`                    | `summary` (3), `authors` (3), `date` (2), `categories` (2) |
| Ground-truth document ID                 | `paper_id` từ clean dataset; mỗi câu chứa `ground_truth_doc_ids` |
| Embedding model                          | `sentence-transformers/all-MiniLM-L6-v2` |
| Vector store/collection                  | ChromaDB persistent (cosine), collections: `papers-baseline`, `papers-corrupted`, `papers-repaired` |
| Retrieval `top_k`                       | 5                             |
| LLM provider/model                       | `mock` (deterministic) |
| Test set dùng chung cho ba trạng thái | `data/eval/test_set.json` — cùng một file không thay đổi |

Test set được giữ nguyên khi đánh giá baseline, corrupted và repaired để cô lập biến độc lập. Nếu câu hỏi thay đổi giữa các lần đo, chênh lệch metrics có thể đến từ độ khó câu hỏi chứ không phải từ chất lượng dữ liệu — làm mất ý nghĩa so sánh. Test set sinh deterministic bằng cách sort theo `published`/`paper_id` rồi chọn 10 vị trí trải đều trên corpus.

## 7. Kết quả baseline

### Artifact checklist

| Artifact                 | Đường dẫn thực tế                | Trạng thái | Ghi chú   |
| ------------------------ | -------------------------------------- | ------------ | ---------- |
| Raw response/records     | `data/raw/crossref_response.json`, `data/raw/crossref_records.json` | Có | 24 records |
| Cleaned dataset          | `data/clean/papers_clean.csv`, `data/clean/papers_clean.json` | Có | 24 rows, 16 cột |
| Embedding manifest/index | `data/chroma/` (ChromaDB collections) | Có | 3 collections |
| Evaluation set           | `data/eval/test_set.json` | Có | 10 câu hỏi |
| Baseline metrics         | `data/results/baseline_metrics.json` | Có | Hit Rate 1.0 |
| Quality/freshness        | `data/quality/baseline_quality_report.json`, `data/quality/freshness_report.json` | Có | PASS tất cả |
| Baseline report          | `data/reports/phase1_report.md` | Có | Generated từ pipeline |

### Baseline metrics

| Metric                 |       Giá trị | Diễn giải                             |
| ---------------------- | --------------: | --------------------------------------- |
| `retrieval_hit_rate` |          1.000 | Agent tìm đúng document trong top-5 cho cả 10/10 câu hỏi |
| `mean_token_f1`      |          1.000 | Câu trả lời của mock LLM khớp hoàn toàn với ground truth |
| `judge_accuracy`     |          1.000 | Judge đánh giá đúng 100% |
| `mean_judge_score`   |              5 | Điểm trung bình tuyệt đối (thang 5) |
| Ragas                | Skipped (N/A) | Không bật `RUN_RAGAS=1` — Ragas đòi LLM thật |

## 8. Data quality và freshness

### Quality checks

| Check        | Quality dimension | Ngưỡng/kỳ vọng | Kết quả baseline      | Bằng chứng |
| ------------ | ----------------- | ------------------ | ----------------------- | ------------ |
| `expect_table_row_count_to_be_between` | Completeness | min=10, max=100 | PASS — 24 rows | `data/quality/baseline_quality_report.json` |
| `expect_column_values_to_not_be_null` (`paper_id`) | Completeness | 0 null | PASS — 0 unexpected | `data/quality/baseline_quality_report.json` |
| `expect_column_values_to_not_be_null` (`title`) | Completeness | 0 null | PASS — 0 unexpected | `data/quality/baseline_quality_report.json` |
| `expect_column_values_to_be_unique` (`paper_id`) | Uniqueness | 0 duplicate | PASS — 0 unexpected | `data/quality/baseline_quality_report.json` |
| `expect_column_value_lengths_to_be_between` (`summary`) | Validity | min=10 chars | PASS — 0 unexpected | `data/quality/baseline_quality_report.json` |

### Freshness

| Thuộc tính               | Giá trị                           |
| -------------------------- | ----------------------------------- |
| Freshness được đo tại | Clean dataset (`papers_clean.json`) |
| Timestamp mới nhất       | 2026-09-15 (latest published)       |
| Timestamp cũ nhất        | 2026-04-01 (oldest published)       |
| Ngưỡng freshness         | 180 ngày; stale ratio limit 25%     |
| Trạng thái baseline      | PASS — `is_fresh=true`              |
| Lý do                     | 0/24 records có `age_days > 180`; stale_ratio = 0.0 |

## 9. Corruption scenarios và repair

| Corruption         | Cách tạo | Record bị tác động | Quality signal kỳ vọng | Tác động thực tế | Cách repair   |
| ------------------ | ---------- | ---------------------: | ------------------------ | --------------------- | -------------- |
| `drop_latest_records` | Xóa 5 records có `published` mới nhất | 5 | Row count giảm → `expect_table_row_count_to_be_between` FAIL | Row count 24→19 (trước duplicate); góp phần FAIL expectation row count | Tái tạo từ raw snapshot |
| `blank_summary`    | Set `summary=""` cho 2 records | 2 | `expect_column_value_lengths_to_be_between` FAIL | 2 summary rỗng → FAIL length check; retrieval mất context | Tái tạo từ raw snapshot |
| `inject_noise`     | Thêm ký tự rác vào `title`/`summary` | 2 | Embedding shift → retrieval Hit Rate giảm | Góp phần Hit Rate sụt −0.2 | Tái tạo từ raw snapshot |
| `truncate_title`   | Cắt ngắn `title` còn ≤10 ký tự | 2 | Embedding quality giảm; agent trả lời sai | Góp phần Token F1 sụt −0.189 | Tái tạo từ raw snapshot |
| `stale_date`       | Set `published=2024-09-26` cho 6 records | 6 | Freshness FAIL: stale_ratio vượt 25% | stale_rows=8/21 (38.1%) → `is_fresh=false` | Tái tạo từ raw snapshot |
| `duplicate_rows`   | Duplicate 2 records | 2 | `expect_column_values_to_be_unique` FAIL | 4/21 rows là duplicate (19%) → FAIL uniqueness | Tái tạo từ raw snapshot |

Corruption log: `data/results/corruption_log.json` — có, ghi đầy đủ từng scenario, danh sách `paper_ids` bị tác động và `after_rows=21`.

Repair đảm bảo tính toàn vẹn bằng cách tái tạo hoàn toàn từ `data/raw/crossref_records.json` (raw snapshot đáng tin cậy từ bước ingestion), chạy lại toàn bộ cleaning pipeline và re-index ChromaDB. Cách này khác với "vá tay" corrupted rows vì nó đảm bảo repair idempotent và không bị rò rỉ lỗi từ corrupted state.

## 10. So sánh baseline, corrupted và repaired

| Metric/signal            | Baseline | Corrupted | Repaired | Thay đổi do corruption | Mức phục hồi | Nhận xét   |
| ------------------------ | -------: | --------: | -------: | -----------------------: | --------------: | ------------ |
| `retrieval_hit_rate`   |    1.000 |     0.800 |    1.000 |                   −0.200 |          +0.200 | Sụt 20% do mất document và noise; phục hồi hoàn toàn |
| `mean_token_f1`        |    1.000 |     0.811 |    1.000 |                   −0.189 |          +0.189 | Giảm do truncate title và blank summary; phục hồi hoàn toàn |
| `judge_accuracy`       |    1.000 |     0.800 |    1.000 |                   −0.200 |          +0.200 | Judge đánh giá 2/10 sai trên dữ liệu bẩn; phục hồi hoàn toàn |
| `mean_judge_score`     |    5.000 |     4.200 |    5.000 |                   −0.800 |          +0.800 | Điểm trung bình giảm 16%; phục hồi về tối đa |
| Quality checks pass/fail | PASS (5/5) | FAIL (2/5 pass) | PASS (5/5) |          3 expectations FAIL |  +3 expectations | Duplicates và blank summary là 2 lỗi rõ nhất |
| Freshness status         | PASS (0/24 stale) | FAIL (8/21 stale, 38.1%) | PASS (0/24 stale) | stale_ratio 0→0.381 | 0.381→0.0 | `stale_date` corruption đẩy 6 records vượt 180 ngày |

**Kết luận nhân quả có artifact hỗ trợ:**

1. `drop_latest_records` (−5 records) + `inject_noise` + `truncate_title` (gây embedding shift) → `retrieval_hit_rate` giảm từ 1.0 xuống 0.8 (`corrupted_metrics.json`) → agent không tìm được 2/10 document → `judge_accuracy` giảm tương ứng xuống 0.8.

2. Repair từ raw snapshot → re-index `papers-repaired` collection với 24 records sạch → quality checks pass 5/5, `is_fresh=true`, `repaired_metrics.json` phục hồi hoàn toàn về baseline (Hit Rate 1.0, Token F1 1.0). Điều này xác nhận corruption không làm hỏng raw snapshot và repair pipeline hoạt động đúng.

## 11. Vấn đề tích hợp quan trọng

- **Triệu chứng:** `build_test_set` raise `ValueError: At least 10 complete documents are required; found 0` ngay lần chạy đầu tiên.
- **Nguyên nhân:** 24/24 records từ Crossref không có trường `subject` → `categories` và `categories_joined` rỗng → validation loại tất cả candidates vì không đủ điều kiện sinh câu hỏi type `categories`.
- **Cách xử lý:** Trong `build_clean_dataframe`, thêm fallback: nếu category nguồn rỗng → dùng `primary_category` nếu có, nếu vẫn rỗng → gán `"Uncategorized"`. Nhờ đó 24 records đều có `categories_joined` hợp lệ.
- **Cách xác minh:** Chạy lại `build_test_set` → nhận đủ 10 câu hỏi, trong đó 2 câu categories có `ground_truth="Uncategorized"`. Kiểm tra `data/eval/test_set.json`.

## 12. Giới hạn và hướng cải thiện

| Giới hạn hiện tại | Ảnh hưởng   | Hướng cải thiện có thể kiểm chứng |
| --------------------- | -------------- | ----------------------------------------- |
| 0/24 records có category thực (tất cả là `Uncategorized`) | Câu hỏi categories kém đa dạng; benchmark không phân biệt được taxonomy | Ánh xạ `Crossref type` (`journal-article`, `posted-content`, `report`) thành category dự phòng; đo bằng số `categories_joined` distinct |
| Mock LLM — không đo được chất lượng ngôn ngữ thật | `mean_judge_score` không phản ánh năng lực ngôn ngữ | Chạy lại với `LLM_PROVIDER=anthropic` hoặc `google`, so sánh `mean_token_f1` và `judge_accuracy` |
| Ragas bị skip (`RUN_RAGAS=1` không bật) | Thiếu metrics faithfulness và answer relevance theo chuẩn RAGAS | Bật `RUN_RAGAS=1` với LLM thật; đo thêm `context_recall` và `faithfulness` |
| Test set chỉ 10 câu | Confidence interval lớn; 1 câu trả lời sai = 10% Hit Rate drop | Mở rộng lên 30–50 câu; giữ tỷ lệ 4 loại question |

## 13. Checklist trước khi nộp

- [x] Thông tin nhóm và repository chính xác.
- [x] Phân công khớp với module, artifact và kết quả thực tế.
- [x] Lệnh tái hiện đã được chạy lại trên phiên bản dùng để nộp.
- [x] Baseline, corrupted và repaired dùng cùng evaluation set (`data/eval/test_set.json`).
- [x] Bảng metrics khớp với các file trong `data/results/`.
- [x] Quality/freshness conclusions khớp với `data/quality/`.
- [x] Các đường dẫn báo cáo và artifact truy cập được.
- [x] Mỗi thành viên đã hoàn thành báo cáo vai trò riêng.
- [x] Không có `.env`, API key, token hoặc secret trong source, report, log hay ảnh.
