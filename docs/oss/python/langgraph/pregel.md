# LangGraph runtime

[`Pregel`](https://reference.langchain.com/python/langgraph/pregel/main/Pregel) triển khai (implement) runtime của LangGraph, quản lý việc thực thi các ứng dụng LangGraph.

Compile một [StateGraph](https://reference.langchain.com/python/langgraph/graph/state/StateGraph) hoặc tạo một [`@entrypoint`](https://reference.langchain.com/python/langgraph/func/entrypoint) sẽ tạo ra một instance [`Pregel`](https://reference.langchain.com/python/langgraph/pregel/main/Pregel) có thể được invoke với input.

Hướng dẫn này giải thích runtime ở mức tổng quan và cung cấp hướng dẫn để triển khai ứng dụng trực tiếp với Pregel.

> **Lưu ý:** Runtime [`Pregel`](https://reference.langchain.com/python/langgraph/pregel/main/Pregel) được đặt tên theo [thuật toán Pregel của Google](https://research.google/pubs/pub37252/), mô tả một phương pháp hiệu quả cho tính toán song song quy mô lớn dùng graph.

## Tổng quan

Trong LangGraph, Pregel kết hợp [**actor**](https://en.wikipedia.org/wiki/Actor_model) và **channel** vào một ứng dụng duy nhất. **Actor** đọc dữ liệu từ channel và ghi dữ liệu vào channel. Pregel tổ chức việc thực thi ứng dụng thành nhiều bước (step), theo mô hình **Pregel Algorithm**/**Bulk Synchronous Parallel**.

Mỗi step gồm ba giai đoạn:

* **Plan**: Xác định **actor** nào sẽ thực thi trong step này. Ví dụ, ở step đầu tiên, chọn các **actor** đăng ký (subscribe) các channel **input** đặc biệt; ở các step sau, chọn các **actor** đăng ký các channel đã được cập nhật ở step trước.
* **Execution**: Thực thi tất cả **actor** đã chọn song song, cho tới khi tất cả hoàn tất, hoặc một cái thất bại, hoặc hết timeout. Trong giai đoạn này, các cập nhật channel không hiển thị với actor cho tới step tiếp theo.
* **Update**: Cập nhật các channel với giá trị mà **actor** đã ghi trong step này.

Lặp lại cho tới khi không còn **actor** nào được chọn để thực thi, hoặc đạt số step tối đa.

## Actor

Một **actor** là một `PregelNode`. Nó đăng ký các channel, đọc dữ liệu từ chúng, và ghi dữ liệu vào chúng. Có thể hình dung nó như một **actor** trong thuật toán Pregel. `PregelNode` triển khai Runnable interface của LangChain.

## Channel

Channel được dùng để giao tiếp giữa các actor (PregelNode). Mỗi channel có một kiểu giá trị (value type), một kiểu cập nhật (update type), và một hàm cập nhật, hàm này nhận một chuỗi các cập nhật và sửa đổi giá trị đã lưu. Channel có thể dùng để gửi dữ liệu từ chain này sang chain khác, hoặc để gửi dữ liệu từ một chain tới chính nó ở một step trong tương lai.

### LastValue

[`LastValue`](https://reference.langchain.com/python/langgraph/channels/last_value/LastValue) là kiểu channel mặc định. Nó lưu giá trị cuối cùng được ghi vào nó, ghi đè lên bất kỳ giá trị trước đó nào. Dùng nó cho các giá trị input và output, hoặc để chuyển dữ liệu từ step này sang step tiếp theo.

```python
from langgraph.channels import LastValue

channel: LastValue[int] = LastValue(int)
```

### Topic

[`Topic`](https://reference.langchain.com/python/langgraph/channels/topic/Topic) là một channel PubSub có thể cấu hình, hữu ích để gửi nhiều giá trị giữa các actor hoặc tích luỹ output qua các step. Nó có thể được cấu hình để loại bỏ trùng lặp giá trị hoặc tích luỹ tất cả giá trị được ghi trong một run.

```python
from langgraph.channels import Topic

# Tích luỹ tất cả giá trị được ghi qua các step
channel: Topic[str] = Topic(str, accumulate=True)
```

### BinaryOperatorAggregate

[`BinaryOperatorAggregate`](https://reference.langchain.com/python/langgraph/channels/binop/BinaryOperatorAggregate) lưu một giá trị bền vững (persistent) được cập nhật bằng cách áp dụng một toán tử nhị phân (binary operator) lên giá trị hiện tại và mỗi cập nhật mới. Dùng nó để tính các phép tổng hợp (aggregate) chạy liên tục qua các step.

```python
import operator
from langgraph.channels import BinaryOperatorAggregate

# Tổng chạy: mỗi lần ghi sẽ cộng vào giá trị hiện tại
total = BinaryOperatorAggregate(int, operator.add)
```

### DeltaChannel

!!! warning ""
    `DeltaChannel` yêu cầu `langgraph>=1.2` và hiện đang trong giai đoạn **beta**. API có thể thay đổi trong các bản phát hành tương lai.

[`DeltaChannel`](https://reference.langchain.com/python/langgraph/channels/delta/DeltaChannel) chỉ lưu phần delta tăng dần (incremental) ở mỗi step thay vì toàn bộ giá trị tích luỹ. Điều này hữu ích nhất cho các channel được ghi thường xuyên và tích luỹ giá trị lớn theo thời gian, ví dụ một danh sách tin nhắn hội thoại trong một thread chạy lâu dài. Không có lưu trữ delta, toàn bộ danh sách sẽ được re-serialize vào mỗi checkpoint; với `DeltaChannel`, chỉ các tin nhắn mới được ghi ở mỗi step mới được lưu.

!!! tip ""
    Cân nhắc dùng `DeltaChannel` khi một channel vừa được ghi thường xuyên vừa phình to theo thời gian. Một tín hiệu tốt: nếu bạn thấy kích thước checkpoint tăng tuyến tính theo độ dài thread đối với một channel cụ thể, `DeltaChannel` có khả năng là lựa chọn phù hợp.

Dùng `DeltaChannel` trong một type annotation `Annotated` giống như cách bạn dùng một reducer thông thường:

```python
from typing import Annotated, Sequence
from typing_extensions import TypedDict
from langgraph.channels import DeltaChannel


def my_reducer(state: list[str], writes: Sequence[list[str]]) -> list[str]:
    result = list(state)
    for write in writes:
        result.extend(write)
    return result


class State(TypedDict):
    messages: Annotated[list[str], DeltaChannel(my_reducer)]
```

#### Yêu cầu về bulk reducer

`reducer` truyền vào `DeltaChannel` là một **bulk reducer**: nó nhận state hiện tại và một *chuỗi* tất cả các lượt ghi từ step hiện tại trong một lần gọi duy nhất, không phải theo từng cặp (pairwise) như một reducer chuẩn. Điều này khác với các reducer theo từng key được dùng với `Annotated` trong một `StateGraph`, nơi reducer được gọi một lần cho mỗi cập nhật.

!!! warning ""
    Bulk reducer **phải có tính kết hợp (associative)** (bất biến với cách gộp lô, batching-invariant):

    ```
    reducer(reducer(state, [xs]), [ys]) == reducer(state, [xs, ys])
    ```

    Nếu reducer của bạn không có tính kết hợp, state được tái tạo lại có thể khác nhau tuỳ vào cách LangGraph gộp lô các lượt ghi qua các step, gây ra hành vi không nhất quán.

!!! warning "Reducer chạy khi tái tạo, không phải khi ghi"
    Khác với [`BinaryOperatorAggregate`](https://reference.langchain.com/python/langgraph/channels/binop/BinaryOperatorAggregate), có reducer được gọi tại thời điểm ghi nên giá trị kết hợp chính là thứ được serialize vào checkpoint, một reducer của `DeltaChannel` được gọi khi giá trị channel được *xây dựng lại* (rebuilt) từ các lượt ghi đã persist. Các lượt ghi thô theo từng step mới là thứ được serialize; reducer chỉ được gọi khi giá trị được hiện thực hoá (materialized), tức khi đọc lần tiếp theo, khi các actor của step tiếp theo chạy, hoặc khi replay lịch sử.

    Hệ quả thực tế khi thiết kế một reducer:

    * **Hãy làm nó thành một hàm thuần (pure function) của `(state, writes)`.** Bất kỳ side effect, tính ngẫu nhiên, hay đọc đồng hồ thực (wall-clock) nào (ví dụ `uuid.uuid4()`, `datetime.now()`) sẽ thực thi mỗi khi giá trị được tái tạo và cho ra kết quả khác nhau ở mỗi lần replay. Chúng *không* được "nướng" (baked) vào các lượt ghi đã persist.
    * **Đừng phụ thuộc vào việc các thay đổi (mutation) lên lượt ghi đến sẽ được persist.** Nếu reducer của bạn mutate một đối tượng ghi (ví dụ, gán một ID ổn định cho một item đến mà chưa có ID), thay đổi đó chỉ tồn tại trong giá trị được tái tạo. Lượt ghi đã lưu vẫn giữ hình dạng ban đầu, nên lần tái tạo tiếp theo sẽ lại thấy input chưa bị mutate.
    * **Gắn định danh (identity) và các metadata ổn định khác từ thượng nguồn (upstream).** Nếu code phía sau cần tham chiếu một item theo ID qua các lượt (ví dụ để cập nhật hoặc xoá nó sau), hãy gán ID đó trước khi giá trị được ghi vào channel, không phải bên trong reducer.

Dưới đây là các bulk reducer cho hai trường hợp phổ biến nhất:

```python
from typing import Any, Sequence


# List: nối tất cả các lượt ghi theo thứ tự
def list_reducer(state: list[Any], writes: Sequence[list[Any]]) -> list[Any]:
    result = list(state)
    for write in writes:
        result.extend(write)
    return result


# Dict: gộp tất cả các lượt ghi, lượt ghi cuối cùng thắng khi key xung đột
def dict_reducer(
    state: dict[str, Any], writes: Sequence[dict[str, Any]]
) -> dict[str, Any]:
    result = dict(state)
    for write in writes:
        result.update(write)
    return result
```

Cả hai đều có tính kết hợp: áp dụng từng lô một cho ra kết quả giống như áp dụng chúng cùng lúc.

#### Dùng snapshot_frequency để giới hạn độ trễ đọc

Không có snapshot, đọc một giá trị `DeltaChannel` đòi hỏi phải replay toàn bộ lịch sử ghi, O(N) đối với một thread có N step. Đặt `snapshot_frequency=K` sẽ ghi một snapshot đầy đủ sau mỗi K pregel step, giới hạn độ sâu đọc tối đa K step:

```python
class State(TypedDict):
    messages: Annotated[
        list[str],
        DeltaChannel(my_reducer, snapshot_frequency=5),
    ]
```

Giá trị `snapshot_frequency` cao hơn giảm chi phí lưu trữ nhưng tăng độ trễ đọc. Giá trị thấp hơn giới hạn độ trễ chặt hơn với cái giá là checkpoint lớn hơn. `None` (mặc định) bỏ qua hoàn toàn snapshot, phù hợp khi việc đọc hiếm khi xảy ra hoặc thread ngắn.

#### Tương thích phiên bản và rollback

!!! warning ""
    **Không hỗ trợ rollback về phiên bản không có `DeltaChannel`.** `langgraph>=1.2` ghi checkpoint delta channel theo định dạng mới mà các phiên bản trước đó không đọc được. Khi một thread đã dùng `DeltaChannel`, hạ cấp (downgrade) LangGraph sẽ khiến các checkpoint đó không đọc được vì các runtime cũ hơn không hiểu định dạng delta và không thể tái tạo lại state của channel. Nếu bạn cần rollback, hãy dùng [script khôi phục delta-channel-dump](https://github.com/langchain-ai/langgraph/tree/main/examples/delta-channel-dump) để migrate các thread bị ảnh hưởng, hoặc bỏ chúng đi, trước khi downgrade.

## Ví dụ

Trong khi hầu hết người dùng sẽ tương tác với Pregel thông qua API [StateGraph](https://reference.langchain.com/python/langgraph/graph/state/StateGraph) hoặc decorator [`@entrypoint`](https://reference.langchain.com/python/langgraph/func/entrypoint), vẫn có thể tương tác trực tiếp với Pregel.

Dưới đây là một vài ví dụ khác nhau để bạn có cảm nhận về Pregel API.

=== "Single node"

    ```python
    from langgraph.channels import EphemeralValue
    from langgraph.pregel import Pregel, NodeBuilder

    node1 = (
        NodeBuilder().subscribe_only("a")
        .do(lambda x: x + x)
        .write_to("b")
    )

    app = Pregel(
        nodes={"node1": node1},
        channels={
            "a": EphemeralValue(str),
            "b": EphemeralValue(str),
        },
        input_channels=["a"],
        output_channels=["b"],
    )

    app.invoke({"a": "foo"})
    ```

    ```con
    {'b': 'foofoo'}
    ```

=== "Multiple nodes"

    ```python
    from langgraph.channels import LastValue, EphemeralValue
    from langgraph.pregel import Pregel, NodeBuilder

    node1 = (
        NodeBuilder().subscribe_only("a")
        .do(lambda x: x + x)
        .write_to("b")
    )

    node2 = (
        NodeBuilder().subscribe_only("b")
        .do(lambda x: x + x)
        .write_to("c")
    )


    app = Pregel(
        nodes={"node1": node1, "node2": node2},
        channels={
            "a": EphemeralValue(str),
            "b": LastValue(str),
            "c": EphemeralValue(str),
        },
        input_channels=["a"],
        output_channels=["b", "c"],
    )

    app.invoke({"a": "foo"})
    ```

    ```con
    {'b': 'foofoo', 'c': 'foofoofoofoo'}
    ```

=== "Topic"

    ```python
    from langgraph.channels import EphemeralValue, Topic
    from langgraph.pregel import Pregel, NodeBuilder

    node1 = (
        NodeBuilder().subscribe_only("a")
        .do(lambda x: x + x)
        .write_to("b", "c")
    )

    node2 = (
        NodeBuilder().subscribe_to("b")
        .do(lambda x: x["b"] + x["b"])
        .write_to("c")
    )

    app = Pregel(
        nodes={"node1": node1, "node2": node2},
        channels={
            "a": EphemeralValue(str),
            "b": EphemeralValue(str),
            "c": Topic(str, accumulate=True),
        },
        input_channels=["a"],
        output_channels=["c"],
    )

    app.invoke({"a": "foo"})
    ```

    ```pycon
    {'c': ['foofoo', 'foofoofoofoo']}
    ```

=== "BinaryOperatorAggregate"

    Ví dụ này minh hoạ cách dùng channel [`BinaryOperatorAggregate`](https://reference.langchain.com/python/langgraph/channels/binop/BinaryOperatorAggregate) để triển khai một reducer.

    ```python
    from langgraph.channels import EphemeralValue, BinaryOperatorAggregate
    from langgraph.pregel import Pregel, NodeBuilder


    node1 = (
        NodeBuilder().subscribe_only("a")
        .do(lambda x: x + x)
        .write_to("b", "c")
    )

    node2 = (
        NodeBuilder().subscribe_only("b")
        .do(lambda x: x + x)
        .write_to("c")
    )

    def reducer(current, update):
        if current:
            return current + " | " + update
        else:
            return update

    app = Pregel(
        nodes={"node1": node1, "node2": node2},
        channels={
            "a": EphemeralValue(str),
            "b": EphemeralValue(str),
            "c": BinaryOperatorAggregate(str, operator=reducer),
        },
        input_channels=["a"],
        output_channels=["c"],
    )

    app.invoke({"a": "foo"})
    ```

    ```console
    { 'c': 'foofoo | foofoofoofoo' }
    ```

=== "Cycle"

    Ví dụ này minh hoạ cách tạo một vòng lặp (cycle) trong graph, bằng cách cho
    một chain ghi vào một channel mà nó đăng ký. Việc thực thi sẽ tiếp tục
    cho tới khi một giá trị `None` được ghi vào channel.

    ```python
    from langgraph.channels import EphemeralValue
    from langgraph.pregel import Pregel, NodeBuilder, ChannelWriteEntry

    example_node = (
        NodeBuilder().subscribe_only("value")
        .do(lambda x: x + x if len(x) < 10 else None)
        .write_to(ChannelWriteEntry("value", skip_none=True))
    )

    app = Pregel(
        nodes={"example_node": example_node},
        channels={
            "value": EphemeralValue(str),
        },
        input_channels=["value"],
        output_channels=["value"],
    )

    app.invoke({"value": "a"})
    ```

    ```pycon
    {'value': 'aaaaaaaaaaaaaaaa'}
    ```

## API mức cao

LangGraph cung cấp hai API mức cao để tạo một ứng dụng Pregel: [StateGraph (Graph API)](./graph-api.md) và [Functional API](./functional-api.md).

=== "StateGraph (Graph API)"

    [StateGraph (Graph API)](https://reference.langchain.com/python/langgraph/graph/state/StateGraph) là một tầng trừu tượng mức cao hơn giúp đơn giản hoá việc tạo ứng dụng Pregel. Nó cho phép bạn định nghĩa một graph gồm các node và cạnh. Khi bạn compile graph, StateGraph API sẽ tự động tạo ứng dụng Pregel cho bạn.

    ```python
    from typing import TypedDict

    from langgraph.constants import START
    from langgraph.graph import StateGraph

    class Essay(TypedDict):
        topic: str
        content: str | None
        score: float | None

    def write_essay(essay: Essay):
        return {
            "content": f"Essay about {essay['topic']}",
        }

    def score_essay(essay: Essay):
        return {
            "score": 10
        }

    builder = StateGraph(Essay)
    builder.add_node(write_essay)
    builder.add_node(score_essay)
    builder.add_edge(START, "write_essay")
    builder.add_edge("write_essay", "score_essay")

    # Compile graph.
    # Việc này sẽ trả về một instance Pregel.
    graph = builder.compile()
    ```

    Instance Pregel đã compile sẽ được gắn với một danh sách node và channel. Bạn có thể kiểm tra các node và channel bằng cách in chúng ra.

    ```python
    print(graph.nodes)
    ```

    Bạn sẽ thấy thứ gì đó như thế này:

    ```pycon
    {'__start__': <langgraph.pregel.read.PregelNode at 0x7d05e3ba1810>,
     'write_essay': <langgraph.pregel.read.PregelNode at 0x7d05e3ba14d0>,
     'score_essay': <langgraph.pregel.read.PregelNode at 0x7d05e3ba1710>}
    ```

    ```python
    print(graph.channels)
    ```

    Bạn sẽ thấy thứ gì đó như thế này

    ```pycon
    {'topic': <langgraph.channels.last_value.LastValue at 0x7d05e3294d80>,
     'content': <langgraph.channels.last_value.LastValue at 0x7d05e3295040>,
     'score': <langgraph.channels.last_value.LastValue at 0x7d05e3295980>,
     '__start__': <langgraph.channels.ephemeral_value.EphemeralValue at 0x7d05e3297e00>,
     'write_essay': <langgraph.channels.ephemeral_value.EphemeralValue at 0x7d05e32960c0>,
     'score_essay': <langgraph.channels.ephemeral_value.EphemeralValue at 0x7d05e2d8ab80>,
     'branch:__start__:__self__:write_essay': <langgraph.channels.ephemeral_value.EphemeralValue at 0x7d05e32941c0>,
     'branch:__start__:__self__:score_essay': <langgraph.channels.ephemeral_value.EphemeralValue at 0x7d05e2d88800>,
     'branch:write_essay:__self__:write_essay': <langgraph.channels.ephemeral_value.EphemeralValue at 0x7d05e3295ec0>,
     'branch:write_essay:__self__:score_essay': <langgraph.channels.ephemeral_value.EphemeralValue at 0x7d05e2d8ac00>,
     'branch:score_essay:__self__:write_essay': <langgraph.channels.ephemeral_value.EphemeralValue at 0x7d05e2d89700>,
     'branch:score_essay:__self__:score_essay': <langgraph.channels.ephemeral_value.EphemeralValue at 0x7d05e2d8b400>,
     'start:write_essay': <langgraph.channels.ephemeral_value.EphemeralValue at 0x7d05e2d8b280>}
    ```

=== "Functional API"

    Trong [Functional API](./functional-api.md), bạn có thể dùng một [`@entrypoint`](https://reference.langchain.com/python/langgraph/func/entrypoint) để tạo một ứng dụng Pregel. Decorator `entrypoint` cho phép bạn định nghĩa một hàm nhận input và trả về output.

    ```python
    from typing import TypedDict

    from langgraph.checkpoint.memory import InMemorySaver
    from langgraph.func import entrypoint

    class Essay(TypedDict):
        topic: str
        content: str | None
        score: float | None


    checkpointer = InMemorySaver()

    @entrypoint(checkpointer=checkpointer)
    def write_essay(essay: Essay):
        return {
            "content": f"Essay about {essay['topic']}",
        }

    print("Nodes: ")
    print(write_essay.nodes)
    print("Channels: ")
    print(write_essay.channels)
    ```

    ```pycon
    Nodes:
    {'write_essay': <langgraph.pregel.read.PregelNode object at 0x7d05e2f9aad0>}
    Channels:
    {'__start__': <langgraph.channels.ephemeral_value.EphemeralValue object at 0x7d05e2c906c0>, '__end__': <langgraph.channels.last_value.LastValue object at 0x7d05e2c90c40>, '__previous__': <langgraph.channels.last_value.LastValue object at 0x7d05e1007280>}
    ```

---

!!! info "Kết nối tài liệu này"
    [Kết nối tài liệu này](https://docs.langchain.com/use-these-docs) với Claude, VSCode, và nhiều công cụ khác qua MCP để có câu trả lời theo thời gian thực.

!!! info "Đóng góp"
    [Chỉnh sửa trang này trên GitHub](https://github.com/langchain-ai/docs/edit/main/src/oss/langgraph/pregel.mdx) hoặc [báo lỗi](https://github.com/langchain-ai/docs/issues/new/choose).
