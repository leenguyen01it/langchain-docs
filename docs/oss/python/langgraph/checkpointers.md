# Checkpointer

> Checkpointer của LangGraph lưu trạng thái graph thành các checkpoint ở mỗi bước, cho phép persistence, human-in-the-loop, và thực thi chịu lỗi (fault-tolerant).

Một checkpointer lưu một snapshot của trạng thái graph tại mỗi super-step, được tổ chức theo **thread**. Compile một graph với checkpointer để bật workflow human-in-the-loop, debug time travel, thực thi chịu lỗi, và bộ nhớ hội thoại (conversational memory).

![Checkpoints](https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/checkpoints.jpg?fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=966566aaae853ed4d240c2d0d067467c)

!!! info "Agent Server tự xử lý checkpointing"
    Khi dùng [Agent Server](https://docs.langchain.com/langsmith/agent-server), bạn không cần tự cài đặt hay cấu hình checkpointer. Server tự xử lý toàn bộ hạ tầng persistence phía sau hậu trường.

!!! tip
    Trace trạng thái đã checkpoint và debug cách agent resume qua các phiên với [LangSmith](https://smith.langchain.com). Làm theo [tracing quickstart](https://docs.langchain.com/langsmith/trace-with-langgraph) để thiết lập.

## Vì sao nên dùng checkpointer

Checkpointer là bắt buộc cho các tính năng sau:

* **Human-in-the-loop**: Checkpointer hỗ trợ [workflow human-in-the-loop](interrupts.md) bằng cách cho phép con người kiểm tra, ngắt (interrupt), và phê duyệt các bước graph. Checkpointer cần thiết cho các workflow này vì con người phải có thể xem trạng thái graph tại bất kỳ thời điểm nào, và graph phải có thể tiếp tục thực thi sau khi con người đã cập nhật trạng thái. Xem [Interrupts](interrupts.md) để biết ví dụ.
* **Bộ nhớ**: Checkpointer cho phép ["bộ nhớ"](https://docs.langchain.com/oss/python/concepts/memory) giữa các tương tác. Với các tương tác lặp lại của con người (như hội thoại), mọi message tiếp theo có thể được gửi tới thread đó, thread sẽ giữ lại bộ nhớ của các message trước. Xem [Add memory](add-memory.md) để biết cách thêm và quản lý bộ nhớ hội thoại bằng checkpointer.
* **Time travel**: Checkpointer cho phép ["time travel"](use-time-travel.md), cho phép người dùng replay lại các lần thực thi graph trước đó để xem lại và/hoặc debug các bước graph cụ thể. Ngoài ra, checkpointer còn giúp fork trạng thái graph tại các checkpoint tùy ý để khám phá các quỹ đạo (trajectory) thay thế.
* **Chịu lỗi (fault-tolerance)**: Checkpointing cung cấp khả năng chịu lỗi và khôi phục lỗi: nếu một hoặc nhiều node lỗi tại một super-step nhất định, bạn có thể khởi động lại graph từ bước thành công cuối cùng.
* **Pending writes**: Khi một node graph lỗi giữa chừng tại một [super-step](#super-step) nhất định, LangGraph lưu các pending checkpoint writes từ mọi node khác đã hoàn thành thành công tại super-step đó. Khi bạn tiếp tục thực thi graph từ super-step đó, các node đã thành công sẽ không chạy lại.

## Khái niệm cốt lõi

### Thread

Thread là một ID duy nhất (thread identifier) được gán cho mỗi checkpoint mà checkpointer lưu. Nó chứa trạng thái tích lũy của một chuỗi [run](https://docs.langchain.com/langsmith/runs). Khi một run được thực thi, [trạng thái](graph-api.md#trang-thai-state) của graph gốc của assistant sẽ được lưu vào thread.

Khi gọi một graph với checkpointer, bạn **bắt buộc** phải chỉ định `thread_id` trong phần `configurable` của config:

```python
{"configurable": {"thread_id": "1"}}
```

Trạng thái hiện tại và lịch sử của một thread có thể được truy xuất. Để lưu trạng thái, một thread phải được tạo trước khi thực thi run. LangSmith API cung cấp một số endpoint để tạo và quản lý thread cùng trạng thái thread. Xem [API reference](https://reference.langchain.com/python/langsmith/) để biết thêm chi tiết.

Checkpointer dùng `thread_id` làm khóa chính để lưu và truy xuất checkpoint. Không có nó, checkpointer không thể lưu trạng thái hoặc tiếp tục thực thi sau một [interrupt](interrupts.md), vì checkpointer dùng `thread_id` để load trạng thái đã lưu.

### Checkpoint

Trạng thái của một thread tại một thời điểm cụ thể được gọi là checkpoint. Checkpoint là một snapshot của trạng thái graph được lưu tại mỗi [super-step](#super-step) và được biểu diễn bằng đối tượng `StateSnapshot` (xem [các trường StateSnapshot](#cac-truong-statesnapshot) để biết đầy đủ tham chiếu trường).

#### Super-step

LangGraph tạo một checkpoint tại mỗi ranh giới **super-step**. Một super-step là một "nhịp" (tick) của graph, nơi mọi node được lên lịch cho bước đó thực thi (có thể song song). Với một graph tuần tự như `START -> A -> B -> END`, có các super-step riêng biệt cho input, node A, và node B, tạo ra một checkpoint sau mỗi bước. Hiểu ranh giới super-step rất quan trọng cho [time travel](use-time-travel.md), vì bạn chỉ có thể tiếp tục thực thi từ một checkpoint (tức là ranh giới super-step).

Ngoài các checkpoint ở cấp super-step, LangGraph còn lưu các write ở **cấp node (task)**. Khi mỗi node trong một super-step hoàn thành, output của nó được ghi vào bảng `checkpoint_writes` của checkpointer dưới dạng các mục task liên kết với checkpoint đang thực hiện. Các write cấp task này là thứ cho phép khôi phục [pending writes](#pending-writes): nếu một node khác trong cùng super-step lỗi, các write của node thành công đã bền vững (durable) và không cần chạy lại. Toàn bộ state snapshot sau đó được commit khi super-step hoàn thành.

LangGraph cũng lưu các write từ từng lần thực thi node riêng lẻ trong một super-step. Các write này được lưu dưới dạng task và dùng cho khả năng chịu lỗi: nếu một node khác trong cùng super-step lỗi, các write của node thành công không cần tính toán lại khi bạn resume. Các task write này không phải checkpoint `StateSnapshot` đầy đủ, nên time travel tiếp tục từ các checkpoint đầy đủ tại ranh giới super-step.

Checkpoint được lưu bền vững và có thể dùng để khôi phục trạng thái của một thread sau này.

Hãy xem checkpoint nào được lưu khi một graph đơn giản được gọi như sau:

```python
from langgraph.graph import StateGraph, START, END
from langgraph.checkpoint.memory import InMemorySaver
from langchain_core.runnables import RunnableConfig
from typing import Annotated
from typing_extensions import TypedDict
from operator import add

class State(TypedDict):
    foo: str
    bar: Annotated[list[str], add]

def node_a(state: State):
    return {"foo": "a", "bar": ["a"]}

def node_b(state: State):
    return {"foo": "b", "bar": ["b"]}


workflow = StateGraph(State)
workflow.add_node(node_a)
workflow.add_node(node_b)
workflow.add_edge(START, "node_a")
workflow.add_edge("node_a", "node_b")
workflow.add_edge("node_b", END)

checkpointer = InMemorySaver()
graph = workflow.compile(checkpointer=checkpointer)

config: RunnableConfig = {"configurable": {"thread_id": "1"}}
graph.invoke({"foo": "", "bar":[]}, config)
```

Sau khi chạy graph, sẽ có đúng 4 checkpoint:

* Checkpoint rỗng với [`START`](https://reference.langchain.com/python/langgraph/constants/START) là node tiếp theo cần thực thi
* Checkpoint với input người dùng `{'foo': '', 'bar': []}` và `node_a` là node tiếp theo cần thực thi
* Checkpoint với output của `node_a` `{'foo': 'a', 'bar': ['a']}` và `node_b` là node tiếp theo cần thực thi
* Checkpoint với output của `node_b` `{'foo': 'b', 'bar': ['a', 'b']}` và không còn node tiếp theo cần thực thi

Lưu ý rằng các giá trị channel `bar` chứa output từ cả hai node vì ví dụ này có một reducer cho channel `bar`.

#### Checkpoint namespace

Mỗi checkpoint có một trường `checkpoint_ns` (checkpoint namespace) xác định nó thuộc về graph hay subgraph nào:

* **`""`** (chuỗi rỗng): Checkpoint thuộc về graph cha (gốc).
* **`"node_name:uuid"`**: Checkpoint thuộc về một subgraph được gọi dưới dạng node đã cho. Với subgraph lồng nhau, namespace được nối bằng dấu phân cách `|` (ví dụ: `"outer_node:uuid|inner_node:uuid"`).

Bạn có thể truy cập checkpoint namespace từ bên trong một node qua config:

```python
from langchain_core.runnables import RunnableConfig

def my_node(state: State, config: RunnableConfig):
    checkpoint_ns = config["configurable"]["checkpoint_ns"]
    # "" cho graph cha, "node_name:uuid" cho subgraph
```

Xem [Subgraphs](use-subgraphs.md) để biết thêm chi tiết về việc làm việc với trạng thái và checkpoint của subgraph.

## Lấy và cập nhật trạng thái

### Lấy trạng thái

Khi tương tác với trạng thái graph đã lưu, bạn **bắt buộc** phải chỉ định [thread identifier](#thread). Bạn có thể xem trạng thái *mới nhất* của graph bằng cách gọi `graph.get_state(config)`. Nó sẽ trả về một đối tượng `StateSnapshot` tương ứng với checkpoint mới nhất gắn với thread ID được cung cấp trong config, hoặc một checkpoint gắn với checkpoint ID cho thread, nếu được cung cấp.

```python
# lấy state snapshot mới nhất
config = {"configurable": {"thread_id": "1"}}
graph.get_state(config)

# lấy state snapshot cho một checkpoint_id cụ thể
config = {"configurable": {"thread_id": "1", "checkpoint_id": "1ef663ba-28fe-6528-8002-5a559208592c"}}
graph.get_state(config)
```

Trong ví dụ này, output của `get_state` sẽ như sau:

```
StateSnapshot(
    values={'foo': 'b', 'bar': ['a', 'b']},
    next=(),
    config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1ef663ba-28fe-6528-8002-5a559208592c'}},
    metadata={'source': 'loop', 'writes': {'node_b': {'foo': 'b', 'bar': ['b']}}, 'step': 2},
    created_at='2024-08-29T19:19:38.821749+00:00',
    parent_config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1ef663ba-28f9-6ec4-8001-31981c2c39f8'}}, tasks=()
)
```

#### Các trường StateSnapshot

| Trường           | Kiểu                     | Mô tả                                                                                                                                                |
| --------------- | ------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `values`        | `dict`                   | Giá trị channel trạng thái tại checkpoint này.                                                                                                                   |
| `next`          | `tuple[str, ...]`        | Tên node sẽ thực thi tiếp theo. `()` rỗng nghĩa là graph đã hoàn thành.                                                                                                        |
| `config`        | `dict`                   | Chứa `thread_id`, `checkpoint_ns`, và `checkpoint_id`.                                                                                                                |
| `metadata`      | `dict`                   | Metadata thực thi. Chứa `source` (`"input"`, `"loop"`, hoặc `"update"`), `writes` (output của node), và `step` (bộ đếm super-step).                      |
| `created_at`    | `str`                    | Timestamp ISO 8601 khi checkpoint này được tạo.                                                                                                    |
| `parent_config` | `dict \| None`           | Config của checkpoint trước đó. `None` cho checkpoint đầu tiên.                                                                                        |
| `tasks`         | `tuple[PregelTask, ...]` | Các task sẽ thực thi tại bước này. Mỗi task có `id`, `name`, `error`, `interrupts`, và tùy chọn `state` (subgraph snapshot, khi dùng `subgraphs=True`). |

### Lấy lịch sử trạng thái

Bạn có thể lấy toàn bộ lịch sử thực thi graph cho một thread nhất định bằng cách gọi [`graph.get_state_history(config)`](https://reference.langchain.com/python/langgraph/graphs/#langgraph.graph.state.CompiledStateGraph.get_state_history). Nó sẽ trả về một danh sách các đối tượng `StateSnapshot` gắn với thread ID được cung cấp trong config. Quan trọng là, các checkpoint sẽ được sắp theo thứ tự thời gian, với `StateSnapshot`/checkpoint gần nhất đứng đầu danh sách.

```python
config = {"configurable": {"thread_id": "1"}}
list(graph.get_state_history(config))
```

Trong ví dụ này, output của [`get_state_history`](https://reference.langchain.com/python/langgraph/graphs/#langgraph.graph.state.CompiledStateGraph.get_state_history) sẽ như sau:

```
[
    StateSnapshot(
        values={'foo': 'b', 'bar': ['a', 'b']},
        next=(),
        config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1ef663ba-28fe-6528-8002-5a559208592c'}},
        metadata={'source': 'loop', 'writes': {'node_b': {'foo': 'b', 'bar': ['b']}}, 'step': 2},
        created_at='2024-08-29T19:19:38.821749+00:00',
        parent_config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1ef663ba-28f9-6ec4-8001-31981c2c39f8'}},
        tasks=(),
    ),
    StateSnapshot(
        values={'foo': 'a', 'bar': ['a']},
        next=('node_b',),
        config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1ef663ba-28f9-6ec4-8001-31981c2c39f8'}},
        metadata={'source': 'loop', 'writes': {'node_a': {'foo': 'a', 'bar': ['a']}}, 'step': 1},
        created_at='2024-08-29T19:19:38.819946+00:00',
        parent_config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1ef663ba-28f4-6b4a-8000-ca575a13d36a'}},
        tasks=(PregelTask(id='6fb7314f-f114-5413-a1f3-d37dfe98ff44', name='node_b', error=None, interrupts=()),),
    ),
    StateSnapshot(
        values={'foo': '', 'bar': []},
        next=('node_a',),
        config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1ef663ba-28f4-6b4a-8000-ca575a13d36a'}},
        metadata={'source': 'loop', 'writes': None, 'step': 0},
        created_at='2024-08-29T19:19:38.817813+00:00',
        parent_config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1ef663ba-28f0-6c66-bfff-6723431e8481'}},
        tasks=(PregelTask(id='f1b14528-5ee5-579c-949b-23ef9bfbed58', name='node_a', error=None, interrupts=()),),
    ),
    StateSnapshot(
        values={'bar': []},
        next=('__start__',),
        config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1ef663ba-28f0-6c66-bfff-6723431e8481'}},
        metadata={'source': 'input', 'writes': {'foo': ''}, 'step': -1},
        created_at='2024-08-29T19:19:38.816205+00:00',
        parent_config=None,
        tasks=(PregelTask(id='6d27aa2e-d72b-5504-a36f-8620e54a76dd', name='__start__', error=None, interrupts=()),),
    )
]
```

![State](https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/get_state.jpg?fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=38ffff52be4d8806b287836295a3c058)

#### Tìm một checkpoint cụ thể

Bạn có thể lọc lịch sử trạng thái để tìm các checkpoint khớp tiêu chí cụ thể:

```python
history = list(graph.get_state_history(config))

# Tìm checkpoint trước khi node_b thực thi
before_node_b = next(s for s in history if s.next == ("node_b",))

# Tìm một checkpoint theo số bước
step_2 = next(s for s in history if s.metadata["step"] == 2)

# Tìm các checkpoint được tạo bởi update_state
forks = [s for s in history if s.metadata["source"] == "update"]

# Tìm checkpoint nơi xảy ra interrupt
interrupted = next(
    s for s in history
    if s.tasks and any(t.interrupts for t in s.tasks)
)
```

### Replay

Replay thực thi lại các bước từ một checkpoint trước đó. Gọi graph với một `checkpoint_id` trước đó để chạy lại các node sau checkpoint đó. Các node trước checkpoint sẽ được bỏ qua (kết quả của chúng đã được lưu). Các node sau checkpoint sẽ thực thi lại, bao gồm mọi lệnh gọi LLM, API request, hoặc [interrupt](interrupts.md), những thứ luôn được kích hoạt lại khi replay.

Xem [Time travel](use-time-travel.md) để biết đầy đủ chi tiết và ví dụ code về việc replay các lần thực thi trước.

![Replay](https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/re_play.png?fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=d7b34b85c106e55d181ae1f4afb50251)

### Cập nhật trạng thái

Bạn có thể chỉnh sửa trạng thái graph bằng [`update_state`](https://reference.langchain.com/python/langgraph/graphs/#langgraph.graph.state.CompiledStateGraph.update_state). Thao tác này tạo một checkpoint mới với giá trị đã cập nhật, nó không chỉnh sửa checkpoint gốc. Việc cập nhật được xử lý giống như một cập nhật của node: giá trị được truyền qua các hàm [reducer](graph-api.md#reducer) khi đã định nghĩa, nên các channel có reducer sẽ *tích lũy* giá trị thay vì ghi đè.

Bạn có thể tùy chọn chỉ định `as_node` để kiểm soát việc cập nhật được xem như đến từ node nào, điều này ảnh hưởng tới việc node nào thực thi tiếp theo. Xem [Time travel: `as_node`](use-time-travel.md#tu-mot-node-cu-the) để biết chi tiết.

![Update](https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/checkpoints_full_story.jpg?fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=a52016b2c44b57bd395d6e1eac47aa36)

## Chế độ durability

LangGraph hỗ trợ ba chế độ durability cho phép bạn cân bằng giữa hiệu năng và tính nhất quán dữ liệu. Bạn có thể chỉ định chế độ durability khi gọi bất kỳ phương thức thực thi graph nào:

```python
graph.stream(
    {"input": "test"},
    durability="sync"
)
```

Các chế độ durability, từ ít bền vững nhất đến bền vững nhất, gồm:

* `"exit"`: LangGraph chỉ lưu thay đổi khi thực thi graph kết thúc, dù thành công, lỗi, hay do human-in-the-loop interrupt. Cách này cho hiệu năng tốt nhất với graph chạy dài nhưng có nghĩa là trạng thái trung gian không được lưu, nên bạn không thể khôi phục từ lỗi hệ thống (như process crash) giữa chừng thực thi.
* `"async"`: LangGraph lưu thay đổi bất đồng bộ trong khi bước tiếp theo thực thi. Cách này cho hiệu năng và độ bền tốt, nhưng có rủi ro nhỏ là LangGraph không ghi được checkpoint nếu process crash trong lúc thực thi.
* `"sync"`: LangGraph lưu thay đổi đồng bộ trước khi bước tiếp theo bắt đầu. Điều này đảm bảo LangGraph ghi mọi checkpoint trước khi tiếp tục thực thi, mang lại độ bền cao với chi phí hiệu năng nhất định.

## Tối ưu hóa lưu trữ checkpoint

Mặc định, checkpoint của LangGraph ghi toàn bộ giá trị của mọi channel trạng thái tại mỗi super-step. Với thread chạy dài có tích lũy lớn, như hội thoại nhiều lượt, điều này có thể khiến dung lượng lưu trữ tăng đáng kể theo thời gian.

[`DeltaChannel`](https://reference.langchain.com/python/langgraph/channels/delta/DeltaChannel) chỉ lưu các delta gia tăng thay vì toàn bộ giá trị tích lũy, giảm đáng kể kích thước checkpoint cho các channel append nhiều. Xem [DeltaChannel](pregel.md#deltachannel) để biết cách dùng và đánh đổi giữa lưu trữ và độ trễ.

!!! warning
    `DeltaChannel` yêu cầu `langgraph>=1.2` và hiện đang ở giai đoạn beta. API có thể thay đổi trong các bản phát hành tương lai.

## Thư viện checkpointer

Bên dưới, checkpointing được vận hành bởi các đối tượng checkpointer tuân theo interface [`BaseCheckpointSaver`](https://reference.langchain.com/python/langgraph/checkpoints/#langgraph.checkpoint.base.BaseCheckpointSaver). LangGraph cung cấp một số cài đặt checkpointer, tất cả được cài đặt qua các thư viện độc lập, có thể cài riêng.

!!! note
    Xem [checkpointer integrations](https://docs.langchain.com/oss/python/integrations/checkpointers/index) để biết các provider có sẵn.

* `langgraph-checkpoint`: Interface cơ sở cho checkpointer saver ([`BaseCheckpointSaver`](https://reference.langchain.com/python/langgraph/checkpoints/#langgraph.checkpoint.base.BaseCheckpointSaver)) và interface serialization/deserialization ([`SerializerProtocol`](https://reference.langchain.com/python/langgraph/checkpoints/#langgraph.checkpoint.serde.base.SerializerProtocol)). Bao gồm cài đặt checkpointer in-memory ([`InMemorySaver`](https://reference.langchain.com/python/langgraph/checkpoints/#langgraph.checkpoint.memory.InMemorySaver)) để thử nghiệm. LangGraph đi kèm sẵn `langgraph-checkpoint`.
* `langgraph-checkpoint-sqlite`: Cài đặt checkpointer LangGraph dùng database SQLite ([`SqliteSaver`](https://reference.langchain.com/python/langgraph/checkpoints/#langgraph.checkpoint.sqlite.SqliteSaver) / [`AsyncSqliteSaver`](https://reference.langchain.com/python/langgraph/checkpoints/#langgraph.checkpoint.sqlite.aio.AsyncSqliteSaver)). Lý tưởng cho thử nghiệm và workflow cục bộ. Cần cài riêng.
* `langgraph-checkpoint-postgres`: Checkpointer nâng cao dùng database Postgres ([`PostgresSaver`](https://reference.langchain.com/python/langgraph/checkpoints/#langgraph.checkpoint.postgres.PostgresSaver) / [`AsyncPostgresSaver`](https://reference.langchain.com/python/langgraph/checkpoints/#langgraph.checkpoint.postgres.aio.AsyncPostgresSaver)), dùng trong LangSmith. Lý tưởng cho production. Cần cài riêng.
* `langchain-azure-cosmosdb`: Cài đặt checkpointer LangGraph dùng Azure Cosmos DB for NoSQL ([`CosmosDBSaverSync`](https://reference.langchain.com/python/langchain-azure-cosmosdb/) / [`CosmosDBSaver`](https://reference.langchain.com/python/langchain-azure-cosmosdb/)). Lý tưởng cho production với Azure. Hỗ trợ cả thao tác đồng bộ và bất đồng bộ, với xác thực Microsoft Entra ID. Cần cài riêng.

### Interface checkpointer

Mỗi checkpointer tuân theo interface [`BaseCheckpointSaver`](https://reference.langchain.com/python/langgraph/checkpoints/#langgraph.checkpoint.base.BaseCheckpointSaver) và cài đặt các phương thức sau:

* `.put`: Lưu một checkpoint cùng config và metadata.
* `.put_writes`: Lưu các write trung gian gắn với một checkpoint (tức [pending writes](#pending-writes)).
* `.get_tuple`: Fetch một checkpoint tuple cho một config nhất định (`thread_id` và `checkpoint_id`). Dùng để điền `StateSnapshot` trong `graph.get_state()`.
* `.list`: Liệt kê các checkpoint khớp một config và tiêu chí lọc nhất định. Dùng để điền lịch sử trạng thái trong `graph.get_state_history()`

Nếu checkpointer được dùng với thực thi graph bất đồng bộ (tức thực thi graph qua `.ainvoke`, `.astream`, `.abatch`), các phiên bản bất đồng bộ của các phương thức trên sẽ được dùng (`.aput`, `.aput_writes`, `.aget_tuple`, `.alist`).

!!! note
    Để chạy graph bất đồng bộ, bạn có thể dùng [`InMemorySaver`](https://reference.langchain.com/python/langgraph/checkpoints/#langgraph.checkpoint.memory.InMemorySaver), hoặc phiên bản bất đồng bộ của checkpointer Sqlite/Postgres, [`AsyncSqliteSaver`](https://reference.langchain.com/python/langgraph/checkpoints/#langgraph.checkpoint.sqlite.aio.AsyncSqliteSaver) / [`AsyncPostgresSaver`](https://reference.langchain.com/python/langgraph/checkpoints/#langgraph.checkpoint.postgres.aio.AsyncPostgresSaver).

### Serializer

Khi checkpointer lưu trạng thái graph, chúng cần serialize các giá trị channel trong trạng thái. Việc này được thực hiện bằng các đối tượng serializer.

`langgraph_checkpoint` định nghĩa [protocol](https://reference.langchain.com/python/langgraph/checkpoints/#langgraph.checkpoint.serde.base.SerializerProtocol) để cài đặt serializer và cung cấp một cài đặt mặc định ([`JsonPlusSerializer`](https://reference.langchain.com/python/langgraph/checkpoints/#langgraph.checkpoint.serde.jsonplus.JsonPlusSerializer)) xử lý nhiều loại type, gồm các primitive của LangChain và LangGraph, datetime, enum, và nhiều hơn nữa.

#### Serialize với `pickle`

Serializer mặc định, [`JsonPlusSerializer`](https://reference.langchain.com/python/langgraph/checkpoints/#langgraph.checkpoint.serde.jsonplus.JsonPlusSerializer), dùng ormsgpack và JSON bên dưới, không phù hợp với mọi loại đối tượng.

Nếu muốn fallback về pickle cho các đối tượng chưa được msgpack encoder hỗ trợ (như Pandas dataframe), bạn có thể dùng tham số `pickle_fallback` của [`JsonPlusSerializer`](https://reference.langchain.com/python/langgraph/checkpoints/#langgraph.checkpoint.serde.jsonplus.JsonPlusSerializer):

```python
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.checkpoint.serde.jsonplus import JsonPlusSerializer

# ... Định nghĩa graph ...
graph.compile(
    checkpointer=InMemorySaver(serde=JsonPlusSerializer(pickle_fallback=True))
)
```

#### Mã hóa (encryption)

Checkpointer có thể tùy chọn mã hóa toàn bộ trạng thái đã lưu. Để bật, truyền một instance của [`EncryptedSerializer`](https://reference.langchain.com/python/langgraph/checkpoints/#langgraph.checkpoint.serde.encrypted.EncryptedSerializer) cho tham số `serde` của bất kỳ cài đặt [`BaseCheckpointSaver`](https://reference.langchain.com/python/langgraph/checkpoints/#langgraph.checkpoint.base.BaseCheckpointSaver) nào. Cách dễ nhất để tạo một serializer đã mã hóa là qua [`from_pycryptodome_aes`](https://reference.langchain.com/python/langgraph/checkpoints/#langgraph.checkpoint.serde.encrypted.EncryptedSerializer.from_pycryptodome_aes), thứ đọc AES key từ biến môi trường `LANGGRAPH_AES_KEY` (hoặc nhận tham số `key`):

```python
import sqlite3

from langgraph.checkpoint.serde.encrypted import EncryptedSerializer
from langgraph.checkpoint.sqlite import SqliteSaver

serde = EncryptedSerializer.from_pycryptodome_aes()  # đọc LANGGRAPH_AES_KEY
checkpointer = SqliteSaver(sqlite3.connect("checkpoint.db"), serde=serde)
```

```python
from langgraph.checkpoint.serde.encrypted import EncryptedSerializer
from langgraph.checkpoint.postgres import PostgresSaver

serde = EncryptedSerializer.from_pycryptodome_aes()
checkpointer = PostgresSaver.from_conn_string("postgresql://...", serde=serde)
checkpointer.setup()
```

Khi chạy trên LangSmith, mã hóa được tự động bật bất cứ khi nào `LANGGRAPH_AES_KEY` có mặt, nên bạn chỉ cần cung cấp biến môi trường. Các cơ chế mã hóa khác có thể dùng bằng cách cài đặt [`CipherProtocol`](https://reference.langchain.com/python/langgraph/checkpoints/#langgraph.checkpoint.serde.base.CipherProtocol) và truyền vào [`EncryptedSerializer`](https://reference.langchain.com/python/langgraph/checkpoints/#langgraph.checkpoint.serde.encrypted.EncryptedSerializer).

## Xây dựng checkpointer tùy chỉnh

!!! tip
    Kiểm chứng cài đặt của bạn khi xây dựng bằng [bộ test conformance](#kiem-thu-voi-bo-conformance-suite). Nó bao trùm cả năm phương thức cơ sở và các khả năng mở rộng gồm delta channel. Chạy nó trong CI trước khi ship.

Phần này bao trùm việc cài đặt `BaseCheckpointSaver` từ đầu cho một storage backend tùy chỉnh. Nếu bạn đã có một checkpointer hoạt động và chỉ cần thêm hỗ trợ delta channel, chuyển tới [Hỗ trợ delta channel](#ho-tro-delta-channel).

### Tổng quan

Tầng persistence của LangGraph được xây dựng trên hai abstraction lưu trữ:

* **Bảng checkpoints**: một dòng cho mỗi superstep; lưu trạng thái graph đã serialize (`channel_values`, `channel_versions`, `versions_seen`) và liên kết tới checkpoint cha của nó.
* **Bảng writes**: một dòng cho mỗi output node trong một superstep; lưu tuple `(task_id, channel, value)` liên kết với một checkpoint.

Checkpointer của bạn quản lý cả hai bảng. `put` ghi một dòng checkpoint; `put_writes` ghi các dòng output node; `get_tuple` đọc cả hai lại thành một `CheckpointTuple`.

### Hợp đồng cơ sở (base contract)

Kế thừa `BaseCheckpointSaver` và cài đặt năm phương thức sau. Tất cả đều bắt buộc, thiếu một phương thức cơ sở sẽ raise `NotImplementedError` khi runtime.

```python
from collections.abc import AsyncIterator, Iterator, Sequence
from typing import Any
from langchain_core.runnables import RunnableConfig
from langgraph.checkpoint.base import (
    BaseCheckpointSaver,
    ChannelVersions,
    Checkpoint,
    CheckpointMetadata,
    CheckpointTuple,
)

class MyCheckpointer(BaseCheckpointSaver):
    async def aput(
        self,
        config: RunnableConfig,
        checkpoint: Checkpoint,
        metadata: CheckpointMetadata,
        new_versions: ChannelVersions,
    ) -> RunnableConfig:
        ...

    async def aput_writes(
        self,
        config: RunnableConfig,
        writes: Sequence[tuple[str, Any]],
        task_id: str,
        task_path: str = "",
    ) -> None:
        ...

    async def aget_tuple(self, config: RunnableConfig) -> CheckpointTuple | None:
        ...

    async def alist(
        self,
        config: RunnableConfig | None,
        *,
        filter: dict[str, Any] | None = None,
        before: RunnableConfig | None = None,
        limit: int | None = None,
    ) -> AsyncIterator[CheckpointTuple]:
        ...
        yield  # để hàm này thành async generator

    async def adelete_thread(self, thread_id: str) -> None:
        ...
```

#### put / aput

Lưu một dòng checkpoint. Trả về config đã cập nhật với `checkpoint_id` đã lưu.

Các yêu cầu quan trọng:

* Serialize checkpoint bằng `self.serde.dumps_typed(checkpoint)`, thao tác này xử lý mọi type gốc của LangGraph gồm cả blob `_DeltaSnapshot` được delta channel dùng.
* Lưu `metadata` đầy đủ, không loại bỏ các khóa lạ. LangGraph thêm các trường metadata mới (như `counters_since_delta_snapshot` cho delta channel) trong các bản phát hành nhỏ; loại bỏ âm thầm sẽ làm hỏng tính năng.
* Lưu `config["configurable"].get("checkpoint_id")` làm checkpoint ID cha để `get_tuple` có thể điền `parent_config`.

```python
async def aput(self, config, checkpoint, metadata, new_versions):
    thread_id = config["configurable"]["thread_id"]
    checkpoint_ns = config["configurable"]["checkpoint_ns"]
    checkpoint_id = checkpoint["id"]
    parent_id = config["configurable"].get("checkpoint_id")

    type_, blob = self.serde.dumps_typed(checkpoint)
    serialized_metadata = self.serde.dumps_typed(metadata)

    await self.db.execute(
        "INSERT INTO checkpoints (...) VALUES (...)",
        thread_id, checkpoint_ns, checkpoint_id, parent_id,
        type_, blob, *serialized_metadata,
    )
    return {
        "configurable": {
            "thread_id": thread_id,
            "checkpoint_ns": checkpoint_ns,
            "checkpoint_id": checkpoint_id,
        }
    }
```

#### put_writes / aput_writes

Lưu các dòng output node cho một task duy nhất trong superstep hiện tại. Các dòng này liên kết với checkpoint qua `(thread_id, checkpoint_ns, checkpoint_id)`.

```python
async def aput_writes(self, config, writes, task_id, task_path=""):
    thread_id = config["configurable"]["thread_id"]
    checkpoint_ns = config["configurable"]["checkpoint_ns"]
    checkpoint_id = config["configurable"]["checkpoint_id"]

    rows = []
    for idx, (channel, value) in enumerate(writes):
        type_, blob = self.serde.dumps_typed(value)
        final_idx = WRITES_IDX_MAP.get(channel, idx)
        rows.append((thread_id, checkpoint_ns, checkpoint_id,
                      task_id, task_path, final_idx, channel, type_, blob))

    await self.db.executemany("INSERT INTO writes (...) VALUES (...)", rows)
```

Import `WRITES_IDX_MAP` từ `langgraph.checkpoint.base`. Nó map các channel đặc biệt (`__error__`, `__interrupt__`, v.v.) tới các chỉ số âm được dành riêng để chúng không xung đột với chỉ số write thông thường.

#### get_tuple / aget_tuple

Truy xuất một checkpoint. Config có thể chứa:

* **Không có `checkpoint_id`**: trả về checkpoint mới nhất cho thread + namespace.
* **Có `checkpoint_id` cụ thể**: trả về đúng checkpoint đó.

**Cả hai đường phải hoạt động đúng.** Đường có id cụ thể được dùng cho time travel và, quan trọng hơn, cho việc tái tạo trạng thái delta channel ở mỗi lần gọi graph (xem [Hỗ trợ delta channel](#ho-tro-delta-channel)). Lookup id cụ thể bị lỗi sẽ âm thầm làm hỏng trạng thái delta channel.

```python
async def aget_tuple(self, config):
    thread_id = config["configurable"]["thread_id"]
    checkpoint_ns = config["configurable"].get("checkpoint_ns", "")
    checkpoint_id = config["configurable"].get("checkpoint_id")

    if checkpoint_id:
        row = await self.db.fetchone(
            "SELECT * FROM checkpoints "
            "WHERE thread_id=? AND checkpoint_ns=? AND checkpoint_id=?",
            thread_id, checkpoint_ns, checkpoint_id,
        )
    else:
        row = await self.db.fetchone(
            "SELECT * FROM checkpoints "
            "WHERE thread_id=? AND checkpoint_ns=? "
            "ORDER BY checkpoint_id DESC LIMIT 1",
            thread_id, checkpoint_ns,
        )

    if row is None:
        return None

    writes = await self.db.fetchall(
        "SELECT task_id, channel, type, value FROM writes "
        "WHERE thread_id=? AND checkpoint_ns=? AND checkpoint_id=? "
        "ORDER BY task_id, idx",
        thread_id, checkpoint_ns, row["checkpoint_id"],
    )
    pending_writes = [
        (w["task_id"], w["channel"], self.serde.loads_typed((w["type"], w["value"])))
        for w in writes
    ]

    checkpoint = self.serde.loads_typed((row["type"], row["blob"]))
    metadata = self.serde.loads_typed((row["metadata_type"], row["metadata"]))

    parent_config = None
    if row["parent_checkpoint_id"]:
        parent_config = {
            "configurable": {
                "thread_id": thread_id,
                "checkpoint_ns": checkpoint_ns,
                "checkpoint_id": row["parent_checkpoint_id"],
            }
        }

    return CheckpointTuple(
        config={
            "configurable": {
                "thread_id": thread_id,
                "checkpoint_ns": checkpoint_ns,
                "checkpoint_id": row["checkpoint_id"],
            }
        },
        checkpoint=checkpoint,
        metadata=metadata,
        parent_config=parent_config,
        pending_writes=pending_writes,
    )
```

!!! warning
    **Thiết kế khóa/index dòng ảnh hưởng tới lookup theo id cụ thể.** Nếu storage của bạn dùng khóa sắp theo thời gian (ví dụ timestamp đảo ngược) không nhúng `checkpoint_id`, bạn không thể đọc trực tiếp một dòng theo id. Bạn phải hoặc mã hóa `checkpoint_id` trong khóa dòng, hoặc xây một index phụ. Quét kèm bộ lọc giá trị ở mỗi lookup hoạt động được nhưng không scale.

#### list / alist

Trả về các checkpoint cho một thread, mới nhất trước. Tôn trọng `before` (chỉ trả về checkpoint cũ hơn `checkpoint_id` trong config đó) và `limit`.

#### delete_thread / adelete_thread

Xóa mọi checkpoint và write cho một thread. Cả dòng checkpoint và dòng write đều phải bị xóa.

### Thiết kế khóa/index dòng

Cách bạn lưu và index checkpoint ảnh hưởng trực tiếp tới tính đúng đắn và hiệu năng.

**Schema đề xuất (SQL):**

```sql
CREATE TABLE checkpoints (
    thread_id          TEXT NOT NULL,
    checkpoint_ns      TEXT NOT NULL DEFAULT '',
    checkpoint_id      TEXT NOT NULL,   -- ULID, sắp được theo thứ tự từ điển, mới nhất ở cuối
    parent_checkpoint_id TEXT,
    type               TEXT,
    checkpoint         BYTEA,
    metadata           JSONB,
    PRIMARY KEY (thread_id, checkpoint_ns, checkpoint_id)
);

CREATE TABLE writes (
    thread_id     TEXT NOT NULL,
    checkpoint_ns TEXT NOT NULL DEFAULT '',
    checkpoint_id TEXT NOT NULL,
    task_id       TEXT NOT NULL,
    task_path     TEXT NOT NULL DEFAULT '',
    idx           INTEGER NOT NULL,
    channel       TEXT NOT NULL,
    type          TEXT,
    value         BYTEA,
    PRIMARY KEY (thread_id, checkpoint_ns, checkpoint_id, task_id, task_path, idx)
);
```

Vì `checkpoint_id` là một ULID, nó sắp theo thứ tự từ điển, giá trị lớn hơn là mới hơn. "Lấy mới nhất" là `ORDER BY checkpoint_id DESC LIMIT 1`; "lấy theo id" là một lookup bằng nhau trên khóa chính.

**Với các store không phải SQL:** nguyên tắc tương tự áp dụng. Dù dùng scheme khóa nào, lookup trực tiếp theo `(thread_id, checkpoint_ns, checkpoint_id)` phải là O(1) hoặc gần O(1). Tránh thiết kế nơi cách duy nhất để tìm một checkpoint theo id là quét mọi dòng của một thread.

### Serialization

Luôn dùng `self.serde` (kế thừa từ `BaseCheckpointSaver`, mặc định là `JsonPlusSerializer`) cho checkpoint, write, và metadata. Không dùng `pickle` trực tiếp cho metadata, nó vẫn hoạt động, nhưng `JsonPlusSerializer` tạo output dễ đọc hơn và xử lý versioning tốt hơn.

`JsonPlusSerializer` tự động xử lý mọi type gốc của LangGraph:

* `_DeltaSnapshot`: blob sentinel dùng bởi delta channel (msgpack ext code 7)
* Model Pydantic v2, dataclass, mảng numpy, datetime, enum, và nhiều hơn nữa

Nếu bạn viết một serializer tùy chỉnh, hãy đảm bảo nó có thể round-trip `_DeltaSnapshot` từ `langgraph.checkpoint.serde.types`.

### Các khả năng mở rộng

Các phương thức sau là tùy chọn nhưng mở khóa thêm tính năng cho Agent Server. Cài đặt chúng nếu storage backend của bạn có thể hỗ trợ hiệu quả.

| Phương thức                       | Chức năng                                          |
| ---------------------------- | -------------------------------------------------------- |
| `adelete_for_runs`           | Rollback chiến lược multitask                              |
| `acopy_thread`               | Fork thread hiệu quả                                 |
| `aprune`                     | Prune lịch sử thread                                 |
| `aget_delta_channel_history` | Tái tạo trạng thái delta channel hiệu quả (xem bên dưới) |

Agent Server tự động phát hiện checkpointer của bạn cài đặt khả năng nào khi khởi động và kích hoạt các tính năng tương ứng.

### Hỗ trợ delta channel

!!! info "DeltaChannel đang ở giai đoạn beta."
    API và biểu diễn trên đĩa có thể thay đổi trong lúc thiết kế ổn định dần.

`DeltaChannel` là một reducer channel chỉ lưu một sentinel (`MISSING`) trong blob checkpoint thay vì toàn bộ giá trị channel. Trạng thái được tái tạo bằng cách replay các write tổ tiên qua reducer. Điều này khiến blob checkpoint có độ phức tạp O(1) mỗi bước thay vì O(N) cho các channel như `messages` tích lũy theo thời gian.

#### Runtime cần gì

Khi load một checkpoint mà các delta channel của nó vắng mặt trong `channel_values`, LangGraph gọi `saver.get_delta_channel_history(config=config, channels=[...])`. Nó trả về, cho mỗi channel:

* **`writes`**: mọi write tới channel đó trong chuỗi tổ tiên, cũ nhất trước, tới snapshot gần nhất.
* **`seed`** (tùy chọn): blob `_DeltaSnapshot` đã lưu tại tổ tiên gần nhất có một cái, vắng mặt nếu quá trình đi ngược tới gốc mà không tìm thấy snapshot.

Runtime sau đó gọi `channel.from_checkpoint(seed)` và `channel.replay_writes(writes)` để tái tạo giá trị hiện hành.

#### Cài đặt mặc định

`BaseCheckpointSaver` cung cấp một `get_delta_channel_history` mặc định hoạt động với bất kỳ cài đặt `get_tuple` đúng nào:

```python
# Đơn giản hóa từ BaseCheckpointSaver
def get_delta_channel_history(self, *, config, channels):
    target = self.get_tuple(config)          # load checkpoint đầu
    cursor = target.parent_config            # đi ngược từ checkpoint cha
    collected = {ch: [] for ch in channels}
    seed = {}
    remaining = set(channels)

    while cursor and remaining:
        tup = self.get_tuple(cursor)         # ← cần lookup theo id đúng
        if tup is None:
            break
        for write in reversed(tup.pending_writes or []):
            if write[1] in remaining:
                collected[write[1]].append(write)
        for ch in list(remaining):
            if ch in tup.checkpoint["channel_values"]:
                seed[ch] = tup.checkpoint["channel_values"][ch]
                remaining.discard(ch)
        cursor = tup.parent_config

    return {
        ch: {"writes": list(reversed(collected[ch])), **({"seed": seed[ch]} if ch in seed else {})}
        for ch in channels
    }
```

**Sự phụ thuộc quan trọng:** `get_tuple(cursor)` luôn được gọi với một `checkpoint_id` cụ thể (id của checkpoint cha). Nếu lookup đó trả về `None`, quá trình đi ngược dừng ngay lập tức và mọi delta channel sẽ tái tạo thành rỗng, âm thầm, không có lỗi. Đây là lý do đường lookup theo id cụ thể trong `get_tuple` phải đúng.

#### Tối ưu hiệu năng

Cách đi ngược mặc định gọi một lần `get_tuple` cho mỗi checkpoint tổ tiên. Với backend hỗ trợ truy vấn tốt, override `get_delta_channel_history` (và bản bất đồng bộ tương ứng) để lấy chuỗi tổ tiên và write trong hai truy vấn:

```python
async def aget_delta_channel_history(self, *, config, channels):
    if not channels:
        return {}

    thread_id = config["configurable"]["thread_id"]
    checkpoint_ns = config["configurable"].get("checkpoint_ns", "")
    checkpoint_id = config["configurable"]["checkpoint_id"]

    # Giai đoạn 1: stream tổ tiên mới nhất trước cho đến khi mọi channel có seed
    ancestors = await self.db.fetchall(
        "SELECT checkpoint_id, parent_checkpoint_id, type, checkpoint "
        "FROM checkpoints "
        "WHERE thread_id=? AND checkpoint_ns=? AND checkpoint_id < ? "
        "ORDER BY checkpoint_id DESC",
        thread_id, checkpoint_ns, checkpoint_id,
    )

    chain_by_ch: dict[str, list[str]] = {ch: [] for ch in channels}
    seed_by_ch: dict[str, Any] = {}
    remaining = set(channels)
    cur_id = config["configurable"]["checkpoint_id"]

    for row in ancestors:
        if not remaining:
            break
        parent_id = row["parent_checkpoint_id"]
        ckpt = self.serde.loads_typed((row["type"], row["checkpoint"]))
        cv = ckpt.get("channel_values") or {}
        for ch in list(remaining):
            chain_by_ch[ch].append(row["checkpoint_id"])
            if ch in cv:
                seed_by_ch[ch] = cv[ch]
                remaining.discard(ch)
        cur_id = parent_id

    # Giai đoạn 2: fetch write cho chuỗi tổ tiên của mỗi channel trong một truy vấn
    result: dict[str, DeltaChannelHistory] = {}
    for ch in channels:
        chain = chain_by_ch[ch]
        if not chain:
            entry: DeltaChannelHistory = {"writes": []}
            if ch in seed_by_ch:
                entry["seed"] = seed_by_ch[ch]
            result[ch] = entry
            continue

        write_rows = await self.db.fetchall(
            f"SELECT checkpoint_id, task_id, idx, type, value FROM writes "
            f"WHERE thread_id=? AND checkpoint_ns=? AND channel=? "
            f"AND checkpoint_id IN ({','.join('?' * len(chain))})"
            f"ORDER BY checkpoint_id, task_id, idx",
            thread_id, checkpoint_ns, ch, *chain,
        )
        writes_by_cid: dict[str, list[PendingWrite]] = {}
        for row in write_rows:
            cid = row["checkpoint_id"]
            value = self.serde.loads_typed((row["type"], row["value"]))
            writes_by_cid.setdefault(cid, []).append((row["task_id"], ch, value))

        # chain sắp mới nhất trước; lặp cũ nhất trước để có thứ tự replay đúng
        collected: list[PendingWrite] = []
        for cid in reversed(chain):
            collected.extend(writes_by_cid.get(cid, []))

        entry = {"writes": collected}
        if ch in seed_by_ch:
            entry["seed"] = seed_by_ch[ch]
        result[ch] = entry

    return result
```

#### Prune với delta channel

Trạng thái `DeltaChannel` không tự chứa trong một checkpoint đơn lẻ, nó phụ thuộc vào chuỗi write tổ tiên ngược tới `_DeltaSnapshot` gần nhất. Nếu bạn cài đặt `prune` hay `delete_for_runs`, bạn không được xóa các dòng write mà delta channel của một checkpoint còn sống đang phụ thuộc vào.

Các phương án an toàn:

1. **Đi ngược trước khi prune**: với mỗi checkpoint bạn dự định giữ, đi ngược chuỗi tổ tiên của nó và đánh dấu mọi dòng write tới `_DeltaSnapshot` gần nhất là không thể xóa.
2. **Buộc tạo snapshot trước khi prune**: viết lại `channel_values[ch] = _DeltaSnapshot(reconstructed_value)` trên checkpoint bạn đang giữ, sau đó tự do xóa tổ tiên.
3. **Bỏ qua prune cho thread có delta channel**: phương án an toàn ngắn hạn nhất nếu bạn chưa cần prune.

#### Copy thread với delta channel

Khi cài đặt `copy_thread`, hãy copy toàn bộ chuỗi tổ tiên, không chỉ checkpoint đầu. Thread đích phải có các dòng write ngược tới ít nhất một `_DeltaSnapshot` cho mỗi delta channel, nếu không các channel đó sẽ tái tạo thành rỗng sau khi copy.

### Kiểm thử với bộ conformance suite

`langgraph-checkpoint-conformance` kiểm chứng cài đặt của bạn theo đúng toàn bộ hợp đồng, gồm cả lịch sử delta channel:

```python
pip install langgraph-checkpoint-conformance
```

```python
import asyncio
from langgraph.checkpoint.conformance import checkpointer_test, validate

@checkpointer_test(name="MyCheckpointer")
async def my_checkpointer():
    async with MyCheckpointer.create() as saver:
        yield saver

async def main():
    report = await validate(my_checkpointer)
    report.print_report()
    # Làm process fail nếu thiếu hoặc hỏng bất kỳ khả năng cơ sở nào
    if not report.passed_all_base():
        raise RuntimeError("Checkpointer failed conformance suite")

asyncio.run(main())
```

Bộ test này tự động phát hiện cài đặt của bạn hỗ trợ khả năng mở rộng nào (gồm cả `aget_delta_channel_history`) và chạy các test liên quan cho từng khả năng. Chạy nó như một phần của CI trước khi ship.

---

!!! info "Kết nối tài liệu này"
    [Kết nối tài liệu này](https://docs.langchain.com/use-these-docs) với Claude, VSCode, và nhiều công cụ khác qua MCP để có câu trả lời theo thời gian thực.

!!! info "Đóng góp"
    [Chỉnh sửa trang này trên GitHub](https://github.com/langchain-ai/docs/edit/main/src/oss/langgraph/checkpointers.mdx) hoặc [báo lỗi](https://github.com/langchain-ai/docs/issues/new/choose).
