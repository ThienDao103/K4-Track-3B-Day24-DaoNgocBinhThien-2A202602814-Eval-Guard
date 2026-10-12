# Phân tích lỗi Phase A

Nhóm: Nhóm 2A202602814

Người phụ trách: Đào Ngọc Bình Thiện - MSSV: 2A202602814

Nguồn: `artifacts/ragas_50q.json`; run timestamp: 2026-10-12T08:08:06.846357+00:00.
Generation mode/fallback và điều kiện chạy: extractive_fallback dựa trên retrieved contexts từ HybridSearch và CrossEncoderReranker kết hợp, thực thi trong môi trường local in-memory Qdrant.
Models generation/evaluator: generation: None (extractive fallback), evaluator: BAAI/bge-m3 và gpt-4o-mini từ module m4.

## Điểm theo distribution

| Metric | factual (20) | multi_hop (20) | adversarial (10) |
|---|---|---|---|
| faithfulness | 1.0000 | 1.0000 | 1.0000 |
| answer_relevancy | 0.8544 | 0.8170 | 0.8946 |
| context_precision | 0.9259 | 0.9818 | 0.9718 |
| context_recall | 0.9165 | 0.8061 | 0.6803 |
| avg_score | 0.9242 | 0.9012 | 0.8867 |

## Bottom 10 và nguyên nhân

Dưới đây là danh sách 10 câu hỏi có điểm trung bình thấp nhất từ Task 3:

| Rank | ID | Distribution | Question | Avg Score | Worst Metric | Diagnosis | Suggested Fix |
|---|---|---|---|---|---|---|---|
| 1 | 5 | factual | Muốn mua thiết bị trị giá 55 triệu cần ai phê duyệt? | 0.7902 | context_recall | Missing relevant chunks | Improve chunking or add BM25 |
| 2 | 50 | adversarial | Nhân viên Manager có thể dùng VPN cá nhân (như NordVPN) khi WFH để tăng bảo mật thêm không? | 0.8184 | context_recall | Missing relevant chunks | Improve chunking or add BM25 |
| 3 | 21 | multi_hop | Một nhân viên Senior có 9 năm thâm niên được nghỉ bao nhiêu ngày phép năm và lương trong khoảng nào? | 0.8378 | context_recall | Missing relevant chunks | Improve chunking or add BM25 |
| 4 | 33 | multi_hop | Nhân viên Manager có thâm niên 12 năm: tổng phụ cấp hàng tháng và số ngày phép năm theo v2024 là bao nhiêu? | 0.8413 | context_recall | Missing relevant chunks | Improve chunking or add BM25 |
| 5 | 45 | adversarial | Có cần kích hoạt xác thực đa yếu tố (MFA) không? | 0.8490 | context_recall | Missing relevant chunks | Improve chunking or add BM25 |
| 6 | 30 | multi_hop | So sánh quyền lợi bảo hiểm giữa nhân viên thử việc và nhân viên chính thức. | 0.8586 | context_recall | Missing relevant chunks | Improve chunking or add BM25 |
| 7 | 44 | adversarial | Bao lâu phải đổi mật khẩu một lần? | 0.8629 | context_recall | Missing relevant chunks | Improve chunking or add BM25 |
| 8 | 6 | factual | Thông tin lương thuộc cấp độ phân loại dữ liệu nào? | 0.8643 | context_recall | Missing relevant chunks | Improve chunking or add BM25 |
| 9 | 48 | adversarial | Nhân viên thử việc có được hưởng bảo hiểm sức khỏe PVI không? | 0.8695 | context_recall | Missing relevant chunks | Improve chunking or add BM25 |
| 10 | 22 | multi_hop | Nếu cần mua một chiếc laptop 30 triệu cho nhân viên mới, ai phê duyệt và cần gì từ phòng CNTT? | 0.8715 | answer_relevancy | Answer doesn't match question | Improve prompt template |

### Phân tích chi tiết các case đại diện:

- **Câu 5 (factual, rank 1, score 0.7902):** Câu hỏi liên quan đến phân cấp hạn mức chi tiêu 55 triệu VNĐ (thuộc thẩm quyền Giám đốc khối do vượt mốc 50 triệu của Manager). Do văn bản quy định tài chính được chia nhỏ theo kích thước chunk 256 tokens, bảng ma trận phân quyền bị ngắt quãng giữa các chunk, khiến bộ tìm kiếm chỉ lấy được đoạn quy định về cấp Manager mà bỏ sót điều khoản cấp Giám đốc khối, dẫn đến `context_recall` thấp.
- **Câu 50 (adversarial, rank 2, score 0.8184):** Câu hỏi gài bẫy về việc sử dụng VPN thương mại cá nhân (NordVPN). Chính sách an toàn thông tin công ty quy định bắt buộc chỉ sử dụng WireGuard nội bộ và nghiêm cấm tuyệt đối mọi dịch vụ VPN bên ngoài. Khi truy vấn dense matching, từ khóa "VPN", "bảo mật", "WFH" kéo về các đoạn mô tả kết nối làm việc từ xa chung chung thay vì kéo về đúng đoạn quy định chế tài cấm VPN bên thứ ba, làm suy giảm `context_recall`.
- **Câu 21 (multi_hop, rank 3, score 0.8378):** Câu hỏi đòi hỏi tổng hợp từ hai tài liệu độc lập: chính sách phép năm theo thâm niên (HR Policy) và khung thang bảng lương cho cấp bậc Senior (Compensation Policy). Với top-3 reranker hiện tại, pipeline chỉ lấy đủ thông tin về ngày nghỉ phép nhưng chỉ lấy được một phần thông tin về dải lương, làm giảm độ bao phủ ngữ cảnh `context_recall`.

## Matrix và nhóm yếu nhất

Ma trận phân bổ worst-metric theo từng phân bố dữ liệu (mỗi câu hỏi chỉ gán cho 1 metric có điểm thấp nhất):

| Metric | factual (20) | multi_hop (20) | adversarial (10) | Tổng |
|---|---|---|---|---|
| faithfulness | 0 | 0 | 0 | 0 |
| answer_relevancy | 11 | 13 | 1 | 25 |
| context_precision | 4 | 0 | 0 | 4 |
| context_recall | 5 | 7 | 9 | 21 |

- Nhóm yếu theo mean thấp nhất: **adversarial**; mean score: **0.8867** (thấp hơn multi_hop 0.9012 và factual 0.9242).
- Worst metric xuất hiện nhiều nhất: **answer_relevancy**; số câu: **25 câu** (factual 11, multi_hop 13, adversarial 1).

### Giải thích nguyên nhân và đề xuất cải tiến:

1. **Nguyên nhân nhóm adversarial yếu nhất:** Điểm `context_recall` của nhóm adversarial tụt sâu xuống mức 0.6803 (trong khi factual đạt 0.9165). Các truy vấn mang tính phủ định hoặc giả định sai thường bị mô hình embedding đánh đồng ngữ nghĩa với câu khẳng định tương đương, dẫn đến việc thiếu các đoạn tài liệu mang tính ràng buộc hoặc phủ định.
2. **Nguyên nhân answer_relevancy chiếm đa số worst-metric:** Khi trích xuất trực tiếp các đoạn văn bản từ context, câu trả lời đảm bảo tính chính xác tuyệt đối (faithfulness 1.0) nhưng mang tính nguyên văn và chứa các câu dẫn không liên quan trực tiếp đến trọng tâm câu hỏi.
3. **Đề xuất khắc phục cụ thể:**
   - Điều chỉnh tham số retrieval: Nâng `RERANK_TOP_K` từ 3 lên 5 để gom đủ ngữ cảnh đa chiều cho các câu hỏi multi-hop và adversarial.
   - Nâng cấp kỹ thuật chunking: Áp dụng parent-document retriever hoặc mở rộng overlap chunking trong `m1_chunking.py` lên 64 tokens để giữ trọn vẹn các bảng biểu ma trận phân quyền.
   - Thử nghiệm lại: Chạy lại lệnh `python scripts/run_experiments.py --phase a --live` và theo dõi sự gia tăng của `context_recall` trên nhóm adversarial (mục tiêu nâng lên >0.80).

## Bẫy adversarial và giới hạn

So sánh với factual và multi_hop, các câu hỏi trong tập adversarial (như ID 44, 45, 48, 50) chứa các giả định sai lầm (như việc đòi hỏi quyền lợi bảo hiểm PVI cho nhân viên thử việc, hay dùng VPN thương mại). Điểm số trung bình của nhóm này thấp hơn các nhóm khác cho thấy pipeline gặp khó khăn trong việc thu thập đủ chứng cứ để bác bỏ các khẳng định sai lệch. Điểm số thấp hơn không tự chứng minh pipeline đã nhận ra bẫy, mà phản ánh sự thiếu hụt ngữ cảnh quy định cụ thể về điều kiện loại trừ.

Giới hạn thực nghiệm: Bộ kiểm thử Phase A gồm 50 câu hỏi trên tập tài liệu nội bộ. Pipeline thực hiện trên môi trường hybrid retrieval và extractive fallback, do đó việc đánh giá ngữ nghĩa chủ yếu dựa trên các độ đo embedding của m4_eval (BAAI/bge-m3).
