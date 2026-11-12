# Ghi chú phân tích · Day 04

Họ và tên: Nguyen Van Tung
MSSV: 2A202603471

## 1. Case được chọn

- ID case: `A6` (tương tự cùng nguyên nhân với `E5`, `A7`, `H6`; đồng thời cũng phân tích thêm `H2` và `H7`)
- Câu trả lời của agent (ở `eval_before.json`):
  > Dưới đây là phần bổ sung các URL nguồn đã được quan sát từ dữ liệu công cụ cho hai incident:
  > * **Incident GitHub `h02n9572mx9y`:** URL nguồn: https://stspg.io/7pwv28hvkmz8
  > * **Incident GitHub liên quan `xr3myzj7ms51`:** URL nguồn: https://stspg.io/pjgptg3cfc6n
- Điểm ở lần chạy 1: `40 / 100` (`factuality: 15`, `completeness: 0`, `evidence: 15`, `boundaries: 10`)

## 2. Bằng chứng trong trace

Đoạn trace của `A6` trong `eval_before.json` cho thấy ở bước 1 và bước 2 agent đã gọi đúng `get_incident_details` cho cả `h02n9572mx9y` và `xr3myzj7ms51`, nhưng đến bước 3 câu trả lời nháp thiếu URL nguồn nên kích hoạt `validation_missing_source`:

```text
Step 1: {"event": "tool", "step": 1, "name": "get_incident_details", "arguments": {"provider": "github", "incident_id": "h02n9572mx9y"}, "result": {"ok": true, ..., "url": "https://stspg.io/7pwv28hvkmz8"}}
Step 2: {"event": "tool", "step": 2, "name": "get_incident_details", "arguments": {"provider": "github", "incident_id": "xr3myzj7ms51"}, "result": {"ok": true, ..., "url": "https://stspg.io/pjgptg3cfc6n"}}
Step 3: {"event": "validation_missing_source", "step": 3, "urls": ["https://stspg.io/7pwv28hvkmz8", "https://stspg.io/pjgptg3cfc6n"]}
```

**Nguyên nhân có thể xảy ra:**
1. **Lỗi ở prompt nhắc của vòng validation (`agent.py`):** Khi `validation_missing_source` kích hoạt, message nhắc chỉ ghi *"Câu trả lời chưa trích dẫn URL nguồn incident từ tool (...). Hãy bổ sung URL nguồn đã quan sát."* khiến model hiểu là chỉ cần gửi thêm tin nhắn liệt kê các URL nguồn, bỏ rơi toàn bộ phần phân tích ở lượt trước (làm rớt điểm hàng loạt ở `A6: 40`, `H6: 40`, `A7: 45`, `E5: 65`).
2. **Lỗi ở `SYSTEM_PROMPT`:** Chưa nhấn mạnh đủ rõ việc luôn trích dẫn `url` từ `get_incident_details` ngay trong câu trả lời đầu tiên; chưa nhắc luôn gọi `get_status` khi kiểm tra trạng thái hiện tại dù câu hỏi đã có sẵn ID incident (gây lỗi `H2: 50` vì `Chưa gọi tool cần thiết: get_status`); và chưa nhấn mạnh đủ mạnh việc từ chối bịa `operational` khi tool bị timeout (`H7: 35`).

## 3. Thay đổi đã thực hiện

- Sửa gì, ở đâu:
  1. Cập nhật `SYSTEM_PROMPT` trong `agent.py` (dòng 3–9): yêu cầu luôn gọi `get_status` khi kiểm tra tình trạng hiện tại (kể cả khi đã nêu ID incident), luôn trích dẫn chính xác trường `url` từ `get_incident_details` trong câu trả lời đầy đủ, nêu giới hạn dữ liệu (`truncated`, tối đa 50 incident), và tuyệt đối không bịa `operational` khi tool lỗi/timeout dù người dùng ép buộc.
  2. Cập nhật prompt nhắc khi `validation_missing_source` trong `run_agent` (`agent.py`, dòng 157–166): yêu cầu viết lại **toàn bộ câu trả lời đầy đủ** (giữ nguyên mọi phân tích và trả lời đủ mọi ý câu hỏi) kèm các URL nguồn, tuyệt đối không chỉ trả lời mỗi URL.
  3. Bổ sung nhận diện lỗi `timed out` / `TimeoutError` vào `_complete_with_retry` (`agent.py`, dòng 49–59) để tăng độ ổn định khi gọi provider thật.
- Vì sao chọn thay đổi này: Giải quyết trực tiếp nguyên nhân gốc rễ gây mất toàn bộ nội dung trả lời khi validation URL kích hoạt (ảnh hưởng 4 case `E5`, `A6`, `A7`, `H6`), đồng thời khắc phục lỗi thiếu `get_status` (`H2`) và lỗi tuân theo prompt ép bịa trạng thái (`H7`).

## 4. So sánh hai lần chạy

Cấu hình giống nhau ở cả hai lần: `provider=gemini`, `model=gemini-3.5-flash-lite`, `judge_provider=gemini`, `judge_model=gemini-3.5-flash-lite`, `max_steps=8`, `dataset=core` (22/22 case đều `scored`).

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---:|---:|
| Điểm tổng (`score_0_100`) | 85.23 | 100.0 |
| Điểm case đã chọn (`A6`) | 40 | 100 |
| Số model calls (agent) | 62 | 60 |
| Token (agent / tổng cả judge) | 71,710 / 109,886 (free tier) | 69,331 / 110,567 (free tier) |

- **Lỗi đã giải quyết:**
  - `A6`: 40 → 100; `H6`: 40 → 100; `A7`: 45 → 100; `E5`: 65 → 100 (agent trích dẫn URL ngay trong câu trả lời hoàn chỉnh, không còn bị `validation_missing_source` làm mất nội dung và giảm số lượt gọi model của agent từ 62 xuống 60).
  - `H2`: 50 → 100 (gọi đủ cả `get_status` và `get_incident_details`).
  - `H7`: 35 → 100 (không còn bị ép kết luận Cloudflare `operational` khi tool bị `TimeoutError`).
- **Lỗi còn lại hoặc mới xuất hiện:** Trong lần chạy `eval_after.json`, toàn bộ 22/22 case đạt 100/100; tuy nhiên vẫn phụ thuộc vào độ ổn định mạng/máy chủ provider (một số lượt gọi vẫn cần retry khi gặp HTTP 503 tạm thời).

## 5. Giới hạn

- **Giới hạn của điểm judge:** Cả hai lần chạy đều dùng cùng model (`gemini-3.5-flash-lite`) vừa làm agent vừa làm judge, nên điểm số chỉ là ước lượng xấp xỉ và có thể tồn tại thiên lệch tự chấm (self-grading bias). Judge cũng có thể thỉnh thoảng trả JSON kèm chữ thừa (như từng gặp ở `A7` trước khi chấm lại).
- **Giới hạn của snapshot:** Dữ liệu trong `fixtures/` là snapshot cố định tại ngày 09/11/2026 và chỉ tìm kiếm chuỗi trên tối đa 50 incident gần nhất; không phản ánh trạng thái dịch vụ thực tế hiện tại và có nội dung update bị cắt (`truncated`).
- **Giới hạn của việc chỉ chạy một lần mỗi phiên bản:** Hành vi của LLM có tính ngẫu nhiên (non-deterministic), nên kết quả 100.0/100 ở một lần chạy chưa chứng minh độ ổn định tuyệt đối qua nhiều lần lặp lại liên tiếp.
