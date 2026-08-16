# INVALID_GRAPH_NODE_RETURN_VALUE

Một [`StateGraph`](https://reference.langchain.com/python/langgraph/graph/state/StateGraph) LangGraph
nhận được kiểu trả về không phải dict từ một node. Ví dụ:

```python
class State(TypedDict):
    some_key: str

def bad_node(state: State):
    # Phải trả về một dict với giá trị cho "some_key", không phải một list
    return ["whoops"]

builder = StateGraph(State)
builder.add_node(bad_node)
...

graph = builder.compile()
```

Invoke graph trên sẽ dẫn tới lỗi như sau:

```python
graph.invoke({ "some_key": "someval" });
```

```
InvalidUpdateError: Expected dict, got ['whoops']
For troubleshooting, visit: https://docs.langchain.com/oss/python/langgraph/errors/INVALID_GRAPH_NODE_RETURN_VALUE
```

Các node trong graph của bạn phải trả về một dict chứa một hoặc nhiều key đã định nghĩa trong state.

## Khắc phục sự cố

Những cách sau có thể giúp khắc phục lỗi này:

* Nếu node của bạn có logic phức tạp, hãy đảm bảo mọi nhánh code đều trả về dict phù hợp với state đã định nghĩa.
