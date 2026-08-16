# INVALID_CONCURRENT_GRAPH_UPDATE

Một [`StateGraph`](https://reference.langchain.com/python/langgraph/graph/state/StateGraph) LangGraph nhận được các cập nhật đồng thời (concurrent) vào state của nó từ nhiều node, cho một thuộc tính state không hỗ trợ điều này.

Một trường hợp có thể xảy ra là khi bạn dùng [fanout](../use-graph-api.md#map-reduce-and-the-send-api)
hoặc thực thi song song khác trong graph, và bạn định nghĩa graph như sau:

```python
class State(TypedDict):
    some_key: str  # [!code highlight]

def node(state: State):
    return {"some_key": "some_string_value"}

def other_node(state: State):
    return {"some_key": "some_string_value"}


builder = StateGraph(State)
builder.add_node(node)
builder.add_node(other_node)
builder.add_edge(START, "node")
builder.add_edge(START, "other_node")
graph = builder.compile()
```

Nếu một node trong graph trên trả về `{ "some_key": "some_string_value" }`, thao tác này sẽ ghi đè giá trị state của `"some_key"` bằng `"some_string_value"`.
Tuy nhiên, nếu nhiều node (ví dụ trong một fanout ở cùng một step) cùng trả về giá trị cho `"some_key"`, graph sẽ ném ra lỗi này vì
không rõ nên cập nhật state nội bộ như thế nào.

Để giải quyết, bạn có thể định nghĩa một reducer để gộp nhiều giá trị lại:

```python
import operator
from typing import Annotated

class State(TypedDict):
    # Hàm reducer operator.add khiến trường này chỉ có thể append  # [!code highlight]
    some_key: Annotated[list, operator.add]  # [!code highlight]
```

Điều này cho phép bạn định nghĩa logic xử lý cùng một key được trả về từ nhiều node chạy song song.

## Khắc phục sự cố

Những cách sau có thể giúp khắc phục lỗi này:

* Nếu graph của bạn thực thi các node song song, hãy đảm bảo bạn đã định nghĩa reducer cho các state key liên quan.
