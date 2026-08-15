# LangSmith Observability

Khi bạn xây dựng và chạy agent với LangChain, bạn cần khả năng quan sát (observability) để biết agent đang hoạt động thế nào: agent gọi [tool](/oss/python/langchain/tools) nào, sinh ra prompt gì, và ra quyết định ra sao. Các agent LangChain được xây dựng bằng [`create_agent`](https://reference.langchain.com/python/langchain/agents/factory/create_agent) tự động hỗ trợ tracing thông qua [LangSmith](/langsmith/observability), một nền tảng dùng để ghi lại, debug, đánh giá (evaluate), và giám sát hành vi của ứng dụng LLM.

[*Trace*](/langsmith/observability-concepts#traces) ghi lại từng bước trong quá trình thực thi của agent, từ input ban đầu của người dùng cho đến response cuối cùng, bao gồm toàn bộ tool call, tương tác với model, và các điểm ra quyết định. Dữ liệu thực thi này giúp bạn debug lỗi, đánh giá hiệu năng trên nhiều input khác nhau, và giám sát pattern sử dụng trong môi trường production.

Hướng dẫn này chỉ cho bạn cách bật tracing cho agent LangChain và dùng LangSmith để phân tích quá trình thực thi của chúng.

## Điều kiện tiên quyết

Trước khi bắt đầu, đảm bảo bạn đã có:

* **Một tài khoản LangSmith**: Đăng ký (miễn phí) hoặc đăng nhập tại [smith.langchain.com](https://smith.langchain.com?utm_source=docs&utm_medium=cta&utm_campaign=langsmith-signup&utm_content=oss-langchain-observability).
* **Một LangSmith API key**: Làm theo hướng dẫn [Create an API key](/langsmith/create-account-api-key).

## Bật tracing

Mọi agent LangChain đều tự động hỗ trợ tracing của LangSmith. Để bật tính năng này, thiết lập các biến môi trường sau:

```bash
export LANGSMITH_TRACING=true
export LANGSMITH_API_KEY=<your-api-key>
```

## Bắt đầu nhanh

Không cần viết thêm code để log một trace lên LangSmith. Chỉ cần chạy code agent như bình thường:

```python
from langchain.agents import create_agent


def send_email(to: str, subject: str, body: str):
    """Send an email to a recipient."""
    # ... logic gửi email
    return f"Email sent to {to}"

def search_web(query: str):
    """Search the web for information."""
    # ... logic tìm kiếm web
    return f"Search results for: {query}"

agent = create_agent(
    model="gpt-5.5",
    tools=[send_email, search_web],
    system_prompt="You are a helpful assistant that can send emails and search the web."
)

# Chạy agent, mọi bước sẽ tự động được trace
response = agent.invoke({
    "messages": [{"role": "user", "content": "Search for the latest AI news and email a summary to john@example.com"}]
})
```

Mặc định, trace sẽ được log vào project có tên `default`. Để cấu hình tên project tùy chỉnh, xem [Log to a project](/langsmith/log-traces-to-project).

## Trace có chọn lọc

Bạn có thể chọn chỉ trace một số lệnh gọi hoặc một phần cụ thể của ứng dụng bằng context manager `tracing_context` của LangSmith:

```python
import langsmith as ls

# Đoạn này SẼ được trace
with ls.tracing_context(enabled=True):
    agent.invoke({"messages": [{"role": "user", "content": "Send a test email to alice@example.com"}]})

# Đoạn này sẽ KHÔNG được trace (nếu LANGSMITH_TRACING chưa được set)
agent.invoke({"messages": [{"role": "user", "content": "Send another email"}]})
```

## Log vào một project

!!! note "Thiết lập tĩnh (statically)"
    Bạn có thể đặt tên project tùy chỉnh cho toàn bộ ứng dụng bằng cách thiết lập biến môi trường `LANGSMITH_PROJECT`:

    ```bash
    export LANGSMITH_PROJECT=my-agent-project
    ```

!!! note "Thiết lập động (dynamically)"
    Bạn có thể đặt tên project bằng code cho từng thao tác cụ thể:

    ```python
    import langsmith as ls

    with ls.tracing_context(project_name="email-agent-test", enabled=True):
        response = agent.invoke({
            "messages": [{"role": "user", "content": "Send a welcome email"}]
        })
    ```

## Thêm metadata vào trace

Bạn có thể gắn thêm metadata và tag tùy chỉnh vào trace:

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

Metadata và tag tùy chỉnh này sẽ được gắn kèm vào trace trên LangSmith.

!!! tip "Mẹo"
    Để tìm hiểu thêm cách dùng trace để debug, đánh giá, và giám sát agent, xem [tài liệu LangSmith](/langsmith/observability).
