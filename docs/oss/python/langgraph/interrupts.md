# Interrupts

Interrupt cho phép bạn tạm dừng việc thực thi graph tại các điểm cụ thể và chờ input từ bên ngoài trước khi tiếp tục. Điều này giúp hiện thực các pattern human-in-the-loop, nơi bạn cần input từ bên ngoài để tiếp tục. Khi một interrupt được kích hoạt, LangGraph lưu state của graph bằng tầng [persistence](./persistence.md) và chờ vô thời hạn cho tới khi bạn resume việc thực thi.

Interrupt hoạt động bằng cách gọi hàm `interrupt()` tại bất kỳ điểm nào trong node của graph. Hàm này chấp nhận bất kỳ giá trị JSON-serializable nào, giá trị đó sẽ được lộ ra cho caller. Khi bạn sẵn sàng tiếp tục, bạn resume việc thực thi bằng cách re-invoke graph dùng `Command`, giá trị này sau đó trở thành giá trị trả về của lệnh gọi `interrupt()` bên trong node.

Khác với breakpoint tĩnh (pause trước hoặc sau các node cụ thể), interrupt là **động (dynamic)**: chúng có thể được đặt ở bất kỳ đâu trong code của bạn và có thể có điều kiện dựa trên logic ứng dụng của bạn.

* **Checkpointing giữ vị trí của bạn:** checkpointer ghi lại chính xác state của graph để bạn có thể resume sau này, kể cả khi đang ở trạng thái lỗi.
* **`thread_id` là con trỏ của bạn:** đặt `config={"configurable": {"thread_id": ...}}` để báo cho checkpointer biết cần load state nào.
* **Payload của interrupt lộ ra qua `stream.interrupts`:** khi dùng [event streaming](./event-streaming.md) (`graph.stream_events(..., version="v3")`), các giá trị bạn truyền vào `interrupt()` sẽ xuất hiện trên `stream.interrupts`, và `stream.interrupted` là `True` khi run tạm dừng chờ input.

`thread_id` bạn chọn thực chất là con trỏ bền vững (persistent cursor) của bạn. Dùng lại nó sẽ resume cùng một checkpoint; dùng một giá trị mới sẽ bắt đầu một thread hoàn toàn mới với state rỗng.

## Tạm dừng bằng `interrupt`

Hàm [`interrupt`](https://reference.langchain.com/python/langgraph/types/interrupt) tạm dừng việc thực thi graph và trả về một giá trị cho caller. Khi bạn gọi [`interrupt`](https://reference.langchain.com/python/langgraph/types/interrupt) bên trong một node, LangGraph lưu state hiện tại của graph và chờ bạn resume việc thực thi kèm input.

Để dùng [`interrupt`](https://reference.langchain.com/python/langgraph/types/interrupt), bạn cần:

1. Một **checkpointer** để persist state của graph (dùng một checkpointer bền vững trong production)
2. Một **thread ID** trong config để runtime biết cần resume từ state nào
3. Gọi `interrupt()` ở nơi bạn muốn tạm dừng (payload phải là JSON-serializable)

```python
from langgraph.types import interrupt

def approval_node(state: State):
    # Tạm dừng và hỏi phê duyệt
    approved = interrupt("Do you approve this action?")

    # Khi bạn resume, Command(resume=...) trả về giá trị đó tại đây
    return {"approved": approved}
```

Khi bạn gọi [`interrupt`](https://reference.langchain.com/python/langgraph/types/interrupt), đây là những gì xảy ra:

1. **Việc thực thi graph bị treo (suspended)** đúng tại điểm gọi [`interrupt`](https://reference.langchain.com/python/langgraph/types/interrupt)

2. **State được lưu** bằng checkpointer để việc thực thi có thể được resume sau này. Trong production, đây nên là một checkpointer bền vững (ví dụ được backed bởi một database)

3. **Giá trị được trả về** cho caller trên `stream.interrupts` khi dùng [event streaming](./event-streaming.md) (`graph.stream_events(..., version="v3")`), hoặc dưới `__interrupt__` với API `invoke()` mặc định; nó có thể là bất kỳ giá trị JSON-serializable nào (string, object, array, v.v.)

4. **Graph chờ vô thời hạn** cho tới khi bạn resume việc thực thi với một phản hồi

5. **Phản hồi được truyền lại** vào node khi bạn resume, trở thành giá trị trả về của lệnh gọi `interrupt()`

## Resume interrupt

Sau khi một interrupt tạm dừng việc thực thi, bạn resume graph bằng cách invoke nó lại với một `Command` chứa giá trị resume. Giá trị resume được truyền lại cho lệnh gọi `interrupt`, cho phép node tiếp tục thực thi với input từ bên ngoài.

Cách được khuyến nghị để điều khiển một graph có thể interrupt là [event streaming](./event-streaming.md), nó lộ interrupt ra qua `stream.interrupts` và `stream.interrupted`, và expose state cuối cùng qua `stream.output`.

```python
from langgraph.types import Command

# Lần chạy đầu tiên - gặp interrupt và tạm dừng
# thread_id là con trỏ bền vững (lưu một ID ổn định trong production)
config = {"configurable": {"thread_id": "thread-1"}}
stream = graph.stream_events({"input": "data"}, config=config, version="v3")

# Xả (drain) stream để điều khiển run; stream.output chờ state cuối cùng.
final = stream.output

# stream.interrupted là True khi run tạm dừng chờ input con người, và
# stream.interrupts chứa các payload đã truyền cho interrupt().
if stream.interrupted:
    print(stream.interrupts)
    # > (Interrupt(value='Do you approve this action?'),)

# Resume với phản hồi của con người
# Payload resume trở thành giá trị trả về của interrupt() bên trong node
resumed = graph.stream_events(Command(resume=True), config=config, version="v3")
final = resumed.output
```

!!! note ""
    API `graph.invoke(...)` mặc định vẫn hoạt động và lộ interrupt ra dưới `result["__interrupt__"]`. Dùng nó khi bạn không cần các projection dạng stream; nếu không, hãy ưu tiên `graph.stream_events(..., version="v3")`.

**Điểm quan trọng cần nhớ khi resume:**

* Bạn phải dùng **cùng thread ID** khi resume đã được dùng khi interrupt xảy ra
* Giá trị truyền cho `Command(resume=...)` trở thành giá trị trả về của lệnh gọi [`interrupt`](https://reference.langchain.com/python/langgraph/types/interrupt)
* Node khởi động lại từ đầu hàm node nơi [`interrupt`](https://reference.langchain.com/python/langgraph/types/interrupt) được gọi khi resume, nên bất kỳ code nào trước [`interrupt`](https://reference.langchain.com/python/langgraph/types/interrupt) sẽ chạy lại
* Bạn có thể truyền bất kỳ giá trị JSON-serializable nào làm giá trị resume

!!! warning ""
    `Command(resume=...)` là pattern `Command` **duy nhất** được thiết kế để dùng làm input cho `invoke()`/`stream()`/`stream_events()`. Các tham số `Command` khác (`update`, `goto`, `graph`) được thiết kế để [trả về từ hàm node](./graph-api.md#command). Đừng truyền `Command(update=...)` làm input để tiếp tục cuộc hội thoại nhiều lượt, hãy truyền một dict input thông thường thay vào đó.

## Các pattern phổ biến

Điều then chốt mà interrupt mở khoá là khả năng tạm dừng việc thực thi và chờ input từ bên ngoài. Điều này hữu ích cho nhiều use case, bao gồm:

* [Workflow phê duyệt](#approve-or-reject): Tạm dừng trước khi thực thi các hành động quan trọng (gọi API, thay đổi database, giao dịch tài chính)
* [Xử lý nhiều interrupt](#handling-multiple-interrupts): Ghép ID interrupt với giá trị resume khi resume nhiều interrupt trong một lần invocation
* [Review và chỉnh sửa](#review-and-edit-state): Cho phép con người review và chỉnh sửa output của LLM hoặc tool call trước khi tiếp tục
* [Interrupt tool call](#interrupts-in-tools): Tạm dừng trước khi thực thi tool call để review và chỉnh sửa tool call trước khi thực thi
* [Xác thực input con người](#validating-human-input): Tạm dừng trước khi chuyển sang bước tiếp theo để xác thực input con người

### Streaming với interrupt human-in-the-loop (HITL)

Khi xây dựng agent tương tác với các workflow human-in-the-loop, bạn có thể dùng [event streaming](./event-streaming.md) để tiêu thụ (consume) các message chunk và state snapshot đồng thời trong khi xử lý interrupt.

Dùng các projection đã được gõ kiểu (typed) trả về bởi `graph.stream_events(..., version="v3")` trong một vòng lặp cho tới khi run kết thúc:

* Stream phản hồi AI theo từng token qua `stream.messages`
* Quan sát state snapshot theo từng step qua `stream.values`
* Phát hiện interrupt qua `stream.interrupted` và đọc payload của chúng từ `stream.interrupts`
* Resume việc thực thi bằng cách gọi lại `stream_events` với `Command(resume=...)` và lặp lại cho tới khi `stream.interrupted` là false

```python
from langgraph.types import Command

stream_input: dict | Command = initial_input

while True:
    stream = graph.stream_events(stream_input, config=config, version="v3")

    # Stream các chunk message của LLM (kể cả trong subgraph) khi chúng đến.
    for message in stream.messages:
        for token in message.text:
            display_streaming_content(token)

    # Sau khi run kết thúc (hoặc tạm dừng), kiểm tra interrupt và resume.
    if not stream.interrupted:
        final_state = stream.output
        break

    interrupt_info = stream.interrupts[0].value
    user_response = get_user_input(interrupt_info)
    stream_input = Command(resume=user_response)
```

* **`stream.messages`**: Output của chat model dưới dạng content block; lặp qua từng `message.text` để lấy token delta. Với subgraph lồng nhau, đọc message chunk từ `stream.subgraphs[*].messages`.
* **`stream.values`**: Snapshot state đầy đủ sau mỗi step
* **`stream.interrupted` / `stream.interrupts`**: Sau mỗi run, kiểm tra xem graph có tạm dừng không; đọc payload từ `stream.interrupts`
* **`Command(resume=...)`**: Truyền làm input `stream_events` tiếp theo để resume; lặp lại cho tới khi run hoàn thành mà không interrupt

### Xử lý nhiều interrupt

Khi các nhánh song song interrupt cùng lúc (ví dụ, fan-out tới nhiều node mà mỗi node đều gọi `interrupt()`), bạn có thể cần resume nhiều interrupt trong một lần invocation.
Khi resume nhiều interrupt bằng một lần invocation, hãy map từng interrupt ID với giá trị resume của nó.
Điều này đảm bảo mỗi phản hồi được ghép đúng với interrupt tương ứng ở runtime.

```python
from typing import Annotated, TypedDict
import operator

from langgraph.checkpoint.memory import InMemorySaver
from langgraph.graph import END, START, StateGraph
from langgraph.types import Command, interrupt


class State(TypedDict):
    vals: Annotated[list[str], operator.add]


def node_a(state):
    answer = interrupt("question_a")
    return {"vals": [f"a:{answer}"]}


def node_b(state):
    answer = interrupt("question_b")
    return {"vals": [f"b:{answer}"]}


graph = (
    StateGraph(State)
    .add_node("a", node_a)
    .add_node("b", node_b)
    .add_edge(START, "a")
    .add_edge(START, "b")
    .add_edge("a", END)
    .add_edge("b", END)
    .compile(checkpointer=InMemorySaver())
)

config = {"configurable": {"thread_id": "1"}}

# Bước 1: stream event để điều khiển run; cả hai node song song đều gặp interrupt() và tạm dừng
stream = graph.stream_events({"vals": []}, config, version="v3")
_ = stream.output  # điều khiển stream chạy tới khi hoàn tất
# stream.interrupts chứa các payload Interrupt đang chờ
print(stream.interrupts)
# > (Interrupt(value='question_a', id='...'), Interrupt(value='question_b', id='...'))

# Bước 2: resume tất cả interrupt đang chờ cùng lúc
resume_map = {
    i.id: f"answer for {i.value}" for i in stream.interrupts
}
resumed = graph.stream_events(Command(resume=resume_map), config, version="v3")

print("Final state:", resumed.output)
# Final state: {'vals': ['a:answer for question_a', 'b:answer for question_b']}
```

### Phê duyệt hoặc từ chối

Một trong những công dụng phổ biến nhất của interrupt là tạm dừng trước một hành động quan trọng và hỏi phê duyệt. Ví dụ, bạn có thể muốn hỏi con người phê duyệt một API call, một thay đổi database, hoặc bất kỳ quyết định quan trọng nào khác.

```python
from typing import Literal
from langgraph.types import interrupt, Command

def approval_node(state: State) -> Command[Literal["proceed", "cancel"]]:
    # Tạm dừng thực thi; payload xuất hiện trên stream.interrupts (với stream_events) hoặc result["__interrupt__"] (với invoke)
    is_approved = interrupt({
        "question": "Do you want to proceed with this action?",
        "details": state["action_details"]
    })

    # Định tuyến dựa trên phản hồi
    if is_approved:
        return Command(goto="proceed")  # Chạy sau khi payload resume được cung cấp
    else:
        return Command(goto="cancel")
```

Khi bạn resume graph, truyền `True` để phê duyệt hoặc `False` để từ chối:

```python
# Để phê duyệt
graph.stream_events(Command(resume=True), config=config, version="v3").output

# Để từ chối
graph.stream_events(Command(resume=False), config=config, version="v3").output
```

??? note "Ví dụ đầy đủ"
    ```python
    from typing import Literal, Optional, TypedDict

    from langgraph.checkpoint.memory import InMemorySaver
    from langgraph.graph import END, START, StateGraph
    from langgraph.types import Command, interrupt


    class ApprovalState(TypedDict):
        action_details: str
        status: Optional[Literal["pending", "approved", "rejected"]]


    def approval_node(state: ApprovalState) -> Command[Literal["proceed", "cancel"]]:
        # Expose các chi tiết để caller có thể render chúng trong UI
        decision = interrupt(
            {
                "question": "Approve this action?",
                "details": state["action_details"],
            }
        )

        # Định tuyến tới node phù hợp sau khi resume
        return Command(goto="proceed" if decision else "cancel")


    def proceed_node(state: ApprovalState):
        return {"status": "approved"}


    def cancel_node(state: ApprovalState):
        return {"status": "rejected"}


    builder = StateGraph(ApprovalState)
    builder.add_node("approval", approval_node)
    builder.add_node("proceed", proceed_node)
    builder.add_node("cancel", cancel_node)
    builder.add_edge(START, "approval")
    builder.add_edge("proceed", END)
    builder.add_edge("cancel", END)

    # Dùng một checkpointer bền vững hơn trong production
    checkpointer = InMemorySaver()
    graph = builder.compile(checkpointer=checkpointer)

    config = {"configurable": {"thread_id": "approval-123"}}
    initial = graph.stream_events(
        {"action_details": "Transfer $500", "status": "pending"},
        config=config,
        version="v3",
    )
    _ = initial.output  # điều khiển stream chạy tới khi hoàn tất
    print(initial.interrupts)  # -> (Interrupt(value={'question': ..., 'details': ...}),)

    # Resume với quyết định; True định tuyến tới proceed, False tới cancel
    resumed = graph.stream_events(Command(resume=True), config=config, version="v3")
    print(resumed.output["status"])
    ```

### Review và chỉnh sửa state

Đôi khi bạn muốn để con người review và chỉnh sửa một phần của state graph trước khi tiếp tục. Điều này hữu ích để sửa lỗi cho LLM, bổ sung thông tin còn thiếu, hoặc thực hiện các điều chỉnh.

```python
from langgraph.types import interrupt

def review_node(state: State):
    # Tạm dừng và hiển thị nội dung hiện tại để review (payload xuất hiện trên stream.interrupts)
    edited_content = interrupt({
        "instruction": "Review and edit this content",
        "content": state["generated_text"]
    })

    # Cập nhật state với phiên bản đã chỉnh sửa
    return {"generated_text": edited_content}
```

Khi resume, cung cấp nội dung đã chỉnh sửa:

```python
graph.stream_events(
    Command(resume="The edited and improved text"),  # Giá trị trở thành giá trị trả về từ interrupt()
    config=config,
    version="v3",
).output
```

??? note "Ví dụ đầy đủ"
    ```python
    from typing import TypedDict

    from langgraph.checkpoint.memory import MemorySaver
    from langgraph.graph import END, START, StateGraph
    from langgraph.types import Command, interrupt


    class ReviewState(TypedDict):
        generated_text: str


    def review_node(state: ReviewState):
        # Yêu cầu một reviewer chỉnh sửa nội dung đã sinh ra
        updated = interrupt(
            {
                "instruction": "Review and edit this content",
                "content": state["generated_text"],
            }
        )
        return {"generated_text": updated}


    builder = StateGraph(ReviewState)
    builder.add_node("review", review_node)
    builder.add_edge(START, "review")
    builder.add_edge("review", END)

    checkpointer = MemorySaver()
    graph = builder.compile(checkpointer=checkpointer)

    config = {"configurable": {"thread_id": "review-42"}}
    initial = graph.stream_events(
        {"generated_text": "Initial draft"}, config=config, version="v3"
    )
    _ = initial.output  # điều khiển stream chạy tới khi hoàn tất
    print(initial.interrupts)  # -> (Interrupt(value={'instruction': ..., 'content': ...}),)

    # Resume với văn bản đã chỉnh sửa từ reviewer
    final_state = graph.stream_events(
        Command(resume="Improved draft after review"),
        config=config,
        version="v3",
    )
    print(final_state.output["generated_text"])  # -> "Improved draft after review"
    ```

### Interrupt trong tool

Bạn cũng có thể đặt interrupt trực tiếp bên trong hàm tool. Điều này khiến chính bản thân tool tạm dừng chờ phê duyệt mỗi khi nó được gọi, và cho phép con người review và chỉnh sửa tool call trước khi nó được thực thi.

Đầu tiên, định nghĩa một tool dùng [`interrupt`](https://reference.langchain.com/python/langgraph/types/interrupt):

```python
from langchain.tools import tool
from langgraph.types import interrupt

@tool
def send_email(to: str, subject: str, body: str):
    """Send an email to a recipient."""

    # Tạm dừng trước khi gửi; payload xuất hiện trên stream.interrupts khi dùng event streaming
    response = interrupt({
        "action": "send_email",
        "to": to,
        "subject": subject,
        "body": body,
        "message": "Approve sending this email?"
    })

    if response.get("action") == "approve":
        # Giá trị resume có thể ghi đè input trước khi thực thi
        final_to = response.get("to", to)
        final_subject = response.get("subject", subject)
        final_body = response.get("body", body)
        return f"Email sent to {final_to} with subject '{final_subject}'"
    return "Email cancelled by user"
```

Cách tiếp cận này hữu ích khi bạn muốn logic phê duyệt gắn liền với chính tool đó, giúp nó có thể tái sử dụng ở nhiều phần khác nhau trong graph của bạn. LLM có thể gọi tool một cách tự nhiên, và interrupt sẽ tạm dừng việc thực thi mỗi khi tool được invoke, cho phép bạn phê duyệt, chỉnh sửa, hoặc huỷ hành động.

??? note "Ví dụ đầy đủ"
    ```python
    import sqlite3
    import operator
    from typing import TypedDict, Annotated, Literal
    from langchain.tools import tool
    from langchain_anthropic import ChatAnthropic
    from langgraph.checkpoint.sqlite import SqliteSaver
    from langgraph.graph import StateGraph, START, END
    from langgraph.types import Command, interrupt
    from langchain.messages import AnyMessage, SystemMessage, ToolMessage


    class AgentState(TypedDict):
        messages: Annotated[list[AnyMessage], operator.add]


    @tool
    def send_email(to: str, subject: str, body: str):
        """Send an email to a recipient."""

        # Tạm dừng trước khi gửi; payload xuất hiện trên stream.interrupts khi dùng event streaming
        response = interrupt({
            "action": "send_email",
            "to": to,
            "subject": subject,
            "body": body,
            "message": "Approve sending this email?",
        })

        if response.get("action") == "approve":
            final_to = response.get("to", to)
            final_subject = response.get("subject", subject)
            final_body = response.get("body", body)

            # Thực sự gửi email (phần triển khai của bạn ở đây)
            print(f"[send_email] to={final_to} subject={final_subject} body={final_body}")
            return f"Email sent to {final_to}"

        return "Email cancelled by user"


    model = ChatAnthropic(model="claude-sonnet-4-6").bind_tools([send_email])
    tools_by_name = {"send_email": send_email}


    def agent_node(state: AgentState):
        # LLM có thể quyết định gọi tool; interrupt tạm dừng trước khi gửi
        result = model.invoke(state["messages"])
        return {"messages": [result]}

    def tool_node(state: AgentState):
        """Thực hiện tool call"""
        result = []
        for tool_call in state["messages"][-1].tool_calls:
            tool = tools_by_name[tool_call["name"]]
            observation = tool.invoke(tool_call["args"])
            result.append(ToolMessage(content=observation, tool_call_id=tool_call["id"]))
        return {"messages": result}

    def should_continue(state: AgentState) -> Literal["tool_node", END]:
        """Quyết định xem có nên tiếp tục vòng lặp hay dừng dựa trên việc LLM có gọi tool hay không"""
        messages = state["messages"]
        last_message = messages[-1]

        if last_message.tool_calls:
            return "tool_node"
        return END

    builder = StateGraph(AgentState)
    builder.add_node("agent", agent_node)
    builder.add_node("tool_node", tool_node)

    builder.add_edge(START, "agent")
    builder.add_conditional_edges("agent", should_continue, ["tool_node", END])  # Định tuyến tới "tools" hoặc END
    builder.add_edge("tool_node", "agent")  # Vòng lại sau khi dùng tool

    checkpointer = SqliteSaver(
        sqlite3.connect("tool-approval.db", check_same_thread=False)
    )
    graph = builder.compile(checkpointer=checkpointer)

    config = {"configurable": {"thread_id": "email-workflow"}}
    initial = graph.stream_events(
        {
            "messages": [
                {"role": "user", "content": "Send an email to alice@example.com about the meeting"}
            ]
        },
        config=config,
        version="v3",
    )
    initial.output  # điều khiển stream chạy tới khi hoàn tất
    print(initial.interrupts)  # -> (Interrupt(value={'action': 'send_email', ...}),)

    # Resume với phê duyệt và tuỳ chọn chỉnh sửa tham số
    resumed = graph.stream_events(
        Command(resume={"action": "approve", "subject": "Updated subject"}),
        config=config,
        version="v3",
    )
    print(resumed.output["messages"][-1])  # -> Kết quả tool do send_email trả về
    ```

### Xác thực input con người

Đôi khi bạn cần xác thực input từ con người và hỏi lại nếu giá trị không hợp lệ. Cách tiếp cận được khuyến nghị là gọi `interrupt()` **một lần cho mỗi lần invoke node**, trả về từ node với message lỗi được lưu trong state, và dùng một **conditional edge** để vòng lại node cho tới khi một giá trị hợp lệ được cung cấp.

!!! warning ""
    **Tránh vòng lặp `while True` + `interrupt()` bên trong một node duy nhất.** Vì node chạy lại từ đầu ở mỗi lần resume (xem [Quy tắc của interrupt](#rules-of-interrupts)), một vòng lặp gọi `interrupt()` nhiều lần khiến mỗi lần resume replay lại tất cả các lượt lặp trước đó: lần resume đầu tiên replay 1 lượt, lần thứ hai replay 2 lượt, và cứ thế. Kết quả là việc thực thi lại theo cấp số nhân (exponential re-execution) của bất kỳ code nào bên trong thân vòng lặp.

Pattern đúng:

1. Lưu câu hỏi hỏi lại vào state (ví dụ `pending_question`).
2. Trong node, gọi `interrupt()` **đúng một lần**, truyền câu hỏi hiện tại từ state.
3. Nếu câu trả lời không hợp lệ, trả về `pending_question` đã cập nhật để lần invoke tiếp theo hỏi lại.
4. Dùng `add_conditional_edges` để định tuyến quay lại node cho tới khi thu thập được một giá trị hợp lệ.

```python
from typing import TypedDict

from langgraph.graph import END, START, StateGraph
from langgraph.types import interrupt


class FormState(TypedDict):
    age: int | None
    pending_question: str | None


def get_age_node(state: FormState):
    question = state.get("pending_question") or "What is your age?"
    answer = interrupt(question)  # được gọi đúng một lần cho mỗi lần invoke
    if isinstance(answer, int) and answer > 0:
        return {"age": answer, "pending_question": None}
    return {"pending_question": f"'{answer}' is not a valid age. Please enter a positive number."}


def route(state: FormState):
    return END if state.get("age") is not None else "collect_age"


builder = StateGraph(FormState)
builder.add_node("collect_age", get_age_node)
builder.add_edge(START, "collect_age")
builder.add_conditional_edges("collect_age", route)
```

Mỗi lần resume invoke `get_age_node` đúng một lần, chạy lệnh gọi `interrupt()` một lần, và thoát. Khi câu trả lời không hợp lệ, conditional edge vòng lại và interrupt tiếp theo hỏi lại với câu hỏi đã cập nhật. Không có code nào chạy nhiều hơn một lần cho mỗi lần resume.

??? note "Ví dụ đầy đủ"
    ```python
    from typing import TypedDict

    from langgraph.checkpoint.memory import InMemorySaver
    from langgraph.graph import END, START, StateGraph
    from langgraph.types import Command, interrupt


    class FormState(TypedDict):
        age: int | None
        pending_question: str | None


    def get_age_node(state: FormState):
        question = state.get("pending_question") or "What is your age?"
        answer = interrupt(question)  # được gọi đúng một lần cho mỗi lần invoke node
        print(f"I got {answer}")  # chạy đúng một lần cho mỗi lần resume
        if isinstance(answer, int) and answer > 0:
            return {"age": answer, "pending_question": None}
        return {"pending_question": f"'{answer}' is not a valid age. Please enter a positive number."}


    def route(state: FormState):
        # Vòng lại collect_age cho tới khi có một age hợp lệ
        return END if state.get("age") is not None else "collect_age"


    builder = StateGraph(FormState)
    builder.add_node("collect_age", get_age_node)
    builder.add_edge(START, "collect_age")
    builder.add_conditional_edges("collect_age", route)

    checkpointer = InMemorySaver()
    graph = builder.compile(checkpointer=checkpointer)

    config = {"configurable": {"thread_id": "form-1"}}
    first = graph.stream_events({"age": None, "pending_question": None}, config=config, version="v3")
    _ = first.output  # điều khiển stream chạy tới khi hoàn tất
    print(first.interrupts)  # -> (Interrupt(value='What is your age?', ...),)

    # Cung cấp dữ liệu không hợp lệ; node hỏi lại qua conditional edge
    retry = graph.stream_events(Command(resume="thirty"), config=config, version="v3")
    _ = retry.output
    print(retry.interrupts)  # -> (Interrupt(value="'thirty' is not a valid age...", ...),)

    # Cung cấp dữ liệu hợp lệ; route() trả về END và graph hoàn thành
    final = graph.stream_events(Command(resume=30), config=config, version="v3")
    print(final.output["age"])  # -> 30
    ```

## Quy tắc của interrupt

Khi bạn gọi [`interrupt`](https://reference.langchain.com/python/langgraph/types/interrupt) bên trong một node, LangGraph treo (suspend) việc thực thi bằng cách raise một exception báo hiệu cho runtime tạm dừng. Exception này lan truyền lên trên qua call stack và được runtime bắt lấy, runtime sau đó báo cho graph lưu state hiện tại và chờ input từ bên ngoài.

Khi việc thực thi resume (sau khi bạn cung cấp input được yêu cầu), runtime khởi động lại *toàn bộ* node từ đầu, nó không resume từ chính xác dòng nơi [`interrupt`](https://reference.langchain.com/python/langgraph/types/interrupt) được gọi. Điều này có nghĩa là bất kỳ code nào chạy trước [`interrupt`](https://reference.langchain.com/python/langgraph/types/interrupt) sẽ chạy lại. Vì lý do đó, có một vài quy tắc quan trọng cần tuân theo khi làm việc với interrupt để đảm bảo chúng hoạt động như mong đợi.

### Đừng bọc lệnh gọi `interrupt` trong try/except

Cách mà [`interrupt`](https://reference.langchain.com/python/langgraph/types/interrupt) tạm dừng việc thực thi tại điểm gọi là bằng cách throw một exception đặc biệt. Nếu bạn bọc lệnh gọi [`interrupt`](https://reference.langchain.com/python/langgraph/types/interrupt) trong một khối try/except, bạn sẽ bắt exception này và interrupt sẽ không được truyền lại cho graph.

* Tách các lệnh gọi [`interrupt`](https://reference.langchain.com/python/langgraph/types/interrupt) khỏi code dễ gây lỗi
* Dùng các loại exception cụ thể trong khối try/except

=== "Tách riêng logic"

    ```python
    def node_a(state: State):
        # ✅ Tốt: interrupt trước, rồi mới xử lý
        # điều kiện lỗi riêng biệt
        interrupt("What's your name?")
        try:
            fetch_data()  # Cái này có thể fail
        except Exception as e:
            print(e)
        return state
    ```

=== "Xử lý exception tường minh"

    ```python
    def node_a(state: State):
        # ✅ Tốt: bắt các loại exception cụ thể
        # sẽ không bắt exception của interrupt
        try:
            name = interrupt("What's your name?")
            fetch_data()  # Cái này có thể fail
        except NetworkException as e:
            print(e)
        return state
    ```

* Đừng bọc lệnh gọi [`interrupt`](https://reference.langchain.com/python/langgraph/types/interrupt) trong khối try/except trần (bare)

```python
def node_a(state: State):
    # ❌ Xấu: bọc interrupt trong try/except trần
    # sẽ bắt luôn exception của interrupt
    try:
        interrupt("What's your name?")
    except Exception as e:
        print(e)
    return state
```

### Đừng đổi thứ tự các lệnh gọi `interrupt` trong một node

Việc dùng nhiều interrupt trong một node là phổ biến, tuy nhiên điều này có thể dẫn tới hành vi không mong muốn nếu không xử lý cẩn thận.

Khi một node chứa nhiều lệnh gọi interrupt, LangGraph giữ một danh sách các giá trị resume dành riêng cho task đang thực thi node đó. Bất cứ khi nào việc thực thi resume, nó bắt đầu từ đầu node. Với mỗi interrupt gặp phải, LangGraph kiểm tra xem có giá trị khớp trong danh sách resume của task hay không. Việc khớp là **hoàn toàn dựa trên index**, vì vậy thứ tự các lệnh gọi interrupt trong node rất quan trọng.

* Giữ các lệnh gọi [`interrupt`](https://reference.langchain.com/python/langgraph/types/interrupt) nhất quán qua các lần thực thi node

```python
def node_a(state: State):
    # ✅ Tốt: các lệnh gọi interrupt xảy ra theo cùng thứ tự mỗi lần
    name = interrupt("What's your name?")
    age = interrupt("What's your age?")
    city = interrupt("What's your city?")

    return {
        "name": name,
        "age": age,
        "city": city
    }
```

* Đừng bỏ qua lệnh gọi [`interrupt`](https://reference.langchain.com/python/langgraph/types/interrupt) có điều kiện bên trong một node
* Đừng lặp lệnh gọi [`interrupt`](https://reference.langchain.com/python/langgraph/types/interrupt) bằng logic không xác định (non-deterministic) qua các lần thực thi, bao gồm cả vòng lặp xác thực `while True`. Hãy dùng một conditional edge thay vào đó (xem [Xác thực input con người](#validating-human-input))

=== "Bỏ qua interrupt"

    ```python
    def node_a(state: State):
        # ❌ Xấu: bỏ qua interrupt có điều kiện làm thay đổi thứ tự
        name = interrupt("What's your name?")

        # Ở lần chạy đầu, cái này có thể bỏ qua interrupt
        # Khi resume, có thể lại không bỏ qua - gây lệch index
        if state.get("needs_age"):
            age = interrupt("What's your age?")

        city = interrupt("What's your city?")

        return {"name": name, "city": city}
    ```

=== "Lặp interrupt"

    ```python
    def node_a(state: State):
        # ❌ Xấu: lặp dựa trên dữ liệu không xác định
        # Số lượng interrupt thay đổi giữa các lần thực thi
        results = []
        for item in state.get("dynamic_list", []):  # Danh sách có thể đổi giữa các lần chạy
            result = interrupt(f"Approve {item}?")
            results.append(result)

        return {"results": results}
    ```

### Đừng trả về giá trị phức tạp trong lệnh gọi `interrupt`

Tuỳ vào checkpointer nào được dùng, các giá trị phức tạp có thể không serialize được (ví dụ bạn không thể serialize một hàm). Để graph của bạn thích ứng được với mọi deployment, best practice là chỉ dùng các giá trị có thể serialize một cách hợp lý.

* Truyền các kiểu đơn giản, JSON-serializable cho [`interrupt`](https://reference.langchain.com/python/langgraph/types/interrupt)
* Truyền dictionary/object với các giá trị đơn giản

=== "Giá trị đơn giản"

    ```python
    def node_a(state: State):
        # ✅ Tốt: truyền các kiểu đơn giản có thể serialize
        name = interrupt("What's your name?")
        count = interrupt(42)
        approved = interrupt(True)

        return {"name": name, "count": count, "approved": approved}
    ```

=== "Dữ liệu có cấu trúc"

    ```python
    def node_a(state: State):
        # ✅ Tốt: truyền dictionary với các giá trị đơn giản
        response = interrupt({
            "question": "Enter user details",
            "fields": ["name", "email", "age"],
            "current_values": state.get("user", {})
        })

        return {"user": response}
    ```

* Đừng truyền hàm, instance class, hoặc các đối tượng phức tạp khác cho [`interrupt`](https://reference.langchain.com/python/langgraph/types/interrupt)

=== "Hàm"

    ```python
    def validate_input(value):
        return len(value) > 0

    def node_a(state: State):
        # ❌ Xấu: truyền một hàm vào interrupt
        # Hàm này không thể serialize
        response = interrupt({
            "question": "What's your name?",
            "validator": validate_input  # Cái này sẽ fail
        })
        return {"name": response}
    ```

=== "Instance class"

    ```python
    class DataProcessor:
        def __init__(self, config):
            self.config = config

    def node_a(state: State):
        processor = DataProcessor({"mode": "strict"})

        # ❌ Xấu: truyền một instance class vào interrupt
        # Instance này không thể serialize
        response = interrupt({
            "question": "Enter data to process",
            "processor": processor  # Cái này sẽ fail
        })
        return {"result": response}
    ```

### Side effect được gọi trước `interrupt` phải idempotent

Vì interrupt hoạt động bằng cách chạy lại các node mà chúng được gọi từ đó, các side effect được gọi trước [`interrupt`](https://reference.langchain.com/python/langgraph/types/interrupt) nên (lý tưởng nhất là) idempotent. Nói thêm về ngữ cảnh, idempotent nghĩa là cùng một thao tác có thể được áp dụng nhiều lần mà không làm thay đổi kết quả so với lần thực thi đầu tiên.

Ví dụ, bạn có thể có một API call để cập nhật một bản ghi bên trong một node. Nếu [`interrupt`](https://reference.langchain.com/python/langgraph/types/interrupt) được gọi sau khi lệnh gọi đó được thực hiện, nó sẽ chạy lại nhiều lần khi node resume, có thể ghi đè lên bản cập nhật ban đầu hoặc tạo ra các bản ghi trùng lặp.

* Dùng các thao tác idempotent trước [`interrupt`](https://reference.langchain.com/python/langgraph/types/interrupt)
* Đặt side effect sau các lệnh gọi [`interrupt`](https://reference.langchain.com/python/langgraph/types/interrupt)
* Tách side effect ra các node riêng biệt khi có thể

=== "Thao tác idempotent"

    ```python
    def node_a(state: State):
        # ✅ Tốt: dùng thao tác upsert vốn idempotent
        # Chạy nhiều lần cái này vẫn cho cùng kết quả
        db.upsert_user(
            user_id=state["user_id"],
            status="pending_approval"
        )

        approved = interrupt("Approve this change?")

        return {"approved": approved}
    ```

=== "Side effect sau interrupt"

    ```python
    def node_a(state: State):
        # ✅ Tốt: đặt side effect sau interrupt
        # Điều này đảm bảo nó chỉ chạy một lần sau khi nhận được phê duyệt
        approved = interrupt("Approve this change?")

        if approved:
            db.create_audit_log(
                user_id=state["user_id"],
                action="approved"
            )

        return {"approved": approved}
    ```

=== "Tách ra các node khác nhau"

    ```python
    def approval_node(state: State):
        # ✅ Tốt: chỉ xử lý interrupt trong node này
        approved = interrupt("Approve this change?")

        return {"approved": approved}

    def notification_node(state: State):
        # ✅ Tốt: side effect xảy ra ở một node riêng
        # Cái này chạy sau khi phê duyệt, nên nó chỉ thực thi một lần
        if (state.approved):
            send_notification(
                user_id=state["user_id"],
                status="approved"
            )

        return state
    ```

* Đừng thực hiện các thao tác không idempotent trước [`interrupt`](https://reference.langchain.com/python/langgraph/types/interrupt)
* Đừng tạo bản ghi mới mà không kiểm tra xem nó đã tồn tại chưa

=== "Tạo bản ghi"

    ```python
    def node_a(state: State):
        # ❌ Xấu: tạo một bản ghi mới trước interrupt
        # Cái này sẽ tạo bản ghi trùng lặp ở mỗi lần resume
        audit_id = db.create_audit_log({
            "user_id": state["user_id"],
            "action": "pending_approval",
            "timestamp": datetime.now()
        })

        approved = interrupt("Approve this change?")

        return {"approved": approved, "audit_id": audit_id}
    ```

=== "Nối vào danh sách"

    ```python
    def node_a(state: State):
        # ❌ Xấu: nối vào một danh sách trước interrupt
        # Cái này sẽ thêm mục nhập trùng lặp ở mỗi lần resume
        db.append_to_history(state["user_id"], "approval_requested")

        approved = interrupt("Approve this change?")

        return {"approved": approved}
    ```

## Dùng với subgraph được gọi như hàm

Khi invoke một subgraph bên trong một node, graph cha sẽ resume việc thực thi từ **đầu node** nơi subgraph được invoke và [`interrupt`](https://reference.langchain.com/python/langgraph/types/interrupt) được kích hoạt. Tương tự, **subgraph** cũng sẽ resume từ đầu node nơi [`interrupt`](https://reference.langchain.com/python/langgraph/types/interrupt) được gọi.

```python
def node_in_parent_graph(state: State):
    some_code()  # <-- Cái này sẽ chạy lại khi resume
    # Invoke một subgraph như một hàm.
    # Subgraph chứa một lệnh gọi `interrupt`.
    subgraph_result = subgraph.invoke(some_input)
    # ...

def node_in_subgraph(state: State):
    some_other_code()  # <-- Cái này cũng sẽ chạy lại khi resume
    result = interrupt("What's your name?")
    # ...
```

## Debug với interrupt

Để debug và test một graph, bạn có thể dùng static interrupt như breakpoint để bước qua từng node của việc thực thi graph. Static interrupt được kích hoạt tại các điểm đã định nghĩa trước hoặc sau khi một node thực thi. Bạn có thể đặt chúng bằng cách chỉ định `interrupt_before` và `interrupt_after` khi compile graph.

!!! note ""
    Static interrupt **không** được khuyến nghị cho các workflow human-in-the-loop. Hãy dùng hàm [`interrupt`](https://reference.langchain.com/python/langgraph/types/interrupt) thay vào đó.

=== "Tại thời điểm compile"

    ```python
    graph = builder.compile(
        interrupt_before=["node_a"],  # [!code highlight]
        interrupt_after=["node_b", "node_c"],  # [!code highlight]
        checkpointer=checkpointer,
    )

    # Truyền một thread ID cho graph
    config = {
        "configurable": {
            "thread_id": "some_thread"
        }
    }

    # Chạy graph cho tới breakpoint
    graph.invoke(inputs, config=config)  # [!code highlight]

    # Resume graph
    graph.invoke(None, config=config)  # [!code highlight]
    ```

    1. Các breakpoint được đặt ở thời điểm `compile`.
    2. `interrupt_before` chỉ định các node nơi việc thực thi nên tạm dừng trước khi node đó được thực thi.
    3. `interrupt_after` chỉ định các node nơi việc thực thi nên tạm dừng sau khi node đó được thực thi.
    4. Một checkpointer là bắt buộc để bật breakpoint.
    5. Graph chạy cho tới khi gặp breakpoint đầu tiên.
    6. Graph được resume bằng cách truyền `None` làm input. Việc này sẽ chạy graph cho tới khi gặp breakpoint tiếp theo.

=== "Tại thời điểm chạy"

    ```python
    config = {
        "configurable": {
            "thread_id": "some_thread"
        }
    }

    # Chạy graph cho tới breakpoint
    graph.invoke(
        inputs,
        interrupt_before=["node_a"],  # [!code highlight]
        interrupt_after=["node_b", "node_c"],  # [!code highlight]
        config=config,
    )

    # Resume graph
    graph.invoke(None, config=config)  # [!code highlight]
    ```

    1. `graph.invoke` được gọi với tham số `interrupt_before` và `interrupt_after`. Đây là cấu hình tại thời điểm chạy (run-time) và có thể thay đổi cho mỗi lần invoke.
    2. `interrupt_before` chỉ định các node nơi việc thực thi nên tạm dừng trước khi node đó được thực thi.
    3. `interrupt_after` chỉ định các node nơi việc thực thi nên tạm dừng sau khi node đó được thực thi.
    4. Graph chạy cho tới khi gặp breakpoint đầu tiên.
    5. Graph được resume bằng cách truyền `None` làm input. Việc này sẽ chạy graph cho tới khi gặp breakpoint tiếp theo.

!!! tip ""
    Để debug các interrupt của bạn, hãy dùng [LangSmith](https://docs.langchain.com/langsmith/observability).

### Dùng LangSmith Studio

Bạn có thể dùng [LangSmith Studio](https://docs.langchain.com/langsmith/studio) để đặt static interrupt trong graph của bạn ngay trên UI trước khi chạy graph. Bạn cũng có thể dùng UI để kiểm tra state của graph tại bất kỳ điểm nào trong quá trình thực thi.

<img src="https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/static-interrupt.png?fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=5aa4e7cea2ab147cef5b4e210dd6c4a1" alt="image" width="1252" height="1040" data-path="oss/images/static-interrupt.png" />

---

!!! info "Kết nối tài liệu này"
    [Kết nối tài liệu này](https://docs.langchain.com/use-these-docs) với Claude, VSCode, và nhiều công cụ khác qua MCP để có câu trả lời theo thời gian thực.

!!! info "Đóng góp"
    [Chỉnh sửa trang này trên GitHub](https://github.com/langchain-ai/docs/edit/main/src/oss/langgraph/interrupts.mdx) hoặc [báo lỗi](https://github.com/langchain-ai/docs/issues/new/choose).
