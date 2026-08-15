# Agent Chat UI

[Agent Chat UI](https://github.com/langchain-ai/agent-chat-ui) là một ứng dụng Next.js cung cấp giao diện hội thoại để tương tác với bất kỳ agent LangChain nào. Nó hỗ trợ chat theo thời gian thực, trực quan hoá tool call, và các tính năng nâng cao như time-travel debugging và state forking. Agent Chat UI hoạt động liền mạch với các agent được tạo bằng [`create_agent`](https://reference.langchain.com/python/langchain/agents/factory/create_agent) và mang lại trải nghiệm tương tác cho agent của bạn chỉ với thiết lập tối thiểu, dù bạn đang chạy local hay trong môi trường đã deploy (ví dụ [LangSmith](https://docs.langchain.com/langsmith/observability)).

Agent Chat UI là mã nguồn mở và có thể tuỳ biến theo nhu cầu ứng dụng của bạn.

!!! tip "Mẹo"
    Bạn có thể dùng generative UI trong Agent Chat UI. Xem thêm [Implement generative user interfaces with LangGraph](https://docs.langchain.com/langsmith/generative-ui-react).

### Bắt đầu nhanh

Cách nhanh nhất để bắt đầu là dùng bản hosted:

1. **Truy cập [Agent Chat UI](https://agentchat.vercel.app)**
2. **Kết nối agent của bạn** bằng cách nhập deployment URL hoặc địa chỉ local server
3. **Bắt đầu chat**: giao diện sẽ tự động phát hiện và hiển thị tool call cùng interrupt

### Phát triển local

Để tuỳ biến hoặc phát triển local, bạn có thể chạy Agent Chat UI ngay trên máy:

=== "Dùng npx"

    ```bash
    # Tạo project Agent Chat UI mới
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

### Kết nối tới agent của bạn

Agent Chat UI có thể kết nối tới cả [agent chạy local](studio.md) lẫn [agent đã deploy](deploy.md).

Sau khi khởi động Agent Chat UI, bạn cần cấu hình để kết nối tới agent:

1. **Graph ID**: nhập tên graph (xem trong mục `graphs` của file `langgraph.json`)
2. **Deployment URL**: endpoint của Agent server (ví dụ `http://localhost:2024` khi phát triển local, hoặc URL agent đã deploy)
3. **LangSmith API key (tuỳ chọn)**: thêm API key LangSmith (không bắt buộc nếu bạn dùng Agent server local)

Sau khi cấu hình xong, Agent Chat UI sẽ tự động lấy và hiển thị mọi thread bị interrupt từ agent của bạn.

!!! tip "Mẹo"
    Agent Chat UI hỗ trợ sẵn việc hiển thị tool call và tool result message. Để tuỳ chỉnh message nào được hiển thị, xem [Hiding Messages in the Chat](https://github.com/langchain-ai/agent-chat-ui?tab=readme-ov-file#hiding-messages-in-the-chat).
