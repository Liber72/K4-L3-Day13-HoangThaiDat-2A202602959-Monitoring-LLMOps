# Template Alert và Runbook

Mỗi alert phải dựa trên triệu chứng người dùng hoặc SLO, không dựa trực tiếp vào tên implementation nội bộ.

## Alert 1

- Tên: High_Latency_P95
- Severity: CRITICAL
- Duration: 5m
- Kênh thông báo: Slack
- SLI/SLO liên quan: fast_successful_requests (Latency P95 <= 3000ms)
- Điều kiện và thời gian duy trì: Độ trễ P95 > 3000ms duy trì liên tục trong 5 phút.
- Ảnh hưởng tới người dùng: Người dùng phải đợi rất lâu mới nhận được phản hồi từ AI, gây bực bội và có thể bỏ phiên chat.
- Ba bước kiểm tra đầu tiên: 
  1. Kiểm tra Dashboard xem độ trễ tăng ở LLM hay Retrieval.
  2. Lọc file `data/logs.jsonl` tìm các request có `latency_ms` > 3000.
  3. Lấy `correlation_id` dán vào Langfuse để xem Trace chi tiết.
- Mitigation tạm thời: Tạm thời chuyển model sang model nhỏ hơn hoặc cache câu trả lời nếu hệ thống quá tải.
- Owner: SRE_Team

## Alert 2

- Tên: High_Error_Rate
- Severity: CRITICAL
- Duration: 5m
- Kênh thông báo: Slack
- SLI/SLO liên quan: error_rate_pct_max <= 2%
- Điều kiện và thời gian duy trì: Tỷ lệ lỗi > 2% duy trì trong 5 phút.
- Ảnh hưởng tới người dùng: Ứng dụng liên tục trả về lỗi 500, người dùng không thể chat được.
- Ba bước kiểm tra đầu tiên:
  1. Mở Dashboard xem loại lỗi (error_type) nào đang tăng cao.
  2. Tra logs xem lỗi xảy ra ở bước tool nào (tool_success=False).
  3. Kiểm tra Langfuse Trace xem LLM có bị lỗi Timeout hay Rate Limit không.
- Mitigation tạm thời: Bật tính năng fallback prompt hoặc tắt tạm các công cụ bị lỗi.
- Owner: SRE_Team

## Alert 3

- Tên: Retrieval_Failure_Rate
- Severity: WARNING
- Duration: 10m
- Kênh thông báo: Slack
- SLI/SLO liên quan: retrieval_success_rate_pct_min >= 90%
- Điều kiện và thời gian duy trì: Tỷ lệ tìm kiếm tài liệu thành công (tool_success_rate) < 90% trong 10 phút.
- Ảnh hưởng tới người dùng: AI trả lời chung chung, thiếu bối cảnh (No domain document matched) do không tìm thấy tài liệu liên quan.
- Ba bước kiểm tra đầu tiên:
  1. Xem Dashboard tìm xem số lượt truy xuất (retrieval) thất bại.
  2. Lấy `correlation_id` từ log của các request bị `tool_success=False`.
  3. Dò lại Vector Store hoặc dịch vụ Search xem có bị quá tải / sập mạng không.
- Mitigation tạm thời: Nếu Vector DB bị lỗi, có thể bypass tính năng RAG để AI dùng kiến thức nội bộ trả lời tạm (fallback).
- Owner: Backend_Team
