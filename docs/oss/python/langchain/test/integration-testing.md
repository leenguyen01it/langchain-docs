# Integration testing

> Kiểm thử agent với API LLM thật bằng cách tổ chức test, quản lý key, xử lý flakiness, và kiểm soát chi phí.

Integration test xác minh agent của bạn hoạt động đúng với API model và các dịch vụ bên ngoài. Khác với [unit test](unit-testing.md) dùng fake và mock, integration test thực hiện các lệnh gọi mạng thật để xác nhận các thành phần hoạt động cùng nhau, credential hợp lệ, và độ trễ chấp nhận được.

Vì phản hồi của LLM là nondeterministic, integration test cần chiến lược khác với test phần mềm truyền thống. Hướng dẫn này trình bày cách tổ chức, viết, và chạy integration test cho agent của bạn. Để biết hạ tầng test tổng quát khi đóng góp cho chính LangChain, xem [Contributing to code](https://docs.langchain.com/oss/python/contributing/code#running-tests).

## Tách riêng unit test và integration test

Integration test chạy chậm hơn và cần credential API, nên hãy tách riêng chúng khỏi unit test. Điều này cho phép bạn chạy unit test nhanh sau mỗi thay đổi và dành integration test cho CI hoặc kiểm tra trước khi deploy.

Dùng pytest marker để đánh dấu integration test:

```python
import pytest

@pytest.mark.integration
def test_agent_with_real_model():
    agent = create_agent("claude-sonnet-4-6", tools=[get_weather])
    result = agent.invoke({
        "messages": [HumanMessage(content="What's the weather in SF?")]
    })
    assert len(result["messages"]) > 1
```

Cấu hình pytest để nhận diện marker và loại integration test khỏi lần chạy mặc định:

=== "pytest.ini"

    ```ini
    [pytest]
    markers =
        integration: tests that call real LLM APIs
    addopts = -m "not integration"
    ```

=== "pyproject.toml"

    ```toml
    [tool.pytest.ini_options]
    markers = [
      "integration: tests that call real LLM APIs"
    ]
    addopts = "-m 'not integration'"
    ```

Chạy integration test một cách tường minh:

```bash
pytest -m integration
```

## Quản lý API key

Integration test cần credential API thật. Load chúng từ biến môi trường để key không lọt vào source control.

Dùng fixture `conftest.py` để xác nhận các key cần thiết đã có sẵn:

```python
import os
import pytest

@pytest.fixture(autouse=True)
def check_api_keys():
    if not os.environ.get("OPENAI_API_KEY"):
        pytest.skip("OPENAI_API_KEY not set")
```

Khi phát triển local, lưu key trong file `.env` và load bằng [`python-dotenv`](https://pypi.org/project/python-dotenv/):

```bash title=".env"
OPENAI_API_KEY=sk-...
```

```python title="conftest.py"
from dotenv import load_dotenv

load_dotenv()
```

!!! warning "Cảnh báo"
    Thêm `.env` vào `.gitignore` để tránh commit credential. Trong CI, inject secret thông qua hệ thống quản lý secret của provider (ví dụ GitHub Actions secrets).

## Assert dựa trên cấu trúc, không phải nội dung

Phản hồi của LLM thay đổi giữa các lần chạy. Thay vì assert trên chuỗi output chính xác, hãy xác minh các thuộc tính cấu trúc của phản hồi: loại message, tên tool call, hình dạng argument, và số lượng message.

```python
def test_agent_calls_weather_tool():
    agent = create_agent("claude-sonnet-4-6", tools=[get_weather])
    result = agent.invoke({
        "messages": [HumanMessage(content="What's the weather in SF?")]
    })

    messages = result["messages"]
    tool_calls = [
        tc
        for msg in messages
        if hasattr(msg, "tool_calls")
        for tc in (msg.tool_calls or [])
    ]

    assert any(tc["name"] == "get_weather" for tc in tool_calls)
    assert isinstance(messages[-1], AIMessage)
    assert len(messages[-1].content) > 0
```

!!! tip "Mẹo"
    Để assert trajectory chặt chẽ hơn, dùng evaluator [AgentEvals](evals.md), hỗ trợ các chế độ fuzzy matching như `unordered` và `superset`.

## Giảm chi phí và độ trễ

Integration test gọi API LLM phát sinh chi phí thật. Một vài thực hành giúp giữ test suite nhanh và tiết kiệm:

* **Dùng model nhỏ hơn**: `gemini-3.1-flash-lite` hoặc tương đương cho các test chỉ cần xác minh tool calling và cấu trúc phản hồi.
* **Đặt `maxTokens`**: giới hạn độ dài phản hồi để tránh completion dài, tốn kém.
* **Giới hạn phạm vi test**: kiểm thử một hành vi cho mỗi test. Tránh kịch bản end-to-end nối chuỗi nhiều lệnh gọi LLM khi một test đơn lượt là đủ.
* **Chạy có chọn lọc**: dùng cách tách test [ở trên](#tach-rieng-unit-test-va-integration-test) để chỉ chạy integration test trong CI hoặc trước khi deploy, không phải mỗi lần lưu file.

```python
agent = create_agent(
    "gemini-3.1-flash-lite",
    tools=[get_weather],
    model_kwargs={"max_tokens": 256},
)
```

## Ghi và phát lại lệnh gọi HTTP

Với các test chạy thường xuyên trong CI, bạn có thể ghi lại tương tác HTTP ở lần chạy đầu tiên và phát lại ở các lần chạy sau mà không cần gọi API thật. Điều này loại bỏ chi phí và độ trễ sau lần ghi ban đầu.

[`vcrpy`](https://pypi.org/project/vcrpy/1.5.2/) ghi các cặp request/response HTTP vào file "cassette" định dạng YAML. Plugin [`pytest-recording`](https://pypi.org/project/pytest-recording/) tích hợp việc này với pytest.

Thiết lập `conftest.py` để lọc thông tin nhạy cảm khỏi cassette:

```py title="conftest.py"
import pytest

@pytest.fixture(scope="session")
def vcr_config():
    return {
        "filter_headers": [
            ("authorization", "XXXX"),
            ("x-api-key", "XXXX"),
        ],
        "filter_query_parameters": [
            ("api_key", "XXXX"),
            ("key", "XXXX"),
        ],
    }
```

Cấu hình project để nhận diện marker `vcr`:

=== "pytest.ini"

    ```ini
    [pytest]
    markers =
        vcr: record/replay HTTP via VCR
    addopts = --record-mode=once
    ```

=== "pyproject.toml"

    ```toml
    [tool.pytest.ini_options]
    markers = [
      "vcr: record/replay HTTP via VCR"
    ]
    addopts = "--record-mode=once"
    ```

!!! info "Thông tin"
    Tuỳ chọn `--record-mode=once` ghi lại tương tác HTTP ở lần chạy đầu tiên và phát lại ở các lần chạy sau.

Đánh dấu test của bạn với marker `vcr`:

```python
@pytest.mark.vcr()
def test_agent_trajectory():
    agent = create_agent("claude-sonnet-4-6", tools=[get_weather])
    result = agent.invoke({
        "messages": [HumanMessage(content="What's the weather in SF?")]
    })
    assert any(
        tc["name"] == "get_weather"
        for msg in result["messages"]
        if hasattr(msg, "tool_calls")
        for tc in (msg.tool_calls or [])
    )
```

Lần chạy đầu tiên thực hiện lệnh gọi mạng thật và tạo file cassette trong `tests/cassettes/`. Các lần chạy sau sẽ phát lại phản hồi đã ghi.

!!! warning "Cảnh báo"
    Khi bạn sửa prompt, thêm tool mới, hoặc thay đổi trajectory kỳ vọng, cassette đã lưu sẽ trở nên lỗi thời và các test hiện có **sẽ fail**. Xoá các file cassette tương ứng và chạy lại test để ghi lại tương tác mới.

## Bước tiếp theo

Tìm hiểu cách đánh giá trajectory của agent bằng so khớp deterministic hoặc evaluator LLM-as-judge trong [Evals](evals.md).
