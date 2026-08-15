# Context engineering trong agent

## Tổng quan

Phần khó nhất khi xây dựng agent (hoặc bất kỳ ứng dụng LLM nào) là làm cho chúng đủ đáng tin cậy. Dù có thể hoạt động tốt ở bản prototype, chúng thường thất bại trong các use case thực tế.

### Tại sao agent thất bại?

Khi agent thất bại, thường là vì lệnh gọi LLM bên trong agent đã thực hiện hành động sai / không làm đúng như kỳ vọng. LLM thất bại vì một trong hai lý do:

1. LLM nền tảng không đủ năng lực
2. Context "đúng" đã không được truyền cho LLM

Phần lớn các trường hợp, chính lý do thứ hai mới là nguyên nhân khiến agent không đáng tin cậy.

**Context engineering** là việc cung cấp đúng thông tin và tool theo đúng định dạng để LLM có thể hoàn thành một task. Đây là công việc số một của AI Engineer. Việc thiếu context "đúng" là điểm nghẽn số một cản trở agent đáng tin cậy hơn, và các abstraction agent của LangChain được thiết kế đặc biệt để hỗ trợ context engineering.

!!! tip "Mẹo"
    Mới tìm hiểu context engineering? Hãy bắt đầu với [tổng quan khái niệm](https://docs.langchain.com/oss/python/concepts/context) để hiểu các loại context khác nhau và khi nào nên dùng chúng.

### Vòng lặp agent

Một vòng lặp agent điển hình gồm hai bước chính:

1. **Model call** – gọi LLM với một prompt và các tool khả dụng, trả về hoặc là một response hoặc một yêu cầu thực thi tool
2. **Tool execution** – thực thi các tool mà LLM đã yêu cầu, trả về kết quả tool

![Sơ đồ vòng lặp agent cốt lõi](https://mintcdn.com/langchain-5e9cc07a/Tazq8zGc0yYUYrDl/oss/images/core_agent_loop.png?fit=max&auto=format&n=Tazq8zGc0yYUYrDl&q=85&s=ac72e48317a9ced68fd1be64e89ec063)

Vòng lặp này tiếp diễn cho tới khi LLM quyết định kết thúc.

### Những gì bạn có thể kiểm soát

Để xây dựng agent đáng tin cậy, bạn cần kiểm soát những gì xảy ra ở mỗi bước của vòng lặp agent, cũng như những gì xảy ra giữa các bước.

| Loại context                                   | Bạn kiểm soát gì                                                                      | Tạm thời hay lâu dài  |
| ----------------------------------------------- | -------------------------------------------------------------------------------------- | --------------------- |
| **[Model Context](#model-context)**            | Những gì đưa vào lệnh gọi model (instruction, lịch sử message, tool, response format)  | Tạm thời (transient)  |
| **[Tool Context](#tool-context)**              | Tool có thể truy cập và tạo ra gì (đọc/ghi vào state, store, runtime context)          | Lâu dài (persistent)  |
| **[Life-cycle Context](#life-cycle-context)**  | Những gì xảy ra giữa lệnh gọi model và tool (tóm tắt, guardrail, logging, v.v.)        | Lâu dài (persistent)  |

**Context tạm thời (transient)**: Những gì LLM nhìn thấy cho một lệnh gọi đơn lẻ. Bạn có thể sửa message, tool, hoặc prompt mà không thay đổi những gì được lưu trong state.

**Context lâu dài (persistent)**: Những gì được lưu vào state qua các lượt. Life-cycle hook và tool write sẽ sửa đổi phần này vĩnh viễn.

### Nguồn dữ liệu

Trong suốt quá trình này, agent của bạn truy cập (đọc / ghi) các nguồn dữ liệu khác nhau:

| Nguồn dữ liệu         | Còn gọi là            | Phạm vi                | Ví dụ                                                                       |
| ---------------------- | ---------------------- | ------------------------ | ------------------------------------------------------------------------- |
| **Runtime Context**    | Cấu hình tĩnh          | Theo phạm vi hội thoại   | User ID, API key, kết nối database, quyền hạn, cấu hình môi trường        |
| **State**               | Short-term memory      | Theo phạm vi hội thoại   | Message hiện tại, file đã upload, trạng thái xác thực, kết quả tool       |
| **Store**               | Long-term memory       | Xuyên suốt các hội thoại | Sở thích người dùng, insight đã trích xuất, memory, dữ liệu lịch sử       |

### Cách hoạt động

[Middleware](middleware/overview.md) của LangChain là cơ chế nền tảng giúp context engineering trở nên khả thi cho các developer dùng LangChain.

Middleware cho phép bạn gắn (hook) vào bất kỳ bước nào trong vòng đời agent và:

* Cập nhật context
* Nhảy tới một bước khác trong vòng đời agent

Xuyên suốt hướng dẫn này, bạn sẽ thấy middleware API được dùng thường xuyên như một phương tiện để đạt mục tiêu context engineering.

## Model context

Kiểm soát những gì đưa vào mỗi lệnh gọi model, gồm instruction, các tool khả dụng, model nào được dùng, và định dạng output. Những quyết định này ảnh hưởng trực tiếp tới độ tin cậy và chi phí.

* **[System Prompt](#system-prompt)**: Instruction nền tảng từ developer gửi tới LLM.
* **[Messages](#messages)**: Danh sách đầy đủ các message (lịch sử hội thoại) gửi tới LLM.
* **[Tools](#tools)**: Các tiện ích mà agent có quyền truy cập để thực hiện hành động.
* **[Model](#model)**: Model thực tế (kèm cấu hình) sẽ được gọi.
* **[Response Format](#response-format)**: Đặc tả schema cho response cuối cùng của model.

Tất cả các loại model context này đều có thể lấy dữ liệu từ **state** (short-term memory), **store** (long-term memory), hoặc **runtime context** (cấu hình tĩnh).

### System Prompt

System prompt định hình hành vi và năng lực của LLM. Người dùng, ngữ cảnh, hoặc giai đoạn hội thoại khác nhau cần instruction khác nhau. Các agent thành công tận dụng memory, sở thích, và cấu hình để cung cấp đúng instruction cho trạng thái hiện tại của hội thoại.

=== "State"

    Truy cập số lượng message hoặc ngữ cảnh hội thoại từ state:

    ```python
    from langchain.agents import create_agent
    from langchain.agents.middleware import dynamic_prompt, ModelRequest

    @dynamic_prompt
    def state_aware_prompt(request: ModelRequest) -> str:
        # request.messages là cách viết tắt của request.state["messages"]
        message_count = len(request.messages)

        base = "You are a helpful assistant."

        if message_count > 10:
            base += "\nThis is a long conversation - be extra concise."

        return base

    agent = create_agent(
        model="gpt-5.5",
        tools=[...],
        middleware=[state_aware_prompt]
    )
    ```

=== "Store"

    Truy cập sở thích người dùng từ long-term memory:

    ```python
    from dataclasses import dataclass
    from langchain.agents import create_agent
    from langchain.agents.middleware import dynamic_prompt, ModelRequest
    from langgraph.store.memory import InMemoryStore

    @dataclass
    class Context:
        user_id: str

    @dynamic_prompt
    def store_aware_prompt(request: ModelRequest) -> str:
        user_id = request.runtime.context.user_id

        # Đọc từ Store: lấy sở thích người dùng
        store = request.runtime.store
        user_prefs = store.get(("preferences",), user_id)

        base = "You are a helpful assistant."

        if user_prefs:
            style = user_prefs.value.get("communication_style", "balanced")
            base += f"\nUser prefers {style} responses."

        return base

    agent = create_agent(
        model="gpt-5.5",
        tools=[...],
        middleware=[store_aware_prompt],
        context_schema=Context,
        store=InMemoryStore()
    )
    ```

=== "Runtime Context"

    Truy cập user ID hoặc cấu hình từ Runtime Context:

    ```python
    from dataclasses import dataclass
    from langchain.agents import create_agent
    from langchain.agents.middleware import dynamic_prompt, ModelRequest

    @dataclass
    class Context:
        user_role: str
        deployment_env: str

    @dynamic_prompt
    def context_aware_prompt(request: ModelRequest) -> str:
        # Đọc từ Runtime Context: user role và environment
        user_role = request.runtime.context.user_role
        env = request.runtime.context.deployment_env

        base = "You are a helpful assistant."

        if user_role == "admin":
            base += "\nYou have admin access. You can perform all operations."
        elif user_role == "viewer":
            base += "\nYou have read-only access. Guide users to read operations only."

        if env == "production":
            base += "\nBe extra careful with any data modifications."

        return base

    agent = create_agent(
        model="gpt-5.5",
        tools=[...],
        middleware=[context_aware_prompt],
        context_schema=Context
    )
    ```

### Messages

Message tạo nên prompt được gửi tới LLM.
Việc quản lý nội dung message là yếu tố then chốt để đảm bảo LLM có đúng thông tin để phản hồi tốt.

=== "State"

    Đưa (inject) ngữ cảnh file đã upload từ State khi liên quan tới truy vấn hiện tại:

    ```python
    from langchain.agents import create_agent
    from langchain.agents.middleware import wrap_model_call, ModelRequest, ModelResponse
    from typing import Callable

    @wrap_model_call
    def inject_file_context(
        request: ModelRequest,
        handler: Callable[[ModelRequest], ModelResponse]
    ) -> ModelResponse:
        """Inject context about files user has uploaded this session."""
        # Đọc từ State: lấy metadata các file đã upload
        uploaded_files = request.state.get("uploaded_files", [])

        if uploaded_files:
            # Xây dựng ngữ cảnh về các file khả dụng
            file_descriptions = []
            for file in uploaded_files:
                file_descriptions.append(
                    f"- {file['name']} ({file['type']}): {file['summary']}"
                )

            file_context = f"""Files you have access to in this conversation:
    {chr(10).join(file_descriptions)}

    Reference these files when answering questions."""

            # Đưa ngữ cảnh file vào trước các message gần đây
            messages = [
                *request.messages,
                {"role": "user", "content": file_context},
            ]
            request = request.override(messages=messages)

        return handler(request)

    agent = create_agent(
        model="gpt-5.5",
        tools=[...],
        middleware=[inject_file_context]
    )
    ```

=== "Store"

    Đưa văn phong viết email của người dùng từ Store vào để định hướng soạn thảo:

    ```python
    from dataclasses import dataclass
    from langchain.agents import create_agent
    from langchain.agents.middleware import wrap_model_call, ModelRequest, ModelResponse
    from typing import Callable
    from langgraph.store.memory import InMemoryStore

    @dataclass
    class Context:
        user_id: str

    @wrap_model_call
    def inject_writing_style(
        request: ModelRequest,
        handler: Callable[[ModelRequest], ModelResponse]
    ) -> ModelResponse:
        """Inject user's email writing style from Store."""
        user_id = request.runtime.context.user_id

        # Đọc từ Store: lấy ví dụ văn phong viết của người dùng
        store = request.runtime.store
        writing_style = store.get(("writing_style",), user_id)

        if writing_style:
            style = writing_style.value
            # Xây dựng style guide từ các ví dụ đã lưu
            style_context = f"""Your writing style:
    - Tone: {style.get('tone', 'professional')}
    - Typical greeting: "{style.get('greeting', 'Hi')}"
    - Typical sign-off: "{style.get('sign_off', 'Best')}"
    - Example email you've written:
    {style.get('example_email', '')}"""

            # Thêm vào cuối - model chú ý nhiều hơn tới các message cuối cùng
            messages = [
                *request.messages,
                {"role": "user", "content": style_context}
            ]
            request = request.override(messages=messages)

        return handler(request)

    agent = create_agent(
        model="gpt-5.5",
        tools=[...],
        middleware=[inject_writing_style],
        context_schema=Context,
        store=InMemoryStore()
    )
    ```

=== "Runtime Context"

    Đưa quy tắc tuân thủ (compliance) từ Runtime Context vào dựa trên khu vực pháp lý của người dùng:

    ```python
    from dataclasses import dataclass
    from langchain.agents import create_agent
    from langchain.agents.middleware import wrap_model_call, ModelRequest, ModelResponse
    from typing import Callable

    @dataclass
    class Context:
        user_jurisdiction: str
        industry: str
        compliance_frameworks: list[str]

    @wrap_model_call
    def inject_compliance_rules(
        request: ModelRequest,
        handler: Callable[[ModelRequest], ModelResponse]
    ) -> ModelResponse:
        """Inject compliance constraints from Runtime Context."""
        # Đọc từ Runtime Context: lấy yêu cầu tuân thủ
        jurisdiction = request.runtime.context.user_jurisdiction
        industry = request.runtime.context.industry
        frameworks = request.runtime.context.compliance_frameworks

        # Xây dựng các ràng buộc tuân thủ
        rules = []
        if "GDPR" in frameworks:
            rules.append("- Must obtain explicit consent before processing personal data")
            rules.append("- Users have right to data deletion")
        if "HIPAA" in frameworks:
            rules.append("- Cannot share patient health information without authorization")
            rules.append("- Must use secure, encrypted communication")
        if industry == "finance":
            rules.append("- Cannot provide financial advice without proper disclaimers")

        if rules:
            compliance_context = f"""Compliance requirements for {jurisdiction}:
    {chr(10).join(rules)}"""

            # Thêm vào cuối - model chú ý nhiều hơn tới các message cuối cùng
            messages = [
                *request.messages,
                {"role": "user", "content": compliance_context}
            ]
            request = request.override(messages=messages)

        return handler(request)

    agent = create_agent(
        model="gpt-5.5",
        tools=[...],
        middleware=[inject_compliance_rules],
        context_schema=Context
    )
    ```

!!! note "Ghi chú"
    **Cập nhật message tạm thời so với lâu dài:**

    Các ví dụ trên dùng `wrap_model_call` để tạo các cập nhật **tạm thời** (transient), tức là sửa những message được gửi tới model cho một lệnh gọi đơn lẻ mà không thay đổi những gì được lưu trong state.

    Đối với các cập nhật **lâu dài** (persistent) làm thay đổi state, bạn có thể:

    * Trả về một [`ExtendedModelResponse`](https://reference.langchain.com/python/langchain/agents/middleware/types/ExtendedModelResponse) kèm [`Command`](https://reference.langchain.com/python/langgraph/types/Command) từ `wrap_model_call` để đưa các cập nhật state vào từ tầng model call.
    * Dùng các life-cycle hook như `before_model`, `after_model`, hoặc `wrap_tool_call` (cho kết quả trả về của tool) để cập nhật lịch sử hội thoại. Xem [tài liệu middleware](middleware/overview.md) để biết thêm chi tiết.

    Xem [Cập nhật state](https://docs.langchain.com/oss/python/langchain/middleware/custom#state-updates) để biết thêm thông tin.

### Tools

Tool cho phép model tương tác với database, API, và hệ thống bên ngoài. Cách bạn định nghĩa và chọn tool ảnh hưởng trực tiếp tới việc model có hoàn thành task hiệu quả hay không.

#### Định nghĩa tool

Mỗi tool cần có tên rõ ràng, mô tả, tên tham số, và mô tả tham số. Đây không chỉ là metadata, chúng định hướng suy luận của model về thời điểm và cách dùng tool.

```python
from langchain.tools import tool

@tool(parse_docstring=True)
def search_orders(
    user_id: str,
    status: str,
    limit: int = 10
) -> str:
    """Search for user orders by status.

    Use this when the user asks about order history or wants to check
    order status. Always filter by the provided status.

    Args:
        user_id: Unique identifier for the user
        status: Order status: 'pending', 'shipped', or 'delivered'
        limit: Maximum number of results to return
    """
    # Implementation here
    pass
```

#### Chọn tool

Không phải tool nào cũng phù hợp với mọi tình huống. Quá nhiều tool có thể làm quá tải model (overload context) và tăng lỗi; quá ít tool sẽ giới hạn năng lực. Chọn tool động điều chỉnh bộ tool khả dụng dựa trên trạng thái xác thực, quyền người dùng, feature flag, hoặc giai đoạn hội thoại.

=== "State"

    Chỉ bật tool nâng cao sau một số mốc nhất định trong hội thoại:

    ```python
    from langchain.agents import create_agent
    from langchain.agents.middleware import wrap_model_call, ModelRequest, ModelResponse
    from typing import Callable

    @wrap_model_call
    def state_based_tools(
        request: ModelRequest,
        handler: Callable[[ModelRequest], ModelResponse]
    ) -> ModelResponse:
        """Filter tools based on conversation State."""
        # Đọc từ State: kiểm tra người dùng đã xác thực chưa
        state = request.state
        is_authenticated = state.get("authenticated", False)
        message_count = len(state["messages"])

        # Chỉ bật tool nhạy cảm sau khi xác thực
        if not is_authenticated:
            tools = [t for t in request.tools if t.name.startswith("public_")]
            request = request.override(tools=tools)
        elif message_count < 5:
            # Giới hạn tool ở đầu hội thoại
            tools = [t for t in request.tools if t.name != "advanced_search"]
            request = request.override(tools=tools)

        return handler(request)

    agent = create_agent(
        model="gpt-5.5",
        tools=[public_search, private_search, advanced_search],
        middleware=[state_based_tools]
    )
    ```

=== "Store"

    Lọc tool dựa trên sở thích người dùng hoặc feature flag trong Store:

    ```python
    from dataclasses import dataclass
    from langchain.agents import create_agent
    from langchain.agents.middleware import wrap_model_call, ModelRequest, ModelResponse
    from typing import Callable
    from langgraph.store.memory import InMemoryStore

    @dataclass
    class Context:
        user_id: str

    @wrap_model_call
    def store_based_tools(
        request: ModelRequest,
        handler: Callable[[ModelRequest], ModelResponse]
    ) -> ModelResponse:
        """Filter tools based on Store preferences."""
        user_id = request.runtime.context.user_id

        # Đọc từ Store: lấy tính năng đã bật của người dùng
        store = request.runtime.store
        feature_flags = store.get(("features",), user_id)

        if feature_flags:
            enabled_features = feature_flags.value.get("enabled_tools", [])
            # Chỉ giữ lại các tool được bật cho người dùng này
            tools = [t for t in request.tools if t.name in enabled_features]
            request = request.override(tools=tools)

        return handler(request)

    agent = create_agent(
        model="gpt-5.5",
        tools=[search_tool, analysis_tool, export_tool],
        middleware=[store_based_tools],
        context_schema=Context,
        store=InMemoryStore()
    )
    ```

=== "Runtime Context"

    Lọc tool dựa trên quyền người dùng từ Runtime Context:

    ```python
    from dataclasses import dataclass
    from langchain.agents import create_agent
    from langchain.agents.middleware import wrap_model_call, ModelRequest, ModelResponse
    from typing import Callable

    @dataclass
    class Context:
        user_role: str

    @wrap_model_call
    def context_based_tools(
        request: ModelRequest,
        handler: Callable[[ModelRequest], ModelResponse]
    ) -> ModelResponse:
        """Filter tools based on Runtime Context permissions."""
        # Đọc từ Runtime Context: lấy user role
        user_role = request.runtime.context.user_role

        if user_role == "admin":
            # Admin được dùng tất cả tool
            pass
        elif user_role == "editor":
            # Editor không được xoá
            tools = [t for t in request.tools if t.name != "delete_data"]
            request = request.override(tools=tools)
        else:
            # Viewer chỉ được dùng tool đọc
            tools = [t for t in request.tools if t.name.startswith("read_")]
            request = request.override(tools=tools)

        return handler(request)

    agent = create_agent(
        model="gpt-5.5",
        tools=[read_data, write_data, delete_data],
        middleware=[context_based_tools],
        context_schema=Context
    )
    ```

Xem [Dynamic tools](https://docs.langchain.com/oss/python/langchain/tools#dynamic-tool-selection) để biết cách vừa lọc các tool đã đăng ký sẵn, vừa đăng ký tool tại runtime (ví dụ, từ MCP server).

### Model

Các model khác nhau có điểm mạnh, chi phí, và context window khác nhau. Chọn đúng model cho task hiện tại, việc này có thể thay đổi trong quá trình chạy agent.

=== "State"

    Dùng model khác nhau dựa trên độ dài hội thoại từ State:

    ```python
    from langchain.agents import create_agent
    from langchain.agents.middleware import wrap_model_call, ModelRequest, ModelResponse
    from langchain.chat_models import init_chat_model
    from typing import Callable

    # Khởi tạo model một lần bên ngoài middleware
    large_model = init_chat_model("claude-sonnet-4-6")
    standard_model = init_chat_model("gpt-5.5")
    efficient_model = init_chat_model("gpt-5.4-mini")

    @wrap_model_call
    def state_based_model(
        request: ModelRequest,
        handler: Callable[[ModelRequest], ModelResponse]
    ) -> ModelResponse:
        """Select model based on State conversation length."""
        # request.messages là cách viết tắt của request.state["messages"]
        message_count = len(request.messages)

        if message_count > 20:
            # Hội thoại dài - dùng model có context window lớn hơn
            model = large_model
        elif message_count > 10:
            # Hội thoại trung bình
            model = standard_model
        else:
            # Hội thoại ngắn - dùng model hiệu quả
            model = efficient_model

        request = request.override(model=model)

        return handler(request)

    agent = create_agent(
        model="gpt-5.4-mini",
        tools=[...],
        middleware=[state_based_model]
    )
    ```

=== "Store"

    Dùng model ưa thích của người dùng từ Store:

    ```python
    from dataclasses import dataclass
    from langchain.agents import create_agent
    from langchain.agents.middleware import wrap_model_call, ModelRequest, ModelResponse
    from langchain.chat_models import init_chat_model
    from typing import Callable
    from langgraph.store.memory import InMemoryStore

    @dataclass
    class Context:
        user_id: str

    # Khởi tạo các model khả dụng một lần
    MODEL_MAP = {
        "gpt-5.5": init_chat_model("gpt-5.5"),
        "gpt-5.4-mini": init_chat_model("gpt-5.4-mini"),
        "claude-sonnet": init_chat_model("claude-sonnet-4-6"),
    }

    @wrap_model_call
    def store_based_model(
        request: ModelRequest,
        handler: Callable[[ModelRequest], ModelResponse]
    ) -> ModelResponse:
        """Select model based on Store preferences."""
        user_id = request.runtime.context.user_id

        # Đọc từ Store: lấy model ưa thích của người dùng
        store = request.runtime.store
        user_prefs = store.get(("preferences",), user_id)

        if user_prefs:
            preferred_model = user_prefs.value.get("preferred_model")
            if preferred_model and preferred_model in MODEL_MAP:
                request = request.override(model=MODEL_MAP[preferred_model])

        return handler(request)

    agent = create_agent(
        model="gpt-5.5",
        tools=[...],
        middleware=[store_based_model],
        context_schema=Context,
        store=InMemoryStore()
    )
    ```

=== "Runtime Context"

    Chọn model dựa trên giới hạn chi phí hoặc môi trường từ Runtime Context:

    ```python
    from dataclasses import dataclass
    from langchain.agents import create_agent
    from langchain.agents.middleware import wrap_model_call, ModelRequest, ModelResponse
    from langchain.chat_models import init_chat_model
    from typing import Callable

    @dataclass
    class Context:
        cost_tier: str
        environment: str

    # Khởi tạo model một lần bên ngoài middleware
    premium_model = init_chat_model("claude-sonnet-4-6")
    standard_model = init_chat_model("gpt-5.5")
    budget_model = init_chat_model("gpt-5.4-mini")

    @wrap_model_call
    def context_based_model(
        request: ModelRequest,
        handler: Callable[[ModelRequest], ModelResponse]
    ) -> ModelResponse:
        """Select model based on Runtime Context."""
        # Đọc từ Runtime Context: cost tier và environment
        cost_tier = request.runtime.context.cost_tier
        environment = request.runtime.context.environment

        if environment == "production" and cost_tier == "premium":
            # Người dùng premium ở production nhận model tốt nhất
            model = premium_model
        elif cost_tier == "budget":
            # Tier budget nhận model hiệu quả
            model = budget_model
        else:
            # Tier tiêu chuẩn
            model = standard_model

        request = request.override(model=model)

        return handler(request)

    agent = create_agent(
        model="gpt-5.5",
        tools=[...],
        middleware=[context_based_model],
        context_schema=Context
    )
    ```

Xem [Dynamic model](https://docs.langchain.com/oss/python/langchain/models#dynamic-model-selection) để biết thêm ví dụ.

### Response format

Structured output biến văn bản phi cấu trúc thành dữ liệu có cấu trúc, đã validate. Khi trích xuất các trường cụ thể hoặc trả dữ liệu cho hệ thống downstream, văn bản tự do là không đủ.

**Cách hoạt động:** Khi bạn cung cấp một schema làm response format, response cuối cùng của model được đảm bảo tuân theo schema đó. Agent chạy vòng lặp model / tool calling cho tới khi model gọi tool xong, sau đó response cuối cùng được ép (coerce) về định dạng đã cung cấp.

#### Định nghĩa format

Định nghĩa schema định hướng model. Tên field, kiểu dữ liệu, và mô tả xác định chính xác định dạng mà output cần tuân theo.

```python
from pydantic import BaseModel, Field

class CustomerSupportTicket(BaseModel):
    """Structured ticket information extracted from customer message."""

    category: str = Field(
        description="Issue category: 'billing', 'technical', 'account', or 'product'"
    )
    priority: str = Field(
        description="Urgency level: 'low', 'medium', 'high', or 'critical'"
    )
    summary: str = Field(
        description="One-sentence summary of the customer's issue"
    )
    customer_sentiment: str = Field(
        description="Customer's emotional tone: 'frustrated', 'neutral', or 'satisfied'"
    )
```

#### Chọn format

Chọn response format động điều chỉnh schema dựa trên sở thích người dùng, giai đoạn hội thoại, hoặc vai trò, trả về format đơn giản ở giai đoạn đầu và format chi tiết hơn khi độ phức tạp tăng lên.

=== "State"

    Cấu hình structured output dựa trên trạng thái hội thoại:

    ```python
    from langchain.agents import create_agent
    from langchain.agents.middleware import wrap_model_call, ModelRequest, ModelResponse
    from pydantic import BaseModel, Field
    from typing import Callable

    class SimpleResponse(BaseModel):
        """Simple response for early conversation."""
        answer: str = Field(description="A brief answer")

    class DetailedResponse(BaseModel):
        """Detailed response for established conversation."""
        answer: str = Field(description="A detailed answer")
        reasoning: str = Field(description="Explanation of reasoning")
        confidence: float = Field(description="Confidence score 0-1")

    @wrap_model_call
    def state_based_output(
        request: ModelRequest,
        handler: Callable[[ModelRequest], ModelResponse]
    ) -> ModelResponse:
        """Select output format based on State."""
        # request.messages là cách viết tắt của request.state["messages"]
        message_count = len(request.messages)

        if message_count < 3:
            # Hội thoại mới bắt đầu - dùng format đơn giản
            request = request.override(response_format=SimpleResponse)
        else:
            # Hội thoại đã ổn định - dùng format chi tiết
            request = request.override(response_format=DetailedResponse)

        return handler(request)

    agent = create_agent(
        model="gpt-5.5",
        tools=[...],
        middleware=[state_based_output]
    )
    ```

=== "Store"

    Cấu hình response format dựa trên sở thích người dùng trong Store:

    ```python
    from dataclasses import dataclass
    from langchain.agents import create_agent
    from langchain.agents.middleware import wrap_model_call, ModelRequest, ModelResponse
    from pydantic import BaseModel, Field
    from typing import Callable
    from langgraph.store.memory import InMemoryStore

    @dataclass
    class Context:
        user_id: str

    class VerboseResponse(BaseModel):
        """Verbose response with details."""
        answer: str = Field(description="Detailed answer")
        sources: list[str] = Field(description="Sources used")

    class ConciseResponse(BaseModel):
        """Concise response."""
        answer: str = Field(description="Brief answer")

    @wrap_model_call
    def store_based_output(
        request: ModelRequest,
        handler: Callable[[ModelRequest], ModelResponse]
    ) -> ModelResponse:
        """Select output format based on Store preferences."""
        user_id = request.runtime.context.user_id

        # Đọc từ Store: lấy văn phong response ưa thích của người dùng
        store = request.runtime.store
        user_prefs = store.get(("preferences",), user_id)

        if user_prefs:
            style = user_prefs.value.get("response_style", "concise")
            if style == "verbose":
                request = request.override(response_format=VerboseResponse)
            else:
                request = request.override(response_format=ConciseResponse)

        return handler(request)

    agent = create_agent(
        model="gpt-5.5",
        tools=[...],
        middleware=[store_based_output],
        context_schema=Context,
        store=InMemoryStore()
    )
    ```

=== "Runtime Context"

    Cấu hình response format dựa trên Runtime Context như user role hoặc environment:

    ```python
    from dataclasses import dataclass
    from langchain.agents import create_agent
    from langchain.agents.middleware import wrap_model_call, ModelRequest, ModelResponse
    from pydantic import BaseModel, Field
    from typing import Callable

    @dataclass
    class Context:
        user_role: str
        environment: str

    class AdminResponse(BaseModel):
        """Response with technical details for admins."""
        answer: str = Field(description="Answer")
        debug_info: dict = Field(description="Debug information")
        system_status: str = Field(description="System status")

    class UserResponse(BaseModel):
        """Simple response for regular users."""
        answer: str = Field(description="Answer")

    @wrap_model_call
    def context_based_output(
        request: ModelRequest,
        handler: Callable[[ModelRequest], ModelResponse]
    ) -> ModelResponse:
        """Select output format based on Runtime Context."""
        # Đọc từ Runtime Context: user role và environment
        user_role = request.runtime.context.user_role
        environment = request.runtime.context.environment

        if user_role == "admin" and environment == "production":
            # Admin ở production nhận output chi tiết
            request = request.override(response_format=AdminResponse)
        else:
            # Người dùng thường nhận output đơn giản
            request = request.override(response_format=UserResponse)

        return handler(request)

    agent = create_agent(
        model="gpt-5.5",
        tools=[...],
        middleware=[context_based_output],
        context_schema=Context
    )
    ```

## Tool context

Tool có điểm đặc biệt là chúng vừa đọc vừa ghi context.

Trong trường hợp cơ bản nhất, khi một tool thực thi, nó nhận các tham số request của LLM và trả về một tool message. Tool thực hiện công việc của nó và tạo ra một kết quả.

Tool cũng có thể lấy thông tin quan trọng cho model, giúp model thực hiện và hoàn thành task.

### Đọc (Reads)

Hầu hết tool trong thực tế cần nhiều hơn chỉ tham số của LLM. Chúng cần user ID cho truy vấn database, API key cho dịch vụ bên ngoài, hoặc trạng thái session hiện tại để ra quyết định. Tool đọc từ state, store, và runtime context để truy cập thông tin này.

=== "State"

    Đọc từ State để kiểm tra thông tin session hiện tại:

    ```python
    from langchain.tools import tool, ToolRuntime
    from langchain.agents import create_agent

    @tool
    def check_authentication(
        runtime: ToolRuntime
    ) -> str:
        """Check if user is authenticated."""
        # Đọc từ State: kiểm tra trạng thái xác thực hiện tại
        current_state = runtime.state
        is_authenticated = current_state.get("authenticated", False)

        if is_authenticated:
            return "User is authenticated"
        else:
            return "User is not authenticated"

    agent = create_agent(
        model="gpt-5.5",
        tools=[check_authentication]
    )
    ```

=== "Store"

    Đọc từ Store để truy cập sở thích người dùng đã được lưu bền vững:

    ```python
    from dataclasses import dataclass
    from langchain.tools import tool, ToolRuntime
    from langchain.agents import create_agent
    from langgraph.store.memory import InMemoryStore

    @dataclass
    class Context:
        user_id: str

    @tool
    def get_preference(
        preference_key: str,
        runtime: ToolRuntime[Context]
    ) -> str:
        """Get user preference from Store."""
        user_id = runtime.context.user_id

        # Đọc từ Store: lấy các sở thích hiện có
        store = runtime.store
        existing_prefs = store.get(("preferences",), user_id)

        if existing_prefs:
            value = existing_prefs.value.get(preference_key)
            return f"{preference_key}: {value}" if value else f"No preference set for {preference_key}"
        else:
            return "No preferences found"

    agent = create_agent(
        model="gpt-5.5",
        tools=[get_preference],
        context_schema=Context,
        store=InMemoryStore()
    )
    ```

=== "Runtime Context"

    Đọc từ Runtime Context cho cấu hình như API key và user ID:

    ```python
    from dataclasses import dataclass
    from langchain.tools import tool, ToolRuntime
    from langchain.agents import create_agent

    @dataclass
    class Context:
        user_id: str
        api_key: str
        db_connection: str

    @tool
    def fetch_user_data(
        query: str,
        runtime: ToolRuntime[Context]
    ) -> str:
        """Fetch data using Runtime Context configuration."""
        # Đọc từ Runtime Context: lấy API key và kết nối DB
        user_id = runtime.context.user_id
        api_key = runtime.context.api_key
        db_connection = runtime.context.db_connection

        # Dùng cấu hình để lấy dữ liệu
        results = perform_database_query(db_connection, query, api_key)

        return f"Found {len(results)} results for user {user_id}"

    agent = create_agent(
        model="gpt-5.5",
        tools=[fetch_user_data],
        context_schema=Context
    )

    # Invoke kèm runtime context
    result = agent.invoke(
        {"messages": [{"role": "user", "content": "Get my data"}]},
        context=Context(
            user_id="user_123",
            api_key="sk-...",
            db_connection="postgresql://..."
        )
    )
    ```

### Ghi (Writes)

Kết quả tool có thể được dùng để giúp agent hoàn thành một task cho trước. Tool có thể vừa trả kết quả trực tiếp cho model, vừa cập nhật memory của agent để làm cho context quan trọng khả dụng cho các bước sau.

=== "State"

    Ghi vào State để theo dõi thông tin riêng của session bằng Command:

    ```python
    from langchain.tools import tool, ToolRuntime
    from langchain.agents import create_agent
    from langgraph.types import Command

    @tool
    def authenticate_user(
        password: str,
        runtime: ToolRuntime
    ) -> Command:
        """Authenticate user and update State."""
        # Thực hiện xác thực (đơn giản hoá)
        if password == "correct":
            # Ghi vào State: đánh dấu đã xác thực bằng Command
            return Command(
                update={"authenticated": True},
            )
        else:
            return Command(update={"authenticated": False})

    agent = create_agent(
        model="gpt-5.5",
        tools=[authenticate_user]
    )
    ```

=== "Store"

    Ghi vào Store để lưu dữ liệu bền vững xuyên suốt các session:

    ```python
    from dataclasses import dataclass
    from langchain.tools import tool, ToolRuntime
    from langchain.agents import create_agent
    from langgraph.store.memory import InMemoryStore

    @dataclass
    class Context:
        user_id: str

    @tool
    def save_preference(
        preference_key: str,
        preference_value: str,
        runtime: ToolRuntime[Context]
    ) -> str:
        """Save user preference to Store."""
        user_id = runtime.context.user_id

        # Đọc các sở thích hiện có
        store = runtime.store
        existing_prefs = store.get(("preferences",), user_id)

        # Gộp với sở thích mới
        prefs = existing_prefs.value if existing_prefs else {}
        prefs[preference_key] = preference_value

        # Ghi vào Store: lưu sở thích đã cập nhật
        store.put(("preferences",), user_id, prefs)

        return f"Saved preference: {preference_key} = {preference_value}"

    agent = create_agent(
        model="gpt-5.5",
        tools=[save_preference],
        context_schema=Context,
        store=InMemoryStore()
    )
    ```

Xem [Tools](https://docs.langchain.com/oss/python/langchain/tools) để biết các ví dụ đầy đủ về truy cập state, store, và runtime context trong tool.

## Life-cycle context

Kiểm soát những gì xảy ra **giữa** các bước cốt lõi của agent, can thiệp vào luồng dữ liệu để triển khai các mối quan tâm xuyên suốt (cross-cutting concern) như tóm tắt, guardrail, và logging.

Như đã thấy ở [Model Context](#model-context) và [Tool Context](#tool-context), [middleware](middleware/overview.md) là cơ chế giúp context engineering trở nên khả thi. Middleware cho phép bạn gắn vào bất kỳ bước nào trong vòng đời agent và:

1. **Cập nhật context** – Sửa state và store để lưu thay đổi bền vững, cập nhật lịch sử hội thoại, hoặc lưu insight
2. **Nhảy trong vòng đời** – Chuyển tới các bước khác trong chu kỳ agent dựa trên context (ví dụ: bỏ qua thực thi tool nếu một điều kiện được đáp ứng, lặp lại lệnh gọi model với context đã sửa đổi)

![Middleware hook trong vòng lặp agent](https://mintcdn.com/langchain-5e9cc07a/RAP6mjwE5G00xYsA/oss/images/middleware_final.png?fit=max&auto=format&n=RAP6mjwE5G00xYsA&q=85&s=eb4404b137edec6f6f0c8ccb8323eaf1)

### Ví dụ: Tóm tắt (Summarization)

Một trong những pattern life-cycle phổ biến nhất là tự động nén lịch sử hội thoại khi nó trở nên quá dài. Khác với việc cắt tỉa (trim) message tạm thời đã trình bày ở [Model Context](#messages), tóm tắt **cập nhật state một cách lâu dài** (persistent), thay thế vĩnh viễn các message cũ bằng một bản tóm tắt được lưu cho mọi lượt về sau.

LangChain cung cấp sẵn middleware cho việc này:

```python
from langchain.agents import create_agent
from langchain.agents.middleware import SummarizationMiddleware

agent = create_agent(
    model="gpt-5.5",
    tools=[...],
    middleware=[
        SummarizationMiddleware(
            model="gpt-5.4-mini",
            trigger={"tokens": 4000},
            keep=("messages", 20),
        ),
    ],
)
```

Khi hội thoại vượt quá giới hạn token, `SummarizationMiddleware` tự động:

1. Tóm tắt các message cũ hơn bằng một lệnh gọi LLM riêng biệt
2. Thay thế chúng bằng một message tóm tắt trong State (vĩnh viễn)
3. Giữ nguyên các message gần đây để duy trì context

Lịch sử hội thoại đã tóm tắt được cập nhật vĩnh viễn, các lượt tiếp theo sẽ thấy bản tóm tắt thay vì các message gốc.

!!! note "Ghi chú"
    Để xem danh sách đầy đủ middleware dựng sẵn, các hook khả dụng, và cách tạo middleware tuỳ chỉnh, xem [tài liệu Middleware](middleware/overview.md).

## Thực hành tốt nhất

1. **Bắt đầu đơn giản** – Bắt đầu với prompt và tool tĩnh, chỉ thêm phần động khi cần thiết
2. **Test từng bước** – Thêm từng tính năng context engineering một
3. **Theo dõi hiệu năng** – Theo dõi lệnh gọi model, mức dùng token, và độ trễ (latency)
4. **Dùng middleware dựng sẵn** – Tận dụng [`SummarizationMiddleware`](https://docs.langchain.com/oss/python/langchain/middleware#summarization), [`LLMToolSelectorMiddleware`](https://docs.langchain.com/oss/python/langchain/middleware#llm-tool-selector), v.v.
5. **Ghi lại chiến lược context của bạn** – Làm rõ context nào đang được truyền và vì sao
6. **Hiểu rõ tạm thời so với lâu dài**: Thay đổi model context là tạm thời (theo từng lệnh gọi), trong khi thay đổi life-cycle context được lưu bền vững vào state

## Tài nguyên liên quan

* [Tổng quan khái niệm Context](https://docs.langchain.com/oss/python/concepts/context) – Hiểu các loại context và khi nào nên dùng
* [Middleware](middleware/overview.md) – Hướng dẫn middleware đầy đủ
* [Tools](https://docs.langchain.com/oss/python/langchain/tools) – Tạo tool và truy cập context
* [Memory](https://docs.langchain.com/oss/python/concepts/memory) – Pattern short-term và long-term memory
* [Agents](agents.md) – Khái niệm cốt lõi về agent
