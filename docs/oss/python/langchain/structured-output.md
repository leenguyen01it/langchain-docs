# Structured output (Đầu ra có cấu trúc)

Structured output cho phép agent trả về dữ liệu theo một định dạng cụ thể, có thể dự đoán trước. Thay vì phải parse phản hồi ngôn ngữ tự nhiên, bạn nhận được dữ liệu có cấu trúc dưới dạng đối tượng JSON, [Pydantic model](https://docs.pydantic.dev/latest/concepts/models/#basic-model-usage), hoặc dataclass mà ứng dụng của bạn có thể dùng trực tiếp.

!!! tip "Mẹo"
    Trang này nói về structured output với agent dùng `create_agent`. Để dùng structured output trực tiếp trên model (bên ngoài agent), xem [Models - Structured output](models.md#structured-output).

[`create_agent`](https://reference.langchain.com/python/langchain/agents/factory/create_agent) của LangChain xử lý structured output tự động. Người dùng thiết lập schema đầu ra có cấu trúc mong muốn, và khi model sinh ra dữ liệu có cấu trúc, nó được capture, validate, và trả về trong key `'structured_response'` của state agent.

```python
def create_agent(
    ...
    response_format: Union[
        ToolStrategy[StructuredResponseT],
        ProviderStrategy[StructuredResponseT],
        type[StructuredResponseT],
        None,
    ]
)
```

## Response format

Dùng `response_format` để kiểm soát cách agent trả về dữ liệu có cấu trúc:

* **`ToolStrategy[StructuredResponseT]`**: dùng tool calling cho structured output
* **`ProviderStrategy[StructuredResponseT]`**: dùng structured output native của provider
* **`type[StructuredResponseT]`**: kiểu schema, tự động chọn strategy tốt nhất dựa trên khả năng của model
* **`None`**: không yêu cầu structured output rõ ràng

Khi một kiểu schema được truyền trực tiếp, LangChain tự động chọn:

* `ProviderStrategy` nếu model và provider được chọn hỗ trợ structured output native (ví dụ [OpenAI](https://docs.langchain.com/oss/python/integrations/providers/openai), [Anthropic (Claude)](https://docs.langchain.com/oss/python/integrations/providers/anthropic), hoặc [xAI (Grok)](https://docs.langchain.com/oss/python/integrations/providers/xai)).
* `ToolStrategy` cho tất cả model khác.

!!! warning "Cảnh báo"
    Dictionary JSON Schema phải được bọc trong một strategy tường minh (`ProviderStrategy` hoặc `ToolStrategy`). Chúng không được tự động phát hiện khi truyền trực tiếp vào `response_format`.

!!! note "Ghi chú"
    Việc hỗ trợ tính năng structured output native được đọc động từ [profile data](models.md#model-profiles) của model nếu dùng `langchain>=1.1`. Nếu dữ liệu không có sẵn, dùng điều kiện khác hoặc chỉ định thủ công:

    ```python
    custom_profile = {
        "structured_output": True,
        # ...
    }
    model = init_chat_model("...", profile=custom_profile)
    ```

    Nếu tool được chỉ định, model phải hỗ trợ dùng đồng thời tool và structured output.

Phản hồi có cấu trúc được trả về trong key `structured_response` của state cuối cùng của agent.

## Provider strategy

Một số nhà cung cấp model hỗ trợ structured output native thông qua API của họ (ví dụ OpenAI, xAI (Grok), Gemini, Anthropic (Claude)). Đây là phương pháp đáng tin cậy nhất khi khả dụng.

Để dùng strategy này, cấu hình một `ProviderStrategy`:

```python
class ProviderStrategy(Generic[SchemaT]):
    schema: type[SchemaT]
    strict: bool | None = None
```

!!! info "Thông tin"
    Tham số `strict` yêu cầu `langchain>=1.2`.

**`schema`** (bắt buộc): schema định nghĩa định dạng structured output. Hỗ trợ:

* **Pydantic model**: lớp con của `BaseModel` có field validation. Trả về instance Pydantic đã validate.
* **Dataclass**: Python dataclass có type annotation. Trả về dict.
* **TypedDict**: lớp typed dictionary. Trả về dict.
* **JSON Schema**: dictionary theo đặc tả JSON schema. Phải có key `title` và `description` ở top-level. Trả về dict.

**`strict`** (tuỳ chọn): tham số boolean để bật chế độ tuân thủ schema nghiêm ngặt. Được hỗ trợ bởi một số provider (ví dụ [OpenAI](https://docs.langchain.com/oss/python/integrations/chat/openai) và [xAI](https://docs.langchain.com/oss/python/integrations/chat/xai)). Mặc định là `None` (tắt).

LangChain tự động dùng `ProviderStrategy` khi bạn truyền một kiểu schema trực tiếp vào [`create_agent.response_format`](https://reference.langchain.com/python/langchain/agents/factory/create_agent) và model hỗ trợ structured output native:

=== "Pydantic Model"

    ```python
    from pydantic import BaseModel, Field
    from langchain.agents import create_agent


    class ContactInfo(BaseModel):
        """Contact information for a person."""
        name: str = Field(description="The name of the person")
        email: str = Field(description="The email address of the person")
        phone: str = Field(description="The phone number of the person")

    agent = create_agent(
        model="gpt-5.5",
        response_format=ContactInfo  # Tự động chọn ProviderStrategy
    )

    result = agent.invoke({
        "messages": [{"role": "user", "content": "Extract contact info from: John Doe, john@example.com, (555) 123-4567"}]
    })

    print(result["structured_response"])
    # ContactInfo(name='John Doe', email='john@example.com', phone='(555) 123-4567')
    ```

=== "Dataclass"

    ```python
    from dataclasses import dataclass
    from langchain.agents import create_agent


    @dataclass
    class ContactInfo:
        """Contact information for a person."""
        name: str # The name of the person
        email: str # The email address of the person
        phone: str # The phone number of the person

    agent = create_agent(
        model="gpt-5.5",
        tools=tools,
        response_format=ContactInfo  # Tự động chọn ProviderStrategy
    )

    result = agent.invoke({
        "messages": [{"role": "user", "content": "Extract contact info from: John Doe, john@example.com, (555) 123-4567"}]
    })

    result["structured_response"]
    # {'name': 'John Doe', 'email': 'john@example.com', 'phone': '(555) 123-4567'}
    ```

=== "TypedDict"

    ```python
    from typing_extensions import TypedDict
    from langchain.agents import create_agent


    class ContactInfo(TypedDict):
        """Contact information for a person."""
        name: str # The name of the person
        email: str # The email address of the person
        phone: str # The phone number of the person

    agent = create_agent(
        model="gpt-5.5",
        tools=tools,
        response_format=ContactInfo  # Tự động chọn ProviderStrategy
    )

    result = agent.invoke({
        "messages": [{"role": "user", "content": "Extract contact info from: John Doe, john@example.com, (555) 123-4567"}]
    })

    result["structured_response"]
    # {'name': 'John Doe', 'email': 'john@example.com', 'phone': '(555) 123-4567'}
    ```

=== "JSON Schema"

    ```python
    from langchain.agents import create_agent
    from langchain.agents.structured_output import ProviderStrategy


    contact_info_schema = {
        "title": "ContactInfo",
        "type": "object",
        "description": "Contact information for a person.",
        "properties": {
            "name": {"type": "string", "description": "The name of the person"},
            "email": {"type": "string", "description": "The email address of the person"},
            "phone": {"type": "string", "description": "The phone number of the person"}
        },
        "required": ["name", "email", "phone"]
    }

    agent = create_agent(
        model="gpt-5.5",
        tools=tools,
        response_format=ProviderStrategy(contact_info_schema)
    )

    result = agent.invoke({
        "messages": [{"role": "user", "content": "Extract contact info from: John Doe, john@example.com, (555) 123-4567"}]
    })

    result["structured_response"]
    # {'name': 'John Doe', 'email': 'john@example.com', 'phone': '(555) 123-4567'}
    ```

Structured output native của provider mang lại độ tin cậy cao và validation nghiêm ngặt vì chính provider của model thực thi schema. Dùng nó khi có sẵn.

!!! note "Ghi chú"
    Nếu provider hỗ trợ native structured output cho model bạn chọn, việc viết `response_format=ProductReview` tương đương về mặt chức năng với `response_format=ProviderStrategy(ProductReview)`.

    Trong cả hai trường hợp, nếu structured output không được hỗ trợ, agent sẽ fallback sang chiến lược tool calling.

## Tool calling strategy

Với các model không hỗ trợ structured output native, LangChain dùng tool calling để đạt kết quả tương tự. Cách này hoạt động với mọi model hỗ trợ tool calling (hầu hết model hiện đại).

Để dùng strategy này, cấu hình một `ToolStrategy`:

```python
class ToolStrategy(Generic[SchemaT]):
    schema: type[SchemaT]
    tool_message_content: str | None
    handle_errors: Union[
        bool,
        str,
        type[Exception],
        tuple[type[Exception], ...],
        Callable[[Exception], str],
    ]
```

**`schema`** (bắt buộc): schema định nghĩa định dạng structured output. Hỗ trợ:

* **Pydantic model**: lớp con của `BaseModel` có field validation. Trả về instance Pydantic đã validate.
* **Dataclass**: Python dataclass có type annotation. Trả về dict.
* **TypedDict**: lớp typed dictionary. Trả về dict.
* **JSON Schema**: dictionary theo đặc tả JSON schema. Phải có key `title` và `description` ở top-level. Trả về dict.
* **Union type**: nhiều lựa chọn schema. Model sẽ chọn schema phù hợp nhất dựa trên ngữ cảnh.

**`tool_message_content`** (tuỳ chọn): nội dung tuỳ chỉnh cho tool message trả về khi structured output được sinh ra. Nếu không cung cấp, mặc định là message hiển thị dữ liệu structured response.

**`handle_errors`** (tuỳ chọn): chiến lược xử lý lỗi cho các trường hợp validation structured output thất bại. Mặc định là `True`.

* **`True`**: bắt tất cả lỗi với template lỗi mặc định
* **`str`**: bắt tất cả lỗi với message tuỳ chỉnh này
* **`type[Exception]`**: chỉ bắt loại exception này với message mặc định
* **`tuple[type[Exception], ...]`**: chỉ bắt các loại exception này với message mặc định
* **`Callable[[Exception], str]`**: hàm tuỳ chỉnh trả về message lỗi
* **`False`**: không retry, để exception tự lan truyền

=== "Pydantic Model"

    ```python
    from pydantic import BaseModel, Field
    from typing import Literal
    from langchain.agents import create_agent
    from langchain.agents.structured_output import ToolStrategy


    class ProductReview(BaseModel):
        """Analysis of a product review."""
        rating: int | None = Field(description="The rating of the product", ge=1, le=5)
        sentiment: Literal["positive", "negative"] = Field(description="The sentiment of the review")
        key_points: list[str] = Field(description="The key points of the review. Lowercase, 1-3 words each.")

    agent = create_agent(
        model="gpt-5.5",
        tools=tools,
        response_format=ToolStrategy(ProductReview)
    )

    result = agent.invoke({
        "messages": [{"role": "user", "content": "Analyze this review: 'Great product: 5 out of 5 stars. Fast shipping, but expensive'"}]
    })
    result["structured_response"]
    # ProductReview(rating=5, sentiment='positive', key_points=['fast shipping', 'expensive'])
    ```

=== "Dataclass"

    ```python
    from dataclasses import dataclass
    from typing import Literal
    from langchain.agents import create_agent
    from langchain.agents.structured_output import ToolStrategy


    @dataclass
    class ProductReview:
        """Analysis of a product review."""
        rating: int | None  # The rating of the product (1-5)
        sentiment: Literal["positive", "negative"]  # The sentiment of the review
        key_points: list[str]  # The key points of the review

    agent = create_agent(
        model="gpt-5.5",
        tools=tools,
        response_format=ToolStrategy(ProductReview)
    )

    result = agent.invoke({
        "messages": [{"role": "user", "content": "Analyze this review: 'Great product: 5 out of 5 stars. Fast shipping, but expensive'"}]
    })
    result["structured_response"]
    # {'rating': 5, 'sentiment': 'positive', 'key_points': ['fast shipping', 'expensive']}
    ```

=== "TypedDict"

    ```python
    from typing import Literal
    from typing_extensions import TypedDict
    from langchain.agents import create_agent
    from langchain.agents.structured_output import ToolStrategy


    class ProductReview(TypedDict):
        """Analysis of a product review."""
        rating: int | None  # The rating of the product (1-5)
        sentiment: Literal["positive", "negative"]  # The sentiment of the review
        key_points: list[str]  # The key points of the review

    agent = create_agent(
        model="gpt-5.5",
        tools=tools,
        response_format=ToolStrategy(ProductReview)
    )

    result = agent.invoke({
        "messages": [{"role": "user", "content": "Analyze this review: 'Great product: 5 out of 5 stars. Fast shipping, but expensive'"}]
    })
    result["structured_response"]
    # {'rating': 5, 'sentiment': 'positive', 'key_points': ['fast shipping', 'expensive']}
    ```

=== "JSON Schema"

    ```python
    from langchain.agents import create_agent
    from langchain.agents.structured_output import ToolStrategy


    product_review_schema = {
        "title": "ProductReview",
        "type": "object",
        "description": "Analysis of a product review.",
        "properties": {
            "rating": {
                "type": ["integer", "null"],
                "description": "The rating of the product (1-5)",
                "minimum": 1,
                "maximum": 5
            },
            "sentiment": {
                "type": "string",
                "enum": ["positive", "negative"],
                "description": "The sentiment of the review"
            },
            "key_points": {
                "type": "array",
                "items": {"type": "string"},
                "description": "The key points of the review"
            }
        },
        "required": ["sentiment", "key_points"]
    }

    agent = create_agent(
        model="gpt-5.5",
        tools=tools,
        response_format=ToolStrategy(product_review_schema)
    )

    result = agent.invoke({
        "messages": [{"role": "user", "content": "Analyze this review: 'Great product: 5 out of 5 stars. Fast shipping, but expensive'"}]
    })
    result["structured_response"]
    # {'rating': 5, 'sentiment': 'positive', 'key_points': ['fast shipping', 'expensive']}
    ```

=== "Union Types"

    ```python
    from pydantic import BaseModel, Field
    from typing import Literal, Union
    from langchain.agents import create_agent
    from langchain.agents.structured_output import ToolStrategy


    class ProductReview(BaseModel):
        """Analysis of a product review."""
        rating: int | None = Field(description="The rating of the product", ge=1, le=5)
        sentiment: Literal["positive", "negative"] = Field(description="The sentiment of the review")
        key_points: list[str] = Field(description="The key points of the review. Lowercase, 1-3 words each.")

    class CustomerComplaint(BaseModel):
        """A customer complaint about a product or service."""
        issue_type: Literal["product", "service", "shipping", "billing"] = Field(description="The type of issue")
        severity: Literal["low", "medium", "high"] = Field(description="The severity of the complaint")
        description: str = Field(description="Brief description of the complaint")

    agent = create_agent(
        model="gpt-5.5",
        tools=tools,
        response_format=ToolStrategy(Union[ProductReview, CustomerComplaint])
    )

    result = agent.invoke({
        "messages": [{"role": "user", "content": "Analyze this review: 'Great product: 5 out of 5 stars. Fast shipping, but expensive'"}]
    })
    result["structured_response"]
    # ProductReview(rating=5, sentiment='positive', key_points=['fast shipping', 'expensive'])
    ```

### Tuỳ chỉnh nội dung tool message

Tham số `tool_message_content` cho phép bạn tuỳ chỉnh message xuất hiện trong lịch sử hội thoại khi structured output được sinh ra:

```python
from pydantic import BaseModel, Field
from typing import Literal
from langchain.agents import create_agent
from langchain.agents.structured_output import ToolStrategy


class MeetingAction(BaseModel):
    """Action items extracted from a meeting transcript."""
    task: str = Field(description="The specific task to be completed")
    assignee: str = Field(description="Person responsible for the task")
    priority: Literal["low", "medium", "high"] = Field(description="Priority level")

agent = create_agent(
    model="gpt-5.5",
    tools=[],
    response_format=ToolStrategy(
        schema=MeetingAction,
        tool_message_content="Action item captured and added to meeting notes!"
    )
)

agent.invoke({
    "messages": [{"role": "user", "content": "From our meeting: Sarah needs to update the project timeline as soon as possible"}]
})
```

```
================================ Human Message =================================

From our meeting: Sarah needs to update the project timeline as soon as possible
================================== Ai Message ==================================
Tool Calls:
  MeetingAction (call_1)
 Call ID: call_1
  Args:
    task: Update the project timeline
    assignee: Sarah
    priority: high
================================= Tool Message =================================
Name: MeetingAction

Action item captured and added to meeting notes!
```

Nếu không có `tool_message_content`, [`ToolMessage`](https://reference.langchain.com/python/langchain-core/messages/tool/ToolMessage) cuối cùng của chúng ta sẽ là:

```
================================= Tool Message =================================
Name: MeetingAction

Returning structured response: {'task': 'update the project timeline', 'assignee': 'Sarah', 'priority': 'high'}
```

### Xử lý lỗi

Model có thể mắc lỗi khi sinh structured output thông qua tool calling. LangChain cung cấp cơ chế retry thông minh để tự động xử lý các lỗi này.

#### Lỗi nhiều structured output

Khi model gọi nhầm nhiều structured output tool cùng lúc, agent cung cấp phản hồi lỗi trong một [`ToolMessage`](https://reference.langchain.com/python/langchain-core/messages/tool/ToolMessage) và yêu cầu model thử lại:

```python
from pydantic import BaseModel, Field
from typing import Union
from langchain.agents import create_agent
from langchain.agents.structured_output import ToolStrategy


class ContactInfo(BaseModel):
    name: str = Field(description="Person's name")
    email: str = Field(description="Email address")

class EventDetails(BaseModel):
    event_name: str = Field(description="Name of the event")
    date: str = Field(description="Event date")

agent = create_agent(
    model="gpt-5.5",
    tools=[],
    response_format=ToolStrategy(Union[ContactInfo, EventDetails])  # Mặc định: handle_errors=True
)

agent.invoke({
    "messages": [{"role": "user", "content": "Extract info: John Doe (john@email.com) is organizing Tech Conference on March 15th"}]
})
```

```
================================ Human Message =================================

Extract info: John Doe (john@email.com) is organizing Tech Conference on March 15th
None
================================== Ai Message ==================================
Tool Calls:
  ContactInfo (call_1)
 Call ID: call_1
  Args:
    name: John Doe
    email: john@email.com
  EventDetails (call_2)
 Call ID: call_2
  Args:
    event_name: Tech Conference
    date: March 15th
================================= Tool Message =================================
Name: ContactInfo

Error: Model incorrectly returned multiple structured responses (ContactInfo, EventDetails) when only one is expected.
 Please fix your mistakes.
================================= Tool Message =================================
Name: EventDetails

Error: Model incorrectly returned multiple structured responses (ContactInfo, EventDetails) when only one is expected.
 Please fix your mistakes.
================================== Ai Message ==================================
Tool Calls:
  ContactInfo (call_3)
 Call ID: call_3
  Args:
    name: John Doe
    email: john@email.com
================================= Tool Message =================================
Name: ContactInfo

Returning structured response: {'name': 'John Doe', 'email': 'john@email.com'}
```

#### Lỗi validation schema

Khi structured output không khớp schema mong đợi, agent cung cấp phản hồi lỗi cụ thể:

```python
from pydantic import BaseModel, Field
from langchain.agents import create_agent
from langchain.agents.structured_output import ToolStrategy


class ProductRating(BaseModel):
    rating: int | None = Field(description="Rating from 1-5", ge=1, le=5)
    comment: str = Field(description="Review comment")

agent = create_agent(
    model="gpt-5.5",
    tools=[],
    response_format=ToolStrategy(ProductRating),  # Mặc định: handle_errors=True
    system_prompt="You are a helpful assistant that parses product reviews. Do not make any field or value up."
)

agent.invoke({
    "messages": [{"role": "user", "content": "Parse this: Amazing product, 10/10!"}]
})
```

```
================================ Human Message =================================

Parse this: Amazing product, 10/10!
================================== Ai Message ==================================
Tool Calls:
  ProductRating (call_1)
 Call ID: call_1
  Args:
    rating: 10
    comment: Amazing product
================================= Tool Message =================================
Name: ProductRating

Error: Failed to parse structured output for tool 'ProductRating': 1 validation error for ProductRating.rating
  Input should be less than or equal to 5 [type=less_than_equal, input_value=10, input_type=int].
 Please fix your mistakes.
================================== Ai Message ==================================
Tool Calls:
  ProductRating (call_2)
 Call ID: call_2
  Args:
    rating: 5
    comment: Amazing product
================================= Tool Message =================================
Name: ProductRating

Returning structured response: {'rating': 5, 'comment': 'Amazing product'}
```

#### Chiến lược xử lý lỗi

Bạn có thể tuỳ chỉnh cách xử lý lỗi bằng tham số `handle_errors`.

**Message lỗi tuỳ chỉnh:**

```python
ToolStrategy(
    schema=ProductRating,
    handle_errors="Please provide a valid rating between 1-5 and include a comment."
)
```

Nếu `handle_errors` là một string, agent sẽ *luôn luôn* yêu cầu model thử lại với một tool message cố định:

```
================================= Tool Message =================================
Name: ProductRating

Please provide a valid rating between 1-5 and include a comment.
```

**Chỉ xử lý exception cụ thể:**

```python
ToolStrategy(
    schema=ProductRating,
    handle_errors=ValueError  # Chỉ retry với ValueError
)
```

Nếu `handle_errors` là một loại exception, agent chỉ retry (dùng message lỗi mặc định) nếu exception phát sinh đúng loại được chỉ định. Trong mọi trường hợp khác, exception sẽ được raise.

**Xử lý nhiều loại exception:**

```python
ToolStrategy(
    schema=ProductRating,
    handle_errors=(ValueError, TypeError)  # Retry với ValueError và TypeError
)
```

Nếu `handle_errors` là một tuple exception, agent chỉ retry (dùng message lỗi mặc định) nếu exception phát sinh thuộc một trong các loại được chỉ định. Trong mọi trường hợp khác, exception sẽ được raise.

**Hàm xử lý lỗi tuỳ chỉnh:**

```python
from langchain.agents.structured_output import StructuredOutputValidationError
from langchain.agents.structured_output import MultipleStructuredOutputsError

def custom_error_handler(error: Exception) -> str:
    if isinstance(error, StructuredOutputValidationError):
        return "There was an issue with the format. Try again."
    elif isinstance(error, MultipleStructuredOutputsError):
        return "Multiple structured outputs were returned. Pick the most relevant one."
    else:
        return f"Error: {str(error)}"


agent = create_agent(
    model="gpt-5.5",
    tools=[],
    response_format=ToolStrategy(
                        schema=Union[ContactInfo, EventDetails],
                        handle_errors=custom_error_handler
                    )  # Mặc định: handle_errors=True
)

result = agent.invoke({
    "messages": [{"role": "user", "content": "Extract info: John Doe (john@email.com) is organizing Tech Conference on March 15th"}]
})

for msg in result['messages']:
    # Nếu message thực sự là đối tượng ToolMessage (không phải dict), kiểm tra tên class
    if type(msg).__name__ == "ToolMessage":
        print(msg.content)
    # Nếu message là dictionary hoặc bạn muốn có fallback
    elif isinstance(msg, dict) and msg.get('tool_call_id'):
        print(msg['content'])

```

Khi gặp `StructuredOutputValidationError`:

```
================================= Tool Message =================================
Name: ToolStrategy

There was an issue with the format. Try again.
```

Khi gặp `MultipleStructuredOutputsError`:

```
================================= Tool Message =================================
Name: ToolStrategy

Multiple structured outputs were returned. Pick the most relevant one.
```

Với các lỗi khác:

```
================================= Tool Message =================================
Name: ToolStrategy

Error: <error message>
```

**Không xử lý lỗi:**

```python
response_format = ToolStrategy(
    schema=ProductRating,
    handle_errors=False  # Mọi lỗi đều được raise
)
```
