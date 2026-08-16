# Test

Sau khi đã dựng thử nghiệm (prototype) agent LangGraph, bước tiếp theo tự nhiên là thêm test. Hướng dẫn này trình bày một số pattern hữu ích khi viết unit test.

Lưu ý hướng dẫn này dành riêng cho LangGraph và bao trùm các kịch bản với graph có cấu trúc tuỳ chỉnh, nếu bạn mới bắt đầu, hãy xem [Test](../langchain/test/index.md) sử dụng [`create_agent`](https://reference.langchain.com/python/langchain/agents/factory/create_agent) có sẵn của LangChain thay vì tự dựng graph.

## Điều kiện tiên quyết

Trước tiên, đảm bảo bạn đã cài [`pytest`](https://docs.pytest.org/):

```bash
$ pip install -U pytest
```

## Bắt đầu

Vì nhiều agent LangGraph phụ thuộc vào state, một pattern hữu ích là tạo graph của bạn trước mỗi test nơi bạn sử dụng nó, sau đó compile nó bên trong test với một checkpointer instance mới.

Ví dụ dưới đây cho thấy cách hoạt động với một graph tuyến tính đơn giản, đi qua `node1` và `node2`. Mỗi node cập nhật một state key duy nhất là `my_key`:

```python
import pytest

from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.checkpoint.memory import MemorySaver

def create_graph() -> StateGraph:
    class MyState(TypedDict):
        my_key: str

    graph = StateGraph(MyState)
    graph.add_node("node1", lambda state: {"my_key": "hello from node1"})
    graph.add_node("node2", lambda state: {"my_key": "hello from node2"})
    graph.add_edge(START, "node1")
    graph.add_edge("node1", "node2")
    graph.add_edge("node2", END)
    return graph

def test_basic_agent_execution() -> None:
    checkpointer = MemorySaver()
    graph = create_graph()
    compiled_graph = graph.compile(checkpointer=checkpointer)
    result = compiled_graph.invoke(
        {"my_key": "initial_value"},
        config={"configurable": {"thread_id": "1"}}
    )
    assert result["my_key"] == "hello from node2"
```

## Kiểm thử từng node và edge riêng lẻ

Các agent LangGraph đã compile expose tham chiếu tới từng node riêng lẻ dưới dạng `graph.nodes`. Bạn có thể tận dụng điều này để kiểm thử từng node riêng lẻ bên trong agent. Lưu ý cách này sẽ bỏ qua mọi checkpointer đã truyền khi compile graph:

```python
import pytest

from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.checkpoint.memory import MemorySaver

def create_graph() -> StateGraph:
    class MyState(TypedDict):
        my_key: str

    graph = StateGraph(MyState)
    graph.add_node("node1", lambda state: {"my_key": "hello from node1"})
    graph.add_node("node2", lambda state: {"my_key": "hello from node2"})
    graph.add_edge(START, "node1")
    graph.add_edge("node1", "node2")
    graph.add_edge("node2", END)
    return graph

def test_individual_node_execution() -> None:
    # Sẽ bị bỏ qua trong ví dụ này
    checkpointer = MemorySaver()
    graph = create_graph()
    compiled_graph = graph.compile(checkpointer=checkpointer)
    # Chỉ gọi node 1
    result = compiled_graph.nodes["node1"].invoke(
        {"my_key": "initial_value"},
    )
    assert result["my_key"] == "hello from node1"
```

## Thực thi một phần

Đối với các agent gồm những graph lớn hơn, bạn có thể muốn kiểm thử một phần đường thực thi bên trong agent thay vì toàn bộ luồng từ đầu tới cuối. Trong một số trường hợp, sẽ hợp lý về mặt ngữ nghĩa nếu [tái cấu trúc các phần này thành subgraph](use-subgraphs.md), lúc đó bạn có thể gọi chúng độc lập như bình thường.

Tuy nhiên, nếu bạn không muốn thay đổi cấu trúc tổng thể của graph agent, bạn có thể dùng cơ chế persistence của LangGraph để mô phỏng một trạng thái nơi agent của bạn tạm dừng ngay trước khi bắt đầu phần mong muốn, và sẽ tạm dừng lại ở cuối phần mong muốn. Các bước như sau:

1. Compile agent của bạn với một checkpointer (checkpointer trong bộ nhớ [`InMemorySaver`](https://reference.langchain.com/python/langgraph/checkpoints/#langgraph.checkpoint.memory.InMemorySaver) là đủ dùng cho việc kiểm thử).
2. Gọi phương thức [`update_state`](use-time-travel.md) của agent với tham số [`as_node`](use-time-travel.md#from-a-specific-node) được đặt là tên của node *ngay trước* node bạn muốn bắt đầu kiểm thử.
3. Gọi agent với cùng `thread_id` bạn đã dùng để cập nhật state và tham số `interrupt_after` được đặt là tên của node bạn muốn dừng lại.

Đây là ví dụ chỉ thực thi node thứ hai và thứ ba trong một graph tuyến tính:

```python
import pytest

from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.checkpoint.memory import MemorySaver

def create_graph() -> StateGraph:
    class MyState(TypedDict):
        my_key: str

    graph = StateGraph(MyState)
    graph.add_node("node1", lambda state: {"my_key": "hello from node1"})
    graph.add_node("node2", lambda state: {"my_key": "hello from node2"})
    graph.add_node("node3", lambda state: {"my_key": "hello from node3"})
    graph.add_node("node4", lambda state: {"my_key": "hello from node4"})
    graph.add_edge(START, "node1")
    graph.add_edge("node1", "node2")
    graph.add_edge("node2", "node3")
    graph.add_edge("node3", "node4")
    graph.add_edge("node4", END)
    return graph

def test_partial_execution_from_node2_to_node3() -> None:
    checkpointer = MemorySaver()
    graph = create_graph()
    compiled_graph = graph.compile(checkpointer=checkpointer)
    compiled_graph.update_state(
        config={
          "configurable": {
            "thread_id": "1"
          }
        },
        # State truyền vào node 2 - mô phỏng trạng thái ở
        # cuối node 1
        values={"my_key": "initial_value"},
        # Cập nhật state đã lưu như thể nó đến từ node 1
        # Việc thực thi sẽ tiếp tục từ node 2
        as_node="node1",
    )
    result = compiled_graph.invoke(
        # Tiếp tục thực thi bằng cách truyền None
        None,
        config={"configurable": {"thread_id": "1"}},
        # Dừng sau node 3 để node 4 không chạy
        interrupt_after="node3",
    )
    assert result["my_key"] == "hello from node3"
```
