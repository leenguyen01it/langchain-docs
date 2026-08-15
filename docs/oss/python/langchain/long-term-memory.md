# Bộ nhớ dài hạn

> Thêm bộ nhớ dài hạn vào các agent LangChain để lưu trữ và truy xuất dữ liệu qua các cuộc hội thoại và phiên làm việc

Bộ nhớ dài hạn cho phép agent của bạn lưu trữ và truy xuất thông tin qua các cuộc hội thoại và phiên làm việc khác nhau.
Khác với [bộ nhớ ngắn hạn](short-term-memory.md), vốn chỉ giới hạn trong một thread, bộ nhớ dài hạn tồn tại xuyên suốt các thread và có thể được truy xuất bất cứ lúc nào.

Bộ nhớ dài hạn được xây dựng trên [LangGraph stores](https://docs.langchain.com/oss/python/langgraph/stores), lưu dữ liệu dưới dạng tài liệu JSON được tổ chức theo namespace và key.

## Cách sử dụng

Để thêm bộ nhớ dài hạn vào một agent, hãy tạo một store và truyền nó vào [`create_agent`](https://reference.langchain.com/python/langchain/agents/factory/create_agent):

=== "InMemoryStore"

    ```python
    from langchain.agents import create_agent
    from langchain_core.runnables import Runnable
    from langgraph.store.memory import InMemoryStore

    # InMemoryStore saves data to an in-memory dictionary. Use a DB-backed store in production use.
    store = InMemoryStore()

    agent: Runnable = create_agent(
        "claude-sonnet-4-6",
        tools=[],
        store=store,
    )
    ```

=== "PostgreSQL"

    === "pip"

        ```bash
        pip install -U langgraph-checkpoint-postgres "psycopg[binary]"
        ```

    === "uv"

        ```bash
        uv add langgraph-checkpoint-postgres "psycopg[binary]"
        ```

    !!! note "Ghi chú"
        Theo mặc định, `langgraph-checkpoint-postgres` cài đặt `psycopg` (Psycopg 3) mà không kèm extras. Lệnh cài đặt ở trên thêm `psycopg[binary]`, được khuyến nghị cho hầu hết người dùng. Để biết thêm các tùy chọn khác, xem [tài liệu cài đặt Psycopg](https://www.psycopg.org/psycopg3/docs/basic/install.html).

    ```python
    from langchain.agents import create_agent
    from langchain_core.runnables import Runnable
    from langgraph.store.postgres import PostgresStore  # type: ignore[import-not-found]

    DB_URI = "postgresql://postgres:postgres@localhost:5432/postgres?sslmode=disable"

    with PostgresStore.from_conn_string(DB_URI) as store:
        store.setup()
        agent: Runnable = create_agent(
            "claude-sonnet-4-6",
            tools=[],
            store=store,
        )
    ```

Sau đó, các tool có thể đọc và ghi vào store thông qua tham số `runtime.store`. Xem [Đọc bộ nhớ dài hạn trong tool](#doc-bo-nho-dai-han-trong-tool) và [Ghi bộ nhớ dài hạn từ tool](#write-long-term) để xem ví dụ.

!!! tip "Mẹo"
    Để tìm hiểu sâu hơn về các loại bộ nhớ (semantic, episodic, procedural) và các chiến lược ghi bộ nhớ, xem [hướng dẫn khái niệm về Memory](https://docs.langchain.com/oss/python/concepts/memory#long-term-memory).

## Lưu trữ bộ nhớ

LangGraph lưu trữ các bộ nhớ dài hạn dưới dạng tài liệu JSON trong một [store](https://docs.langchain.com/oss/python/langgraph/stores).

Mỗi bộ nhớ được tổ chức theo một `namespace` tùy chỉnh (tương tự như một thư mục) và một `key` riêng biệt (giống như tên file). Namespace thường bao gồm user ID, org ID hoặc các nhãn khác giúp việc tổ chức thông tin dễ dàng hơn.

Cấu trúc này cho phép tổ chức bộ nhớ theo dạng phân cấp. Việc tìm kiếm xuyên namespace sau đó được hỗ trợ thông qua các bộ lọc nội dung (content filters).

=== "InMemoryStore"

    ```python
    from collections.abc import Sequence

    from langgraph.store.base import IndexConfig
    from langgraph.store.memory import InMemoryStore


    def embed(texts: Sequence[str]) -> list[list[float]]:
        # Replace with an actual embedding function or LangChain embeddings object
        return [[1.0, 2.0] for _ in texts]


    # InMemoryStore saves data to an in-memory dictionary. Use a DB-backed store in production use.
    store = InMemoryStore(index=IndexConfig(embed=embed, dims=2))
    user_id = "my-user"
    application_context = "chitchat"
    namespace = (user_id, application_context)
    store.put(
        namespace,
        "a-memory",
        {
            "rules": [
                "User likes short, direct language",
                "User only speaks English & python",
            ],
            "my-key": "my-value",
        },
    )
    # get the "memory" by ID
    item = store.get(namespace, "a-memory")
    # search for "memories" within this namespace, filtering on content equivalence, sorted by vector similarity
    items = store.search(
        namespace, filter={"my-key": "my-value"}, query="language preferences"
    )
    ```

=== "PostgreSQL"

    ```python
    from collections.abc import Sequence

    from langgraph.store.base import IndexConfig
    from langgraph.store.postgres import PostgresStore  # type: ignore[import-not-found]


    def embed(texts: Sequence[str]) -> list[list[float]]:
        # Replace with an actual embedding function or LangChain embeddings object
        return [[1.0, 2.0] for _ in texts]


    DB_URI = "postgresql://postgres:postgres@localhost:5432/postgres?sslmode=disable"

    with PostgresStore.from_conn_string(
        DB_URI,
        index=IndexConfig(embed=embed, dims=2),  # type: ignore[arg-type]
    ) as store:
        store.setup()
        user_id = "my-user"
        application_context = "chitchat"
        namespace = (user_id, application_context)
        store.put(
            namespace,
            "a-memory",
            {
                "rules": [
                    "User likes short, direct language",
                    "User only speaks English & python",
                ],
                "my-key": "my-value",
            },
        )
        item = store.get(namespace, "a-memory")
        items = store.search(
            namespace, filter={"my-key": "my-value"}, query="language preferences"
        )
    ```

Để biết thêm thông tin về memory store, xem hướng dẫn [Persistence](https://docs.langchain.com/oss/python/langgraph/stores).

## Đọc bộ nhớ dài hạn trong tool

=== "InMemoryStore"

    === "Google"

        ```python
        from dataclasses import dataclass

        from langchain.agents import create_agent
        from langchain.tools import ToolRuntime, tool
        from langchain_core.runnables import Runnable
        from langgraph.store.memory import InMemoryStore


        @dataclass
        class Context:
            user_id: str


        # InMemoryStore saves data to an in-memory dictionary. Use a DB-backed store in production.
        store = InMemoryStore()

        # Write sample data to the store using the put method
        store.put(
            (
                "users",
            ),  # Namespace to group related data together (users namespace for user data)
            "user_123",  # Key within the namespace (user ID as key)
            {
                "name": "John Smith",
                "language": "English",
            },  # Data to store for the given user
        )


        @tool
        def get_user_info(runtime: ToolRuntime[Context]) -> str:
            """Look up user info."""
            # Access the store - same as that provided to `create_agent`
            assert runtime.store is not None
            user_id = runtime.context.user_id
            # Retrieve data from store - returns StoreValue object with value and metadata
            user_info = runtime.store.get(("users",), user_id)
            return str(user_info.value) if user_info else "Unknown user"


        agent: Runnable = create_agent(
            model="google_genai:gemini-3.6-flash",
            tools=[get_user_info],
            # Pass store to agent - enables agent to access store when running tools
            store=store,
            context_schema=Context,
        )

        # Run the agent
        agent.invoke(
            {"messages": [{"role": "user", "content": "look up user information"}]},
            context=Context(user_id="user_123"),
        )
        ```

    === "OpenAI"

        ```python
        from dataclasses import dataclass

        from langchain.agents import create_agent
        from langchain.tools import ToolRuntime, tool
        from langchain_core.runnables import Runnable
        from langgraph.store.memory import InMemoryStore


        @dataclass
        class Context:
            user_id: str


        # InMemoryStore saves data to an in-memory dictionary. Use a DB-backed store in production.
        store = InMemoryStore()

        # Write sample data to the store using the put method
        store.put(
            (
                "users",
            ),  # Namespace to group related data together (users namespace for user data)
            "user_123",  # Key within the namespace (user ID as key)
            {
                "name": "John Smith",
                "language": "English",
            },  # Data to store for the given user
        )


        @tool
        def get_user_info(runtime: ToolRuntime[Context]) -> str:
            """Look up user info."""
            # Access the store - same as that provided to `create_agent`
            assert runtime.store is not None
            user_id = runtime.context.user_id
            # Retrieve data from store - returns StoreValue object with value and metadata
            user_info = runtime.store.get(("users",), user_id)
            return str(user_info.value) if user_info else "Unknown user"


        agent: Runnable = create_agent(
            model="openai:gpt-5.5",
            tools=[get_user_info],
            # Pass store to agent - enables agent to access store when running tools
            store=store,
            context_schema=Context,
        )

        # Run the agent
        agent.invoke(
            {"messages": [{"role": "user", "content": "look up user information"}]},
            context=Context(user_id="user_123"),
        )
        ```

    === "Anthropic"

        ```python
        from dataclasses import dataclass

        from langchain.agents import create_agent
        from langchain.tools import ToolRuntime, tool
        from langchain_core.runnables import Runnable
        from langgraph.store.memory import InMemoryStore


        @dataclass
        class Context:
            user_id: str


        # InMemoryStore saves data to an in-memory dictionary. Use a DB-backed store in production.
        store = InMemoryStore()

        # Write sample data to the store using the put method
        store.put(
            (
                "users",
            ),  # Namespace to group related data together (users namespace for user data)
            "user_123",  # Key within the namespace (user ID as key)
            {
                "name": "John Smith",
                "language": "English",
            },  # Data to store for the given user
        )


        @tool
        def get_user_info(runtime: ToolRuntime[Context]) -> str:
            """Look up user info."""
            # Access the store - same as that provided to `create_agent`
            assert runtime.store is not None
            user_id = runtime.context.user_id
            # Retrieve data from store - returns StoreValue object with value and metadata
            user_info = runtime.store.get(("users",), user_id)
            return str(user_info.value) if user_info else "Unknown user"


        agent: Runnable = create_agent(
            model="anthropic:claude-sonnet-4-6",
            tools=[get_user_info],
            # Pass store to agent - enables agent to access store when running tools
            store=store,
            context_schema=Context,
        )

        # Run the agent
        agent.invoke(
            {"messages": [{"role": "user", "content": "look up user information"}]},
            context=Context(user_id="user_123"),
        )
        ```

    === "OpenRouter"

        ```python
        from dataclasses import dataclass

        from langchain.agents import create_agent
        from langchain.tools import ToolRuntime, tool
        from langchain_core.runnables import Runnable
        from langgraph.store.memory import InMemoryStore


        @dataclass
        class Context:
            user_id: str


        # InMemoryStore saves data to an in-memory dictionary. Use a DB-backed store in production.
        store = InMemoryStore()

        # Write sample data to the store using the put method
        store.put(
            (
                "users",
            ),  # Namespace to group related data together (users namespace for user data)
            "user_123",  # Key within the namespace (user ID as key)
            {
                "name": "John Smith",
                "language": "English",
            },  # Data to store for the given user
        )


        @tool
        def get_user_info(runtime: ToolRuntime[Context]) -> str:
            """Look up user info."""
            # Access the store - same as that provided to `create_agent`
            assert runtime.store is not None
            user_id = runtime.context.user_id
            # Retrieve data from store - returns StoreValue object with value and metadata
            user_info = runtime.store.get(("users",), user_id)
            return str(user_info.value) if user_info else "Unknown user"


        agent: Runnable = create_agent(
            model="openrouter:z-ai/glm-5.2",
            tools=[get_user_info],
            # Pass store to agent - enables agent to access store when running tools
            store=store,
            context_schema=Context,
        )

        # Run the agent
        agent.invoke(
            {"messages": [{"role": "user", "content": "look up user information"}]},
            context=Context(user_id="user_123"),
        )
        ```

    === "Fireworks"

        ```python
        from dataclasses import dataclass

        from langchain.agents import create_agent
        from langchain.tools import ToolRuntime, tool
        from langchain_core.runnables import Runnable
        from langgraph.store.memory import InMemoryStore


        @dataclass
        class Context:
            user_id: str


        # InMemoryStore saves data to an in-memory dictionary. Use a DB-backed store in production.
        store = InMemoryStore()

        # Write sample data to the store using the put method
        store.put(
            (
                "users",
            ),  # Namespace to group related data together (users namespace for user data)
            "user_123",  # Key within the namespace (user ID as key)
            {
                "name": "John Smith",
                "language": "English",
            },  # Data to store for the given user
        )


        @tool
        def get_user_info(runtime: ToolRuntime[Context]) -> str:
            """Look up user info."""
            # Access the store - same as that provided to `create_agent`
            assert runtime.store is not None
            user_id = runtime.context.user_id
            # Retrieve data from store - returns StoreValue object with value and metadata
            user_info = runtime.store.get(("users",), user_id)
            return str(user_info.value) if user_info else "Unknown user"


        agent: Runnable = create_agent(
            model="fireworks:accounts/fireworks/models/glm-5p2",
            tools=[get_user_info],
            # Pass store to agent - enables agent to access store when running tools
            store=store,
            context_schema=Context,
        )

        # Run the agent
        agent.invoke(
            {"messages": [{"role": "user", "content": "look up user information"}]},
            context=Context(user_id="user_123"),
        )
        ```

    === "Baseten"

        ```python
        from dataclasses import dataclass

        from langchain.agents import create_agent
        from langchain.tools import ToolRuntime, tool
        from langchain_core.runnables import Runnable
        from langgraph.store.memory import InMemoryStore


        @dataclass
        class Context:
            user_id: str


        # InMemoryStore saves data to an in-memory dictionary. Use a DB-backed store in production.
        store = InMemoryStore()

        # Write sample data to the store using the put method
        store.put(
            (
                "users",
            ),  # Namespace to group related data together (users namespace for user data)
            "user_123",  # Key within the namespace (user ID as key)
            {
                "name": "John Smith",
                "language": "English",
            },  # Data to store for the given user
        )


        @tool
        def get_user_info(runtime: ToolRuntime[Context]) -> str:
            """Look up user info."""
            # Access the store - same as that provided to `create_agent`
            assert runtime.store is not None
            user_id = runtime.context.user_id
            # Retrieve data from store - returns StoreValue object with value and metadata
            user_info = runtime.store.get(("users",), user_id)
            return str(user_info.value) if user_info else "Unknown user"


        agent: Runnable = create_agent(
            model="baseten:zai-org/GLM-5.2",
            tools=[get_user_info],
            # Pass store to agent - enables agent to access store when running tools
            store=store,
            context_schema=Context,
        )

        # Run the agent
        agent.invoke(
            {"messages": [{"role": "user", "content": "look up user information"}]},
            context=Context(user_id="user_123"),
        )
        ```

    === "Ollama"

        ```python
        from dataclasses import dataclass

        from langchain.agents import create_agent
        from langchain.tools import ToolRuntime, tool
        from langchain_core.runnables import Runnable
        from langgraph.store.memory import InMemoryStore


        @dataclass
        class Context:
            user_id: str


        # InMemoryStore saves data to an in-memory dictionary. Use a DB-backed store in production.
        store = InMemoryStore()

        # Write sample data to the store using the put method
        store.put(
            (
                "users",
            ),  # Namespace to group related data together (users namespace for user data)
            "user_123",  # Key within the namespace (user ID as key)
            {
                "name": "John Smith",
                "language": "English",
            },  # Data to store for the given user
        )


        @tool
        def get_user_info(runtime: ToolRuntime[Context]) -> str:
            """Look up user info."""
            # Access the store - same as that provided to `create_agent`
            assert runtime.store is not None
            user_id = runtime.context.user_id
            # Retrieve data from store - returns StoreValue object with value and metadata
            user_info = runtime.store.get(("users",), user_id)
            return str(user_info.value) if user_info else "Unknown user"


        agent: Runnable = create_agent(
            model="ollama:north-mini-code-1.0",
            tools=[get_user_info],
            # Pass store to agent - enables agent to access store when running tools
            store=store,
            context_schema=Context,
        )

        # Run the agent
        agent.invoke(
            {"messages": [{"role": "user", "content": "look up user information"}]},
            context=Context(user_id="user_123"),
        )
        ```

=== "PostgreSQL"

    ```python
    from dataclasses import dataclass

    from langchain.agents import create_agent
    from langchain.tools import ToolRuntime, tool
    from langchain_core.runnables import Runnable
    from langgraph.store.postgres import PostgresStore  # type: ignore[import-not-found]


    @dataclass
    class Context:
        user_id: str


    DB_URI = "postgresql://postgres:postgres@localhost:5432/postgres?sslmode=disable"

    with PostgresStore.from_conn_string(DB_URI) as store:
        store.setup()
        store.put(("users",), "user_123", {"name": "John Smith", "language": "English"})

        @tool
        def get_user_info(runtime: ToolRuntime[Context]) -> str:
            """Look up user info."""
            assert runtime.store is not None
            user_info = runtime.store.get(("users",), runtime.context.user_id)
            return str(user_info.value) if user_info else "Unknown user"

        agent: Runnable = create_agent(
            "claude-sonnet-4-6",
            tools=[get_user_info],
            store=store,
            context_schema=Context,
        )

        result = agent.invoke(
            {"messages": [{"role": "user", "content": "look up user information"}]},
            context=Context(user_id="user_123"),
        )
    ```

<a id="write-long-term" />

## Ghi bộ nhớ dài hạn từ tool

=== "InMemoryStore"

    === "Google"

        ```python
        from dataclasses import dataclass

        from langchain.agents import create_agent
        from langchain.tools import ToolRuntime, tool
        from langchain_core.runnables import Runnable
        from langgraph.store.memory import InMemoryStore
        from typing_extensions import TypedDict

        # InMemoryStore saves data to an in-memory dictionary. Use a DB-backed store in production.
        store = InMemoryStore()


        @dataclass
        class Context:
            user_id: str


        # TypedDict defines the structure of user information for the LLM
        class UserInfo(TypedDict):
            name: str


        # Tool that allows agent to update user information (useful for chat applications)
        @tool
        def save_user_info(user_info: UserInfo, runtime: ToolRuntime[Context]) -> str:
            """Save user info."""
            # Access the store - same as that provided to `create_agent`
            assert runtime.store is not None
            store = runtime.store
            user_id = runtime.context.user_id
            # Store data in the store (namespace, key, data)
            store.put(("users",), user_id, dict(user_info))
            return "Successfully saved user info."


        agent: Runnable = create_agent(
            model="google_genai:gemini-3.6-flash",
            tools=[save_user_info],
            store=store,
            context_schema=Context,
        )

        # Run the agent
        agent.invoke(
            {"messages": [{"role": "user", "content": "My name is John Smith"}]},
            # user_id passed in context to identify whose information is being updated
            context=Context(user_id="user_123"),
        )

        # You can access the store directly to get the value
        item = store.get(("users",), "user_123")
        ```

    === "OpenAI"

        ```python
        from dataclasses import dataclass

        from langchain.agents import create_agent
        from langchain.tools import ToolRuntime, tool
        from langchain_core.runnables import Runnable
        from langgraph.store.memory import InMemoryStore
        from typing_extensions import TypedDict

        # InMemoryStore saves data to an in-memory dictionary. Use a DB-backed store in production.
        store = InMemoryStore()


        @dataclass
        class Context:
            user_id: str


        # TypedDict defines the structure of user information for the LLM
        class UserInfo(TypedDict):
            name: str


        # Tool that allows agent to update user information (useful for chat applications)
        @tool
        def save_user_info(user_info: UserInfo, runtime: ToolRuntime[Context]) -> str:
            """Save user info."""
            # Access the store - same as that provided to `create_agent`
            assert runtime.store is not None
            store = runtime.store
            user_id = runtime.context.user_id
            # Store data in the store (namespace, key, data)
            store.put(("users",), user_id, dict(user_info))
            return "Successfully saved user info."


        agent: Runnable = create_agent(
            model="openai:gpt-5.5",
            tools=[save_user_info],
            store=store,
            context_schema=Context,
        )

        # Run the agent
        agent.invoke(
            {"messages": [{"role": "user", "content": "My name is John Smith"}]},
            # user_id passed in context to identify whose information is being updated
            context=Context(user_id="user_123"),
        )

        # You can access the store directly to get the value
        item = store.get(("users",), "user_123")
        ```

    === "Anthropic"

        ```python
        from dataclasses import dataclass

        from langchain.agents import create_agent
        from langchain.tools import ToolRuntime, tool
        from langchain_core.runnables import Runnable
        from langgraph.store.memory import InMemoryStore
        from typing_extensions import TypedDict

        # InMemoryStore saves data to an in-memory dictionary. Use a DB-backed store in production.
        store = InMemoryStore()


        @dataclass
        class Context:
            user_id: str


        # TypedDict defines the structure of user information for the LLM
        class UserInfo(TypedDict):
            name: str


        # Tool that allows agent to update user information (useful for chat applications)
        @tool
        def save_user_info(user_info: UserInfo, runtime: ToolRuntime[Context]) -> str:
            """Save user info."""
            # Access the store - same as that provided to `create_agent`
            assert runtime.store is not None
            store = runtime.store
            user_id = runtime.context.user_id
            # Store data in the store (namespace, key, data)
            store.put(("users",), user_id, dict(user_info))
            return "Successfully saved user info."


        agent: Runnable = create_agent(
            model="anthropic:claude-sonnet-4-6",
            tools=[save_user_info],
            store=store,
            context_schema=Context,
        )

        # Run the agent
        agent.invoke(
            {"messages": [{"role": "user", "content": "My name is John Smith"}]},
            # user_id passed in context to identify whose information is being updated
            context=Context(user_id="user_123"),
        )

        # You can access the store directly to get the value
        item = store.get(("users",), "user_123")
        ```

    === "OpenRouter"

        ```python
        from dataclasses import dataclass

        from langchain.agents import create_agent
        from langchain.tools import ToolRuntime, tool
        from langchain_core.runnables import Runnable
        from langgraph.store.memory import InMemoryStore
        from typing_extensions import TypedDict

        # InMemoryStore saves data to an in-memory dictionary. Use a DB-backed store in production.
        store = InMemoryStore()


        @dataclass
        class Context:
            user_id: str


        # TypedDict defines the structure of user information for the LLM
        class UserInfo(TypedDict):
            name: str


        # Tool that allows agent to update user information (useful for chat applications)
        @tool
        def save_user_info(user_info: UserInfo, runtime: ToolRuntime[Context]) -> str:
            """Save user info."""
            # Access the store - same as that provided to `create_agent`
            assert runtime.store is not None
            store = runtime.store
            user_id = runtime.context.user_id
            # Store data in the store (namespace, key, data)
            store.put(("users",), user_id, dict(user_info))
            return "Successfully saved user info."


        agent: Runnable = create_agent(
            model="openrouter:z-ai/glm-5.2",
            tools=[save_user_info],
            store=store,
            context_schema=Context,
        )

        # Run the agent
        agent.invoke(
            {"messages": [{"role": "user", "content": "My name is John Smith"}]},
            # user_id passed in context to identify whose information is being updated
            context=Context(user_id="user_123"),
        )

        # You can access the store directly to get the value
        item = store.get(("users",), "user_123")
        ```

    === "Fireworks"

        ```python
        from dataclasses import dataclass

        from langchain.agents import create_agent
        from langchain.tools import ToolRuntime, tool
        from langchain_core.runnables import Runnable
        from langgraph.store.memory import InMemoryStore
        from typing_extensions import TypedDict

        # InMemoryStore saves data to an in-memory dictionary. Use a DB-backed store in production.
        store = InMemoryStore()


        @dataclass
        class Context:
            user_id: str


        # TypedDict defines the structure of user information for the LLM
        class UserInfo(TypedDict):
            name: str


        # Tool that allows agent to update user information (useful for chat applications)
        @tool
        def save_user_info(user_info: UserInfo, runtime: ToolRuntime[Context]) -> str:
            """Save user info."""
            # Access the store - same as that provided to `create_agent`
            assert runtime.store is not None
            store = runtime.store
            user_id = runtime.context.user_id
            # Store data in the store (namespace, key, data)
            store.put(("users",), user_id, dict(user_info))
            return "Successfully saved user info."


        agent: Runnable = create_agent(
            model="fireworks:accounts/fireworks/models/glm-5p2",
            tools=[save_user_info],
            store=store,
            context_schema=Context,
        )

        # Run the agent
        agent.invoke(
            {"messages": [{"role": "user", "content": "My name is John Smith"}]},
            # user_id passed in context to identify whose information is being updated
            context=Context(user_id="user_123"),
        )

        # You can access the store directly to get the value
        item = store.get(("users",), "user_123")
        ```

    === "Baseten"

        ```python
        from dataclasses import dataclass

        from langchain.agents import create_agent
        from langchain.tools import ToolRuntime, tool
        from langchain_core.runnables import Runnable
        from langgraph.store.memory import InMemoryStore
        from typing_extensions import TypedDict

        # InMemoryStore saves data to an in-memory dictionary. Use a DB-backed store in production.
        store = InMemoryStore()


        @dataclass
        class Context:
            user_id: str


        # TypedDict defines the structure of user information for the LLM
        class UserInfo(TypedDict):
            name: str


        # Tool that allows agent to update user information (useful for chat applications)
        @tool
        def save_user_info(user_info: UserInfo, runtime: ToolRuntime[Context]) -> str:
            """Save user info."""
            # Access the store - same as that provided to `create_agent`
            assert runtime.store is not None
            store = runtime.store
            user_id = runtime.context.user_id
            # Store data in the store (namespace, key, data)
            store.put(("users",), user_id, dict(user_info))
            return "Successfully saved user info."


        agent: Runnable = create_agent(
            model="baseten:zai-org/GLM-5.2",
            tools=[save_user_info],
            store=store,
            context_schema=Context,
        )

        # Run the agent
        agent.invoke(
            {"messages": [{"role": "user", "content": "My name is John Smith"}]},
            # user_id passed in context to identify whose information is being updated
            context=Context(user_id="user_123"),
        )

        # You can access the store directly to get the value
        item = store.get(("users",), "user_123")
        ```

    === "Ollama"

        ```python
        from dataclasses import dataclass

        from langchain.agents import create_agent
        from langchain.tools import ToolRuntime, tool
        from langchain_core.runnables import Runnable
        from langgraph.store.memory import InMemoryStore
        from typing_extensions import TypedDict

        # InMemoryStore saves data to an in-memory dictionary. Use a DB-backed store in production.
        store = InMemoryStore()


        @dataclass
        class Context:
            user_id: str


        # TypedDict defines the structure of user information for the LLM
        class UserInfo(TypedDict):
            name: str


        # Tool that allows agent to update user information (useful for chat applications)
        @tool
        def save_user_info(user_info: UserInfo, runtime: ToolRuntime[Context]) -> str:
            """Save user info."""
            # Access the store - same as that provided to `create_agent`
            assert runtime.store is not None
            store = runtime.store
            user_id = runtime.context.user_id
            # Store data in the store (namespace, key, data)
            store.put(("users",), user_id, dict(user_info))
            return "Successfully saved user info."


        agent: Runnable = create_agent(
            model="ollama:north-mini-code-1.0",
            tools=[save_user_info],
            store=store,
            context_schema=Context,
        )

        # Run the agent
        agent.invoke(
            {"messages": [{"role": "user", "content": "My name is John Smith"}]},
            # user_id passed in context to identify whose information is being updated
            context=Context(user_id="user_123"),
        )

        # You can access the store directly to get the value
        item = store.get(("users",), "user_123")
        ```

=== "PostgreSQL"

    ```python
    from dataclasses import dataclass

    from langchain.agents import create_agent
    from langchain.tools import ToolRuntime, tool
    from langchain_core.runnables import Runnable
    from langgraph.store.postgres import PostgresStore  # type: ignore[import-not-found]
    from typing_extensions import TypedDict


    @dataclass
    class Context:
        user_id: str


    class UserInfo(TypedDict):
        name: str


    @tool
    def save_user_info(user_info: UserInfo, runtime: ToolRuntime[Context]) -> str:
        """Save user info."""
        assert runtime.store is not None
        runtime.store.put(("users",), runtime.context.user_id, dict(user_info))
        return "Successfully saved user info."


    DB_URI = "postgresql://postgres:postgres@localhost:5432/postgres?sslmode=disable"

    with PostgresStore.from_conn_string(DB_URI) as store:
        store.setup()
        agent: Runnable = create_agent(
            "claude-sonnet-4-6",
            tools=[save_user_info],
            store=store,
            context_schema=Context,
        )

        agent.invoke(
            {"messages": [{"role": "user", "content": "My name is John Smith"}]},
            context=Context(user_id="user_123"),
        )
    ```

---

[Kết nối tài liệu này](https://docs.langchain.com/use-these-docs) với Claude, VSCode và nhiều công cụ khác qua MCP để nhận câu trả lời theo thời gian thực.

[Chỉnh sửa trang này trên GitHub](https://github.com/langchain-ai/docs/edit/main/src/oss/langchain/long-term-memory.mdx) hoặc [báo lỗi](https://github.com/langchain-ai/docs/issues/new/choose).
