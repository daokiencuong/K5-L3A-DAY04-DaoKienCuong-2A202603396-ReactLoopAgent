# Ghi chú phân tích · Day 04

Họ và tên: Đào Kiên Cường  
MSSV: 2A202603396

## 1. Case được chọn

- ID case: A5
- Câu trả lời của agent: Agent khẳng định không có incident GitHub Actions trong năm 2025, rồi dùng năm incident tháng 11/2026 làm danh sách gần đây. Dữ liệu tool không hỗ trợ kết luận về năm 2025.
- Điểm ở lần chạy 1: 35/100

## 2. Bằng chứng trong trace

Dán đoạn trace chứng minh lỗi và nêu nguyên nhân có thể xảy ra.

```text
step 1: get_status(provider=github, component=Actions)
  matching_incidents_in_last_50 = 23
  note: chỉ tìm trong 50 incident gần nhất; không tìm thấy không chứng minh không có sự cố

step 2: get_incident_details được gọi cho 5 incident
  pfbmpxxfq1cy — created_at 2026-11-05
  t89g2q1800fr — created_at 2026-11-05
  h02n9572mx9y — created_at 2026-11-04
  xr3myzj7ms51 — created_at 2026-11-04
  smtdswd1h2w4 — created_at 2026-11-04

step 3: model finish=stop
  Agent kết luận không có incident Actions trong năm 2025.
```

Các incident chi tiết trong trace đều thuộc tháng 11/2026. Nguyên nhân có thể là model đã ngoại suy từ kết quả gần đây, dù tool ghi rõ giới hạn 50 incident và không có dữ liệu để xác nhận danh sách năm 2025.

## 3. Thay đổi đã thực hiện

- Sửa gì, ở đâu: Trong `agent.py`, nêu rõ giới hạn status search; thêm kiểm tra truy vấn lịch sử đòi danh sách đầy đủ, ghi event `validation_limited_history` và trả lời giới hạn dữ liệu. Bổ sung hướng dẫn phải gọi `get_status` khi được hỏi trạng thái hiện tại; lỗi tool được ghi thêm event `error` cùng `tool_call_id`.
- Vì sao chọn thay đổi này: A5 cho thấy model đưa ra kết luận lịch sử không có căn cứ. Hướng dẫn trạng thái hiện tại xử lý thiếu sót H2 ở lần chạy trước, khi câu trả lời nêu incident đã resolved nhưng bỏ qua snapshot hiện tại.

## 4. So sánh hai lần chạy

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---|---|
| Điểm tổng | 80.82/100 (22/22 case) | 84.09/100 (22/22 case) |
| Điểm case đã chọn | A5: 35/100 | A5: 100/100 |
| Số model calls | 52 | 49 |
| Token, chi phí nếu có | 45,979 tokens (35,244 prompt; 10,735 completion); không có dữ liệu chi phí | 45,574 tokens (37,131 prompt; 8,443 completion); không có dữ liệu chi phí |

- Lỗi đã giải quyết: Với A5, agent không còn khẳng định có hoặc không có incident trong năm 2025 dựa trên tập 50 incident gần nhất; trace có `validation_limited_history`. Điểm A5 tăng từ 35 lên 100. H2 đạt 100 ở cả hai lần; lần sau trace gọi cả `get_status` và `get_incident_details` để trả lời snapshot hiện tại cùng incident lịch sử.
- Lỗi còn lại hoặc mới xuất hiện: H3 (100 → 65) nói ID không có trong hệ thống dù tool chỉ tìm trong 50 incident gần nhất. H7 (65 → 70) báo timeout nhưng sau đó vẫn kết luận Cloudflare operational. E2 (100 → 75) và E1 (65 → 50) cũng giảm; cần đọc lại answer và trace thay vì chỉ dựa vào điểm judge.

### Kết quả eval bonus

Đã chạy eval bonus trên đủ 10 case; cả 10 case đều được chấm, không có case `unscored`. Điểm trung bình là **83.5/100**. Kết quả nằm trong `eval_bonus.json`. Điểm cao nhất là B6 và B8 (100); điểm thấp nhất là B5 (60).

## 5. Giới hạn

Judge là LLM dùng cùng model `gpt-4o-mini` với agent nên điểm có thể biến động hoặc nhận xét chưa khớp hoàn toàn với bằng chứng. Snapshot chỉ đại diện cho thời điểm được ghi nhận; status search giới hạn ở 50 incident gần nhất nên không chứng minh được lịch sử đầy đủ. Mỗi phiên bản và bộ eval bonus chỉ được chạy một lần, vì vậy điểm eval và mức tăng 3.27 điểm chưa chứng minh agent ổn định qua nhiều lần chạy.
