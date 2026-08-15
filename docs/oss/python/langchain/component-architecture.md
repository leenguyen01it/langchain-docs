# Kiến trúc component

Sức mạnh của LangChain đến từ cách các component của nó phối hợp với nhau để tạo ra các ứng dụng AI phức tạp. Trang này cung cấp các sơ đồ minh hoạ mối quan hệ giữa các component khác nhau.

## Hệ sinh thái component cốt lõi

Sơ đồ dưới đây cho thấy các component chính của LangChain kết nối với nhau như thế nào để tạo thành các ứng dụng AI hoàn chỉnh:

```mermaid
graph TD
    %% Input processing
    subgraph "📥 Input processing"
        A[Text input] --> B[Document loaders]
        B --> C[Text splitters]
        C --> D[Documents]
    end

    %% Embedding & storage
    subgraph "🔢 Embedding & storage"
        D --> E[Embedding models]
        E --> F[Vectors]
        F --> G[(Vector stores)]
    end

    %% Retrieval
    subgraph "🔍 Retrieval"
        H[User Query] --> I[Embedding models]
        I --> J[Query vector]
        J --> K[Retrievers]
        K --> G
        G --> L[Relevant context]
    end

    %% Generation
    subgraph "🤖 Generation"
        M[Chat models] --> N[Tools]
        N --> O[Tool results]
        O --> M
        L --> M
        M --> P[AI response]
    end

    %% Orchestration
    subgraph "🎯 Orchestration"
        Q[Agents] --> M
        Q --> N
        Q --> K
        Q --> R[Memory]
    end

    classDef trigger fill:#F6FFDB,stroke:#6E8900,stroke-width:2px,color:#2E3900
    classDef process fill:#E5F4FF,stroke:#006DDD,stroke-width:2px,color:#030710
    classDef output fill:#EBD0F0,stroke:#885270,stroke-width:2px,color:#441E33
    classDef neutral fill:#F2FAFF,stroke:#40668D,stroke-width:2px,color:#2F4B68

    class A,H trigger
    class B,C,E,I,K,M,N,Q process
    class D,F,J,L,O,P,R neutral
    class G output
```

### Các component kết nối với nhau như thế nào

Mỗi lớp component xây dựng dựa trên các lớp trước đó:

1. **Input processing (xử lý input)** – Chuyển đổi dữ liệu thô thành document có cấu trúc
2. **Embedding & storage (embedding và lưu trữ)** – Chuyển văn bản thành các biểu diễn vector có thể tìm kiếm
3. **Retrieval (truy xuất)** – Tìm thông tin liên quan dựa trên truy vấn của người dùng
4. **Generation (sinh nội dung)** – Dùng AI model để tạo ra phản hồi, có thể kèm tool
5. **Orchestration (điều phối)** – Điều phối toàn bộ thông qua agent và hệ thống memory

## Các nhóm component

LangChain tổ chức các component thành những nhóm chính sau:

| Nhóm                                                                 | Mục đích                    | Component chính                     | Trường hợp sử dụng                                 |
| --------------------------------------------------------------------- | --------------------------- | ------------------------------------ | ---------------------------------------------------- |
| **[Models](https://docs.langchain.com/oss/python/langchain/models)** | Suy luận và sinh nội dung AI | Chat models, LLM, Embedding models  | Sinh văn bản, suy luận, hiểu ngữ nghĩa               |
| **[Tools](https://docs.langchain.com/oss/python/langchain/tools)**   | Khả năng bên ngoài          | API, database, v.v.                 | Tìm kiếm web, truy cập dữ liệu, tính toán            |
| **[Agents](agents.md)**                                               | Điều phối và suy luận       | ReAct agent, tool calling agent     | Workflow phi xác định (nondeterministic), ra quyết định |
| **[Memory](short-term-memory.md)**                                    | Duy trì ngữ cảnh            | Lịch sử message, state tuỳ chỉnh    | Hội thoại, tương tác có trạng thái                   |
| **[Retrievers](https://docs.langchain.com/oss/python/integrations/retrievers)** | Truy cập thông tin | Vector retriever, web retriever     | RAG, tìm kiếm knowledge base                         |
| **[Xử lý document](https://docs.langchain.com/oss/python/integrations/document_loaders)** | Nạp dữ liệu       | Loader, splitter, transformer       | Xử lý PDF, thu thập dữ liệu web (web scraping)       |
| **[Vector Stores](https://docs.langchain.com/oss/python/integrations/vectorstores)** | Tìm kiếm ngữ nghĩa | Chroma, Pinecone, FAISS             | Tìm kiếm tương đồng, lưu trữ embedding               |

## Các pattern phổ biến

### RAG (Retrieval-Augmented Generation)

```mermaid
graph LR
    A[User question] --> B[Retriever]
    B --> C[Relevant docs]
    C --> D[Chat model]
    A --> D
    D --> E[Informed response]

    classDef trigger fill:#F6FFDB,stroke:#6E8900,stroke-width:2px,color:#2E3900
    classDef process fill:#E5F4FF,stroke:#006DDD,stroke-width:2px,color:#030710
    classDef neutral fill:#F2FAFF,stroke:#40668D,stroke-width:2px,color:#2F4B68

    class A trigger
    class B,D process
    class C,E neutral
```

### Agent kèm tool

```mermaid
graph LR
    A[User request] --> B[Agent]
    B --> C{Need tool?}
    C -->|Yes| D[Call tool]
    D --> E[Tool result]
    E --> B
    C -->|No| F[Final answer]

    classDef trigger fill:#F6FFDB,stroke:#6E8900,stroke-width:2px,color:#2E3900
    classDef process fill:#E5F4FF,stroke:#006DDD,stroke-width:2px,color:#030710
    classDef decision fill:#FDF3FF,stroke:#7E65AE,stroke-width:2px,color:#504B5F
    classDef neutral fill:#F2FAFF,stroke:#40668D,stroke-width:2px,color:#2F4B68

    class A trigger
    class B,D process
    class C decision
    class E,F neutral
```

### Hệ thống multi-agent

```mermaid
graph LR
    A[Complex Task] --> B[Supervisor agent]
    B --> C[Specialist agent 1]
    B --> D[Specialist agent 2]
    C --> E[Results]
    D --> E
    E --> B
    B --> F[Coordinated response]

    classDef trigger fill:#F6FFDB,stroke:#6E8900,stroke-width:2px,color:#2E3900
    classDef process fill:#E5F4FF,stroke:#006DDD,stroke-width:2px,color:#030710
    classDef neutral fill:#F2FAFF,stroke:#40668D,stroke-width:2px,color:#2F4B68

    class A trigger
    class B,C,D process
    class E,F neutral
```

## Tìm hiểu thêm

* [Tạo agent](agents.md)
* [Làm việc với tools](https://docs.langchain.com/oss/python/langchain/tools)
* [Khám phá các integration](https://docs.langchain.com/oss/python/integrations/providers/overview)
