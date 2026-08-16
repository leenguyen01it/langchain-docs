# Dùng time-travel

> Phát lại (replay) các lần thực thi trước và fork để khám phá các đường đi thay thế trong LangGraph

## Tổng quan

LangGraph hỗ trợ time travel thông qua [checkpoint](./checkpointers.md#checkpoints):

* **[Replay](#replay)**: Thử lại từ một checkpoint trước đó.
* **[Fork](#fork)**: Phân nhánh từ một checkpoint trước đó với state đã sửa đổi để khám phá một đường đi thay thế.

Cả hai cách đều hoạt động bằng cách resume từ một checkpoint trước đó. Các node trước checkpoint không được thực thi lại (kết quả đã được lưu sẵn). Các node sau checkpoint sẽ thực thi lại, bao gồm cả các lệnh gọi LLM, request API, và [interrupt](./interrupts.md) (có thể tạo ra kết quả khác).

## Replay

Gọi graph với config của một checkpoint trước đó để replay từ điểm đó.

!!! warning ""
    Replay thực thi lại các node, nó không chỉ đọc từ cache. Các lệnh gọi LLM, request API, và [interrupt](./interrupts.md) sẽ chạy lại và có thể trả về kết quả khác. Replay từ checkpoint cuối cùng (không có node `next`) sẽ không làm gì cả (no-op).

<img src="https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/re_play.png?fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=d7b34b85c106e55d181ae1f4afb50251" alt="Replay" width="2276" height="986" data-path="oss/images/re_play.png" />

Dùng [`get_state_history`](https://reference.langchain.com/python/langgraph/graphs/#langgraph.graph.state.CompiledStateGraph.get_state_history) để tìm checkpoint bạn muốn replay từ đó, sau đó gọi [`invoke`](https://reference.langchain.com/python/langgraph/graphs/#langgraph.graph.state.CompiledStateGraph.invoke) với config của checkpoint đó:

```python
from langgraph.graph import StateGraph, START
from langgraph.checkpoint.memory import InMemorySaver
from typing_extensions import TypedDict, NotRequired
from langchain_core.utils.uuid import uuid7

class State(TypedDict):
    topic: NotRequired[str]
    joke: NotRequired[str]


def generate_topic(state: State):
    return {"topic": "socks in the dryer"}


def write_joke(state: State):
    return {"joke": f"Why do {state['topic']} disappear? They elope!"}


checkpointer = InMemorySaver()
graph = (
    StateGraph(State)
    .add_node("generate_topic", generate_topic)
    .add_node("write_joke", write_joke)
    .add_edge(START, "generate_topic")
    .add_edge("generate_topic", "write_joke")
    .compile(checkpointer=checkpointer)
)

# Bước 1: Chạy graph
config = {"configurable": {"thread_id": str(uuid7())}}
result = graph.invoke({}, config)

# Bước 2: Tìm một checkpoint để replay từ đó
history = list(graph.get_state_history(config))
# History theo thứ tự ngược thời gian
for state in history:
    print(f"next={state.next}, checkpoint_id={state.config['configurable']['checkpoint_id']}")

# Bước 3: Replay từ một checkpoint cụ thể
# Tìm checkpoint trước write_joke
before_joke = next(s for s in history if s.next == ("write_joke",))
replay_result = graph.invoke(None, before_joke.config)
# write_joke thực thi lại (chạy lại), generate_topic thì không
```

## Fork

Fork tạo một nhánh mới từ một checkpoint trong quá khứ với state đã sửa đổi. Gọi [`update_state`](https://reference.langchain.com/python/langgraph/graphs/#langgraph.graph.state.CompiledStateGraph.update_state) trên một checkpoint trước đó để tạo fork, sau đó [`invoke`](https://reference.langchain.com/python/langgraph/graphs/#langgraph.graph.state.CompiledStateGraph.invoke) với `None` để tiếp tục thực thi.

<img src="https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/checkpoints_full_story.jpg?fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=a52016b2c44b57bd395d6e1eac47aa36" alt="Fork" width="3705" height="2598" data-path="oss/images/checkpoints_full_story.jpg" />

!!! warning ""
    `update_state` **không** roll back một thread. Nó tạo ra một checkpoint mới phân nhánh từ điểm được chỉ định. Lịch sử thực thi gốc vẫn được giữ nguyên vẹn.

```python
# Tìm checkpoint trước write_joke
history = list(graph.get_state_history(config))
before_joke = next(s for s in history if s.next == ("write_joke",))

# Fork: cập nhật state để đổi chủ đề (topic)
fork_config = graph.update_state(
    before_joke.config,
    values={"topic": "chickens"},
)

# Resume từ fork: write_joke thực thi lại với topic mới
fork_result = graph.invoke(None, fork_config)
print(fork_result["joke"])  # Một câu đùa về gà, không phải tất
```

### Từ một node cụ thể

Khi bạn gọi [`update_state`](https://reference.langchain.com/python/langgraph/graphs/#langgraph.graph.state.CompiledStateGraph.update_state), các giá trị được áp dụng bằng writer của node được chỉ định (bao gồm cả [reducer](./graph-api.md#reducers)). Checkpoint ghi nhận node đó là node đã tạo ra cập nhật, và thực thi resume từ các node kế tiếp của nó.

Theo mặc định, LangGraph tự suy luận `as_node` từ lịch sử phiên bản của checkpoint. Khi fork từ một checkpoint cụ thể, suy luận này gần như luôn đúng.

Chỉ định `as_node` tường minh khi:

* **Các nhánh song song**: Nhiều node cùng cập nhật state trong cùng một bước, và LangGraph không thể xác định node nào chạy sau cùng (`InvalidUpdateError`).
* **Không có lịch sử thực thi**: Thiết lập state trên một thread mới hoàn toàn (thường gặp khi [test](./test.md)).
* **Bỏ qua node**: Đặt `as_node` thành một node sau đó để làm cho graph nghĩ rằng node đó đã chạy rồi.

```python
# graph: generate_topic -> write_joke

# Coi cập nhật này như thể generate_topic đã tạo ra nó.
# Thực thi resume tại write_joke (node kế tiếp của generate_topic).
fork_config = graph.update_state(
    before_joke.config,
    values={"topic": "chickens"},
    as_node="generate_topic",
)
```

## Interrupt

Nếu graph của bạn dùng [`interrupt`](https://reference.langchain.com/python/langgraph/types/interrupt) cho các workflow [human-in-the-loop](./interrupts.md), interrupt luôn được kích hoạt lại trong quá trình time travel. Node chứa interrupt sẽ thực thi lại, và `interrupt()` tạm dừng để chờ một `Command(resume=...)` mới.

```python
from langgraph.types import interrupt, Command

class State(TypedDict):
    value: list[str]

def ask_human(state: State):
    answer = interrupt("What is your name?")
    return {"value": [f"Hello, {answer}!"]}

def final_step(state: State):
    return {"value": ["Done"]}

graph = (
    StateGraph(State)
    .add_node("ask_human", ask_human)
    .add_node("final_step", final_step)
    .add_edge(START, "ask_human")
    .add_edge("ask_human", "final_step")
    .compile(checkpointer=InMemorySaver())
)

config = {"configurable": {"thread_id": "1"}}

# Lần chạy đầu: gặp interrupt
graph.invoke({"value": []}, config)
# Resume với câu trả lời
graph.invoke(Command(resume="Alice"), config)

# Replay từ trước ask_human
history = list(graph.get_state_history(config))
before_ask = [s for s in history if s.next == ("ask_human",)][-1]

replay_result = graph.invoke(None, before_ask.config)
# Tạm dừng tại interrupt: chờ Command(resume=...) mới

# Fork từ trước ask_human
fork_config = graph.update_state(before_ask.config, {"value": ["forked"]})
fork_result = graph.invoke(None, fork_config)
# Tạm dừng tại interrupt: chờ Command(resume=...) mới

# Resume interrupt đã fork với câu trả lời khác
graph.invoke(Command(resume="Bob"), fork_config)
# Kết quả: {"value": ["forked", "Hello, Bob!", "Done"]}
```

### Nhiều interrupt

Nếu graph của bạn thu thập input tại nhiều điểm (ví dụ một form nhiều bước), bạn có thể fork từ giữa các interrupt để thay đổi một câu trả lời sau mà không cần hỏi lại các câu trước đó.

```python
def ask_name(state):
    name = interrupt("What is your name?")
    return {"value": [f"name:{name}"]}

def ask_age(state):
    age = interrupt("How old are you?")
    return {"value": [f"age:{age}"]}

# Graph: ask_name -> ask_age -> final
# Sau khi hoàn thành cả hai interrupt:

# Fork từ GIỮA hai interrupt (sau ask_name, trước ask_age)
history = list(graph.get_state_history(config))
between = [s for s in history if s.next == ("ask_age",)][-1]

fork_config = graph.update_state(between.config, {"value": ["modified"]})
result = graph.invoke(None, fork_config)
# Kết quả của ask_name được giữ nguyên ("name:Alice")
# ask_age tạm dừng tại interrupt: chờ câu trả lời mới
```

## Subgraph

Time travel với [subgraph](./use-subgraphs.md) phụ thuộc vào việc subgraph có checkpointer riêng hay không. Điều này quyết định độ chi tiết (granularity) của các checkpoint bạn có thể time travel từ đó.

=== "Checkpointer kế thừa (mặc định)"

    Theo mặc định, một subgraph kế thừa checkpointer của graph cha. Graph cha coi toàn bộ subgraph như một **super-step duy nhất**, chỉ có một checkpoint ở cấp cha cho toàn bộ quá trình thực thi subgraph. Time travel từ trước subgraph sẽ thực thi lại nó từ đầu.

    Bạn không thể time travel tới một điểm *giữa* các node trong một subgraph mặc định, bạn chỉ có thể time travel từ cấp cha.

    ```python
    # Subgraph không có checkpointer riêng (mặc định)
    subgraph = (
        StateGraph(State)
        .add_node("step_a", step_a)       # Có interrupt()
        .add_node("step_b", step_b)       # Có interrupt()
        .add_edge(START, "step_a")
        .add_edge("step_a", "step_b")
        .compile()  # Không có checkpointer: kế thừa từ cha
    )

    graph = (
        StateGraph(State)
        .add_node("subgraph_node", subgraph)
        .add_edge(START, "subgraph_node")
        .compile(checkpointer=InMemorySaver())
    )

    config = {"configurable": {"thread_id": "1"}}

    # Hoàn thành cả hai interrupt
    graph.invoke({"value": []}, config)            # Gặp interrupt tại step_a
    graph.invoke(Command(resume="Alice"), config)  # Gặp interrupt tại step_b
    graph.invoke(Command(resume="30"), config)     # Hoàn thành

    # Time travel từ trước subgraph
    history = list(graph.get_state_history(config))
    before_sub = [s for s in history if s.next == ("subgraph_node",)][-1]

    fork_config = graph.update_state(before_sub.config, {"value": ["forked"]})
    result = graph.invoke(None, fork_config)
    # Toàn bộ subgraph thực thi lại từ đầu
    # Bạn không thể time travel tới một điểm giữa step_a và step_b
    ```

=== "Checkpointer riêng cho subgraph"

    Đặt `checkpointer=True` trên subgraph để nó có lịch sử checkpoint riêng. Điều này tạo ra checkpoint tại mỗi bước **bên trong** subgraph, cho phép bạn time travel từ một điểm cụ thể bên trong nó, ví dụ giữa hai interrupt.

    Dùng [`get_state`](https://reference.langchain.com/python/langgraph/graphs/#langgraph.graph.state.CompiledStateGraph.get_state) với `subgraphs=True` để truy cập config checkpoint riêng của subgraph, sau đó fork từ đó:

    ```python
    # Subgraph có checkpointer riêng
    subgraph = (
        StateGraph(State)
        .add_node("step_a", step_a)       # Có interrupt()
        .add_node("step_b", step_b)       # Có interrupt()
        .add_edge(START, "step_a")
        .add_edge("step_a", "step_b")
        .compile(checkpointer=True)  # Lịch sử checkpoint riêng
    )

    graph = (
        StateGraph(State)
        .add_node("subgraph_node", subgraph)
        .add_edge(START, "subgraph_node")
        .compile(checkpointer=InMemorySaver())
    )

    config = {"configurable": {"thread_id": "1"}}

    # Chạy tới khi gặp interrupt tại step_a
    graph.invoke({"value": []}, config)

    # Resume step_a -> gặp interrupt tại step_b
    graph.invoke(Command(resume="Alice"), config)

    # Lấy checkpoint riêng của subgraph (giữa step_a và step_b)
    parent_state = graph.get_state(config, subgraphs=True)
    sub_config = parent_state.tasks[0].state.config

    # Fork từ checkpoint của subgraph
    fork_config = graph.update_state(sub_config, {"value": ["forked"]})
    result = graph.invoke(None, fork_config)
    # step_b thực thi lại, kết quả của step_a được giữ nguyên
    ```

Xem [subgraph persistence](./use-subgraphs.md#subgraph-persistence) để biết thêm về cách cấu hình checkpointer cho subgraph.

---

!!! info "Kết nối tài liệu này"
    [Kết nối tài liệu này](https://docs.langchain.com/use-these-docs) với Claude, VSCode, và nhiều công cụ khác qua MCP để có câu trả lời theo thời gian thực.

!!! info "Đóng góp"
    [Chỉnh sửa trang này trên GitHub](https://github.com/langchain-ai/docs/edit/main/src/oss/langgraph/use-time-travel.mdx) hoặc [báo lỗi](https://github.com/langchain-ai/docs/issues/new/choose).
