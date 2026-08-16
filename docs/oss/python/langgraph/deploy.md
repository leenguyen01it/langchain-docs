# Deployment

> Deploy agent LangGraph lên production với LangSmith Cloud hoặc các framework/hosting platform JavaScript.

Khi bạn sẵn sàng đưa agent LangGraph lên production, hãy chọn mô hình hosting phù hợp với stack của bạn. **[LangSmith Cloud](https://docs.langchain.com/langsmith/deploy-to-cloud)** cung cấp hạ tầng được quản lý toàn phần (fully managed) cho các agent có trạng thái (stateful), chạy lâu dài, với state được lưu trữ bền vững (persistent) và thực thi nền (background execution).

!!! tip
    LangSmith cung cấp nhiều lựa chọn deployment ngoài Cloud, bao gồm [hybrid](https://docs.langchain.com/langsmith/hybrid), [standalone server](https://docs.langchain.com/langsmith/deploy-standalone-server), và [self-hosted với control plane](https://docs.langchain.com/langsmith/deploy-with-control-plane). Xem thêm tại [tổng quan Deployment của LangSmith](https://docs.langchain.com/langsmith/deployment).

## LangSmith Cloud

Phần này hướng dẫn deploy agent của bạn lên LangSmith Cloud từ một repository GitHub. LangSmith sẽ xử lý hạ tầng, khả năng mở rộng (scaling), và các vấn đề vận hành.

### Điều kiện tiên quyết

Trước khi bắt đầu, hãy đảm bảo bạn có:

* Một [tài khoản GitHub](https://github.com/)
* Một [tài khoản LangSmith](https://smith.langchain.com) (đăng ký miễn phí)

### Deploy agent của bạn

#### 1. Tạo repository trên GitHub

Code ứng dụng của bạn phải nằm trong một repository GitHub để có thể deploy trên LangSmith. Cả repository công khai (public) và riêng tư (private) đều được hỗ trợ. Với hướng dẫn nhanh này, trước tiên hãy đảm bảo app của bạn tương thích với LangGraph bằng cách làm theo [hướng dẫn thiết lập local agent server](studio.md#set-up-local-agent-server). Sau đó, push code lên repository.

#### 2. Deploy lên LangSmith

1. **Vào mục Deployment của LangSmith**

   Đăng nhập vào [LangSmith](https://smith.langchain.com). Trong sidebar bên trái, chọn **Deployments**.

2. **Tạo deployment mới**

   Nhấp nút **+ New Deployment**. Một khung sẽ mở ra để bạn điền các trường bắt buộc.

3. **Liên kết repository**

   Nếu bạn là người dùng lần đầu hoặc đang thêm một repository riêng tư chưa được kết nối trước đó, nhấp nút **Add new account** và làm theo hướng dẫn để kết nối tài khoản GitHub.

4. **Deploy repository**

   Chọn repository ứng dụng của bạn. Nhấp **Submit** để deploy. Quá trình này có thể mất khoảng 15 phút để hoàn tất. Bạn có thể kiểm tra trạng thái trong màn hình **Deployment details**.

#### 3. Kiểm thử ứng dụng trong Studio

Sau khi ứng dụng đã được deploy:

1. Chọn deployment bạn vừa tạo để xem thêm chi tiết.
2. Nhấp nút **Studio** ở góc trên bên phải. Studio sẽ mở ra và hiển thị graph của bạn.

#### 4. Lấy API URL cho deployment của bạn

1. Trong màn hình **Deployment details** của LangGraph, nhấp **API URL** để copy vào clipboard.
2. Nhấp vào `URL` để copy vào clipboard.

#### 5. Kiểm thử API

Bây giờ bạn có thể kiểm thử API:

=== "Python"
    1. Cài đặt LangGraph SDK:

    ```shell
    pip install langgraph-sdk
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
