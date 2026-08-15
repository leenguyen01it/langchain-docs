# Streaming

> Stream cập nhật theo thời gian thực từ các lần chạy agent

!!! tip "Mẹo"
    Với ứng dụng mới, chúng tôi khuyến nghị dùng [event streaming](event-streaming.md), API typed-projection được giới thiệu trong LangChain v1.3. Event streaming cung cấp các iterator riêng biệt cho từng projection (message, value, tool call, subgraph) để bạn tiêu thụ độc lập thay vì phải rẽ nhánh theo chunk `stream_mode`.

LangChain triển khai một hệ thống streaming để hiển thị cập nhật theo thời gian thực.

Streaming rất quan trọng để tăng khả năng phản hồi của ứng dụng xây dựng trên LLM. Bằng cách hiển thị output dần dần, ngay cả trước khi có phản hồi hoàn chỉnh, streaming cải thiện đáng kể trải nghiệm người dùng (UX), đặc biệt khi phải đối mặt với độ trễ của LLM.

## Tổng quan

Hệ thống streaming của LangChain cho phép bạn hiển thị phản hồi trực tiếp từ các lần chạy agent tới ứng dụng của bạn.

Những gì có thể làm được với streaming của LangChain:

* [**Stream tiến trình agent**](#tien-trinh-agent): nhận cập nhật state sau mỗi bước agent.
* [**Stream token LLM**](#token-llm): stream token của language model ngay khi được sinh ra.
* [**Stream token thinking / reasoning**](#stream-token-thinking--reasoning): hiển thị quá trình suy luận của model ngay khi được sinh ra.
* [**Stream cập nhật tuỳ chỉnh**](#cap-nhat-tuy-chinh): phát ra tín hiệu do người dùng định nghĩa (ví dụ: `"Fetched 10/100 records"`).
* [**Stream nhiều mode**](#stream-nhieu-mode): chọn giữa `updates` (tiến trình agent), `messages` (token LLM + metadata), hoặc `custom` (dữ liệu tuỳ ý của người dùng).

Xem phần [pattern thường gặp](#cac-pattern-thuong-gap) bên dưới để có thêm ví dụ end-to-end.

## Các stream mode được hỗ trợ

Truyền một hoặc nhiều mode sau dưới dạng list vào phương thức [`stream`](https://reference.langchain.com/python/langgraph/graphs/#langgraph.graph.state.CompiledStateGraph.stream) hoặc [`astream`](https://reference.langchain.com/python/langgraph/graphs/#langgraph.graph.state.CompiledStateGraph.astream):

| Mode | Mô tả |
| --- | --- |
| `updates` | Stream cập nhật state sau mỗi bước agent. Nếu nhiều cập nhật xảy ra trong cùng một bước (ví dụ nhiều node chạy), các cập nhật đó được stream riêng biệt. |
| `messages` | Stream các tuple `(token, metadata)` từ bất kỳ node graph nào có LLM được invoke. |
| `custom` | Stream dữ liệu tuỳ chỉnh từ bên trong node graph của bạn bằng stream writer. |

## Tiến trình agent

Để stream tiến trình agent, dùng phương thức [`stream`](https://reference.langchain.com/python/langgraph/graphs/#langgraph.graph.state.CompiledStateGraph.stream) hoặc [`astream`](https://reference.langchain.com/python/langgraph/graphs/#langgraph.graph.state.CompiledStateGraph.astream) với `stream_mode="updates"`. Việc này phát ra một sự kiện sau mỗi bước agent.

Ví dụ, nếu bạn có một agent gọi một tool một lần, bạn sẽ thấy các cập nhật sau:

* **Node LLM**: [`AIMessage`](https://reference.langchain.com/python/langchain-core/messages/ai/AIMessage) với yêu cầu tool call
* **Node Tool**: [`ToolMessage`](https://reference.langchain.com/python/langchain-core/messages/tool/ToolMessage) với kết quả thực thi
* **Node LLM**: phản hồi AI cuối cùng

Truyền `thread_id` qua `config` để cuộc hội thoại được checkpoint và các lượt tiếp theo có thể tiếp tục cùng lịch sử. `thread_id` độc lập với `stream_mode`; bạn cũng có thể truyền `context` cùng với nó cho dữ liệu riêng theo lần chạy mà tool của bạn đọc từ `runtime.context`.

=== "Google"

    ```python
    from langchain.agents import create_agent
    from langchain_core.utils.uuid import uuid7
    from langgraph.checkpoint.memory import InMemorySaver

    def get_weather(city: str) -> str:
        """Get weather for a given city."""
        return f"It's always sunny in {city}!"

    agent = create_agent(
        model="google_genai:gemini-3.6-flash",
        tools=[get_weather],
        checkpointer=InMemorySaver()
    )
    config = {"configurable": {"thread_id": str(uuid7())}}
    stream = agent.stream_events(  # [!code highlight]
        {"messages": [{"role": "user", "content": "What is the weather in SF?"}]},
        config=config,
        version="v3",  # [!code highlight]
    )
    for kind, item in stream.interleave("messages", "tool_calls"):  # [!code highlight]
        if kind == "messages":
            for token in item.text:
                print(token, end="", flush=True)
        elif kind == "tool_calls":
            print(f"\nTool call: {item.tool_name}({item.input})")
            for delta in item.output_deltas:
                print(delta, end="", flush=True)
            print(f"\nTool result: {item.output}")

    final_state = stream.output  # [!code highlight]
    ```

=== "OpenAI"

    ```python
    from langchain.agents import create_agent
    from langchain_core.utils.uuid import uuid7
    from langgraph.checkpoint.memory import InMemorySaver

    def get_weather(city: str) -> str:
        """Get weather for a given city."""
        return f"It's always sunny in {city}!"

    agent = create_agent(
        model="openai:gpt-5.5",
        tools=[get_weather],
        checkpointer=InMemorySaver()
    )
    config = {"configurable": {"thread_id": str(uuid7())}}
    stream = agent.stream_events(  # [!code highlight]
        {"messages": [{"role": "user", "content": "What is the weather in SF?"}]},
        config=config,
        version="v3",  # [!code highlight]
    )
    for kind, item in stream.interleave("messages", "tool_calls"):  # [!code highlight]
        if kind == "messages":
            for token in item.text:
                print(token, end="", flush=True)
        elif kind == "tool_calls":
            print(f"\nTool call: {item.tool_name}({item.input})")
            for delta in item.output_deltas:
                print(delta, end="", flush=True)
            print(f"\nTool result: {item.output}")

    final_state = stream.output  # [!code highlight]
    ```

=== "Anthropic"

    ```python
    from langchain.agents import create_agent
    from langchain_core.utils.uuid import uuid7
    from langgraph.checkpoint.memory import InMemorySaver

    def get_weather(city: str) -> str:
        """Get weather for a given city."""
        return f"It's always sunny in {city}!"

    agent = create_agent(
        model="anthropic:claude-sonnet-4-6",
        tools=[get_weather],
        checkpointer=InMemorySaver()
    )
    config = {"configurable": {"thread_id": str(uuid7())}}
    stream = agent.stream_events(  # [!code highlight]
        {"messages": [{"role": "user", "content": "What is the weather in SF?"}]},
        config=config,
        version="v3",  # [!code highlight]
    )
    for kind, item in stream.interleave("messages", "tool_calls"):  # [!code highlight]
        if kind == "messages":
            for token in item.text:
                print(token, end="", flush=True)
        elif kind == "tool_calls":
            print(f"\nTool call: {item.tool_name}({item.input})")
            for delta in item.output_deltas:
                print(delta, end="", flush=True)
            print(f"\nTool result: {item.output}")

    final_state = stream.output  # [!code highlight]
    ```

=== "OpenRouter"

    ```python
    from langchain.agents import create_agent
    from langchain_core.utils.uuid import uuid7
    from langgraph.checkpoint.memory import InMemorySaver

    def get_weather(city: str) -> str:
        """Get weather for a given city."""
        return f"It's always sunny in {city}!"

    agent = create_agent(
        model="openrouter:z-ai/glm-5.2",
        tools=[get_weather],
        checkpointer=InMemorySaver()
    )
    config = {"configurable": {"thread_id": str(uuid7())}}
    stream = agent.stream_events(  # [!code highlight]
        {"messages": [{"role": "user", "content": "What is the weather in SF?"}]},
        config=config,
        version="v3",  # [!code highlight]
    )
    for kind, item in stream.interleave("messages", "tool_calls"):  # [!code highlight]
        if kind == "messages":
            for token in item.text:
                print(token, end="", flush=True)
        elif kind == "tool_calls":
            print(f"\nTool call: {item.tool_name}({item.input})")
            for delta in item.output_deltas:
                print(delta, end="", flush=True)
            print(f"\nTool result: {item.output}")

    final_state = stream.output  # [!code highlight]
    ```

=== "Fireworks"

    ```python
    from langchain.agents import create_agent
    from langchain_core.utils.uuid import uuid7
    from langgraph.checkpoint.memory import InMemorySaver

    def get_weather(city: str) -> str:
        """Get weather for a given city."""
        return f"It's always sunny in {city}!"

    agent = create_agent(
        model="fireworks:accounts/fireworks/models/glm-5p2",
        tools=[get_weather],
        checkpointer=InMemorySaver()
    )
    config = {"configurable": {"thread_id": str(uuid7())}}
    stream = agent.stream_events(  # [!code highlight]
        {"messages": [{"role": "user", "content": "What is the weather in SF?"}]},
        config=config,
        version="v3",  # [!code highlight]
    )
    for kind, item in stream.interleave("messages", "tool_calls"):  # [!code highlight]
        if kind == "messages":
            for token in item.text:
                print(token, end="", flush=True)
        elif kind == "tool_calls":
            print(f"\nTool call: {item.tool_name}({item.input})")
            for delta in item.output_deltas:
                print(delta, end="", flush=True)
            print(f"\nTool result: {item.output}")

    final_state = stream.output  # [!code highlight]
    ```

=== "Baseten"

    ```python
    from langchain.agents import create_agent
    from langchain_core.utils.uuid import uuid7
    from langgraph.checkpoint.memory import InMemorySaver

    def get_weather(city: str) -> str:
        """Get weather for a given city."""
        return f"It's always sunny in {city}!"

    agent = create_agent(
        model="baseten:zai-org/GLM-5.2",
        tools=[get_weather],
        checkpointer=InMemorySaver()
    )
    config = {"configurable": {"thread_id": str(uuid7())}}
    stream = agent.stream_events(  # [!code highlight]
        {"messages": [{"role": "user", "content": "What is the weather in SF?"}]},
        config=config,
        version="v3",  # [!code highlight]
    )
    for kind, item in stream.interleave("messages", "tool_calls"):  # [!code highlight]
        if kind == "messages":
            for token in item.text:
                print(token, end="", flush=True)
        elif kind == "tool_calls":
            print(f"\nTool call: {item.tool_name}({item.input})")
            for delta in item.output_deltas:
                print(delta, end="", flush=True)
            print(f"\nTool result: {item.output}")

    final_state = stream.output  # [!code highlight]
    ```

=== "Ollama"

    ```python
    from langchain.agents import create_agent
    from langchain_core.utils.uuid import uuid7
    from langgraph.checkpoint.memory import InMemorySaver

    def get_weather(city: str) -> str:
        """Get weather for a given city."""
        return f"It's always sunny in {city}!"

    agent = create_agent(
        model="ollama:north-mini-code-1.0",
        tools=[get_weather],
        checkpointer=InMemorySaver()
    )
    config = {"configurable": {"thread_id": str(uuid7())}}
    stream = agent.stream_events(  # [!code highlight]
        {"messages": [{"role": "user", "content": "What is the weather in SF?"}]},
        config=config,
        version="v3",  # [!code highlight]
    )
    for kind, item in stream.interleave("messages", "tool_calls"):  # [!code highlight]
        if kind == "messages":
            for token in item.text:
                print(token, end="", flush=True)
        elif kind == "tool_calls":
            print(f"\nTool call: {item.tool_name}({item.input})")
            for delta in item.output_deltas:
                print(delta, end="", flush=True)
            print(f"\nTool result: {item.output}")

    final_state = stream.output  # [!code highlight]
    ```

```shell title="Output"
step: model
content: [{'type': 'tool_call', 'name': 'get_weather', 'args': {'city': 'San Francisco'}, 'id': 'call_9lBtsDbmmobzyA8xc4I4Ctne'}]
step: tools
content: [{'type': 'text', 'text': "It's always sunny in San Francisco!"}]
step: model
content: [{'type': 'text', 'text': "San Francisco weather: It's always sunny in San Francisco!\n\nIf you'd like the exact current conditions (temperature, humidity, wind) and a short forecast, I can fetch that next. Would you like me to pull live details for San Francisco?"}]
```

!!! note "Ghi chú"
    Việc lưu lịch sử hội thoại bằng `thread_id` yêu cầu agent được cấu hình với một [checkpointer](long-term-memory.md). Trên các [LangSmith deployment](https://docs.langchain.com/langsmith/deployment), checkpointer được cấp phát tự động. Khi chạy local, hãy truyền tường minh, ví dụ `create_agent(..., checkpointer=InMemorySaver())`. Các đoạn code còn lại trong trang này bỏ qua `thread_id` cho ngắn gọn, nhưng bạn nên truyền nó trong production.

## Token LLM

Để stream token khi LLM sinh ra, dùng `stream_mode="messages"`. Dưới đây là output của agent stream tool call và phản hồi cuối cùng.

```python title="Streaming LLM tokens"
from langchain.agents import create_agent


def get_weather(city: str) -> str:
    """Get weather for a given city."""

    return f"It's always sunny in {city}!"

agent = create_agent(
    model="gpt-5-nano",
    tools=[get_weather],
)
for chunk in agent.stream(  # [!code highlight]
    {"messages": [{"role": "user", "content": "What is the weather in SF?"}]},
    stream_mode="messages",
    version="v2",  # [!code highlight]
):
    if chunk["type"] == "messages":  # [!code highlight]
        token, metadata = chunk["data"]  # [!code highlight]
        print(f"node: {metadata['langgraph_node']}")
        print(f"content: {token.content_blocks}")
        print("\n")
```

```shell title="Output"
node: model
content: [{'type': 'tool_call_chunk', 'id': 'call_vbCyBcP8VuneUzyYlSBZZsVa', 'name': 'get_weather', 'args': '', 'index': 0}]


node: model
content: [{'type': 'tool_call_chunk', 'id': None, 'name': None, 'args': '{"', 'index': 0}]


node: model
content: [{'type': 'tool_call_chunk', 'id': None, 'name': None, 'args': 'city', 'index': 0}]


node: model
content: [{'type': 'tool_call_chunk', 'id': None, 'name': None, 'args': '":"', 'index': 0}]


node: model
content: [{'type': 'tool_call_chunk', 'id': None, 'name': None, 'args': 'San', 'index': 0}]


node: model
content: [{'type': 'tool_call_chunk', 'id': None, 'name': None, 'args': ' Francisco', 'index': 0}]


node: model
content: [{'type': 'tool_call_chunk', 'id': None, 'name': None, 'args': '"}', 'index': 0}]


node: model
content: []


node: tools
content: [{'type': 'text', 'text': "It's always sunny in San Francisco!"}]


node: model
content: []


node: model
content: [{'type': 'text', 'text': 'Here'}]


node: model
content: [{'type': 'text', 'text': ''s'}]


node: model
content: [{'type': 'text', 'text': ' what'}]


node: model
content: [{'type': 'text', 'text': ' I'}]


node: model
content: [{'type': 'text', 'text': ' got'}]


node: model
content: [{'type': 'text', 'text': ':'}]


node: model
content: [{'type': 'text', 'text': ' "'}]


node: model
content: [{'type': 'text', 'text': "It's"}]


node: model
content: [{'type': 'text', 'text': ' always'}]


node: model
content: [{'type': 'text', 'text': ' sunny'}]


node: model
content: [{'type': 'text', 'text': ' in'}]


node: model
content: [{'type': 'text', 'text': ' San'}]


node: model
content: [{'type': 'text', 'text': ' Francisco'}]


node: model
content: [{'type': 'text', 'text': '!"\n\n'}]
```

!!! note "Ghi chú"
    **Bọc một agent thành node trong một `StateGraph` cha?** [`create_agent`](https://reference.langchain.com/python/langchain/agents/factory/create_agent) trả về một graph đã compile, nên dùng nó như một node sẽ biến nó thành subgraph. `stream_mode="messages"` trên graph cha sẽ không phát ra token chunk từ lệnh gọi LLM của agent bên trong trừ khi bạn truyền `subgraphs=True`. Xem [Subgraph outputs](https://docs.langchain.com/oss/python/langgraph/streaming#subgraph-outputs).

## Cập nhật tuỳ chỉnh

Để stream cập nhật từ tool khi chúng đang thực thi, bạn có thể dùng [`get_stream_writer`](https://reference.langchain.com/python/langgraph/config/get_stream_writer).

```python title="Streaming custom updates"
from langchain.agents import create_agent
from langgraph.config import get_stream_writer  # [!code highlight]


def get_weather(city: str) -> str:
    """Get weather for a given city."""
    writer = get_stream_writer()  # [!code highlight]
    # stream bất kỳ dữ liệu tuỳ ý nào
    writer(f"Looking up data for city: {city}")
    writer(f"Acquired data for city: {city}")
    return f"It's always sunny in {city}!"

agent = create_agent(
    model="claude-sonnet-4-6",
    tools=[get_weather],
)

for chunk in agent.stream(
    {"messages": [{"role": "user", "content": "What is the weather in SF?"}]},
    stream_mode="custom",  # [!code highlight]
    version="v2",  # [!code highlight]
):
    if chunk["type"] == "custom":  # [!code highlight]
        print(chunk["data"])  # [!code highlight]
```

```shell title="Output"
Looking up data for city: San Francisco
Acquired data for city: San Francisco
```

!!! note "Ghi chú"
    Nếu bạn thêm [`get_stream_writer`](https://reference.langchain.com/python/langgraph/config/get_stream_writer) bên trong tool, bạn sẽ không thể invoke tool đó bên ngoài ngữ cảnh thực thi LangGraph.

## Stream nhiều mode

Bạn có thể chỉ định nhiều streaming mode bằng cách truyền stream mode dưới dạng list: `stream_mode=["updates", "custom"]`.

Mỗi chunk được stream là một dict `StreamPart` với các key `type`, `ns`, và `data`. Dùng `chunk["type"]` để xác định stream mode và `chunk["data"]` để truy cập payload.

```python title="Streaming multiple modes"
from langchain.agents import create_agent
from langgraph.config import get_stream_writer


def get_weather(city: str) -> str:
    """Get weather for a given city."""
    writer = get_stream_writer()
    writer(f"Looking up data for city: {city}")
    writer(f"Acquired data for city: {city}")
    return f"It's always sunny in {city}!"

agent = create_agent(
    model="gpt-5-nano",
    tools=[get_weather],
)

for chunk in agent.stream(  # [!code highlight]
    {"messages": [{"role": "user", "content": "What is the weather in SF?"}]},
    stream_mode=["updates", "custom"],
    version="v2",  # [!code highlight]
):
    print(f"stream_mode: {chunk['type']}")  # [!code highlight]
    print(f"content: {chunk['data']}")  # [!code highlight]
    print("\n")
```

```shell title="Output"
stream_mode: updates
content: {'model': {'messages': [AIMessage(content='', response_metadata={'token_usage': {'completion_tokens': 280, 'prompt_tokens': 132, 'total_tokens': 412, 'completion_tokens_details': {'accepted_prediction_tokens': 0, 'audio_tokens': 0, 'reasoning_tokens': 256, 'rejected_prediction_tokens': 0}, 'prompt_tokens_details': {'audio_tokens': 0, 'cached_tokens': 0}}, 'model_provider': 'openai', 'model_name': 'gpt-5-nano-2025-08-07', 'system_fingerprint': None, 'id': 'chatcmpl-C9tlgBzGEbedGYxZ0rTCz5F7OXpL7', 'service_tier': 'default', 'finish_reason': 'tool_calls', 'logprobs': None}, id='lc_run--480c07cb-e405-4411-aa7f-0520fddeed66-0', tool_calls=[{'name': 'get_weather', 'args': {'city': 'San Francisco'}, 'id': 'call_KTNQIftMrl9vgNwEfAJMVu7r', 'type': 'tool_call'}], usage_metadata={'input_tokens': 132, 'output_tokens': 280, 'total_tokens': 412, 'input_token_details': {'audio': 0, 'cache_read': 0}, 'output_token_details': {'audio': 0, 'reasoning': 256}})]}}


stream_mode: custom
content: Looking up data for city: San Francisco


stream_mode: custom
content: Acquired data for city: San Francisco


stream_mode: updates
content: {'tools': {'messages': [ToolMessage(content="It's always sunny in San Francisco!", name='get_weather', tool_call_id='call_KTNQIftMrl9vgNwEfAJMVu7r')]}}


stream_mode: updates
content: {'model': {'messages': [AIMessage(content='San Francisco weather: It's always sunny in San Francisco!\n\n', response_metadata={'token_usage': {'completion_tokens': 764, 'prompt_tokens': 168, 'total_tokens': 932, 'completion_tokens_details': {'accepted_prediction_tokens': 0, 'audio_tokens': 0, 'reasoning_tokens': 704, 'rejected_prediction_tokens': 0}, 'prompt_tokens_details': {'audio_tokens': 0, 'cached_tokens': 0}}, 'model_provider': 'openai', 'model_name': 'gpt-5-nano-2025-08-07', 'system_fingerprint': None, 'id': 'chatcmpl-C9tljDFVki1e1haCyikBptAuXuHYG', 'service_tier': 'default', 'finish_reason': 'stop', 'logprobs': None}, id='lc_run--acbc740a-18fe-4a14-8619-da92a0d0ee90-0', usage_metadata={'input_tokens': 168, 'output_tokens': 764, 'total_tokens': 932, 'input_token_details': {'audio': 0, 'cache_read': 0}, 'output_token_details': {'audio': 0, 'reasoning': 704}})]}}
```

## Các pattern thường gặp

Dưới đây là các ví dụ cho những use case streaming phổ biến.

### Stream token thinking / reasoning

Một số model thực hiện suy luận nội bộ trước khi đưa ra câu trả lời cuối cùng. Bạn có thể stream các token thinking/reasoning này ngay khi được sinh ra bằng cách lọc [standard content block](messages.md#standard-content-blocks) theo `type` là `"reasoning"`.

!!! note "Ghi chú"
    Reasoning output phải được bật trên model.

    Xem [phần reasoning](models.md#reasoning) và trang tích hợp của [provider của bạn](https://docs.langchain.com/oss/python/integrations/providers/overview) để biết chi tiết cấu hình.

    Để kiểm tra nhanh model nào hỗ trợ reasoning, xem [models.dev](https://models.dev).

Để stream thinking token từ một agent, dùng `stream_mode="messages"` và lọc theo reasoning content block:

```python
from langchain.agents import create_agent
from langchain_anthropic import ChatAnthropic
from langchain_core.runnables import Runnable


def get_weather(city: str) -> str:
    """Get weather for a given city."""
    return f"It's always sunny in {city}!"


model = ChatAnthropic(
    model_name="claude-sonnet-4-6",
    timeout=None,
    stop=None,
    thinking={"type": "enabled", "budget_tokens": 5000},
)
agent: Runnable = create_agent(
    model=model,
    tools=[get_weather],
)

stream = agent.stream_events(  # [!code highlight]
    {"messages": [{"role": "user", "content": "What is the weather in SF?"}]},
    version="v3",
)
for message in stream.messages:
    for token in message.reasoning:
        print(f"[thinking] {token}", end="")
    for token in message.text:
        print(token, end="", flush=True)
```

```shell title="Output"
[thinking] The user is asking about the weather in San Francisco. I have a tool
[thinking]  available to get this information. Let me call the get_weather tool
[thinking]  with "San Francisco" as the city parameter.
The weather in San Francisco is: It's always sunny in San Francisco!
```

Cách này hoạt động giống nhau bất kể provider model nào: LangChain chuẩn hoá các định dạng riêng của từng provider (khối `thinking` của Anthropic, tóm tắt `reasoning` của OpenAI, v.v.) thành một loại content block `"reasoning"` chuẩn thông qua thuộc tính [`content_blocks`](messages.md#standard-content-blocks).

Để stream reasoning token trực tiếp từ một chat model (không qua agent), xem [streaming với chat model](models.md#reasoning).

### Stream tool call

Bạn có thể muốn stream cả hai:

1. JSON từng phần khi [tool call](models.md#tool-calling) được sinh ra
2. Tool call hoàn chỉnh, đã parse, được thực thi

Chỉ định [`stream_mode="messages"`](#token-llm) sẽ stream các [message chunk](messages.md#streaming-and-chunks) gia tăng được sinh bởi mọi lệnh gọi LLM trong agent. Để truy cập message hoàn chỉnh với tool call đã parse:

1. Nếu các message đó được theo dõi trong [state](short-term-memory.md) (như trong model node của [`create_agent`](agents.md)), dùng `stream_mode=["messages", "updates"]` để truy cập message hoàn chỉnh thông qua [state update](#tien-trinh-agent) (minh hoạ bên dưới).
2. Nếu các message đó không được theo dõi trong state, dùng [cập nhật tuỳ chỉnh](#cap-nhat-tuy-chinh) hoặc gộp các chunk trong vòng lặp streaming ([phần tiếp theo](#truy-cap-message-hoan-chinh)).

!!! note "Ghi chú"
    Xem phần bên dưới về [streaming từ sub-agent](#stream-tu-sub-agent) nếu agent của bạn có nhiều LLM.

```python
from typing import Any

from langchain.agents import create_agent
from langchain.messages import AIMessage, AIMessageChunk, AnyMessage, ToolMessage


def get_weather(city: str) -> str:
    """Get weather for a given city."""

    return f"It's always sunny in {city}!"


agent = create_agent("openai:gpt-5.5", tools=[get_weather])


def _render_message_chunk(token: AIMessageChunk) -> None:
    if token.text:
        print(token.text, end="|")
    if token.tool_call_chunks:
        print(token.tool_call_chunks)
    # N.B. toàn bộ content đều có sẵn thông qua token.content_blocks


def _render_completed_message(message: AnyMessage) -> None:
    if isinstance(message, AIMessage) and message.tool_calls:
        print(f"Tool calls: {message.tool_calls}")
    if isinstance(message, ToolMessage):
        print(f"Tool response: {message.content_blocks}")


input_message = {"role": "user", "content": "What is the weather in Boston?"}
for chunk in agent.stream(
    {"messages": [input_message]},
    stream_mode=["messages", "updates"],  # [!code highlight]
    version="v2",  # [!code highlight]
):
    if chunk["type"] == "messages":  # [!code highlight]
        token, metadata = chunk["data"]  # [!code highlight]
        if isinstance(token, AIMessageChunk):
            _render_message_chunk(token)  # [!code highlight]
    elif chunk["type"] == "updates":  # [!code highlight]
        for source, update in chunk["data"].items():  # [!code highlight]
            if source in ("model", "tools"):  # `source` chứa tên node
                _render_completed_message(update["messages"][-1])  # [!code highlight]
```

```shell title="Output"
[{'name': 'get_weather', 'args': '', 'id': 'call_D3Orjr89KgsLTZ9hTzYv7Hpf', 'index': 0, 'type': 'tool_call_chunk'}]
[{'name': None, 'args': '{"', 'id': None, 'index': 0, 'type': 'tool_call_chunk'}]
[{'name': None, 'args': 'city', 'id': None, 'index': 0, 'type': 'tool_call_chunk'}]
[{'name': None, 'args': '":"', 'id': None, 'index': 0, 'type': 'tool_call_chunk'}]
[{'name': None, 'args': 'Boston', 'id': None, 'index': 0, 'type': 'tool_call_chunk'}]
[{'name': None, 'args': '"}', 'id': None, 'index': 0, 'type': 'tool_call_chunk'}]
Tool calls: [{'name': 'get_weather', 'args': {'city': 'Boston'}, 'id': 'call_D3Orjr89KgsLTZ9hTzYv7Hpf', 'type': 'tool_call'}]
Tool response: [{'type': 'text', 'text': "It's always sunny in Boston!"}]
The| weather| in| Boston| is| **|sun|ny|**|.|
```

#### Truy cập message hoàn chỉnh

!!! note "Ghi chú"
    Nếu message hoàn chỉnh được theo dõi trong [state](short-term-memory.md) của agent, bạn có thể dùng `stream_mode=["messages", "updates"]` như đã minh hoạ ở phần [Stream tool call](#stream-tool-call) để truy cập message hoàn chỉnh trong khi streaming.

Trong một số trường hợp, message hoàn chỉnh không được phản ánh trong [state update](#tien-trinh-agent). Nếu bạn có quyền truy cập nội bộ agent, bạn có thể dùng [cập nhật tuỳ chỉnh](#cap-nhat-tuy-chinh) để truy cập các message này trong khi streaming. Nếu không, bạn có thể gộp message chunk trong vòng lặp streaming (xem bên dưới).

Xét ví dụ bên dưới, nơi chúng ta kết hợp một [stream writer](#cap-nhat-tuy-chinh) vào một [guardrail middleware](guardrails.md#after-agent-guardrails) đơn giản hoá. Middleware này minh hoạ việc gọi tool để sinh ra đánh giá "safe / unsafe" có cấu trúc (bạn cũng có thể dùng [structured output](models.md#structured-output) cho việc này):

```python
from typing import Any, Literal

from langchain.agents.middleware import after_agent, AgentState
from langgraph.runtime import Runtime
from langchain.messages import AIMessage
from langchain.chat_models import init_chat_model
from langgraph.config import get_stream_writer  # [!code highlight]
from pydantic import BaseModel


class ResponseSafety(BaseModel):
    """Evaluate a response as safe or unsafe."""
    evaluation: Literal["safe", "unsafe"]


safety_model = init_chat_model("openai:gpt-5.5")

@after_agent(can_jump_to=["end"])
def safety_guardrail(state: AgentState, runtime: Runtime) -> dict[str, Any] | None:
    """Model-based guardrail: Use an LLM to evaluate response safety."""
    stream_writer = get_stream_writer()  # [!code highlight]
    # Get the model response
    if not state["messages"]:
        return None

    last_message = state["messages"][-1]
    if not isinstance(last_message, AIMessage):
        return None

    # Use another model to evaluate safety
    model_with_tools = safety_model.bind_tools([ResponseSafety], tool_choice="any")
    result = model_with_tools.invoke(
        [
            {
                "role": "system",
                "content": "Evaluate this AI response as generally safe or unsafe."
            },
            {
                "role": "user",
                "content": f"AI response: {last_message.text}"
            }
        ]
    )
    stream_writer(result)  # [!code highlight]

    tool_call = result.tool_calls[0]
    if tool_call["args"]["evaluation"] == "unsafe":
        last_message.content = "I cannot provide that response. Please rephrase your request."

    return None
```

Sau đó chúng ta có thể tích hợp middleware này vào agent và bao gồm các custom stream event của nó:

```python
from typing import Any

from langchain.agents import create_agent
from langchain.messages import AIMessageChunk, AIMessage, AnyMessage


def get_weather(city: str) -> str:
    """Get weather for a given city."""

    return f"It's always sunny in {city}!"


agent = create_agent(
    model="openai:gpt-5.5",
    tools=[get_weather],
    middleware=[safety_guardrail],  # [!code highlight]
)

def _render_message_chunk(token: AIMessageChunk) -> None:
    if token.text:
        print(token.text, end="|")
    if token.tool_call_chunks:
        print(token.tool_call_chunks)


def _render_completed_message(message: AnyMessage) -> None:
    if isinstance(message, AIMessage) and message.tool_calls:
        print(f"Tool calls: {message.tool_calls}")
    if isinstance(message, ToolMessage):
        print(f"Tool response: {message.content_blocks}")


input_message = {"role": "user", "content": "What is the weather in Boston?"}
for chunk in agent.stream(
    {"messages": [input_message]},
    stream_mode=["messages", "updates", "custom"],  # [!code highlight]
    version="v2",  # [!code highlight]
):
    if chunk["type"] == "messages":  # [!code highlight]
        token, metadata = chunk["data"]  # [!code highlight]
        if isinstance(token, AIMessageChunk):
            _render_message_chunk(token)
    elif chunk["type"] == "updates":  # [!code highlight]
        for source, update in chunk["data"].items():  # [!code highlight]
            if source in ("model", "tools"):
                _render_completed_message(update["messages"][-1])
    elif chunk["type"] == "custom":  # [!code highlight]
        # truy cập message hoàn chỉnh trong stream
        print(f"Tool calls: {chunk['data'].tool_calls}")  # [!code highlight]
```

```shell title="Output"
[{'name': 'get_weather', 'args': '', 'id': 'call_je6LWgxYzuZ84mmoDalTYMJC', 'index': 0, 'type': 'tool_call_chunk'}]
[{'name': None, 'args': '{"', 'id': None, 'index': 0, 'type': 'tool_call_chunk'}]
[{'name': None, 'args': 'city', 'id': None, 'index': 0, 'type': 'tool_call_chunk'}]
[{'name': None, 'args': '":"', 'id': None, 'index': 0, 'type': 'tool_call_chunk'}]
[{'name': None, 'args': 'Boston', 'id': None, 'index': 0, 'type': 'tool_call_chunk'}]
[{'name': None, 'args': '"}', 'id': None, 'index': 0, 'type': 'tool_call_chunk'}]
Tool calls: [{'name': 'get_weather', 'args': {'city': 'Boston'}, 'id': 'call_je6LWgxYzuZ84mmoDalTYMJC', 'type': 'tool_call'}]
Tool response: [{'type': 'text', 'text': "It's always sunny in Boston!"}]
The| weather| in| **|Boston|**| is| **|sun|ny|**|.|[{'name': 'ResponseSafety', 'args': '', 'id': 'call_O8VJIbOG4Q9nQF0T8ltVi58O', 'index': 0, 'type': 'tool_call_chunk'}]
[{'name': None, 'args': '{"', 'id': None, 'index': 0, 'type': 'tool_call_chunk'}]
[{'name': None, 'args': 'evaluation', 'id': None, 'index': 0, 'type': 'tool_call_chunk'}]
[{'name': None, 'args': '":"', 'id': None, 'index': 0, 'type': 'tool_call_chunk'}]
[{'name': None, 'args': 'safe', 'id': None, 'index': 0, 'type': 'tool_call_chunk'}]
[{'name': None, 'args': '"}', 'id': None, 'index': 0, 'type': 'tool_call_chunk'}]
Tool calls: [{'name': 'ResponseSafety', 'args': {'evaluation': 'safe'}, 'id': 'call_O8VJIbOG4Q9nQF0T8ltVi58O', 'type': 'tool_call'}]
```

Ngoài ra, nếu bạn không thể thêm custom event vào stream, bạn có thể gộp các message chunk trong vòng lặp streaming:

```python
input_message = {"role": "user", "content": "What is the weather in Boston?"}
full_message = None  # [!code highlight]
for chunk in agent.stream(
    {"messages": [input_message]},
    stream_mode=["messages", "updates"],
    version="v2",  # [!code highlight]
):
    if chunk["type"] == "messages":  # [!code highlight]
        token, metadata = chunk["data"]  # [!code highlight]
        if isinstance(token, AIMessageChunk):
            _render_message_chunk(token)
            full_message = token if full_message is None else full_message + token  # [!code highlight]
            if token.chunk_position == "last":  # [!code highlight]
                if full_message.tool_calls:  # [!code highlight]
                    print(f"Tool calls: {full_message.tool_calls}")  # [!code highlight]
                full_message = None  # [!code highlight]
    elif chunk["type"] == "updates":  # [!code highlight]
        for source, update in chunk["data"].items():  # [!code highlight]
            if source == "tools":
                _render_completed_message(update["messages"][-1])
```

### Streaming với human-in-the-loop

Để xử lý [interrupt](human-in-the-loop.md) human-in-the-loop, chúng ta xây dựng dựa trên [ví dụ ở trên](#stream-tool-call):

1. Chúng ta cấu hình agent với [middleware human-in-the-loop và một checkpointer](human-in-the-loop.md#configuring-interrupts)
2. Chúng ta thu thập các interrupt được sinh ra trong stream mode `"updates"`
3. Chúng ta phản hồi các interrupt đó bằng một [command](human-in-the-loop.md#responding-to-interrupts)

```python
from typing import Any

from langchain.agents import create_agent
from langchain.agents.middleware import HumanInTheLoopMiddleware
from langchain.messages import AIMessage, AIMessageChunk, AnyMessage, ToolMessage
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.types import Command, Interrupt


def get_weather(city: str) -> str:
    """Get weather for a given city."""

    return f"It's always sunny in {city}!"


checkpointer = InMemorySaver()

agent = create_agent(
    "openai:gpt-5.5",
    tools=[get_weather],
    middleware=[  # [!code highlight]
        HumanInTheLoopMiddleware(interrupt_on={"get_weather": True}),  # [!code highlight]
    ],  # [!code highlight]
    checkpointer=checkpointer,  # [!code highlight]
)


def _render_message_chunk(token: AIMessageChunk) -> None:
    if token.text:
        print(token.text, end="|")
    if token.tool_call_chunks:
        print(token.tool_call_chunks)


def _render_completed_message(message: AnyMessage) -> None:
    if isinstance(message, AIMessage) and message.tool_calls:
        print(f"Tool calls: {message.tool_calls}")
    if isinstance(message, ToolMessage):
        print(f"Tool response: {message.content_blocks}")


def _render_interrupt(interrupt: Interrupt) -> None:  # [!code highlight]
    interrupts = interrupt.value  # [!code highlight]
    for request in interrupts["action_requests"]:  # [!code highlight]
        print(request["description"])  # [!code highlight]


input_message = {
    "role": "user",
    "content": (
        "Can you look up the weather in Boston and San Francisco?"
    ),
}
config = {"configurable": {"thread_id": "some_id"}}  # [!code highlight]
interrupts = []  # [!code highlight]
for chunk in agent.stream(
    {"messages": [input_message]},
    config=config,  # [!code highlight]
    stream_mode=["messages", "updates"],
    version="v2",  # [!code highlight]
):
    if chunk["type"] == "messages":  # [!code highlight]
        token, metadata = chunk["data"]  # [!code highlight]
        if isinstance(token, AIMessageChunk):
            _render_message_chunk(token)
    elif chunk["type"] == "updates":  # [!code highlight]
        for source, update in chunk["data"].items():  # [!code highlight]
            if source in ("model", "tools"):
                _render_completed_message(update["messages"][-1])
            if source == "__interrupt__":  # [!code highlight]
                interrupts.extend(update)  # [!code highlight]
                _render_interrupt(update[0])  # [!code highlight]
```

```shell title="Output"
[{'name': 'get_weather', 'args': '', 'id': 'call_GOwNaQHeqMixay2qy80padfE', 'index': 0, 'type': 'tool_call_chunk'}]
[{'name': None, 'args': '{"ci', 'id': None, 'index': 0, 'type': 'tool_call_chunk'}]
[{'name': None, 'args': 'ty": ', 'id': None, 'index': 0, 'type': 'tool_call_chunk'}]
[{'name': None, 'args': '"Bosto', 'id': None, 'index': 0, 'type': 'tool_call_chunk'}]
[{'name': None, 'args': 'n"}', 'id': None, 'index': 0, 'type': 'tool_call_chunk'}]
[{'name': 'get_weather', 'args': '', 'id': 'call_Ndb4jvWm2uMA0JDQXu37wDH6', 'index': 1, 'type': 'tool_call_chunk'}]
[{'name': None, 'args': '{"ci', 'id': None, 'index': 1, 'type': 'tool_call_chunk'}]
[{'name': None, 'args': 'ty": ', 'id': None, 'index': 1, 'type': 'tool_call_chunk'}]
[{'name': None, 'args': '"San F', 'id': None, 'index': 1, 'type': 'tool_call_chunk'}]
[{'name': None, 'args': 'ranc', 'id': None, 'index': 1, 'type': 'tool_call_chunk'}]
[{'name': None, 'args': 'isco"', 'id': None, 'index': 1, 'type': 'tool_call_chunk'}]
[{'name': None, 'args': '}', 'id': None, 'index': 1, 'type': 'tool_call_chunk'}]
Tool calls: [{'name': 'get_weather', 'args': {'city': 'Boston'}, 'id': 'call_GOwNaQHeqMixay2qy80padfE', 'type': 'tool_call'}, {'name': 'get_weather', 'args': {'city': 'San Francisco'}, 'id': 'call_Ndb4jvWm2uMA0JDQXu37wDH6', 'type': 'tool_call'}]
Tool execution requires approval

Tool: get_weather
Args: {'city': 'Boston'}
Tool execution requires approval

Tool: get_weather
Args: {'city': 'San Francisco'}
```

Tiếp theo chúng ta thu thập một [quyết định](human-in-the-loop.md#interrupt-decision-types) cho mỗi interrupt. Quan trọng là, thứ tự quyết định phải khớp với thứ tự hành động đã thu thập.

Để minh hoạ, chúng ta sẽ sửa một tool call và chấp nhận cái còn lại:

```python
def _get_interrupt_decisions(interrupt: Interrupt) -> list[dict]:
    return [
        {
            "type": "edit",
            "edited_action": {
                "name": "get_weather",
                "args": {"city": "Boston, U.K."},
            },
        }
        if "boston" in request["description"].lower()
        else {"type": "approve"}
        for request in interrupt.value["action_requests"]
    ]

decisions = {}
for interrupt in interrupts:
    decisions[interrupt.id] = {
        "decisions": _get_interrupt_decisions(interrupt)
    }

decisions
```

```shell title="Output"
{
    'a96c40474e429d661b5b32a8d86f0f3e': {
        'decisions': [
            {
                'type': 'edit',
                 'edited_action': {
                     'name': 'get_weather',
                     'args': {'city': 'Boston, U.K.'}
                 }
            },
            {'type': 'approve'},
        ]
    }
}
```

Chúng ta có thể tiếp tục bằng cách truyền một [command](human-in-the-loop.md#responding-to-interrupts) vào cùng vòng lặp streaming:

```python
interrupts = []
for chunk in agent.stream(
    Command(resume=decisions),  # [!code highlight]
    config=config,
    stream_mode=["messages", "updates"],
    version="v2",  # [!code highlight]
):
    # Vòng lặp streaming không đổi
    if chunk["type"] == "messages":  # [!code highlight]
        token, metadata = chunk["data"]  # [!code highlight]
        if isinstance(token, AIMessageChunk):
            _render_message_chunk(token)
    elif chunk["type"] == "updates":  # [!code highlight]
        for source, update in chunk["data"].items():  # [!code highlight]
            if source in ("model", "tools"):
                _render_completed_message(update["messages"][-1])
            if source == "__interrupt__":
                interrupts.extend(update)
                _render_interrupt(update[0])
```

```shell title="Output"
Tool response: [{'type': 'text', 'text': "It's always sunny in Boston, U.K.!"}]
Tool response: [{'type': 'text', 'text': "It's always sunny in San Francisco!"}]
-| **|Boston|**|:| It|'s| always| sunny| in| Boston|,| U|.K|.|
|-| **|San| Francisco|**|:| It|'s| always| sunny| in| San| Francisco|!|
```

### Stream từ sub-agent

Khi có nhiều LLM tại bất kỳ điểm nào trong một agent, việc phân biệt nguồn gốc của message khi chúng được sinh ra thường là cần thiết.

Để làm điều này, truyền một [`name`](https://reference.langchain.com/python/langchain/agents/#langchain.agents.create_agent\(name\)) cho mỗi agent khi tạo nó. Tên này sau đó có sẵn trong metadata qua key `lc_agent_name` khi streaming ở mode `"messages"`.

Bên dưới, chúng ta cập nhật ví dụ [stream tool call](#stream-tool-call):

1. Chúng ta thay tool bằng một tool `call_weather_agent` gọi một agent bên trong
2. Chúng ta thêm `name` cho mỗi agent
3. Chúng ta chỉ định [`subgraphs=True`](https://docs.langchain.com/oss/python/langgraph/use-subgraphs#stream-subgraph-outputs) khi tạo stream
4. Việc xử lý stream giống như trước, nhưng chúng ta thêm logic để theo dõi agent nào đang hoạt động bằng tham số `name` của `create_agent`

!!! tip "Mẹo"
    Khi bạn đặt một `name` cho agent, tên đó cũng được gắn vào mọi `AIMessage` do agent đó sinh ra.

Đầu tiên chúng ta xây dựng agent:

```python
from typing import Any

from langchain.agents import create_agent
from langchain.chat_models import init_chat_model
from langchain.messages import AIMessage, AnyMessage


def get_weather(city: str) -> str:
    """Get weather for a given city."""

    return f"It's always sunny in {city}!"


weather_model = init_chat_model("openai:gpt-5.5")
weather_agent = create_agent(
    model=weather_model,
    tools=[get_weather],
    name="weather_agent",  # [!code highlight]
)


def call_weather_agent(query: str) -> str:
    """Query the weather agent."""
    result = weather_agent.invoke({
        "messages": [{"role": "user", "content": query}]
    })
    return result["messages"][-1].text


supervisor_model = init_chat_model("openai:gpt-5.5")
agent = create_agent(
    model=supervisor_model,
    tools=[call_weather_agent],
    name="supervisor",  # [!code highlight]
)
```

Tiếp theo, chúng ta thêm logic vào vòng lặp streaming để báo cáo agent nào đang phát ra token:

```python
def _render_message_chunk(token: AIMessageChunk) -> None:
    if token.text:
        print(token.text, end="|")
    if token.tool_call_chunks:
        print(token.tool_call_chunks)


def _render_completed_message(message: AnyMessage) -> None:
    if isinstance(message, AIMessage) and message.tool_calls:
        print(f"Tool calls: {message.tool_calls}")
    if isinstance(message, ToolMessage):
        print(f"Tool response: {message.content_blocks}")


input_message = {"role": "user", "content": "What is the weather in Boston?"}
current_agent = None  # [!code highlight]
for chunk in agent.stream(
    {"messages": [input_message]},
    stream_mode=["messages", "updates"],
    subgraphs=True,  # [!code highlight]
    version="v2",  # [!code highlight]
):
    if chunk["type"] == "messages":  # [!code highlight]
        token, metadata = chunk["data"]  # [!code highlight]
        if agent_name := metadata.get("lc_agent_name"):  # [!code highlight]
            if agent_name != current_agent:  # [!code highlight]
                print(f"🤖 {agent_name}: ")  # [!code highlight]
                current_agent = agent_name  # [!code highlight]
        if isinstance(token, AIMessageChunk):
            _render_message_chunk(token)
    elif chunk["type"] == "updates":  # [!code highlight]
        for source, update in chunk["data"].items():  # [!code highlight]
            if source in ("model", "tools"):
                _render_completed_message(update["messages"][-1])
```

```shell title="Output"
🤖 supervisor:
[{'name': 'call_weather_agent', 'args': '', 'id': 'call_asorzUf0mB6sb7MiKfgojp7I', 'index': 0, 'type': 'tool_call_chunk'}]
[{'name': None, 'args': '{"', 'id': None, 'index': 0, 'type': 'tool_call_chunk'}]
[{'name': None, 'args': 'query', 'id': None, 'index': 0, 'type': 'tool_call_chunk'}]
[{'name': None, 'args': '":"', 'id': None, 'index': 0, 'type': 'tool_call_chunk'}]
[{'name': None, 'args': 'Boston', 'id': None, 'index': 0, 'type': 'tool_call_chunk'}]
[{'name': None, 'args': ' weather', 'id': None, 'index': 0, 'type': 'tool_call_chunk'}]
[{'name': None, 'args': ' right', 'id': None, 'index': 0, 'type': 'tool_call_chunk'}]
[{'name': None, 'args': ' now', 'id': None, 'index': 0, 'type': 'tool_call_chunk'}]
[{'name': None, 'args': ' and', 'id': None, 'index': 0, 'type': 'tool_call_chunk'}]
[{'name': None, 'args': " today's", 'id': None, 'index': 0, 'type': 'tool_call_chunk'}]
[{'name': None, 'args': ' forecast', 'id': None, 'index': 0, 'type': 'tool_call_chunk'}]
[{'name': None, 'args': '"}', 'id': None, 'index': 0, 'type': 'tool_call_chunk'}]
Tool calls: [{'name': 'call_weather_agent', 'args': {'query': "Boston weather right now and today's forecast"}, 'id': 'call_asorzUf0mB6sb7MiKfgojp7I', 'type': 'tool_call'}]
🤖 weather_agent:
[{'name': 'get_weather', 'args': '', 'id': 'call_LZ89lT8fW6w8vqck5pZeaDIx', 'index': 0, 'type': 'tool_call_chunk'}]
[{'name': None, 'args': '{"', 'id': None, 'index': 0, 'type': 'tool_call_chunk'}]
[{'name': None, 'args': 'city', 'id': None, 'index': 0, 'type': 'tool_call_chunk'}]
[{'name': None, 'args': '":"', 'id': None, 'index': 0, 'type': 'tool_call_chunk'}]
[{'name': None, 'args': 'Boston', 'id': None, 'index': 0, 'type': 'tool_call_chunk'}]
[{'name': None, 'args': '"}', 'id': None, 'index': 0, 'type': 'tool_call_chunk'}]
Tool calls: [{'name': 'get_weather', 'args': {'city': 'Boston'}, 'id': 'call_LZ89lT8fW6w8vqck5pZeaDIx', 'type': 'tool_call'}]
Tool response: [{'type': 'text', 'text': "It's always sunny in Boston!"}]
Boston| weather| right| now|:| **|Sunny|**|.

|Today|'s| forecast| for| Boston|:| **|Sunny| all| day|**|.|Tool response: [{'type': 'text', 'text': 'Boston weather right now: **Sunny**.\n\nToday's forecast for Boston: **Sunny all day**.'}]
🤖 supervisor:
Boston| weather| right| now|:| **|Sunny|**|.

|Today|'s| forecast| for| Boston|:| **|Sunny| all| day|**|.|
```

## Tắt streaming

Trong một số ứng dụng, bạn có thể cần tắt streaming của từng token cho một model cụ thể. Điều này hữu ích khi:

* Làm việc với hệ thống [multi-agent](multi-agent/index.md) để kiểm soát agent nào stream output của nó
* Kết hợp model hỗ trợ streaming với model không hỗ trợ
* Deploy lên [LangSmith](https://docs.langchain.com/langsmith/observability) và muốn ngăn một số output model bị stream tới client

Đặt `streaming=False` khi khởi tạo model.

```python
from langchain_openai import ChatOpenAI

model = ChatOpenAI(
    model="gpt-5.5",
    streaming=False  # [!code highlight]
)
```

!!! tip "Mẹo"
    Khi deploy lên LangSmith, đặt `streaming=False` cho bất kỳ model nào bạn không muốn output của nó bị stream tới client. Điều này được cấu hình trong code graph của bạn trước khi deploy.

!!! note "Ghi chú"
    Không phải tích hợp chat model nào cũng hỗ trợ tham số `streaming`. Nếu model của bạn không hỗ trợ, dùng `disable_streaming=True` thay thế. Tham số này có sẵn trên mọi chat model thông qua base class.

Xem [hướng dẫn LangGraph streaming](https://docs.langchain.com/oss/python/langgraph/streaming#disable-streaming-for-specific-chat-models) để biết thêm chi tiết.

## Định dạng streaming v2

!!! note "Ghi chú"
    Yêu cầu LangGraph >= 1.1.

Truyền `version="v2"` vào `stream()` hoặc `astream()` để nhận định dạng output thống nhất. Mỗi chunk là một dict `StreamPart` với các key `type`, `ns`, và `data`, cùng một hình dạng bất kể stream mode hay số lượng mode:

=== "v2 (mới)"

    ```python
    # Định dạng thống nhất, không cần unpack tuple nữa
    for chunk in agent.stream(
        {"messages": [{"role": "user", "content": "What is the weather in SF?"}]},
        stream_mode=["updates", "custom"],
        version="v2",
    ):
        print(chunk["type"])  # "updates" hoặc "custom"
        print(chunk["data"])  # payload
    ```

=== "v1 (mặc định hiện tại)"

    ```python
    # Phải unpack tuple (mode, data)
    for mode, chunk in agent.stream(
        {"messages": [{"role": "user", "content": "What is the weather in SF?"}]},
        stream_mode=["updates", "custom"],
    ):
        print(mode)   # "updates" hoặc "custom"
        print(chunk)  # payload
    ```

Định dạng v2 cũng cải thiện `invoke()`: nó trả về một đối tượng `GraphOutput` với thuộc tính `.value` và `.interrupts`, tách rõ ràng state khỏi metadata interrupt:

```python
result = agent.invoke(
    {"messages": [{"role": "user", "content": "Hello"}]},
    version="v2",
)
print(result.value)       # state (dict, Pydantic model, hoặc dataclass)
print(result.interrupts)  # tuple các đối tượng Interrupt (rỗng nếu không có)
```

Xem [tài liệu LangGraph streaming](https://docs.langchain.com/oss/python/langgraph/streaming#stream-output-format-v2) để biết thêm chi tiết về định dạng v2, bao gồm type narrowing, coercion Pydantic/dataclass, và streaming subgraph.

## Liên quan

* [Frontend streaming](frontend/overview.md): xây dựng UI React với [`useStream`](https://reference.langchain.com/javascript/langchain-react/index/useStream) cho tương tác agent theo thời gian thực
* [Streaming với chat model](models.md#stream): stream token trực tiếp từ chat model mà không cần agent hay graph
* [Reasoning với chat model](models.md#reasoning): cấu hình và truy cập reasoning output từ chat model
* [Standard content blocks](messages.md#standard-content-blocks): hiểu định dạng content block chuẩn hoá dùng cho reasoning, text, và các loại nội dung khác
* [Streaming với human-in-the-loop](human-in-the-loop.md#streaming-with-human-in-the-loop): stream tiến trình agent trong khi xử lý interrupt để human review
* [LangGraph streaming](https://docs.langchain.com/oss/python/langgraph/streaming): các tuỳ chọn streaming nâng cao bao gồm mode `values`, `debug`, và streaming subgraph
