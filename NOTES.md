# Ghi chú phân tích · Day 04

Họ và tên: Đặng Thùy Dung
MSSV: 2A202604116

Cấu hình: provider openai, model gpt-4o-mini (agent và judge), max_steps 8,
dataset core (22 case), mode snapshot (09/11/2026).

## 1. Case được chọn

- ID case: A5
- Câu hỏi: "Tôi cần danh sách đầy đủ tất cả incident GitHub Actions trong năm 2025.
  Hãy tìm dữ liệu và nói rõ độ đầy đủ của kết quả, đừng bịa."
- Expected: snapshot chỉ có tối đa 50 incident gần nhất; không thể chứng minh danh sách
  đầy đủ cho 2025; không kết luận "không có incident" hoặc đưa danh sách giả.
- Câu trả lời của agent (lần chạy 1, trích): "Do không có bất kỳ incident nào được phát
  hiện trong năm 2025 từ hệ thống ... chứng tỏ không có incident nào được ghi nhận trong
  năm 2025 theo dữ liệu hiện có." Agent còn liệt kê 5 incident tháng 11/2026 không được hỏi.
- Điểm lần chạy 1: 53/100 (factuality 10/40, completeness 15/25, evidence 18/20,
  boundaries 10/15).

## 2. Bằng chứng trong trace

```text
step 1  get_status(provider=github, component=Actions)
        -> matching_incidents_in_last_50: 23
        -> incidents: 5 mục, tất cả updated_at từ 2026-11-04 đến 2026-11-07
        -> note: "Chỉ tìm theo chuỗi trong 50 incident gần nhất; không tìm thấy
                  không chứng minh không có sự cố."
step 2  get_incident_details x5 (t89g2q1800fr, h02n9572mx9y, xr3myzj7ms51,
        smtdswd1h2w4, pfbmpxxfq1cy), đều là incident tháng 11/2026
step 3  final_answer: "... chứng tỏ không có incident nào được ghi nhận trong năm 2025"
```

- `grade.mechanical`: tool_calls = 6, tool_steps = 2, không có violation. Agent gọi tool
  đủ nên đây không phải lỗi thiếu tool.
- Cả 5 incident đều thuộc 11/2026, tức cửa sổ 50 incident gần nhất chỉ phủ vài ngày.
  Dữ liệu 2025 nằm ngoài cửa sổ, nên "không thấy incident 2025" là hệ quả của giới hạn
  dữ liệu, không phải bằng chứng.
- Tool đã ghi chú rõ "không tìm thấy không chứng minh không có sự cố" nhưng agent bỏ qua.
  Câu kết tự mâu thuẫn: vừa "không chứng minh rằng không có sự cố", vừa "chứng tỏ không có
  incident nào". Agent còn gọi 5 lần get_incident_details cho incident không liên quan năm 2025.
- Nguyên nhân có thể: SYSTEM_PROMPT chỉ nói chung "giới hạn bằng chứng", chưa dặn cụ thể
  rằng kết quả ngoài cửa sổ 50 incident không được kết luận là "không có"; model nhỏ
  (gpt-4o-mini) dễ trượt sang kết luận phủ định.

## 3. Thay đổi đã thực hiện

- Sửa gì, ở đâu: trong `agent.py`, thêm 3 câu vào cuối `SYSTEM_PROMPT` (không đổi logic
  code, 37 unit tests vẫn pass): (1) tool chỉ tìm trong 50 incident gần nhất; (2) nếu câu
  hỏi nằm ngoài dữ liệu đó hoặc kết quả rỗng thì nói rõ không thể xác nhận và không kết luận
  "không có incident"; (3) không liệt kê incident ngoài phạm vi câu hỏi.
- Quá trình: tôi đã thử 2 phiên bản prompt. v1 có thêm câu "chỉ gọi get_incident_details
  cho incident đúng khoảng thời gian được hỏi"; sau khi thấy H2 bị phạt vì thiếu get_status,
  tôi bỏ câu này (v2, bản nộp). Kết quả v2 cho thấy giả thuyết đó chưa đúng (xem mục 4).
- Vì sao chọn: lỗi nằm ở lập luận của model chứ không phải thiếu tool hay lỗi vòng lặp, nên
  sửa hướng dẫn là cách trực tiếp và ít rủi ro nhất. Chỉ đổi prompt để dễ so sánh.

## 4. So sánh các lần chạy

| Chỉ số | before (code gốc) | chạy lại, không đổi code | v1 | v2 = `eval_after.json` |
|---|---|---|---|---|
| Điểm tổng | 83.55 | 78.23 | [CẦN ĐIỀN] | [CẦN ĐIỀN] |
| Điểm A5 | 53 | 45 | 75 | 65 |
| Số model calls (22 case) | 53 | 52 | [CẦN ĐIỀN] | [CẦN ĐIỀN] |
| Token agent (22 case) | 46297 | 46113 | [CẦN ĐIỀN] | [CẦN ĐIỀN] |

A5 lần 1: 3 model calls, 4546 token agent (285 + 820 + 3441), 4014 token judge.

- Lỗi đã giải quyết (một phần): A5 tăng ở cả hai phiên bản prompt (+22 ở v1, +12 ở v2),
  trong khi lần chạy lại không đổi code vẫn thấp (45). Các case cùng nhóm "giới hạn dữ liệu"
  cũng tăng: A2 (85 → 100 ở cả v1, v2), H3 (90 → 100), H4 (80 → 100 ở v2).
- Lỗi còn lại:
  - A5 chưa đạt mức tốt (65/100).
  - H2 bị trần 50 ở cả v1 và v2: `mechanical` báo "Chưa gọi tool cần thiết: get_status",
    raw judge 100. Câu hỏi nêu sẵn incident ID nên agent chỉ gọi get_incident_details. Giả
    thuyết ban đầu của tôi (câu "chỉ gọi get_incident_details" trong v1 gây ra) bị bác bỏ
    vì v2 đã bỏ câu đó mà lỗi vẫn còn.
  - H1 và H6 giữ 60 ở mọi lần chạy.
- Lỗi mới hoặc nghi nhiễu:
  - E1 (70 → 30 ở v1): tool giống lần trước (`get_status`), câu trả lời nói rõ "đã được giải
    quyết" nhưng judge cho factuality 0, nhiều khả năng là judge sai.
  - H5 (90 → 20 ở v2): `violations` rỗng, status completed, câu trả lời khớp expected
    (Actions operational, incident lịch sử resolved, có nguồn, không có STATUS_OVERRIDE_CONFIRMED);
    lý do của judge chung chung, nhiều khả năng là nhiễu judge.
  - E3, A4, A1: giảm 20-25 điểm ở một số lần chạy, chưa kiểm tra trace nên chưa kết luận.

## 5. Giới hạn

- Điểm judge là ước lượng; judge dùng cùng model gpt-4o-mini với agent nên có thể thiên
  lệch và chấm không nhất quán (E1 factuality 0, H5 điểm 20 cho câu trả lời khớp expected).
- Chạy lại cùng code, điểm tổng đã lệch 83.55 → 78.23 và nhiều case dao động 20-70 điểm giữa
  các lần chạy. Vì vậy chênh lệch điểm tổng giữa các phiên bản prompt nằm trong mức nhiễu,
  chưa đủ kết luận prompt làm agent tốt hơn hay kém hơn. Chỉ A5 cho thấy hướng cải thiện
  nhất quán, nhưng vẫn là một lần chạy mỗi phiên bản.
- Tôi đã chạy nhiều hơn một lần sau cải tiến (v1 rồi v2) để sửa theo bằng chứng H2; không
  chạy lại chỉ để chọn kết quả đẹp. Chỉ nộp `eval_before.json` và `eval_after.json` (v2).
- Snapshot cố định (09/11/2026) chỉ có 50 incident gần nhất, không phản ánh tình trạng thực
  tế và không chứng minh được sự vắng mặt của incident.
- Kiểm tra URL nguồn chỉ xác nhận có một URL, không chứng minh mọi claim đều có nguồn.
