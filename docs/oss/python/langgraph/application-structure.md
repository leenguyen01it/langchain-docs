# Cấu trúc ứng dụng

Một ứng dụng LangGraph gồm một hoặc nhiều graph, một file cấu hình (`langgraph.json`), một file khai báo dependency, và tùy chọn một file `.env` khai báo biến môi trường.

Hướng dẫn này trình bày cấu trúc điển hình của một ứng dụng và cách cung cấp cấu hình cần thiết để deploy ứng dụng bằng [LangSmith Deployment](https://docs.langchain.com/langsmith/deployment).

!!! info "LangSmith Deployment"
    LangSmith Deployment là nền tảng hosting được quản lý để deploy và scale agent LangGraph. Nó xử lý hạ tầng, khả năng mở rộng, và các vấn đề vận hành để bạn có thể deploy agent trạng thái (stateful), chạy dài (long-running) trực tiếp từ repository của mình. Tìm hiểu thêm tại [tài liệu Deployment](https://docs.langchain.com/langsmith/deployment).

## Khái niệm chính

Để deploy bằng LangSmith, bạn cần cung cấp các thông tin sau:

1. Một [file cấu hình LangGraph](#configuration-file-concepts) (`langgraph.json`) khai báo dependency, graph, và biến môi trường dùng cho ứng dụng.
2. Các [graph](#graphs) triển khai logic của ứng dụng.
3. Một file khai báo [dependency](#dependencies) cần thiết để chạy ứng dụng.
4. Các [biến môi trường](#environment-variables) cần thiết để ứng dụng chạy được.

## Cấu trúc file

Dưới đây là ví dụ về cấu trúc thư mục cho ứng dụng:

=== "Python (requirements.txt)"

    ```plaintext
    my-app/
    ├── my_agent # toàn bộ code của project nằm trong đây
    │   ├── utils # các tiện ích cho graph của bạn
    │   │   ├── __init__.py
    │   │   ├── tools.py # các tool cho graph của bạn
    │   │   ├── nodes.py # các hàm node cho graph của bạn
    │   │   └── state.py # định nghĩa state của graph
    │   ├── __init__.py
    │   └── agent.py # code để xây dựng graph của bạn
    ├── .env # biến môi trường
    ├── requirements.txt # dependency của package
    └── langgraph.json # file cấu hình cho LangGraph
    ```

=== "Python (pyproject.toml)"

    ```plaintext
    my-app/
    ├── my_agent # toàn bộ code của project nằm trong đây
    │   ├── utils # các tiện ích cho graph của bạn
    │   │   ├── __init__.py
    │   │   ├── tools.py # các tool cho graph của bạn
    │   │   ├── nodes.py # các hàm node cho graph của bạn
    │   │   └── state.py # định nghĩa state của graph
    │   ├── __init__.py
    │   └── agent.py # code để xây dựng graph của bạn
    ├── .env # biến môi trường
    ├── langgraph.json  # file cấu hình cho LangGraph
    └── pyproject.toml # dependency cho project của bạn
    ```

!!! note
    Cấu trúc thư mục của một ứng dụng LangGraph có thể khác nhau tùy theo ngôn ngữ lập trình và trình quản lý package được sử dụng.

<a id="configuration-file-concepts" />

## File cấu hình

File `langgraph.json` là một file JSON khai báo dependency, graph, biến môi trường, và các thiết lập khác cần thiết để deploy một ứng dụng LangGraph.

Xem [tài liệu tham khảo file cấu hình LangGraph](https://docs.langchain.com/langsmith/cli#configuration-file) để biết chi tiết toàn bộ key được hỗ trợ trong file JSON.

!!! tip
    [LangGraph CLI](https://docs.langchain.com/langsmith/cli) mặc định dùng file cấu hình `langgraph.json` trong thư mục hiện tại.

### Ví dụ

* Dependency gồm một package cục bộ tùy chỉnh và package `langchain_openai`.
* Một graph duy nhất được load từ file `./your_package/your_file.py` với biến `variable`.
* Biến môi trường được load từ file `.env`.

```json
{
  "dependencies": ["langchain_openai", "./your_package"],
  "graphs": {
    "my_agent": "./your_package/your_file.py:agent"
  },
  "env": "./.env"
}
```

## Dependency

Một ứng dụng LangGraph có thể phụ thuộc vào các package Python khác.

Nhìn chung bạn cần khai báo các thông tin sau để dependency được thiết lập đúng:

1. Một file trong thư mục khai báo dependency (ví dụ `requirements.txt`, `pyproject.toml`, hoặc `package.json`).

2. Một key `dependencies` trong [file cấu hình LangGraph](#configuration-file-concepts) khai báo dependency cần thiết để chạy ứng dụng LangGraph.

3. Mọi binary hoặc thư viện hệ thống bổ sung có thể được khai báo bằng key `dockerfile_lines` trong [file cấu hình LangGraph](#configuration-file-concepts).

## Graph

Dùng key `graphs` trong [file cấu hình LangGraph](#configuration-file-concepts) để khai báo những graph nào sẽ có sẵn trong ứng dụng LangGraph đã deploy.

Bạn có thể khai báo một hoặc nhiều graph trong file cấu hình. Mỗi graph được xác định bằng một tên (phải là duy nhất) và một đường dẫn tới: (1) graph đã compile, hoặc (2) một hàm tạo ra graph được định nghĩa.

## Biến môi trường

Nếu bạn làm việc với một ứng dụng LangGraph đã deploy ở local, bạn có thể cấu hình biến môi trường trong key `env` của [file cấu hình LangGraph](#configuration-file-concepts).

Với môi trường production, bạn thường sẽ cấu hình biến môi trường trực tiếp trong môi trường deploy.
