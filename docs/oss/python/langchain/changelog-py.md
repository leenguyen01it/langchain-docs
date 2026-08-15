# Changelog

> Nhật ký cập nhật và cải tiến cho các package Python của chúng tôi

!!! info "Đăng ký nhận tin"
    Changelog này có kèm [RSS feed](https://docs.langchain.com/oss/python/releases/changelog/rss.xml), có thể tích hợp với [Slack](https://slack.com/help/articles/218688467-Add-RSS-feeds-to-Slack), [email](https://zapier.com/apps/email/integrations/rss/1441/send-new-rss-feed-entries-via-email), các Discord bot như [Readybot](https://readybot.io/) hoặc [RSS Feeds to Discord Bot](https://rss.app/en/bots/rssfeeds-discord-bot), và các công cụ đăng ký khác.

### Jul 24, 2026 · `deepagents`

## `deepagents` v0.7.0

Một harness gọn nhẹ hơn, cấu hình linh hoạt hơn theo mặc định. Trên một lượt agent mặc định, số token đầu vào giảm **65%** (5.395 → 1.895), đã được kiểm chứng bằng [bộ evaluation đã làm mới](https://www.langchain.com/blog/how-we-benchmark-deep-agents) mà không giảm chất lượng.

### Tối ưu hoá

* **Prompt gọn nhẹ theo mặc định**: Prompt gốc được soạn sẵn giờ bắt đầu rỗng, và phần văn xuôi hướng dẫn dùng tool trùng lặp với schema của tool đã được cắt bớt. Chỉ tính riêng schema tool của agent mặc định, tổng số token mô tả giảm **43%** (4.005 → 2.302); kết hợp với prompt gốc rỗng và todo tuỳ chọn (opt-in), số token đầu vào của một lượt agent mặc định giảm **65%** (5.395 → 1.895). Hành vi của tool không đổi. ([#4859](https://github.com/langchain-ai/deepagents/pull/4859), [#4979](https://github.com/langchain-ai/deepagents/pull/4979), [#5009](https://github.com/langchain-ai/deepagents/pull/5009))

### Tính năng mới

* **[Ghi đè một middleware mặc định](https://docs.langchain.com/oss/python/deepagents/customization#middleware)**: Một instance `middleware=` (hoặc `middleware` của subagent) có `.name` trùng với một middleware có sẵn giờ sẽ thay thế middleware mặc định đó tại chỗ, thay vì báo lỗi trùng lặp. Ví dụ, truyền `SummarizationMiddleware(...)` của riêng bạn để đổi ngưỡng token kích hoạt hoặc model tóm tắt mà không cần tắt middleware mặc định. ([#4251](https://github.com/langchain-ai/deepagents/pull/4251))
* **Các tool filesystem**: Tool [`delete`](https://docs.langchain.com/oss/python/deepagents/tools#built-in-harness-tools) mới xoá một file hoặc xoá đệ quy một thư mục ([#3659](https://github.com/langchain-ai/deepagents/pull/3659), [#3851](https://github.com/langchain-ai/deepagents/pull/3851)); `write_file` giờ ghi đè file đã tồn tại thay vì báo lỗi ([#4109](https://github.com/langchain-ai/deepagents/pull/4109)); `FilesystemMiddleware` chấp nhận [allowlist tool](https://docs.langchain.com/oss/python/deepagents/overview#virtual-filesystem-access) để chỉ expose các tool có sẵn được chọn ([#4325](https://github.com/langchain-ai/deepagents/pull/4325), [#4698](https://github.com/langchain-ai/deepagents/pull/4698)); và các thao tác đọc/tìm kiếm được tinh chỉnh cho các model mở, `read_file` phân trang báo tổng số dòng và số dòng còn lại kèm `offset` tiếp theo ([#4540](https://github.com/langchain-ai/deepagents/pull/4540)), `grep`/`glob` trả về kết quả một phần kèm cờ `truncated` thay vì bị treo với cây thư mục lớn ([#4063](https://github.com/langchain-ai/deepagents/pull/4063)), và `grep` có giới hạn 1.000 kết quả khớp kèm output dạng stream và tuỳ chọn dòng ngữ cảnh ([#4570](https://github.com/langchain-ai/deepagents/pull/4570), [#4706](https://github.com/langchain-ai/deepagents/pull/4706)).
* **Hỗ trợ prompt caching mở rộng**: Bedrock prompt caching qua extra `deepagents[aws]` ([#4108](https://github.com/langchain-ai/deepagents/issues/4108)), và tự động gán session affinity cho Fireworks prompt cache ([#4598](https://github.com/langchain-ai/deepagents/pull/4598)).
* **Hỗ trợ NVIDIA**: Một harness profile Nemotron 3 Ultra có sẵn cùng gán nhãn nguồn gốc (attribution) app-origin cho NIM. ([#4192](https://github.com/langchain-ai/deepagents/pull/4192), [#4455](https://github.com/langchain-ai/deepagents/pull/4455))

### Breaking changes

* **Todo lập kế hoạch giờ là opt-in**: `create_deep_agent` không còn tự động bao gồm `TodoListMiddleware`, nên tool `write_todos`, state channel `todos`, và prompt lập kế hoạch todo sẽ không có trừ khi bạn khôi phục bằng `middleware=[TodoListMiddleware()]`. (Harness profile OpenAI Codex vẫn tự động bật.) ([#4929](https://github.com/langchain-ai/deepagents/pull/4929))
* **Đã gỡ bỏ các shim tương thích backend**: Hãy truyền các instance `BackendProtocol` cụ thể thay vì factory, cấu hình `StoreBackend` với `namespace` tường minh, và dùng các API `ls` / `glob` / `grep` / `ReadResult` hiện tại. Các symbol đã bị gỡ gồm `BackendFactory`, `BACKEND_TYPES`, `FileFormat`, và `Unset`. File mới lưu `FileData.content` dạng string; nội dung `list[str]` cũ vẫn đọc được và sẽ tự chuyển đổi ở lần ghi tiếp theo. ([#4541](https://github.com/langchain-ai/deepagents/pull/4541))
* **Thay đổi định dạng output**: Output rỗng của `ls` / `glob` giờ là `No files found` thay vì `[]`, và `read_file` không còn render gutter kiểu `cat -n` độ rộng cố định nữa, hãy cập nhật mọi bộ phân tích (parser) đọc output thô của tool.

Copy prompt sau vào AI coding assistant của bạn để migrate codebase theo các breaking change này (nội dung giữ nguyên tiếng Anh vì đây là prompt dùng để dán trực tiếp vào công cụ AI):

!!! note "Prompt migrate deepagents v0.6.x → v0.7"
    Migrate this codebase from `deepagents` v0.6.x to v0.7 to account for the following breaking changes:

    1. `create_deep_agent` no longer includes `TodoListMiddleware` by default. If this codebase relies on the `write_todos` tool, the `todos` state channel, or the todo-planning prompt, restore it by importing `TodoListMiddleware` from `langchain.agents.middleware` (not `deepagents`) and passing it to `create_deep_agent`:

        ```python
        from langchain.agents.middleware import TodoListMiddleware
        from deepagents import create_deep_agent

        agent = create_deep_agent(middleware=[TodoListMiddleware()])
        ```

    2. Backend compatibility shims were removed: `BackendFactory`, `BACKEND_TYPES`, `FileFormat`, and `Unset` no longer exist. Replace any backend factories with concrete `BackendProtocol` instances, and add an explicit `namespace` to every `StoreBackend` configuration:

        ```python
        from deepagents import create_deep_agent
        from deepagents.backends import StoreBackend

        # Before (v0.6.x): factory callable, and StoreBackend with no explicit namespace
        agent = create_deep_agent(backend=lambda rt: StoreBackend())  # [!code --]

        # After (v0.7): concrete backend instance with an explicit namespace
        agent = create_deep_agent(backend=StoreBackend(namespace=lambda rt: (rt.server_info.user.identity,)))  # [!code ++]
        ```

        Also update calls to use the current `ls`, `glob`, `grep`, and `ReadResult` APIs.

    3. Tool output formats changed: empty `ls` / `glob` output is now the string `No files found` instead of `[]`, and `read_file` no longer renders a fixed-width `cat -n`-style line-number gutter. Update any code that parses these tool outputs.

    Search the codebase for usages of the removed symbols and for parsing logic that depends on the old output formats, apply the necessary changes, and flag anything that needs manual review.

### May 12, 2026 · `deepagents`

## `deepagents` v0.6.0

* **[`CodeInterpreterMiddleware`](https://docs.langchain.com/oss/python/deepagents/interpreters)**: (thử nghiệm) `deepagents` giờ hỗ trợ thực thi code và gọi tool theo hướng lập trình (programmatic tool calling) thông qua một QuickJS runtime giới hạn phạm vi (scoped).
* Hỗ trợ `version="v3"` trong `stream_events` / `astream_events`. Xem hướng dẫn [event streaming](https://docs.langchain.com/oss/python/deepagents/event-streaming) để biết chi tiết.
* **[`DeltaChannel`](https://docs.langchain.com/oss/python/langgraph/pregel#deltachannel) (beta)** ([blog](https://www.langchain.com/blog/delta-channels-evolving-agent-runtime)): Deep Agents giờ dùng `DeltaChannel` cho lịch sử message và file của agent. Thay vì serialize lại toàn bộ giá trị tích luỹ vào mỗi checkpoint, chỉ phần delta gia tăng được ghi ở mỗi bước được lưu, giữ kích thước checkpoint nhỏ khi thread kéo dài.
* **[Harness profiles](https://docs.langchain.com/oss/python/deepagents/profiles)**: Đăng ký các bộ cấu hình theo từng provider hoặc model (`HarnessProfile`) mà `create_deep_agent` tự động áp dụng khi một model được chọn, gồm điều chỉnh system prompt, ghi đè tool, thay đổi middleware, và giá trị mặc định cho subagent, mà không cần sửa nơi gọi.
* **[`ContextHubBackend`](https://docs.langchain.com/oss/python/deepagents/backends#contexthubbackend)** ([blog](https://www.langchain.com/blog/introducing-context-hub)): Một backend filesystem mới dựa trên LangSmith Hub. Các file của agent (skill, memory, và context được lưu trữ khác) được lưu dưới dạng Hub commit, mang lại lịch sử phiên bản cho mỗi lần ghi và độ bền vững (durability) native của LangSmith mà không cần thiết lập một LangGraph store riêng.

### May 12, 2026 · `langchain`

## `langchain` v1.3.0

Bản phát hành này thêm hỗ trợ `version="v3"` trong `stream_events` / `astream_events` cho agent `langchain`. Xem hướng dẫn [event streaming](event-streaming.md) để biết chi tiết.

### May 12, 2026 · `langgraph`

## `langgraph` v1.2.0

Bản phát hành này bổ sung khả năng kiểm soát chi tiết hơn với việc thực thi node (timeout, khôi phục lỗi, và tắt an toàn), một loại channel mới giúp giảm overhead checkpoint cho các thread chạy dài, và một API streaming mới lấy content block làm trung tâm (v3) với các projection có kiểu (typed), theo từng channel.

* **[`DeltaChannel`](https://docs.langchain.com/oss/python/langgraph/pregel#deltachannel) (beta)**: Một loại channel mới chỉ lưu phần delta gia tăng ở mỗi bước thay vì serialize lại toàn bộ giá trị tích luỹ. Hữu ích nhất với các channel phình to theo thời gian, ví dụ danh sách message trong một thread chạy dài. Dùng `snapshot_frequency=K` để ghi một snapshot đầy đủ sau mỗi K bước và giới hạn độ trễ đọc.

* **[Timeout theo từng node](https://docs.langchain.com/oss/python/langgraph/fault-tolerance#timeouts)**: Truyền `timeout=` vào [`add_node`](https://reference.langchain.com/python/langgraph/graph/state/StateGraph/add_node) để giới hạn thời gian tối đa cho một lần thử. Đặt giới hạn wall-clock cứng (`run_timeout`), giới hạn idle được reset khi có tiến triển (`idle_timeout`), hoặc cả hai qua [`TimeoutPolicy`](https://reference.langchain.com/python/langgraph/types/TimeoutPolicy). Khi giới hạn kích hoạt, LangGraph raise [`NodeTimeoutError`](https://reference.langchain.com/python/langgraph/errors/NodeTimeoutError), xoá các write của lần thử đó, và chuyển sang retry policy. Chỉ áp dụng cho node bất đồng bộ (async).

* **[Trình xử lý lỗi cấp node](https://docs.langchain.com/oss/python/langgraph/fault-tolerance#error-handling)**: Truyền `error_handler=` vào [`add_node`](https://reference.langchain.com/python/langgraph/graph/state/StateGraph/add_node) để chạy một hàm khôi phục sau khi hết số lần retry. Handler nhận một [`NodeError`](https://reference.langchain.com/python/langgraph/errors/NodeError) có kiểu và có thể trả về một [`Command`](https://reference.langchain.com/python/langgraph/types/Command) để cập nhật state và định tuyến sang node khác, hữu ích cho các pattern Saga/compensation.

* **[Tắt an toàn (graceful shutdown)](https://docs.langchain.com/oss/python/langgraph/fault-tolerance#graceful-shutdown)**: Dừng một lần chạy đang diễn ra một cách hợp tác (cooperative) sau khi superstep hiện tại hoàn tất, và lưu một checkpoint có thể resume. Tạo một [`RunControl`](https://reference.langchain.com/python/langgraph/runtime/RunControl) và gọi `request_drain()` từ bất kỳ thread nào; lần chạy sẽ raise `GraphDrained` và có thể được resume sau với cùng config.

* **API event streaming mới (beta)**: Truyền `version="v3"` vào `stream_events()` / `astream_events()` để có một giao thức lấy content block làm trung tâm, với các projection có kiểu theo từng channel (`run.values`, `run.messages`, `run.lifecycle`, `run.subgraphs`) cùng các transformer tuỳ chọn cho update, custom event, checkpoint, task, và debug. `run.messages` trả về một `ChatModelStream` cho mỗi lần gọi LLM, kèm các sub-projection có kiểu cho text, reasoning, tool call, và usage. `version="v1"` và `version="v2"` không đổi.

Timeout và error handler chỉ dành cho Python; retry policy vẫn hoạt động trên cả Python và TypeScript.

### Apr 7, 2026 · `deepagents`

## `deepagents` v0.5.0

* **[Async subagents](https://docs.langchain.com/oss/python/deepagents/async-subagents)**: Deep Agents có thể khởi chạy các tác vụ nền không chặn (non-blocking), để người dùng tiếp tục tương tác với agent trong khi subagent làm việc song song. Cần [LangSmith Deployment](https://docs.langchain.com/langsmith/deployment) cho sub-agent.

* **Hỗ trợ đa phương tiện (multi-modal)**: Tool `read_file` giờ hỗ trợ PDF, audio, và video ngoài hình ảnh.

* **Thay đổi backend**: Chúng tôi đã thực hiện các thay đổi tương thích ngược cho [backend protocol](https://github.com/langchain-ai/deepagents/blob/main/libs/deepagents/deepagents/backends/protocol.py) của Deep Agents:
    * Cập nhật định dạng file được lưu trong [backend State và Store](https://docs.langchain.com/oss/python/deepagents/backends) để hỗ trợ file nhị phân.
    * Cải thiện việc lan truyền lỗi (error propagation) từ backend đến tool.
    * Giờ bạn có thể khởi tạo `StateBackend()` và `StoreBackend()` trực tiếp. Việc chỉ định bằng factory (ví dụ `backend=(lambda rt: StateBackend(rt))`) đã bị deprecated.

* **Cải thiện prompt caching cho Anthropic**: Chúng tôi đã cải thiện hiệu năng prompt caching cho các model Anthropic.

### Mar 10, 2026 · `langgraph`

## `langgraph` v1.1.0

* **Streaming type-safe (`version="v2"`)**: Truyền `version="v2"` vào `stream()` / `astream()` để nhận output `StreamPart` thống nhất với các key `type`, `ns`, và `data` trên mỗi chunk. Mỗi mode có `TypedDict` riêng, đều import được từ `langgraph.types`. Xem [tài liệu streaming](https://docs.langchain.com/oss/python/langgraph/streaming#stream-output-format-v2).

* **Invoke type-safe (`version="v2"`)**: Truyền `version="v2"` vào `invoke()` / `ainvoke()` để nhận một object `GraphOutput` với thuộc tính `.value` và `.interrupts`. Xem [tài liệu invoke](https://docs.langchain.com/oss/python/langgraph/streaming#v2-invoke-format).

* **Ép kiểu Pydantic và dataclass**: Với `version="v2"`, output của `invoke()` và stream mode `values` được tự động ép kiểu (coerce) sang Pydantic model hoặc dataclass mà bạn khai báo.

* **Sửa lỗi time travel với interrupt và subgraph**: Việc replay không còn tái sử dụng giá trị `RESUME` cũ, và subgraph khôi phục đúng checkpoint cho state lịch sử của parent.

* **Tương thích ngược hoàn toàn**: `version="v2"` là opt-in. `GraphOutput` hỗ trợ truy cập kiểu dict đã deprecated để migrate dần dần.

### Feb 10, 2026 · `deepagents`

## `deepagents` v0.4.0

* Các package tích hợp mới cho sandbox có thể cắm được (pluggable): [`langchain-modal`](https://pypi.org/project/langchain-modal/), [`langchain-daytona`](https://pypi.org/project/langchain-daytona/), và [`langchain-runloop`](https://pypi.org/project/langchain-runloop/). Xem [hướng dẫn sandbox](https://docs.langchain.com/oss/python/deepagents/sandboxes) và [tutorial phân tích dữ liệu](https://docs.langchain.com/oss/python/deepagents/data-analysis) ví dụ.
* Thay đổi trong [tóm tắt lịch sử hội thoại](https://docs.langchain.com/oss/python/deepagents/context-engineering#summarization):
    * Việc tóm tắt giờ diễn ra ở model node qua sự kiện `wrap_model_call`. Nhờ đó chúng ta giữ lại toàn bộ lịch sử message trong graph state.
    * Đếm token chính xác hơn.
    * Việc tóm tắt giờ sẽ tự động kích hoạt nếu một chat model raise [`ContextOverflowError`](https://reference.langchain.com/python/langchain-core/exceptions/ContextOverflowError) (định nghĩa trong `langchain-core`). Hiện `langchain-anthropic` và `langchain-openai` hỗ trợ điều này.
* Chúng tôi giờ mặc định dùng Responses API cho các chuỗi model có tiền tố `"openai:"`.

    **Tắt lưu trữ dữ liệu (data retention) với Responses API**

    ```python
    from langchain.chat_models import init_chat_model

    agent = create_deep_agent(
        model=init_chat_model(
            "openai:...",
            use_responses_api=True,
            store=False,
            include=["reasoning.encrypted_content"],
        )
    )
    ```

### Dec 15, 2025 · `langchain`, `integrations`

## `langchain` v1.2.0

* [`create_agent`](agents.md): Đơn giản hoá hỗ trợ các tham số và định nghĩa tool đặc thù theo provider thông qua thuộc tính [`extras`](https://reference.langchain.com/python/langchain/tools/#langchain.tools.BaseTool.extras) mới trên [tool](tools.md). Ví dụ:
    * Cấu hình đặc thù theo provider như [programmatic tool calling](https://docs.langchain.com/oss/python/integrations/chat/anthropic#programmatic-tool-calling) và [tool search](https://docs.langchain.com/oss/python/integrations/chat/anthropic#tool-search) của Anthropic.
    * Các tool có sẵn được thực thi ở phía client, như được hỗ trợ bởi [Anthropic](https://docs.langchain.com/oss/python/integrations/chat/anthropic#built-in-tools), [OpenAI](https://docs.langchain.com/oss/python/integrations/chat/openai#responses-api), và các provider khác.
* Hỗ trợ tuân thủ schema nghiêm ngặt (strict) trong `response_format` của agent (xem tài liệu [`ProviderStrategy`](structured-output.md#provider-strategy)).

### Dec 8, 2025 · `langchain`, `integrations`

## `langchain-google-genai` v4.0.0

Chúng tôi đã viết lại tích hợp Google GenAI để dùng Generative AI SDK hợp nhất của Google, cung cấp quyền truy cập Gemini API và Vertex AI Platform qua cùng một interface. Điều này bao gồm một số breaking change nhỏ cũng như các package bị deprecated trong `langchain-google-vertexai`.

Xem đầy đủ [release notes và hướng dẫn migration](https://github.com/langchain-ai/langchain-google/discussions/1422) để biết chi tiết.

### Nov 25, 2025 · `langchain`

## `langchain` v1.1.0

* [Model profiles](models.md#model-profiles): Chat model giờ expose các tính năng và khả năng được hỗ trợ qua thuộc tính `.profile`. Dữ liệu này lấy từ [models.dev](https://models.dev), một dự án mã nguồn mở cung cấp dữ liệu khả năng của model.
* [Summarization middleware](middleware/built-in.md#summarization): Cập nhật để hỗ trợ các điểm kích hoạt linh hoạt, dùng model profile cho việc tóm tắt theo ngữ cảnh (context-aware).
* [Structured output](structured-output.md): Hỗ trợ `ProviderStrategy` (structured output native) giờ có thể được suy luận từ model profile.
* [`SystemMessage` cho `create_agent`](middleware/custom.md#dynamic-prompt): Hỗ trợ truyền trực tiếp instance `SystemMessage` vào tham số `system_prompt` của `create_agent`, mở khoá các tính năng nâng cao như cache control và content block có cấu trúc.
* [Model retry middleware](middleware/built-in.md#model-retry): Middleware mới để tự động retry các lần gọi model thất bại với exponential backoff có thể cấu hình.
* [Content moderation middleware](https://docs.langchain.com/oss/python/integrations/middleware/openai#content-moderation): Middleware kiểm duyệt nội dung của OpenAI để phát hiện và xử lý nội dung không an toàn trong tương tác agent. Hỗ trợ kiểm tra input người dùng, output model, và kết quả tool.

### Oct 20, 2025 · `langchain`, `langgraph`

## v1.0.0

### `langchain`

* [Release notes](https://docs.langchain.com/oss/python/releases/langchain-v1)
* [Hướng dẫn migration](https://docs.langchain.com/oss/python/migrate/langchain-v1)

### `langgraph`

* [Release notes](https://docs.langchain.com/oss/python/releases/langgraph-v1)
* [Hướng dẫn migration](https://docs.langchain.com/oss/python/migrate/langgraph-v1)

!!! info "Phản hồi"
    Nếu bạn gặp vấn đề hoặc có phản hồi, vui lòng [mở một issue](https://github.com/langchain-ai/docs/issues/new?template=01-langchain.yml) để chúng tôi cải thiện. Để xem tài liệu v0.x, [truy cập nội dung lưu trữ](https://github.com/langchain-ai/langchain/tree/v0.3/docs/docs) và [API reference](https://reference.langchain.com/v0.3/python/).

---

!!! info "Kết nối tài liệu này"
    [Kết nối tài liệu này](https://docs.langchain.com/use-these-docs) với Claude, VSCode, và nhiều công cụ khác qua MCP để có câu trả lời theo thời gian thực.

!!! info "Đóng góp"
    [Chỉnh sửa trang này trên GitHub](https://github.com/langchain-ai/docs/edit/main/src/oss/python/releases/changelog.mdx) hoặc [báo lỗi](https://github.com/langchain-ai/docs/issues/new/choose).
