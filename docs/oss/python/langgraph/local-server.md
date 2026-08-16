# Chạy server cục bộ

Hướng dẫn này chỉ cho bạn cách chạy một ứng dụng LangGraph trên máy cục bộ.

## Yêu cầu tiên quyết

Trước khi bắt đầu, hãy đảm bảo bạn có:

* Một API key cho [LangSmith](https://smith.langchain.com/settings), miễn phí đăng ký

## 1. Cài đặt LangGraph CLI

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

## 2. Tạo một ứng dụng LangGraph

Tạo một ứng dụng mới từ [template `new-langgraph-project-python`](https://github.com/langchain-ai/new-langgraph-project). Template này minh họa một ứng dụng một node mà bạn có thể mở rộng bằng logic riêng.

```shell
langgraph new path/to/your/app --template new-langgraph-project-python
```

!!! tip "Các template khác"
    Nếu bạn dùng `langgraph new` mà không chỉ định template, bạn sẽ thấy một menu tương tác cho phép chọn từ danh sách template có sẵn.

## 3. Cài dependency

Ở thư mục gốc của ứng dụng LangGraph mới, cài dependency ở chế độ `edit` để server dùng đúng các thay đổi cục bộ của bạn:

=== "pip"
    ```bash
    cd path/to/your/app
    pip install -e .
    ```

=== "uv"
    ```bash
    cd path/to/your/app
    uv sync
    ```

## 4. Tạo file `.env`

Bạn sẽ thấy một file `.env.example` ở thư mục gốc của ứng dụng LangGraph mới. Tạo một file `.env` ở thư mục gốc và copy nội dung của `.env.example` vào đó, điền các API key cần thiết:

```bash
LANGSMITH_API_KEY=lsv2...
```

## 5. Khởi động Agent server

Khởi động LangGraph API server trên máy cục bộ:

```shell
langgraph dev
```

Ví dụ output:

```
INFO:langgraph_api.cli:

        Welcome to

╦  ┌─┐┌┐┌┌─┐╔═╗┬─┐┌─┐┌─┐┬ ┬
║  ├─┤││││ ┬║ ╦├┬┘├─┤├─┘├─┤
╩═╝┴ ┴┘└┘└─┘╚═╝┴└─┴ ┴┴  ┴ ┴

- 🚀 API: http://127.0.0.1:2024
- 🎨 Studio UI: https://smith.langchain.com/studio/?baseUrl=http://127.0.0.1:2024
- 📚 API Docs: http://127.0.0.1:2024/docs

This in-memory server is designed for development and testing.
For production use, please use LangSmith Deployment.
```

Lệnh `langgraph dev` khởi động Agent Server ở chế độ in-memory. Chế độ này phù hợp cho mục đích phát triển và kiểm thử. Với môi trường production, hãy deploy Agent Server với quyền truy cập vào một backend lưu trữ bền vững (persistent storage). Xem thêm tại [Platform setup overview](https://docs.langchain.com/langsmith/platform-setup).

## 6. Kiểm thử ứng dụng trong Studio

[Studio](https://docs.langchain.com/langsmith/studio) là một UI chuyên dụng mà bạn có thể kết nối với LangGraph API server để trực quan hóa, tương tác, và debug ứng dụng cục bộ. Kiểm thử graph trong Studio bằng cách truy cập URL được cung cấp trong output của lệnh `langgraph dev`:

```
>    - LangGraph Studio Web UI: https://smith.langchain.com/studio/?baseUrl=http://127.0.0.1:2024
```

Với Agent Server chạy trên host/port tùy chỉnh, cập nhật query parameter `baseUrl` trong URL. Ví dụ, nếu server chạy trên `http://myhost:3000`:

```
https://smith.langchain.com/studio/?baseUrl=http://myhost:3000
```

??? note "Khả năng tương thích với Safari"
    Dùng flag `--tunnel` kèm lệnh của bạn để tạo một tunnel an toàn, vì Safari có một số hạn chế khi kết nối tới server localhost:

    ```shell
    langgraph dev --tunnel
    ```

## 7. Kiểm thử API

=== "Python SDK (async)"
    1. Cài LangGraph Python SDK:
       ```shell
       pip install langgraph-sdk
       ```
    2. Gửi một message tới assistant (threadless run):
       ```python
       from langgraph_sdk import get_client
       import asyncio

       client = get_client(url="http://localhost:2024")

       async def main():
           async for chunk in client.runs.stream(
               None,  # Threadless run
               "agent", # Tên assistant. Định nghĩa trong langgraph.json.
               input={
               "messages": [{
                   "role": "human",
                   "content": "What is LangGraph?",
                   }],
               },
           ):
               print(f"Receiving new event of type: {chunk.event}...")
               print(chunk.data)
               print("\n\n")

       asyncio.run(main())
       ```

=== "Python SDK (sync)"
    1. Cài LangGraph Python SDK:
       ```shell
       pip install langgraph-sdk
       ```
    2. Gửi một message tới assistant (threadless run):
       ```python
       from langgraph_sdk import get_sync_client

       client = get_sync_client(url="http://localhost:2024")

       for chunk in client.runs.stream(
           None,  # Threadless run
           "agent", # Tên assistant. Định nghĩa trong langgraph.json.
           input={
               "messages": [{
                   "role": "human",
                   "content": "What is LangGraph?",
               }],
           },
           stream_mode="messages-tuple",
       ):
           print(f"Receiving new event of type: {chunk.event}...")
           print(chunk.data)
           print("\n\n")
       ```

=== "Rest API"
    ```bash
    curl -s --request POST \
        --url "http://localhost:2024/runs/stream" \
        --header 'Content-Type: application/json' \
        --data "{
            \"assistant_id\": \"agent\",
            \"input\": {
                \"messages\": [
                    {
                        \"role\": \"human\",
                        \"content\": \"What is LangGraph?\"
                    }
                ]
            },
            \"stream_mode\": \"messages-tuple\"
        }"
    ```

## Bước tiếp theo

Giờ bạn đã có một ứng dụng LangGraph chạy cục bộ, hãy tiếp tục hành trình bằng cách khám phá việc deploy và các tính năng nâng cao:

* [Deployment quickstart](https://docs.langchain.com/langsmith/deployment-quickstart): Deploy ứng dụng LangGraph bằng LangSmith.

* [LangSmith](https://docs.langchain.com/langsmith/observability): Tìm hiểu các khái niệm nền tảng của LangSmith.

* [SDK Reference](https://reference.langchain.com/python/langsmith/deployment/sdk/): Khám phá SDK API Reference.

---

!!! info "Kết nối tài liệu này"
    [Kết nối tài liệu này](https://docs.langchain.com/use-these-docs) với Claude, VSCode, và nhiều công cụ khác qua MCP để có câu trả lời theo thời gian thực.

!!! info "Đóng góp"
    [Chỉnh sửa trang này trên GitHub](https://github.com/langchain-ai/docs/edit/main/src/oss/langgraph/local-server.mdx) hoặc [báo lỗi](https://github.com/langchain-ai/docs/issues/new/choose).
