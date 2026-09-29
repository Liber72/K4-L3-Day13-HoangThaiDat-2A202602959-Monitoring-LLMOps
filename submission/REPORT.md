# Báo cáo cá nhân — K4-L3A Day 13 Monitoring & LLMOps

> Mỗi học viên hoàn thiện một file duy nhất này. Khi dẫn evidence, dùng đường dẫn tương đối, ví dụ `evidence/07-trace-waterfall.png`.

## 1. Thông tin học viên

- **Họ và tên:**Hoàng Thái Đạt  
- **MSSV:**2A202602959  
- **Lớp:** K4-L3A
- **Repository URL:**https://github.com/Liber72/K4-L3-Day13-HoangThaiDat-2A202602959-Monitoring-LLMOps
- **Commit SHA cuối:**8735ce19fb396493e3cf57aca52ba88d4d364a82
- **Challenge ID:**K3-L3A-Challenge
- **Tên project Langfuse cá nhân:** `day13-k4-l3a-2A202602959`

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
| `validate_logs.py` |Estimated Score: 30/100|Estimated Score: 100/100| |
| `validate_dashboard.py` |HỢP LỆ: 6/6 panel |HỢP LỆ: 6/6 panel| |
| `pytest` |22 passed |22 passed| |
| Số traces hợp lệ |10|120| |
| Số PII leak |0|0||
| Latency P95 |967.25ms|2681.2ms|Hệ thống RAG giảm tốc độ khi số lượng request tăng lên do retrival|
| Retrieval success rate |100%|65%|Hệ thống RAG hiệu suất giảm khi có nhiều request đến cùng lúc|

## 4. Logging và PII

- **Cách tạo/nhận và truyền correlation ID:** Trong `app/middleware.py`, hệ thống ưu tiên lấy `x-request-id` từ Header HTTP. Nếu không có, nó sẽ tự sinh ra một mã mới dạng `req-<8-char-hex>` (dùng `uuid4`). Sau đó, mã này được gán vào context thông qua `bind_contextvars` của thư viện `structlog`, đảm bảo mọi dòng log tiếp theo trong cùng 1 request đều mang chung ID này. Cuối cùng, ID được trả về qua Header của Response.
- **Các metadata được ghi vào structured log:** Các trường mặc định do `structlog` tự thêm bao gồm: `ts` (thời gian theo chuẩn ISO), `level` (INFO, ERROR, v.v.), `correlation_id` (được truyền từ middleware). Ngoài ra, còn có `event` (tên sự kiện/hành động) và dictionary `payload` chứa thông tin bổ sung tùy ngữ cảnh.
- **Cách bảo đảm PII được scrub trước khi ghi:** Hệ thống sử dụng Regex pattern khai báo sẵn trong `app/pii.py` (bắt email, số điện thoại VN, CCCD, thẻ tín dụng, passport). Thư viện `structlog` được gắn thêm một processor tùy chỉnh là `scrub_event` trong file `logging_config.py`. Khi có lệnh in log, processor này sẽ tự động chặn lại, quét qua chuỗi văn bản và thay thế các dữ liệu nhạy cảm bằng chuỗi an toàn như `[REDACTED_EMAIL]` rồi mới ghi vào file `logs.jsonl`.
- **Cách kiểm chứng kết quả:** Chạy script kiểm thử bằng lệnh `python scripts/validate_logs.py data/logs.jsonl`. Script này sẽ quét toàn bộ file JSONL, đếm số lượng trace hợp lệ và thông báo lỗi (nếu có regex PII nào bị lọt ra ngoài). Ngoài ra có thể chạy `pytest` để pass các unit test liên quan đến PII.


## 5. Tracing và prompt versioning

- **Cách xác nhận traces do chính tôi tạo trong project cá nhân:** Các trace xuất hiện trên Project Dashboard của tôi trùng khớp với thời gian tôi gửi request và có chứa `correlation_id` do hệ thống local sinh ra.
- **Cấu trúc root/retrieval/generation observations:** Root là `lab-agent-run` (agent). Bên trong có 2 child spans: `retrieval` (span) và `llm-generation` (generation).
- **Cách nối trace với log:** Thông qua `correlation_id` được gán vào metadata của trace (trong `propagate_attributes`).
- **Prompt name:** `day13-chat`
- **Version/label baseline:** Version 1 / Label: `baseline`, `production`
- **Version/label candidate:** Version 2 / Label: `candidate`
- **Trace ID của mỗi version:** b7955b0a0dbedc623d1f9fda484c844c / 6ce1e064ca1e76de05f888680a75e53e
- **Cách promote và rollback `production`:** Vào giao diện Prompts trên Langfuse, chọn version cần dùng rồi bấm Add label -> gõ "production". Để rollback, tương tự đổi nhãn `production` quay về version 1.

## 6. Dashboard, SLO và alerts

- **Dashboard và sáu panel:** Gồm P 95 Latency by User, P 95 Latency by Level, Avg Time To First Token by Prompt Name, P 95 Latency by Model, Count Traffic, P 95 Total Cost (Observations).
- **SLO và lý do chọn:** Chọn SLO "fast_successful_requests" với mức Latency P95 <= 3000ms vì ứng dụng chat đòi hỏi phản hồi theo thời gian thực, độ trễ cao sẽ làm người dùng bỏ đi.
- **Cách tính error budget:** Ngân sách lỗi là 100% - 99.5% (Target) = 0.5% (tính trên chu kỳ 28 ngày).
- **Ba alert và runbook tương ứng:**
  - **High_Latency_P95 (CRITICAL - 5m):** Cảnh báo khi Latency P95 > 3000ms duy trì liên tục trong 5 phút. *Runbook:* 1. Kiểm tra Dashboard xem độ trễ ở LLM hay Retrieval. 2. Lọc log tìm request chậm. 3. Dán `correlation_id` vào Langfuse xem Trace. (Mitigation: Chuyển model nhỏ hoặc bật cache).
  - **High_Error_Rate (CRITICAL - 5m):** Cảnh báo khi Tỷ lệ lỗi > 2% duy trì trong 5 phút. *Runbook:* 1. Mở Dashboard xem loại lỗi. 2. Tra logs xem lỗi tool nào. 3. Xem Trace Langfuse check Timeout/Rate Limit. (Mitigation: Fallback prompt hoặc tắt tool lỗi).
  - **Retrieval_Failure_Rate (WARNING - 10m):** Cảnh báo khi Tỷ lệ tìm kiếm thành công < 90% trong 10 phút. *Runbook:* 1. Xem Dashboard tìm lượt truy xuất fail. 2. Lấy `correlation_id` từ log. 3. Kiểm tra mạng/Vector Store. (Mitigation: Bypass RAG, dùng kiến thức nội bộ).


## 7. Điều tra challenge

- **Challenge ID:**K4-L3A-Challenge
- **Khoảng thời gian điều tra:**16:00
- **Triệu chứng từ metrics:**  Panel Latency P95 trên Dashboard bất ngờ vọt lên mức ~2.6 giây, vi phạm SLO.
- **Log line và correlation ID liên quan:**`{"event": "response_sent", "latency_ms": 2673, "correlation_id": "req-3b6c06cd"...}`
- **Trace ID và span gây ảnh hưởng:**Trace ID `9694098cd58020d261a1a90299ef9dbd`, nhánh span `retrival` tăng lên 2.51s 
- **Root cause:** RAG bị chậm khiến việc truy suất tài liệu bị treo .5s mỗi lượt.
- **Fix action:** Scale up server Vector DB hoặc kiểm tra kết nối mạng giữa Backend và Vector DB. Tạm thời bật Cache nếu câu hỏi trùng lặp.
- **Preventive measure:** Thêm bộ đếm Timeout ngắn hơn cho hàm retrieval.

## 8. Giải thích và tự đánh giá

- **Một quyết định kỹ thuật quan trọng và lý do:** Quyết định sử dụng `structlog` kết hợp `bind_contextvars` ở tầng Middleware để xử lý `correlation_id`. Lý do: Cách tiếp cận này giúp gán ID cho toàn bộ vòng đời của một request một cách tự động, giữ cho code nghiệp vụ (business logic) ở các hàm bên trong được sạch sẽ, không phải truyền `correlation_id` thủ công qua từng hàm.
- **Một lỗi/blocker đã gặp:** Không lấy prompt version mới trên langfuse
- **Cách tìm nguyên nhân và xử lý:** thử sinh version mới, kiểm tra điểm kết nối giữa project và langfuse. Và tìm ra vấn đề là do đặt tên dự án Prompt mới sai với biến khai báo trên dự án.
- **Cách hiểu luồng Metrics → Logs → Traces:** Metrics  đóng vai trò cảnh báo bề mặt. Từ đó, ta tìm kiếm Logs trong khoảng thời gian đó để lấy `correlation_id`. Cuối cùng, mang `correlation_id` đó vào hệ thống Traces trên Langfuse để nhìn xuyên thấu vào bên trong xem chính xác Span nào đang chạy chậm.
- **Vai trò của prompt version, token/cost, SLO hoặc rollback trong vận hành LLM:** Quản lý vòng đời AI an toàn hơn. Tách bạch code khỏi prompt (DevOps có thể đổi prompt qua Label không cần deploy lại code). SLO giúp định lượng trải nghiệm người dùng, còn Rollback giúp hệ thống "quay xe" lập tức về version cũ nếu prompt mới sinh ra ảo giác hoặc tốn quá nhiều cost.
- **Điều quan trọng nhất đã học:** Tư duy Observability (Khả năng quan sát). Đưa một mô hình LLM ra production không chỉ là gọi API thành công, mà phải giám sát chặt chẽ được Độ trễ (đặc biệt là TTFT), Chi phí và Tỷ lệ lỗi để không bị "mù" khi hệ thống gặp sự cố.
- **Hạn chế hoặc phần chưa hoàn thành, nếu có:** Việc rollback hiện tại vẫn phải làm thủ công trên giao diện Langfuse. Trong tương lai mong muốn có thể cấu hình Webhook để tự động Rollback (Automated Rollback) ngay khi hệ thống Alert vi phạm SLO.


## 9. Checklist trước khi nộp

- [x] Kết quả và evidence thuộc commit SHA cuối.
- [x] Tất cả ảnh/output mở được bằng đường dẫn tương đối.
- [x] Incident evidence nối đúng metric → log → trace.
- [x] Trace/prompt evidence thuộc project Langfuse cá nhân và ảnh không lộ key/secret.
- [x] Repository chạy lại được theo README.
- [x] Không có secret, API key, PII thô hoặc evidence của người khác/lớp khác.
- [x] URL repo và commit SHA cuối đã được nộp trên LMS/Codelabs.
