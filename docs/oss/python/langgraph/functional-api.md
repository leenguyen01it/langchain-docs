# Functional API overview

**Functional API** cho phép bạn thêm các tính năng cốt lõi của LangGraph ([persistence](./persistence.md), [memory](./add-memory.md), [human-in-the-loop](./interrupts.md), và [streaming](./streaming.md)) vào ứng dụng của mình với thay đổi tối thiểu đối với code hiện có.

Nó được thiết kế để tích hợp các tính năng này vào code hiện có, vốn có thể dùng các primitive ngôn ngữ chuẩn để rẽ nhánh và điều khiển luồng, như câu lệnh `if`, vòng lặp `for`, và lời gọi hàm. Khác với nhiều framework điều phối dữ liệu yêu cầu tái cấu trúc code thành một pipeline hoặc DAG tường minh, Functional API cho phép bạn tích hợp các khả năng này mà không áp đặt một mô hình thực thi cứng nhắc.

Functional API dùng hai khối xây dựng chính:

* **`@entrypoint`**: Đánh dấu một hàm là điểm bắt đầu của workflow, đóng gói logic và quản lý luồng thực thi, bao gồm xử lý các tác vụ chạy dài và interrupt.
* **[`@task`](https://reference.langchain.com/python/langgraph/func/task)**: Đại diện cho một đơn vị công việc rời rạc, như một lời gọi API hay một bước xử lý dữ liệu, có thể được thực thi bất đồng bộ bên trong một entrypoint. Task trả về một đối tượng dạng future có thể được await hoặc resolve đồng bộ.

Điều này cung cấp một tầng trừu tượng tối giản để xây dựng workflow với quản lý state và streaming.

!!! tip ""
    Để biết cách dùng functional API, xem [Use Functional API](./use-functional-api.md).

## Functional API so với Graph API

Với những người dùng thích cách tiếp cận khai báo (declarative) hơn, [Graph API](./graph-api.md) của LangGraph cho phép bạn định nghĩa workflow bằng mô hình Graph. Cả hai API đều dùng chung một runtime bên dưới, nên bạn có thể dùng chúng cùng nhau trong cùng một ứng dụng.

Dưới đây là một số khác biệt chính:

* **Control flow**: Functional API không yêu cầu bạn phải nghĩ về cấu trúc graph. Bạn có thể dùng các cấu trúc Python chuẩn để định nghĩa workflow. Điều này thường giúp giảm lượng code cần viết.
* **Short-term memory**: **Graph API** yêu cầu khai báo một [**State**](./graph-api.md#state) và có thể yêu cầu định nghĩa [**reducer**](./graph-api.md#reducers) để quản lý cập nhật cho state của graph. `@entrypoint` và `@task` không yêu cầu quản lý state tường minh vì state của chúng được giới hạn trong phạm vi hàm và không được chia sẻ giữa các hàm.
* **Checkpointing**: Cả hai API đều tạo ra và dùng checkpoint. Trong **Graph API**, một checkpoint mới được tạo sau mỗi [superstep](./graph-api.md). Trong **Functional API**, khi task được thực thi, kết quả của chúng được lưu vào một checkpoint đã có sẵn gắn với entrypoint tương ứng thay vì tạo checkpoint mới.
* **Trực quan hoá**: Graph API giúp dễ dàng trực quan hoá workflow dưới dạng graph, hữu ích cho việc debug, hiểu workflow, và chia sẻ với người khác. Functional API không hỗ trợ trực quan hoá vì graph được tạo động trong lúc chạy.

## Ví dụ

Dưới đây ta minh hoạ một ứng dụng đơn giản viết một bài luận và [interrupt](./interrupts.md) để yêu cầu con người xem xét.

```python
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.func import entrypoint, task
from langgraph.types import interrupt

@task
def write_essay(topic: str) -> str:
    """Write an essay about the given topic."""
    time.sleep(1) # A placeholder for a long-running task.
    return f"An essay about topic: {topic}"

@entrypoint(checkpointer=InMemorySaver())
def workflow(topic: str) -> dict:
    """A simple workflow that writes an essay and asks for a review."""
    essay = write_essay("cat").result()
    is_approved = interrupt({
        # Any json-serializable payload provided to interrupt as argument.
        # It will be surfaced on the client side as an Interrupt when streaming data
        # from the workflow.
        "essay": essay, # The essay we want reviewed.
        # We can add any additional information that we need.
        # For example, introduce a key called "action" with some instructions.
        "action": "Please approve/reject the essay",
    })

    return {
        "essay": essay, # The essay that was generated
        "is_approved": is_approved, # Response from HIL
    }
```

??? note "Giải thích chi tiết"
    Workflow này sẽ viết một bài luận về chủ đề "cat" rồi tạm dừng để lấy đánh giá từ con người. Workflow có thể bị interrupt trong một khoảng thời gian không xác định cho tới khi có đánh giá được cung cấp.

    Khi workflow được resume, nó thực thi lại từ đầu, nhưng vì kết quả của task `writeEssay` đã được lưu, kết quả của task sẽ được nạp từ checkpoint thay vì tính toán lại.

    ```python
    import time

    from langchain_core.utils.uuid import uuid7
    from langgraph.checkpoint.memory import InMemorySaver
    from langgraph.func import entrypoint, task
    from langgraph.types import Command, interrupt


    @task
    def write_essay(topic: str) -> str:
        """Write an essay about the given topic."""
        time.sleep(1)  # This is a placeholder for a long-running task.
        return f"An essay about topic: {topic}"


    @entrypoint(checkpointer=InMemorySaver())
    def workflow(topic: str) -> dict:
        """A simple workflow that writes an essay and asks for a review."""
        essay = write_essay("cat").result()
        is_approved = interrupt(
            {
                # Any json-serializable payload provided to interrupt as argument.
                # It will be surfaced on the client side as an Interrupt when streaming data
                # from the workflow.
                "essay": essay,  # The essay we want reviewed.
                # We can add any additional information that we need.
                # For example, introduce a key called "action" with some instructions.
                "action": "Please approve/reject the essay",
            }
        )
        return {
            "essay": essay,  # The essay that was generated
            "is_approved": is_approved,  # Response from HIL
        }


    thread_id = str(uuid7())
    config = {"configurable": {"thread_id": thread_id}}
    stream = workflow.stream_events("cat", config, version="v3")
    _ = stream.output
    print({"write_essay": stream.interrupts[0].value["essay"]})
    print({"__interrupt__": stream.interrupts})
    # {'write_essay': 'An essay about topic: cat'}
    # {
    #   '__interrupt__': [
    #     Interrupt(
    #       value={
    #           'essay': 'An essay about topic: cat',
    #           'action': 'Please approve/reject the essay'
    #       },
    #       id='369d44b3d93d4a631ae583367ac6b5cc'
    #     )
    #   ]
    # }
    ```

    Một bài luận đã được viết xong và sẵn sàng để xem xét. Khi đánh giá được cung cấp, ta có thể resume workflow:

    ```python
    # Get review from a user (e.g., via a UI)
    # In this case, we're using a bool, but this can be any json-serializable value.
    human_review = True

    resumed_stream = workflow.stream_events(Command(resume=human_review), config, version="v3")
    print(resumed_stream.output)
    # {'essay': 'An essay about topic: cat', 'is_approved': True}
    ```

    Workflow đã hoàn tất và đánh giá đã được thêm vào bài luận.

## Entrypoint

Decorator [`@entrypoint`](https://reference.langchain.com/python/langgraph/func/entrypoint) có thể được dùng để tạo một workflow từ một hàm. Nó đóng gói logic workflow và quản lý luồng thực thi, bao gồm xử lý *các tác vụ chạy dài* và [interrupt](./interrupts.md).

### Định nghĩa

Một **entrypoint** được định nghĩa bằng cách decorate một hàm với decorator `@entrypoint`.

Hàm **phải nhận đúng một tham số vị trí (positional)**, dùng làm input của workflow. Nếu bạn cần truyền nhiều dữ liệu, hãy dùng một dictionary làm kiểu input cho tham số đầu tiên.

Decorate một hàm với `entrypoint` sẽ tạo ra một instance [`Pregel`](https://reference.langchain.com/python/langgraph/pregel/#langgraph.pregel.Pregel.stream), giúp quản lý việc thực thi workflow (ví dụ: xử lý streaming, resumption, và checkpointing).

Bạn thường sẽ muốn truyền một **checkpointer** vào decorator `@entrypoint` để bật persistence và dùng các tính năng như **human-in-the-loop**.

=== "Sync"
    ```python
    from langgraph.func import entrypoint

    @entrypoint(checkpointer=checkpointer)
    def my_workflow(some_input: dict) -> int:
        # some logic that may involve long-running tasks like API calls,
        # and may be interrupted for human-in-the-loop.
        ...
        return result
    ```

=== "Async"
    ```python
    from langgraph.func import entrypoint

    @entrypoint(checkpointer=checkpointer)
    async def my_workflow(some_input: dict) -> int:
        # some logic that may involve long-running tasks like API calls,
        # and may be interrupted for human-in-the-loop
        ...
        return result
    ```

!!! warning "Serialization"
    **Input** và **output** của entrypoint phải JSON-serializable để hỗ trợ checkpointing. Xem phần [serialization](#serialization) để biết thêm chi tiết.

### Các tham số có thể inject

Khi khai báo một `entrypoint`, bạn có thể yêu cầu truy cập các tham số bổ sung sẽ được inject tự động lúc chạy. Các tham số này gồm:

| Tham số | Mô tả |
| ------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **previous** | Truy cập state gắn với `checkpoint` trước đó cho thread hiện tại. Xem [short-term-memory](#short-term-memory). |
| **store** | Một instance của \[BaseStore]\[langgraph.store.base.BaseStore]. Hữu ích cho [long-term memory](./use-functional-api.md#long-term-memory). |
| **writer** | Dùng để truy cập StreamWriter khi làm việc với Async Python \< 3.11. Xem [streaming với functional API để biết chi tiết](./use-functional-api.md#streaming). |
| **config** | Để truy cập cấu hình lúc chạy. Xem [RunnableConfig](https://python.langchain.com/docs/concepts/runnables/#runnableconfig) để biết thêm thông tin. |

!!! warning ""
    Khai báo các tham số với tên và type annotation phù hợp.

??? note "Yêu cầu các tham số có thể inject"
    ```python
    from langchain_core.runnables import RunnableConfig
    from langgraph.func import entrypoint
    from langgraph.store.base import BaseStore
    from langgraph.store.memory import InMemoryStore
    from langgraph.checkpoint.memory import InMemorySaver
    from langgraph.types import StreamWriter

    in_memory_checkpointer = InMemorySaver(...)
    in_memory_store = InMemoryStore(...)  # An instance of InMemoryStore for long-term memory

    @entrypoint(
        checkpointer=in_memory_checkpointer,  # Specify the checkpointer
        store=in_memory_store  # Specify the store
    )
    def my_workflow(
        some_input: dict,  # The input (e.g., passed via `invoke`)
        *,
        previous: Any = None, # For short-term memory
        store: BaseStore,  # For long-term memory
        writer: StreamWriter,  # For streaming custom data
        config: RunnableConfig  # For accessing the configuration passed to the entrypoint
    ) -> ...:
    ```

### Thực thi

Dùng [`@entrypoint`](#entrypoint) tạo ra một đối tượng [`Pregel`](https://reference.langchain.com/python/langgraph/pregel/#langgraph.pregel.Pregel.stream) có thể được thực thi bằng các phương thức `invoke`, `ainvoke`, `stream`, và `astream`.

=== "Invoke"
    ```python
    config = {
        "configurable": {
            "thread_id": "some_thread_id"
        }
    }
    my_workflow.invoke(some_input, config)  # Wait for the result synchronously
    ```

=== "Async Invoke"
    ```python
    config = {
        "configurable": {
            "thread_id": "some_thread_id"
        }
    }
    await my_workflow.ainvoke(some_input, config)  # Await result asynchronously
    ```

=== "Stream"
    ```python
    config = {
        "configurable": {
            "thread_id": "some_thread_id"
        }
    }

    stream = my_workflow.stream_events(some_input, config, version="v3")
    for message in stream.messages:
        for token in message.text:
            print(token, end="", flush=True)
    ```

=== "Async Stream"
    ```python
    config = {
        "configurable": {
            "thread_id": "some_thread_id"
        }
    }

    stream = await my_workflow.astream_events(some_input, config, version="v3")
    async for message in stream.messages:
        async for token in message.text:
            print(token, end="", flush=True)
    ```

### Resuming

Resume một lần thực thi sau một [interrupt](https://reference.langchain.com/python/langgraph/types/interrupt) có thể được thực hiện bằng cách truyền một giá trị **resume** vào primitive [`Command`](https://reference.langchain.com/python/langgraph/types/Command).

=== "Invoke"
    ```python
    from langgraph.types import Command

    config = {
        "configurable": {
            "thread_id": "some_thread_id"
        }
    }

    my_workflow.invoke(Command(resume=some_resume_value), config)
    ```

=== "Async Invoke"
    ```python
    from langgraph.types import Command

    config = {
        "configurable": {
            "thread_id": "some_thread_id"
        }
    }

    await my_workflow.ainvoke(Command(resume=some_resume_value), config)
    ```

=== "Stream"
    ```python
    from langgraph.types import Command

    config = {
        "configurable": {
            "thread_id": "some_thread_id"
        }
    }

    stream = my_workflow.stream_events(Command(resume=some_resume_value), config, version="v3")
    for message in stream.messages:
        for token in message.text:
            print(token, end="", flush=True)
    ```

=== "Async Stream"
    ```python
    from langgraph.types import Command

    config = {
        "configurable": {
            "thread_id": "some_thread_id"
        }
    }

    stream = await my_workflow.astream_events(Command(resume=some_resume_value), config, version="v3")
    async for message in stream.messages:
        async for token in message.text:
            print(token, end="", flush=True)
    ```

**Resuming sau lỗi**

Để resume sau một lỗi, chạy `entrypoint` với `None` và cùng **thread id** (config).

Điều này giả định rằng **lỗi** bên dưới đã được khắc phục và quá trình thực thi có thể tiếp tục thành công.

=== "Invoke"
    ```python

    config = {
        "configurable": {
            "thread_id": "some_thread_id"
        }
    }

    my_workflow.invoke(None, config)
    ```

=== "Async Invoke"
    ```python

    config = {
        "configurable": {
            "thread_id": "some_thread_id"
        }
    }

    await my_workflow.ainvoke(None, config)
    ```

=== "Stream"
    ```python

    config = {
        "configurable": {
            "thread_id": "some_thread_id"
        }
    }

    stream = my_workflow.stream_events(None, config, version="v3")
    for message in stream.messages:
        for token in message.text:
            print(token, end="", flush=True)
    ```

=== "Async Stream"
    ```python

    config = {
        "configurable": {
            "thread_id": "some_thread_id"
        }
    }

    stream = await my_workflow.astream_events(None, config, version="v3")
    async for message in stream.messages:
        async for token in message.text:
            print(token, end="", flush=True)
    ```

### Short-term memory

Khi một `entrypoint` được định nghĩa với một `checkpointer`, nó lưu thông tin giữa các lần gọi liên tiếp trên cùng một **thread id** trong [checkpoint](./checkpointers.md#checkpoints).

Điều này cho phép truy cập state từ lần gọi trước bằng tham số `previous`.

Theo mặc định, tham số `previous` là giá trị trả về của lần gọi trước đó.

```python
@entrypoint(checkpointer=checkpointer)
def my_workflow(number: int, *, previous: Any = None) -> int:
    previous = previous or 0
    return number + previous

config = {
    "configurable": {
        "thread_id": "some_thread_id"
    }
}

my_workflow.invoke(1, config)  # 1 (previous was None)
my_workflow.invoke(2, config)  # 3 (previous was 1 from the previous invocation)
```

#### `entrypoint.final`

[`entrypoint.final`](https://reference.langchain.com/python/langgraph/func/entrypoint/final) là một primitive đặc biệt có thể được trả về từ một entrypoint, cho phép **tách rời** giá trị **được lưu trong checkpoint** khỏi **giá trị trả về của entrypoint**.

Giá trị đầu tiên là giá trị trả về của entrypoint, còn giá trị thứ hai là giá trị sẽ được lưu vào checkpoint. Type annotation là `entrypoint.final[return_type, save_type]`.

```python
@entrypoint(checkpointer=checkpointer)
def my_workflow(number: int, *, previous: Any = None) -> entrypoint.final[int, int]:
    previous = previous or 0
    # This will return the previous value to the caller, saving
    # 2 * number to the checkpoint, which will be used in the next invocation
    # for the `previous` parameter.
    return entrypoint.final(value=previous, save=2 * number)

config = {
    "configurable": {
        "thread_id": "1"
    }
}

my_workflow.invoke(3, config)  # 0 (previous was None)
my_workflow.invoke(1, config)  # 6 (previous was 3 * 2 from the previous invocation)
```

## Task

Một **task** đại diện cho một đơn vị công việc rời rạc, như một lời gọi API hay một bước xử lý dữ liệu. Nó có hai đặc điểm chính:

* **Thực thi bất đồng bộ**: Task được thiết kế để thực thi bất đồng bộ, cho phép nhiều thao tác chạy đồng thời mà không bị chặn.
* **Checkpointing**: Kết quả task được lưu vào checkpoint, cho phép resume workflow từ trạng thái đã lưu gần nhất. (Xem [persistence](./persistence.md) để biết thêm chi tiết).

### Định nghĩa

Task được định nghĩa bằng decorator `@task`, bọc quanh một hàm Python thông thường.

```python
from langgraph.func import task

@task()
def slow_computation(input_value):
    # Simulate a long-running operation
    ...
    return result
```

!!! warning "Serialization"
    **Output** của task phải JSON-serializable để hỗ trợ checkpointing.

### Thực thi

**Task** chỉ có thể được gọi từ bên trong một **entrypoint**, một **task** khác, hoặc một [node của state graph](./graph-api.md#nodes).

Task *không thể* được gọi trực tiếp từ code ứng dụng chính.

Khi bạn gọi một **task**, nó trả về *ngay lập tức* một đối tượng future. Future là một placeholder cho một kết quả sẽ có sau.

Để lấy kết quả của một **task**, bạn có thể chờ đồng bộ (dùng `result()`) hoặc await bất đồng bộ (dùng `await`).

=== "Gọi đồng bộ"
    ```python
    @entrypoint(checkpointer=checkpointer)
    def my_workflow(some_input: int) -> int:
        future = slow_computation(some_input)
        return future.result()  # Wait for the result synchronously
    ```

=== "Gọi bất đồng bộ"
    ```python
    @entrypoint(checkpointer=checkpointer)
    async def my_workflow(some_input: int) -> int:
        return await slow_computation(some_input)  # Await result asynchronously
    ```

## Khi nào nên dùng task

**Task** hữu ích trong các tình huống sau:

* **Checkpointing**: Khi bạn cần lưu kết quả của một thao tác chạy dài vào checkpoint, để không phải tính toán lại khi resume workflow.
* **Human-in-the-loop**: Nếu bạn đang xây dựng một workflow cần sự can thiệp của con người, bạn PHẢI dùng **task** để đóng gói mọi yếu tố ngẫu nhiên (ví dụ: lời gọi API) nhằm đảm bảo workflow có thể resume đúng cách. Xem phần [determinism](#determinism) để biết thêm chi tiết.
* **Thực thi song song**: Với các tác vụ I/O-bound, **task** cho phép thực thi song song, giúp nhiều thao tác chạy đồng thời mà không bị chặn (ví dụ: gọi nhiều API).
* **Observability**: Bọc các thao tác trong **task** cung cấp cách theo dõi tiến trình của workflow và giám sát việc thực thi từng thao tác riêng lẻ bằng [LangSmith](https://docs.langchain.com/langsmith/observability).
* **Công việc có thể retry**: Khi công việc cần được retry để xử lý lỗi hoặc sự không nhất quán, **task** cung cấp cách đóng gói và quản lý logic retry.

## Serialization

Có hai khía cạnh chính của serialization trong LangGraph:

1. Input và output của `entrypoint` phải JSON-serializable.
2. Output của `task` phải JSON-serializable.

Các yêu cầu này cần thiết để bật checkpointing và resume workflow. Hãy dùng các kiểu Python nguyên thuỷ như dictionary, list, string, số, và boolean để đảm bảo input/output của bạn serializable.

Serialization đảm bảo rằng state của workflow, như kết quả task và các giá trị trung gian, có thể được lưu và khôi phục một cách đáng tin cậy. Điều này rất quan trọng để hỗ trợ tương tác human-in-the-loop, khả năng chịu lỗi (fault tolerance), và thực thi song song.

Cung cấp input/output không serializable sẽ gây ra lỗi runtime khi workflow được cấu hình với một checkpointer.

## Determinism

Khi bạn resume một lần chạy workflow, code **KHÔNG** resume từ **cùng dòng code** nơi thực thi dừng lại. Việc thực thi quay về ranh giới checkpoint gần nhất, và workflow **replay** tiến về phía trước cho tới khi đạt tới điểm tạm dừng lần nữa.

Với Functional API, replay bắt đầu từ đầu **entrypoint** trong khi LangGraph khôi phục kết quả [**task**](./functional-api.md#task) và [**subgraph**](./use-subgraphs.md) đã hoàn thành từ checkpointer thay vì tính toán lại chúng. Điều này giữ nguyên thứ tự các bước đã ghi lại qua các lần tạm dừng, kể cả với các output **task** chạy dài hoặc không xác định (non-deterministic).

Để dùng các tính năng như **human-in-the-loop**, bạn phải đặt các thao tác không xác định (ví dụ: giá trị ngẫu nhiên) và các side effect (ví dụ: ghi file hoặc lời gọi API) vào trong [**task**](./functional-api.md#task).

Các lần chạy khác nhau của một workflow có thể cho kết quả khác nhau, nhưng resume một thread **cụ thể** thì nên replay lại đúng các kết quả **task** và **subgraph** đã được lưu.

Để đảm bảo workflow của bạn xác định (deterministic) và có thể replay nhất quán, hãy tuân theo các nguyên tắc sau:

* **Tránh lặp lại công việc**: Trong một **entrypoint**, nếu bạn nối nhiều side effect (ví dụ: log, ghi file, hoặc gọi mạng), hãy tách mỗi cái thành một **task** riêng để khi resume, output của chúng được khôi phục từ checkpointer thay vì chạy lại.
* **Đóng gói các thao tác không xác định**: Giữ các giá trị có thể thay đổi giữa các lần thử (ví dụ: số ngẫu nhiên hoặc đọc đồng hồ hệ thống) bên trong **task**, để replay khớp với những gì đã được checkpoint.
* **Dùng các thao tác idempotent**: Với các trường hợp task thất bại một phần và retry, xem [Idempotency](#idempotency).

## Idempotency

Idempotency đảm bảo rằng chạy cùng một thao tác nhiều lần cho ra cùng một kết quả. Điều này giúp ngăn các lời gọi API trùng lặp và xử lý dư thừa nếu một bước bị chạy lại do lỗi. Luôn đặt các lời gọi API bên trong các hàm **task** để checkpointing, và thiết kế chúng idempotent phòng trường hợp thực thi lại.
Điều này đặc biệt quan trọng với các thao tác gây ra ghi dữ liệu.
Khi một workflow resume, LangGraph replay lại các kết quả **task** đã hoàn thành từ checkpoint. Một **task** đã bắt đầu nhưng chưa hoàn thành có thể chạy lại lúc resume, nên hãy thiết kế side effect idempotent. Dùng idempotency key hoặc kiểm tra kết quả hiện có để tránh trùng lặp ngoài ý muốn.

## Các lỗi thường gặp

### Xử lý side effect

Đóng gói các side effect (ví dụ: ghi file, gửi email) trong task để đảm bảo chúng không bị thực thi nhiều lần khi resume một workflow.

=== "Sai"
    Trong ví dụ này, một side effect (ghi file) được đưa trực tiếp vào workflow, nên nó sẽ bị thực thi lần thứ hai khi resume workflow.

    ```python
    @entrypoint(checkpointer=checkpointer)
    def my_workflow(inputs: dict) -> int:
        # This code will be executed a second time when resuming the workflow.
        # Which is likely not what you want.
        with open("output.txt", "w") as f:  # [!code highlight]
            f.write("Side effect executed")  # [!code highlight]
        value = interrupt("question")
        return value
    ```

=== "Đúng"
    Trong ví dụ này, side effect được đóng gói trong một task, đảm bảo thực thi nhất quán khi resume.

    ```python
    from langgraph.func import task

    @task  # [!code highlight]
    def write_to_file():  # [!code highlight]
        with open("output.txt", "w") as f:
            f.write("Side effect executed")

    @entrypoint(checkpointer=checkpointer)
    def my_workflow(inputs: dict) -> int:
        # The side effect is now encapsulated in a task.
        write_to_file().result()
        value = interrupt("question")
        return value
    ```

### Control flow không xác định

Các thao tác có thể cho kết quả khác nhau mỗi lần (như lấy thời gian hiện tại hoặc số ngẫu nhiên) nên được đóng gói trong task để đảm bảo khi resume, cùng một kết quả được trả về.

* Trong một task: Lấy số ngẫu nhiên (5) → interrupt → resume → (trả về lại 5) → ...
* Không trong task: Lấy số ngẫu nhiên (5) → interrupt → resume → lấy số ngẫu nhiên mới (7) → ...

Điều này đặc biệt quan trọng khi dùng workflow **human-in-the-loop** với nhiều lời gọi interrupt. LangGraph giữ một danh sách các giá trị resume cho mỗi task/entrypoint. Khi gặp một interrupt, nó được khớp với giá trị resume tương ứng. Việc khớp này hoàn toàn dựa trên **chỉ số (index)**, nên thứ tự các giá trị resume phải khớp với thứ tự các interrupt.

Nếu thứ tự thực thi không được giữ nguyên khi resume, một lời gọi [`interrupt`](https://reference.langchain.com/python/langgraph/types/interrupt) có thể bị khớp sai với giá trị `resume`, dẫn tới kết quả không chính xác.

Xem phần [determinism](#determinism) để biết thêm chi tiết.

=== "Sai"
    Trong ví dụ này, workflow dùng thời gian hiện tại để quyết định task nào sẽ thực thi. Đây là không xác định (non-deterministic) vì kết quả của workflow phụ thuộc vào thời điểm nó được thực thi.

    ```python
    from langgraph.func import entrypoint

    @entrypoint(checkpointer=checkpointer)
    def my_workflow(inputs: dict) -> int:
        t0 = inputs["t0"]
        t1 = time.time()  # [!code highlight]

        delta_t = t1 - t0

        if delta_t > 1:
            result = slow_task(1).result()
            value = interrupt("question")
        else:
            result = slow_task(2).result()
            value = interrupt("question")

        return {
            "result": result,
            "value": value
        }
    ```

=== "Đúng"
    Trong ví dụ này, workflow dùng input `t0` để quyết định task nào sẽ thực thi. Đây là xác định (deterministic) vì kết quả của workflow chỉ phụ thuộc vào input.

    ```python
    import time

    from langgraph.func import task

    @task  # [!code highlight]
    def get_time() -> float:  # [!code highlight]
        return time.time()

    @entrypoint(checkpointer=checkpointer)
    def my_workflow(inputs: dict) -> int:
        t0 = inputs["t0"]
        t1 = get_time().result()  # [!code highlight]

        delta_t = t1 - t0

        if delta_t > 1:
            result = slow_task(1).result()
            value = interrupt("question")
        else:
            result = slow_task(2).result()
            value = interrupt("question")

        return {
            "result": result,
            "value": value
        }
    ```

## Tìm hiểu thêm

* [Cách dùng Functional API](./use-functional-api.md)
* [Tổng quan khái niệm Graph API](./graph-api.md)
* [Chọn giữa Graph API và Functional API](./choosing-apis.md)

---

!!! info "Kết nối tài liệu này"
    [Kết nối tài liệu này](https://docs.langchain.com/use-these-docs) với Claude, VSCode, và nhiều công cụ khác qua MCP để có câu trả lời theo thời gian thực.

!!! info "Đóng góp"
    [Chỉnh sửa trang này trên GitHub](https://github.com/langchain-ai/docs/edit/main/src/oss/langgraph/functional-api.mdx) hoặc [báo lỗi](https://github.com/langchain-ai/docs/issues/new/choose).
