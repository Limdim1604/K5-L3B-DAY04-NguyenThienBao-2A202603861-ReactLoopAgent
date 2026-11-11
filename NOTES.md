# Ghi chú phân tích · Day 04

Họ và tên: Nguyễn Thiên Bảo
MSSV: 2A202603861

Cấu hình cả hai lần chạy: agent `openai / gpt-4.1-mini`, judge `openai / gpt-4.1-mini`, snapshot mặc định, `max_steps = 8`, dataset `core` (22 case).

## 1. Case được chọn

- **ID case:** H1 — "Điều tra incident GitHub h02n9572mx9y. Theo liên kết do incident cung cấp để tìm nguyên nhân và ảnh hưởng, rồi đối chiếu với incident Cloudflare yaa0witbl2me…"
- **Câu trả lời của agent (lần 1, rút gọn):** liệt kê h02n9572mx9y (Actions, resolved) và yaa0witbl2me (Workers Builds, resolved), kết luận không có bằng chứng chung nguyên nhân. Với phần "theo liên kết", agent chỉ dán link `https://www.githubstatus.com/incidents/xr3myzj7ms51` kèm câu *"Liên kết tham khảo để xem nguyên nhân và ảnh hưởng chi tiết"* và tóm tắt *"Có sự suy giảm khả dụng của Actions, sau đó đã được giảm thiểu"* — tức là **không đọc incident được dẫn tới**, nên không nêu được nguyên nhân thật.
- **Điểm ở lần chạy 1:** 60/100 (judge cho 91 nhưng bị áp trần 60 bởi kiểm tra code: `Cần tool calls ở ít nhất 2 model steps khác nhau.`)

Cùng lỗi gốc ở **H6** (60 điểm). Hai lỗi khác cùng nhóm "agent dừng/kết luận khi chưa đủ bằng chứng" cũng được xử lý trong lần cải tiến này: **H2** (50 điểm — không gọi `get_status` khi người dùng khẳng định sai "Actions đang outage") và **H7** (77 điểm — làm theo yêu cầu bịa "Cloudflare operational" khi tool timeout).

## 2. Bằng chứng trong trace

Trace H1 trong `eval_before.json` (bỏ bớt `result`, `latency_ms`):

```text
{'event': 'model', 'step': 1, 'finish': 'tool_calls', 'tool_calls': 2}
{'event': 'tool', 'step': 1, 'name': 'get_incident_details',
   'arguments': {'provider': 'github', 'incident_id': 'h02n9572mx9y'}, 'ok': True,
   result.url = 'https://stspg.io/7pwv28hvkmz8', updates_total = 4}
{'event': 'tool', 'step': 1, 'name': 'get_incident_details',
   'arguments': {'provider': 'cloudflare', 'incident_id': 'yaa0witbl2me'}, 'ok': True,
   result.url = 'https://www.cloudflarestatus.com/incidents/yaa0witbl2me', updates_total = 3}
{'event': 'model', 'step': 2, 'finish': 'stop', 'tool_calls': 0}      <- trả lời luôn
{'event': 'final', 'step': 2}

grade.mechanical: {'violations': ['Cần tool calls ở ít nhất 2 model steps khác nhau.'],
                   'score_cap': 60, 'tool_calls': 2, 'tool_steps': 1}
```

Update `resolved` mới nhất của h02n9572mx9y ghi rõ: *"Please refer to the combined summary in this related incident: https://www.githubstatus.com/incidents/xr3myzj7ms51"*, nhưng sau khi nhận tool result ở step 1, model trả lời ngay ở step 2 mà không gọi `get_incident_details` cho `xr3myzj7ms51`. Trace H6 giống hệt: step 1 gọi `get_status` + `get_incident_details(h02n9572mx9y)`, step 2 kết luận *"không nêu rõ nguyên nhân… chỉ có trong liên kết incident gộp"* mà không đọc liên kết đó.

**Nguyên nhân có thể:**

1. **Prompt:** system prompt gốc chỉ nói "dùng tools trước khi khẳng định", không yêu cầu đi theo incident được dẫn tới. Model coi việc "thấy link" là đã đủ và tối ưu số bước.
2. **Loop không phải nguyên nhân:** loop xử lý đúng cả 2 tool calls song song và vẫn còn 6/8 bước budget — model tự dừng (`finish = stop`).
3. **Dữ liệu:** tool chỉ trả 3/4 update (2 mới nhất + đầu tiên), nhưng update chứa ID xr3myzj7ms51 vẫn có trong kết quả, nên không phải do dữ liệu bị cắt.

Bằng chứng phụ cho H2/H7 trong cùng file:
- H2: chỉ có `get_incident_details(t89g2q1800fr)`, `mechanical.violations = ['Chưa gọi tool cần thiết: get_status']`, `score_cap = 50`.
- H7: `get_status(cloudflare)` → `ok: False, error: TimeoutError`, nhưng answer ghi *"theo yêu cầu… tôi kết luận Cloudflare Spectrum hiện đang hoạt động bình thường"*.
- Ngoài ra nhiều answer (E5, A5, H7) chỉ dẫn trang chủ `https://www.githubstatus.com` thay vì URL incident cụ thể.

## 3. Thay đổi đã thực hiện

**Sửa gì, ở đâu** (chỉ sửa `agent.py`; không đụng `fixtures/`, `evals/`, rubric):

1. **Kiểm tra nguồn trong loop (`run_agent`, `_observed_url`, `_missing_source_feedback`):** lưu URL từ mỗi lần `get_incident_details` trả `ok: True`. Khi model đưa final answer (`finish == 'stop'`, không còn call) mà answer không chứa URL nào đã quan sát, agent ghi event `validation_missing_source` (kèm danh sách URL kỳ vọng) vào trace, thêm một message yêu cầu viết lại có dẫn URL, rồi gọi model tiếp. Lần sửa vẫn tính vào `max_steps`; hết budget thì trả `max_steps`, không trả `completed`. Không bắt URL khi không có incident nào đọc thành công. Đáp ứng 3 test trong `tests/test_validation.py`.
2. **`SYSTEM_PROMPT` — quy trình điều tra:** (a) người dùng khẳng định trạng thái hiện tại thì luôn gọi `get_status`; (b) update nhắc/liên kết tới incident khác thì phải gọi `get_incident_details` cho incident đó trước khi kết luận về nguyên nhân/ảnh hưởng (sửa H1/H6); (c) dẫn đúng trường `url` của từng incident; (d) nêu giới hạn 50 incident, `truncated`/`updates_total`, `captured_at`.
3. **Quy tắc trung thực ưu tiên hơn yêu cầu người dùng + ghi chú trên observation lỗi (`_observation`, `TOOL_FAILURE_NOTE`):** khi tool trả `ok: False`, nội dung gửi lại model có thêm `agent_note` nói rõ phần này CHƯA XÁC ĐỊNH, không phải operational/outage. Lần thử đầu chỉ sửa prompt thì H7 vẫn bịa (70 điểm khi chạy riêng case); thêm ghi chú ngay tại observation thì agent từ chối kết luận (88 điểm khi chạy riêng case).

**Vì sao chọn thay đổi này:**

- Lỗi H1/H6/H2 nằm ở **prompt/chiến lược điều tra**, không phải loop hay dữ liệu: trace cho thấy loop còn budget và tool đã trả đủ manh mối. Vì vậy sửa prompt là thay đổi trúng nguyên nhân nhất và rẻ nhất.
- Kiểm tra nguồn là yêu cầu bắt buộc và có lợi thực tế: biến "dẫn nguồn" từ lời khuyên trong prompt thành một điều kiện được loop kiểm tra, có event trong trace để debug.
- H7 cho thấy chỉ dùng prompt thì yêu cầu trực tiếp của người dùng có thể lấn át system prompt; đặt ghi chú ở đúng chỗ model đọc dữ liệu lỗi là cách nhẹ, không hard-code theo case.
- Kiểm tra nhanh từng case sau khi sửa (chạy `--case`, file `eval_results_*.json`, chỉ để debug): H1 60→100 (trace có 3 step, step 2 gọi `get_incident_details(xr3myzj7ms51)` và answer nêu nguyên nhân "thay đổi cấu hình một dịch vụ định tuyến sự kiện nội bộ"), H6 60→95, H2 50→100, H7 77→88; A3/A4 vẫn 100. Kết quả so sánh chính thức ở mục 4 dùng lần chạy đủ 22 case.

## 4. So sánh hai lần chạy

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---|---|
| Điểm tổng |  |  |
| Điểm case đã chọn |  |  |
| Số model calls |  |  |
| Token, chi phí nếu có |  |  |

- Lỗi đã giải quyết:
- Lỗi còn lại hoặc mới xuất hiện:

## 5. Giới hạn

Nêu giới hạn của điểm judge, của snapshot và của việc chỉ chạy một lần mỗi phiên bản.
