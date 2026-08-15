# Triển khai

> Triển khai agent LangChain lên production với LangSmith Cloud hoặc các framework JavaScript và nền tảng hosting.

Khi đã sẵn sàng triển khai agent LangChain lên production, hãy chọn mô hình hosting phù hợp với stack của bạn. **[LangSmith Cloud](https://docs.langchain.com/langsmith/deploy-to-cloud)** cung cấp hạ tầng được quản lý hoàn toàn (fully managed) cho các agent stateful, chạy dài hạn với trạng thái được lưu trữ bền vững (persistent state) và thực thi nền (background execution).

!!! tip "Mẹo"
    LangSmith cung cấp nhiều tuỳ chọn triển khai khác ngoài Cloud, bao gồm [hybrid](https://docs.langchain.com/langsmith/hybrid), [standalone servers](https://docs.langchain.com/langsmith/deploy-standalone-server), và [self-hosted kèm control plane](https://docs.langchain.com/langsmith/deploy-with-control-plane). Để biết thêm thông tin, xem [tổng quan triển khai LangSmith](https://docs.langchain.com/langsmith/deployment).

## LangSmith Cloud

Phần này hướng dẫn cách triển khai agent của bạn lên LangSmith Cloud từ một GitHub repository. LangSmith xử lý hạ tầng, khả năng mở rộng, và các vấn đề vận hành.

### Yêu cầu tiên quyết

Trước khi bắt đầu, hãy đảm bảo bạn có:

* Một [tài khoản GitHub](https://github.com/)
* Một [tài khoản LangSmith](https://smith.langchain.com?utm_source=docs&utm_medium=cta&utm_campaign=langsmith-signup&utm_content=oss-langchain-deploy) (đăng ký miễn phí)

### Triển khai agent của bạn

#### 1. Tạo một repository trên GitHub

Code ứng dụng của bạn phải nằm trong một GitHub repository để có thể triển khai trên LangSmith. Cả repository public và private đều được hỗ trợ. Với hướng dẫn nhanh này, trước tiên hãy đảm bảo ứng dụng của bạn tương thích với LangGraph bằng cách làm theo [hướng dẫn thiết lập local server](studio.md). Sau đó, push code của bạn lên repository.

#### 2. Triển khai lên LangSmith

1. **Truy cập trang Deployment của LangSmith**

    Đăng nhập vào [LangSmith](https://smith.langchain.com?utm_source=docs&utm_medium=cta&utm_campaign=langsmith-signup&utm_content=snippets-oss-deploy-py). Trong sidebar bên trái, chọn **Deployments**.

2. **Tạo deployment mới**

    Nhấn nút **+ New Deployment**. Một panel sẽ mở ra để bạn điền các trường bắt buộc.

3. **Liên kết repository**

    Nếu bạn là người dùng lần đầu hoặc đang thêm một private repository chưa từng kết nối trước đó, hãy nhấn nút **Add new account** và làm theo hướng dẫn để kết nối tài khoản GitHub của bạn.

4. **Triển khai repository**

    Chọn repository ứng dụng của bạn. Nhấn **Submit** để triển khai. Quá trình này có thể mất khoảng 15 phút để hoàn tất. Bạn có thể kiểm tra trạng thái trong màn hình **Deployment details**.

#### 3. Kiểm tra ứng dụng trong Studio

Sau khi ứng dụng được triển khai:

1. Chọn deployment bạn vừa tạo để xem thêm chi tiết.
2. Nhấn nút **Studio** ở góc trên bên phải. Studio sẽ mở ra để hiển thị graph của bạn.

#### 4. Lấy API URL cho deployment của bạn

1. Trong màn hình **Deployment details** ở LangGraph, nhấn **API URL** để copy vào clipboard.
2. Nhấn `URL` để copy vào clipboard.

#### 5. Kiểm tra API

Giờ bạn có thể kiểm tra API:

=== "Python"

    1. Cài đặt LangGraph Python:

    === "pip"

        ```bash
        pip install -U langgraph-sdk
        ```

    === "uv"

        ```bash
        uv add langgraph-sdk
        ```

    2. Gửi một message tới agent:

    ```python
    from langgraph_sdk import get_sync_client # hoặc get_client cho async

    client = get_sync_client(url="your-deployment-url", api_key="your-langsmith-api-key")

    for chunk in client.runs.stream(
        None,    # Run không có thread
        "agent", # Tên agent. Được định nghĩa trong langgraph.json.
        input={
            "messages": [{
                "role": "human",
                "content": "What is LangGraph?",
            }],
        },
        stream_mode="updates",
    ):
        print(f"Receiving new event of type: {chunk.event}...")
        print(chunk.data)
        print("\n\n")
    ```

=== "Rest API"

    ```bash
    curl -s --request POST \
        --url <DEPLOYMENT_URL>/runs/stream \
        --header 'Content-Type: application/json' \
        --header "X-Api-Key: <LANGSMITH API KEY> \
        --data "{
            \"assistant_id\": \"agent\", `# Tên agent. Được định nghĩa trong langgraph.json.`
            \"input\": {
                \"messages\": [
                    {
                        \"role\": \"human\",
                        \"content\": \"What is LangGraph?\"
                    }
                ]
            },
            \"stream_mode\": \"updates\"
        }"
    ```

!!! tip "Mẹo"
    LangSmith cung cấp thêm các tuỳ chọn hosting khác, bao gồm self-hosted và hybrid. Để biết thêm thông tin, xem [tổng quan thiết lập Platform](https://docs.langchain.com/langsmith/platform-setup).
