# Middleware dựng sẵn

> Middleware dựng sẵn cho các trường hợp sử dụng agent phổ biến

LangChain và [Deep Agents](https://docs.langchain.com/oss/python/deepagents/overview) cung cấp các middleware dựng sẵn cho những trường hợp sử dụng phổ biến. Mỗi middleware đều sẵn sàng cho production và có thể cấu hình theo nhu cầu cụ thể của bạn.

## Middleware không phụ thuộc nhà cung cấp

Các middleware sau hoạt động với mọi nhà cung cấp LLM:

| Middleware                                    | Mô tả                                                                                          |
| ---------------------------------------------- | ----------------------------------------------------------------------------------------------- |
| [Tool error](#tool-error)                      | Bắt các exception phát sinh khi thực thi tool và chuyển đổi chúng thành thông báo lỗi cho model. |
| [Tool retry](#tool-retry)                      | Tự động thử lại các lệnh gọi tool thất bại với backoff theo cấp số nhân.                          |
| [Model retry](#model-retry)                    | Tự động thử lại các lệnh gọi model thất bại với backoff theo cấp số nhân.                         |
| [Model fallback](#model-fallback)              | Tự động chuyển sang model thay thế khi model chính gặp lỗi.                                       |
| [Summarization](#summarization)                | Tự động tóm tắt lịch sử hội thoại khi gần đạt giới hạn token.                                     |
| [Human-in-the-loop](#human-in-the-loop)        | Tạm dừng thực thi để chờ con người phê duyệt lệnh gọi tool.                                       |
| [Model call limit](#model-call-limit)          | Giới hạn số lượng lệnh gọi model để tránh chi phí phát sinh quá mức.                               |
| [Tool call limit](#tool-call-limit)            | Kiểm soát việc thực thi tool bằng cách giới hạn số lần gọi.                                        |
| [PII detection](#pii-detection)                | Phát hiện và xử lý Thông tin nhận dạng cá nhân (PII).                                              |
| [To-do list](#to-do-list)                      | Trang bị cho agent khả năng lập kế hoạch và theo dõi tác vụ.                                       |
| [LLM tool selector](#llm-tool-selector)        | Dùng một LLM để chọn các tool liên quan trước khi gọi model chính.                                 |
| [Provider tool search](#provider-tool-search)  | Trì hoãn việc nạp tool phía sau cơ chế tìm kiếm tool phía server của nhà cung cấp, chỉ hiển thị chúng khi cần. |
| [Shell tool](#shell-tool)                      | Cung cấp cho agent một phiên shell liên tục để thực thi lệnh.                                      |
| [Filesystem](#filesystem-middleware)           | Cung cấp cho agent một hệ thống tệp để lưu trữ context và bộ nhớ dài hạn.                          |
| [Subagent](#subagent)                          | Thêm khả năng tạo (spawn) các subagent.                                                            |
| [Rubric grading (Beta)](#rubric-grading)       | Áp dụng chấm điểm kiểu LLM-as-a-judge để agent tự đánh giá và lặp lại cho đến khi thỏa mãn rubric.  |
| [File search](#file-search)                    | Cung cấp các tool tìm kiếm Glob và Grep trên các tệp trong hệ thống tệp.                           |
| [Context editing](#context-editing)            | Quản lý context hội thoại bằng cách cắt bớt hoặc xóa các lượt sử dụng tool.                        |
| [LLM tool emulator](#llm-tool-emulator)        | Giả lập việc thực thi tool bằng một LLM cho mục đích kiểm thử.                                     |

### Tool error

Bắt các exception phát sinh trong quá trình thực thi tool và chuyển đổi chúng thành các `ToolMessage` báo lỗi mà model có thể nhìn thấy và khôi phục, thay vì làm dừng hẳn phiên chạy của agent. Tool error hữu ích cho các trường hợp sau:

* Cho phép model thử lại một lệnh gọi tool thất bại với các tham số đã được sửa.
* Hiển thị các thông báo lỗi được kiểm soát, đã làm sạch thay vì chi tiết exception thô.
* Ngăn các exception tool bất ngờ làm crash agent.

!!! note "Ghi chú"
    Tool error middleware không tự động thử lại các lệnh gọi thất bại. Để thử lại, hãy kết hợp với middleware [Tool retry](#tool-retry) đặt ở vị trí *bên trong* (sớm hơn trong danh sách `middleware`) và cấu hình `on_failure="error"` để exception được truyền tới tool error middleware. Xem [ví dụ đầy đủ](#tool-error-full-example) bên dưới.

**Tham chiếu API:** [`ToolErrorMiddleware`](https://reference.langchain.com/python/langchain/agents/middleware/tool_error/ToolErrorMiddleware)

!!! note "Ghi chú"
    `ToolErrorMiddleware` yêu cầu `langchain>=1.3.14`.

```python
from langchain.agents import create_agent
from langchain.agents.middleware import ToolErrorMiddleware


def on_error(exc: Exception, request: ToolCallRequest) -> str | None:
    if isinstance(exc, ValueError):
        return f"`{request.tool_call['name']}` failed with {type(exc).__name__}."
    # propagate everything else


agent = create_agent(
    model="gpt-5.5",
    tools=[your_tools],
    middleware=[ToolErrorMiddleware(on_error)],
)
```

??? note "Tùy chọn cấu hình"
    **`on_error`** (`Callable[[Exception, ToolCallRequest], str | list[ContentBlock] | None]`)
        Trình xử lý đồng bộ được gọi cho mỗi exception phát sinh khi thực thi tool. Trả về nội dung (một `str` hoặc danh sách content block) để chuyển đổi exception đó thành `ToolMessage(status="error")`. Trả về `None` hoặc bỏ qua câu lệnh return để cho phép exception lan truyền tiếp. Được dùng trên đường đồng bộ (sync) và, trừ khi có cung cấp `aon_error`, cả trên đường bất đồng bộ (async).

    **`aon_error`** (`Callable[[Exception, ToolCallRequest], Awaitable[str | list[ContentBlock] | None]]`)
        Trình xử lý bất đồng bộ tùy chọn, dùng trên đường thực thi async. Nếu không được cung cấp sẽ dùng lại `on_error`.

    **`tools`** (`list[BaseTool | str]`)
        Danh sách tool hoặc tên tool tùy chọn để áp dụng xử lý lỗi. Nếu `None`, áp dụng cho tất cả các tool.

??? note "Ví dụ đầy đủ về Tool error"
    Trình xử lý `on_error` nhận exception và `ToolCallRequest` (bao gồm dict tool call với tên, tham số và call ID). Trả về `None` cho các exception bạn không muốn xử lý, và chúng sẽ lan truyền bình thường.

    ```python
    from langchain.agents import create_agent
    from langchain.agents.middleware import ToolErrorMiddleware, ToolRetryMiddleware


    def on_error(exc: Exception, request: ToolCallRequest) -> str | None:
        # Surface ValueError to the model so it can correct the input
        if isinstance(exc, ValueError):
            return f"`{request.tool_call['name']}` failed: {type(exc).__name__}. Fix the input and retry."
        # Let all other exceptions propagate (halts the run)
        return None


    # Async-only usage
    async def aon_error(exc: Exception, request: ToolCallRequest) -> str | None:
        if isinstance(exc, ConnectionError):
            return f"Tool `{request.tool_call['name']}` encountered a connection error."
        return None


    agent = create_agent(
        model="gpt-5.5",
        tools=[search_tool, database_tool],
        middleware=[
            # Place retry inner so exceptions reach ToolErrorMiddleware after retries are exhausted
            ToolRetryMiddleware(max_retries=3, on_failure="error"),
            ToolErrorMiddleware(on_error=on_error, tools=["search_tool"]),
        ],
    )

    # Async-only: pass aon_error alone (do not pass on_error)
    async_agent = create_agent(
        model="gpt-5.5",
        tools=[api_tool],
        middleware=[ToolErrorMiddleware(aon_error=aon_error)],
    )
    ```

    !!! note "Ghi chú"
        Nên ưu tiên trả về nội dung nêu rõ loại exception hơn là thông báo exception thô, vì thông báo thô có thể chứa chi tiết nhạy cảm hoặc nội bộ. Trình xử lý `on_error` kiểm soát mức độ tiết lộ: thông báo exception thô sẽ không bao giờ được gửi tới model trừ khi bạn chủ động chọn đưa vào.

### Tool retry

Tự động thử lại các lệnh gọi tool thất bại với cấu hình backoff theo cấp số nhân. Tool retry hữu ích cho các trường hợp sau:

* Xử lý các lỗi tạm thời trong lệnh gọi API bên ngoài.
* Cải thiện độ tin cậy của các tool phụ thuộc mạng.
* Xây dựng agent bền bỉ, có thể xử lý ổn thỏa các lỗi tạm thời.

**Tham chiếu API:** [`ToolRetryMiddleware`](https://reference.langchain.com/python/langchain/agents/middleware/tool_retry/ToolRetryMiddleware)

```python
from langchain.agents import create_agent
from langchain.agents.middleware import ToolRetryMiddleware

agent = create_agent(
    model="gpt-5.5",
    tools=[search_tool, database_tool],
    middleware=[
        ToolRetryMiddleware(
            max_retries=3,
            backoff_factor=2.0,
            initial_delay=1.0,
        ),
    ],
)
```

??? note "Tùy chọn cấu hình"
    **`max_retries`** (`number`, mặc định `2`)
        Số lần thử lại tối đa sau lần gọi đầu tiên (tổng cộng 3 lần thử với giá trị mặc định)

    **`tools`** (`list[BaseTool | str]`)
        Danh sách tool hoặc tên tool tùy chọn để áp dụng logic thử lại. Nếu `None`, áp dụng cho tất cả các tool.

    **`retry_on`** (`tuple[type[Exception], ...] | callable`, mặc định `(Exception,)`)
        Là một tuple các loại exception cần thử lại, hoặc một callable nhận vào một exception và trả về `True` nếu nó cần được thử lại. Theo mặc định, mọi exception đều được thử lại. Các exception không khớp sẽ lan truyền ngay lập tức và không được `on_failure` xử lý.

    **`on_failure`** (`string | callable`, mặc định `continue`)
        Hành vi khi đã hết số lần thử lại. Các lựa chọn:

        * `'continue'` (mặc định): trả về một `ToolMessage` kèm chi tiết lỗi, cho phép LLM tự xử lý lỗi
        * `'error'`: re-raise exception, dừng thực thi agent
        * Callable tùy chỉnh: hàm nhận exception và trả về chuỗi làm nội dung cho `ToolMessage`

        **Giá trị đã lỗi thời:** `'return_message'` (dùng `'continue'` thay thế) và `'raise'` (dùng `'error'` thay thế).

    **`backoff_factor`** (`number`, mặc định `2.0`)
        Hệ số nhân cho backoff theo cấp số nhân. Mỗi lần thử lại chờ `initial_delay * (backoff_factor ** retry_number)` giây. Đặt `0.0` để có độ trễ cố định.

    **`initial_delay`** (`number`, mặc định `1.0`)
        Độ trễ ban đầu tính bằng giây trước lần thử lại đầu tiên

    **`max_delay`** (`number`, mặc định `60.0`)
        Độ trễ tối đa tính bằng giây giữa các lần thử lại (giới hạn mức tăng của backoff theo cấp số nhân)

    **`jitter`** (`boolean`, mặc định `true`)
        Có thêm độ trễ ngẫu nhiên (`±25%`) hay không để tránh hiệu ứng "thundering herd"

??? note "Ví dụ đầy đủ"
    Middleware này tự động thử lại các lệnh gọi tool thất bại với backoff theo cấp số nhân.

    **Cấu hình chính:**

    * `max_retries`: số lần thử lại (mặc định: 2)
    * `backoff_factor`: hệ số nhân cho backoff theo cấp số nhân (mặc định: 2.0)
    * `initial_delay`: độ trễ khởi đầu tính bằng giây (mặc định: 1.0)
    * `max_delay`: giới hạn mức tăng của độ trễ (mặc định: 60.0)
    * `jitter`: thêm biến thiên ngẫu nhiên (mặc định: True)

    **Xử lý khi thất bại:**

    * `on_failure='continue'` (mặc định): trả về thông báo lỗi
    * `on_failure='error'`: re-raise exception
    * Hàm tùy chỉnh: hàm trả về thông báo lỗi

    ```python
    from langchain.agents import create_agent
    from langchain.agents.middleware import ToolRetryMiddleware


    agent = create_agent(
        model="gpt-5.5",
        tools=[search_tool, database_tool, api_tool],
        middleware=[
            ToolRetryMiddleware(
                max_retries=3,
                backoff_factor=2.0,
                initial_delay=1.0,
                max_delay=60.0,
                jitter=True,
                tools=["api_tool"],
                retry_on=(ConnectionError, TimeoutError),
                on_failure="continue",
            ),
        ],
    )
    ```

### Model retry

Tự động thử lại các lệnh gọi model thất bại với cấu hình backoff theo cấp số nhân. Model retry hữu ích cho các trường hợp sau:

* Xử lý các lỗi tạm thời trong lệnh gọi API của model.
* Cải thiện độ tin cậy của các yêu cầu model phụ thuộc mạng.
* Xây dựng agent bền bỉ, có thể xử lý ổn thỏa các lỗi model tạm thời.

**Tham chiếu API:** [`ModelRetryMiddleware`](https://reference.langchain.com/python/langchain/agents/middleware/model_retry/ModelRetryMiddleware)

```python
from langchain.agents import create_agent
from langchain.agents.middleware import ModelRetryMiddleware

agent = create_agent(
    model="gpt-5.5",
    tools=[search_tool, database_tool],
    middleware=[
        ModelRetryMiddleware(
            max_retries=3,
            backoff_factor=2.0,
            initial_delay=1.0,
        ),
    ],
)
```

??? note "Tùy chọn cấu hình"
    **`max_retries`** (`number`, mặc định `2`)
        Số lần thử lại tối đa sau lần gọi đầu tiên (tổng cộng 3 lần thử với giá trị mặc định)

    **`retry_on`** (`tuple[type[Exception], ...] | callable`, mặc định `(Exception,)`)
        Là một tuple các loại exception cần thử lại, hoặc một callable nhận vào một exception và trả về `True` nếu nó cần được thử lại.

    **`on_failure`** (`string | callable`, mặc định `continue`)
        Hành vi khi đã hết số lần thử lại. Các lựa chọn:

        * `'continue'` (mặc định): trả về một `AIMessage` kèm chi tiết lỗi, cho phép agent có khả năng xử lý ổn thỏa lỗi này
        * `'error'`: re-raise exception (dừng thực thi agent)
        * Callable tùy chỉnh: hàm nhận exception và trả về chuỗi làm nội dung cho `AIMessage`

    **`backoff_factor`** (`number`, mặc định `2.0`)
        Hệ số nhân cho backoff theo cấp số nhân. Mỗi lần thử lại chờ `initial_delay * (backoff_factor ** retry_number)` giây. Đặt `0.0` để có độ trễ cố định.

    **`initial_delay`** (`number`, mặc định `1.0`)
        Độ trễ ban đầu tính bằng giây trước lần thử lại đầu tiên

    **`max_delay`** (`number`, mặc định `60.0`)
        Độ trễ tối đa tính bằng giây giữa các lần thử lại (giới hạn mức tăng của backoff theo cấp số nhân)

    **`jitter`** (`boolean`, mặc định `true`)
        Có thêm độ trễ ngẫu nhiên (`±25%`) hay không để tránh hiệu ứng "thundering herd"

??? note "Ví dụ đầy đủ"
    Middleware này tự động thử lại các lệnh gọi model thất bại với backoff theo cấp số nhân.

    ```python
    from langchain.agents import create_agent
    from langchain.agents.middleware import ModelRetryMiddleware


    # Basic usage with default settings (2 retries, exponential backoff)
    agent = create_agent(
        model="gpt-5.5",
        tools=[search_tool],
        middleware=[ModelRetryMiddleware()],
    )

    # Custom exception filtering
    class TimeoutError(Exception):
        """Custom exception for timeout errors."""
        pass

    class ConnectionError(Exception):
        """Custom exception for connection errors."""
        pass

    # Retry specific exceptions only
    retry = ModelRetryMiddleware(
        max_retries=4,
        retry_on=(TimeoutError, ConnectionError),
        backoff_factor=1.5,
    )


    def should_retry(error: Exception) -> bool:
        # Only retry on rate limit errors
        if isinstance(error, TimeoutError):
            return True
        # Or check for specific HTTP status codes
        if hasattr(error, "status_code"):
            return error.status_code in (429, 503)
        return False

    retry_with_filter = ModelRetryMiddleware(
        max_retries=3,
        retry_on=should_retry,
    )

    # Return error message instead of raising
    retry_continue = ModelRetryMiddleware(
        max_retries=4,
        on_failure="continue",  # Return AIMessage with error instead of raising
    )

    # Custom error message formatting
    def format_error(error: Exception) -> str:
        return f"Model call failed: {error}. Please try again later."

    retry_with_formatter = ModelRetryMiddleware(
        max_retries=4,
        on_failure=format_error,
    )

    # Constant backoff (no exponential growth)
    constant_backoff = ModelRetryMiddleware(
        max_retries=5,
        backoff_factor=0.0,  # No exponential growth
        initial_delay=2.0,  # Always wait 2 seconds
    )

    # Raise exception on failure
    strict_retry = ModelRetryMiddleware(
        max_retries=2,
        on_failure="error",  # Re-raise exception instead of returning message
    )
    ```

### Model fallback

Tự động chuyển sang các model thay thế khi model chính gặp lỗi. Model fallback hữu ích cho các trường hợp sau:

* Xây dựng agent bền bỉ, có thể xử lý tình trạng model ngừng hoạt động.
* Tối ưu chi phí bằng cách chuyển sang các model rẻ hơn.
* Dự phòng nhà cung cấp giữa OpenAI, Anthropic, v.v.

**Tham chiếu API:** [`ModelFallbackMiddleware`](https://reference.langchain.com/python/langchain/agents/middleware/model_fallback/ModelFallbackMiddleware)

```python
from langchain.agents import create_agent
from langchain.agents.middleware import ModelFallbackMiddleware

agent = create_agent(
    model="gpt-5.5",
    tools=[],
    middleware=[
        ModelFallbackMiddleware(
            "gpt-5.4-mini",
            "claude-3-5-sonnet-20241022",
        ),
    ],
)
```

Xem [video hướng dẫn](https://www.youtube.com/watch?v=8rCRO0DUeIM) minh họa hành vi của middleware Model Fallback.

??? note "Tùy chọn cấu hình"
    **`first_model`** (`string | BaseChatModel`, bắt buộc)
        Model dự phòng đầu tiên được thử khi model chính gặp lỗi. Có thể là một chuỗi định danh model (ví dụ: `'openai:gpt-5.4-mini'`) hoặc một instance `BaseChatModel`.

    **`*additional_models`** (`string | BaseChatModel`)
        Các model dự phòng bổ sung được thử lần lượt theo thứ tự nếu các model trước đó gặp lỗi

### Summarization

Tự động tóm tắt lịch sử hội thoại khi gần đạt giới hạn token, giữ lại các tin nhắn gần đây trong khi nén context cũ hơn. Summarization hữu ích cho các trường hợp sau:

* Các cuộc hội thoại kéo dài vượt quá giới hạn context window.
* Các hội thoại nhiều lượt với lịch sử dài.
* Các ứng dụng cần giữ nguyên context hội thoại đầy đủ.

!!! note "Ghi chú"
    Summarization là kỹ thuật nén context hướng văn bản. Nó không thay đổi kích thước, giảm mẫu, hay nén các payload hình ảnh/âm thanh/video theo cách khác. Các tin nhắn gần đây được giữ lại bởi `keep` vẫn bao gồm nguyên các block đa phương tiện gốc, trong khi các tin nhắn đa phương tiện cũ hơn bị tóm tắt chỉ còn được biểu diễn bằng đoạn tóm tắt văn bản được tạo ra. Đối với các ứng dụng dùng nhiều hình ảnh, hãy lưu media trong một filesystem hoặc object store và truyền URL hoặc tham chiếu tệp qua lịch sử tin nhắn.

**Tham chiếu API:** [`SummarizationMiddleware`](https://reference.langchain.com/python/langchain/agents/middleware/summarization/SummarizationMiddleware)

```python
from langchain.agents import create_agent
from langchain.agents.middleware import SummarizationMiddleware

agent = create_agent(
    model="gpt-5.5",
    tools=[your_weather_tool, your_calculator_tool],
    middleware=[
        SummarizationMiddleware(
            model="gpt-5.4-mini",
            trigger=("tokens", 4000),
            keep=("messages", 20),
        ),
    ],
)
```

??? note "Tùy chọn cấu hình"
    !!! tip "Mẹo"
        Điều kiện `fraction` cho `trigger` và `keep` (được nêu bên dưới) phụ thuộc vào [dữ liệu profile](../models.md#model-profiles) của chat model nếu dùng `langchain>=1.1`. Nếu không có dữ liệu này, hãy dùng một điều kiện khác hoặc chỉ định thủ công:

        ```python
        from langchain.chat_models import init_chat_model

        custom_profile = {
            "max_input_tokens": 100_000,
            # ...
        }
        model = init_chat_model("gpt-5.5", profile=custom_profile)
        ```

    **`model`** (`string | BaseChatModel`, bắt buộc)
        Model dùng để tạo bản tóm tắt. Có thể là một chuỗi định danh model (ví dụ: `'openai:gpt-5.4-mini'`) hoặc một instance `BaseChatModel`. Xem [`init_chat_model`](https://reference.langchain.com/python/langchain/chat_models/base/init_chat_model) để biết thêm thông tin.

    **`trigger`** (`ContextSize | TriggerClause | list[ContextSize | TriggerClause] | None`)
        (Các) điều kiện để kích hoạt summarization. Có thể là:

        * Một tuple [`ContextSize`](https://reference.langchain.com/python/langchain/agents/middleware/summarization/ContextSize) duy nhất (ngưỡng được chỉ định phải được đáp ứng)
        * Một dict [`TriggerClause`](https://reference.langchain.com/python/langchain/agents/middleware/summarization/TriggerClause) duy nhất (tất cả các ngưỡng được chỉ định phải được đáp ứng, logic AND)
        * Một danh sách kết hợp cả hai dạng trên (chỉ cần một mục được đáp ứng, logic OR)

        Các ngưỡng được hỗ trợ là:

        * `fraction` (float): tỷ lệ so với kích thước context của model (0-1)
        * `tokens` (int): số lượng token tuyệt đối
        * `messages` (int): số lượng tin nhắn

        Một tuple [`ContextSize`](https://reference.langchain.com/python/langchain/agents/middleware/summarization/ContextSize) biểu diễn đúng một ngưỡng. Một dict [`TriggerClause`](https://reference.langchain.com/python/langchain/agents/middleware/summarization/TriggerClause) có thể bao gồm một hoặc nhiều ngưỡng, ví dụ `{"tokens": 4000, "messages": 10}`, và tất cả các ngưỡng trong dict đó phải được đáp ứng (AND).

        Mỗi dict [`TriggerClause`](https://reference.langchain.com/python/langchain/agents/middleware/summarization/TriggerClause) phải chỉ định ít nhất một ngưỡng. Nếu không cung cấp `trigger`, summarization sẽ không tự động kích hoạt.

        Xem tham chiếu API cho [`ContextSize`](https://reference.langchain.com/python/langchain/agents/middleware/summarization/ContextSize) và [`TriggerClause`](https://reference.langchain.com/python/langchain/agents/middleware/summarization/TriggerClause) để biết thêm thông tin.

    **`keep`** (`ContextSize`, mặc định `('messages', 20)`)
        Lượng context cần giữ lại sau khi tóm tắt. Chỉ định đúng một trong:

        * `fraction` (float): tỷ lệ giữ lại so với kích thước context của model (0-1)
        * `tokens` (int): số lượng token tuyệt đối cần giữ lại
        * `messages` (int): số lượng tin nhắn gần đây cần giữ lại

        Xem tham chiếu API cho [`ContextSize`](https://reference.langchain.com/python/langchain/agents/middleware/summarization/ContextSize) để biết thêm thông tin.

    **`token_counter`** (`function`)
        Hàm đếm token tùy chỉnh. Mặc định đếm theo số ký tự.

    **`summary_prompt`** (`string`)
        Mẫu prompt tùy chỉnh cho việc tóm tắt. Dùng mẫu dựng sẵn nếu không được chỉ định. Mẫu cần bao gồm placeholder `{messages}` nơi lịch sử hội thoại sẽ được chèn vào.

    **`trim_tokens_to_summarize`** (`number`, mặc định `4000`)
        Số lượng token tối đa được đưa vào khi tạo bản tóm tắt. Các tin nhắn sẽ được cắt bớt để phù hợp với giới hạn này trước khi tóm tắt.

    **`summary_prefix`** (`string`, đã lỗi thời)
        **Đã lỗi thời:** dùng `summary_prompt` để cung cấp toàn bộ prompt thay thế.

    **`max_tokens_before_summary`** (`number`, đã lỗi thời)
        **Đã lỗi thời:** dùng `trigger: ("tokens", value)` thay thế. Ngưỡng token để kích hoạt summarization.

    **`messages_to_keep`** (`number`, đã lỗi thời)
        **Đã lỗi thời:** dùng `keep: ("messages", value)` thay thế. Số tin nhắn gần đây cần giữ lại.

??? note "Ví dụ đầy đủ"
    Middleware summarization theo dõi số lượng token của tin nhắn và tự động tóm tắt các tin nhắn cũ khi đạt ngưỡng.

    **Điều kiện trigger** kiểm soát thời điểm summarization chạy:

    * Một ngưỡng duy nhất sẽ kích hoạt khi ngưỡng đó được đáp ứng
    * Một trigger clause với nhiều ngưỡng chỉ kích hoạt khi tất cả ngưỡng được đáp ứng (logic AND)
    * Một danh sách các điều kiện trigger sẽ kích hoạt khi bất kỳ mục nào được đáp ứng (logic OR)
    * Mỗi ngưỡng có thể dùng `fraction` (so với kích thước context của model), `tokens` (số lượng tuyệt đối), hoặc `messages` (số lượng tin nhắn)

    **Điều kiện keep** kiểm soát lượng context cần giữ lại (chỉ định đúng một):

    * `fraction`: tỷ lệ giữ lại so với kích thước context của model
    * `tokens`: số lượng token tuyệt đối cần giữ lại
    * `messages`: số lượng tin nhắn gần đây cần giữ lại

    ```python
    from langchain.agents import create_agent
    from langchain.agents.middleware import SummarizationMiddleware


    # Single condition: trigger if tokens >= 4000
    agent = create_agent(
        model="gpt-5.5",
        tools=[your_weather_tool, your_calculator_tool],
        middleware=[
            SummarizationMiddleware(
                model="gpt-5.4-mini",
                trigger=("tokens", 4000),
                keep=("messages", 20),
            ),
        ],
    )

    # Multiple conditions: trigger if number of tokens >= 3000 OR messages >= 6
    agent2 = create_agent(
        model="gpt-5.5",
        tools=[your_weather_tool, your_calculator_tool],
        middleware=[
            SummarizationMiddleware(
                model="gpt-5.4-mini",
                trigger=[
                    ("tokens", 3000),
                    ("messages", 6),
                ],
                keep=("messages", 20),
            ),
        ],
    )

    # AND logic: trigger only when tokens >= 4000 AND messages >= 10
    agent3 = create_agent(
        model="gpt-5.5",
        tools=[your_weather_tool, your_calculator_tool],
        middleware=[
            SummarizationMiddleware(
                model="gpt-5.4-mini",
                trigger={"tokens": 4000, "messages": 10},
                keep=("messages", 20),
            ),
        ],
    )

    # Combine AND and OR: trigger if (tokens >= 5000 AND messages >= 3)
    # OR (tokens >= 3000 AND messages >= 6)
    agent4 = create_agent(
        model="gpt-5.5",
        tools=[your_weather_tool, your_calculator_tool],
        middleware=[
            SummarizationMiddleware(
                model="gpt-5.4-mini",
                trigger=[
                    {"tokens": 5000, "messages": 3},
                    {"tokens": 3000, "messages": 6},
                ],
                keep=("messages", 20),
            ),
        ],
    )

    # Using fractional limits
    agent5 = create_agent(
        model="gpt-5.5",
        tools=[your_weather_tool, your_calculator_tool],
        middleware=[
            SummarizationMiddleware(
                model="gpt-5.4-mini",
                trigger=("fraction", 0.8),
                keep=("fraction", 0.3),
            ),
        ],
    )
    ```

### Human-in-the-loop

Tạm dừng thực thi agent để chờ con người phê duyệt, chỉnh sửa, hoặc từ chối các lệnh gọi tool trước khi chúng được thực thi. [Human-in-the-loop](../human-in-the-loop.md) hữu ích cho các trường hợp sau:

* Các thao tác quan trọng, rủi ro cao cần con người phê duyệt (ví dụ: ghi vào database, giao dịch tài chính).
* Các quy trình tuân thủ (compliance) nơi việc giám sát của con người là bắt buộc.
* Các cuộc hội thoại kéo dài nơi phản hồi của con người định hướng agent.

**Tham chiếu API:** [`HumanInTheLoopMiddleware`](https://reference.langchain.com/python/langchain/agents/middleware/human_in_the_loop/HumanInTheLoopMiddleware)

!!! warning "Cảnh báo"
    Human-in-the-loop middleware yêu cầu một [checkpointer](https://docs.langchain.com/oss/python/langgraph/checkpointers#checkpoints) để duy trì state qua các lần bị gián đoạn.

```python
from langchain.agents import create_agent
from langchain.agents.middleware import HumanInTheLoopMiddleware
from langgraph.checkpoint.memory import InMemorySaver


def your_read_email_tool(email_id: str) -> str:
    """Mock function to read an email by its ID."""
    return f"Email content for ID: {email_id}"

def your_send_email_tool(recipient: str, subject: str, body: str) -> str:
    """Mock function to send an email."""
    return f"Email sent to {recipient} with subject '{subject}'"

agent = create_agent(
    model="gpt-5.5",
    tools=[your_read_email_tool, your_send_email_tool],
    checkpointer=InMemorySaver(),
    middleware=[
        HumanInTheLoopMiddleware(
            interrupt_on={
                "your_send_email_tool": {
                    "allowed_decisions": ["approve", "edit", "reject"],
                },
                "your_read_email_tool": False,
            }
        ),
    ],
)
```

!!! tip "Mẹo"
    Để xem các ví dụ đầy đủ, tùy chọn cấu hình, và mẫu tích hợp, xem [tài liệu Human-in-the-loop](../human-in-the-loop.md).

Xem [video hướng dẫn](https://www.youtube.com/watch?v=SpfT6-YAVPk) minh họa hành vi của middleware Human-in-the-loop.

### Model call limit

Giới hạn số lượng lệnh gọi model để tránh vòng lặp vô hạn hoặc chi phí phát sinh quá mức. Model call limit hữu ích cho các trường hợp sau:

* Ngăn các agent chạy mất kiểm soát thực hiện quá nhiều lệnh gọi API.
* Áp dụng kiểm soát chi phí trên các triển khai production.
* Kiểm thử hành vi agent trong một ngân sách lệnh gọi cụ thể.

**Tham chiếu API:** [`ModelCallLimitMiddleware`](https://reference.langchain.com/python/langchain/agents/middleware/model_call_limit/ModelCallLimitMiddleware)

```python
from langchain.agents import create_agent
from langchain.agents.middleware import ModelCallLimitMiddleware
from langgraph.checkpoint.memory import InMemorySaver

agent = create_agent(
    model="gpt-5.5",
    checkpointer=InMemorySaver(),  # Required for thread limiting
    tools=[],
    middleware=[
        ModelCallLimitMiddleware(
            thread_limit=10,
            run_limit=5,
            exit_behavior="end",
        ),
    ],
)
```

Xem [video hướng dẫn](https://www.youtube.com/watch?v=nJEER0uaNkE) minh họa hành vi của middleware Model Call Limit.

??? note "Tùy chọn cấu hình"
    **`thread_limit`** (`number`)
        Số lượng lệnh gọi model tối đa qua tất cả các lần chạy trong một thread. Mặc định không giới hạn.

    **`run_limit`** (`number`)
        Số lượng lệnh gọi model tối đa cho một lần gọi (invocation) duy nhất. Mặc định không giới hạn.

    **`exit_behavior`** (`string`, mặc định `end`)
        Hành vi khi đạt giới hạn. Các lựa chọn: `'end'` (kết thúc êm) hoặc `'error'` (raise exception)

### Tool call limit

Kiểm soát việc thực thi agent bằng cách giới hạn số lượng lệnh gọi tool, hoặc trên toàn bộ tool hoặc cho từng tool cụ thể. Giới hạn lệnh gọi tool hữu ích cho các trường hợp sau:

* Ngăn các lệnh gọi quá mức tới các API bên ngoài tốn kém.
* Giới hạn số lượt tìm kiếm web hoặc truy vấn database.
* Áp dụng rate limit trên việc sử dụng một tool cụ thể.
* Bảo vệ khỏi các vòng lặp agent chạy mất kiểm soát.

**Tham chiếu API:** [`ToolCallLimitMiddleware`](https://reference.langchain.com/python/langchain/agents/middleware/tool_call_limit/ToolCallLimitMiddleware)

```python
from langchain.agents import create_agent
from langchain.agents.middleware import ToolCallLimitMiddleware

agent = create_agent(
    model="gpt-5.5",
    tools=[search_tool, database_tool],
    middleware=[
        # Global limit
        ToolCallLimitMiddleware(thread_limit=20, run_limit=10),
        # Tool-specific limit
        ToolCallLimitMiddleware(
            tool_name="search",
            thread_limit=5,
            run_limit=3,
        ),
    ],
)
```

Xem [video hướng dẫn](https://www.youtube.com/watch?v=6gYlaJJ8t0w) minh họa hành vi của middleware Tool Call Limit.

??? note "Tùy chọn cấu hình"
    **`tool_name`** (`string`)
        Tên của tool cụ thể cần giới hạn. Nếu không được cung cấp, giới hạn sẽ áp dụng cho **tất cả tool trên toàn cục**.

    **`thread_limit`** (`number`)
        Số lượng lệnh gọi tool tối đa qua tất cả các lần chạy trong một thread (hội thoại). Được duy trì qua nhiều lần gọi (invocation) với cùng một thread ID. Yêu cầu một checkpointer để duy trì state. `None` nghĩa là không giới hạn theo thread.

    **`run_limit`** (`number`)
        Số lượng lệnh gọi tool tối đa cho một lần gọi (invocation) duy nhất (một chu kỳ tin nhắn người dùng → phản hồi). Được reset với mỗi tin nhắn người dùng mới. `None` nghĩa là không giới hạn theo lần chạy.

        **Lưu ý:** Phải chỉ định ít nhất một trong `thread_limit` hoặc `run_limit`.

    **`exit_behavior`** (`string`, mặc định `continue`)
        Hành vi khi đạt giới hạn:

        * `'continue'` (mặc định): chặn các lệnh gọi tool vượt giới hạn bằng thông báo lỗi, cho phép các tool khác và model tiếp tục. Model tự quyết định khi nào dừng dựa trên các thông báo lỗi.
        * `'error'`: raise một exception `ToolCallLimitExceededError`, dừng thực thi ngay lập tức
        * `'end'`: dừng thực thi ngay lập tức với một `ToolMessage` và AI message cho lệnh gọi tool vượt giới hạn. Chỉ hoạt động khi giới hạn một tool duy nhất; raise `NotImplementedError` nếu các tool khác còn lệnh gọi đang chờ.

??? note "Ví dụ đầy đủ"
    Chỉ định giới hạn với:

    * **Thread limit**: số lượng lệnh gọi tối đa qua tất cả các lần chạy trong một hội thoại (yêu cầu checkpointer)
    * **Run limit**: số lượng lệnh gọi tối đa cho một lần gọi (reset mỗi lượt)

    Các hành vi khi thoát (exit behavior):

    * `'continue'` (mặc định): chặn các lệnh gọi vượt giới hạn bằng thông báo lỗi, agent tiếp tục
    * `'error'`: raise exception ngay lập tức
    * `'end'`: dừng lại với ToolMessage + AI message (chỉ dành cho trường hợp giới hạn một tool duy nhất)

    ```python
    from langchain.agents import create_agent
    from langchain.agents.middleware import ToolCallLimitMiddleware


    global_limiter = ToolCallLimitMiddleware(thread_limit=20, run_limit=10)
    search_limiter = ToolCallLimitMiddleware(tool_name="search", thread_limit=5, run_limit=3)
    database_limiter = ToolCallLimitMiddleware(tool_name="query_database", thread_limit=10)
    strict_limiter = ToolCallLimitMiddleware(tool_name="scrape_webpage", run_limit=2, exit_behavior="error")

    agent = create_agent(
        model="gpt-5.5",
        tools=[search_tool, database_tool, scraper_tool],
        middleware=[global_limiter, search_limiter, database_limiter, strict_limiter],
    )
    ```

### PII detection

Phát hiện và xử lý Thông tin nhận dạng cá nhân (PII) trong hội thoại bằng các chiến lược có thể cấu hình. PII detection hữu ích cho các trường hợp sau:

* Các ứng dụng y tế và tài chính có yêu cầu tuân thủ (compliance).
* Các agent chăm sóc khách hàng cần làm sạch log.
* Bất kỳ ứng dụng nào xử lý dữ liệu người dùng nhạy cảm.

!!! note "Ghi chú"
    Với `apply_to_output=True`, `PIIMiddleware` cũng biên tập (redact) cả đầu ra dạng wire được stream, bao gồm các đoạn delta văn bản, tham số tool call, đầu ra tool, và state snapshot, thông qua một stream transformer đã đăng ký. Yêu cầu `langchain>=1.3.2`. Xem [Đăng ký transformer trên middleware](../event-streaming.md#register-transformers-on-middleware).

**Tham chiếu API:** [`PIIMiddleware`](https://reference.langchain.com/python/langchain/agents/middleware/pii/PIIMiddleware)

```python
from langchain.agents import create_agent
from langchain.agents.middleware import PIIMiddleware

agent = create_agent(
    model="gpt-5.5",
    tools=[],
    middleware=[
        PIIMiddleware("email", strategy="redact", apply_to_input=True),
        PIIMiddleware("credit_card", strategy="mask", apply_to_input=True),
    ],
)
```

#### Loại PII tùy chỉnh

Bạn có thể tạo các loại PII tùy chỉnh bằng cách cung cấp tham số `detector`. Điều này cho phép bạn phát hiện các mẫu (pattern) đặc thù cho trường hợp sử dụng của mình, vượt ra ngoài các loại dựng sẵn.

**Ba cách để tạo detector tùy chỉnh:**

1. **Chuỗi mẫu regex**: khớp mẫu đơn giản

2. **Hàm tùy chỉnh**: logic phát hiện phức tạp với xác thực (validation)

```python
from langchain.agents import create_agent
from langchain.agents.middleware import PIIMiddleware
import re


# Method 1: Regex pattern string
agent1 = create_agent(
    model="gpt-5.5",
    tools=[],
    middleware=[
        PIIMiddleware(
            "api_key",
            detector=r"sk-[a-zA-Z0-9]{32}",
            strategy="block",
        ),
    ],
)

# Method 2: Compiled regex pattern
agent2 = create_agent(
    model="gpt-5.5",
    tools=[],
    middleware=[
        PIIMiddleware(
            "phone_number",
            detector=re.compile(r"\+?\d{1,3}[\s.-]?\d{3,4}[\s.-]?\d{4}"),
            strategy="mask",
        ),
    ],
)

# Method 3: Custom detector function
def detect_ssn(content: str) -> list[dict[str, str | int]]:
    """Detect SSN with validation.

    Returns a list of dictionaries with 'text', 'start', and 'end' keys.
    """
    import re
    matches = []
    pattern = r"\d{3}-\d{2}-\d{4}"
    for match in re.finditer(pattern, content):
        ssn = match.group(0)
        # Validate: first 3 digits shouldn't be 000, 666, or 900-999
        first_three = int(ssn[:3])
        if first_three not in [0, 666] and not (900 <= first_three <= 999):
            matches.append({
                "text": ssn,
                "start": match.start(),
                "end": match.end(),
            })
    return matches

agent3 = create_agent(
    model="gpt-5.5",
    tools=[],
    middleware=[
        PIIMiddleware(
            "ssn",
            detector=detect_ssn,
            strategy="hash",
        ),
    ],
)
```

**Chữ ký của hàm detector tùy chỉnh:**

Hàm detector phải nhận vào một chuỗi (nội dung) và trả về các kết quả khớp:

Trả về một danh sách các dict với các key `text`, `start`, và `end`:

```python
def detector(content: str) -> list[dict[str, str | int]]:
    return [
        {"text": "matched_text", "start": 0, "end": 12},
        # ... more matches
    ]
```

!!! tip "Mẹo"
    Đối với các detector tùy chỉnh:

    * Dùng chuỗi regex cho các mẫu đơn giản
    * Dùng đối tượng RegExp khi cần các flag (ví dụ: khớp không phân biệt hoa thường)
    * Dùng hàm tùy chỉnh khi cần logic xác thực vượt ngoài việc khớp mẫu
    * Hàm tùy chỉnh cho bạn toàn quyền kiểm soát logic phát hiện và có thể triển khai các quy tắc xác thực phức tạp

??? note "Tùy chọn cấu hình"
    **`pii_type`** (`string`, bắt buộc)
        Loại PII cần phát hiện. Có thể là một loại dựng sẵn (`email`, `credit_card`, `ip`, `mac_address`, `url`) hoặc một tên loại tùy chỉnh.

    **`strategy`** (`string`, mặc định `redact`)
        Cách xử lý PII được phát hiện. Các lựa chọn:

        * `'block'`: raise exception khi phát hiện
        * `'redact'`: thay thế bằng `[REDACTED_{PII_TYPE}]`
        * `'mask'`: che một phần (ví dụ: `****-****-****-1234`)
        * `'hash'`: thay thế bằng hash xác định (deterministic)

    **`detector`** (`function | regex`)
        Hàm detector tùy chỉnh hoặc mẫu regex. Nếu không được cung cấp, dùng detector dựng sẵn cho loại PII đó.

    **`apply_to_input`** (`boolean`, mặc định `True`)
        Kiểm tra các tin nhắn người dùng trước khi gọi model

    **`apply_to_output`** (`boolean`, mặc định `False`)
        Kiểm tra các tin nhắn AI sau khi gọi model. Với `langchain>=1.3.2`, cũng biên tập cả đầu ra dạng wire được stream (đoạn delta văn bản, tham số tool call, đầu ra tool, state snapshot) thông qua một stream transformer đã đăng ký. Xem [event streaming](../event-streaming.md#register-transformers-on-middleware).

    **`apply_to_tool_results`** (`boolean`, mặc định `False`)
        Kiểm tra các tin nhắn kết quả tool sau khi thực thi

### To-do list

Trang bị cho agent khả năng lập kế hoạch và theo dõi tác vụ cho các công việc nhiều bước phức tạp. To-do list hữu ích cho các trường hợp sau:

* Các tác vụ nhiều bước phức tạp cần phối hợp qua nhiều tool.
* Các thao tác chạy dài nơi khả năng theo dõi tiến độ là quan trọng.

!!! note "Ghi chú"
    Middleware này tự động cung cấp cho agent một tool `write_todos` cùng các system prompt để hướng dẫn việc lập kế hoạch tác vụ hiệu quả.

**Tham chiếu API:** [`TodoListMiddleware`](https://reference.langchain.com/python/langchain/agents/middleware/todo/TodoListMiddleware)

```python
from langchain.agents import create_agent
from langchain.agents.middleware import TodoListMiddleware

agent = create_agent(
    model="gpt-5.5",
    tools=[read_file, write_file, run_tests],
    middleware=[TodoListMiddleware()],
)
```

Xem [video hướng dẫn](https://www.youtube.com/watch?v=yTWocbVKQxw) minh họa hành vi của middleware To-do List.

??? note "Tùy chọn cấu hình"
    **`system_prompt`** (`string`)
        System prompt tùy chỉnh để hướng dẫn việc sử dụng todo. Dùng prompt dựng sẵn nếu không được chỉ định.

    **`tool_description`** (`string`)
        Mô tả tùy chỉnh cho tool `write_todos`. Dùng mô tả dựng sẵn nếu không được chỉ định.

### LLM tool selector

Dùng một LLM để chọn thông minh các tool liên quan trước khi gọi model chính. LLM tool selector hữu ích cho các trường hợp sau:

* Các agent có nhiều tool (10+) mà phần lớn không liên quan cho mỗi truy vấn.
* Giảm mức sử dụng token bằng cách lọc bỏ các tool không liên quan.
* Cải thiện sự tập trung và độ chính xác của model.

Middleware này dùng structured output để hỏi một LLM xem tool nào liên quan nhất cho truy vấn hiện tại. Schema structured output định nghĩa các tên và mô tả tool khả dụng. Các nhà cung cấp model thường tự động thêm thông tin structured output này vào system prompt.

**Tham chiếu API:** [`LLMToolSelectorMiddleware`](https://reference.langchain.com/python/langchain/agents/middleware/tool_selection/LLMToolSelectorMiddleware)

```python
from langchain.agents import create_agent
from langchain.agents.middleware import LLMToolSelectorMiddleware

agent = create_agent(
    model="gpt-5.5",
    tools=[tool1, tool2, tool3, tool4, tool5, ...],
    middleware=[
        LLMToolSelectorMiddleware(
            model="gpt-5.4-mini",
            max_tools=3,
            always_include=["search"],
        ),
    ],
)
```

??? note "Tùy chọn cấu hình"
    **`model`** (`string | BaseChatModel`)
        Model dùng để chọn tool. Có thể là một chuỗi định danh model (ví dụ: `'openai:gpt-5.4-mini'`) hoặc một instance `BaseChatModel`. Xem [`init_chat_model`](https://reference.langchain.com/python/langchain/chat_models/base/init_chat_model) để biết thêm thông tin.

        Mặc định dùng model chính của agent.

    **`system_prompt`** (`string`)
        Hướng dẫn cho model chọn lựa. Dùng prompt dựng sẵn nếu không được chỉ định.

    **`max_tools`** (`number`)
        Số lượng tool tối đa được chọn. Nếu model chọn nhiều hơn, chỉ max_tools mục đầu tiên được dùng. Không giới hạn nếu không được chỉ định.

    **`always_include`** (`list[string]`)
        Tên tool luôn được đưa vào bất kể lựa chọn. Các tool này không tính vào giới hạn max_tools.

### Provider tool search

Trì hoãn việc nạp một số tool được chọn phía sau cơ chế tìm kiếm tool phía server của nhà cung cấp model, để model tự khám phá chúng khi cần thay vì nhận toàn bộ schema tool ngay từ đầu. Provider tool search hữu ích cho:

* Giảm tình trạng context bị phình to khi dùng nhiều tool.
* Cải thiện độ chính xác trong việc chọn tool bằng cách chỉ hiển thị các tool liên quan.

!!! note "Ghi chú"
    Yêu cầu một model có hỗ trợ tìm kiếm tool phía server: Anthropic (Claude Sonnet 4+/Opus 4+/Haiku 4.5+) hoặc OpenAI (gpt-5.5+). Các nhà cung cấp khác sẽ raise `ValueError`.

**Tham chiếu API:** [`ProviderToolSearchMiddleware`](https://reference.langchain.com/python/langchain/agents/middleware/provider_tool_search/ProviderToolSearchMiddleware)

```python
from langchain.agents import create_agent
from langchain.agents.middleware import ProviderToolSearchMiddleware

agent = create_agent(
    model="anthropic:claude-opus-4-8",
    tools=[get_weather, lookup_order],
    middleware=[
        ProviderToolSearchMiddleware(searchable_tools=["lookup_order"]),
    ],
)
```

??? note "Tùy chọn cấu hình"
    **`searchable_tools`** (`list[str | BaseTool]`)
        Các tool cần trì hoãn phía sau cơ chế tìm kiếm tool của nhà cung cấp, chỉ định theo tên hoặc instance. Các tool bị trì hoãn sẽ không được đưa cho model cho đến khi cơ chế tìm kiếm của model làm chúng xuất hiện. Các tool được khởi tạo với `extras={"defer_loading": True}` sẽ luôn bị trì hoãn bất kể tùy chọn này; nếu `searchable_tools` không được cung cấp, chỉ những tool đã được đánh dấu trước đó mới bị trì hoãn.

??? note "Ví dụ đầy đủ"
    Middleware này cho phép tất cả các tool có trong `searchable_tools` được trì hoãn và tìm kiếm. Một tool cũng có thể tự đăng ký trì hoãn ngay tại thời điểm khởi tạo bằng cách đặt `extras={"defer_loading": True}`.

    ```python
    from langchain.agents import create_agent
    from langchain.agents.middleware import ProviderToolSearchMiddleware
    from langchain.tools import tool


    # Marked `defer_loading` at construction, so it's deferred on its own —
    # no need to list it in `searchable_tools`.
    @tool(extras={"defer_loading": True})
    def send_email(to: str) -> str:
        """Send an email."""
        return "sent"


    agent = create_agent(
        model="anthropic:claude-opus-4-8",
        tools=[send_email],
        middleware=[ProviderToolSearchMiddleware()],
    )
    ```

### Shell tool

Cung cấp cho agent một phiên shell liên tục để thực thi lệnh. Shell tool middleware hữu ích cho các trường hợp sau:

* Các agent cần thực thi lệnh hệ thống
* Các tác vụ tự động hóa phát triển và triển khai
* Các quy trình kiểm thử và xác thực
* Các thao tác trên hệ thống tệp và thực thi script

!!! warning "Cảnh báo"
    **Lưu ý bảo mật**: hãy dùng chính sách thực thi phù hợp (`HostExecutionPolicy`, `DockerExecutionPolicy`, hoặc `CodexSandboxExecutionPolicy`) để khớp với yêu cầu bảo mật của môi trường triển khai.

!!! note "Ghi chú"
    **Hạn chế**: các phiên shell liên tục hiện chưa hoạt động với các lượt ngắt (human-in-the-loop). Chúng tôi dự kiến sẽ bổ sung hỗ trợ này trong tương lai.

**Tham chiếu API:** [`ShellToolMiddleware`](https://reference.langchain.com/python/langchain/agents/middleware/shell_tool/ShellToolMiddleware)

```python
from langchain.agents import create_agent
from langchain.agents.middleware import (
    ShellToolMiddleware,
    HostExecutionPolicy,
)

agent = create_agent(
    model="gpt-5.5",
    tools=[search_tool],
    middleware=[
        ShellToolMiddleware(
            workspace_root="/workspace",
            execution_policy=HostExecutionPolicy(),
        ),
    ],
)
```

??? note "Tùy chọn cấu hình"
    **`workspace_root`** (`str | Path | None`)
        Thư mục gốc cho phiên shell. Nếu bỏ trống, một thư mục tạm sẽ được tạo khi agent khởi động và bị xóa khi agent kết thúc.

    **`startup_commands`** (`tuple[str, ...] | list[str] | str | None`)
        Các lệnh tùy chọn được thực thi tuần tự sau khi phiên khởi động

    **`shutdown_commands`** (`tuple[str, ...] | list[str] | str | None`)
        Các lệnh tùy chọn được thực thi trước khi phiên tắt

    **`execution_policy`** (`BaseExecutionPolicy | None`)
        Chính sách thực thi kiểm soát thời gian chờ (timeout), giới hạn đầu ra, và cấu hình tài nguyên. Các lựa chọn:

        * `HostExecutionPolicy`: truy cập đầy đủ vào host (mặc định); phù hợp nhất cho các môi trường đáng tin cậy nơi agent đã chạy trong một container hoặc VM
        * `DockerExecutionPolicy`: khởi chạy một container Docker riêng cho mỗi lần chạy agent, mang lại mức cô lập mạnh hơn
        * `CodexSandboxExecutionPolicy`: tái sử dụng sandbox của Codex CLI để có thêm hạn chế về syscall/filesystem

    **`redaction_rules`** (`tuple[RedactionRule, ...] | list[RedactionRule] | None`)
        Các quy tắc biên tập (redaction) tùy chọn để làm sạch đầu ra lệnh trước khi trả về cho model.

        !!! warning "Cảnh báo"
            Các quy tắc biên tập được áp dụng sau khi thực thi và không ngăn được việc rò rỉ (exfiltration) secret hoặc dữ liệu nhạy cảm khi dùng `HostExecutionPolicy`.

    **`tool_description`** (`str | None`)
        Ghi đè tùy chọn cho mô tả của tool shell đã đăng ký

    **`shell_command`** (`Sequence[str] | str | None`)
        Chương trình shell (chuỗi) hoặc dãy tham số tùy chọn dùng để khởi chạy phiên liên tục. Mặc định là `/bin/bash`.

    **`env`** (`Mapping[str, Any] | None`)
        Các biến môi trường tùy chọn cung cấp cho phiên shell. Các giá trị được ép kiểu về chuỗi trước khi thực thi lệnh.

??? note "Ví dụ đầy đủ"
    Middleware này cung cấp một phiên shell liên tục duy nhất mà agent có thể dùng để thực thi lệnh tuần tự.

    **Các chính sách thực thi:**

    * `HostExecutionPolicy` (mặc định): thực thi trực tiếp với quyền truy cập đầy đủ vào host
    * `DockerExecutionPolicy`: thực thi cô lập trong container Docker
    * `CodexSandboxExecutionPolicy`: thực thi trong sandbox qua Codex CLI

    ```python
    from langchain.agents import create_agent
    from langchain.agents.middleware import (
        ShellToolMiddleware,
        HostExecutionPolicy,
        DockerExecutionPolicy,
        RedactionRule,
    )


    # Basic shell tool with host execution
    agent = create_agent(
        model="gpt-5.5",
        tools=[search_tool],
        middleware=[
            ShellToolMiddleware(
                workspace_root="/workspace",
                execution_policy=HostExecutionPolicy(),
            ),
        ],
    )

    # Docker isolation with startup commands
    agent_docker = create_agent(
        model="gpt-5.5",
        tools=[],
        middleware=[
            ShellToolMiddleware(
                workspace_root="/workspace",
                startup_commands=["pip install requests", "export PYTHONPATH=/workspace"],
                execution_policy=DockerExecutionPolicy(
                    image="python:3.11-slim",
                    command_timeout=60.0,
                ),
            ),
        ],
    )

    # With output redaction (applied post execution)
    agent_redacted = create_agent(
        model="gpt-5.5",
        tools=[],
        middleware=[
            ShellToolMiddleware(
                workspace_root="/workspace",
                redaction_rules=[
                    RedactionRule(pii_type="api_key", detector=r"sk-[a-zA-Z0-9]{32}"),
                ],
            ),
        ],
    )
    ```

### Filesystem middleware

Context engineering là một thách thức chính trong việc xây dựng agent hiệu quả. Điều này đặc biệt khó khăn khi dùng các tool trả về kết quả có độ dài thay đổi (ví dụ: `web_search` và RAG), vì các kết quả tool dài có thể nhanh chóng làm đầy context window của bạn.

`FilesystemMiddleware` từ [Deep Agents](https://docs.langchain.com/oss/python/deepagents/overview) cung cấp bốn tool để tương tác với cả bộ nhớ ngắn hạn và dài hạn:

* `ls`: liệt kê các tệp trong hệ thống tệp
* `read_file`: đọc toàn bộ một tệp hoặc một số dòng nhất định từ tệp
* `write_file`: ghi một tệp mới vào hệ thống tệp
* `edit_file`: chỉnh sửa một tệp có sẵn trong hệ thống tệp

```python
from langchain.agents import create_agent
from deepagents.middleware.filesystem import FilesystemMiddleware

# FilesystemMiddleware is included by default in create_deep_agent
# You can customize it if building a custom agent
agent = create_agent(
    model="claude-sonnet-4-6",
    middleware=[
        FilesystemMiddleware(
            backend=None,  # Optional: custom backend (defaults to StateBackend)
            system_prompt="Write to the filesystem when...",  # Optional custom addition to the system prompt
            custom_tool_descriptions={
                "ls": "Use the ls tool when...",
                "read_file": "Use the read_file tool to..."
            },  # Optional: Custom descriptions for filesystem tools
            tools=["read_file", "ls", "glob", "grep"],  # Optional: Allowlist restricting which filesystem tools are exposed
        ),
    ],
)
```

#### Hệ thống tệp ngắn hạn và dài hạn

Theo mặc định, các tool này ghi vào một "hệ thống tệp" cục bộ trong state của graph. Để bật lưu trữ bền vững (persistent) qua các thread, hãy cấu hình một `CompositeBackend` để định tuyến các đường dẫn cụ thể (như `/memories/`) tới một `StoreBackend`.

```python
from langchain.agents import create_agent
from deepagents.middleware import FilesystemMiddleware
from deepagents.backends import CompositeBackend, StateBackend, StoreBackend
from langgraph.store.memory import InMemoryStore

store = InMemoryStore()

agent = create_agent(
    model="claude-sonnet-4-6",
    store=store,
    middleware=[
        FilesystemMiddleware(
            backend=CompositeBackend(
                default=StateBackend(),
                routes={"/memories/": StoreBackend()}
            ),
            custom_tool_descriptions={
                "ls": "Use the ls tool when...",
                "read_file": "Use the read_file tool to..."
            }  # Optional: Custom descriptions for filesystem tools
        ),
    ],
)
```

Khi bạn cấu hình một `CompositeBackend` với một `StoreBackend` cho `/memories/`, mọi tệp có tiền tố **/memories/** sẽ được lưu vào bộ nhớ bền vững và tồn tại qua các thread khác nhau. Các tệp không có tiền tố này vẫn ở trong bộ nhớ trạng thái tạm thời (ephemeral).

### Subagent

Việc giao tác vụ cho subagent giúp cô lập context, giữ cho context window của agent chính (supervisor) sạch sẽ trong khi vẫn có thể đi sâu vào một tác vụ.

Middleware subagent từ [Deep Agents](https://docs.langchain.com/oss/python/deepagents/overview) cho phép bạn cung cấp các subagent thông qua một tool `task`.

```python
from langchain.tools import tool
from langchain.agents import create_agent
from deepagents.middleware.subagents import SubAgentMiddleware


@tool
def get_weather(city: str) -> str:
    """Get the weather in a city."""
    return f"The weather in {city} is sunny."

agent = create_agent(
    model="claude-sonnet-4-6",
    middleware=[
        SubAgentMiddleware(
            default_model="claude-sonnet-4-6",
            default_tools=[],
            subagents=[
                {
                    "name": "weather",
                    "description": "This subagent can get weather in cities.",
                    "system_prompt": "Use the get_weather tool to get the weather in a city.",
                    "tools": [get_weather],
                    "model": "gpt-5.5",
                    "middleware": [],
                }
            ],
        )
    ],
)
```

Một subagent được định nghĩa với **tên**, **mô tả**, **system prompt**, và **tool**. Bạn cũng có thể cung cấp cho subagent một **model** tùy chỉnh, hoặc thêm **middleware** bổ sung. Điều này đặc biệt hữu ích khi bạn muốn cung cấp cho subagent một state key bổ sung để chia sẻ với agent chính.

Đối với các trường hợp sử dụng phức tạp hơn, bạn cũng có thể cung cấp một graph LangGraph dựng sẵn của riêng mình làm subagent.

```python
from langchain.agents import create_agent
from deepagents.middleware.subagents import SubAgentMiddleware
from deepagents import CompiledSubAgent
from langgraph.graph import StateGraph

# Create a custom LangGraph graph
def create_weather_graph():
    workflow = StateGraph(...)
    # Build your custom graph
    return workflow.compile()

weather_graph = create_weather_graph()

# Wrap it in a CompiledSubAgent
weather_subagent = CompiledSubAgent(
    name="weather",
    description="This subagent can get weather in cities.",
    runnable=weather_graph
)

agent = create_agent(
    model="claude-sonnet-4-6",
    middleware=[
        SubAgentMiddleware(
            default_model="claude-sonnet-4-6",
            default_tools=[],
            subagents=[weather_subagent],
        )
    ],
)
```

Bên cạnh mọi subagent do người dùng định nghĩa, agent chính luôn có quyền truy cập vào một subagent `general-purpose`. Subagent này có cùng chỉ dẫn với agent chính và tất cả các tool mà agent chính có quyền truy cập. Mục đích chính của subagent `general-purpose` là cô lập context: agent chính có thể giao một tác vụ phức tạp cho subagent này và nhận lại một câu trả lời ngắn gọn mà không bị phình to bởi các lệnh gọi tool trung gian.

### Rubric grading

!!! note "Ghi chú"
    `RubricMiddleware` yêu cầu `deepagents>=0.6.5`. Middleware này đang ở giai đoạn [**beta**](https://docs.langchain.com/oss/python/versioning); API có thể thay đổi trong tương lai.

Một số tác vụ có định nghĩa rõ ràng về "hoàn thành" mà agent không thể đảm bảo đạt được ngay trong lần thử đầu tiên. `RubricMiddleware` cho phép bạn khai báo *thế nào là hoàn thành* dưới dạng một rubric, và để agent tự đánh giá cũng như lặp lại cho đến khi rubric được thỏa mãn hoặc đạt số lần lặp tối đa.

**Tham chiếu API:** [`RubricMiddleware`](https://reference.langchain.com/python/deepagents/middleware/rubric/RubricMiddleware)

=== "Google"

    ```python
    from deepagents import RubricMiddleware, create_deep_agent
    from langgraph.checkpoint.memory import InMemorySaver

    agent = create_deep_agent(
        model="google_genai:gemini-3.6-flash",
        middleware=[
            RubricMiddleware(
                model="anthropic:claude-haiku-4-5",
                max_iterations=3,
            ),
        ],
        checkpointer=InMemorySaver(),
    )
    ```

=== "OpenAI"

    ```python
    from deepagents import RubricMiddleware, create_deep_agent
    from langgraph.checkpoint.memory import InMemorySaver

    agent = create_deep_agent(
        model="openai:gpt-5.5",
        middleware=[
            RubricMiddleware(
                model="anthropic:claude-haiku-4-5",
                max_iterations=3,
            ),
        ],
        checkpointer=InMemorySaver(),
    )
    ```

=== "Anthropic"

    ```python
    from deepagents import RubricMiddleware, create_deep_agent
    from langgraph.checkpoint.memory import InMemorySaver

    agent = create_deep_agent(
        model="anthropic:claude-sonnet-4-6",
        middleware=[
            RubricMiddleware(
                model="anthropic:claude-haiku-4-5",
                max_iterations=3,
            ),
        ],
        checkpointer=InMemorySaver(),
    )
    ```

=== "OpenRouter"

    ```python
    from deepagents import RubricMiddleware, create_deep_agent
    from langgraph.checkpoint.memory import InMemorySaver

    agent = create_deep_agent(
        model="openrouter:z-ai/glm-5.2",
        middleware=[
            RubricMiddleware(
                model="anthropic:claude-haiku-4-5",
                max_iterations=3,
            ),
        ],
        checkpointer=InMemorySaver(),
    )
    ```

=== "Fireworks"

    ```python
    from deepagents import RubricMiddleware, create_deep_agent
    from langgraph.checkpoint.memory import InMemorySaver

    agent = create_deep_agent(
        model="fireworks:accounts/fireworks/models/glm-5p2",
        middleware=[
            RubricMiddleware(
                model="anthropic:claude-haiku-4-5",
                max_iterations=3,
            ),
        ],
        checkpointer=InMemorySaver(),
    )
    ```

=== "Baseten"

    ```python
    from deepagents import RubricMiddleware, create_deep_agent
    from langgraph.checkpoint.memory import InMemorySaver

    agent = create_deep_agent(
        model="baseten:zai-org/GLM-5.2",
        middleware=[
            RubricMiddleware(
                model="anthropic:claude-haiku-4-5",
                max_iterations=3,
            ),
        ],
        checkpointer=InMemorySaver(),
    )
    ```

=== "Ollama"

    ```python
    from deepagents import RubricMiddleware, create_deep_agent
    from langgraph.checkpoint.memory import InMemorySaver

    agent = create_deep_agent(
        model="ollama:north-mini-code-1.0",
        middleware=[
            RubricMiddleware(
                model="anthropic:claude-haiku-4-5",
                max_iterations=3,
            ),
        ],
        checkpointer=InMemorySaver(),
    )
    ```

Để xem đầy đủ tùy chọn cấu hình, các sự kiện streaming, và một ví dụ đầy đủ về sinh code, xem [Grading rubrics](https://docs.langchain.com/oss/python/deepagents/rubric).

### File search

Cung cấp các tool tìm kiếm Glob và Grep trên một hệ thống tệp. File search middleware hữu ích cho các trường hợp sau:

* Khám phá và phân tích code
* Tìm tệp theo mẫu tên
* Tìm kiếm nội dung code bằng regex
* Các codebase lớn nơi cần khám phá tệp

**Tham chiếu API:** [`FilesystemFileSearchMiddleware`](https://reference.langchain.com/python/langchain/agents/middleware/file_search/FilesystemFileSearchMiddleware)

```python
from langchain.agents import create_agent
from langchain.agents.middleware import FilesystemFileSearchMiddleware

agent = create_agent(
    model="gpt-5.5",
    tools=[],
    middleware=[
        FilesystemFileSearchMiddleware(
            root_path="/workspace",
            use_ripgrep=True,
        ),
    ],
)
```

??? note "Tùy chọn cấu hình"
    **`root_path`** (`str`, bắt buộc)
        Thư mục gốc để tìm kiếm. Mọi thao tác tệp đều tương đối với đường dẫn này.

    **`use_ripgrep`** (`bool`, mặc định `True`)
        Có dùng ripgrep để tìm kiếm hay không. Sẽ chuyển sang dùng regex Python nếu ripgrep không khả dụng.

    **`max_file_size_mb`** (`int`, mặc định `10`)
        Kích thước tệp tối đa để tìm kiếm, tính bằng MB. Các tệp lớn hơn mức này sẽ bị bỏ qua.

??? note "Ví dụ đầy đủ"
    Middleware này bổ sung hai tool tìm kiếm cho agent:

    **Tool Glob**: khớp mẫu tệp nhanh:

    * Hỗ trợ các mẫu như `**/*.py`, `src/**/*.ts`
    * Trả về các đường dẫn tệp khớp, sắp xếp theo thời gian chỉnh sửa

    **Tool Grep**: tìm kiếm nội dung bằng regex:

    * Hỗ trợ đầy đủ cú pháp regex
    * Lọc theo mẫu tệp với tham số `include`
    * Ba chế độ đầu ra: `files_with_matches`, `content`, `count`

    ```python
    from langchain.agents import create_agent
    from langchain.agents.middleware import FilesystemFileSearchMiddleware
    from langchain.messages import HumanMessage


    agent = create_agent(
        model="gpt-5.5",
        tools=[],
        middleware=[
            FilesystemFileSearchMiddleware(
                root_path="/workspace",
                use_ripgrep=True,
                max_file_size_mb=10,
            ),
        ],
    )

    # Agent can now use glob_search and grep_search tools
    result = agent.invoke({
        "messages": [HumanMessage("Find all Python files containing 'async def'")]
    })

    # The agent will use:
    # 1. glob_search(pattern="**/*.py") to find Python files
    # 2. grep_search(pattern="async def", include="*.py") to find async functions
    ```

### Context editing

Quản lý context hội thoại bằng cách xóa các đầu ra tool call cũ khi đạt giới hạn token, trong khi vẫn giữ lại các kết quả gần đây. Điều này giúp giữ context window ở mức kiểm soát được trong các hội thoại dài có nhiều lệnh gọi tool. Context editing hữu ích cho các trường hợp sau:

* Các hội thoại dài có nhiều lệnh gọi tool vượt giới hạn token
* Giảm chi phí token bằng cách loại bỏ các đầu ra tool cũ không còn liên quan
* Chỉ duy trì N kết quả tool gần đây nhất trong context

**Tham chiếu API:** [`ContextEditingMiddleware`](https://reference.langchain.com/python/langchain/agents/middleware/context_editing/ContextEditingMiddleware), [`ClearToolUsesEdit`](https://reference.langchain.com/python/langchain/agents/middleware/context_editing/ClearToolUsesEdit)

```python
from langchain.agents import create_agent
from langchain.agents.middleware import ContextEditingMiddleware, ClearToolUsesEdit

agent = create_agent(
    model="gpt-5.5",
    tools=[],
    middleware=[
        ContextEditingMiddleware(
            edits=[
                ClearToolUsesEdit(
                    trigger=100000,
                    keep=3,
                ),
            ],
        ),
    ],
)
```

??? note "Tùy chọn cấu hình"
    **`edits`** (`list[ContextEdit]`, mặc định `[ClearToolUsesEdit()]`)
        Danh sách các chiến lược [`ContextEdit`](https://reference.langchain.com/python/langchain/agents/middleware/context_editing/ContextEdit) cần áp dụng

    **`token_count_method`** (`string`, mặc định `approximate`)
        Phương pháp đếm token. Các lựa chọn: `'approximate'` hoặc `'model'`

    **Các tùy chọn của [`ClearToolUsesEdit`](https://reference.langchain.com/python/langchain/agents/middleware/context_editing/ClearToolUsesEdit):**

    **`trigger`** (`number`, mặc định `100000`)
        Số lượng token kích hoạt việc chỉnh sửa. Khi hội thoại vượt quá số token này, các đầu ra tool cũ hơn sẽ bị xóa.

    **`clear_at_least`** (`number`, mặc định `0`)
        Số token tối thiểu cần thu hồi khi thao tác chỉnh sửa chạy. Nếu đặt là 0, sẽ xóa đến mức cần thiết.

    **`keep`** (`number`, mặc định `3`)
        Số kết quả tool gần đây nhất phải được giữ lại. Các kết quả này sẽ không bao giờ bị xóa.

    **`clear_tool_inputs`** (`boolean`, mặc định `False`)
        Có xóa các tham số tool call gốc trên AI message hay không. Khi `True`, các tham số tool call sẽ được thay bằng đối tượng rỗng.

    **`exclude_tools`** (`list[string]`, mặc định `()`)
        Danh sách tên tool được loại trừ khỏi việc xóa. Đầu ra của các tool này sẽ không bao giờ bị xóa.

    **`placeholder`** (`string`, mặc định `[cleared]`)
        Văn bản placeholder được chèn vào cho các đầu ra tool đã bị xóa. Văn bản này thay thế nội dung tool message gốc.

??? note "Ví dụ đầy đủ"
    Middleware này áp dụng các chiến lược chỉnh sửa context khi đạt giới hạn token. Chiến lược phổ biến nhất là `ClearToolUsesEdit`, xóa các kết quả tool cũ trong khi vẫn giữ lại các kết quả gần đây.

    **Cách hoạt động:**

    1. Theo dõi số lượng token trong hội thoại
    2. Khi đạt ngưỡng, xóa các đầu ra tool cũ hơn
    3. Giữ lại N kết quả tool gần đây nhất
    4. Tùy chọn giữ lại tham số tool call để có context

    ```python
    from langchain.agents import create_agent
    from langchain.agents.middleware import ContextEditingMiddleware, ClearToolUsesEdit


    agent = create_agent(
        model="gpt-5.5",
        tools=[search_tool, your_calculator_tool, database_tool],
        middleware=[
            ContextEditingMiddleware(
                edits=[
                    ClearToolUsesEdit(
                        trigger=2000,
                        keep=3,
                        clear_tool_inputs=False,
                        exclude_tools=[],
                        placeholder="[cleared]",
                    ),
                ],
            ),
        ],
    )
    ```

### LLM tool emulator

Giả lập việc thực thi tool bằng một LLM cho mục đích kiểm thử, thay thế các lệnh gọi tool thật bằng các phản hồi do AI tạo ra. LLM tool emulator hữu ích cho các trường hợp sau:

* Kiểm thử hành vi agent mà không cần thực thi tool thật.
* Phát triển agent khi các tool bên ngoài không khả dụng hoặc tốn kém.
* Xây dựng prototype cho quy trình agent trước khi triển khai tool thật.

**Tham chiếu API:** [`LLMToolEmulator`](https://reference.langchain.com/python/langchain/agents/middleware/tool_emulator/LLMToolEmulator)

```python
from langchain.agents import create_agent
from langchain.agents.middleware import LLMToolEmulator

agent = create_agent(
    model="gpt-5.5",
    tools=[get_weather, search_database, send_email],
    middleware=[
        LLMToolEmulator(),  # Emulate all tools
    ],
)
```

??? note "Tùy chọn cấu hình"
    **`tools`** (`list[str | BaseTool]`)
        Danh sách tên tool (str) hoặc instance BaseTool cần giả lập. Nếu `None` (mặc định), TẤT CẢ tool sẽ được giả lập. Nếu là danh sách rỗng `[]`, không tool nào được giả lập. Nếu là mảng chứa tên/instance tool, chỉ những tool đó mới được giả lập.

    **`model`** (`string | BaseChatModel`)
        Model dùng để tạo các phản hồi tool giả lập. Có thể là một chuỗi định danh model (ví dụ: `'google_genai:gemini-3.6-flash'`) hoặc một instance `BaseChatModel`. Mặc định dùng model của agent nếu không được chỉ định. Xem [`init_chat_model`](https://reference.langchain.com/python/langchain/chat_models/base/init_chat_model) để biết thêm thông tin.

??? note "Ví dụ đầy đủ"
    Middleware này dùng một LLM để tạo ra các phản hồi hợp lý cho lệnh gọi tool thay vì thực thi tool thật.

    ```python
    from langchain.agents import create_agent
    from langchain.agents.middleware import LLMToolEmulator
    from langchain.tools import tool


    @tool
    def get_weather(location: str) -> str:
        """Get the current weather for a location."""
        return f"Weather in {location}"

    @tool
    def send_email(to: str, subject: str, body: str) -> str:
        """Send an email."""
        return "Email sent"


    # Emulate all tools (default behavior)
    agent = create_agent(
        model="gpt-5.5",
        tools=[get_weather, send_email],
        middleware=[LLMToolEmulator()],
    )

    # Emulate specific tools only
    agent2 = create_agent(
        model="gpt-5.5",
        tools=[get_weather, send_email],
        middleware=[LLMToolEmulator(tools=["get_weather"])],
    )

    # Use custom model for emulation
    agent4 = create_agent(
        model="gpt-5.5",
        tools=[get_weather, send_email],
        middleware=[LLMToolEmulator(model="claude-sonnet-4-6")],
    )
    ```

## Middleware theo nhà cung cấp cụ thể

Các middleware sau được tối ưu cho các nhà cung cấp LLM cụ thể. Xem tài liệu của từng nhà cung cấp để biết đầy đủ chi tiết và ví dụ.

**[Anthropic](https://docs.langchain.com/oss/python/integrations/middleware/anthropic)**

Middleware về prompt caching, bash tool, text editor, memory, và file search dành cho các model Claude.

**[AWS](https://docs.langchain.com/oss/python/integrations/middleware/aws)**

Middleware về prompt caching dành cho các model Amazon Bedrock.

**[OpenAI](https://docs.langchain.com/oss/python/integrations/middleware/openai)**

Middleware kiểm duyệt nội dung (content moderation) dành cho các model OpenAI.

***

[Kết nối các tài liệu này](https://docs.langchain.com/use-these-docs) với Claude, VSCode, và nhiều công cụ khác qua MCP để có câu trả lời theo thời gian thực.

[Chỉnh sửa trang này trên GitHub](https://github.com/langchain-ai/docs/edit/main/src/oss/langchain/middleware/built-in.mdx) hoặc [báo lỗi](https://github.com/langchain-ai/docs/issues/new/choose).
