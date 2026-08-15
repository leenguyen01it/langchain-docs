# INVALID_PROMPT_INPUT

Xảy ra khi một [prompt template](https://github.com/langchain-ai/langchain/blob/v0.3/docs/docs/concepts/prompt_templates.mdx) nhận input variable bị thiếu hoặc không hợp lệ.

## Khắc phục sự cố

Để khắc phục lỗi này, bạn có thể:

1. Kiểm tra lại prompt template xem có đúng không. Khi dùng định dạng f-string, đảm bảo escape dấu ngoặc nhọn đúng cách:
    * Dùng `{{` cho dấu ngoặc đơn trong f-string
    * Dùng `{{{{` cho dấu ngoặc kép trong f-string
2. Khi dùng component `MessagesPlaceholder`, xác nhận bạn đang truyền mảng message hoặc object dạng message. Nếu dùng tuple viết tắt, bọc tên biến trong dấu ngoặc nhọn như `["placeholder", "{messages}"]`
3. Debug bằng cách kiểm tra input thực tế được truyền vào prompt template, dùng [LangSmith](https://docs.langchain.com/langsmith/observability) hoặc ghi log để xác nhận chúng đúng như kỳ vọng
4. Nếu lấy prompt từ LangChain [Prompt Hub](https://smith.langchain.com/prompts), hãy tách riêng và test prompt đó với input mẫu để đảm bảo nó hoạt động đúng ý

---

!!! info "Kết nối tài liệu này"
    [Kết nối tài liệu này](https://docs.langchain.com/use-these-docs) với Claude, VSCode, và nhiều công cụ khác qua MCP để có câu trả lời theo thời gian thực.

!!! info "Đóng góp"
    [Chỉnh sửa trang này trên GitHub](https://github.com/langchain-ai/docs/edit/main/src/oss/langchain/errors/INVALID_PROMPT_INPUT.mdx) hoặc [báo lỗi](https://github.com/langchain-ai/docs/issues/new/choose).
