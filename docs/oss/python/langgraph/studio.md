# LangSmith Studio

Khi xây dựng agent với LangChain trên máy cục bộ, việc trực quan hoá những gì đang diễn ra bên trong agent, tương tác với nó theo thời gian thực và debug vấn đề ngay khi chúng xảy ra rất hữu ích. **LangSmith Studio** là một giao diện trực quan miễn phí để phát triển và kiểm thử agent LangChain của bạn ngay trên máy cục bộ.

Studio kết nối tới agent đang chạy cục bộ của bạn để hiển thị từng bước agent thực hiện: các prompt gửi tới model, các lệnh gọi tool và kết quả của chúng, cùng output cuối cùng. Bạn có thể thử nhiều input khác nhau, kiểm tra các trạng thái trung gian và cải tiến hành vi của agent mà không cần thêm code hay deploy.

Trang này mô tả cách thiết lập Studio với agent LangChain cục bộ của bạn.

## Điều kiện tiên quyết

Trước khi bắt đầu, hãy đảm bảo bạn có:

* **Một tài khoản LangSmith**: Đăng ký (miễn phí) hoặc đăng nhập tại [smith.langchain.com](https://smith.langchain.com?utm_source=docs&utm_medium=cta&utm_campaign=langsmith-signup&utm_content=oss-langgraph-studio).
* **Một LangSmith API key**: Làm theo hướng dẫn [Create an API key](https://docs.langchain.com/langsmith/create-account-api-key).
* Nếu bạn không muốn dữ liệu được [trace](https://docs.langchain.com/langsmith/observability-concepts#traces) tới LangSmith, hãy đặt `LANGSMITH_TRACING=false` trong file `.env` của ứng dụng. Khi tắt tracing, không có dữ liệu nào rời khỏi server cục bộ của bạn.

## Thiết lập Agent server cục bộ

### 1. Cài đặt LangGraph CLI

[LangGraph CLI](https://docs.langchain.com/langsmith/cli) cung cấp một development server cục bộ (còn gọi là [Agent Server](https://docs.langchain.com/langsmith/agent-server)) kết nối agent của bạn với Studio.

```shell
# Yêu cầu Python >= 3.11.
pip install --upgrade "langgraph-cli[inmem]"
```

### 2. Chuẩn bị agent

Nếu bạn đã có sẵn một agent LangChain, bạn có thể dùng trực tiếp. Ví dụ này dùng một agent email đơn giản:

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

Studio yêu cầu một LangSmith API key để kết nối với agent cục bộ của bạn. Tạo file `.env` ở thư mục gốc dự án và thêm API key của bạn từ [LangSmith](https://smith.langchain.com/settings).

!!! warning
    Đảm bảo file `.env` của bạn không được commit vào hệ thống quản lý phiên bản, ví dụ như Git.

```bash .env
LANGSMITH_API_KEY=lsv2...
```

### 4. Tạo file cấu hình LangGraph

LangGraph CLI dùng một file cấu hình để xác định vị trí agent của bạn và quản lý dependency. Tạo file `langgraph.json` trong thư mục ứng dụng của bạn:

```json title="langgraph.json"
{
  "dependencies": ["."],
  "graphs": {
    "agent": "./src/agent.py:agent"
  },
  "env": ".env"
}
```

Hàm [`create_agent`](https://reference.langchain.com/python/langchain/agents/factory/create_agent) tự động trả về một graph LangGraph đã được compile, đúng với những gì key `graphs` yêu cầu trong file cấu hình.

!!! info
    Để biết giải thích chi tiết từng key trong đối tượng JSON của file cấu hình, tham khảo [tài liệu tham chiếu file cấu hình LangGraph](https://docs.langchain.com/langsmith/cli#configuration-file).

Đến bước này, cấu trúc dự án sẽ trông như sau:

```bash
my-app/
├── src
│   └── agent.py
├── .env
└── langgraph.json
```

### 5. Cài đặt dependency

Cài đặt các dependency của dự án từ thư mục gốc:

=== "pip"
    ```shell
    pip install langchain langchain-openai
    ```

=== "uv"
    ```shell
    uv add langchain langchain-openai
    ```

### 6. Xem agent của bạn trong Studio

Khởi động development server để kết nối agent của bạn với Studio:

```shell
langgraph dev
```

!!! warning
    Safari chặn các kết nối `localhost` tới Studio. Để khắc phục, chạy lệnh trên với `--tunnel` để truy cập Studio qua một tunnel bảo mật. Bạn cần thêm thủ công URL tunnel vào danh sách origin được phép bằng cách bấm **Connect to a local server** trong giao diện Studio. Xem [hướng dẫn khắc phục sự cố](https://docs.langchain.com/langsmith/troubleshooting-studio#safari-connection-issues) để biết các bước thực hiện.

Khi server đã chạy, agent của bạn có thể truy cập cả qua API tại `http://127.0.0.1:2024` lẫn qua giao diện Studio tại `https://smith.langchain.com/studio/?baseUrl=http://127.0.0.1:2024`:

![Agent view in the Studio UI](https://mintcdn.com/langchain-5e9cc07a/TCDks4pdsHdxWmuJ/oss/images/studio_create-agent.png?fit=max&auto=format&n=TCDks4pdsHdxWmuJ&q=85&s=ebd259e9fa24af7d011dfcc568f74be2)

Khi Studio đã kết nối với agent cục bộ của bạn, bạn có thể cải tiến nhanh hành vi của agent. Chạy một input thử nghiệm, kiểm tra toàn bộ execution trace bao gồm prompt, tham số tool, giá trị trả về và các số liệu token/độ trễ trong [LangSmith](https://docs.langchain.com/langsmith/observability-studio). Khi có lỗi xảy ra, Studio ghi lại exception cùng trạng thái xung quanh để giúp bạn hiểu chuyện gì đã xảy ra.

Development server hỗ trợ hot-reload, chỉ cần thay đổi prompt hoặc chữ ký (signature) của tool trong code, Studio sẽ phản ánh ngay lập tức. Bạn có thể chạy lại các thread hội thoại từ bất kỳ bước nào để kiểm thử thay đổi mà không cần bắt đầu lại từ đầu. Quy trình làm việc này mở rộng được từ các agent đơn giản một tool cho tới các graph đa node phức tạp.

Để biết thêm thông tin về cách chạy Studio, tham khảo các hướng dẫn sau trong [tài liệu LangSmith](https://docs.langchain.com/langsmith/observability):

* [Chạy ứng dụng](https://docs.langchain.com/langsmith/use-studio#run-application)
* [Quản lý assistant](https://docs.langchain.com/langsmith/use-studio#manage-assistants)
* [Quản lý thread](https://docs.langchain.com/langsmith/use-studio#manage-threads)
* [Cải tiến prompt](https://docs.langchain.com/langsmith/observability-studio)
* [Debug LangSmith trace](https://docs.langchain.com/langsmith/observability-studio#debug-langsmith-traces)
* [Thêm node vào dataset](https://docs.langchain.com/langsmith/observability-studio#add-node-to-dataset)

## Video hướng dẫn

Xem video hướng dẫn tại: [https://www.youtube.com/embed/Mi1gSlHwZLM](https://www.youtube.com/embed/Mi1gSlHwZLM)
