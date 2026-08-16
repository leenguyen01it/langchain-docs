# Sử dụng Graph API

Hướng dẫn này trình bày những kiến thức cơ bản về Graph API của LangGraph. Nội dung đi qua [state](#dinh-nghia-va-cap-nhat-state), cũng như cách kết hợp các cấu trúc graph phổ biến như [chuỗi tuần tự](#tao-mot-chuoi-cac-buoc), [nhánh rẽ](#tao-nhanh-re), và [vòng lặp](#tao-va-kiem-soat-vong-lap). Hướng dẫn cũng bao gồm các tính năng điều khiển của LangGraph, bao gồm [Send API](#map-reduce-va-send-api) cho các luồng công việc map-reduce và [Command API](#ket-hop-dieu-khien-luong-va-cap-nhat-state-voi-command) để kết hợp cập nhật state với việc "nhảy" (hop) giữa các node.

## Thiết lập

Cài đặt `langgraph`:

=== "pip"
    ```bash
    pip install -U langgraph
    ```

=== "uv"
    ```bash
    uv add langgraph
    ```

!!! tip "Thiết lập LangSmith để debug tốt hơn"
    Đăng ký [LangSmith](https://smith.langchain.com?utm_source=docs&utm_medium=cta&utm_campaign=langsmith-signup&utm_content=oss-langgraph-use-graph-api) để nhanh chóng phát hiện vấn đề và cải thiện hiệu năng cho các dự án LangGraph của bạn. LangSmith cho phép bạn dùng dữ liệu trace để debug, test, và giám sát các ứng dụng LLM được xây dựng bằng LangGraph, đọc thêm về cách bắt đầu trong [tài liệu](https://docs.langchain.com/langsmith/observability).

## Định nghĩa và cập nhật state

Ở đây chúng ta sẽ xem cách định nghĩa và cập nhật [state](./graph-api.md#state) trong LangGraph. Chúng ta sẽ trình bày:

1. Cách dùng state để định nghĩa [schema](./graph-api.md#schema) của một graph
2. Cách dùng [reducer](./graph-api.md#reducers) để kiểm soát cách các cập nhật state được xử lý.

### Định nghĩa state

[State](./graph-api.md#state) trong LangGraph có thể là `TypedDict`, model `Pydantic`, hoặc dataclass. Bên dưới chúng ta sẽ dùng `TypedDict`. Xem [Dùng model Pydantic cho state của graph](#dung-model-pydantic-cho-state-cua-graph) để biết chi tiết về việc dùng Pydantic.

Mặc định, các graph sẽ có cùng schema đầu vào và đầu ra, và state quyết định schema đó. Xem [Định nghĩa schema đầu vào và đầu ra](#dinh-nghia-schema-dau-vao-va-dau-ra) để biết cách định nghĩa các schema đầu vào và đầu ra khác nhau.

Hãy xem xét một ví dụ đơn giản dùng [messages](./graph-api.md#messagesstate). Đây là một cách biểu diễn state linh hoạt cho nhiều ứng dụng LLM. Xem [trang khái niệm](./graph-api.md#lam-viec-voi-messages-trong-state-cua-graph) của chúng tôi để biết thêm chi tiết.

```python
from langchain.messages import AnyMessage
from typing_extensions import TypedDict

class State(TypedDict):
    messages: list[AnyMessage]
    extra_field: int
```

State này theo dõi một danh sách các object [message](https://python.langchain.com/docs/concepts/messages/), cùng với một field số nguyên bổ sung.

### Cập nhật state

Hãy xây dựng một graph ví dụ với một node duy nhất. [Node](./graph-api.md#nodes) của chúng ta chỉ đơn giản là một hàm Python đọc state của graph và thực hiện các cập nhật lên nó. Đối số đầu tiên của hàm này luôn là state:

```python
from langchain.messages import AIMessage

def node(state: State):
    messages = state["messages"]
    new_message = AIMessage("Hello!")
    return {"messages": messages + [new_message], "extra_field": 10}
```

Node này chỉ đơn giản là thêm một message vào danh sách message của chúng ta, và điền vào một field bổ sung.

!!! warning "Cảnh báo"
    Node nên trả về các cập nhật cho state một cách trực tiếp, thay vì thay đổi (mutate) state.

Tiếp theo, hãy định nghĩa một graph đơn giản chứa node này. Chúng ta dùng [`StateGraph`](./graph-api.md#stategraph) để định nghĩa một graph hoạt động trên state này. Sau đó chúng ta dùng [`add_node`](./graph-api.md#nodes) để điền dữ liệu vào graph.

```python
from langgraph.graph import StateGraph

builder = StateGraph(State)
builder.add_node(node)
builder.set_entry_point("node")
graph = builder.compile()
```

LangGraph cung cấp các tiện ích có sẵn để trực quan hóa graph của bạn. Hãy kiểm tra graph của chúng ta. Xem [Trực quan hóa graph của bạn](#truc-quan-hoa-graph-cua-ban) để biết chi tiết về trực quan hóa.

```python
from IPython.display import Image, display

display(Image(graph.get_graph().draw_mermaid_png()))
```

Trong trường hợp này, graph của chúng ta chỉ thực thi một node duy nhất. Hãy tiếp tục với một lần gọi (invocation) đơn giản:

```python
from langchain.messages import HumanMessage

result = graph.invoke({"messages": [HumanMessage("Hi")]})
result
```

```
{'messages': [HumanMessage(content='Hi'), AIMessage(content='Hello!')], 'extra_field': 10}
```

Lưu ý rằng:

* Chúng ta khởi động việc gọi bằng cách cập nhật một key duy nhất của state.
* Chúng ta nhận về toàn bộ state trong kết quả gọi.

Để thuận tiện, chúng ta thường xuyên kiểm tra nội dung của [message object](https://python.langchain.com/docs/concepts/messages/) qua pretty-print:

```python
for message in result["messages"]:
    message.pretty_print()
```

```
================================ Human Message ================================

Hi
================================== Ai Message ==================================

Hello!
```

### Xử lý cập nhật state với reducer

Mỗi key trong state có thể có hàm [reducer](./graph-api.md#reducers) riêng, điều khiển cách các cập nhật từ node được áp dụng. Nếu không có hàm reducer nào được chỉ định rõ ràng thì mặc định mọi cập nhật cho key đó sẽ ghi đè nó.

Với schema state kiểu `TypedDict`, chúng ta có thể định nghĩa reducer bằng cách gắn annotation cho field tương ứng của state với một hàm reducer.

Trong ví dụ trước, node của chúng ta cập nhật key `"messages"` trong state bằng cách thêm một message vào đó. Bên dưới, chúng ta thêm một reducer cho key này, để các cập nhật tự động được nối thêm vào:

```python
from typing_extensions import Annotated

def add(left, right):
    """Cũng có thể import `add` từ built-in `operator`."""
    return left + right

class State(TypedDict):
    messages: Annotated[list[AnyMessage], add]  # [!code highlight]
    extra_field: int
```

Giờ node của chúng ta có thể được đơn giản hóa:

```python
def node(state: State):
    new_message = AIMessage("Hello!")
    return {"messages": [new_message], "extra_field": 10}  # [!code highlight]
```

```python
from langgraph.graph import START

graph = StateGraph(State).add_node(node).add_edge(START, "node").compile()

result = graph.invoke({"messages": [HumanMessage("Hi")]})

for message in result["messages"]:
    message.pretty_print()
```

```
================================ Human Message ================================

Hi
================================== Ai Message ==================================

Hello!
```

#### MessagesState

Trong thực tế, có thêm một số điều cần cân nhắc khi cập nhật danh sách message:

* Chúng ta có thể muốn cập nhật một message đã tồn tại trong state.
* Chúng ta có thể muốn chấp nhận các dạng viết tắt cho [định dạng message](./graph-api.md#su-dung-messages-trong-graph-cua-ban), chẳng hạn như [định dạng OpenAI](https://python.langchain.com/docs/concepts/messages/#openai-format).

LangGraph có sẵn một reducer là [`add_messages`](https://reference.langchain.com/python/langgraph/graph/message/add_messages) xử lý các cân nhắc này:

```python
from langgraph.graph.message import add_messages

class State(TypedDict):
    messages: Annotated[list[AnyMessage], add_messages]  # [!code highlight]
    extra_field: int

def node(state: State):
    new_message = AIMessage("Hello!")
    return {"messages": [new_message], "extra_field": 10}

graph = StateGraph(State).add_node(node).set_entry_point("node").compile()
```

```python
input_message = {"role": "user", "content": "Hi"}  # [!code highlight]

result = graph.invoke({"messages": [input_message]})

for message in result["messages"]:
    message.pretty_print()
```

```
================================ Human Message ================================

Hi
================================== Ai Message ==================================

Hello!
```

Đây là một cách biểu diễn state linh hoạt cho các ứng dụng liên quan đến [chat model](https://python.langchain.com/docs/concepts/chat_models/). LangGraph có sẵn một `MessagesState` dựng sẵn để tiện dùng, để chúng ta có thể viết:

```python
from langgraph.graph import MessagesState

class State(MessagesState):
    extra_field: int
```

### Bỏ qua reducer với `Overwrite`

Trong một số trường hợp, bạn có thể muốn bỏ qua một reducer và ghi đè trực tiếp một giá trị state. LangGraph cung cấp kiểu [`Overwrite`](https://reference.langchain.com/python/langgraph/types/) cho mục đích này. Khi một node trả về một giá trị được bọc bởi `Overwrite`, reducer sẽ bị bỏ qua và channel được set trực tiếp thành giá trị đó.

Điều này hữu ích khi bạn muốn reset hoặc thay thế state đã tích lũy thay vì merge nó với các giá trị hiện có.

```python
from langgraph.graph import StateGraph, START, END
from langgraph.types import Overwrite
from typing_extensions import Annotated, TypedDict
import operator

class State(TypedDict):
    messages: Annotated[list, operator.add]

def add_message(state: State):
    return {"messages": ["first message"]}

def replace_messages(state: State):
    # Bỏ qua reducer và thay thế toàn bộ danh sách messages
    return {"messages": Overwrite(["replacement message"])}

builder = StateGraph(State)
builder.add_node("add_message", add_message)
builder.add_node("replace_messages", replace_messages)
builder.add_edge(START, "add_message")
builder.add_edge("add_message", "replace_messages")
builder.add_edge("replace_messages", END)

graph = builder.compile()

result = graph.invoke({"messages": ["initial"]})
print(result["messages"])
```

```
['replacement message']
```

Bạn cũng có thể dùng định dạng JSON với key đặc biệt `"__overwrite__"`:

```python
def replace_messages(state: State):
    return {"messages": {"__overwrite__": ["replacement message"]}}
```

!!! warning "Cảnh báo"
    Khi các node thực thi song song, chỉ một node được phép dùng `Overwrite` trên cùng một state key trong một super-step nhất định. Nếu nhiều node cùng cố gắng ghi đè cùng một key trong cùng một super-step, một `InvalidUpdateError` sẽ được raise.

### Định nghĩa schema đầu vào và đầu ra

Mặc định, `StateGraph` hoạt động với một schema duy nhất, và mọi node đều được kỳ vọng giao tiếp bằng schema đó. Tuy nhiên, cũng có thể định nghĩa các schema đầu vào và đầu ra khác nhau cho một graph.

Khi các schema khác nhau được chỉ định, một schema nội bộ vẫn sẽ được dùng để giao tiếp giữa các node. Schema đầu vào đảm bảo rằng đầu vào được cung cấp khớp với cấu trúc mong đợi, trong khi schema đầu ra lọc dữ liệu nội bộ để chỉ trả về thông tin liên quan theo schema đầu ra đã định nghĩa.

Bên dưới, chúng ta sẽ xem cách định nghĩa schema đầu vào và đầu ra khác nhau.

```python
from langgraph.graph import StateGraph, START, END
from typing_extensions import TypedDict

# Định nghĩa schema cho đầu vào
class InputState(TypedDict):
    question: str

# Định nghĩa schema cho đầu ra
class OutputState(TypedDict):
    answer: str

# Định nghĩa schema tổng thể, kết hợp cả đầu vào và đầu ra
class OverallState(InputState, OutputState):
    pass

# Định nghĩa node xử lý đầu vào và tạo ra câu trả lời
def answer_node(state: InputState):
    # Câu trả lời ví dụ và một key bổ sung
    return {"answer": "bye", "question": state["question"]}

# Xây dựng graph với schema đầu vào và đầu ra được chỉ định
builder = StateGraph(OverallState, input_schema=InputState, output_schema=OutputState)
builder.add_node(answer_node)  # Thêm node answer
builder.add_edge(START, "answer_node")  # Định nghĩa edge bắt đầu
builder.add_edge("answer_node", END)  # Định nghĩa edge kết thúc
graph = builder.compile()  # Compile graph

# Gọi graph với một đầu vào và in kết quả
print(graph.invoke({"question": "hi"}))
```

```
{'answer': 'bye'}
```

Lưu ý rằng đầu ra của invoke chỉ bao gồm schema đầu ra.

### Truyền private state giữa các node

Trong một số trường hợp, bạn có thể muốn các node trao đổi thông tin quan trọng cho logic trung gian nhưng không cần là một phần của schema chính của graph. Dữ liệu riêng tư (private) này không liên quan đến đầu vào/đầu ra tổng thể của graph và chỉ nên được chia sẻ giữa một số node nhất định.

Bên dưới, chúng ta sẽ tạo một graph tuần tự ví dụ gồm ba node (node_1, node_2 và node_3), trong đó dữ liệu riêng tư được truyền giữa hai bước đầu (node_1 và node_2), còn bước thứ ba (node_3) chỉ có quyền truy cập vào state tổng thể công khai.

```python
from langgraph.graph import StateGraph, START, END
from typing_extensions import TypedDict

# State tổng thể của graph (đây là state công khai được chia sẻ giữa các node)
class OverallState(TypedDict):
    a: str

# Đầu ra từ node_1 chứa dữ liệu riêng tư không phải một phần của state tổng thể
class Node1Output(TypedDict):
    private_data: str

# Dữ liệu riêng tư chỉ được chia sẻ giữa node_1 và node_2
def node_1(state: OverallState) -> Node1Output:
    output = {"private_data": "set by node_1"}
    print(f"Entered node `node_1`:\n\tInput: {state}.\n\tReturned: {output}")
    return output

# Đầu vào của node 2 chỉ yêu cầu dữ liệu riêng tư có sẵn sau node_1
class Node2Input(TypedDict):
    private_data: str

def node_2(state: Node2Input) -> OverallState:
    output = {"a": "set by node_2"}
    print(f"Entered node `node_2`:\n\tInput: {state}.\n\tReturned: {output}")
    return output

# Node 3 chỉ có quyền truy cập state tổng thể (không truy cập được dữ liệu riêng tư từ node_1)
def node_3(state: OverallState) -> OverallState:
    output = {"a": "set by node_3"}
    print(f"Entered node `node_3`:\n\tInput: {state}.\n\tReturned: {output}")
    return output

# Kết nối các node theo trình tự
# node_2 nhận dữ liệu riêng tư từ node_1, trong khi
# node_3 không thấy được dữ liệu riêng tư đó.
builder = StateGraph(OverallState).add_sequence([node_1, node_2, node_3])
builder.add_edge(START, "node_1")
graph = builder.compile()

# Gọi graph với state ban đầu
response = graph.invoke(
    {
        "a": "set at start",
    }
)

print()
print(f"Output of graph invocation: {response}")
```

```
Entered node `node_1`:
    Input: {'a': 'set at start'}.
    Returned: {'private_data': 'set by node_1'}
Entered node `node_2`:
    Input: {'private_data': 'set by node_1'}.
    Returned: {'a': 'set by node_2'}
Entered node `node_3`:
    Input: {'a': 'set by node_2'}.
    Returned: {'a': 'set by node_3'}

Output of graph invocation: {'a': 'set by node_3'}
```

### Dùng model Pydantic cho state của graph

Một [StateGraph](https://reference.langchain.com/python/langgraph/graph/state/StateGraph) chấp nhận đối số [`state_schema`](https://reference.langchain.com/python/langchain/middleware/#langchain.agents.middleware.AgentMiddleware.state_schema) khi khởi tạo, xác định "hình dạng" của state mà các node trong graph có thể truy cập và cập nhật.

Trong các ví dụ của chúng ta, chúng ta thường dùng `TypedDict` gốc của Python hoặc [`dataclass`](https://docs.python.org/3/library/dataclasses.html) cho `state_schema`, nhưng [`state_schema`](https://reference.langchain.com/python/langchain/middleware/#langchain.agents.middleware.AgentMiddleware.state_schema) có thể là bất kỳ [type](https://docs.python.org/3/library/stdtypes.html#type-objects) nào.

Ở đây, chúng ta sẽ xem cách [Pydantic BaseModel](https://docs.pydantic.dev/latest/api/base_model/) có thể được dùng cho [`state_schema`](https://reference.langchain.com/python/langchain/middleware/#langchain.agents.middleware.AgentMiddleware.state_schema) để thêm xác thực runtime trên **đầu vào**.

!!! note "Các giới hạn đã biết"
    * Hiện tại, đầu ra của graph sẽ **KHÔNG** phải là một instance của model pydantic.
    * Xác thực runtime chỉ xảy ra trên đầu vào của node đầu tiên trong graph, không xảy ra ở các node hoặc đầu ra tiếp theo.
    * Trace lỗi xác thực từ pydantic không cho biết node nào phát sinh lỗi.
    * Xác thực đệ quy của Pydantic có thể chậm. Với các ứng dụng nhạy cảm về hiệu năng, bạn có thể cân nhắc dùng `dataclass` thay thế.

```python
from langgraph.graph import StateGraph, START, END
from typing_extensions import TypedDict
from pydantic import BaseModel

# State tổng thể của graph (đây là state công khai được chia sẻ giữa các node)
class OverallState(BaseModel):
    a: str

def node(state: OverallState):
    return {"a": "goodbye"}

# Xây dựng state graph
builder = StateGraph(OverallState)
builder.add_node(node)  # node_1 là node đầu tiên
builder.add_edge(START, "node")  # Bắt đầu graph với node_1
builder.add_edge("node", END)  # Kết thúc graph sau node_1
graph = builder.compile()

# Test graph với một đầu vào hợp lệ
graph.invoke({"a": "hello"})
```

Gọi graph với một đầu vào **không hợp lệ**

```python
try:
    graph.invoke({"a": 123})  # Nên là một string
except Exception as e:
    print("An exception was raised because `a` is an integer rather than a string.")
    print(e)
```

```
An exception was raised because `a` is an integer rather than a string.
1 validation error for OverallState
a
  Input should be a valid string [type=string_type, input_value=123, input_type=int]
    For further information visit https://errors.pydantic.dev/2.9/v/string_type
```

Xem bên dưới để biết thêm các tính năng khác của state kiểu Pydantic model:

??? note "Hành vi serialization"
    Khi dùng model Pydantic làm state schema, điều quan trọng là hiểu cách serialization hoạt động, đặc biệt khi:

    * Truyền các object Pydantic làm đầu vào
    * Nhận đầu ra từ graph
    * Làm việc với các model Pydantic lồng nhau

    Hãy xem các hành vi này khi hoạt động.

    ```python
    from langgraph.graph import StateGraph, START, END
    from pydantic import BaseModel

    class NestedModel(BaseModel):
        value: str

    class ComplexState(BaseModel):
        text: str
        count: int
        nested: NestedModel

    def process_node(state: ComplexState):
        # Node nhận một object Pydantic đã được xác thực
        print(f"Input state type: {type(state)}")
        print(f"Nested type: {type(state.nested)}")
        # Trả về một cập nhật dictionary
        return {"text": state.text + " processed", "count": state.count + 1}

    # Xây dựng graph
    builder = StateGraph(ComplexState)
    builder.add_node("process", process_node)
    builder.add_edge(START, "process")
    builder.add_edge("process", END)
    graph = builder.compile()

    # Tạo một instance Pydantic làm đầu vào
    input_state = ComplexState(text="hello", count=0, nested=NestedModel(value="test"))
    print(f"Input object type: {type(input_state)}")

    # Gọi graph với một instance Pydantic
    result = graph.invoke(input_state)
    print(f"Output type: {type(result)}")
    print(f"Output content: {result}")

    # Chuyển đổi ngược lại thành model Pydantic nếu cần
    output_model = ComplexState(**result)
    print(f"Converted back to Pydantic: {type(output_model)}")
    ```

??? note "Ép kiểu runtime (runtime type coercion)"
    Pydantic thực hiện ép kiểu runtime cho một số kiểu dữ liệu nhất định. Điều này có thể hữu ích nhưng cũng có thể dẫn đến hành vi không mong muốn nếu bạn không để ý.

    ```python
    from langgraph.graph import StateGraph, START, END
    from pydantic import BaseModel

    class CoercionExample(BaseModel):
        # Pydantic sẽ ép kiểu string số thành integer
        number: int
        # Pydantic sẽ parse string boolean thành bool
        flag: bool

    def inspect_node(state: CoercionExample):
        print(f"number: {state.number} (type: {type(state.number)})")
        print(f"flag: {state.flag} (type: {type(state.flag)})")
        return {}

    builder = StateGraph(CoercionExample)
    builder.add_node("inspect", inspect_node)
    builder.add_edge(START, "inspect")
    builder.add_edge("inspect", END)
    graph = builder.compile()

    # Minh họa việc ép kiểu với đầu vào string sẽ được chuyển đổi
    result = graph.invoke({"number": "42", "flag": "true"})

    # Điều này sẽ lỗi với một validation error
    try:
        graph.invoke({"number": "not-a-number", "flag": "true"})
    except Exception as e:
        print(f"\nExpected validation error: {e}")
    ```

??? note "Làm việc với Message Model"
    Khi làm việc với các kiểu message của LangChain trong state schema, có những điều quan trọng cần lưu ý về serialization. Bạn nên dùng `AnyMessage` (thay vì `BaseMessage`) để serialization/deserialization đúng cách khi truyền message object qua lại.

    ```python
    from langgraph.graph import StateGraph, START, END
    from pydantic import BaseModel
    from langchain.messages import HumanMessage, AIMessage, AnyMessage
    from typing import List

    class ChatState(BaseModel):
        messages: List[AnyMessage]
        context: str

    def add_message(state: ChatState):
        return {"messages": state.messages + [AIMessage(content="Hello there!")]}

    builder = StateGraph(ChatState)
    builder.add_node("add_message", add_message)
    builder.add_edge(START, "add_message")
    builder.add_edge("add_message", END)
    graph = builder.compile()

    # Tạo đầu vào với một message
    initial_state = ChatState(
        messages=[HumanMessage(content="Hi")], context="Customer support chat"
    )

    result = graph.invoke(initial_state)
    print(f"Output: {result}")

    # Chuyển đổi ngược lại thành model Pydantic để xem các kiểu message
    output_model = ChatState(**result)
    for i, msg in enumerate(output_model.messages):
        print(f"Message {i}: {type(msg).__name__} - {msg.content}")
    ```

## Thêm cấu hình runtime

Đôi khi bạn muốn có thể cấu hình graph của mình khi gọi nó. Ví dụ, bạn có thể muốn chỉ định LLM hoặc system prompt nào sẽ dùng ở runtime, *mà không làm ô nhiễm state của graph với các tham số này*.

Để thêm cấu hình runtime:

1. Chỉ định một schema cho cấu hình của bạn
2. Thêm cấu hình vào chữ ký hàm của node hoặc conditional edge
3. Truyền cấu hình vào graph.

Xem ví dụ đơn giản bên dưới:

```python
from langgraph.graph import END, StateGraph, START
from langgraph.runtime import Runtime
from typing_extensions import TypedDict

# 1. Chỉ định config schema
class ContextSchema(TypedDict):
    my_runtime_value: str

# 2. Định nghĩa một graph truy cập config trong một node
class State(TypedDict):
    my_state_value: str

def node(state: State, runtime: Runtime[ContextSchema]):  # [!code highlight]
    if runtime.context["my_runtime_value"] == "a":  # [!code highlight]
        return {"my_state_value": 1}
    elif runtime.context["my_runtime_value"] == "b":  # [!code highlight]
        return {"my_state_value": 2}
    else:
        raise ValueError("Unknown values.")

builder = StateGraph(State, context_schema=ContextSchema)  # [!code highlight]
builder.add_node(node)
builder.add_edge(START, "node")
builder.add_edge("node", END)

graph = builder.compile()

# 3. Truyền cấu hình vào ở runtime:
print(graph.invoke({}, context={"my_runtime_value": "a"}))  # [!code highlight]
print(graph.invoke({}, context={"my_runtime_value": "b"}))  # [!code highlight]
```

```
{'my_state_value': 1}
{'my_state_value': 2}
```

??? note "Ví dụ mở rộng: chỉ định LLM ở runtime"
    Bên dưới chúng ta trình bày một ví dụ thực tế trong đó chúng ta cấu hình LLM nào sẽ dùng ở runtime. Chúng ta sẽ dùng cả model OpenAI và Anthropic.

    ```python
    from dataclasses import dataclass

    from langchain.chat_models import init_chat_model
    from langgraph.graph import MessagesState, END, StateGraph, START
    from langgraph.runtime import Runtime
    from typing_extensions import TypedDict

    @dataclass
    class ContextSchema:
        model_provider: str = "anthropic"

    MODELS = {
        "anthropic": init_chat_model("claude-haiku-4-5-20251001"),
        "openai": init_chat_model("gpt-5.4-mini"),
    }

    def call_model(state: MessagesState, runtime: Runtime[ContextSchema]):
        model = MODELS[runtime.context.model_provider]
        response = model.invoke(state["messages"])
        return {"messages": [response]}

    builder = StateGraph(MessagesState, context_schema=ContextSchema)
    builder.add_node("model", call_model)
    builder.add_edge(START, "model")
    builder.add_edge("model", END)

    graph = builder.compile()

    # Cách dùng
    input_message = {"role": "user", "content": "hi"}
    # Không có cấu hình, dùng mặc định (Anthropic)
    response_1 = graph.invoke({"messages": [input_message]}, context=ContextSchema())["messages"][-1]
    # Hoặc, có thể set OpenAI
    response_2 = graph.invoke({"messages": [input_message]}, context={"model_provider": "openai"})["messages"][-1]

    print(response_1.response_metadata["model_name"])
    print(response_2.response_metadata["model_name"])
    ```

    ```
    claude-haiku-4-5-20251001
    gpt-5.4-mini
    ```

??? note "Ví dụ mở rộng: chỉ định model và system message ở runtime"
    Bên dưới chúng ta trình bày một ví dụ thực tế trong đó chúng ta cấu hình hai tham số: LLM và system message sẽ dùng ở runtime.

    ```python
    from dataclasses import dataclass
    from langchain.chat_models import init_chat_model
    from langchain.messages import SystemMessage
    from langgraph.graph import END, MessagesState, StateGraph, START
    from langgraph.runtime import Runtime
    from typing_extensions import TypedDict

    @dataclass
    class ContextSchema:
        model_provider: str = "anthropic"
        system_message: str | None = None

    MODELS = {
        "anthropic": init_chat_model("claude-haiku-4-5-20251001"),
        "openai": init_chat_model("gpt-5.4-mini"),
    }

    def call_model(state: MessagesState, runtime: Runtime[ContextSchema]):
        model = MODELS[runtime.context.model_provider]
        messages = state["messages"]
        if (system_message := runtime.context.system_message):
            messages = [SystemMessage(system_message)] + messages
        response = model.invoke(messages)
        return {"messages": [response]}

    builder = StateGraph(MessagesState, context_schema=ContextSchema)
    builder.add_node("model", call_model)
    builder.add_edge(START, "model")
    builder.add_edge("model", END)

    graph = builder.compile()

    # Cách dùng
    input_message = {"role": "user", "content": "hi"}
    response = graph.invoke({"messages": [input_message]}, context={"model_provider": "openai", "system_message": "Respond in Italian."})
    for message in response["messages"]:
        message.pretty_print()
    ```

    ```
    ================================ Human Message ================================

    hi
    ================================== Ai Message ==================================

    Ciao! Come posso aiutarti oggi?
    ```

## Thêm chính sách retry

Có nhiều trường hợp bạn muốn node của mình có một chính sách retry tùy chỉnh, ví dụ nếu bạn đang gọi một API, truy vấn một database, hoặc gọi một LLM, v.v. LangGraph cho phép bạn thêm chính sách retry vào node.

Để cấu hình một chính sách retry, truyền tham số `retry_policy` vào [`add_node`](https://reference.langchain.com/python/langgraph/graph/state/StateGraph/add_node). Tham số `retry_policy` nhận một object named tuple `RetryPolicy`. Bên dưới chúng ta khởi tạo một object `RetryPolicy` với các tham số mặc định và gắn nó vào một node:

```python
from langgraph.types import RetryPolicy

builder.add_node(
    "node_name",
    node_function,
    retry_policy=RetryPolicy(),
)
```

Mặc định, tham số `retry_on` dùng hàm `default_retry_on`, hàm này retry với mọi exception ngoại trừ các loại sau:

* `ValueError`
* `TypeError`
* `ArithmeticError`
* `ImportError`
* `LookupError`
* `NameError`
* `SyntaxError`
* `RuntimeError`
* `ReferenceError`
* `StopIteration`
* `StopAsyncIteration`
* `OSError`

Ngoài ra, với các exception từ các thư viện HTTP request phổ biến như `requests` và `httpx`, nó chỉ retry với các mã trạng thái 5xx.

??? note "Ví dụ mở rộng: tùy chỉnh chính sách retry"
    Xét một ví dụ trong đó chúng ta đọc từ một database SQL. Bên dưới chúng ta truyền hai chính sách retry khác nhau cho các node:

    ```python
    import sqlite3
    from typing_extensions import TypedDict
    from langchain.chat_models import init_chat_model
    from langgraph.graph import END, MessagesState, StateGraph, START
    from langgraph.types import RetryPolicy
    from langchain.messages import AIMessage

    con = sqlite3.connect(":memory:")
    model = init_chat_model("claude-haiku-4-5-20251001")

    def query_database(state: MessagesState):
        cursor = con.cursor()
        cursor.execute("SELECT * FROM Artist LIMIT 10;")
        query_result = str(cursor.fetchall())
        return {"messages": [AIMessage(content=query_result)]}

    def call_model(state: MessagesState):
        response = model.invoke(state["messages"])
        return {"messages": [response]}

    # Định nghĩa một graph mới
    builder = StateGraph(MessagesState)
    builder.add_node(
        "query_database",
        query_database,
        retry_policy=RetryPolicy(retry_on=sqlite3.OperationalError),
    )
    builder.add_node("model", call_model, retry_policy=RetryPolicy(max_attempts=5))
    builder.add_edge(START, "model")
    builder.add_edge("model", "query_database")
    builder.add_edge("query_database", END)
    graph = builder.compile()
    ```

## Set node timeout (thời gian chờ tối đa cho node)

Dùng tham số `timeout` với [`add_node`](https://reference.langchain.com/python/langgraph/graph/state/StateGraph/add_node) để giới hạn thời gian một lần gọi node async có thể chạy. Cung cấp timeout theo giây hoặc dưới dạng `datetime.timedelta`.

```python
import asyncio
from typing_extensions import TypedDict

from langgraph.errors import NodeTimeoutError
from langgraph.graph import END, START, StateGraph


class State(TypedDict):
    value: str


async def call_model(state: State) -> State:
    await asyncio.sleep(2)
    return {"value": "done"}


builder = StateGraph(State)
builder.add_node("model", call_model, timeout=1.0)
builder.add_edge(START, "model")
builder.add_edge("model", END)
graph = builder.compile()

try:
    await graph.ainvoke({"value": "start"})
except NodeTimeoutError:
    print("Node timed out")
```

Node timeout chỉ được hỗ trợ cho node async. Nếu bạn set `timeout` trên một node sync, LangGraph sẽ raise một lỗi khi graph được compile vì việc thực thi Python sync không thể bị hủy an toàn trong tiến trình.

Khi một node vượt quá timeout, LangGraph raise `NodeTimeoutError`, một lớp con của `TimeoutError` có sẵn trong Python. Nếu node có một `retry_policy` retry `TimeoutError` hoặc `NodeTimeoutError`, lần thử bị timeout sẽ được retry. Timeout áp dụng riêng cho mỗi lần thử, nên bộ đếm thời gian sẽ reset ở mỗi lần retry.

Các lần thử bị timeout không commit các write đã buffer của chúng. Điều này ngăn việc cập nhật state hoặc lên lịch child-task rò rỉ ra ngoài sau ranh giới timeout.

## Cấu hình node timeout

Tham số `timeout=` trên [`add_node`](https://reference.langchain.com/python/langgraph/graph/state/StateGraph/add_node) giới hạn thời gian một lần thử node async có thể chạy. Truyền một số (giây), một `timedelta`, hoặc một [`TimeoutPolicy`](https://reference.langchain.com/python/langgraph/types/TimeoutPolicy) để kiểm soát chi tiết hơn thời gian run và idle timeout. Khi vượt giới hạn, LangGraph raise [`NodeTimeoutError`](https://reference.langchain.com/python/langgraph/errors/NodeTimeoutError) và để chính sách retry quyết định có retry hay không.

!!! note "Ghi chú"
    Node timeout riêng lẻ yêu cầu `langgraph>=1.2`.

```python
from langgraph.types import TimeoutPolicy

builder.add_node(
    "call_model",
    call_model,
    timeout=TimeoutPolicy(run_timeout=120, idle_timeout=30),
)
```

Xem [Khả năng chịu lỗi (fault tolerance)](./fault-tolerance.md#timeouts) để biết vòng đời timeout đầy đủ, các nguồn refresh idle-timeout, và `runtime.heartbeat()`.

## Xử lý lỗi node

Tham số `error_handler=` trên [`add_node`](https://reference.langchain.com/python/langgraph/graph/state/StateGraph/add_node) đăng ký một hàm chạy sau khi một node lỗi và mọi lần retry đã cạn kiệt. Handler nhận state hiện tại và một [`NodeError`](https://reference.langchain.com/python/langgraph/errors/NodeError) có kiểu, chứa ngữ cảnh lỗi, và có thể route sang một nhánh phục hồi thông qua [`Command`](https://reference.langchain.com/python/langgraph/types/Command):

!!! note "Ghi chú"
    Error handler cấp node yêu cầu `langgraph>=1.2`.

```python
from langgraph.errors import NodeError
from langgraph.types import Command, RetryPolicy

def payment_error_handler(state: State, error: NodeError) -> Command:
    return Command(
        update={"status": f"compensated: {error.error}"},
        goto="finalize",
    )

builder.add_node(
    "charge_payment",
    charge_payment,
    retry_policy=RetryPolicy(max_attempts=3, retry_on=ConnectionError),
    error_handler=payment_error_handler,
)
```

Xem [Khả năng chịu lỗi (fault tolerance)](./fault-tolerance.md#error-handling) để biết các mẫu bù trừ (compensation pattern) và cách route bằng `Command`.

## Set mặc định cho node ở cấp graph

!!! note "Ghi chú"
    Yêu cầu `langgraph>=1.2`.

Dùng [`set_node_defaults`](https://reference.langchain.com/python/langgraph/graph/state/StateGraph/set_node_defaults) để set `retry_policy`, `timeout`, `cache_policy`, hoặc `error_handler` một lần cho mọi node trong một graph, thay vì lặp lại chúng ở mỗi lần gọi [`add_node`](https://reference.langchain.com/python/langgraph/graph/state/StateGraph/add_node). Giá trị riêng của từng node luôn được ưu tiên, và các giá trị mặc định được áp dụng tại thời điểm [`StateGraph.compile`](https://reference.langchain.com/python/langgraph/graph/state/StateGraph/compile):

```python
from langgraph.types import RetryPolicy, TimeoutPolicy

graph = (
    StateGraph(State)
    .set_node_defaults(
        retry_policy=RetryPolicy(max_attempts=3),
        timeout=TimeoutPolicy(run_timeout=30),
        error_handler=fallback_handler,
    )
    .add_node("a", node_a)
    .add_node("b", node_b, retry_policy=RetryPolicy(max_attempts=5))  # ghi đè mặc định
    .add_edge(START, "a")
    .compile()
)
```

Mặc định `retry_policy` và `timeout` áp dụng cho mọi node, kể cả node error-handler. Mặc định `cache_policy` và `error_handler` chỉ áp dụng cho node thông thường, handler không bao giờ tự bắt lỗi của chính nó, và việc cache kết quả của một handler là không an toàn. Các giá trị mặc định không được kế thừa bởi subgraph.

Xem [Khả năng chịu lỗi (fault tolerance)](./fault-tolerance.md#graph-defaults) để biết đầy đủ quy tắc ưu tiên và bảng khả năng áp dụng.

### Truy cập thông tin thực thi bên trong một node

Bạn có thể truy cập định danh thực thi (execution identity) và thông tin retry qua `runtime.execution_info`. Điều này cung cấp các định danh thread, run, và checkpoint cũng như trạng thái retry, mà không cần đọc trực tiếp từ `config`.

| Thuộc tính                 | Kiểu            | Mô tả                                                                                      |
| ------------------------- | --------------- | ------------------------------------------------------------------------------------------------ |
| `thread_id`               | `str \| None`   | Thread ID cho lần thực thi hiện tại. `None` nếu không có checkpointer.                              |
| `run_id`                  | `str \| None`   | Run ID cho lần thực thi hiện tại. `None` khi không được cung cấp trong config.                            |
| `checkpoint_id`           | `str`           | Checkpoint ID cho lần thực thi hiện tại.                                                         |
| `checkpoint_ns`           | `str`           | Checkpoint namespace cho lần thực thi hiện tại.                                                  |
| `task_id`                 | `str`           | Task ID cho lần thực thi hiện tại.                                                               |
| `node_attempt`            | `int`           | Số thứ tự lần thử thực thi hiện tại (bắt đầu từ 1). `1` ở lần thử đầu, `2` ở lần retry đầu tiên, v.v. |
| `node_first_attempt_time` | `float \| None` | Unix timestamp (giây) của thời điểm lần thử đầu tiên bắt đầu. Giữ nguyên qua các lần retry.       |

#### Truy cập thread ID và run ID

Dùng `execution_info` để truy cập thread ID, run ID, và các field định danh khác bên trong một node:

```python
from langgraph.graph import StateGraph, START, END
from langgraph.runtime import Runtime
from typing_extensions import TypedDict

class State(TypedDict):
    result: str

def my_node(state: State, runtime: Runtime):
    info = runtime.execution_info
    print(f"Thread: {info.thread_id}, Run: {info.run_id}")  # [!code highlight]
    return {"result": "done"}

builder = StateGraph(State)
builder.add_node("my_node", my_node)
builder.add_edge(START, "my_node")
builder.add_edge("my_node", END)
graph = builder.compile()
```

#### Điều chỉnh hành vi dựa trên trạng thái retry

Khi một node có chính sách retry, dùng `execution_info` để kiểm tra số lần thử hiện tại và chuyển sang phương án dự phòng sau khi lần thử đầu tiên thất bại:

```python
from langgraph.graph import StateGraph, START, END
from langgraph.runtime import Runtime
from langgraph.types import RetryPolicy
from typing_extensions import TypedDict

class State(TypedDict):
    result: str

def my_node(state: State, runtime: Runtime):
    info = runtime.execution_info
    if info.node_attempt > 1:  # [!code highlight]
        # dùng phương án dự phòng khi retry
        return {"result": call_fallback_api()}
    return {"result": call_primary_api()}

builder = StateGraph(State)
builder.add_node("my_node", my_node, retry_policy=RetryPolicy(max_attempts=3))
builder.add_edge(START, "my_node")
builder.add_edge("my_node", END)
graph = builder.compile()
```

`execution_info` luôn có sẵn trên object `Runtime` kể cả khi không có chính sách retry, `node_attempt` mặc định là `1` và `node_first_attempt_time` được set bằng thời điểm node bắt đầu thực thi.

### Truy cập thông tin server bên trong một node

Khi graph của bạn chạy trên LangGraph Server, bạn có thể truy cập metadata riêng của server qua `runtime.server_info`. Điều này cung cấp assistant ID, graph ID, và người dùng đã xác thực mà không cần đọc trực tiếp từ config metadata hoặc các key configurable.

| Thuộc tính      | Kiểu               | Mô tả                                                                     |
| -------------- | ------------------ | -------------------------------------------------------------------------------- |
| `assistant_id` | `str`              | Assistant ID cho lần deploy hiện tại.                                                    |
| `graph_id`     | `str`              | Graph ID cho lần deploy hiện tại.                                                        |
| `user`         | `BaseUser \| None` | Người dùng đã xác thực, nếu [custom auth](https://docs.langchain.com/langsmith/custom-auth) được cấu hình. |

```python
from langgraph.graph import StateGraph, START, END
from langgraph.runtime import Runtime
from typing_extensions import TypedDict

class State(TypedDict):
    result: str

def my_node(state: State, runtime: Runtime):
    server = runtime.server_info
    if server is not None:
        print(f"Assistant: {server.assistant_id}, Graph: {server.graph_id}")  # [!code highlight]
        if server.user is not None:
            print(f"User: {server.user.identity}")
    return {"result": "done"}

builder = StateGraph(State)
builder.add_node("my_node", my_node)
builder.add_edge(START, "my_node")
builder.add_edge("my_node", END)
graph = builder.compile()
```

`server_info` là `None` khi graph không chạy trên LangGraph Server (ví dụ, trong quá trình phát triển hoặc test cục bộ).

!!! note "Ghi chú"
    Yêu cầu `deepagents>=0.5.0` (hoặc `langgraph>=1.1.5`) cho `runtime.execution_info` và `runtime.server_info`.

### Truy cập trạng thái drain bên trong một node

Khi một [graceful shutdown](./fault-tolerance.md#graceful-shutdown) đã được yêu cầu, `runtime.drain_requested` là `True`. Đọc giá trị này bên trong một node để bỏ qua các công việc tốn kém trước ranh giới superstep tiếp theo:

```python
from langgraph.runtime import Runtime

def my_node(state: State, runtime: Runtime) -> State:
    if runtime.drain_requested:  # [!code highlight]
        return {"status": "skipped", "reason": runtime.drain_reason}
    return {"status": do_work()}
```

| Thuộc tính          | Kiểu          | Mô tả                                                                          |
| ----------------- | ------------- | ------------------------------------------------------------------------------------ |
| `drain_requested` | `bool`        | `True` nếu `RunControl.request_drain()` đã được gọi cho lần chạy này.                 |
| `drain_reason`    | `str \| None` | Chuỗi lý do được truyền vào `request_drain()`, hoặc `None` nếu drain chưa được yêu cầu. |

!!! note "Ghi chú"
    Yêu cầu `langgraph>=1.2`. Xem [Graceful shutdown](./fault-tolerance.md#graceful-shutdown) để biết đầy đủ API `RunControl`.

## Thêm caching cho node

Caching cho node hữu ích trong các trường hợp bạn muốn tránh lặp lại các thao tác, chẳng hạn khi làm điều gì đó tốn kém (về thời gian hoặc chi phí). LangGraph cho phép bạn thêm chính sách caching riêng cho từng node trong một graph.

Để cấu hình một chính sách cache, truyền tham số `cache_policy` vào hàm [`add_node`](https://reference.langchain.com/python/langgraph/graph/state/StateGraph/add_node). Trong ví dụ sau, một object [`CachePolicy`](https://reference.langchain.com/python/langgraph/types/CachePolicy) được khởi tạo với thời gian tồn tại (time to live) là 120 giây và bộ tạo `key_func` mặc định. Sau đó nó được gắn vào một node:

```python
from langgraph.types import CachePolicy

builder.add_node(
    "node_name",
    node_function,
    cache_policy=CachePolicy(ttl=120),
)
```

Sau đó, để bật caching cấp node cho một graph, set đối số `cache` khi compile graph. Ví dụ bên dưới dùng `InMemoryCache` để thiết lập một graph với cache trong bộ nhớ, nhưng `SqliteCache` cũng có sẵn.

```python
from langgraph.cache.memory import InMemoryCache

graph = builder.compile(cache=InMemoryCache())
```

## Tạo một chuỗi các bước

!!! info "Điều kiện tiên quyết"
    Hướng dẫn này giả định bạn đã quen thuộc với phần trên về [state](#dinh-nghia-va-cap-nhat-state).

Ở đây chúng ta trình bày cách xây dựng một chuỗi các bước đơn giản. Chúng ta sẽ trình bày:

1. Cách xây dựng một graph tuần tự
2. Cú pháp viết tắt dựng sẵn để xây dựng các graph tương tự.

Để thêm một chuỗi các node, chúng ta dùng các phương thức [`add_node`](https://reference.langchain.com/python/langgraph/graph/state/StateGraph/add_node) và [`add_edge`](https://reference.langchain.com/python/langgraph/pregel/_draw/add_edge) của [graph](./graph-api.md#stategraph):

```python
from langgraph.graph import START, StateGraph

builder = StateGraph(State)

# Thêm các node
builder.add_node(step_1)
builder.add_node(step_2)
builder.add_node(step_3)

# Thêm các edge
builder.add_edge(START, "step_1")
builder.add_edge("step_1", "step_2")
builder.add_edge("step_2", "step_3")
```

Chúng ta cũng có thể dùng cú pháp viết tắt dựng sẵn `.add_sequence`:

```python
builder = StateGraph(State).add_sequence([step_1, step_2, step_3])
builder.add_edge(START, "step_1")
```

??? note "Vì sao nên chia các bước ứng dụng thành một chuỗi với LangGraph?"
    LangGraph giúp dễ dàng thêm một lớp lưu trữ bền vững (persistence) bên dưới cho ứng dụng của bạn.
    Điều này cho phép state được checkpoint giữa các lần thực thi node, nên các node LangGraph của bạn chi phối:

    * Cách các cập nhật state được [checkpoint](./persistence.md)
    * Cách các gián đoạn được resume trong các luồng công việc [human-in-the-loop](./interrupts.md)
    * Cách chúng ta có thể "tua lại" (rewind) và rẽ nhánh (branch-off) các lần thực thi bằng tính năng [time travel](./use-time-travel.md) của LangGraph

    Chúng cũng quyết định cách các bước thực thi được [stream](./streaming.md), và cách ứng dụng của bạn được trực quan hóa và debug bằng [Studio](https://docs.langchain.com/langsmith/studio).

    Hãy trình bày một ví dụ đầy đủ. Chúng ta sẽ tạo một chuỗi ba bước:

    1. Điền một giá trị vào một key của state
    2. Cập nhật cùng giá trị đó
    3. Điền một giá trị khác

    Trước tiên hãy định nghĩa [state](./graph-api.md#state) của chúng ta. Điều này chi phối [schema của graph](./graph-api.md#schema), và cũng có thể chỉ định cách áp dụng cập nhật. Xem [Xử lý cập nhật state với reducer](#xu-ly-cap-nhat-state-voi-reducer) để biết thêm chi tiết.

    Trong trường hợp này, chúng ta chỉ theo dõi hai giá trị:

    ```python
    from typing_extensions import TypedDict

    class State(TypedDict):
        value_1: str
        value_2: int
    ```

    [Node](./graph-api.md#nodes) của chúng ta chỉ đơn giản là các hàm Python đọc state của graph và thực hiện các cập nhật lên nó. Đối số đầu tiên của hàm này luôn là state:

    ```python
    def step_1(state: State):
        return {"value_1": "a"}

    def step_2(state: State):
        current_value_1 = state["value_1"]
        return {"value_1": f"{current_value_1} b"}

    def step_3(state: State):
        return {"value_2": 10}
    ```

    !!! note "Ghi chú"
        Lưu ý rằng khi phát ra các cập nhật cho state, mỗi node chỉ cần chỉ định giá trị của key mà nó muốn cập nhật.

        Mặc định, điều này sẽ **ghi đè** giá trị của key tương ứng. Bạn cũng có thể dùng [reducer](./graph-api.md#reducers) để kiểm soát cách các cập nhật được xử lý, ví dụ, bạn có thể nối thêm các cập nhật liên tiếp vào một key thay vì ghi đè. Xem [Xử lý cập nhật state với reducer](#xu-ly-cap-nhat-state-voi-reducer) để biết thêm chi tiết.

    Cuối cùng, chúng ta định nghĩa graph. Chúng ta dùng [StateGraph](./graph-api.md#stategraph) để định nghĩa một graph hoạt động trên state này.

    Sau đó chúng ta sẽ dùng [`add_node`](./graph-api.md#messagesstate) và [`add_edge`](./graph-api.md#edges) để điền dữ liệu vào graph và định nghĩa luồng điều khiển của nó.

    ```python
    from langgraph.graph import START, StateGraph

    builder = StateGraph(State)

    # Thêm các node
    builder.add_node(step_1)
    builder.add_node(step_2)
    builder.add_node(step_3)

    # Thêm các edge
    builder.add_edge(START, "step_1")
    builder.add_edge("step_1", "step_2")
    builder.add_edge("step_2", "step_3")
    ```

    !!! tip "Chỉ định tên tùy chỉnh"
        Bạn có thể chỉ định tên tùy chỉnh cho node bằng [`add_node`](https://reference.langchain.com/python/langgraph/graph/state/StateGraph/add_node):

        ```python
        builder.add_node("my_node", step_1)
        ```

    Lưu ý rằng:

    * [`add_edge`](https://reference.langchain.com/python/langgraph/pregel/_draw/add_edge) nhận tên của các node, và với các hàm thì mặc định là `node.__name__`.
    * Chúng ta phải chỉ định điểm bắt đầu của graph. Để làm điều đó chúng ta thêm một edge với [node START](./graph-api.md#start-node).
    * Graph dừng lại khi không còn node nào để thực thi.

    Tiếp theo chúng ta [compile](./graph-api.md#compiling-your-graph) graph của mình. Việc này cung cấp một số kiểm tra cơ bản về cấu trúc của graph (ví dụ, xác định các node bị cô lập). Nếu chúng ta đang thêm persistence cho ứng dụng của mình thông qua một [checkpointer](./persistence.md), nó cũng sẽ được truyền vào ở đây.

    ```python
    graph = builder.compile()
    ```

    LangGraph cung cấp các tiện ích có sẵn để trực quan hóa graph của bạn. Hãy kiểm tra chuỗi của chúng ta. Xem [Trực quan hóa graph của bạn](#truc-quan-hoa-graph-cua-ban) để biết chi tiết về trực quan hóa.

    ```python
    from IPython.display import Image, display

    display(Image(graph.get_graph().draw_mermaid_png()))
    ```

    Hãy tiếp tục với một lần gọi đơn giản:

    ```python
    graph.invoke({"value_1": "c"})
    ```

    ```
    {'value_1': 'a b', 'value_2': 10}
    ```

    Lưu ý rằng:

    * Chúng ta khởi động việc gọi bằng cách cung cấp một giá trị cho một state key duy nhất. Chúng ta phải luôn cung cấp giá trị cho ít nhất một key.
    * Giá trị chúng ta truyền vào đã bị ghi đè bởi node đầu tiên.
    * Node thứ hai đã cập nhật giá trị.
    * Node thứ ba điền vào một giá trị khác.

    !!! tip "Cú pháp viết tắt dựng sẵn"
        `langgraph>=0.2.46` có sẵn cú pháp viết tắt `add_sequence` để thêm chuỗi node. Bạn có thể compile cùng một graph như sau:

        ```python
        builder = StateGraph(State).add_sequence([step_1, step_2, step_3])  # [!code highlight]
        builder.add_edge(START, "step_1")

        graph = builder.compile()

        graph.invoke({"value_1": "c"})
        ```

## Tạo nhánh rẽ

Thực thi song song các node là yếu tố thiết yếu để tăng tốc hoạt động tổng thể của graph. LangGraph hỗ trợ sẵn việc thực thi song song các node, giúp cải thiện đáng kể hiệu năng của các luồng công việc dựa trên graph. Việc song song hóa này đạt được thông qua các cơ chế fan-out và fan-in, sử dụng cả edge tiêu chuẩn và [conditional_edges](https://reference.langchain.com/python/langgraph/graph/state/StateGraph/add_conditional_edges). Bên dưới là một số ví dụ minh họa cách tạo các luồng dữ liệu rẽ nhánh phù hợp với bạn.

### Chạy các node của graph song song

Trong ví dụ này, chúng ta fan-out từ `Node A` sang `B và C` rồi fan-in về `D`. Với state của chúng ta, [chúng ta chỉ định reducer là phép cộng](./graph-api.md#reducers). Điều này sẽ kết hợp hoặc tích lũy các giá trị cho key cụ thể trong State, thay vì chỉ ghi đè giá trị hiện có. Với danh sách, điều này có nghĩa là nối danh sách mới vào danh sách hiện có. Xem phần trên về [state reducer](#xu-ly-cap-nhat-state-voi-reducer) để biết thêm chi tiết về cập nhật state với reducer.

```python
import operator
from typing import Annotated, Any
from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END

class State(TypedDict):
    # Hàm reducer operator.add làm cho field này chỉ có thể nối thêm (append-only)
    aggregate: Annotated[list, operator.add]

def a(state: State):
    print(f'Adding "A" to {state["aggregate"]}')
    return {"aggregate": ["A"]}

def b(state: State):
    print(f'Adding "B" to {state["aggregate"]}')
    return {"aggregate": ["B"]}

def c(state: State):
    print(f'Adding "C" to {state["aggregate"]}')
    return {"aggregate": ["C"]}

def d(state: State):
    print(f'Adding "D" to {state["aggregate"]}')
    return {"aggregate": ["D"]}

builder = StateGraph(State)
builder.add_node(a)
builder.add_node(b)
builder.add_node(c)
builder.add_node(d)
builder.add_edge(START, "a")
builder.add_edge("a", "b")
builder.add_edge("a", "c")
builder.add_edge("b", "d")
builder.add_edge("c", "d")
builder.add_edge("d", END)
graph = builder.compile()
```

```python
from IPython.display import Image, display

display(Image(graph.get_graph().draw_mermaid_png()))
```

Với reducer, bạn có thể thấy rằng các giá trị được thêm ở mỗi node được tích lũy lại.

```python
graph.invoke({"aggregate": []}, {"configurable": {"thread_id": "foo"}})
```

```
Adding "A" to []
Adding "B" to ['A']
Adding "C" to ['A']
Adding "D" to ['A', 'B', 'C']
```

!!! note "Ghi chú"
    Trong ví dụ trên, node `"b"` và `"c"` được thực thi đồng thời trong cùng một [superstep](./graph-api.md#graphs). Vì chúng nằm trong cùng một bước, node `"d"` chỉ thực thi sau khi cả `"b"` và `"c"` đã hoàn thành.

    Điều quan trọng là các cập nhật từ một superstep song song có thể không được sắp xếp thứ tự nhất quán. Nếu bạn cần một thứ tự nhất quán, xác định trước cho các cập nhật từ một superstep song song, bạn nên ghi đầu ra vào một field riêng trong state cùng với một giá trị để sắp xếp thứ tự chúng.

??? note "Xử lý ngoại lệ?"
    LangGraph thực thi các node bên trong [superstep](./graph-api.md#graphs), nghĩa là dù các nhánh song song được thực thi song song, toàn bộ superstep là **có tính giao dịch (transactional)**. Nếu bất kỳ nhánh nào trong số này raise một exception, **không** cập nhật nào được áp dụng vào state (toàn bộ superstep lỗi).

    Điều quan trọng là, khi dùng một [checkpointer](./persistence.md), kết quả từ các node thành công trong một superstep được lưu lại, và không lặp lại khi resume.

    Nếu bạn có các trường hợp dễ lỗi (chẳng hạn muốn xử lý các API call chập chờn), LangGraph cung cấp hai cách để giải quyết điều này:

    1. Bạn có thể viết code Python thông thường bên trong node của mình để bắt và xử lý exception.
    2. Bạn có thể set một **[retry_policy](https://langchain-ai.github.io/langgraph/reference/types/#langgraph.types.RetryPolicy)** để chỉ định graph retry các node raise một số loại exception nhất định. Chỉ các nhánh lỗi mới được retry, nên bạn không cần lo lắng về việc thực hiện lại công việc dư thừa.

    Kết hợp lại, những cơ chế này cho phép bạn vừa thực thi song song vừa kiểm soát hoàn toàn việc xử lý exception.

!!! tip "Set concurrency tối đa"
    Bạn có thể kiểm soát số lượng task đồng thời tối đa bằng cách set `max_concurrency` trong [cấu hình](https://reference.langchain.com/python/langchain-core/runnables/config/RunnableConfig) khi gọi graph.

    ```python
    graph.invoke({"value_1": "c"}, {"configurable": {"max_concurrency": 10}})
    ```

### Trì hoãn thực thi node (defer)

Trì hoãn thực thi node hữu ích khi bạn muốn hoãn việc thực thi một node cho đến khi mọi task đang chờ khác hoàn thành. Điều này đặc biệt liên quan khi các nhánh có độ dài khác nhau, thường gặp trong các luồng công việc như map-reduce.

Ví dụ trên cho thấy cách fan-out và fan-in khi mỗi đường chỉ có một bước. Nhưng nếu một nhánh có nhiều hơn một bước thì sao? Hãy thêm một node `"b_2"` vào nhánh `"b"`:

```python
import operator
from typing import Annotated, Any
from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END

class State(TypedDict):
    # Hàm reducer operator.add làm cho field này chỉ có thể nối thêm (append-only)
    aggregate: Annotated[list, operator.add]

def a(state: State):
    print(f'Adding "A" to {state["aggregate"]}')
    return {"aggregate": ["A"]}

def b(state: State):
    print(f'Adding "B" to {state["aggregate"]}')
    return {"aggregate": ["B"]}

def b_2(state: State):
    print(f'Adding "B_2" to {state["aggregate"]}')
    return {"aggregate": ["B_2"]}

def c(state: State):
    print(f'Adding "C" to {state["aggregate"]}')
    return {"aggregate": ["C"]}

def d(state: State):
    print(f'Adding "D" to {state["aggregate"]}')
    return {"aggregate": ["D"]}

builder = StateGraph(State)
builder.add_node(a)
builder.add_node(b)
builder.add_node(b_2)
builder.add_node(c)
builder.add_node(d, defer=True)  # [!code highlight]
builder.add_edge(START, "a")
builder.add_edge("a", "b")
builder.add_edge("a", "c")
builder.add_edge("b", "b_2")
builder.add_edge("b_2", "d")
builder.add_edge("c", "d")
builder.add_edge("d", END)
graph = builder.compile()
```

```python
from IPython.display import Image, display

display(Image(graph.get_graph().draw_mermaid_png()))
```

```python
graph.invoke({"aggregate": []})
```

```
Adding "A" to []
Adding "B" to ['A']
Adding "C" to ['A']
Adding "B_2" to ['A', 'B', 'C']
Adding "D" to ['A', 'B', 'C', 'B_2']
```

Trong ví dụ trên, node `"b"` và `"c"` được thực thi đồng thời trong cùng một superstep. Chúng ta set `defer=True` trên node `d` để nó không thực thi cho đến khi mọi task đang chờ đã hoàn thành. Trong trường hợp này, nghĩa là `"d"` chờ thực thi cho đến khi toàn bộ nhánh `"b"` hoàn thành.

### Rẽ nhánh có điều kiện

Nếu việc fan-out của bạn cần thay đổi ở runtime dựa trên state, bạn có thể dùng [`add_conditional_edges`](https://reference.langchain.com/python/langgraph/graph/state/StateGraph/add_conditional_edges) để chọn một hoặc nhiều đường đi dùng state của graph. Xem ví dụ bên dưới, trong đó node `a` tạo ra một cập nhật state quyết định node tiếp theo.

```python
import operator
from typing import Annotated, Literal, Sequence
from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END

class State(TypedDict):
    aggregate: Annotated[list, operator.add]
    # Thêm một key vào state. Chúng ta sẽ set key này để quyết định
    # cách chúng ta rẽ nhánh.
    which: str

def a(state: State):
    print(f'Adding "A" to {state["aggregate"]}')
    return {"aggregate": ["A"], "which": "c"}  # [!code highlight]

def b(state: State):
    print(f'Adding "B" to {state["aggregate"]}')
    return {"aggregate": ["B"]}

def c(state: State):
    print(f'Adding "C" to {state["aggregate"]}')
    return {"aggregate": ["C"]}

builder = StateGraph(State)
builder.add_node(a)
builder.add_node(b)
builder.add_node(c)
builder.add_edge(START, "a")
builder.add_edge("b", END)
builder.add_edge("c", END)

def conditional_edge(state: State) -> Literal["b", "c"]:
    # Điền logic tùy ý ở đây, dùng state
    # để xác định node tiếp theo
    return state["which"]

builder.add_conditional_edges("a", conditional_edge)  # [!code highlight]

graph = builder.compile()
```

```python
from IPython.display import Image, display

display(Image(graph.get_graph().draw_mermaid_png()))
```

```python
result = graph.invoke({"aggregate": []})
print(result)
```

```
Adding "A" to []
Adding "C" to ['A']
{'aggregate': ['A', 'C'], 'which': 'c'}
```

!!! tip "Mẹo"
    Conditional edge của bạn có thể route đến nhiều node đích. Ví dụ:

    ```python
    def route_bc_or_cd(state: State) -> Sequence[str]:
        if state["which"] == "cd":
            return ["c", "d"]
        return ["b", "c"]
    ```

## Map-Reduce và Send API

LangGraph hỗ trợ map-reduce và các mẫu rẽ nhánh nâng cao khác bằng Send API. Đây là ví dụ về cách dùng nó:

```python
from langgraph.graph import StateGraph, START, END
from langgraph.types import Send
from typing_extensions import TypedDict, Annotated
import operator

class OverallState(TypedDict):
    topic: str
    subjects: list[str]
    jokes: Annotated[list[str], operator.add]
    best_selected_joke: str

def generate_topics(state: OverallState):
    return {"subjects": ["lions", "elephants", "penguins"]}

def generate_joke(state: OverallState):
    joke_map = {
        "lions": "Why don't lions like fast food? Because they can't catch it!",
        "elephants": "Why don't elephants use computers? They're afraid of the mouse!",
        "penguins": "Why don't penguins like talking to strangers at parties? Because they find it hard to break the ice."
    }
    return {"jokes": [joke_map[state["subject"]]]}

def continue_to_jokes(state: OverallState):
    return [Send("generate_joke", {"subject": s}) for s in state["subjects"]]

def best_joke(state: OverallState):
    return {"best_selected_joke": "penguins"}

builder = StateGraph(OverallState)
builder.add_node("generate_topics", generate_topics)
builder.add_node("generate_joke", generate_joke)
builder.add_node("best_joke", best_joke)
builder.add_edge(START, "generate_topics")
builder.add_conditional_edges("generate_topics", continue_to_jokes, ["generate_joke"])
builder.add_edge("generate_joke", "best_joke")
builder.add_edge("best_joke", END)
graph = builder.compile()
```

```python
from IPython.display import Image, display

display(Image(graph.get_graph().draw_mermaid_png()))
```

```python
# Gọi graph: ở đây chúng ta gọi nó để tạo một danh sách các câu đùa
stream = graph.stream_events({"topic": "animals"}, version="v3")
for message in stream.messages:
    for token in message.text:
        print(token, end="", flush=True)
```

```
{'generate_topics': {'subjects': ['lions', 'elephants', 'penguins']}}
{'generate_joke': {'jokes': ["Why don't lions like fast food? Because they can't catch it!"]}}
{'generate_joke': {'jokes': ["Why don't elephants use computers? They're afraid of the mouse!"]}}
{'generate_joke': {'jokes': ['Why don't penguins like talking to strangers at parties? Because they find it hard to break the ice.']}}
{'best_joke': {'best_selected_joke': 'penguins'}}
```

## Tạo và kiểm soát vòng lặp

Khi tạo một graph có vòng lặp, chúng ta cần một cơ chế để kết thúc việc thực thi. Cách phổ biến nhất để làm điều này là thêm một [conditional edge](./graph-api.md#conditional-edges) route đến node [END](./graph-api.md#end-node) khi đạt đến một điều kiện kết thúc nào đó.

Bạn cũng có thể set giới hạn đệ quy (recursion limit) của graph khi gọi hoặc stream graph. Giới hạn đệ quy set số lượng [super-step](./graph-api.md#graphs) mà graph được phép thực thi trước khi nó raise một lỗi. Đọc thêm về [khái niệm giới hạn đệ quy](./graph-api.md#recursion-limit).

Hãy xem xét một graph đơn giản có vòng lặp để hiểu rõ hơn cách các cơ chế này hoạt động.

!!! tip "Mẹo"
    Để trả về giá trị cuối cùng của state thay vì nhận lỗi giới hạn đệ quy, xem [phần tiếp theo](#ap-dat-gioi-han-de-quy).

Khi tạo một vòng lặp, bạn có thể thêm một conditional edge chỉ định điều kiện kết thúc:

```python
builder = StateGraph(State)
builder.add_node(a)
builder.add_node(b)

def route(state: State) -> Literal["b", END]:
    if termination_condition(state):
        return END
    else:
        return "b"

builder.add_edge(START, "a")
builder.add_conditional_edges("a", route)
builder.add_edge("b", "a")
graph = builder.compile()
```

Để kiểm soát giới hạn đệ quy, chỉ định `"recursion_limit"` trong config. Điều này sẽ raise một `GraphRecursionError`, mà bạn có thể bắt và xử lý:

```python
from langgraph.errors import GraphRecursionError

try:
    graph.invoke(inputs, {"recursion_limit": 3})
except GraphRecursionError:
    print("Recursion Error")
```

Hãy định nghĩa một graph với một vòng lặp đơn giản. Lưu ý rằng chúng ta dùng một conditional edge để triển khai điều kiện kết thúc.

```python
import operator
from typing import Annotated, Literal
from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END

class State(TypedDict):
    # Hàm reducer operator.add làm cho field này chỉ có thể nối thêm (append-only)
    aggregate: Annotated[list, operator.add]

def a(state: State):
    print(f'Node A sees {state["aggregate"]}')
    return {"aggregate": ["A"]}

def b(state: State):
    print(f'Node B sees {state["aggregate"]}')
    return {"aggregate": ["B"]}

# Định nghĩa các node
builder = StateGraph(State)
builder.add_node(a)
builder.add_node(b)

# Định nghĩa các edge
def route(state: State) -> Literal["b", END]:
    if len(state["aggregate"]) < 7:
        return "b"
    else:
        return END

builder.add_edge(START, "a")
builder.add_conditional_edges("a", route)
builder.add_edge("b", "a")
graph = builder.compile()
```

```python
from IPython.display import Image, display

display(Image(graph.get_graph().draw_mermaid_png()))
```

Kiến trúc này tương tự một [agent ReAct](./workflows-agents.md) trong đó node `"a"` là một model gọi tool, và node `"b"` đại diện cho các tool.

Trong conditional edge `route` của chúng ta, chúng ta chỉ định rằng nên kết thúc sau khi danh sách `"aggregate"` trong state vượt quá một độ dài ngưỡng.

Khi gọi graph, chúng ta thấy nó luân phiên giữa node `"a"` và `"b"` trước khi kết thúc khi đạt điều kiện dừng.

```python
graph.invoke({"aggregate": []})
```

```
Node A sees []
Node B sees ['A']
Node A sees ['A', 'B']
Node B sees ['A', 'B', 'A']
Node A sees ['A', 'B', 'A', 'B']
Node B sees ['A', 'B', 'A', 'B', 'A']
Node A sees ['A', 'B', 'A', 'B', 'A', 'B']
```

### Áp đặt giới hạn đệ quy

Trong một số ứng dụng, chúng ta có thể không đảm bảo sẽ đạt được một điều kiện kết thúc nhất định. Trong các trường hợp này, chúng ta có thể set [giới hạn đệ quy](./graph-api.md#recursion-limit) của graph. Điều này sẽ raise một `GraphRecursionError` sau một số [superstep](./graph-api.md#graphs) nhất định. Sau đó chúng ta có thể bắt và xử lý exception này:

```python
from langgraph.errors import GraphRecursionError

try:
    graph.invoke({"aggregate": []}, {"recursion_limit": 4})
except GraphRecursionError:
    print("Recursion Error")
```

```
Node A sees []
Node B sees ['A']
Node C sees ['A', 'B']
Node D sees ['A', 'B']
Node A sees ['A', 'B', 'C', 'D']
Recursion Error
```

??? note "Ví dụ mở rộng: trả về state khi chạm giới hạn đệ quy"
    Thay vì raise `GraphRecursionError`, chúng ta có thể đưa vào state một key mới theo dõi số bước còn lại trước khi đạt giới hạn đệ quy. Sau đó chúng ta có thể dùng key này để quyết định có nên kết thúc lần chạy hay không.

    LangGraph triển khai một annotation đặc biệt là `RemainingSteps`. Bên dưới, nó tạo ra một channel `ManagedValue`, một state channel chỉ tồn tại trong suốt lần chạy graph của chúng ta rồi biến mất.

    ```python
    import operator
    from typing import Annotated, Literal
    from typing_extensions import TypedDict
    from langgraph.graph import StateGraph, START, END
    from langgraph.managed.is_last_step import RemainingSteps

    class State(TypedDict):
        aggregate: Annotated[list, operator.add]
        remaining_steps: RemainingSteps

    def a(state: State):
        print(f'Node A sees {state["aggregate"]}')
        return {"aggregate": ["A"]}

    def b(state: State):
        print(f'Node B sees {state["aggregate"]}')
        return {"aggregate": ["B"]}

    # Định nghĩa các node
    builder = StateGraph(State)
    builder.add_node(a)
    builder.add_node(b)

    # Định nghĩa các edge
    def route(state: State) -> Literal["b", END]:
        if state["remaining_steps"] <= 2:
            return END
        else:
            return "b"

    builder.add_edge(START, "a")
    builder.add_conditional_edges("a", route)
    builder.add_edge("b", "a")
    graph = builder.compile()

    # Thử nghiệm
    result = graph.invoke({"aggregate": []}, {"recursion_limit": 4})
    print(result)
    ```

    ```
    Node A sees []
    Node B sees ['A']
    Node A sees ['A', 'B']
    {'aggregate': ['A', 'B', 'A']}
    ```

??? note "Ví dụ mở rộng: vòng lặp có nhánh rẽ"
    Để hiểu rõ hơn cách giới hạn đệ quy hoạt động, hãy xem xét một ví dụ phức tạp hơn. Bên dưới chúng ta triển khai một vòng lặp, nhưng một bước fan-out ra hai node:

    ```python
    import operator
    from typing import Annotated, Literal
    from typing_extensions import TypedDict
    from langgraph.graph import StateGraph, START, END

    class State(TypedDict):
        aggregate: Annotated[list, operator.add]

    def a(state: State):
        print(f'Node A sees {state["aggregate"]}')
        return {"aggregate": ["A"]}

    def b(state: State):
        print(f'Node B sees {state["aggregate"]}')
        return {"aggregate": ["B"]}

    def c(state: State):
        print(f'Node C sees {state["aggregate"]}')
        return {"aggregate": ["C"]}

    def d(state: State):
        print(f'Node D sees {state["aggregate"]}')
        return {"aggregate": ["D"]}

    # Định nghĩa các node
    builder = StateGraph(State)
    builder.add_node(a)
    builder.add_node(b)
    builder.add_node(c)
    builder.add_node(d)

    # Định nghĩa các edge
    def route(state: State) -> Literal["b", END]:
        if len(state["aggregate"]) < 7:
            return "b"
        else:
            return END

    builder.add_edge(START, "a")
    builder.add_conditional_edges("a", route)
    builder.add_edge("b", "c")
    builder.add_edge("b", "d")
    builder.add_edge(["c", "d"], "a")
    graph = builder.compile()
    ```

    ```python
    from IPython.display import Image, display

    display(Image(graph.get_graph().draw_mermaid_png()))
    ```

    Graph này trông phức tạp, nhưng có thể được hình dung như một vòng lặp gồm các [superstep](./graph-api.md#graphs):

    1. Node A
    2. Node B
    3. Node C và D
    4. Node A
    5. ...

    Chúng ta có một vòng lặp gồm bốn superstep, trong đó node C và D được thực thi đồng thời.

    Gọi graph như trước, chúng ta thấy hoàn thành hai "vòng" đầy đủ trước khi chạm điều kiện kết thúc:

    ```python
    result = graph.invoke({"aggregate": []})
    ```

    ```
    Node A sees []
    Node B sees ['A']
    Node D sees ['A', 'B']
    Node C sees ['A', 'B']
    Node A sees ['A', 'B', 'C', 'D']
    Node B sees ['A', 'B', 'C', 'D', 'A']
    Node D sees ['A', 'B', 'C', 'D', 'A', 'B']
    Node C sees ['A', 'B', 'C', 'D', 'A', 'B']
    Node A sees ['A', 'B', 'C', 'D', 'A', 'B', 'C', 'D']
    ```

    Tuy nhiên, nếu chúng ta set giới hạn đệ quy là bốn, chúng ta chỉ hoàn thành một vòng vì mỗi vòng là bốn superstep:

    ```python
    from langgraph.errors import GraphRecursionError

    try:
        result = graph.invoke({"aggregate": []}, {"recursion_limit": 4})
    except GraphRecursionError:
        print("Recursion Error")
    ```

    ```
    Node A sees []
    Node B sees ['A']
    Node C sees ['A', 'B']
    Node D sees ['A', 'B']
    Node A sees ['A', 'B', 'C', 'D']
    Recursion Error
    ```

## Async

Dùng mô hình lập trình async có thể mang lại cải thiện hiệu năng đáng kể khi chạy code [IO-bound](https://en.wikipedia.org/wiki/I/O_bound) đồng thời (ví dụ, gửi các API request đồng thời đến một nhà cung cấp chat model).

Để chuyển một triển khai `sync` của graph thành triển khai `async`, bạn cần:

1. Cập nhật `node` dùng `async def` thay vì `def`.
2. Cập nhật code bên trong để dùng `await` phù hợp.
3. Gọi graph bằng `.ainvoke` hoặc `.astream` tùy ý.

Vì nhiều object LangChain triển khai [Runnable Protocol](https://python.langchain.com/docs/expression_language/interface/) có các biến thể `async` cho mọi phương thức `sync`, việc nâng cấp một graph `sync` thành graph `async` thường khá nhanh chóng.

Xem ví dụ bên dưới. Để minh họa các lần gọi async của LLM bên dưới, chúng ta sẽ đưa vào một chat model:

=== "OpenAI"
    👉 Đọc [tài liệu tích hợp OpenAI chat model](https://docs.langchain.com/oss/python/integrations/chat/openai/)

    Cài đặt package:

    === "pip"
        ```bash
        pip install -U "langchain[openai]"
        ```

    === "uv"
        ```bash
        uv add "langchain[openai]"
        ```

    Khởi tạo model:

    === "init_chat_model"
        ```python
        import os
        from langchain.chat_models import init_chat_model

        os.environ["OPENAI_API_KEY"] = "sk-..."

        model = init_chat_model("gpt-5.5")
        ```

    === "Model Class"
        ```python
        import os
        from langchain_openai import ChatOpenAI

        os.environ["OPENAI_API_KEY"] = "sk-..."

        model = ChatOpenAI(model="gpt-5.5")
        ```

=== "Anthropic"
    👉 Đọc [tài liệu tích hợp Anthropic chat model](https://docs.langchain.com/oss/python/integrations/chat/anthropic/)

    Cài đặt package:

    === "pip"
        ```bash
        pip install -U "langchain[anthropic]"
        ```

    === "uv"
        ```bash
        uv add "langchain[anthropic]"
        ```

    Khởi tạo model:

    === "init_chat_model"
        ```python
        import os
        from langchain.chat_models import init_chat_model

        os.environ["ANTHROPIC_API_KEY"] = "sk-..."

        model = init_chat_model("claude-sonnet-4-6")
        ```

    === "Model Class"
        ```python
        import os
        from langchain_anthropic import ChatAnthropic

        os.environ["ANTHROPIC_API_KEY"] = "sk-..."

        model = ChatAnthropic(model="claude-sonnet-4-6")
        ```

=== "Azure"
    👉 Đọc [tài liệu tích hợp Azure chat model](https://docs.langchain.com/oss/python/integrations/chat/azure_chat_openai/)

    Cài đặt package:

    === "pip"
        ```bash
        pip install -U "langchain[openai]"
        ```

    === "uv"
        ```bash
        uv add "langchain[openai]"
        ```

    Khởi tạo model:

    === "init_chat_model"
        ```python
        import os
        from langchain.chat_models import init_chat_model

        os.environ["AZURE_OPENAI_API_KEY"] = "..."
        os.environ["AZURE_OPENAI_ENDPOINT"] = "..."
        os.environ["OPENAI_API_VERSION"] = "2025-03-01-preview"

        model = init_chat_model(
            "azure_openai:gpt-5.5",
            azure_deployment=os.environ["AZURE_OPENAI_DEPLOYMENT_NAME"],
        )
        ```

    === "Model Class"
        ```python
        import os
        from langchain_openai import AzureChatOpenAI

        os.environ["AZURE_OPENAI_API_KEY"] = "..."
        os.environ["AZURE_OPENAI_ENDPOINT"] = "..."
        os.environ["OPENAI_API_VERSION"] = "2025-03-01-preview"

        model = AzureChatOpenAI(
            model="gpt-5.5",
            azure_deployment=os.environ["AZURE_OPENAI_DEPLOYMENT_NAME"]
        )
        ```

=== "Google Gemini"
    👉 Đọc [tài liệu tích hợp Google GenAI chat model](https://docs.langchain.com/oss/python/integrations/chat/google_generative_ai/)

    Cài đặt package:

    === "pip"
        ```bash
        pip install -U "langchain[google-genai]"
        ```

    === "uv"
        ```bash
        uv add "langchain[google-genai]"
        ```

    Khởi tạo model:

    === "init_chat_model"
        ```python
        import os
        from langchain.chat_models import init_chat_model

        os.environ["GOOGLE_API_KEY"] = "..."

        model = init_chat_model("google_genai:gemini-2.5-flash-lite")
        ```

    === "Model Class"
        ```python
        import os
        from langchain_google_genai import ChatGoogleGenerativeAI

        os.environ["GOOGLE_API_KEY"] = "..."

        model = ChatGoogleGenerativeAI(model="gemini-2.5-flash-lite")
        ```

=== "AWS Bedrock"
    👉 Đọc [tài liệu tích hợp AWS Bedrock chat model](https://docs.langchain.com/oss/python/integrations/chat/bedrock/)

    Cài đặt package:

    === "pip"
        ```bash
        pip install -U "langchain[aws]"
        ```

    === "uv"
        ```bash
        uv add "langchain[aws]"
        ```

    Khởi tạo model:

    === "init_chat_model"
        ```python
        from langchain.chat_models import init_chat_model

        # Làm theo các bước ở đây để cấu hình thông tin xác thực của bạn:
        # https://docs.aws.amazon.com/bedrock/latest/userguide/getting-started.html

        model = init_chat_model(
            "us.anthropic.claude-sonnet-4-6",
            model_provider="bedrock_converse",
        )
        ```

    === "Model Class"
        ```python
        from langchain_aws import ChatBedrock

        model = ChatBedrock(model="us.anthropic.claude-sonnet-4-6")
        ```

=== "HuggingFace"
    👉 Đọc [tài liệu tích hợp HuggingFace chat model](https://docs.langchain.com/oss/python/integrations/chat/huggingface/)

    Cài đặt package:

    === "pip"
        ```bash
        pip install -U "langchain[huggingface]"
        ```

    === "uv"
        ```bash
        uv add "langchain[huggingface]"
        ```

    Khởi tạo model:

    === "init_chat_model"
        ```python
        import os
        from langchain.chat_models import init_chat_model

        os.environ["HUGGINGFACEHUB_API_TOKEN"] = "hf_..."

        model = init_chat_model(
            "microsoft/Phi-3-mini-4k-instruct",
            model_provider="huggingface",
            temperature=0.7,
            max_tokens=1024,
        )
        ```

    === "Model Class"
        ```python
        import os
        from langchain_huggingface import ChatHuggingFace, HuggingFaceEndpoint

        os.environ["HUGGINGFACEHUB_API_TOKEN"] = "hf_..."

        llm = HuggingFaceEndpoint(
            repo_id="microsoft/Phi-3-mini-4k-instruct",
            temperature=0.7,
            max_length=1024,
        )
        model = ChatHuggingFace(llm=llm)
        ```

=== "OpenRouter"
    👉 Đọc [tài liệu tích hợp OpenRouter chat model](https://docs.langchain.com/oss/python/integrations/chat/openrouter/)

    Cài đặt package:

    === "pip"
        ```bash
        pip install -U "langchain-openrouter"
        ```

    === "uv"
        ```bash
        uv add "langchain-openrouter"
        ```

    Khởi tạo model:

    === "init_chat_model"
        ```python
        import os
        from langchain.chat_models import init_chat_model

        os.environ["OPENROUTER_API_KEY"] = "sk-..."

        model = init_chat_model(
            "auto",
            model_provider="openrouter",
        )
        ```

    === "Model Class"
        ```python
        import os
        from langchain_openrouter import ChatOpenRouter

        os.environ["OPENROUTER_API_KEY"] = "sk-..."

        model = ChatOpenRouter(model="auto")
        ```

```python
from langchain.chat_models import init_chat_model
from langgraph.graph import MessagesState, StateGraph

async def node(state: MessagesState):  # [!code highlight]
    new_message = await llm.ainvoke(state["messages"])  # [!code highlight]
    return {"messages": [new_message]}

builder = StateGraph(MessagesState).add_node(node).set_entry_point("node")
graph = builder.compile()

input_message = {"role": "user", "content": "Hello"}
result = await graph.ainvoke({"messages": [input_message]})  # [!code highlight]
```

!!! tip "Streaming async"
    Xem [hướng dẫn streaming](./streaming.md) để có ví dụ về streaming với async.

## Kết hợp control flow và cập nhật state với `Command`

Đôi khi sẽ hữu ích khi kết hợp control flow (cạnh/edge) với cập nhật state (node). Ví dụ, bạn có thể vừa muốn thực hiện cập nhật state VÀ quyết định node tiếp theo cần đi tới trong CÙNG một node. LangGraph cung cấp cách làm điều này bằng cách trả về một đối tượng [Command](https://reference.langchain.com/python/langgraph/types/Command) từ hàm node:

```python
def my_node(state: State) -> Command[Literal["my_other_node"]]:
    return Command(
        # state update
        update={"foo": "bar"},
        # control flow
        goto="my_other_node"
    )
```

Dưới đây là một ví dụ đầy đủ. Hãy tạo một graph đơn giản với 3 node: A, B và C. Đầu tiên ta sẽ thực thi node A, sau đó quyết định đi tới Node B hay Node C dựa trên output của node A.

```python
import random
from typing_extensions import TypedDict, Literal
from langgraph.graph import StateGraph, START
from langgraph.types import Command

# Định nghĩa state của graph
class State(TypedDict):
    foo: str

# Định nghĩa các node

def node_a(state: State) -> Command[Literal["node_b", "node_c"]]:
    print("Called A")
    value = random.choice(["b", "c"])
    # đây là thay thế cho một hàm conditional edge
    if value == "b":
        goto = "node_b"
    else:
        goto = "node_c"

    # lưu ý Command cho phép bạn VỪA cập nhật state của graph VỪA định tuyến tới node tiếp theo
    return Command(
        # đây là cập nhật state
        update={"foo": value},
        # đây là thay thế cho một cạnh (edge)
        goto=goto,
    )

def node_b(state: State):
    print("Called B")
    return {"foo": state["foo"] + "b"}

def node_c(state: State):
    print("Called C")
    return {"foo": state["foo"] + "c"}
```

Giờ ta có thể tạo [`StateGraph`](https://reference.langchain.com/python/langgraph/graph/state/StateGraph) với các node ở trên. Chú ý rằng graph này không có [conditional edge](./graph-api.md#conditional-edges) nào để định tuyến! Đó là vì control flow đã được định nghĩa bằng [`Command`](https://reference.langchain.com/python/langgraph/types/Command) bên trong `node_a`.

```python
builder = StateGraph(State)
builder.add_edge(START, "node_a")
builder.add_node(node_a)
builder.add_node(node_b)
builder.add_node(node_c)
# LƯU Ý: không có cạnh nào giữa node A, B và C!

graph = builder.compile()
```

!!! warning ""
    Bạn có thể để ý rằng ta đã dùng [`Command`](https://reference.langchain.com/python/langgraph/types/Command) làm annotation kiểu trả về, ví dụ `Command[Literal["node_b", "node_c"]]`. Điều này cần thiết cho việc render graph và cho LangGraph biết rằng `node_a` có thể điều hướng tới `node_b` và `node_c`.

```python
from IPython.display import display, Image

display(Image(graph.get_graph().draw_mermaid_png()))
```

<img src="https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/graph_api_image_11.png?fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=f11e5cddedbf2760d40533f294c44aea" alt="Command-based graph navigation" width="232" height="333" data-path="oss/images/graph_api_image_11.png" />

Nếu ta chạy graph này nhiều lần, ta sẽ thấy nó đi theo các đường khác nhau (A -> B hoặc A -> C) dựa trên lựa chọn ngẫu nhiên trong node A.

```python
graph.invoke({"foo": ""})
```

```
Called A
Called C
```

### Điều hướng tới một node trong graph cha

Nếu bạn đang dùng [subgraph](./use-subgraphs.md), bạn có thể muốn điều hướng từ một node bên trong subgraph tới một subgraph khác (tức một node khác trong graph cha). Để làm điều này, bạn có thể chỉ định `graph=Command.PARENT` trong `Command`:

```python
def my_node(state: State) -> Command[Literal["other_subgraph"]]:
    return Command(
        update={"foo": "bar"},
        goto="other_subgraph",  # trong đó `other_subgraph` là một node trong graph cha
        graph=Command.PARENT
    )
```

Hãy minh hoạ điều này bằng ví dụ ở trên. Ta sẽ làm việc này bằng cách biến `nodeA` trong ví dụ trên thành một graph một-node mà ta sẽ thêm làm subgraph vào graph cha của mình.

!!! warning "Cập nhật state với `Command.PARENT`"
    Khi bạn gửi cập nhật từ một node subgraph tới một node graph cha cho một key được chia sẻ bởi cả [state schema](./graph-api.md#schema) của cha lẫn subgraph, bạn **bắt buộc** phải định nghĩa một [reducer](./graph-api.md#reducers) cho key mà bạn đang cập nhật trong state của graph cha. Xem ví dụ bên dưới.

```python
import operator
from typing_extensions import Annotated

class State(TypedDict):
    # LƯU Ý: ta định nghĩa một reducer ở đây
    foo: Annotated[str, operator.add]  # [!code highlight]

def node_a(state: State):
    print("Called A")
    value = random.choice(["a", "b"])
    # đây là thay thế cho một hàm conditional edge
    if value == "a":
        goto = "node_b"
    else:
        goto = "node_c"

    # lưu ý Command cho phép bạn VỪA cập nhật state của graph VỪA định tuyến tới node tiếp theo
    return Command(
        update={"foo": value},
        goto=goto,
        # điều này báo cho LangGraph điều hướng tới node_b hoặc node_c trong graph cha
        # LƯU Ý: việc này sẽ điều hướng tới graph cha gần nhất tính từ subgraph
        graph=Command.PARENT,  # [!code highlight]
    )

subgraph = StateGraph(State).add_node(node_a).add_edge(START, "node_a").compile()

def node_b(state: State):
    print("Called B")
    # LƯU Ý: vì ta đã định nghĩa một reducer, ta không cần tự nối thêm
    # ký tự mới vào giá trị 'foo' hiện có. thay vào đó, reducer sẽ tự động
    # nối chúng (thông qua operator.add)
    return {"foo": "b"}  # [!code highlight]

def node_c(state: State):
    print("Called C")
    return {"foo": "c"}  # [!code highlight]

builder = StateGraph(State)
builder.add_edge(START, "subgraph")
builder.add_node("subgraph", subgraph)
builder.add_node(node_b)
builder.add_node(node_c)

graph = builder.compile()
```

```python
graph.invoke({"foo": ""})
```

```
Called A
Called C
```

### Dùng bên trong tool

Một trường hợp dùng phổ biến là cập nhật state của graph từ bên trong một tool. Ví dụ, trong một ứng dụng hỗ trợ khách hàng, bạn có thể muốn tra cứu thông tin khách hàng dựa trên số tài khoản hoặc ID của họ ngay từ đầu cuộc hội thoại. Để cập nhật state của graph từ tool, bạn có thể trả về `Command(update={"my_custom_key": "foo", "messages": [...]})` từ tool:

```python
from langchain.tools import ToolRuntime

@tool
def lookup_user_info(runtime: ToolRuntime):
    """Use this to look up user information to better assist them with their questions."""
    user_info = get_user_info(runtime.server_info.user.identity)  # [!code highlight]
    return Command(
        update={
            # cập nhật các key trong state
            "user_info": user_info,
            # cập nhật lịch sử tin nhắn
            "messages": [ToolMessage("Successfully looked up user information", tool_call_id=runtime.tool_call_id)]
        }
    )
```

!!! warning ""
    Bạn BẮT BUỘC phải bao gồm `messages` (hoặc bất kỳ state key nào dùng cho lịch sử tin nhắn) trong `Command.update` khi trả về [`Command`](https://reference.langchain.com/python/langgraph/types/Command) từ một tool, và danh sách tin nhắn trong `messages` PHẢI chứa một `ToolMessage`. Điều này cần thiết để lịch sử tin nhắn kết quả hợp lệ (các nhà cung cấp LLM yêu cầu AI message có tool call phải được theo sau bởi các tool result message).

Nếu bạn đang dùng các tool cập nhật state qua [`Command`](https://reference.langchain.com/python/langgraph/types/Command), chúng tôi khuyến nghị dùng [`ToolNode`](https://reference.langchain.com/python/langgraph/agents/#langgraph.prebuilt.tool_node.ToolNode) dựng sẵn, tự động xử lý các tool trả về đối tượng [`Command`](https://reference.langchain.com/python/langgraph/types/Command) và lan truyền chúng vào state của graph. Nếu bạn viết một node tuỳ chỉnh gọi tool, bạn sẽ cần tự lan truyền các đối tượng [`Command`](https://reference.langchain.com/python/langgraph/types/Command) mà tool trả về như là cập nhật từ node.

## Trực quan hoá graph

Ở đây ta sẽ minh hoạ cách trực quan hoá các graph bạn tạo ra.

Bạn có thể trực quan hoá bất kỳ [Graph](https://langchain-ai.github.io/langgraph/reference/graphs/) tuỳ ý nào, bao gồm cả [StateGraph](https://langchain-ai.github.io/langgraph/reference/graphs/#langgraph.graph.state.StateGraph).

Hãy vui một chút bằng cách vẽ fractal :).

```python
import random
from typing import Annotated, Literal
from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages

class State(TypedDict):
    messages: Annotated[list, add_messages]

class MyNode:
    def __init__(self, name: str):
        self.name = name
    def __call__(self, state: State):
        return {"messages": [("assistant", f"Called node {self.name}")]}

def route(state) -> Literal["entry_node", END]:
    if len(state["messages"]) > 10:
        return END
    return "entry_node"

def add_fractal_nodes(builder, current_node, level, max_level):
    if level > max_level:
        return
    # Số lượng node cần tạo ở cấp này
    num_nodes = random.randint(1, 3)  # Điều chỉnh độ ngẫu nhiên nếu cần
    for i in range(num_nodes):
        nm = ["A", "B", "C"][i]
        node_name = f"node_{current_node}_{nm}"
        builder.add_node(node_name, MyNode(node_name))
        builder.add_edge(current_node, node_name)
        # Đệ quy thêm nhiều node hơn
        r = random.random()
        if r > 0.2 and level + 1 < max_level:
            add_fractal_nodes(builder, node_name, level + 1, max_level)
        elif r > 0.05:
            builder.add_conditional_edges(node_name, route, node_name)
        else:
            # Kết thúc
            builder.add_edge(node_name, END)

def build_fractal_graph(max_level: int):
    builder = StateGraph(State)
    entry_point = "entry_node"
    builder.add_node(entry_point, MyNode(entry_point))
    builder.add_edge(START, entry_point)
    add_fractal_nodes(builder, entry_point, 1, max_level)
    # Tuỳ chọn: đặt một điểm kết thúc nếu cần
    builder.add_edge(entry_point, END)  # hoặc bất kỳ node cụ thể nào
    return builder.compile()

app = build_fractal_graph(3)
```

### Mermaid

Ta cũng có thể chuyển một graph class thành cú pháp Mermaid.

```python
print(app.get_graph().draw_mermaid())
```

```
%%{init: {'flowchart': {'curve': 'linear'}}}%%
graph TD;
    tart__([<p>__start__</p>]):::first
    ry_node(entry_node)
    e_entry_node_A(node_entry_node_A)
    e_entry_node_B(node_entry_node_B)
    e_node_entry_node_B_A(node_node_entry_node_B_A)
    e_node_entry_node_B_B(node_node_entry_node_B_B)
    e_node_entry_node_B_C(node_node_entry_node_B_C)
    nd__([<p>__end__</p>]):::last
    tart__ --> entry_node;
    ry_node --> __end__;
    ry_node --> node_entry_node_A;
    ry_node --> node_entry_node_B;
    e_entry_node_B --> node_node_entry_node_B_A;
    e_entry_node_B --> node_node_entry_node_B_B;
    e_entry_node_B --> node_node_entry_node_B_C;
    e_entry_node_A -.-> entry_node;
    e_entry_node_A -.-> __end__;
    e_node_entry_node_B_A -.-> entry_node;
    e_node_entry_node_B_A -.-> __end__;
    e_node_entry_node_B_B -.-> entry_node;
    e_node_entry_node_B_B -.-> __end__;
    e_node_entry_node_B_C -.-> entry_node;
    e_node_entry_node_B_C -.-> __end__;
    ssDef default fill:#f2f0ff,line-height:1.2
    ssDef first fill-opacity:0
    ssDef last fill:#bfb6fc
```

### PNG

Nếu muốn, ta có thể render Graph thành file `.png`. Ở đây ta có ba lựa chọn:

* Dùng API của Mermaid.ink (không cần cài thêm package)
* Dùng Mermaid + Pyppeteer (cần `pip install pyppeteer`)
* Dùng graphviz (cần `pip install graphviz`)

**Dùng Mermaid.Ink**

Theo mặc định, `draw_mermaid_png()` dùng API của Mermaid.Ink để tạo sơ đồ.

```python
from IPython.display import Image, display
from langchain_core.runnables.graph import CurveStyle, MermaidDrawMethod, NodeStyles

display(Image(app.get_graph().draw_mermaid_png()))
```

<img src="https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/graph_api_image_10.png?fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=6cb916b7c627e81c2816cc74ebf3f913" alt="Fractal graph visualization" width="2382" height="1131" data-path="oss/images/graph_api_image_10.png" />

**Dùng Mermaid + Pyppeteer**

```python
import nest_asyncio

nest_asyncio.apply()  # Cần thiết để Jupyter Notebook chạy được các hàm async

display(
    Image(
        app.get_graph().draw_mermaid_png(
            curve_style=CurveStyle.LINEAR,
            node_colors=NodeStyles(first="#ffdfba", last="#baffc9", default="#fad7de"),
            wrap_label_n_words=9,
            output_file_path=None,
            draw_method=MermaidDrawMethod.PYPPETEER,
            background_color="white",
            padding=10,
        )
    )
)
```

**Dùng Graphviz**

```python
try:
    display(Image(app.get_graph().draw_png()))
except ImportError:
    print(
        "You likely need to install dependencies for pygraphviz, see more here https://github.com/pygraphviz/pygraphviz/blob/main/INSTALL.txt"
    )
```

---

!!! info "Kết nối tài liệu này"
    [Kết nối tài liệu này](https://docs.langchain.com/use-these-docs) với Claude, VSCode, và nhiều công cụ khác qua MCP để có câu trả lời theo thời gian thực.

!!! info "Đóng góp"
    [Chỉnh sửa trang này trên GitHub](https://github.com/langchain-ai/docs/edit/main/src/oss/langgraph/use-graph-api.mdx) hoặc [báo lỗi](https://github.com/langchain-ai/docs/issues/new/choose).
