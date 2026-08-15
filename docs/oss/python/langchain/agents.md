# Agents

Một agent là một model gọi tool (tool) theo vòng lặp cho tới khi hoàn thành một task cho trước.

![Sơ đồ vòng lặp agent cốt lõi](https://mintcdn.com/langchain-5e9cc07a/jtty0O--UJOKG0nK/oss/images/core_agent_loop.svg?fit=max&auto=format&n=jtty0O--UJOKG0nK&q=85&s=4b4cbb497b6273758a565de1bc90ece0)

Một harness là toàn bộ những gì bao quanh vòng lặp đó: prompt, các tool, và bất kỳ middleware nào định hình hành vi của model.

!!! note "Agent = Model + Harness"
    Nhiệm vụ của một harness: đưa cho model đúng ngữ cảnh (context) vào đúng thời điểm cho task đang thực hiện.

[`create_agent`](https://reference.langchain.com/python/langchain/agents/factory/create_agent) là một harness có khả năng cấu hình cao. Ở mức đơn giản nhất, bạn có thể tạo một agent với:

=== "Google"
    ```python
    from langchain.agents import create_agent

    agent = create_agent(model="google_genai:gemini-3.6-flash", tools=tools)
    ```

=== "OpenAI"
    ```python
    from langchain.agents import create_agent

    agent = create_agent(model="openai:gpt-5.5", tools=tools)
    ```

=== "Anthropic"
    ```python
    from langchain.agents import create_agent

    agent = create_agent(model="anthropic:claude-sonnet-4-6", tools=tools)
    ```

!!! note "Ghi chú"
    Bản gốc còn có thêm các tab cho OpenRouter, Fireworks, Baseten, và Ollama với cú pháp tương tự, chỉ khác chuỗi định danh model. Từ đây trở đi trong trang này, các CodeGroup nhiều provider được rút gọn còn 3 tab tiêu biểu (Google, OpenAI, Anthropic) để tránh lặp lại quá nhiều; các provider khác dùng cùng pattern, chỉ đổi giá trị `model=`.

Từ nền tảng đó, bạn có thể cấu hình các thành phần cơ bản trực tiếp qua tham số `model=`, `tools=`, và `system_prompt=`. Để có khả năng nâng cao hơn, mở rộng harness bằng [middleware](#cau-hinh-harness).

!!! tip "Mẹo"
    [Deep Agents](https://docs.langchain.com/oss/python/deepagents/overview) được xây trên `create_agent` và đi kèm sẵn các khả năng thường dùng như lập kế hoạch, tool filesystem, subagent, và memory. Dùng `create_agent` khi bạn cần tự cấu hình harness.

## Thành phần cốt lõi

![Sơ đồ các thành phần model và harness của agent](https://mintcdn.com/langchain-5e9cc07a/jtty0O--UJOKG0nK/oss/images/agent_model_harness.svg?fit=max&auto=format&n=jtty0O--UJOKG0nK&q=85&s=5ac6a7e0343af7cb5ba3ca632e2224af)

### Model

Truyền một chuỗi định danh model (`"provider:model"`) hoặc một instance model đã khởi tạo để chọn model cho agent. Xem [Models](models.md) để biết tham số, cách thiết lập provider, và chọn model động.

=== "Google"
    ```python
    from langchain.agents import create_agent

    agent = create_agent(model="google_genai:gemini-3.6-flash", tools=tools)
    ```

=== "OpenAI"
    ```python
    from langchain.agents import create_agent

    agent = create_agent(model="openai:gpt-5.5", tools=tools)
    ```

=== "Anthropic"
    ```python
    from langchain.agents import create_agent

    agent = create_agent(model="anthropic:claude-sonnet-4-6", tools=tools)
    ```

### Tools

Để cấp tool cho agent, truyền bất kỳ Python callable, LangChain tool, hoặc tool dict nào. Xem [Tools](tools.md) để biết cách định nghĩa tool, truy cập context, và chọn tool động.

=== "Google"
    ```python
    from langchain.agents import create_agent
    from langchain.tools import tool


    @tool
    def search(query: str) -> str:
        """Search for information."""
        return f"Results for: {query}"


    agent = create_agent(model="google_genai:gemini-3.6-flash", tools=[search])
    ```

=== "OpenAI"
    ```python
    from langchain.agents import create_agent
    from langchain.tools import tool


    @tool
    def search(query: str) -> str:
        """Search for information."""
        return f"Results for: {query}"


    agent = create_agent(model="openai:gpt-5.5", tools=[search])
    ```

=== "Anthropic"
    ```python
    from langchain.agents import create_agent
    from langchain.tools import tool


    @tool
    def search(query: str) -> str:
        """Search for information."""
        return f"Results for: {query}"


    agent = create_agent(model="anthropic:claude-sonnet-4-6", tools=[search])
    ```

### System prompt

Định hình cách agent tiếp cận task. Tham số system prompt nhận một string hoặc `SystemMessage`. Để có prompt động tại runtime, dùng [middleware](middleware/overview.md).

=== "OpenAI"
    ```python
    agent = create_agent(
        model="openai:gpt-5.5",
        tools=tools,
        system_prompt="You are a helpful assistant. Be concise and accurate.",
    )
    ```

=== "Anthropic"
    ```python
    agent = create_agent(
        model="anthropic:claude-sonnet-4-6",
        tools=tools,
        system_prompt="You are a helpful assistant. Be concise and accurate.",
    )
    ```

### Structured output

Trả về một schema đã validate từ agent bằng `response_format=`. Xem [Structured output](structured-output.md) để biết các chiến lược (strategy) và ví dụ.

=== "OpenAI"
    ```python
    from pydantic import BaseModel
    from langchain.agents import create_agent


    class Answer(BaseModel):
        summary: str
        confidence: float


    agent = create_agent(model="openai:gpt-5.5", tools=tools, response_format=Answer)
    result = agent.invoke({"messages": [{"role": "user", "content": "Summarize AI trends"}]})
    result["structured_response"]  # Answer(summary=..., confidence=...)
    ```

=== "Anthropic"
    ```python
    from pydantic import BaseModel
    from langchain.agents import create_agent


    class Answer(BaseModel):
        summary: str
        confidence: float


    agent = create_agent(model="anthropic:claude-sonnet-4-6", tools=tools, response_format=Answer)
    result = agent.invoke({"messages": [{"role": "user", "content": "Summarize AI trends"}]})
    result["structured_response"]  # Answer(summary=..., confidence=...)
    ```

### Agent state

Mỗi agent quản lý execution context của nó qua [`AgentState`](https://reference.langchain.com/python/langchain/agents/middleware/types/AgentState), một typed dictionary lưu lịch sử hội thoại hiện tại và bất kỳ field tuỳ chỉnh nào mà tool và middleware của bạn cần.

Field có sẵn là:

| Field      | Kiểu                | Mô tả                                                                                                |
| ---------- | ------------------- | ---------------------------------------------------------------------------------------------------------- |
| `messages` | `list[BaseMessage]` | Toàn bộ lịch sử hội thoại cho thread hiện tại. Chỉ được thêm vào (append-only): message mới được thêm, không bao giờ bị thay thế. |

`AgentState` cũng là kiểu (type signature) cho mọi hook middleware kiểu node (`before_model`, `after_model`, và tương tự). Hook nhận state hiện tại và có thể trả về một dict cập nhật để merge ngược lại vào state.

Để thêm field tuỳ chỉnh (ví dụ `user_id` hoặc một bộ đếm), subclass `AgentState` và truyền subclass đó vào `create_agent` qua `state_schema=`:

=== "OpenAI"
    ```python
    from langchain.agents import AgentState, create_agent


    class MyState(AgentState):
        user_id: str
        call_count: int


    agent = create_agent(
        model="openai:gpt-5.5",
        tools=[],
        state_schema=MyState,
    )
    ```

=== "Anthropic"
    ```python
    from langchain.agents import AgentState, create_agent


    class MyState(AgentState):
        user_id: str
        call_count: int


    agent = create_agent(
        model="anthropic:claude-sonnet-4-6",
        tools=[],
        state_schema=MyState,
    )
    ```

Để biết chi tiết đầy đủ, ví dụ, và state schema cấp middleware, xem [Short-term memory](short-term-memory.md#customizing-agent-memory) và [Custom middleware](middleware/custom.md#state-updates).

## Gọi agent (Invocation)

!!! tip "Mẹo"
    Trace từng bước của vòng lặp này, debug tool call, và evaluate output của agent với [LangSmith](https://smith.langchain.com). Làm theo [tracing quickstart](https://docs.langchain.com/langsmith/trace-with-langchain) để thiết lập. Chúng tôi khuyến nghị bạn cũng thiết lập [LangSmith Engine](https://docs.langchain.com/langsmith/engine), công cụ theo dõi trace, phát hiện vấn đề, và đề xuất cách sửa.

Bạn có thể invoke một agent bằng một message. Phía sau, việc này truyền một update tới [`State`](https://docs.langchain.com/oss/python/langgraph/graph-api#state) của agent. Mọi agent đều có một [chuỗi message](https://docs.langchain.com/oss/python/langgraph/use-graph-api#messagesstate) trong state của chúng; để invoke agent, truyền một message mới kèm `thread_id` để agent có thể lưu và tiếp tục lịch sử hội thoại:

=== "OpenAI"
    ```python
    from langchain.agents import create_agent
    from langchain_core.utils.uuid import uuid7
    from langgraph.checkpoint.memory import InMemorySaver

    agent = create_agent(
        model="openai:gpt-5.5",
        tools=[],
        checkpointer=InMemorySaver(),
    )

    config = {"configurable": {"thread_id": str(uuid7())}}

    result = agent.invoke(
        {"messages": [{"role": "user", "content": "What's the weather in San Francisco?"}]},
        config=config,
    )

    # Một lượt tiếp theo trên cùng hội thoại: dùng lại cùng thread_id để giữ lịch sử
    result = agent.invoke(
        {"messages": [{"role": "user", "content": "What about tomorrow?"}]},
        config=config,
    )
    ```

=== "Anthropic"
    ```python
    from langchain.agents import create_agent
    from langchain_core.utils.uuid import uuid7
    from langgraph.checkpoint.memory import InMemorySaver

    agent = create_agent(
        model="anthropic:claude-sonnet-4-6",
        tools=[],
        checkpointer=InMemorySaver(),
    )

    config = {"configurable": {"thread_id": str(uuid7())}}

    result = agent.invoke(
        {"messages": [{"role": "user", "content": "What's the weather in San Francisco?"}]},
        config=config,
    )

    # Một lượt tiếp theo trên cùng hội thoại: dùng lại cùng thread_id để giữ lịch sử
    result = agent.invoke(
        {"messages": [{"role": "user", "content": "What about tomorrow?"}]},
        config=config,
    )
    ```

!!! note "Ghi chú"
    Việc lưu lịch sử hội thoại với `thread_id` yêu cầu agent được cấu hình với một [checkpointer](long-term-memory.md). Khi triển khai (deploy) trên [LangSmith](https://docs.langchain.com/langsmith/deployment), một checkpointer được cấp phát tự động. Khi chạy local, hãy truyền một checkpointer tường minh, ví dụ `create_agent(..., checkpointer=InMemorySaver())`.

Nếu bạn cũng cần truyền cấu hình theo từng lần chạy (per-run), chẳng hạn user ID, API key, hoặc feature flag, cho tool và middleware, hãy truyền dưới dạng `context` bên cạnh `config`. Định nghĩa hình dạng (shape) của dữ liệu đó bằng `context_schema` và truy cập qua `runtime.context`:

=== "OpenAI"
    ```python
    from dataclasses import dataclass

    from langchain.agents import create_agent
    from langchain_core.utils.uuid import uuid7
    from langgraph.checkpoint.memory import InMemorySaver


    @dataclass
    class Context:
        user_id: str


    agent = create_agent(
        model="openai:gpt-5.5",
        tools=[],
        context_schema=Context,
        checkpointer=InMemorySaver(),
    )

    result = agent.invoke(
        {"messages": [{"role": "user", "content": "What's the weather in San Francisco?"}]},
        config={"configurable": {"thread_id": str(uuid7())}},
        context=Context(user_id="user-123"),
    )
    ```

=== "Anthropic"
    ```python
    from dataclasses import dataclass

    from langchain.agents import create_agent
    from langchain_core.utils.uuid import uuid7
    from langgraph.checkpoint.memory import InMemorySaver


    @dataclass
    class Context:
        user_id: str


    agent = create_agent(
        model="anthropic:claude-sonnet-4-6",
        tools=[],
        context_schema=Context,
        checkpointer=InMemorySaver(),
    )

    result = agent.invoke(
        {"messages": [{"role": "user", "content": "What's the weather in San Francisco?"}]},
        config={"configurable": {"thread_id": str(uuid7())}},
        context=Context(user_id="user-123"),
    )
    ```

`thread_id` xác định phạm vi cho *hội thoại* (lịch sử message, checkpoint), còn `context` mang dữ liệu *theo từng lần chạy* mà tool và middleware của bạn đọc tại thời điểm invoke. Cả hai thường được truyền cùng nhau. Xem thêm [tool context](tools.md#context) và [Runtime](runtime.md).

## Streaming

`invoke` trả về response cuối cùng khi kết thúc một lần chạy. Nếu agent thực hiện nhiều tool call, người dùng thường cần cập nhật tiến trình trước khi hoàn tất. Dùng streaming để hiển thị message trung gian và hoạt động của tool ngay khi chúng xảy ra.

```python
from langchain.messages import AIMessage, HumanMessage


stream = agent.stream_events(
    {"messages": [{"role": "user", "content": "Search for AI news and summarize the findings"}]},
    version="v3",
)
for snapshot in stream.values:
    # Mỗi snapshot chứa toàn bộ state tại thời điểm đó
    latest_message = snapshot["messages"][-1]
    if latest_message.content:
        if isinstance(latest_message, HumanMessage):
            print(f"User: {latest_message.content}")
        elif isinstance(latest_message, AIMessage):
            print(f"Agent: {latest_message.content}")
    elif latest_message.tool_calls:
        print(f"Calling tools: {[tc['name'] for tc in latest_message.tool_calls]}")
```

!!! tip "Mẹo"
    Để biết các mode streaming, loại event, và pattern UI, xem [Streaming](streaming.md).

## Cấu hình harness {#cau-hinh-harness}

`create_agent` có khả năng mở rộng cao. Middleware là primitive để tuỳ biến: mỗi phần xử lý một mối quan tâm (concern), gắn vào vòng lặp agent đúng thời điểm, và kết hợp tự do với bất kỳ phần nào khác. Chỉ lấy đúng những gì use case của bạn cần và bỏ qua phần còn lại.

Các pattern phổ biến được đóng gói sẵn dưới dạng middleware hạng nhất (first-class). Bạn có thể tự xây bất kỳ thứ gì khác dưới dạng [middleware tuỳ chỉnh](middleware/custom.md).

![Các nhóm khả năng của agent harness](https://mintcdn.com/langchain-5e9cc07a/jtty0O--UJOKG0nK/oss/images/agent_harness_capabilities.svg?fit=max&auto=format&n=jtty0O--UJOKG0nK&q=85&s=0ff671d72badd0844826660dfcb04391)

Khi agent đảm nhận công việc phức tạp hơn, chúng cần được hỗ trợ ở một số mảng chính. Hệ sinh thái middleware cung cấp:

* **[Môi trường thực thi](#moi-truong-thuc-thi)**: Tool, filesystem, sandbox, và thực thi code
* **[Quản lý context](#quan-ly-context)**: Tóm tắt, memory, skill, và prompt caching
* **[Lập kế hoạch và uỷ quyền](#lap-ke-hoach-va-uy-quyen)**: Todo list và subagent cho công việc song song, cô lập
* **[Chịu lỗi (Fault tolerance)](#chiu-loi)**: Retry, fallback, và giới hạn số lần gọi
* **[Guardrails](#guardrails)**: Phát hiện PII và kiểm soát nội dung
* **[Điều hướng (Steering)](#dieu-huong)**: Con người phê duyệt trước các hành động tác động lớn (human-in-the-loop)

!!! tip "Mẹo"
    `create_deep_agent` đóng gói sẵn toàn bộ stack này cho các task coding và research chạy dài (filesystem, tóm tắt, subagent, và prompt caching có sẵn theo mặc định). Xem [Deep Agents](https://docs.langchain.com/oss/python/deepagents/harness) để biết harness dựng sẵn đầy đủ.

### Môi trường thực thi {#moi-truong-thuc-thi}

Agent đặc biệt hữu ích khi chúng có thể hành động thay vì chỉ sinh văn bản. Môi trường thực thi cấp cho agent một không gian làm việc: tool nó có thể gọi, filesystem để đọc/ghi file qua các lượt, và khả năng thực thi code để chạy script hoặc lệnh shell.

=== "OpenAI"
    ```python
    from langchain.agents import create_agent
    from deepagents.backends import StateBackend
    from deepagents.middleware import FilesystemMiddleware

    agent = create_agent(
        model="openai:gpt-5.5",
        tools=[search],
        middleware=[FilesystemMiddleware(backend=StateBackend())],
    )
    ```

=== "Anthropic"
    ```python
    from langchain.agents import create_agent
    from deepagents.backends import StateBackend
    from deepagents.middleware import FilesystemMiddleware

    agent = create_agent(
        model="anthropic:claude-sonnet-4-6",
        tools=[search],
        middleware=[FilesystemMiddleware(backend=StateBackend())],
    )
    ```

Xem [`FilesystemMiddleware`](https://reference.langchain.com/python/deepagents/middleware/filesystem/FilesystemMiddleware), [Sandboxes](https://docs.langchain.com/oss/python/deepagents/sandboxes), [Interpreters](https://docs.langchain.com/oss/python/deepagents/interpreters).

!!! note "Ghi chú"
    Ví dụ này import từ package `deepagents`. Cài đặt bằng:

    ```bash
    pip install deepagents
    ```

    hoặc

    ```bash
    uv add deepagents
    ```

### Quản lý context {#quan-ly-context}

Mỗi lần gọi model có một context window kích thước cố định. Khi agent chạy, cửa sổ đó dần lấp đầy bởi lịch sử tích luỹ, kết quả tool, và các bước trung gian. Tóm tắt (summarization) nén lịch sử trước khi tràn; memory nạp các chỉ dẫn lâu dài (persistent) khi khởi động để kiến thức được giữ xuyên suốt các session; skill hiển thị kiến thức chuyên biệt theo nhu cầu thay vì nạp hết mọi thứ ngay từ đầu.

=== "OpenAI"
    ```python
    from deepagents.backends import StateBackend
    from deepagents.middleware import FilesystemMiddleware, MemoryMiddleware, SkillsMiddleware, SummarizationMiddleware

    backend = StateBackend()
    model = "openai:gpt-5.5"

    agent = create_agent(
        model=model,
        tools=[search],
        middleware=[
            FilesystemMiddleware(backend=backend),
            SummarizationMiddleware(model=model, backend=backend),
            MemoryMiddleware(backend=backend, sources=["./AGENTS.md"]),
            SkillsMiddleware(backend=backend, sources=["./skills/"]),
        ],
    )
    ```

=== "Anthropic"
    ```python
    from deepagents.backends import StateBackend
    from deepagents.middleware import FilesystemMiddleware, MemoryMiddleware, SkillsMiddleware, SummarizationMiddleware

    backend = StateBackend()
    model = "anthropic:claude-sonnet-4-6"

    agent = create_agent(
        model=model,
        tools=[search],
        middleware=[
            FilesystemMiddleware(backend=backend),
            SummarizationMiddleware(model=model, backend=backend),
            MemoryMiddleware(backend=backend, sources=["./AGENTS.md"]),
            SkillsMiddleware(backend=backend, sources=["./skills/"]),
        ],
    )
    ```

Xem [`SummarizationMiddleware`](https://reference.langchain.com/python/langchain/agents/middleware/summarization/SummarizationMiddleware), [`MemoryMiddleware`](https://reference.langchain.com/python/deepagents/middleware/memory/MemoryMiddleware), [Skills](multi-agent/skills.md), [Context engineering](https://docs.langchain.com/oss/python/deepagents/context-engineering).

!!! note "Ghi chú"
    Ví dụ này import từ package `deepagents`. Cài đặt bằng `pip install deepagents` hoặc `uv add deepagents`.

### Lập kế hoạch và uỷ quyền {#lap-ke-hoach-va-uy-quyen}

Các task phức tạp thường vượt quá những gì một context window đơn lẻ có thể xử lý. Uỷ quyền (delegation) cho phép agent chính chia công việc thành từng phần, giao cho các subagent chạy trong context cô lập riêng, và giữ agent chính tập trung vào điều phối thay vì thực thi. Công việc có thể chạy song song; context của agent chính luôn sạch.

=== "OpenAI"
    ```python
    from deepagents.backends import StateBackend
    from deepagents.middleware import FilesystemMiddleware
    from deepagents.middleware.subagents import SubAgentMiddleware
    from langchain.agents import create_agent
    from langchain.agents.middleware import TodoListMiddleware
    from langchain.tools import tool


    @tool
    def search(query: str) -> str:
        """Search for a query and return a short summary."""
        return f"Search results for: {query}"


    backend = StateBackend()

    agent = create_agent(
        model="openai:gpt-5.5",
        tools=[search],
        middleware=[
            FilesystemMiddleware(backend=backend),
            TodoListMiddleware(),
            SubAgentMiddleware(
                backend=backend,
                subagents=[
                    {
                        "name": "researcher",
                        "description": "Searches and returns a structured summary.",
                        "system_prompt": "Use the search tool to research the question and summarize key points.",
                        "tools": [search],
                        "model": "anthropic:claude-sonnet-4-6",
                        "middleware": [],
                    }
                ],
            ),
        ],
    )
    ```

=== "Anthropic"
    ```python
    from deepagents.backends import StateBackend
    from deepagents.middleware import FilesystemMiddleware
    from deepagents.middleware.subagents import SubAgentMiddleware
    from langchain.agents import create_agent
    from langchain.agents.middleware import TodoListMiddleware
    from langchain.tools import tool


    @tool
    def search(query: str) -> str:
        """Search for a query and return a short summary."""
        return f"Search results for: {query}"


    backend = StateBackend()

    agent = create_agent(
        model="anthropic:claude-sonnet-4-6",
        tools=[search],
        middleware=[
            FilesystemMiddleware(backend=backend),
            TodoListMiddleware(),
            SubAgentMiddleware(
                backend=backend,
                subagents=[
                    {
                        "name": "researcher",
                        "description": "Searches and returns a structured summary.",
                        "system_prompt": "Use the search tool to research the question and summarize key points.",
                        "tools": [search],
                        "model": "anthropic:claude-sonnet-4-6",
                        "middleware": [],
                    }
                ],
            ),
        ],
    )
    ```

Xem [Subagents](multi-agent/subagents.md).

!!! note "Ghi chú"
    Ví dụ này import từ package `deepagents`. Cài đặt bằng `pip install deepagents` hoặc `uv add deepagents`.

### Đặt tên agent

Tuỳ chọn dùng một định danh cho agent. Điều này đặc biệt hữu ích khi nhúng agent như một subgraph trong hệ thống [multi-agent](multi-agent/index.md).

=== "OpenAI"
    ```python
    agent = create_agent(model="openai:gpt-5.5", tools=tools, name="research_assistant")
    ```

=== "Anthropic"
    ```python
    agent = create_agent(model="anthropic:claude-sonnet-4-6", tools=tools, name="research_assistant")
    ```

### Chịu lỗi (Fault tolerance) {#chiu-loi}

Agent trong production gặp phải các lỗi hiếm khi xuất hiện lúc phát triển: rate limit, model timeout, lỗi API tạm thời. Middleware chịu lỗi xử lý những vấn đề này ở tầng hạ tầng (infrastructure) để tool và business logic của bạn không cần try/catch quanh mỗi lần gọi.

=== "OpenAI"
    ```python
    from langchain.agents import create_agent
    from langchain.agents.middleware import ModelRetryMiddleware, ToolRetryMiddleware
    from langchain.tools import tool


    @tool
    def search(query: str) -> str:
        """Search for a query and return a short summary."""
        return f"Search results for: {query}"


    agent = create_agent(
        model="openai:gpt-5.5",
        tools=[search],
        middleware=[
            ModelRetryMiddleware(max_retries=3),
            ToolRetryMiddleware(max_retries=2),
        ],
    )
    ```

=== "Anthropic"
    ```python
    from langchain.agents import create_agent
    from langchain.agents.middleware import ModelRetryMiddleware, ToolRetryMiddleware
    from langchain.tools import tool


    @tool
    def search(query: str) -> str:
        """Search for a query and return a short summary."""
        return f"Search results for: {query}"


    agent = create_agent(
        model="anthropic:claude-sonnet-4-6",
        tools=[search],
        middleware=[
            ModelRetryMiddleware(max_retries=3),
            ToolRetryMiddleware(max_retries=2),
        ],
    )
    ```

Xem [`ModelRetryMiddleware`](https://reference.langchain.com/python/langchain/agents/middleware/model_retry/ModelRetryMiddleware), [`ToolRetryMiddleware`](https://reference.langchain.com/python/langchain/agents/middleware/tool_retry/ToolRetryMiddleware), [Prebuilt middleware](middleware/built-in.md).

### Guardrails

Một số chính sách không thể sống trong prompt, chúng cần được thực thi một cách xác định (deterministic) bất kể model làm gì. Guardrail chặn dữ liệu khi nó chảy qua vòng lặp agent, áp dụng quy tắc tuân thủ (compliance) hoặc chính sách nội dung trước khi kết quả tool tới context của model.

=== "OpenAI"
    ```python
    from langchain.agents import create_agent
    from langchain.agents.middleware import PIIMiddleware
    from langchain.tools import tool


    @tool
    def search(query: str) -> str:
        """Search for a query and return a short summary."""
        return f"Search results for: {query}"


    agent = create_agent(
        model="openai:gpt-5.5",
        tools=[search],
        middleware=[PIIMiddleware("email")],
    )
    ```

=== "Anthropic"
    ```python
    from langchain.agents import create_agent
    from langchain.agents.middleware import PIIMiddleware
    from langchain.tools import tool


    @tool
    def search(query: str) -> str:
        """Search for a query and return a short summary."""
        return f"Search results for: {query}"


    agent = create_agent(
        model="anthropic:claude-sonnet-4-6",
        tools=[search],
        middleware=[PIIMiddleware("email")],
    )
    ```

Xem [`PIIMiddleware`](https://reference.langchain.com/python/langchain/agents/middleware/pii/PIIMiddleware), [Prebuilt middleware](middleware/built-in.md).

### Điều hướng (Steering) {#dieu-huong}

Tự chủ hoàn toàn không phải lúc nào cũng phù hợp. Steering cho phép bạn đặt con người vào các điểm ra quyết định cụ thể (trước khi ghi đè phá huỷ, gọi API tốn kém, hoặc bất kỳ điều gì cần phán đoán) mà không cần tái cấu trúc agent. Agent tạm dừng và chờ; con người phê duyệt, chỉnh sửa, hoặc từ chối; việc thực thi tiếp tục.

=== "OpenAI"
    ```python
    from langchain.agents import create_agent
    from langchain.agents.middleware import HumanInTheLoopMiddleware
    from langchain.tools import tool


    @tool
    def search(query: str) -> str:
        """Search for a query and return a short summary."""
        return f"Search results for: {query}"


    agent = create_agent(
        model="openai:gpt-5.5",
        tools=[search],
        middleware=[HumanInTheLoopMiddleware(interrupt_on={"write_file": True})],
    )
    ```

=== "Anthropic"
    ```python
    from langchain.agents import create_agent
    from langchain.agents.middleware import HumanInTheLoopMiddleware
    from langchain.tools import tool


    @tool
    def search(query: str) -> str:
        """Search for a query and return a short summary."""
        return f"Search results for: {query}"


    agent = create_agent(
        model="anthropic:claude-sonnet-4-6",
        tools=[search],
        middleware=[HumanInTheLoopMiddleware(interrupt_on={"write_file": True})],
    )
    ```

Xem [`HumanInTheLoopMiddleware`](https://reference.langchain.com/python/langchain/agents/middleware/human_in_the_loop/HumanInTheLoopMiddleware), [Human-in-the-loop](human-in-the-loop.md).

### Tài nguyên middleware

* **[Middleware overview](middleware/overview.md)**: Cách stack middleware hoạt động và thời điểm hook được gọi
* **[Prebuilt middleware](middleware/built-in.md)**: Tài liệu tham khảo đầy đủ kèm ví dụ cấu hình
* **[Custom middleware](middleware/custom.md)**: Tự viết hook cho business logic, làm sạch PII, và nhiều hơn nữa

---

!!! info "Kết nối tài liệu này"
    [Kết nối tài liệu này](https://docs.langchain.com/use-these-docs) với Claude, VSCode, và nhiều công cụ khác qua MCP để có câu trả lời theo thời gian thực.

!!! info "Đóng góp"
    [Chỉnh sửa trang này trên GitHub](https://github.com/langchain-ai/docs/edit/main/src/oss/langchain/agents.mdx) hoặc [báo lỗi](https://github.com/langchain-ai/docs/issues/new/choose).
