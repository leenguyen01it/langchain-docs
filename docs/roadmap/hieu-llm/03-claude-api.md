# Bài 3: Claude API chuyên sâu

!!! abstract "Tóm tắt bài học"
    - **Mục tiêu:** dùng thành thạo Messages API của Claude qua Python SDK: hội thoại, streaming, ảnh và PDF, structured output, đếm token, prompt caching, xử lý lỗi, batch, và xử lý từ chối.
    - **Thời lượng:** khoảng 1 tuần.
    - **Yêu cầu trước:** [Bài 2](02-sampling-reasoning.md).

Bài này là "sổ tay" bạn sẽ quay lại nhiều lần. Mỗi mục là một tính năng độc lập, kèm code chạy được.

## 1. Cài đặt và xác thực

```bash
uv add anthropic
```

```python
import anthropic

# Tự đọc ANTHROPIC_API_KEY từ biến môi trường
client = anthropic.Anthropic()
async_client = anthropic.AsyncAnthropic()  # dùng trong code async (FastAPI, asyncio)
```

| Model | ID | Giá input / output (USD mỗi 1 triệu token) | Dùng khi |
|---|---|---|---|
| Claude Fable 5.1 | `claude-fable-5-1` | 10 / 50 | Tác vụ khó nhất, agent chạy rất dài |
| Claude Opus 5 | `claude-opus-5` | 5 / 25 | Mặc định cho phần lớn tác vụ cần chất lượng cao |
| Claude Sonnet 5 | `claude-sonnet-5` | 2 / 10 | Khối lượng lớn, cân bằng chất lượng và chi phí |
| Claude Haiku 4.5 | `claude-haiku-4-5` | 1 / 5 | Tác vụ đơn giản, cần nhanh, số lượng rất lớn |

Giá trên được ghi lại tại thời điểm viết; luôn kiểm tra bảng giá chính thức. Dùng **đúng chuỗi ID** như bảng, không tự thêm hậu tố ngày tháng. Có thể tra cứu model và giới hạn của nó bằng API:

```python
info = client.models.retrieve("claude-opus-5")
print(info.display_name, info.max_input_tokens, info.max_tokens)
```

## 2. Messages API cơ bản

```python
response = client.messages.create(
    model="claude-opus-5",
    max_tokens=16000,
    system="Bạn là trợ lý chăm sóc khách hàng của một cửa hàng thời trang. Trả lời ngắn gọn, lịch sự.",
    messages=[{"role": "user", "content": "Shop có đổi size được không?"}],
)

for block in response.content:
    if block.type == "text":
        print(block.text)

print(response.stop_reason, response.usage.input_tokens, response.usage.output_tokens)
```

Cấu trúc response cần nắm:

| Trường | Ý nghĩa |
|---|---|
| `content` | **Danh sách** các khối: `text`, `thinking`, `tool_use`... Luôn kiểm tra `block.type` |
| `stop_reason` | Vì sao dừng (xem [Bài 2](02-sampling-reasoning.md#4-max_tokens-va-stop-reason)) |
| `usage` | Số token input, output, cache. Dùng để tính chi phí |
| `model` | Model thực sự đã phục vụ request |
| `_request_id` | Mã request, ghi vào log để báo lỗi cho Anthropic khi cần |

**System prompt** chứa chỉ dẫn về vai trò, quy tắc, bối cảnh. Tin nhắn `user` chứa yêu cầu cụ thể. Tách bạch hai phần giúp prompt dễ quản lý và dễ cache.

## 3. Hội thoại nhiều lượt

API **không lưu trạng thái**: mỗi request phải gửi lại toàn bộ lịch sử.

```python
history: list[dict] = []


def chat(user_message: str) -> str:
    history.append({"role": "user", "content": user_message})
    response = client.messages.create(
        model="claude-opus-5",
        max_tokens=16000,
        system="Bạn là trợ lý bán hàng.",
        messages=history,
    )
    # Lưu lại toàn bộ content (kể cả khối thinking), không chỉ phần text
    history.append({"role": "assistant", "content": response.content})
    return next(b.text for b in response.content if b.type == "text")


print(chat("Tôi cao 1m70, nặng 65kg"))
print(chat("Vậy tôi nên mặc áo size gì?"))  # model nhớ chiều cao, cân nặng nhờ lịch sử
```

Quy tắc: tin nhắn đầu tiên phải là `user`; khi tiếp tục hội thoại trên cùng model, hãy gửi lại **nguyên vẹn** `response.content` của lượt trước thay vì chỉ lấy phần text.

## 4. Streaming

```python
with client.messages.stream(
    model="claude-opus-5",
    max_tokens=64000,
    messages=[{"role": "user", "content": "Viết mô tả sản phẩm cho áo khoác gió nam"}],
) as stream:
    for text in stream.text_stream:
        print(text, end="", flush=True)
    final = stream.get_final_message()  # message hoàn chỉnh, có usage và stop_reason

print("\n", final.stop_reason, final.usage.output_tokens)
```

!!! tip "Mặc định nên dùng streaming"
    Streaming không chỉ để hiển thị chữ dần dần. Với request có input dài hoặc output dài, streaming giúp **tránh timeout** của kết nối HTTP. Nếu không cần xử lý từng đoạn, vẫn có thể stream rồi gọi `get_final_message()` để lấy kết quả cuối.

Streaming ra API web: xem [Nền tảng kỹ thuật, Bài 2](../nen-tang-ky-thuat/02-fastapi-streaming.md).

## 5. Ảnh và PDF

### 5.1 Ảnh

```python
import base64
from pathlib import Path

image_data = base64.standard_b64encode(Path("hoa-don.jpg").read_bytes()).decode()

response = client.messages.create(
    model="claude-opus-5",
    max_tokens=16000,
    messages=[{
        "role": "user",
        "content": [
            {"type": "image", "source": {"type": "base64", "media_type": "image/jpeg", "data": image_data}},
            {"type": "text", "text": "Trích xuất tên cửa hàng, ngày, tổng tiền từ hóa đơn này."},
        ],
    }],
)
```

Đặt ảnh **trước** câu hỏi. Có thể dùng `{"type": "url", "url": "https://..."}` thay cho base64 nếu ảnh có URL công khai.

### 5.2 PDF

```python
pdf_data = base64.standard_b64encode(Path("hop-dong.pdf").read_bytes()).decode()

response = client.messages.create(
    model="claude-opus-5",
    max_tokens=16000,
    messages=[{
        "role": "user",
        "content": [
            {
                "type": "document",
                "source": {"type": "base64", "media_type": "application/pdf", "data": pdf_data},
                "citations": {"enabled": True},  # trả về trích dẫn chính xác từ tài liệu
            },
            {"type": "text", "text": "Hợp đồng này có điều khoản phạt khi chấm dứt trước hạn không?"},
        ],
    }],
)

for block in response.content:
    if block.type == "text":
        print(block.text)
        for c in block.citations or []:
            print(f"   ↳ trang {c.start_page_number}: {c.cited_text!r}")
```

Claude đọc cả văn bản lẫn hình ảnh trong PDF (bảng, biểu đồ). Với tài liệu ngắn và vừa, gửi thẳng PDF thường là cách **đơn giản nhất và chính xác nhất**, trước khi nghĩ tới RAG.

## 6. Structured output

Khi code của bạn cần **đọc** output của LLM (lưu database, gọi API khác), hãy ép output theo schema.

```python
from typing import Literal

from pydantic import BaseModel


class Invoice(BaseModel):
    store_name: str
    date: str
    total_vnd: int
    payment_method: Literal["cash", "card", "transfer", "unknown"]


response = client.messages.parse(
    model="claude-opus-5",
    max_tokens=16000,
    messages=[{"role": "user", "content": "Hóa đơn Highlands Coffee ngày 12/03/2026, tổng 89.000đ, quẹt thẻ."}],
    output_format=Invoice,
)
invoice = response.parsed_output  # đối tượng Invoice đã được kiểm tra
print(invoice.total_vnd)  # 89000
```

`messages.parse` đảm bảo output **khớp schema**. Nếu không dùng Pydantic, có thể truyền JSON Schema trực tiếp qua tham số `output_config={"format": {"type": "json_schema", "schema": {...}}}` của `messages.create`.

!!! note "Khớp schema không có nghĩa là đúng"
    Structured output đảm bảo **định dạng**, không đảm bảo **nội dung**. Model vẫn có thể đọc sai tổng tiền. Kiểm tra logic nghiệp vụ (tổng tiền dương, ngày hợp lệ) và đánh giá độ chính xác bằng eval.

## 7. Đếm token và ước tính chi phí

```python
count = client.messages.count_tokens(
    model="claude-opus-5",
    system="Bạn là trợ lý bán hàng.",
    messages=[{"role": "user", "content": "Tư vấn cho tôi một đôi giày chạy bộ"}],
)
print(count.input_tokens)

price_input_per_million = 5.00  # claude-opus-5
print(f"Chi phí input ước tính: ${count.input_tokens * price_input_per_million / 1e6:.6f}")
```

Dùng `count_tokens` để kiểm tra trước khi gửi tài liệu lớn, hoặc để cắt bớt lịch sử hội thoại khi gần chạm giới hạn. Không dùng tokenizer của nhà cung cấp khác để đếm token cho Claude, vì kết quả sẽ sai lệch.

## 8. Prompt caching

### 8.1 Cơ chế

Nhiều request có chung một phần đầu dài: system prompt, tài liệu tham khảo, định nghĩa tool. **Prompt caching** lưu phần này lại; các request sau đọc từ cache với giá chỉ khoảng **10% giá input** và thời gian phản hồi nhanh hơn.

Cache hoạt động theo **tiền tố (prefix)**: thứ tự ghép prompt là `tools`, rồi `system`, rồi `messages`. **Bất kỳ byte nào thay đổi** trong phần đầu sẽ làm mất cache của mọi thứ phía sau.

### 8.2 Cách dùng

```python
POLICY_DOCUMENT = Path("chinh-sach-cua-hang.md").read_text(encoding="utf-8")  # tài liệu dài

response = client.messages.create(
    model="claude-opus-5",
    max_tokens=16000,
    system=[
        {"type": "text", "text": "Bạn là trợ lý chăm sóc khách hàng. Chỉ trả lời dựa trên chính sách dưới đây."},
        {
            "type": "text",
            "text": POLICY_DOCUMENT,
            "cache_control": {"type": "ephemeral"},  # đánh dấu điểm cache (mặc định sống 5 phút)
        },
    ],
    messages=[{"role": "user", "content": "Hàng giảm giá có được đổi trả không?"}],
)

u = response.usage
print(f"ghi cache: {u.cache_creation_input_tokens}, đọc cache: {u.cache_read_input_tokens}, "
      f"input thường: {u.input_tokens}")
```

Lần gọi đầu: `cache_creation_input_tokens` lớn (ghi cache, giá cao hơn input thường một chút). Các lần gọi sau trong vòng 5 phút: `cache_read_input_tokens` lớn, chi phí giảm mạnh. Có thể đặt `"ttl": "1h"` nếu các request cách nhau lâu hơn.

Cách đơn giản hơn: đặt `cache_control={"type": "ephemeral"}` ở **cấp request** (tham số của `messages.create`), SDK tự đặt điểm cache ở khối cuối cùng có thể cache.

### 8.3 Những lỗi làm cache vô hiệu một cách âm thầm

| Lỗi | Ví dụ |
|---|---|
| Nội dung thay đổi ở đầu prompt | Chèn `datetime.now()` hoặc ID request vào system prompt |
| Thứ tự không ổn định | `json.dumps(dict)` không sắp xếp khóa; danh sách tool thay đổi thứ tự giữa các request |
| Prefix quá ngắn | Phần muốn cache ngắn hơn ngưỡng tối thiểu (từ vài trăm tới vài nghìn token tùy model) thì sẽ không được cache |
| Đổi model hoặc tham số | Cache gắn với từng model |

**Luôn kiểm chứng** bằng `usage.cache_read_input_tokens`. Nếu nó luôn bằng 0 dù gọi lặp lại, có thứ gì đó đang thay đổi prefix. Nguyên tắc thiết kế: **phần cố định đặt trước, phần thay đổi (câu hỏi, thời gian, dữ liệu người dùng) đặt sau cùng**.

## 9. Xử lý lỗi và retry

SDK **tự động retry** các lỗi tạm thời (lỗi mạng, 408, 409, 429, 5xx) với backoff tăng dần, mặc định 2 lần. Bạn cấu hình thêm khi cần:

```python
client = anthropic.Anthropic(max_retries=4, timeout=120.0)

# Ghi đè cho một request cụ thể
client.with_options(timeout=10.0).messages.create(...)
```

Bắt lỗi theo **chuỗi từ cụ thể tới tổng quát**, vì mỗi loại lỗi cần cách xử lý khác nhau:

```python
try:
    response = client.messages.create(
        model="claude-opus-5",
        max_tokens=16000,
        messages=[{"role": "user", "content": "Xin chào"}],
    )
except anthropic.BadRequestError as e:
    # 400: request sai (tham số không hợp lệ, prompt quá dài...). Retry vô ích, phải sửa code
    log.error("Bad request: %s", e.message)
except anthropic.AuthenticationError:
    # 401: API key sai hoặc hết hạn. Lỗi cấu hình
    raise
except anthropic.NotFoundError:
    # 404: sai tên model hoặc endpoint
    raise
except anthropic.RateLimitError as e:
    # 429: đã retry mà vẫn quá giới hạn. Báo người dùng thử lại sau, hoặc xếp vào hàng đợi
    retry_after = e.response.headers.get("retry-after")
except anthropic.APIStatusError as e:
    # Các mã lỗi còn lại; >= 500 là lỗi phía máy chủ
    log.error("API error %s: %s", e.status_code, e.message)
except anthropic.APIConnectionError:
    # Lỗi mạng sau khi đã retry
    log.error("Không kết nối được tới API")
```

| Lỗi | Retry có ích không? |
|---|---|
| 400, 401, 403, 404 | **Không**. Phải sửa request hoặc cấu hình |
| 429, 5xx, lỗi mạng, timeout | **Có**. SDK đã retry sẵn; nếu vẫn lỗi, xếp hàng đợi hoặc báo người dùng |

Luôn ghi lại `e.request_id` hoặc `response._request_id` vào log để tra cứu khi cần hỗ trợ.

## 10. Message Batches: xử lý hàng loạt rẻ hơn 50%

Khi cần xử lý hàng nghìn yêu cầu và **không cần kết quả ngay** (phân loại toàn bộ đánh giá sản phẩm, dịch cả catalogue, sinh dữ liệu eval), dùng Batches API: **giảm 50% chi phí**, không lo rate limit. Phần lớn batch hoàn thành trong vòng 1 giờ, tối đa 24 giờ.

```python
import time

from anthropic.types.message_create_params import MessageCreateParamsNonStreaming
from anthropic.types.messages.batch_create_params import Request

reviews = ["Áo đẹp, vải mát", "Giao sai màu, shop không phản hồi", "Tạm được"]

batch = client.messages.batches.create(
    requests=[
        Request(
            custom_id=f"review-{i}",
            params=MessageCreateParamsNonStreaming(
                model="claude-opus-5",
                max_tokens=2048,  # token suy nghĩ cũng tính vào giới hạn này
                output_config={"effort": "low"},  # tác vụ đơn giản: suy nghĩ ít, rẻ hơn
                messages=[{
                    "role": "user",
                    "content": f"Phân loại cảm xúc (positive/neutral/negative), chỉ trả lời một từ: {text}",
                }],
            ),
        )
        for i, text in enumerate(reviews)
    ]
)

while (batch := client.messages.batches.retrieve(batch.id)).processing_status != "ended":
    time.sleep(30)

results = {}
for item in client.messages.batches.results(batch.id):
    if item.result.type == "succeeded":
        message = item.result.message
        results[item.custom_id] = next((b.text for b in message.content if b.type == "text"), "")
    else:
        results[item.custom_id] = f"LỖI: {item.result.type}"

print(results)
```

Kết quả trả về **không theo thứ tự**: luôn dùng `custom_id` để ghép lại với dữ liệu gốc.

## 11. Khi model từ chối: `refusal`

Với các yêu cầu bị hệ thống an toàn đánh giá là có rủi ro, response có `stop_reason == "refusal"` (HTTP vẫn là 200). Luôn kiểm tra trước khi đọc `content`:

```python
if response.stop_reason == "refusal":
    category = response.stop_details.category if response.stop_details else None
    log.warning("Refused, category=%s", category)
    return "Xin lỗi, tôi không thể hỗ trợ yêu cầu này."
```

Với Claude Opus 5, API có tính năng **fallback phía server**: khi model từ chối, request tự động được chạy lại trên một model khác phù hợp, ngay trong cùng một lệnh gọi. Tính năng đang ở dạng beta, bật như sau:

```python
response = client.beta.messages.create(
    model="claude-opus-5",
    max_tokens=16000,
    betas=["server-side-fallback-2026-07-01"],
    fallbacks="default",  # API tự chọn model dự phòng theo loại từ chối
    messages=[{"role": "user", "content": "..."}],
)
```

Nếu `stop_reason` vẫn là `refusal` sau khi đã bật fallback, nghĩa là mọi model trong chuỗi đều từ chối. Với ứng dụng hợp lệ, từ chối hiếm khi xảy ra, nhưng hệ thống của bạn cần có nhánh xử lý cho nó.

## Bài tập

**Bài 3.1.** Viết chatbot dòng lệnh nhiều lượt có streaming, hiển thị số token và chi phí ước tính sau mỗi lượt.

**Bài 3.2.** Chụp ảnh 3 hóa đơn thật, trích xuất thông tin bằng `messages.parse` với schema `Invoice`. Kiểm tra độ chính xác bằng mắt.

**Bài 3.3.** Viết chatbot hỏi đáp dựa trên một tài liệu chính sách dài (ít nhất vài nghìn token) đặt trong system prompt có `cache_control`. Hỏi 5 câu liên tiếp, in `usage` mỗi lần. Tính chi phí thực tế và so sánh với trường hợp không dùng cache.

??? tip "Gợi ý"
    Nếu `cache_read_input_tokens` luôn bằng 0: kiểm tra tài liệu đã đủ dài chưa (vượt ngưỡng tối thiểu), system prompt có chứa thứ gì thay đổi giữa các lần gọi không, và các lần gọi có cách nhau quá 5 phút không.

**Bài 3.4.** Cố tình gây ra từng loại lỗi: sai API key, sai tên model, `max_tokens` âm. Quan sát exception và thông báo lỗi.

**Bài 3.5.** Dùng Batches API phân loại 100 đánh giá sản phẩm (có thể tự sinh bằng LLM). So sánh chi phí với cách gọi từng request.

## Checklist

- [ ] Luôn lọc `response.content` theo `block.type`.
- [ ] Hội thoại nhiều lượt gửi lại đầy đủ `response.content`.
- [ ] Dùng streaming cho output dài.
- [ ] Output máy đọc luôn qua structured output và kiểm tra nghiệp vụ.
- [ ] Prompt caching được kiểm chứng bằng `cache_read_input_tokens`.
- [ ] Xử lý lỗi theo từng loại, không bắt chung một exception.
- [ ] Có nhánh xử lý cho `max_tokens` và `refusal`.

## Đọc thêm

- [Tài liệu Claude API](https://docs.claude.com/): Messages, prompt caching, structured outputs, batches.
- [Models](../../oss/python/langchain/models.md) và [Messages](../../oss/python/langchain/messages.md): cách LangChain bọc các khái niệm này.

---

**Bài trước:** [Bài 2: Sampling, reasoning và context](02-sampling-reasoning.md) · [Tổng quan track](index.md) · **Bài tiếp theo:** [Bài 4: Tool use từ con số 0](04-tool-use.md)
