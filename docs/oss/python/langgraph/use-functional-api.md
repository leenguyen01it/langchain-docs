# Use the functional API

[**Functional API**](./functional-api.md) cho phép bạn thêm các tính năng cốt lõi của LangGraph ([persistence](./persistence.md), [memory](./add-memory.md), [human-in-the-loop](./interrupts.md), và [streaming](./streaming.md)) vào ứng dụng của mình với thay đổi tối thiểu đối với code hiện có.

!!! tip ""
    Để biết thông tin khái niệm về functional API, xem [Functional API](./functional-api.md).

## Tạo một workflow đơn giản

Khi định nghĩa một `entrypoint`, input bị giới hạn ở tham số đầu tiên của hàm. Để truyền nhiều input, bạn có thể dùng một dictionary.

```python
@entrypoint(checkpointer=checkpointer)
def my_workflow(inputs: dict) -> int:
    value = inputs["value"]
    another_value = inputs["another_value"]
    ...

my_workflow.invoke({"value": 1, "another_value": 2})
```

??? note "Ví dụ mở rộng: workflow đơn giản"
    ```python
    from langchain_core.utils.uuid import uuid7
    from langgraph.func import entrypoint, task
    from langgraph.checkpoint.memory import InMemorySaver

    # Task that checks if a number is even
    @task
    def is_even(number: int) -> bool:
        return number % 2 == 0

    # Task that formats a message
    @task
    def format_message(is_even: bool) -> str:
        return "The number is even." if is_even else "The number is odd."

    # Create a checkpointer for persistence
    checkpointer = InMemorySaver()

    @entrypoint(checkpointer=checkpointer)
    def workflow(inputs: dict) -> str:
        """Simple workflow to classify a number."""
        even = is_even(inputs["number"]).result()
        return format_message(even).result()

    # Run the workflow with a unique thread ID
    config = {"configurable": {"thread_id": str(uuid7())}}
    result = workflow.invoke({"number": 7}, config=config)
    print(result)
    ```

??? note "Ví dụ mở rộng: Soạn một bài luận với LLM"
    Ví dụ này minh hoạ cách dùng decorator `@task` và `@entrypoint`
    về mặt cú pháp. Vì một checkpointer đã được cung cấp, kết quả workflow sẽ
    được lưu vào checkpointer.

    ```python
    import uuid
    from langchain.chat_models import init_chat_model
    from langgraph.func import entrypoint, task
    from langgraph.checkpoint.memory import InMemorySaver

    model = init_chat_model('gpt-3.5-turbo')

    # Task: generate essay using an LLM
    @task
    def compose_essay(topic: str) -> str:
        """Generate an essay about the given topic."""
        return model.invoke([
            {"role": "system", "content": "You are a helpful assistant that writes essays."},
            {"role": "user", "content": f"Write an essay about {topic}."}
        ]).content

    # Create a checkpointer for persistence
    checkpointer = InMemorySaver()

    @entrypoint(checkpointer=checkpointer)
    def workflow(topic: str) -> str:
        """Simple workflow that generates an essay with an LLM."""
        return compose_essay(topic).result()

    # Execute the workflow
    config = {"configurable": {"thread_id": str(uuid7())}}
    result = workflow.invoke("the history of flight", config=config)
    print(result)
    ```

## Thực thi song song

Task có thể được thực thi song song bằng cách gọi chúng đồng thời và chờ kết quả. Điều này hữu ích để cải thiện hiệu năng trong các tác vụ IO-bound (ví dụ: gọi API cho LLM).

```python
@task
def add_one(number: int) -> int:
    return number + 1

@entrypoint(checkpointer=checkpointer)
def graph(numbers: list[int]) -> list[str]:
    futures = [add_one(i) for i in numbers]
    return [f.result() for f in futures]
```

??? note "Ví dụ mở rộng: gọi LLM song song"
    Ví dụ này minh hoạ cách chạy nhiều lời gọi LLM song song bằng `@task`. Mỗi lời gọi tạo một đoạn văn về một chủ đề khác nhau, và kết quả được ghép lại thành một output văn bản duy nhất.

    ```python
    import uuid
    from langchain.chat_models import init_chat_model
    from langgraph.func import entrypoint, task
    from langgraph.checkpoint.memory import InMemorySaver

    # Initialize the LLM model
    model = init_chat_model("gpt-3.5-turbo")

    # Task that generates a paragraph about a given topic
    @task
    def generate_paragraph(topic: str) -> str:
        response = model.invoke([
            {"role": "system", "content": "You are a helpful assistant that writes educational paragraphs."},
            {"role": "user", "content": f"Write a paragraph about {topic}."}
        ])
        return response.content

    # Create a checkpointer for persistence
    checkpointer = InMemorySaver()

    @entrypoint(checkpointer=checkpointer)
    def workflow(topics: list[str]) -> str:
        """Generates multiple paragraphs in parallel and combines them."""
        futures = [generate_paragraph(topic) for topic in topics]
        paragraphs = [f.result() for f in futures]
        return "\n\n".join(paragraphs)

    # Run the workflow
    config = {"configurable": {"thread_id": str(uuid7())}}
    result = workflow.invoke(["quantum computing", "climate change", "history of aviation"], config=config)
    print(result)
    ```

    Ví dụ này dùng mô hình concurrency của LangGraph để cải thiện thời gian thực thi, đặc biệt khi task liên quan tới I/O như hoàn thành LLM.

## Gọi graph

**Functional API** và [**Graph API**](./graph-api.md) có thể được dùng cùng nhau trong cùng một ứng dụng vì chúng dùng chung một runtime bên dưới.

```python
from langgraph.func import entrypoint
from langgraph.graph import StateGraph

builder = StateGraph()
...
some_graph = builder.compile()

@entrypoint()
def some_workflow(some_input: dict) -> int:
    # Call a graph defined using the graph API
    result_1 = some_graph.invoke(...)
    # Call another graph defined using the graph API
    result_2 = another_graph.invoke(...)
    return {
        "result_1": result_1,
        "result_2": result_2
    }
```

??? note "Ví dụ mở rộng: gọi một graph đơn giản từ functional API"
    ```python
    import uuid
    from typing import TypedDict
    from langgraph.func import entrypoint
    from langgraph.checkpoint.memory import InMemorySaver
    from langgraph.graph import StateGraph

    # Define the shared state type
    class State(TypedDict):
        foo: int

    # Define a simple transformation node
    def double(state: State) -> State:
        return {"foo": state["foo"] * 2}

    # Build the graph using the Graph API
    builder = StateGraph(State)
    builder.add_node("double", double)
    builder.set_entry_point("double")
    graph = builder.compile()

    # Define the functional API workflow
    checkpointer = InMemorySaver()

    @entrypoint(checkpointer=checkpointer)
    def workflow(x: int) -> dict:
        result = graph.invoke({"foo": x})
        return {"bar": result["foo"]}

    # Execute the workflow
    config = {"configurable": {"thread_id": str(uuid7())}}
    print(workflow.invoke(5, config=config))  # Output: {'bar': 10}
    ```

## Gọi các entrypoint khác

Bạn có thể gọi các **entrypoint** khác từ bên trong một **entrypoint** hoặc một **task**.

```python
@entrypoint() # Will automatically use the checkpointer from the parent entrypoint
def some_other_workflow(inputs: dict) -> int:
    return inputs["value"]

@entrypoint(checkpointer=checkpointer)
def my_workflow(inputs: dict) -> int:
    value = some_other_workflow.invoke({"value": 1})
    return value
```

??? note "Ví dụ mở rộng: gọi một entrypoint khác"
    ```python
    import uuid
    from langgraph.func import entrypoint
    from langgraph.checkpoint.memory import InMemorySaver

    # Initialize a checkpointer
    checkpointer = InMemorySaver()

    # A reusable sub-workflow that multiplies a number
    @entrypoint()
    def multiply(inputs: dict) -> int:
        return inputs["a"] * inputs["b"]

    # Main workflow that invokes the sub-workflow
    @entrypoint(checkpointer=checkpointer)
    def main(inputs: dict) -> dict:
        result = multiply.invoke({"a": inputs["x"], "b": inputs["y"]})
        return {"product": result}

    # Execute the main workflow
    config = {"configurable": {"thread_id": str(uuid7())}}
    print(main.invoke({"x": 6, "y": 7}, config=config))  # Output: {'product': 42}
    ```

## Streaming

**Functional API** dùng chung cơ chế streaming với **Graph API**. Hãy đọc
phần [**hướng dẫn streaming**](./streaming.md) để biết thêm chi tiết.

Ví dụ dùng streaming API để stream các value chunk từ một lần chạy workflow.

```python
config = {"configurable": {"thread_id": str(uuid7())}}

stream = main.stream_events({"x": 5}, config=config, version="v3")
for mode, chunk in stream.interleave("values"):
    print(f"{mode}: {chunk}")
# values: 10
```

1. Import [`get_stream_writer`](https://reference.langchain.com/python/langgraph/config/get_stream_writer) từ `langgraph.config`.
2. Lấy một instance stream writer bên trong entrypoint.
3. Emit dữ liệu tuỳ chỉnh trước khi bắt đầu tính toán.
4. Emit một message tuỳ chỉnh khác sau khi tính xong kết quả.
5. Dùng `stream_events()` để xử lý output đã stream.
6. Duyệt qua các cặp `(mode, chunk)` từ `interleave("values")`.

!!! warning "Async với Python \< 3.11"
    Nếu dùng Python \< 3.11 và viết code async, dùng [`get_stream_writer`](https://reference.langchain.com/python/langgraph/config/get_stream_writer) sẽ không hoạt động. Thay vào đó hãy
    dùng class `StreamWriter` trực tiếp. Xem [Async với Python \< 3.11](./streaming.md#async) để biết thêm chi tiết.

    ```python
    from langgraph.types import StreamWriter

    @entrypoint(checkpointer=checkpointer)
    async def main(inputs: dict, writer: StreamWriter) -> int:  # [!code highlight]
    ...
    ```

## Retry policy

```python
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.func import entrypoint, task
from langgraph.types import RetryPolicy

# This variable is just used for demonstration purposes to simulate a network failure.
# It's not something you will have in your actual code.
attempts = 0

# Let's configure the RetryPolicy to retry on ValueError.
# The default RetryPolicy is optimized for retrying specific network errors.
retry_policy = RetryPolicy(retry_on=ValueError)

@task(retry_policy=retry_policy)
def get_info():
    global attempts
    attempts += 1

    if attempts < 2:
        raise ValueError('Failure')
    return "OK"

checkpointer = InMemorySaver()

@entrypoint(checkpointer=checkpointer)
def main(inputs, writer):
    return get_info().result()

config = {
    "configurable": {
        "thread_id": "1"
    }
}

main.invoke({'any_input': 'foobar'}, config=config)
```

```pycon
'OK'
```

## Đặt timeout cho task và entrypoint

Dùng tham số `timeout` với `@task` hoặc `@entrypoint` để giới hạn thời gian một lần thử async có thể chạy. Cung cấp timeout theo giây hoặc dưới dạng `datetime.timedelta`.

```python
import asyncio

from langgraph.errors import NodeTimeoutError
from langgraph.func import entrypoint, task
from langgraph.types import RetryPolicy


@task(
    timeout=1.0,
    retry_policy=RetryPolicy(retry_on=NodeTimeoutError),
)
async def call_api(url: str) -> str:
    await asyncio.sleep(2)
    return f"result from {url}"


@entrypoint(timeout=5.0)
async def workflow(inputs: dict) -> str:
    return await call_api(inputs["url"])


try:
    await workflow.ainvoke({"url": "https://example.com"})
except NodeTimeoutError:
    print("Task timed out")
```

Timeout chỉ được hỗ trợ cho task và entrypoint async. Nếu bạn đặt `timeout` trên một hàm sync, LangGraph sẽ báo lỗi khi task hoặc entrypoint được khai báo.

Khi một task hoặc entrypoint vượt quá timeout, LangGraph báo lỗi `NodeTimeoutError`, kế thừa từ `TimeoutError` có sẵn của Python. Nếu một retry policy retry `TimeoutError` hoặc `NodeTimeoutError`, lần thử bị timeout sẽ được retry. Timeout áp dụng độc lập cho mỗi lần thử, nên bộ đếm giờ được reset cho mỗi lần retry.

## Cache task

```python
import time
from langgraph.cache.memory import InMemoryCache
from langgraph.func import entrypoint, task
from langgraph.types import CachePolicy


@task(cache_policy=CachePolicy(ttl=120))    # [!code highlight]
def slow_add(x: int) -> int:
    time.sleep(1)
    return x * 2


@entrypoint(cache=InMemoryCache())
def main(inputs: dict) -> dict[str, int]:
    result1 = slow_add(inputs["x"]).result()
    result2 = slow_add(inputs["x"]).result()
    return {"result1": result1, "result2": result2}


stream = main.stream_events({"x": 5}, version="v3")
for snapshot in stream.values:
    print(snapshot)
```

1. `ttl` được chỉ định theo giây. Cache sẽ bị vô hiệu hoá sau khoảng thời gian này.

## Resume sau lỗi

```python
import time
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.func import entrypoint, task
from langgraph.types import StreamWriter

# This variable is just used for demonstration purposes to simulate a network failure.
# It's not something you will have in your actual code.
attempts = 0

@task()
def get_info():
    """
    Simulates a task that fails once before succeeding.
    Raises an exception on the first attempt, then returns "OK" on subsequent tries.
    """
    global attempts
    attempts += 1

    if attempts < 2:
        raise ValueError("Failure")  # Simulate a failure on the first attempt
    return "OK"

# Initialize an in-memory checkpointer for persistence
checkpointer = InMemorySaver()

@task
def slow_task():
    """
    Simulates a slow-running task by introducing a 1-second delay.
    """
    time.sleep(1)
    return "Ran slow task."

@entrypoint(checkpointer=checkpointer)
def main(inputs, writer: StreamWriter):
    """
    Main workflow function that runs the slow_task and get_info tasks sequentially.

    Parameters:
    - inputs: Dictionary containing workflow input values.
    - writer: StreamWriter for streaming custom data.

    The workflow first executes `slow_task` and then attempts to execute `get_info`,
    which will fail on the first invocation.
    """
    slow_task_result = slow_task().result()  # Blocking call to slow_task
    get_info().result()  # Exception will be raised here on the first attempt
    return slow_task_result

# Workflow execution configuration with a unique thread identifier
config = {
    "configurable": {
        "thread_id": "1"  # Unique identifier to track workflow execution
    }
}

# This invocation will take ~1 second due to the slow_task execution
try:
    # First invocation will raise an exception due to the `get_info` task failing
    main.invoke({'any_input': 'foobar'}, config=config)
except ValueError:
    pass  # Handle the failure gracefully
```

Khi ta resume thực thi, ta sẽ không cần chạy lại `slow_task` vì kết quả của nó đã được lưu trong checkpoint.

```python
main.invoke(None, config=config)
```

```pycon
'Ran slow task.'
```

## Human-in-the-loop

Functional API hỗ trợ workflow [human-in-the-loop](./interrupts.md) bằng hàm [`interrupt`](https://reference.langchain.com/python/langgraph/types/interrupt) và primitive `Command`.

### Workflow human-in-the-loop cơ bản

Ta sẽ tạo ba [task](./functional-api.md#task):

1. Nối thêm `"bar"`.
2. Tạm dừng để chờ input con người. Khi resume, nối thêm input con người.
3. Nối thêm `"qux"`.

```python
from langgraph.func import entrypoint, task
from langgraph.types import Command, interrupt


@task
def step_1(input_query):
    """Append bar."""
    return f"{input_query} bar"


@task
def human_feedback(input_query):
    """Append user input."""
    feedback = interrupt(f"Please provide feedback: {input_query}")
    return f"{input_query} {feedback}"


@task
def step_3(input_query):
    """Append qux."""
    return f"{input_query} qux"
```

Giờ ta có thể kết hợp các task này trong một [entrypoint](./functional-api.md#entrypoint):

```python
from langgraph.checkpoint.memory import InMemorySaver

checkpointer = InMemorySaver()


@entrypoint(checkpointer=checkpointer)
def graph(input_query):
    result_1 = step_1(input_query).result()
    result_2 = human_feedback(result_1).result()
    result_3 = step_3(result_2).result()

    return result_3
```

[interrupt()](./interrupts.md#pause-using-interrupt) được gọi bên trong một task, cho phép con người xem xét và chỉnh sửa output của task trước đó. Kết quả của các task trước đó, trong trường hợp này là `step_1`, được lưu lại, nên chúng không chạy lại sau [`interrupt`](https://reference.langchain.com/python/langgraph/types/interrupt).

Hãy gửi một chuỗi query:

```python
config = {"configurable": {"thread_id": "1"}}

stream = graph.stream_events("foo", config, version="v3")
for message in stream.messages:
    for token in message.text:
        print(token, end="", flush=True)
```

Chú ý rằng ta đã tạm dừng với một [`interrupt`](https://reference.langchain.com/python/langgraph/types/interrupt) sau `step_1`. Interrupt cung cấp hướng dẫn để resume lần chạy. Để resume, ta gửi một [`Command`](./interrupts.md#resuming-interrupts) chứa dữ liệu mà task `human_feedback` mong đợi.

```python
# Continue execution
stream = graph.stream_events(Command(resume="baz"), config, version="v3")
for message in stream.messages:
    for token in message.text:
        print(token, end="", flush=True)
```

Sau khi resume, lần chạy tiếp tục qua bước còn lại và kết thúc như mong đợi.

### Xem xét lời gọi tool

Để xem xét lời gọi tool trước khi thực thi, ta thêm một hàm `review_tool_call` gọi [`interrupt`](./interrupts.md#pause-using-interrupt). Khi hàm này được gọi, thực thi sẽ tạm dừng cho tới khi ta gửi một command để resume nó.

Với một lời gọi tool cho trước, hàm của ta sẽ [`interrupt`](https://reference.langchain.com/python/langgraph/types/interrupt) để con người xem xét. Tại thời điểm đó ta có thể:

* Chấp nhận lời gọi tool
* Chỉnh sửa lời gọi tool và tiếp tục
* Tạo một tool message tuỳ chỉnh (ví dụ: yêu cầu model format lại lời gọi tool của nó)

```python
from typing import Union

def review_tool_call(tool_call: ToolCall) -> Union[ToolCall, ToolMessage]:
    """Review a tool call, returning a validated version."""
    human_review = interrupt(
        {
            "question": "Is this correct?",
            "tool_call": tool_call,
        }
    )
    review_action = human_review["action"]
    review_data = human_review.get("data")
    if review_action == "continue":
        return tool_call
    elif review_action == "update":
        updated_tool_call = {**tool_call, **{"args": review_data}}
        return updated_tool_call
    elif review_action == "feedback":
        return ToolMessage(
            content=review_data, name=tool_call["name"], tool_call_id=tool_call["id"]
        )
```

Giờ ta có thể cập nhật [entrypoint](./functional-api.md#entrypoint) của mình để xem xét các lời gọi tool được tạo ra. Nếu một lời gọi tool được chấp nhận hoặc chỉnh sửa, ta thực thi giống như trước. Ngược lại, ta chỉ nối thêm [`ToolMessage`](https://reference.langchain.com/python/langchain-core/messages/tool/ToolMessage) do con người cung cấp. Kết quả của các task trước đó, trong trường hợp này là lời gọi model ban đầu, được lưu lại, nên chúng không chạy lại sau [`interrupt`](https://reference.langchain.com/python/langgraph/types/interrupt).

```python
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.graph.message import add_messages
from langgraph.types import Command, interrupt


checkpointer = InMemorySaver()


@entrypoint(checkpointer=checkpointer)
def agent(messages, previous):
    if previous is not None:
        messages = add_messages(previous, messages)

    model_response = call_model(messages).result()
    while True:
        if not model_response.tool_calls:
            break

        # Review tool calls
        tool_results = []
        tool_calls = []
        for i, tool_call in enumerate(model_response.tool_calls):
            review = review_tool_call(tool_call)
            if isinstance(review, ToolMessage):
                tool_results.append(review)
            else:  # is a validated tool call
                tool_calls.append(review)
                if review != tool_call:
                    model_response.tool_calls[i] = review  # update message

        # Execute remaining tool calls
        tool_result_futures = [call_tool(tool_call) for tool_call in tool_calls]
        remaining_tool_results = [fut.result() for fut in tool_result_futures]

        # Append to message list
        messages = add_messages(
            messages,
            [model_response, *tool_results, *remaining_tool_results],
        )

        # Call model again
        model_response = call_model(messages).result()

    # Generate final response
    messages = add_messages(messages, model_response)
    return entrypoint.final(value=model_response, save=messages)
```

## Short-term memory

Short-term memory cho phép lưu thông tin qua các **lần gọi** khác nhau của cùng một **thread id**. Xem [short-term memory](./functional-api.md#short-term-memory) để biết thêm chi tiết.

### Quản lý checkpoint

Bạn có thể xem và xoá thông tin được lưu bởi checkpointer.

<a id="checkpoint" />

#### Xem state của thread

```python
config = {
    "configurable": {
        "thread_id": "1",  # [!code highlight]
        # optionally provide an ID for a specific checkpoint,
        # otherwise the latest checkpoint is shown
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

<a id="checkpoints" />

#### Xem lịch sử của thread

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

### Tách rời giá trị trả về khỏi giá trị được lưu

Dùng `entrypoint.final` để tách rời những gì trả về cho caller khỏi những gì được lưu trong checkpoint. Điều này hữu ích khi:

* Bạn muốn trả về một kết quả đã tính (ví dụ: một bản tóm tắt hoặc status), nhưng lưu một giá trị nội bộ khác để dùng cho lần gọi tiếp theo.
* Bạn cần kiểm soát những gì được truyền vào tham số `previous` ở lần chạy tiếp theo.

```python
from langgraph.func import entrypoint
from langgraph.checkpoint.memory import InMemorySaver

checkpointer = InMemorySaver()

@entrypoint(checkpointer=checkpointer)
def accumulate(n: int, *, previous: int | None) -> entrypoint.final[int, int]:
    previous = previous or 0
    total = previous + n
    # Return the *previous* value to the caller but save the *new* total to the checkpoint.
    return entrypoint.final(value=previous, save=total)

config = {"configurable": {"thread_id": "my-thread"}}

print(accumulate.invoke(1, config=config))  # 0
print(accumulate.invoke(2, config=config))  # 1
print(accumulate.invoke(3, config=config))  # 3
```

### Ví dụ chatbot

Một ví dụ về chatbot đơn giản dùng functional API và checkpointer [`InMemorySaver`](https://reference.langchain.com/python/langgraph/checkpoints/#langgraph.checkpoint.memory.InMemorySaver).

Bot có thể nhớ cuộc hội thoại trước đó và tiếp tục từ nơi nó dừng lại.

```python
from langchain.messages import BaseMessage
from langgraph.graph import add_messages
from langgraph.func import entrypoint, task
from langgraph.checkpoint.memory import InMemorySaver
from langchain_anthropic import ChatAnthropic

model = ChatAnthropic(model="claude-sonnet-4-6")

@task
def call_model(messages: list[BaseMessage]):
    response = model.invoke(messages)
    return response

checkpointer = InMemorySaver()

@entrypoint(checkpointer=checkpointer)
def workflow(inputs: list[BaseMessage], *, previous: list[BaseMessage]):
    if previous:
        inputs = add_messages(previous, inputs)

    response = call_model(inputs).result()
    return entrypoint.final(value=response, save=add_messages(inputs, response))

config = {"configurable": {"thread_id": "1"}}
input_message = {"role": "user", "content": "hi! I'm bob"}
stream = workflow.stream_events([input_message], config, version="v3")
for snapshot in stream.values:
    print(snapshot)

input_message = {"role": "user", "content": "what's my name?"}
stream = workflow.stream_events([input_message], config, version="v3")
for snapshot in stream.values:
    print(snapshot)
```

## Long-term memory

[long-term memory](https://docs.langchain.com/oss/python/concepts/memory#long-term-memory) cho phép lưu thông tin qua các **thread id** khác nhau. Điều này hữu ích để học thông tin về một người dùng cụ thể trong một cuộc hội thoại và dùng nó trong cuộc hội thoại khác.

## Workflow

* Hướng dẫn [Workflow và agent](./workflows-agents.md) có thêm ví dụ về cách xây dựng workflow bằng Functional API.

## Tích hợp với các thư viện khác

* [Thêm tính năng của LangGraph vào các framework khác bằng functional API](https://docs.langchain.com/langsmith/deploy-other-frameworks): Thêm các tính năng LangGraph như persistence, memory và streaming vào các framework agent khác không có sẵn những tính năng này.

---

!!! info "Kết nối tài liệu này"
    [Kết nối tài liệu này](https://docs.langchain.com/use-these-docs) với Claude, VSCode, và nhiều công cụ khác qua MCP để có câu trả lời theo thời gian thực.

!!! info "Đóng góp"
    [Chỉnh sửa trang này trên GitHub](https://github.com/langchain-ai/docs/edit/main/src/oss/langgraph/use-functional-api.mdx) hoặc [báo lỗi](https://github.com/langchain-ai/docs/issues/new/choose).
