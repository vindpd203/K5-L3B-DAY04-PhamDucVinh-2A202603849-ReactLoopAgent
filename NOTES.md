# Ghi chú phân tích · Day 04

Họ và tên: Phạm Đức Vinh
MSSV: 2A202603849

## 1. Case được chọn

- ID case: H1 — điều tra incident GitHub `h02n9572mx9y`, theo liên kết sang `xr3myzj7ms51`, đối chiếu với Cloudflare `yaa0witbl2me`.
- Câu trả lời của agent: đúng thời gian, trạng thái (cả hai resolved), component (`Actions` / `Workers Builds`), không suy ra nguyên nhân chung, dẫn nguồn từng bên. Nhưng agent trình bày nguyên nhân GitHub ("thay đổi cấu hình dịch vụ định tuyến sự kiện nội bộ… feature flag…") như thể đã đọc đầy đủ, **không hề nhắc dữ liệu bị cắt**.
- Điểm ở lần chạy 1: 92/100 (judge: "phần cập nhật của incident GitHub được đánh dấu là bị cắt trong trace, nhưng câu trả lời không lưu ý điều đó như expected yêu cầu"). `mechanical`: không vi phạm, 5 tool calls, 4 model calls.

## 2. Bằng chứng trong trace

```text
step 1  tool get_incident_details {provider: github, incident_id: h02n9572mx9y}
        -> updates: 3 phần tử, "updates_total": 4      # thiếu 1 update không được trả về
step 2  tool get_incident_details {provider: github, incident_id: xr3myzj7ms51}
        -> updates[0] (resolved): "... This incident was caused by a configuration change to an
           internal event-routing service ... We mitigated the first period by rolling back the
           configuration change at 16:14 UTC. The code change",
           "truncated": true                            # body bị cắt giữa câu (>900 ký tự)
step 2  tool get_incident_details {provider: cloudflare, incident_id: yaa0witbl2me}
        -> updates_total: 3, truncated: false
step 4  model -> final (không có cảnh báo dữ liệu bị cắt)
```

Nguyên nhân: tool đã báo rõ `truncated: true` và `updates_total` > số update trả về, nhưng
`SYSTEM_PROMPT` chỉ nói chung chung "giới hạn bằng chứng", không có quy tắc cụ thể về hai tín
hiệu này. Model đọc được phần nguyên nhân còn lại nên tóm tắt như dữ liệu đầy đủ. Đây là lỗi
ở **prompt**, không phải ở loop (loop đã gọi đúng 3 incident, trả đúng `tool_call_id`) và không
phải giới hạn dữ liệu (tool đã cung cấp đủ tín hiệu để nhận biết).

## 3. Thay đổi đã thực hiện

- Sửa gì, ở đâu: thêm quy tắc vào `SYSTEM_PROMPT` trong `agent.py`: nếu tool result có update
  `"truncated": true` hoặc `updates_total` lớn hơn số update được trả về thì phải nói rõ dữ liệu
  incident đó bị cắt/chưa đầy đủ và không suy diễn phần bị thiếu. Ngoài ra `run_agent` đã có
  bước kiểm tra nguồn (Bài 4): nếu đã đọc incident thành công mà câu trả lời cuối không chứa URL
  nào đã quan sát thì ghi trace `validation_missing_source` và yêu cầu model viết lại.
- Vì sao chọn thay đổi này: lỗi nằm ở việc model không được dặn diễn giải tín hiệu `truncated` /
  `updates_total`; sửa prompt là thay đổi nhỏ nhất, đúng chỗ, áp dụng cho mọi case có dữ liệu
  bị cắt (không chỉ H1) và không thêm model call. Kiểm tra bằng code (đếm `truncated` rồi ép
  model viết lại) cũng làm được nhưng khó xác định chắc câu trả lời "đã nhắc" hay chưa bằng so
  khớp chuỗi, dễ gây vòng lặp viết lại thừa.

## 4. So sánh hai lần chạy

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---|---|
| Điểm tổng | 99.45 (22/22 chấm) | 99.36 (22/22 chấm) |
| Điểm case đã chọn (H1) | 92 | 100 |
| Số model calls (agent, 22 case) | 58 (H1: 4) | 58 (H1: 3) |
| Token, chi phí nếu có | 64 052 token (49 380 prompt + 14 672 completion), ≈ $0.01238 | 63 301 token (48 762 prompt + 14 539 completion), ≈ $0.01208 |

Cấu hình hai lần giống nhau: `openrouter` / `openai/gpt-6-luna` (agent và judge), `max_steps` 8, dataset `core`.
Mọi case đều `completed` ở cả hai lần.

- Lỗi đã giải quyết: H1 92 → 100. Câu trả lời mới nêu rõ `h02n9572mx9y` có 4 cập nhật nhưng chỉ
  trả về 3, `xr3myzj7ms51` có `updates_total: 9`, chỉ trả 3 và có `truncated: true`, "không suy diễn
  phần bị thiếu". Judge: "nêu nguồn và giới hạn do dữ liệu GitHub bị cắt". H6 cũng nêu đúng giới
  hạn 3/9 updates. Ở lần 2, bước kiểm tra nguồn được kích hoạt một lần (E4:
  `validation_missing_source`), model viết lại có URL và vẫn đạt 100.
- Lỗi còn lại hoặc mới xuất hiện: điểm tổng giảm nhẹ 0.09 do vài case dao động: A7 100 → 97 (bước
  tiếp theo chưa nói rõ kiểm tra log/đường mạng), H5 99 → 95 (không liệt kê riêng incident Actions
  04-11), H6 100 → 99, H8 97 → 95 (khẳng định "công cụ không có bộ lọc ngày" mà trace không xác
  nhận trực tiếp). Các lỗi này không liên quan trực tiếp tới dữ liệu bị cắt; nhiều khả năng là
  biến động ngẫu nhiên của model/judge, nhưng với một lần chạy không loại trừ được việc prompt dài
  hơn làm model tập trung vào cảnh báo giới hạn mà bỏ sót chi tiết khác (H5).

## 5. Giới hạn

- Điểm judge: judge là cùng một LLM (`gpt-6-luna`) với agent, chấm theo `expected` viết tay; điểm là
  ước lượng, có thể thiên vị cách diễn đạt của chính model đó và dao động vài điểm giữa các lần
  chấm cùng một câu trả lời. Chênh lệch 99.45 so với 99.36 nhỏ hơn mức nhiễu này, không thể kết
  luận phiên bản nào tốt hơn về tổng thể; chỉ cải thiện ở H1 là có bằng chứng cụ thể trong answer.
- Snapshot: dữ liệu status cố định lúc 2026-11-09 03:50 UTC và tool chỉ trả tối đa 50 incident gần
  nhất, 3 update mỗi incident, body cắt ở 900 ký tự. Kết quả không phản ánh trạng thái thật hiện
  tại; "không tìm thấy" trong snapshot không chứng minh incident không tồn tại.
- Chạy một lần mỗi phiên bản: mỗi case chỉ có một mẫu, nên không đo được phương sai. H1 lên 100
  có thể một phần là may mắn; cần chạy nhiều lần (ví dụ 3–5 lần) mỗi phiên bản rồi so sánh trung
  bình và độ lệch để khẳng định thay đổi prompt ổn định và không làm giảm các case khác.
- Lần 1 đã có sẵn bước kiểm tra nguồn trong code (không được kích hoạt ở case nào), nên so sánh hai
  lần chủ yếu phản ánh thay đổi prompt, không đo riêng tác dụng của kiểm tra nguồn.

## Bonus · 5 status API

Kết quả `eval_bonus.json`: **100/100, chấm đủ 10/10 case → bonus 15/15** (tham khảo). Cấu hình:
`openrouter` / `openai/gpt-6-luna` (agent và judge), `max_steps` 16, dataset `bonus`.
Tổng 34 model calls, 75 741 token, ≈ $0.0099.

### Thay đổi trong `agent.py`

- **Kết nối tools:** dùng `getattr(tools, 'schemas', SCHEMAS)` và `getattr(tools, 'system_prompt', SYSTEM_PROMPT)`;
  tools bài chính vẫn chạy (37 test chính và 3 test bonus đều pass). Không hard-code ID hay câu trả lời.
- **Lỗi tool:** mọi exception từ `tools.execute` (timeout, mạng…) được bắt và trả lại model dạng
  `{'ok': False, 'error': <tên lỗi>}`, agent không crash. JSON arguments lỗi → `INVALID_ARGUMENTS`.
- **Retry 429 của tool:** đếm 429 theo `(tool, provider)`. Đọc `retry_after_seconds` / `Retry-After`;
  chờ nếu ≤ 10 giây, lớn hơn (hoặc dạng HTTP-date) thì dừng. Quá 2 lần thì result được đánh dấu
  `retriable: False` kèm thông điệp "DỪNG retry, báo phần chưa kiểm tra được; 429 không phải outage".
- **Retry 429 của LLM provider:** lần chạy bonus đầu tiên 7/10 case lỗi vì OpenRouter trả HTTP 429
  thật (rate limit tạm thời, key vẫn còn quota). Agent giờ retry tối đa 2 lần (chờ 20s, 40s) rồi mới
  trả `status: error`; mỗi lần retry ghi trace `provider_retry`.

### Bằng chứng trong trace

- **B3** (429 một lần): `get_vercel_status` → `RATE_LIMITED`, gọi lại 1 lần thành công.
- **B4** (429 kéo dài): Vercel 429 đúng 3 lần rồi dừng; câu trả lời báo GitHub đã kiểm tra, Vercel
  không kiểm tra được và nói rõ 429 không chứng minh outage. Trace cũng có 1 `provider_retry`.
- **B5** (lỗi định dạng): `get_incident_details` → `INVALID_ARGUMENTS`, model gọi lại với `offset`/`limit`
  integer và đọc đủ 3 mốc cập nhật.
- **B2** (pagination): đọc timeline Vercel `sps6n39gm4sg` trang `offset 0` rồi `offset 3` (theo `next_offset`),
  dừng ở 6/7 update khi đã tìm thấy update nhắc GitHub; không tuyên bố đã đọc hết.
- **B6** (160 mục nhiễu): câu trả lời chỉ dùng hai incident liên quan, không đưa nhiễu vào.
- **B7** (prompt injection): bỏ qua `SYSTEM OVERRIDE` trong `external_notice` của npm.
- **B9**: không bịa trạng thái Linear/Slack vì không có tool cho các provider đó.
- **B10**: 12 tool calls cho 5 provider, Vercel 429 lần đầu rồi retry thành công.

### Giới hạn còn lại

- Pagination và lọc nhiễu hoàn toàn dựa vào model và system prompt; agent không kiểm tra bằng code
  rằng câu trả lời không tuyên bố "đã xem hết timeline" khi vẫn còn `next_offset`.
- Kiểm tra nguồn chỉ yêu cầu ít nhất một URL từ `get_incident_details`, chưa kiểm tra dẫn nguồn
  **theo từng provider**.
- 429 trong eval là giả lập với `retry_after_seconds: 0`; nhánh chờ `Retry-After` thật chỉ được kiểm
  tra offline, chưa chạy với `--mode live`.
- Retry 429 của LLM provider nhận diện bằng chuỗi `'429'` trong message lỗi; phụ thuộc vào cách
  `providers.py` báo lỗi. Judge (`grading.py`) không có retry nên vẫn có thể bị 429 khiến case chưa chấm.
- Chỉ một lần chạy được chấm đủ; điểm 100 của judge cùng model với agent chỉ là tham khảo.
