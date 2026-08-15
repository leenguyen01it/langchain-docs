# Bộ nhớ ngắn hạn (Short-term memory)

## Tổng quan

Bộ nhớ (memory) là hệ thống ghi nhớ thông tin về các tương tác trước đó. Đối với AI agent, bộ nhớ rất quan trọng vì nó cho phép agent nhớ lại các tương tác trước, học hỏi từ phản hồi (feedback), và thích nghi với sở thích của người dùng. Khi agent xử lý các tác vụ phức tạp hơn với nhiều tương tác người dùng, khả năng này trở nên thiết yếu cho cả hiệu quả lẫn sự hài lòng của người dùng.

Bộ nhớ ngắn hạn cho phép ứng dụng của bạn nhớ các tương tác trước đó trong cùng một thread hoặc cuộc hội thoại.

!!! note "Ghi chú"
    Một thread tổ chức nhiều tương tác trong một phiên (session), tương tự cách email nhóm các tin nhắn trong cùng một cuộc hội thoại.

Lịch sử hội thoại (conversation history) là dạng bộ nhớ ngắn hạn phổ biến nhất. Các cuộc hội thoại dài đặt ra thách thức cho các LLM hiện nay; toàn bộ lịch sử có thể không vừa trong context window của LLM, dẫn đến mất context hoặc lỗi.

Ngay cả khi model của bạn hỗ trợ độ dài context đầy đủ, hầu hết LLM vẫn hoạt động kém với context dài. Chúng bị "phân tâm" bởi nội dung cũ hoặc không liên quan, đồng thời chịu thời gian phản hồi chậm hơn và chi phí cao hơn.

Chat model nhận context thông qua [messages](messages.md), bao gồm hướng dẫn (system message) và đầu vào (human message). Trong ứng dụng chat, các message xen kẽ giữa đầu vào của người dùng và phản hồi của model, tạo ra một danh sách message ngày càng dài theo thời gian. Vì context window có giới hạn, nhiều ứng dụng có thể hưởng lợi từ các kỹ thuật loại bỏ hoặc "quên" thông tin cũ.

!!! tip "Mẹo"
    Cần nhớ thông tin **giữa** các cuộc hội thoại? Dùng [bộ nhớ dài hạn](long-term-memory.md) để lưu trữ và gọi lại dữ liệu theo người dùng hoặc theo ứng dụng qua các thread và session khác nhau.

## Cách sử dụng

Để thêm bộ nhớ ngắn hạn (persistence ở cấp thread) vào một agent, bạn cần chỉ định `checkpointer` khi tạo agent.

!!! info "Thông tin"
    Agent của LangChain quản lý bộ nhớ ngắn hạn như một phần của state của agent.

    Bằng cách lưu các thông tin này trong state của graph, agent có thể truy cập toàn bộ context cho một cuộc hội thoại nhất định, đồng thời vẫn giữ sự tách biệt giữa các thread khác nhau.

    State được lưu vào một cơ sở dữ liệu (hoặc bộ nhớ) thông qua checkpointer để thread có thể được tiếp tục bất cứ lúc nào.

    Bộ nhớ ngắn hạn được cập nhật khi agent được gọi (invoke) hoặc khi một bước (như một tool call) hoàn tất, và state được đọc vào đầu mỗi bước.

=== "Google"
    ```python
    from langchain.agents import create_agent
    from langgraph.checkpoint.memory import InMemorySaver  # [!code highlight]


    def get_user_info() -> str:
        """Look up information about the current user."""
        return "No user profile on file."


    agent = create_agent(
        model="google_genai:gemini-3.6-flash",
        tools=[get_user_info],
        checkpointer=InMemorySaver(),  # [!code highlight]
    )

    thread_config = {"configurable": {"thread_id": "1"}}
    response = agent.invoke(
        {"messages": [{"role": "user", "content": "Hi! My name is Bob."}]},
        thread_config,  # [!code highlight]
    )["messages"][-1].content

    print(response)  # "Hi Bob! Nice to see you here. How are you doing?"

    response = agent.invoke(
        {"messages": [{"role": "user", "content": "What's my name?"}]},
        thread_config,  # [!code highlight]
    )["messages"][-1].content

    print(response)  # "You are Bob!"
    ```

=== "OpenAI"
    ```python
    from langchain.agents import create_agent
    from langgraph.checkpoint.memory import InMemorySaver  # [!code highlight]


    def get_user_info() -> str:
        """Look up information about the current user."""
        return "No user profile on file."


    agent = create_agent(
        model="openai:gpt-5.5",
        tools=[get_user_info],
        checkpointer=InMemorySaver(),  # [!code highlight]
    )

    thread_config = {"configurable": {"thread_id": "1"}}
    response = agent.invoke(
        {"messages": [{"role": "user", "content": "Hi! My name is Bob."}]},
        thread_config,  # [!code highlight]
    )["messages"][-1].content

    print(response)  # "Hi Bob! Nice to see you here. How are you doing?"

    response = agent.invoke(
        {"messages": [{"role": "user", "content": "What's my name?"}]},
        thread_config,  # [!code highlight]
    )["messages"][-1].content

    print(response)  # "You are Bob!"
    ```

=== "Anthropic"
    ```python
    from langchain.agents import create_agent
    from langgraph.checkpoint.memory import InMemorySaver  # [!code highlight]


    def get_user_info() -> str:
        """Look up information about the current user."""
        return "No user profile on file."


    agent = create_agent(
        model="anthropic:claude-sonnet-4-6",
        tools=[get_user_info],
        checkpointer=InMemorySaver(),  # [!code highlight]
    )

    thread_config = {"configurable": {"thread_id": "1"}}
    response = agent.invoke(
        {"messages": [{"role": "user", "content": "Hi! My name is Bob."}]},
        thread_config,  # [!code highlight]
    )["messages"][-1].content

    print(response)  # "Hi Bob! Nice to see you here. How are you doing?"

    response = agent.invoke(
        {"messages": [{"role": "user", "content": "What's my name?"}]},
        thread_config,  # [!code highlight]
    )["messages"][-1].content

    print(response)  # "You are Bob!"
    ```

=== "OpenRouter"
    ```python
    from langchain.agents import create_agent
    from langgraph.checkpoint.memory import InMemorySaver  # [!code highlight]


    def get_user_info() -> str:
        """Look up information about the current user."""
        return "No user profile on file."


    agent = create_agent(
        model="openrouter:z-ai/glm-5.2",
        tools=[get_user_info],
        checkpointer=InMemorySaver(),  # [!code highlight]
    )

    thread_config = {"configurable": {"thread_id": "1"}}
    response = agent.invoke(
        {"messages": [{"role": "user", "content": "Hi! My name is Bob."}]},
        thread_config,  # [!code highlight]
    )["messages"][-1].content

    print(response)  # "Hi Bob! Nice to see you here. How are you doing?"

    response = agent.invoke(
        {"messages": [{"role": "user", "content": "What's my name?"}]},
        thread_config,  # [!code highlight]
    )["messages"][-1].content

    print(response)  # "You are Bob!"
    ```

=== "Fireworks"
    ```python
    from langchain.agents import create_agent
    from langgraph.checkpoint.memory import InMemorySaver  # [!code highlight]


    def get_user_info() -> str:
        """Look up information about the current user."""
        return "No user profile on file."


    agent = create_agent(
        model="fireworks:accounts/fireworks/models/glm-5p2",
        tools=[get_user_info],
        checkpointer=InMemorySaver(),  # [!code highlight]
    )

    thread_config = {"configurable": {"thread_id": "1"}}
    response = agent.invoke(
        {"messages": [{"role": "user", "content": "Hi! My name is Bob."}]},
        thread_config,  # [!code highlight]
    )["messages"][-1].content

    print(response)  # "Hi Bob! Nice to see you here. How are you doing?"

    response = agent.invoke(
        {"messages": [{"role": "user", "content": "What's my name?"}]},
        thread_config,  # [!code highlight]
    )["messages"][-1].content

    print(response)  # "You are Bob!"
    ```

=== "Baseten"
    ```python
    from langchain.agents import create_agent
    from langgraph.checkpoint.memory import InMemorySaver  # [!code highlight]


    def get_user_info() -> str:
        """Look up information about the current user."""
        return "No user profile on file."


    agent = create_agent(
        model="baseten:zai-org/GLM-5.2",
        tools=[get_user_info],
        checkpointer=InMemorySaver(),  # [!code highlight]
    )

    thread_config = {"configurable": {"thread_id": "1"}}
    response = agent.invoke(
        {"messages": [{"role": "user", "content": "Hi! My name is Bob."}]},
        thread_config,  # [!code highlight]
    )["messages"][-1].content

    print(response)  # "Hi Bob! Nice to see you here. How are you doing?"

    response = agent.invoke(
        {"messages": [{"role": "user", "content": "What's my name?"}]},
        thread_config,  # [!code highlight]
    )["messages"][-1].content

    print(response)  # "You are Bob!"
    ```

=== "Ollama"
    ```python
    from langchain.agents import create_agent
    from langgraph.checkpoint.memory import InMemorySaver  # [!code highlight]


    def get_user_info() -> str:
        """Look up information about the current user."""
        return "No user profile on file."


    agent = create_agent(
        model="ollama:north-mini-code-1.0",
        tools=[get_user_info],
        checkpointer=InMemorySaver(),  # [!code highlight]
    )

    thread_config = {"configurable": {"thread_id": "1"}}
    response = agent.invoke(
        {"messages": [{"role": "user", "content": "Hi! My name is Bob."}]},
        thread_config,  # [!code highlight]
    )["messages"][-1].content

    print(response)  # "Hi Bob! Nice to see you here. How are you doing?"

    response = agent.invoke(
        {"messages": [{"role": "user", "content": "What's my name?"}]},
        thread_config,  # [!code highlight]
    )["messages"][-1].content

    print(response)  # "You are Bob!"
    ```

### Trong môi trường production

Trong môi trường production, hãy dùng một checkpointer được lưu trên một cơ sở dữ liệu:

=== "pip"
    ```bash
    pip install -U langgraph-checkpoint-postgres "psycopg[binary]"
    ```

=== "uv"
    ```bash
    uv add langgraph-checkpoint-postgres "psycopg[binary]"
    ```

!!! note "Ghi chú"
    Mặc định, `langgraph-checkpoint-postgres` cài `psycopg` (Psycopg 3) mà không kèm extra nào. Lệnh cài ở trên thêm `psycopg[binary]`, được khuyến nghị cho hầu hết người dùng. Với các tuỳ chọn khác, xem [tài liệu cài đặt Psycopg](https://www.psycopg.org/psycopg3/docs/basic/install.html).

```python
from langchain.agents import create_agent
from langgraph.checkpoint.postgres import PostgresSaver  # [!code highlight]

def get_user_info() -> str:
    """Look up information about the current user."""
    return "No user profile on file."

DB_URI = "postgresql://postgres:postgres@localhost:5432/postgres?sslmode=disable"
with PostgresSaver.from_conn_string(DB_URI) as checkpointer:
    checkpointer.setup() # auto create tables in PostgreSQL
    agent = create_agent(
        "gpt-5.5",
        tools=[get_user_info],
        checkpointer=checkpointer,  # [!code highlight]
    )
```

!!! note "Ghi chú"
    Để biết thêm các tuỳ chọn checkpointer khác bao gồm SQLite, Postgres, và Azure Cosmos DB, xem [danh sách các thư viện checkpointer](https://docs.langchain.com/oss/python/langgraph/checkpointers#checkpointer-libraries) trong tài liệu về Persistence.

## Tuỳ chỉnh bộ nhớ agent

Mặc định, agent dùng [`AgentState`](https://reference.langchain.com/python/langchain/agents/middleware/types/AgentState) để quản lý bộ nhớ ngắn hạn, cụ thể là lịch sử hội thoại thông qua key `messages`.

Bạn có thể mở rộng [`AgentState`](https://reference.langchain.com/python/langchain/agents/middleware/types/AgentState) để thêm các field khác. Custom state schema được truyền vào [`create_agent`](https://reference.langchain.com/python/langchain/agents/factory/create_agent) bằng tham số [`state_schema`](https://reference.langchain.com/python/langchain/middleware/#langchain.agents.middleware.AgentMiddleware.state_schema).

```python
from langchain.agents import create_agent, AgentState
from langgraph.checkpoint.memory import InMemorySaver


class CustomAgentState(AgentState):  # [!code highlight]
    user_id: str  # [!code highlight]
    preferences: dict  # [!code highlight]

agent = create_agent(
    "gpt-5.5",
    tools=[get_user_info],
    state_schema=CustomAgentState,  # [!code highlight]
    checkpointer=InMemorySaver(),
)

# Custom state can be passed in invoke
result = agent.invoke(
    {
        "messages": [{"role": "user", "content": "Hello"}],
        "user_id": "user_123",  # [!code highlight]
        "preferences": {"theme": "dark"}  # [!code highlight]
    },
    {"configurable": {"thread_id": "1"}})
```

## Các pattern phổ biến

Khi [bộ nhớ ngắn hạn](#cach-su-dung) được bật, các cuộc hội thoại dài có thể vượt quá context window của LLM. Các giải pháp phổ biến là:

**Trim messages** ([xem bên dưới](#trim-messages))
Loại bỏ N message đầu tiên hoặc cuối cùng (trước khi gọi LLM)

**Delete messages** ([xem bên dưới](#delete-messages))
Xoá vĩnh viễn message khỏi state của LangGraph

**Summarize messages** ([xem bên dưới](#summarize-messages))
Tóm tắt các message trước đó trong lịch sử và thay thế chúng bằng một bản tóm tắt

**Custom strategies**
Các chiến lược tuỳ chỉnh (ví dụ: lọc message, v.v.)

Điều này cho phép agent theo dõi cuộc hội thoại mà không vượt quá context window của LLM.

### Trim messages

Hầu hết LLM đều có context window tối đa được hỗ trợ (tính bằng token).

Một cách để quyết định khi nào nên cắt bớt (truncate) message là đếm số token trong lịch sử message và cắt bớt bất cứ khi nào nó tiến gần đến giới hạn đó. Nếu bạn đang dùng LangChain, bạn có thể dùng tiện ích trim messages và chỉ định số token cần giữ lại từ danh sách, cũng như `strategy` (ví dụ: giữ lại `max_tokens` cuối cùng) để xử lý ranh giới.

Để trim lịch sử message trong một agent, dùng decorator middleware [`@before_model`](https://reference.langchain.com/python/langchain/agents/middleware/types/before_model):

```python
from langchain.messages import RemoveMessage
from langgraph.graph.message import REMOVE_ALL_MESSAGES
from langgraph.checkpoint.memory import InMemorySaver
from langchain.agents import create_agent, AgentState
from langchain.agents.middleware import before_model
from langgraph.runtime import Runtime
from langchain_core.runnables import RunnableConfig
from typing import Any


@before_model
def trim_messages(state: AgentState, runtime: Runtime) -> dict[str, Any] | None:
    """Keep only the last few messages to fit context window."""
    messages = state["messages"]

    if len(messages) <= 3:
        return None  # No changes needed

    first_msg = messages[0]
    recent_messages = messages[-3:] if len(messages) % 2 == 0 else messages[-4:]
    new_messages = [first_msg] + recent_messages

    return {
        "messages": [
            RemoveMessage(id=REMOVE_ALL_MESSAGES),
            *new_messages
        ]
    }

agent = create_agent(
    "gpt-5.5",
    tools=[...],
    middleware=[trim_messages],
    checkpointer=InMemorySaver(),
)

config: RunnableConfig = {"configurable": {"thread_id": "1"}}

agent.invoke({"messages": "hi, my name is bob"}, config)
agent.invoke({"messages": "write a short poem about cats"}, config)
agent.invoke({"messages": "now do the same but for dogs"}, config)
final_response = agent.invoke({"messages": "what's my name?"}, config)

final_response["messages"][-1].pretty_print()
"""
================================== Ai Message ==================================

Your name is Bob. You told me that earlier.
If you'd like me to call you a nickname or use a different name, just say the word.
"""
```

### Delete messages

Bạn có thể xoá message khỏi state của graph để quản lý lịch sử message.

Điều này hữu ích khi bạn muốn loại bỏ các message cụ thể hoặc xoá toàn bộ lịch sử message.

Để xoá message khỏi state của graph, bạn có thể dùng `RemoveMessage`.

Để `RemoveMessage` hoạt động, bạn cần dùng một state key với [reducer](https://docs.langchain.com/oss/python/langgraph/graph-api#reducers) [`add_messages`](https://reference.langchain.com/python/langgraph/graph/message/add_messages).

`AgentState` mặc định ([`AgentState`](https://reference.langchain.com/python/langchain/agents/middleware/types/AgentState)) đã cung cấp sẵn điều này.

Để xoá các message cụ thể:

```python
from langchain.messages import RemoveMessage  # [!code highlight]

def delete_messages(state):
    messages = state["messages"]
    if len(messages) > 2:
        # remove the earliest two messages
        return {"messages": [RemoveMessage(id=m.id) for m in messages[:2]]}  # [!code highlight]
```

Để xoá **tất cả** message:

```python
from langgraph.graph.message import REMOVE_ALL_MESSAGES  # [!code highlight]

def delete_messages(state):
    return {"messages": [RemoveMessage(id=REMOVE_ALL_MESSAGES)]}  # [!code highlight]
```

!!! warning "Cảnh báo"
    Khi xoá message, **hãy đảm bảo** rằng lịch sử message kết quả vẫn hợp lệ. Kiểm tra các giới hạn của nhà cung cấp LLM bạn đang dùng. Ví dụ:

    * Một số nhà cung cấp yêu cầu lịch sử message phải bắt đầu bằng một message `user`
    * Hầu hết nhà cung cấp yêu cầu message `assistant` có tool call phải được theo sau bởi các message kết quả `tool` tương ứng.

```python
from langchain.messages import RemoveMessage
from langchain.agents import create_agent, AgentState
from langchain.agents.middleware import after_model
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.runtime import Runtime
from langchain_core.runnables import RunnableConfig


@after_model
def delete_old_messages(state: AgentState, runtime: Runtime) -> dict | None:
    """Remove old messages to keep conversation manageable."""
    messages = state["messages"]
    if len(messages) > 2:
        # remove the earliest two messages
        return {"messages": [RemoveMessage(id=m.id) for m in messages[:2]]}
    return None


agent = create_agent(
    "gpt-5-nano",
    tools=[...],
    system_prompt="Please be concise and to the point.",
    middleware=[delete_old_messages],
    checkpointer=InMemorySaver(),
)

config: RunnableConfig = {"configurable": {"thread_id": "1"}}

stream = agent.stream_events(
    {"messages": [{"role": "user", "content": "hi! I'm bob"}]},
    config,
    version="v3",
)
for snapshot in stream.values:
    print([(message.type, message.content) for message in snapshot["messages"]])

stream = agent.stream_events(
    {"messages": [{"role": "user", "content": "write a short poem about cats"}]},
    config,
    version="v3",
)
for snapshot in stream.values:
    print([(message.type, message.content) for message in snapshot["messages"]])

stream = agent.stream_events(
    {"messages": [{"role": "user", "content": "what's my name?"}]},
    config,
    version="v3",
)
for snapshot in stream.values:
    print([(message.type, message.content) for message in snapshot["messages"]])
```

```
[('human', "hi! I'm bob")]
[('human', "hi! I'm bob"), ('ai', 'Hi Bob! Nice to meet you. How can I help you today? I can answer questions, brainstorm ideas, draft text, explain things, or help with code.')]
[('human', "hi! I'm bob"), ('ai', 'Hi Bob! Nice to meet you. How can I help you today? I can answer questions, brainstorm ideas, draft text, explain things, or help with code.'), ('human', "write a short poem about cats")]
[('human', "hi! I'm bob"), ('ai', 'Hi Bob! Nice to meet you. How can I help you today? I can answer questions, brainstorm ideas, draft text, explain things, or help with code.'), ('human', "write a short poem about cats"), ('ai', 'There once was a cat on a wall, Who barely moved at all...')]
[('human', 'write a short poem about cats'), ('ai', 'There once was a cat on a wall, Who barely moved at all...')]
[('human', 'write a short poem about cats'), ('ai', 'There once was a cat on a wall, Who barely moved at all...'), ('human', "what's my name?")]
[('human', 'write a short poem about cats'), ('ai', 'There once was a cat on a wall, Who barely moved at all...'), ('human', "what's my name?"), ('ai', "I don't know your name - you haven't told me!")]
[('human', "what's my name?"), ('ai', "I don't know your name - you haven't told me!")]
```

### Summarize messages

Vấn đề với việc trim hoặc xoá message, như đã trình bày ở trên, là bạn có thể mất thông tin do việc cắt bỏ hàng đợi message.
Vì vậy, một số ứng dụng sẽ hưởng lợi từ cách tiếp cận tinh vi hơn: tóm tắt lịch sử message bằng một chat model.

<img src="https://mintcdn.com/langchain-5e9cc07a/ybiAaBfoBvFquMDz/oss/images/summary.png?fit=max&auto=format&n=ybiAaBfoBvFquMDz&q=85&s=c8ed3facdccd4ef5c7e52902c72ba938" alt="Summary" width="609" height="242" />

Để tóm tắt lịch sử message trong một agent, dùng [`SummarizationMiddleware`](https://docs.langchain.com/oss/python/langchain/middleware#summarization) có sẵn:

```python
from langchain.agents import create_agent
from langchain.agents.middleware import SummarizationMiddleware
from langgraph.checkpoint.memory import InMemorySaver
from langchain_core.runnables import RunnableConfig


checkpointer = InMemorySaver()

agent = create_agent(
    model="gpt-5.5",
    tools=[...],
    middleware=[
        SummarizationMiddleware(
            model="gpt-5.4-mini",
            trigger=("tokens", 4000),
            keep=("messages", 20)
        )
    ],
    checkpointer=checkpointer,
)

config: RunnableConfig = {"configurable": {"thread_id": "1"}}
agent.invoke({"messages": "hi, my name is bob"}, config)
agent.invoke({"messages": "write a short poem about cats"}, config)
agent.invoke({"messages": "now do the same but for dogs"}, config)
final_response = agent.invoke({"messages": "what's my name?"}, config)

final_response["messages"][-1].pretty_print()
"""
================================== Ai Message ==================================

Your name is Bob!
"""
```

Xem [`SummarizationMiddleware`](https://docs.langchain.com/oss/python/langchain/middleware#summarization) để biết thêm các tuỳ chọn cấu hình.

## Truy cập bộ nhớ

Bạn có thể truy cập và chỉnh sửa bộ nhớ ngắn hạn (state) của một agent theo nhiều cách:

### Tools

#### Đọc bộ nhớ ngắn hạn trong một tool

Truy cập bộ nhớ ngắn hạn (state) trong một tool bằng tham số `runtime` (có kiểu `ToolRuntime`).

Tham số `runtime` bị ẩn khỏi chữ ký (signature) của tool (nên model không thấy nó), nhưng tool có thể truy cập state thông qua nó.

```python
from langchain.agents import create_agent, AgentState
from langchain.tools import tool, ToolRuntime


class CustomState(AgentState):
    user_id: str

@tool
def get_user_info(
    runtime: ToolRuntime
) -> str:
    """Look up user info."""
    user_id = runtime.state["user_id"]
    return "User is John Smith" if user_id == "user_123" else "Unknown user"

agent = create_agent(
    model="gpt-5-nano",
    tools=[get_user_info],
    state_schema=CustomState,
)

result = agent.invoke({
    "messages": "look up user information",
    "user_id": "user_123"
})
print(result["messages"][-1].content)
# > User is John Smith.
```

#### Ghi bộ nhớ ngắn hạn từ tools

Để chỉnh sửa bộ nhớ ngắn hạn (state) của agent trong khi thực thi, bạn có thể trả về các cập nhật state trực tiếp từ tool.

Điều này hữu ích để lưu lại các kết quả trung gian hoặc làm cho thông tin có thể truy cập được bởi các tool hoặc prompt tiếp theo.

```python
from langchain.tools import tool, ToolRuntime
from langchain_core.runnables import RunnableConfig
from langchain.messages import ToolMessage
from langchain.agents import create_agent, AgentState
from langgraph.types import Command
from pydantic import BaseModel


class CustomState(AgentState):  # [!code highlight]
    user_name: str

class CustomContext(BaseModel):
    user_id: str

@tool
def update_user_info(
    runtime: ToolRuntime[CustomContext, CustomState],
) -> Command:
    """Look up and update user info."""
    user_id = runtime.context.user_id
    name = "John Smith" if user_id == "user_123" else "Unknown user"
    return Command(update={  # [!code highlight]
        "user_name": name,
        # update the message history
        "messages": [
            ToolMessage(
                "Successfully looked up user information",
                tool_call_id=runtime.tool_call_id
            )
        ]
    })

@tool
def greet(
    runtime: ToolRuntime[CustomContext, CustomState]
) -> str | Command:
    """Use this to greet the user once you found their info."""
    user_name = runtime.state.get("user_name", None)
    if user_name is None:
       return Command(update={
            "messages": [
                ToolMessage(
                    "Please call the 'update_user_info' tool it will get and update the user's name.",
                    tool_call_id=runtime.tool_call_id
                )
            ]
        })
    return f"Hello {user_name}!"

agent = create_agent(
    model="gpt-5-nano",
    tools=[update_user_info, greet],
    state_schema=CustomState, # [!code highlight]
    context_schema=CustomContext,
)

agent.invoke(
    {"messages": [{"role": "user", "content": "greet the user"}]},
    context=CustomContext(user_id="user_123"),
)
```

### Prompt

Truy cập bộ nhớ ngắn hạn (state) trong middleware để tạo prompt động dựa trên lịch sử hội thoại hoặc các custom state field.

```python
from langchain.agents import create_agent
from typing import TypedDict
from langchain.agents.middleware import dynamic_prompt, ModelRequest


class CustomContext(TypedDict):
    user_name: str


def get_weather(city: str) -> str:
    """Get the weather in a city."""
    return f"The weather in {city} is always sunny!"


@dynamic_prompt
def dynamic_system_prompt(request: ModelRequest) -> str:
    user_name = request.runtime.context["user_name"]
    system_prompt = f"You are a helpful assistant. Address the user as {user_name}."
    return system_prompt


agent = create_agent(
    model="gpt-5-nano",
    tools=[get_weather],
    middleware=[dynamic_system_prompt],
    context_schema=CustomContext,
)

result = agent.invoke(
    {"messages": [{"role": "user", "content": "What is the weather in SF?"}]},
    context=CustomContext(user_name="John Smith"),
)
for msg in result["messages"]:
    msg.pretty_print()

```

Kết quả:

```shell
================================ Human Message =================================

What is the weather in SF?
================================== Ai Message ==================================
Tool Calls:
  get_weather (call_WFQlOGn4b2yoJrv7cih342FG)
 Call ID: call_WFQlOGn4b2yoJrv7cih342FG
  Args:
    city: San Francisco
================================= Tool Message =================================
Name: get_weather

The weather in San Francisco is always sunny!
================================== Ai Message ==================================

Hi John Smith, the weather in San Francisco is always sunny!
```

### Before model

Truy cập bộ nhớ ngắn hạn (state) trong middleware [`@before_model`](https://reference.langchain.com/python/langchain/agents/middleware/types/before_model) để xử lý message trước khi gọi model.

```mermaid
%%{
    init: {
        "fontFamily": "monospace",
        "flowchart": {
        "curve": "basis"
        }
    }
}%%
graph TD
    S(["\_\_start\_\_"])
    PRE(before_model)
    MODEL(model)
    TOOLS(tools)
    END(["\_\_end\_\_"])
    S --> PRE
    PRE --> MODEL
    MODEL -.-> TOOLS
    MODEL -.-> END
    TOOLS --> PRE
    classDef blueHighlight fill:#E5F4FF,stroke:#006DDD,color:#030710;
    classDef neutral fill:#F2FAFF,stroke:#40668D,stroke-width:2px,color:#2F4B68;
    class S blueHighlight;
    class END blueHighlight;
    class PRE,MODEL,TOOLS neutral;
```

```python
from langchain.messages import RemoveMessage
from langgraph.graph.message import REMOVE_ALL_MESSAGES
from langgraph.checkpoint.memory import InMemorySaver
from langchain.agents import create_agent, AgentState
from langchain.agents.middleware import before_model
from langchain_core.runnables import RunnableConfig
from langgraph.runtime import Runtime
from typing import Any


@before_model
def trim_messages(state: AgentState, runtime: Runtime) -> dict[str, Any] | None:
    """Keep only the last few messages to fit context window."""
    messages = state["messages"]

    if len(messages) <= 3:
        return None  # No changes needed

    first_msg = messages[0]
    recent_messages = messages[-3:] if len(messages) % 2 == 0 else messages[-4:]
    new_messages = [first_msg] + recent_messages

    return {
        "messages": [
            RemoveMessage(id=REMOVE_ALL_MESSAGES),
            *new_messages
        ]
    }


agent = create_agent(
    "gpt-5-nano",
    tools=[],
    middleware=[trim_messages],
    checkpointer=InMemorySaver()
)

config: RunnableConfig = {"configurable": {"thread_id": "1"}}

agent.invoke({"messages": "hi, my name is bob"}, config)
agent.invoke({"messages": "write a short poem about cats"}, config)
agent.invoke({"messages": "now do the same but for dogs"}, config)
final_response = agent.invoke({"messages": "what's my name?"}, config)

final_response["messages"][-1].pretty_print()
"""
================================== Ai Message ==================================

Your name is Bob. You told me that earlier.
If you'd like me to call you a nickname or use a different name, just say the word.
"""
```

### After model

Truy cập bộ nhớ ngắn hạn (state) trong middleware [`@after_model`](https://reference.langchain.com/python/langchain/agents/middleware/types/after_model) để xử lý message sau khi gọi model.

```mermaid
%%{
    init: {
        "fontFamily": "monospace",
        "flowchart": {
        "curve": "basis"
        }
    }
}%%
graph TD
    S(["\_\_start\_\_"])
    MODEL(model)
    POST(after_model)
    TOOLS(tools)
    END(["\_\_end\_\_"])
    S --> MODEL
    MODEL --> POST
    POST -.-> END
    POST -.-> TOOLS
    TOOLS --> MODEL
    classDef blueHighlight fill:#E5F4FF,stroke:#006DDD,color:#030710;
    classDef greenHighlight fill:#F6FFDB,stroke:#6E8900,color:#2E3900;
    classDef neutral fill:#F2FAFF,stroke:#40668D,stroke-width:2px,color:#2F4B68;
    class S blueHighlight;
    class END blueHighlight;
    class POST greenHighlight;
    class MODEL,TOOLS neutral;
```

```python
from langchain.messages import RemoveMessage
from langgraph.checkpoint.memory import InMemorySaver
from langchain.agents import create_agent, AgentState
from langchain.agents.middleware import after_model
from langgraph.runtime import Runtime


@after_model
def validate_response(state: AgentState, runtime: Runtime) -> dict | None:
    """Remove messages containing sensitive words."""
    STOP_WORDS = ["password", "secret"]
    last_message = state["messages"][-1]
    if any(word in last_message.content for word in STOP_WORDS):
        return {"messages": [RemoveMessage(id=last_message.id)]}
    return None

agent = create_agent(
    model="gpt-5-nano",
    tools=[],
    middleware=[validate_response],
    checkpointer=InMemorySaver(),
)
```
