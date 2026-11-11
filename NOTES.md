# Ghi chú phân tích – Day 04: ReAct Agent Loop

* **Học viên:** Can Xuan Nghia
* **MSSV:** 2A202603805
* **Repository:** https://github.com/xuannghia2/K5-L3B-Day04-CanXuanNghia-2A202603805-ReactLoopAgent
* **Ngày tổng hợp:** 11/11/2026

## Tóm tắt kết quả

Đã triển khai agent điều tra trạng thái GitHub/Cloudflare theo vòng lặp **model → tool → model**, xử lý tool call, lỗi, giới hạn số bước và lưu trace. Từ phiên bản baseline, bổ sung bước kiểm tra URL dẫn nguồn đã được tool trả về trước khi cho phép kết thúc bình thường.

| Chỉ số | Trước cải tiến (baseline) | Sau cải tiến (improved) |
|---|---:|---:|
| Số case eval bài chính được chấm | 22/22 | 22/22 |
| Case chưa chấm (`unscored`) | 0 | 0 |
| **Điểm đánh giá (0–100)** | **93,41** | **95,68** |
| Test chính offline đạt | 35/37 | 37/37 |
| Test bonus offline đạt | Chưa đo riêng | 3/3 |

**Chênh lệch điểm eval: +2,27 điểm/100.** Đây là điểm đánh giá bộ 22 case, **không phải** điểm chấm bài LAB theo thang 85 điểm hoặc điểm bonus theo thang 15 điểm.

## 1. Case và vấn đề được chọn

**Vấn đề chọn cải tiến:** Agent có thể đưa ra nhận định về một incident sau khi đã tra cứu tool `get_incident_details`, nhưng trả lời cuối không chứa URL nguồn mà tool đã cung cấp. Agent baseline vẫn có thể trả `status = completed` trong tình huống đó.

Hai kiểm thử minh họa trực tiếp:

- `test_missing_url_gets_revision_within_budget`: thiếu URL nhưng còn bước để model sửa.
- `test_missing_url_at_budget_limit_is_not_completed`: thiếu URL khi đã dùng hết ngân sách bước.

**Câu trả lời baseline trong kiểm thử giả lập:** `Incident đã xử lý.`  
**URL mà tool cung cấp:** `https://stspg.io/pjgptg3cfc6n`  
**Nhận xét:** Câu trả lời không trích lại URL đã quan sát, làm yếu khả năng kiểm chứng. Đây là ví dụ kiểm thử offline, **không phải câu trả lời của Gemini trong case eval**.

Ngoài ra, kết quả LLM judge cho thấy hai case cần tiếp tục phân tích là **H2** (50 điểm ở cả hai bản) và **H7** (10 → 55 điểm). Không suy đoán nguyên nhân cụ thể của H2/H7 nếu chưa đọc `grade.reason`, `grade.mechanical`, `answer` và `trace` của chính hai case đó trong file eval.

## 2. Bằng chứng trong trace và nguyên nhân

### 2.1. Bằng chứng kiểm thử offline có thể tái lập

Kịch bản sử dụng `FakeProvider` trong `tests/test_validation.py` và `StatusTools` đọc snapshot cố định. Sau khi gọi `get_incident_details` cho incident `xr3myzj7ms51`, tool trả URL `https://stspg.io/pjgptg3cfc6n`. Model giả lập ban đầu trả `Incident đã xử lý.` mà không có URL.

Tóm tắt trace thực tế khi chạy lại cùng kịch bản kiểm thử trên hai phiên bản code:

```text
BASELINE (max_steps=4):
step 1: event=model, finish=tool_calls
step 1: event=tool, name=get_incident_details, result.url=https://stspg.io/pjgptg3cfc6n
step 2: event=model, finish=stop
status=completed
answer="Incident đã xử lý."

IMPROVED (max_steps=4):
step 1: event=model, finish=tool_calls
step 1: event=tool, name=get_incident_details, result.url=https://stspg.io/pjgptg3cfc6n
step 2: event=model, finish=stop
step 2: event=validation_missing_source,
        missing_urls=["https://stspg.io/pjgptg3cfc6n"]
step 3: event=model, finish=stop
status=completed
answer="Incident đã xử lý. https://stspg.io/pjgptg3cfc6n"
```

Đây là **trace kiểm thử offline với phản hồi model giả lập**, được tái lập từ mã và fixtures của bài; không được trình bày như một trace của LLM judge. Các trace Gemini thật nằm trong `eval_before.json` và `eval_after.json`.

Khi chỉ có `max_steps=2`, baseline trả `completed` dù thiếu URL; phiên bản improved trả `max_steps` thay vì xác nhận hoàn thành không đủ căn cứ. Điều này được kiểm tra bởi `test_missing_url_at_budget_limit_is_not_completed`.

**Nguyên nhân:** Baseline đưa ra quyết định `completed` ngay khi model trả `finish == 'stop'` mà không đối chiếu URL trong câu trả lời với URL đã nhận từ tool thành công. Do đó việc model dừng không đồng nghĩa câu trả lời đã có dẫn chứng.

### 2.2. Bằng chứng từ LLM eval

Kết quả đã ghi trong file JSON của hai lượt đánh giá:

- `eval_before.json`: 22/22 case có điểm, tổng **93,41/100**.
- `eval_after.json`: 22/22 case có điểm, tổng **95,68/100**.
- E1: **95 → 100**; H7: **10 → 55**; H2: **50 → 50**.

Các mức điểm này là bằng chứng về **thay đổi kết quả đánh giá**, nhưng bản thân chúng không chứng minh tính năng kiểm tra URL là nguyên nhân trực tiếp của mọi thay đổi. Để kiểm tra nhận định cụ thể, cần đối chiếu `cases[].grade.reason`, `cases[].grade.mechanical`, `cases[].result.answer` và `cases[].result.trace` trong hai file.

## 3. Thay đổi đã thực hiện

**Trong `agent.py`:**

1. Duy trì cấu trúc ReAct có giới hạn `max_steps`: nối nguyên vẹn assistant message vào history; phân tích JSON arguments; thực thi **mọi** tool call và trả result theo đúng `tool_call_id`; ghi `model`, `tool`, `provider_error` và các sự kiện xử lý lỗi vào trace.
2. Lưu URL trong `observed_urls` khi tool result thành công (`ok == True`) và có trường `url` là chuỗi.
3. Trước khi kết thúc theo `finish == 'stop'`, so sánh những URL quan sát được với câu trả lời cuối.
4. Nếu thiếu, ghi event `validation_missing_source` kèm `missing_urls`; nếu còn budget thì gửi yêu cầu model sửa câu trả lời và dẫn đúng URL; nếu hết budget thì trả `max_steps`.
5. Không yêu cầu URL khi tool lỗi hoặc không có URL nguồn thành công. Không giả tạo incident, URL hay trạng thái dịch vụ.
6. Tương thích với công cụ bonus bằng `tools.schemas` và `tools.system_prompt` khi có; tool error được gửi lại model như observation và hạn chế lặp lại 429 trên cùng lời gọi.

**Trong `providers.py`:** Đã bổ sung retry có giới hạn cho các lỗi API tạm thời như HTTP 429 và 503; có khoảng chờ tăng dần và đọc `Retry-After` khi được cung cấp. Retry nhằm tăng độ bền trước sự cố hạ tầng, **không phải** phương pháp vượt quota hoặc bằng chứng dịch vụ GitHub/Cloudflare bị outage. Thay đổi này không làm thay đổi các file fixtures, scenarios hoặc grading của bài lab.

**Lý do chọn cải tiến:** URL nguồn là bằng chứng có thể kiểm tra trực tiếp; validation có điều kiện rõ ràng, test offline tái lập được và tránh báo `completed` khi chưa đủ URL mà agent đã quan sát.

## 4. So sánh hai lần chạy

### 4.1. Điểm tổng và case

| Chỉ số | `eval_before.json` | `eval_after.json` | Chênh lệch |
|---|---:|---:|---:|
| Tổng số case | 22 | 22 | 0 |
| Case đã chấm | 22 | 22 | 0 |
| Case chưa chấm | 0 | 0 | 0 |
| **Điểm tổng /100** | **93,41** | **95,68** | **+2,27** |
| E1 | 95 | 100 | +5 |
| H2 | 50 | 50 | 0 |
| H7 | 10 | 55 | +45 |
| 19 case còn lại | 100/case | 100/case | 0 |
| Model calls | Chưa tổng hợp từ `usage`/`trace` | Chưa tổng hợp | Chưa kết luận |
| Token/chi phí | Chưa tổng hợp; chi phí không xác định | Chưa tổng hợp; chi phí không xác định | Chưa kết luận |

**Mô hình đánh giá:** Các lệnh eval được chạy với provider `gemini`, model `gemini-3.5-flash-lite`; không thấy cấu hình judge riêng trong các lệnh đã sử dụng, nên theo thiết lập mặc định của runner, judge dùng cùng provider/model. Dữ liệu là bộ 22 case E1–E6, A1–A8, H1–H8 và snapshot/fixtures của bài. Kết quả được chạy từng case, sau đó ghép lại bằng `evals.grading.summarize()`; trong khi đối chiếu cuối cần kiểm tra trường `config` trong cả hai JSON để xác nhận thống nhất `max_steps` và các cấu hình khác.

**Diễn giải:** Bản cải tiến có điểm cao hơn 2,27. H7 tăng đáng kể nhưng vẫn chỉ đạt 55; H2 không thay đổi. Không nên nói toàn bộ 22 case đều cải thiện, cũng không diễn giải điểm `completed` như bằng chứng đáp án chắc chắn đúng.

### 4.2. Kiểm thử offline

```text
Baseline: Ran 37 tests; FAILED (failures=2)  => 35/37 đạt
Improved: Ran 37 tests; OK                   => 37/37 đạt
Improved bonus: Ran 3 tests; OK             => 3/3 đạt
```

**Đã giải quyết rõ ràng:** Hai lỗi thiếu URL nguồn và kết thúc sai ở giới hạn bước theo `tests/test_validation.py`.

**Chưa giải quyết hoàn toàn:** Điểm H2 vẫn là 50 và H7 vẫn là 55 trên LLM eval. Cần đọc trace/judge feedback của từng case để xác định chính xác lỗi còn lại; không khẳng định nguyên nhân từ điểm số đơn lẻ.

### 4.3. Chi phí / usage

Các provider trả `usage` theo từng model call nếu API có hỗ trợ. Báo cáo này **không tự đặt số model calls, số token hoặc chi phí** khi chưa tổng hợp từ hai file JSON. Có thể dùng các trường `cases[].result.usage` và `cases[].result.trace` để kiểm chứng trước khi điền thêm. Chi phí tiền tệ chỉ có thể tính khi biết đơn giá thực tế, loại token và hạn mức/ưu đãi áp dụng.

## 5. Giới hạn và bài học

- **LLM judge chỉ là ước lượng:** Có thể thiên lệch hoặc thiếu nhất quán; khi cùng model làm agent và judge, đánh giá có nguy cơ thiên lệch. Mỗi phiên bản mới chạy một lượt đầy đủ nên chưa chứng minh độ ổn định thống kê.
- **Giới hạn snapshot:** Status tools làm việc với snapshot cố định và một số incident gần đây. Kết quả không đảm bảo phản ánh trạng thái live hay đầy đủ mọi component, khu vực, tenant và timeline.
- **Giới hạn kiểm tra URL:** Việc URL xuất hiện chỉ xác nhận có dẫn nguồn đã quan sát; không bảo đảm URL hỗ trợ tất cả nội dung trong đáp án hoặc lời giải thích là đúng.
- **Partial failure:** Timeout, HTTP 429 hoặc HTTP 503 khi gọi model/tool không có nghĩa GitHub/Cloudflare đang bị sự cố. Hệ thống phải trình bày rõ phần chưa xác minh và giới hạn của dữ liệu.
- **Ngân sách bước:** Sửa câu trả lời để dẫn URL có thể tiêu tốn thêm model call; hết `max_steps` thì không được tự nhận `completed`.
- **Đánh giá theo từng case:** Cả hai bộ được tổng hợp từ các JSON đơn lẻ. Phải bảo đảm cùng model, judge, dataset, snapshot và giới hạn bước; không báo điểm tổng nếu còn case `unscored`.
- **Bonus:** 3/3 test bonus offline đạt, nhưng **chưa có điểm bonus chính thức** nếu chưa chạy đủ 10 case bonus được LLM judge chấm và tạo `eval_bonus.json`.

## 6. Kết luận

Agent đã đáp ứng hợp đồng ReAct loop, có xử lý lỗi và trace, đồng thời bổ sung validation URL nguồn có thể kiểm thử. Kiểm thử offline tăng từ **35/37 lên 37/37**; bộ đánh giá LLM 22 case tăng từ **93,41 lên 95,68/100** (**+2,27**). Điểm tăng là kết quả thực nghiệm quan sát được, không phải bằng chứng quan hệ nhân quả duy nhất. Cần ưu tiên phân tích sâu `H2` và `H7` khi muốn cải tiến tiếp.

**Các file bài chính cần nộp:** `agent.py` (bản improved), `eval_before.json`, `eval_after.json`, `NOTES.md`. Cần bảo đảm không đưa `.env`, API key, token hay phản hồi nhạy cảm vào repository. Giữ nguyên fixtures và bộ chấm của đề.
