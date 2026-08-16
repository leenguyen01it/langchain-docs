# Graph API overview

## Graphs

Về cốt lõi, LangGraph mô hình hoá workflow của agent dưới dạng graph. Bạn định nghĩa hành vi của agent bằng ba thành phần chính:

1. [`State`](#state): Một cấu trúc dữ liệu dùng chung, thể hiện snapshot hiện tại của ứng dụng. Nó có thể là bất kỳ kiểu dữ liệu nào, nhưng thường được định nghĩa bằng một schema state dùng chung.

2. [`Nodes`](#nodes): Các hàm mã hoá logic của agent. Chúng nhận state hiện tại làm input, thực hiện một phép tính hoặc side-effect nào đó, và trả về state đã cập nhật.

3. [`Edges`](#edges): Các hàm xác định `Node` nào sẽ được thực thi tiếp theo dựa trên state hiện tại. Chúng có thể là các nhánh có điều kiện hoặc các chuyển tiếp cố định.

Bằng cách kết hợp `Nodes` và `Edges`, bạn có thể tạo ra các workflow lặp phức tạp, tiến hoá state theo thời gian. Tuy nhiên, sức mạnh thực sự đến từ cách LangGraph quản lý state đó.

Cần nhấn mạnh: `Nodes` và `Edges` không gì khác hơn là các hàm, chúng có thể chứa một LLM hoặc chỉ là code thông thường.

Nói ngắn gọn: *node thực hiện công việc, edge cho biết việc tiếp theo cần làm*.

Thuật toán graph cốt lõi của LangGraph sử dụng [truyền message (message passing)](https://en.wikipedia.org/wiki/Message_passing) để định nghĩa một chương trình tổng quát. Khi một Node hoàn tất thao tác của nó, nó gửi message dọc theo một hoặc nhiều edge tới (các) node khác. Các node nhận này sau đó thực thi hàm của chúng, truyền message kết quả tới tập node tiếp theo, và quá trình tiếp tục. Lấy cảm hứng từ hệ thống [Pregel](https://research.google/pubs/pregel-a-system-for-large-scale-graph-processing/) của Google, chương trình tiến triển theo các "super-step" rời rạc.

Một super-step có thể được coi là một lần lặp duy nhất qua các node của graph. Các node chạy song song thuộc cùng một super-step, trong khi các node chạy tuần tự thuộc các super-step riêng biệt. Khi bắt đầu thực thi graph, tất cả node bắt đầu ở trạng thái `inactive`. Một node trở nên `active` khi nó nhận một message mới (state) trên bất kỳ edge (hay "channel") đến nào của nó. Node active sau đó chạy hàm của nó và phản hồi bằng các cập nhật. Vào cuối mỗi super-step, các node không có message đến sẽ bỏ phiếu `halt` bằng cách tự đánh dấu mình là `inactive`. Việc thực thi graph kết thúc khi tất cả node đều `inactive` và không còn message nào đang truyền đi.

### StateGraph

Lớp [`StateGraph`](https://reference.langchain.com/python/langgraph/graph/state/StateGraph) là lớp graph chính để sử dụng. Nó được tham số hoá bởi một đối tượng `State` do người dùng định nghĩa.

### Compile graph của bạn

Để xây dựng graph, trước tiên bạn định nghĩa [state](#state), sau đó thêm [node](#nodes) và [edge](#edges), rồi compile nó. Compile graph chính xác là gì và tại sao cần thiết?

Compile là một bước khá đơn giản. Nó cung cấp một vài kiểm tra cơ bản về cấu trúc của graph (không có node mồ côi, v.v.). Đây cũng là nơi bạn có thể chỉ định các tham số runtime như [checkpointer](persistence.md) và breakpoint. Bạn compile graph chỉ bằng cách gọi phương thức `.compile`:

```python
graph = graph_builder.compile(...)
```

!!! warning
    Bạn **BẮT BUỘC** phải compile graph trước khi có thể sử dụng nó.

## State

Việc đầu tiên bạn làm khi định nghĩa một graph là định nghĩa `State` của graph. `State` bao gồm [schema của graph](#schema) cũng như các hàm [`reducer`](#reducers) chỉ định cách áp dụng cập nhật vào state. Schema của `State` sẽ là schema input cho tất cả `Nodes` và `Edges` trong graph, và có thể là `TypedDict` hoặc model `Pydantic`. Tất cả `Nodes` sẽ phát ra các cập nhật cho `State`, sau đó được áp dụng bằng hàm `reducer` đã chỉ định.

### Schema

Cách chính thức được tài liệu hoá để chỉ định schema của một graph là dùng [`TypedDict`](https://docs.python.org/3/library/typing.html#typing.TypedDict). Nếu bạn muốn cung cấp giá trị mặc định trong state, hãy dùng [`dataclass`](https://docs.python.org/3/library/dataclasses.html). Chúng tôi cũng hỗ trợ dùng một [`BaseModel`](use-graph-api.md#use-pydantic-models-for-graph-state) của Pydantic làm state của graph nếu bạn muốn kiểm tra dữ liệu đệ quy (recursive data validation) (dù lưu ý rằng Pydantic kém hiệu năng hơn `TypedDict` hoặc `dataclass`).

Mặc định, graph sẽ có cùng schema input và output. Nếu bạn muốn thay đổi điều này, bạn cũng có thể chỉ định trực tiếp schema input và output riêng biệt. Điều này hữu ích khi bạn có nhiều key, và một số dành riêng cho input còn số khác cho output. Xem [hướng dẫn](use-graph-api.md#define-input-and-output-schemas) để biết thêm thông tin.

!!! info
    Factory bậc cao [`create_agent`](../langchain/agents.md) trong `langchain` không hỗ trợ schema state Pydantic.

#### Nhiều schema

Thông thường, tất cả node của graph giao tiếp với một schema duy nhất. Điều này có nghĩa là chúng sẽ đọc và ghi vào cùng các state channel. Nhưng có những trường hợp chúng ta muốn kiểm soát tốt hơn việc này:

* Các node nội bộ có thể truyền thông tin không cần thiết cho input/output của graph.
* Chúng ta cũng có thể muốn dùng schema input/output khác nhau cho graph. Output, ví dụ, có thể chỉ chứa một key output liên quan duy nhất.

Có thể để các node ghi vào các state channel riêng tư (private) bên trong graph cho giao tiếp nội bộ giữa các node. Chúng ta có thể đơn giản định nghĩa một schema riêng tư, `PrivateState`.

Cũng có thể định nghĩa schema input và output tường minh cho một graph. Trong các trường hợp này, chúng ta định nghĩa một schema "nội bộ" chứa *tất cả* key liên quan tới hoạt động của graph. Nhưng chúng ta cũng định nghĩa schema `input` và `output` là tập con của schema "nội bộ" để giới hạn input và output của graph. Xem [Định nghĩa schema input và output](use-graph-api.md#define-input-and-output-schemas) để biết chi tiết hơn.

Hãy cùng xem một ví dụ:

```python
from typing import TypedDict

from langgraph.graph import END, START, StateGraph


class InputState(TypedDict):
    user_input: str


class OutputState(TypedDict):
    graph_output: str


class OverallState(TypedDict):
    foo: str
    user_input: str
    graph_output: str


class PrivateState(TypedDict):
    bar: str


def node_1(state: InputState) -> OverallState:
    # Ghi vào OverallState
    return {"foo": state["user_input"] + " name"}


def node_2(state: OverallState) -> PrivateState:
    # Đọc từ OverallState, ghi vào PrivateState
    return {"bar": state["foo"] + " is"}


def node_3(state: PrivateState) -> OutputState:
    # Đọc từ PrivateState, ghi vào OutputState
    return {"graph_output": state["bar"] + " Lance"}


builder = StateGraph(OverallState, input_schema=InputState, output_schema=OutputState)
builder.add_node("node_1", node_1)
builder.add_node("node_2", node_2)
builder.add_node("node_3", node_3)
builder.add_edge(START, "node_1")
builder.add_edge("node_1", "node_2")
builder.add_edge("node_2", "node_3")
builder.add_edge("node_3", END)

graph = builder.compile()
graph.invoke({"user_input": "My"})
# {'graph_output': 'My name is Lance'}
```

Có hai điểm tinh tế và quan trọng cần lưu ý ở đây:

1. Chúng ta truyền `state: InputState` làm schema input cho `node_1`. Nhưng chúng ta ghi ra `foo`, một channel trong `OverallState`. Làm sao chúng ta có thể ghi ra một state channel không nằm trong schema input? Đó là vì một node *có thể ghi vào bất kỳ state channel nào trong state của graph.* State của graph là hợp (union) của các state channel được định nghĩa lúc khởi tạo, bao gồm `OverallState` và các bộ lọc `InputState` cùng `OutputState`.

2. Chúng ta khởi tạo graph với:

   ```python
   StateGraph(
       OverallState,
       input_schema=InputState,
       output_schema=OutputState
   )
   ```

   Làm sao chúng ta có thể ghi vào `PrivateState` trong `node_2`? Làm sao graph có quyền truy cập schema này nếu nó không được truyền vào lúc khởi tạo `StateGraph`?

   Chúng ta có thể làm vậy vì `_nodes` cũng có thể khai báo thêm `channels_` state miễn là định nghĩa schema state đó tồn tại. Trong trường hợp này, schema `PrivateState` đã được định nghĩa, nên chúng ta có thể thêm `bar` như một state channel mới trong graph và ghi vào nó.

!!! warning
    **Các channel riêng tư không bị ẩn (redact) khi streaming.**

    Các schema input, output và private giới hạn những gì mỗi node *đọc* (schema input của nó) và những gì `invoke` *trả về* (schema output). Chúng **không** ẩn các channel khỏi `stream`.

    Khi bạn stream với `stream_mode="values"`, graph phát ra **tất cả** các state channel của nó theo mặc định, bao gồm cả các channel riêng tư, vì streaming dạng values mặc định trả về toàn bộ tập state channel thay vì schema output. Đây là lý do vì sao một channel riêng tư như `bar` bị ẩn khi dùng `invoke` nhưng lại hiển thị khi streaming:

    ```python
    stream = graph.stream_events({"user_input": "My"}, version="v3")
    for snapshot in stream.values:
        print(snapshot)
    # {'user_input': 'My'}
    # {'foo': 'My name', 'user_input': 'My'}
    # {'foo': 'My name', 'user_input': 'My', 'bar': 'My name is'}        # <-- channel riêng tư
    # {'foo': 'My name', 'user_input': 'My', 'graph_output': 'My name is Lance', 'bar': 'My name is'}
    ```

    Để giới hạn các giá trị được stream về một tập channel cụ thể (ví dụ chỉ schema output), hãy truyền `output_keys`:

    ```python
    stream = graph.stream_events(
        {"user_input": "My"},
        version="v3",
        output_keys=["graph_output"],  # [!code highlight]
    )
    for snapshot in stream.values:
        print(snapshot)
    # {'graph_output': 'My name is Lance'}
    ```

    Nếu bạn chỉ cần các channel mà một node thực sự tạo ra ở mỗi bước (thay vì toàn bộ state tích luỹ), hãy dùng `stream_mode="updates"` thay thế.

### Reducer

Reducer là chìa khoá để hiểu cách các cập nhật từ node được áp dụng vào `State`. Mỗi key trong `State` có hàm reducer độc lập riêng. Nếu không có hàm reducer nào được chỉ định tường minh thì mặc định mọi cập nhật vào key đó sẽ ghi đè nó. Có một vài kiểu reducer khác nhau, bắt đầu với kiểu reducer mặc định:

#### Tham số của reducer

Mỗi reducer là một hàm nhị phân (binary function) với hai tham số vị trí:

* **Tham số bên trái (left)**: Giá trị hiện tại đã lưu trong state cho key đó.
* **Tham số bên phải (right)**: Cập nhật cho key đó được trả về bởi một node.

Khi một node trả về một cập nhật một phần, LangGraph gọi reducer cho mỗi key đã cập nhật và lưu giá trị trả về làm giá trị state mới:

```python
new_value = reducer(left=current_state[key], right=node_update[key])
```

Tham số bên trái luôn đến từ state tích luỹ. Tham số bên phải luôn đến từ cập nhật node mới nhất. Ví dụ sau đặt tên tường minh cho cả hai tham số:

```python
from typing import Annotated

from typing_extensions import TypedDict


def append_strings(left: list[str], right: list[str]) -> list[str]:
    """Kết hợp giá trị state hiện có (left) với cập nhật từ node (right)."""
    return left + right


class State(TypedDict):
    tags: Annotated[list[str], append_strings]
```

Giả sử state là `{"tags": ["draft"]}` và một node trả về `{"tags": ["review"]}`. LangGraph gọi:

```python
append_strings(left=["draft"], right=["review"])  # trả về ["draft", "review"]
```

Giá trị state mới cho `tags` là `["draft", "review"]`.

Reducer tuỳ chỉnh kết hợp tham số bên trái và bên phải. [Reducer mặc định](#default-reducer) bỏ qua tham số bên trái và chỉ giữ tham số bên phải.

#### Reducer mặc định

Reducer mặc định bỏ qua tham số bên trái và thay thế giá trị state bằng tham số bên phải. Ví dụ này cho thấy cách dùng reducer mặc định:

```python
from typing_extensions import TypedDict


class State(TypedDict):
    foo: int
    bar: list[str]
```

Trong ví dụ này, không có hàm reducer nào được chỉ định cho bất kỳ key nào. Giả sử input cho graph là:

`{"foo": 1, "bar": ["hi"]}`. Giả sử `Node` đầu tiên trả về `{"foo": 2}`. Điều này được coi là một cập nhật cho state. Lưu ý rằng `Node` không cần trả về toàn bộ schema `State`, chỉ cần một cập nhật. Sau khi áp dụng cập nhật này, `State` sẽ là `{"foo": 2, "bar": ["hi"]}`. Nếu node thứ hai trả về `{"bar": ["bye"]}` thì `State` sẽ là `{"foo": 2, "bar": ["bye"]}`

#### Reducer tuỳ chỉnh

Một reducer tuỳ chỉnh kết hợp tham số bên trái và bên phải thay vì thay thế giá trị state, điều này hữu ích để tích luỹ giá trị, chẳng hạn nối thêm các cập nhật vào một list. Ví dụ này cho thấy cách chỉ định một reducer tuỳ chỉnh:

```python
from operator import add
from typing import Annotated

from typing_extensions import TypedDict


class State(TypedDict):
    foo: int
    bar: Annotated[list[str], add]
```

Trong ví dụ này, chúng ta đã dùng kiểu `Annotated` để chỉ định một hàm reducer (`operator.add`) cho key thứ hai (`bar`). Lưu ý key đầu tiên không thay đổi. Giả sử input cho graph là `{"foo": 1, "bar": ["hi"]}`. Giả sử `Node` đầu tiên trả về `{"foo": 2}`. Điều này được coi là một cập nhật cho state. Lưu ý rằng `Node` không cần trả về toàn bộ schema `State`, chỉ cần một cập nhật. Sau khi áp dụng cập nhật này, `State` sẽ là `{"foo": 2, "bar": ["hi"]}`. Nếu node thứ hai trả về `{"bar": ["bye"]}` thì `State` sẽ là `{"foo": 2, "bar": ["hi", "bye"]}`. Lưu ý ở đây key `bar` được cập nhật bằng cách cộng hai list lại với nhau.

#### Overwrite

!!! tip
    Trong một số trường hợp, bạn có thể muốn bỏ qua một reducer và ghi đè trực tiếp một giá trị state. LangGraph cung cấp kiểu [`Overwrite`](https://reference.langchain.com/python/langgraph/types/) cho mục đích này. [Xem cách dùng `Overwrite` tại đây](use-graph-api.md#bypass-reducers-with-overwrite).

### Làm việc với message trong state của graph

#### Vì sao dùng message?

Hầu hết các nhà cung cấp LLM hiện đại có giao diện chat model chấp nhận một list message làm input. Giao diện [chat model](../langchain/models.md) của LangChain nói riêng chấp nhận một list đối tượng message làm input. Các message này có nhiều dạng như [`HumanMessage`](https://reference.langchain.com/python/langchain-core/messages/human/HumanMessage) (input của người dùng) hoặc [`AIMessage`](https://reference.langchain.com/python/langchain-core/messages/ai/AIMessage) (phản hồi của LLM).

Để đọc thêm về đối tượng message là gì, hãy tham khảo [hướng dẫn khái niệm Messages](../langchain/messages.md).

#### Dùng message trong graph của bạn

Trong nhiều trường hợp, việc lưu lịch sử hội thoại trước đó dưới dạng một list message trong state của graph rất hữu ích. Để làm vậy, chúng ta có thể thêm một key (channel) vào state của graph lưu một list đối tượng `Message` và gắn annotation bằng một hàm reducer (xem key `messages` trong ví dụ dưới đây). Hàm reducer rất quan trọng để cho graph biết cách cập nhật list đối tượng `Message` trong state với mỗi lần cập nhật state (ví dụ, khi một node gửi một cập nhật). Nếu bạn không chỉ định reducer, mỗi lần cập nhật state sẽ ghi đè list message bằng giá trị được cung cấp gần nhất. Nếu bạn chỉ muốn nối thêm message vào list hiện có, bạn có thể dùng `operator.add` làm reducer.

Tuy nhiên, bạn cũng có thể muốn cập nhật thủ công message trong state của graph (ví dụ human-in-the-loop). Nếu bạn dùng `operator.add`, các cập nhật state thủ công bạn gửi tới graph sẽ được nối thêm vào list message hiện có, thay vì cập nhật các message hiện có. Để tránh điều đó, bạn cần một reducer có thể theo dõi message ID và ghi đè các message hiện có, nếu được cập nhật. Để đạt được điều này, bạn có thể dùng hàm dựng sẵn [`add_messages`](https://reference.langchain.com/python/langgraph/graph/message/add_messages). Với các message hoàn toàn mới, nó sẽ đơn giản nối thêm vào list hiện có, nhưng nó cũng sẽ xử lý đúng các cập nhật cho message hiện có.

#### Serialization

Bên cạnh việc theo dõi message ID, hàm [`add_messages`](https://reference.langchain.com/python/langgraph/graph/message/add_messages) cũng sẽ cố gắng deserialize message thành đối tượng `Message` của LangChain bất cứ khi nào một cập nhật state được nhận trên channel `messages`.

Để biết thêm thông tin, xem [serialization/deserialization của LangChain](https://python.langchain.com/docs/how_to/serialization/). Điều này cho phép gửi input/cập nhật state của graph theo định dạng sau:

```python
# hỗ trợ dạng này
{"messages": [HumanMessage(content="message")]}

# và dạng này cũng được hỗ trợ
{"messages": [{"type": "human", "content": "message"}]}
```

Vì các cập nhật state luôn được deserialize thành `Messages` của LangChain khi dùng [`add_messages`](https://reference.langchain.com/python/langgraph/graph/message/add_messages), bạn nên dùng dot notation để truy cập thuộc tính message, như `state["messages"][-1].content`.

Dưới đây là ví dụ về một graph dùng [`add_messages`](https://reference.langchain.com/python/langgraph/graph/message/add_messages) làm hàm reducer của nó.

```python
from langchain.messages import AnyMessage
from langgraph.graph.message import add_messages
from typing import Annotated
from typing_extensions import TypedDict

class GraphState(TypedDict):
    messages: Annotated[list[AnyMessage], add_messages]
```

#### MessagesState

Vì việc có một list message trong state là rất phổ biến, có sẵn một state dựng sẵn tên `MessagesState` giúp dễ dàng dùng message. `MessagesState` được định nghĩa với một key `messages` duy nhất là một list đối tượng `AnyMessage` và dùng reducer [`add_messages`](https://reference.langchain.com/python/langgraph/graph/message/add_messages). Thông thường, có nhiều state cần theo dõi hơn là chỉ message, nên chúng ta thấy mọi người subclass state này và thêm nhiều trường hơn, như:

```python
from langgraph.graph import MessagesState

class State(MessagesState):
    documents: list[str]
```

## Nodes

Trong LangGraph, node là các hàm Python (đồng bộ hoặc bất đồng bộ) chấp nhận các tham số sau:

1. `state`, [state](#state) của graph
2. `config`, một đối tượng [`RunnableConfig`](https://reference.langchain.com/python/langchain-core/runnables/config/RunnableConfig) chứa thông tin cấu hình như `thread_id` và thông tin tracing như `tags`
3. `runtime`, một đối tượng `Runtime` chứa [`context`](#runtime-context) runtime và thông tin khác như `store`, `stream_writer`, `execution_info`, `server_info`, `heartbeat` (để làm mới idle timeout), và `control` (cho [tắt an toàn (graceful shutdown)](fault-tolerance.md#graceful-shutdown))

Tương tự `NetworkX`, bạn thêm các node này vào một graph bằng phương thức [`add_node`](https://reference.langchain.com/python/langgraph/graph/state/StateGraph/add_node):

```python
from dataclasses import dataclass
from typing_extensions import TypedDict

from langgraph.graph import StateGraph
from langgraph.runtime import Runtime

class State(TypedDict):
    input: str
    results: str

@dataclass
class Context:
    user_id: str

builder = StateGraph(State)

def plain_node(state: State):
    return state

def node_with_runtime(state: State, runtime: Runtime[Context]):
    print("In node: ", runtime.context.user_id)
    return {"results": f"Hello, {state['input']}!"}

def node_with_execution_info(state: State, runtime: Runtime):
    print("In node with thread_id: ", runtime.execution_info.thread_id)  # [!code highlight]
    return {"results": f"Hello, {state['input']}!"}


builder.add_node("plain_node", plain_node)
builder.add_node("node_with_runtime", node_with_runtime)
builder.add_node("node_with_execution_info", node_with_execution_info)
...
```

Ở phía sau, các hàm được chuyển đổi thành [`RunnableLambda`](https://reference.langchain.com/python/langchain-core/runnables/base/RunnableLambda), giúp thêm hỗ trợ batch và async cho hàm của bạn, cùng với [tracing và debug gốc (native)](https://docs.langchain.com/langsmith/observability).

Nếu bạn thêm một node vào graph mà không chỉ định tên, nó sẽ được đặt tên mặc định bằng tên hàm.

```python
builder.add_node(my_node)
# Sau đó bạn có thể tạo edge đến/từ node này bằng cách tham chiếu nó là `"my_node"`
```

### Thực thi lại và tính idempotent

Khi bạn compile với một [checkpointer](persistence.md), LangGraph lưu checkpoint tại ranh giới [super-step](#graphs), không phải giữa chừng hàm bên trong một node. Nếu việc thực thi dừng lại và sau đó tiếp tục (ví dụ sau một [interrupt](interrupts.md) hoặc một [retry](fault-tolerance.md#retries)), **node** bị ảnh hưởng sẽ chạy lại từ đầu hàm của nó. Code và side-effect trước khi tạm dừng sẽ chạy lại.

**Tính idempotent.** Thiết kế logic **node** sao cho việc thực thi lại không làm hỏng state. Nếu một node chèn một dòng vào database, chạy nó hai lần không nên tạo ra các dòng trùng lặp trừ khi đó là chủ đích. Dùng idempotency key, upsert, hoặc kiểm tra trước khi ghi (read-before-write). Đối với các side-effect xung quanh `interrupt()`, xem [Side-effect được gọi trước `interrupt` phải mang tính idempotent](interrupts.md#side-effects-called-before-interrupt-must-be-idempotent).

**Thay đổi graph.** Các quy tắc [tính xác định (determinism)](functional-api.md#determinism) về thay đổi code không áp dụng cho cấu trúc graph. Bạn có thể thêm hoặc xoá **node** và edge mà không làm hỏng việc resume cho các thread hiện có. Các lần chạy resume dùng state đã lưu và thực thi bất kỳ graph nào bạn compile hiện tại.

**Task và interrupt bên trong một node.** Nếu một **node** gọi [**task**](functional-api.md#task) hoặc [`interrupt`](https://reference.langchain.com/python/langgraph/types/interrupt), các quy tắc xác định nghiêm ngặt hơn áp dụng khi resume. LangGraph khôi phục kết quả **task** đã hoàn thành từ checkpointer, nhưng việc thay đổi thứ tự **task** hoặc [`interrupt`](https://reference.langchain.com/python/langgraph/types/interrupt) trong code trước điểm resume có thể làm sai lệch các giá trị đã cache. Một **entrypoint** của [Functional API](functional-api.md) compile thành một **node** duy nhất chạy toàn bộ phương thức entrypoint theo cách này. Xem [Tính xác định](functional-api.md#determinism), [Tính idempotent](functional-api.md#idempotency), và [Dùng task trong node](#using-tasks-in-nodes).

### Dùng task trong node

Nếu một [node](#nodes) chứa nhiều thao tác, bạn có thể thấy dễ dàng hơn khi triển khai mỗi thao tác như một [**task**](functional-api.md#task) thay vì chia logic ra nhiều node. Kết quả task được checkpoint khi graph dùng checkpointer, nên việc resume một thread có thể bỏ qua công việc **task** đã hoàn thành bên trong node.

=== "Bản gốc"
    ```python
    from typing import NotRequired

    import requests
    from langchain_core.utils.uuid import uuid7
    from langgraph.checkpoint.memory import InMemorySaver
    from langgraph.graph import END, START, StateGraph
    from typing_extensions import TypedDict


    class State(TypedDict):
        url: str
        result: NotRequired[str]


    def call_api(state: State):
        """Node ví dụ thực hiện một request API."""
        result = requests.get(state["url"]).text[:100]  # [!code highlight]
        return {"result": result}


    builder = StateGraph(State)
    builder.add_node("call_api", call_api)
    builder.add_edge(START, "call_api")
    builder.add_edge("call_api", END)

    checkpointer = InMemorySaver()
    graph = builder.compile(checkpointer=checkpointer)

    thread_id = str(uuid7())
    config = {"configurable": {"thread_id": thread_id}}

    graph.invoke({"url": "https://www.example.com"}, config)
    ```

=== "Với task"
    ```python
    from typing import NotRequired

    import requests
    from langchain_core.utils.uuid import uuid7
    from langgraph.checkpoint.memory import InMemorySaver
    from langgraph.func import task
    from langgraph.graph import END, START, StateGraph
    from typing_extensions import TypedDict


    class State(TypedDict):
        urls: list[str]
        results: NotRequired[list[str]]


    @task
    def _make_request(url: str):
        """Thực hiện một request."""
        return requests.get(url).text[:100]  # [!code highlight]


    def call_api(state: State):
        """Node ví dụ thực hiện các request API dưới dạng task được checkpoint."""
        futures = [_make_request(url) for url in state["urls"]]  # [!code highlight]
        results = [f.result() for f in futures]
        return {"results": results}


    builder = StateGraph(State)
    builder.add_node("call_api", call_api)
    builder.add_edge(START, "call_api")
    builder.add_edge("call_api", END)

    checkpointer = InMemorySaver()
    graph = builder.compile(checkpointer=checkpointer)

    thread_id = str(uuid7())
    config = {"configurable": {"thread_id": thread_id}}

    graph.invoke({"urls": ["https://www.example.com"]}, config)
    ```

### Node `START`

Node [`START`](https://reference.langchain.com/python/langgraph/constants/START) là một node đặc biệt đại diện cho node gửi input của người dùng vào graph. Mục đích chính của việc tham chiếu node này là xác định node nào nên được gọi đầu tiên.

```python
from langgraph.graph import START

graph.add_edge(START, "node_a")
```

### Node `END`

Node `END` là một node đặc biệt đại diện cho một node kết thúc (terminal). Node này được tham chiếu khi bạn muốn biểu thị edge nào không còn hành động nào sau khi hoàn tất.

```python
from langgraph.graph import END

graph.add_edge("node_a", END)
```

### Cache node

LangGraph hỗ trợ cache task/node dựa trên input của node. Để dùng caching:

* Chỉ định một cache khi compile một graph (hoặc chỉ định một entrypoint)
* Chỉ định một cache policy cho node. Mỗi cache policy hỗ trợ:
  * `key_func` dùng để tạo cache key dựa trên input của node, mặc định là một `hash` của input bằng pickle.
  * `ttl`, thời gian tồn tại (time to live) cho cache tính bằng giây. Nếu không chỉ định, cache sẽ không bao giờ hết hạn.

Ví dụ:

```python
import time
from typing_extensions import TypedDict
from langgraph.graph import StateGraph
from langgraph.cache.memory import InMemoryCache
from langgraph.types import CachePolicy


class State(TypedDict):
    x: int
    result: int


builder = StateGraph(State)


def expensive_node(state: State) -> dict[str, int]:
    # phép tính tốn kém
    time.sleep(2)
    return {"result": state["x"] * 2}


builder.add_node("expensive_node", expensive_node, cache_policy=CachePolicy(ttl=3))
builder.set_entry_point("expensive_node")
builder.set_finish_point("expensive_node")

graph = builder.compile(cache=InMemoryCache())

print(graph.invoke({"x": 5}, stream_mode='updates'))    # [!code highlight]
# [{'expensive_node': {'result': 10}}]
print(graph.invoke({"x": 5}, stream_mode='updates'))    # [!code highlight]
# [{'expensive_node': {'result': 10}, '__metadata__': {'cached': True}}]
```

!!! note
    `set_entry_point(node)` định nghĩa node đầu tiên graph sẽ thực thi.
    Nó tương đương với `builder.add_edge(START, node)`.

    `set_finish_point(node)` định nghĩa node cuối cùng trong graph.
    Nó tương đương với `builder.add_edge(node, END)`.

    Cả hai phương thức đều hợp lệ nhưng `add_edge(START, ...)` và `add_edge(..., END)`
    là cú pháp hiện đại được khuyến nghị.

1. Lần chạy đầu tiên mất hai giây để chạy (do phép tính tốn kém giả lập).
2. Lần chạy thứ hai tận dụng cache và trả về nhanh chóng.

## Edges

Edge định nghĩa cách logic được định tuyến và cách graph quyết định dừng lại. Đây là phần lớn trong cách các agent của bạn hoạt động và cách các node khác nhau giao tiếp với nhau. Có một vài loại edge chính:

* Normal Edges (edge thường): Đi trực tiếp từ node này sang node tiếp theo.
* Conditional Edges (edge có điều kiện): Gọi một hàm để xác định (các) node nào sẽ đi tới tiếp theo.
* Entry Point (điểm vào): Node nào sẽ được gọi đầu tiên khi input người dùng đến.
* Conditional Entry Point (điểm vào có điều kiện): Gọi một hàm để xác định (các) node nào sẽ được gọi đầu tiên khi input người dùng đến.

Một node có thể có nhiều edge đi ra. Nếu một node có nhiều edge đi ra, **tất cả** các node đích đó sẽ được thực thi song song như một phần của superstep tiếp theo.

!!! warning
    Với mỗi node, chọn một cơ chế định tuyến duy nhất: dùng edge thường cho định tuyến tĩnh, hoặc dùng edge có điều kiện / [`Command`](https://reference.langchain.com/python/langgraph/types/Command) cho định tuyến động. Không trộn edge thường và định tuyến động từ cùng một node, vì cả hai đường có thể cùng thực thi và khiến hành vi của graph khó suy luận hơn.

### Edge thường

Nếu bạn **luôn luôn** muốn đi từ node A sang node B, bạn có thể dùng trực tiếp phương thức [`add_edge`](https://reference.langchain.com/python/langgraph/pregel/_draw/add_edge).

```python
graph.add_edge("node_a", "node_b")
```

### Edge có điều kiện

Nếu bạn muốn **tuỳ chọn** định tuyến tới một hoặc nhiều edge (hoặc tuỳ chọn kết thúc), bạn có thể dùng phương thức [`add_conditional_edges`](https://reference.langchain.com/python/langgraph/graph/state/StateGraph/add_conditional_edges). Phương thức này nhận tên của một node và một "hàm định tuyến" để gọi sau khi node đó được thực thi:

```python
graph.add_conditional_edges("node_a", routing_function)
```

Tương tự node, `routing_function` nhận `state` hiện tại của graph và trả về một giá trị.

Mặc định, giá trị trả về của `routing_function` được dùng làm tên của node (hoặc list node) để gửi state tới tiếp theo. Tất cả các node đó sẽ chạy song song như một phần của superstep tiếp theo.

Bạn có thể tuỳ chọn cung cấp một dictionary ánh xạ output của `routing_function` tới tên của node tiếp theo.

```python
graph.add_conditional_edges("node_a", routing_function, {True: "node_b", False: "node_c"})
```

!!! tip
    Dùng [`Command`](#command) thay vì edge có điều kiện nếu bạn muốn kết hợp cập nhật state và định tuyến trong một hàm duy nhất.

### Entry point

Entry point là (các) node đầu tiên được chạy khi graph bắt đầu. Bạn có thể dùng phương thức [`add_edge`](https://reference.langchain.com/python/langgraph/pregel/_draw/add_edge) từ node ảo [`START`](https://reference.langchain.com/python/langgraph/constants/START) tới node đầu tiên cần thực thi để chỉ định nơi vào graph.

```python
from langgraph.graph import START

graph.add_edge(START, "node_a")
```

### Conditional entry point

Một conditional entry point cho phép bạn bắt đầu tại các node khác nhau tuỳ theo logic tuỳ chỉnh. Bạn có thể dùng [`add_conditional_edges`](https://reference.langchain.com/python/langgraph/graph/state/StateGraph/add_conditional_edges) từ node ảo [`START`](https://reference.langchain.com/python/langgraph/constants/START) để thực hiện điều này.

```python
from langgraph.graph import START

graph.add_conditional_edges(START, routing_function)
```

Bạn có thể tuỳ chọn cung cấp một dictionary ánh xạ output của `routing_function` tới tên của node tiếp theo.

```python
graph.add_conditional_edges(START, routing_function, {True: "node_b", False: "node_c"})
```

## `Send`

Mặc định, `Nodes` và `Edges` được định nghĩa trước và hoạt động trên cùng một state dùng chung. Tuy nhiên, có những trường hợp các edge chính xác không được biết trước và/hoặc bạn muốn nhiều phiên bản của `State` tồn tại cùng lúc. Một ví dụ phổ biến cho điều này là với các pattern thiết kế [map-reduce](use-graph-api.md#map-reduce-and-the-send-api). Trong pattern thiết kế này, một node đầu tiên có thể tạo ra một list đối tượng, và bạn có thể muốn áp dụng một node khác cho tất cả các đối tượng đó. Số lượng đối tượng có thể không biết trước (nghĩa là số lượng edge có thể không biết trước) và `State` input cho `Node` phía sau nên khác nhau (một cho mỗi đối tượng được tạo ra).

Để hỗ trợ pattern thiết kế này, LangGraph hỗ trợ trả về đối tượng [`Send`](https://reference.langchain.com/python/langgraph/types/Send) từ các edge có điều kiện. `Send` nhận hai tham số: đầu tiên là tên của node, và thứ hai là state truyền cho node đó.

```python
from langgraph.types import Send

def continue_to_jokes(state: OverallState):
    return [Send("generate_joke", {"subject": s}) for s in state['subjects']]

graph.add_conditional_edges("node_a", continue_to_jokes)
```

## `Command`

[`Command`](https://reference.langchain.com/python/langgraph/types/Command) là một primitive linh hoạt để điều khiển việc thực thi graph. Nó chấp nhận bốn tham số:

* `update`: Áp dụng cập nhật state (tương tự trả về cập nhật từ một node).
* `goto`: Điều hướng tới các node cụ thể (tương tự [edge có điều kiện](#conditional-edges)).
* `graph`: Nhắm tới một graph cha khi điều hướng từ [subgraph](use-subgraphs.md).
* `resume`: Cung cấp một giá trị để tiếp tục thực thi sau một [interrupt](interrupts.md).

`Command` được dùng trong ba ngữ cảnh:

* **[Trả về từ node](#return-from-nodes)**: Dùng `update`, `goto`, và `graph` để kết hợp cập nhật state với luồng điều khiển.
* **[Input cho `invoke` hoặc `stream`](#input-to-invoke-or-stream)**: Dùng `resume` để tiếp tục thực thi sau một interrupt.
* **[Trả về từ tool](#return-from-tools)**: Tương tự trả về từ node, kết hợp cập nhật state và luồng điều khiển từ bên trong một tool.

### Trả về từ node

#### `update` và `goto`

Trả về [`Command`](https://reference.langchain.com/python/langgraph/types/Command) từ hàm node để cập nhật state và định tuyến tới node tiếp theo trong một bước duy nhất:

```python
def my_node(state: State) -> Command[Literal["my_other_node"]]:
    return Command(
        # cập nhật state
        update={"foo": "bar"},
        # luồng điều khiển
        goto="my_other_node"
    )
```

Với [`Command`](https://reference.langchain.com/python/langgraph/types/Command) bạn cũng có thể đạt được hành vi luồng điều khiển động (giống hệt [edge có điều kiện](#conditional-edges)):

```python
def my_node(state: State) -> Command[Literal["my_other_node"]]:
    if state["foo"] == "bar":
        return Command(update={"foo": "baz"}, goto="my_other_node")
```

Dùng [`Command`](https://reference.langchain.com/python/langgraph/types/Command) khi bạn cần **vừa** cập nhật state **vừa** định tuyến tới một node khác. Nếu bạn chỉ cần định tuyến mà không cập nhật state, hãy dùng [edge có điều kiện](#conditional-edges) thay thế.

!!! note
    Khi trả về [`Command`](https://reference.langchain.com/python/langgraph/types/Command) trong hàm node của bạn, bạn phải thêm annotation kiểu trả về với list tên node mà node đó định tuyến tới, ví dụ `Command[Literal["my_other_node"]]`. Điều này cần thiết để render graph và cho LangGraph biết `my_node` có thể điều hướng tới `my_other_node`.

!!! warning
    [`Command`](https://reference.langchain.com/python/langgraph/types/Command) chỉ thêm edge động, các edge tĩnh định nghĩa bằng `add_edge` / `addEdge` vẫn thực thi. Ví dụ, nếu `node_a` trả về `Command(goto="my_other_node")` và bạn cũng có `graph.add_edge("node_a", "node_b")`, cả `node_b` và `my_other_node` sẽ chạy. Với mỗi node, hãy dùng hoặc [`Command`](https://reference.langchain.com/python/langgraph/types/Command) hoặc edge tĩnh để định tuyến tới node tiếp theo, không dùng cả hai.

Xem [hướng dẫn cách làm](use-graph-api.md#combine-control-flow-and-state-updates-with-command) này để có ví dụ đầy đủ về cách dùng [`Command`](https://reference.langchain.com/python/langgraph/types/Command).

#### `graph`

Nếu bạn đang dùng [subgraph](use-subgraphs.md), bạn có thể điều hướng từ một node bên trong subgraph tới một node khác trong graph cha bằng cách chỉ định `graph=Command.PARENT` trong [`Command`](https://reference.langchain.com/python/langgraph/types/Command):

```python
def my_node(state: State) -> Command[Literal["other_subgraph"]]:
    return Command(
        update={"foo": "bar"},
        goto="other_subgraph",  # với `other_subgraph` là một node trong graph cha
        graph=Command.PARENT
    )
```

!!! note
    Đặt `graph` thành `Command.PARENT` sẽ điều hướng tới graph cha gần nhất.

    Khi bạn gửi cập nhật từ một node subgraph tới một node graph cha cho một key được chia sẻ bởi cả [schema state](#schema) của cha lẫn subgraph, bạn **phải** định nghĩa một [reducer](#reducers) cho key bạn đang cập nhật trong state của graph cha. Xem [ví dụ này](use-graph-api.md#navigate-to-a-node-in-a-parent-graph).

Điều này đặc biệt hữu ích khi triển khai [handoff đa agent](../langchain/multi-agent/handoffs.md). Xem [Điều hướng tới một node trong graph cha](use-graph-api.md#navigate-to-a-node-in-a-parent-graph) để biết chi tiết.

### Input cho `invoke` hoặc `stream`

!!! warning
    `Command(resume=...)` là pattern **duy nhất** của `Command` được dùng làm input cho `invoke()`/`stream()` (tuỳ chọn kết hợp với `update=...` để cũng áp dụng thay đổi state khi resume). Không dùng `Command(update=...)` một mình làm input để tiếp tục hội thoại nhiều lượt, vì việc truyền bất kỳ `Command` nào làm input sẽ resume từ checkpoint mới nhất (tức bước cuối cùng đã chạy, không phải `__start__`), graph sẽ có vẻ như bị treo nếu nó đã hoàn tất. Để tiếp tục một hội thoại trên một thread hiện có, hãy truyền một dict input thông thường:

    ```python
    # SAI - graph resume từ checkpoint mới nhất
    # (bước cuối cùng đã chạy), có vẻ như bị treo
    graph.invoke(Command(update={  # [!code --]
        "messages": [{"role": "user", "content": "follow up"}]  # [!code --]
    }), config)  # [!code --]

    # ĐÚNG - dict thông thường khởi động lại từ __start__
    graph.invoke( {  # [!code ++]
        "messages": [{"role": "user", "content": "follow up"}]  # [!code ++]
    }, config)  # [!code ++]
    ```

#### `resume`

Dùng `Command(resume=...)` để cung cấp một giá trị và tiếp tục thực thi graph sau một [interrupt](interrupts.md). Giá trị truyền vào `resume` trở thành giá trị trả về của lệnh gọi `interrupt()` bên trong node đang tạm dừng:

```python
from typing import TypedDict

from langgraph.checkpoint.memory import InMemorySaver
from langgraph.graph import END, START, StateGraph
from langgraph.types import Command, interrupt


class State(TypedDict):
    messages: list[dict]


def human_review(state: State):
    # Tạm dừng graph và chờ một giá trị
    answer = interrupt("Do you approve?")
    return {"messages": [{"role": "user", "content": answer}]}


graph = (
    StateGraph(State)
    .add_node("human_review", human_review)
    .add_edge(START, "human_review")
    .add_edge("human_review", END)
    .compile(checkpointer=InMemorySaver())
)

config = {"configurable": {"thread_id": "graph-api-resume"}}

# Lần chạy đầu tiên - gặp interrupt và tạm dừng
stream = graph.stream_events({"messages": []}, config, version="v3")
_ = stream.output  # đưa stream chạy tới khi hoàn tất
print(stream.interrupts)

# Resume với một giá trị - lệnh gọi interrupt() trả về "yes"
resumed = graph.stream_events(Command(resume="yes"), config, version="v3")
final = resumed.output
```

Xem [hướng dẫn khái niệm interrupt](interrupts.md) để biết đầy đủ chi tiết về các pattern interrupt, bao gồm nhiều interrupt và vòng lặp kiểm tra hợp lệ.

### Trả về từ tool

Bạn có thể trả về [`Command`](https://reference.langchain.com/python/langgraph/types/Command) từ tool để cập nhật state của graph và luồng điều khiển. Dùng `update` để sửa đổi state (ví dụ, lưu thông tin khách hàng đã tra cứu trong một hội thoại) và `goto` để định tuyến tới một node cụ thể sau khi tool hoàn tất.

!!! warning
    Khi dùng bên trong tool, `goto` thêm một edge động, mọi edge tĩnh đã được định nghĩa trên node đã gọi tool đó vẫn sẽ thực thi. Với mỗi node, hãy dùng hoặc định tuyến động do tool điều khiển hoặc edge tĩnh để định tuyến tới node tiếp theo, không dùng cả hai.

Tham khảo [Dùng bên trong tool](use-graph-api.md#use-inside-tools) để biết chi tiết.

## Migration graph

LangGraph có thể dễ dàng xử lý việc migrate định nghĩa graph (node, edge, và state) ngay cả khi dùng một checkpointer để theo dõi state.

* Với các thread đã ở cuối graph (tức không bị interrupt) bạn có thể thay đổi toàn bộ cấu trúc topology của graph (tức tất cả node và edge, xoá, thêm, đổi tên, v.v.)
* Với các thread hiện đang bị interrupt, chúng tôi hỗ trợ mọi thay đổi topology khác ngoại trừ đổi tên/xoá node (vì thread đó có thể sắp vào một node không còn tồn tại nữa), nếu đây là điều cản trở bạn, hãy liên hệ và chúng tôi có thể ưu tiên một giải pháp.
* Đối với việc sửa đổi state, chúng tôi có khả năng tương thích ngược và xuôi đầy đủ cho việc thêm và xoá key
* Các state key bị đổi tên sẽ mất state đã lưu trong các thread hiện có
* Các state key có kiểu thay đổi theo cách không tương thích hiện có thể gây ra vấn đề trong các thread có state từ trước khi thay đổi, nếu đây là điều cản trở bạn, hãy liên hệ và chúng tôi có thể ưu tiên một giải pháp.

!!! tip
    Đối với các thay đổi về mặt kỹ thuật là tương thích nhưng làm thay đổi logic nghiệp vụ, chẳng hạn viết lại tập tool hoặc tái cấu trúc luồng hội thoại, xem [Tương thích nghiệp vụ (Business compatibility)](backward-compatibility.md#business-compatibility). Trang đó bàn về việc ghim một phiên bản hành vi trong state để các thread hiện có giữ đường đi cũ trong khi các thread mới dùng phiên bản mới nhất.

## Runtime context

Khi tạo một graph, bạn có thể chỉ định một `context_schema` cho context runtime truyền tới node. Điều này hữu ích để truyền thông tin tới node mà không phải là một phần của state graph. Ví dụ, bạn có thể muốn truyền các dependency như tên model hoặc một kết nối database.

```python
@dataclass
class ContextSchema:
    llm_provider: str = "openai"

graph = StateGraph(State, context_schema=ContextSchema)
```

Sau đó bạn có thể truyền context này vào graph bằng tham số `context` của phương thức `invoke`.

```python
graph.invoke(inputs, context={"llm_provider": "anthropic"})
```

Sau đó bạn có thể truy cập và dùng context này bên trong một node hoặc conditional edge:

```python
from langgraph.runtime import Runtime

def node_a(state: State, runtime: Runtime[ContextSchema]):
    llm = get_llm(runtime.context.llm_provider)
    # ...
```

Xem [Thêm cấu hình runtime](use-graph-api.md#add-runtime-configuration) để biết đầy đủ chi tiết về cấu hình.

### Giới hạn đệ quy (recursion limit)

Giới hạn đệ quy đặt số [super-step](#graphs) tối đa graph có thể thực thi trong một lần chạy duy nhất. Khi đạt giới hạn, LangGraph sẽ raise `GraphRecursionError`. Bắt đầu từ phiên bản 1.0.6, giới hạn đệ quy mặc định được đặt là 1000 bước. Giới hạn đệ quy có thể được đặt trên bất kỳ graph nào tại runtime, và được truyền tới `invoke`/`stream` qua dictionary config. Quan trọng là, `recursion_limit` là một key `config` độc lập và không nên được truyền bên trong key `configurable` như các cấu hình do người dùng định nghĩa khác. Xem ví dụ dưới đây:

```python
graph.invoke(inputs, config={"recursion_limit": 5}, context={"llm": "anthropic"})
```

Đọc [Giới hạn đệ quy](graph-api.md#recursion-limit) để tìm hiểu thêm về cách giới hạn đệ quy hoạt động.

### Truy cập và xử lý bộ đếm đệ quy

Bộ đếm bước hiện tại có thể truy cập được trong `config["metadata"]["langgraph_step"]` bên trong bất kỳ node nào, cho phép xử lý đệ quy chủ động trước khi đạt giới hạn đệ quy. Điều này cho phép bạn triển khai các chiến lược suy giảm nhẹ nhàng (graceful degradation) bên trong logic graph của bạn.

#### Cách hoạt động

Bộ đếm bước được lưu trong `config["metadata"]["langgraph_step"]`. LangGraph tăng bộ đếm này khi graph thực thi và raise `GraphRecursionError` khi vượt quá `recursion_limit` đã cấu hình.

#### Truy cập bộ đếm bước hiện tại

Bạn có thể truy cập bộ đếm bước hiện tại bên trong bất kỳ node nào để theo dõi tiến trình thực thi.

```python
from langchain_core.runnables import RunnableConfig
from langgraph.graph import StateGraph

def my_node(state: dict, config: RunnableConfig) -> dict:
    current_step = config["metadata"]["langgraph_step"]
    print(f"Currently on step: {current_step}")
    return state
```

#### Xử lý đệ quy chủ động

LangGraph cung cấp một managed value `RemainingSteps` theo dõi số bước còn lại trước khi đạt giới hạn đệ quy. Điều này cho phép suy giảm nhẹ nhàng bên trong graph của bạn.

```python
from typing import Annotated, Literal
from langgraph.graph import StateGraph, START, END
from langgraph.managed import RemainingSteps

class State(TypedDict):
    messages: Annotated[list, lambda x, y: x + y]
    remaining_steps: RemainingSteps  # Managed value - theo dõi số bước tới giới hạn

def reasoning_node(state: State) -> dict:
    # RemainingSteps tự động được điền bởi LangGraph
    remaining = state["remaining_steps"]

    # Kiểm tra xem chúng ta có sắp hết bước không
    if remaining <= 2:
        return {"messages": ["Approaching limit, wrapping up..."]}

    # Xử lý bình thường
    return {"messages": ["thinking..."]}

def route_decision(state: State) -> Literal["reasoning_node", "fallback_node"]:
    """Định tuyến dựa trên số bước còn lại"""
    if state["remaining_steps"] <= 2:
        return "fallback_node"
    return "reasoning_node"

def fallback_node(state: State) -> dict:
    """Xử lý trường hợp sắp đạt giới hạn đệ quy"""
    return {"messages": ["Reached complexity limit, providing best effort answer"]}

# Xây dựng graph
builder = StateGraph(State)
builder.add_node("reasoning_node", reasoning_node)
builder.add_node("fallback_node", fallback_node)
builder.add_edge(START, "reasoning_node")
builder.add_conditional_edges("reasoning_node", route_decision)
builder.add_edge("fallback_node", END)

graph = builder.compile()

# RemainingSteps hoạt động với bất kỳ recursion_limit nào
result = graph.invoke({"messages": []}, {"recursion_limit": 10})
```

#### Cách tiếp cận chủ động và bị động

Có hai cách tiếp cận chính để xử lý giới hạn đệ quy: chủ động (theo dõi bên trong graph) và bị động (bắt lỗi từ bên ngoài).

```python
from typing import Annotated, Literal, TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.managed import RemainingSteps
from langgraph.errors import GraphRecursionError

class State(TypedDict):
    messages: Annotated[list, lambda x, y: x + y]
    remaining_steps: RemainingSteps

# Cách tiếp cận chủ động (được khuyến nghị) - dùng RemainingSteps
def agent_with_monitoring(state: State) -> dict:
    """Chủ động theo dõi và xử lý đệ quy bên trong graph"""
    remaining = state["remaining_steps"]

    # Phát hiện sớm - định tuyến tới xử lý nội bộ
    if remaining <= 2:
        return {
            "messages": ["Approaching limit, returning partial result"]
        }

    # Xử lý bình thường
    return {"messages": [f"Processing... ({remaining} steps remaining)"]}

def route_decision(state: State) -> Literal["agent", END]:
    if state["remaining_steps"] <= 2:
        return END
    return "agent"

# Xây dựng graph
builder = StateGraph(State)
builder.add_node("agent", agent_with_monitoring)
builder.add_edge(START, "agent")
builder.add_conditional_edges("agent", route_decision)
graph = builder.compile()

# Chủ động: Graph hoàn tất một cách nhẹ nhàng
result = graph.invoke({"messages": []}, {"recursion_limit": 10})

# Cách tiếp cận bị động (dự phòng) - bắt lỗi từ bên ngoài
try:
    result = graph.invoke({"messages": []}, {"recursion_limit": 10})
except GraphRecursionError as e:
    # Xử lý từ bên ngoài sau khi thực thi graph thất bại
    result = {"messages": ["Fallback: recursion limit exceeded"]}
```

Sự khác biệt chính giữa các cách tiếp cận này là:

| Cách tiếp cận | Phát hiện | Xử lý | Luồng điều khiển |
| ----------------------------------------- | --------------------- | ------------------------------------- | ------------------------------------ |
| Chủ động (dùng `RemainingSteps`) | Trước khi đạt giới hạn | Bên trong graph qua định tuyến điều kiện | Graph tiếp tục tới node hoàn tất |
| Bị động (bắt `GraphRecursionError`) | Sau khi vượt giới hạn | Bên ngoài graph trong try/catch | Việc thực thi graph bị chấm dứt |

**Ưu điểm của cách tiếp cận chủ động:**

* Suy giảm nhẹ nhàng bên trong graph
* Có thể lưu state trung gian trong checkpoint
* Trải nghiệm người dùng tốt hơn với kết quả một phần
* Graph hoàn tất bình thường (không có exception)

**Ưu điểm của cách tiếp cận bị động:**

* Triển khai đơn giản hơn
* Không cần sửa đổi logic graph
* Xử lý lỗi tập trung

#### Metadata khác có sẵn

Cùng với `langgraph_step`, các metadata sau cũng có sẵn trong `config["metadata"]`:

```python
def inspect_metadata(state: dict, config: RunnableConfig) -> dict:
    metadata = config["metadata"]

    print(f"Step: {metadata['langgraph_step']}")
    print(f"Node: {metadata['langgraph_node']}")
    print(f"Triggers: {metadata['langgraph_triggers']}")
    print(f"Path: {metadata['langgraph_path']}")
    print(f"Checkpoint NS: {metadata['langgraph_checkpoint_ns']}")

    return state
```

## Trực quan hoá

Việc trực quan hoá graph thường rất hữu ích, đặc biệt khi chúng trở nên phức tạp hơn. LangGraph đi kèm nhiều cách dựng sẵn để trực quan hoá graph. Xem [Trực quan hoá graph của bạn](use-graph-api.md#visualize-your-graph) để biết thêm thông tin.

## Observability và Tracing

Để trace, debug và đánh giá agent của bạn, dùng [LangSmith](https://docs.langchain.com/langsmith/observability).

## Tìm hiểu thêm

* [Cách dùng Graph API](use-graph-api.md)
* [Tổng quan khái niệm Functional API](functional-api.md)
* [Chọn giữa Graph API và Functional API](choosing-apis.md)
