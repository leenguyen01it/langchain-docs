# Cài đặt LangGraph

Để cài đặt package LangGraph cơ bản:

=== "pip"
    ```bash
    pip install -U langgraph
    ```

=== "uv"
    ```bash
    uv add langgraph
    ```

Để dùng LangGraph, bạn thường sẽ cần truy cập LLM và định nghĩa tool. Bạn có thể làm điều này theo bất kỳ cách nào phù hợp.

Một cách để làm việc này (cũng là cách được dùng xuyên suốt trong tài liệu) là dùng [LangChain](../langchain/overview.md).

Cài đặt LangChain với:

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

Để làm việc với các package nhà cung cấp LLM cụ thể, bạn cần cài đặt chúng riêng.

Tham khảo trang [integrations](https://docs.langchain.com/oss/python/integrations/providers/overview) để biết hướng dẫn cài đặt theo từng nhà cung cấp.

---

[Kết nối tài liệu này](https://docs.langchain.com/use-these-docs) với Claude, VSCode, và nhiều công cụ khác qua MCP để nhận câu trả lời theo thời gian thực.

[Chỉnh sửa trang này trên GitHub](https://github.com/langchain-ai/docs/edit/main/src/oss/langgraph/install.mdx) hoặc [báo lỗi (file an issue)](https://github.com/langchain-ai/docs/issues/new/choose).
