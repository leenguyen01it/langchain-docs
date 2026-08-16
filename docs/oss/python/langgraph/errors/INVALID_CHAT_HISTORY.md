# INVALID_CHAT_HISTORY

Lỗi này được ném ra trong [`create_agent`](https://reference.langchain.com/python/langchain/agents/factory/create_agent) dựng sẵn khi node `call_model` của graph nhận được một danh sách message bị sai định dạng. Cụ thể, danh sách bị coi là sai định dạng khi có `AIMessage` mang `tool_calls` (LLM yêu cầu gọi tool) nhưng không có [`ToolMessage`](https://reference.langchain.com/python/langchain-core/messages/tool/ToolMessage) tương ứng (kết quả gọi tool trả về cho LLM).

Có một vài lý do khiến bạn gặp lỗi này:

1. Bạn đã truyền thủ công một danh sách message sai định dạng khi invoke graph, ví dụ `graph.invoke({'messages': [AIMessage(..., tool_calls=[...])]})`
2. Graph bị interrupt trước khi nhận được cập nhật từ node `tools` (tức một danh sách [`ToolMessage`](https://reference.langchain.com/python/langchain-core/messages/tool/ToolMessage))
   và bạn invoke nó với input không phải None hoặc ToolMessage,
   ví dụ `graph.invoke({'messages': [HumanMessage(...)]}, config)`.
   Interrupt này có thể được kích hoạt theo một trong các cách sau:
   * Bạn tự đặt `interrupt_before = ['tools']` trong `create_agent`
   * Một trong các tool đã ném lỗi mà không được [`ToolNode`](https://reference.langchain.com/python/langgraph/agents/#langgraph.prebuilt.tool_node.ToolNode) (`"tools"`) xử lý

## Khắc phục sự cố

Để khắc phục, bạn có thể làm một trong các cách sau:

1. Không invoke graph với danh sách message sai định dạng
2. Trong trường hợp bị interrupt (thủ công hoặc do lỗi), bạn có thể:

* cung cấp các đối tượng [`ToolMessage`](https://reference.langchain.com/python/langchain-core/messages/tool/ToolMessage) khớp với các tool call hiện có và gọi `graph.invoke({'messages': [ToolMessage(...)]})`.
  **LƯU Ý**: cách này sẽ thêm các message vào lịch sử và chạy graph từ node START.
  * cập nhật state thủ công và resume graph từ điểm interrupt:
    1. lấy danh sách message gần nhất từ state của graph bằng `graph.get_state(config)`
    2. sửa danh sách message để loại bỏ các tool call chưa được trả lời khỏi AIMessage

hoặc thêm các đối tượng [`ToolMessage`](https://reference.langchain.com/python/langchain-core/messages/tool/ToolMessage) có `tool_call_ids` khớp với các tool call chưa được trả lời, sau đó gọi `graph.update_state(config, {'messages': ...})` với danh sách message đã sửa, rồi resume graph, ví dụ gọi `graph.invoke(None, config)`
