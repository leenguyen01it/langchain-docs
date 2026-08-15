# Tổng quan

> Kiểm soát và tùy chỉnh việc thực thi agent ở mọi bước

Middleware cung cấp một cách để kiểm soát chặt chẽ hơn những gì xảy ra bên trong agent. Middleware hữu ích cho những việc sau:

* Theo dõi hành vi của agent bằng logging, phân tích (analytics), và debug.
* Biến đổi (transform) prompt, [lựa chọn tool](built-in.md#llm-tool-selector), và định dạng output.
* Thêm [cơ chế retry](built-in.md#tool-retry), [fallback](built-in.md#model-fallback), và logic dừng sớm (early termination).
* Áp dụng [giới hạn tốc độ (rate limit)](built-in.md#model-call-limit), guardrail, và [phát hiện PII](built-in.md#pii-detection).

Thêm middleware bằng cách truyền chúng vào [`create_agent`](https://reference.langchain.com/python/langchain/agents/factory/create_agent):

```python
from langchain.agents import create_agent
from langchain.agents.middleware import SummarizationMiddleware, HumanInTheLoopMiddleware

agent = create_agent(
    model="gpt-5.5",
    tools=[...],
    middleware=[
        SummarizationMiddleware(...),
        HumanInTheLoopMiddleware(...)
    ],
)
```

## Vòng lặp agent

Vòng lặp agent cốt lõi bao gồm việc gọi model, để model chọn tool cần thực thi, rồi kết thúc khi model không gọi thêm tool nào nữa:

<img src="https://mintcdn.com/langchain-5e9cc07a/Tazq8zGc0yYUYrDl/oss/images/core_agent_loop.png?fit=max&auto=format&n=Tazq8zGc0yYUYrDl&q=85&s=ac72e48317a9ced68fd1be64e89ec063" alt="Sơ đồ vòng lặp agent cốt lõi" style="height:200px;width:auto;justify-content:center" class="rounded-lg block mx-auto" width="300" height="268" />

Middleware cung cấp các hook trước và sau mỗi bước đó:

<img src="https://mintcdn.com/langchain-5e9cc07a/RAP6mjwE5G00xYsA/oss/images/middleware_final.png?fit=max&auto=format&n=RAP6mjwE5G00xYsA&q=85&s=eb4404b137edec6f6f0c8ccb8323eaf1" alt="Sơ đồ luồng middleware" style="height:300px;width:auto;justify-content:center" class="rounded-lg mx-auto" width="500" height="560" />

## Sử dụng middleware bên trong một workflow LangGraph

Middleware không phải là một runtime riêng biệt: các hook chạy bên trong [LangGraph](https://docs.langchain.com/oss/python/langgraph/overview) đã được compile mà [`create_agent`](https://reference.langchain.com/python/langchain/agents/factory/create_agent) trả về. Bạn có thể đưa toàn bộ agent (kèm theo middleware) vào một [StateGraph](https://reference.langchain.com/python/langgraph/graph/state/StateGraph) lớn hơn dưới dạng một node hoặc subgraph, và mọi hook middleware vẫn tiếp tục hoạt động.

Hãy dùng pattern này khi cấu trúc (topology) xung quanh phức tạp hơn một vòng lặp tiêu chuẩn kiểu "lặp cho đến khi xong": phân loại input trước khi định tuyến đến một trong nhiều agent, phân tán (fan out) công việc song song, hoặc kết nối các lệnh gọi agent với nhau bằng các bước xử lý tất định (deterministic).

`HumanInTheLoopMiddleware` so khớp (match) dựa trên `.name` của từng tool.

Các hàm được decorate bằng `@tool` lấy tên từ chính hàm đó, vì vậy key bên dưới là `"send_email"`.

```python
from langchain.agents import AgentState, create_agent
from langchain.agents.middleware import HumanInTheLoopMiddleware
from langgraph.graph import START, StateGraph

# Giả định rằng read_email, send_email, classify_node, và route đã được định nghĩa ở nơi khác.
email_agent = create_agent(
    model="claude-sonnet-4-6",
    tools=[read_email, send_email],
    middleware=[HumanInTheLoopMiddleware(interrupt_on={"send_email": True})],
)

graph = (
    StateGraph(AgentState)
    .add_node("classify", classify_node)
    .add_node("email_agent", email_agent)
    .add_edge(START, "classify")
    .add_conditional_edges("classify", route)
    .compile()
)
```

Interrupt của HITL, summarization, redaction PII, retry, và mọi hook tùy chỉnh đều đi kèm với node agent. Xem [Use subgraphs](https://docs.langchain.com/oss/python/langgraph/use-subgraphs) để biết đầy đủ các pattern kết hợp (composition pattern), bao gồm cả phạm vi (scoping) checkpointer của subgraph (theo từng lần gọi so với theo từng thread).

## Tài nguyên bổ sung

**[Built-in middleware](built-in.md)**
Khám phá middleware có sẵn (built-in) cho các use case phổ biến.

**[Custom middleware](custom.md)**
Xây dựng middleware của riêng bạn với hook và decorator.

**[Middleware API reference](https://reference.langchain.com/python/langchain/middleware/)**
Tài liệu tham khảo API đầy đủ cho middleware.

**[Middleware integrations](https://docs.langchain.com/oss/python/integrations/middleware/)**
Middleware dành riêng cho nhà cung cấp (provider-specific) như Anthropic, AWS, OpenAI, và nhiều hơn nữa.

**[Testing agents](../test/index.md)**
Kiểm thử (test) các agent của bạn với LangSmith.

---

[Kết nối tài liệu này](https://docs.langchain.com/use-these-docs) với Claude, VSCode, và nhiều công cụ khác qua MCP để nhận câu trả lời theo thời gian thực.

[Chỉnh sửa trang này trên GitHub](https://github.com/langchain-ai/docs/edit/main/src/oss/langchain/middleware/overview.mdx) hoặc [báo lỗi (file an issue)](https://github.com/langchain-ai/docs/issues/new/choose).
