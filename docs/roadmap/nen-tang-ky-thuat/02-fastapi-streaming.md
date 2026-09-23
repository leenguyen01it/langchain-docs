# Bài 2: FastAPI và streaming

!!! abstract "Tóm tắt bài học"
    - **Mục tiêu:** đưa logic gọi LLM ra thành API: endpoint thường, endpoint streaming bằng Server-Sent Events, xử lý lỗi đúng mã HTTP, giới hạn tốc độ theo người dùng.
    - **Thời lượng:** khoảng 1 tuần.
    - **Yêu cầu trước:** [Bài 1](01-python-hien-dai.md).

## 1. Vì sao FastAPI?

[FastAPI](https://fastapi.tiangolo.com/) là lựa chọn phổ biến nhất cho backend ứng dụng AI bằng Python:

- **Async từ gốc**, phù hợp với việc chờ API LLM.
- Dùng **Pydantic** cho request và response, tái sử dụng được các model đã viết ở Bài 1.
- Tự sinh tài liệu API tương tác tại `/docs`.
- Hỗ trợ streaming dễ dàng.

```bash
uv add "fastapi[standard]"
```

## 2. Cấu trúc ứng dụng

```python title="app/main.py"
from contextlib import asynccontextmanager

import anthropic
from fastapi import FastAPI

from app.config import settings

state: dict = {}


@asynccontextmanager
async def lifespan(app: FastAPI):
    # Tạo client một lần khi khởi động, dùng chung cho mọi request
    state["client"] = anthropic.AsyncAnthropic(
        api_key=settings.anthropic_api_key.get_secret_value(),
        max_retries=3,
    )
    yield
    await state["client"].close()


app = FastAPI(title="LLM Service", lifespan=lifespan)


def get_client() -> anthropic.AsyncAnthropic:
    return state["client"]
```

**Tạo client một lần** trong `lifespan` thay vì mỗi request: client giữ connection pool, tái sử dụng kết nối HTTP giúp giảm độ trễ đáng kể.

`get_client` sẽ được dùng như một **dependency**. Nhờ vậy, khi viết test ở Bài 4, ta có thể thay client thật bằng client giả.

## 3. Endpoint thường: phân tích phản hồi

```python title="app/main.py (tiếp)"
from fastapi import Depends
from pydantic import BaseModel, Field

from app.llm import SYSTEM
from app.schemas import FeedbackAnalysis


class AnalyzeRequest(BaseModel):
    text: str = Field(min_length=1, max_length=5000)


@app.post("/analyze", response_model=FeedbackAnalysis)
async def analyze(
    req: AnalyzeRequest,
    client: anthropic.AsyncAnthropic = Depends(get_client),
) -> FeedbackAnalysis:
    response = await client.messages.parse(
        model=settings.model,
        max_tokens=16000,
        system=SYSTEM,
        messages=[{"role": "user", "content": req.text}],
        output_format=FeedbackAnalysis,
    )
    return response.parsed_output
```

```bash
uv run fastapi dev app/main.py
```

Mở `http://127.0.0.1:8000/docs` để thử API ngay trên trình duyệt.

!!! warning "Luôn giới hạn độ dài đầu vào"
    `max_length=5000` không chỉ để validate: nó **bảo vệ chi phí**. Không có giới hạn, một người dùng có thể gửi văn bản 500.000 ký tự và bạn trả tiền cho toàn bộ số token đó.

## 4. Streaming với Server-Sent Events

### 4.1 Vì sao cần streaming?

Một câu trả lời dài có thể mất 10 đến 20 giây để sinh xong. Nếu người dùng phải nhìn màn hình trống suốt thời gian đó, họ nghĩ ứng dụng bị treo. Với streaming, **chữ đầu tiên xuất hiện sau khoảng một giây**. Tổng thời gian không đổi, nhưng trải nghiệm tốt hơn hẳn.

Chỉ số quan trọng: **TTFT (Time To First Token)**, thời gian tới token đầu tiên.

### 4.2 Server-Sent Events (SSE)

SSE là giao thức đơn giản để server đẩy dữ liệu một chiều xuống client qua HTTP. Mỗi sự kiện có dạng:

```text
event: delta
data: {"text": "Xin chào"}

```

(mỗi sự kiện kết thúc bằng một dòng trống)

So với WebSocket, SSE đơn giản hơn, đi qua proxy và load balancer dễ hơn, và đủ cho phần lớn ứng dụng chat (client gửi câu hỏi bằng một request POST, server stream câu trả lời về).

### 4.3 Endpoint streaming

```python title="app/main.py (tiếp)"
import json
from collections.abc import AsyncIterator

from fastapi import Request
from fastapi.responses import StreamingResponse


class ChatRequest(BaseModel):
    message: str = Field(min_length=1, max_length=5000)


def sse(event: str, data: dict) -> str:
    return f"event: {event}\ndata: {json.dumps(data, ensure_ascii=False)}\n\n"


@app.post("/chat/stream")
async def chat_stream(
    req: ChatRequest,
    request: Request,
    client: anthropic.AsyncAnthropic = Depends(get_client),
) -> StreamingResponse:
    async def events() -> AsyncIterator[str]:
        try:
            async with client.messages.stream(
                model=settings.model,
                max_tokens=64000,
                messages=[{"role": "user", "content": req.message}],
            ) as stream:
                async for text in stream.text_stream:
                    if await request.is_disconnected():
                        # Người dùng đóng tab: dừng sinh để không tốn token vô ích
                        return
                    yield sse("delta", {"text": text})
                final = await stream.get_final_message()
            yield sse("done", {
                "stop_reason": final.stop_reason,
                "input_tokens": final.usage.input_tokens,
                "output_tokens": final.usage.output_tokens,
            })
        except anthropic.APIError:
            # Header 200 đã gửi đi rồi, lỗi phải báo qua một sự kiện
            yield sse("error", {"message": "Dịch vụ AI tạm thời gián đoạn, vui lòng thử lại."})

    return StreamingResponse(
        events(),
        media_type="text/event-stream",
        headers={"Cache-Control": "no-cache", "X-Accel-Buffering": "no"},
    )
```

Ba chi tiết dễ bị bỏ qua:

1. **Kiểm tra client ngắt kết nối.** Thoát khỏi khối `async with` sẽ đóng stream tới Claude, dừng việc sinh token mà không ai đọc.
2. **Lỗi giữa chừng không thể trả mã 500**, vì header `200 OK` đã được gửi khi stream bắt đầu. Hãy định nghĩa một loại sự kiện `error` và để client xử lý.
3. **Tắt buffering của proxy.** Header `X-Accel-Buffering: no` báo cho Nginx không gom dữ liệu lại, nếu không người dùng sẽ nhận cả câu trả lời một lúc ở cuối.

### 4.4 Thử bằng curl và JavaScript

```bash
curl -N -X POST http://127.0.0.1:8000/chat/stream \
  -H "Content-Type: application/json" \
  -d '{"message": "Viết 3 mẹo viết mô tả sản phẩm hấp dẫn"}'
```

Phía trình duyệt, vì `EventSource` chỉ hỗ trợ GET, ta đọc stream bằng `fetch`:

```javascript
const res = await fetch("/chat/stream", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ message: "Xin chào" }),
});
const reader = res.body.pipeThrough(new TextDecoderStream()).getReader();
let buffer = "";
while (true) {
  const { value, done } = await reader.read();
  if (done) break;
  buffer += value;
  const events = buffer.split("\n\n");
  buffer = events.pop();               // phần chưa trọn sự kiện, chờ lần đọc sau
  for (const raw of events) {
    const event = raw.match(/^event: (.*)$/m)?.[1];
    const data = JSON.parse(raw.match(/^data: (.*)$/m)?.[1] ?? "{}");
    if (event === "delta") output.textContent += data.text;
  }
}
```

!!! tip "Không cần tự viết giao diện chat"
    Các thư viện giao diện như những thư viện trong mục [Frontend](../../oss/python/langchain/frontend/overview.md) của site đã xử lý sẵn streaming, markdown, lịch sử. Tự viết một lần để hiểu cơ chế, rồi dùng thư viện cho dự án thật.

## 5. Xử lý lỗi đúng mã HTTP

Lỗi từ nhà cung cấp LLM cần được **dịch** sang mã HTTP có ý nghĩa với client của bạn, và **không được lộ chi tiết nội bộ** (API key sai, tên model...).

```python title="app/main.py (tiếp)"
import logging

from fastapi.responses import JSONResponse

log = logging.getLogger("api")


@app.exception_handler(anthropic.APIError)
async def handle_llm_error(request: Request, exc: anthropic.APIError) -> JSONResponse:
    log.exception("LLM error on %s", request.url.path)
    if isinstance(exc, anthropic.RateLimitError):
        return JSONResponse({"detail": "Hệ thống đang bận, vui lòng thử lại sau ít phút."},
                            status_code=503, headers={"Retry-After": "30"})
    if isinstance(exc, anthropic.APITimeoutError):
        return JSONResponse({"detail": "Yêu cầu xử lý quá lâu."}, status_code=504)
    if isinstance(exc, anthropic.BadRequestError):
        return JSONResponse({"detail": "Yêu cầu không hợp lệ."}, status_code=400)
    return JSONResponse({"detail": "Dịch vụ AI tạm thời gián đoạn."}, status_code=502)
```

| Lỗi từ LLM | Trả cho client | Lý do |
|---|---|---|
| `RateLimitError` (429) | 503 + `Retry-After` | Lỗi quá tải của **bạn**, không phải do client gửi quá nhiều |
| `APITimeoutError` | 504 | Hết thời gian chờ upstream |
| `BadRequestError` (400) | 400 hoặc 500 | Tùy lỗi do dữ liệu người dùng hay do code của bạn |
| `AuthenticationError` (401) | 500 | Lỗi cấu hình phía server, **không** được tiết lộ cho client |
| Lỗi 5xx của nhà cung cấp | 502 | Upstream lỗi |

## 6. Giới hạn tốc độ theo người dùng

Mỗi request tới LLM tốn tiền thật. Không có giới hạn, một người dùng (hoặc một bot) có thể đốt hết ngân sách của bạn trong vài giờ.

```python title="app/ratelimit.py"
import time
from collections import defaultdict, deque

from fastapi import Header, HTTPException

WINDOW_SECONDS = 60
MAX_REQUESTS = 10
_hits: dict[str, deque[float]] = defaultdict(deque)


def rate_limit(x_user_id: str = Header(...)) -> str:
    """Tối đa 10 request mỗi phút cho mỗi người dùng (sliding window)."""
    now = time.monotonic()
    hits = _hits[x_user_id]
    while hits and now - hits[0] > WINDOW_SECONDS:
        hits.popleft()
    if len(hits) >= MAX_REQUESTS:
        raise HTTPException(status_code=429, detail="Bạn gửi quá nhiều yêu cầu, hãy thử lại sau.")
    hits.append(now)
    return x_user_id
```

```python
@app.post("/chat/stream")
async def chat_stream(req: ChatRequest, request: Request,
                      user_id: str = Depends(rate_limit),
                      client: anthropic.AsyncAnthropic = Depends(get_client)):
    ...
```

!!! note "Giới hạn trong bộ nhớ chỉ đúng với một process"
    Khi chạy nhiều worker hoặc nhiều server, mỗi process có bộ đếm riêng. [Bài 3](03-du-lieu-hang-doi.md) chuyển bộ đếm sang Redis để dùng chung. Trong thực tế, danh tính người dùng lấy từ token xác thực (JWT), không phải từ một header tự khai.

Ngoài số request, nên giới hạn cả **số token mỗi ngày** cho mỗi người dùng, vì một request dài có thể tốn gấp trăm lần một request ngắn.

## Bài tập

**Bài 2.1.** Chạy API, thử `/analyze` qua giao diện `/docs`. Gửi một văn bản rỗng và một văn bản dài 6.000 ký tự, quan sát mã lỗi.

**Bài 2.2.** Thử `/chat/stream` bằng `curl -N`. Nhấn `Ctrl+C` giữa chừng, rồi kiểm tra log xem stream tới Claude có dừng không.

??? tip "Gợi ý"
    Thêm `log.info` sau vòng lặp và trong nhánh `is_disconnected()`. Nếu không thấy log của nhánh ngắt kết nối, hãy kiểm tra: với một số server, việc phát hiện ngắt kết nối chỉ xảy ra khi server cố gửi dữ liệu tiếp theo.

**Bài 2.3.** Viết một trang HTML đơn giản (một ô nhập, một vùng hiển thị) gọi `/chat/stream` và hiện chữ dần dần.

**Bài 2.4.** Thêm endpoint `/chat/stream` hỗ trợ **hội thoại nhiều lượt**: request nhận thêm `history: list[{"role", "content"}]`. Giới hạn lịch sử tối đa 20 lượt để kiểm soát chi phí.

**Bài 2.5 (mở rộng).** Đo TTFT và tổng thời gian của `/chat/stream` cho 10 câu hỏi. Ghi lại p50, p95.

## Checklist

- [ ] Client LLM được tạo một lần và dùng chung qua dependency.
- [ ] Mọi input có giới hạn độ dài.
- [ ] Endpoint streaming dừng khi client ngắt kết nối, báo lỗi giữa chừng bằng sự kiện `error`.
- [ ] Lỗi của nhà cung cấp được dịch sang mã HTTP phù hợp, không lộ chi tiết nội bộ.
- [ ] Có giới hạn tốc độ theo người dùng.

## Đọc thêm

- [FastAPI](https://fastapi.tiangolo.com/), [MDN: Server-sent events](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events)
- [Streaming trong LangChain](../../oss/python/langchain/streaming.md), [Frontend](../../oss/python/langchain/frontend/overview.md)

---

**Bài trước:** [Bài 1: Python hiện đại cho AI](01-python-hien-dai.md) · [Tổng quan track](index.md) · **Bài tiếp theo:** [Bài 3: Database, cache và hàng đợi](03-du-lieu-hang-doi.md)
