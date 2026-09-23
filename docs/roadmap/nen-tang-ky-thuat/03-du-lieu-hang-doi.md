# Bài 3: Database, cache và hàng đợi

!!! abstract "Tóm tắt bài học"
    - **Mục tiêu:** lưu lại mọi lệnh gọi LLM kèm chi phí, cache kết quả để tiết kiệm, giới hạn tốc độ dùng chung giữa nhiều server, và xử lý tác vụ chạy lâu bằng hàng đợi.
    - **Thời lượng:** khoảng 1 tuần.
    - **Yêu cầu trước:** [Bài 2](02-fastapi-streaming.md), đã cài Docker.

## 1. Hạ tầng cần thiết

```bash
docker run -d -p 5432:5432 -e POSTGRES_PASSWORD=postgres pgvector/pgvector:pg17
docker run -d -p 6379:6379 redis:7
uv add "sqlalchemy[asyncio]" asyncpg redis arq
```

```bash title=".env (bổ sung)"
DATABASE_URL=postgresql+asyncpg://postgres:postgres@localhost:5432/postgres
REDIS_URL=redis://localhost:6379/0
```

Thêm hai trường tương ứng vào `Settings` ở `app/config.py`: `database_url: str` và `redis_url: str`.

| Thành phần | Vai trò trong ứng dụng LLM |
|---|---|
| **PostgreSQL** | Hội thoại, người dùng, log từng lệnh gọi LLM (token, chi phí), kết quả tác vụ. Cộng thêm pgvector cho RAG |
| **Redis** | Cache kết quả, bộ đếm giới hạn tốc độ, hàng đợi tác vụ |
| **Hàng đợi (Arq)** | Chạy tác vụ lâu (tóm tắt tài liệu 100 trang, xử lý hàng loạt) ngoài luồng request |

## 2. Ghi log chi phí từng lệnh gọi LLM

Câu hỏi đầu tiên sếp bạn sẽ hỏi khi sản phẩm chạy thật: **"Mỗi tháng tốn bao nhiêu tiền, cho tính năng nào, cho khách hàng nào?"**. Hãy ghi lại ngay từ ngày đầu.

```python title="app/db.py"
from datetime import datetime

from sqlalchemy import DateTime, String, func
from sqlalchemy.ext.asyncio import async_sessionmaker, create_async_engine
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column

from app.config import settings

engine = create_async_engine(settings.database_url, pool_size=10)
SessionLocal = async_sessionmaker(engine, expire_on_commit=False)


class Base(DeclarativeBase):
    pass


class LLMCall(Base):
    __tablename__ = "llm_calls"

    id: Mapped[int] = mapped_column(primary_key=True)
    user_id: Mapped[str] = mapped_column(String(64), index=True)
    feature: Mapped[str] = mapped_column(String(64), index=True)  # "analyze", "chat"...
    model: Mapped[str] = mapped_column(String(64))
    input_tokens: Mapped[int]
    output_tokens: Mapped[int]
    cache_read_tokens: Mapped[int] = mapped_column(default=0)
    cache_write_tokens: Mapped[int] = mapped_column(default=0)
    cost_usd: Mapped[float]
    latency_ms: Mapped[int]
    request_id: Mapped[str | None] = mapped_column(String(64))
    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), server_default=func.now(), index=True
    )


async def init_db() -> None:
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
```

!!! note "Migration"
    `create_all` chỉ phù hợp khi học. Dự án thật dùng [Alembic](https://alembic.sqlalchemy.org/) để quản lý thay đổi schema theo phiên bản.

### 2.1 Tính chi phí

```python title="app/cost.py"
# Giá USD cho mỗi 1 triệu token (input, output). Cập nhật theo bảng giá chính thức.
PRICES = {
    "claude-opus-5": (5.00, 25.00),
    "claude-sonnet-5": (2.00, 10.00),
    "claude-haiku-4-5": (1.00, 5.00),
}
CACHE_READ_MULTIPLIER = 0.1    # token đọc từ cache: khoảng 10% giá input
CACHE_WRITE_MULTIPLIER = 1.25  # token ghi vào cache (TTL 5 phút): khoảng 125% giá input


def cost_usd(model: str, usage) -> float:
    price_in, price_out = PRICES[model]
    cache_read = usage.cache_read_input_tokens or 0
    cache_write = usage.cache_creation_input_tokens or 0
    return (
        usage.input_tokens * price_in
        + cache_read * price_in * CACHE_READ_MULTIPLIER
        + cache_write * price_in * CACHE_WRITE_MULTIPLIER
        + usage.output_tokens * price_out
    ) / 1_000_000
```

`usage.input_tokens` là phần input **không** dùng cache; hai trường cache được tính riêng. Hãy kiểm tra lại hệ số với trang giá chính thức, vì giá có thể thay đổi theo thời gian.

### 2.2 Ghi log sau mỗi lệnh gọi

```python title="app/tracking.py"
import time

from app.cost import cost_usd
from app.db import LLMCall, SessionLocal


async def record_call(user_id: str, feature: str, response, started: float) -> None:
    usage = response.usage
    async with SessionLocal() as session:
        session.add(LLMCall(
            user_id=user_id,
            feature=feature,
            model=response.model,
            input_tokens=usage.input_tokens,
            output_tokens=usage.output_tokens,
            cache_read_tokens=usage.cache_read_input_tokens or 0,
            cache_write_tokens=usage.cache_creation_input_tokens or 0,
            cost_usd=cost_usd(response.model, usage),
            latency_ms=int((time.perf_counter() - started) * 1000),
            request_id=response._request_id,
        ))
        await session.commit()
```

Với endpoint streaming, gọi `record_call` với message cuối lấy từ `stream.get_final_message()`.

Truy vấn báo cáo chi phí theo tính năng:

```sql
SELECT feature,
       date_trunc('day', created_at) AS day,
       count(*)                      AS calls,
       round(sum(cost_usd)::numeric, 2) AS cost_usd,
       percentile_cont(0.95) WITHIN GROUP (ORDER BY latency_ms) AS p95_ms
FROM llm_calls
GROUP BY feature, day
ORDER BY day DESC, cost_usd DESC;
```

## 3. Cache kết quả với Redis

### 3.1 Khi nào nên cache?

| Nên cache | Không nên cache |
|---|---|
| Phân loại, trích xuất, tóm tắt: **cùng input thì cùng output mong muốn** | Chat hội thoại: mỗi lượt phụ thuộc ngữ cảnh riêng |
| Câu hỏi FAQ lặp lại nhiều | Câu trả lời phụ thuộc dữ liệu thay đổi liên tục (tồn kho, giá) |
| Tác vụ tốn kém, gọi lại nhiều lần (ví dụ khi người dùng tải lại trang) | Nội dung mang tính cá nhân, phụ thuộc quyền truy cập |

!!! note "Cache kết quả khác với prompt caching"
    **Cache kết quả** (bài này) lưu toàn bộ câu trả lời, trúng cache thì **không gọi LLM**. **Prompt caching** của nhà cung cấp vẫn gọi LLM nhưng giảm giá phần prompt lặp lại. Hai kỹ thuật bổ trợ nhau. Xem prompt caching ở [track Hiểu LLM](../hieu-llm/03-claude-api.md).

### 3.2 Code

```python title="app/cache.py"
import hashlib
import json

import redis.asyncio as redis
from pydantic import BaseModel

from app.config import settings

r = redis.from_url(settings.redis_url, decode_responses=True)

# Tăng khi đổi prompt hoặc schema, để cache cũ tự động không còn được dùng
PROMPT_VERSION = "analyze-v3"


def cache_key(*parts: str) -> str:
    raw = json.dumps([PROMPT_VERSION, *parts], ensure_ascii=False)
    return "llm:" + hashlib.sha256(raw.encode()).hexdigest()


async def get_cached(key: str, model: type[BaseModel]) -> BaseModel | None:
    raw = await r.get(key)
    return model.model_validate_json(raw) if raw else None


async def set_cached(key: str, value: BaseModel, ttl_seconds: int = 7 * 24 * 3600) -> None:
    await r.set(key, value.model_dump_json(), ex=ttl_seconds)
```

```python title="app/main.py (cập nhật /analyze)"
from app.cache import cache_key, get_cached, set_cached


@app.post("/analyze", response_model=FeedbackAnalysis)
async def analyze(req: AnalyzeRequest, client=Depends(get_client)) -> FeedbackAnalysis:
    key = cache_key(settings.model, req.text.strip())
    if cached := await get_cached(key, FeedbackAnalysis):
        return cached
    response = await client.messages.parse(...)  # như Bài 2
    await set_cached(key, response.parsed_output)
    return response.parsed_output
```

Hai chi tiết quan trọng: **khóa cache chứa phiên bản prompt và tên model** (đổi prompt thì cache cũ tự vô hiệu), và **luôn đặt TTL** để cache không phình mãi.

## 4. Giới hạn tốc độ và ngân sách dùng chung

Bộ đếm trong bộ nhớ ở Bài 2 không hoạt động khi có nhiều server. Chuyển sang Redis, đồng thời giới hạn thêm **số token mỗi ngày**:

```python title="app/ratelimit.py"
from datetime import date

from fastapi import Header, HTTPException

from app.cache import r

REQUESTS_PER_MINUTE = 10
TOKENS_PER_DAY = 200_000


async def rate_limit(x_user_id: str = Header(...)) -> str:
    minute_key = f"rl:req:{x_user_id}"
    count = await r.incr(minute_key)
    if count == 1:
        await r.expire(minute_key, 60)
    if count > REQUESTS_PER_MINUTE:
        raise HTTPException(429, "Bạn gửi quá nhiều yêu cầu, hãy thử lại sau.")

    used = int(await r.get(f"rl:tok:{x_user_id}:{date.today()}") or 0)
    if used >= TOKENS_PER_DAY:
        raise HTTPException(429, "Bạn đã dùng hết hạn mức hôm nay.")
    return x_user_id


async def add_token_usage(user_id: str, tokens: int) -> None:
    key = f"rl:tok:{user_id}:{date.today()}"
    await r.incrby(key, tokens)
    await r.expire(key, 2 * 24 * 3600)
```

Gọi `add_token_usage(user_id, usage.input_tokens + usage.output_tokens)` sau mỗi lệnh gọi LLM.

## 5. Hàng đợi cho tác vụ chạy lâu

### 5.1 Vấn đề

Tóm tắt một tài liệu 100 trang có thể mất vài phút. Giữ một request HTTP mở lâu như vậy gặp đủ loại rắc rối: timeout của load balancer, người dùng tải lại trang, server khởi động lại giữa chừng làm mất kết quả.

Giải pháp: API chỉ **nhận việc và trả về mã tác vụ ngay**, một **worker** riêng xử lý nền, client **hỏi trạng thái** định kỳ (hoặc nhận thông báo qua webhook).

```mermaid
sequenceDiagram
  participant C as Client
  participant A as API
  participant Q as Redis (hàng đợi)
  participant W as Worker
  C->>A: POST /summaries (tài liệu)
  A->>Q: enqueue job
  A-->>C: 202 kèm job_id
  W->>Q: lấy job
  W->>W: gọi LLM (vài phút)
  W->>Q: lưu kết quả
  C->>A: GET /summaries/job_id
  A-->>C: status complete kèm result
```

### 5.2 Worker với Arq

[Arq](https://arq-docs.helpmanual.io/) là thư viện hàng đợi async gọn nhẹ dùng Redis, hợp với code async của FastAPI. (Celery là lựa chọn phổ biến khác, nhiều tính năng hơn nhưng nặng hơn.)

Worker dưới đây dùng kỹ thuật **map-reduce** để tóm tắt tài liệu dài: tóm tắt song song từng phần, rồi gộp các bản tóm tắt lại.

```python title="app/worker.py"
import asyncio

import anthropic
from arq.connections import RedisSettings

from app.config import settings

PART_CHARS = 20_000


async def startup(ctx: dict) -> None:
    ctx["client"] = anthropic.AsyncAnthropic(
        api_key=settings.anthropic_api_key.get_secret_value()
    )


async def _summarize(client, text: str, instruction: str) -> str:
    response = await client.messages.create(
        model=settings.model,
        max_tokens=16000,
        messages=[{"role": "user", "content": f"{instruction}\n\n<document>\n{text}\n</document>"}],
    )
    return next(b.text for b in response.content if b.type == "text")


async def summarize_document(ctx: dict, text: str) -> str:
    client = ctx["client"]
    parts = [text[i:i + PART_CHARS] for i in range(0, len(text), PART_CHARS)]
    limiter = asyncio.Semaphore(4)

    async def summarize_part(part: str) -> str:
        async with limiter:
            return await _summarize(client, part, "Tóm tắt các ý chính của phần tài liệu sau.")

    partials = await asyncio.gather(*(summarize_part(p) for p in parts))  # map
    return await _summarize(                                                # reduce
        client,
        "\n\n".join(partials),
        "Dưới đây là tóm tắt của từng phần một tài liệu. Viết bản tóm tắt tổng hợp, "
        "có cấu trúc, không lặp ý.",
    )


class WorkerSettings:
    functions = [summarize_document]
    on_startup = startup
    redis_settings = RedisSettings.from_dsn(settings.redis_url)
    max_jobs = 5          # số tác vụ chạy đồng thời trên mỗi worker
    job_timeout = 900     # giây
    keep_result = 24 * 3600
```

```bash
uv run arq app.worker.WorkerSettings
```

!!! tip "Tài liệu vừa context window thì sao?"
    Model hiện đại có context rất dài, nên nhiều tài liệu có thể gửi nguyên văn trong **một** lệnh gọi, cho chất lượng tóm tắt mạch lạc hơn map-reduce. Map-reduce vẫn hữu ích khi tài liệu vượt quá context, hoặc khi muốn giảm độ trễ bằng cách xử lý song song.

### 5.3 API nhận việc và trả trạng thái

```python title="app/main.py (tiếp)"
from arq import create_pool
from arq.connections import RedisSettings
from arq.jobs import Job, JobStatus
from fastapi import HTTPException


class SummaryRequest(BaseModel):
    text: str = Field(min_length=1, max_length=2_000_000)
    idempotency_key: str | None = None


@app.post("/summaries", status_code=202)
async def create_summary(req: SummaryRequest) -> dict:
    pool = await create_pool(RedisSettings.from_dsn(settings.redis_url))
    # Cùng idempotency_key thì không tạo job trùng khi client gửi lại
    job = await pool.enqueue_job("summarize_document", req.text, _job_id=req.idempotency_key)
    if job is None:
        return {"job_id": req.idempotency_key, "duplicate": True}
    return {"job_id": job.job_id}


@app.get("/summaries/{job_id}")
async def get_summary(job_id: str) -> dict:
    pool = await create_pool(RedisSettings.from_dsn(settings.redis_url))
    job = Job(job_id, pool)
    status = await job.status()
    if status == JobStatus.not_found:
        raise HTTPException(404, "Không tìm thấy tác vụ")
    if status != JobStatus.complete:
        return {"status": status.value}
    return {"status": "complete", "result": await job.result(timeout=1)}
```

Trong code thật, hãy tạo `pool` một lần trong `lifespan` như client LLM, thay vì mỗi request.

**Idempotency** rất quan trọng với tác vụ tốn tiền: khi mạng chập chờn, client có thể gửi lại cùng một yêu cầu. Nếu không có khóa chống trùng, bạn trả tiền hai lần cho cùng một kết quả.

## Bài tập

**Bài 3.1.** Thêm ghi log `LLMCall` vào cả `/analyze` và `/chat/stream`. Gọi mỗi endpoint vài lần rồi chạy truy vấn báo cáo chi phí.

**Bài 3.2.** Thêm cache cho `/analyze`. Gọi cùng một văn bản hai lần, đo thời gian mỗi lần. Đổi `PROMPT_VERSION` và xác nhận cache cũ không còn được dùng.

**Bài 3.3.** Viết endpoint `GET /usage/me` trả về số token đã dùng hôm nay và chi phí trong tháng của người dùng hiện tại.

??? tip "Gợi ý lời giải"
    ```python
    from sqlalchemy import func, select

    @app.get("/usage/me")
    async def my_usage(user_id: str = Depends(rate_limit)) -> dict:
        async with SessionLocal() as s:
            month_cost = await s.scalar(
                select(func.coalesce(func.sum(LLMCall.cost_usd), 0))
                .where(LLMCall.user_id == user_id,
                       LLMCall.created_at >= func.date_trunc("month", func.now()))
            )
        tokens_today = int(await r.get(f"rl:tok:{user_id}:{date.today()}") or 0)
        return {"tokens_today": tokens_today, "cost_this_month_usd": round(month_cost, 4)}
    ```

**Bài 3.4.** Chạy worker, gửi một tài liệu dài (ví dụ một cuốn sách miễn phí bản quyền) qua `/summaries`, rồi hỏi trạng thái định kỳ. Thử tắt worker giữa chừng rồi bật lại: chuyện gì xảy ra với tác vụ?

**Bài 3.5 (mở rộng).** Thay vì client hỏi trạng thái, hãy stream tiến độ (đã xong bao nhiêu phần) qua SSE, dùng Redis pub/sub để worker báo tiến độ cho API.

## Checklist

- [ ] Mọi lệnh gọi LLM được ghi lại cùng token, chi phí, độ trễ, tính năng và người dùng.
- [ ] Có truy vấn trả lời được "tháng này tốn bao nhiêu, cho tính năng nào".
- [ ] Cache có phiên bản prompt trong khóa và có TTL.
- [ ] Giới hạn tốc độ và hạn mức token dùng chung qua Redis.
- [ ] Tác vụ lâu chạy trong worker, có idempotency key.

## Đọc thêm

- [SQLAlchemy 2.0](https://docs.sqlalchemy.org/en/20/), [Redis](https://redis.io/docs/), [Arq](https://arq-docs.helpmanual.io/)
- [Persistence trong LangGraph](../../oss/python/langgraph/persistence.md): lưu trạng thái agent vào database.

---

**Bài trước:** [Bài 2: FastAPI và streaming](02-fastapi-streaming.md) · [Tổng quan track](index.md) · **Bài tiếp theo:** [Bài 4: Test, Docker và CI](04-test-docker-ci.md)
