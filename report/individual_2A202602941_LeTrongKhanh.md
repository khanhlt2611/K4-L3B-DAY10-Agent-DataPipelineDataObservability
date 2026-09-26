# Member Role Report — Day 10: Data Pipeline & Data Observability

## 1. Thông tin cá nhân

| Thông tin | Nội dung |
| --- | --- |
| Họ và tên | Lê Trọng Khánh |
| MSSV | 2A202602941 |
| Khóa/Lớp | K4-L3B-DAY10 |
| Tên nhóm | Agent |
| Vai trò chính | Data Model & Evaluation-Set Owner |
| Repository | https://github.com/khanhlt2611/K4-L3B-DAY10-Agent-DataPipelineDataObservability |
| Ngày hoàn thành | 2026-09-26 |

## 2. Vai trò và phạm vi công việc

### Phần việc sở hữu

| Module/deliverable | File/hàm phụ trách | Input nhận vào | Output bàn giao | Trạng thái |
| --- | --- | --- | --- | --- |
| Data cleaning và data model | `src/ingestion/cleaning.py::build_clean_dataframe` | `list[PaperRecord]` và `run_date` | DataFrame 16 cột; `data/clean/papers_clean.csv`; `data/clean/papers_clean.json` | Hoàn thành |
| Evaluation set | `src/evaluation/testset.py::build_test_set` | Clean DataFrame và đường dẫn output | `data/eval/test_set.json` gồm 10 câu hỏi | Hoàn thành |

Data model của tôi là contract chung cho các module indexing, observability, corruption và repair. Evaluation set được giữ cố định để nhóm có thể so sánh công bằng ba trạng thái baseline, corrupted và repaired.

### Việc hỗ trợ ngoài phạm vi chính

| Hoạt động | Thành viên/module được hỗ trợ | Kết quả |
| --- | --- | --- |
| Kiểm tra contract dữ liệu với observability | `src/observability/quality.py` | Artifact baseline xác nhận 24 dòng, `paper_id` đầy đủ và duy nhất, summary đạt yêu cầu độ dài |
| Chuẩn bị dữ liệu cho vector index và pipeline | `src/retrieval/index.py`, `src/pipelines/phase1.py` | Các cột `paper_id`, `title`, `published`, `authors_joined`, `categories_joined`, `summary`, URL và `text_for_embedding` có schema ổn định |
| Debug dữ liệu Crossref thiếu category | Cleaning và evaluation | Xác định 0/24 raw records có `subject`; thêm giá trị dự phòng `Uncategorized` để không làm mất toàn bộ evaluation candidates |

## 3. Kết quả theo vai trò

| Nhiệm vụ đã thực hiện | File/hàm/artifact liên quan | Kết quả bàn giao | Cách xác minh |
| --- | --- | --- | --- |
| Chuẩn hóa text và loại JATS/XML | `src/ingestion/cleaning.py` | Title, summary, author và category được trim, chuẩn hóa khoảng trắng; HTML entity được giải mã | Kiểm tra trực tiếp clean JSON và edge-case test |
| Chuẩn hóa ngày và tính độ tuổi | `build_clean_dataframe` | `published`/`updated` ở dạng ISO; `age_days = run_date - published` | Clean artifact có `age_days` từ 11 đến 178 |
| Khử trùng lặp | `build_clean_dataframe` | DOI/paper ID được đối chiếu không phân biệt hoa thường | 24 dòng và 24 paper ID duy nhất |
| Tạo nội dung embedding | `text_for_embedding` | Mỗi document có 5 phần: Title, Authors, Categories, Published, Summary | Kiểm tra `data/clean/papers_clean.json` |
| Sinh benchmark deterministic | `build_test_set` | 10 câu: 3 summary, 3 authors, 2 date, 2 categories | Kiểm tra `data/eval/test_set.json` |
| Validate input evaluation | `build_test_set` | Báo lỗi rõ khi thiếu cột hoặc có dưới 10 document hoàn chỉnh | Chạy test với dataframe thiếu schema và dataframe chỉ có một dòng |

Output cụ thể do phần việc của tôi tạo ra là clean dataset gồm 24 records và evaluation set gồm 10 câu hỏi. Test set có ID ổn định `q001`–`q010`; mỗi câu chứa `ground_truth`, `question_type` và `ground_truth_doc_ids` trỏ tới đúng `paper_id` trong clean dataset.

## 4. Giải thích phần kỹ thuật đã thực hiện

### Vấn đề cần giải quyết

Metadata từ Crossref không đồng nhất: abstract có thể chứa JATS/XML, khoảng trắng thừa, trường ngày có thể không hợp lệ, author/category có thể rỗng hoặc trùng, và DOI có thể khác nhau về chữ hoa/thường. Nếu đưa trực tiếp dữ liệu này vào embedding, chất lượng truy hồi và các quality checks sẽ thiếu ổn định. Ngoài ra, benchmark cần tái lập được và phải khớp cơ chế nhận dạng câu hỏi của QA agent.

### Cách triển khai

Trong `build_clean_dataframe`, tôi giải mã HTML entity, bỏ markup bằng biểu thức chính quy và chuẩn hóa khoảng trắng. Authors và categories được làm sạch, loại trùng nhưng vẫn giữ thứ tự. Record thiếu `paper_id`, `title`, `summary` hoặc ngày xuất bản hợp lệ bị loại; duplicate ID được loại không phân biệt hoa thường.

Ngày được parse về UTC để xử lý được cả `run_date` có và không có timezone. DataFrame cuối giữ ngày ở dạng chuỗi ISO nhằm tương thích với metadata của ChromaDB. Hàm tạo thêm `authors_joined`, `categories_joined`, `summary_chars`, `age_days` và `text_for_embedding`. Nội dung embedding gồm đủ năm nhãn: Title, Authors, Categories, Published và Summary.

Trong `build_test_set`, tôi kiểm tra schema, sắp xếp theo `published` và `paper_id`, rồi chọn 10 vị trí trải đều trên corpus. Cách này deterministic và không phụ thuộc thứ tự dòng đầu vào. Mẫu câu dùng đúng các cụm từ mà `retrieval/qa.py` nhận dạng, đồng thời đặt title trong dấu nháy đơn để QA agent exact-match document trước khi trả lời.

### Input, output và contract

| Thành phần | Mô tả |
| --- | --- |
| Input cleaning | `list[PaperRecord]` gồm ID, title, summary, authors, categories, ngày và URL; cùng `run_date` |
| Output cleaning | DataFrame 16 cột, một dòng trên một `paper_id`, đã sẵn sàng cho quality checks và embedding |
| Input evaluation | Clean DataFrame có `paper_id`, `title`, `summary`, `authors_joined`, `published`, `categories_joined` |
| Output evaluation | Danh sách 10 dictionary và artifact `data/eval/test_set.json` |
| Module phụ thuộc | `ingestion.crossref`, `core.utils` |
| Module sử dụng output | `retrieval.index`, `evaluation.metrics`, `observability.quality`, `pipelines.phase1`, `pipelines.corruption_flow` |
| Điều kiện lỗi cần xử lý | Thiếu cột, ít hơn 10 document hoàn chỉnh, ID trùng, ngày sai, text rỗng và Crossref không có subject/category |

### Cách xác minh

```powershell
python -c "from datetime import datetime, timezone; from core.config import load_settings; from ingestion.crossref import load_raw_records; from ingestion.cleaning import build_clean_dataframe; s=load_settings(); df=build_clean_dataframe(load_raw_records(s.paths.raw_records_json), datetime.now(timezone.utc)); print(f'Clean thành công {len(df)} dòng')"

python -c "from core.config import load_settings; from evaluation.testset import build_test_set; import pandas as pd; s=load_settings(); df=pd.read_json(s.paths.clean_json); ts=build_test_set(df, s.paths.eval_testset); print(f'Sinh được {len(ts)} câu hỏi test')"
```

- **Kết quả mong đợi:** Clean được 24 dòng, không trùng ID và sinh đúng 10 câu hỏi đủ bốn loại.
- **Kết quả thực tế:** 24 clean records, 24 unique IDs; 10 câu gồm 3 summary, 3 authors, 2 date và 2 categories.
- **Artifact/log:** `data/clean/papers_clean.csv`, `data/clean/papers_clean.json`, `data/eval/test_set.json`, `data/quality/baseline_quality_report.json`.

## 5. Một quyết định kỹ thuật quan trọng

- **Bối cảnh:** Evaluation set phải dùng lại cho cả baseline, corrupted và repaired, đồng thời không được thay đổi chỉ vì thứ tự dataframe thay đổi.
- **Các phương án đã cân nhắc:** Chọn ngẫu nhiên 10 paper; lấy 10 dòng đầu tiên; hoặc sort và chọn các vị trí trải đều trên toàn corpus.
- **Phương án đã chọn:** Sort ổn định theo `published`/`paper_id`, sau đó chọn 10 vị trí trải đều.
- **Lý do:** Chọn ngẫu nhiên làm benchmark thay đổi giữa các lần chạy; lấy 10 dòng đầu gây lệch về nhóm bài mới nhất. Chọn trải đều vừa tái lập được vừa đại diện tốt hơn cho corpus mà không cần quản lý random seed.
- **Bằng chứng quyết định phù hợp:** Cùng input nhưng đảo ngẫu nhiên thứ tự dataframe vẫn tạo ra danh sách câu hỏi giống hệt; JSON ghi ra trùng với giá trị hàm trả về.

## 6. Một lỗi hoặc blocker đã xử lý

- **Triệu chứng/lỗi nguyên văn:** `ValueError: At least 10 complete documents are required to build the evaluation set; found 0.`
- **Lệnh hoặc bước tái hiện:** Đọc `papers_clean.json` bằng pandas rồi gọi `build_test_set(df, s.paths.eval_testset)`.
- **Nguyên nhân gốc:** Payload Crossref mới có 24 items nhưng không item nào cung cấp trường `subject`. Vì vậy `categories` và `categories_joined` rỗng ở cả 24 dòng; validation của test-set builder loại tất cả candidates.
- **Cách xử lý:** Trong cleaning, nếu category nguồn rỗng thì dùng `primary_category` nếu có, nếu vẫn rỗng thì gán nhãn minh bạch `Uncategorized`. `primary_category` được đồng bộ với category đầu tiên.
- **Cách xác minh sau khi sửa:** Sinh lại clean artifacts; kiểm tra 24/24 dòng có `categories_joined`; gọi lại `build_test_set` và nhận đúng 10 câu hỏi, trong đó hai câu categories có ground truth `Uncategorized`.
- **Điều học được:** Validation cần phản ánh đặc điểm thực tế của nguồn. Không nên âm thầm bỏ toàn bộ dataset khi một metadata tùy chọn bị thiếu; fallback phải rõ ràng, nhất quán và có thể quan sát được.

## 7. Hiểu biết về luồng end-to-end

1. Crossref API trả raw payload; ingestion lưu nguyên response và chuyển từng item thành `PaperRecord`. Cleaning chuẩn hóa records thành dataframe, tạo các trường dẫn xuất và `text_for_embedding`. Embedding model `all-MiniLM-L6-v2` mã hóa nội dung này, sau đó ChromaDB lưu vector cùng metadata để phục vụ truy hồi.
2. Mỗi evaluation item chứa câu hỏi, đáp án chuẩn và `ground_truth_doc_ids`. Retrieval hit xảy ra khi danh sách document lấy về chứa ít nhất một ID chuẩn. Câu trả lời của agent được so với `ground_truth` bằng Token F1 và judge, nhờ đó tách được chất lượng retrieval khỏi chất lượng trả lời.
3. Quality checks kiểm tra cấu trúc và tính hợp lệ như số dòng, null, uniqueness và độ dài summary. Freshness monitoring tập trung vào thời gian: tính `age_days`, tỷ lệ records quá 180 ngày và cảnh báo nếu tỷ lệ stale vượt 25%.
4. Dùng chung một test set giúp cô lập biến độc lập là trạng thái dữ liệu. Nếu đổi câu hỏi giữa baseline, corrupted và repaired thì chênh lệch metrics có thể đến từ độ khó test set chứ không phải corruption/repair.
5. Repair thành công khi dữ liệu được tái tạo từ raw snapshot, quality/freshness trở lại mức baseline và các metrics trong `repaired_metrics.json` phục hồi gần `baseline_metrics.json`. Việc chỉ sửa report hoặc patch tay corrupted rows không chứng minh được repair idempotent.

## 8. Phân tích kết quả

### Metrics chính

Sau khi pipeline chạy thành công end-to-end, các artifact metrics đã được sinh đầy đủ:

| Metric/signal | Baseline | Corrupted | Repaired | Nhận xét của cá nhân |
| --- | ---: | ---: | ---: | --- |
| `retrieval_hit_rate` | 1.000 | 0.800 | 1.000 | Clean data model của tôi giúp 10/10 câu baseline retrieve đúng; khi 5 records bị xóa và 2 bị noise, 2/10 câu miss — phục hồi hoàn toàn sau repair |
| `mean_token_f1` | 1.000 | 0.811 | 1.000 | Blank summary và truncate title làm context nghèo → F1 sụt 18.9%; repair tái tạo từ raw snapshot phục hồi hoàn toàn |
| `judge_accuracy` | 1.000 | 0.800 | 1.000 | Judge đánh giá đúng 10/10 baseline; corrupted: 8/10; repaired: 10/10 |
| `mean_judge_score` | 5.000 | 4.200 | 5.000 | Điểm trung bình giảm 16% do 2 câu sai hoàn toàn; phục hồi về tối đa |
| Quality checks | PASS (5/5) | FAIL (2/5 pass) | PASS (5/5) | Baseline: tất cả expectations pass trên 24 dòng; corrupted: fail row_count, uniqueness, summary_length |
| Freshness status | PASS (stale=0/24, ratio=0.0) | FAIL (stale=8/21, ratio=0.381) | PASS (stale=0/24, ratio=0.0) | `stale_date` corruption đẩy 6 records vượt 180 ngày, ratio 0→38.1%; repair về 0 |

### Kết luận từ số liệu

1. `drop_latest_records` (−5 records) + `inject_noise` + `truncate_title` (gây embedding shift) → `retrieval_hit_rate` giảm 0.2 và `mean_token_f1` giảm 0.189 (`corrupted_metrics.json`). Đây là chuỗi nhân quả rõ ràng: mất document và ngữ nghĩa embedding bị nhiễu → agent không tìm đủ context → câu trả lời sai.

2. Repair từ raw snapshot (`data/raw/crossref_records.json`) → re-clean → re-index `papers-repaired` → quality/freshness trở về PASS → `repaired_metrics.json` phục hồi hoàn toàn về baseline. Repair idempotent: không patch tay corrupted rows mà tái tạo từ nguồn đáng tin cậy.

Corruption ảnh hưởng rõ nhất là tổ hợp `drop_latest_records` + `inject_noise` + `truncate_title`: ba kịch bản này cùng tấn công chất lượng embedding — giảm số tài liệu trong index và làm lệch vector representation — dẫn đến sụt Hit Rate 20%, đây là chỉ số cốt lõi nhất của retrieval.

Kết quả khác kỳ vọng là dữ liệu Crossref không có category cho cả 24 records. Tôi kiểm tra raw response và xác nhận 0/24 items có `subject`; đây không phải lỗi pandas hay lỗi ghi JSON. Fallback `Uncategorized` giúp pipeline hoàn thành, nhưng làm hai câu categories kém đa dạng. Đây là giới hạn dữ liệu cần nêu rõ thay vì che giấu.

## 9. Điều học được và hướng cải thiện

### Ba điều quan trọng nhất

1. Data contract cần ổn định về tên cột, kiểu dữ liệu và quy tắc null vì một thay đổi nhỏ ở cleaning có thể làm hỏng indexing, observability và evaluation cùng lúc.
2. Validation phải đi kèm khả năng chẩn đoán. Thông báo “found 0” chỉ là triệu chứng; cần kiểm tra số non-null/non-blank theo từng cột mới tìm được category là nguyên nhân gốc.
3. Chất lượng metadata ảnh hưởng trực tiếp tới RAG: trường bị rỗng làm context nghèo đi, ground truth không tạo được và có thể làm benchmark hoặc metric sai lệch dù pipeline không crash.

### Nếu có thêm thời gian

Tôi sẽ phối hợp với Source Owner để ánh xạ `Crossref type` (`journal-article`, `posted-content`, `report`) thành category dự phòng thay cho nhãn chung `Uncategorized`. Cải thiện được đo bằng số category distinct, tỷ lệ records có category có ý nghĩa và Token F1 của nhóm câu hỏi categories, trong khi vẫn giữ nguyên test-set generation deterministic.

## 10. Cam kết của thành viên

- [x] Nội dung báo cáo phản ánh đúng phần việc và mức hiểu của tôi.
- [x] Tôi có thể giải thích luồng end-to-end, không chỉ module mình phụ trách.
- [x] Mọi kết luận về kết quả đều có artifact hoặc metric để đối chiếu.
- [x] Tôi không ghi “đã chạy thành công” cho phần chưa được kiểm chứng.
- [x] Báo cáo không chứa `.env`, API key, token hoặc secret.
- [x] Báo cáo này không phải bản sao nguyên văn của báo cáo nhóm hoặc báo cáo thành viên khác.

**Họ và tên:** Lê Trọng Khánh  
**Ngày xác nhận:** 2026-09-26
