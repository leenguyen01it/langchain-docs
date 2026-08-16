# Streaming

!!! tip ""
    Với ứng dụng mới, chúng tôi khuyến nghị dùng [event streaming](./event-streaming.md), API typed-projection được giới thiệu trong LangGraph v1.2. Event streaming cho bạn các iterator riêng biệt cho từng projection (messages, values, subgraphs, output) để bạn có thể tiêu thụ chúng độc lập thay vì phải rẽ nhánh theo các chunk `stream_mode`.

Trang này nói về stream-mode API của LangGraph. Nó phơi bày việc thực thi graph qua các stream mode như `updates`, `values`, `messages`, `custom`, `checkpoints`, `tasks`, và `debug`. Dùng nó khi bạn cần truy cập trực tiếp vào các sự kiện graph-runtime hoặc output của một stream-mode cụ thể.

## Bắt đầu

### Cách dùng cơ bản

Graph của LangGraph phơi bày các phương thức [`stream`](https://reference.langchain.com/python/langgraph/pregel/#langgraph.pregel.Pregel.stream) (đồng bộ) và [`astream`](https://reference.langchain.com/python/langgraph/pregel/#langgraph.pregel.Pregel.astream) (bất đồng bộ) để trả về output đã stream dưới dạng iterator. Truyền một hoặc nhiều [stream mode](#stream-modes) để kiểm soát dữ liệu bạn nhận được.

```python
for chunk in graph.stream(
    {"topic": "ice cream"},
    stream_mode=["updates", "custom"],  # [!code highlight]
    version="v2",  # [!code highlight]
):
    if chunk["type"] == "updates":
        for node_name, state in chunk["data"].items():
            print(f"Node {node_name} updated: {state}")
    elif chunk["type"] == "custom":
        print(f"Status: {chunk['data']['status']}")
```

```shell title="Output"
Status: thinking of a joke...
Node generate_joke updated: {'joke': 'Why did the ice cream go to school? To get a sundae education!'}
```

??? note "Ví dụ đầy đủ"
    ```python
    from typing import TypedDict
    from langgraph.graph import StateGraph, START, END
    from langgraph.config import get_stream_writer


    class State(TypedDict):
        topic: str
        joke: str


    def generate_joke(state: State):
        writer = get_stream_writer()
        writer({"status": "thinking of a joke..."})
        return {"joke": f"Why did the {state['topic']} go to school? To get a sundae education!"}

    graph = (
        StateGraph(State)
        .add_node(generate_joke)
        .add_edge(START, "generate_joke")
        .add_edge("generate_joke", END)
        .compile()
    )

    for chunk in graph.stream(
        {"topic": "ice cream"},
        stream_mode=["updates", "custom"],
        version="v2",
    ):
        if chunk["type"] == "updates":
            for node_name, state in chunk["data"].items():
                print(f"Node {node_name} updated: {state}")
        elif chunk["type"] == "custom":
            print(f"Status: {chunk['data']['status']}")
    ```

    ```shell title="Output"
    Status: thinking of a joke...
    Node generate_joke updated: {'joke': 'Why did the ice cream go to school? To get a sundae education!'}
    ```

!!! tip ""
    Debug các sự kiện streaming, kiểm tra output LLM theo từng token, và theo dõi độ trễ với [LangSmith](https://smith.langchain.com?utm_source=docs&utm_medium=cta&utm_campaign=langsmith-signup&utm_content=oss-langgraph-streaming). Làm theo [tracing quickstart](https://docs.langchain.com/langsmith/trace-with-langgraph) để thiết lập.

### Định dạng output stream (v2)

!!! note ""
    Yêu cầu LangGraph >= 1.1. Tất cả ví dụ trên trang này dùng `version="v2"`.

Truyền `version="v2"` cho `stream()` hoặc `astream()` để có định dạng output thống nhất. Mỗi chunk là một dict `StreamPart` có cấu trúc nhất quán, bất kể stream mode, số lượng mode, hay cấu hình subgraph:

```python
{
    "type": "values" | "updates" | "messages" | "custom" | "checkpoints" | "tasks" | "debug",
    "ns": (),           # tuple namespace, có giá trị cho sự kiện subgraph
    "data": ...,        # payload thực tế (kiểu khác nhau tuỳ stream mode)
}
```

Mỗi stream mode có một `TypedDict` tương ứng gồm [`ValuesStreamPart`](https://reference.langchain.com/python/langgraph/types/ValuesStreamPart), [`UpdatesStreamPart`](https://reference.langchain.com/python/langgraph/types/UpdatesStreamPart), [`MessagesStreamPart`](https://reference.langchain.com/python/langgraph/types/MessagesStreamPart), [`CustomStreamPart`](https://reference.langchain.com/python/langgraph/types/CustomStreamPart), [`CheckpointStreamPart`](https://reference.langchain.com/python/langgraph/types/CheckpointStreamPart), [`TasksStreamPart`](https://reference.langchain.com/python/langgraph/types/TasksStreamPart), [`DebugStreamPart`](https://reference.langchain.com/python/langgraph/types/DebugStreamPart). Bạn có thể import các kiểu này từ `langgraph.types`. Kiểu union [`StreamPart`](https://reference.langchain.com/python/langgraph/types/StreamPart) là một disjoint union trên `part["type"]`, cho phép type narrowing đầy đủ trong editor và type checker.

Với v1 (mặc định), định dạng output thay đổi tuỳ theo tuỳ chọn streaming của bạn (một mode duy nhất trả về raw data, nhiều mode trả về tuple `(mode, data)`, subgraph trả về tuple `(namespace, data)`). Với v2, định dạng luôn giống nhau:

=== "v2 (mới)"
    ```python
    for chunk in graph.stream(inputs, stream_mode="updates", version="v2"):
        print(chunk["type"])  # "updates"
        print(chunk["ns"])    # ()
        print(chunk["data"])  # {"node_name": {"key": "value"}}
    ```

=== "v1 (mặc định hiện tại)"
    ```python
    for chunk in graph.stream(inputs, stream_mode="updates"):
        print(chunk)  # {"node_name": {"key": "value"}}
    ```

Định dạng v2 cũng cho phép type narrowing, nghĩa là bạn có thể lọc chunk theo `chunk["type"]` và nhận đúng kiểu payload. Mỗi nhánh thu hẹp `part["data"]` về đúng kiểu cho mode đó:

```python
for part in graph.stream(
    {"topic": "ice cream"},
    stream_mode=["values", "updates", "messages", "custom"],
    version="v2",
):
    if part["type"] == "values":
        # ValuesStreamPart: snapshot state đầy đủ sau mỗi bước
        print(f"State: topic={part['data']['topic']}")
    elif part["type"] == "updates":
        # UpdatesStreamPart: chỉ các key đã thay đổi từ mỗi node
        for node_name, state in part["data"].items():
            print(f"Node `{node_name}` updated: {state}")
    elif part["type"] == "messages":
        # MessagesStreamPart: (message_chunk, metadata) từ lệnh gọi LLM
        msg, metadata = part["data"]
        print(msg.content, end="", flush=True)
    elif part["type"] == "custom":
        # CustomStreamPart: dữ liệu tuỳ ý từ get_stream_writer()
        print(f"Progress: {part['data']['progress']}%")
```

## Các stream mode

Truyền một hoặc nhiều trong số các stream mode sau dưới dạng list cho phương thức [`stream`](https://reference.langchain.com/python/langgraph/graphs/#langgraph.graph.state.CompiledStateGraph.stream) hoặc [`astream`](https://reference.langchain.com/python/langgraph/graphs/#langgraph.graph.state.CompiledStateGraph.astream):

| Mode                        | Kiểu                                                                                                  | Mô tả                                                                                                                          |
| :-------------------------- | :---------------------------------------------------------------------------------------------------- | :----------------------------------------------------------------------------------------------------------------------------------- |
| [values](#trang-thai-graph)      | [`ValuesStreamPart`](https://reference.langchain.com/python/langgraph/types/ValuesStreamPart)         | State đầy đủ sau mỗi bước.                                                                                                          |
| [updates](#trang-thai-graph)     | [`UpdatesStreamPart`](https://reference.langchain.com/python/langgraph/types/UpdatesStreamPart)       | Các cập nhật state sau mỗi bước. Nhiều cập nhật trong cùng một bước được stream riêng biệt.                                            |
| [messages](#llm-tokens)     | [`MessagesStreamPart`](https://reference.langchain.com/python/langgraph/types/MessagesStreamPart)     | Tuple 2 phần tử (LLM token, metadata) từ lệnh gọi LLM.                                                                                    |
| [custom](#du-lieu-tuy-chinh)      | [`CustomStreamPart`](https://reference.langchain.com/python/langgraph/types/CustomStreamPart)         | Dữ liệu tuỳ chỉnh phát ra từ các node qua [`get_stream_writer`](https://reference.langchain.com/python/langgraph/config/get_stream_writer). |
| [checkpoints](#checkpoint) | [`CheckpointStreamPart`](https://reference.langchain.com/python/langgraph/types/CheckpointStreamPart) | Sự kiện checkpoint (cùng định dạng với output của `get_state()`). Yêu cầu checkpointer.                                                           |
| [tasks](#tasks)             | [`TasksStreamPart`](https://reference.langchain.com/python/langgraph/types/TasksStreamPart)           | Sự kiện task bắt đầu/kết thúc kèm kết quả và lỗi. Yêu cầu checkpointer.                                                           |
| [debug](#debug)             | [`DebugStreamPart`](https://reference.langchain.com/python/langgraph/types/DebugStreamPart)           | Toàn bộ thông tin có sẵn, kết hợp `checkpoints` và `tasks` với metadata bổ sung.                                                         |

### Trạng thái graph

Dùng stream mode `updates` và `values` để stream state của graph khi nó thực thi.

* `updates` stream **các cập nhật** vào state sau mỗi bước của graph.
* `values` stream **toàn bộ giá trị** của state sau mỗi bước của graph.

```python
from typing import TypedDict
from langgraph.graph import StateGraph, START, END


class State(TypedDict):
  topic: str
  joke: str


def refine_topic(state: State):
    return {"topic": state["topic"] + " and cats"}


def generate_joke(state: State):
    return {"joke": f"This is a joke about {state['topic']}"}

graph = (
  StateGraph(State)
  .add_node(refine_topic)
  .add_node(generate_joke)
  .add_edge(START, "refine_topic")
  .add_edge("refine_topic", "generate_joke")
  .add_edge("generate_joke", END)
  .compile()
)
```

=== "updates"
    Dùng cái này để chỉ stream **các cập nhật state** mà node trả về sau mỗi bước. Output được stream bao gồm tên node cũng như cập nhật đó.

    ```python
    for chunk in graph.stream(
        {"topic": "ice cream"},
        stream_mode="updates",  # [!code highlight]
        version="v2",  # [!code highlight]
    ):
        if chunk["type"] == "updates":
            for node_name, state in chunk["data"].items():
                print(f"Node `{node_name}` updated: {state}")
    ```

    ```shell title="Output"
    Node `refine_topic` updated: {'topic': 'ice cream and cats'}
    Node `generate_joke` updated: {'joke': 'This is a joke about ice cream and cats'}
    ```

=== "values"
    Dùng cái này để stream **toàn bộ state** của graph sau mỗi bước.

    ```python
    for chunk in graph.stream(
        {"topic": "ice cream"},
        stream_mode="values",  # [!code highlight]
        version="v2",  # [!code highlight]
    ):
        if chunk["type"] == "values":
            print(f"topic: {chunk['data']['topic']}, joke: {chunk['data']['joke']}")
    ```

    ```shell title="Output"
    topic: ice cream, joke:
    topic: ice cream and cats, joke:
    topic: ice cream and cats, joke: This is a joke about ice cream and cats
    ```

### LLM tokens

Dùng streaming mode `messages` để stream output của Large Language Model (LLM) **theo từng token** từ bất kỳ phần nào trong graph của bạn, bao gồm node, tool, subgraph, hoặc task.

Output được stream từ [`messages` mode](#cac-stream-mode) là một tuple `(message_chunk, metadata)`, trong đó:

* `message_chunk`: token hoặc đoạn message từ LLM.
* `metadata`: dictionary chứa chi tiết về node của graph và lệnh gọi LLM.

> Nếu LLM của bạn không có sẵn dưới dạng tích hợp LangChain, bạn có thể stream output của nó bằng mode `custom` thay thế. Xem [dùng với LLM bất kỳ](#dung-voi-llm-bat-ky) để biết chi tiết.

!!! warning "Cần cấu hình thủ công cho async trong Python < 3.11"
    Khi dùng Python < 3.11 với code async, bạn phải truyền tường minh [`RunnableConfig`](https://reference.langchain.com/python/langchain-core/runnables/config/RunnableConfig) cho `ainvoke()` để bật streaming đúng cách. Xem [Async với Python < 3.11](#async-voi-python-311) để biết chi tiết hoặc nâng cấp lên Python 3.11+.

```python
from dataclasses import dataclass

from langchain.chat_models import init_chat_model
from langgraph.graph import StateGraph, START


@dataclass
class MyState:
    topic: str
    joke: str = ""


model = init_chat_model(model="gpt-5.4-mini")

def call_model(state: MyState):
    """Call the LLM to generate a joke about a topic"""
    # Lưu ý các sự kiện message vẫn được phát ra ngay cả khi LLM được chạy bằng .invoke thay vì .stream
    model_response = model.invoke(  # [!code highlight]
        [
            {"role": "user", "content": f"Generate a joke about {state.topic}"}
        ]
    )
    return {"joke": model_response.content}

graph = (
    StateGraph(MyState)
    .add_node(call_model)
    .add_edge(START, "call_model")
    .compile()
)

# Stream mode "messages" stream các token LLM kèm metadata
# Dùng version="v2" để có định dạng StreamPart thống nhất
for chunk in graph.stream(
    {"topic": "ice cream"},
    stream_mode="messages",  # [!code highlight]
    version="v2",  # [!code highlight]
):
    if chunk["type"] == "messages":
        message_chunk, metadata = chunk["data"]
        if message_chunk.content:
            print(message_chunk.content, end="|", flush=True)
```

#### Lọc theo lệnh gọi LLM

Bạn có thể gắn `tags` với các lệnh gọi LLM để lọc các token được stream theo lệnh gọi LLM.

```python
from langchain.chat_models import init_chat_model

# model_1 được gắn tag "joke"
model_1 = init_chat_model(model="gpt-5.4-mini", tags=['joke'])
# model_2 được gắn tag "poem"
model_2 = init_chat_model(model="gpt-5.4-mini", tags=['poem'])

graph = ... # định nghĩa một graph dùng các LLM này

# stream_mode được đặt thành "messages" để stream token LLM
# metadata chứa thông tin về lệnh gọi LLM, bao gồm cả tags
async for chunk in graph.astream(
    {"topic": "cats"},
    stream_mode="messages",  # [!code highlight]
    version="v2",  # [!code highlight]
):
    if chunk["type"] == "messages":
        msg, metadata = chunk["data"]
        # Lọc các token được stream theo field tags trong metadata để chỉ lấy
        # các token từ lệnh gọi LLM có tag "joke"
        if metadata["tags"] == ["joke"]:
            print(msg.content, end="|", flush=True)
```

??? note "Ví dụ mở rộng: lọc theo tag"
    ```python
    from typing import TypedDict

    from langchain.chat_models import init_chat_model
    from langgraph.graph import START, StateGraph

    # joke_model được gắn tag "joke"
    joke_model = init_chat_model(model="gpt-5.4-mini", tags=["joke"])
    # poem_model được gắn tag "poem"
    poem_model = init_chat_model(model="gpt-5.4-mini", tags=["poem"])


    class State(TypedDict):
          topic: str
          joke: str
          poem: str


    async def call_model(state, config):
          topic = state["topic"]
          print("Writing joke...")
          # Lưu ý: truyền config tường minh là bắt buộc với python < 3.11
          # vì hỗ trợ context var chưa có trước phiên bản đó: https://docs.python.org/3/library/asyncio-task.html#creating-tasks
          # config được truyền tường minh để đảm bảo context var được lan truyền đúng cách
          # Điều này cần thiết cho Python < 3.11 khi dùng code async. Xem phần async để biết chi tiết
          joke_response = await joke_model.ainvoke(
                [{"role": "user", "content": f"Write a joke about {topic}"}],
                config,
          )
          print("\n\nWriting poem...")
          poem_response = await poem_model.ainvoke(
                [{"role": "user", "content": f"Write a short poem about {topic}"}],
                config,
          )
          return {"joke": joke_response.content, "poem": poem_response.content}


    graph = (
          StateGraph(State)
          .add_node(call_model)
          .add_edge(START, "call_model")
          .compile()
    )

    # stream_mode được đặt thành "messages" để stream token LLM
    # metadata chứa thông tin về lệnh gọi LLM, bao gồm cả tags
    async for chunk in graph.astream(
          {"topic": "cats"},
          stream_mode="messages",
          version="v2",
    ):
        if chunk["type"] == "messages":
            msg, metadata = chunk["data"]
            if metadata["tags"] == ["joke"]:
                print(msg.content, end="|", flush=True)
    ```

#### Bỏ message ra khỏi stream

Dùng tag `nostream` để loại hoàn toàn output LLM khỏi stream. Các lệnh gọi được gắn tag `nostream` vẫn chạy và tạo output; chỉ là token của chúng không được phát ra ở mode `messages`.

Điều này hữu ích khi:

* Bạn cần output LLM để xử lý nội bộ (ví dụ structured output) nhưng không muốn stream nó cho client
* Bạn stream cùng nội dung đó qua một kênh khác (ví dụ custom UI message) và muốn tránh output trùng lặp trong stream `messages`

```python
from typing import Any, TypedDict

from langchain_anthropic import ChatAnthropic
from langgraph.graph import START, StateGraph

stream_model = ChatAnthropic(model_name="claude-haiku-4-5-20251001")
internal_model = ChatAnthropic(model_name="claude-haiku-4-5-20251001").with_config(
    {"tags": ["nostream"]}
)


class State(TypedDict):
    topic: str
    answer: str
    notes: str


def answer(state: State) -> dict[str, Any]:
    r = stream_model.invoke(
        [{"role": "user", "content": f"Reply briefly about {state['topic']}"}]
    )
    return {"answer": r.content}


def internal_notes(state: State) -> dict[str, Any]:
    # Token từ model này bị loại khỏi stream_mode="messages" vì có tag nostream
    r = internal_model.invoke(
        [{"role": "user", "content": f"Private notes on {state['topic']}"}]
    )
    return {"notes": r.content}


graph = (
    StateGraph(State)
    .add_node("write_answer", answer)
    .add_node("internal_notes", internal_notes)
    .add_edge(START, "write_answer")
    .add_edge("write_answer", "internal_notes")
    .compile()
)

initial_state: State = {"topic": "AI", "answer": "", "notes": ""}
stream = graph.stream_events(initial_state, version="v3")
```

#### Lọc theo node

Để chỉ stream token từ các node cụ thể, dùng `stream_mode="messages"` và lọc output theo field `langgraph_node` trong metadata được stream:

```python
# Stream mode "messages" stream các token LLM kèm metadata
# Dùng version="v2" để có định dạng StreamPart thống nhất
for chunk in graph.stream(
    inputs,
    stream_mode="messages",  # [!code highlight]
    version="v2",  # [!code highlight]
):
    if chunk["type"] == "messages":
        msg, metadata = chunk["data"]
        # Lọc các token được stream theo field langgraph_node trong metadata
        # để chỉ lấy token từ node được chỉ định
        if msg.content and metadata["langgraph_node"] == "some_node_name":
            ...
```

??? note "Ví dụ mở rộng: stream token LLM từ các node cụ thể"
    ```python
    from typing import TypedDict
    from langgraph.graph import START, StateGraph
    from langchain_openai import ChatOpenAI

    model = ChatOpenAI(model="gpt-5.4-mini")


    class State(TypedDict):
          topic: str
          joke: str
          poem: str


    def write_joke(state: State):
          topic = state["topic"]
          joke_response = model.invoke(
                [{"role": "user", "content": f"Write a joke about {topic}"}]
          )
          return {"joke": joke_response.content}


    def write_poem(state: State):
          topic = state["topic"]
          poem_response = model.invoke(
                [{"role": "user", "content": f"Write a short poem about {topic}"}]
          )
          return {"poem": poem_response.content}


    graph = (
          StateGraph(State)
          .add_node(write_joke)
          .add_node(write_poem)
          # viết cả joke và poem đồng thời
          .add_edge(START, "write_joke")
          .add_edge(START, "write_poem")
          .compile()
    )

    # Stream mode "messages" stream các token LLM kèm metadata
    # Dùng version="v2" để có định dạng StreamPart thống nhất
    for chunk in graph.stream(
        {"topic": "cats"},
        stream_mode="messages",  # [!code highlight]
        version="v2",  # [!code highlight]
    ):
        if chunk["type"] == "messages":
            msg, metadata = chunk["data"]
            # Lọc các token được stream theo field langgraph_node trong metadata
            # để chỉ lấy token từ node write_poem
            if msg.content and metadata["langgraph_node"] == "write_poem":
                print(msg.content, end="|", flush=True)
    ```

### Dữ liệu tuỳ chỉnh

Để gửi **dữ liệu tuỳ ý do người dùng định nghĩa** từ bên trong một node hoặc tool của LangGraph, làm theo các bước sau:

1. Dùng [`get_stream_writer`](https://reference.langchain.com/python/langgraph/config/get_stream_writer) để truy cập stream writer và phát ra dữ liệu tuỳ chỉnh.
2. Đặt `stream_mode="custom"` khi gọi `.stream()` hoặc `.astream()` để nhận dữ liệu tuỳ chỉnh trong stream. Bạn có thể kết hợp nhiều mode (ví dụ `["updates", "custom"]`), nhưng ít nhất một mode phải là `"custom"`.

!!! warning "Không có `get_stream_writer` trong async với Python < 3.11"
    Trong code async chạy trên Python < 3.11, [`get_stream_writer`](https://reference.langchain.com/python/langgraph/config/get_stream_writer) sẽ không hoạt động.
    Thay vào đó, hãy thêm một tham số `writer` vào node hoặc tool của bạn và truyền nó thủ công.
    Xem [Async với Python < 3.11](#async-voi-python-311) để có ví dụ cách dùng.

=== "node"
    ```python
    from typing import TypedDict
    from langgraph.config import get_stream_writer
    from langgraph.graph import StateGraph, START

    class State(TypedDict):
        query: str
        answer: str

    def node(state: State):
        # Lấy stream writer để gửi dữ liệu tuỳ chỉnh
        writer = get_stream_writer()
        # Phát ra một cặp key-value tuỳ chỉnh (ví dụ cập nhật tiến độ)
        writer({"custom_key": "Generating custom data inside node"})
        return {"answer": "some data"}

    graph = (
        StateGraph(State)
        .add_node(node)
        .add_edge(START, "node")
        .compile()
    )

    inputs = {"query": "example"}

    # Đặt stream_mode="custom" để nhận dữ liệu tuỳ chỉnh trong stream
    for chunk in graph.stream(inputs, stream_mode="custom", version="v2"):
        if chunk["type"] == "custom":
            print(f"Custom event: {chunk['data']['custom_key']}")
    ```

=== "tool"
    ```python
    from langchain.tools import tool
    from langgraph.config import get_stream_writer

    @tool
    def query_database(query: str) -> str:
        """Query the database."""
        # Truy cập stream writer để gửi dữ liệu tuỳ chỉnh
        writer = get_stream_writer()  # [!code highlight]
        # Phát ra một cặp key-value tuỳ chỉnh (ví dụ cập nhật tiến độ)
        writer({"data": "Retrieved 0/100 records", "type": "progress"})  # [!code highlight]
        # thực hiện query
        # Phát ra một cặp key-value tuỳ chỉnh khác
        writer({"data": "Retrieved 100/100 records", "type": "progress"})
        return "some-answer"


    graph = ... # định nghĩa một graph dùng tool này

    # Đặt stream_mode="custom" để nhận dữ liệu tuỳ chỉnh trong stream
    for chunk in graph.stream(inputs, stream_mode="custom", version="v2"):
        if chunk["type"] == "custom":
            print(f"{chunk['data']['type']}: {chunk['data']['data']}")
    ```

### Output của subgraph

Để bao gồm output từ các [subgraph](./use-subgraphs.md) trong output được stream, bạn có thể đặt `subgraphs=True` trong phương thức `.stream()` của graph cha. Điều này sẽ stream output từ cả graph cha và mọi subgraph.

Output sẽ được stream dưới dạng tuple `(namespace, data)`, trong đó `namespace` là một tuple chứa đường dẫn tới node nơi một subgraph được gọi, ví dụ `("parent_node:<task_id>", "child_node:<task_id>")`.

=== "v2 (LangGraph >= 1.1)"
    Với `version="v2"`, sự kiện subgraph dùng cùng định dạng `StreamPart`. Field `ns` xác định nguồn:

    ```python
    for chunk in graph.stream(
        {"foo": "foo"},
        subgraphs=True,  # [!code highlight]
        stream_mode="updates",
        version="v2", # [!code highlight]
    ):
        print(chunk["type"])  # "updates"
        print(chunk["ns"])    # () cho root, ("node_name:<task_id>",) cho subgraph
        print(chunk["data"])  # {"node_name": {"key": "value"}}
    ```

=== "v1 (mặc định)"
    ```python
    for chunk in graph.stream(
        {"foo": "foo"},
        # Đặt subgraphs=True để stream output từ subgraph
        subgraphs=True,  # [!code highlight]
        stream_mode="updates",
    ):
        print(chunk)
    ```

!!! note ""
    Điều này áp dụng cho mọi `stream_mode`, bao gồm cả `"messages"`. Các agent builder như [`create_agent`](https://reference.langchain.com/python/langchain/agents/factory/create_agent) trả về một **graph đã compile**, nên việc thêm nó làm node biến nó thành một subgraph. Nếu không có `subgraphs=True`, `stream_mode="messages"` trên graph cha sẽ không phát ra các chunk token từ lệnh gọi LLM của agent bên trong. Gọi trực tiếp `agent.stream(...)` thì sẽ phát ra, đó là lý do vấn đề này thường chỉ lộ ra sau khi wrap.

    ```python
    from langchain.agents import create_agent
    from langgraph.graph import END, START, StateGraph

    graph = (
        StateGraph(State)
        .add_node("agent", create_agent(model, tools, state_schema=State))
        .add_edge(START, "agent")
        .add_edge("agent", END)
        .compile()
    )

    for chunk in graph.stream(
        {"messages": [{"role": "user", "content": "..."}]},
        stream_mode="messages",
        subgraphs=True,  # [!code highlight]
        version="v2",
    ):
        print(chunk["type"])  # "messages"
        print(chunk["ns"])    # () cho root, ("agent:<task_id>",) cho subgraph
        print(chunk["data"])  # (token, metadata)
    ```

??? note "Ví dụ mở rộng: stream từ subgraph"
    ```python
    from langgraph.graph import START, StateGraph
    from typing import TypedDict

    # Định nghĩa subgraph
    class SubgraphState(TypedDict):
        foo: str  # lưu ý key này được chia sẻ với state của graph cha
        bar: str

    def subgraph_node_1(state: SubgraphState):
        return {"bar": "bar"}

    def subgraph_node_2(state: SubgraphState):
        return {"foo": state["foo"] + state["bar"]}

    subgraph_builder = StateGraph(SubgraphState)
    subgraph_builder.add_node(subgraph_node_1)
    subgraph_builder.add_node(subgraph_node_2)
    subgraph_builder.add_edge(START, "subgraph_node_1")
    subgraph_builder.add_edge("subgraph_node_1", "subgraph_node_2")
    subgraph = subgraph_builder.compile()

    # Định nghĩa graph cha
    class ParentState(TypedDict):
        foo: str

    def node_1(state: ParentState):
        return {"foo": "hi! " + state["foo"]}

    builder = StateGraph(ParentState)
    builder.add_node("node_1", node_1)
    builder.add_node("node_2", subgraph)
    builder.add_edge(START, "node_1")
    builder.add_edge("node_1", "node_2")
    graph = builder.compile()

    for chunk in graph.stream(
        {"foo": "foo"},
        stream_mode="updates",
        # Đặt subgraphs=True để stream output từ subgraph
        subgraphs=True,  # [!code highlight]
        version="v2",  # [!code highlight]
    ):
        if chunk["type"] == "updates":
            if chunk["ns"]:
                print(f"Subgraph {chunk['ns']}: {chunk['data']}")
            else:
                print(f"Root: {chunk['data']}")
    ```

    ```
    Root: {'node_1': {'foo': 'hi! foo'}}
    Subgraph ('node_2:dfddc4ba-c3c5-6887-5012-a243b5b377c2',): {'subgraph_node_1': {'bar': 'bar'}}
    Subgraph ('node_2:dfddc4ba-c3c5-6887-5012-a243b5b377c2',): {'subgraph_node_2': {'foo': 'hi! foobar'}}
    Root: {'node_2': {'foo': 'hi! foobar'}}
    ```

    **Lưu ý** rằng ta không chỉ nhận được cập nhật của node, mà còn nhận cả namespace cho biết ta đang stream từ graph (hay subgraph) nào.

### Checkpoint

Dùng streaming mode `checkpoints` để nhận các sự kiện checkpoint khi graph thực thi. Mỗi sự kiện checkpoint có cùng định dạng với output của `get_state()`. Yêu cầu một [checkpointer](./persistence.md).

```python
from langgraph.checkpoint.memory import MemorySaver

graph = (
    StateGraph(State)
    .add_node(refine_topic)
    .add_node(generate_joke)
    .add_edge(START, "refine_topic")
    .add_edge("refine_topic", "generate_joke")
    .add_edge("generate_joke", END)
    .compile(checkpointer=MemorySaver())
)

config = {"configurable": {"thread_id": "1"}}

for chunk in graph.stream(
    {"topic": "ice cream"},
    config=config,
    stream_mode="checkpoints",  # [!code highlight]
    version="v2",  # [!code highlight]
):
    if chunk["type"] == "checkpoints":
        print(chunk["data"])
```

### Tasks

Dùng streaming mode `tasks` để nhận sự kiện task bắt đầu và kết thúc khi graph thực thi. Sự kiện task bao gồm thông tin về node nào đang chạy, kết quả của nó, và mọi lỗi. Yêu cầu một [checkpointer](./persistence.md).

```python
from langgraph.checkpoint.memory import MemorySaver

graph = (
    StateGraph(State)
    .add_node(refine_topic)
    .add_node(generate_joke)
    .add_edge(START, "refine_topic")
    .add_edge("refine_topic", "generate_joke")
    .add_edge("generate_joke", END)
    .compile(checkpointer=MemorySaver())
)

config = {"configurable": {"thread_id": "1"}}

for chunk in graph.stream(
    {"topic": "ice cream"},
    config=config,
    stream_mode="tasks",  # [!code highlight]
    version="v2",  # [!code highlight]
):
    if chunk["type"] == "tasks":
        print(chunk["data"])
```

### Debug

Dùng streaming mode `debug` để stream nhiều thông tin nhất có thể trong suốt quá trình thực thi graph. Output được stream bao gồm tên node cũng như toàn bộ state.

```python
for chunk in graph.stream(
    {"topic": "ice cream"},
    stream_mode="debug",  # [!code highlight]
    version="v2",  # [!code highlight]
):
    if chunk["type"] == "debug":
        print(chunk["data"])
```

!!! note ""
    Mode `debug` kết hợp các sự kiện `checkpoints` và `tasks` với metadata bổ sung. Dùng trực tiếp `checkpoints` hoặc `tasks` nếu bạn chỉ cần một phần thông tin debug.

### Nhiều mode cùng lúc

Bạn có thể truyền một list làm tham số `stream_mode` để stream nhiều mode cùng lúc.

Với `version="v2"`, mỗi chunk là một dict `StreamPart`. Dùng `chunk["type"]` để phân biệt giữa các mode:

=== "v2"
    ```python
    for chunk in graph.stream(inputs, stream_mode=["updates", "custom"], version="v2"):
        if chunk["type"] == "updates":
            for node_name, state in chunk["data"].items():
                print(f"Node `{node_name}` updated: {state}")
        elif chunk["type"] == "custom":
            print(f"Custom event: {chunk['data']}")
    ```

=== "v1"
    ```python
    for mode, chunk in graph.stream(inputs, stream_mode=["updates", "custom"]):
        print(chunk)
    ```

## Nâng cao

### Dùng với LLM bất kỳ

Bạn có thể dùng `stream_mode="custom"` để stream dữ liệu từ **bất kỳ LLM API nào**, kể cả khi API đó **không** triển khai giao diện chat model của LangChain.

Điều này cho phép bạn tích hợp các client LLM thô hoặc các dịch vụ bên ngoài có giao diện streaming riêng, giúp LangGraph rất linh hoạt cho các thiết lập tuỳ chỉnh.

```python
from langgraph.config import get_stream_writer

def call_arbitrary_model(state):
    """Example node that calls an arbitrary model and streams the output"""
    # Lấy stream writer để gửi dữ liệu tuỳ chỉnh
    writer = get_stream_writer()  # [!code highlight]
    # Giả sử bạn có một client streaming trả về từng chunk
    # Sinh token LLM bằng client streaming tuỳ chỉnh của bạn
    for chunk in your_custom_streaming_client(state["topic"]):
        # Dùng writer để gửi dữ liệu tuỳ chỉnh vào stream
        writer({"custom_llm_chunk": chunk})  # [!code highlight]
    return {"result": "completed"}

graph = (
    StateGraph(State)
    .add_node(call_arbitrary_model)
    # Thêm các node và cạnh khác nếu cần
    .compile()
)
# Đặt stream_mode="custom" để nhận dữ liệu tuỳ chỉnh trong stream
for chunk in graph.stream(
    {"topic": "cats"},
    stream_mode="custom",  # [!code highlight]
    version="v2",  # [!code highlight]
):
    if chunk["type"] == "custom":
        # Dữ liệu chunk sẽ chứa dữ liệu tuỳ chỉnh được stream từ llm
        print(chunk["data"])
```

??? note "Ví dụ mở rộng: stream một chat model bất kỳ"
    ```python
    import operator
    import json

    from typing import TypedDict
    from typing_extensions import Annotated
    from langgraph.graph import StateGraph, START

    from openai import AsyncOpenAI

    openai_client = AsyncOpenAI()
    model_name = "gpt-5.4-mini"


    async def stream_tokens(model_name: str, messages: list[dict]):
        response = await openai_client.chat.completions.create(
            messages=messages, model=model_name, stream=True
        )
        role = None
        async for chunk in response:
            delta = chunk.choices[0].delta

            if delta.role is not None:
                role = delta.role

            if delta.content:
                yield {"role": role, "content": delta.content}


    # đây là tool của ta
    async def get_items(place: str) -> str:
        """Use this tool to list items one might find in a place you're asked about."""
        writer = get_stream_writer()
        response = ""
        async for msg_chunk in stream_tokens(
            model_name,
            [
                {
                    "role": "user",
                    "content": (
                        "Can you tell me what kind of items "
                        f"i might find in the following place: '{place}'. "
                        "List at least 3 such items separating them by a comma. "
                        "And include a brief description of each item."
                    ),
                }
            ],
        ):
            response += msg_chunk["content"]
            writer(msg_chunk)

        return response


    class State(TypedDict):
        messages: Annotated[list[dict], operator.add]


    # đây là node graph gọi tool
    async def call_tool(state: State):
        ai_message = state["messages"][-1]
        tool_call = ai_message["tool_calls"][-1]

        function_name = tool_call["function"]["name"]
        if function_name != "get_items":
            raise ValueError(f"Tool {function_name} not supported")

        function_arguments = tool_call["function"]["arguments"]
        arguments = json.loads(function_arguments)

        function_response = await get_items(**arguments)
        tool_message = {
            "tool_call_id": tool_call["id"],
            "role": "tool",
            "name": function_name,
            "content": function_response,
        }
        return {"messages": [tool_message]}


    graph = (
        StateGraph(State)
        .add_node(call_tool)
        .add_edge(START, "call_tool")
        .compile()
    )
    ```

    Hãy gọi graph với một [`AIMessage`](https://reference.langchain.com/python/langchain-core/messages/ai/AIMessage) có kèm một tool call:

    ```python
    inputs = {
        "messages": [
            {
                "content": None,
                "role": "assistant",
                "tool_calls": [
                    {
                        "id": "1",
                        "function": {
                            "arguments": '{"place":"bedroom"}',
                            "name": "get_items",
                        },
                        "type": "function",
                    }
                ],
            }
        ]
    }

    async for chunk in graph.astream(
        inputs,
        stream_mode="custom",
        version="v2",
    ):
        if chunk["type"] == "custom":
            print(chunk["data"]["content"], end="|", flush=True)
    ```

### Tắt streaming cho các chat model cụ thể

Nếu ứng dụng của bạn kết hợp các model hỗ trợ streaming với các model không hỗ trợ, bạn có thể cần tắt streaming tường minh cho những model không hỗ trợ.

Đặt `streaming=False` khi khởi tạo model.

=== "init_chat_model"
    ```python
    from langchain.chat_models import init_chat_model

    model = init_chat_model(
        "claude-sonnet-4-6",
        # Đặt streaming=False để tắt streaming cho chat model
        streaming=False  # [!code highlight]
    )
    ```

=== "Giao diện chat model"
    ```python
    from langchain_openai import ChatOpenAI

    # Đặt streaming=False để tắt streaming cho chat model
    model = ChatOpenAI(model="gpt-5.5", streaming=False)
    ```

!!! note ""
    Không phải mọi tích hợp chat model đều hỗ trợ tham số `streaming`. Nếu model của bạn không hỗ trợ, hãy dùng `disable_streaming=True` thay thế. Tham số này có sẵn trên mọi chat model thông qua base class.

### Chuyển sang v2

Định dạng streaming v2 (dùng xuyên suốt trang này) cung cấp một định dạng output thống nhất. Dưới đây là tóm tắt các khác biệt chính và cách chuyển đổi:

| Kịch bản                    | v1 (mặc định)                       | v2 (`version="v2"`)                               |
| --------------------------- | ---------------------------------- | ------------------------------------------------- |
| Một stream mode duy nhất          | Raw data (dict)                    | dict `StreamPart` với `type`, `ns`, `data`       |
| Nhiều stream mode       | Tuple `(mode, data)`              | Cùng dict `StreamPart`, lọc theo `chunk["type"]` |
| Streaming subgraph          | Tuple `(namespace, data)`         | Cùng dict `StreamPart`, kiểm tra `chunk["ns"]`       |
| Nhiều mode + subgraph  | Triple `(namespace, mode, data)`  | Cùng dict `StreamPart`                            |
| Kiểu trả về của `invoke()`      | Dict thuần (state)                 | `GraphOutput` với `.value` và `.interrupts`     |
| Vị trí interrupt (stream) | Key `__interrupt__` trong state dict  | Field `interrupts` trên các phần stream `values` |
| Vị trí interrupt (invoke) | Key `__interrupt__` trong result dict | Thuộc tính `.interrupts` trên `GraphOutput`          |
| Output Pydantic/dataclass   | Trả về dict thuần                 | Ép kiểu về instance model/dataclass               |

#### Định dạng invoke v2

Khi bạn truyền `version="v2"` cho `invoke()` hoặc `ainvoke()`, nó trả về một đối tượng [`GraphOutput`](https://reference.langchain.com/python/langgraph/types/GraphOutput) với các thuộc tính `.value` và `.interrupts`:

```python
from langgraph.types import GraphOutput

result = graph.invoke(inputs, version="v2")

assert isinstance(result, GraphOutput)
result.value       # output của bạn: dict, Pydantic model, hoặc dataclass
result.interrupts  # tuple[Interrupt, ...], rỗng nếu không có interrupt nào xảy ra
```

Với bất kỳ stream mode nào khác ngoài `"values"` mặc định, `invoke(..., stream_mode="updates", version="v2")` trả về `list[StreamPart]` thay vì `list[tuple]`.

!!! warning ""
    Truy cập kiểu dict trên `GraphOutput` (`result["key"]`, `"key" in result`, `result["__interrupt__"]`) vẫn hoạt động để tương thích ngược nhưng đã bị **deprecated** và sẽ bị gỡ bỏ trong một phiên bản tương lai. Hãy chuyển sang dùng `result.value` và `result.interrupts`.

Điều này tách biệt state khỏi metadata interrupt. Với v1, interrupt được nhúng trong dict trả về dưới key `__interrupt__`:

=== "v2 (mới)"
    ```python
    config = {"configurable": {"thread_id": "thread-1"}}
    result = graph.invoke(inputs, config=config, version="v2")

    if result.interrupts:
        print(result.interrupts[0].value)
        graph.invoke(Command(resume=True), config=config, version="v2")
    ```

=== "v1 (mặc định hiện tại)"
    ```python
    config = {"configurable": {"thread_id": "thread-1"}}
    result = graph.invoke(inputs, config=config)

    if "__interrupt__" in result:
        print(result["__interrupt__"][0].value)
        graph.invoke(Command(resume=True), config=config)
    ```

#### Ép kiểu state Pydantic và dataclass

Khi state của graph là một Pydantic model hoặc dataclass, mode `values` của v2 tự động ép output về đúng kiểu:

```python
from pydantic import BaseModel
from typing import Annotated
import operator

class MyState(BaseModel):
    value: str
    items: Annotated[list[str], operator.add]

# Với version="v2", chunk["data"] là một instance MyState
for chunk in graph.stream(
    {"value": "x", "items": []}, stream_mode="values", version="v2"
):
    print(type(chunk["data"]))  # <class 'MyState'>
```

### Async với Python < 3.11

Trong các phiên bản Python < 3.11, [asyncio task](https://docs.python.org/3/library/asyncio-task.html#asyncio.create_task) không hỗ trợ tham số `context`.
Điều này giới hạn khả năng của LangGraph trong việc tự động lan truyền context, và ảnh hưởng tới cơ chế streaming của LangGraph theo hai cách chính:

1. Bạn **bắt buộc** phải truyền tường minh [`RunnableConfig`](https://python.langchain.com/docs/concepts/runnables/#runnableconfig) vào các lệnh gọi LLM async (ví dụ `ainvoke()`), vì callback không được tự động lan truyền.
2. Bạn **không thể** dùng [`get_stream_writer`](https://reference.langchain.com/python/langgraph/config/get_stream_writer) trong node hoặc tool async, bạn phải truyền tham số `writer` trực tiếp.

??? note "Ví dụ mở rộng: lệnh gọi LLM async với config thủ công"
    ```python
    from typing import TypedDict
    from langgraph.graph import START, StateGraph
    from langchain.chat_models import init_chat_model

    model = init_chat_model(model="gpt-5.4-mini")

    class State(TypedDict):
        topic: str
        joke: str

    # Nhận config như một tham số trong hàm node async
    async def call_model(state, config):
        topic = state["topic"]
        print("Generating joke...")
        # Truyền config cho model.ainvoke() để đảm bảo lan truyền context đúng cách
        joke_response = await model.ainvoke(  # [!code highlight]
            [{"role": "user", "content": f"Write a joke about {topic}"}],
            config,
        )
        return {"joke": joke_response.content}

    graph = (
        StateGraph(State)
        .add_node(call_model)
        .add_edge(START, "call_model")
        .compile()
    )

    # Đặt stream_mode="messages" để stream token LLM
    async for chunk in graph.astream(
        {"topic": "ice cream"},
        stream_mode="messages",  # [!code highlight]
        version="v2",  # [!code highlight]
    ):
        if chunk["type"] == "messages":
            message_chunk, metadata = chunk["data"]
            if message_chunk.content:
                print(message_chunk.content, end="|", flush=True)
    ```

??? note "Ví dụ mở rộng: streaming tuỳ chỉnh async với stream writer"
    ```python
    from typing import TypedDict
    from langgraph.types import StreamWriter

    class State(TypedDict):
          topic: str
          joke: str

    # Thêm writer như một tham số trong chữ ký hàm của node hoặc tool async
    # LangGraph sẽ tự động truyền stream writer vào hàm
    async def generate_joke(state: State, writer: StreamWriter):  # [!code highlight]
          writer({"custom_key": "Streaming custom data while generating a joke"})
          return {"joke": f"This is a joke about {state['topic']}"}

    graph = (
          StateGraph(State)
          .add_node(generate_joke)
          .add_edge(START, "generate_joke")
          .compile()
    )

    # Đặt stream_mode="custom" để nhận dữ liệu tuỳ chỉnh trong stream  # [!code highlight]
    async for chunk in graph.astream(
          {"topic": "ice cream"},
          stream_mode="custom",
          version="v2",
    ):
          if chunk["type"] == "custom":
              print(chunk["data"])
    ```

---

!!! info "Kết nối tài liệu này"
    [Kết nối tài liệu này](https://docs.langchain.com/use-these-docs) với Claude, VSCode, và nhiều công cụ khác qua MCP để có câu trả lời theo thời gian thực.

!!! info "Đóng góp"
    [Chỉnh sửa trang này trên GitHub](https://github.com/langchain-ai/docs/edit/main/src/oss/langgraph/streaming.mdx) hoặc [báo lỗi](https://github.com/langchain-ai/docs/issues/new/choose).
