# Event streaming

> Stream (truyền phát) cập nhật theo thời gian thực từ các lượt chạy agent của LangChain

Agent của LangChain được xây dựng trên nền LangGraph, do đó chúng hỗ trợ cùng một streaming stack, với các projection (hình chiếu dữ liệu) tập trung vào agent cho messages, tool calls, state, và các update tuỳ chỉnh.

Với hầu hết các trường hợp dùng cho ứng dụng và frontend, hãy dùng **Event Streaming** thông qua `stream_events(..., version="v3")`. Event Streaming trả về một run object với các projection có kiểu (typed projections), nhờ đó mỗi projection có thể được tiêu thụ (consume) độc lập thay vì phải tự parse các stream-mode tuple.

```py
from langchain.agents import create_agent


def get_weather(city: str) -> str:
    """Get weather for a city."""
    return f"It's always sunny in {city}!"


agent = create_agent(
    model="gpt-5-nano",
    tools=[get_weather],
)

stream = agent.stream_events({
    "messages": [{"role": "user", "content": "What is the weather in SF?"}],
}, version="v3")

for message in stream.messages:
    for delta in message.text:
        print(delta, end="", flush=True)

final_state = stream.output
```

## Những gì bạn có thể stream

| Projection             | Công dụng                                                                    |
| ----------------------- | ------------------------------------------------------------------------- |
| `for event in stream`   | Các event thô của protocol, với đầy đủ envelope và quyền truy cập mọi channel. |
| `stream.messages`       | Luồng model message, mỗi luồng ứng với một lượt gọi LLM.                      |
| `message.text`          | Các đoạn text delta và text cuối cùng của một message.                        |
| `message.reasoning`     | Các đoạn reasoning delta, dành cho model có expose nội dung reasoning.        |
| `message.tool_calls`    | Các chunk argument của tool call và tool call đã hoàn tất (finalized).        |
| `message.output`        | Đối tượng message cuối cùng sau khi lượt gọi model hoàn tất.                  |
| `stream.values`         | Snapshot của agent state.                                                     |
| `stream.output`         | Agent state cuối cùng.                                                        |
| `stream.subgraphs`      | Các lượt chạy graph lồng nhau (sub-agent và subgraph thông thường).           |
| `stream.extensions`     | Projection từ các transformer tuỳ chỉnh.                                      |
| `stream.tool_calls`     | Vòng đời thực thi tool: input, các output delta, output cuối cùng, và lỗi.    |

`stream.messages` trả về các object `ChatModelStream`. Mỗi message stream expose các thuộc tính `.text`, `.reasoning`, `.tool_calls`, và `.output`. Các projection đồng bộ (sync) có thể iterate để lấy delta theo thời gian thực, và có thể "drain" để lấy giá trị cuối cùng: dùng `str(message.text)` để lấy text cuối cùng và `message.tool_calls.get()` để lấy tool call đã hoàn tất.

## Agent messages

Dùng `stream.messages` khi bạn muốn lấy output của model từ mỗi lượt gọi LLM.

```py
stream = agent.stream_events(input, version="v3")

for message in stream.messages:
    print(f"[{message.node}] ", end="")
    for delta in message.text:
        print(delta, end="", flush=True)

    full_message = message.output
    usage = full_message.usage_metadata
    if usage:
        print(usage)
```

`message.output` cho bạn AI message đã hoàn tất, bao gồm cả các content block đặc thù theo từng provider. Trong TypeScript, dùng `message.usage` khi bạn chỉ cần số lượng token hoặc thông tin usage khác; trong Python, đọc usage từ `message.output.usage_metadata`.

## Reasoning content

Reasoning content dùng chung cấu trúc với text content, nhưng chỉ khả dụng khi model được chọn có phát ra (emit) các reasoning block.

```py
stream = agent.stream_events(input, version="v3")

for message in stream.messages:
    for delta in message.reasoning:
        print(f"[thinking] {delta}", end="", flush=True)

    for delta in message.text:
        print(delta, end="", flush=True)
```

Xem [hướng dẫn về reasoning](models.md#reasoning) và trang tích hợp của provider bạn dùng để biết chi tiết cấu hình model.

## Tool calls

Có hai projection hữu ích cho tool call:

* `message.tool_calls` stream các chunk argument của tool call trong lúc model đang tạo ra tool call đó.
* `stream.tool_calls` stream vòng đời của việc thực thi tool, sau khi tool call bắt đầu.

```py
stream = agent.stream_events(input, version="v3")

for message in stream.messages:
    for chunk in message.tool_calls:
        print(f"tool call chunk: {chunk}")

    finalized = message.tool_calls.get()
    if finalized:
        print(f"finalized tool calls: {finalized}")

for call in stream.tool_calls:
    print(f"{call.tool_name}({call.input})")
    for delta in call.output_deltas:
        print(delta, end="", flush=True)
    print(call.output, call.error)
```

## Streaming sub-agent

Khi một lượt gọi `create_agent` gọi tới một `create_agent` khác có đặt tên (thường thông qua một tool bao bọc), các event của agent con (inner agent) sẽ chảy trong một namespace lồng nhau (nested namespace). Tham số `name=` mà bạn truyền cho `create_agent` xác định agent con đó trong stream, nhờ vậy bạn có thể filter và gắn nhãn theo từng agent.

Các sub-agent có tên sẽ xuất hiện ở projection riêng `stream.subagents`. Mỗi handle expose các thuộc tính `.messages`, `.values`, `.tool_calls`, `.output` của chính agent con đó, cộng thêm `.name` (giá trị `name=` bạn đã truyền) và `.cause` (tool call đã dispatch sub-agent đó). Vì chỉ những lượt chạy `create_agent` có tên mới xuất hiện ở đây, bạn không cần tự filter bỏ các subgraph thông thường.

```py
from langchain.agents import create_agent
from langchain.chat_models import init_chat_model


def get_weather(city: str) -> str:
    """Get weather for a given city."""
    return f"It's always sunny in {city}!"


weather_agent = create_agent(
    model=init_chat_model("openai:gpt-5.5"),
    tools=[get_weather],
    name="weather_agent",
)


def call_weather(query: str) -> str:
    """Query the weather agent."""
    result = weather_agent.invoke({"messages": [{"role": "user", "content": query}]})
    return result["messages"][-1].text


supervisor = create_agent(
    model=init_chat_model("openai:gpt-5.5"),
    tools=[call_weather],
    name="supervisor",
)

stream = supervisor.stream_events(
    {"messages": [{"role": "user", "content": "What's the weather in Boston?"}]},
    version="v3",
)

for subagent in stream.subagents:
    print(f"{subagent.name}: ", end="")
    for message in subagent.messages:
        for token in message.text:
            print(token, end="", flush=True)
    print()
```

Các subgraph `StateGraph` thông thường được gọi từ một tool cũng xuất hiện ở `stream.subgraphs`, hãy đặt `name=` khi gọi `.compile(name=...)` để có một nhãn tại `subagent.graph_name`.

`stream.subagents` là view tập trung vào các sub-agent `create_agent` có tên, còn `stream.subgraphs` bao phủ mọi graph lồng nhau. Hãy dùng cái nào phù hợp với UI của bạn.

## State và output cuối cùng

Dùng `stream.values` để lấy snapshot của state, và `stream.output` để lấy agent state cuối cùng.

```py
stream = agent.stream_events(input, version="v3")

for snapshot in stream.values:
    print(snapshot)

final_state = stream.output
```

## Nhiều projection cùng lúc

Để tiêu thụ đồng thời (concurrent) trong code bất đồng bộ (async), dùng `astream_events` cùng `asyncio.gather`:

```py
import asyncio

stream = await agent.astream_events(input, version="v3")

async def consume_messages():
    async for message in stream.messages:
        print(await message.text)

async def consume_tool_calls():
    async for call in stream.tool_calls:
        print(call.tool_name, call.input)

await asyncio.gather(consume_messages(), consume_tool_calls())
```

Với code đồng bộ, dùng `stream.interleave(...)` thay thế:

```py
stream = agent.stream_events(input, version="v3")

for name, item in stream.interleave("messages", "tool_calls", "values"):
    if name == "messages":
        print(item.text)
    elif name == "tool_calls":
        print(item.tool_name, item.input)
    elif name == "values":
        print(item)
```

Để truy cập các channel không được expose dưới dạng projection có kiểu (typed projection), hoặc để xem toàn bộ envelope của event, hãy iterate qua các raw protocol event:

```py
for event in stream:
    print(event["method"], event["params"]["namespace"], event["params"]["data"])
```

## Custom update

Dùng custom stream transformer khi ứng dụng của bạn cần một projection không có sẵn, ví dụ như tiến trình retrieval, artifact, hoặc các event đặc thù theo domain.

```py
stream = agent.stream_events(
    input,
    version="v3",
    transformers=[ToolActivityTransformer],
)

for activity in stream.extensions["tool_activity"]:
    print(activity)
```

### Đăng ký transformer trên middleware

!!! note "Ghi chú"
    Transformer đăng ký qua middleware yêu cầu `langchain>=1.3.2`.

Middleware có thể khai báo các transformer factory bên cạnh hook và tool của nó. Cấu trúc factory khác nhau giữa các ngôn ngữ:

Đặt thuộc tính `transformers` trên một subclass của `AgentMiddleware` thành một dãy (sequence) các factory. Mỗi factory có dạng `Callable[[tuple[str, ...]], StreamTransformer]` và được gọi dưới dạng `factory(scope)`, trong đó `scope` là tuple scope của mini-mux (`()` cho mux gốc, không rỗng đối với subgraph). Việc trả về một transformer mới cho mỗi lần gọi giúp mỗi subgraph được cách ly (isolated) độc lập.

```py
from langchain.agents import create_agent
from langchain.agents.middleware import AgentMiddleware


class ToolActivityMiddleware(AgentMiddleware):
    transformers = (ToolActivityTransformer,)


agent = create_agent(
    model="gpt-5-nano",
    tools=[get_weather],
    middleware=[ToolActivityMiddleware()],
)
```

Tại thời điểm compile, `create_agent` gộp (merge) các factory đăng ký qua middleware với những gì được truyền vào tham số `transformers=` của chính nó. Thứ tự cuối cùng trên graph đã compile là:

1. `ToolCallTransformer` dựng sẵn.
2. Các factory đăng ký qua middleware, theo đúng thứ tự middleware.
3. `transformers=` do caller truyền vào `create_agent`.

Điều này giữ cho projection tool-call dựng sẵn luôn đứng trước các transformer do consumer cung cấp, và để cho các entry do caller cung cấp có tiếng nói cuối cùng.

`PIIMiddleware` dựng sẵn dùng hook này để redact (che/ẩn) thông tin PII khỏi wire output đang stream. Với `apply_to_output=True`, transformer đã đăng ký của nó sẽ scrub (xoá sạch) PII phát hiện được khỏi text delta, argument của tool call, output của tool, và state snapshot trước khi chúng rời khỏi lượt chạy, khép lại khoảng hở mà việc redact ở cấp state trong `after_model` lẽ ra sẽ để lộ PII thô cho các reader đang live-đọc `stream_events(version="v3")`.

```py
from langchain.agents import create_agent
from langchain.agents.middleware import PIIMiddleware

agent = create_agent(
    model="gpt-5-nano",
    tools=[],
    middleware=[
        PIIMiddleware("email", strategy="redact", apply_to_output=True),
    ],
)
```

Xem [PII detection](middleware/built-in.md#pii-detection) để biết đầy đủ bề mặt cấu hình.

Xem [Tự xây dựng projection của riêng bạn](https://docs.langchain.com/oss/python/langgraph/event-streaming#build-your-own-projection) để biết hợp đồng (contract) của transformer.

## Liên quan

* [Streaming](streaming.md) trình bày về các stream mode cấp thấp của Pregel.
* [Tự xây dựng projection của riêng bạn](https://docs.langchain.com/oss/python/langgraph/event-streaming#build-your-own-projection) trình bày cách viết projection đặc thù theo ứng dụng.
* [Các pattern streaming ở Frontend](frontend/overview.md) minh hoạ các use case UI được xây dựng dựa trên state được stream.
