# Đánh giá LLM Judge Phase B

Nhóm: Nhóm 2A202602814

Người phụ trách: Đào Ngọc Bình Thiện - MSSV: 2A202602814

Nguồn: `artifacts/judge_results.json`; run timestamp: 2026-10-12T08:20:42.687891+00:00.
Models generation/judge: generation: None (extractive fallback), judge: gemini-3.5-flash-lite.

## Năm cặp A/B và hai lượt

Nguồn cố định: Candidate A từ `artifacts/answers_50q.json:answer`; Candidate B từ `fixtures/human_labels_10q.json:model_answer`; reference từ `fixtures/test_set_50q.json:ground_truth`.
Các câu hỏi được chỉ định đánh giá: IDs 1, 5, 12, 21, 41. Không thay đổi nguồn sau khi chạy thực nghiệm.

| ID | Câu hỏi | Winner Pass 1 | Winner Pass 2 (quy chuẩn) | Final Winner | Nhất quán vị trí | Điểm Pass 1 (A/B) | Điểm Pass 2 (A/B) |
|---|---|---|---|---|---|---|---|
| 1 | Nhân viên được nghỉ bao nhiêu ngày khi kết hôn? | tie | A | tie | False | 1.0 / 1.0 | 1.0 / 0.8 |
| 5 | Muốn mua thiết bị trị giá 55 triệu cần ai phê duyệt? | tie | tie | tie | True | 0.0 / 0.0 | 0.0 / 0.0 |
| 12 | Thưởng Tết tối thiểu cho nhân viên chính thức có từ 6 tháng trở lên là bao nhiêu? | A | B | tie | False | 1.0 / 0.6 | 0.7 / 1.0 |
| 21 | Một nhân viên Senior có 9 năm thâm niên được nghỉ bao nhiêu ngày phép năm và lương trong khoảng nào? | B | B | B | True | 0.5 / 1.0 | 0.5 / 1.0 |
| 41 | Nhân viên được nghỉ bao nhiêu ngày phép năm? | A | A | A | True | 0.9 / 0.2 | 0.9 / 0.2 |

- Position bias count: 2 / 5 câu (tỷ lệ: 40.0%).
- Verbosity win rate: 50.0% (1 / 2 trường hợp quyết định - decisive cases).
- Decisive cases: 2 câu (ID 21 và ID 41).

### Phân tích hiện tượng bias:
1. **Hiện tượng Position Bias:** Ở ID 1 và ID 12, mô hình LLM Judge thay đổi quyết định khi đảo thứ tự trình bày câu trả lời (Pass 1 vs Pass 2). Cụ thể ở ID 12, khi Answer A đứng trước, mô hình đánh giá cao trích dẫn văn bản của A; nhưng khi đảo vị trí đưa B lên đầu, mô hình lại ưu tiên tính súc tích của B. Điều này chứng minh LLM Judge có xu hướng bị ảnh hưởng bởi thứ tự hiển thị (primacy/recency effect). Cơ chế swap-and-average đã quy kết quả về "tie" khi có xung đột, giúp loại bỏ phán quyết thiên kiến.
2. **Hiện tượng Verbosity Bias:** Trong 2 câu có kết quả quyết định (decisive), 1 câu phần thắng thuộc về câu dài hơn (ID 41: A dài 328 ký tự thắng B 39 ký tự vì B dùng sai chính sách cũ v2023), và 1 câu phần thắng thuộc về câu ngắn hơn (ID 21: B dài 77 ký tự thắng A dài 277 ký tự vì B trả lời đủ cả 2 vế phép năm lẫn dải lương). Tỷ lệ thắng của câu dài hơn là 50.0%, cho thấy mô hình không thiên vị độ dài một cách mù quáng mà vẫn ưu tiên tính đầy đủ và tính chính xác của nội dung.

## Calibration đúng model_answer

Đánh giá 10 câu hỏi hiệu chuẩn trên đúng nội dung `model_answer` từ `fixtures/human_labels_10q.json`, đối chiếu với nhãn `human_label` chuyên gia. Prompt chỉ nhận câu hỏi, câu trả lời cần chấm và văn bản tham chiếu chuẩn; hoàn toàn không gửi trước nhãn hoặc ghi chú của con người.

| ID | Câu hỏi | Model Answer | Judge Label | Human Label | Nhất quán | Diễn giải phán quyết của Judge |
|---|---|---|---|---|---|---|
| 1 | Nhân viên được nghỉ bao nhiêu ngày khi kết hôn? | Nhân viên được nghỉ 3 ngày làm việc có lương khi kết hôn. | 1 | 1 | Khớp | Trả lời chính xác 3 ngày làm việc có lương theo quy định. |
| 5 | Muốn mua thiết bị trị giá 55 triệu cần ai phê duyệt? | Cần Giám đốc phòng ban phê duyệt. | 0 | 0 | Khớp | Sai thẩm quyền; trên 50 triệu phải do Tổng Giám đốc (CEO) phê duyệt. |
| 12 | Thưởng Tết tối thiểu cho nhân viên chính thức có từ 6 tháng trở lên là bao nhiêu? | Nhân viên được thưởng Tết tối thiểu 1 tháng lương. | 1 | 1 | Khớp | Trả lời chính xác mức tối thiểu 1 tháng lương cho nhân viên đủ điều kiện. |
| 21 | Một nhân viên Senior có 9 năm thâm niên được nghỉ bao nhiêu ngày phép năm và lương trong khoảng nào? | Được nghỉ 18 ngày phép (15 cơ bản + 3 thâm niên). Lương Senior từ 20 đến 35 triệu. | 1 | 1 | Khớp | Trả lời đủ và đúng cả số ngày phép lẫn dải lương Senior. |
| 23 | Nhân viên được tài trợ khóa học 25 triệu, nghỉ việc sau 8 tháng. Phải hoàn trả bao nhiêu? | Hoàn trả 25 triệu vì chưa đủ 1 năm cam kết. | 1 | 1 | Khớp | Đúng quy định bồi hoàn 100% chi phí nếu nghỉ việc dưới 12 tháng. |
| 29 | Nhân viên tạm ứng 8 triệu, chưa thanh toán sau 30 ngày. Ai phê duyệt và phí phạt là bao nhiêu? | Trưởng phòng phê duyệt. Phạt 2% tháng trên 8 triệu. | 0 | 0 | Khớp | Thiếu vai trò Kế toán trưởng và tính sai thời gian phạt quá hạn. |
| 33 | Nhân viên Manager có thâm niên 12 năm: tổng phụ cấp hàng tháng và số ngày phép năm? | Phép: 19 ngày. Phụ cấp: 1.500.000 VNĐ/tháng. | 1 | 1 | Khớp | Đúng số ngày phép (15+4) và phụ cấp chức vụ Manager 1.5 triệu/tháng. |
| 41 | Nhân viên được nghỉ bao nhiêu ngày phép năm? | Nhân viên được nghỉ 12 ngày phép năm. | 0 | 0 | Khớp | Dùng dữ liệu chính sách cũ v2023 (12 ngày) thay vì v2024 (15 ngày). |
| 46 | Nhân viên thử việc có được nghỉ phép năm không? | Nhân viên thử việc không được nghỉ phép năm và phải xin nghỉ không lương nếu cần. | 1 | 1 | Khớp | Nắm bắt chính xác điều kiện phủ định của nhân viên thử việc. |
| 50 | Nhân viên Manager có thể dùng VPN cá nhân (NordVPN) khi WFH không? | Được, miễn là đảm bảo kết nối an toàn. | 0 | 0 | Khớp | Sai hoàn toàn; quy định cấm tuyệt đối VPN thương mại của bên thứ ba. |

- Cohen’s κ: **1.0000**
- Tính toán thỏa thuận:
  - Tỷ lệ đồng thuận quan sát được: $P_o = 10 / 10 = 1.0000$ (100%).
  - Tỷ lệ đồng thuận ngẫu nhiên: $P_e = (0.6 \times 0.6) + (0.4 \times 0.4) = 0.36 + 0.16 = 0.5200$.
  - Hệ số Cohen's Kappa: $\kappa = (P_o - P_e) / (1 - P_e) = (1.0 - 0.52) / (1.0 - 0.52) = 1.0000$.
- Nhận xét: LLM Judge đạt độ tương thích hoàn hảo ($\kappa = 1.0$) với chuyên gia con người trên toàn bộ 10 ca kiểm thử hiệu chuẩn, nhận diện chính xác các lỗi sai về thẩm quyền (ID 5), chính sách lỗi thời (ID 41), điều khoản phủ định (ID 46) và vi phạm an ninh thông tin (ID 50).

## Giới hạn và cách dùng judge

Mặc dù hệ số Kappa đạt 1.0 trên tập 10 câu hỏi kiểm chuẩn, cần lưu ý các giới hạn thực tế:
1. **Quy mô tập mẫu còn nhỏ:** 10 câu hỏi hiệu chuẩn và 5 cặp so sánh A/B là bộ mẫu đại diện để phát hiện sai lệch cơ bản, chưa phản ánh toàn diện các trường hợp biên phức tạp.
2. **Không tuyệt đối hóa kết quả Judge:** LLM Judge (gemini-3.5-flash-lite) vẫn bộc lộ Position Bias (40% ở bài toán pairwise). Do đó, trong môi trường sản xuất, bắt buộc phải áp dụng quy trình Swap-and-Average và kết hợp Human-in-the-loop để kiểm tra các trường hợp phán quyết hòa hoặc độ tin cậy thấp.
3. **Chiến lược sử dụng tối ưu:** Dùng LLM Judge làm bộ lọc tự động vòng 1 (triage) để phát hiện câu trả lời sai nghiêm trọng hoặc vi phạm chính sách rõ ràng, đồng thời lưu trữ log phán đoán để kiểm toán định kỳ.
