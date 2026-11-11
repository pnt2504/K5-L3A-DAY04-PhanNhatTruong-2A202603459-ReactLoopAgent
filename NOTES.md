# Ghi chú phân tích · Day 04

**Họ và tên** — Phan Nhật Trường

**MSSV** — 2A202603459

## 1. Case được chọn

- **ID case** — H1. Điều tra incident GitHub Actions, đọc incident tổng hợp được liên kết, rồi đối chiếu thời gian, phạm vi ảnh hưởng và bằng chứng về nguyên nhân với incident Cloudflare Workers Builds.
- **Câu trả lời của agent ở lần chạy 1** — Agent báo cả hai incident đã `resolved`, nêu nguyên nhân nội bộ được GitHub công bố và dẫn nguồn cho từng bên. Agent kết luận “Không có bằng chứng trong các chi tiết incident đã truy xuất cho thấy hai sự cố có chung nguyên nhân” và lưu ý bản tóm tắt GitHub bị cắt ngắn. [Câu trả lời đầy đủ trong eval_before.json](eval_before.json)
- **Điểm ở lần chạy 1** — **50/100** sau khi áp trần. LLM judge cho **94/100** trước kiểm tra bằng code.

## 2. Bằng chứng trong trace

Trace của H1 trong `eval_before.json`, chỉ trích các trường cần đối chiếu:

```jsonl
{"step":1,"type":"tool","name":"get_incident_details","arguments":"{\"provider\":\"github\",\"incident_id\":\"h02n9572mx9y\"}","success":true}
{"step":2,"type":"tool","name":"get_incident_details","arguments":"{\"provider\":\"github\",\"incident_id\":\"xr3myzj7ms51\"}","success":true}
{"step":3,"type":"tool","name":"get_incident_details","arguments":"{\"provider\":\"cloudflare\",\"incident_id\":\"yaa0witbl2me\"}","success":true}
```

Trace có ba lần gọi thành công ở ba bước. Tuy nhiên, `grade.mechanical` ghi `tool_calls: 0`, `tool_steps: 0` và báo thiếu `get_incident_details` lẫn số bước gọi tool. Bộ chấm lọc bằng `t.get('event') == 'tool'`. Báo cáo lần 1 ghi `type: "tool"`, còn đoạn code cũ đính kèm ghi `"event  "` có hai dấu cách. Cả hai đều lệch key `event` mà bộ chấm đọc. [Báo cáo lần 1](eval_before.json) · [Code kiểm tra trace](evals/grading.py#L14)

## 3. Thay đổi đã thực hiện

- **Sửa gì, ở đâu** — Trong `run_agent()` của `agent.py`, chuẩn hóa key `event` cho trace và dùng `"event": "tool"` khi thực thi tool. Agent cũng chuyển sang gọi `tools.execute(name, args)` để nhận đúng lỗi và dữ liệu giả lập của từng case. Kiểm tra URL nguồn chỉ áp dụng sau khi đọc thành công chi tiết incident. [agent.py](agent.py)
- **Vì sao chọn thay đổi này** — Trace đã có đủ dữ liệu điều tra H1. Sửa trường đánh dấu event giúp bộ chấm đếm được số lần gọi tool và số bước thực tế. Gọi qua `execute` giúp case như A3 nhận lỗi timeout đã cấu hình.

## 4. So sánh hai lần chạy

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---|---|
| Điểm tổng | 82,86/100 (22/22 case) | 98,82/100 (22/22 case) |
| Điểm H1 sau kiểm tra bằng code | 50/100 | 88/100 |
| Điểm H1 do judge chấm trước khi áp trần | 94/100 | 88/100 |
| Số lần gọi tool được bộ chấm nhận ở H1 | 0, dù trace có 3 lần gọi | 3 |
| Số lần gọi model của agent trên toàn bộ 22 case | 64 | 69 |
| Token của agent trên toàn bộ 22 case | 57.261 | 67.776 |
| Token của judge trên toàn bộ 22 case | 49.567 | 52.126 |
| Tổng token agent và judge | 106.828 | 119.902 |
| Chi phí | Báo cáo không có số tiền | Báo cáo không có số tiền |

Hai báo cáo dùng dataset `core`, provider `openai` và model `cx/gpt-6-luna` cho cả agent lẫn judge, với `max_steps = 8`. Riêng H1 dùng 4 lần gọi model ở cả hai bản, với 4.764 → 4.917 token của agent. [Báo cáo lần 1](eval_before.json) · [Báo cáo lần 2](eval_after.json)

- **Lỗi đã giải quyết** — H1 có 3 tool calls được ghi nhận ở 3 bước và được gỡ trần 50 điểm. A3 tăng từ 46 lên 99. Trace lần 2 cho thấy Cloudflare trả `TimeoutError`, và agent vẫn báo kết quả GitHub đồng thời nói rõ chưa xác minh được Cloudflare.
- **Lỗi còn lại** — Judge trừ điểm H1 ở lần 2 vì cách diễn đạt mốc 17:01 UTC chưa phân biệt rõ thời điểm ghi update với thời gian ảnh hưởng và thời điểm kết thúc sự cố. Tổng điểm tăng 15,96 điểm trong lần so sánh này, còn điểm judge riêng của H1 giảm 6 điểm.

## 5. Giới hạn

- **Điểm judge** — Agent và judge dùng cùng model. Điểm đánh giá nội dung có thể dao động, nên cần đọc lại câu trả lời và trace khi đối chiếu kết quả. Với H1, điểm 50 ở lần đầu đến từ trần do kiểm tra bằng code.
- **Snapshot** — Kết quả phản ánh dữ liệu được chụp ngày 09/11/2026. Danh sách incident có giới hạn và một số cập nhật bị cắt. Dữ liệu này chỉ hỗ trợ kết luận trong phạm vi snapshot và nội dung đã đọc.
- **Số lần chạy** — Mỗi phiên bản có một lượt eval và có nhiều thay đổi giữa hai bản. Trace xác nhận lỗi đếm tool ở H1 đã được sửa, còn mức tăng điểm toàn bộ không thể quy riêng cho một thay đổi hoặc dùng để kết luận độ ổn định.

## Bonus — Retry, phân trang và context

- Agent dùng schema và system prompt của bộ tools bonus để điều tra GitHub, Cloudflare, Vercel, npm và DigitalOcean. Tool trả lỗi có cấu trúc để model sửa tham số hoặc báo phần chưa xác minh.
- Khi một provider trả HTTP 429, agent cho phép tối đa ba lần nhận 429 trong một lượt điều tra. Agent đọc `Retry-After` ở dạng số giây hoặc ngày HTTP, chờ tối đa hai giây cho một lần retry và dừng gọi provider đó nếu thời gian chờ yêu cầu dài hơn. Khi thiếu header, thời gian chờ mặc định là một giây. Các provider khác vẫn được xử lý.
- Với timeline, model nhận `next_offset` và có thể gọi trang tiếp theo với `limit` từ 1 đến 3. Khi câu hỏi cần thêm update, model được hướng dẫn đọc tiếp và ghi rõ phạm vi các trang đã xem.
- Agent bỏ trường `distractors` trong chế độ bonus, ghi lại số mục đã bỏ và giữ lại status, incident, timestamp và URL. System prompt yêu cầu model dùng dữ liệu tool làm bằng chứng và bỏ qua chỉ dẫn bên ngoài.
- Giới hạn: snapshot phản ánh thời điểm thu thập. Kết quả phụ thuộc vào các trang model đã đọc, số bước còn lại và cách model diễn giải dữ liệu. Kiểm tra offline xác nhận cơ chế retry và phân trang với phản hồi model giả lập.
- Eval bonus chấm đủ 10/10 case, đạt **99,40/100**, quy đổi tham khảo **14,91/15**. Trace B3 cho thấy Vercel thành công sau một lần retry. B4 dừng sau ba phản hồi 429 và vẫn có kết quả GitHub. B2 đọc các offset 0, 3 và 6, tìm thấy cập nhật về GitHub-connected builds ở offset 3. B6 ghi `distractors_omitted: 160`. B8 còn nhầm mốc bắt đầu ảnh hưởng với mốc incident, và B9 đưa thêm trạng thái ngoài phạm vi câu hỏi. [Báo cáo bonus](eval_bonus.json)
