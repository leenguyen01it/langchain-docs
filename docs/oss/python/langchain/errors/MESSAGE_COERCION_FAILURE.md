# MESSAGE_COERCION_FAILURE

Lỗi này xảy ra khi các object message không tuân theo định dạng mong đợi.

## Các định dạng message được chấp nhận

Các module LangChain chấp nhận `MessageLikeRepresentation`, được định nghĩa như sau:

```python
from typing import Union

from langchain_core.prompts.chat import (
    BaseChatPromptTemplate,
    BaseMessage,
    BaseMessagePromptTemplate,
)

MessageLikeRepresentation = Union[
    Union[BaseMessagePromptTemplate, BaseMessage, BaseChatPromptTemplate],
    tuple[
        Union[str, type],
        Union[str, list[dict], list[object]],
    ],
    str,
]
```

Bao gồm các object message kiểu OpenAI (`{ role: "user", content: "Hello world!" }`), tuple, và chuỗi thuần (sẽ được chuyển thành object [`HumanMessage`](https://reference.langchain.com/python/langchain-core/messages/human/HumanMessage)).

Nếu một module nhận một giá trị nằm ngoài các định dạng này, bạn sẽ gặp lỗi:

```python
from langchain_anthropic import ChatAnthropic

uncoercible_message = {"role": "HumanMessage", "random_field": "random value"}

model = ChatAnthropic(model="claude-sonnet-4-6")

model.invoke([uncoercible_message])
```

```text
ValueError: Message dict must contain 'role' and 'content' keys, got {'role': 'HumanMessage', 'random_field': 'random value'}
```

## Khắc phục sự cố

Để khắc phục lỗi này:

1. **Đảm bảo đúng định dạng**: Mọi input truyền vào chat model phải là một mảng các class message của LangChain hoặc một định dạng message-like được hỗ trợ
2. Kiểm tra không có việc chuyển thành chuỗi (stringification) hay biến đổi ngoài ý muốn xảy ra với message của bạn
3. Xem stack trace của lỗi và thêm câu lệnh logging để kiểm tra các object message trước khi chúng được truyền vào model

---

!!! info "Kết nối tài liệu này"
    [Kết nối tài liệu này](https://docs.langchain.com/use-these-docs) với Claude, VSCode, và nhiều công cụ khác qua MCP để có câu trả lời theo thời gian thực.

!!! info "Đóng góp"
    [Chỉnh sửa trang này trên GitHub](https://github.com/langchain-ai/docs/edit/main/src/oss/langchain/errors/MESSAGE_COERCION_FAILURE.mdx) hoặc [báo lỗi](https://github.com/langchain-ai/docs/issues/new/choose).
