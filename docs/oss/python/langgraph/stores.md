# Stores

> LangGraph store cung cấp bộ nhớ dài hạn xuyên luồng (cross-thread long-term memory), bổ sung cho khả năng lưu trữ per-thread của checkpointer.

Store cho phép agent lưu giữ thông tin xuyên suốt các thread, bao gồm sở thích người dùng, kiến thức tích luỹ, và các sự kiện cần tồn tại lâu hơn một cuộc hội thoại đơn lẻ. Khác với [checkpointer](./checkpointers.md), vốn lưu toàn bộ state của graph giới hạn trong một thread, store lưu trữ dữ liệu key-value tuỳ ý, có thể truy cập từ bất kỳ thread nào.

<img src="https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/shared_state.png?fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=354526fb48c5eb11b4b2684a2df40d6c" alt="Model of shared state" width="1482" height="777" data-path="oss/images/shared_state.png" />

!!! info "Agent Server tự động xử lý store"
    Khi dùng [Agent Server](https://docs.langchain.com/langsmith/agent-server), bạn không cần tự triển khai hay cấu hình store thủ công. API sẽ xử lý toàn bộ hạ tầng lưu trữ cho bạn ở phía sau.

!!! note ""
    [InMemoryStore](https://reference.langchain.com/python/langchain-core/stores/InMemoryStore) phù hợp cho phát triển và kiểm thử. Với production, hãy dùng một store bền vững (persistent) như `PostgresStore`, `MongoDBStore`, `RedisStore`, hoặc `UpstashStore`. Mọi triển khai đều kế thừa [BaseStore](https://reference.langchain.com/python/langchain-core/stores/BaseStore), đây là kiểu annotation cần dùng trong chữ ký hàm node.

!!! note ""
    Xem [tích hợp store](https://docs.langchain.com/oss/python/integrations/long-term-memory/index) để có danh sách đầy đủ các nhà cung cấp.

## Cách dùng cơ bản

Đoạn code sau minh hoạ [InMemoryStore](https://reference.langchain.com/python/langchain-core/stores/InMemoryStore) một cách độc lập, không dùng LangGraph:

```python
from langgraph.store.memory import InMemoryStore
store = InMemoryStore()
```

Bộ nhớ (memory) được namespace hoá bởi một `tuple`, trong ví dụ dưới đây là `(<user_id>, "memories")`. Namespace có thể có độ dài bất kỳ và biểu diễn bất cứ điều gì, không nhất thiết phải gắn với người dùng.

```python
user_id = "1"
namespace_for_memory = (user_id, "memories")
```

Dùng phương thức `store.put` để lưu bộ nhớ vào namespace trong store. Chỉ định namespace như đã định nghĩa ở trên, cùng một cặp key-value cho bộ nhớ: key đơn giản là một định danh duy nhất cho bộ nhớ (`memory_id`), còn value (một dictionary) chính là bộ nhớ đó.

```python
memory_id = str(uuid.uuid4())
memory = {"food_preference" : "I like pizza"}
store.put(namespace_for_memory, memory_id, memory)
```

Đọc bộ nhớ từ namespace của bạn bằng phương thức `store.search`, phương thức này trả về bộ nhớ cho một người dùng nhất định dưới dạng danh sách, tối đa theo tham số `limit` (mặc định `10`). Với `InMemoryStore`, các item được trả về theo thứ tự chèn (insertion order), nên bộ nhớ gần nhất nằm cuối danh sách; các backend khác có thể sắp xếp bộ nhớ theo cách khác (xem [Liệt kê item trong một namespace](#listing-items-in-a-namespace)).

```python
memories = store.search(namespace_for_memory)
memories[-1].dict()
{'value': {'food_preference': 'I like pizza'},
 'key': '07e0caf4-1631-47b7-b15f-65515d4c1843',
 'namespace': ['1', 'memories'],
 'created_at': '2024-10-02T17:22:31.590602+00:00',
 'updated_at': '2024-10-02T17:22:31.590605+00:00'}
```

Mỗi loại bộ nhớ là một Python class ([`Item`](https://langchain-ai.github.io/langgraph/reference/store/#langgraph.store.base.Item)) với một số thuộc tính nhất định. Ta có thể truy cập nó dưới dạng dictionary bằng cách convert với `.dict`.

Các thuộc tính nó có:

* `value`: Giá trị (bản thân nó là một dictionary) của bộ nhớ này

* `key`: Một key duy nhất cho bộ nhớ này trong namespace này

* `namespace`: Một tuple các chuỗi, là namespace của loại bộ nhớ này

    !!! note ""
        Dù kiểu là `tuple[str, ...]`, nó có thể được serialize thành một list khi chuyển sang JSON (ví dụ, `['1', 'memories']`).

* `created_at`: Timestamp cho thời điểm bộ nhớ này được tạo

* `updated_at`: Timestamp cho thời điểm bộ nhớ này được cập nhật

## Liệt kê item trong một namespace

Gọi [`store.search`](https://reference.langchain.com/python/langgraph/store/#langgraph.store.base.BaseStore.search) (hoặc phiên bản bất đồng bộ [`store.asearch`](https://reference.langchain.com/python/langgraph/store/#langgraph.store.base.BaseStore.asearch)) mà không có `query` và không có `filter` sẽ trả về các item được lưu dưới `namespace_prefix`, tối đa theo `limit`. Dùng cách này để liệt kê mọi thứ trong một namespace khi bạn không cần xếp hạng theo ngữ nghĩa (semantic ranking).

```python
# Return up to 100 items stored under ("alice", "memories").
items = store.search(("alice", "memories"), limit=100)
```

Ba hành vi cần lưu ý:

* **`namespace_prefix` khớp theo tiền tố (prefix), không khớp chính xác.** `("alice",)` cũng trả về các item dưới `("alice", "memories")`, `("alice", "preferences")`, v.v. Để giới hạn ở một cấp duy nhất, hãy truyền namespace đầy đủ hoặc lọc các item trả về ở phía client dựa trên `item.namespace`.
* **Kết quả vượt quá `limit` sẽ bị cắt bớt âm thầm.** Không có tín hiệu báo tràn (overflow signal), hãy đặt `limit` cao hơn mức tối đa dự kiến, hoặc phân trang bằng `offset`.
* **Thứ tự mặc định phụ thuộc vào store backend.** `PostgresStore` và `AsyncPostgresStore` trả về kết quả sắp theo `updated_at` giảm dần (cập nhật gần nhất trước). `InMemoryStore` trả về kết quả theo thứ tự chèn (chèn gần nhất sau cùng). Đừng dựa vào một thứ tự cụ thể giữa các triển khai; hãy sắp xếp phía client theo `item.updated_at` nếu thứ tự quan trọng.

Để phân trang qua một namespace lớn:

```python
page_size = 50
offset = 0
while True:
    page = store.search(("alice", "memories"), limit=page_size, offset=offset)
    if not page:
        break
    for item in page:
        pass
    offset += page_size
```

Để khám phá những namespace nào tồn tại (ví dụ, để lặp qua từng người dùng trước khi liệt kê bộ nhớ của họ), dùng [`store.list_namespaces`](https://reference.langchain.com/python/langgraph/store/#langgraph.store.base.BaseStore.list_namespaces) hoặc [`store.alist_namespaces`](https://reference.langchain.com/python/langgraph/store/#langgraph.store.base.BaseStore.alist_namespaces):

```python
# All namespaces that start with ("alice",), truncated to two levels deep.
namespaces = store.list_namespaces(prefix=("alice",), max_depth=2)
```

## Tìm kiếm ngữ nghĩa (semantic search)

Ngoài việc truy xuất đơn giản, store còn hỗ trợ tìm kiếm ngữ nghĩa, cho phép bạn tìm bộ nhớ dựa trên ý nghĩa thay vì khớp chính xác. Để bật tính năng này, cấu hình store với một embedding model:

```python
from langchain.embeddings import init_embeddings

store = InMemoryStore(
    index={
        "embed": init_embeddings("openai:text-embedding-3-small"),  # Embedding provider
        "dims": 1536,                              # Embedding dimensions
        "fields": ["food_preference", "$"]              # Fields to embed
    }
)
```

Giờ khi tìm kiếm, bạn có thể dùng truy vấn ngôn ngữ tự nhiên để tìm bộ nhớ liên quan:

```python
# Find memories about food preferences
# (This can be done after putting memories into the store)
memories = store.search(
    namespace_for_memory,
    query="What does the user like to eat?",
    limit=3  # Return top 3 matches
)
```

Bạn có thể kiểm soát phần nào của bộ nhớ được embed bằng cách cấu hình tham số `fields`, hoặc bằng cách chỉ định tham số `index` khi lưu bộ nhớ:

```python
# Store with specific fields to embed
store.put(
    namespace_for_memory,
    str(uuid.uuid4()),
    {
        "food_preference": "I love Italian cuisine",
        "context": "Discussing dinner plans"
    },
    index=["food_preference"]  # Only embed "food_preferences" field
)

# Store without embedding (still retrievable, but not searchable)
store.put(
    namespace_for_memory,
    str(uuid.uuid4()),
    {"system_info": "Last updated: 2024-01-01"},
    index=False
)
```

## Dùng trong LangGraph

Store hoạt động song hành với checkpointer: checkpointer lưu state vào các thread như đã bàn ở trên, còn store cho phép bạn lưu thông tin tuỳ ý để truy cập *xuyên* các thread. Compile graph với cả checkpointer và store như sau.

```python
from dataclasses import dataclass
from langgraph.checkpoint.memory import InMemorySaver

@dataclass
class Context:
    user_id: str

# We need this because we want to enable threads (conversations)
checkpointer = InMemorySaver()

# ... Define the graph ...

# Compile the graph with the checkpointer and store
builder = StateGraph(MessagesState, context_schema=Context)
# ... add nodes and edges ...
graph = builder.compile(checkpointer=checkpointer, store=store)
```

Sau đó invoke graph với `thread_id` như trước, và thêm cả `user_id`, dùng làm namespace cho bộ nhớ của người dùng cụ thể này như trước.

```python
# Invoke the graph
config = {"configurable": {"thread_id": "1"}}

# First let's just say hi to the AI
for update in graph.stream(
    {"messages": [{"role": "user", "content": "hi"}]},
    config,
    stream_mode="updates",
    context=Context(user_id="1"),
):
    print(update)
```

Bạn có thể truy cập store và `user_id` từ *bất kỳ node nào* bằng cách dùng đối tượng `Runtime`. `Runtime` được LangGraph tự động chèn vào khi bạn thêm nó làm tham số cho hàm node của mình. Bạn có thể dùng nó để lưu bộ nhớ:

```python
from langgraph.runtime import Runtime
from dataclasses import dataclass

@dataclass
class Context:
    user_id: str

async def update_memory(state: MessagesState, runtime: Runtime[Context]):

    # Get the user id from the runtime context
    user_id = runtime.context.user_id

    # Namespace the memory
    namespace = (user_id, "memories")

    # ... Analyze conversation and create a new memory

    # Create a new memory ID
    memory_id = str(uuid.uuid4())

    # We create a new memory
    await runtime.store.aput(namespace, memory_id, {"memory": memory})

```

Bạn cũng có thể truy cập store từ bất kỳ node nào và dùng phương thức `store.search` để lấy bộ nhớ. Bộ nhớ được trả về dưới dạng một danh sách đối tượng, có thể convert thành dictionary.

```python
memories[-1].dict()
{'value': {'food_preference': 'I like pizza'},
 'key': '07e0caf4-1631-47b7-b15f-65515d4c1843',
 'namespace': ['1', 'memories'],
 'created_at': '2024-10-02T17:22:31.590602+00:00',
 'updated_at': '2024-10-02T17:22:31.590605+00:00'}
```

Bạn truy cập bộ nhớ và dùng chúng trong các lần gọi model.

```python
from dataclasses import dataclass
from langgraph.runtime import Runtime

@dataclass
class Context:
    user_id: str

async def call_model(state: MessagesState, runtime: Runtime[Context]):
    # Get the user id from the runtime context
    user_id = runtime.context.user_id

    # Namespace the memory
    namespace = (user_id, "memories")

    # Search based on the most recent message
    memories = await runtime.store.asearch(
        namespace,
        query=state["messages"][-1].content,
        limit=3
    )
    info = "\n".join([d.value["memory"] for d in memories])

    # ... Use memories in the model call
```

Nếu bạn tạo một thread mới, bạn vẫn có thể truy cập cùng bộ nhớ đó miễn là `user_id` giống nhau.

```python
# Invoke the graph on a new thread
config = {"configurable": {"thread_id": "2"}}

# Let's say hi again
for update in graph.stream(
    {"messages": [{"role": "user", "content": "hi, tell me about my memories"}]},
    config,
    stream_mode="updates",
    context=Context(user_id="1"),
):
    print(update)
```

Khi bạn dùng LangSmith cục bộ (ví dụ, trong [Studio](https://docs.langchain.com/langsmith/studio)) hoặc [hosted](https://docs.langchain.com/langsmith/platform-setup), base store có sẵn để dùng theo mặc định và bạn không cần chỉ định nó khi compile graph. Tuy nhiên, để bật tìm kiếm ngữ nghĩa, bạn **cần** cấu hình các cài đặt indexing trong file `langgraph.json`. Ví dụ:

```json
{
    ...
    "store": {
        "index": {
            "embed": "openai:text-embeddings-3-small",
            "dims": 1536,
            "fields": ["$"]
        }
    }
}
```

Xem [hướng dẫn deploy](https://docs.langchain.com/langsmith/semantic-search) để biết thêm chi tiết và tuỳ chọn cấu hình.

## Xây dựng store tuỳ chỉnh

Để dùng một storage backend khác với các triển khai dựng sẵn, hãy subclass [BaseStore](https://reference.langchain.com/python/langchain-core/stores/BaseStore) và triển khai các phương thức bắt buộc của nó. [InMemoryStore](https://reference.langchain.com/python/langchain-core/stores/InMemoryStore) dựng sẵn là triển khai tham chiếu đơn giản nhất.

### Hợp đồng cơ bản (base contract)

Cả năm phương thức async đều bắt buộc. Phiên bản đồng bộ tương ứng (`put`, `get`, `delete`, `search`, `list_namespaces`) là tuỳ chọn nhưng được khuyến nghị để tương thích với thực thi graph đồng bộ.

| Phương thức | Mô tả |
| --- | --- |
| `aput(namespace, key, value, index=None)` | Lưu hoặc ghi đè một item duy nhất |
| `aget(namespace, key)` | Truy xuất một item duy nhất theo key; trả về `None` nếu không tồn tại |
| `adelete(namespace, key)` | Xoá một item duy nhất |
| `asearch(namespace_prefix, *, query=None, filter=None, limit=10, offset=0)` | Tìm kiếm item dưới một tiền tố namespace; tuỳ chọn theo truy vấn ngữ nghĩa |
| `alist_namespaces(*, prefix=None, suffix=None, max_depth=None, limit=100, offset=0)` | Liệt kê các namespace khớp mẫu tiền tố/hậu tố |

Tra cứu chữ ký chính xác trước khi triển khai:

```python
import inspect
from langgraph.store.base import BaseStore
print(inspect.getsource(BaseStore))
```

### Thiết kế namespace

Namespace là các tuple chuỗi, ví dụ `("user_id", "memories")`. Các triển khai store phải hỗ trợ:

* **Khớp tiền tố (prefix matching)**: `asearch(("alice",))` trả về các item dưới `("alice",)`, `("alice", "memories")`, và bất kỳ sub-namespace nào khác.
* **Tra cứu key chính xác**: `aget(("alice", "memories"), "some-key")` phải có độ phức tạp O(1) hoặc gần với nó.

Với backend SQL, một schema phổ biến:

```sql
CREATE TABLE store_items (
    namespace   TEXT[] NOT NULL,
    key         TEXT NOT NULL,
    value       JSONB NOT NULL,
    created_at  TIMESTAMPTZ DEFAULT now(),
    updated_at  TIMESTAMPTZ DEFAULT now(),
    PRIMARY KEY (namespace, key)
);

CREATE INDEX ON store_items USING gin(namespace);
```

### Serialization

Giá trị store là các dict Python thuần, không cần serializer đặc biệt nào. Serialize bằng `json.dumps` / `json.loads` hoặc trực tiếp bằng cột JSONB. Đừng lưu các đối tượng Python thô không thể serialize sang JSON.

### Hỗ trợ tìm kiếm ngữ nghĩa

Nếu backend của bạn hỗ trợ tìm kiếm vector, hãy triển khai tham số `query` trên `asearch`:

* Chấp nhận một tham số `query: str | None`.
* Khi `query` khác `None`, embed nó và xếp hạng kết quả theo độ tương đồng cosine (cosine similarity).
* Kết quả nên có một trường `score` trên mỗi `Item` khi `query` được cung cấp.

Nếu backend của bạn không hỗ trợ tìm kiếm vector, hãy raise `NotImplementedError` khi `query` được truyền vào.

### Kiểm thử

Hiện chưa có bộ conformance suite cho store tuỳ chỉnh. Hãy kiểm thử đối chiếu với [InMemoryStore](https://reference.langchain.com/python/langchain-core/stores/InMemoryStore) làm tham chiếu:

```python
import pytest
from langgraph.store.memory import InMemoryStore
from your_module import YourStore

@pytest.fixture
async def store():
    async with YourStore.create() as s:
        yield s

@pytest.fixture
def reference():
    return InMemoryStore()

async def test_put_and_get(store, reference):
    ns = ("test", "ns")
    for s in [store, reference]:
        await s.aput(ns, "k1", {"val": 1})
        item = await s.aget(ns, "k1")
        assert item is not None
        assert item.value == {"val": 1}

async def test_delete(store, reference):
    ns = ("test", "ns")
    for s in [store, reference]:
        await s.aput(ns, "k1", {"val": 1})
        await s.adelete(ns, "k1")
        assert await s.aget(ns, "k1") is None

async def test_search_prefix(store, reference):
    for s in [store, reference]:
        await s.aput(("user", "memories"), "m1", {"text": "likes pizza"})
        results = await s.asearch(("user",))
        assert any(r.key == "m1" for r in results)
```

### Bước tiếp theo

* [Thêm store tuỳ chỉnh vào Agent Server](https://docs.langchain.com/langsmith/custom-store), triển khai (deploy) implementation của bạn
* [Checkpointer](./checkpointers.md), lưu trữ state giới hạn trong thread

---

!!! info "Kết nối tài liệu này"
    [Kết nối tài liệu này](https://docs.langchain.com/use-these-docs) với Claude, VSCode, và nhiều công cụ khác qua MCP để có câu trả lời theo thời gian thực.

!!! info "Đóng góp"
    [Chỉnh sửa trang này trên GitHub](https://github.com/langchain-ai/docs/edit/main/src/oss/langgraph/stores.mdx) hoặc [báo lỗi](https://github.com/langchain-ai/docs/issues/new/choose).
