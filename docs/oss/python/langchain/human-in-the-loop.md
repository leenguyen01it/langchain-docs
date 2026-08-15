# Human-in-the-loop

[Middleware](middleware/built-in.md#human-in-the-loop) Human-in-the-Loop (HITL) cho phép bạn thêm sự giám sát của con người vào các tool call của agent. Khi model đề xuất một hành động có thể cần review, ví dụ ghi vào file hoặc thực thi SQL, middleware có thể tạm dừng thực thi và chờ quyết định.

Nó thực hiện điều này bằng cách kiểm tra mỗi tool call theo một policy có thể cấu hình. Nếu cần can thiệp, middleware phát ra một [interrupt](https://reference.langchain.com/python/langgraph/types/interrupt) để dừng thực thi. Graph state được lưu bằng [persistence layer](https://docs.langchain.com/oss/python/langgraph/persistence) của LangGraph, nên việc thực thi có thể tạm dừng an toàn và tiếp tục sau đó.

Một quyết định của con người sau đó sẽ quyết định điều gì xảy ra tiếp theo: hành động có thể được phê duyệt nguyên trạng (`approve`), chỉnh sửa trước khi chạy (`edit`), từ chối kèm phản hồi (`reject`), hoặc được trả lời trực tiếp (`respond`) cho các tool kiểu "hỏi người dùng".

## Các loại quyết định interrupt

[Middleware](middleware/built-in.md#human-in-the-loop) định nghĩa bốn cách phản hồi có sẵn mà con người có thể dùng cho một interrupt:

| Loại quyết định | Mô tả | Ví dụ trường hợp dùng |
| --- | --- | --- |
| ✅ `approve` | Thực thi tool với các tham số gốc như agent đã đề xuất. | Gửi email draft đúng như đã viết |
| ✏️ `edit` | Chỉnh sửa tham số tool trước khi thực thi. | Đổi người nhận trước khi gửi email |
| ❌ `reject` | Bỏ qua hoàn toàn việc thực thi tool call này và trả phản hồi từ chối cho agent. | Từ chối xoá file và giải thích lý do |
| 💬 `respond` | Trả trực tiếp message của con người như một kết quả tool tổng hợp, bỏ qua việc thực thi, dành cho tool kiểu "hỏi người dùng". | Trả lời một prompt `"ask_user"` bằng phản hồi trực tiếp |

Các loại quyết định khả dụng cho mỗi tool phụ thuộc vào policy bạn cấu hình trong `interrupt_on`. Khi nhiều tool call bị tạm dừng cùng lúc, mỗi hành động cần một quyết định riêng. Các quyết định phải được cung cấp theo đúng thứ tự các hành động xuất hiện trong interrupt request.

Dùng `reject` khi con người đang từ chối hành động được yêu cầu. Chỉ dùng `respond` khi con người đang đóng vai trò như chính tool đó, chẳng hạn trả lời một prompt `ask_user`. Không dùng `respond` để từ chối các tool có side-effect, vì message của nó được coi là một kết quả tool thành công.

!!! tip "Mẹo"
    Khi **chỉnh sửa (edit)** tham số tool, hãy thay đổi một cách thận trọng. Các sửa đổi đáng kể so với tham số gốc có thể khiến model đánh giá lại cách tiếp cận và có thể thực thi tool nhiều lần hoặc thực hiện hành động ngoài dự kiến.

## Cấu hình interrupt

Để dùng HITL, thêm [middleware](middleware/built-in.md#human-in-the-loop) vào danh sách `middleware` của agent khi tạo agent.

Bạn cấu hình nó bằng một mapping từ hành động tool sang các loại quyết định được phép cho mỗi hành động. Middleware sẽ ngắt thực thi khi một tool call khớp với một hành động trong mapping.

```python
from langchain.agents import create_agent
from langchain.agents.middleware import HumanInTheLoopMiddleware
from langgraph.checkpoint.memory import InMemorySaver


agent = create_agent(
    model="gpt-5.5",
    tools=[write_file, execute_sql, read_data],
    middleware=[
        HumanInTheLoopMiddleware(
            interrupt_on={
                "write_file": True,  # Cho phép mọi quyết định (approve, edit, reject, respond)
                "execute_sql": {"allowed_decisions": ["approve", "reject"]},  # Không cho phép edit
                "read_data": False, # Thao tác an toàn, không cần phê duyệt
            },
            # Tiền tố cho message interrupt - kết hợp với tên tool và args để tạo message đầy đủ
            # ví dụ: "Tool execution pending approval: execute_sql with query='DELETE FROM...'"
            # Từng tool riêng lẻ có thể ghi đè bằng cách chỉ định "description" trong cấu hình interrupt của nó
            description_prefix="Tool execution pending approval",
        ),
    ],
    # Human-in-the-loop yêu cầu checkpointing để xử lý interrupt.
    # Trong production, dùng checkpointer bền vững như AsyncPostgresSaver hoặc MongoDBSaver.
    checkpointer=InMemorySaver(),
)
```

!!! info "Thông tin"
    Bạn phải cấu hình một checkpointer để lưu graph state qua các lần interrupt. Trong production, dùng checkpointer bền vững như [`AsyncPostgresSaver`](https://reference.langchain.com/python/langgraph/checkpoints/#langgraph.checkpoint.postgres.aio.AsyncPostgresSaver) hoặc [`MongoDBSaver`](https://pypi.org/project/langgraph-checkpoint-mongodb/). Để test hoặc prototype, dùng [`InMemorySaver`](https://reference.langchain.com/python/langgraph/checkpoints/#langgraph.checkpoint.memory.InMemorySaver).

    Khi invoke agent, truyền một `config` bao gồm **thread ID** để gắn việc thực thi với một thread hội thoại. Xem [tài liệu interrupt của LangGraph](https://docs.langchain.com/oss/python/langgraph/interrupts) để biết chi tiết.

??? note "Các tuỳ chọn cấu hình"
    **`interrupt_on`** (`dict`, bắt buộc)
    Mapping từ tên tool sang cấu hình phê duyệt. Giá trị có thể là `True` (interrupt với cấu hình mặc định), `False` (tự động phê duyệt), hoặc một object `InterruptOnConfig`.

    **`description_prefix`** (`string`, mặc định `"Tool execution requires approval"`)
    Tiền tố cho mô tả action request

    **Các tuỳ chọn của `InterruptOnConfig`:**

    **`allowed_decisions`** (`list[string]`)
    Danh sách các quyết định được phép: `'approve'`, `'edit'`, `'reject'`, hoặc `'respond'`

    **`description`** (`string | callable`)
    Chuỗi tĩnh hoặc hàm callable cho mô tả tuỳ chỉnh

    **`when`** (`callable`)
    Predicate tuỳ chọn nhận vào một [ToolCallRequest](https://reference.langchain.com/python/langgraph.prebuilt/tool_node/ToolCallRequest) và trả về `True` để interrupt hoặc `False` để tự động phê duyệt. Dùng nó để gate interrupt dựa trên tham số của lệnh gọi. Yêu cầu `langchain>=1.3.3`.

## Interrupt có điều kiện

Theo mặc định, mọi tool call được liệt kê trong `interrupt_on` sẽ tạm dừng để review. Để chỉ tạm dừng một số lệnh gọi, thêm một predicate `when` vào `InterruptOnConfig` của một tool. Predicate nhận vào một `ToolCallRequest` và trả về `True` để interrupt hoặc `False` để tự động phê duyệt, nên bạn có thể gate dựa trên tham số của tool.

!!! note "Ghi chú"
    Interrupt có điều kiện yêu cầu `langchain>=1.3.3`.

```python
from langchain.agents import create_agent
from langchain.agents.middleware import HumanInTheLoopMiddleware, ToolCallRequest
from langgraph.checkpoint.memory import InMemorySaver


def writes_outside_workspace(request: ToolCallRequest) -> bool:
    """Tạm dừng việc ghi ra ngoài thư mục workspace."""
    path = request.tool_call["args"].get("path", "")
    return not path.startswith("/workspace/")


def is_write_query(request: ToolCallRequest) -> bool:
    """Tạm dừng SQL không phải SELECT chỉ đọc."""
    query = request.tool_call["args"].get("query", "")
    return not query.lstrip().upper().startswith("SELECT")


agent = create_agent(
    model="gpt-5.5",
    tools=[write_file, execute_sql, read_data],
    middleware=[
        HumanInTheLoopMiddleware(
            interrupt_on={
                "write_file": {
                    "allowed_decisions": ["approve", "edit", "reject"],
                    "when": writes_outside_workspace,
                },
                "execute_sql": {
                    "allowed_decisions": ["approve", "reject"],
                    "when": is_write_query,
                },
            },
        ),
    ],
    checkpointer=InMemorySaver(),
)
```

Khi predicate `when` trả về `False`, lệnh gọi chạy mà không interrupt. Khi nó trả về `True`, hoặc khi bạn bỏ qua `when`, lệnh gọi tạm dừng như bình thường. Các lệnh gọi được đánh giá là `False` không bao giờ được thêm vào batch interrupt, nên người review chỉ thấy các hành động thực sự cần quyết định.

## Phản hồi interrupt

Khi bạn invoke agent, nó chạy cho đến khi hoàn tất hoặc một interrupt được phát ra. Một interrupt được kích hoạt khi một tool call khớp với policy bạn đã cấu hình trong `interrupt_on`. Với `version="v2"`, kết quả là một `GraphOutput` có thuộc tính `interrupts` chứa các hành động cần review. Sau đó bạn có thể trình bày các hành động đó cho người review và tiếp tục thực thi khi quyết định đã được cung cấp.

```python
from langgraph.types import Command

# Human-in-the-loop tận dụng persistence layer của LangGraph.
# Bạn phải cung cấp một thread ID để gắn việc thực thi với một thread hội thoại,
# để cuộc hội thoại có thể được tạm dừng và tiếp tục (như cần thiết cho review của con người).
config = {"configurable": {"thread_id": "some_id"}}
# Chạy graph cho đến khi gặp interrupt.
result = agent.invoke(
    {
        "messages": [
            {
                "role": "user",
                "content": "Delete old records from the database",
            }
        ]
    },
    config=config,
    version="v2",
)

# result là một GraphOutput với .value và .interrupts
print(result.interrupts)
# > (
# >    Interrupt(
# >       value={
# >          'action_requests': [
# >             {
# >                'name': 'execute_sql',
# >                'arguments': {'query': 'DELETE FROM records WHERE created_at < NOW() - INTERVAL \'30 days\';'},
# >                'description': 'Tool execution pending approval\n\nTool: execute_sql\nArgs: {...}'
# >             }
# >          ],
# >          'review_configs': [
# >             {
# >                'action_name': 'execute_sql',
# >                'allowed_decisions': ['approve', 'reject']
# >             }
# >          ]
# >       }
# >    ),
# > )


# Tiếp tục với quyết định phê duyệt
agent.invoke(
    Command(
        resume={"decisions": [{"type": "approve"}]}  # hoặc "reject"
    ),
    config=config, # Cùng thread ID để tiếp tục hội thoại đang tạm dừng
    version="v2",
)
```

### Các loại quyết định

=== "✅ approve"
    Dùng `approve` để phê duyệt tool call nguyên trạng và thực thi nó mà không thay đổi.

    ```python
    agent.invoke(
        Command(
            # Các quyết định được cung cấp dưới dạng list, mỗi hành động đang review một quyết định.
            # Thứ tự quyết định phải khớp với thứ tự hành động trong interrupt request.
            resume={
                "decisions": [
                    {
                        "type": "approve",
                    }
                ]
            }
        ),
        config=config,  # Cùng thread ID để tiếp tục hội thoại đang tạm dừng
        version="v2",
    )
    ```

=== "✏️ edit"
    Dùng `edit` để chỉnh sửa tool call trước khi thực thi. Cung cấp hành động đã chỉnh sửa với tên tool và tham số mới.

    ```python
    agent.invoke(
        Command(
            # Các quyết định được cung cấp dưới dạng list, mỗi hành động đang review một quyết định.
            # Thứ tự quyết định phải khớp với thứ tự hành động trong interrupt request.
            resume={
                "decisions": [
                    {
                        "type": "edit",
                        # Hành động đã chỉnh sửa với tên tool và args
                        "edited_action": {
                            # Tên tool cần gọi.
                            # Thường sẽ giống với hành động gốc.
                            "name": "new_tool_name",
                            # Tham số truyền cho tool.
                            "args": {"key1": "new_value", "key2": "original_value"},
                        }
                    }
                ]
            }
        ),
        config=config,  # Cùng thread ID để tiếp tục hội thoại đang tạm dừng
        version="v2",
    )
    ```

    !!! tip "Mẹo"
        Khi **chỉnh sửa (edit)** tham số tool, hãy thay đổi một cách thận trọng. Các sửa đổi đáng kể so với tham số gốc có thể khiến model đánh giá lại cách tiếp cận và có thể thực thi tool nhiều lần hoặc thực hiện hành động ngoài dự kiến.

=== "❌ reject"
    Dùng `reject` để từ chối tool call và cung cấp phản hồi thay vì thực thi. Tool không được thực thi.

    ```python
    agent.invoke(
        Command(
            # Các quyết định được cung cấp dưới dạng list, mỗi hành động đang review một quyết định.
            # Thứ tự quyết định phải khớp với thứ tự hành động trong interrupt request.
            resume={
                "decisions": [
                    {
                        "type": "reject",
                        # Tuỳ chọn: giải thích vì sao hành động bị từ chối
                        # và liệu agent có nên thử lại theo cách khác hay không.
                        "message": "User rejected this action. Do not retry this tool call.",
                    }
                ]
            }
        ),
        config=config,  # Cùng thread ID để tiếp tục hội thoại đang tạm dừng
        version="v2",
    )
    ```

    `message` được thêm vào hội thoại như phản hồi để giúp agent hiểu vì sao hành động bị từ chối và nó nên làm gì tiếp theo. Khi bạn bỏ qua `message`, middleware dùng một message từ chối mặc định báo cho model biết tool không được thực thi và không nên thử lại cùng tool call trừ khi người dùng yêu cầu. Với các tool có side-effect, hãy cung cấp một message theo domain cụ thể, nói rõ liệu agent nên bỏ hành động, hỏi thêm câu hỏi làm rõ, hay thử một phương án an toàn hơn.

=== "💬 respond"
    Dùng `respond` cho các tool kiểu "hỏi người dùng" nơi triển khai thực sự của tool chính là phản hồi của con người. Nội dung `message` được trả trực tiếp làm kết quả tool; bản thân tool không được thực thi.

    ```python
    agent.invoke(
        Command(
            # Các quyết định được cung cấp dưới dạng list, mỗi hành động đang review một quyết định.
            # Thứ tự quyết định phải khớp với thứ tự hành động trong interrupt request.
            resume={
                "decisions": [
                    {
                        "type": "respond",
                        # Phản hồi của con người, được trả trực tiếp làm kết quả tool
                        "message": "Blue.",
                    }
                ]
            }
        ),
        config=config,  # Cùng thread ID để tiếp tục hội thoại đang tạm dừng
        version="v2",
    )
    ```

    `message` được trả về cho agent như một `ToolMessage` thành công. Dùng `respond` khi tool cố tình là một placeholder cho input của con người, ví dụ một tool `ask_user` yêu cầu làm rõ. Không dùng `respond` để từ chối một hành động được đề xuất, vì nó báo cho model rằng tool đã hoàn tất thành công.

---

### Nhiều quyết định

Khi nhiều hành động đang được review, cung cấp một quyết định cho mỗi hành động theo đúng thứ tự chúng xuất hiện trong interrupt:

```python
{
    "decisions": [
        {"type": "approve"},
        {
            "type": "edit",
            "edited_action": {
                "name": "tool_name",
                "args": {"param": "new_value"}
            }
        },
        {
            "type": "reject",
            "message": "This action is not allowed"
        }
    ]
}
```

## Streaming với human-in-the-loop

Bạn có thể streaming cập nhật theo thời gian thực trong khi agent chạy và xử lý interrupt bằng `stream_events()`. Dùng `stream.messages` để streaming token LLM và `stream.values` để kiểm tra snapshot state của agent cho interrupt.

```python
from langgraph.types import Command

config = {"configurable": {"thread_id": "some_id"}}

# Streaming tiến trình agent và token LLM cho đến khi gặp interrupt
stream = agent.stream_events(
    {"messages": [{"role": "user", "content": "Delete old records from the database"}]},
    config=config,
    version="v3",
)
for message in stream.messages:
    for token in message.text:
        print(token, end="", flush=True)

# Kiểm tra xem run có tạm dừng để chờ input của con người không
if stream.interrupted:
    print(f"\n\nInterrupt: {stream.interrupts}")

# Tiếp tục với streaming sau khi có quyết định của con người
stream = agent.stream_events(
    Command(resume={"decisions": [{"type": "approve"}]}),
    config=config,
    version="v3",
)
for message in stream.messages:
    for token in message.text:
        print(token, end="", flush=True)
```

Xem hướng dẫn [Streaming](streaming.md) để biết thêm chi tiết về các stream mode.

## Vòng đời thực thi

Middleware định nghĩa một hook `after_model` chạy sau khi model sinh ra phản hồi nhưng trước khi bất kỳ tool call nào được thực thi:

1. Agent invoke model để sinh phản hồi.
2. Middleware kiểm tra phản hồi để tìm tool call.
3. Nếu có lệnh gọi nào cần input của con người, middleware xây dựng một `HITLRequest` với `action_requests` và `review_configs` rồi gọi [interrupt](https://reference.langchain.com/python/langgraph/types/interrupt).
4. Agent chờ quyết định của con người.
5. Dựa trên các quyết định trong `HITLResponse`, middleware thực thi các lệnh gọi đã được phê duyệt hoặc chỉnh sửa, tổng hợp [ToolMessage](https://reference.langchain.com/python/langchain-core/messages/tool/ToolMessage) cho các lệnh gọi bị từ chối, trả trực tiếp phản hồi của con người dưới dạng [ToolMessage](https://reference.langchain.com/python/langchain-core/messages/tool/ToolMessage) cho các quyết định `respond`, và tiếp tục thực thi.

## Logic HITL tuỳ chỉnh

Với các workflow chuyên biệt hơn, bạn có thể xây dựng logic HITL tuỳ chỉnh trực tiếp bằng primitive [interrupt](https://reference.langchain.com/python/langgraph/types/interrupt) và abstraction [middleware](middleware/overview.md).

Xem lại phần [vòng đời thực thi](#vong-doi-thuc-thi) ở trên để hiểu cách tích hợp interrupt vào hoạt động của agent.
