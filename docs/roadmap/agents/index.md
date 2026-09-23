# Track: Agents

**Agent** là hệ thống trong đó LLM tự quyết định các bước tiếp theo: gọi tool nào, với tham số gì, khi nào dừng. Agent mở ra những sản phẩm mà workflow cố định không làm được, nhưng cũng đem theo rủi ro: khó đoán, tốn kém, khó kiểm thử. Track này dạy bạn xây agent **đáng tin cậy**, và quan trọng không kém, biết **khi nào không nên dùng agent**.

| Bài | Nội dung | Thời lượng |
|---|---|---|
| [Bài 1: Tư duy agent](01-agent-loop.md) | Workflow hay agent, các mẫu thiết kế, giải phẫu agent, thiết kế tool, các kiểu thất bại | 4 đến 5 ngày |
| [Bài 2: Agent với LangChain](02-langchain-agents.md) | `create_agent`, tool, structured output, bộ nhớ, middleware, human-in-the-loop | 1 tuần |
| [Bài 3: Workflow với LangGraph](03-langgraph.md) | State, node, edge, persistence, interrupt, time travel; khi nào cần xuống tầng thấp | 1 tuần |
| [Bài 4: MCP](04-mcp.md) | Model Context Protocol: viết MCP server, kết nối từ agent, bảo mật | 4 đến 5 ngày |
| [Bài 5: Hệ thống nhiều agent](05-multi-agent.md) | Subagent, handoff, router; khi nào nhiều agent có lợi và khi nào có hại | 4 đến 5 ngày |

**Yêu cầu trước:** [Hiểu LLM, Bài 4: Tool use từ con số 0](../hieu-llm/04-tool-use.md) (bắt buộc), [Prompt & Context, Bài 3](../prompt-context/03-context-engineering.md) (nên có).

## Dự án xuyên suốt: trợ lý cửa hàng

Cả track xây dần một **trợ lý chăm sóc khách hàng cho cửa hàng online** (có thể áp dụng cho cửa hàng Shopify của bạn):

- Bài 2: agent tra đơn hàng, tư vấn sản phẩm, có bộ nhớ hội thoại, yêu cầu duyệt khi hoàn tiền.
- Bài 3: quy trình xử lý hoàn tiền nhiều bước bằng LangGraph, có điểm dừng chờ quản lý phê duyệt.
- Bài 4: tách các tool thành MCP server để dùng lại ở nhiều nơi (agent, Claude Desktop, IDE).
- Bài 5: tách thành các agent chuyên biệt khi hệ thống lớn lên.

!!! note "Liên kết với tài liệu đã dịch"
    Track này là lộ trình học, còn chi tiết API nằm ở phần tài liệu LangChain và LangGraph trong site. Mỗi bài dẫn tới các trang tương ứng; hãy đọc chúng song song.

---

[Lộ trình AI Engineer](../index.md)
