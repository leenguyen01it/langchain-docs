# LangSmith Observability

Trace là một chuỗi các bước mà ứng dụng của bạn thực hiện để đi từ input đến output. Mỗi bước riêng lẻ trong đó được biểu diễn bởi một run. Bạn có thể dùng [LangSmith](https://smith.langchain.com) để trực quan hoá các bước thực thi này. Để dùng nó, hãy [bật tracing cho ứng dụng của bạn](https://docs.langchain.com/langsmith/trace-with-langgraph). Điều này cho phép bạn:

* [Debug một ứng dụng đang chạy local](https://docs.langchain.com/langsmith/observability-studio#debug-langsmith-traces).
* [Đánh giá hiệu năng ứng dụng](../langchain/test/evals.md).
* [Giám sát ứng dụng](https://docs.langchain.com/langsmith/dashboards).

## Điều kiện tiên quyết

Trước khi bắt đầu, hãy đảm bảo bạn có:

* **Một tài khoản LangSmith**: Đăng ký (miễn phí) hoặc đăng nhập tại [smith.langchain.com](https://smith.langchain.com).
* **Một LangSmith API key**: Làm theo hướng dẫn [Tạo API key](https://docs.langchain.com/langsmith/create-account-api-key).

## Bật tracing

Để bật tracing cho ứng dụng của bạn, thiết lập các biến môi trường sau:

```python
export LANGSMITH_TRACING=true
export LANGSMITH_API_KEY=<your-api-key>
```

Theo mặc định, trace sẽ được ghi vào project có tên `default`. Để cấu hình tên project tuỳ chỉnh, xem [Ghi log vào một project](#log-to-a-project).

Xem thêm tại [Trace với LangGraph](https://docs.langchain.com/langsmith/trace-with-langgraph).

## Trace có chọn lọc

Bạn có thể chọn chỉ trace một số lệnh gọi hoặc một số phần cụ thể của ứng dụng bằng context manager `tracing_context` của LangSmith:

```python
import langsmith as ls

# Đoạn này SẼ được trace
with ls.tracing_context(enabled=True):
    agent.invoke({"messages": [{"role": "user", "content": "Send a test email to alice@example.com"}]})

# Đoạn này sẽ KHÔNG được trace (nếu LANGSMITH_TRACING chưa được set)
agent.invoke({"messages": [{"role": "user", "content": "Send another email"}]})
```

## Ghi log vào một project

??? note "Tĩnh (Statically)"
    Bạn có thể đặt tên project tuỳ chỉnh cho toàn bộ ứng dụng bằng cách thiết lập biến môi trường `LANGSMITH_PROJECT`:

    ```bash
    export LANGSMITH_PROJECT=my-agent-project
    ```

??? note "Động (Dynamically)"
    Bạn có thể đặt tên project theo chương trình (programmatically) cho các thao tác cụ thể:

    ```python
    import langsmith as ls

    with ls.tracing_context(project_name="email-agent-test", enabled=True):
        response = agent.invoke({
            "messages": [{"role": "user", "content": "Send a welcome email"}]
        })
    ```

## Thêm metadata vào trace

Bạn có thể gắn chú thích (annotate) cho trace của mình với metadata và tag tuỳ chỉnh:

```python
response = agent.invoke(
    {"messages": [{"role": "user", "content": "Send a welcome email"}]},
    config={
        "tags": ["production", "email-assistant", "v1.0"],
        "metadata": {
            "user_id": "user_123",
            "session_id": "session_456",
            "environment": "production"
        }
    }
)
```

`tracing_context` cũng nhận tag và metadata để kiểm soát chi tiết hơn:

```python
with ls.tracing_context(
    project_name="email-agent-test",
    enabled=True,
    tags=["production", "email-assistant", "v1.0"],
    metadata={"user_id": "user_123", "session_id": "session_456", "environment": "production"}):
    response = agent.invoke(
        {"messages": [{"role": "user", "content": "Send a welcome email"}]}
    )
```

Metadata và tag tuỳ chỉnh này sẽ được gắn vào trace trong LangSmith.

!!! tip
    Để tìm hiểu thêm cách dùng trace để debug, đánh giá, và giám sát agent của bạn, xem [tài liệu LangSmith](https://docs.langchain.com/langsmith/observability).

## Dùng anonymizer để tránh ghi log dữ liệu nhạy cảm trong trace

Bạn có thể muốn che (mask) dữ liệu nhạy cảm để tránh ghi log lên LangSmith. Bạn có thể tạo [anonymizer](https://docs.langchain.com/langsmith/mask-inputs-outputs#rule-based-masking-of-inputs-and-outputs) và áp dụng chúng vào
graph của mình thông qua cấu hình. Ví dụ dưới đây sẽ ẩn đi mọi chuỗi khớp định dạng Social Security Number XXX-XX-XXXX khỏi trace gửi tới LangSmith.

```python Python
from langchain_core.tracers.langchain import LangChainTracer
from langgraph.graph import StateGraph, MessagesState
from langsmith import Client
from langsmith.anonymizer import create_anonymizer

anonymizer = create_anonymizer([
    # Khớp với SSN
    { "pattern": r"\b\d{3}-?\d{2}-?\d{4}\b", "replace": "<ssn>" }
])

tracer_client = Client(anonymizer=anonymizer)
tracer = LangChainTracer(client=tracer_client)
# Định nghĩa graph
graph = (
    StateGraph(MessagesState)
    ...
    .compile()
    .with_config({'callbacks': [tracer]})
)
```
