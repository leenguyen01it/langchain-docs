# Runtime

## Tổng quan

[`create_agent`](https://reference.langchain.com/python/langchain/agents/factory/create_agent) của LangChain chạy trên runtime của LangGraph bên dưới.

LangGraph expose một object [`Runtime`](https://reference.langchain.com/python/langgraph/runtime/Runtime) chứa các thông tin sau:

1. **Context**: thông tin tĩnh như user id, kết nối database, hoặc các dependency khác cho một lần invoke agent
2. **Store**: một instance [BaseStore](https://reference.langchain.com/python/langchain-core/stores/BaseStore) dùng cho [long-term memory](/oss/python/langchain/long-term-memory)
3. **Stream writer**: một object dùng để stream thông tin qua stream mode `"custom"`
4. **Execution info**: thông tin định danh và retry cho lần thực thi hiện tại (thread ID, run ID, số lần thử)
5. **Server info**: metadata đặc thù của server khi chạy trên LangGraph Server (assistant ID, graph ID, user đã xác thực)

!!! tip "Mẹo"
    Runtime context cung cấp **dependency injection** cho tool và middleware của bạn. Thay vì hardcode giá trị hoặc dùng global state, bạn có thể inject các dependency runtime (như kết nối database, user ID, hoặc cấu hình) khi invoke agent. Điều này giúp tool của bạn dễ test, dễ tái sử dụng, và linh hoạt hơn.

Bạn có thể truy cập thông tin runtime bên trong [tool](#inside-tools) và [middleware](#inside-middleware).

## Truy cập

Khi tạo agent bằng [`create_agent`](https://reference.langchain.com/python/langchain/agents/factory/create_agent), bạn có thể chỉ định `context_schema` để định nghĩa cấu trúc của `context` được lưu trong [`Runtime`](https://reference.langchain.com/python/langgraph/runtime/Runtime) của agent.

Khi invoke agent, truyền argument `context` với cấu hình liên quan cho lần chạy đó:

```python
from dataclasses import dataclass

from langchain.agents import create_agent


@dataclass
class Context:
    user_name: str

agent = create_agent(
    model="gpt-5-nano",
    tools=[...],
    context_schema=Context  # [!code highlight]
)

agent.invoke(
    {"messages": [{"role": "user", "content": "What's my name?"}]},
    context=Context(user_name="John Smith")  # [!code highlight]
)
```

### Bên trong tool

Bạn có thể truy cập thông tin runtime bên trong tool để:

* Truy cập context
* Đọc hoặc ghi long-term memory
* Ghi vào [custom stream](/oss/python/langchain/streaming#custom-updates) (ví dụ: tiến trình/cập nhật của tool)

Dùng tham số `ToolRuntime` để truy cập object [`Runtime`](https://reference.langchain.com/python/langgraph/runtime/Runtime) bên trong một tool.

```python
from dataclasses import dataclass
from langchain.tools import tool, ToolRuntime  # [!code highlight]

@dataclass
class Context:
    user_id: str

@tool
def fetch_user_email_preferences(runtime: ToolRuntime[Context]) -> str:  # [!code highlight]
    """Fetch the user's email preferences from the store."""
    user_id = runtime.context.user_id  # [!code highlight]

    preferences: str = "The user prefers you to write a brief and polite email."
    if runtime.store:  # [!code highlight]
        if memory := runtime.store.get(("users",), user_id):  # [!code highlight]
            preferences = memory.value["preferences"]

    return preferences
```

### Execution info và server info bên trong tool

Truy cập định danh thực thi (thread ID, run ID) qua `runtime.execution_info`, và metadata đặc thù của server (assistant ID, user đã xác thực) qua `runtime.server_info` khi chạy trên LangGraph Server:

```python
from langchain.tools import tool, ToolRuntime

@tool
def context_aware_tool(runtime: ToolRuntime) -> str:
    """A tool that uses execution and server info."""
    # Truy cập thread ID và run ID
    info = runtime.execution_info
    print(f"Thread: {info.thread_id}, Run: {info.run_id}")  # [!code highlight]

    # Truy cập server info (chỉ khả dụng trên LangGraph Server)
    server = runtime.server_info
    if server is not None:
        print(f"Assistant: {server.assistant_id}")  # [!code highlight]
        if server.user is not None:
            print(f"User: {server.user.identity}")  # [!code highlight]

    return "done"
```

`server_info` sẽ là `None` khi không chạy trên LangGraph Server (ví dụ: khi phát triển local).

!!! note "Ghi chú"
    Yêu cầu `deepagents>=0.5.0` (hoặc `langgraph>=1.1.5`) để dùng `runtime.execution_info` và `runtime.server_info`.

### Bên trong middleware

Bạn có thể truy cập thông tin runtime trong middleware để tạo prompt động, chỉnh sửa message, hoặc kiểm soát hành vi agent dựa trên context của user.

Dùng tham số `Runtime` để truy cập object [`Runtime`](https://reference.langchain.com/python/langgraph/runtime/Runtime) bên trong [node-style hooks](/oss/python/langchain/middleware/custom#node-style-hooks). Với [wrap-style hooks](/oss/python/langchain/middleware/custom#wrap-style-hooks), object `Runtime` khả dụng bên trong tham số [`ModelRequest`](https://reference.langchain.com/python/langchain/agents/middleware/types/ModelRequest).

```python
from dataclasses import dataclass

from langchain.messages import AnyMessage
from langchain.agents import create_agent, AgentState
from langchain.agents.middleware import dynamic_prompt, ModelRequest, before_model, after_model
from langgraph.runtime import Runtime


@dataclass
class Context:
    user_name: str

# Prompt động
@dynamic_prompt
def dynamic_system_prompt(request: ModelRequest) -> str:
    user_name = request.runtime.context.user_name  # [!code highlight]
    system_prompt = f"You are a helpful assistant. Address the user as {user_name}."
    return system_prompt

# Hook trước khi gọi model
@before_model
def log_before_model(state: AgentState, runtime: Runtime[Context]) -> dict | None:  # [!code highlight]
    print(f"Processing request for user: {runtime.context.user_name}")  # [!code highlight]
    return None

# Hook sau khi gọi model
@after_model
def log_after_model(state: AgentState, runtime: Runtime[Context]) -> dict | None:  # [!code highlight]
    print(f"Completed request for user: {runtime.context.user_name}")  # [!code highlight]
    return None

agent = create_agent(
    model="gpt-5-nano",
    tools=[...],
    middleware=[dynamic_system_prompt, log_before_model, log_after_model],  # [!code highlight]
    context_schema=Context
)

agent.invoke(
    {"messages": [{"role": "user", "content": "What's my name?"}]},
    context=Context(user_name="John Smith")
)
```

### Execution info và server info bên trong middleware

Các hook middleware cũng có thể truy cập `runtime.execution_info` và `runtime.server_info`:

```python
from langchain.agents import AgentState
from langchain.agents.middleware import before_model
from langgraph.runtime import Runtime


@before_model
def auth_gate(state: AgentState, runtime: Runtime) -> dict | None:
    """Block unauthenticated users when running on LangGraph Server."""
    server = runtime.server_info
    if server is not None and server.user is None:  # [!code highlight]
        raise ValueError("Authentication required")
    print(f"Thread: {runtime.execution_info.thread_id}")  # [!code highlight]
    return None
```

!!! note "Ghi chú"
    Yêu cầu `deepagents>=0.5.0` (hoặc `langgraph>=1.1.5`).
