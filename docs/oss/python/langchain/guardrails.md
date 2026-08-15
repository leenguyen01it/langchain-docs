# Guardrails

> Triển khai kiểm tra an toàn và lọc nội dung cho agent của bạn

Guardrails giúp bạn xây dựng ứng dụng AI an toàn, tuân thủ bằng cách validate và lọc nội dung tại các điểm quan trọng trong quá trình thực thi của agent. Chúng có thể phát hiện thông tin nhạy cảm, áp dụng chính sách nội dung, validate output, và ngăn chặn hành vi không an toàn trước khi chúng gây ra vấn đề.

Các trường hợp sử dụng phổ biến gồm:

* Ngăn rò rỉ PII (thông tin cá nhân)
* Phát hiện và chặn các cuộc tấn công prompt injection
* Chặn nội dung không phù hợp hoặc có hại
* Áp dụng quy tắc kinh doanh và yêu cầu tuân thủ
* Validate chất lượng và độ chính xác của output

Bạn có thể triển khai guardrails bằng [middleware](middleware/overview.md) để can thiệp vào quá trình thực thi tại các điểm chiến lược: trước khi agent bắt đầu, sau khi nó hoàn tất, hoặc xung quanh các lệnh gọi model và tool.

Guardrails có thể được triển khai bằng hai cách tiếp cận bổ sung cho nhau:

**Guardrails xác định (deterministic)**
Dùng logic dựa trên rule như regex pattern, khớp từ khoá, hoặc kiểm tra tường minh. Nhanh, dễ đoán, và tiết kiệm chi phí, nhưng có thể bỏ sót các vi phạm tinh vi.

**Guardrails dựa trên model**
Dùng LLM hoặc classifier để đánh giá nội dung với khả năng hiểu ngữ nghĩa. Bắt được các vấn đề tinh vi mà rule bỏ sót, nhưng chậm hơn và tốn kém hơn.

LangChain cung cấp cả guardrails có sẵn (ví dụ: [phát hiện PII](#phat-hien-pii), [human-in-the-loop](#human-in-the-loop)) và một hệ thống middleware linh hoạt để xây dựng guardrails tuỳ chỉnh bằng một trong hai cách tiếp cận trên.

## Guardrails có sẵn

### Phát hiện PII {#phat-hien-pii}

LangChain cung cấp middleware có sẵn để phát hiện và xử lý thông tin định danh cá nhân (PII) trong hội thoại. Middleware này có thể phát hiện các loại PII phổ biến như email, thẻ tín dụng, địa chỉ IP, và nhiều hơn nữa.

Middleware phát hiện PII hữu ích cho các trường hợp như ứng dụng y tế và tài chính có yêu cầu tuân thủ, agent chăm sóc khách hàng cần làm sạch log, và nói chung là bất kỳ ứng dụng nào xử lý dữ liệu người dùng nhạy cảm.

Middleware PII hỗ trợ nhiều chiến lược xử lý PII đã phát hiện:

| Chiến lược | Mô tả | Ví dụ |
| --- | --- | --- |
| `redact` | Thay bằng `[REDACTED_{PII_TYPE}]` | `[REDACTED_EMAIL]` |
| `mask` | Che một phần (ví dụ 4 số cuối) | `****-****-****-1234` |
| `hash` | Thay bằng hash xác định | `a8f5f167...` |
| `block` | Ném ra exception khi phát hiện | Lỗi được ném ra |

!!! note "Ghi chú"
    Với `apply_to_output=True`, `PIIMiddleware` cũng redact cả wire output đang streaming (text delta, tool-call args, tool output, và state snapshot) qua một stream transformer đã đăng ký. Yêu cầu `langchain>=1.3.2`. Xem [Đăng ký transformer trên middleware](event-streaming.md#register-transformers-on-middleware).

```python
from langchain.agents import create_agent
from langchain.agents.middleware import PIIMiddleware


agent = create_agent(
    model="gpt-5.5",
    tools=[customer_service_tool, email_tool],
    middleware=[
        # Redact email trong input người dùng trước khi gửi cho model
        PIIMiddleware(
            "email",
            strategy="redact",
            apply_to_input=True,
        ),
        # Mask thẻ tín dụng trong input người dùng
        PIIMiddleware(
            "credit_card",
            strategy="mask",
            apply_to_input=True,
        ),
        # Chặn API key - ném lỗi nếu phát hiện
        PIIMiddleware(
            "api_key",
            detector=r"sk-[a-zA-Z0-9]{32}",
            strategy="block",
            apply_to_input=True,
        ),
    ],
)

# Khi người dùng cung cấp PII, nó sẽ được xử lý theo chiến lược đã cấu hình
result = agent.invoke({
    "messages": [{"role": "user", "content": "My email is john.doe@example.com and card is 5105-1051-0510-5100"}]
})
```

??? note "Các loại PII có sẵn và cấu hình"
    **Các loại PII có sẵn:**

    * `email` - Địa chỉ email
    * `credit_card` - Số thẻ tín dụng (đã validate bằng thuật toán Luhn)
    * `ip` - Địa chỉ IP
    * `mac_address` - Địa chỉ MAC
    * `url` - URL

    **Các tuỳ chọn cấu hình:**

    | Tham số | Mô tả | Mặc định |
    | --- | --- | --- |
    | `pii_type` | Loại PII cần phát hiện (có sẵn hoặc tuỳ chỉnh) | Bắt buộc |
    | `strategy` | Cách xử lý PII đã phát hiện (`"block"`, `"redact"`, `"mask"`, `"hash"`) | `"redact"` |
    | `detector` | Hàm detector tuỳ chỉnh hoặc regex pattern | `None` (dùng loại có sẵn) |
    | `apply_to_input` | Kiểm tra message người dùng trước khi gọi model | `True` |
    | `apply_to_output` | Kiểm tra AI message sau khi gọi model | `False` |
    | `apply_to_tool_results` | Kiểm tra message kết quả tool sau khi thực thi | `False` |

Xem [tài liệu middleware](middleware/overview.md#pii-detection) để biết chi tiết đầy đủ về khả năng phát hiện PII.

### Human-in-the-loop {#human-in-the-loop}

LangChain cung cấp middleware có sẵn để yêu cầu con người phê duyệt trước khi thực thi các thao tác nhạy cảm. Đây là một trong những guardrail hiệu quả nhất cho các quyết định rủi ro cao.

Middleware human-in-the-loop hữu ích cho các trường hợp như giao dịch và chuyển khoản tài chính, xoá hoặc sửa dữ liệu production, gửi thông tin liên lạc cho bên ngoài, và bất kỳ thao tác nào có ảnh hưởng kinh doanh đáng kể.

```python
from langchain.agents import create_agent
from langchain.agents.middleware import HumanInTheLoopMiddleware
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.types import Command


agent = create_agent(
    model="gpt-5.5",
    tools=[search_tool, send_email_tool, delete_database_tool],
    middleware=[
        HumanInTheLoopMiddleware(
            interrupt_on={
                # Yêu cầu phê duyệt cho thao tác nhạy cảm
                "send_email": True,
                "delete_database": True,
                # Tự động phê duyệt thao tác an toàn
                "search": False,
            }
        ),
    ],
    # Lưu state qua các lần interrupt
    checkpointer=InMemorySaver(),
)

# Human-in-the-loop yêu cầu một thread ID để lưu state
config = {"configurable": {"thread_id": "some_id"}}

# Agent sẽ tạm dừng và chờ phê duyệt trước khi thực thi tool nhạy cảm
result = agent.invoke(
    {"messages": [{"role": "user", "content": "Send an email to the team"}]},
    config=config
)

result = agent.invoke(
    Command(resume={"decisions": [{"type": "approve"}]}),
    config=config  # Cùng thread ID để tiếp tục hội thoại đang tạm dừng
)
```

!!! tip "Mẹo"
    Xem [tài liệu human-in-the-loop](human-in-the-loop.md) để biết chi tiết đầy đủ về cách triển khai workflow phê duyệt.

## Guardrails tuỳ chỉnh

Với các guardrail phức tạp hơn, bạn có thể tạo middleware tuỳ chỉnh chạy trước hoặc sau khi agent thực thi. Điều này cho bạn toàn quyền kiểm soát logic validate, lọc nội dung, và kiểm tra an toàn.

### Guardrails before agent

Dùng hook "before agent" để validate request một lần khi bắt đầu mỗi lần invoke. Hữu ích cho các kiểm tra cấp session như xác thực, giới hạn tốc độ (rate limiting), hoặc chặn các request không phù hợp trước khi bất kỳ xử lý nào bắt đầu.

=== "Class syntax"
    ```python
    from typing import Any

    from langchain.agents.middleware import AgentMiddleware, AgentState, hook_config
    from langgraph.runtime import Runtime

    class ContentFilterMiddleware(AgentMiddleware):
        """Guardrail xác định: chặn request chứa từ khoá bị cấm."""

        def __init__(self, banned_keywords: list[str]):
            super().__init__()
            self.banned_keywords = [kw.lower() for kw in banned_keywords]

        @hook_config(can_jump_to=["end"])
        def before_agent(self, state: AgentState, runtime: Runtime) -> dict[str, Any] | None:
            # Lấy message đầu tiên của người dùng
            if not state["messages"]:
                return None

            first_message = state["messages"][0]
            if first_message.type != "human":
                return None

            content = first_message.content.lower()

            # Kiểm tra từ khoá bị cấm
            for keyword in self.banned_keywords:
                if keyword in content:
                    # Chặn thực thi trước khi có bất kỳ xử lý nào
                    return {
                        "messages": [{
                            "role": "assistant",
                            "content": "I cannot process requests containing inappropriate content. Please rephrase your request."
                        }],
                        "jump_to": "end"
                    }

            return None

    # Dùng guardrail tuỳ chỉnh
    from langchain.agents import create_agent

    agent = create_agent(
        model="gpt-5.5",
        tools=[search_tool, calculator_tool],
        middleware=[
            ContentFilterMiddleware(
                banned_keywords=["hack", "exploit", "malware"]
            ),
        ],
    )

    # Request này sẽ bị chặn trước khi có bất kỳ xử lý nào
    result = agent.invoke({
        "messages": [{"role": "user", "content": "How do I hack into a database?"}]
    })
    ```

=== "Decorator syntax"
    ```python
    from typing import Any

    from langchain.agents.middleware import before_agent, AgentState, hook_config
    from langgraph.runtime import Runtime

    banned_keywords = ["hack", "exploit", "malware"]

    @before_agent(can_jump_to=["end"])
    def content_filter(state: AgentState, runtime: Runtime) -> dict[str, Any] | None:
        """Guardrail xác định: chặn request chứa từ khoá bị cấm."""
        # Lấy message đầu tiên của người dùng
        if not state["messages"]:
            return None

        first_message = state["messages"][0]
        if first_message.type != "human":
            return None

        content = first_message.content.lower()

        # Kiểm tra từ khoá bị cấm
        for keyword in banned_keywords:
            if keyword in content:
                # Chặn thực thi trước khi có bất kỳ xử lý nào
                return {
                    "messages": [{
                        "role": "assistant",
                        "content": "I cannot process requests containing inappropriate content. Please rephrase your request."
                    }],
                    "jump_to": "end"
                }

        return None

    # Dùng guardrail tuỳ chỉnh
    from langchain.agents import create_agent

    agent = create_agent(
        model="gpt-5.5",
        tools=[search_tool, calculator_tool],
        middleware=[content_filter],
    )

    # Request này sẽ bị chặn trước khi có bất kỳ xử lý nào
    result = agent.invoke({
        "messages": [{"role": "user", "content": "How do I hack into a database?"}]
    })
    ```

### Guardrails after agent

Dùng hook "after agent" để validate output cuối cùng một lần trước khi trả về cho người dùng. Hữu ích cho kiểm tra an toàn dựa trên model, validate chất lượng, hoặc quét tuân thủ cuối cùng trên toàn bộ phản hồi của agent.

=== "Class syntax"
    ```python
    from langchain.agents.middleware import AgentMiddleware, AgentState, hook_config
    from langgraph.runtime import Runtime
    from langchain.messages import AIMessage
    from langchain.chat_models import init_chat_model
    from typing import Any

    class SafetyGuardrailMiddleware(AgentMiddleware):
        """Guardrail dựa trên model: dùng một LLM để đánh giá độ an toàn của phản hồi."""

        def __init__(self):
            super().__init__()
            self.safety_model = init_chat_model("gpt-5.4-mini")

        @hook_config(can_jump_to=["end"])
        def after_agent(self, state: AgentState, runtime: Runtime) -> dict[str, Any] | None:
            # Lấy phản hồi AI cuối cùng
            if not state["messages"]:
                return None

            last_message = state["messages"][-1]
            if not isinstance(last_message, AIMessage):
                return None

            # Dùng một model để đánh giá độ an toàn
            safety_prompt = f"""Evaluate if this response is safe and appropriate.
            Respond with only 'SAFE' or 'UNSAFE'.

            Response: {last_message.content}"""

            result = self.safety_model.invoke([{"role": "user", "content": safety_prompt}])

            if "UNSAFE" in result.content:
                last_message.content = "I cannot provide that response. Please rephrase your request."

            return None

    # Dùng safety guardrail
    from langchain.agents import create_agent

    agent = create_agent(
        model="gpt-5.5",
        tools=[search_tool, calculator_tool],
        middleware=[SafetyGuardrailMiddleware()],
    )

    result = agent.invoke({
        "messages": [{"role": "user", "content": "How do I make explosives?"}]
    })
    ```

=== "Decorator syntax"
    ```python
    from langchain.agents.middleware import after_agent, AgentState, hook_config
    from langgraph.runtime import Runtime
    from langchain.messages import AIMessage
    from langchain.chat_models import init_chat_model
    from typing import Any

    safety_model = init_chat_model("gpt-5.4-mini")

    @after_agent(can_jump_to=["end"])
    def safety_guardrail(state: AgentState, runtime: Runtime) -> dict[str, Any] | None:
        """Guardrail dựa trên model: dùng một LLM để đánh giá độ an toàn của phản hồi."""
        # Lấy phản hồi AI cuối cùng
        if not state["messages"]:
            return None

        last_message = state["messages"][-1]
        if not isinstance(last_message, AIMessage):
            return None

        # Dùng một model để đánh giá độ an toàn
        safety_prompt = f"""Evaluate if this response is safe and appropriate.
        Respond with only 'SAFE' or 'UNSAFE'.

        Response: {last_message.content}"""

        result = safety_model.invoke([{"role": "user", "content": safety_prompt}])

        if "UNSAFE" in result.content:
            last_message.content = "I cannot provide that response. Please rephrase your request."

        return None

    # Dùng safety guardrail
    from langchain.agents import create_agent

    agent = create_agent(
        model="gpt-5.5",
        tools=[search_tool, calculator_tool],
        middleware=[safety_guardrail],
    )

    result = agent.invoke({
        "messages": [{"role": "user", "content": "How do I make explosives?"}]
    })
    ```

### Kết hợp nhiều guardrails

Bạn có thể xếp chồng nhiều guardrail bằng cách thêm chúng vào mảng middleware. Chúng thực thi theo thứ tự, cho phép bạn xây dựng bảo vệ nhiều lớp:

```python
from langchain.agents import create_agent
from langchain.agents.middleware import PIIMiddleware, HumanInTheLoopMiddleware

agent = create_agent(
    model="gpt-5.5",
    tools=[search_tool, send_email_tool],
    middleware=[
        # Lớp 1: Bộ lọc input xác định (trước agent)
        ContentFilterMiddleware(banned_keywords=["hack", "exploit"]),

        # Lớp 2: Bảo vệ PII (trước và sau model)
        PIIMiddleware("email", strategy="redact", apply_to_input=True),
        PIIMiddleware("email", strategy="redact", apply_to_output=True),

        # Lớp 3: Phê duyệt của con người cho tool nhạy cảm
        HumanInTheLoopMiddleware(interrupt_on={"send_email": True}),

        # Lớp 4: Kiểm tra an toàn dựa trên model (sau agent)
        SafetyGuardrailMiddleware(),
    ],
)
```

## Tài nguyên bổ sung

* [Tài liệu middleware](middleware/overview.md) - Hướng dẫn đầy đủ về middleware tuỳ chỉnh
* [API reference middleware](https://reference.langchain.com/python/langchain/middleware/) - Hướng dẫn đầy đủ về middleware tuỳ chỉnh
* [Human-in-the-loop](human-in-the-loop.md) - Thêm review của con người cho các thao tác nhạy cảm
* [Kiểm thử agent](test/index.md) - Chiến lược kiểm thử các cơ chế an toàn
