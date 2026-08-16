# MULTIPLE_SUBGRAPHS

Lỗi này xảy ra khi bạn [gọi một subgraph bên trong một node](../use-subgraphs.md#call-a-subgraph-inside-a-node) nhiều lần, và subgraph đó được compile với `checkpointer=True` (chế độ continuations).

## Khắc phục sự cố

Chọn một trong các cách sau tuỳ theo nhu cầu của bạn:

1. **Không cần interrupt?** Dùng `checkpointer=False` để bỏ hoàn toàn checkpointing:
   ```python
   subgraph = subgraph_builder.compile(checkpointer=False)
   ```

2. **Cần interrupt nhưng không cần persistence xuyên suốt các lần invoke?** Dùng chế độ kế thừa mặc định bằng cách bỏ qua `checkpointer`:
   ```python
   subgraph = subgraph_builder.compile()
   ```
   Mỗi lần invoke sẽ nhận một namespace riêng, nên việc thực thi song song vẫn hoạt động. Subgraph khởi động lại từ đầu mỗi lần nhưng vẫn có thể dùng `interrupt()`.

3. **Cần persistence xuyên suốt các lần invoke?** Dùng `checkpointer=True`. LangGraph sẽ gán cho mỗi lần invoke một hậu tố namespace dựa trên vị trí (`calling_node`, `calling_node|1`, v.v.) để tránh xung đột. Để có namespace ổn định dựa trên tên, hãy bọc mỗi subgraph bằng một tên node riêng biệt, xem [subgraph song song](../use-subgraphs.md#subgraph-persistence).

## Liên quan

* [Subgraph persistence](../use-subgraphs.md#subgraph-persistence): so sánh đầy đủ các chế độ checkpointer
* [Persistence](../persistence.md): cách checkpointer hoạt động trong LangGraph
