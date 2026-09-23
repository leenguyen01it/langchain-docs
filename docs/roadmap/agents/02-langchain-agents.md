# Bài 2: Agent với LangChain

!!! abstract "Tóm tắt bài học"
    - **Mục tiêu:** xây trợ lý cửa hàng hoàn chỉnh bằng `create_agent`: tool có ngữ cảnh người dùng, bộ nhớ hội thoại, streaming, structured output, middleware (giới hạn, dự phòng, log chi phí) và phê duyệt của con người trước hành động nhạy cảm.
    - **Thời lượng:** khoảng 1 tuần.
    - **Yêu cầu trước:** [Bài 1](01-agent-loop.md).

## 1. Vì sao dùng framework?

Ở [Hiểu LLM, Bài 4](../hieu-llm/04-tool-use.md) bạn đã tự viết vòng lặp agent. Với sản phẩm thật, bạn còn cần: lưu và khôi phục hội thoại, streaming, giới hạn số lượt, dự phòng khi model lỗi, tạm dừng chờ con người duyệt, tóm tắt lịch sử dài, log và trace... `create_agent` của LangChain cung cấp sẵn những thứ này, xây trên nền LangGraph.

```bash
uv add langchain langchain-anthropic langgraph
```

## 2. Tool có ngữ cảnh người dùng

```python title="shop/tools.py"
from dataclasses import dataclass

from langchain.tools import ToolRuntime, tool


@dataclass
class Customer:
    """Thông tin phiên đăng nhập, do hệ thống xác thực cung cấp, không phải do LLM."""
    customer_id: str
    name: str


ORDERS = {
    "DH1024": {"customer_id": "KH01", "status": "Đang giao", "eta": "15/03", "total": 890_000},
    "DH1025": {"customer_id": "KH01", "status": "Đã giao", "total": 450_000},
    "DH2001": {"customer_id": "KH02", "status": "Đang giao", "total": 1_200_000},
}
PRODUCTS = {
    "ao-linen": "Áo sơ mi linen cổ tàu, 450.000đ, size S-XL, màu be/trắng/xanh rêu",
    "quan-kaki": "Quần kaki ống đứng, 520.000đ, size 29-34, màu be/đen",
}


@tool
def list_my_orders(runtime: ToolRuntime[Customer]) -> str:
    """Liệt kê các đơn hàng của khách đang chat, kèm trạng thái."""
    me = runtime.context.customer_id
    rows = [f"{oid}: {o['status']}, {o['total']:,}đ" for oid, o in ORDERS.items() if o["customer_id"] == me]
    return "\n".join(rows) or "Khách chưa có đơn hàng nào."


@tool
def get_order(order_id: str, runtime: ToolRuntime[Customer]) -> str:
    """Xem chi tiết một đơn hàng của khách đang chat.

    Args:
        order_id: Mã đơn hàng dạng DH + 4 chữ số, ví dụ DH1024.
    """
    order = ORDERS.get(order_id.upper())
    # Kiểm tra quyền: chỉ cho xem đơn của chính khách hàng này
    if order is None or order["customer_id"] != runtime.context.customer_id:
        return f"Không tìm thấy đơn {order_id} trong tài khoản của khách. Hãy hỏi lại mã đơn."
    return str(order)


@tool
def search_products(query: str) -> str:
    """Tìm sản phẩm trong cửa hàng theo tên, loại, màu sắc.

    Args:
        query: Từ khóa tìm kiếm, ví dụ "áo linen", "quần màu đen".
    """
    q = query.lower()
    hits = [desc for desc in PRODUCTS.values() if any(w in desc.lower() for w in q.split())]
    return "\n".join(hits) or "Không tìm thấy sản phẩm phù hợp."


@tool
def request_refund(order_id: str, reason: str, runtime: ToolRuntime[Customer]) -> str:
    """Tạo yêu cầu hoàn tiền cho một đơn hàng đã giao. Cần nhân viên phê duyệt.

    Args:
        order_id: Mã đơn hàng.
        reason: Lý do hoàn tiền theo lời khách.
    """
    order = ORDERS.get(order_id.upper())
    if order is None or order["customer_id"] != runtime.context.customer_id:
        return "Không tìm thấy đơn hàng này trong tài khoản của khách."
    if order["status"] != "Đã giao":
        return "Chỉ hoàn tiền cho đơn đã giao. Đơn này chưa giao xong."
    return f"Đã tạo yêu cầu hoàn tiền {order['total']:,}đ cho đơn {order_id}."
```

`runtime: ToolRuntime[Customer]` được LangChain **tự động truyền vào và ẩn khỏi LLM**: model không thấy tham số này trong schema, nên không thể giả mạo `customer_id` của người khác. Xem [Tools](../../oss/python/langchain/tools.md).

## 3. Tạo agent

```python title="shop/agent.py"
from langchain.agents import create_agent
from langchain.agents.middleware import (
    HumanInTheLoopMiddleware,
    ModelCallLimitMiddleware,
    ModelFallbackMiddleware,
    ToolCallLimitMiddleware,
)
from langgraph.checkpoint.memory import InMemorySaver

from shop.tools import Customer, get_order, list_my_orders, request_refund, search_products

SYSTEM_PROMPT = """Bạn là trợ lý chăm sóc khách hàng của Mộc, thương hiệu thời trang nam.
Xưng "Mộc", gọi khách là "bạn". Trả lời ngắn gọn, 2 đến 4 câu.

Dùng tool để tra cứu đơn hàng và sản phẩm; không đoán thông tin.
Khi khách muốn hoàn tiền, hỏi rõ mã đơn và lý do trước khi tạo yêu cầu.
Nếu khách bức xúc hoặc yêu cầu vượt quá khả năng của tool, đề nghị chuyển nhân viên."""

agent = create_agent(
    model="anthropic:claude-opus-5",
    tools=[list_my_orders, get_order, search_products, request_refund],
    system_prompt=SYSTEM_PROMPT,
    context_schema=Customer,
    checkpointer=InMemorySaver(),  # production: PostgresSaver
    middleware=[
        ModelCallLimitMiddleware(run_limit=10, exit_behavior="end"),
        ToolCallLimitMiddleware(run_limit=8),
        ModelFallbackMiddleware("anthropic:claude-sonnet-5"),
        HumanInTheLoopMiddleware(
            interrupt_on={"request_refund": {"allowed_decisions": ["approve", "edit", "reject"]}},
        ),
    ],
)
```

| Middleware | Tác dụng |
|---|---|
| `ModelCallLimitMiddleware` | Chặn agent gọi model quá 10 lần mỗi lượt, chống lặp vô tận |
| `ToolCallLimitMiddleware` | Giới hạn số lần gọi tool |
| `ModelFallbackMiddleware` | Model chính lỗi (quá tải, sự cố) thì chuyển sang model dự phòng |
| `HumanInTheLoopMiddleware` | Tạm dừng trước khi chạy `request_refund`, chờ nhân viên duyệt |

Danh sách đầy đủ ở [Middleware có sẵn](../../oss/python/langchain/middleware/built-in.md).

## 4. Chạy agent: bộ nhớ và streaming

```python title="chat.py"
import uuid

from shop.agent import agent
from shop.tools import Customer

config = {"configurable": {"thread_id": str(uuid.uuid4())}}  # một hội thoại = một thread
customer = Customer(customer_id="KH01", name="Minh")

for text in ["Đơn hàng của mình đến đâu rồi?", "Còn đơn kia thì sao?"]:
    print(f"\n>>> {text}")
    for token, meta in agent.stream(
        {"messages": [{"role": "user", "content": text}]},
        config=config,
        context=customer,
        stream_mode="messages",
    ):
        if meta.get("langgraph_node") == "model" and token.text:
            print(token.text, end="", flush=True)
```

- **Bộ nhớ ngắn hạn:** checkpointer lưu toàn bộ state theo `thread_id`. Lượt thứ hai hiểu "đơn kia" nhờ lịch sử lượt đầu. Xem [Short-term Memory](../../oss/python/langchain/short-term-memory.md).
- **Streaming:** `stream_mode="messages"` stream từng token; lọc theo `langgraph_node == "model"` để bỏ qua output của tool. Xem [Streaming](../../oss/python/langchain/streaming.md).
- **Bảo mật thread:** trong production, ghép `customer_id` vào `thread_id` để một khách không thể đọc hội thoại của khách khác bằng cách đoán thread ID.

## 5. Human-in-the-loop: duyệt hoàn tiền

Khi agent muốn gọi `request_refund`, nó **tạm dừng** và trả về thông tin cần duyệt. Nhân viên xem và quyết định; agent tiếp tục từ đúng chỗ đã dừng.

```python title="refund_flow.py"
import uuid

from langgraph.types import Command

from shop.agent import agent
from shop.tools import Customer

config = {"configurable": {"thread_id": str(uuid.uuid4())}}
customer = Customer(customer_id="KH01", name="Minh")

result = agent.invoke(
    {"messages": [{"role": "user", "content": "Áo đơn DH1025 bị lỗi đường may, mình muốn hoàn tiền"}]},
    config=config,
    context=customer,
    version="v2",
)

if result.interrupts:
    request = result.interrupts[0].value["action_requests"][0]
    print("Cần duyệt:", request["name"], request["arguments"])

    decision = input("Duyệt? (y/n) ").strip().lower()
    resume = (
        {"decisions": [{"type": "approve"}]}
        if decision == "y"
        else {"decisions": [{"type": "reject", "message": "Nhân viên từ chối: cần ảnh chụp lỗi sản phẩm."}]}
    )
    result = agent.invoke(Command(resume=resume), config=config, context=customer, version="v2")

print(result.value["messages"][-1].text)
```

Trong ứng dụng thật, bước "duyệt" không phải `input()` mà là một màn hình quản trị: yêu cầu được lưu lại (nhờ checkpointer, state tồn tại cả khi server khởi động lại), nhân viên duyệt sau vài phút hoặc vài giờ, hệ thống gọi `invoke(Command(resume=...))` với cùng `thread_id`. Xem [Human-in-the-loop](../../oss/python/langchain/human-in-the-loop.md).

## 6. Middleware tùy chỉnh: log chi phí

Middleware có thể can thiệp vào mọi điểm trong vòng lặp agent. Ví dụ ghi log token và độ trễ của mỗi lệnh gọi model:

```python title="shop/middleware.py"
import logging
import time
from collections.abc import Callable

from langchain.agents.middleware import ModelRequest, ModelResponse, wrap_model_call

log = logging.getLogger("agent")


@wrap_model_call
def log_model_usage(
    request: ModelRequest,
    handler: Callable[[ModelRequest], ModelResponse],
) -> ModelResponse:
    start = time.perf_counter()
    response = handler(request)  # gọi model thật
    elapsed_ms = (time.perf_counter() - start) * 1000
    for message in response.result:
        usage = getattr(message, "usage_metadata", None)
        if usage:
            log.info("model_call input=%d output=%d latency_ms=%.0f",
                     usage["input_tokens"], usage["output_tokens"], elapsed_ms)
    return response
```

Thêm `log_model_usage` vào danh sách `middleware` của agent. Các hook khác: `before_model`, `after_model`, `wrap_tool_call`... Xem [Middleware tùy chỉnh](../../oss/python/langchain/middleware/custom.md).

## 7. Structured output từ agent

Khi agent là một bước trong hệ thống lớn hơn (không trả lời trực tiếp người dùng), yêu cầu output có cấu trúc:

```python
from typing import Literal

from pydantic import BaseModel


class TicketTriage(BaseModel):
    category: Literal["don_hang", "doi_tra", "san_pham", "khac"]
    order_ids: list[str]
    needs_human: bool
    summary: str


triage_agent = create_agent(
    model="anthropic:claude-opus-5",
    tools=[list_my_orders, get_order],
    context_schema=Customer,
    response_format=TicketTriage,
)
result = triage_agent.invoke({"messages": [...]}, context=customer)
print(result["structured_response"])
```

Xem [Structured Output](../../oss/python/langchain/structured-output.md).

## Bài tập

**Bài 2.1.** Dựng agent ở mục 3, chạy `chat.py`. Thử hỏi về đơn `DH2001` (thuộc khách khác). Agent có bị lộ thông tin không?

**Bài 2.2.** Chạy `refund_flow.py` ba lần: duyệt, từ chối, và **sửa** (edit) số tiền hoặc lý do. Tra cứu định dạng quyết định `edit` trong trang [Human-in-the-loop](../../oss/python/langchain/human-in-the-loop.md).

**Bài 2.3.** Thêm `log_model_usage`, chạy một hội thoại 5 lượt, tính tổng chi phí từ log.

**Bài 2.4.** Viết 20 tin nhắn khách hàng thử nghiệm (có câu hỏi đơn, câu cần nhiều tool, câu cố tình lừa agent hoàn tiền không qua duyệt, câu hỏi ngoài phạm vi). Chạy tất cả, ghi lại những câu agent xử lý chưa tốt, sửa prompt hoặc tool, chạy lại.

**Bài 2.5 (mở rộng).** Kết nối agent với dữ liệu thật từ cửa hàng Shopify của bạn (Admin API) thay cho dữ liệu giả, chỉ với quyền đọc.

## Checklist

- [ ] Tool nhận danh tính người dùng qua `ToolRuntime`, kiểm tra quyền truy cập trong tool.
- [ ] Agent có checkpointer, streaming, giới hạn số lượt và model dự phòng.
- [ ] Hành động nhạy cảm đi qua human-in-the-loop.
- [ ] Có middleware ghi log token và độ trễ.
- [ ] Có bộ tin nhắn thử nghiệm, kể cả tin nhắn cố tình tấn công.

## Đọc thêm

- [Agents](../../oss/python/langchain/agents.md), [Tools](../../oss/python/langchain/tools.md), [Runtime](../../oss/python/langchain/runtime.md)
- [Middleware: tổng quan](../../oss/python/langchain/middleware/overview.md), [có sẵn](../../oss/python/langchain/middleware/built-in.md), [tùy chỉnh](../../oss/python/langchain/middleware/custom.md)
- [Human-in-the-loop](../../oss/python/langchain/human-in-the-loop.md), [Guardrails](../../oss/python/langchain/guardrails.md)
- Ví dụ đầy đủ: [Handoffs: Customer Support](../../oss/python/langchain/multi-agent/handoffs-customer-support.md)

---

**Bài trước:** [Bài 1: Tư duy agent](01-agent-loop.md) · [Tổng quan track](index.md) · **Bài tiếp theo:** [Bài 3: Workflow với LangGraph](03-langgraph.md)
