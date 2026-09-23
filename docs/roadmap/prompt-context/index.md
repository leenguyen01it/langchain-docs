# Track: Prompt & Context Engineering

Prompt là **giao diện lập trình** của LLM. Cùng một model, prompt tốt và prompt tồi có thể cho chất lượng chênh lệch như hai sản phẩm khác nhau. Khi hệ thống phát triển thành agent chạy nhiều bước, câu hỏi không còn là "viết prompt thế nào" mà là "**đưa những gì vào context, vào lúc nào**". Đó là context engineering.

| Bài | Nội dung | Thời lượng |
|---|---|---|
| [Bài 1: Nguyên tắc viết prompt](01-nguyen-tac-prompt.md) | Rõ ràng, có ngữ cảnh, ví dụ, cấu trúc XML, định dạng output, viết prompt tiếng Việt | 4 đến 5 ngày |
| [Bài 2: Kỹ thuật nâng cao](02-ky-thuat-nang-cao.md) | Chuỗi prompt, định tuyến, tự đánh giá và sửa, trích xuất an toàn, dùng LLM cải thiện prompt | 4 đến 5 ngày |
| [Bài 3: Context engineering](03-context-engineering.md) | Context là tài nguyên khan hiếm, bốn chiến lược: ghi, chọn, nén, cô lập; áp dụng cho agent | 1 tuần |
| [Bài 4: Quản lý prompt như code](04-quan-ly-prompt.md) | Phiên bản hóa, template, cải thiện prompt dựa trên eval, phát hành an toàn | 3 đến 4 ngày |

**Yêu cầu trước:** [track Hiểu LLM](../hieu-llm/index.md), ít nhất Bài 3 (Claude API).

!!! quote "Tư duy cốt lõi"
    Hãy viết prompt như thể đang giao việc cho **một đồng nghiệp rất giỏi nhưng vừa mới vào công ty**: họ thông minh, nhưng không biết gì về sản phẩm, khách hàng, quy ước nội bộ của bạn, và không thể hỏi lại. Những gì bạn không viết ra, họ không biết.

Sau track này, bạn đã sẵn sàng cho [RAG](../rag/index.md) và [Agents](../agents/index.md).

---

[Lộ trình AI Engineer](../index.md)
