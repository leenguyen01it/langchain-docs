# Persistence

> Lớp persistence của LangGraph cung cấp bộ nhớ ngắn hạn cho agent thông qua checkpointer, và bộ nhớ dài hạn thông qua store.

<a id="checkpoints" />

<a id="threads" />

<a id="memory-store" />

<a id="checkpointer-libraries" />

<a id="pending-writes" />

<a id="durability-modes" />

Persistence cho phép ứng dụng LangGraph giữ lại thông tin hữu ích vượt ra ngoài một lần chạy graph. Điều này quan trọng khi agent cần tiếp tục một cuộc hội thoại, resume sau khi bị gián đoạn, phục hồi sau lỗi, hoặc ghi nhớ thông tin qua các lượt tương tác.

LangGraph cung cấp hai hệ thống persistence bổ trợ cho nhau:

* **[Checkpointer](checkpointers.md)** lưu trạng thái graph của một thread dưới dạng các checkpoint. Dùng cho bộ nhớ ngắn hạn, giới hạn trong phạm vi thread, bao gồm tính liên tục của cuộc hội thoại, quy trình human-in-the-loop, time travel, và khả năng chịu lỗi (fault tolerance).
* **[Store](stores.md)** lưu dữ liệu do ứng dụng định nghĩa, nằm ngoài trạng thái graph. Dùng cho bộ nhớ dài hạn, dùng chung giữa các thread, bao gồm tùy chọn người dùng, các fact, và kiến thức dùng chung.

Hầu hết ứng dụng có thể dùng cả hai: một [checkpointer](checkpointers.md) theo dõi thread hiện tại, còn một [store](stores.md) theo dõi thông tin bền vững qua nhiều thread.

## Bắt đầu nhanh

Compile graph của bạn với một checkpointer, một store, hoặc cả hai:

```python
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.store.memory import InMemoryStore

checkpointer = InMemorySaver()
store = InMemoryStore()

graph = builder.compile(checkpointer=checkpointer, store=store)

result = graph.invoke(
    {"messages": [{"role": "user", "content": "Hi, my name is Bob."}]},
    {"configurable": {"thread_id": "thread-1"}},
)
```

!!! info "Agent Server tự xử lý persistence"
    Khi dùng [Agent Server](https://docs.langchain.com/langsmith/agent-server), bạn không cần tự triển khai hay cấu hình checkpointer hoặc store. Server tự xử lý hạ tầng persistence phía sau.

## Checkpointer so với store

|                | Checkpointer                                                                 | Store                                               |
| -------------- | ---------------------------------------------------------------------------- | --------------------------------------------------- |
| Lưu trữ        | Snapshot trạng thái graph                                                    | Dữ liệu key-value do ứng dụng định nghĩa            |
| Phạm vi        | Một thread duy nhất                                                          | Qua nhiều thread                                    |
| Loại bộ nhớ    | Bộ nhớ ngắn hạn, giới hạn trong thread                                       | Bộ nhớ dài hạn, dùng chung qua nhiều thread          |
| Dùng cho       | Tính liên tục hội thoại, human-in-the-loop, time travel, khả năng chịu lỗi   | Tùy chọn người dùng, fact, kiến thức dùng chung      |
| Cách truy cập  | Truyền `thread_id` trong config của graph                                   | Đọc và ghi item từ node hoặc code ứng dụng           |
| Hướng dẫn đầy đủ | [Checkpointer](checkpointers.md)                                           | [Store](stores.md)                                  |

## Xử lý sự cố thường gặp

### PostgresSaver: `thread_id` quá dài

Khi dùng `PostgresSaver` (hoặc `AsyncPostgresSaver`), `thread_id` được lưu trong một cột có giới hạn độ dài. Nếu `thread_id` của bạn vượt quá kích thước cột, bạn sẽ gặp lỗi database.

**Cách khắc phục:** Giữ giá trị `thread_id` dưới 255 ký tự. Dùng UUID hoặc hash nếu bạn cần ID xác định (deterministic):

```python
import uuid

config = {"configurable": {"thread_id": str(uuid.uuid4())[:255]}}
```

### `MemorySaver` không lưu lại giữa các lần restart

`MemorySaver` và `InMemorySaver` lưu checkpoint trong RAM. Khi process restart, toàn bộ checkpoint sẽ mất.

**Cách khắc phục:** Dùng một checkpointer bền vững (persistent) cho production:

* `PostgresSaver`: PostgreSQL với hỗ trợ async
* `SqliteSaver`: Lưu trữ dựa trên file cục bộ, dùng cho phát triển

### Checkpoint tăng không giới hạn

Qua các cuộc hội thoại dài, checkpoint sẽ tích lũy dần. Điều này có thể làm tăng độ trễ và chi phí lưu trữ.

**Cách khắc phục:** Định kỳ dọn bớt checkpoint cũ hoặc đặt chính sách retention:

```python
from langgraph.checkpoint.postgres import PostgresSaver

checkpointer = PostgresSaver.from_conn_string("postgresql://...")
checkpointer.setup()  # Tạo các bảng kèm index
# Cân nhắc thêm một cron job để xóa checkpoint cũ hơn N ngày
```

### Truy cập trạng thái từ graph cha sang subgraph

Khi một subgraph cập nhật trạng thái, graph cha có thể không thấy thay đổi ngay lập tức. Nguyên nhân là mỗi subgraph quản lý namespace checkpoint riêng của nó.

**Cách khắc phục:** Dùng [shared state qua Store](stores.md) cho dữ liệu cần vượt qua ranh giới graph, hoặc cấu hình subgraph của bạn để ghi vào checkpoint của graph cha.

## Bước tiếp theo

* [Dùng checkpointer](checkpointers.md) để lưu và kiểm tra trạng thái thread.
* [Dùng store](stores.md) để lưu dữ liệu bền vững qua nhiều thread.
