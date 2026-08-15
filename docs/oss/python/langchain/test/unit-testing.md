# Unit testing

> Kiểm thử logic agent mà không cần gọi API, dùng fake chat model và in-memory persistence.

Unit test kiểm tra các phần nhỏ, deterministic của agent một cách độc lập. Bằng cách thay LLM thật bằng một in-memory fake (còn gọi là fixture), bạn có thể script chính xác các phản hồi (text, tool call, và lỗi) để test chạy nhanh, miễn phí, và lặp lại được mà không cần API key.

## Mock chat model

LangChain cung cấp [`GenericFakeChatModel`](https://reference.langchain.com/python/langchain-core/language_models/fake_chat_models/GenericFakeChatModel) để mock phản hồi văn bản. Nó nhận một iterator các phản hồi (đối tượng [`AIMessage`](https://reference.langchain.com/python/langchain-core/messages/ai/AIMessage) hoặc string) và trả về một phản hồi cho mỗi lần invoke. Nó hỗ trợ cả cách dùng thông thường lẫn streaming.

```python
from langchain_core.language_models.fake_chat_models import GenericFakeChatModel

model = GenericFakeChatModel(messages=iter([
    AIMessage(content="", tool_calls=[ToolCall(name="foo", args={"bar": "baz"}, id="call_1")]),
    "bar"
]))

model.invoke("hello")
# AIMessage(content='', ..., tool_calls=[{'name': 'foo', 'args': {'bar': 'baz'}, 'id': 'call_1', 'type': 'tool_call'}])
```

Nếu chúng ta invoke model lần nữa, nó sẽ trả về item tiếp theo trong iterator:

```python
model.invoke("hello, again!")
# AIMessage(content='bar', ...)
```

## InMemorySaver checkpointer

Để bật persistence trong quá trình test, bạn có thể dùng checkpointer [`InMemorySaver`](https://reference.langchain.com/python/langgraph/checkpoints/#langgraph.checkpoint.memory.InMemorySaver). Điều này cho phép bạn mô phỏng nhiều lượt để kiểm thử hành vi phụ thuộc state:

```python
from langgraph.checkpoint.memory import InMemorySaver

agent = create_agent(
    model,
    tools=[],
    checkpointer=InMemorySaver()
)

# Lần invoke đầu tiên
agent.invoke(
    {"messages": [HumanMessage(content="I live in Sydney, Australia")]},
    config={"configurable": {"thread_id": "session-1"}}
)

# Lần invoke thứ hai: message đầu tiên đã được persist (vị trí Sydney), nên model trả về giờ GMT+10
agent.invoke(
    {"messages": [HumanMessage(content="What's my local time?")]},
    config={"configurable": {"thread_id": "session-1"}}
)
```

## Bước tiếp theo

Tìm hiểu cách kiểm thử agent với API provider model thật trong [Integration testing](integration-testing.md).
