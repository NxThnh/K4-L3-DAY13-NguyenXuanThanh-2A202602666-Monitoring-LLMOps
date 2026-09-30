# Template Alert và Runbook

Mỗi alert phải dựa trên triệu chứng người dùng hoặc SLO, không dựa trực tiếp vào tên implementation nội bộ.

## Alert mẫu để tham khảo

Ví dụ dưới đây minh họa mức độ cụ thể cần có. Học viên không cần copy nguyên, nhưng ba alert trong bài nộp nên rõ ràng tương tự: điều kiện là gì, kéo dài bao lâu, ảnh hưởng tới user ra sao và người trực cần kiểm tra gì trước.

- Tên: `HighLatencyP95`
- Severity: `warning`
- Duration: `5m`
- Kênh thông báo: Slack `#k4-l3b-alerts`
- SLI/SLO liên quan: latency P95 của `response_sent.latency_ms`
- Điều kiện và thời gian duy trì: `p95(latency_ms) > 3000ms` trong 5 phút
- Ảnh hưởng tới người dùng: người dùng phải chờ lâu hơn trước khi nhận câu trả lời
- Ba bước kiểm tra đầu tiên:
  1. Mở dashboard latency để xác nhận P95/P99 và khoảng thời gian tăng.
  2. Lọc `data/logs.jsonl` trong khoảng đó, lấy một `correlation_id` có `latency_ms` cao.
  3. Mở trace cùng `correlation_id` trên Langfuse, so sánh các span chính để xác định bước nào bất thường.
- Mitigation tạm thời: dựa trên evidence thực tế để rollback prompt, khôi phục cấu hình liên quan, tắt practice scenario hoặc giảm tải khi demo.
- Owner: `student-<MSSV>`

## Alert 1

- Tên: `HighLatencyP95`
- Severity: `warning`
- Duration: `5m`
- Kênh thông báo: Slack `#k4-l3b-alerts`
- SLI/SLO liên quan: Latency P95 của `response_sent.latency_ms`
- Điều kiện và thời gian duy trì: `p95(latency_ms) > 3000ms` liên tục trong 5 phút
- Ảnh hưởng tới người dùng: Người dùng phải chờ lâu hơn bình thường trước khi nhận câu trả lời
- Ba bước kiểm tra đầu tiên:
  1. Mở dashboard latency để xác nhận P95/P99 và khoảng thời gian bắt đầu tăng đột biến.
  2. Lọc `data/logs.jsonl` trong khoảng đó, lấy correlation ID của các request có `latency_ms > 3000`.
  3. Mở trace có correlation ID đó trên Langfuse, so sánh thời gian của span `retrieval` và span `generation` để xác định bước gây nghẽn.
- Mitigation tạm thời: Nếu span `retrieval` bị chậm (do incident RAG hoặc vector store quá tải), chuyển sang fallback context hoặc tắt kịch bản inject. Nếu span `generation` chậm do prompt mới hoặc model quá tải, rollback prompt label `production` về version trước đó.
- Owner: `student-2A202602666`

## Alert 2

- Tên: `HighErrorRate`
- Severity: `critical`
- Duration: `3m`
- Kênh thông báo: Slack `#k4-l3b-alerts`
- SLI/SLO liên quan: Tỉ lệ lỗi tổng thể `count(request_failed) / count(request_received) * 100`
- Điều kiện và thời gian duy trì: `error_rate_pct > 2%` liên tục trong 3 phút
- Ảnh hưởng tới người dùng: Người dùng nhận mã lỗi HTTP 500 hoặc không nhận được câu trả lời
- Ba bước kiểm tra đầu tiên:
  1. Mở dashboard errors để kiểm tra tỉ lệ lỗi và phân bố `error_type` (ví dụ RuntimeError, TimeoutError).
  2. Lọc log `request_failed` trong `data/logs.jsonl` để trích xuất `correlation_id` và `payload.detail`.
  3. Mở trace tương ứng trên Langfuse để xem span nào bị crash hoặc trả về exception.
- Mitigation tạm thời: Khởi động lại service nếu deadlock, kích hoạt fallback cơ chế trả lời an toàn, ngắt incident giả lập hoặc chuyển traffic sang instance dự phòng.
- Owner: `student-2A202602666`

## Alert 3

- Tên: `LowRetrievalSuccess`
- Severity: `warning`
- Duration: `5m`
- Kênh thông báo: Slack `#k4-l3b-alerts`
- SLI/SLO liên quan: Tỉ lệ truy vấn tri thức thành công `tool_success_rate_pct`
- Điều kiện và thời gian duy trì: `tool_success_rate_pct < 90%` trong 5 phút
- Ảnh hưởng tới người dùng: Câu trả lời của LLM thiếu ngữ cảnh tài liệu chuyên sâu, suy giảm chất lượng thông tin
- Ba bước kiểm tra đầu tiên:
  1. Mở panel Errors trên dashboard để quan sát tỉ lệ `tool_success_rate_pct` và kiểm tra `tool_name="retrieval"`.
  2. Tìm kiếm log có `tool_success: false` trong `data/logs.jsonl` và ghi nhận lỗi liên quan đến vector store.
  3. Mở trace trên Langfuse, kiểm tra observation `retrieval` để phân tích query đầu vào và lỗi vector store timeout.
- Mitigation tạm thời: Chuyển hướng retrieval sang cache cục bộ hoặc kho lưu trữ dự phòng; tạm thời vô hiệu hóa incident tool failure nếu đang trong môi trường thử nghiệm.
- Owner: `student-2A202602666`
