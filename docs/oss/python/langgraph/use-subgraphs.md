# Subgraph

Hướng dẫn này giải thích cơ chế dùng subgraph. Subgraph là một [graph](./graph-api.md#graphs) được dùng làm một [node](./graph-api.md#nodes) trong một graph khác.

Subgraph hữu ích cho:

* Xây dựng [hệ thống multi-agent](../langchain/multi-agent/index.md)
* Tái sử dụng một tập hợp node trong nhiều graph
* Phân chia phát triển: khi bạn muốn các team khác nhau làm việc độc lập trên các phần khác nhau của graph, bạn có thể định nghĩa mỗi phần như một subgraph, và miễn là interface của subgraph (schema đầu vào và đầu ra) được tuân thủ, graph cha có thể được xây dựng mà không cần biết chi tiết bên trong subgraph

## Cài đặt

=== "pip"

    ```bash
    pip install -U langgraph
    ```

=== "uv"

    ```bash
    uv add langgraph
    ```

!!! tip "Thiết lập LangSmith cho phát triển LangGraph"
    Đăng ký [LangSmith](https://smith.langchain.com?utm_source=docs&utm_medium=cta&utm_campaign=langsmith-signup&utm_content=oss-langgraph-use-subgraphs) để nhanh chóng phát hiện vấn đề và cải thiện hiệu suất các dự án LangGraph của bạn. LangSmith cho phép bạn dùng dữ liệu trace để debug, test, và giám sát ứng dụng LLM xây dựng bằng LangGraph, đọc thêm về [cách bắt đầu với LangSmith](https://docs.smith.langchain.com).

## Định nghĩa giao tiếp giữa subgraph

Khi thêm subgraph, bạn cần định nghĩa cách graph cha và subgraph giao tiếp với nhau:

| Pattern | Khi nào dùng | State schema |
| --- | --- | --- |
| [Gọi subgraph bên trong một node](#call-a-subgraph-inside-a-node) | Graph cha và subgraph có **state schema khác nhau** (không có key chung), hoặc bạn cần biến đổi state giữa chúng | Bạn viết một hàm wrapper ánh xạ state của cha thành input của subgraph, và output của subgraph trở lại thành state của cha |
| [Thêm subgraph như một node](#add-a-subgraph-as-a-node) | Graph cha và subgraph **chia sẻ state key**, subgraph đọc và ghi vào cùng channel với cha | Bạn truyền trực tiếp subgraph đã compile vào `add_node`, không cần hàm wrapper |

### Gọi subgraph bên trong một node

Khi graph cha và subgraph có **state schema khác nhau** (không có key chung), hãy invoke subgraph bên trong một hàm node. Đây là cách làm phổ biến khi bạn muốn giữ lịch sử tin nhắn riêng tư cho mỗi agent trong một hệ thống [multi-agent](../langchain/multi-agent/index.md).

Hàm node biến đổi state của cha thành state của subgraph trước khi invoke subgraph, và biến đổi kết quả trở lại thành state của cha trước khi return.

```python
from typing_extensions import TypedDict
from langgraph.graph.state import StateGraph, START

class SubgraphState(TypedDict):
    bar: str

# Subgraph

def subgraph_node_1(state: SubgraphState):
    return {"bar": "hi! " + state["bar"]}

subgraph_builder = StateGraph(SubgraphState)
subgraph_builder.add_node(subgraph_node_1)
subgraph_builder.add_edge(START, "subgraph_node_1")
subgraph = subgraph_builder.compile()

# Parent graph

class State(TypedDict):
    foo: str

def call_subgraph(state: State):
    # Transform the state to the subgraph state
    subgraph_output = subgraph.invoke({"bar": state["foo"]})  # [!code highlight]
    # Transform response back to the parent state
    return {"foo": subgraph_output["bar"]}

builder = StateGraph(State)
builder.add_node("node_1", call_subgraph)
builder.add_edge(START, "node_1")
graph = builder.compile()
```

??? note "Ví dụ đầy đủ: state schema khác nhau"
    ```python
    from typing_extensions import TypedDict
    from langgraph.graph.state import StateGraph, START

    # Define subgraph
    class SubgraphState(TypedDict):
        # note that none of these keys are shared with the parent graph state
        bar: str
        baz: str

    def subgraph_node_1(state: SubgraphState):
        return {"baz": "baz"}

    def subgraph_node_2(state: SubgraphState):
        return {"bar": state["bar"] + state["baz"]}

    subgraph_builder = StateGraph(SubgraphState)
    subgraph_builder.add_node(subgraph_node_1)
    subgraph_builder.add_node(subgraph_node_2)
    subgraph_builder.add_edge(START, "subgraph_node_1")
    subgraph_builder.add_edge("subgraph_node_1", "subgraph_node_2")
    subgraph = subgraph_builder.compile()

    # Define parent graph
    class ParentState(TypedDict):
        foo: str

    def node_1(state: ParentState):
        return {"foo": "hi! " + state["foo"]}

    def node_2(state: ParentState):
        # Transform the state to the subgraph state
        response = subgraph.invoke({"bar": state["foo"]})
        # Transform response back to the parent state
        return {"foo": response["bar"]}


    builder = StateGraph(ParentState)
    builder.add_node("node_1", node_1)
    builder.add_node("node_2", node_2)
    builder.add_edge(START, "node_1")
    builder.add_edge("node_1", "node_2")
    graph = builder.compile()

    stream = graph.stream_events({"foo": "foo"}, version="v3")
    for event in stream:
        if event["method"] == "updates":
            print(event["params"]["namespace"], event["params"]["data"])
    ```

    ```
    [] {'node_1': {'foo': 'hi! foo'}}
    ['node_2:577b710b-64ae-31fb-9455-6a4d4cc2b0b9'] {'subgraph_node_1': {'baz': 'baz'}}
    ['node_2:577b710b-64ae-31fb-9455-6a4d4cc2b0b9'] {'subgraph_node_2': {'bar': 'hi! foobaz'}}
    [] {'node_2': {'foo': 'hi! foobaz'}}
    ```

??? note "Ví dụ đầy đủ: state schema khác nhau (hai cấp subgraph)"
    Đây là ví dụ với hai cấp subgraph: cha -> con -> cháu.

    ```python
    # Grandchild graph
    from typing_extensions import TypedDict
    from langgraph.graph.state import StateGraph, START, END

    class GrandChildState(TypedDict):
        my_grandchild_key: str

    def grandchild_1(state: GrandChildState) -> GrandChildState:
        # NOTE: child or parent keys will not be accessible here
        return {"my_grandchild_key": state["my_grandchild_key"] + ", how are you"}


    grandchild = StateGraph(GrandChildState)
    grandchild.add_node("grandchild_1", grandchild_1)

    grandchild.add_edge(START, "grandchild_1")
    grandchild.add_edge("grandchild_1", END)

    grandchild_graph = grandchild.compile()

    # Child graph
    class ChildState(TypedDict):
        my_child_key: str

    def call_grandchild_graph(state: ChildState) -> ChildState:
        # NOTE: parent or grandchild keys won't be accessible here
        grandchild_graph_input = {"my_grandchild_key": state["my_child_key"]}
        grandchild_graph_output = grandchild_graph.invoke(grandchild_graph_input)
        return {"my_child_key": grandchild_graph_output["my_grandchild_key"] + " today?"}

    child = StateGraph(ChildState)
    # We're passing a function here instead of just compiled graph (`grandchild_graph`)
    child.add_node("child_1", call_grandchild_graph)
    child.add_edge(START, "child_1")
    child.add_edge("child_1", END)
    child_graph = child.compile()

    # Parent graph
    class ParentState(TypedDict):
        my_key: str

    def parent_1(state: ParentState) -> ParentState:
        # NOTE: child or grandchild keys won't be accessible here
        return {"my_key": "hi " + state["my_key"]}

    def parent_2(state: ParentState) -> ParentState:
        return {"my_key": state["my_key"] + " bye!"}

    def call_child_graph(state: ParentState) -> ParentState:
        child_graph_input = {"my_child_key": state["my_key"]}
        child_graph_output = child_graph.invoke(child_graph_input)
        return {"my_key": child_graph_output["my_child_key"]}

    parent = StateGraph(ParentState)
    parent.add_node("parent_1", parent_1)
    # We're passing a function here instead of just a compiled graph (`child_graph`)
    parent.add_node("child", call_child_graph)
    parent.add_node("parent_2", parent_2)

    parent.add_edge(START, "parent_1")
    parent.add_edge("parent_1", "child")
    parent.add_edge("child", "parent_2")
    parent.add_edge("parent_2", END)

    parent_graph = parent.compile()

    stream = parent_graph.stream_events({"my_key": "Bob"}, version="v3")
    for event in stream:
        if event["method"] == "updates":
            print(event["params"]["namespace"], event["params"]["data"])
    ```

    ```
    [] {'parent_1': {'my_key': 'hi Bob'}}
    ['child:2e26e9ce-602f-862c-aa66-1ea5a4655e3b', 'child_1:781bb3b1-3971-84ce-810b-acf819a03f9c'] {'grandchild_1': {'my_grandchild_key': 'hi Bob, how are you'}}
    ['child:2e26e9ce-602f-862c-aa66-1ea5a4655e3b'] {'child_1': {'my_child_key': 'hi Bob, how are you today?'}}
    [] {'child': {'my_key': 'hi Bob, how are you today?'}}
    [] {'parent_2': {'my_key': 'hi Bob, how are you today? bye!'}}
    ```

### Thêm subgraph như một node

Khi graph cha và subgraph **chia sẻ state key**, bạn có thể truyền trực tiếp một subgraph đã compile vào `add_node`. Không cần hàm wrapper, subgraph tự động đọc và ghi vào channel state của cha. Ví dụ, trong hệ thống [multi-agent](../langchain/multi-agent/index.md), các agent thường giao tiếp qua một key [messages](./graph-api.md#why-use-messages) chung.

<img src="https://mintcdn.com/langchain-5e9cc07a/ybiAaBfoBvFquMDz/oss/images/subgraph.png?fit=max&auto=format&n=ybiAaBfoBvFquMDz&q=85&s=c280df5c968cd4237b0b5d03823d8946" alt="SQL agent graph" style="height:450px" width="1177" height="818" data-path="oss/images/subgraph.png" />

Nếu subgraph của bạn chia sẻ state key với graph cha, bạn có thể làm theo các bước sau để thêm nó vào graph:

1. Định nghĩa workflow của subgraph (`subgraph_builder` trong ví dụ dưới đây) và compile nó
2. Truyền subgraph đã compile vào phương thức [`add_node`](https://reference.langchain.com/python/langgraph/graph/state/StateGraph/add_node) khi định nghĩa workflow của graph cha

```python
from typing_extensions import TypedDict
from langgraph.graph.state import StateGraph, START

class State(TypedDict):
    foo: str

# Subgraph

def subgraph_node_1(state: State):
    return {"foo": "hi! " + state["foo"]}

subgraph_builder = StateGraph(State)
subgraph_builder.add_node(subgraph_node_1)
subgraph_builder.add_edge(START, "subgraph_node_1")
subgraph = subgraph_builder.compile()

# Parent graph

builder = StateGraph(State)
builder.add_node("node_1", subgraph)  # [!code highlight]
builder.add_edge(START, "node_1")
graph = builder.compile()
```

??? note "Ví dụ đầy đủ: state schema chia sẻ"
    ```python
    from typing_extensions import TypedDict
    from langgraph.graph.state import StateGraph, START

    # Define subgraph
    class SubgraphState(TypedDict):
        foo: str  # shared with parent graph state
        bar: str  # private to SubgraphState

    def subgraph_node_1(state: SubgraphState):
        return {"bar": "bar"}

    def subgraph_node_2(state: SubgraphState):
        # note that this node is using a state key ('bar') that is only available in the subgraph
        # and is sending update on the shared state key ('foo')
        return {"foo": state["foo"] + state["bar"]}

    subgraph_builder = StateGraph(SubgraphState)
    subgraph_builder.add_node(subgraph_node_1)
    subgraph_builder.add_node(subgraph_node_2)
    subgraph_builder.add_edge(START, "subgraph_node_1")
    subgraph_builder.add_edge("subgraph_node_1", "subgraph_node_2")
    subgraph = subgraph_builder.compile()

    # Define parent graph
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

    stream = graph.stream_events({"foo": "foo"}, version="v3")
    for event in stream:
        if event["method"] == "updates" and not event["params"]["namespace"]:
            print(event["params"]["data"])
    ```

    ```
    {'node_1': {'foo': 'hi! foo'}}
    {'node_2': {'foo': 'hi! foobar'}}
    ```

## Tính bền vững (persistence) của subgraph

Khi dùng một subgraph, bạn cần quyết định điều gì xảy ra với dữ liệu nội bộ của nó giữa các lần gọi. Hãy xét một bot hỗ trợ khách hàng, uỷ quyền cho các subagent chuyên biệt: liệu subagent "chuyên gia thanh toán" có nên nhớ các câu hỏi trước đó của khách hàng, hay bắt đầu lại từ đầu mỗi lần được gọi?

Tham số `checkpointer` trên `.compile()` kiểm soát tính bền vững của subgraph:

| Chế độ | `checkpointer=` | Hành vi |
| --- | --- | --- |
| [Theo lần gọi (per-invocation)](#per-invocation-default) | `None` (mặc định) | Mỗi lần gọi bắt đầu lại từ đầu và kế thừa checkpointer của cha để hỗ trợ [interrupt](./interrupts.md) và [thực thi bền vững](./persistence.md) trong một lần gọi duy nhất. |
| [Theo thread (per-thread)](#per-thread) | `True` | State tích luỹ qua các lần gọi trên cùng một thread. Mỗi lần gọi tiếp tục từ nơi lần trước dừng lại. |
| [Không trạng thái (stateless)](#stateless) | `False` | Không checkpoint gì cả, chạy như một lời gọi hàm thông thường. Không có interrupt hay thực thi bền vững. |

Theo lần gọi (per-invocation) là lựa chọn phù hợp cho hầu hết ứng dụng, bao gồm cả hệ thống [multi-agent](../langchain/multi-agent/index.md) nơi các subagent xử lý các yêu cầu độc lập. Dùng theo thread (per-thread) khi một subagent cần bộ nhớ hội thoại đa lượt (multi-turn), ví dụ một trợ lý nghiên cứu xây dựng ngữ cảnh qua nhiều lượt trao đổi.

!!! note ""
    Graph cha phải được compile với một checkpointer để các tính năng bền vững của subgraph (interrupt, kiểm tra state, bộ nhớ theo thread) hoạt động. Xem [persistence](./persistence.md).

!!! info ""
    Các ví dụ bên dưới dùng [`create_agent`](https://reference.langchain.com/python/langchain/agents/factory/create_agent) của LangChain, một cách phổ biến để xây dựng agent. `create_agent` tạo ra một [graph LangGraph](./graph-api.md) bên dưới, nên mọi khái niệm về tính bền vững của subgraph đều áp dụng trực tiếp. Nếu bạn xây dựng bằng `StateGraph` thuần của LangGraph, các pattern và tuỳ chọn cấu hình tương tự vẫn áp dụng, xem [Graph API](./graph-api.md) để biết chi tiết.

### Có trạng thái (stateful)

Subgraph có trạng thái kế thừa checkpointer của graph cha, cho phép [interrupt](./interrupts.md), [persistence](./persistence.md), và kiểm tra state. Hai chế độ có trạng thái khác nhau ở thời gian state được giữ lại.

#### Theo lần gọi (mặc định)

!!! tip ""
    Đây là chế độ được khuyến nghị cho hầu hết ứng dụng, bao gồm cả hệ thống [multi-agent](../langchain/multi-agent/index.md) nơi subagent được invoke như tool. Nó hỗ trợ [interrupt](./interrupts.md), [persistence](./persistence.md), và các lần gọi song song trong khi vẫn giữ mỗi lần gọi độc lập.

Dùng persistence theo lần gọi khi mỗi lần gọi subgraph độc lập và subagent không cần nhớ gì từ các lần gọi trước. Đây là pattern phổ biến nhất, đặc biệt với hệ thống [multi-agent](../langchain/multi-agent/index.md) nơi subagent xử lý các yêu cầu một lần như "tra cứu đơn hàng của khách hàng này" hoặc "tóm tắt tài liệu này".

Bỏ qua `checkpointer` hoặc đặt nó là `None`. Mỗi lần gọi bắt đầu lại từ đầu, nhưng trong một lần gọi, subgraph kế thừa checkpointer của cha và có thể dùng `interrupt()` để tạm dừng và tiếp tục.

Các ví dụ sau dùng hai subagent (chuyên gia trái cây, chuyên gia rau củ) được bọc thành tool cho một agent bên ngoài:

```python
from langchain.agents import create_agent
from langchain.tools import tool
from langgraph.checkpoint.memory import MemorySaver
from langgraph.types import Command, interrupt

@tool
def fruit_info(fruit_name: str) -> str:
    """Look up fruit info."""
    return f"Info about {fruit_name}"

@tool
def veggie_info(veggie_name: str) -> str:
    """Look up veggie info."""
    return f"Info about {veggie_name}"

# Subagents - no checkpointer setting (inherits parent)
fruit_agent = create_agent(
    model="gpt-5.4-mini",
    tools=[fruit_info],
    prompt="You are a fruit expert. Use the fruit_info tool. Respond in one sentence.",
)

veggie_agent = create_agent(
    model="gpt-5.4-mini",
    tools=[veggie_info],
    prompt="You are a veggie expert. Use the veggie_info tool. Respond in one sentence.",
)

# Wrap subagents as tools for the outer agent
@tool
def ask_fruit_expert(question: str) -> str:
    """Ask the fruit expert. Use for ALL fruit questions."""
    response = fruit_agent.invoke(
        {"messages": [{"role": "user", "content": question}]},
    )
    return response["messages"][-1].content

@tool
def ask_veggie_expert(question: str) -> str:
    """Ask the veggie expert. Use for ALL veggie questions."""
    response = veggie_agent.invoke(
        {"messages": [{"role": "user", "content": question}]},
    )
    return response["messages"][-1].content

# Outer agent with checkpointer
agent = create_agent(
    model="gpt-5.4-mini",
    tools=[ask_fruit_expert, ask_veggie_expert],
    prompt=(
        "You have two experts: ask_fruit_expert and ask_veggie_expert. "
        "ALWAYS delegate questions to the appropriate expert."
    ),
    checkpointer=MemorySaver(),
)
```

=== "Interrupt"

    Mỗi lần gọi có thể dùng `interrupt()` để tạm dừng và tiếp tục. Thêm `interrupt()` vào một hàm tool để yêu cầu người dùng phê duyệt trước khi tiếp tục:

    ```python
    @tool
    def fruit_info(fruit_name: str) -> str:
        """Look up fruit info."""
        interrupt("continue?")  # [!code highlight]
        return f"Info about {fruit_name}"
    ```

    ```python
    from langgraph.types import Command

    config = {"configurable": {"thread_id": "1"}}

    # Stream events - the subagent's tool calls interrupt()
    stream = agent.stream_events(
        {"messages": [{"role": "user", "content": "Tell me about apples"}]},
        config=config,
        version="v3",
    )
    output = stream.output  # drive the stream to completion
    # stream.interrupts contains pending interrupts (and stream.interrupted is True)

    # Resume - approve the interrupt
    resumed = agent.stream_events(Command(resume=True), config=config, version="v3")
    final = resumed.output
    ```

=== "Đa lượt"

    Mỗi lần gọi bắt đầu với state subagent hoàn toàn mới. Subagent không nhớ các lần gọi trước:

    ```python
    config = {"configurable": {"thread_id": "1"}}

    # First call
    response = agent.invoke(
        {"messages": [{"role": "user", "content": "Tell me about apples"}]},
        config=config,
    )
    # Subagent message count: 4

    # Second call - subagent starts fresh, no memory of apples
    response = agent.invoke(
        {"messages": [{"role": "user", "content": "Now tell me about bananas"}]},
        config=config,
    )
    # Subagent message count: 4 (still fresh!)
    ```

=== "Nhiều lần gọi subgraph"

    Nhiều lần gọi tới cùng một subgraph hoạt động không xung đột, vì mỗi lần gọi có namespace checkpoint riêng:

    ```python
    config = {"configurable": {"thread_id": "1"}}

    # LLM calls ask_fruit_expert for both apples and bananas
    response = agent.invoke(
        {"messages": [{"role": "user", "content": "Tell me about apples and bananas"}]},
        config=config,
    )
    # Subagent message count: 4 (apples - fresh)
    # Subagent message count: 4 (bananas - fresh)
    ```

#### Theo thread (per-thread)

Dùng persistence theo thread khi một subagent cần nhớ các tương tác trước đó. Ví dụ, một trợ lý nghiên cứu xây dựng ngữ cảnh qua nhiều lượt trao đổi, hoặc một trợ lý coding theo dõi những file nào nó đã chỉnh sửa. Lịch sử hội thoại và dữ liệu của subagent tích luỹ qua các lần gọi trên cùng một thread. Mỗi lần gọi tiếp tục từ nơi lần trước dừng lại.

Compile với `checkpointer=True` để bật hành vi này.

!!! warning ""
    Subgraph theo thread không hỗ trợ gọi tool song song. Khi một LLM có quyền truy cập một subagent theo thread như một tool, nó có thể cố gọi tool đó nhiều lần song song (ví dụ, hỏi chuyên gia trái cây về táo và chuối cùng lúc). Điều này gây xung đột checkpoint vì cả hai lần gọi đều ghi vào cùng một namespace.

    Các ví dụ bên dưới dùng `ToolCallLimitMiddleware` của LangChain để ngăn điều này. Nếu bạn xây dựng bằng `StateGraph` thuần của LangGraph, bạn cần tự ngăn việc gọi tool song song, ví dụ bằng cách cấu hình model của bạn để tắt gọi tool song song, hoặc thêm logic đảm bảo cùng một subgraph không được invoke nhiều lần song song.

Các ví dụ sau dùng một subagent chuyên gia trái cây được compile với `checkpointer=True`:

```python
from langchain.agents import create_agent
from langchain.agents.middleware import ToolCallLimitMiddleware
from langchain.tools import tool
from langgraph.checkpoint.memory import MemorySaver
from langgraph.types import Command, interrupt

@tool
def fruit_info(fruit_name: str) -> str:
    """Look up fruit info."""
    return f"Info about {fruit_name}"

# Subagent with checkpointer=True for persistent state
fruit_agent = create_agent(
    model="gpt-5.4-mini",
    tools=[fruit_info],
    prompt="You are a fruit expert. Use the fruit_info tool. Respond in one sentence.",
    checkpointer=True,  # [!code highlight]
)

# Wrap subagent as a tool for the outer agent
@tool
def ask_fruit_expert(question: str) -> str:
    """Ask the fruit expert. Use for ALL fruit questions."""
    response = fruit_agent.invoke(
        {"messages": [{"role": "user", "content": question}]},
    )
    return response["messages"][-1].content

# Outer agent with checkpointer
# Use ToolCallLimitMiddleware to prevent parallel calls to per-thread subagents,
# which would cause checkpoint conflicts.
agent = create_agent(
    model="gpt-5.4-mini",
    tools=[ask_fruit_expert],
    prompt="You have a fruit expert. ALWAYS delegate fruit questions to ask_fruit_expert.",
    middleware=[  # [!code highlight]
        ToolCallLimitMiddleware(tool_name="ask_fruit_expert", run_limit=1),  # [!code highlight]
    ],  # [!code highlight]
    checkpointer=MemorySaver(),
)
```

=== "Interrupt"

    Subagent theo thread hỗ trợ `interrupt()` giống như theo lần gọi. Thêm `interrupt()` vào một hàm tool để yêu cầu người dùng phê duyệt:

    ```python
    @tool
    def fruit_info(fruit_name: str) -> str:
        """Look up fruit info."""
        interrupt("continue?")  # [!code highlight]
        return f"Info about {fruit_name}"
    ```

    ```python
    from langgraph.types import Command

    config = {"configurable": {"thread_id": "1"}}

    # Stream events - the subagent's tool calls interrupt()
    stream = agent.stream_events(
        {"messages": [{"role": "user", "content": "Tell me about apples"}]},
        config=config,
        version="v3",
    )
    output = stream.output  # drive the stream to completion
    # stream.interrupts contains pending interrupts (and stream.interrupted is True)

    # Resume - approve the interrupt
    resumed = agent.stream_events(Command(resume=True), config=config, version="v3")
    final = resumed.output
    ```

=== "Đa lượt"

    State tích luỹ qua các lần gọi, subagent nhớ các hội thoại trước:

    ```python
    config = {"configurable": {"thread_id": "1"}}

    # First call
    response = agent.invoke(
        {"messages": [{"role": "user", "content": "Tell me about apples"}]},
        config=config,
    )
    # Subagent message count: 4

    # Second call - subagent REMEMBERS apples conversation
    response = agent.invoke(
        {"messages": [{"role": "user", "content": "Now tell me about bananas"}]},
        config=config,
    )
    # Subagent message count: 8 (accumulated!)
    ```

=== "Nhiều lần gọi subgraph"

    Khi bạn có nhiều subgraph theo thread **khác nhau** (ví dụ, một chuyên gia trái cây và một chuyên gia rau củ), mỗi cái cần không gian lưu trữ riêng để checkpoint không ghi đè lên nhau. Đây gọi là **cách ly namespace** (namespace isolation).

    Nếu bạn [gọi subgraph bên trong một node](#call-a-subgraph-inside-a-node), LangGraph gán namespace dựa trên thứ tự gọi (lần gọi thứ nhất, thứ hai, v.v.). Điều này có nghĩa việc sắp xếp lại thứ tự gọi có thể làm lẫn lộn subgraph nào tải state nào. Để tránh điều này, hãy bọc mỗi subagent trong `StateGraph` riêng với tên node duy nhất, việc này cho mỗi subgraph một namespace ổn định, duy nhất:

    ```python
    from langgraph.graph import MessagesState, StateGraph

    def create_sub_agent(model, *, name, **kwargs):
        """Wrap an agent with a unique node name for namespace isolation."""
        agent = create_agent(model=model, name=name, **kwargs)
        return (
            StateGraph(MessagesState)
            .add_node(name, agent)  # unique name → stable namespace  # [!code highlight]
            .add_edge("__start__", name)
            .compile()
        )

    fruit_agent = create_sub_agent(
        "gpt-5.4-mini", name="fruit_agent",
        tools=[fruit_info], prompt="...", checkpointer=True,
    )
    veggie_agent = create_sub_agent(
        "gpt-5.4-mini", name="veggie_agent",
        tools=[veggie_info], prompt="...", checkpointer=True,
    )

    config = {"configurable": {"thread_id": "1"}}

    # First call - LLM calls both fruit and veggie experts
    response = agent.invoke(
        {"messages": [{"role": "user", "content": "Tell me about cherries and broccoli"}]},
        config=config,
    )
    # Fruit subagent message count: 4
    # Veggie subagent message count: 4

    # Second call - both agents accumulate independently
    response = agent.invoke(
        {"messages": [{"role": "user", "content": "Now tell me about oranges and carrots"}]},
        config=config,
    )
    # Fruit subagent message count: 8 (remembers cherries!)
    # Veggie subagent message count: 8 (remembers broccoli!)
    ```

    Subgraph [được thêm như node](#add-a-subgraph-as-a-node) đã tự động có namespace dựa trên tên, nên không cần wrapper này.

### Không trạng thái (stateless)

Dùng khi bạn muốn chạy một subagent như một lời gọi hàm thông thường, không có overhead checkpoint. Subgraph không thể tạm dừng/tiếp tục và không được hưởng lợi từ [thực thi bền vững](./persistence.md). Compile với `checkpointer=False`.

!!! warning ""
    Không có checkpoint, subgraph không có thực thi bền vững. Nếu tiến trình crash giữa chừng, subgraph không thể khôi phục và phải chạy lại từ đầu.

```python
subgraph_builder = StateGraph(...)
subgraph = subgraph_builder.compile(checkpointer=False)  # [!code highlight]
```

### Tham chiếu checkpointer

Kiểm soát tính bền vững của subgraph bằng tham số `checkpointer` trên `.compile()`:

```python
subgraph = builder.compile(checkpointer=False)  # or True / None
```

| Tính năng | Theo lần gọi (mặc định) | Theo thread | Không trạng thái |
| --- | --- | --- | --- |
| `checkpointer=` | `None` | `True` | `False` |
| Interrupt (HITL) | ✅ | ✅ | ❌ |
| Bộ nhớ đa lượt | ❌ | ✅ | ❌ |
| Nhiều lần gọi (subgraph khác nhau) | ✅ | ⚠️ (các lần gọi tới nhiều subgraph theo thread khác nhau trong cùng một node có thể gây xung đột namespace; có cách khắc phục) | ✅ |
| Nhiều lần gọi (cùng subgraph) | ✅ | ❌ | ✅ |
| Kiểm tra state | ⚠️ (kiểm tra state với persistence theo lần gọi chỉ khả dụng cho lần gọi hiện tại, trong khi bị interrupt; mỗi lần gọi bắt đầu mới nên không có state tích luỹ để kiểm tra sau khi lần gọi hoàn tất) | ✅ | ❌ |

* **Interrupt (HITL)**: Subgraph có thể dùng [interrupt()](./interrupts.md) để tạm dừng thực thi và chờ input người dùng, sau đó tiếp tục từ nơi đã dừng.
* **Bộ nhớ đa lượt**: Subgraph giữ lại state qua nhiều lần invoke trong cùng một [thread](./checkpointers.md#threads). Mỗi lần gọi tiếp tục từ nơi lần trước dừng thay vì bắt đầu mới.
* **Nhiều lần gọi (subgraph khác nhau)**: Nhiều instance subgraph khác nhau có thể được invoke trong một node duy nhất mà không gây xung đột namespace checkpoint.
* **Nhiều lần gọi (cùng subgraph)**: Cùng một instance subgraph có thể được invoke nhiều lần trong một node duy nhất. Với persistence có trạng thái, các lần gọi này ghi vào cùng namespace checkpoint và xung đột, hãy dùng persistence theo lần gọi thay thế.
* **Kiểm tra state**: State của subgraph khả dụng qua `get_state(config, subgraphs=True)` để debug và giám sát.

## Xem state của subgraph

Khi bạn bật [persistence](./persistence.md), bạn có thể kiểm tra state của subgraph bằng tuỳ chọn subgraphs. Với checkpoint [không trạng thái](#stateless) (`checkpointer=False`), không có checkpoint subgraph nào được lưu, nên state của subgraph không khả dụng.

!!! note ""
    Việc xem state của subgraph yêu cầu LangGraph có thể **phát hiện tĩnh** (statically discover) subgraph, tức là nó được [thêm như một node](#add-a-subgraph-as-a-node) hoặc [gọi bên trong một node](#call-a-subgraph-inside-a-node). Việc này không hoạt động khi subgraph được gọi bên trong một hàm [tool](../langchain/tools.md) hoặc gián tiếp theo cách khác (ví dụ, pattern [subagent](../langchain/multi-agent/subagents.md)). Interrupt vẫn lan truyền lên graph cấp cao nhất bất kể mức độ lồng nhau.

=== "Theo lần gọi"

    Trả về state subgraph cho **lần gọi hiện tại**. Mỗi lần gọi bắt đầu mới.

    ```python
    from langgraph.graph import START, StateGraph
    from langgraph.checkpoint.memory import MemorySaver
    from langgraph.types import interrupt, Command
    from typing_extensions import TypedDict

    class State(TypedDict):
        foo: str

    # Subgraph
    def subgraph_node_1(state: State):
        value = interrupt("Provide value:")
        return {"foo": state["foo"] + value}

    subgraph_builder = StateGraph(State)
    subgraph_builder.add_node(subgraph_node_1)
    subgraph_builder.add_edge(START, "subgraph_node_1")
    subgraph = subgraph_builder.compile()  # inherits parent checkpointer

    # Parent graph
    builder = StateGraph(State)
    builder.add_node("node_1", subgraph)
    builder.add_edge(START, "node_1")

    checkpointer = MemorySaver()
    graph = builder.compile(checkpointer=checkpointer)

    config = {"configurable": {"thread_id": "1"}}

    graph.invoke({"foo": ""}, config)

    # View subgraph state for the current invocation
    subgraph_state = graph.get_state(config, subgraphs=True).tasks[0].state  # [!code highlight]

    # Resume the subgraph
    graph.invoke(Command(resume="bar"), config)
    ```

=== "Theo thread"

    Trả về state **tích luỹ** của subgraph qua tất cả lần gọi trên thread này.

    ```python
    from langgraph.graph import START, StateGraph, MessagesState
    from langgraph.checkpoint.memory import MemorySaver

    # Subgraph with its own persistent state
    subgraph_builder = StateGraph(MessagesState)
    # ... add nodes and edges
    subgraph = subgraph_builder.compile(checkpointer=True)  # [!code highlight]

    # Parent graph
    builder = StateGraph(MessagesState)
    builder.add_node("agent", subgraph)
    builder.add_edge(START, "agent")

    checkpointer = MemorySaver()
    graph = builder.compile(checkpointer=checkpointer)

    config = {"configurable": {"thread_id": "1"}}

    graph.invoke({"messages": [{"role": "user", "content": "hi"}]}, config)
    graph.invoke({"messages": [{"role": "user", "content": "what did I say?"}]}, config)

    # View accumulated subgraph state (includes messages from both invocations)
    subgraph_state = graph.get_state(config, subgraphs=True).tasks[0].state  # [!code highlight]
    ```

## Stream output từ subgraph

Để quan sát việc thực thi graph lồng nhau, ta khuyến nghị [event streaming](./event-streaming.md): phép chiếu `stream.subgraphs` phát hiện từng lần chạy lồng nhau và phơi bày `path`, `messages`, và `values` của nó mà không cần parse chuỗi namespace.

```python
stream = graph.stream_events({"foo": "foo"}, version="v3")  # [!code highlight]

for subgraph in stream.subgraphs:
    print(subgraph.graph_name, subgraph.path)

    for snapshot in subgraph.values:
        print(subgraph.path, snapshot)
```

Nếu bạn cần các event protocol thô, hãy lặp trực tiếp qua stream và lọc theo `event["method"]` và `event["params"]["namespace"]`:

```python
stream = graph.stream_events({"foo": "foo"}, version="v3")
for event in stream:
    if event["method"] == "updates":
        print(event["params"]["namespace"], event["params"]["data"])
```

??? note "Stream từ subgraph"
    ```python
    from typing_extensions import TypedDict
    from langgraph.graph.state import StateGraph, START

    # Define subgraph
    class SubgraphState(TypedDict):
        foo: str
        bar: str

    def subgraph_node_1(state: SubgraphState):
        return {"bar": "bar"}

    def subgraph_node_2(state: SubgraphState):
        # note that this node is using a state key ('bar') that is only available in the subgraph
        # and is sending update on the shared state key ('foo')
        return {"foo": state["foo"] + state["bar"]}

    subgraph_builder = StateGraph(SubgraphState)
    subgraph_builder.add_node(subgraph_node_1)
    subgraph_builder.add_node(subgraph_node_2)
    subgraph_builder.add_edge(START, "subgraph_node_1")
    subgraph_builder.add_edge("subgraph_node_1", "subgraph_node_2")
    subgraph = subgraph_builder.compile()

    # Define parent graph
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

    stream = graph.stream_events({"foo": "foo"}, version="v3")  # [!code highlight]
    for event in stream:
        if event["method"] == "updates":
            print(event["params"]["namespace"], event["params"]["data"])
    ```

    ```
    [] {'node_1': {'foo': 'hi! foo'}}
    ['node_2:e58e5673-a661-ebb0-70d4-e298a7fc28b7'] {'subgraph_node_1': {'bar': 'bar'}}
    ['node_2:e58e5673-a661-ebb0-70d4-e298a7fc28b7'] {'subgraph_node_2': {'foo': 'hi! foobar'}}
    [] {'node_2': {'foo': 'hi! foobar'}}
    ```

---

!!! info "Kết nối tài liệu này"
    [Kết nối tài liệu này](https://docs.langchain.com/use-these-docs) với Claude, VSCode, và nhiều công cụ khác qua MCP để có câu trả lời theo thời gian thực.

!!! info "Đóng góp"
    [Chỉnh sửa trang này trên GitHub](https://github.com/langchain-ai/docs/edit/main/src/oss/langgraph/use-subgraphs.mdx) hoặc [báo lỗi](https://github.com/langchain-ai/docs/issues/new/choose).
