# MISSING_CHECKPOINTER

Bạn đang cố dùng cơ chế persistence dựng sẵn của LangGraph mà không cung cấp checkpointer.

Lỗi này xảy ra khi `checkpointer` bị thiếu trong phương thức `compile()` của [`StateGraph`](https://reference.langchain.com/python/langgraph/graph/state/StateGraph) hoặc [`@entrypoint`](https://reference.langchain.com/python/langgraph/func/entrypoint).

## Khắc phục sự cố

Những cách sau có thể giúp khắc phục lỗi này:

* Khởi tạo và truyền checkpointer vào phương thức `compile()` của [`StateGraph`](https://reference.langchain.com/python/langgraph/graph/state/StateGraph) hoặc [`@entrypoint`](https://reference.langchain.com/python/langgraph/func/entrypoint).

```python
from langgraph.checkpoint.memory import InMemorySaver
checkpointer = InMemorySaver()

# Graph API
graph = StateGraph(...).compile(checkpointer=checkpointer)

# Functional API
@entrypoint(checkpointer=checkpointer)
def workflow(messages: list[str]) -> str:
    ...
```

* Dùng LangGraph API để không cần tự triển khai hay cấu hình checkpointer thủ công. API này xử lý toàn bộ hạ tầng persistence cho bạn.

## Liên quan

* Tìm hiểu thêm về [persistence](../persistence.md).
