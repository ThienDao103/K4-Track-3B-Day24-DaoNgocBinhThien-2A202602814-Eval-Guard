# Blueprint CI/CD cho RAG và guardrails

Nhóm: Nhóm 2A202602814

Người phụ trách: Đào Ngọc Bình Thiện - MSSV: 2A202602814

Đây là **đề xuất CI/CD**, không phải workflow đã được triển khai/chạy.
Nguồn kết quả: `artifacts/ragas_50q.json`, `artifacts/judge_results.json`, `artifacts/guard_results.json`.
Run timestamp: 2026-10-12T08:32:26.041853+00:00; guard models: gemini-3.5-flash-lite.
Models generation/evaluator/judge từ A/B:
- Generation: None (extractive fallback từ answers_50q.json).
- Evaluator: BAAI/bge-m3 và gpt-4o-mini (từ ragas_50q.json).
- Judge: gemini-3.5-flash-lite (từ judge_results.json).
- Guard: gemini-3.5-flash-lite (từ guard_results.json).

## Kiến trúc và failure action

Luồng xử lý dữ liệu chuẩn sản xuất:
`Input → Layer 1: Presidio PII Scan → Layer 2: NeMo Input Rails → Layer 3: RAG Core → Layer 4: NeMo Output Rails → Client Response`.

Chi tiết kiến trúc và hành động khi phát hiện lỗi (Failure Actions):
1. **Tầng 1 - Presidio PII Detection (Input Guard):**
   - Quét các thực thể định danh cá nhân nhạy cảm gồm số CCCD (12 số), CMND (9 số), số điện thoại Việt Nam và địa chỉ email.
   - *Failure action:* Nếu câu hỏi chứa PII injection nguy hại (như CCCD/SĐT người khác), hệ thống lập tức ngắt mạch (short-circuit), từ chối xử lý và trả về mã lỗi bảo mật. Nếu câu hỏi hợp lệ của nhân viên cần ẩn danh, Presidio thực hiện masking thành các thẻ `<VN_CCCD>`, `<VN_PHONE>` trước khi chuyển tiếp.
2. **Tầng 2 - NeMo Guardrails Input Rail (Topic & Jailbreak Guard):**
   - Kiểm tra hành vi prompt injection, jailbreak (DAN pattern, đóng vai quản lý), câu hỏi lạc đề (thơ ca, tiền số, giải toán) và yêu cầu truy xuất dữ liệu mật.
   - *Failure action:* Ngắt mạch ngay lập tức (short-circuit), không kích hoạt hệ thống RAG tốn kém, trả về câu từ chối chuẩn: *"Tôi là trợ lý chính sách công ty, tôi chỉ hỗ trợ các câu hỏi liên quan đến chính sách nhân sự và công nghệ thông tin nội bộ."*
3. **Tầng 3 - RAG Core Engine:**
   - Thực thi Hybrid Retrieval (BM25 kết hợp BGE-M3 Dense Search) trên Qdrant vector database, sắp xếp lại qua Cross-Encoder Reranker để chọn lọc top-k chunks phù hợp nhất.
4. **Tầng 4 - NeMo Guardrails Output Rail (Data Leakage Guard):**
   - Kiểm tra câu trả lời sinh ra trước khi gửi về client, ngăn chặn việc rò rỉ dữ liệu mật nội bộ (mật khẩu admin, khóa bảo mật, dữ liệu cá nhân của đồng nghiệp).
   - *Failure action:* Redact và thay thế toàn bộ phản hồi bằng thông điệp an toàn theo `rails.co` (*"Xin lỗi, tôi không thể cung cấp thông tin này do chính sách bảo mật nội bộ"*).
5. **Quản lý Secrets và Logging bảo mật:**
   - Toàn bộ API keys (Google Gemini, OpenAI, Qdrant) được lưu trữ an toàn trong Secret Manager hoặc biến môi trường runtime, không bao giờ commit vào mã nguồn git.
   - Hệ thống logging tập trung ghi nhận metadata, trạng thái kích hoạt guardrails và độ trễ nhưng bắt buộc đã anonymize toàn bộ văn bản để tuân thủ quyền riêng tư dữ liệu.
   - Điều kiện fallback: Nếu dịch vụ guardrail LLM gặp sự cố (HTTP 503/429 hoặc timeout >2.5s), hệ thống tự động kích hoạt fail-closed đối với các mẫu kiểm tra an ninh cơ bản (regex/heuristics) để bảo vệ hệ thống.

## Phạm vi và số đo latency

Bắt buộc Task 12: Đo phạm vi **input_layers_sequential** (Presidio + NeMo input tuần tự, không ngắt mạch short-circuit nhằm đo lường trọn vẹn chi phí an ninh đầu vào). Công cụ đo được khởi tạo sẵn; thực hiện 2 lượt warm-up thực tế (được loại trừ khỏi mẫu thống kê) và lấy chính xác 20 mẫu đo lường thời gian độc lập. Các phân vị được tính toán bằng phương pháp nội suy tuyến tính (linear interpolation).

Số mẫu timed: 20; warm-up: 2; phương pháp: linear.

| Layer / scope | P50 ms | P95 ms | P99 ms | Tình trạng |
|---|---|---|---|---|
| Presidio | 5.76 | 7.95 | 8.38 | Đạt ngân sách |
| NeMo input | 687.48 | 1140.07 | 5845.57 | Vượt ngân sách cloud LLM |
| Input guard sequential | 693.04 | 1146.30 | 5851.88 | Vượt ngân sách cloud LLM |
| RAG | chưa đo | chưa đo | chưa đo | Ngoài phạm vi bắt buộc |
| NeMo output | chưa đo | chưa đo | chưa đo | Ngoài phạm vi bắt buộc |
| Full guard / short-circuit request | chưa đo | chưa đo | chưa đo | Không suy từ input_guard_ms |

- Ngân sách trễ đầu vào mục tiêu (Input-layer budget target): **500 ms**.
- Kết quả thực tế: P95 = **1146.30 ms**; `latency_budget_ok`: **False**.

### Phân tích Bottleneck và kế hoạch tối ưu:
- **Nguyên nhân vượt ngưỡng:** Tầng Presidio chạy cục bộ trên CPU xử lý cực nhanh (P50 chỉ 5.76 ms, P95 chỉ 7.95 ms, chỉ chiếm dưới 1% tổng thời gian). Điểm nghẽn tuyệt đối nằm ở NeMo Input Rail gọi Cloud LLM qua mạng internet (P95 đạt 1140.07 ms) do độ trễ khứ hồi mạng (network RTT) và thời gian chờ sinh token trên hạ tầng đám mây.
- **Kế hoạch tối ưu đưa P95 < 500 ms:**
  1. Thay thế Cloud LLM ở tầng NeMo bằng mô hình ngôn ngữ nhỏ chạy cục bộ (như Llama-Guard-3-1B hoặc Qwen-2.5-0.5B tối ưu hóa ONNX/TensorRT-LLM) phục vụ trực tiếp trên GPU máy chủ nội bộ. Độ trễ dự kiến giảm xuống còn 40 - 80 ms.
  2. Bổ sung tầng Semantic Cache bằng Redis/Qdrant cho các truy vấn phổ biến để bỏ qua bước gọi LLM guardrail đối với các câu hỏi đã được xác thực an toàn trước đó.
  3. Kế hoạch đo lường RAG và NeMo Output: Bổ sung bộ đếm thời gian phân đoạn cho các thành phần chưa đo trong giai đoạn thử nghiệm tải tiếp theo với 100 mẫu truy vấn ngẫu nhiên.

## CI gates đề xuất

Bộ tiêu chí kiểm soát chất lượng (Quality Gates) đề xuất trong pipeline tích hợp liên tục:
- **Gate 1 - RAGAS Quality:** Faithfulness đạt ≥ 0.75 và avg_score đạt ≥ 0.65 trên tập 50 câu hỏi chuẩn hóa (trong thực nghiệm đạt avg_score = 0.9040).
- **Gate 2 - Guardrail Security:** Tấn công bị chặn đạt ≥ 18/20 câu (trong thực nghiệm đạt tuyệt đối 20/20 câu, tỷ lệ 100%); Tỷ lệ chặn nhầm (False Positive Rate) trên tập câu hỏi sạch đạt 0% (8/8 câu được cho phép); 100% ca kiểm thử đầu ra (output_set_4) cho kết quả chính xác.
- **Gate 3 - LLM Judge Calibration:** Hệ số Cohen's Kappa đạt ≥ 0.60 so với nhãn chuyên gia (trong thực nghiệm đạt 1.0000).
- **Gate 4 - Latency Enforcement:** Ngưỡng P95 < 500 ms sẽ được kích hoạt làm blocking gate sau khi hoàn tất triển khai mô hình Local Small Guardrail.

Lệnh thực thi kiểm tra tự động trong CI Job:
```bash
python -m pytest tests/ -q
python scripts/check_lab.py --with-tests
```
Nếu bất kỳ điều kiện nào không thỏa mãn, runner trả về exit code 1 và hủy quy trình triển khai nhằm ngăn chặn lỗi đưa lên môi trường Staging/Production.

## Monitoring và cải tiến

Quy trình giám sát hệ thống trong môi trường Production:
1. **Chỉ số giám sát và cảnh báo thời gian thực:**
   - Tỷ lệ chặn đầu vào (Block rate) vượt quá 15% tổng lưu lượng trong 10 phút: Cảnh báo nguy cơ bị tấn công diện rộng hoặc prompt drift.
   - Bất kỳ trường hợp False Positive nào trên tập golden dataset được phát hiện: Kích hoạt alert mức độ nghiêm trọng (P1) cho đội ngũ AI Engineer.
   - P95 latency vượt quá 1500 ms trong 5 phút liên tục: Kích hoạt tự động mở rộng tài nguyên (auto-scaling).
2. **Số liệu thực nghiệm baseline từ hệ thống:**
   - RAGAS mean score: 0.9040; phân bố yếu nhất: adversarial (mean score: 0.8867).
   - Cohen’s κ: 1.0000 (độ tương đồng tuyệt đối với nhãn chuyên gia).
   - Tấn công bị chặn: 20/20 câu (100%); tỷ lệ dương tính giả (clean false positive rate): 0.0000 (0%).
3. **Các hành động cải tiến cụ thể có thể kiểm chứng:**
   - Triển khai Local Small Guard Model để giảm P95 latency tầng guardrail xuống dưới 150 ms.
   - Tăng cường mẫu regex nhận diện CCCD có định dạng đặc biệt (có dấu cách hoặc dấu gạch nối) trong `m9_guard`.
   - Nâng số lượng chunks lấy lại sau rerank (`RERANK_TOP_K`) từ 3 lên 5 để cải thiện `context_recall` cho nhóm câu hỏi adversarial từ 0.6803 lên > 0.85.
   - Xây dựng dashboard trực quan hóa trên Grafana hiển thị real-time số lượng request bị chặn theo từng danh mục (PII, Jailbreak, Off-topic).
