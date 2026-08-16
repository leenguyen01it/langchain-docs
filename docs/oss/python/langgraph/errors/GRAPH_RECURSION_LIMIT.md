# GRAPH_RECURSION_LIMIT

[`StateGraph`](https://reference.langchain.com/python/langgraph/graph/state/StateGraph) LangGraph của bạn đã đạt số bước tối đa trước khi gặp điều kiện dừng.
Điều này thường do một vòng lặp vô hạn gây ra bởi đoạn code như ví dụ dưới đây:

```python
class State(TypedDict):
    some_key: str

builder = StateGraph(State)
builder.add_node("a", ...)
builder.add_node("b", ...)
builder.add_edge("a", "b")
builder.add_edge("b", "a")
...

graph = builder.compile()
```

Tuy nhiên, các graph phức tạp cũng có thể tự nhiên chạm giới hạn mặc định.

## Khắc phục sự cố

* Nếu bạn không kỳ vọng graph của mình chạy qua nhiều vòng lặp, rất có thể bạn đang có một chu trình (cycle). Hãy kiểm tra lại logic để tìm vòng lặp vô hạn.

* Nếu bạn có một graph phức tạp, bạn có thể truyền vào một giá trị `recursion_limit` cao hơn trong đối tượng `config` khi gọi graph, như sau:

```python
graph.invoke({...}, {"recursion_limit": 1000})
```
