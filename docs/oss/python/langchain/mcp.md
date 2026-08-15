# Model Context Protocol (MCP)

[Model Context Protocol (MCP)](https://modelcontextprotocol.io/introduction) là một giao thức mở giúp chuẩn hóa cách các ứng dụng cung cấp tool và context cho LLM. Các agent của LangChain có thể sử dụng các tool được định nghĩa trên MCP server thông qua thư viện [`langchain-mcp-adapters`](https://github.com/langchain-ai/langchain-mcp-adapters).

## Bắt đầu nhanh

Cài đặt thư viện `langchain-mcp-adapters`:

=== "pip"

    ```bash
    pip install langchain-mcp-adapters
    ```

=== "uv"

    ```bash
    uv add langchain-mcp-adapters
    ```

`langchain-mcp-adapters` cho phép agent sử dụng các tool được định nghĩa trên một hoặc nhiều MCP server.

!!! note "Ghi chú"
    `MultiServerMCPClient` **stateless theo mặc định**. Mỗi lần gọi tool sẽ tạo một `ClientSession` MCP mới, thực thi tool, rồi dọn dẹp. Xem phần [phiên làm việc có trạng thái](#stateful-sessions) bên dưới để biết thêm chi tiết.

```python title="Truy cập nhiều MCP server"
import asyncio
from langchain_mcp_adapters.client import MultiServerMCPClient  # [!code highlight]
from langchain.agents import create_agent

async def main():
    client = MultiServerMCPClient(  # [!code highlight]
        {
            "math": {
                "transport": "stdio",  # Local subprocess communication
                "command": "python",
                # Absolute path to your math_server.py file
                "args": ["/path/to/math_server.py"],
            },
            "weather": {
                "transport": "http",  # HTTP-based remote server
                # Ensure you start your weather server on port 8000
                "url": "http://localhost:8000/mcp",
            }
        }
    )

    tools = await client.get_tools()  # [!code highlight]
    agent = create_agent(
        "claude-sonnet-4-6",
        tools  # [!code highlight]
    )
    math_response = await agent.ainvoke(
        {"messages": [{"role": "user", "content": "what's (3 + 5) x 12?"}]}
    )
    weather_response = await agent.ainvoke(
        {"messages": [{"role": "user", "content": "what is the weather in nyc?"}]}
    )
    print(math_response)
    print(weather_response)

if __name__ == "__main__":
    asyncio.run(main())
```

!!! tip "Mẹo"
    Trace các lệnh gọi MCP tool song song với các bước suy luận (reasoning) của agent bằng [LangSmith](https://smith.langchain.com?utm_source=docs\&utm_medium=cta\&utm_campaign=langsmith-signup\&utm_content=oss-langchain-mcp). Làm theo [hướng dẫn bắt đầu nhanh về tracing](https://docs.langchain.com/langsmith/trace-with-langchain) để thiết lập.

## Máy chủ tùy chỉnh

Để tạo một MCP server tùy chỉnh, hãy dùng thư viện [FastMCP](https://gofastmcp.com/getting-started/welcome):

=== "pip"

    ```bash
    pip install fastmcp
    ```

=== "uv"

    ```bash
    uv add fastmcp
    ```

Để kiểm thử agent của bạn với các MCP tool server, hãy dùng các ví dụ sau:

=== "Math server (stdio transport)"

    ```python
    from fastmcp import FastMCP

    mcp = FastMCP("Math")

    @mcp.tool()
    def add(a: int, b: int) -> int:
        """Add two numbers"""
        return a + b

    @mcp.tool()
    def multiply(a: int, b: int) -> int:
        """Multiply two numbers"""
        return a * b

    if __name__ == "__main__":
        mcp.run(transport="stdio")
    ```

=== "Weather server (streamable HTTP transport)"

    ```python
    from fastmcp import FastMCP

    mcp = FastMCP("Weather")

    @mcp.tool()
    async def get_weather(location: str) -> str:
        """Get weather for location."""
        return "It's always sunny in New York"

    if __name__ == "__main__":
        mcp.run(transport="streamable-http")
    ```

## Các cơ chế transport

MCP hỗ trợ nhiều cơ chế transport khác nhau cho việc giao tiếp giữa client và server.

### HTTP

Transport `http` (còn gọi là `streamable-http`) dùng các HTTP request để giao tiếp giữa client và server. Xem [đặc tả HTTP transport của MCP](https://modelcontextprotocol.io/specification/2025-03-26/basic/transports#streamable-http) để biết thêm chi tiết.

Dùng một URL cục bộ cho các server bạn tự chạy, hoặc một URL được host sẵn như [MCP server của tài liệu LangChain](https://docs.langchain.com/use-these-docs) (`https://docs.langchain.com/mcp`), là server công khai và không yêu cầu API key.

```python
from langchain.agents import create_agent
from langchain_mcp_adapters.client import MultiServerMCPClient

client = MultiServerMCPClient(
    {
        "mcp": {
            "transport": "http",
            # "url": "http://localhost:8000/mcp",  # Local server
            "url": "https://docs.langchain.com/mcp",  # Hosted server
        }
    }
)
tools = await client.get_tools()
agent = create_agent("openai:gpt-5.4", tools)
response = await agent.ainvoke(
    {
        "messages": [
            {
                "role": "user",
                "content": "How do I connect LangChain to an MCP server over HTTP?",
            }
        ]
    }
)
```

#### Truyền header

Khi kết nối tới MCP server qua HTTP, bạn có thể thêm các header tùy chỉnh (ví dụ: để xác thực hoặc tracing) bằng trường `headers` trong cấu hình kết nối. Điều này được hỗ trợ cho các transport `sse` (đã bị MCP spec đánh dấu deprecated) và `streamable_http`.

```python title="Truyền header với MultiServerMCPClient"
from langchain_mcp_adapters.client import MultiServerMCPClient
from langchain.agents import create_agent

client = MultiServerMCPClient(
    {
        "weather": {
            "transport": "http",
            "url": "http://localhost:8000/mcp",
            "headers": {  # [!code highlight]
                "Authorization": "Bearer YOUR_TOKEN",  # [!code highlight]
                "X-Custom-Header": "custom-value"  # [!code highlight]
            },  # [!code highlight]
        }
    }
)
tools = await client.get_tools()
agent = create_agent("openai:gpt-5.5", tools)
response = await agent.ainvoke({"messages": "what is the weather in nyc?"})
```

#### Xác thực

Thư viện `langchain-mcp-adapters` sử dụng [MCP SDK](https://github.com/modelcontextprotocol/python-sdk) chính thức bên dưới, cho phép bạn cung cấp một cơ chế xác thực tùy chỉnh bằng cách triển khai interface `httpx.Auth`.

```python
from langchain_mcp_adapters.client import MultiServerMCPClient

client = MultiServerMCPClient(
    {
        "weather": {
            "transport": "http",
            "url": "http://localhost:8000/mcp",
            "auth": auth, # [!code highlight]
        }
    }
)
```

* [Ví dụ triển khai xác thực tùy chỉnh](https://github.com/modelcontextprotocol/python-sdk/blob/main/examples/clients/simple-auth-client/mcp_simple_auth_client/main.py)
* [Luồng OAuth có sẵn](https://github.com/modelcontextprotocol/python-sdk/blob/main/src/mcp/client/auth/oauth2.py#L216)

### stdio

Client khởi chạy server dưới dạng subprocess và giao tiếp qua standard input/output. Phù hợp nhất với các tool cục bộ và các thiết lập đơn giản.

!!! note "Ghi chú"
    Khác với các transport HTTP, kết nối `stdio` vốn dĩ **có trạng thái (stateful)**: subprocess tồn tại trong suốt vòng đời của kết nối client. Tuy nhiên, khi dùng `MultiServerMCPClient` mà không quản lý session một cách tường minh, mỗi lệnh gọi tool vẫn sẽ tạo một session mới. Xem [phiên làm việc có trạng thái](#stateful-sessions) để biết cách quản lý các kết nối bền vững.

```python
client = MultiServerMCPClient(
    {
        "math": {
            "transport": "stdio",
            "command": "python",
            "args": ["/path/to/math_server.py"],
        }
    }
)
```

## Phiên làm việc có trạng thái {#stateful-sessions}

Theo mặc định, `MultiServerMCPClient` là **stateless**: mỗi lần gọi tool sẽ tạo một session MCP mới, thực thi tool, rồi dọn dẹp.

Nếu bạn cần kiểm soát [vòng đời (lifecycle)](https://modelcontextprotocol.io/specification/2025-03-26/basic/lifecycle) của một MCP session (ví dụ: khi làm việc với một server có trạng thái duy trì context xuyên suốt các lệnh gọi tool), bạn có thể tạo một `ClientSession` bền vững bằng `client.session()`.

```python title="Dùng MCP ClientSession để gọi tool có trạng thái"
from langchain_mcp_adapters.client import MultiServerMCPClient
from langchain_mcp_adapters.tools import load_mcp_tools
from langchain.agents import create_agent

client = MultiServerMCPClient({...})

# Create a session explicitly
async with client.session("server_name") as session:  # [!code highlight]
    # Pass the session to load tools, resources, or prompts
    tools = await load_mcp_tools(session)  # [!code highlight]
    agent = create_agent(
        "google_genai:gemini-3.6-flash",
        tools
    )
```

## Các tính năng cốt lõi

### Tools

[Tools](https://modelcontextprotocol.io/docs/concepts/tools) cho phép MCP server công bố các hàm có thể thực thi mà LLM có thể gọi để thực hiện hành động, chẳng hạn như truy vấn database, gọi API, hoặc tương tác với các hệ thống bên ngoài. LangChain chuyển đổi MCP tools thành [tools](https://docs.langchain.com/oss/python/langchain/tools) của LangChain, giúp chúng có thể dùng trực tiếp trong bất kỳ agent hay workflow nào của LangChain.

#### Nạp tools

Dùng `client.get_tools()` để lấy các tool từ MCP server và truyền chúng vào agent của bạn:

```python
from langchain_mcp_adapters.client import MultiServerMCPClient
from langchain.agents import create_agent

client = MultiServerMCPClient({...})
tools = await client.get_tools()  # [!code highlight]
agent = create_agent("claude-sonnet-4-6", tools)
```

Theo mặc định, khi một MCP tool gặp lỗi, lỗi đó được trả về cho model dưới dạng một tool message có `status="error"` thay vì raise exception. Điều này cho phép agent đọc lỗi và thử lại. Để raise exception thay vào đó, đặt `handle_tool_errors=False` trên `MultiServerMCPClient` hoặc `load_mcp_tools`.

Điều này chỉ áp dụng cho lỗi thực thi tool (`CallToolResult(isError=True)`). Các lỗi transport, session, và chuyển đổi nội dung (content-conversion) luôn raise exception.

!!! note "Ghi chú"
    Việc trả lỗi MCP tool dưới dạng tool message thất bại yêu cầu `langchain-mcp-adapters>=0.3.0`. Các phiên bản trước đó sẽ raise `ToolException`.

#### Nội dung có cấu trúc (Structured content)

MCP tools có thể trả về [structured content](https://modelcontextprotocol.io/specification/2025-03-26/server/tools#structured-content) song song với phần phản hồi văn bản dễ đọc. Điều này hữu ích khi một tool cần trả về dữ liệu có thể phân tích được bằng máy (ví dụ JSON) bên cạnh phần văn bản hiển thị cho model.

Khi một MCP tool trả về `structuredContent`, adapter sẽ bọc nó trong một [`MCPToolArtifact`](https://reference.langchain.com/python/langchain_mcp_adapters/#langchain_mcp_adapters.tools.MCPToolArtifact) và trả về dưới dạng artifact của tool. Bạn có thể truy cập nó qua trường `artifact` trên `ToolMessage`. Bạn cũng có thể dùng [interceptor](#tool-interceptors) để tự động xử lý hoặc biến đổi structured content.

**Trích xuất structured content từ artifact**

Sau khi gọi agent, bạn có thể truy cập structured content từ tool message trong kết quả trả về:

```python
from langchain_mcp_adapters.client import MultiServerMCPClient
from langchain.agents import create_agent
from langchain.messages import ToolMessage

client = MultiServerMCPClient({...})
tools = await client.get_tools()
agent = create_agent("claude-sonnet-4-6", tools)

result = await agent.ainvoke(
    {"messages": [{"role": "user", "content": "Get data from the server"}]}
)

# Extract structured content from tool messages
for message in result["messages"]:
    if isinstance(message, ToolMessage) and message.artifact:
        structured_content = message.artifact["structured_content"]
```

**Nối thêm structured content thông qua interceptor**

Nếu bạn muốn structured content hiển thị trong lịch sử hội thoại (để model nhìn thấy), bạn có thể dùng một [interceptor](#tool-interceptors) để tự động nối thêm structured content vào kết quả của tool:

```python
import json

from langchain_mcp_adapters.client import MultiServerMCPClient
from langchain_mcp_adapters.interceptors import MCPToolCallRequest
from mcp.types import TextContent

async def append_structured_content(request: MCPToolCallRequest, handler):
    """Append structured content from artifact to tool message."""
    result = await handler(request)
    if result.structuredContent:
        result.content += [
            TextContent(type="text", text=json.dumps(result.structuredContent)),
        ]
    return result

client = MultiServerMCPClient({...}, tool_interceptors=[append_structured_content])
```

#### Nội dung tool đa phương thức (Multimodal)

MCP tools có thể trả về [nội dung đa phương thức](https://modelcontextprotocol.io/specification/2025-03-26/server/tools#tool-result) (hình ảnh, văn bản, v.v.) trong phản hồi của chúng. Khi một MCP server trả về nội dung gồm nhiều phần (ví dụ: văn bản và hình ảnh), adapter sẽ chuyển đổi chúng thành [standard content blocks](https://docs.langchain.com/oss/python/langchain/messages#standard-content-blocks) của LangChain. Bạn có thể truy cập biểu diễn chuẩn hóa này qua thuộc tính `content_blocks` trên `ToolMessage`:

```python
from langchain.agents import create_agent
from langchain_mcp_adapters.client import MultiServerMCPClient

async def access_multimodal_tool_content():
    client = MultiServerMCPClient({})
    tools = await client.get_tools()
    agent = create_agent("claude-sonnet-4-6", tools)

    result = await agent.ainvoke(
        {"messages": [{"role": "user", "content": "Take a screenshot of the current page"}]}
    )

    # Access multimodal content from tool messages
    for message in result["messages"]:
        if message.type == "tool":
            # Raw content in provider-native format
            print(f"Raw content: {message.content}")

            # Standardized content blocks  # [!code highlight]
            for block in message.content_blocks:  # [!code highlight]
                if block["type"] == "text":  # [!code highlight]
                    print(f"Text: {block['text']}")  # [!code highlight]
                elif block["type"] == "image":  # [!code highlight]
                    print(f"Image URL: {block.get('url')}")  # [!code highlight]
                    print(f"Image base64: {block.get('base64', '')[:50]}...")  # [!code highlight]
```

Điều này cho phép bạn xử lý phản hồi tool đa phương thức theo cách không phụ thuộc vào nhà cung cấp (provider-agnostic), bất kể MCP server bên dưới định dạng nội dung của nó như thế nào.

### Resources

[Resources](https://modelcontextprotocol.io/docs/concepts/resources) cho phép MCP server công bố dữ liệu, chẳng hạn như file, bản ghi database, hoặc phản hồi API, để client có thể đọc. LangChain chuyển đổi MCP resources thành các đối tượng [Blob](https://reference.langchain.com/python/langchain_core/documents/#langchain_core.documents.base.Blob), cung cấp một giao diện thống nhất để xử lý cả nội dung dạng văn bản lẫn nhị phân.

#### Nạp resources

Dùng `client.get_resources()` để nạp resources từ một MCP server:

```python
from langchain_mcp_adapters.client import MultiServerMCPClient

client = MultiServerMCPClient({...})

# Load all resources from a server
blobs = await client.get_resources("server_name")  # [!code highlight]

# Or load specific resources by URI
blobs = await client.get_resources("server_name", uris=["file:///path/to/file.txt"])  # [!code highlight]

for blob in blobs:
    print(f"URI: {blob.metadata['uri']}, MIME type: {blob.mimetype}")
    print(blob.as_string())  # For text content
```

Bạn cũng có thể dùng trực tiếp [`load_mcp_resources`](https://reference.langchain.com/python/langchain_mcp_adapters/#langchain_mcp_adapters.resources.load_mcp_resources) với một session để có nhiều quyền kiểm soát hơn:

```python
from langchain_mcp_adapters.client import MultiServerMCPClient
from langchain_mcp_adapters.resources import load_mcp_resources

client = MultiServerMCPClient({...})

async with client.session("server_name") as session:
    # Load all resources
    blobs = await load_mcp_resources(session)

    # Or load specific resources by URI
    blobs = await load_mcp_resources(session, uris=["file:///path/to/file.txt"])
```

### Prompts

[Prompts](https://modelcontextprotocol.io/docs/concepts/prompts) cho phép MCP server công bố các prompt template có thể tái sử dụng, để client lấy về và sử dụng. LangChain chuyển đổi MCP prompts thành [messages](https://docs.langchain.com/oss/python/langchain/messages), giúp chúng dễ dàng tích hợp vào các workflow dạng chat.

#### Nạp prompts

Dùng `client.get_prompt()` để nạp một prompt từ MCP server:

```python
from langchain_mcp_adapters.client import MultiServerMCPClient

client = MultiServerMCPClient({...})

# Load a prompt by name
messages = await client.get_prompt("server_name", "summarize")  # [!code highlight]

# Load a prompt with arguments
messages = await client.get_prompt(  # [!code highlight]
    "server_name",  # [!code highlight]
    "code_review",  # [!code highlight]
    arguments={"language": "python", "focus": "security"}  # [!code highlight]
)  # [!code highlight]

# Use the messages in your workflow
for message in messages:
    print(f"{message.type}: {message.content}")
```

Bạn cũng có thể dùng trực tiếp [`load_mcp_prompt`](https://reference.langchain.com/python/langchain_mcp_adapters/#langchain_mcp_adapters.prompts.load_mcp_prompt) với một session để có nhiều quyền kiểm soát hơn:

```python
from langchain_mcp_adapters.client import MultiServerMCPClient
from langchain_mcp_adapters.prompts import load_mcp_prompt

client = MultiServerMCPClient({...})

async with client.session("server_name") as session:
    # Load a prompt by name
    messages = await load_mcp_prompt(session, "summarize")

    # Load a prompt with arguments
    messages = await load_mcp_prompt(
        session,
        "code_review",
        arguments={"language": "python", "focus": "security"}
    )
```

## Các tính năng nâng cao

### Tool interceptors {#tool-interceptors}

MCP server chạy như các process riêng biệt, chúng không thể truy cập thông tin runtime của LangGraph như [store](https://docs.langchain.com/oss/python/langgraph/stores), [context](https://docs.langchain.com/oss/python/langchain/context-engineering), hay agent state. **Interceptor lấp đầy khoảng trống này** bằng cách cho bạn quyền truy cập vào runtime context trong lúc MCP tool đang thực thi.

Interceptor cũng cung cấp khả năng kiểm soát lệnh gọi tool tương tự middleware: bạn có thể sửa đổi request, triển khai retry, thêm header động, hoặc short-circuit việc thực thi hoàn toàn.

| Phần | Mô tả |
| --- | --- |
| [Truy cập runtime context](#accessing-runtime-context) | Đọc user ID, API key, dữ liệu store, và agent state |
| [Cập nhật state và command](#state-updates-and-commands) | Cập nhật agent state hoặc điều khiển luồng graph bằng `Command` |
| [Viết interceptor](#custom-interceptors) | Các pattern để sửa đổi request, kết hợp nhiều interceptor, và xử lý lỗi |

#### Truy cập runtime context {#accessing-runtime-context}

Khi MCP tools được dùng bên trong một agent của LangChain (qua `create_agent`), interceptor sẽ nhận được quyền truy cập vào context `ToolRuntime`. Điều này cung cấp quyền truy cập vào tool call ID, state, config, và store, cho phép các pattern mạnh mẽ để truy cập dữ liệu người dùng, lưu trữ thông tin, và điều khiển hành vi của agent.

=== "Runtime context"

    Truy cập cấu hình theo từng người dùng như user ID, API key, hoặc quyền hạn được truyền vào tại thời điểm gọi:

    ```python title="Chèn user context vào các lệnh gọi MCP tool"
    from dataclasses import dataclass
    from langchain_mcp_adapters.client import MultiServerMCPClient
    from langchain_mcp_adapters.interceptors import MCPToolCallRequest
    from langchain.agents import create_agent

    @dataclass
    class Context:
        user_id: str
        api_key: str

    async def inject_user_context(
        request: MCPToolCallRequest,
        handler,
    ):
        """Inject user credentials into MCP tool calls."""
        runtime = request.runtime
        user_id = runtime.context.user_id  # [!code highlight]
        api_key = runtime.context.api_key  # [!code highlight]

        # Add user context to tool arguments
        modified_request = request.override(
            args={**request.args, "user_id": user_id}
        )
        return await handler(modified_request)

    client = MultiServerMCPClient(
        {...},
        tool_interceptors=[inject_user_context],
    )
    tools = await client.get_tools()
    agent = create_agent("gpt-5.5", tools, context_schema=Context)

    # Invoke with user context
    result = await agent.ainvoke(
        {"messages": [{"role": "user", "content": "Search my orders"}]},
        context={"user_id": "user_123", "api_key": "sk-..."}
    )
    ```

=== "Store"

    Truy cập bộ nhớ dài hạn (long-term memory) để lấy lại tùy chọn người dùng hoặc lưu trữ dữ liệu xuyên suốt các cuộc hội thoại:

    ```python title="Truy cập tùy chọn người dùng từ store"
    from dataclasses import dataclass
    from langchain_mcp_adapters.client import MultiServerMCPClient
    from langchain_mcp_adapters.interceptors import MCPToolCallRequest
    from langchain.agents import create_agent
    from langgraph.store.memory import InMemoryStore

    @dataclass
    class Context:
        user_id: str

    async def personalize_search(
        request: MCPToolCallRequest,
        handler,
    ):
        """Personalize MCP tool calls using stored preferences."""
        runtime = request.runtime
        user_id = runtime.context.user_id
        store = runtime.store  # [!code highlight]

        # Read user preferences from store
        prefs = store.get(("preferences",), user_id)  # [!code highlight]

        if prefs and request.name == "search":
            # Apply user's preferred language and result limit
            modified_args = {
                **request.args,
                "language": prefs.value.get("language", "en"),
                "limit": prefs.value.get("result_limit", 10),
            }
            request = request.override(args=modified_args)

        return await handler(request)

    client = MultiServerMCPClient(
        {...},
        tool_interceptors=[personalize_search],
    )
    tools = await client.get_tools()
    agent = create_agent(
        "gpt-5.5",
        tools,
        context_schema=Context,
        store=InMemoryStore()
    )
    ```

=== "State"

    Truy cập conversation state để đưa ra quyết định dựa trên phiên làm việc hiện tại:

    ```python title="Lọc tool dựa trên trạng thái xác thực"
    from langchain_mcp_adapters.client import MultiServerMCPClient
    from langchain_mcp_adapters.interceptors import MCPToolCallRequest
    from langchain.messages import ToolMessage

    async def require_authentication(
        request: MCPToolCallRequest,
        handler,
    ):
        """Block sensitive MCP tools if user is not authenticated."""
        runtime = request.runtime
        state = runtime.state  # [!code highlight]
        is_authenticated = state.get("authenticated", False)  # [!code highlight]

        sensitive_tools = ["delete_file", "update_settings", "export_data"]

        if request.name in sensitive_tools and not is_authenticated:
            # Return error instead of calling tool
            return ToolMessage(
                content="Authentication required. Please log in first.",
                tool_call_id=runtime.tool_call_id,
            )

        return await handler(request)

    client = MultiServerMCPClient(
        {...},
        tool_interceptors=[require_authentication],
    )
    ```

=== "Tool call ID"

    Truy cập tool call ID để trả về phản hồi đúng định dạng hoặc theo dõi các lần thực thi tool:

    ```python title="Trả về phản hồi tùy chỉnh cùng tool call ID"
    from langchain_mcp_adapters.client import MultiServerMCPClient
    from langchain_mcp_adapters.interceptors import MCPToolCallRequest
    from langchain.messages import ToolMessage

    async def rate_limit_interceptor(
        request: MCPToolCallRequest,
        handler,
    ):
        """Rate limit expensive MCP tool calls."""
        runtime = request.runtime
        tool_call_id = runtime.tool_call_id  # [!code highlight]

        # Check rate limit (simplified example)
        if is_rate_limited(request.name):
            return ToolMessage(
                content="Rate limit exceeded. Please try again later.",
                tool_call_id=tool_call_id,  # [!code highlight]
            )

        result = await handler(request)

        # Log successful tool call
        log_tool_execution(tool_call_id, request.name, success=True)

        return result

    client = MultiServerMCPClient(
        {...},
        tool_interceptors=[rate_limit_interceptor],
    )
    ```

Để biết thêm các pattern về context engineering, xem [Context engineering](https://docs.langchain.com/oss/python/langchain/context-engineering) và [Tools](https://docs.langchain.com/oss/python/langchain/tools).

#### Cập nhật state và command {#state-updates-and-commands}

Interceptor có thể trả về đối tượng `Command` để cập nhật agent state hoặc điều khiển luồng thực thi của graph. Điều này hữu ích để theo dõi tiến độ tác vụ, chuyển đổi giữa các agent, hoặc kết thúc việc thực thi sớm.

```python title="Đánh dấu task hoàn tất và chuyển sang agent khác"
from langchain.agents import AgentState, create_agent
from langchain_mcp_adapters.interceptors import MCPToolCallRequest
from langchain.messages import ToolMessage
from langgraph.types import Command

async def handle_task_completion(
    request: MCPToolCallRequest,
    handler,
):
    """Mark task complete and hand off to summary agent."""
    result = await handler(request)

    if request.name == "submit_order":
        return Command(
            update={
                "messages": [result] if isinstance(result, ToolMessage) else [],
                "task_status": "completed",  # [!code highlight]
            },
            goto="summary_agent",  # [!code highlight]
        )

    return result
```

Dùng `Command` với `goto="__end__"` để kết thúc việc thực thi sớm:

```python title="Kết thúc agent run khi hoàn tất"
async def end_on_success(
    request: MCPToolCallRequest,
    handler,
):
    """End agent run when task is marked complete."""
    result = await handler(request)

    if request.name == "mark_complete":
        return Command(
            update={"messages": [result], "status": "done"},
            goto="__end__",  # [!code highlight]
        )

    return result
```

#### Interceptor tùy chỉnh {#custom-interceptors}

Interceptor là các hàm async bọc quanh việc thực thi tool, cho phép sửa đổi request/response, triển khai logic retry, và các mối quan tâm xuyên suốt (cross-cutting concerns) khác. Chúng tuân theo pattern "onion" (củ hành), trong đó interceptor đầu tiên trong danh sách là lớp ngoài cùng.

**Pattern cơ bản**

Một interceptor là một hàm async nhận vào một request và một handler. Bạn có thể sửa đổi request trước khi gọi handler, sửa đổi response sau đó, hoặc bỏ qua handler hoàn toàn.

```python title="Pattern interceptor cơ bản"
from langchain_mcp_adapters.client import MultiServerMCPClient
from langchain_mcp_adapters.interceptors import MCPToolCallRequest

async def logging_interceptor(
    request: MCPToolCallRequest,
    handler,
):
    """Log tool calls before and after execution."""
    print(f"Calling tool: {request.name} with args: {request.args}")
    result = await handler(request)
    print(f"Tool {request.name} returned: {result}")
    return result

client = MultiServerMCPClient(
    {"math": {"transport": "stdio", "command": "python", "args": ["/path/to/server.py"]}},
    tool_interceptors=[logging_interceptor],  # [!code highlight]
)
```

**Sửa đổi request**

Dùng `request.override()` để tạo một request đã sửa đổi. Cách này tuân theo pattern bất biến (immutable), giữ nguyên request gốc.

```python title="Sửa đổi tham số của tool"
async def double_args_interceptor(
    request: MCPToolCallRequest,
    handler,
):
    """Double all numeric arguments before execution."""
    modified_args = {k: v * 2 for k, v in request.args.items()}
    modified_request = request.override(args=modified_args)  # [!code highlight]
    return await handler(modified_request)

# Original call: add(a=2, b=3) becomes add(a=4, b=6)
```

**Sửa đổi header tại thời điểm chạy**

Interceptor có thể sửa đổi HTTP header một cách động dựa trên ngữ cảnh của request:

```python title="Sửa đổi header động"
async def auth_header_interceptor(
    request: MCPToolCallRequest,
    handler,
):
    """Add authentication headers based on the tool being called."""
    token = get_token_for_tool(request.name)
    modified_request = request.override(
        headers={"Authorization": f"Bearer {token}"}  # [!code highlight]
    )
    return await handler(modified_request)
```

**Kết hợp nhiều interceptor**

Nhiều interceptor kết hợp với nhau theo thứ tự "onion": interceptor đầu tiên trong danh sách là lớp ngoài cùng:

```python title="Kết hợp nhiều interceptor"
async def outer_interceptor(request, handler):
    print("outer: before")
    result = await handler(request)
    print("outer: after")
    return result

async def inner_interceptor(request, handler):
    print("inner: before")
    result = await handler(request)
    print("inner: after")
    return result

client = MultiServerMCPClient(
    {...},
    tool_interceptors=[outer_interceptor, inner_interceptor],  # [!code highlight]
)

# Execution order:
# outer: before -> inner: before -> tool execution -> inner: after -> outer: after
```

**Xử lý lỗi**

Dùng interceptor để bắt các exception từ việc thực thi tool, chẳng hạn lỗi transport hoặc runtime, và thêm logic retry. Các lỗi thực thi tool (`CallToolResult(isError=True)`) không raise theo mặc định, vì vậy các interceptor bắt exception sẽ không bao giờ được kích hoạt với các lỗi này. Để bắt được chúng dưới dạng exception ở đây, hãy đặt `handle_tool_errors=False`.

```python title="Retry khi có lỗi"
import asyncio

async def retry_interceptor(
    request: MCPToolCallRequest,
    handler,
    max_retries: int = 3,
    delay: float = 1.0,
):
    """Retry failed tool calls with exponential backoff."""
    last_error = None
    for attempt in range(max_retries):
        try:
            return await handler(request)
        except Exception as e:
            last_error = e
            if attempt < max_retries - 1:
                wait_time = delay * (2 ** attempt)  # Exponential backoff
                print(f"Tool {request.name} failed (attempt {attempt + 1}), retrying in {wait_time}s...")
                await asyncio.sleep(wait_time)
    raise last_error

client = MultiServerMCPClient(
    {...},
    tool_interceptors=[retry_interceptor],  # [!code highlight]
)
```

Bạn cũng có thể bắt các loại lỗi cụ thể và trả về giá trị fallback:

```python title="Xử lý lỗi kèm fallback"
async def fallback_interceptor(
    request: MCPToolCallRequest,
    handler,
):
    """Return a fallback value if tool execution fails."""
    try:
        return await handler(request)
    except TimeoutError:
        return f"Tool {request.name} timed out. Please try again later."
    except ConnectionError:
        return f"Could not connect to {request.name} service. Using cached data."
```

### Thông báo tiến độ (Progress notifications)

Đăng ký nhận cập nhật tiến độ cho các lượt thực thi tool chạy lâu:

```python title="Callback tiến độ"
from langchain_mcp_adapters.client import MultiServerMCPClient
from langchain_mcp_adapters.callbacks import Callbacks, CallbackContext

async def on_progress(
    progress: float,
    total: float | None,
    message: str | None,
    context: CallbackContext,
):
    """Handle progress updates from MCP servers."""
    percent = (progress / total * 100) if total else progress
    tool_info = f" ({context.tool_name})" if context.tool_name else ""
    print(f"[{context.server_name}{tool_info}] Progress: {percent:.1f}% - {message}")

client = MultiServerMCPClient(
    {...},
    callbacks=Callbacks(on_progress=on_progress),  # [!code highlight]
)
```

`CallbackContext` cung cấp:

* `server_name`: Tên của MCP server
* `tool_name`: Tên của tool đang được thực thi (chỉ có sẵn trong lúc gọi tool)

### Logging

Giao thức MCP hỗ trợ các thông báo [logging](https://modelcontextprotocol.io/specification/2025-03-26/server/utilities/logging#log-levels) từ server. Dùng class `Callbacks` để đăng ký nhận các sự kiện này.

```python title="Callback logging"
from langchain_mcp_adapters.client import MultiServerMCPClient
from langchain_mcp_adapters.callbacks import Callbacks, CallbackContext
from mcp.types import LoggingMessageNotificationParams

async def on_logging_message(
    params: LoggingMessageNotificationParams,
    context: CallbackContext,
):
    """Handle log messages from MCP servers."""
    print(f"[{context.server_name}] {params.level}: {params.data}")

client = MultiServerMCPClient(
    {...},
    callbacks=Callbacks(on_logging_message=on_logging_message),  # [!code highlight]
)
```

### Elicitation

[Elicitation](https://modelcontextprotocol.io/specification/2025-11-25/client/elicitation#elicitation) cho phép MCP server yêu cầu người dùng cung cấp thêm input trong lúc thực thi tool. Thay vì phải yêu cầu toàn bộ input ngay từ đầu, server có thể hỏi thêm thông tin một cách tương tác khi cần.

#### Thiết lập server

Định nghĩa một tool dùng `ctx.elicit()` để yêu cầu input từ người dùng theo một schema:

```python title="MCP server có elicitation"
from pydantic import BaseModel
from mcp.server.fastmcp import Context, FastMCP

server = FastMCP("Profile")

class UserDetails(BaseModel):
    email: str
    age: int

@server.tool()
async def create_profile(name: str, ctx: Context) -> str:
    """Create a user profile, requesting details via elicitation."""
    result = await ctx.elicit(  # [!code highlight]
        message=f"Please provide details for {name}'s profile:",  # [!code highlight]
        schema=UserDetails,  # [!code highlight]
    )  # [!code highlight]
    if result.action == "accept" and result.data:
        return f"Created profile for {name}: email={result.data.email}, age={result.data.age}"
    if result.action == "decline":
        return f"User declined. Created minimal profile for {name}."
    return "Profile creation cancelled."

if __name__ == "__main__":
    server.run(transport="http")
```

#### Thiết lập client

Xử lý các yêu cầu elicitation bằng cách cung cấp một callback cho `MultiServerMCPClient`:

```python title="Xử lý yêu cầu elicitation"
from langchain_mcp_adapters.client import MultiServerMCPClient
from langchain_mcp_adapters.callbacks import Callbacks, CallbackContext
from mcp.shared.context import RequestContext
from mcp.types import ElicitRequestParams, ElicitResult

async def on_elicitation(
    mcp_context: RequestContext,
    params: ElicitRequestParams,
    context: CallbackContext,
) -> ElicitResult:
    """Handle elicitation requests from MCP servers."""
    # In a real application, you would prompt the user for input
    # based on params.message and params.requestedSchema
    return ElicitResult(  # [!code highlight]
        action="accept",  # [!code highlight]
        content={"email": "user@example.com", "age": 25},  # [!code highlight]
    )  # [!code highlight]

client = MultiServerMCPClient(
    {
        "profile": {
            "url": "http://localhost:8000/mcp",
            "transport": "http",
        }
    },
    callbacks=Callbacks(on_elicitation=on_elicitation),  # [!code highlight]
)
```

#### Các hành động phản hồi

Callback elicitation có thể trả về một trong ba hành động sau:

| Hành động | Mô tả |
| --- | --- |
| `accept` | Người dùng đã cung cấp input hợp lệ. Đưa dữ liệu vào trường `content`. |
| `decline` | Người dùng chọn không cung cấp thông tin được yêu cầu. |
| `cancel` | Người dùng đã hủy hoàn toàn thao tác. |

```python title="Ví dụ các hành động phản hồi"
# Accept with data
ElicitResult(action="accept", content={"email": "user@example.com", "age": 25})

# Decline (user doesn't want to provide info)
ElicitResult(action="decline")

# Cancel (abort the operation)
ElicitResult(action="cancel")
```

## Tài nguyên bổ sung

* [Tài liệu MCP](https://modelcontextprotocol.io/introduction)
* [Tài liệu MCP Transport](https://modelcontextprotocol.io/docs/concepts/transports)
* [`langchain-mcp-adapters`](https://github.com/langchain-ai/langchain-mcp-adapters)

---

[Kết nối các tài liệu này](https://docs.langchain.com/use-these-docs) với Claude, VSCode và nhiều công cụ khác qua MCP để nhận câu trả lời theo thời gian thực.

[Chỉnh sửa trang này trên GitHub](https://github.com/langchain-ai/docs/edit/main/src/oss/langchain/mcp.mdx) hoặc [báo lỗi](https://github.com/langchain-ai/docs/issues/new/choose).
