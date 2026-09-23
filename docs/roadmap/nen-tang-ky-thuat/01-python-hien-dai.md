# Bài 1: Python hiện đại cho AI

!!! abstract "Tóm tắt bài học"
    - **Mục tiêu:** thiết lập dự án Python chuẩn, dùng type hints và Pydantic để kiểm soát dữ liệu vào ra của LLM, quản lý cấu hình an toàn, và gọi LLM song song bằng async.
    - **Thời lượng:** khoảng 1 tuần.
    - **Yêu cầu trước:** Python cơ bản.

## 1. Quản lý dự án với uv

[uv](https://docs.astral.sh/uv/) thay thế cùng lúc `pip`, `venv`, `pip-tools` và `pyenv`: nhanh, có lockfile, cài được cả phiên bản Python.

```bash
uv init llm-service
cd llm-service
uv add anthropic pydantic pydantic-settings
uv add --dev pytest ruff
```

Kết quả là file `pyproject.toml` (khai báo dependency) và `uv.lock` (khóa chính xác phiên bản). **Commit cả hai vào Git** để mọi máy, mọi môi trường cài ra đúng một bộ thư viện.

| Lệnh | Tác dụng |
|---|---|
| `uv add <gói>` | Thêm dependency |
| `uv sync` | Cài đúng theo lockfile |
| `uv run <lệnh>` | Chạy lệnh trong môi trường của dự án |
| `uv run ruff check .` | Kiểm tra lỗi và phong cách code |

## 2. Type hints và Pydantic: kiểm soát dữ liệu từ LLM

Output của LLM là **dữ liệu không đáng tin**: có thể thiếu trường, sai kiểu, sai định dạng. Pydantic cho phép khai báo "hình dạng" dữ liệu mong muốn và kiểm tra tự động.

```python title="app/schemas.py"
from typing import Literal

from pydantic import BaseModel, Field


class FeedbackAnalysis(BaseModel):
    """Kết quả phân tích một phản hồi của khách hàng."""

    sentiment: Literal["positive", "neutral", "negative"]
    topics: list[str] = Field(description="Các chủ đề được nhắc tới, ví dụ: giao hàng, giá")
    summary: str = Field(description="Tóm tắt một câu bằng tiếng Việt")
    urgent: bool = Field(description="True nếu cần xử lý ngay (khiếu nại nghiêm trọng, đe dọa hủy)")
```

Pydantic mang lại ba thứ:

1. **Kiểm tra dữ liệu:** `FeedbackAnalysis.model_validate_json(text)` báo lỗi rõ ràng nếu JSON không khớp.
2. **Sinh JSON Schema:** `FeedbackAnalysis.model_json_schema()` là thứ SDK gửi cho LLM để ép output đúng cấu trúc (Bài 3 của [track Hiểu LLM](../hieu-llm/03-claude-api.md)).
3. **Tài liệu sống:** `description` vừa giải thích cho người đọc code, vừa hướng dẫn cho LLM.

```python
from pydantic import ValidationError

try:
    FeedbackAnalysis.model_validate_json('{"sentiment": "angry", "topics": []}')
except ValidationError as e:
    print(e)
    # sentiment: Input should be 'positive', 'neutral' or 'negative'
    # summary: Field required
    # urgent: Field required
```

## 3. Cấu hình và bí mật

**Không bao giờ** viết API key vào code. Dùng biến môi trường và `pydantic-settings` để đọc, kiểm tra kiểu và đặt giá trị mặc định ở một chỗ duy nhất.

```python title="app/config.py"
from pydantic import SecretStr
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", extra="ignore")

    anthropic_api_key: SecretStr
    model: str = "claude-opus-5"
    max_concurrency: int = 5
    request_timeout: float = 60.0


settings = Settings()
```

```bash title=".env (KHÔNG commit file này)"
ANTHROPIC_API_KEY=sk-ant-...
MAX_CONCURRENCY=8
```

- `SecretStr` ngăn key bị in lộ ra log (`print(settings)` hiển thị `**********`).
- Thêm `.env` vào `.gitignore`. Tạo file `.env.example` (không có giá trị thật) để người khác biết cần cấu hình gì.
- Thiếu biến bắt buộc, chương trình **báo lỗi ngay khi khởi động** thay vì hỏng giữa chừng.

## 4. Gọi LLM có cấu trúc

```python title="app/llm.py"
import anthropic

from app.config import settings
from app.schemas import FeedbackAnalysis

client = anthropic.Anthropic(
    api_key=settings.anthropic_api_key.get_secret_value(),
    timeout=settings.request_timeout,
)

SYSTEM = """Bạn phân tích phản hồi của khách hàng cho một cửa hàng thương mại điện tử.
Đánh dấu urgent khi khách hàng khiếu nại nghiêm trọng, đòi hoàn tiền hoặc đe dọa hủy đơn."""


def analyze(feedback: str) -> FeedbackAnalysis:
    response = client.messages.parse(
        model=settings.model,
        max_tokens=16000,
        system=SYSTEM,
        messages=[{"role": "user", "content": feedback}],
        output_format=FeedbackAnalysis,
    )
    return response.parsed_output


if __name__ == "__main__":
    print(analyze("Giao hàng trễ 5 ngày, hộp bị móp. Lần sau chắc không mua nữa."))
```

`messages.parse` gửi schema của `FeedbackAnalysis` cho Claude, và trả về **một đối tượng Pydantic đã được kiểm tra**. Bạn làm việc với `result.sentiment`, `result.urgent` thay vì tự parse chuỗi JSON.

## 5. Async: gọi nhiều LLM cùng lúc

### 5.1 Vì sao cần async?

Một lệnh gọi LLM mất từ 1 đến 30 giây, và gần như toàn bộ thời gian đó chương trình chỉ **chờ mạng**. Xử lý 100 phản hồi tuần tự, mỗi cái 3 giây, mất 5 phút. Gửi đồng thời 10 request một lúc, thời gian còn khoảng 30 giây.

`async`/`await` cho phép một luồng Python chờ nhiều thao tác I/O cùng lúc. Trong lúc request A đang chờ phản hồi, chương trình gửi tiếp request B, C, D...

### 5.2 Giới hạn số request đồng thời

Gửi 1.000 request **cùng một lúc** sẽ bị nhà cung cấp chặn (lỗi 429, rate limit). Dùng `asyncio.Semaphore` để giới hạn số request đang chạy:

```python title="app/batch.py"
import asyncio
import time

import anthropic

from app.config import settings
from app.llm import SYSTEM
from app.schemas import FeedbackAnalysis

async_client = anthropic.AsyncAnthropic(
    api_key=settings.anthropic_api_key.get_secret_value(),
    max_retries=4,  # SDK tự retry lỗi 429, 5xx với backoff
)


async def analyze_async(feedback: str, limiter: asyncio.Semaphore) -> FeedbackAnalysis | None:
    async with limiter:  # tối đa N request cùng lúc
        try:
            response = await async_client.messages.parse(
                model=settings.model,
                max_tokens=16000,
                system=SYSTEM,
                messages=[{"role": "user", "content": feedback}],
                output_format=FeedbackAnalysis,
            )
            return response.parsed_output
        except anthropic.APIError as e:
            # Một phản hồi lỗi không được làm hỏng cả lô
            print(f"Lỗi khi phân tích: {e}")
            return None


async def analyze_many(feedbacks: list[str]) -> list[FeedbackAnalysis | None]:
    limiter = asyncio.Semaphore(settings.max_concurrency)
    return await asyncio.gather(*(analyze_async(f, limiter) for f in feedbacks))


if __name__ == "__main__":
    samples = ["Sản phẩm tốt, giao nhanh."] * 20
    start = time.perf_counter()
    results = asyncio.run(analyze_many(samples))
    print(f"{len(results)} kết quả trong {time.perf_counter() - start:.1f}s")
```

Những điểm quan trọng:

- `asyncio.gather` giữ **đúng thứ tự** kết quả theo thứ tự đầu vào.
- Bắt lỗi **bên trong** từng tác vụ, để một lỗi không hủy cả lô.
- SDK Anthropic đã tự retry các lỗi tạm thời (429, 5xx, lỗi mạng) với backoff tăng dần. Chỉ cần tự viết logic retry khi muốn hành vi khác mặc định.

!!! tip "Khi nào không cần async?"
    Nếu bạn cần xử lý hàng nghìn tác vụ và **không cần kết quả ngay**, dùng Message Batches API: rẻ hơn 50%, không lo rate limit. Xem [track Hiểu LLM, Bài 3](../hieu-llm/03-claude-api.md#10-message-batches-xu-ly-hang-loat-re-hon-50).

## 6. Generator: nền tảng của streaming

Streaming (hiện chữ dần dần như ChatGPT) dựa trên **generator**: hàm dùng `yield` để trả ra từng phần thay vì trả một lần.

```python
from collections.abc import Iterator

from app.llm import client
from app.config import settings


def stream_summary(text: str) -> Iterator[str]:
    with client.messages.stream(
        model=settings.model,
        max_tokens=64000,
        messages=[{"role": "user", "content": f"Tóm tắt ngắn gọn:\n\n{text}"}],
    ) as stream:
        for piece in stream.text_stream:
            yield piece


for piece in stream_summary("..."):
    print(piece, end="", flush=True)
```

Bài 2 sẽ đưa generator này ra API bằng Server-Sent Events.

## 7. Logging

`print` không đủ cho production. Dùng module `logging` với định dạng thống nhất, và **ghi lại thông tin của mỗi lệnh gọi LLM**: model, số token, thời gian, `request_id` (để báo lỗi cho nhà cung cấp khi cần).

```python
import logging
import time

logging.basicConfig(level=logging.INFO, format="%(asctime)s %(levelname)s %(name)s %(message)s")
log = logging.getLogger("llm")

start = time.perf_counter()
response = client.messages.create(model=settings.model, max_tokens=16000, messages=[...])
log.info(
    "llm_call model=%s input_tokens=%d output_tokens=%d latency_ms=%d request_id=%s",
    response.model,
    response.usage.input_tokens,
    response.usage.output_tokens,
    (time.perf_counter() - start) * 1000,
    response._request_id,
)
```

!!! warning "Dữ liệu nhạy cảm trong log"
    Đừng ghi toàn bộ nội dung prompt và câu trả lời vào log thường, vì chúng có thể chứa thông tin cá nhân của khách hàng. Dùng hệ thống tracing có phân quyền truy cập (xem [track LLMOps](../llmops/01-observability.md)).

## Bài tập

**Bài 1.1.** Tạo dự án `llm-service` với uv, cài dependency, cấu hình `.env`, chạy `app/llm.py` với 5 phản hồi tiếng Việt khác nhau.

**Bài 1.2.** Thêm trường `language: Literal["vi", "en", "other"]` vào `FeedbackAnalysis`. Thử với một phản hồi tiếng Anh.

**Bài 1.3.** Chạy `app/batch.py` với 30 phản hồi, lần lượt với `max_concurrency` là 1, 5, 10. Ghi lại thời gian. Tốc độ tăng tuyến tính không? Vì sao?

??? tip "Gợi ý"
    Tốc độ tăng gần tuyến tính ở mức đồng thời thấp, rồi chững lại khi chạm rate limit của tài khoản (bạn sẽ thấy SDK retry lỗi 429) hoặc khi độ trễ của từng request tăng lên. Mức đồng thời tối ưu phụ thuộc vào giới hạn của tài khoản, không có con số chung.

**Bài 1.4.** Viết hàm `analyze_many` sao cho, ngoài danh sách kết quả, nó còn trả về tổng số token input, output của cả lô.

??? tip "Gợi ý lời giải"
    Trả về cả `response.usage` từ `analyze_async` (ví dụ dạng tuple), rồi cộng dồn sau `gather`. Đừng cộng vào một biến toàn cục từ nhiều tác vụ nếu sau này chuyển sang đa luồng; với asyncio một luồng thì an toàn, nhưng trả giá trị về vẫn là thiết kế rõ ràng hơn.

## Checklist

- [ ] Dự án có `pyproject.toml`, `uv.lock` trong Git; `.env` không nằm trong Git.
- [ ] Output của LLM luôn đi qua một Pydantic model trước khi dùng.
- [ ] Gọi song song có giới hạn đồng thời, một lỗi không làm hỏng cả lô.
- [ ] Mỗi lệnh gọi LLM được log model, token, độ trễ, `request_id`.

## Đọc thêm

- [uv documentation](https://docs.astral.sh/uv/), [Pydantic](https://docs.pydantic.dev/), [asyncio](https://docs.python.org/3/library/asyncio.html)
- [Structured Output trong LangChain](../../oss/python/langchain/structured-output.md)

---

[Tổng quan track](index.md) · **Bài tiếp theo:** [Bài 2: FastAPI và streaming](02-fastapi-streaming.md)
