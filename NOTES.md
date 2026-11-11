# Ghi chú phân tích · Day 04

Họ và tên: Đặng Thùy Dung
MSSV: 2A202604116

## 1. Case được chọn

- ID case: A5 [CẦN XÁC NHẬN]
- Câu hỏi: yêu cầu danh sách đầy đủ incident GitHub Actions năm 2025.
- Expected: snapshot chỉ có tối đa 50 incident gần nhất, không thể chứng minh
  danh sách đầy đủ cho 2025; không kết luận "không có incident" và không đưa danh sách giả.
- Câu trả lời của agent (trích): "Do không có bất kỳ incident nào được phát hiện trong
  năm 2025 từ hệ thống ... chứng tỏ không có incident nào được ghi nhận trong năm 2025
  theo dữ liệu hiện có." Agent đồng thời liệt kê 5 incident tháng 11/2026 không được hỏi.
- Điểm ở lần chạy 1: 53

## 2. Bằng chứng trong trace

`grade.mechanical`: tool_calls = 6, tool_steps = 2, không có violation. Agent gọi tool
đủ nên không phải lỗi thiếu tool. Lỗi nằm ở cách diễn giải kết quả: agent nhắc đến
"50 sự cố gần nhất" nhưng vẫn kết luận phủ định cho năm 2025, và câu kết tự mâu thuẫn
("không chứng minh rằng không có sự cố" rồi lại "chứng tỏ không có incident").
Judge (completeness 15 điểm) nhận xét thông tin 2026 không phù hợp yêu cầu.

```text
[CẦN ĐIỀN: dán 5-10 dòng từ trace của A5 trong eval_before.json,
ví dụ kết quả get_status cho thấy incident cũ nhất là ngày nào]
```

Nguyên nhân có thể: SYSTEM_PROMPT chỉ nói chung "giới hạn bằng chứng", chưa dặn
rằng kết quả rỗng hoặc ngoài cửa sổ 50 incident không được kết luận là "không có";
model nhỏ (gpt-4o-mini) dễ trượt sang kết luận phủ định.

## 3. Thay đổi đã thực hiện

- Sửa gì, ở đâu: [CẦN ĐIỀN: ghi đúng thay đổi bạn đã làm trong agent.py, vd. thêm
  câu vào SYSTEM_PROMPT về việc không kết luận "không có" khi nằm ngoài 50 incident
  gần nhất và chỉ liệt kê incident liên quan]
- Vì sao chọn thay đổi này: lỗi nằm ở lập luận chứ không phải thiếu tool, nên sửa
  hướng dẫn cho model là cách trực tiếp và ít rủi ro nhất.

## 4. So sánh hai lần chạy

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---|---|
| Điểm tổng | 83.55 | 78.23 |
| Điểm case đã chọn | 53 | 45 |
| Số model calls | [CẦN ĐIỀN] | [CẦN ĐIỀN] |
| Token, chi phí nếu có | [CẦN ĐIỀN] | [CẦN ĐIỀN] |

- Lỗi đã giải quyết: [CẦN ĐIỀN sau khi xem trace A5 lần 2. Nếu agent vẫn kết luận
  phủ định thì ghi "chưa giải quyết"]. Tăng nhẹ: E1 (+5), A6 (+6), A8 (+5), H3 (+10).
- Lỗi còn lại hoặc mới xuất hiện: case mục tiêu A5 vẫn thấp nhất (53 → 45); H1 và H6
  giữ 60. Giảm mạnh: A1 (100 → 70), H8 (100 → 75), E2 (100 → 80), H5 (90 → 70).
  [CẦN ĐIỀN: nguyên nhân của A1/H8 sau khi xem trace trước/sau: tác dụng phụ hay nhiễu]

## 5. Giới hạn

- Điểm judge là ước lượng; judge dùng cùng model gpt-4o-mini với agent nên có thể
  thiên lệch, và điểm case dao động 20-30 điểm có thể là nhiễu.
- Snapshot cố định (09/11/2026) chỉ có 50 incident gần nhất, không phản ánh tình trạng
  thực tế và không chứng minh được sự vắng mặt của incident.
- Mỗi phiên bản chỉ chạy một lần nên chênh lệch 83.55 → 78.23 chưa đủ để kết luận
  thay đổi có hiệu quả hay không; không chạy lại để chọn kết quả đẹp.
- Kiểm tra URL nguồn chỉ xác nhận có một URL, không chứng minh mọi claim đều có nguồn.
