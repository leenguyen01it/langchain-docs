# INVALID_TOOL_RESULTS

!!! note "Ghi chú"
    Hiện chỉ dùng trong `langchainjs` (JavaScript/TypeScript).

Lỗi này xảy ra khi truyền các object [`ToolMessage`](https://reference.langchain.com/python/langchain-core/messages/tool/ToolMessage) không khớp, thiếu, hoặc dư vào model trong các thao tác gọi tool.

Lỗi này bắt nguồn từ một yêu cầu cơ bản: một assistant message có `tool_calls` phải được theo sau bởi các tool message phản hồi cho từng `tool_call_id`.

Khi một model trả về một [`AIMessage`](https://reference.langchain.com/python/langchain-core/messages/ai/AIMessage) kèm tool call, bạn phải cung cấp đúng một [`ToolMessage`](https://reference.langchain.com/python/langchain-core/messages/tool/ToolMessage) tương ứng cho mỗi tool call, với giá trị `tool_call_id` khớp nhau.

## Nguyên nhân phổ biến

* **Phản hồi thiếu**: Nếu model yêu cầu hai lần thực thi tool nhưng bạn chỉ cung cấp một message phản hồi, model sẽ từ chối chuỗi message không đầy đủ
* **Phản hồi trùng lặp**: Cung cấp nhiều object [`ToolMessage`](https://reference.langchain.com/python/langchain-core/messages/tool/ToolMessage) cho cùng một tool call ID sẽ bị từ chối, tương tự với ID không khớp
* **Tool message mồ côi**: Gửi một [`ToolMessage`](https://reference.langchain.com/python/langchain-core/messages/tool/ToolMessage) mà không có [`AIMessage`](https://reference.langchain.com/python/langchain-core/messages/ai/AIMessage) chứa tool call đứng trước sẽ vi phạm yêu cầu giao thức (protocol)

Đây là ví dụ về một pattern có vấn đề:

```python
# Model yêu cầu hai tool call
response_message.tool_calls  # Trả về 2 call

# Nhưng chỉ có một ToolMessage được cung cấp
chat_history.append(ToolMessage(
    content=str(tool_response),
    tool_call_id=tool_call.get("id")
))

model_with_tools.invoke(chat_history)
```

## Khắc phục sự cố

Để khắc phục lỗi này:

* **Đếm số cặp khớp nhau**: Đảm bảo có một [`ToolMessage`](https://reference.langchain.com/python/langchain-core/messages/tool/ToolMessage) tồn tại cho mỗi tool call trong [`AIMessage`](https://reference.langchain.com/python/langchain-core/messages/ai/AIMessage) đứng trước
* **Kiểm tra ID**: Xác nhận mỗi `ToolMessage.tool_call_id` khớp với một tool call identifier thực tế

---

!!! info "Kết nối tài liệu này"
    [Kết nối tài liệu này](https://docs.langchain.com/use-these-docs) với Claude, VSCode, và nhiều công cụ khác qua MCP để có câu trả lời theo thời gian thực.

!!! info "Đóng góp"
    [Chỉnh sửa trang này trên GitHub](https://github.com/langchain-ai/docs/edit/main/src/oss/langchain/errors/INVALID_TOOL_RESULTS.mdx) hoặc [báo lỗi](https://github.com/langchain-ai/docs/issues/new/choose).
