# LangSmith Studio

Khi xây dựng agent với LangChain ở môi trường local, sẽ rất hữu ích nếu bạn có thể trực quan hoá những gì đang diễn ra bên trong agent, tương tác với nó theo thời gian thực, và debug lỗi ngay khi chúng xảy ra. **LangSmith Studio** là một giao diện trực quan miễn phí để phát triển và kiểm thử agent LangChain ngay trên máy local của bạn.

Studio kết nối tới agent đang chạy local để hiển thị từng bước agent thực hiện: prompt gửi tới model, tool call cùng kết quả, và output cuối cùng. Bạn có thể thử các input khác nhau, kiểm tra state trung gian, và lặp lại việc tinh chỉnh hành vi agent mà không cần thêm code hay deploy.

Trang này mô tả cách thiết lập Studio với agent LangChain local của bạn.

## Điều kiện tiên quyết

Trước khi bắt đầu, đảm bảo bạn có:

* **Tài khoản LangSmith**: đăng ký (miễn phí) hoặc đăng nhập tại [smith.langchain.com](https://smith.langchain.com?utm_source=docs&utm_medium=cta&utm_campaign=langsmith-signup&utm_content=snippets-oss-studio-py).
* **API key LangSmith**: làm theo hướng dẫn [Create an API key](https://docs.langchain.com/langsmith/create-account-api-key).
* Nếu bạn không muốn dữ liệu bị [trace](https://docs.langchain.com/langsmith/observability-concepts#traces) lên LangSmith, đặt `LANGSMITH_TRACING=false` trong file `.env` của ứng dụng. Khi tắt tracing, không dữ liệu nào rời khỏi local server.

## Thiết lập Agent server local

### 1. Cài đặt LangGraph CLI

[LangGraph CLI](https://docs.langchain.com/langsmith/cli) cung cấp một development server local (còn gọi là [Agent Server](https://docs.langchain.com/langsmith/agent-server)) kết nối agent của bạn với Studio.

=== "pip"

    ```bash
    # Yêu cầu Python >= 3.11.
    pip install -U "langgraph-cli[inmem]"
    ```

=== "uv"

    ```bash
    # Yêu cầu Python >= 3.11.
    uv add "langgraph-cli[inmem]"
    ```

### 2. Chuẩn bị agent

Nếu bạn đã có sẵn agent LangChain, có thể dùng trực tiếp. Ví dụ dưới đây dùng một email agent đơn giản:

```python title="agent.py"
from langchain.agents import create_agent

def send_email(to: str, subject: str, body: str):
    """Send an email"""
    email = {
        "to": to,
        "subject": subject,
        "body": body
    }
    # ... logic gửi email

    return f"Email sent to {to}"

agent = create_agent(
    "gpt-5.5",
    tools=[send_email],
    system_prompt="You are an email assistant. Always use the send_email tool.",
)
```

### 3. Biến môi trường

Studio cần một API key LangSmith để kết nối agent local của bạn. Tạo file `.env` ở thư mục gốc project và thêm API key lấy từ [LangSmith](https://smith.langchain.com/settings).

!!! warning "Cảnh báo"
    Đảm bảo file `.env` không được commit vào hệ thống quản lý phiên bản, ví dụ Git.

```bash title=".env"
LANGSMITH_API_KEY=lsv2...
```

### 4. Tạo file cấu hình LangGraph

LangGraph CLI dùng một file cấu hình để xác định vị trí agent và quản lý dependency. Tạo file `langgraph.json` trong thư mục ứng dụng của bạn:

```json title="langgraph.json"
{
  "dependencies": ["."],
  "graphs": {
    "agent": "./src/agent.py:agent"
  },
  "env": ".env"
}
```

Hàm [`create_agent`](https://reference.langchain.com/python/langchain/agents/factory/create_agent) tự động trả về một LangGraph graph đã compile, đúng như định dạng key `graphs` yêu cầu trong file cấu hình.

!!! info "Thông tin"
    Để biết giải thích chi tiết từng key trong đối tượng JSON của file cấu hình, xem [LangGraph configuration file reference](https://docs.langchain.com/langsmith/cli#configuration-file).

Đến bước này, cấu trúc project sẽ trông như sau:

```bash
my-app/
├── src
│   └── agent.py
├── .env
└── langgraph.json
```

### 5. Cài dependency

Cài dependency của project từ thư mục gốc:

=== "pip"

    ```shell
    pip install langchain langchain-openai
    ```

=== "uv"

    ```shell
    uv add langchain langchain-openai
    ```

### 6. Xem agent trong Studio

Khởi động development server để kết nối agent với Studio:

```shell
langgraph dev
```

!!! warning "Cảnh báo"
    Safari chặn kết nối `localhost` tới Studio. Để khắc phục, chạy lệnh trên kèm `--tunnel` để truy cập Studio qua secure tunnel.

Khi server đã chạy, agent của bạn có thể truy cập cả qua API tại `http://127.0.0.1:2024` lẫn qua giao diện Studio tại `https://smith.langchain.com/studio/?baseUrl=http://127.0.0.1:2024`:

![Agent view in the Studio UI](https://mintcdn.com/langchain-5e9cc07a/TCDks4pdsHdxWmuJ/oss/images/studio_create-agent.png?fit=max&auto=format&n=TCDks4pdsHdxWmuJ&q=85&s=ebd259e9fa24af7d011dfcc568f74be2)

Khi Studio đã kết nối với agent local, bạn có thể nhanh chóng lặp lại việc tinh chỉnh hành vi agent. Chạy thử một input, kiểm tra toàn bộ execution trace bao gồm prompt, tool argument, giá trị trả về, và các chỉ số token/độ trễ. Khi có lỗi xảy ra, Studio ghi lại exception cùng state xung quanh để giúp bạn hiểu chuyện gì đã xảy ra.

Development server hỗ trợ hot-reload: thay đổi prompt hoặc tool signature trong code, Studio sẽ phản ánh ngay lập tức. Chạy lại thread hội thoại từ bất kỳ bước nào để kiểm thử thay đổi mà không cần bắt đầu lại từ đầu. Quy trình này mở rộng tốt từ agent một tool đơn giản đến graph đa node phức tạp.

Để biết thêm về cách chạy Studio, xem các hướng dẫn sau trong [tài liệu LangSmith](https://docs.langchain.com/langsmith/observability):

* [Run application](https://docs.langchain.com/langsmith/use-studio#run-application)
* [Manage assistants](https://docs.langchain.com/langsmith/use-studio#manage-assistants)
* [Manage threads](https://docs.langchain.com/langsmith/use-studio#manage-threads)
* [Iterate on prompts](https://docs.langchain.com/langsmith/observability-studio)
* [Debug LangSmith traces](https://docs.langchain.com/langsmith/observability-studio#debug-langsmith-traces)
* [Add node to dataset](https://docs.langchain.com/langsmith/observability-studio#add-node-to-dataset)

## Video hướng dẫn

Xem video hướng dẫn tại: [https://www.youtube.com/watch?v=Mi1gSlHwZLM](https://www.youtube.com/watch?v=Mi1gSlHwZLM)

!!! tip "Mẹo"
    Để biết thêm về agent đã deploy, xem [Deploy](deploy.md).
