# Memory

Ứng dụng AI cần [memory](../langchain/../langchain/overview.md) để chia sẻ ngữ cảnh (context) qua nhiều lượt tương tác. Trong LangGraph, bạn có thể thêm hai loại memory:

* [Thêm short-term memory](#add-short-term-memory) như một phần của [state](graph-api.md#state) của agent để hỗ trợ hội thoại nhiều lượt.
* [Thêm long-term memory](#add-long-term-memory) để lưu dữ liệu theo user hoặc theo ứng dụng xuyên suốt các session.

## Thêm short-term memory

**Short-term memory** (persistence ở cấp độ thread) giúp agent theo dõi hội thoại nhiều lượt. Để thêm short-term memory:

```python
from langgraph.checkpoint.memory import InMemorySaver  # [!code highlight]
from langgraph.graph import StateGraph

checkpointer = InMemorySaver()  # [!code highlight]

builder = StateGraph(...)
graph = builder.compile(checkpointer=checkpointer)  # [!code highlight]

graph.invoke(
    {"messages": [{"role": "user", "content": "hi! i am Bob"}]},
    {"configurable": {"thread_id": "1"}},  # [!code highlight]
)
```

### Dùng trong production

Trong production, hãy dùng checkpointer được lưu trữ bởi một database:

```python
from langgraph.checkpoint.postgres import PostgresSaver

DB_URI = "postgresql://postgres:postgres@localhost:5432/postgres?sslmode=disable"
with PostgresSaver.from_conn_string(DB_URI) as checkpointer:  # [!code highlight]
    builder = StateGraph(...)
    graph = builder.compile(checkpointer=checkpointer)  # [!code highlight]
```

??? note "Ví dụ: dùng Postgres checkpointer"
    ```
    pip install -U "psycopg[binary,pool]" langgraph langgraph-checkpoint-postgres
    ```

    !!! tip
        Bạn cần gọi `checkpointer.setup()` trong lần đầu tiên dùng Postgres checkpointer

    === "Sync"
        ```python
        from langchain.chat_models import init_chat_model
        from langgraph.graph import StateGraph, MessagesState, START
        from langgraph.checkpoint.postgres import PostgresSaver  # [!code highlight]

        model = init_chat_model(model="claude-haiku-4-5-20251001")

        DB_URI = "postgresql://postgres:postgres@localhost:5432/postgres?sslmode=disable"
        with PostgresSaver.from_conn_string(DB_URI) as checkpointer:  # [!code highlight]
            # checkpointer.setup()

            def call_model(state: MessagesState):
                response = model.invoke(state["messages"])
                return {"messages": response}

            builder = StateGraph(MessagesState)
            builder.add_node(call_model)
            builder.add_edge(START, "call_model")

            graph = builder.compile(checkpointer=checkpointer)  # [!code highlight]

            config = {
                "configurable": {
                    "thread_id": "1"  # [!code highlight]
                }
            }

            stream = graph.stream_events(
                {"messages": [{"role": "user", "content": "hi! I'm bob"}]},
                config,  # [!code highlight]
                version="v3",
            )
            for snapshot in stream.values:
                print(snapshot)

            stream = graph.stream_events(
                {"messages": [{"role": "user", "content": "what's my name?"}]},
                config,  # [!code highlight]
                version="v3",
            )
            for snapshot in stream.values:
                print(snapshot)
        ```

    === "Async"
        ```python
        from langchain.chat_models import init_chat_model
        from langgraph.graph import StateGraph, MessagesState, START
        from langgraph.checkpoint.postgres.aio import AsyncPostgresSaver  # [!code highlight]

        model = init_chat_model(model="claude-haiku-4-5-20251001")

        DB_URI = "postgresql://postgres:postgres@localhost:5432/postgres?sslmode=disable"
        async with AsyncPostgresSaver.from_conn_string(DB_URI) as checkpointer:  # [!code highlight]
            # await checkpointer.setup()

            async def call_model(state: MessagesState):
                response = await model.ainvoke(state["messages"])
                return {"messages": response}

            builder = StateGraph(MessagesState)
            builder.add_node(call_model)
            builder.add_edge(START, "call_model")

            graph = builder.compile(checkpointer=checkpointer)  # [!code highlight]

            config = {
                "configurable": {
                    "thread_id": "1"  # [!code highlight]
                }
            }

            stream = await graph.astream_events(
                {"messages": [{"role": "user", "content": "hi! I'm bob"}]},
                config,  # [!code highlight]
                version="v3",
            )
            async for message in stream.messages:
                async for token in message.text:
                    print(token, end="", flush=True)

            stream = await graph.astream_events(
                {"messages": [{"role": "user", "content": "what's my name?"}]},
                config,  # [!code highlight]
                version="v3",
            )
            async for message in stream.messages:
                async for token in message.text:
                    print(token, end="", flush=True)
        ```

??? note "Ví dụ: dùng MongoDB checkpointer"
    ```
    pip install -U pymongo langgraph langgraph-checkpoint-mongodb
    ```

    !!! tip "Thiết lập"
        Để dùng [MongoDB checkpointer](https://pypi.org/project/langgraph-checkpoint-mongodb/), bạn cần một MongoDB cluster. Làm theo [hướng dẫn này](https://www.mongodb.com/docs/guides/atlas/cluster/) để tạo cluster nếu bạn chưa có.

    === "Sync"
        ```python
        from langchain.chat_models import init_chat_model
        from langgraph.graph import StateGraph, MessagesState, START
        from langgraph.checkpoint.mongodb import MongoDBSaver  # [!code highlight]

        model = init_chat_model(model="claude-haiku-4-5-20251001")

        MONGODB_URI = "localhost:27017"
        with MongoDBSaver.from_conn_string(MONGODB_URI) as checkpointer:  # [!code highlight]

            def call_model(state: MessagesState):
                response = model.invoke(state["messages"])
                return {"messages": response}

            builder = StateGraph(MessagesState)
            builder.add_node(call_model)
            builder.add_edge(START, "call_model")

            graph = builder.compile(checkpointer=checkpointer)  # [!code highlight]

            config = {
                "configurable": {
                    "thread_id": "1"  # [!code highlight]
                }
            }

            stream = graph.stream_events(
                {"messages": [{"role": "user", "content": "hi! I'm bob"}]},
                config,  # [!code highlight]
                version="v3",
            )
            for snapshot in stream.values:
                print(snapshot)

            stream = graph.stream_events(
                {"messages": [{"role": "user", "content": "what's my name?"}]},
                config,  # [!code highlight]
                version="v3",
            )
            for snapshot in stream.values:
                print(snapshot)
        ```

    === "Async"
        ```python
        from langchain.chat_models import init_chat_model
        from langgraph.graph import StateGraph, MessagesState, START
        from langgraph.checkpoint.mongodb.aio import AsyncMongoDBSaver  # [!code highlight]

        model = init_chat_model(model="claude-haiku-4-5-20251001")

        MONGODB_URI = "localhost:27017"
        async with AsyncMongoDBSaver.from_conn_string(MONGODB_URI) as checkpointer:  # [!code highlight]

            async def call_model(state: MessagesState):
                response = await model.ainvoke(state["messages"])
                return {"messages": response}

            builder = StateGraph(MessagesState)
            builder.add_node(call_model)
            builder.add_edge(START, "call_model")

            graph = builder.compile(checkpointer=checkpointer)  # [!code highlight]

            config = {
                "configurable": {
                    "thread_id": "1"  # [!code highlight]
                }
            }

            stream = await graph.astream_events(
                {"messages": [{"role": "user", "content": "hi! I'm bob"}]},
                config,  # [!code highlight]
                version="v3",
            )
            async for message in stream.messages:
                async for token in message.text:
                    print(token, end="", flush=True)

            stream = await graph.astream_events(
                {"messages": [{"role": "user", "content": "what's my name?"}]},
                config,  # [!code highlight]
                version="v3",
            )
            async for message in stream.messages:
                async for token in message.text:
                    print(token, end="", flush=True)
        ```

??? note "Ví dụ: dùng Redis checkpointer"
    ```
    pip install -U langgraph langgraph-checkpoint-redis
    ```

    !!! tip
        Bạn cần gọi `checkpointer.setup()` trong lần đầu tiên dùng Redis checkpointer.

    === "Sync"
        ```python
        from langchain.chat_models import init_chat_model
        from langgraph.graph import StateGraph, MessagesState, START
        from langgraph.checkpoint.redis import RedisSaver  # [!code highlight]

        model = init_chat_model(model="claude-haiku-4-5-20251001")

        DB_URI = "redis://localhost:6379"
        with RedisSaver.from_conn_string(DB_URI) as checkpointer:  # [!code highlight]
            # checkpointer.setup()

            def call_model(state: MessagesState):
                response = model.invoke(state["messages"])
                return {"messages": response}

            builder = StateGraph(MessagesState)
            builder.add_node(call_model)
            builder.add_edge(START, "call_model")

            graph = builder.compile(checkpointer=checkpointer)  # [!code highlight]

            config = {
                "configurable": {
                    "thread_id": "1"  # [!code highlight]
                }
            }

            stream = graph.stream_events(
                {"messages": [{"role": "user", "content": "hi! I'm bob"}]},
                config,  # [!code highlight]
                version="v3",
            )
            for snapshot in stream.values:
                print(snapshot)

            stream = graph.stream_events(
                {"messages": [{"role": "user", "content": "what's my name?"}]},
                config,  # [!code highlight]
                version="v3",
            )
            for snapshot in stream.values:
                print(snapshot)
        ```

    === "Async"
        ```python
        from langchain.chat_models import init_chat_model
        from langgraph.graph import StateGraph, MessagesState, START
        from langgraph.checkpoint.redis.aio import AsyncRedisSaver  # [!code highlight]

        model = init_chat_model(model="claude-haiku-4-5-20251001")

        DB_URI = "redis://localhost:6379"
        async with AsyncRedisSaver.from_conn_string(DB_URI) as checkpointer:  # [!code highlight]
            # await checkpointer.asetup()

            async def call_model(state: MessagesState):
                response = await model.ainvoke(state["messages"])
                return {"messages": response}

            builder = StateGraph(MessagesState)
            builder.add_node(call_model)
            builder.add_edge(START, "call_model")

            graph = builder.compile(checkpointer=checkpointer)  # [!code highlight]

            config = {
                "configurable": {
                    "thread_id": "1"  # [!code highlight]
                }
            }

            stream = await graph.astream_events(
                {"messages": [{"role": "user", "content": "hi! I'm bob"}]},
                config,  # [!code highlight]
                version="v3",
            )
            async for message in stream.messages:
                async for token in message.text:
                    print(token, end="", flush=True)

            stream = await graph.astream_events(
                {"messages": [{"role": "user", "content": "what's my name?"}]},
                config,  # [!code highlight]
                version="v3",
            )
            async for message in stream.messages:
                async for token in message.text:
                    print(token, end="", flush=True)
        ```

??? note "Ví dụ: dùng Oracle checkpointer"
    ```
    pip install -U langgraph langgraph-oracledb
    ```

    !!! note "Thiết lập"
        Để dùng [Oracle checkpointer](https://pypi.org/project/langgraph-oracledb/), bạn cần một instance Oracle AI Database. Một container local (ví dụ `gvenzl/oracle-free:23-slim`) hoặc một Oracle Autonomous Database trên OCI đều dùng được.

    !!! tip
        Bạn cần gọi `checkpointer.setup()` trong lần đầu tiên dùng Oracle checkpointer.

    === "Sync"
        ```python
        from langchain.chat_models import init_chat_model
        from langgraph.graph import StateGraph, MessagesState, START
        from langgraph_oracledb.checkpoint.oracle import OracleSaver  # [!code highlight]

        model = init_chat_model(model="claude-haiku-4-5-20251001")

        DB_URI = "user/password@localhost:1521/FREEPDB1"
        with OracleSaver.from_conn_string(DB_URI) as checkpointer:  # [!code highlight]
            # checkpointer.setup()

            def call_model(state: MessagesState):
                response = model.invoke(state["messages"])
                return {"messages": response}

            builder = StateGraph(MessagesState)
            builder.add_node(call_model)
            builder.add_edge(START, "call_model")

            graph = builder.compile(checkpointer=checkpointer)  # [!code highlight]

            config = {
                "configurable": {
                    "thread_id": "1"  # [!code highlight]
                }
            }

            stream = graph.stream_events(
                {"messages": [{"role": "user", "content": "hi! I'm bob"}]},
                config,  # [!code highlight]
                version="v3",
            )
            for snapshot in stream.values:
                print(snapshot)

            stream = graph.stream_events(
                {"messages": [{"role": "user", "content": "what's my name?"}]},
                config,  # [!code highlight]
                version="v3",
            )
            for snapshot in stream.values:
                print(snapshot)
        ```

    === "Async"
        ```python
        from langchain.chat_models import init_chat_model
        from langgraph.graph import StateGraph, MessagesState, START
        from langgraph_oracledb.checkpoint.oracle import AsyncOracleSaver  # [!code highlight]

        model = init_chat_model(model="claude-haiku-4-5-20251001")

        DB_URI = "user/password@localhost:1521/FREEPDB1"
        async with AsyncOracleSaver.from_conn_string(DB_URI) as checkpointer:  # [!code highlight]
            # await checkpointer.setup()

            async def call_model(state: MessagesState):
                response = await model.ainvoke(state["messages"])
                return {"messages": response}

            builder = StateGraph(MessagesState)
            builder.add_node(call_model)
            builder.add_edge(START, "call_model")

            graph = builder.compile(checkpointer=checkpointer)  # [!code highlight]

            config = {
                "configurable": {
                    "thread_id": "1"  # [!code highlight]
                }
            }

            stream = await graph.astream_events(
                {"messages": [{"role": "user", "content": "hi! I'm bob"}]},
                config,  # [!code highlight]
                version="v3",
            )
            async for message in stream.messages:
                async for token in message.text:
                    print(token, end="", flush=True)

            stream = await graph.astream_events(
                {"messages": [{"role": "user", "content": "what's my name?"}]},
                config,  # [!code highlight]
                version="v3",
            )
            async for message in stream.messages:
                async for token in message.text:
                    print(token, end="", flush=True)
        ```

### Dùng trong subgraph

Nếu graph của bạn chứa [subgraph](use-subgraphs.md), bạn chỉ cần cung cấp checkpointer khi compile graph cha. LangGraph sẽ tự động truyền checkpointer xuống các subgraph con.

```python
from langgraph.graph import START, StateGraph
from langgraph.checkpoint.memory import InMemorySaver
from typing import TypedDict

class State(TypedDict):
    foo: str

# Subgraph

def subgraph_node_1(state: State):
    return {"foo": state["foo"] + "bar"}

subgraph_builder = StateGraph(State)
subgraph_builder.add_node(subgraph_node_1)
subgraph_builder.add_edge(START, "subgraph_node_1")
subgraph = subgraph_builder.compile()  # [!code highlight]

# Parent graph

builder = StateGraph(State)
builder.add_node("node_1", subgraph)  # [!code highlight]
builder.add_edge(START, "node_1")

checkpointer = InMemorySaver()
graph = builder.compile(checkpointer=checkpointer)  # [!code highlight]
```

Bạn có thể cấu hình hành vi checkpointing riêng cho subgraph. Xem [subgraph persistence](use-subgraphs.md#subgraph-persistence) để biết chi tiết về các mức persistence, gồm cả hỗ trợ interrupt và continuation có trạng thái (stateful).

```python
subgraph_builder = StateGraph(...)
subgraph = subgraph_builder.compile(checkpointer=True)  # [!code highlight]
```

## Thêm long-term memory

Dùng long-term memory để lưu dữ liệu theo user hoặc theo ứng dụng xuyên suốt các hội thoại.

```python
from langgraph.store.memory import InMemoryStore  # [!code highlight]
from langgraph.graph import StateGraph

store = InMemoryStore()  # [!code highlight]

builder = StateGraph(...)
graph = builder.compile(store=store)  # [!code highlight]
```

### Truy cập store bên trong node

Sau khi compile graph với một store, LangGraph tự động inject store vào các hàm node của bạn. Cách được khuyến nghị để truy cập store là thông qua đối tượng `Runtime`.

```python
from dataclasses import dataclass
from langgraph.runtime import Runtime
from langgraph.graph import StateGraph, MessagesState, START
import uuid

@dataclass
class Context:
    user_id: str

async def call_model(state: MessagesState, runtime: Runtime[Context]):  # [!code highlight]
    user_id = runtime.context.user_id  # [!code highlight]
    namespace = (user_id, "memories")

    # Tìm các memory liên quan
    memories = await runtime.store.asearch(  # [!code highlight]
        namespace, query=state["messages"][-1].content, limit=3
    )
    info = "\n".join([d.value["data"] for d in memories])

    # ... Dùng memory trong lời gọi model

    # Lưu một memory mới
    await runtime.store.aput(  # [!code highlight]
        namespace, str(uuid.uuid4()), {"data": "User prefers dark mode"}
    )

builder = StateGraph(MessagesState, context_schema=Context)  # [!code highlight]
builder.add_node(call_model)
builder.add_edge(START, "call_model")
graph = builder.compile(store=store)

# Truyền context tại thời điểm invoke
graph.invoke(
    {"messages": [{"role": "user", "content": "hi"}]},
    {"configurable": {"thread_id": "1"}},
    context=Context(user_id="1"),  # [!code highlight]
)
```

### Dùng trong production

Trong production, hãy dùng store được lưu trữ bởi một database:

```python
from langgraph.store.postgres import PostgresStore

DB_URI = "postgresql://postgres:postgres@localhost:5432/postgres?sslmode=disable"
with PostgresStore.from_conn_string(DB_URI) as store:  # [!code highlight]
    builder = StateGraph(...)
    graph = builder.compile(store=store)  # [!code highlight]
```

??? note "Ví dụ: dùng Postgres store"
    ```
    pip install -U "psycopg[binary,pool]" langgraph langgraph-checkpoint-postgres
    ```

    !!! tip
        Bạn cần gọi `store.setup()` trong lần đầu tiên dùng Postgres store

    === "Async"
        ```python
        from dataclasses import dataclass
        from langchain.chat_models import init_chat_model
        from langgraph.graph import StateGraph, MessagesState, START
        from langgraph.checkpoint.postgres.aio import AsyncPostgresSaver
        from langgraph.store.postgres.aio import AsyncPostgresStore  # [!code highlight]
        from langgraph.runtime import Runtime  # [!code highlight]
        import uuid

        model = init_chat_model(model="claude-haiku-4-5-20251001")

        @dataclass
        class Context:
            user_id: str

        async def call_model(  # [!code highlight]
            state: MessagesState,
            runtime: Runtime[Context],  # [!code highlight]
        ):
            user_id = runtime.context.user_id  # [!code highlight]
            namespace = ("memories", user_id)
            memories = await runtime.store.asearch(namespace, query=str(state["messages"][-1].content))  # [!code highlight]
            info = "\n".join([d.value["data"] for d in memories])
            system_msg = f"You are a helpful assistant talking to the user. User info: {info}"

            # Lưu memory mới nếu user yêu cầu model ghi nhớ
            last_message = state["messages"][-1]
            if "remember" in last_message.content.lower():
                memory = "User name is Bob"
                await runtime.store.aput(namespace, str(uuid.uuid4()), {"data": memory})  # [!code highlight]

            response = await model.ainvoke(
                [{"role": "system", "content": system_msg}] + state["messages"]
            )
            return {"messages": response}

        DB_URI = "postgresql://postgres:postgres@localhost:5432/postgres?sslmode=disable"

        async with (
            AsyncPostgresStore.from_conn_string(DB_URI) as store,  # [!code highlight]
            AsyncPostgresSaver.from_conn_string(DB_URI) as checkpointer,
        ):
            # await store.setup()
            # await checkpointer.setup()

            builder = StateGraph(MessagesState, context_schema=Context)  # [!code highlight]
            builder.add_node(call_model)
            builder.add_edge(START, "call_model")

            graph = builder.compile(
                checkpointer=checkpointer,
                store=store,  # [!code highlight]
            )

            config = {"configurable": {"thread_id": "1"}}
            stream = await graph.astream_events(
                {"messages": [{"role": "user", "content": "Hi! Remember: my name is Bob"}]},
                config,
                version="v3",
                context=Context(user_id="1"),  # [!code highlight]
            )
            async for message in stream.messages:
                async for token in message.text:
                    print(token, end="", flush=True)

            config = {"configurable": {"thread_id": "2"}}
            stream = await graph.astream_events(
                {"messages": [{"role": "user", "content": "what is my name?"}]},
                config,
                version="v3",
                context=Context(user_id="1"),  # [!code highlight]
            )
            async for message in stream.messages:
                async for token in message.text:
                    print(token, end="", flush=True)
        ```

    === "Sync"
        ```python
        from dataclasses import dataclass
        from langchain.chat_models import init_chat_model
        from langgraph.graph import StateGraph, MessagesState, START
        from langgraph.checkpoint.postgres import PostgresSaver
        from langgraph.store.postgres import PostgresStore  # [!code highlight]
        from langgraph.runtime import Runtime  # [!code highlight]
        import uuid

        model = init_chat_model(model="claude-haiku-4-5-20251001")

        @dataclass
        class Context:
            user_id: str

        def call_model(  # [!code highlight]
            state: MessagesState,
            runtime: Runtime[Context],  # [!code highlight]
        ):
            user_id = runtime.context.user_id  # [!code highlight]
            namespace = ("memories", user_id)
            memories = runtime.store.search(namespace, query=str(state["messages"][-1].content))  # [!code highlight]
            info = "\n".join([d.value["data"] for d in memories])
            system_msg = f"You are a helpful assistant talking to the user. User info: {info}"

            # Lưu memory mới nếu user yêu cầu model ghi nhớ
            last_message = state["messages"][-1]
            if "remember" in last_message.content.lower():
                memory = "User name is Bob"
                runtime.store.put(namespace, str(uuid.uuid4()), {"data": memory})  # [!code highlight]

            response = model.invoke(
                [{"role": "system", "content": system_msg}] + state["messages"]
            )
            return {"messages": response}

        DB_URI = "postgresql://postgres:postgres@localhost:5432/postgres?sslmode=disable"

        with (
            PostgresStore.from_conn_string(DB_URI) as store,  # [!code highlight]
            PostgresSaver.from_conn_string(DB_URI) as checkpointer,
        ):
            # store.setup()
            # checkpointer.setup()

            builder = StateGraph(MessagesState, context_schema=Context)  # [!code highlight]
            builder.add_node(call_model)
            builder.add_edge(START, "call_model")

            graph = builder.compile(
                checkpointer=checkpointer,
                store=store,  # [!code highlight]
            )

            config = {"configurable": {"thread_id": "1"}}
            stream = graph.stream_events(
                {"messages": [{"role": "user", "content": "Hi! Remember: my name is Bob"}]},
                config,
                version="v3",
                context=Context(user_id="1"),  # [!code highlight]
            )
            for snapshot in stream.values:
                print(snapshot)

            config = {"configurable": {"thread_id": "2"}}
            stream = graph.stream_events(
                {"messages": [{"role": "user", "content": "what is my name?"}]},
                config,
                version="v3",
                context=Context(user_id="1"),  # [!code highlight]
            )
            for snapshot in stream.values:
                print(snapshot)
        ```

??? note "Ví dụ: dùng MongoDB store"

??? note "Ví dụ: dùng Redis store"
    ```
    pip install -U langgraph langgraph-checkpoint-redis
    ```

    !!! tip
        Bạn cần gọi `store.setup()` trong lần đầu tiên dùng [Redis store](https://pypi.org/project/langgraph-checkpoint-redis/).

    === "Async"
        ```python
        from dataclasses import dataclass
        from langchain.chat_models import init_chat_model
        from langgraph.graph import StateGraph, MessagesState, START
        from langgraph.checkpoint.redis.aio import AsyncRedisSaver
        from langgraph.store.redis.aio import AsyncRedisStore  # [!code highlight]
        from langgraph.runtime import Runtime  # [!code highlight]
        import uuid

        model = init_chat_model(model="claude-haiku-4-5-20251001")

        @dataclass
        class Context:
            user_id: str

        async def call_model(  # [!code highlight]
            state: MessagesState,
            runtime: Runtime[Context],  # [!code highlight]
        ):
            user_id = runtime.context.user_id  # [!code highlight]
            namespace = ("memories", user_id)
            memories = await runtime.store.asearch(namespace, query=str(state["messages"][-1].content))  # [!code highlight]
            info = "\n".join([d.value["data"] for d in memories])
            system_msg = f"You are a helpful assistant talking to the user. User info: {info}"

            # Lưu memory mới nếu user yêu cầu model ghi nhớ
            last_message = state["messages"][-1]
            if "remember" in last_message.content.lower():
                memory = "User name is Bob"
                await runtime.store.aput(namespace, str(uuid.uuid4()), {"data": memory})  # [!code highlight]

            response = await model.ainvoke(
                [{"role": "system", "content": system_msg}] + state["messages"]
            )
            return {"messages": response}

        DB_URI = "redis://localhost:6379"

        async with (
            AsyncRedisStore.from_conn_string(DB_URI) as store,  # [!code highlight]
            AsyncRedisSaver.from_conn_string(DB_URI) as checkpointer,
        ):
            # await store.setup()
            # await checkpointer.asetup()

            builder = StateGraph(MessagesState, context_schema=Context)  # [!code highlight]
            builder.add_node(call_model)
            builder.add_edge(START, "call_model")

            graph = builder.compile(
                checkpointer=checkpointer,
                store=store,  # [!code highlight]
            )

            config = {"configurable": {"thread_id": "1"}}
            stream = await graph.astream_events(
                {"messages": [{"role": "user", "content": "Hi! Remember: my name is Bob"}]},
                config,
                version="v3",
                context=Context(user_id="1"),  # [!code highlight]
            )
            async for snapshot in stream.values:
                snapshot["messages"][-1].pretty_print()

            config = {"configurable": {"thread_id": "2"}}
            stream = await graph.astream_events(
                {"messages": [{"role": "user", "content": "what is my name?"}]},
                config,
                version="v3",
                context=Context(user_id="1"),  # [!code highlight]
            )
            async for snapshot in stream.values:
                snapshot["messages"][-1].pretty_print()
        ```

    === "Sync"
        ```python
        from dataclasses import dataclass
        from langchain.chat_models import init_chat_model
        from langgraph.graph import StateGraph, MessagesState, START
        from langgraph.checkpoint.redis import RedisSaver
        from langgraph.store.redis import RedisStore  # [!code highlight]
        from langgraph.runtime import Runtime  # [!code highlight]
        import uuid

        model = init_chat_model(model="claude-haiku-4-5-20251001")

        @dataclass
        class Context:
            user_id: str

        def call_model(  # [!code highlight]
            state: MessagesState,
            runtime: Runtime[Context],  # [!code highlight]
        ):
            user_id = runtime.context.user_id  # [!code highlight]
            namespace = ("memories", user_id)
            memories = runtime.store.search(namespace, query=str(state["messages"][-1].content))  # [!code highlight]
            info = "\n".join([d.value["data"] for d in memories])
            system_msg = f"You are a helpful assistant talking to the user. User info: {info}"

            # Lưu memory mới nếu user yêu cầu model ghi nhớ
            last_message = state["messages"][-1]
            if "remember" in last_message.content.lower():
                memory = "User name is Bob"
                runtime.store.put(namespace, str(uuid.uuid4()), {"data": memory})  # [!code highlight]

            response = model.invoke(
                [{"role": "system", "content": system_msg}] + state["messages"]
            )
            return {"messages": response}

        DB_URI = "redis://localhost:6379"

        with (
            RedisStore.from_conn_string(DB_URI) as store,  # [!code highlight]
            RedisSaver.from_conn_string(DB_URI) as checkpointer,
        ):
            store.setup()
            checkpointer.setup()

            builder = StateGraph(MessagesState, context_schema=Context)  # [!code highlight]
            builder.add_node(call_model)
            builder.add_edge(START, "call_model")

            graph = builder.compile(
                checkpointer=checkpointer,
                store=store,  # [!code highlight]
            )

            config = {"configurable": {"thread_id": "1"}}
            stream = graph.stream_events(
                {"messages": [{"role": "user", "content": "Hi! Remember: my name is Bob"}]},
                config,
                version="v3",
                context=Context(user_id="1"),  # [!code highlight]
            )
            for snapshot in stream.values:
                snapshot["messages"][-1].pretty_print()

            config = {"configurable": {"thread_id": "2"}}
            stream = graph.stream_events(
                {"messages": [{"role": "user", "content": "what is my name?"}]},
                config,
                version="v3",
                context=Context(user_id="1"),  # [!code highlight]
            )
            for snapshot in stream.values:
                snapshot["messages"][-1].pretty_print()
        ```

??? note "Ví dụ: dùng Oracle store"
    ```
    pip install -U langgraph langgraph-oracledb langchain-openai
    ```

    !!! note "Thiết lập"
        Để dùng [Oracle store](https://pypi.org/project/langgraph-oracledb/), bạn cần một instance Oracle AI Database, vector index dùng cho `search` semantic yêu cầu [Oracle AI Vector Search](https://docs.oracle.com/en/database/oracle/oracle-database/23/vecse/).

    !!! tip
        Bạn cần gọi `store.setup()` và `checkpointer.setup()` trong lần đầu tiên dùng Oracle store và checkpointer.

    === "Sync"
        ```python
        import uuid

        from langchain.chat_models import init_chat_model
        from langchain.embeddings import init_embeddings
        from langchain_core.runnables import RunnableConfig
        from langgraph.graph import StateGraph, MessagesState, START
        from langgraph.store.base import BaseStore
        from langgraph_oracledb.checkpoint.oracle import OracleSaver
        from langgraph_oracledb.store.oracle import OracleStore  # [!code highlight]

        model = init_chat_model(model="claude-haiku-4-5-20251001")
        embeddings = init_embeddings("openai:text-embedding-3-small")

        DB_URI = "user/password@localhost:1521/FREEPDB1"

        with (
            OracleStore.from_conn_string(  # [!code highlight]
                DB_URI,
                index={"embed": embeddings, "dims": 1536},  # [!code highlight]
            ) as store,
            OracleSaver.from_conn_string(DB_URI) as checkpointer,
        ):
            store.setup()
            checkpointer.setup()

            def call_model(
                state: MessagesState,
                config: RunnableConfig,
                *,
                store: BaseStore,  # [!code highlight]
            ):
                user_id = config["configurable"]["user_id"]
                namespace = ("memories", user_id)
                memories = store.search(namespace, query=str(state["messages"][-1].content))  # [!code highlight]
                info = "\n".join([d.value["data"] for d in memories])
                system_msg = f"You are a helpful assistant talking to the user. User info: {info}"

                # Lưu memory mới nếu user yêu cầu model ghi nhớ
                last_message = state["messages"][-1]
                if "remember" in last_message.content.lower():
                    memory = "User name is Bob"
                    store.put(namespace, str(uuid.uuid4()), {"data": memory})  # [!code highlight]

                response = model.invoke(
                    [{"role": "system", "content": system_msg}] + state["messages"]
                )
                return {"messages": response}

            builder = StateGraph(MessagesState)
            builder.add_node(call_model)
            builder.add_edge(START, "call_model")

            graph = builder.compile(
                checkpointer=checkpointer,
                store=store,  # [!code highlight]
            )

            config = {
                "configurable": {
                    "thread_id": "1",  # [!code highlight]
                    "user_id": "1",  # [!code highlight]
                }
            }
            stream = graph.stream_events(
                {"messages": [{"role": "user", "content": "Hi! Remember: my name is Bob"}]},
                config,  # [!code highlight]
                version="v3",
            )
            for snapshot in stream.values:
                snapshot["messages"][-1].pretty_print()

            config = {
                "configurable": {
                    "thread_id": "2",  # [!code highlight]
                    "user_id": "1",
                }
            }

            stream = graph.stream_events(
                {"messages": [{"role": "user", "content": "what is my name?"}]},
                config,  # [!code highlight]
                version="v3",
            )
            for snapshot in stream.values:
                snapshot["messages"][-1].pretty_print()
        ```

    === "Async"
        ```python
        import uuid

        from langchain.chat_models import init_chat_model
        from langchain.embeddings import init_embeddings
        from langchain_core.runnables import RunnableConfig
        from langgraph.graph import StateGraph, MessagesState, START
        from langgraph.store.base import BaseStore
        from langgraph_oracledb.checkpoint.oracle import AsyncOracleSaver
        from langgraph_oracledb.store.oracle import AsyncOracleStore  # [!code highlight]

        model = init_chat_model(model="claude-haiku-4-5-20251001")
        embeddings = init_embeddings("openai:text-embedding-3-small")

        DB_URI = "user/password@localhost:1521/FREEPDB1"

        async with (
            AsyncOracleStore.from_conn_string(  # [!code highlight]
                DB_URI,
                index={"embed": embeddings, "dims": 1536},  # [!code highlight]
            ) as store,
            AsyncOracleSaver.from_conn_string(DB_URI) as checkpointer,
        ):
            await store.setup()
            await checkpointer.setup()

            async def call_model(
                state: MessagesState,
                config: RunnableConfig,
                *,
                store: BaseStore,  # [!code highlight]
            ):
                user_id = config["configurable"]["user_id"]
                namespace = ("memories", user_id)
                memories = await store.asearch(namespace, query=str(state["messages"][-1].content))  # [!code highlight]
                info = "\n".join([d.value["data"] for d in memories])
                system_msg = f"You are a helpful assistant talking to the user. User info: {info}"

                # Lưu memory mới nếu user yêu cầu model ghi nhớ
                last_message = state["messages"][-1]
                if "remember" in last_message.content.lower():
                    memory = "User name is Bob"
                    await store.aput(namespace, str(uuid.uuid4()), {"data": memory})  # [!code highlight]

                response = await model.ainvoke(
                    [{"role": "system", "content": system_msg}] + state["messages"]
                )
                return {"messages": response}

            builder = StateGraph(MessagesState)
            builder.add_node(call_model)
            builder.add_edge(START, "call_model")

            graph = builder.compile(
                checkpointer=checkpointer,
                store=store,  # [!code highlight]
            )

            config = {
                "configurable": {
                    "thread_id": "1",  # [!code highlight]
                    "user_id": "1",  # [!code highlight]
                }
            }
            stream = await graph.astream_events(
                {"messages": [{"role": "user", "content": "Hi! Remember: my name is Bob"}]},
                config,  # [!code highlight]
                version="v3",
            )
            async for snapshot in stream.values:
                snapshot["messages"][-1].pretty_print()

            config = {
                "configurable": {
                    "thread_id": "2",  # [!code highlight]
                    "user_id": "1",
                }
            }

            stream = await graph.astream_events(
                {"messages": [{"role": "user", "content": "what is my name?"}]},
                config,  # [!code highlight]
                version="v3",
            )
            async for snapshot in stream.values:
                snapshot["messages"][-1].pretty_print()
        ```

### Dùng semantic search

Bật semantic search trong store memory của graph để agent trong graph có thể tìm các item trong store theo độ tương đồng ngữ nghĩa (semantic similarity).

```python
from langchain.embeddings import init_embeddings
from langgraph.store.memory import InMemoryStore

# Tạo store với semantic search được bật
embeddings = init_embeddings("openai:text-embedding-3-small")
store = InMemoryStore(
    index={
        "embed": embeddings,
        "dims": 1536,
    }
)

store.put(("user_123", "memories"), "1", {"text": "I love pizza"})
store.put(("user_123", "memories"), "2", {"text": "I am a plumber"})

items = store.search(
    ("user_123", "memories"), query="I'm hungry", limit=1
)
```

??? note "Long-term memory với semantic search"
    ```python
    from langchain.embeddings import init_embeddings
    from langchain.chat_models import init_chat_model
    from langgraph.store.memory import InMemoryStore
    from langgraph.graph import START, MessagesState, StateGraph
    from langgraph.runtime import Runtime  # [!code highlight]

    model = init_chat_model("gpt-5.4-mini")

    # Tạo store với semantic search được bật
    embeddings = init_embeddings("openai:text-embedding-3-small")
    store = InMemoryStore(
        index={
            "embed": embeddings,
            "dims": 1536,
        }
    )

    store.put(("user_123", "memories"), "1", {"text": "I love pizza"})
    store.put(("user_123", "memories"), "2", {"text": "I am a plumber"})

    async def chat(state: MessagesState, runtime: Runtime):  # [!code highlight]
        # Tìm dựa trên message cuối cùng của user
        items = await runtime.store.asearch(  # [!code highlight]
            ("user_123", "memories"), query=state["messages"][-1].content, limit=2
        )
        memories = "\n".join(item.value["text"] for item in items)
        memories = f"## Memories of user\n{memories}" if memories else ""
        response = await model.ainvoke(
            [
                {"role": "system", "content": f"You are a helpful assistant.\n{memories}"},
                *state["messages"],
            ]
        )
        return {"messages": [response]}


    builder = StateGraph(MessagesState)
    builder.add_node(chat)
    builder.add_edge(START, "chat")
    graph = builder.compile(store=store)

    stream = await graph.astream_events(
        {"messages": [{"role": "user", "content": "I'm hungry"}]},
        version="v3",
    )
    async for message in stream.messages:
        async for token in message.text:
            print(token, end="", flush=True)
    ```

## Quản lý short-term memory

Khi bật [short-term memory](#add-short-term-memory), các hội thoại dài có thể vượt quá context window của LLM. Các cách xử lý phổ biến là:

* [Cắt bớt message](#trim-messages): Loại bỏ N message đầu hoặc cuối (trước khi gọi LLM)
* [Xoá message](#delete-messages) khỏi state của LangGraph vĩnh viễn
* [Tóm tắt message](#summarize-messages): Tóm tắt các message trước đó trong lịch sử và thay thế bằng một bản tóm tắt
* [Quản lý checkpoint](#manage-checkpoints) để lưu và truy xuất lịch sử message
* Các chiến lược tuỳ chỉnh (ví dụ: lọc message, v.v.)

Cách này giúp agent theo dõi hội thoại mà không vượt quá context window của LLM.

### Cắt bớt message (Trim messages)

Hầu hết LLM có context window tối đa được hỗ trợ (tính theo token). Một cách để quyết định thời điểm cắt bớt message là đếm số token trong lịch sử message và cắt bớt khi gần chạm giới hạn đó. Nếu bạn dùng LangChain, bạn có thể dùng tiện ích trim messages và chỉ định số token cần giữ lại từ danh sách, cũng như `strategy` (ví dụ: giữ `max_tokens` cuối cùng) để xử lý ranh giới.

Để cắt bớt lịch sử message, dùng hàm [`trim_messages`](https://reference.langchain.com/python/langchain-core/messages/utils/trim_messages):

```python
from langchain_core.messages.utils import (  # [!code highlight]
    trim_messages,  # [!code highlight]
    count_tokens_approximately  # [!code highlight]
)  # [!code highlight]

def call_model(state: MessagesState):
    messages = trim_messages(  # [!code highlight]
        state["messages"],
        strategy="last",
        token_counter=count_tokens_approximately,
        max_tokens=128,
        start_on="human",
        end_on=("human", "tool"),
    )
    response = model.invoke(messages)
    return {"messages": [response]}

builder = StateGraph(MessagesState)
builder.add_node(call_model)
...
```

??? note "Ví dụ đầy đủ: cắt bớt message"
    ```python
    from langchain_core.messages.utils import (
        trim_messages,  # [!code highlight]
        count_tokens_approximately  # [!code highlight]
    )
    from langchain.chat_models import init_chat_model
    from langgraph.graph import StateGraph, START, MessagesState

    model = init_chat_model("claude-sonnet-4-6")
    summarization_model = model.bind(max_tokens=128)

    def call_model(state: MessagesState):
        messages = trim_messages(  # [!code highlight]
            state["messages"],
            strategy="last",
            token_counter=count_tokens_approximately,
            max_tokens=128,
            start_on="human",
            end_on=("human", "tool"),
        )
        response = model.invoke(messages)
        return {"messages": [response]}

    checkpointer = InMemorySaver()
    builder = StateGraph(MessagesState)
    builder.add_node(call_model)
    builder.add_edge(START, "call_model")
    graph = builder.compile(checkpointer=checkpointer)

    config = {"configurable": {"thread_id": "1"}}
    graph.invoke({"messages": "hi, my name is bob"}, config)
    graph.invoke({"messages": "write a short poem about cats"}, config)
    graph.invoke({"messages": "now do the same but for dogs"}, config)
    final_response = graph.invoke({"messages": "what's my name?"}, config)

    final_response["messages"][-1].pretty_print()
    ```

    ```
    ================================== Ai Message ==================================

    Your name is Bob, as you mentioned when you first introduced yourself.
    ```

### Xoá message (Delete messages)

Bạn có thể xoá message khỏi state của graph để quản lý lịch sử message. Điều này hữu ích khi bạn muốn xoá các message cụ thể hoặc xoá toàn bộ lịch sử message.

Để xoá message khỏi state của graph, bạn dùng `RemoveMessage`. Để `RemoveMessage` hoạt động, bạn cần dùng một state key với [reducer](graph-api.md#reducers) [`add_messages`](https://reference.langchain.com/python/langgraph/graph/message/add_messages), như [`MessagesState`](graph-api.md#messagesstate).

Để xoá các message cụ thể:

```python
from langchain.messages import RemoveMessage  # [!code highlight]

def delete_messages(state):
    messages = state["messages"]
    if len(messages) > 2:
        # xoá hai message sớm nhất
        return {"messages": [RemoveMessage(id=m.id) for m in messages[:2]]}  # [!code highlight]
```

Để xoá **toàn bộ** message:

```python
from langgraph.graph.message import REMOVE_ALL_MESSAGES  # [!code highlight]

def delete_messages(state):
    return {"messages": [RemoveMessage(id=REMOVE_ALL_MESSAGES)]}  # [!code highlight]
```

!!! warning
    Khi xoá message, **hãy đảm bảo** lịch sử message còn lại vẫn hợp lệ. Kiểm tra các giới hạn của nhà cung cấp LLM bạn đang dùng. Ví dụ:

    * Một số provider yêu cầu lịch sử message phải bắt đầu bằng message `user`
    * Hầu hết provider yêu cầu message `assistant` có tool call phải được theo sau bởi message kết quả `tool` tương ứng.

??? note "Ví dụ đầy đủ: xoá message"
    ```python
    from langchain.messages import RemoveMessage  # [!code highlight]

    def delete_messages(state):
        messages = state["messages"]
        if len(messages) > 2:
            # xoá hai message sớm nhất
            return {"messages": [RemoveMessage(id=m.id) for m in messages[:2]]}  # [!code highlight]

    def call_model(state: MessagesState):
        response = model.invoke(state["messages"])
        return {"messages": response}

    builder = StateGraph(MessagesState)
    builder.add_sequence([call_model, delete_messages])
    builder.add_edge(START, "call_model")

    checkpointer = InMemorySaver()
    app = builder.compile(checkpointer=checkpointer)

    stream = app.stream_events(
        {"messages": [{"role": "user", "content": "hi! I'm bob"}]},
        config,
        version="v3"
    )
    for snapshot in stream.values:
        print([(message.type, message.content) for message in snapshot["messages"]])

    stream = app.stream_events(
        {"messages": [{"role": "user", "content": "what's my name?"}]},
        config,
        version="v3"
    )
    for snapshot in stream.values:
        print([(message.type, message.content) for message in snapshot["messages"]])
    ```

    ```
    [('human', "hi! I'm bob")]
    [('human', "hi! I'm bob"), ('ai', 'Hi Bob! How are you doing today? Is there anything I can help you with?')]
    [('human', "hi! I'm bob"), ('ai', 'Hi Bob! How are you doing today? Is there anything I can help you with?'), ('human', "what's my name?")]
    [('human', "hi! I'm bob"), ('ai', 'Hi Bob! How are you doing today? Is there anything I can help you with?'), ('human', "what's my name?"), ('ai', 'Your name is Bob.')]
    [('human', "what's my name?"), ('ai', 'Your name is Bob.')]
    ```

### Tóm tắt message (Summarize messages)

Vấn đề của việc cắt bớt hoặc xoá message như trên là bạn có thể mất thông tin do bị loại khỏi hàng đợi message. Vì vậy, một số ứng dụng dùng cách tiếp cận phức tạp hơn: tóm tắt lịch sử message bằng một chat model.

<img src="https://mintcdn.com/langchain-5e9cc07a/ybiAaBfoBvFquMDz/oss/images/summary.png?fit=max&auto=format&n=ybiAaBfoBvFquMDz&q=85&s=c8ed3facdccd4ef5c7e52902c72ba938" alt="Summary" width="609" height="242" />

Bạn có thể dùng logic prompting và orchestration để tóm tắt lịch sử message. Ví dụ, trong LangGraph bạn có thể mở rộng [`MessagesState`](graph-api.md#working-with-messages-in-graph-state) để thêm key `summary`:

```python
from langgraph.graph import MessagesState
class State(MessagesState):
    summary: str
```

Sau đó, bạn có thể tạo bản tóm tắt lịch sử chat, dùng bản tóm tắt hiện có (nếu có) làm ngữ cảnh cho bản tóm tắt tiếp theo. Node `summarize_conversation` này có thể được gọi sau khi một số lượng message nhất định đã tích luỹ trong state key `messages`.

```python
def summarize_conversation(state: State):

    # Trước tiên, lấy bản tóm tắt hiện có (nếu có)
    summary = state.get("summary", "")

    # Tạo prompt tóm tắt
    if summary:

        # Đã có một bản tóm tắt
        summary_message = (
            f"This is a summary of the conversation to date: {summary}\n\n"
            "Extend the summary by taking into account the new messages above:"
        )

    else:
        summary_message = "Create a summary of the conversation above:"

    # Thêm prompt vào lịch sử
    messages = state["messages"] + [HumanMessage(content=summary_message)]
    response = model.invoke(messages)

    # Xoá tất cả trừ 2 message gần nhất
    delete_messages = [RemoveMessage(id=m.id) for m in state["messages"][:-2]]
    return {"summary": response.content, "messages": delete_messages}
```

??? note "Ví dụ đầy đủ: tóm tắt message"
    ```python
    from typing import Any, TypedDict

    from langchain.chat_models import init_chat_model
    from langchain.messages import AnyMessage
    from langchain_core.messages.utils import count_tokens_approximately
    from langgraph.graph import StateGraph, START, MessagesState
    from langgraph.checkpoint.memory import InMemorySaver
    from langmem.short_term import SummarizationNode, RunningSummary  # [!code highlight]

    model = init_chat_model("claude-sonnet-4-6")
    summarization_model = model.bind(max_tokens=128)

    class State(MessagesState):
        context: dict[str, RunningSummary]  # [!code highlight]

    class LLMInputState(TypedDict):  # [!code highlight]
        summarized_messages: list[AnyMessage]
        context: dict[str, RunningSummary]

    summarization_node = SummarizationNode(  # [!code highlight]
        token_counter=count_tokens_approximately,
        model=summarization_model,
        max_tokens=256,
        max_tokens_before_summary=256,
        max_summary_tokens=128,
    )

    def call_model(state: LLMInputState):  # [!code highlight]
        response = model.invoke(state["summarized_messages"])
        return {"messages": [response]}

    checkpointer = InMemorySaver()
    builder = StateGraph(State)
    builder.add_node(call_model)
    builder.add_node("summarize", summarization_node)  # [!code highlight]
    builder.add_edge(START, "summarize")
    builder.add_edge("summarize", "call_model")
    graph = builder.compile(checkpointer=checkpointer)

    # Invoke graph
    config = {"configurable": {"thread_id": "1"}}
    graph.invoke({"messages": "hi, my name is bob"}, config)
    graph.invoke({"messages": "write a short poem about cats"}, config)
    graph.invoke({"messages": "now do the same but for dogs"}, config)
    final_response = graph.invoke({"messages": "what's my name?"}, config)

    final_response["messages"][-1].pretty_print()
    print("\nSummary:", final_response["context"]["running_summary"].summary)
    ```

    1. Ta theo dõi bản tóm tắt hiện tại trong field `context` (được `SummarizationNode` kỳ vọng).
    2. Định nghĩa state riêng (private) chỉ dùng để lọc input cho node `call_model`.
    3. Ta truyền một private input state ở đây để cô lập các message được trả về từ node tóm tắt.

    ```
    ================================== Ai Message ==================================

    From our conversation, I can see that you introduced yourself as Bob. That's the name you shared with me when we began talking.

    Summary: In this conversation, I was introduced to Bob, who then asked me to write a poem about cats. I composed a poem titled "The Mystery of Cats" that captured cats' graceful movements, independent nature, and their special relationship with humans. Bob then requested a similar poem about dogs, so I wrote "The Joy of Dogs," which highlighted dogs' loyalty, enthusiasm, and loving companionship. Both poems were written in a similar style but emphasized the distinct characteristics that make each pet special.
    ```

### Quản lý checkpoint (Manage checkpoints)

Bạn có thể xem và xoá thông tin được lưu bởi checkpointer.

#### Xem state của thread

=== "Graph/Functional API"
    ```python
    config = {
        "configurable": {
            "thread_id": "1",  # [!code highlight]
            # tuỳ chọn cung cấp ID cho một checkpoint cụ thể,
            # nếu không sẽ hiển thị checkpoint mới nhất
            # "checkpoint_id": "1f029ca3-1f5b-6704-8004-820c16b69a5a"  # [!code highlight]

        }
    }
    graph.get_state(config)  # [!code highlight]
    ```

    ```
    StateSnapshot(
        values={'messages': [HumanMessage(content="hi! I'm bob"), AIMessage(content='Hi Bob! How are you doing today?), HumanMessage(content="what's my name?"), AIMessage(content='Your name is Bob.')]}, next=(),
        config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1f029ca3-1f5b-6704-8004-820c16b69a5a'}},
        metadata={
            'source': 'loop',
            'writes': {'call_model': {'messages': AIMessage(content='Your name is Bob.')}},
            'step': 4,
            'parents': {},
            'thread_id': '1'
        },
        created_at='2025-05-05T16:01:24.680462+00:00',
        parent_config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1f029ca3-1790-6b0a-8003-baf965b6a38f'}},
        tasks=(),
        interrupts=()
    )
    ```

=== "Checkpointer API"
    ```python
    config = {
        "configurable": {
            "thread_id": "1",  # [!code highlight]
            # tuỳ chọn cung cấp ID cho một checkpoint cụ thể,
            # nếu không sẽ hiển thị checkpoint mới nhất
            # "checkpoint_id": "1f029ca3-1f5b-6704-8004-820c16b69a5a"  # [!code highlight]

        }
    }
    checkpointer.get_tuple(config)  # [!code highlight]
    ```

    ```
    CheckpointTuple(
        config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1f029ca3-1f5b-6704-8004-820c16b69a5a'}},
        checkpoint={
            'v': 3,
            'ts': '2025-05-05T16:01:24.680462+00:00',
            'id': '1f029ca3-1f5b-6704-8004-820c16b69a5a',
            'channel_versions': {'__start__': '00000000000000000000000000000005.0.5290678567601859', 'messages': '00000000000000000000000000000006.0.3205149138784782', 'branch:to:call_model': '00000000000000000000000000000006.0.14611156755133758'}, 'versions_seen': {'__input__': {}, '__start__': {'__start__': '00000000000000000000000000000004.0.5736472536395331'}, 'call_model': {'branch:to:call_model': '00000000000000000000000000000005.0.1410174088651449'}},
            'channel_values': {'messages': [HumanMessage(content="hi! I'm bob"), AIMessage(content='Hi Bob! How are you doing today?), HumanMessage(content="what's my name?"), AIMessage(content='Your name is Bob.')]},
        },
        metadata={
            'source': 'loop',
            'writes': {'call_model': {'messages': AIMessage(content='Your name is Bob.')}},
            'step': 4,
            'parents': {},
            'thread_id': '1'
        },
        parent_config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1f029ca3-1790-6b0a-8003-baf965b6a38f'}},
        pending_writes=[]
    )
    ```

#### Xem lịch sử của thread

=== "Graph/Functional API"
    ```python
    config = {
        "configurable": {
            "thread_id": "1"  # [!code highlight]
        }
    }
    list(graph.get_state_history(config))  # [!code highlight]
    ```

    ```
    [
        StateSnapshot(
            values={'messages': [HumanMessage(content="hi! I'm bob"), AIMessage(content='Hi Bob! How are you doing today? Is there anything I can help you with?'), HumanMessage(content="what's my name?"), AIMessage(content='Your name is Bob.')]},
            next=(),
            config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1f029ca3-1f5b-6704-8004-820c16b69a5a'}},
            metadata={'source': 'loop', 'writes': {'call_model': {'messages': AIMessage(content='Your name is Bob.')}}, 'step': 4, 'parents': {}, 'thread_id': '1'},
            created_at='2025-05-05T16:01:24.680462+00:00',
            parent_config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1f029ca3-1790-6b0a-8003-baf965b6a38f'}},
            tasks=(),
            interrupts=()
        ),
        StateSnapshot(
            values={'messages': [HumanMessage(content="hi! I'm bob"), AIMessage(content='Hi Bob! How are you doing today? Is there anything I can help you with?'), HumanMessage(content="what's my name?")]},
            next=('call_model',),
            config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1f029ca3-1790-6b0a-8003-baf965b6a38f'}},
            metadata={'source': 'loop', 'writes': None, 'step': 3, 'parents': {}, 'thread_id': '1'},
            created_at='2025-05-05T16:01:23.863421+00:00',
            parent_config={...}
            tasks=(PregelTask(id='8ab4155e-6b15-b885-9ce5-bed69a2c305c', name='call_model', path=('__pregel_pull', 'call_model'), error=None, interrupts=(), state=None, result={'messages': AIMessage(content='Your name is Bob.')}),),
            interrupts=()
        ),
        StateSnapshot(
            values={'messages': [HumanMessage(content="hi! I'm bob"), AIMessage(content='Hi Bob! How are you doing today? Is there anything I can help you with?')]},
            next=('__start__',),
            config={...},
            metadata={'source': 'input', 'writes': {'__start__': {'messages': [{'role': 'user', 'content': "what's my name?"}]}}, 'step': 2, 'parents': {}, 'thread_id': '1'},
            created_at='2025-05-05T16:01:23.863173+00:00',
            parent_config={...}
            tasks=(PregelTask(id='24ba39d6-6db1-4c9b-f4c5-682aeaf38dcd', name='__start__', path=('__pregel_pull', '__start__'), error=None, interrupts=(), state=None, result={'messages': [{'role': 'user', 'content': "what's my name?"}]}),),
            interrupts=()
        ),
        StateSnapshot(
            values={'messages': [HumanMessage(content="hi! I'm bob"), AIMessage(content='Hi Bob! How are you doing today? Is there anything I can help you with?')]},
            next=(),
            config={...},
            metadata={'source': 'loop', 'writes': {'call_model': {'messages': AIMessage(content='Hi Bob! How are you doing today? Is there anything I can help you with?')}}, 'step': 1, 'parents': {}, 'thread_id': '1'},
            created_at='2025-05-05T16:01:23.862295+00:00',
            parent_config={...}
            tasks=(),
            interrupts=()
        ),
        StateSnapshot(
            values={'messages': [HumanMessage(content="hi! I'm bob")]},
            next=('call_model',),
            config={...},
            metadata={'source': 'loop', 'writes': None, 'step': 0, 'parents': {}, 'thread_id': '1'},
            created_at='2025-05-05T16:01:22.278960+00:00',
            parent_config={...}
            tasks=(PregelTask(id='8cbd75e0-3720-b056-04f7-71ac805140a0', name='call_model', path=('__pregel_pull', 'call_model'), error=None, interrupts=(), state=None, result={'messages': AIMessage(content='Hi Bob! How are you doing today? Is there anything I can help you with?')}),),
            interrupts=()
        ),
        StateSnapshot(
            values={'messages': []},
            next=('__start__',),
            config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1f029ca3-0870-6ce2-bfff-1f3f14c3e565'}},
            metadata={'source': 'input', 'writes': {'__start__': {'messages': [{'role': 'user', 'content': "hi! I'm bob"}]}}, 'step': -1, 'parents': {}, 'thread_id': '1'},
            created_at='2025-05-05T16:01:22.277497+00:00',
            parent_config=None,
            tasks=(PregelTask(id='d458367b-8265-812c-18e2-33001d199ce6', name='__start__', path=('__pregel_pull', '__start__'), error=None, interrupts=(), state=None, result={'messages': [{'role': 'user', 'content': "hi! I'm bob"}]}),),
            interrupts=()
        )
    ]
    ```

=== "Checkpointer API"
    ```python
    config = {
        "configurable": {
            "thread_id": "1"  # [!code highlight]
        }
    }
    list(checkpointer.list(config))  # [!code highlight]
    ```

    ```
    [
        CheckpointTuple(
            config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1f029ca3-1f5b-6704-8004-820c16b69a5a'}},
            checkpoint={
                'v': 3,
                'ts': '2025-05-05T16:01:24.680462+00:00',
                'id': '1f029ca3-1f5b-6704-8004-820c16b69a5a',
                'channel_versions': {'__start__': '00000000000000000000000000000005.0.5290678567601859', 'messages': '00000000000000000000000000000006.0.3205149138784782', 'branch:to:call_model': '00000000000000000000000000000006.0.14611156755133758'},
                'versions_seen': {'__input__': {}, '__start__': {'__start__': '00000000000000000000000000000004.0.5736472536395331'}, 'call_model': {'branch:to:call_model': '00000000000000000000000000000005.0.1410174088651449'}},
                'channel_values': {'messages': [HumanMessage(content="hi! I'm bob"), AIMessage(content='Hi Bob! How are you doing today? Is there anything I can help you with?'), HumanMessage(content="what's my name?"), AIMessage(content='Your name is Bob.')]},
            },
            metadata={'source': 'loop', 'writes': {'call_model': {'messages': AIMessage(content='Your name is Bob.')}}, 'step': 4, 'parents': {}, 'thread_id': '1'},
            parent_config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1f029ca3-1790-6b0a-8003-baf965b6a38f'}},
            pending_writes=[]
        ),
        CheckpointTuple(
            config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1f029ca3-1790-6b0a-8003-baf965b6a38f'}},
            checkpoint={
                'v': 3,
                'ts': '2025-05-05T16:01:23.863421+00:00',
                'id': '1f029ca3-1790-6b0a-8003-baf965b6a38f',
                'channel_versions': {'__start__': '00000000000000000000000000000005.0.5290678567601859', 'messages': '00000000000000000000000000000006.0.3205149138784782', 'branch:to:call_model': '00000000000000000000000000000006.0.14611156755133758'},
                'versions_seen': {'__input__': {}, '__start__': {'__start__': '00000000000000000000000000000004.0.5736472536395331'}, 'call_model': {'branch:to:call_model': '00000000000000000000000000000005.0.1410174088651449'}},
                'channel_values': {'messages': [HumanMessage(content="hi! I'm bob"), AIMessage(content='Hi Bob! How are you doing today? Is there anything I can help you with?'), HumanMessage(content="what's my name?")], 'branch:to:call_model': None}
            },
            metadata={'source': 'loop', 'writes': None, 'step': 3, 'parents': {}, 'thread_id': '1'},
            parent_config={...},
            pending_writes=[('8ab4155e-6b15-b885-9ce5-bed69a2c305c', 'messages', AIMessage(content='Your name is Bob.'))]
        ),
        CheckpointTuple(
            config={...},
            checkpoint={
                'v': 3,
                'ts': '2025-05-05T16:01:23.863173+00:00',
                'id': '1f029ca3-1790-616e-8002-9e021694a0cd',
                'channel_versions': {'__start__': '00000000000000000000000000000004.0.5736472536395331', 'messages': '00000000000000000000000000000003.0.7056767754077798', 'branch:to:call_model': '00000000000000000000000000000003.0.22059023329132854'},
                'versions_seen': {'__input__': {}, '__start__': {'__start__': '00000000000000000000000000000001.0.7040775356287469'}, 'call_model': {'branch:to:call_model': '00000000000000000000000000000002.0.9300422176788571'}},
                'channel_values': {'__start__': {'messages': [{'role': 'user', 'content': "what's my name?"}]}, 'messages': [HumanMessage(content="hi! I'm bob"), AIMessage(content='Hi Bob! How are you doing today? Is there anything I can help you with?')]}
            },
            metadata={'source': 'input', 'writes': {'__start__': {'messages': [{'role': 'user', 'content': "what's my name?"}]}}, 'step': 2, 'parents': {}, 'thread_id': '1'},
            parent_config={...},
            pending_writes=[('24ba39d6-6db1-4c9b-f4c5-682aeaf38dcd', 'messages', [{'role': 'user', 'content': "what's my name?"}]), ('24ba39d6-6db1-4c9b-f4c5-682aeaf38dcd', 'branch:to:call_model', None)]
        ),
        CheckpointTuple(
            config={...},
            checkpoint={
                'v': 3,
                'ts': '2025-05-05T16:01:23.862295+00:00',
                'id': '1f029ca3-178d-6f54-8001-d7b180db0c89',
                'channel_versions': {'__start__': '00000000000000000000000000000002.0.18673090920108737', 'messages': '00000000000000000000000000000003.0.7056767754077798', 'branch:to:call_model': '00000000000000000000000000000003.0.22059023329132854'},
                'versions_seen': {'__input__': {}, '__start__': {'__start__': '00000000000000000000000000000001.0.7040775356287469'}, 'call_model': {'branch:to:call_model': '00000000000000000000000000000002.0.9300422176788571'}},
                'channel_values': {'messages': [HumanMessage(content="hi! I'm bob"), AIMessage(content='Hi Bob! How are you doing today? Is there anything I can help you with?')]}
            },
            metadata={'source': 'loop', 'writes': {'call_model': {'messages': AIMessage(content='Hi Bob! How are you doing today? Is there anything I can help you with?')}}, 'step': 1, 'parents': {}, 'thread_id': '1'},
            parent_config={...},
            pending_writes=[]
        ),
        CheckpointTuple(
            config={...},
            checkpoint={
                'v': 3,
                'ts': '2025-05-05T16:01:22.278960+00:00',
                'id': '1f029ca3-0874-6612-8000-339f2abc83b1',
                'channel_versions': {'__start__': '00000000000000000000000000000002.0.18673090920108737', 'messages': '00000000000000000000000000000002.0.30296526818059655', 'branch:to:call_model': '00000000000000000000000000000002.0.9300422176788571'},
                'versions_seen': {'__input__': {}, '__start__': {'__start__': '00000000000000000000000000000001.0.7040775356287469'}},
                'channel_values': {'messages': [HumanMessage(content="hi! I'm bob")], 'branch:to:call_model': None}
            },
            metadata={'source': 'loop', 'writes': None, 'step': 0, 'parents': {}, 'thread_id': '1'},
            parent_config={...},
            pending_writes=[('8cbd75e0-3720-b056-04f7-71ac805140a0', 'messages', AIMessage(content='Hi Bob! How are you doing today? Is there anything I can help you with?'))]
        ),
        CheckpointTuple(
            config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1f029ca3-0870-6ce2-bfff-1f3f14c3e565'}},
            checkpoint={
                'v': 3,
                'ts': '2025-05-05T16:01:22.277497+00:00',
                'id': '1f029ca3-0870-6ce2-bfff-1f3f14c3e565',
                'channel_versions': {'__start__': '00000000000000000000000000000001.0.7040775356287469'},
                'versions_seen': {'__input__': {}},
                'channel_values': {'__start__': {'messages': [{'role': 'user', 'content': "hi! I'm bob"}]}}
            },
            metadata={'source': 'input', 'writes': {'__start__': {'messages': [{'role': 'user', 'content': "hi! I'm bob"}]}}, 'step': -1, 'parents': {}, 'thread_id': '1'},
            parent_config=None,
            pending_writes=[('d458367b-8265-812c-18e2-33001d199ce6', 'messages', [{'role': 'user', 'content': "hi! I'm bob"}]), ('d458367b-8265-812c-18e2-33001d199ce6', 'branch:to:call_model', None)]
        )
    ]
    ```

#### Xoá toàn bộ checkpoint của một thread

```python
thread_id = "1"
checkpointer.delete_thread(thread_id)
```

## Quản lý database

Nếu bạn dùng bất kỳ triển khai persistence nào được lưu trữ bởi database (như Postgres, Redis, hoặc Oracle) để lưu short-term và/hoặc long-term memory, bạn cần chạy migration để thiết lập schema cần thiết trước khi có thể dùng với database của mình.

Theo quy ước, hầu hết các thư viện đặc thù cho từng database định nghĩa một phương thức `setup()` trên instance checkpointer hoặc store để chạy các migration cần thiết. Tuy nhiên, bạn nên kiểm tra với triển khai [`BaseCheckpointSaver`](https://reference.langchain.com/python/langgraph/checkpoints/#langgraph.checkpoint.base.BaseCheckpointSaver) hoặc [`BaseStore`](https://reference.langchain.com/python/langchain-core/stores/BaseStore) cụ thể của bạn để xác nhận tên phương thức và cách dùng chính xác.

Chúng tôi khuyến nghị chạy migration như một bước triển khai (deployment) riêng biệt, hoặc bạn có thể đảm bảo chúng được chạy như một phần của quá trình khởi động server.

---

!!! info "Kết nối tài liệu này"
    [Kết nối tài liệu này](https://docs.langchain.com/use-these-docs) với Claude, VSCode, và nhiều công cụ khác qua MCP để có câu trả lời theo thời gian thực.

!!! info "Đóng góp"
    [Chỉnh sửa trang này trên GitHub](https://github.com/langchain-ai/docs/edit/main/src/oss/langgraph/add-memory.mdx) hoặc [báo lỗi](https://github.com/langchain-ai/docs/issues/new/choose).
