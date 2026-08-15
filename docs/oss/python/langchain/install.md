# Cài đặt LangChain

Để cài đặt package LangChain:

=== "pip"
    ```bash
    pip install -U langchain
    # Yêu cầu Python 3.10+
    ```

=== "uv"
    ```bash
    uv add langchain
    # Yêu cầu Python 3.10+
    ```

LangChain cung cấp tích hợp với hàng trăm LLM và hàng nghìn tích hợp khác. Các tích hợp này nằm trong các package nhà cung cấp (provider package) độc lập.

=== "pip"
    ```bash
    # Cài đặt tích hợp OpenAI
    pip install -U langchain-openai

    # Cài đặt tích hợp Anthropic
    pip install -U langchain-anthropic
    ```

=== "uv"
    ```bash
    # Cài đặt tích hợp OpenAI
    uv add langchain-openai

    # Cài đặt tích hợp Anthropic
    uv add langchain-anthropic
    ```

!!! tip "Mẹo"
    Xem [tab Integrations](https://docs.langchain.com/oss/python/integrations/providers/overview) để có danh sách đầy đủ các tích hợp hiện có.

Sau khi cài đặt LangChain xong, bạn có thể bắt đầu bằng cách theo dõi [hướng dẫn Quickstart](https://docs.langchain.com/oss/python/langchain/quickstart).

!!! tip "Mẹo"
    Thiết lập tracing của [LangSmith](https://smith.langchain.com?utm_source=docs&utm_medium=cta&utm_campaign=langsmith-signup&utm_content=oss-langchain-install) để debug ứng dụng LangChain đầu tiên của bạn. Theo dõi [hướng dẫn nhanh về tracing](https://docs.langchain.com/langsmith/trace-with-langchain) để bắt đầu. Chúng tôi cũng khuyến nghị bạn thiết lập [LangSmith Engine](https://docs.langchain.com/langsmith/engine), công cụ giám sát trace, phát hiện vấn đề, và đề xuất cách khắc phục.

---

[Kết nối tài liệu này](https://docs.langchain.com/use-these-docs) với Claude, VSCode, và nhiều công cụ khác qua MCP để nhận câu trả lời theo thời gian thực.

[Chỉnh sửa trang này trên GitHub](https://github.com/langchain-ai/docs/edit/main/src/oss/langchain/install.mdx) hoặc [báo lỗi (file an issue)](https://github.com/langchain-ai/docs/issues/new/choose).
