# Agent Chat UI

[Agent Chat UI](https://github.com/langchain-ai/agent-chat-ui) là một ứng dụng Next.js cung cấp giao diện hội thoại để tương tác với bất kỳ agent LangChain nào. Nó hỗ trợ chat theo thời gian thực, trực quan hóa tool call, và các tính năng nâng cao như debug time-travel và fork trạng thái. Agent Chat UI hoạt động liền mạch với các agent được tạo bằng [`create_agent`](https://reference.langchain.com/python/langchain/agents/factory/create_agent) và cung cấp trải nghiệm tương tác cho agent của bạn với thiết lập tối thiểu, dù bạn đang chạy cục bộ hay trong môi trường đã deploy (như [LangSmith](https://docs.langchain.com/langsmith/observability)).

Agent Chat UI là mã nguồn mở và có thể được điều chỉnh theo nhu cầu ứng dụng của bạn.

!!! tip
    Bạn có thể dùng generative UI trong Agent Chat UI. Để biết thêm thông tin, xem [Implement generative user interfaces with LangGraph](https://docs.langchain.com/langsmith/generative-ui-react).

### Bắt đầu nhanh

Cách nhanh nhất để bắt đầu là dùng bản hosted:

1. **Truy cập [Agent Chat UI](https://agentchat.vercel.app)**
2. **Kết nối agent của bạn** bằng cách nhập URL deploy hoặc địa chỉ server cục bộ
3. **Bắt đầu chat**, giao diện sẽ tự động phát hiện và render tool call cùng interrupt

### Phát triển cục bộ

Để tùy chỉnh hoặc phát triển cục bộ, bạn có thể chạy Agent Chat UI trên máy:

=== "Dùng npx"
    ```bash
    # Tạo một project Agent Chat UI mới
    npx create-agent-chat-app --project-name my-chat-ui
    cd my-chat-ui

    # Cài dependency và khởi động
    pnpm install
    pnpm dev
    ```

=== "Clone repository"
    ```bash
    # Clone repository
    git clone https://github.com/langchain-ai/agent-chat-ui.git
    cd agent-chat-ui

    # Cài dependency và khởi động
    pnpm install
    pnpm dev
    ```

### Kết nối với agent của bạn

Agent Chat UI có thể kết nối với cả [agent cục bộ](studio.md#set-up-local-agent-server) lẫn [agent đã deploy](deploy.md).

Sau khi khởi động Agent Chat UI, bạn cần cấu hình để nó kết nối với agent của mình:

1. **Graph ID**: Nhập tên graph của bạn (tìm trong mục `graphs` của file `langgraph.json`)
2. **Deployment URL**: Endpoint của Agent server (ví dụ: `http://localhost:2024` khi phát triển cục bộ, hoặc URL của agent đã deploy)
3. **LangSmith API key (tùy chọn)**: Thêm LangSmith API key (không bắt buộc nếu bạn đang dùng Agent server cục bộ)

Sau khi cấu hình, Agent Chat UI sẽ tự động fetch và hiển thị mọi thread bị interrupt từ agent của bạn.

!!! tip
    Agent Chat UI hỗ trợ sẵn việc render tool call và message kết quả tool. Để tùy chỉnh message nào được hiển thị, xem [Hiding Messages in the Chat](https://github.com/langchain-ai/agent-chat-ui?tab=readme-ov-file#hiding-messages-in-the-chat).

---

!!! info "Kết nối tài liệu này"
    [Kết nối tài liệu này](https://docs.langchain.com/use-these-docs) với Claude, VSCode, và nhiều công cụ khác qua MCP để có câu trả lời theo thời gian thực.

!!! info "Đóng góp"
    [Chỉnh sửa trang này trên GitHub](https://github.com/langchain-ai/docs/edit/main/src/oss/langgraph/ui.mdx) hoặc [báo lỗi](https://github.com/langchain-ai/docs/issues/new/choose).
