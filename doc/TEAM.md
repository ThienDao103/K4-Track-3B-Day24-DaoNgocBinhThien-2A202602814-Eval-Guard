# Nhóm thực hiện K4 Track 3B

Tên nhóm: Team-EvalGuard-VinAI

Repo chung: https://github.com/VinUni-AI20k/K4-L3-DAY24-TenNhom-Eval-Guard

Trưởng nhóm/người nộp: Đào Ngọc Bình Thiện - 2A202602814

Ngày lab/deadline áp dụng: 12/10/2026 23:59:00 — Asia/Ho_Chi_Minh (UTC+07:00)

Commit dùng để chấm: d98f9b4

Mỗi nhóm nộp một repo. Điền đủ thành viên và một mục INDIVIDUAL cho từng người.
Thêm/xóa rows để khớp nhóm thật; mẫu ba rows không ép nhóm phải có ba người.

## Thành viên và phân công

| STT | Họ và tên | MSSV | Vai trò / công việc thật | Bằng chứng |
|---|---|---|---|---|
| 1 | Đào Ngọc Bình Thiện | 2A202602814 | Trưởng nhóm, tích hợp pipeline, thực hiện Phase A/B/C và hoàn thiện báo cáo | Commit d98f9b4, artifacts/answers_50q.json, artifacts/ragas_50q.json, artifacts/judge_results.json, artifacts/guard_results.json |

Người tích hợp/chạy batch/nộp link: Đào Ngọc Bình Thiện, phụ trách toàn bộ pipeline, runner và submission evidence.

## Môi trường và chuẩn bị trước lớp

OS/Python/compiler: Windows 11 / Python 3.13 / MSVC C++ Build Tools.
Virtualenv/dependencies/imports: Đã cài đặt dependencies: ragas, presidio-analyzer, presidio-anonymizer, nemoguardrails, spacy (en_core_web_sm), qdrant-client, rank-bm25, sentence-transformers, flashrank, jsonschema. Chạy pytest 86 passed 100%.
Models/Qdrant/50 answers đã chuẩn bị: Embedding BAAI/bge-m3, Reranker BAAI/bge-reranker-v2-m3, Qdrant in-memory, LLM evaluation & judge qua gpt-4o-mini / gemini API adapter.

## INDIVIDUAL

### DaoNgocBinhThien-2A202602814

- Phần được giao và phần thực tế đã làm: Phụ trách toàn bộ quy trình: Tích hợp 5 module RAG từ Day 18 (m1-m5), triển khai Phase A (RAGAS 4 metric, bottom 10, cluster analysis), Phase B (Pairwise judge, swap-and-average, Cohen's kappa, bias analysis), Phase C (Presidio PII, NeMo input/output rails, adversarial test suite, P95 latency measurement), viết các báo cáo kỹ thuật failure_clusters.md, bias_report.md, blueprint.md.
- Bằng chứng: Các commit trên repo bài nộp, file artifacts/answers_50q.json, artifacts/ragas_50q.json, artifacts/judge_results.json, artifacts/guard_results.json và 3 báo cáo markdown trong doc/reports/.
- Cách kiểm tra: Unit tests qua `python -m pytest tests/ -q` (86 tests passed 100%), thực nghiệm batch live qua `scripts/run_experiments.py`, và nghiệm thu qua `scripts/check_lab.py --with-tests`.
- Điều đã học: Hiểu sâu sắc về các khía cạnh đánh giá pipeline RAG (faithfulness, relevancy, precision, recall), hiện tượng position bias và verbosity bias trong LLM-as-a-Judge, cách đo độ tương đồng Cohen's kappa với nhãn người, thiết kế kiến trúc bảo vệ đa lớp (Presidio PII + NeMo Guardrails) và đo lường percentile latency chuẩn xác.
- Giới hạn/cách xử lý: Xử lý fallback cho các trường hợp không có API key hoặc network timeout; tối ưu hóa tải model SpaCy bằng en_core_web_sm để giảm độ trễ và dung lượng.
- Nguồn/AI hỗ trợ nếu có: Tham khảo tài liệu chính thức từ NeMo Guardrails, RAGAS documentation, Presidio documentation và sử dụng Antigravity coding assistant hỗ trợ pair-programming và rà soát hợp đồng dữ liệu.

## Cập nhật sau khi nộp

Không có cập nhật bổ sung sau deadline.
