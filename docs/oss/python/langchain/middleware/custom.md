# Middleware tùy chỉnh

Xây dựng middleware tùy chỉnh bằng cách triển khai các hook chạy tại những điểm cụ thể trong luồng thực thi của agent.

## Hooks

Middleware cung cấp hai kiểu hook để can thiệp vào việc thực thi agent:

**[Node-style hooks](#node-style-hooks)**

Chạy tuần tự tại các điểm thực thi cụ thể.

**[Wrap-style hooks](#wrap-style-hooks)**

Chạy bao quanh mỗi lệnh gọi model hoặc tool.

### Node-style hooks

Chạy tuần tự tại các điểm thực thi cụ thể. Dùng cho việc logging, xác thực (validation), và cập nhật state.

Chọn các hook mà middleware của bạn cần. Bạn có thể chọn giữa node-style hooks và wrap-style hooks.

**Node-style hooks** chạy tại các điểm thực thi cụ thể:

| Hook           | Thời điểm chạy                                        |
| -------------- | ------------------------------------------------------- |
| `before_agent` | Trước khi agent bắt đầu (một lần cho mỗi lần gọi)        |
| `before_model` | Trước mỗi lệnh gọi model                                 |
| `after_model`  | Sau mỗi phản hồi của model                               |
| `after_agent`  | Sau khi agent hoàn tất (một lần cho mỗi lần gọi)          |

**Wrap-style hooks** chạy bao quanh mỗi lệnh gọi, cho bạn quyền kiểm soát việc thực thi:

| Hook               | Thời điểm chạy            |
| ------------------ | --------------------------- |
| `wrap_model_call`   | Bao quanh mỗi lệnh gọi model |
| `wrap_tool_call`    | Bao quanh mỗi lệnh gọi tool  |

**Ví dụ:**

=== "Decorator"

    ```python
    from langchain.agents.middleware import before_model, after_model, AgentState
    from langchain.messages import AIMessage
    from langgraph.runtime import Runtime
    from typing import Any


    @before_model(can_jump_to=["end"])
    def check_message_limit(state: AgentState, runtime: Runtime) -> dict[str, Any] | None:
        if len(state["messages"]) >= 50:
            return {
                "messages": [AIMessage("Conversation limit reached.")],
                "jump_to": "end"
            }
        return None

    @after_model
    def log_response(state: AgentState, runtime: Runtime) -> dict[str, Any] | None:
        print(f"Model returned: {state['messages'][-1].content}")
        return None
    ```

=== "Class"

    ```python
    from langchain.agents.middleware import AgentMiddleware, AgentState, hook_config
    from langchain.messages import AIMessage
    from langgraph.runtime import Runtime
    from typing import Any

    class MessageLimitMiddleware(AgentMiddleware):
        def __init__(self, max_messages: int = 50):
            super().__init__()
            self.max_messages = max_messages

        @hook_config(can_jump_to=["end"])
        def before_model(self, state: AgentState, runtime: Runtime) -> dict[str, Any] | None:
            if len(state["messages"]) >= self.max_messages:
                return {
                    "messages": [AIMessage("Conversation limit reached.")],
                    "jump_to": "end"
                }
            return None

        def after_model(self, state: AgentState, runtime: Runtime) -> dict[str, Any] | None:
            print(f"Model returned: {state['messages'][-1].content}")
            return None
    ```

### Wrap-style hooks

Can thiệp vào việc thực thi và kiểm soát thời điểm handler được gọi. Dùng cho việc thử lại (retry), caching, và biến đổi (transformation).

Bạn quyết định handler được gọi không lần nào (short-circuit), một lần (luồng bình thường), hay nhiều lần (logic thử lại).

**Các hook khả dụng:**

* `wrap_model_call`: bao quanh mỗi lệnh gọi model
* `wrap_tool_call`: bao quanh mỗi lệnh gọi tool

**Ví dụ:**

=== "Decorator"

    ```python
    from langchain.agents.middleware import wrap_model_call, ModelRequest, ModelResponse
    from typing import Callable


    @wrap_model_call
    def retry_model(
        request: ModelRequest,
        handler: Callable[[ModelRequest], ModelResponse],
    ) -> ModelResponse:
        for attempt in range(3):
            try:
                return handler(request)
            except Exception as e:
                if attempt == 2:
                    raise
                print(f"Retry {attempt + 1}/3 after error: {e}")
    ```

=== "Class"

    ```python
    from langchain.agents.middleware import AgentMiddleware, ModelRequest, ModelResponse
    from typing import Callable

    class RetryMiddleware(AgentMiddleware):
        def __init__(self, max_retries: int = 3):
            super().__init__()
            self.max_retries = max_retries

        def wrap_model_call(
            self,
            request: ModelRequest,
            handler: Callable[[ModelRequest], ModelResponse],
        ) -> ModelResponse:
            for attempt in range(self.max_retries):
                try:
                    return handler(request)
                except Exception as e:
                    if attempt == self.max_retries - 1:
                        raise
                    print(f"Retry {attempt + 1}/{self.max_retries} after error: {e}")
    ```

## Cập nhật state

Cả node-style hook và wrap-style hook đều có thể cập nhật state của agent. Cơ chế thực hiện có khác nhau:

* **Node-style hooks** (`before_agent`, `before_model`, `after_model`, `after_agent`): trả về trực tiếp một dict. Dict này được áp dụng vào state của agent thông qua các reducer của graph.
* **Wrap-style hooks** (`wrap_model_call`, `wrap_tool_call`): đối với lệnh gọi model, trả về [`ExtendedModelResponse`](https://reference.langchain.com/python/langchain/agents/middleware/types/ExtendedModelResponse) kèm một [`Command`](https://reference.langchain.com/python/langgraph/types/Command) để chèn các cập nhật state cùng với phản hồi model. Đối với lệnh gọi tool, trả về trực tiếp một [`Command`](https://reference.langchain.com/python/langgraph/types/Command). Dùng cách này khi bạn cần theo dõi hoặc cập nhật state dựa trên logic chạy trong quá trình gọi model hoặc tool, chẳng hạn như các điểm kích hoạt tóm tắt (summarization), usage metadata, hoặc các trường tùy chỉnh được tính toán từ request hoặc response.

### Node-style hooks

Trả về một dict từ node-style hook để gộp các cập nhật vào state của agent. Các key trong dict ánh xạ tới các trường state.

```python
from langchain.agents.middleware import after_model, AgentState
from langgraph.runtime import Runtime
from typing import Any
from typing_extensions import NotRequired


class TrackingState(AgentState):
    model_call_count: NotRequired[int]


@after_model(state_schema=TrackingState)
def increment_after_model(state: TrackingState, runtime: Runtime) -> dict[str, Any] | None:
    return {"model_call_count": state.get("model_call_count", 0) + 1}
```

### Wrap-style hooks

Trả về một [`ExtendedModelResponse`](https://reference.langchain.com/python/langchain/agents/middleware/types/ExtendedModelResponse) kèm một [`Command`](https://reference.langchain.com/python/langgraph/types/Command) từ `wrap_model_call` để chèn các cập nhật state từ lớp gọi model:

```python
from typing import Callable
from langchain.agents.middleware import (
    wrap_model_call,
    ModelRequest,
    ModelResponse,
    AgentState,
    ExtendedModelResponse
)
from langgraph.types import Command
from typing_extensions import NotRequired

class UsageTrackingState(AgentState):
    """Agent state with token usage tracking."""

    last_model_call_tokens: NotRequired[int]


@wrap_model_call(state_schema=UsageTrackingState)
def track_usage(
    request: ModelRequest,
    handler: Callable[[ModelRequest], ModelResponse],
) -> ExtendedModelResponse:
    response = handler(request)
    return ExtendedModelResponse(
        model_response=response,
        command=Command(update={"last_model_call_tokens": 150}),
    )
```

[`Command`](https://reference.langchain.com/python/langgraph/types/Command) đi qua các reducer của graph, vì vậy các cập nhật được áp dụng đúng cách và các tin nhắn được cộng dồn (additive) thay vì thay thế state hiện có.

#### Kết hợp nhiều middleware

Khi nhiều lớp middleware cùng trả về `ExtendedModelResponse`, các command của chúng sẽ được kết hợp:

* **Các command được áp dụng thông qua reducer:** mỗi `Command` trở thành một cập nhật state riêng biệt. Đối với tin nhắn, điều này có nghĩa là chúng được cộng dồn.
* **Middleware ngoài cùng thắng khi có xung đột:** đối với các trường state không dùng reducer, các command được áp dụng theo thứ tự trong trước, ngoài sau. Giá trị của middleware ngoài cùng sẽ được ưu tiên khi các key bị xung đột.
* **An toàn khi thử lại (retry-safe):** nếu middleware ngoài triển khai logic có thể dẫn đến việc gọi `handler()` nhiều lần (ví dụ: logic retry), các command từ những lần gọi trước đó sẽ bị loại bỏ.

```python
from typing import Annotated, Callable

from langchain.agents.middleware import (
    AgentMiddleware,
    AgentState,
    ExtendedModelResponse,
    ModelRequest,
    ModelResponse,
)
from langchain.messages import SystemMessage
from langgraph.types import Command
from typing_extensions import NotRequired


def _last_wins(_a: str, b: str) -> str:
    """Reducer: last writer wins (outer overwrites inner)."""
    return b


class CustomMiddlewareState(AgentState):
    """Agent state: trace_layer uses last-wins (outer wins), messages use additive reducer."""

    # Non-reducer field with last-wins: both middleware write; outermost value wins
    trace_layer: NotRequired[Annotated[str, _last_wins]]


class OuterMiddleware(AgentMiddleware):
    def wrap_model_call(
        self,
        request: ModelRequest,
        handler: Callable[[ModelRequest], ModelResponse],
    ) -> ExtendedModelResponse:
        response = handler(request)
        return ExtendedModelResponse(
            model_response=response,
            command=Command(update={
                "trace_layer": "outer",
                "messages": [SystemMessage(content="[Outer ran]")],
            }),
        )


class InnerMiddleware(AgentMiddleware):
    """Adds trace_layer and message. Outer adds to same keys; trace_layer: outer wins, messages: additive."""

    def wrap_model_call(
        self,
        request: ModelRequest,
        handler: Callable[[ModelRequest], ModelResponse],
    ):
        response = handler(request)
        return ExtendedModelResponse(
            model_response=response,
            command=Command(update={
                "trace_layer": "inner",
                "messages": [SystemMessage(content="[Inner ran]")],
            }),
        )
```

## Tạo middleware

Bạn có thể tạo middleware theo hai cách:

**[Decorator-based middleware](#decorator-based-middleware)**

Nhanh chóng và đơn giản cho middleware chỉ có một hook. Dùng decorator để bọc các hàm riêng lẻ.

**[Class-based middleware](#class-based-middleware)**

Mạnh mẽ hơn cho middleware phức tạp với nhiều hook hoặc cấu hình.

### Decorator-based middleware

Nhanh chóng và đơn giản cho middleware chỉ có một hook. Dùng decorator để bọc các hàm riêng lẻ.

**Các decorator khả dụng:**

**Node-style:**

* [`@before_agent`](https://reference.langchain.com/python/langchain/agents/middleware/types/before_agent): chạy trước khi agent bắt đầu (một lần cho mỗi lần gọi)
* [`@before_model`](https://reference.langchain.com/python/langchain/agents/middleware/types/before_model): chạy trước mỗi lệnh gọi model
* [`@after_model`](https://reference.langchain.com/python/langchain/agents/middleware/types/after_model): chạy sau mỗi phản hồi của model
* [`@after_agent`](https://reference.langchain.com/python/langchain/agents/middleware/types/after_agent): chạy sau khi agent hoàn tất (một lần cho mỗi lần gọi)

**Wrap-style:**

* [`@wrap_model_call`](https://reference.langchain.com/python/langchain/agents/middleware/types/wrap_model_call): bọc mỗi lệnh gọi model với logic tùy chỉnh
* [`@wrap_tool_call`](https://reference.langchain.com/python/langchain/agents/middleware/types/wrap_tool_call): bọc mỗi lệnh gọi tool với logic tùy chỉnh

**Tiện ích:**

* [`@dynamic_prompt`](https://reference.langchain.com/python/langchain/agents/middleware/types/dynamic_prompt): tạo system prompt động

**Ví dụ:**

```python
from langchain.agents.middleware import (
    before_model,
    wrap_model_call,
    AgentState,
    ModelRequest,
    ModelResponse,
)
from langchain.agents import create_agent
from langgraph.runtime import Runtime
from typing import Any, Callable


@before_model
def log_before_model(state: AgentState, runtime: Runtime) -> dict[str, Any] | None:
    print(f"About to call model with {len(state['messages'])} messages")
    return None

@wrap_model_call
def retry_model(
    request: ModelRequest,
    handler: Callable[[ModelRequest], ModelResponse],
) -> ModelResponse:
    for attempt in range(3):
        try:
            return handler(request)
        except Exception as e:
            if attempt == 2:
                raise
            print(f"Retry {attempt + 1}/3 after error: {e}")

agent = create_agent(
    model="gpt-5.5",
    middleware=[log_before_model, retry_model],
    tools=[...],
)
```

**Khi nào dùng decorator:**

* Chỉ cần một hook duy nhất
* Không cần cấu hình phức tạp
* Xây dựng prototype nhanh

### Class-based middleware

Mạnh mẽ hơn cho middleware phức tạp với nhiều hook hoặc cấu hình. Dùng class khi bạn cần định nghĩa cả bản triển khai đồng bộ và bất đồng bộ cho cùng một hook, hoặc khi bạn muốn kết hợp nhiều hook trong một middleware duy nhất.

Một lớp con `AgentMiddleware` có thể khai báo ba thuộc tính lớp mà agent factory sẽ đọc tại thời điểm compile:

* `state_schema`: mở rộng state của agent với các trường tùy chỉnh. Xem [Custom state schema](#custom-state-schema).
* `tools`: đăng ký thêm các tool đi kèm với middleware (ví dụ: `write_todos` trên to-do list middleware).
* `transformers`: đăng ký các factory transformer luồng theo phạm vi (scope-aware). Xem [Custom stream transformers](#custom-stream-transformers).

**Ví dụ:**

```python
from langchain.agents.middleware import (
    AgentMiddleware,
    AgentState,
    ModelRequest,
    ModelResponse,
)
from langgraph.runtime import Runtime
from typing import Any, Callable

class LoggingMiddleware(AgentMiddleware):
    def before_model(self, state: AgentState, runtime: Runtime) -> dict[str, Any] | None:
        print(f"About to call model with {len(state['messages'])} messages")
        return None

    def after_model(self, state: AgentState, runtime: Runtime) -> dict[str, Any] | None:
        print(f"Model returned: {state['messages'][-1].content}")
        return None

    async def abefore_model(
        self, state: AgentState, runtime: Runtime
    ) -> dict[str, Any] | None:
        # Async version of before_model
        return None

    async def aafter_model(
        self, state: AgentState, runtime: Runtime
    ) -> dict[str, Any] | None:
        # Async version of after_model
        print(f"Model returned: {state['messages'][-1].content}")
        return None


agent = create_agent(
    model="gpt-5.5",
    middleware=[LoggingMiddleware()],
    tools=[...],
)
```

**Khi nào dùng class:**

* Cần định nghĩa cả bản triển khai đồng bộ và bất đồng bộ cho cùng một hook
* Cần nhiều hook trong một middleware duy nhất
* Cần cấu hình phức tạp (ví dụ: ngưỡng có thể cấu hình, model tùy chỉnh)
* Tái sử dụng giữa các dự án với cấu hình tại thời điểm khởi tạo

## Custom state schema

Nếu middleware của bạn cần theo dõi state qua các hook, middleware có thể mở rộng state của agent với các thuộc tính tùy chỉnh. Điều này cho phép middleware:

* **Theo dõi state xuyên suốt quá trình thực thi**: duy trì các bộ đếm, cờ (flag), hoặc các giá trị khác tồn tại xuyên suốt vòng đời thực thi của agent

* **Chia sẻ dữ liệu giữa các hook**: truyền thông tin từ `before_model` sang `after_model` hoặc giữa các instance middleware khác nhau

* **Triển khai các mối quan tâm xuyên suốt (cross-cutting concerns)**: thêm các chức năng như rate limiting, theo dõi mức sử dụng, context người dùng, hoặc audit logging mà không cần chỉnh sửa logic lõi của agent

* **Đưa ra quyết định có điều kiện**: dùng state đã tích lũy để quyết định có tiếp tục thực thi, nhảy tới các node khác, hoặc thay đổi hành vi một cách linh hoạt hay không

=== "Decorator"

    ```python
    from langchain.agents import create_agent
    from langchain.messages import HumanMessage
    from langchain.agents.middleware import AgentState, before_model, after_model
    from typing_extensions import NotRequired
    from typing import Any
    from langgraph.runtime import Runtime


    class CustomState(AgentState):
        model_call_count: NotRequired[int]
        user_id: NotRequired[str]


    @before_model(state_schema=CustomState, can_jump_to=["end"])
    def check_call_limit(state: CustomState, runtime: Runtime) -> dict[str, Any] | None:
        count = state.get("model_call_count", 0)
        if count > 10:
            return {"jump_to": "end"}
        return None


    @after_model(state_schema=CustomState)
    def increment_counter(state: CustomState, runtime: Runtime) -> dict[str, Any] | None:
        return {"model_call_count": state.get("model_call_count", 0) + 1}


    agent = create_agent(
        model="gpt-5.5",
        middleware=[check_call_limit, increment_counter],
        tools=[],
    )

    # Invoke with custom state
    result = agent.invoke({
        "messages": [HumanMessage("Hello")],
        "model_call_count": 0,
        "user_id": "user-123",
    })
    ```

=== "Class"

    ```python
    from langchain.agents import create_agent
    from langchain.messages import HumanMessage
    from langchain.agents.middleware import AgentState, AgentMiddleware
    from typing_extensions import NotRequired
    from typing import Any


    class CustomState(AgentState):
        model_call_count: NotRequired[int]
        user_id: NotRequired[str]


    class CallCounterMiddleware(AgentMiddleware[CustomState]):
        state_schema = CustomState

        def before_model(self, state: CustomState, runtime) -> dict[str, Any] | None:
            count = state.get("model_call_count", 0)
            if count > 10:
                return {"jump_to": "end"}
            return None

        def after_model(self, state: CustomState, runtime) -> dict[str, Any] | None:
            return {"model_call_count": state.get("model_call_count", 0) + 1}


    agent = create_agent(
        model="gpt-5.5",
        middleware=[CallCounterMiddleware()],
        tools=[],
    )

    # Invoke with custom state
    result = agent.invoke({
        "messages": [HumanMessage("Hello")],
        "model_call_count": 0,
        "user_id": "user-123",
    })
    ```

## Custom stream transformers

!!! note "Ghi chú"
    Các transformer đăng ký qua middleware yêu cầu `langchain>=1.3.2`.

Middleware có thể đăng ký các factory transformer luồng để chiếu (project) các sự kiện từ luồng agent trực tiếp lên các kênh mở rộng (extension channel) có kiểu dữ liệu xác định. Điều này hữu ích để hiển thị bộ đếm, các artifact kênh phụ (side-channel), đầu ra một phần, hoặc biên tập (redaction) ở mức wire mà không cần gắn kết với các phép chiếu dựng sẵn của framework.

Tại thời điểm compile, các factory được đăng ký qua middleware sẽ được gộp với bất kỳ factory nào caller truyền trực tiếp vào agent factory. [Các quy tắc sắp xếp thứ tự cuối cùng](../event-streaming.md#register-transformers-on-middleware) giữ `ToolCallTransformer` dựng sẵn ở vị trí đầu và để các mục do caller cung cấp nằm ở cuối.

Đặt thuộc tính lớp `transformers` thành một tuple các callable factory. Mỗi factory có dạng `Callable[[tuple[str, ...]], StreamTransformer]` và được gọi dưới dạng `factory(scope)`, trong đó `scope` là tuple phạm vi mini-mux (`()` cho gốc, không rỗng đối với subgraph); việc trả về một transformer mới cho mỗi lần gọi giúp giữ cho mỗi subgraph được cô lập.

```python
from langchain.agents import create_agent
from langchain.agents.middleware import AgentMiddleware


class ToolActivityMiddleware(AgentMiddleware):
    transformers = (ToolActivityTransformer,)


agent = create_agent(
    model="gpt-5-nano",
    tools=[...],
    middleware=[ToolActivityMiddleware()],
)
```

Xem [Đăng ký transformer trên middleware](../event-streaming.md#register-transformers-on-middleware) để biết đầy đủ quy tắc sắp xếp thứ tự và ví dụ về biên tập PII.

## Execution order

Khi dùng nhiều middleware, cần hiểu cách chúng thực thi:

```python
agent = create_agent(
    model="gpt-5.5",
    middleware=[middleware1, middleware2, middleware3],
    tools=[...],
)
```

??? note "Luồng thực thi"
    **Các hook before chạy theo thứ tự:**

    1. `middleware1.before_agent()`
    2. `middleware2.before_agent()`
    3. `middleware3.before_agent()`

    **Vòng lặp agent bắt đầu**

    4. `middleware1.before_model()`
    5. `middleware2.before_model()`
    6. `middleware3.before_model()`

    **Các hook wrap lồng nhau như các lệnh gọi hàm:**

    7. `middleware1.wrap_model_call()` → `middleware2.wrap_model_call()` → `middleware3.wrap_model_call()` → model

    **Các hook after chạy theo thứ tự ngược lại:**

    8. `middleware3.after_model()`
    9. `middleware2.after_model()`
    10. `middleware1.after_model()`

    **Vòng lặp agent kết thúc**

    11. `middleware3.after_agent()`
    12. `middleware2.after_agent()`
    13. `middleware1.after_agent()`

**Các quy tắc chính:**

* Hook `before_*`: từ đầu tới cuối
* Hook `after_*`: từ cuối tới đầu (ngược lại)
* Hook `wrap_*`: lồng nhau (middleware đầu tiên bọc tất cả các middleware khác)

## Agent jumps

Để thoát sớm khỏi middleware, trả về một dict có `jump_to`:

**Các đích nhảy khả dụng:**

* `'end'`: nhảy tới điểm kết thúc của việc thực thi agent (hoặc hook `after_agent` đầu tiên)
* `'tools'`: nhảy tới node tools
* `'model'`: nhảy tới node model (hoặc hook `before_model` đầu tiên)

=== "Decorator"

    ```python
    from langchain.agents.middleware import after_model, hook_config, AgentState
    from langchain.messages import AIMessage
    from langgraph.runtime import Runtime
    from typing import Any


    @after_model
    @hook_config(can_jump_to=["end"])
    def check_for_blocked(state: AgentState, runtime: Runtime) -> dict[str, Any] | None:
        last_message = state["messages"][-1]
        if "BLOCKED" in last_message.content:
            return {
                "messages": [AIMessage("I cannot respond to that request.")],
                "jump_to": "end"
            }
        return None
    ```

=== "Class"

    ```python
    from langchain.agents.middleware import AgentMiddleware, hook_config, AgentState
    from langchain.messages import AIMessage
    from langgraph.runtime import Runtime
    from typing import Any

    class BlockedContentMiddleware(AgentMiddleware):
        @hook_config(can_jump_to=["end"])
        def after_model(self, state: AgentState, runtime: Runtime) -> dict[str, Any] | None:
            last_message = state["messages"][-1]
            if "BLOCKED" in last_message.content:
                return {
                    "messages": [AIMessage("I cannot respond to that request.")],
                    "jump_to": "end"
                }
            return None
    ```

## Thực hành tốt nhất

1. Giữ middleware tập trung, mỗi middleware chỉ nên làm tốt một việc
2. Xử lý lỗi một cách ổn thỏa, không để lỗi middleware làm crash agent
3. **Dùng đúng loại hook**:
   * Node-style cho logic tuần tự (logging, validation)
   * Wrap-style cho luồng điều khiển (retry, fallback, caching)
4. Ghi rõ tài liệu cho mọi thuộc tính state tùy chỉnh
5. Kiểm thử đơn vị (unit test) middleware độc lập trước khi tích hợp
6. Cân nhắc thứ tự thực thi, đặt middleware quan trọng lên đầu danh sách
7. Ưu tiên dùng middleware dựng sẵn khi có thể

## Ví dụ

### Dynamic prompt

Chỉnh sửa động system prompt tại thời điểm chạy để chèn context, chỉ dẫn theo từng người dùng, hoặc các thông tin khác trước mỗi lệnh gọi model. Đây là một trong những trường hợp sử dụng middleware phổ biến nhất.

Dùng trường `system_message` trên `ModelRequest` để đọc và chỉnh sửa system prompt. Trường này chứa một đối tượng [`SystemMessage`](https://reference.langchain.com/python/langchain-core/messages/system/SystemMessage) (kể cả khi agent được tạo với một `system_prompt` dạng chuỗi).

=== "Decorator"

    ```python
    from collections.abc import Callable

    from langchain.agents.middleware import ModelRequest, ModelResponse, wrap_model_call
    from langchain.messages import SystemMessage


    @wrap_model_call
    def add_context(
        request: ModelRequest,
        handler: Callable[[ModelRequest], ModelResponse],
    ) -> ModelResponse:
        new_content = list(request.system_message.content_blocks) + [
            {"type": "text", "text": "Additional context."}
        ]
        new_system_message = SystemMessage(content=new_content)
        return handler(request.override(system_message=new_system_message))
    ```

=== "Class"

    ```python
    from collections.abc import Callable

    from langchain.agents.middleware import AgentMiddleware, ModelRequest, ModelResponse


    class ContextMiddleware(AgentMiddleware):
        def wrap_model_call(
            self,
            request: ModelRequest,
            handler: Callable[[ModelRequest], ModelResponse],
        ) -> ModelResponse:
            new_content = list(request.system_message.content_blocks) + [
                {"type": "text", "text": "Additional context."}
            ]
            new_system_message = SystemMessage(content=new_content)
            return handler(request.override(system_message=new_system_message))
    ```

!!! note "Ghi chú"
    * `ModelRequest.system_message` luôn là một đối tượng [`SystemMessage`](https://reference.langchain.com/python/langchain-core/messages/system/SystemMessage), kể cả khi agent được tạo với `system_prompt="string"`
    * Dùng `SystemMessage.content_blocks` để truy cập nội dung dưới dạng danh sách block, bất kể nội dung gốc là chuỗi hay danh sách
    * Khi chỉnh sửa system message, hãy dùng `content_blocks` và thêm các block mới vào để giữ nguyên cấu trúc hiện có
    * Bạn có thể truyền trực tiếp đối tượng [`SystemMessage`](https://reference.langchain.com/python/langchain-core/messages/system/SystemMessage) vào tham số `system_prompt` của `create_agent` cho các trường hợp sử dụng nâng cao như kiểm soát cache

### Dynamic model selection

=== "Decorator"

    ```python
    from collections.abc import Callable

    from langchain.agents.middleware import ModelRequest, ModelResponse, wrap_model_call
    from langchain.chat_models import init_chat_model

    complex_model = init_chat_model("claude-sonnet-4-6")
    simple_model = init_chat_model("claude-haiku-4-5-20251001")


    @wrap_model_call
    def dynamic_model(
        request: ModelRequest,
        handler: Callable[[ModelRequest], ModelResponse],
    ) -> ModelResponse:
        if len(request.messages) > 10:
            model = complex_model
        else:
            model = simple_model
        return handler(request.override(model=model))
    ```

=== "Class"

    ```python
    from collections.abc import Callable

    from langchain.agents.middleware import AgentMiddleware, ModelRequest, ModelResponse
    from langchain.chat_models import init_chat_model

    complex_model = init_chat_model("claude-sonnet-4-6")
    simple_model = init_chat_model("claude-haiku-4-5-20251001")


    class DynamicModelMiddleware(AgentMiddleware):
        def wrap_model_call(
            self,
            request: ModelRequest,
            handler: Callable[[ModelRequest], ModelResponse],
        ) -> ModelResponse:
            if len(request.messages) > 10:
                model = complex_model
            else:
                model = simple_model
            return handler(request.override(model=model))
    ```

### Dynamically selecting tools

Chọn các tool liên quan tại thời điểm chạy để cải thiện hiệu năng và độ chính xác. Phần này đề cập tới việc lọc các tool đã đăng ký sẵn. Đối với việc đăng ký các tool được khám phá tại thời điểm chạy (ví dụ: từ MCP server), xem [Runtime tool registration](../tools.md#dynamic-tool-selection).

**Lợi ích:**

* **Prompt ngắn hơn**: giảm độ phức tạp bằng cách chỉ hiển thị các tool liên quan
* **Độ chính xác tốt hơn**: model chọn đúng hơn khi có ít lựa chọn hơn
* **Kiểm soát quyền truy cập**: lọc tool linh hoạt dựa trên quyền truy cập của người dùng

=== "Decorator"

    ```python
    from langchain.agents import create_agent
    from langchain.agents.middleware import wrap_model_call, ModelRequest, ModelResponse
    from typing import Callable


    @wrap_model_call
    def select_tools(
        request: ModelRequest,
        handler: Callable[[ModelRequest], ModelResponse],
    ) -> ModelResponse:
        """Middleware to select relevant tools based on state/context."""
        # Select a small, relevant subset of tools based on state/context
        relevant_tools = select_relevant_tools(request.state, request.runtime)
        return handler(request.override(tools=relevant_tools))

    agent = create_agent(
        model="gpt-5.5",
        tools=all_tools,  # All available tools need to be registered upfront
        middleware=[select_tools],
    )
    ```

=== "Class"

    ```python
    from langchain.agents import create_agent
    from langchain.agents.middleware import AgentMiddleware, ModelRequest, ModelResponse
    from typing import Callable


    class ToolSelectorMiddleware(AgentMiddleware):
        def wrap_model_call(
            self,
            request: ModelRequest,
            handler: Callable[[ModelRequest], ModelResponse],
        ) -> ModelResponse:
            """Middleware to select relevant tools based on state/context."""
            # Select a small, relevant subset of tools based on state/context
            relevant_tools = select_relevant_tools(request.state, request.runtime)
            return handler(request.override(tools=relevant_tools))

    agent = create_agent(
        model="gpt-5.5",
        tools=all_tools,  # All available tools need to be registered upfront
        middleware=[ToolSelectorMiddleware()],
    )
    ```

### Tool call monitoring

=== "Decorator"

    ```python
    from collections.abc import Callable

    from langchain.agents.middleware import wrap_tool_call
    from langchain.messages import ToolMessage
    from langchain.tools.tool_node import ToolCallRequest
    from langgraph.types import Command


    @wrap_tool_call
    def monitor_tool(
        request: ToolCallRequest,
        handler: Callable[[ToolCallRequest], ToolMessage | Command],
    ) -> ToolMessage | Command:
        print(f"Executing tool: {request.tool_call['name']}")
        print(f"Arguments: {request.tool_call['args']}")
        try:
            result = handler(request)
            print("Tool completed successfully")
            return result
        except Exception as e:
            print(f"Tool failed: {e}")
            raise
    ```

=== "Class"

    ```python
    from collections.abc import Callable

    from langchain.agents.middleware import AgentMiddleware
    from langchain.messages import ToolMessage
    from langchain.tools.tool_node import ToolCallRequest
    from langgraph.types import Command


    class ToolMonitoringMiddleware(AgentMiddleware):
        def wrap_tool_call(
            self,
            request: ToolCallRequest,
            handler: Callable[[ToolCallRequest], ToolMessage | Command],
        ) -> ToolMessage | Command:
            print(f"Executing tool: {request.tool_call['name']}")
            print(f"Arguments: {request.tool_call['args']}")
            try:
                result = handler(request)
                print("Tool completed successfully")
                return result
            except Exception as e:
                print(f"Tool failed: {e}")
                raise
    ```

### Prompt caching (Anthropic)

Khi làm việc với các model Anthropic, hãy dùng các content block có cấu trúc kèm chỉ thị kiểm soát cache để cache các system prompt lớn:

=== "Decorator"

    ```python
    from langchain.agents.middleware import wrap_model_call, ModelRequest, ModelResponse
    from langchain.messages import SystemMessage
    from typing import Callable


    @wrap_model_call
    def add_cached_context(
        request: ModelRequest,
        handler: Callable[[ModelRequest], ModelResponse],
    ) -> ModelResponse:
        # Always work with content blocks
        new_content = list(request.system_message.content_blocks) + [
            {
                "type": "text",
                "text": "Here is a large document to analyze:\n\n<document>...</document>",
                # content up until this point is cached
                "cache_control": {"type": "ephemeral"}
            }
        ]

        new_system_message = SystemMessage(content=new_content)
        return handler(request.override(system_message=new_system_message))
    ```

=== "Class"

    ```python
    from langchain.agents.middleware import AgentMiddleware, ModelRequest, ModelResponse
    from langchain.messages import SystemMessage
    from typing import Callable


    class CachedContextMiddleware(AgentMiddleware):
        def wrap_model_call(
            self,
            request: ModelRequest,
            handler: Callable[[ModelRequest], ModelResponse],
        ) -> ModelResponse:
            # Always work with content blocks
            new_content = list(request.system_message.content_blocks) + [
                {
                    "type": "text",
                    "text": "Here is a large document to analyze:\n\n<document>...</document>",
                    "cache_control": {"type": "ephemeral"}  # This content will be cached
                }
            ]

            new_system_message = SystemMessage(content=new_content)
            return handler(request.override(system_message=new_system_message))
    ```

**Lưu ý:**

* `ModelRequest.system_message` luôn là một đối tượng [`SystemMessage`](https://reference.langchain.com/python/langchain-core/messages/system/SystemMessage), kể cả khi agent được tạo với `system_prompt="string"`
* Dùng `SystemMessage.content_blocks` để truy cập nội dung dưới dạng danh sách block, bất kể nội dung gốc là chuỗi hay danh sách
* Khi chỉnh sửa system message, hãy dùng `content_blocks` và thêm các block mới vào để giữ nguyên cấu trúc hiện có
* Bạn có thể truyền trực tiếp đối tượng [`SystemMessage`](https://reference.langchain.com/python/langchain-core/messages/system/SystemMessage) vào tham số `system_prompt` của `create_agent` cho các trường hợp sử dụng nâng cao như kiểm soát cache

## Tài nguyên bổ sung

* [Tài liệu tham chiếu API Middleware](https://reference.langchain.com/python/langchain/middleware/)
* [Middleware dựng sẵn](built-in.md)
* [Kiểm thử agent](https://docs.langchain.com/oss/python/langchain/test/)

***

[Kết nối các tài liệu này](https://docs.langchain.com/use-these-docs) với Claude, VSCode, và nhiều công cụ khác qua MCP để có câu trả lời theo thời gian thực.

[Chỉnh sửa trang này trên GitHub](https://github.com/langchain-ai/docs/edit/main/src/oss/langchain/middleware/custom.mdx) hoặc [báo lỗi](https://github.com/langchain-ai/docs/issues/new/choose).
