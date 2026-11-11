# Ghi chú phân tích · Day 04

Họ và tên: Kiều Văn Thắng
MSSV: 2A202603424

## 1. Case được chọn

- ID case: **A5**
- Câu trả lời của agent: Agent khẳng định “không có incident nào được ghi nhận cho GitHub Actions trong năm 2025”, dù dữ liệu trả về không bao phủ đầy đủ năm 2025.
- Điểm ở lần chạy 1: **45/100**

## 2. Bằng chứng trong trace

Trace cho thấy tool chỉ tìm trong 50 incident gần nhất tại snapshot 09/11/2026. Trace đồng thời cảnh báo rằng kết quả rỗng không chứng minh incident không tồn tại. Dù vậy, câu trả lời vẫn đưa ra kết luận phủ định cho năm 2025. Nguyên nhân có thể là model không tuân thủ đầy đủ giới hạn nguồn trong tool result.

```text
{"event":"tool","step":1,"name":"get_status","arguments":"{\"provider\":\"github\",\"component\":\"Actions\"}",
 "result":{"captured_at":"2026-11-09T03:50:47.065020+00:00",
 "matching_incidents_in_last_50":23,
 "incidents":[
  {"name":"Degraded performance for multiple GitHub services","updated_at":"2026-11-06T00:20:49.369Z"},
  {"name":"Incident with Actions","updated_at":"2026-11-05T17:42:35.455Z"},
  {"name":"Incident with Actions","updated_at":"2026-11-07T01:10:18.506Z"},
  {"name":"Incident with Actions and Pages","updated_at":"2026-11-07T01:04:38.647Z"},
  {"name":"Incident with Pull Requests, Git Operations and Issues","updated_at":"2026-11-04T11:50:06.459Z"}],
 "note":"Chỉ tìm theo chuỗi trong 50 incident gần nhất; không tìm thấy không chứng minh không có sự cố."}}
{"event":"model","step":2,"finish":"stop","tool_calls":0}
Answer: “có thể khẳng định rằng không có incident nào được ghi nhận cho GitHub Actions trong năm 2025.”
```

## 3. Thay đổi đã thực hiện

- Sửa gì, ở đâu: Cập nhật `SYSTEM_PROMPT` trong `agent.py`, yêu cầu agent không suy ra “không có incident” từ tìm kiếm giới hạn; với câu hỏi về năm cụ thể hoặc lịch sử đầy đủ ngoài phạm vi dữ liệu, phải nêu không đủ bằng chứng. Thêm test trong `tests/test_validation.py` để bảo đảm hướng dẫn giới hạn lịch sử được gửi trong system prompt.
- Vì sao chọn thay đổi này: Thay đổi nhắm trực tiếp vào lỗi A5: dữ liệu chỉ gồm 50 incident gần nhất, trong khi câu hỏi yêu cầu danh sách năm 2025. Đây là hướng dẫn cho model, không bảo đảm tuyệt đối mọi model đều tuân thủ; cần đánh giá lại bằng eval.

## 4. So sánh hai lần chạy

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---|---|
| Điểm tổng | 76.27/100 (22/22 case) | 79.55/100 (22/22 case) |
| Điểm case đã chọn | A5: 45/100 | A5: 60/100 |
| Số model calls | 48 | 47 |
| Token, chi phí nếu có | 38,660 tokens; $0.00967005 | 38,113 tokens; $0.00949065 |

- Lỗi đã giải quyết một phần: Điểm A5 tăng 15 điểm. Lần chạy sau nêu rõ không thể lập danh sách đầy đủ từ snapshot 50 incident gần nhất và thừa nhận không thể xác nhận incident năm 2025.
- Lỗi còn lại: Câu trả lời A5 vẫn nói “không có sự cố nào được báo cáo cho năm 2025” dựa trên việc không tìm thấy trong 50 incident gần nhất. Do đó vẫn còn hàm ý phủ định chưa được dữ liệu chứng minh; system prompt cải thiện nhưng chưa loại bỏ hoàn toàn lỗi.
- Nhận xét chung: Điểm tổng tăng 3.28 điểm, số lần gọi model giảm 1; token giảm 547 và chi phí do provider báo cáo giảm $0.00017940. Hai báo cáo có cùng provider/model, judge, bộ dữ liệu và `max_steps=8`; cả hai chấm đủ 22 case.

## 5. Giới hạn

Điểm do LLM judge chấm là ước lượng, không phải bằng chứng chắc chắn rằng agent đúng; cả agent và judge đều dùng `openai/gpt-4o-mini` qua OpenRouter nên có thể có thiên lệch chung. Snapshot GitHub được ghi nhận ngày 09/11/2026 và tool chỉ tìm trong 50 incident gần nhất, không phải kho lịch sử đầy đủ hay trạng thái live; do đó không đủ để xác nhận toàn bộ incident trong năm 2025. Mỗi phiên bản chỉ chạy một lần, nên chênh lệch điểm có thể do biến thiên của model/judge chứ chưa chứng minh cải thiện ổn định. Chi phí/token là số liệu provider ghi trong báo cáo, có thể không phản ánh mọi khoản phí.
