# Backward compatibility

> Cập nhật code graph LangGraph trong production mà không làm hỏng các run đang chạy dở.

Phần mềm cần thay đổi trong production. Các yêu cầu mới, sửa lỗi, và refactor cuối cùng đều sẽ được đưa vào code graph của bạn. Vì LangGraph chạy graph mới nhất đã deploy dựa trên state đã được [persist](./persistence.md) cho các thread hiện có, mọi thay đổi bạn ship thực chất là một thay đổi API tương thích ngược (backward-compatible) đối với các checkpoint hiện có của bạn.

Khác với các workflow engine ghim (pin) một run vào phiên bản code mà nó bắt đầu chạy, LangGraph áp dụng graph mới nhất ngay lập tức cho *mọi* thread, cả thread mới lẫn thread đang resume từ một checkpoint. Điều này tiện lợi: các bản sửa lỗi lan truyền tới các cuộc hội thoại và agent đang chạy dở mà không cần nghi thức gì thêm. Nhưng nó cũng có nghĩa là bạn phải suy xét cách mỗi thay đổi tương tác với các run đã bắt đầu dưới phiên bản code trước đó.

Có ba nhóm vấn đề tương thích cần lưu ý, theo thứ tự bạn thường gặp phải:

1. [Tương thích kỹ thuật](#tuong-thich-ky-thuat): Phổ biến nhất; code mới vẫn phải load và thực thi được với State hiện có.
2. [Tương thích nghiệp vụ](#tuong-thich-nghiep-vu): Ít phổ biến hơn; các run hiện có vẫn nên tiếp tục theo logic nghiệp vụ cũ dù code đã thay đổi.
3. [Tính không xác định (Non-determinism)](#tinh-khong-xac-dinh): Chỉ áp dụng cho [Functional API](./functional-api.md).

!!! tip ""
    Để xem tóm tắt ngắn gọn về những thay đổi topology và state nào runtime hỗ trợ theo mặc định, xem [Graph migrations](./graph-api.md#graph-migrations). Phần còn lại của trang này trình bày các pattern bạn có thể áp dụng khi một thay đổi nằm ngoài tập được hỗ trợ đó.

## Tương thích kỹ thuật

Tương thích kỹ thuật tương đương với một breaking change API trong một microservice. "API" ở đây là hợp đồng giữa code graph của bạn và dữ liệu đã được [checkpointer](./checkpointers.md#checkpointer-libraries) persist cho các thread hiện có. Khi một thread resume, LangGraph deserialize state đã lưu, dispatch nó tới một node theo tên, và mong đợi node trả về các giá trị khớp với state schema.

Các lỗi phá vỡ tương thích kỹ thuật thường gặp:

* **Đổi tên hoặc xoá một node** trong khi các thread đang tạm dừng tại hoặc sắp vào node đó, ví dụ tại một [`interrupt`](https://reference.langchain.com/python/langgraph/types/interrupt) hoặc qua một conditional edge đã được checkpoint mà vẫn định tuyến tới tên cũ. Khi resume, LangGraph không tìm được node theo tên đã lưu và run sẽ thất bại. Điểm bắt đầu để resume một run là đầu hàm node nơi việc thực thi đã dừng, nên một node bị thiếu sẽ không có nơi nào để resume từ đó.
* **Đổi tên hoặc xoá một State key** mà các checkpoint cũ vẫn chứa hoặc các node phía sau vẫn đọc.
* **Siết chặt một trường State**, chẳng hạn biến một trường `Optional` thành bắt buộc, thu hẹp một kiểu dữ liệu, hoặc thêm một trường bắt buộc mới không có giá trị mặc định. Các checkpoint hiện có sẽ không thoả mãn schema mới.

Bản thân topology của cạnh (edge) *không* được persist trong checkpoint. Thêm, xoá, hoặc định tuyến lại các cạnh giữa các node vẫn còn tồn tại là an toàn cho các thread đang chạy dở. Theo tóm tắt [Graph migrations](./graph-api.md#graph-migrations), thay đổi topology duy nhất có thể phá vỡ một thread đang bị interrupt là đổi tên hoặc xoá một node.

### Pattern khuyến nghị

* Thêm trường state mới dạng `NotRequired` (hoặc `Optional[...] = None`) để các checkpoint cũ vẫn validate được:

  ```python
  from typing import NotRequired
  from typing_extensions import TypedDict

  class State(TypedDict):
      messages: list
      summary: NotRequired[str]  # [!code ++]
  ```

* Coi việc xoá bỏ như một sự deprecate. Giữ trường đó được định nghĩa trên state ít nhất qua một chu kỳ "rút cạn" (drain cycle), ngay cả khi không node nào đọc nó, để các checkpoint hiện có tiếp tục load được.

* Đổi tên theo kiểu *thêm rồi mới xoá*. Thêm trường hoặc node mới song song với cái cũ, ghi kép (dual-write) hoặc định tuyến tới cả hai trong một khoảng thời gian deprecate, rồi mới xoá cái cũ sau khi bạn đã xác nhận không còn thread nào đang chạy dở phụ thuộc vào nó.

* Giữ cho các hàm node dung nạp (tolerant) được các key không xác định. `TypedDict` bỏ qua các key thừa ở runtime, nên state còn sót lại từ một phiên bản code cũ sẽ không gây lỗi trừ khi một node đọc tường minh một key bị thiếu.

* Dùng [time travel](./use-time-travel.md) và [`graph.get_state`](https://reference.langchain.com/python/langgraph/graphs/#langgraph.graph.state.CompiledStateGraph.get_state) để kiểm tra nhanh (spot-check) các thread hiện có với code mới trên một deployment staging trước khi rollout.

### Phát hiện các thread đang chạy dở

Trước khi xoá một node, đổi tên một State key, hoặc thực hiện thay đổi khác mà các thread cũ không thể chịu được, bạn muốn biết liệu có thread nào đang "đỗ" (parked) trên phiên bản code mà bạn sắp gỡ bỏ hay không. Bản thân LangGraph không duy trì một index tìm kiếm trên state của thread, nên câu trả lời phụ thuộc vào nơi graph của bạn chạy.

**Nếu bạn deploy lên [LangSmith](https://docs.langchain.com/langsmith/deployment).** Dùng tính năng tìm kiếm thread của Agent Server để lọc theo status. Trường `status` chấp nhận `idle`, `busy`, `interrupted`, và `error`, nên bạn có thể truy vấn hàng loạt các thread `interrupted` hoặc `busy`, có thể thu hẹp thêm bằng bộ lọc metadata. Xem [Filter by thread status](https://docs.langchain.com/langsmith/use-threads#filter-by-thread-status) và [List threads](https://docs.langchain.com/langsmith/use-threads#list-threads).

**Ở bất kỳ đâu LangGraph chạy.** Dùng [LangSmith tracing](./observability.md) để theo dõi node nào đang được vào và ra trong production. Đây là tín hiệu đáng tin cậy nhất cho biết một node hoặc trường state không còn có thể chạm tới trong bất kỳ code path đang hoạt động nào.

**Khi bạn đã có sẵn một `thread_id`.** Kiểm tra trực tiếp thread đó:

* [`graph.get_state(config)`](https://reference.langchain.com/python/langgraph/graphs/#langgraph.graph.state.CompiledStateGraph.get_state) trả về checkpoint mới nhất, bao gồm node nào thread đang tạm dừng và các interrupt đang chờ.
* [`graph.get_state_history(config)`](https://reference.langchain.com/python/langgraph/graphs/#langgraph.graph.state.CompiledStateGraph.get_state_history) trả về danh sách đầy đủ theo thứ tự thời gian của các checkpoint cho thread.

Khi còn nghi ngờ, hãy giữ node hoặc trường đã deprecate cho tới khi cả danh sách thread của Agent Server lẫn tracing đều cho thấy không còn hoạt động nào trên nó nữa.

## Tương thích nghiệp vụ

Đôi khi một thay đổi hợp lệ về mặt kỹ thuật (mọi checkpoint hiện có vẫn load được và mọi node vẫn resolve được), nhưng *ý nghĩa* của graph mới lại khác với graph cũ. Hành vi mới là đúng đối với các thread mới, và bạn không muốn áp dụng nó hồi tố (retroactively) cho các thread đã bắt đầu dưới logic cũ.

Ví dụ, giả sử graph của bạn chạy `intake → triage → respond`, và bạn quyết định chèn thêm một bước `policy_check` mới giữa `triage` và `respond`:

* Các thread đã đi qua `triage` nên tiếp tục thẳng tới `respond` (luồng cũ).
* Các thread mới nên chạy toàn bộ luồng mới.

Pattern khuyến nghị là ghi lại *phiên bản hành vi* (behavioral version) liên quan vào state khi thread bắt đầu, sau đó rẽ nhánh dựa trên nó bằng một [conditional edge](./graph-api.md#conditional-edges):

```python
from typing import NotRequired
from typing_extensions import TypedDict

from langgraph.graph import END, START, StateGraph


class State(TypedDict):
    request: str
    flow_version: NotRequired[int]
    response: NotRequired[str]


def intake(state: State) -> dict:
    # Đánh dấu các thread mới với phiên bản luồng hiện tại. Các thread hiện có
    # resume qua `intake` giữ nguyên giá trị đã được lưu từ trước.
    return {"flow_version": state.get("flow_version", 2)}


def triage(state: State) -> dict: ...
def policy_check(state: State) -> dict: ...
def respond(state: State) -> dict: ...


def after_triage(state: State) -> str:
    if state.get("flow_version", 1) >= 2:
        return "policy_check"
    return "respond"


builder = StateGraph(State)
builder.add_node("intake", intake)
builder.add_node("triage", triage)
builder.add_node("policy_check", policy_check)
builder.add_node("respond", respond)
builder.add_edge(START, "intake")
builder.add_edge("intake", "triage")
builder.add_conditional_edges("triage", after_triage, ["policy_check", "respond"])
builder.add_edge("policy_check", "respond")
builder.add_edge("respond", END)

graph = builder.compile()
```

Các thread cũ resume sau `triage` đọc `flow_version` từ state đã lưu của chúng (hoặc rơi về giá trị mặc định v1) và bỏ qua `policy_check`. Các thread mới bắt đầu tại `intake`, được đánh dấu `flow_version=2`, và chạy đường đi mới. Khi tất cả thread v1 đã hoàn tất, bạn có thể xoá cờ phiên bản và conditional edge đó.

Pattern này chỉ hoạt động nếu bạn đặt phiên bản *ngay khi thread bắt đầu*, trước bất kỳ nhánh nào cần được version hoá. Đặt nó muộn hơn nghĩa là các thread hiện có sẽ không có giá trị này khi chúng cần.

## Tính không xác định

Nhóm này chỉ áp dụng cho [Functional API](./functional-api.md) và cho các lệnh gọi [**task**](./functional-api.md#task) hoặc [`interrupt`](https://reference.langchain.com/python/langgraph/types/interrupt) bên trong một **node** của [Graph API](./graph-api.md). Các **node** thuần của Graph API [chạy lại từ đầu hàm node](./graph-api.md#re-execution-and-idempotency) khi resume; hãy thiết kế side effect sao cho idempotent, nhưng bạn không cần bảo toàn thứ tự gọi task trừ khi bạn dùng **task** hoặc [`interrupt`](https://reference.langchain.com/python/langgraph/types/interrupt) trong **node** đó.

Một **entrypoint** của Functional API biên dịch thành một **node** duy nhất, node này replay lại phần thân entrypoint từ đầu khi một run resume, dùng các kết quả [`@task`](https://reference.langchain.com/python/langgraph/func/task) đã cache để bỏ qua công việc đã làm rồi. Hai loại thay đổi phá vỡ mô hình này:

* **Thêm, xoá, hoặc đổi thứ tự các lệnh gọi `@task` hoặc [`interrupt`](https://reference.langchain.com/python/langgraph/types/interrupt)** xuất hiện *trước* điểm resume. LangGraph khớp các kết quả đã cache và các giá trị resume với lệnh gọi dựa trên vị trí của chúng trong lượt replay, nên việc dịch chuyển vị trí đó có thể khiến giá trị cache sai bị replay nhầm vào một lệnh gọi khác.
* **Đưa vào các thao tác không xác định (non-deterministic) bên ngoài một `@task`**, chẳng hạn `time.time()`, `random.random()`, hoặc một cuộc gọi mạng được viết inline trong phần thân entrypoint. Khi replay, chúng cho ra giá trị khác với lần chạy đầu tiên, điều này có thể làm thay đổi control flow.

Để hiểu sâu hơn kèm ví dụ, xem [Determinism](./functional-api.md#determinism) và [Common pitfalls](./functional-api.md#common-pitfalls) trong hướng dẫn Functional API.

Nếu bạn cần thực hiện các thay đổi code không tầm thường (non-trivial) cho một `@entrypoint` đang có run chạy dở, các lựa chọn an toàn nhất là:

* Để các run đang chạy dở "rút cạn" (drain) trước khi deploy thay đổi.
* Bọc logic mới bất kỳ trong một `@task` mới để kết quả của nó được checkpoint độc lập.
* Đăng ký một entrypoint mới dưới một tên graph mới trong `langgraph.json` cho hành vi mới, và định tuyến các thread mới tới đó.

---

!!! info "Kết nối tài liệu này"
    [Kết nối tài liệu này](https://docs.langchain.com/use-these-docs) với Claude, VSCode, và nhiều công cụ khác qua MCP để có câu trả lời theo thời gian thực.

!!! info "Đóng góp"
    [Chỉnh sửa trang này trên GitHub](https://github.com/langchain-ai/docs/edit/main/src/oss/langgraph/backward-compatibility.mdx) hoặc [báo lỗi](https://github.com/langchain-ai/docs/issues/new/choose).
