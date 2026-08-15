# Messages

Message là đơn vị context cơ bản cho model trong LangChain. Chúng đại diện cho input và output của model, mang cả nội dung lẫn metadata cần thiết để biểu diễn state của một cuộc hội thoại khi tương tác với LLM.

Message là các object chứa:

* [**Role**](#message-types), xác định loại message (ví dụ `system`, `user`)
* [**Content**](#message-content), đại diện cho nội dung thực tế của message (như text, ảnh, audio, document, v.v.)
* [**Metadata**](#message-metadata), các field tuỳ chọn như thông tin response, message ID, và token usage

LangChain cung cấp một message type chuẩn hoạt động trên mọi provider model, đảm bảo hành vi nhất quán bất kể model nào được gọi.

## Cách dùng cơ bản

Cách đơn giản nhất để dùng message là tạo các object message và truyền chúng cho model khi [invoke](models.md#invocation).

```python
from langchain.chat_models import init_chat_model
from langchain.messages import HumanMessage, AIMessage, SystemMessage

model = init_chat_model("gpt-5-nano")

system_msg = SystemMessage("You are a helpful assistant.")
human_msg = HumanMessage("Hello, how are you?")

# Dùng với chat model
messages = [system_msg, human_msg]
response = model.invoke(messages)  # Trả về AIMessage
```

!!! tip "Mẹo"
    Các [agent](agents.md) nhiều lượt (multi-turn) tích luỹ lịch sử message dài. [LangSmith](https://docs.langchain.com/langsmith/observability) ghi lại mỗi turn, kết quả tool, và phản hồi model để bạn có thể kiểm tra toàn bộ hội thoại. Theo [hướng dẫn tracing quickstart](https://docs.langchain.com/langsmith/trace-with-langchain) để bật tracing.

    Bạn cũng nên thiết lập [LangSmith Engine](https://docs.langchain.com/langsmith/engine), công cụ giám sát trace, phát hiện vấn đề, và đề xuất cách khắc phục.

### Text prompt

Text prompt là các chuỗi, lý tưởng cho các tác vụ sinh nội dung đơn giản khi bạn không cần giữ lại lịch sử hội thoại.

```python
response = model.invoke("Write a haiku about spring")
```

**Dùng text prompt khi:**

* Bạn có một request đơn lẻ, độc lập
* Bạn không cần lịch sử hội thoại
* Bạn muốn độ phức tạp code tối thiểu

### Message prompt

Ngoài ra, bạn có thể truyền một danh sách message cho model bằng cách cung cấp một danh sách các object message.

```python
from langchain.messages import SystemMessage, HumanMessage, AIMessage

messages = [
    SystemMessage("You are a poetry expert"),
    HumanMessage("Write a haiku about spring"),
    AIMessage("Cherry blossoms bloom...")
]
response = model.invoke(messages)
```

**Dùng message prompt khi:**

* Quản lý hội thoại nhiều lượt (multi-turn)
* Làm việc với nội dung đa phương thức (ảnh, audio, file)
* Cần đưa vào chỉ dẫn hệ thống (system instruction)

### Định dạng dictionary

Bạn cũng có thể chỉ định message trực tiếp theo định dạng OpenAI chat completions.

```python
messages = [
    {"role": "system", "content": "You are a poetry expert"},
    {"role": "user", "content": "Write a haiku about spring"},
    {"role": "assistant", "content": "Cherry blossoms bloom..."}
]
response = model.invoke(messages)
```

## Các loại message

* [System message](#system-message), cho model biết cách hành xử và cung cấp context cho tương tác
* [Human message](#human-message), đại diện cho input và tương tác của người dùng với model
* [AI message](#ai-message), phản hồi do model sinh ra, bao gồm nội dung văn bản, tool call, và metadata
* [Tool message](#tool-message), đại diện cho output của [tool call](models.md#tool-calling)

### System message

Một [`SystemMessage`](https://reference.langchain.com/python/langchain-core/messages/system/SystemMessage) đại diện cho một tập chỉ dẫn ban đầu định hình hành vi của model. Bạn có thể dùng system message để đặt tông giọng, định nghĩa vai trò của model, và thiết lập nguyên tắc cho các phản hồi.

=== "Chỉ dẫn cơ bản"
    ```python
    system_msg = SystemMessage("You are a helpful coding assistant.")

    messages = [
        system_msg,
        HumanMessage("How do I create a REST API?")
    ]
    response = model.invoke(messages)
    ```

=== "Persona chi tiết"
    ```python
    from langchain.messages import SystemMessage, HumanMessage

    system_msg = SystemMessage("""
    You are a senior Python developer with expertise in web frameworks.
    Always provide code examples and explain your reasoning.
    Be concise but thorough in your explanations.
    """)

    messages = [
        system_msg,
        HumanMessage("How do I create a REST API?")
    ]
    response = model.invoke(messages)
    ```

### Human message

Một [`HumanMessage`](https://reference.langchain.com/python/langchain-core/messages/human/HumanMessage) đại diện cho input và tương tác của người dùng. Nó có thể chứa văn bản, ảnh, audio, file, và bất kỳ [content](#message-content) đa phương thức nào khác.

#### Nội dung văn bản

=== "Object message"
    ```python
    response = model.invoke([
      HumanMessage("What is machine learning?")
    ])
    ```

=== "Rút gọn bằng string"
    ```python
    # Dùng một string là cách rút gọn cho một HumanMessage duy nhất
    response = model.invoke("What is machine learning?")
    ```

#### Metadata của message

```python
# Thêm metadata
human_msg = HumanMessage(
    content="Hello!",
    name="alice",  # Tuỳ chọn: định danh người dùng khác nhau
    id="msg_123",  # Tuỳ chọn: định danh duy nhất để tracing
)
```

!!! note "Ghi chú"
    Hành vi của field `name` khác nhau tuỳ provider, một số dùng nó để định danh người dùng, số khác bỏ qua. Để kiểm tra, xem [reference](https://reference.langchain.com/python/integrations/) của provider model.

### AI message

Một [`AIMessage`](https://reference.langchain.com/python/langchain-core/messages/ai/AIMessage) đại diện cho output của một lần gọi model. Nó có thể bao gồm dữ liệu đa phương thức, tool call, và metadata riêng của provider mà bạn có thể truy cập sau đó.

```python
response = model.invoke("Explain AI")
print(type(response))  # <class 'langchain.messages.AIMessage'>
```

Object [`AIMessage`](https://reference.langchain.com/python/langchain-core/messages/ai/AIMessage) được model trả về khi gọi nó, chứa toàn bộ metadata liên quan trong response.

Các provider cân nhắc/đặt ngữ cảnh cho từng loại message khác nhau, nghĩa là đôi khi hữu ích khi tự tạo thủ công một object [`AIMessage`](https://reference.langchain.com/python/langchain-core/messages/ai/AIMessage) mới và chèn nó vào lịch sử message như thể nó đến từ model.

```python
from langchain.messages import AIMessage, SystemMessage, HumanMessage

# Tạo một AI message thủ công (ví dụ cho lịch sử hội thoại)
ai_msg = AIMessage("I'd be happy to help you with that question!")

# Thêm vào lịch sử hội thoại
messages = [
    SystemMessage("You are a helpful assistant"),
    HumanMessage("Can you help me?"),
    ai_msg,  # Chèn như thể nó đến từ model
    HumanMessage("Great! What's 2+2?")
]

response = model.invoke(messages)
```

??? note "Thuộc tính"
    * **`text`** (`string`): Nội dung văn bản của message.
    * **`content`** (`string | dict[]`): Nội dung thô của message.
    * **`content_blocks`** (`ContentBlock[]`): Các [content block](#message-content) đã chuẩn hoá của message.
    * **`tool_calls`** (`dict[] | None`): Các tool call mà model đã thực hiện. Rỗng nếu không có tool nào được gọi.
    * **`id`** (`string`): Định danh duy nhất cho message (do LangChain tự sinh hoặc được trả về trong response của provider).
    * **`usage_metadata`** (`dict | None`): Metadata usage của message, có thể chứa số lượng token khi có sẵn.
    * **`response_metadata`** (`ResponseMetadata | None`): Metadata response của message.

#### Tool call

Khi model thực hiện [tool call](models.md#tool-calling), chúng được đưa vào [`AIMessage`](https://reference.langchain.com/python/langchain-core/messages/ai/AIMessage):

```python
from langchain.chat_models import init_chat_model

model = init_chat_model("gpt-5-nano")

def get_weather(location: str) -> str:
    """Get the weather at a location."""
    ...

model_with_tools = model.bind_tools([get_weather])
response = model_with_tools.invoke("What's the weather in Paris?")

for tool_call in response.tool_calls:
    print(f"Tool: {tool_call['name']}")
    print(f"Args: {tool_call['args']}")
    print(f"ID: {tool_call['id']}")
```

Các dữ liệu có cấu trúc khác, như reasoning hay citation, cũng có thể xuất hiện trong [content](#message-content) của message.

#### Token usage

Một [`AIMessage`](https://reference.langchain.com/python/langchain-core/messages/ai/AIMessage) có thể chứa số lượng token và các metadata usage khác trong field [`usage_metadata`](https://reference.langchain.com/python/langchain-core/messages/ai/UsageMetadata):

```python
from langchain.chat_models import init_chat_model

model = init_chat_model("gpt-5-nano")

response = model.invoke("Hello!")
response.usage_metadata
```

```
{'input_tokens': 8,
 'output_tokens': 304,
 'total_tokens': 312,
 'input_token_details': {'audio': 0, 'cache_read': 0},
 'output_token_details': {'audio': 0, 'reasoning': 256}}
```

Xem [`UsageMetadata`](https://reference.langchain.com/python/langchain-core/messages/ai/UsageMetadata) để biết chi tiết.

#### Streaming và chunk

Trong khi streaming, bạn sẽ nhận được các object [`AIMessageChunk`](https://reference.langchain.com/python/langchain-core/messages/ai/AIMessageChunk) có thể kết hợp lại thành một object message đầy đủ:

```python
chunks = []
full_message = None
for chunk in model.stream("Hi"):
    chunks.append(chunk)
    print(chunk.text)
    full_message = chunk if full_message is None else full_message + chunk
```

!!! note "Ghi chú"
    Tìm hiểu thêm:

    * [Streaming token từ chat model](models.md#stream)
    * [Streaming token và/hoặc step từ agent](streaming.md)

### Tool message

Với các model hỗ trợ [tool calling](models.md#tool-calling), AI message có thể chứa tool call. Tool message dùng để truyền kết quả của một lần thực thi tool duy nhất trở lại cho model.

[Tool](tools.md) có thể sinh trực tiếp object [`ToolMessage`](https://reference.langchain.com/python/langchain-core/messages/tool/ToolMessage). Dưới đây là một ví dụ đơn giản. Đọc thêm tại [hướng dẫn tools](tools.md).

```python
from langchain.messages import AIMessage
from langchain.messages import ToolMessage

# Sau khi model thực hiện một tool call
# (Ở đây, chúng ta minh hoạ việc tự tạo message thủ công cho ngắn gọn)
ai_message = AIMessage(
    content=[],
    tool_calls=[{
        "name": "get_weather",
        "args": {"location": "San Francisco"},
        "id": "call_123"
    }]
)

# Thực thi tool và tạo message kết quả
weather_result = "Sunny, 72°F"
tool_message = ToolMessage(
    content=weather_result,
    tool_call_id="call_123"  # Phải khớp với call ID
)

# Tiếp tục hội thoại
messages = [
    HumanMessage("What's the weather in San Francisco?"),
    ai_message,  # Tool call của model
    tool_message,  # Kết quả thực thi tool
]
response = model.invoke(messages)  # Model xử lý kết quả
```

??? note "Thuộc tính"
    * **`content`** (`string`, bắt buộc): Output dạng chuỗi của tool call.
    * **`tool_call_id`** (`string`, bắt buộc): ID của tool call mà message này đang phản hồi. Phải khớp với ID của tool call trong [`AIMessage`](https://reference.langchain.com/python/langchain-core/messages/ai/AIMessage).
    * **`name`** (`string`, bắt buộc): Tên của tool đã được gọi.
    * **`artifact`** (`dict`): Dữ liệu bổ sung không gửi cho model nhưng có thể truy cập bằng code.

!!! note "Ghi chú"
    Field `artifact` lưu dữ liệu bổ sung sẽ không được gửi cho model nhưng có thể truy cập bằng code. Điều này hữu ích để lưu kết quả thô, thông tin debug, hoặc dữ liệu cho xử lý downstream mà không làm rối context của model.

    ??? note "Ví dụ: Dùng artifact cho metadata retrieval"
        Ví dụ, một tool [retrieval](https://docs.langchain.com/oss/python/deepagents/retrieval) có thể lấy một đoạn văn từ document để model tham chiếu. Trong khi `content` của message chứa văn bản mà model sẽ tham chiếu, `artifact` có thể chứa document identifier hoặc metadata khác mà ứng dụng có thể dùng (ví dụ để render một trang). Xem ví dụ bên dưới:

        ```python
        from langchain.messages import ToolMessage

        # Gửi cho model
        message_content = "It was the best of times, it was the worst of times."

        # Artifact dùng ở downstream
        artifact = {"document_id": "doc_123", "page": 0}

        tool_message = ToolMessage(
            content=message_content,
            tool_call_id="call_123",
            name="search_books",
            artifact=artifact,
        )
        ```

        Xem [tutorial RAG](https://docs.langchain.com/oss/python/deepagents/rag) để có ví dụ đầy đủ về xây dựng [agent](agents.md) retrieval với LangChain.

## Message content

Bạn có thể hình dung content của một message là payload dữ liệu được gửi tới model. Message có thuộc tính `content` với kiểu lỏng lẻo (loosely-typed), hỗ trợ string và list các object không định kiểu (ví dụ dictionary). Điều này cho phép hỗ trợ các cấu trúc gốc của provider trực tiếp trong chat model của LangChain, như nội dung [đa phương thức](#multimodal) và các dữ liệu khác.

Ngoài ra, LangChain cung cấp các content type riêng cho văn bản, reasoning, citation, dữ liệu đa phương thức, server-side tool call, và các nội dung message khác. Xem [content block](#standard-content-blocks) bên dưới.

Chat model của LangChain nhận content message trong thuộc tính `content`.

Nó có thể chứa:

1. Một string
2. Một list các content block theo định dạng gốc của provider
3. Một list [content block chuẩn của LangChain](#standard-content-blocks)

Xem ví dụ bên dưới dùng input [đa phương thức](#multimodal):

```python
from langchain.messages import HumanMessage

# Content dạng string
human_message = HumanMessage("Hello, how are you?")

# Định dạng gốc của provider (ví dụ OpenAI)
human_message = HumanMessage(content=[
    {"type": "text", "text": "Hello, how are you?"},
    {"type": "image_url", "image_url": {"url": "https://example.com/image.jpg"}}
])

# List content block chuẩn
human_message = HumanMessage(content_blocks=[
    {"type": "text", "text": "Hello, how are you?"},
    {"type": "image", "url": "https://example.com/image.jpg"},
])
```

!!! tip "Mẹo"
    Chỉ định `content_blocks` khi khởi tạo một message vẫn sẽ điền vào `content` của message, nhưng cung cấp một interface type-safe để làm điều đó.

### Content block chuẩn

LangChain cung cấp một biểu diễn chuẩn cho content của message, hoạt động trên mọi provider.

Object message triển khai một thuộc tính `content_blocks` sẽ lazily parse thuộc tính `content` thành một biểu diễn chuẩn, type-safe. Ví dụ, message được sinh từ [`ChatAnthropic`](https://docs.langchain.com/oss/python/integrations/chat/anthropic) hoặc [`ChatOpenAI`](https://docs.langchain.com/oss/python/integrations/chat/openai) sẽ bao gồm block `thinking` hoặc `reasoning` theo định dạng riêng của từng provider, nhưng có thể được lazily parse thành một biểu diễn [`ReasoningContentBlock`](#content-block-reference) nhất quán:

=== "Anthropic"
    ```python
    from langchain.messages import AIMessage

    message = AIMessage(
        content=[
            {"type": "thinking", "thinking": "...", "signature": "WaUjzkyp..."},
            {"type": "text", "text": "..."},
        ],
        response_metadata={"model_provider": "anthropic"}
    )
    message.content_blocks
    ```

    ```
    [{'type': 'reasoning',
      'reasoning': '...',
      'extras': {'signature': 'WaUjzkyp...'}},
     {'type': 'text', 'text': '...'}]
    ```

=== "OpenAI"
    ```python
    from langchain.messages import AIMessage

    message = AIMessage(
        content=[
            {
                "type": "reasoning",
                "id": "rs_abc123",
                "summary": [
                    {"type": "summary_text", "text": "summary 1"},
                    {"type": "summary_text", "text": "summary 2"},
                ],
            },
            {"type": "text", "text": "...", "id": "msg_abc123"},
        ],
        response_metadata={"model_provider": "openai"}
    )
    message.content_blocks
    ```

    ```
    [{'type': 'reasoning', 'id': 'rs_abc123', 'reasoning': 'summary 1'},
     {'type': 'reasoning', 'id': 'rs_abc123', 'reasoning': 'summary 2'},
     {'type': 'text', 'text': '...', 'id': 'msg_abc123'}]
    ```

Xem [hướng dẫn tích hợp](https://docs.langchain.com/oss/python/integrations/providers/overview) để bắt đầu với inference provider bạn chọn.

!!! note "Ghi chú"
    **Serialize content chuẩn**

    Nếu một ứng dụng bên ngoài LangChain cần truy cập biểu diễn content block chuẩn, bạn có thể chọn lưu content block vào content của message.

    Để làm điều này, đặt biến môi trường `LC_OUTPUT_VERSION` thành `v1`. Hoặc khởi tạo bất kỳ chat model nào với `output_version="v1"`:

    ```python
    from langchain.chat_models import init_chat_model

    model = init_chat_model("gpt-5-nano", output_version="v1")
    ```

### Đa phương thức (Multimodal)

**Đa phương thức (multimodality)** đề cập đến khả năng làm việc với dữ liệu ở nhiều dạng khác nhau, như văn bản, audio, ảnh, và video. LangChain có các type chuẩn cho các loại dữ liệu này, dùng được trên mọi provider.

[Chat model](models.md) có thể nhận dữ liệu đa phương thức làm input và sinh ra dữ liệu đa phương thức làm output. Dưới đây là các ví dụ ngắn về message input có dữ liệu đa phương thức.

!!! note "Ghi chú"
    Các key bổ sung có thể được đưa vào ở cấp cao nhất của content block hoặc lồng trong `"extras": {"key": value}`.

    [OpenAI](https://docs.langchain.com/oss/python/integrations/chat/openai), ví dụ, yêu cầu một filename cho PDF. Xem [trang provider](https://docs.langchain.com/oss/python/integrations/providers/overview) của model bạn chọn để biết chi tiết cụ thể.

=== "Input ảnh"
    ```python
    # Từ URL
    message = {
        "role": "user",
        "content": [
            {"type": "text", "text": "Describe the content of this image."},
            {"type": "image", "url": "https://example.com/path/to/image.jpg"},
        ]
    }

    # Từ dữ liệu base64
    message = {
        "role": "user",
        "content": [
            {"type": "text", "text": "Describe the content of this image."},
            {
                "type": "image",
                "base64": "AAAAIGZ0eXBtcDQyAAAAAGlzb21tcDQyAAACAGlzb2...",
                "mime_type": "image/jpeg",
            },
        ]
    }

    # Từ File ID do provider quản lý
    message = {
        "role": "user",
        "content": [
            {"type": "text", "text": "Describe the content of this image."},
            {"type": "image", "file_id": "file-abc123"},
        ]
    }
    ```

=== "Input document PDF"
    ```python
    # Từ URL
    message = {
        "role": "user",
        "content": [
            {"type": "text", "text": "Describe the content of this document."},
            {"type": "file", "url": "https://example.com/path/to/document.pdf"},
        ]
    }

    # Từ dữ liệu base64
    message = {
        "role": "user",
        "content": [
            {"type": "text", "text": "Describe the content of this document."},
            {
                "type": "file",
                "base64": "AAAAIGZ0eXBtcDQyAAAAAGlzb21tcDQyAAACAGlzb2...",
                "mime_type": "application/pdf",
            },
        ]
    }

    # Từ File ID do provider quản lý
    message = {
        "role": "user",
        "content": [
            {"type": "text", "text": "Describe the content of this document."},
            {"type": "file", "file_id": "file-abc123"},
        ]
    }
    ```

=== "Input audio"
    ```python
    # Từ dữ liệu base64
    message = {
        "role": "user",
        "content": [
            {"type": "text", "text": "Describe the content of this audio."},
            {
                "type": "audio",
                "base64": "AAAAIGZ0eXBtcDQyAAAAAGlzb21tcDQyAAACAGlzb2...",
                "mime_type": "audio/wav",
            },
        ]
    }

    # Từ File ID do provider quản lý
    message = {
        "role": "user",
        "content": [
            {"type": "text", "text": "Describe the content of this audio."},
            {"type": "audio", "file_id": "file-abc123"},
        ]
    }
    ```

=== "Input video"
    ```python
    # Từ dữ liệu base64
    message = {
        "role": "user",
        "content": [
            {"type": "text", "text": "Describe the content of this video."},
            {
                "type": "video",
                "base64": "AAAAIGZ0eXBtcDQyAAAAAGlzb21tcDQyAAACAGlzb2...",
                "mime_type": "video/mp4",
            },
        ]
    }

    # Từ File ID do provider quản lý
    message = {
        "role": "user",
        "content": [
            {"type": "text", "text": "Describe the content of this video."},
            {"type": "video", "file_id": "file-abc123"},
        ]
    }
    ```

!!! warning "Cảnh báo"
    Không phải model nào cũng hỗ trợ mọi loại file. Kiểm tra [reference](https://reference.langchain.com/python/integrations/) của provider model để biết định dạng và giới hạn kích thước hỗ trợ.

### Tham chiếu content block

Content block được biểu diễn (khi tạo message hoặc truy cập thuộc tính `content_blocks`) dưới dạng một list các typed dictionary. Mỗi item trong list phải tuân theo một trong các block type sau:

#### Core

??? note "TextContentBlock"
    **Mục đích:** Output văn bản chuẩn

    * **`type`** (`string`, bắt buộc): Luôn là `"text"`
    * **`text`** (`string`, bắt buộc): Nội dung văn bản
    * **`annotations`** (`object[]`): Danh sách annotation cho văn bản
    * **`extras`** (`object`): Dữ liệu bổ sung riêng của provider

    **Ví dụ:**

    ```python
    {
        "type": "text",
        "text": "Hello world",
        "annotations": []
    }
    ```

??? note "ReasoningContentBlock"
    **Mục đích:** Các bước suy luận (reasoning) của model

    * **`type`** (`string`, bắt buộc): Luôn là `"reasoning"`
    * **`reasoning`** (`string`): Nội dung reasoning
    * **`extras`** (`object`): Dữ liệu bổ sung riêng của provider

    **Ví dụ:**

    ```python
    {
        "type": "reasoning",
        "reasoning": "The user is asking about...",
        "extras": {"signature": "abc123"},
    }
    ```

#### Đa phương thức

??? note "ImageContentBlock"
    **Mục đích:** Dữ liệu ảnh

    * **`type`** (`string`, bắt buộc): Luôn là `"image"`
    * **`url`** (`string`): URL trỏ tới vị trí ảnh.
    * **`base64`** (`string`): Dữ liệu ảnh mã hoá base64.
    * **`id`** (`string`): Định danh duy nhất cho content block này (do provider hoặc LangChain sinh ra).
    * **`mime_type`** (`string`): [MIME type](https://www.iana.org/assignments/media-types/media-types.xhtml#image) của ảnh (ví dụ `image/jpeg`, `image/png`). Bắt buộc với dữ liệu base64.

??? note "AudioContentBlock"
    **Mục đích:** Dữ liệu audio

    * **`type`** (`string`, bắt buộc): Luôn là `"audio"`
    * **`url`** (`string`): URL trỏ tới vị trí audio.
    * **`base64`** (`string`): Dữ liệu audio mã hoá base64.
    * **`id`** (`string`): Định danh duy nhất cho content block này (do provider hoặc LangChain sinh ra).
    * **`mime_type`** (`string`): [MIME type](https://www.iana.org/assignments/media-types/media-types.xhtml#audio) của audio (ví dụ `audio/mpeg`, `audio/wav`). Bắt buộc với dữ liệu base64.

??? note "VideoContentBlock"
    **Mục đích:** Dữ liệu video

    * **`type`** (`string`, bắt buộc): Luôn là `"video"`
    * **`url`** (`string`): URL trỏ tới vị trí video.
    * **`base64`** (`string`): Dữ liệu video mã hoá base64.
    * **`id`** (`string`): Định danh duy nhất cho content block này (do provider hoặc LangChain sinh ra).
    * **`mime_type`** (`string`): [MIME type](https://www.iana.org/assignments/media-types/media-types.xhtml#video) của video (ví dụ `video/mp4`, `video/webm`). Bắt buộc với dữ liệu base64.

??? note "FileContentBlock"
    **Mục đích:** File chung (PDF, v.v.)

    * **`type`** (`string`, bắt buộc): Luôn là `"file"`
    * **`url`** (`string`): URL trỏ tới vị trí file.
    * **`base64`** (`string`): Dữ liệu file mã hoá base64.
    * **`id`** (`string`): Định danh duy nhất cho content block này (do provider hoặc LangChain sinh ra).
    * **`mime_type`** (`string`): [MIME type](https://www.iana.org/assignments/media-types/media-types.xhtml) của file (ví dụ `application/pdf`). Bắt buộc với dữ liệu base64.

??? note "PlainTextContentBlock"
    **Mục đích:** Văn bản document (`.txt`, `.md`)

    * **`type`** (`string`, bắt buộc): Luôn là `"text-plain"`
    * **`text`** (`string`): Nội dung văn bản
    * **`mime_type`** (`string`): [MIME type](https://www.iana.org/assignments/media-types/media-types.xhtml) của văn bản (ví dụ `text/plain`, `text/markdown`)

#### Tool Calling

??? note "ToolCall"
    **Mục đích:** Lệnh gọi hàm (function call)

    * **`type`** (`string`, bắt buộc): Luôn là `"tool_call"`
    * **`name`** (`string`, bắt buộc): Tên của tool cần gọi
    * **`args`** (`object`, bắt buộc): Tham số truyền cho tool
    * **`id`** (`string`, bắt buộc): Định danh duy nhất cho tool call này

    **Ví dụ:**

    ```python
    {
        "type": "tool_call",
        "name": "search",
        "args": {"query": "weather"},
        "id": "call_123"
    }
    ```

??? note "ToolCallChunk"
    **Mục đích:** Các mảnh tool call khi streaming

    * **`type`** (`string`, bắt buộc): Luôn là `"tool_call_chunk"`
    * **`name`** (`string`): Tên của tool đang được gọi
    * **`args`** (`string`): Tham số tool chưa hoàn chỉnh (có thể là JSON chưa đầy đủ)
    * **`id`** (`string`): Định danh tool call
    * **`index`** (`number | string`): Vị trí của chunk này trong stream

??? note "InvalidToolCall"
    **Mục đích:** Lệnh gọi sai định dạng, dùng để bắt lỗi parse JSON.

    * **`type`** (`string`, bắt buộc): Luôn là `"invalid_tool_call"`
    * **`name`** (`string`): Tên của tool bị gọi thất bại
    * **`args`** (`object`): Tham số truyền cho tool
    * **`error`** (`string`): Mô tả lỗi đã xảy ra

#### Thực thi Tool phía Server (Server-Side)

??? note "ServerToolCall"
    **Mục đích:** Tool call được thực thi ở phía server.

    * **`type`** (`string`, bắt buộc): Luôn là `"server_tool_call"`
    * **`id`** (`string`, bắt buộc): Định danh gắn với tool call.
    * **`name`** (`string`, bắt buộc): Tên của tool cần gọi.
    * **`args`** (`string`, bắt buộc): Tham số tool chưa hoàn chỉnh (có thể là JSON chưa đầy đủ)

??? note "ServerToolCallChunk"
    **Mục đích:** Các mảnh server-side tool call khi streaming

    * **`type`** (`string`, bắt buộc): Luôn là `"server_tool_call_chunk"`
    * **`id`** (`string`): Định danh gắn với tool call.
    * **`name`** (`string`): Tên của tool đang được gọi
    * **`args`** (`string`): Tham số tool chưa hoàn chỉnh (có thể là JSON chưa đầy đủ)
    * **`index`** (`number | string`): Vị trí của chunk này trong stream

??? note "ServerToolResult"
    **Mục đích:** Kết quả tìm kiếm

    * **`type`** (`string`, bắt buộc): Luôn là `"server_tool_result"`
    * **`tool_call_id`** (`string`, bắt buộc): Định danh của server tool call tương ứng.
    * **`id`** (`string`): Định danh gắn với server tool result.
    * **`status`** (`string`, bắt buộc): Trạng thái thực thi của tool phía server. `"success"` hoặc `"error"`.
    * **`output`**: Output của tool đã thực thi.

#### Block riêng của provider

??? note "NonStandardContentBlock"
    **Mục đích:** Lối thoát (escape hatch) riêng của provider

    * **`type`** (`string`, bắt buộc): Luôn là `"non_standard"`
    * **`value`** (`object`, bắt buộc): Cấu trúc dữ liệu riêng của provider

    **Cách dùng:** Cho các tính năng thử nghiệm hoặc đặc thù riêng của provider

Các content type khác riêng của từng provider có thể được tìm thấy trong [tài liệu reference](https://docs.langchain.com/oss/python/integrations/providers/overview) của mỗi provider model.

!!! tip "Mẹo"
    Xem các định nghĩa type chuẩn (canonical) tại [API reference](https://reference.langchain.com/python/langchain/messages).

!!! info "Thông tin"
    Content block được giới thiệu như một thuộc tính mới trên message trong LangChain v1 để chuẩn hoá định dạng content trên các provider trong khi vẫn giữ khả năng tương thích ngược với code hiện có.

    Content block không phải là một sự thay thế cho thuộc tính [`content`](https://reference.langchain.com/python/langchain-core/messages/base/BaseMessage), mà là một thuộc tính mới có thể dùng để truy cập content của một message theo định dạng chuẩn.

## Serialization

Bạn có thể serialize message thành plain object để lưu trữ và deserialize ngược lại thành các instance message. Điều này hữu ích để lưu lại lịch sử hội thoại và tiếp tục session.

```python
from langchain.messages import HumanMessage
from langchain_core.load import dumpd, load

message = HumanMessage("What is the capital of France?")

# Serialize thành một dict thuần
serialized = dumpd(message)

# Deserialize ngược lại thành object message
restored = load(serialized)
```

!!! warning "Cảnh báo"
    **`load()` khởi tạo các object Python và có thể kích hoạt side effect trong quá trình deserialize. Không bao giờ gọi `load()` với dữ liệu từ nguồn không đáng tin cậy hoặc chưa được xác thực.**

## Dùng với chat model

[Chat model](models.md) nhận một chuỗi các object message làm input và trả về một [`AIMessage`](https://reference.langchain.com/python/langchain-core/messages/ai/AIMessage) làm output. Các tương tác thường stateless, vì vậy một vòng lặp hội thoại đơn giản gồm việc gọi model với một danh sách message ngày càng dài.

Xem các hướng dẫn bên dưới để tìm hiểu thêm:

* Các tính năng có sẵn để [lưu trữ và quản lý lịch sử hội thoại](short-term-memory.md)
* Các chiến lược quản lý context window, bao gồm [cắt bớt và tóm tắt message](short-term-memory.md#common-patterns)
