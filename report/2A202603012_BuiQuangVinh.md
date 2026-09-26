# Member Role Report — Day 10: Data Pipeline & Data Observability

> Mỗi thành viên trong nhóm tự hoàn thành mẫu này để báo cáo đúng vai trò, phần việc và mức hiểu của mình. Không sao chép nguyên báo cáo chung hoặc báo cáo của thành viên khác. Thay nội dung trong dấu `[ ]` và xóa các dòng hướng dẫn không cần thiết trước khi nộp.

## 1. Thông tin cá nhân

| Thông tin         | Nội dung                  |
| ------------------ | -------------------------- |
| Họ và tên       | Bùi Quang Vinh             |
| MSSV               | 2A202603012                     |
| Khóa/Lớp         | K4-L3B              |
| Tên nhóm         | Agent     |
| Vai trò chính    | Thành viên 4 — Corruption & Integration Owner                 |
| Repository         | K4-L3B-DAY10-Agent-DataPipelineDataObservability |
| Ngày hoàn thành | 2026-09-26               |

## 2. Vai trò và phạm vi công việc

### Phần việc sở hữu

| Module/deliverable | File/hàm phụ trách | Input nhận vào | Output bàn giao  | Trạng thái                                 |
| ------------------ | --------------------- | ---------------- | ----------------- | -------------------------------------------- |
| Corruption Suite | `src/ingestion/corruption.py` — `corrupt_clean_dataframe()` | Clean DataFrame từ baseline | Corrupted DataFrame, `data/results/corruption_log.json` | Hoàn thành |
| Pipeline Integration | `src/pipelines/phase1.py`, `src/pipelines/corruption_flow.py` | Các module ingestion, cleaning, retrieval, evaluation, observability | Baseline pipeline và Corruption → Evaluate → Repair → Compare flow chạy end-to-end | Hoàn thành |

Chỉ nhận ownership cho phần bạn trực tiếp thực hiện. Liên hệ rõ phần việc của bạn với đầu vào, đầu ra và các thành viên phụ thuộc vào phần đó.

### Việc hỗ trợ ngoài phạm vi chính

| Hoạt động                         | Thành viên/module được hỗ trợ | Kết quả                    |
| ------------------------------------ | ------------------------------------ | ---------------------------- |
| Tích hợp các module của TV1, TV2 và TV3 vào luồng chạy chung | `ingestion`, `evaluation`, `observability`, `retrieval` | Đồng bộ input/output giữa các module, đảm bảo baseline, corrupted và repaired dùng cùng evaluation set và sinh đủ artifact |

## 3. Kết quả theo vai trò

| Nhiệm vụ đã thực hiện | File/hàm/artifact liên quan | Kết quả bàn giao       | Cách xác minh         |
| --------------------------- | ----------------------------- | ------------------------- | ----------------------- |
| Xây dựng baseline orchestration | `src/pipelines/phase1.py` | Nối raw → clean → index → test set → evaluate → quality/freshness → report | Chạy `uv run python script/run_phase1.py` và kiểm tra các artifact baseline |
| Xây dựng 6 kịch bản corruption | `src/ingestion/corruption.py`, `data/results/corruption_log.json` | Drop latest records, blank summary, inject noise, truncate title, stale date, duplicate rows | Kiểm tra `corruption_log.json` và corrupted dataset |
| Tích hợp corruption/repair flow | `src/pipelines/corruption_flow.py` | Corrupted metrics giảm, sau repair từ raw snapshot các metrics và quality/freshness phục hồi | Chạy `uv run python script/run_corruption_flow.py` và kiểm tra 3 bộ metrics |

Nêu một output cụ thể mà phần việc của bạn tạo ra hoặc giúp xác minh:

`corruption_flow.py` tạo đầy đủ chuỗi artifact cho ba trạng thái. Baseline đạt Hit Rate/F1 = 1.0/1.0, corrupted giảm còn 0.8/0.8108, sau repair trở lại 1.0/1.0. Đồng thời Quality Gate chuyển PASS → FAIL → PASS và Freshness chuyển PASS → FAIL → PASS.

## 4. Giải thích phần kỹ thuật đã thực hiện

### Vấn đề cần giải quyết

Phần việc của tôi tập trung vào việc nối các module riêng lẻ thành pipeline chạy được end-to-end và tạo một corruption flow có thể chứng minh dữ liệu lỗi ảnh hưởng trực tiếp tới chất lượng RAG. Luồng cần đảm bảo dữ liệu baseline, corrupted và repaired được đánh giá theo cùng một benchmark để so sánh có ý nghĩa, đồng thời bước repair phải khôi phục từ nguồn raw đáng tin cậy thay vì sửa tay dữ liệu đã bị làm bẩn.

### Cách triển khai

Trong `phase1.py`, tôi nối tuần tự các bước load/fetch raw data, cleaning, lưu clean artifacts, build Chroma index, tạo hoặc tái sử dụng evaluation set, chạy evaluation, Quality Gate, Freshness SLA và sinh report baseline.

Trong `corruption.py`, tôi triển khai sáu lỗi dữ liệu theo cách deterministic: xóa khoảng 20% record mới nhất, làm rỗng summary, chèn noise, cắt ngắn title, làm cũ ngày published và duplicate records. Sau khi thay đổi dữ liệu, `age_days`, `summary_chars` và `text_for_embedding` được cập nhật lại để index phản ánh đúng trạng thái corrupted. Mỗi scenario đều được ghi vào `corruption_log.json`.

Trong `corruption_flow.py`, tôi lấy baseline artifacts làm mốc, tạo corrupted dataset rồi build lại index và đánh giá. Sau đó pipeline repair bằng cách đọc lại `crossref_records.json`, chạy lại cleaning từ đầu, build collection repaired, đánh giá lại quality/freshness và metrics, cuối cùng sinh báo cáo so sánh ba trạng thái.

### Input, output và contract

| Thành phần                   | Mô tả                                     |
| ------------------------------ | ------------------------------------------- |
| Input                          | Raw snapshot, clean DataFrame, evaluation set, `Settings` và các module chung của nhóm |
| Output                         | Clean/corrupted/repaired datasets, embeddings, metrics, answers, quality/freshness reports và comparison report |
| Module phụ thuộc             | `ingestion.crossref`, `ingestion.cleaning`, `retrieval.index`, `evaluation.metrics`, `observability.quality`, `observability.reporting` |
| Module sử dụng output        | Script `run_phase1.py`, `run_corruption_flow.py` và phần demo/report cuối |
| Điều kiện lỗi cần xử lý | Thiếu baseline artifact, corrupted DataFrame rỗng, schema không đồng bộ, corruption không đủ mạnh để kích hoạt quality/freshness signal |

### Cách xác minh

```bash
uv run python script/run_phase1.py
uv run python script/run_corruption_flow.py
```

- **Kết quả mong đợi:** Cả hai flow chạy hoàn chỉnh, sinh đủ baseline/corrupted/repaired artifacts; corrupted phải thể hiện suy giảm và repair phải phục hồi.
- **Kết quả thực tế:** Baseline Hit Rate/F1 = `1.0/1.0`; corrupted = `0.8/0.8108`; repaired = `1.0/1.0`. Quality Gate và Freshness đều fail ở corrupted và pass trở lại sau repair.
- **Artifact/log:** `data/results/baseline_metrics.json`, `corrupted_metrics.json`, `repaired_metrics.json`, `corruption_log.json`, `data/reports/corruption_report.md`.

## 5. Một quyết định kỹ thuật quan trọng

- **Bối cảnh:** Sau khi phát hiện corrupted data, cần quyết định repair trực tiếp trên DataFrame lỗi hay tái tạo dataset từ nguồn raw ban đầu.
- **Các phương án đã cân nhắc:** Patch từng row bị corruption; hoặc bỏ trạng thái corrupted và rebuild lại từ raw snapshot.
- **Phương án đã chọn:** Repair bằng cách reload raw records rồi chạy lại `build_clean_dataframe()` và re-index toàn bộ collection.
- **Lý do:** Tránh để lỗi còn sót lại trong corrupted state, giữ data lineage rõ ràng và làm cho repair có tính lặp lại/idempotent.
- **Bằng chứng quyết định phù hợp:** Sau repair, dataset trở lại 24 records, Quality Gate và Freshness PASS, Hit Rate/F1 phục hồi từ `0.8/0.8108` về `1.0/1.0`.

## 6. Một lỗi hoặc blocker đã xử lý

- **Triệu chứng/lỗi nguyên văn:** Corruption flow chạy được nhưng corruption về thời gian chưa làm Freshness SLA fail ổn định, đồng thời `text_for_embedding` sau corruption chưa hoàn toàn đồng bộ với clean schema.
- **Lệnh hoặc bước tái hiện:** Chạy `uv run python script/run_corruption_flow.py` và đối chiếu corrupted freshness report cùng dữ liệu embedding.
- **Nguyên nhân gốc:** Số record bị đổi ngày ban đầu quá ít để stale ratio vượt ngưỡng 25%; phần rebuild embedding text cũng thiếu trường `Published` so với cấu trúc clean data.
- **Cách xử lý:** Điều chỉnh stale corruption lên khoảng 30% số rows bằng `ceil`, tính lại `age_days`, đồng thời thêm `Published` vào `text_for_embedding` sau corruption.
- **Cách xác minh sau khi sửa:** Corrupted có 8/21 rows stale, stale ratio khoảng 38.1% nên Freshness FAIL; repaired trở lại 0/24 stale và PASS.
- **Điều học được:** Corruption test không chỉ cần thay đổi dữ liệu mà phải tạo ra tín hiệu quan sát được và vẫn giữ đúng data contract của pipeline.

## 7. Hiểu biết về luồng end-to-end

Giải thích ngắn gọn bằng lời của bạn:

1. Dữ liệu đi từ Crossref đến vector index như thế nào?
2. Evaluation set và ground-truth document IDs dùng để đo retrieval/answer quality ra sao?
3. Quality checks khác freshness monitoring ở điểm nào trong bài lab?
4. Vì sao phải dùng cùng test set cho baseline, corrupted và repaired?
5. Repair được xem là thành công dựa trên artifact và metric nào?

**Câu trả lời:**

1. Crossref API hoặc raw snapshot được parse thành `PaperRecord`, sau đó cleaning chuẩn hóa thành DataFrame và tạo `text_for_embedding`. MiniLM sinh embedding và ChromaDB lưu vector cùng metadata để phục vụ retrieval.
2. Mỗi câu trong evaluation set có ground truth answer và `ground_truth_doc_ids`. Retrieval Hit Rate kiểm tra document đúng có xuất hiện trong kết quả retrieve hay không; Token F1 và judge metrics đo chất lượng câu trả lời so với ground truth.
3. Quality checks kiểm tra cấu trúc và tính hợp lệ của dataset như số dòng, null, uniqueness và độ dài summary. Freshness tập trung riêng vào tuổi dữ liệu thông qua `age_days`, stale rows và stale ratio.
4. Dùng cùng test set giúp giữ độ khó benchmark cố định. Khi đó thay đổi metrics giữa ba trạng thái có thể quy về trạng thái dữ liệu thay vì do câu hỏi thay đổi.
5. Repair thành công khi reconstructed dataset quay về schema/row count đúng, Quality Gate và Freshness PASS trở lại, đồng thời các metrics repaired phục hồi về gần hoặc bằng baseline.

## 8. Phân tích kết quả

### Metrics chính

| Metric/signal          | Baseline | Corrupted | Repaired | Nhận xét của cá nhân |
| ---------------------- | -------: | --------: | -------: | ------------------------- |
| `retrieval_hit_rate` | 1.0000 | 0.8000 | 1.0000 | Corruption làm mất/biến dạng context khiến 2/10 câu không còn retrieve đúng; repair phục hồi hoàn toàn |
| `mean_token_f1`      | 1.0000 | 0.8108 | 1.0000 | Chất lượng answer giảm theo retrieval/context và trở lại baseline sau repair |
| `judge_accuracy`     | 1.0000 | 0.8000 | 1.0000 | Dữ liệu bẩn làm 2/10 kết quả không đạt; repaired trở lại 100% |
| `mean_judge_score`   | 5.0000 | 4.2000 | 5.0000 | Điểm judge giảm ở corrupted và phục hồi về mức tối đa |
| Quality checks         | PASS | FAIL | PASS | Corrupted fail row count, uniqueness và summary length; repaired pass toàn bộ |
| Freshness status       | PASS | FAIL | PASS | Corrupted có 8/21 stale rows (~38.1%), repaired còn 0/24 stale |

### Kết luận từ số liệu

Hoàn thành hai chuỗi nguyên nhân–bằng chứng sau:

1. Corruption làm mất record, tạo duplicate/summary rỗng và làm cũ published date → Quality Gate/Freshness chuyển sang FAIL → Hit Rate giảm từ `1.0` xuống `0.8`, Token F1 giảm còn `0.8108`.
2. Repair từ trusted raw snapshot → dữ liệu được cleaning và re-index lại → Quality Gate/Freshness trở lại PASS → Hit Rate và Token F1 phục hồi về `1.0`.

Corruption nào ảnh hưởng rõ nhất và vì sao?

Chưa thể kết luận corruption nào ảnh hưởng mạnh nhất vì cả 6 scenario được chạy đồng thời. `drop_latest_records` có cơ chế tác động trực tiếp tới retrieval do có thể loại document khỏi index, còn blank summary/noise/truncate title làm giảm thông tin context; muốn xác định mức ảnh hưởng riêng cần chạy ablation test từng scenario.

Kết quả nào khác với kỳ vọng ban đầu?

Freshness corruption ban đầu chưa đủ mạnh để chắc chắn vượt stale ratio limit. Sau khi kiểm tra threshold thực tế, tôi tăng tỷ lệ record bị stale lên khoảng 30% và đồng bộ lại `age_days`/embedding text; kết quả corrupted sau đó tạo được Freshness FAIL như mong đợi.

## 9. Điều học được và hướng cải thiện

### Ba điều quan trọng nhất

1. Một pipeline nhiều module cần contract rõ giữa các bước; chỉ cần schema hoặc artifact path lệch là toàn bộ flow tích hợp bị ảnh hưởng.
2. Corruption test cần có cả lỗi dữ liệu và observability signal tương ứng, nếu không sẽ không chứng minh được hệ thống phát hiện silent failure.
3. Chất lượng dữ liệu tác động trực tiếp đến retrieval và answer quality; việc repair từ raw source giúp phục hồi cả quality signal lẫn RAG metrics.

### Nếu có thêm thời gian

Tôi sẽ bổ sung pytest/CI cho hai flow end-to-end và chạy ablation test từng corruption riêng biệt để định lượng tác động của từng lỗi lên Hit Rate, Token F1, Quality Gate và Freshness.

## 10. Cam kết của thành viên

Đánh dấu sau khi tự kiểm tra:

- [x] Nội dung báo cáo phản ánh đúng phần việc và mức hiểu của tôi.
- [x] Tôi có thể giải thích luồng end-to-end, không chỉ module mình phụ trách.
- [x] Mọi kết luận về kết quả đều có artifact hoặc metric để đối chiếu.
- [x] Tôi không ghi “đã chạy thành công” cho phần chưa được kiểm chứng.
- [x] Báo cáo không chứa `.env`, API key, token hoặc secret.
- [x] Báo cáo này không phải bản sao nguyên văn của báo cáo nhóm hoặc báo cáo thành viên khác.

**Họ và tên:** Bùi Quang Vinh
**Ngày xác nhận:** 2026-09-26
