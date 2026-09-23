# Track: LLMOps

Đưa một demo lên production chỉ là bắt đầu. Hệ thống LLM trên production cần được **quan sát** (điều gì đang xảy ra), **tin cậy** (vẫn chạy khi nhà cung cấp gặp sự cố), **kinh tế** (chi phí nằm trong tầm kiểm soát), **an toàn** (không bị lợi dụng, không làm lộ dữ liệu), và **phát hành được an toàn** (thay đổi không làm hỏng trải nghiệm). Đó là LLMOps.

| Bài | Nội dung | Thời lượng |
|---|---|---|
| [Bài 1: Observability](01-observability.md) | Trace, span, những gì cần ghi lại, LangSmith, OpenTelemetry, dashboard và cảnh báo | 4 đến 5 ngày |
| [Bài 2: Độ tin cậy](02-do-tin-cay.md) | Timeout, retry, rate limit, fallback, circuit breaker, kiểm tra output, xuống cấp có kiểm soát | 4 đến 5 ngày |
| [Bài 3: Chi phí và độ trễ](03-chi-phi-do-tre.md) | Hồ sơ token, các đòn bẩy tiết kiệm theo thứ tự ưu tiên, giảm độ trễ, ngân sách và hạn mức | 4 đến 5 ngày |
| [Bài 4: Bảo mật](04-bao-mat.md) | OWASP Top 10 cho ứng dụng LLM, prompt injection, xử lý output, dữ liệu nhạy cảm, chuỗi cung ứng | 1 tuần |
| [Bài 5: Triển khai và vận hành](05-trien-khai.md) | Môi trường, cấu hình, chiến lược phát hành, mở rộng, xử lý sự cố, checklist ra mắt | 4 đến 5 ngày |

**Yêu cầu trước:** [Nền tảng kỹ thuật](../nen-tang-ky-thuat/index.md), [Hiểu LLM](../hieu-llm/index.md), và đã xây ít nhất một ứng dụng ở track [RAG](../rag/index.md) hoặc [Agents](../agents/index.md). [RAG Bài 8](../rag/08-production.md) là ví dụ áp dụng cụ thể cho RAG.

---

[Lộ trình AI Engineer](../index.md)
