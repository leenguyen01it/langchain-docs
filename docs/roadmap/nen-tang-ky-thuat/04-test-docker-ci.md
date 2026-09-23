# Bài 4: Test, Docker và CI

!!! abstract "Tóm tắt bài học"
    - **Mục tiêu:** viết test cho ứng dụng có LLM mà không tốn tiền và không chập chờn, đóng gói bằng Docker, chạy cả hệ thống bằng Docker Compose, và tự động kiểm tra mỗi lần push bằng GitHub Actions.
    - **Thời lượng:** khoảng 1 tuần.
    - **Yêu cầu trước:** [Bài 3](03-du-lieu-hang-doi.md).

## 1. Test ứng dụng LLM khác gì?

Output của LLM **không xác định trước** và **tốn tiền** mỗi lần gọi. Nếu test gọi LLM thật, test sẽ chậm, tốn phí, và thỉnh thoảng thất bại vô cớ. Giải pháp là tách thành nhiều tầng:

| Tầng | Kiểm tra gì | Gọi LLM thật? | Chạy khi nào |
|---|---|---|---|
| **Unit test** | Code xác định: tính chi phí, tạo khóa cache, parse, định dạng prompt | Không | Mỗi lần push |
| **API test** | Endpoint, validate input, xử lý lỗi, luồng dữ liệu | Không, dùng **client giả** | Mỗi lần push |
| **Integration test** | Kết nối thật với API LLM vẫn hoạt động, schema còn hợp lệ | Có, vài request | Hằng đêm, trước khi phát hành |
| **Eval** | **Chất lượng** câu trả lời trên bộ dữ liệu | Có, nhiều request | Khi đổi prompt, model (xem [track Evaluation](../evaluation/index.md)) |

Test kiểm tra **code có đúng không**. Eval kiểm tra **AI có tốt không**. Đừng trộn lẫn hai việc này.

## 2. Unit test cho code xác định

```python title="tests/test_cost.py"
from types import SimpleNamespace

import pytest

from app.cost import cost_usd


def usage(input_tokens=0, output_tokens=0, cache_read=0, cache_write=0):
    return SimpleNamespace(
        input_tokens=input_tokens,
        output_tokens=output_tokens,
        cache_read_input_tokens=cache_read,
        cache_creation_input_tokens=cache_write,
    )


def test_cost_without_cache():
    # 1 triệu input * $5 + 1 triệu output * $25
    assert cost_usd("claude-opus-5", usage(1_000_000, 1_000_000)) == pytest.approx(30.0)


def test_cache_read_is_cheaper_than_regular_input():
    regular = cost_usd("claude-opus-5", usage(input_tokens=100_000))
    cached = cost_usd("claude-opus-5", usage(cache_read=100_000))
    assert cached < regular / 5


def test_unknown_model_raises():
    with pytest.raises(KeyError):
        cost_usd("model-khong-ton-tai", usage())
```

```python title="tests/test_cache.py"
from app.cache import cache_key


def test_same_input_same_key():
    assert cache_key("m", "xin chào") == cache_key("m", "xin chào")


def test_model_changes_key():
    assert cache_key("model-a", "x") != cache_key("model-b", "x")
```

## 3. API test với client giả

Nhờ thiết kế dependency `get_client` ở Bài 2, ta thay client thật bằng một đối tượng giả trả về kết quả định sẵn.

```python title="tests/conftest.py"
import os

# Đặt biến môi trường giả TRƯỚC khi import app, để Settings khởi tạo được
os.environ.setdefault("ANTHROPIC_API_KEY", "test-key")
os.environ.setdefault("DATABASE_URL", "postgresql+asyncpg://test:test@localhost/test")
os.environ.setdefault("REDIS_URL", "redis://localhost:6379/15")

from types import SimpleNamespace  # noqa: E402

import pytest  # noqa: E402
from fastapi.testclient import TestClient  # noqa: E402

from app.main import app, get_client  # noqa: E402
from app.schemas import FeedbackAnalysis  # noqa: E402


class FakeMessages:
    def __init__(self, result=None, error=None):
        self.result, self.error, self.calls = result, error, []

    async def parse(self, **kwargs):
        self.calls.append(kwargs)
        if self.error:
            raise self.error
        return SimpleNamespace(parsed_output=self.result)


class FakeClient:
    def __init__(self, **kwargs):
        self.messages = FakeMessages(**kwargs)


@pytest.fixture(autouse=True)
def clean_redis():
    # Xóa DB Redis dành riêng cho test (số 15), tránh trúng cache của lần chạy trước
    import redis
    redis.Redis.from_url(os.environ["REDIS_URL"]).flushdb()


@pytest.fixture
def fake_client():
    client = FakeClient(result=FeedbackAnalysis(
        sentiment="negative", topics=["giao hàng"], summary="Giao trễ.", urgent=True,
    ))
    app.dependency_overrides[get_client] = lambda: client
    yield client
    app.dependency_overrides.clear()


@pytest.fixture
def api():
    return TestClient(app)
```

```python title="tests/test_api.py"
import anthropic
import httpx2 as httpx

from tests.conftest import FakeClient
from app.main import app, get_client


def test_analyze_returns_structured_result(api, fake_client):
    res = api.post("/analyze", json={"text": "Giao hàng trễ quá"})
    assert res.status_code == 200
    assert res.json()["sentiment"] == "negative"
    # Kiểm tra đúng văn bản người dùng được gửi cho LLM
    sent = fake_client.messages.calls[0]["messages"][0]["content"]
    assert sent == "Giao hàng trễ quá"


def test_empty_text_is_rejected_without_calling_llm(api, fake_client):
    res = api.post("/analyze", json={"text": ""})
    assert res.status_code == 422
    assert fake_client.messages.calls == []


def test_rate_limit_from_provider_maps_to_503(api):
    request = httpx.Request("POST", "https://api.anthropic.com/v1/messages")
    error = anthropic.RateLimitError(
        "rate limited", response=httpx.Response(429, request=request), body=None
    )
    app.dependency_overrides[get_client] = lambda: FakeClient(error=error)
    try:
        res = api.post("/analyze", json={"text": "xin chào"})
        assert res.status_code == 503
        assert "Retry-After" in res.headers
    finally:
        app.dependency_overrides.clear()
```

!!! note "`httpx2` và SDK Anthropic 1.x"
    SDK Anthropic phiên bản 1.x dùng thư viện HTTP `httpx2`. Khi cần tạo đối tượng request, response giả cho exception của SDK, hãy import `httpx2` như trên. Với SDK 0.x cũ, dùng `httpx`.

Những test này **chạy trong mili giây, miễn phí, và luôn cho cùng kết quả**, kiểm tra được phần lớn logic của ứng dụng.

## 4. Integration test với LLM thật

Vẫn cần một vài test gọi API thật để phát hiện những thay đổi bên ngoài (model bị ngừng, schema không còn được chấp nhận). Đánh dấu riêng để không chạy mỗi lần push:

```toml title="pyproject.toml (bổ sung)"
[tool.pytest.ini_options]
markers = ["llm: test gọi API LLM thật (tốn phí, chạy hằng đêm)"]
```

```python title="tests/test_llm_integration.py"
import pytest

from app.llm import analyze


@pytest.mark.llm
def test_obvious_negative_feedback():
    result = analyze("Hàng giả, shop lừa đảo, tôi sẽ báo công an và đòi hoàn tiền ngay.")
    # Chỉ kiểm tra những gì chắc chắn, không so khớp nguyên văn
    assert result.sentiment == "negative"
    assert result.urgent is True
```

Nguyên tắc: integration test chỉ khẳng định những điều **hiển nhiên** (phản hồi cực kỳ tiêu cực thì phải là `negative`). Đánh giá chất lượng tinh tế thuộc về eval.

```bash
uv run pytest -m "not llm"   # nhanh, miễn phí: chạy mỗi lần push
uv run pytest -m llm         # gọi API thật: chạy hằng đêm
```

## 5. Đóng gói bằng Docker

```dockerfile title="Dockerfile"
FROM python:3.12-slim

COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /bin/
WORKDIR /app
ENV UV_COMPILE_BYTECODE=1 UV_LINK_MODE=copy

# Cài dependency trước, tận dụng cache của Docker khi chỉ code thay đổi
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev --no-install-project

COPY app ./app
RUN uv sync --frozen --no-dev

# Không chạy bằng root
RUN useradd --create-home appuser
USER appuser

EXPOSE 8000
CMD ["uv", "run", "--no-sync", "fastapi", "run", "app/main.py", "--port", "8000"]
```

```text title=".dockerignore"
.env
.venv
__pycache__
tests
.git
```

!!! danger "Không đưa `.env` vào image"
    Nếu `.env` bị copy vào image, bất kỳ ai có image đều đọc được API key của bạn. Truyền bí mật qua biến môi trường lúc chạy container (hoặc qua hệ thống quản lý bí mật của nền tảng triển khai), và luôn có `.dockerignore`.

## 6. Chạy cả hệ thống bằng Docker Compose

```yaml title="compose.yaml"
services:
  api:
    build: .
    ports: ["8000:8000"]
    env_file: .env
    environment:
      DATABASE_URL: postgresql+asyncpg://postgres:postgres@db:5432/postgres
      REDIS_URL: redis://redis:6379/0
    depends_on:
      db: { condition: service_healthy }
      redis: { condition: service_started }

  worker:
    build: .
    command: ["uv", "run", "--no-sync", "arq", "app.worker.WorkerSettings"]
    env_file: .env
    environment:
      DATABASE_URL: postgresql+asyncpg://postgres:postgres@db:5432/postgres
      REDIS_URL: redis://redis:6379/0
    depends_on: [db, redis]

  db:
    image: pgvector/pgvector:pg17
    environment: { POSTGRES_PASSWORD: postgres }
    volumes: ["pgdata:/var/lib/postgresql/data"]
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "postgres"]
      interval: 5s
      retries: 10

  redis:
    image: redis:7

volumes:
  pgdata:
```

```bash
docker compose up --build
```

API và worker dùng **cùng một image**, chỉ khác lệnh chạy. Trong mạng nội bộ của Compose, các service gọi nhau bằng tên (`db`, `redis`) thay vì `localhost`.

## 7. CI với GitHub Actions

```yaml title=".github/workflows/ci.yml"
name: CI

on:
  push:
  pull_request:
  schedule:
    - cron: "0 19 * * *"   # 2 giờ sáng giờ Việt Nam, chạy integration test

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      redis:
        image: redis:7
        ports: ["6379:6379"]
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v5
      - run: uv sync --frozen
      - run: uv run ruff check .
      - run: uv run pytest -m "not llm"

  llm-integration:
    if: github.event_name == 'schedule'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v5
      - run: uv sync --frozen
      - run: uv run pytest -m llm
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
```

- Job `test` chạy mỗi lần push và pull request, không cần API key thật.
- Job `llm-integration` chỉ chạy theo lịch, lấy API key từ **GitHub Secrets** (Settings > Secrets and variables > Actions). Nên dùng một API key riêng cho CI, có giới hạn chi tiêu.

## Bài tập

**Bài 4.1.** Viết đủ unit test cho `cost_usd` và `cache_key`, cùng 3 API test cho `/analyze`. Đảm bảo `uv run pytest -m "not llm"` chạy trong vài giây.

**Bài 4.2.** Viết API test cho `/chat/stream` với client giả trả về một stream giả gồm 3 đoạn chữ. Kiểm tra response có đủ 3 sự kiện `delta` và 1 sự kiện `done`.

??? tip "Gợi ý lời giải"
    Client giả cần `messages.stream(...)` trả về một async context manager, có thuộc tính `text_stream` là async iterator và phương thức `get_final_message()`:
    ```python
    class FakeStream:
        def __init__(self, pieces):
            self.pieces = pieces
        async def __aenter__(self):
            return self
        async def __aexit__(self, *exc):
            return False
        @property
        async def text_stream(self):
            for p in self.pieces:
                yield p
        async def get_final_message(self):
            return SimpleNamespace(stop_reason="end_turn",
                                   usage=SimpleNamespace(input_tokens=10, output_tokens=3))
    ```
    Đọc response bằng `api.post(...).text` rồi đếm số lần xuất hiện `event: delta`.

**Bài 4.3.** Viết Dockerfile, chạy `docker compose up --build`, gọi thử API từ máy của bạn. Kiểm tra image không chứa file `.env` (`docker run --rm <image> ls -la /app`).

**Bài 4.4.** Đưa dự án lên GitHub, cấu hình workflow CI. Cố tình làm hỏng một test để thấy CI báo đỏ.

## Checklist

- [ ] Test không gọi LLM thật chạy nhanh, miễn phí, cho kết quả ổn định.
- [ ] Integration test gọi LLM thật được đánh dấu riêng và chạy theo lịch.
- [ ] Docker image không chứa bí mật, không chạy bằng root.
- [ ] Cả hệ thống (API, worker, database, Redis) khởi động bằng một lệnh.
- [ ] CI chạy lint và test mỗi lần push.

## Đọc thêm

- [pytest](https://docs.pytest.org/), [FastAPI testing](https://fastapi.tiangolo.com/tutorial/testing/), [uv với Docker](https://docs.astral.sh/uv/guides/integration/docker/)
- [Unit Testing](../../oss/python/langchain/test/unit-testing.md) và [Integration Testing](../../oss/python/langchain/test/integration-testing.md) cho ứng dụng LangChain.

---

**Bài trước:** [Bài 3: Database, cache và hàng đợi](03-du-lieu-hang-doi.md) · [Tổng quan track](index.md) · **Track tiếp theo:** [Hiểu LLM](../hieu-llm/index.md)
