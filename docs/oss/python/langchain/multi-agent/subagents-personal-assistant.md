# Xây dựng trợ lý cá nhân với subagent

## Tổng quan

**Mẫu supervisor** là một kiến trúc [multi-agent](index.md) trong đó một agent supervisor trung tâm điều phối các agent worker chuyên biệt. Cách tiếp cận này phát huy hiệu quả khi tác vụ đòi hỏi nhiều loại chuyên môn khác nhau. Thay vì xây dựng một agent duy nhất phải quản lý việc lựa chọn tool trên nhiều lĩnh vực, bạn tạo ra các chuyên gia tập trung (focused specialist), được điều phối bởi một supervisor hiểu rõ toàn bộ quy trình làm việc.

Trong hướng dẫn này, bạn sẽ xây dựng một hệ thống trợ lý cá nhân minh họa những lợi ích đó thông qua một quy trình làm việc thực tế. Hệ thống sẽ điều phối hai chuyên gia với trách nhiệm khác biệt về cơ bản:

* Một **calendar agent** xử lý việc lên lịch, kiểm tra tình trạng rảnh và quản lý sự kiện.
* Một **email agent** quản lý giao tiếp, soạn thảo tin nhắn và gửi thông báo.

Chúng ta cũng sẽ tích hợp [human-in-the-loop review](../human-in-the-loop.md) để cho phép người dùng phê duyệt, chỉnh sửa hoặc từ chối các hành động (chẳng hạn như email gửi đi) theo ý muốn.

!!! note "Ghi chú"
    Nếu bạn đang di chuyển từ package [`langgraph-supervisor`](https://github.com/langchain-ai/langgraph-supervisor-py), xem [Migrate from langgraph-supervisor](https://docs.langchain.com/oss/python/migrate/langgraph-supervisor) để biết các mẫu trước và sau khi chuyển đổi, bao gồm cả luồng interrupt và resume.

### Tại sao nên dùng supervisor?

Kiến trúc multi-agent cho phép bạn phân chia [tool](../tools.md) giữa các worker, mỗi worker có prompt hoặc chỉ dẫn riêng. Hãy xem xét một agent có quyền truy cập trực tiếp vào tất cả API calendar và email: nó phải chọn trong số rất nhiều tool tương tự nhau, hiểu định dạng chính xác của từng API, và xử lý nhiều lĩnh vực cùng lúc. Nếu hiệu năng giảm sút, việc tách các tool liên quan và prompt tương ứng thành các nhóm logic riêng biệt (một phần để dễ quản lý các cải tiến lặp lại) có thể sẽ hữu ích.

### Khái niệm

Chúng ta sẽ tìm hiểu các khái niệm sau:

* [Hệ thống multi-agent](index.md)
* [Human-in-the-loop review](../human-in-the-loop.md)

## Cài đặt

### Cài đặt package

Hướng dẫn này yêu cầu package `langchain`:

=== "pip"

    ```bash
    pip install langchain
    ```

=== "conda"

    ```bash
    conda install langchain -c conda-forge
    ```

Để biết thêm chi tiết, xem [hướng dẫn cài đặt](../install.md) của chúng tôi.

### LangSmith

Thiết lập [LangSmith](https://smith.langchain.com?utm_source=docs&utm_medium=cta&utm_campaign=langsmith-signup&utm_content=oss-langchain-multi-agent-subagents-personal-assistant) để theo dõi những gì đang diễn ra bên trong agent của bạn. Sau đó thiết lập các biến môi trường sau:

=== "Shell"

    ```bash
    export LANGSMITH_TRACING="true"
    export LANGSMITH_API_KEY="..."
    ```

=== "Python"

    ```python
    import getpass
    import os

    os.environ["LANGSMITH_TRACING"] = "true"
    os.environ["LANGSMITH_API_KEY"] = getpass.getpass()
    ```

### Thành phần

Chúng ta cần chọn một chat model từ bộ tích hợp của LangChain:

=== "OpenAI"

    👉 Đọc [tài liệu tích hợp chat model OpenAI](https://docs.langchain.com/oss/python/integrations/chat/openai/)

    === "pip"

        ```bash
        pip install -U "langchain[openai]"
        ```

    === "uv"

        ```bash
        uv add "langchain[openai]"
        ```

    === "init_chat_model"

        ```python
        import os
        from langchain.chat_models import init_chat_model

        os.environ["OPENAI_API_KEY"] = "sk-..."

        model = init_chat_model("gpt-5.5")
        ```

    === "Model Class"

        ```python
        import os
        from langchain_openai import ChatOpenAI

        os.environ["OPENAI_API_KEY"] = "sk-..."

        model = ChatOpenAI(model="gpt-5.5")
        ```

=== "Anthropic"

    👉 Đọc [tài liệu tích hợp chat model Anthropic](https://docs.langchain.com/oss/python/integrations/chat/anthropic/)

    === "pip"

        ```bash
        pip install -U "langchain[anthropic]"
        ```

    === "uv"

        ```bash
        uv add "langchain[anthropic]"
        ```

    === "init_chat_model"

        ```python
        import os
        from langchain.chat_models import init_chat_model

        os.environ["ANTHROPIC_API_KEY"] = "sk-..."

        model = init_chat_model("claude-sonnet-4-6")
        ```

    === "Model Class"

        ```python
        import os
        from langchain_anthropic import ChatAnthropic

        os.environ["ANTHROPIC_API_KEY"] = "sk-..."

        model = ChatAnthropic(model="claude-sonnet-4-6")
        ```

=== "Azure"

    👉 Đọc [tài liệu tích hợp chat model Azure](https://docs.langchain.com/oss/python/integrations/chat/azure_chat_openai/)

    === "pip"

        ```bash
        pip install -U "langchain[openai]"
        ```

    === "uv"

        ```bash
        uv add "langchain[openai]"
        ```

    === "init_chat_model"

        ```python
        import os
        from langchain.chat_models import init_chat_model

        os.environ["AZURE_OPENAI_API_KEY"] = "..."
        os.environ["AZURE_OPENAI_ENDPOINT"] = "..."
        os.environ["OPENAI_API_VERSION"] = "2025-03-01-preview"

        model = init_chat_model(
            "azure_openai:gpt-5.5",
            azure_deployment=os.environ["AZURE_OPENAI_DEPLOYMENT_NAME"],
        )
        ```

    === "Model Class"

        ```python
        import os
        from langchain_openai import AzureChatOpenAI

        os.environ["AZURE_OPENAI_API_KEY"] = "..."
        os.environ["AZURE_OPENAI_ENDPOINT"] = "..."
        os.environ["OPENAI_API_VERSION"] = "2025-03-01-preview"

        model = AzureChatOpenAI(
            model="gpt-5.5",
            azure_deployment=os.environ["AZURE_OPENAI_DEPLOYMENT_NAME"]
        )
        ```

=== "Google Gemini"

    👉 Đọc [tài liệu tích hợp chat model Google GenAI](https://docs.langchain.com/oss/python/integrations/chat/google_generative_ai/)

    === "pip"

        ```bash
        pip install -U "langchain[google-genai]"
        ```

    === "uv"

        ```bash
        uv add "langchain[google-genai]"
        ```

    === "init_chat_model"

        ```python
        import os
        from langchain.chat_models import init_chat_model

        os.environ["GOOGLE_API_KEY"] = "..."

        model = init_chat_model("google_genai:gemini-2.5-flash-lite")
        ```

    === "Model Class"

        ```python
        import os
        from langchain_google_genai import ChatGoogleGenerativeAI

        os.environ["GOOGLE_API_KEY"] = "..."

        model = ChatGoogleGenerativeAI(model="gemini-2.5-flash-lite")
        ```

=== "AWS Bedrock"

    👉 Đọc [tài liệu tích hợp chat model AWS Bedrock](https://docs.langchain.com/oss/python/integrations/chat/bedrock/)

    === "pip"

        ```bash
        pip install -U "langchain[aws]"
        ```

    === "uv"

        ```bash
        uv add "langchain[aws]"
        ```

    === "init_chat_model"

        ```python
        from langchain.chat_models import init_chat_model

        # Follow the steps here to configure your credentials:
        # https://docs.aws.amazon.com/bedrock/latest/userguide/getting-started.html

        model = init_chat_model(
            "us.anthropic.claude-sonnet-4-6",
            model_provider="bedrock_converse",
        )
        ```

    === "Model Class"

        ```python
        from langchain_aws import ChatBedrock

        model = ChatBedrock(model="us.anthropic.claude-sonnet-4-6")
        ```

=== "HuggingFace"

    👉 Đọc [tài liệu tích hợp chat model HuggingFace](https://docs.langchain.com/oss/python/integrations/chat/huggingface/)

    === "pip"

        ```bash
        pip install -U "langchain[huggingface]"
        ```

    === "uv"

        ```bash
        uv add "langchain[huggingface]"
        ```

    === "init_chat_model"

        ```python
        import os
        from langchain.chat_models import init_chat_model

        os.environ["HUGGINGFACEHUB_API_TOKEN"] = "hf_..."

        model = init_chat_model(
            "microsoft/Phi-3-mini-4k-instruct",
            model_provider="huggingface",
            temperature=0.7,
            max_tokens=1024,
        )
        ```

    === "Model Class"

        ```python
        import os
        from langchain_huggingface import ChatHuggingFace, HuggingFaceEndpoint

        os.environ["HUGGINGFACEHUB_API_TOKEN"] = "hf_..."

        llm = HuggingFaceEndpoint(
            repo_id="microsoft/Phi-3-mini-4k-instruct",
            temperature=0.7,
            max_length=1024,
        )
        model = ChatHuggingFace(llm=llm)
        ```

=== "OpenRouter"

    👉 Đọc [tài liệu tích hợp chat model OpenRouter](https://docs.langchain.com/oss/python/integrations/chat/openrouter/)

    === "pip"

        ```bash
        pip install -U "langchain-openrouter"
        ```

    === "uv"

        ```bash
        uv add "langchain-openrouter"
        ```

    === "init_chat_model"

        ```python
        import os
        from langchain.chat_models import init_chat_model

        os.environ["OPENROUTER_API_KEY"] = "sk-..."

        model = init_chat_model(
            "auto",
            model_provider="openrouter",
        )
        ```

    === "Model Class"

        ```python
        import os
        from langchain_openrouter import ChatOpenRouter

        os.environ["OPENROUTER_API_KEY"] = "sk-..."

        model = ChatOpenRouter(model="auto")
        ```

## 1. Định nghĩa tool

Bắt đầu bằng việc định nghĩa các tool yêu cầu input có cấu trúc. Trong ứng dụng thực tế, các tool này sẽ gọi đến API thật (Google Calendar, SendGrid, v.v.). Trong hướng dẫn này, bạn sẽ dùng các stub (hàm giả lập) để minh họa mẫu thiết kế.

```python
from langchain.tools import tool

@tool
def create_calendar_event(
    title: str,
    start_time: str,       # ISO format: "2024-01-15T14:00:00"
    end_time: str,         # ISO format: "2024-01-15T15:00:00"
    attendees: list[str],  # email addresses
    location: str = ""
) -> str:
    """Create a calendar event. Requires exact ISO datetime format."""
    # Stub: In practice, this would call Google Calendar API, Outlook API, etc.
    return f"Event created: {title} from {start_time} to {end_time} with {len(attendees)} attendees"


@tool
def send_email(
    to: list[str],  # email addresses
    subject: str,
    body: str,
    cc: list[str] = []
) -> str:
    """Send an email via email API. Requires properly formatted addresses."""
    # Stub: In practice, this would call SendGrid, Gmail API, etc.
    return f"Email sent to {', '.join(to)} - Subject: {subject}"


@tool
def get_available_time_slots(
    attendees: list[str],
    date: str,  # ISO format: "2024-01-15"
    duration_minutes: int
) -> list[str]:
    """Check calendar availability for given attendees on a specific date."""
    # Stub: In practice, this would query calendar APIs
    return ["09:00", "14:00", "16:00"]
```

## 2. Tạo các sub-agent chuyên biệt

Tiếp theo, chúng ta sẽ tạo các sub-agent chuyên biệt để xử lý từng lĩnh vực.

### Tạo calendar agent

Calendar agent hiểu các yêu cầu lên lịch bằng ngôn ngữ tự nhiên và chuyển chúng thành các lệnh gọi API chính xác. Nó xử lý việc phân tích ngày tháng, kiểm tra tình trạng rảnh và tạo sự kiện.

```python
from datetime import date

from langchain.agents import create_agent


CALENDAR_AGENT_PROMPT = (
    f"Today's date is {date.today().isoformat()}. "
    "You are a calendar scheduling assistant. "
    "Parse natural language scheduling requests (e.g., 'next Tuesday at 2pm') "
    "into proper ISO datetime formats. "
    "Use get_available_time_slots to check availability when needed. "
    "If there is no suitable time slot, stop and confirm unavailability in your response. "
    "Use create_calendar_event to schedule events. "
    "Always confirm what was scheduled in your final response."
)

calendar_agent = create_agent(
    model,
    tools=[create_calendar_event, get_available_time_slots],
    system_prompt=CALENDAR_AGENT_PROMPT,
)
```

Kiểm tra calendar agent để xem cách nó xử lý yêu cầu lên lịch bằng ngôn ngữ tự nhiên:

```python
query = "Schedule a team meeting next Tuesday at 2pm for 1 hour"

stream = calendar_agent.stream_events(
    {"messages": [{"role": "user", "content": query}]},
    version="v3",
)
for kind, item in stream.interleave("messages", "tool_calls"):
    if kind == "messages":
        for token in item.text:
            print(token, end="", flush=True)
    elif kind == "tool_calls":
        print(f"\nTool call: {item.tool_name}({item.input})")
        print(f"Tool result: {item.output}")
```

```
================================== Ai Message ==================================
Tool Calls:
  get_available_time_slots (call_EIeoeIi1hE2VmwZSfHStGmXp)
 Call ID: call_EIeoeIi1hE2VmwZSfHStGmXp
  Args:
    attendees: []
    date: 2024-06-18
    duration_minutes: 60
================================= Tool Message =================================
Name: get_available_time_slots

["09:00", "14:00", "16:00"]
================================== Ai Message ==================================
Tool Calls:
  create_calendar_event (call_zgx3iJA66Ut0W8S3NpT93kEB)
 Call ID: call_zgx3iJA66Ut0W8S3NpT93kEB
  Args:
    title: Team Meeting
    start_time: 2024-06-18T14:00:00
    end_time: 2024-06-18T15:00:00
    attendees: []
================================= Tool Message =================================
Name: create_calendar_event

Event created: Team Meeting from 2024-06-18T14:00:00 to 2024-06-18T15:00:00 with 0 attendees
================================== Ai Message ==================================

The team meeting has been scheduled for next Tuesday, June 18th, at 2:00 PM and will last for 1 hour. If you need to add attendees or a location, please let me know!
```

Agent phân tích cụm "next Tuesday at 2pm" thành định dạng ISO ("2024-01-16T14:00:00"), tính toán thời gian kết thúc, gọi `create_calendar_event`, và trả về xác nhận bằng ngôn ngữ tự nhiên.

### Tạo email agent

Email agent xử lý việc soạn thảo và gửi tin nhắn. Nó tập trung vào việc trích xuất thông tin người nhận, soạn tiêu đề và nội dung email phù hợp, và quản lý giao tiếp qua email.

```python
EMAIL_AGENT_PROMPT = (
    "You are an email assistant. "
    "Compose professional emails based on natural language requests. "
    "Extract recipient information and craft appropriate subject lines and body text. "
    "Use send_email to send the message. "
    "Always confirm what was sent in your final response."
)

email_agent = create_agent(
    model,
    tools=[send_email],
    system_prompt=EMAIL_AGENT_PROMPT,
)
```

Kiểm tra email agent với một yêu cầu bằng ngôn ngữ tự nhiên:

```python
query = "Send the design team a reminder about reviewing the new mockups"

stream = email_agent.stream_events(
    {"messages": [{"role": "user", "content": query}]},
    version="v3",
)
for kind, item in stream.interleave("messages", "tool_calls"):
    if kind == "messages":
        for token in item.text:
            print(token, end="", flush=True)
    elif kind == "tool_calls":
        print(f"\nTool call: {item.tool_name}({item.input})")
        print(f"Tool result: {item.output}")
```

```
================================== Ai Message ==================================
Tool Calls:
  send_email (call_OMl51FziTVY6CRZvzYfjYOZr)
 Call ID: call_OMl51FziTVY6CRZvzYfjYOZr
  Args:
    to: ['design-team@example.com']
    subject: Reminder: Please Review the New Mockups
    body: Hi Design Team,

This is a friendly reminder to review the new mockups at your earliest convenience. Your feedback is important to ensure that we stay on track with our project timeline.

Please let me know if you have any questions or need additional information.

Thank you!

Best regards,
================================= Tool Message =================================
Name: send_email

Email sent to design-team@example.com - Subject: Reminder: Please Review the New Mockups
================================== Ai Message ==================================

I've sent a reminder to the design team asking them to review the new mockups. If you need any further communication on this topic, just let me know!
```

Agent suy luận ra người nhận từ yêu cầu không chính thức, soạn tiêu đề và nội dung email chuyên nghiệp, gọi `send_email`, và trả về xác nhận. Mỗi sub-agent có phạm vi tập trung hẹp với tool và prompt đặc thù cho lĩnh vực của mình, giúp nó thực hiện tốt tác vụ cụ thể của mình.

## 3. Bọc sub-agent thành tool

Bây giờ hãy bọc mỗi sub-agent thành một tool mà supervisor có thể gọi. Đây là bước kiến trúc then chốt tạo nên hệ thống phân tầng. Supervisor sẽ chỉ nhìn thấy các tool cấp cao như "schedule_event", chứ không phải các tool cấp thấp như "create_calendar_event".

```python
@tool
def schedule_event(request: str) -> str:
    """Schedule calendar events using natural language.

    Use this when the user wants to create, modify, or check calendar appointments.
    Handles date/time parsing, availability checking, and event creation.

    Input: Natural language scheduling request (e.g., 'meeting with design team
    next Tuesday at 2pm')
    """
    result = calendar_agent.invoke({
        "messages": [{"role": "user", "content": request}]
    })
    return result["messages"][-1].text


@tool
def manage_email(request: str) -> str:
    """Send emails using natural language.

    Use this when the user wants to send notifications, reminders, or any email
    communication. Handles recipient extraction, subject generation, and email
    composition.

    Input: Natural language email request (e.g., 'send them a reminder about
    the meeting')
    """
    result = email_agent.invoke({
        "messages": [{"role": "user", "content": request}]
    })
    return result["messages"][-1].text
```

Mô tả tool giúp supervisor quyết định khi nào nên dùng từng tool, vì vậy hãy viết mô tả rõ ràng và cụ thể. Chúng ta chỉ trả về phản hồi cuối cùng của sub-agent, vì supervisor không cần thấy quá trình suy luận trung gian hay các tool call.

## 4. Tạo supervisor agent

Bây giờ hãy tạo supervisor điều phối các sub-agent. Supervisor chỉ nhìn thấy các tool cấp cao và đưa ra quyết định định tuyến ở cấp độ lĩnh vực, không phải ở cấp độ API riêng lẻ.

```python
SUPERVISOR_PROMPT = (
    "You are a helpful personal assistant. "
    "You can schedule calendar events and send emails. "
    "Break down user requests into appropriate tool calls and coordinate the results. "
    "When a request involves multiple actions, use multiple tools in sequence or in parallel as appropriate."
)

supervisor_agent = create_agent(
    model,
    tools=[schedule_event, manage_email],
    system_prompt=SUPERVISOR_PROMPT,
)
```

## 5. Sử dụng supervisor

Bây giờ hãy kiểm tra hệ thống hoàn chỉnh của bạn với các yêu cầu phức tạp đòi hỏi phối hợp giữa nhiều lĩnh vực:

### Ví dụ 1: Yêu cầu đơn giản trong một lĩnh vực

```python
query = "Schedule a team standup for tomorrow at 9am"

stream = supervisor_agent.stream_events(
    {"messages": [{"role": "user", "content": query}]},
    version="v3",
)
for kind, item in stream.interleave("messages", "tool_calls"):
    if kind == "messages":
        for token in item.text:
            print(token, end="", flush=True)
    elif kind == "tool_calls":
        print(f"\nTool call: {item.tool_name}({item.input})")
        print(f"Tool result: {item.output}")
```

```
================================== Ai Message ==================================
Tool Calls:
  schedule_event (call_mXFJJDU8bKZadNUZPaag8Lct)
 Call ID: call_mXFJJDU8bKZadNUZPaag8Lct
  Args:
    request: Schedule a team standup for tomorrow at 9am with Alice and Bob.
================================= Tool Message =================================
Name: schedule_event

The team standup has been scheduled for tomorrow at 9:00 AM with Alice and Bob. If you need to make any changes or add more details, just let me know!
================================== Ai Message ==================================

The team standup with Alice and Bob is scheduled for tomorrow at 9:00 AM. If you need any further arrangements or adjustments, please let me know!
```

Supervisor xác định đây là một tác vụ calendar, gọi `schedule_event`, và calendar agent xử lý việc phân tích ngày tháng và tạo sự kiện.

!!! tip "Mẹo"
    Để xem toàn bộ luồng thông tin, bao gồm prompt và phản hồi cho từng lệnh gọi chat model, xem [LangSmith trace](https://smith.langchain.com/public/91a9a95f-fba9-4e84-aff0-371861ad2f4a/r) cho lần chạy ở trên.

### Ví dụ 2: Yêu cầu phức tạp liên quan nhiều lĩnh vực

```python
query = (
    "Schedule a meeting with the design team next Tuesday at 2pm for 1 hour, "
    "and send them an email reminder about reviewing the new mockups."
)

stream = supervisor_agent.stream_events(
    {"messages": [{"role": "user", "content": query}]},
    version="v3",
)
for kind, item in stream.interleave("messages", "tool_calls"):
    if kind == "messages":
        for token in item.text:
            print(token, end="", flush=True)
    elif kind == "tool_calls":
        print(f"\nTool call: {item.tool_name}({item.input})")
        print(f"Tool result: {item.output}")
```

```
================================== Ai Message ==================================
Tool Calls:
  schedule_event (call_YA68mqF0koZItCFPx0kGQfZi)
 Call ID: call_YA68mqF0koZItCFPx0kGQfZi
  Args:
    request: meeting with the design team next Tuesday at 2pm for 1 hour
  manage_email (call_XxqcJBvVIuKuRK794ZIzlLxx)
 Call ID: call_XxqcJBvVIuKuRK794ZIzlLxx
  Args:
    request: send the design team an email reminder about reviewing the new mockups
================================= Tool Message =================================
Name: schedule_event

Your meeting with the design team is scheduled for next Tuesday, June 18th, from 2:00pm to 3:00pm. Let me know if you need to add more details or make any changes!
================================= Tool Message =================================
Name: manage_email

I've sent an email reminder to the design team requesting them to review the new mockups. If you need to include more information or recipients, just let me know!
================================== Ai Message ==================================

Your meeting with the design team is scheduled for next Tuesday, June 18th, from 2:00pm to 3:00pm.

I've also sent an email reminder to the design team, asking them to review the new mockups.

Let me know if you'd like to add more details to the meeting or include additional information in the email!
```

Supervisor nhận ra yêu cầu này cần cả hành động calendar và email, gọi `schedule_event` cho cuộc họp, sau đó gọi `manage_email` cho lời nhắc. Mỗi sub-agent hoàn thành tác vụ của mình, và supervisor tổng hợp cả hai kết quả thành một phản hồi mạch lạc.

!!! note "Ghi chú"
    Theo mặc định, supervisor gửi tác vụ đến các subagent một cách tuần tự. Mỗi tool call hoàn thành trước khi tool call tiếp theo bắt đầu. Tuy nhiên, nhiều LLM sẽ đưa ra nhiều tool call trong cùng một phản hồi (như minh họa trong trace ở trên, khi cả `schedule_event` và `manage_email` được gọi cùng lúc), và runtime sẽ thực thi chúng song song. Bạn cũng có thể cấu hình việc gửi song song một cách tường minh. Xem [tài liệu tham khảo `create_supervisor`](https://reference.langchain.com/python/langgraph-supervisor/supervisor/create_supervisor) để biết chi tiết.

!!! tip "Mẹo"
    Tham khảo [LangSmith trace](https://smith.langchain.com/public/95cd00a3-d1f9-4dba-9731-7bf733fb6a3c/r) để xem chi tiết luồng thông tin cho lần chạy ở trên, bao gồm từng prompt và phản hồi của chat model.

### Ví dụ hoàn chỉnh có thể chạy được

Dưới đây là toàn bộ các phần được kết hợp trong một script có thể chạy được:

```python
"""
Personal Assistant Supervisor Example

This example demonstrates the tool calling pattern for multi-agent systems.
A supervisor agent coordinates specialized sub-agents (calendar and email)
that are wrapped as tools.
"""

from datetime import date

from langchain.tools import tool
from langchain.agents import create_agent
from langchain.chat_models import init_chat_model

# ============================================================================
# Step 1: Define low-level API tools (stubbed)
# ============================================================================

@tool
def create_calendar_event(
    title: str,
    start_time: str,  # ISO format: "2024-01-15T14:00:00"
    end_time: str,    # ISO format: "2024-01-15T15:00:00"
    attendees: list[str],  # email addresses
    location: str = ""
) -> str:
    """Create a calendar event. Requires exact ISO datetime format."""
    return f"Event created: {title} from {start_time} to {end_time} with {len(attendees)} attendees"


@tool
def send_email(
    to: list[str],      # email addresses
    subject: str,
    body: str,
    cc: list[str] = []
) -> str:
    """Send an email via email API. Requires properly formatted addresses."""
    return f"Email sent to {', '.join(to)} - Subject: {subject}"


@tool
def get_available_time_slots(
    attendees: list[str],
    date: str,  # ISO format: "2024-01-15"
    duration_minutes: int
) -> list[str]:
    """Check calendar availability for given attendees on a specific date."""
    return ["09:00", "14:00", "16:00"]


# ============================================================================
# Step 2: Create specialized sub-agents
# ============================================================================

model = init_chat_model("gpt-5.5")  # for example

calendar_agent = create_agent(
    model,
    tools=[create_calendar_event, get_available_time_slots],
    system_prompt=(
        f"Today's date is {date.today().isoformat()}. "
        "You are a calendar scheduling assistant. "
        "Parse natural language scheduling requests (e.g., 'next Tuesday at 2pm') "
        "into proper ISO datetime formats. "
        "Use get_available_time_slots to check availability when needed. "
        "If there is no suitable time slot, stop and confirm unavailability in your response. "
        "Use create_calendar_event to schedule events. "
        "Always confirm what was scheduled in your final response."
    )
)

email_agent = create_agent(
    model,
    tools=[send_email],
    system_prompt=(
        "You are an email assistant. "
        "Compose professional emails based on natural language requests. "
        "Extract recipient information and craft appropriate subject lines and body text. "
        "Use send_email to send the message. "
        "Always confirm what was sent in your final response."
    )
)

# ============================================================================
# Step 3: Wrap sub-agents as tools for the supervisor
# ============================================================================

@tool
def schedule_event(request: str) -> str:
    """Schedule calendar events using natural language.

    Use this when the user wants to create, modify, or check calendar appointments.
    Handles date/time parsing, availability checking, and event creation.

    Input: Natural language scheduling request (e.g., 'meeting with design team
    next Tuesday at 2pm')
    """
    result = calendar_agent.invoke({
        "messages": [{"role": "user", "content": request}]
    })
    return result["messages"][-1].text


@tool
def manage_email(request: str) -> str:
    """Send emails using natural language.

    Use this when the user wants to send notifications, reminders, or any email
    communication. Handles recipient extraction, subject generation, and email
    composition.

    Input: Natural language email request (e.g., 'send them a reminder about
    the meeting')
    """
    result = email_agent.invoke({
        "messages": [{"role": "user", "content": request}]
    })
    return result["messages"][-1].text


# ============================================================================
# Step 4: Create the supervisor agent
# ============================================================================

supervisor_agent = create_agent(
    model,
    tools=[schedule_event, manage_email],
    system_prompt=(
        "You are a helpful personal assistant. "
        "You can schedule calendar events and send emails. "
        "Break down user requests into appropriate tool calls and coordinate the results. "
        "When a request involves multiple actions, use multiple tools in sequence or in parallel as appropriate."
    )
)

# ============================================================================
# Step 5: Use the supervisor
# ============================================================================

if __name__ == "__main__":
    # Example: User request requiring both calendar and email coordination
    user_request = (
        "Schedule a meeting with the design team next Tuesday at 2pm for 1 hour, "
        "and send them an email reminder about reviewing the new mockups."
    )

    print("User Request:", user_request)
    print("\n" + "="*80 + "\n")

    stream = supervisor_agent.stream_events(
        {"messages": [{"role": "user", "content": user_request}]},
        version="v3",
    )
    for kind, item in stream.interleave("messages", "tool_calls"):
        if kind == "messages":
            for token in item.text:
                print(token, end="", flush=True)
        elif kind == "tool_calls":
            print(f"\nTool call: {item.tool_name}({item.input})")
            print(f"Tool result: {item.output}")
```

### Hiểu về kiến trúc

Hệ thống của bạn có ba tầng. Tầng dưới cùng chứa các tool API cứng nhắc, yêu cầu định dạng chính xác. Tầng giữa chứa các sub-agent tiếp nhận ngôn ngữ tự nhiên, chuyển đổi thành lệnh gọi API có cấu trúc, và trả về xác nhận bằng ngôn ngữ tự nhiên. Tầng trên cùng chứa supervisor, định tuyến đến các năng lực cấp cao và tổng hợp kết quả.

Sự phân tách trách nhiệm này mang lại một số lợi ích: mỗi tầng có trách nhiệm tập trung riêng, bạn có thể thêm lĩnh vực mới mà không ảnh hưởng đến các lĩnh vực hiện có, và bạn có thể kiểm thử, cải tiến từng tầng một cách độc lập.

## 6. Thêm human-in-the-loop review

Việc tích hợp [human-in-the-loop review](../human-in-the-loop.md) cho các hành động nhạy cảm là điều nên làm. LangChain có sẵn [middleware tích hợp sẵn](../human-in-the-loop.md#configuring-interrupts) để review các tool call, trong trường hợp này là các tool được sub-agent gọi.

Hãy thêm human-in-the-loop review cho cả hai sub-agent:

* Chúng ta cấu hình các tool `create_calendar_event` và `send_email` để dừng lại (interrupt), cho phép tất cả các [loại phản hồi](../human-in-the-loop.md) (`approve`, `edit`, `reject`)
* Chúng ta thêm một [checkpointer](../short-term-memory.md) **chỉ cho agent cấp cao nhất**. Điều này là bắt buộc để có thể tạm dừng và tiếp tục thực thi.

```python
from langchain.agents import create_agent
from langchain.agents.middleware import HumanInTheLoopMiddleware # [!code highlight]
from langgraph.checkpoint.memory import InMemorySaver # [!code highlight]


calendar_agent = create_agent(
    model,
    tools=[create_calendar_event, get_available_time_slots],
    system_prompt=CALENDAR_AGENT_PROMPT,
    middleware=[ # [!code highlight]
        HumanInTheLoopMiddleware( # [!code highlight]
            interrupt_on={"create_calendar_event": True}, # [!code highlight]
            description_prefix="Calendar event pending approval", # [!code highlight]
        ), # [!code highlight]
    ], # [!code highlight]
)

email_agent = create_agent(
    model,
    tools=[send_email],
    system_prompt=EMAIL_AGENT_PROMPT,
    middleware=[ # [!code highlight]
        HumanInTheLoopMiddleware( # [!code highlight]
            interrupt_on={"send_email": True}, # [!code highlight]
            description_prefix="Outbound email pending approval", # [!code highlight]
        ), # [!code highlight]
    ], # [!code highlight]
)

supervisor_agent = create_agent(
    model,
    tools=[schedule_event, manage_email],
    system_prompt=SUPERVISOR_PROMPT,
    checkpointer=InMemorySaver(), # [!code highlight]
)
```

Hãy lặp lại truy vấn. Lưu ý rằng chúng ta thu thập các sự kiện interrupt vào một danh sách để truy cập sau đó:

```python
query = (
    "Schedule a meeting with the design team next Tuesday at 2pm for 1 hour, "
    "and send them an email reminder about reviewing the new mockups."
)

config = {"configurable": {"thread_id": "6"}}

interrupts = []
stream = supervisor_agent.stream_events(
    {"messages": [{"role": "user", "content": query}]},
    config,
    version="v3",
)
for kind, item in stream.interleave("messages", "tool_calls"):
    if kind == "messages":
        for token in item.text:
            print(token, end="", flush=True)
    elif kind == "tool_calls":
        print(f"\nTool call: {item.tool_name}({item.input})")
if stream.interrupted:
    for interrupt_ in stream.interrupts:
        interrupts.append(interrupt_)
        print(f"\nINTERRUPTED: {interrupt_.id}")
```

```
================================== Ai Message ==================================
Tool Calls:
  schedule_event (call_t4Wyn32ohaShpEZKuzZbl83z)
 Call ID: call_t4Wyn32ohaShpEZKuzZbl83z
  Args:
    request: Schedule a meeting with the design team next Tuesday at 2pm for 1 hour.
  manage_email (call_JWj4vDJ5VMnvkySymhCBm4IR)
 Call ID: call_JWj4vDJ5VMnvkySymhCBm4IR
  Args:
    request: Send an email reminder to the design team about reviewing the new mockups before our meeting next Tuesday at 2pm.

INTERRUPTED: 4f994c9721682a292af303ec1a46abb7

INTERRUPTED: 2b56f299be313ad8bc689eff02973f16
```

Lần này việc thực thi đã bị dừng lại (interrupt). Hãy kiểm tra các sự kiện interrupt:

```python
for interrupt_ in interrupts:
    for request in interrupt_.value["action_requests"]:
        print(f"INTERRUPTED: {interrupt_.id}")
        print(f"{request['description']}\n")
```

```
INTERRUPTED: 4f994c9721682a292af303ec1a46abb7
Calendar event pending approval

Tool: create_calendar_event
Args: {'title': 'Meeting with the Design Team', 'start_time': '2024-06-18T14:00:00', 'end_time': '2024-06-18T15:00:00', 'attendees': ['design team']}

INTERRUPTED: 2b56f299be313ad8bc689eff02973f16
Outbound email pending approval

Tool: send_email
Args: {'to': ['designteam@example.com'], 'subject': 'Reminder: Review New Mockups Before Meeting Next Tuesday at 2pm', 'body': "Hello Team,\n\nThis is a reminder to review the new mockups ahead of our meeting scheduled for next Tuesday at 2pm. Your feedback and insights will be valuable for our discussion and next steps.\n\nPlease ensure you've gone through the designs and are ready to share your thoughts during the meeting.\n\nThank you!\n\nBest regards,\n[Your Name]"}
```

Chúng ta có thể chỉ định quyết định cho từng interrupt bằng cách tham chiếu đến ID của nó thông qua [`Command`](https://reference.langchain.com/python/langgraph/types/Command). Tham khảo [hướng dẫn human-in-the-loop](../human-in-the-loop.md) để biết thêm chi tiết. Để minh họa, ở đây chúng ta sẽ chấp nhận sự kiện calendar, nhưng chỉnh sửa tiêu đề của email gửi đi:

```python
from langgraph.types import Command # [!code highlight]

resume = {}
for interrupt_ in interrupts:
    if interrupt_.id == "2b56f299be313ad8bc689eff02973f16":
        # Edit email
        edited_action = interrupt_.value["action_requests"][0].copy()
        edited_action["args"]["subject"] = "Mockups reminder"
        resume[interrupt_.id] = {
            "decisions": [{"type": "edit", "edited_action": edited_action}]
        }
    else:
        resume[interrupt_.id] = {"decisions": [{"type": "approve"}]}

interrupts = []
stream = supervisor_agent.stream_events(
    Command(resume=resume), # [!code highlight]
    config,
    version="v3",
)
for kind, item in stream.interleave("messages", "tool_calls"):
    if kind == "messages":
        for token in item.text:
            print(token, end="", flush=True)
    elif kind == "tool_calls":
        print(f"\nTool call: {item.tool_name}({item.input})")
if stream.interrupted:
    for interrupt_ in stream.interrupts:
        interrupts.append(interrupt_)
        print(f"\nINTERRUPTED: {interrupt_.id}")
```

```
================================= Tool Message =================================
Name: schedule_event

Your meeting with the design team has been scheduled for next Tuesday, June 18th, from 2:00 pm to 3:00 pm.
================================= Tool Message =================================
Name: manage_email

Your email reminder to the design team has been sent. Here's what was sent:

- Recipient: designteam@example.com
- Subject: Mockups reminder
- Body: A reminder to review the new mockups before the meeting next Tuesday at 2pm, with a request for feedback and readiness for discussion.

Let me know if you need any further assistance!
================================== Ai Message ==================================

- Your meeting with the design team has been scheduled for next Tuesday, June 18th, from 2:00 pm to 3:00 pm.
- An email reminder has been sent to the design team about reviewing the new mockups before the meeting.

Let me know if you need any further assistance!
```

Quá trình chạy tiếp tục với input của chúng ta.

## 7. Nâng cao: Kiểm soát luồng thông tin

Theo mặc định, sub-agent chỉ nhận chuỗi request từ supervisor. Bạn có thể muốn truyền thêm context, chẳng hạn như lịch sử hội thoại hoặc tùy chọn của người dùng.

### Truyền thêm context hội thoại cho sub-agent

```python
from langchain.tools import tool, ToolRuntime

@tool
def schedule_event(
    request: str,
    runtime: ToolRuntime
) -> str:
    """Schedule calendar events using natural language."""
    # Customize context received by sub-agent
    original_user_message = next(
        message for message in runtime.state["messages"]
        if message.type == "human"
    )
    prompt = (
        "You are assisting with the following user inquiry:\n\n"
        f"{original_user_message.text}\n\n"
        "You are tasked with the following sub-request:\n\n"
        f"{request}"
    )
    result = calendar_agent.invoke({
        "messages": [{"role": "user", "content": prompt}],
    })
    return result["messages"][-1].text
```

Điều này cho phép sub-agent thấy được toàn bộ context hội thoại, có thể hữu ích để giải quyết các trường hợp mơ hồ như "đặt lịch vào cùng giờ này ngày mai" (tham chiếu đến một cuộc hội thoại trước đó).

!!! tip "Mẹo"
    Bạn có thể xem toàn bộ context mà sub agent nhận được trong [lệnh gọi chat model](https://smith.langchain.com/public/c7d54882-afb8-4039-9c5a-4112d0f458b0/r/6803571e-af78-4c68-904a-ecf55771084d) của LangSmith trace.

### Kiểm soát thông tin supervisor nhận được

Bạn cũng có thể tùy chỉnh thông tin nào được trả về cho supervisor:

```python
import json

@tool
def schedule_event(request: str) -> str:
    """Schedule calendar events using natural language."""
    result = calendar_agent.invoke({
        "messages": [{"role": "user", "content": request}]
    })

    # Option 1: Return just the confirmation message
    return result["messages"][-1].text

    # Option 2: Return structured data
    # return json.dumps({
    #     "status": "success",
    #     "event_id": "evt_123",
    #     "summary": result["messages"][-1].text
    # })
```

**Quan trọng:** Đảm bảo prompt của sub-agent nhấn mạnh rằng tin nhắn cuối cùng của chúng phải chứa đầy đủ thông tin liên quan. Một lỗi thường gặp là sub-agent thực hiện tool call nhưng không đưa kết quả vào phản hồi cuối cùng của mình.

## 8. Những điểm chính cần ghi nhớ

Mẫu supervisor tạo ra các tầng trừu tượng, trong đó mỗi tầng có trách nhiệm rõ ràng. Khi thiết kế một hệ thống supervisor, hãy bắt đầu với ranh giới lĩnh vực rõ ràng và trang bị cho mỗi sub-agent các tool và prompt tập trung. Viết mô tả tool rõ ràng cho supervisor, kiểm thử từng tầng một cách độc lập trước khi tích hợp, và kiểm soát luồng thông tin theo nhu cầu cụ thể của bạn.

!!! tip "Mẹo"
    **Khi nào nên dùng mẫu supervisor**

    Sử dụng mẫu supervisor khi bạn có nhiều lĩnh vực riêng biệt (calendar, email, CRM, cơ sở dữ liệu), mỗi lĩnh vực có nhiều tool hoặc logic phức tạp, bạn muốn kiểm soát quy trình làm việc tập trung, và các sub-agent không cần trò chuyện trực tiếp với người dùng.

    Đối với các trường hợp đơn giản hơn chỉ với vài tool, hãy dùng một agent duy nhất. Khi agent cần trò chuyện với người dùng, hãy dùng [handoffs](handoffs.md) thay thế. Đối với sự cộng tác ngang hàng (peer-to-peer) giữa các agent, hãy cân nhắc các mẫu multi-agent khác.

## Bước tiếp theo

Tìm hiểu về [handoffs](handoffs.md) cho các cuộc hội thoại giữa agent với agent, khám phá [context engineering](../context-engineering.md) để tinh chỉnh luồng thông tin, đọc [tổng quan multi-agent](index.md) để so sánh các mẫu khác nhau, và sử dụng [LangSmith](https://smith.langchain.com?utm_source=docs&utm_medium=cta&utm_campaign=langsmith-signup&utm_content=oss-langchain-multi-agent-subagents-personal-assistant) để debug và giám sát hệ thống multi-agent của bạn.
