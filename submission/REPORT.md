# Báo cáo cá nhân — K4-L3B Day 13 Monitoring & LLMOps

> Mỗi học viên hoàn thiện một file duy nhất này. Khi dẫn evidence, dùng đường dẫn tương đối, ví dụ `evidence/07-trace-waterfall.png`.

## 1. Thông tin học viên

- **Họ và tên:** Nguyễn Xuân Thành
- **MSSV:** 2A202602666
- **Lớp:** K4-L3B
- **Repository URL:** https://github.com/NxThnh/K4-L3-DAY13-NguyenXuanThanh-2A202602666-Monitoring-LLMOps.git
- **Commit SHA cuối:** `77a01ff`
- **Challenge ID:** practice-rag_slow
- **Tên project Langfuse cá nhân:** `day13-k4-l3b-2A202602666`

## 2. Evidence index

Điền đúng đường dẫn tới evidence thực tế. Có thể đổi tên hoặc dùng nhiều ảnh nếu cần.

| Evidence | Đường dẫn |
|---|---|
| Pytest cuối | `evidence/01-pytest.png` |
| Log validator | `evidence/02-log-validator.png` |
| Dashboard validator | `evidence/03-dashboard-validator.png` |
| Structured log | `evidence/04-structured-log.png` |
| PII redaction | `evidence/05-pii-redaction.png` |
| Trace list | `evidence/06-trace-list.png` |
| Trace waterfall | `evidence/07-trace-waterfall.png` |
| Trace metadata | `evidence/08-trace-metadata.png` |
| Prompt versions | `evidence/09-prompt-versions.png` |
| Prompt rollback | `evidence/10-prompt-rollback.png` |
| Dashboard runtime | `evidence/11-dashboard-overview.png` |
| Incident metric | `evidence/12-incident-metric.png` |
| Incident log | `evidence/13-incident-log.png` |
| Incident trace | `evidence/14-incident-trace.png` |

## 3. Kết quả kỹ thuật

| Nội dung | Baseline | Kết quả cuối | Nhận xét |
|---|---|---|---|
| `validate_logs.py` | 80/100 (thiếu correlation ID) | 100/100 | Đạt toàn diện: schema, correlation ID propagation, enrichment, PII scrubbing |
| `validate_dashboard.py` | 6/6 panel | 6/6 panel | Hợp lệ toàn bộ 6 panel contract YAML |
| `pytest` | 22 passed | 22 passed | Toàn bộ 22/22 tests kiểm thử chức năng và tích hợp đều vượt qua |
| Số traces hợp lệ | 0 | 15+ traces | Đầy đủ span tree chuẩn (root agent -> retrieval + generation) trên Langfuse |
| Số PII leak | 0 | 0 leak | 100% email, phone VN, CCCD, thẻ thanh toán được che trước khi ghi |
| Latency P95 / TTFT P95 | N/A | 172ms / 55ms | Thời gian phản hồi xuất sắc, nằm sâu dưới ngưỡng SLO (3000ms) |
| Retrieval success rate | N/A | 100% | Tỉ lệ trích xuất tri thức thành công đạt mức tối đa |

## 4. Logging và PII

- **Cách tạo/nhận và truyền correlation ID:**
  Tại `CorrelationIdMiddleware` (`app/middleware.py`), trước khi xử lý mỗi request, middleware gọi `clear_contextvars()` nhằm ngăn chặn rò rỉ ngữ cảnh giữa các request song song. Sau đó trích xuất header `x-request-id`, nếu client không truyền thì sinh mới theo định dạng `req-<8-hex>` (`req-` + `uuid.uuid4().hex[:8]`). Correlation ID này được bind vào contextvars qua `bind_contextvars(correlation_id=correlation_id)` và gán vào `request.state.correlation_id`. Khi trả response, middleware tự động đính kèm ID vào header `x-request-id` và thời gian phản hồi vào `x-response-time-ms`.
- **Các metadata được ghi vào structured log:**
  Mỗi bản ghi log dạng JSON chuẩn hóa chứa các trường: `ts` (ISO timestamp UTC), `level` (info/warning/error), `service` (`api`), `event` (`request_received`, `response_sent`, `request_failed`), `correlation_id`, `user_id_hash` (băm SHA-256 từ `user_id` thô, lấy 12 ký tự), `session_id`, `feature`, `model` (`claude-sonnet-4-5`), `env` (`dev`), cùng các trường đo lường hiệu năng: `latency_ms`, `ttft_ms`, `tokens_in`, `tokens_out`, `cost_usd`, `quality_score`, `tool_name`, `tool_success` và `payload` tóm tắt an toàn.
- **Cách bảo đảm PII được scrub trước khi ghi:**
  Trong `app/logging_config.py`, bộ xử lý `scrub_event` được đăng ký như một structlog processor đặt ngay trước `JsonlFileProcessor` và `JSONRenderer`. Hàm này đệ quy duyệt qua toàn bộ giá trị kiểu chuỗi trong `event_dict` (bao gồm cả nested payload và preview) để áp dụng tập regex `PII_PATTERNS` từ `app/pii.py` (Email, SĐT Việt Nam, CCCD 12 số, Thẻ tín dụng, Hộ chiếu) và thay bằng nhãn an toàn như `[REDACTED_EMAIL]`, `[REDACTED_PHONE_VN]`, `[REDACTED_CREDIT_CARD]`, `[REDACTED_CCCD]`.
- **Cách kiểm chứng kết quả:**
  Chạy lệnh `python scripts/validate_logs.py`. Bộ kiểm tra độc lập quét toàn bộ `data/logs.jsonl` và xác nhận 100/100 điểm: 0 bản ghi thiếu trường, 0 bản ghi thiếu enrichment, 10/10 unique correlation IDs và 0 PII leak.

## 5. Tracing và prompt versioning

- **Cách xác nhận traces do chính tôi tạo trong project cá nhân:**
  Các trace được gửi trực tiếp tới project Langfuse Cloud cá nhân tên `day13-k4-l3b-2A202602666` sử dụng key pair cá nhân. Mọi trace đều mang tag `student-2A202602666`, môi trường `dev`, và gắn metadata `correlation_id` trùng khớp 1:1 với `correlation_id` trong file log cục bộ.
- **Cấu trúc root/retrieval/generation observations:**
  Trace cấp cao nhất là `day13-agent-request`, chứa root span agent `lab-agent-run`. Bên trong root span, sử dụng `client.start_as_current_observation()` để tạo 2 child observations:
  1. `retrieval` (as_type=`retriever`): đo thời gian trích xuất tài liệu từ vector store/corpus (bình thường ~1ms).
  2. `generation` (as_type=`generation`): theo dõi LLM call (`claude-sonnet-4-5`), nhận prompt từ Langfuse, cập nhật usage (`tokens_in`, `tokens_out`, `total`) và chi phí ước tính `cost_usd`.
- **Cách nối trace với log:**
  Thông qua trường `correlation_id` được gán vào metadata của trace Langfuse (`propagate_attributes(metadata={"correlation_id": correlation_id})`) và xuất hiện đồng thời trong mọi bản ghi log `request_received` và `response_sent`.
- **Prompt name:** `day13-chat`
- **Version/label baseline:** Version 1 (`v1`), gắn nhãn `baseline` và `production`.
- **Version/label candidate:** Version 2 (`v2`), gắn nhãn `candidate`.
- **Trace ID của mỗi version:**
  - v1 (baseline / prod): Trace `14a784d9f47b8d51dafe09ec2adb1ac0` (CID: `req-ecb02fa6`)
  - v2 (candidate / promoted prod): Trace `dd14adcc02200eec1149e9676954af17` (CID: `req-4494b4d0`)
  - v1 (reverted rollback): Trace `50c0799fdc50d6692e934217acbd4749` (CID: `req-89fdb341`)
- **Cách promote và rollback `production`:**
  - **Promote:** Chuyển label `production` từ Version 1 sang Version 2 trên Langfuse UI hoặc SDK: `client.update_prompt(name='day13-chat', version=2, new_labels=['candidate', 'production'])`. Ứng dụng tự động tải phiên bản mới khi resolve prompt mà không cần sửa code.
  - **Rollback:** Khi nhận thấy token tăng hoặc latency suy giảm, hoàn tác label `production` về Version 1: `client.update_prompt(name='day13-chat', version=1, new_labels=['baseline', 'production'])`. Ứng dụng ngay lập tức trở lại template ổn định trong vòng 1 giây.

## 6. Dashboard, SLO và alerts

- **Dashboard và sáu panel:**
  Dashboard thời gian thực gồm 6 panel được kiểm chứng đạt chuẩn theo `config/dashboard.yaml`:
  1. *Panel Latency:* Hiển thị P50 (171ms), P95 (172ms), P99 (1390ms) và TTFT P95 (55ms), đường ngưỡng SLO 3000ms.
  2. *Panel Traffic:* Thống kê số lượng request theo phút (RPM), ngưỡng tối thiểu 1 RPM.
  3. *Panel Errors:* Thống kê tỉ lệ lỗi hệ thống (0%) và tỉ lệ retrieval thành công (100%), ngưỡng lỗi tối đa 2%.
  4. *Panel Cost:* Biểu đồ chi phí tích lũy theo phút ($0.0475), ngưỡng ngân sách 2.50 USD/ngày.
  5. *Panel Tokens:* Biểu đồ thanh biểu diễn tổng số token in (480) và token out (1320), trần cảnh báo 50,000 tokens.
  6. *Panel Quality:* Điểm chất lượng trung bình (0.88), ngưỡng chất lượng tối thiểu 0.75.
- **SLO và lý do chọn:**
  SLO chính: `fast_successful_requests`: 99.5% request hoàn thành thành công và độ trễ `latency_ms <= 3000ms` trong chu kỳ 28 ngày. Lý do chọn: 3 giây là giới hạn chịu đựng tối đa của người dùng khi chờ câu trả lời chatbot; nếu vượt quá mức này trải nghiệm tương tác bị gián đoạn.
- **Cách tính error budget:**
  SLO 99.5% trong 28 ngày tương ứng với error budget là 0.5%. Với tổng lưu lượng ước tính 10,000 request trong chu kỳ, hệ thống cho phép tối đa `10,000 * 0.5% = 50 request` thất bại hoặc có độ trễ lớn hơn 3000ms.
- **Ba alert và runbook tương ứng:**
  1. `HighLatencyP95`: cảnh báo warning khi P95(latency_ms) > 3000ms trong 5m. Runbook: [docs/alerts.md#alert-1](../docs/alerts.md#alert-1).
  2. `HighErrorRate`: cảnh báo critical khi error_rate_pct > 2% trong 3m. Runbook: [docs/alerts.md#alert-2](../docs/alerts.md#alert-2).
  3. `LowRetrievalSuccess`: cảnh báo warning khi tool_success_rate_pct < 90% trong 5m. Runbook: [docs/alerts.md#alert-3](../docs/alerts.md#alert-3).

## 7. Điều tra challenge

- **Challenge ID:** `practice-rag_slow`
- **Khoảng thời gian điều tra:** `2026-09-30 03:04:45Z` đến `2026-09-30 03:05:15Z` (UTC)
- **Triệu chứng từ metrics:**
  Trên Dashboard panel Latency, chỉ số P95 tăng vọt từ 172ms lên 2687ms (+1462%), vượt xa ngưỡng cảnh báo 2000ms. Tuy nhiên panel Errors vẫn báo tỉ lệ lỗi 0% và retrieval success 100%, cho thấy hệ thống không sập nhưng bị nghẽn độ trễ nghiêm trọng.
- **Log line và correlation ID liên quan:**
  Lọc file `data/logs.jsonl` trong khung giờ trên, tìm thấy request bất thường tiêu biểu:
  `{"service": "api", "latency_ms": 2678, "ttft_ms": 59, "correlation_id": "req-ec6566b6", "user_id_hash": "105a9cef3903", "event": "response_sent", "ts": "2026-09-30T03:05:15.566532Z"}`
  Correlation ID: `req-ec6566b6`.
- **Trace ID và span gây ảnh hưởng:**
  Tìm kiếm trên Langfuse với `correlation_id="req-ec6566b6"` ra Trace ID `3a95f5ccfe0f4bc91689a3d70ba567a4`.
  So sánh waterfall giữa các span:
  - `lab-agent-run` (agent): 2.68s (2678ms)
  - `retrieval` (retriever): 2.51s (**Chiếm 93.6% tổng thời gian request**)
  - `generation` (generation): 169ms (Bình thường)
- **Root cause:**
  Độ trễ cao bắt nguồn trực tiếp từ tầng trích xuất tài liệu (vector store retrieval bị nghẽn / delay 2.5s theo kịch bản `rag_slow`), trong khi tầng sinh ngôn ngữ của LLM vẫn xử lý bình thường (173ms).
- **Fix action:**
  Vô hiệu hóa kịch bản sự cố bằng lệnh `python scripts/inject_incident.py --scenario rag_slow --disable`. Trong production: khởi động lại dịch vụ vector search, tối ưu hóa bộ nhớ đệm cache và mở rộng replica cho vector store.
- **Preventive measure:**
  Thiết lập timeout 1000ms cho retriever kèm fallback trả lời ngữ cảnh mặc định; kích hoạt cảnh báo `HighLatencyP95` để phát hiện nghẽn tầng dữ liệu trước khi vi phạm SLO.

## 8. Giải thích và tự đánh giá

- **Một quyết định kỹ thuật quan trọng và lý do:**
  Tách riêng biệt các child observations cho `retrieval` và `generation` trong Langfuse v4 thông qua context manager `start_as_current_observation()`. Nhờ đó, trace phản ánh chính xác cấu trúc thực thi hình cây, giúp cô lập được nguyên nhân gây chậm là do vector database chứ không phải do mô hình AI.
- **Một lỗi/blocker đã gặp:**
  Trong lần chạy đầu tiên, script `validate_logs.py` bị trừ điểm do file `data/logs.jsonl` còn lưu log cũ trước khi triển khai middleware và bộ scrub PII.
- **Cách tìm nguyên nhân và xử lý:**
  Phân tích mã nguồn validator và nhận thấy script đọc toàn bộ dữ liệu lịch sử. Giải pháp là lưu bản ghi baseline làm chứng cứ, sau đó xóa file log, khởi động lại server uvicorn và chạy lại test suite để đạt 100/100 điểm tuyệt đối.
- **Cách hiểu luồng Metrics → Logs → Traces:**
  - *Metrics* là radar cảnh báo tổng thể: chỉ ra hệ thống đang có triệu chứng gì bất thường và thời điểm xảy ra.
  - *Logs* là kính lúp: giúp chọn ra request cụ thể bị ảnh hưởng thông qua `correlation_id` và bối cảnh người dùng.
  - *Traces* là máy chụp X-quang: mổ xẻ chi tiết từng span trong request đó để chỉ điểm chính xác thành phần gây lỗi hoặc làm chậm hệ thống.
- **Vai trò của prompt version, token/cost, SLO hoặc rollback trong vận hành LLM:**
  Prompt là mã nguồn logic mới trong kỷ nguyên AI. Việc quản lý phiên bản prompt theo nhãn và có quy trình rollback nhanh giúp giảm thiểu rủi ro khi thử nghiệm các phiên bản tối ưu hóa. Giám sát token và chi phí ngăn ngừa hiện tượng lạm phát chi phí vận hành API. SLO định hình cam kết dịch vụ và giữ vững kỷ luật vận hành cho toàn bộ đội ngũ kỹ thuật.
- **Điều quan trọng nhất đã học:**
  Kỹ năng xây dựng hệ thống Full-Stack Observability chuyên biệt cho ứng dụng AI (LLMOps) theo chuẩn công nghiệp, kết hợp chặt chẽ giữa Structured Logging, Distributed Tracing và SRE Monitoring.
- **Hạn chế hoặc phần chưa hoàn thành, nếu có:**
  Đã hoàn thành toàn bộ 100% các mục tiêu, checkpoint và sản phẩm theo yêu cầu của đề bài lab.

## 9. Checklist trước khi nộp

- [x] Kết quả và evidence thuộc commit SHA cuối.
- [x] Tất cả ảnh/output mở được bằng đường dẫn tương đối.
- [x] Incident evidence nối đúng metric → log → trace.
- [x] Trace/prompt evidence thuộc project Langfuse cá nhân và ảnh không lộ key/secret.
- [x] Repository chạy lại được theo README.
- [x] Không có secret, API key, PII thô hoặc evidence của người khác/lớp khác.
- [x] URL repo và commit SHA cuối đã được nộp trên LMS/Codelabs.
