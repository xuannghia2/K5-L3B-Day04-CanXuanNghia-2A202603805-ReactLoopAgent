# Ghi chú phân tích – Day 04: ReAct Agent Loop

## 1. Mục tiêu

Xây dựng Agent sử dụng vòng lặp ReAct (Model → Tool → Model) để điều tra trạng thái dịch vụ GitHub và Cloudflare.

Agent cần xử lý tool calls, kiểm soát lỗi, giới hạn số bước, lưu trace và cung cấp câu trả lời dựa trên bằng chứng.

## 2. Case được chọn

**Vấn đề:** Agent đưa ra câu trả lời về incident nhưng thiếu URL nguồn được cung cấp bởi tool.

Hai bài kiểm thử liên quan:

- `test_missing_url_gets_revision_within_budget`
- `test_missing_url_at_budget_limit_is_not_completed`

Ở phiên bản baseline, hai bài kiểm thử này thất bại.

## 3. Bằng chứng trước cải tiến

Kết quả kiểm thử baseline:

```text
Ran 37 tests
FAILED (failures=2)
```

Lỗi thứ nhất: Câu trả lời cuối không có URL nguồn đã quan sát, mặc dù tool đã cung cấp URL hợp lệ.

Lỗi thứ hai: Agent trả về trạng thái `completed` dù chưa đủ nguồn dẫn chứng và không còn bước để chỉnh sửa.

**Nguyên nhân:** Agent baseline chưa có bước kiểm tra nguồn dẫn chứng trước khi hoàn thành.

## 4. Giải pháp cải tiến

Bổ sung cơ chế kiểm tra URL nguồn trong `agent.py`.

- Lưu URL từ các tool result thành công vào `observed_urls`.
- Kiểm tra URL đã xuất hiện trong câu trả lời cuối cùng hay chưa.
- Nếu thiếu URL, ghi event `validation_missing_source` vào trace.
- Nếu còn lượt gọi model, yêu cầu model bổ sung URL nguồn.
- Nếu đã hết `max_steps`, trả về trạng thái `max_steps` thay vì `completed`.

Cải tiến này giúp hạn chế việc trả lời về incident mà không dẫn nguồn đã quan sát.

## 5. Kết quả kiểm thử trước và sau

| Chỉ số | Baseline | Improved |
|---|---|---|
| Test chính thành công | 35/37 | 37/37 |
| Test thất bại | 2 | 0 |
| Kiểm tra URL nguồn | Chưa có | Đã có |
| Xử lý thiếu URL khi hết bước | Chưa đúng | Đạt test |
| Bonus offline | Chưa kiểm tra riêng | 3/3 |

Kết quả cho thấy hai lỗi được phát hiện trong kiểm thử baseline đã được khắc phục ở phiên bản improved.

## 6. Đánh giá bằng LLM

**Model dự kiến:** Gemini 3.5 Flash Lite.

**Số case bài chính:** 22 (E1–E6, A1–A8, H1–H8).

Các lượt đánh giá thực tế bằng LLM judge chưa hoàn thành do Gemini API nhiều lần trả về HTTP 429.

Vì vậy, chưa có đủ dữ liệu hợp lệ để báo cáo điểm `eval_before.json`, `eval_after.json`, mức thay đổi điểm tổng hoặc số token sử dụng trong toàn bộ bộ đánh giá.

## 7. Giới hạn

- Các bài kiểm thử offline không chứng minh Agent luôn trả lời chính xác trong thực tế.
- Kiểm tra sự hiện diện URL không chứng minh mọi nhận định đều có nguồn phù hợp.
- Tool chỉ cung cấp dữ liệu trong phạm vi snapshot và giới hạn tìm kiếm.
- HTTP 429 là lỗi giới hạn API, không phải bằng chứng dịch vụ đang gặp sự cố.
- Kết quả đánh giá bằng LLM có thể thay đổi theo model và cấu hình judge.

## 8. Kết luận

Đã triển khai vòng lặp ReAct Agent, xử lý tool calls, lỗi, trace và giới hạn bước.

Cải tiến kiểm tra URL nguồn giúp kết quả kiểm thử offline tăng từ 35/37 lên 37/37. Phiên bản improved cũng vượt qua 3/3 test bonus.

Phần đánh giá trước/sau bằng LLM vẫn cần thực hiện với cùng model, judge, snapshot, dataset và cấu hình số bước để hoàn tất việc so sánh theo yêu cầu bài lab.