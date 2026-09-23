# Bài 3: Embeddings và Vector Database

!!! abstract "Tóm tắt bài học"
    - **Mục tiêu:** chọn model embedding bằng số liệu thay vì cảm tính, hiểu vector search hoạt động bên dưới, và chuyển từ `InMemoryVectorStore` sang một vector database thật.
    - **Thời lượng:** khoảng 1 tuần.
    - **Yêu cầu trước:** [Bài 2](02-ingestion-chunking.md), đã cài Docker.

## 1. Embedding model hoạt động thế nào?

Model embedding dùng cho RAG thường là **bi-encoder**: một mạng Transformer đọc đoạn văn và nén toàn bộ ý nghĩa thành **một vector** có độ dài cố định. Model được huấn luyện bằng *contrastive learning*: kéo vector của cặp (câu hỏi, đoạn trả lời đúng) lại gần nhau, đẩy các cặp không liên quan ra xa.

Hệ quả thực tế:

- Câu hỏi và tài liệu được embed **độc lập**, nên tài liệu có thể embed trước một lần và lưu lại. Đây là lý do vector search nhanh.
- Nhưng cũng vì độc lập nên độ chính xác có giới hạn. [Bài 4](04-retrieval-nang-cao.md) sẽ dùng *cross-encoder* (reranker) đọc câu hỏi và tài liệu cùng lúc để bù lại.

### Các thông số cần để ý

| Thông số | Ý nghĩa | Ảnh hưởng |
|---|---|---|
| Số chiều (dimensions) | Độ dài vector, ví dụ 384, 1024, 1536, 3072 | Nhiều chiều thì biểu diễn tốt hơn nhưng tốn bộ nhớ, tìm chậm hơn |
| Max input tokens | Độ dài tối đa model đọc được, phần thừa bị **cắt bỏ âm thầm** | Chunk dài hơn giới hạn sẽ mất nội dung cuối |
| Đa ngôn ngữ | Có được huấn luyện trên tiếng Việt không | Quyết định chất lượng với dữ liệu tiếng Việt |
| Bất đối xứng (asymmetric) | Một số model yêu cầu tiền tố khác nhau cho câu hỏi và tài liệu | Quên tiền tố làm giảm chất lượng rõ rệt |
| Matryoshka | Có thể cắt ngắn vector mà vẫn giữ phần lớn chất lượng | Tiết kiệm bộ nhớ khi dữ liệu lớn |

Ví dụ về model bất đối xứng: dòng `multilingual-e5` yêu cầu thêm `"query: "` trước câu hỏi và `"passage: "` trước tài liệu.

## 2. Chọn model embedding

### 2.1 Các lựa chọn phổ biến

| Model | Loại | Số chiều | Ghi chú |
|---|---|---|---|
| OpenAI `text-embedding-3-small` | API | 1536 (cắt được) | Rẻ, đa ngôn ngữ khá, dễ bắt đầu |
| OpenAI `text-embedding-3-large` | API | 3072 (cắt được) | Chất lượng cao hơn, đắt hơn |
| Cohere `embed-multilingual` | API | 1024 | Mạnh với đa ngôn ngữ |
| Voyage AI | API | Tùy model | Có model chuyên biệt cho code, luật, tài chính |
| `BAAI/bge-m3` | Mã nguồn mở | 1024 | Đa ngôn ngữ, input tới 8192 token, hỗ trợ cả dense và sparse |
| `intfloat/multilingual-e5-large` | Mã nguồn mở | 1024 | Cần tiền tố `query:` / `passage:` |

Bảng xếp hạng [MTEB](https://huggingface.co/spaces/mteb/leaderboard) là điểm khởi đầu tốt để chọn ứng viên, nhưng **điểm benchmark không thay được việc đo trên dữ liệu của bạn**, đặc biệt với tiếng Việt và thuật ngữ chuyên ngành.

!!! warning "Không trộn embedding"
    Câu hỏi và tài liệu **phải** được embed bằng cùng một model (và cùng số chiều). Đổi model đồng nghĩa với việc index lại toàn bộ dữ liệu. Hãy lưu tên model vào metadata hoặc tên collection để tránh nhầm lẫn.

### 2.2 Tự đánh giá model trên dữ liệu của bạn

Tạo file `eval/questions.jsonl` với các câu hỏi baseline, kèm một **từ khóa** chắc chắn phải xuất hiện trong chunk đúng. Đây là cách gán nhãn đơn giản nhất; [Bài 6](06-evaluation.md) sẽ làm kỹ hơn.

```json title="eval/questions.jsonl"
{"question": "Nhân viên được nghỉ phép bao nhiêu ngày mỗi năm?", "keyword": "12 ngày"}
{"question": "Thử việc kéo dài bao lâu?", "keyword": "60 ngày"}
{"question": "Công ty hỗ trợ gửi xe không?", "keyword": "gửi xe"}
```

```bash
uv add langchain-huggingface sentence-transformers
```

```python title="compare_embeddings.py"
import json
import time

from langchain.embeddings import init_embeddings
from langchain_core.vectorstores import InMemoryVectorStore
from langchain_huggingface import HuggingFaceEmbeddings

from rag.ingest import DATA_DIR, chunk_file

chunks = [c for p in sorted(DATA_DIR.glob("*.pdf")) for c in chunk_file(p)]
questions = [json.loads(line) for line in open("eval/questions.jsonl", encoding="utf-8")]

candidates = {
    "openai-3-small": init_embeddings("openai:text-embedding-3-small"),
    "bge-m3": HuggingFaceEmbeddings(
        model_name="BAAI/bge-m3", encode_kwargs={"normalize_embeddings": True}
    ),
}


def hit_rate(store: InMemoryVectorStore, k: int) -> float:
    hits = 0
    for q in questions:
        docs = store.similarity_search(q["question"], k=k)
        hits += any(q["keyword"].lower() in d.page_content.lower() for d in docs)
    return hits / len(questions)


for name, embeddings in candidates.items():
    start = time.perf_counter()
    store = InMemoryVectorStore(embeddings)
    store.add_documents(chunks)
    elapsed = time.perf_counter() - start
    print(
        f"{name:16} hit@1={hit_rate(store, 1):.2f} "
        f"hit@5={hit_rate(store, 5):.2f} index={elapsed:.0f}s"
    )
```

!!! tip "Chạy model mã nguồn mở"
    `bge-m3` khá nặng (khoảng 2 GB). Có GPU thì nhanh, chạy CPU vẫn được với vài trăm chunk. Lần chạy đầu sẽ tải model từ Hugging Face.

### 2.3 Chi phí

Chi phí embedding thường **nhỏ** so với chi phí sinh câu trả lời, vì mỗi chunk chỉ embed một lần. Ví dụ: 10.000 trang, khoảng 5 triệu token, với model giá vài cent cho mỗi triệu token thì tổng chỉ vài chục cent đến vài đô la. Chi phí thật sự nằm ở việc **index lại** khi đổi model, nên hãy chọn kỹ từ đầu.

## 3. Vector search bên dưới

### 3.1 Tìm chính xác (flat search)

Cách đơn giản nhất: tính similarity giữa câu hỏi và **mọi** vector, rồi lấy top k. `InMemoryVectorStore` làm đúng như vậy. Với 1 triệu vector 1536 chiều, mỗi truy vấn cần khoảng 1,5 tỉ phép nhân, quá chậm cho production.

### 3.2 Tìm gần đúng (ANN) với HNSW

**ANN (Approximate Nearest Neighbor)** chấp nhận đôi khi bỏ sót một kết quả để đổi lấy tốc độ nhanh hơn hàng trăm lần. Thuật toán phổ biến nhất là **HNSW (Hierarchical Navigable Small World)**:

- Các vector được nối thành một **đồ thị**, mỗi điểm nối với vài "hàng xóm" gần nhất.
- Đồ thị có **nhiều tầng**: tầng trên thưa (như đường cao tốc), tầng dưới dày (như đường nội thành).
- Khi tìm, bắt đầu từ tầng trên cùng, đi dần về phía câu hỏi, rồi xuống tầng dưới để tinh chỉnh. Giống như đi từ cao tốc xuống quốc lộ rồi vào hẻm.

| Tham số | Ý nghĩa | Tăng lên thì |
|---|---|---|
| `M` | Số hàng xóm của mỗi điểm | Chính xác hơn, tốn bộ nhớ hơn |
| `ef_construction` | Độ kỹ khi xây đồ thị | Index chậm hơn, chất lượng đồ thị tốt hơn |
| `ef_search` | Số ứng viên xem xét khi tìm | Tìm chậm hơn, recall cao hơn |

Với vài trăm nghìn vector trở xuống, giá trị mặc định của các vector DB thường đã đủ tốt.

### 3.3 Hàm khoảng cách

- **Cosine**: phổ biến nhất cho embedding văn bản.
- **Dot product (inner product)**: tương đương cosine nếu vector đã được chuẩn hóa độ dài 1, và nhanh hơn.
- **Euclid (L2)**: dùng khi model được huấn luyện với L2.

Hãy dùng đúng hàm khoảng cách mà nhà cung cấp model khuyến nghị. Lưu ý mỗi vector store trả về "score" theo quy ước riêng: có nơi điểm càng cao càng giống, có nơi là khoảng cách (càng thấp càng giống).

### 3.4 Quantization

Nén vector để tiết kiệm bộ nhớ: **scalar quantization** (float32 thành int8, giảm 4 lần), **binary quantization** (mỗi chiều còn 1 bit, giảm 32 lần). Thường kết hợp với bước chấm lại bằng vector gốc để giữ độ chính xác. Chỉ cần quan tâm khi dữ liệu lên tới hàng triệu vector.

## 4. Chọn vector database

| Vector DB | Điểm mạnh | Phù hợp |
|---|---|---|
| `InMemoryVectorStore`, FAISS | Không cần server | Học tập, prototype, script |
| Chroma | Cài đặt đơn giản, lưu file local | Ứng dụng nhỏ, một máy |
| **pgvector** (PostgreSQL) | Dùng chung DB có sẵn, JOIN với dữ liệu nghiệp vụ, transaction, backup quen thuộc | Đa số ứng dụng web |
| Qdrant | Lọc metadata rất mạnh, hybrid search, quantization | Production tự host, cần hiệu năng cao |
| Pinecone | Managed hoàn toàn, serverless | Không muốn vận hành hạ tầng |
| Weaviate, Milvus | Quy mô lớn, nhiều tính năng | Hàng chục triệu vector trở lên |
| Elasticsearch, OpenSearch | Full-text search rất mạnh kèm vector | Đã có sẵn hạ tầng Elastic |

!!! tip "Lời khuyên thực tế"
    Nếu ứng dụng đã dùng PostgreSQL, hãy bắt đầu bằng **pgvector**. Một hệ thống ít thành phần dễ vận hành hơn nhiều. Chỉ chuyển sang DB chuyên dụng khi **đo được** rằng pgvector không đáp ứng.

## 5. Thực hành: chuyển sang vector database thật

=== "pgvector (khuyến nghị)"

    Chạy PostgreSQL có sẵn extension pgvector:

    ```bash
    docker run -d -p 5432:5432 -e POSTGRES_PASSWORD=postgres pgvector/pgvector:pg17
    uv add langchain-postgres "psycopg[binary]"
    ```

    ```python title="rag/config.py"
    from dotenv import load_dotenv
    from langchain.chat_models import init_chat_model
    from langchain.embeddings import init_embeddings
    from langchain_postgres import PGVector

    load_dotenv()

    CHAT_MODEL = "anthropic:claude-sonnet-5"
    EMBEDDING_MODEL = "openai:text-embedding-3-small"
    EMBEDDING_DIM = 1536

    chat_model = init_chat_model(CHAT_MODEL)
    embeddings = init_embeddings(EMBEDDING_MODEL)

    vector_store = PGVector(
        embeddings=embeddings,
        # Đặt tên collection theo model để không bao giờ trộn embedding
        collection_name="rag_lab__text-embedding-3-small",
        connection="postgresql+psycopg://postgres:postgres@localhost:5432/postgres",
        embedding_length=EMBEDDING_DIM,  # cần khai báo để tạo được index HNSW
        use_jsonb=True,
    )
    ```

    Tạo index HNSW (chạy một lần trong `psql` hoặc công cụ quản lý DB):

    ```sql
    CREATE INDEX IF NOT EXISTS idx_embedding_hnsw
    ON langchain_pg_embedding
    USING hnsw (embedding vector_cosine_ops);
    ```

    Lọc theo metadata:

    ```python
    docs = vector_store.similarity_search(
        "chính sách nghỉ phép",
        k=5,
        filter={"doc_type": {"$eq": "policy"}, "access_group": {"$in": ["all", "hr"]}},
    )
    ```

    !!! note "Ghi chú"
        Package `langchain-postgres` còn có lớp mới `PGVectorStore` với thiết kế schema linh hoạt hơn. Lớp `PGVector` dùng ở đây đơn giản hơn cho mục đích học tập.

=== "Qdrant"

    ```bash
    docker run -d -p 6333:6333 qdrant/qdrant
    uv add langchain-qdrant
    ```

    ```python title="rag/config.py (phần vector store)"
    from langchain_qdrant import QdrantVectorStore
    from qdrant_client import QdrantClient
    from qdrant_client.models import Distance, VectorParams

    client = QdrantClient(url="http://localhost:6333")
    collection = "rag_lab__text-embedding-3-small"
    if not client.collection_exists(collection):
        client.create_collection(
            collection, vectors_config=VectorParams(size=1536, distance=Distance.COSINE)
        )

    vector_store = QdrantVectorStore(
        client=client, collection_name=collection, embedding=embeddings
    )
    ```

    Lọc theo metadata (LangChain lưu metadata dưới khóa `metadata`):

    ```python
    from qdrant_client import models

    docs = vector_store.similarity_search(
        "chính sách nghỉ phép",
        k=5,
        filter=models.Filter(
            must=[
                models.FieldCondition(
                    key="metadata.doc_type", match=models.MatchValue(value="policy")
                )
            ]
        ),
    )
    ```

=== "Chroma"

    ```bash
    uv add langchain-chroma
    ```

    ```python title="rag/config.py (phần vector store)"
    from langchain_chroma import Chroma

    vector_store = Chroma(
        collection_name="rag_lab__text-embedding-3-small",
        embedding_function=embeddings,
        persist_directory="./chroma_db",  # dữ liệu lưu thành file, không mất khi tắt
    )
    ```

    Lọc theo metadata:

    ```python
    docs = vector_store.similarity_search(
        "chính sách nghỉ phép", k=5, filter={"doc_type": "policy"}
    )
    ```

Vì mọi vector store trong LangChain đều có chung interface (`add_documents`, `similarity_search`, `delete`...), các module `ingest.py`, `retrieve.py`, `generate.py` **không cần sửa gì**. Đây là lợi ích lớn nhất của lớp trừu tượng trong LangChain.

!!! warning "Index hai lần"
    Với database thật, chạy `build_index()` hai lần sẽ tạo bản ghi trùng lặp **trừ khi** bạn truyền `ids` cố định như ở Bài 2 (`ids=[c.metadata["chunk_id"] ...]`). Khi có `ids`, bản ghi cũ được ghi đè. Bài 8 sẽ xử lý đầy đủ việc cập nhật và xóa.

## 6. Lọc metadata: pre-filter và post-filter

- **Pre-filter**: lọc trước rồi mới tìm vector trong tập đã lọc. Kết quả luôn đủ k, nhưng có thể chậm nếu bộ lọc phức tạp.
- **Post-filter**: tìm top k trước rồi mới lọc. Nhanh, nhưng nếu phần lớn kết quả bị lọc bỏ thì còn lại **ít hơn k**, thậm chí rỗng.

Qdrant và pgvector (với cấu hình phù hợp) hỗ trợ lọc trong lúc tìm. Khi bộ lọc rất chặt (ví dụ mỗi khách hàng chỉ có vài trăm chunk trong một bảng hàng triệu chunk), hãy kiểm tra kỹ số kết quả trả về. Đây là vấn đề quan trọng khi làm phân quyền ở [Bài 8](08-production.md).

## Bài tập

**Bài 3.1.** Chạy `compare_embeddings.py` với ít nhất 2 model (một API, một mã nguồn mở). Ghi bảng kết quả hit@1, hit@5, thời gian index.

**Bài 3.2.** Thử `multilingual-e5-large` **có** và **không có** tiền tố `query:` / `passage:`. Chênh lệch bao nhiêu?

??? tip "Gợi ý lời giải"
    `HuggingFaceEmbeddings` nhận tham số `query_encode_kwargs` và `encode_kwargs` riêng cho câu hỏi và tài liệu. Cách đơn giản và rõ ràng nhất là tự thêm tiền tố:

    ```python
    from langchain_core.embeddings import Embeddings

    class E5Embeddings(Embeddings):
        def __init__(self, base: Embeddings):
            self.base = base

        def embed_documents(self, texts: list[str]) -> list[list[float]]:
            return self.base.embed_documents([f"passage: {t}" for t in texts])

        def embed_query(self, text: str) -> list[float]:
            return self.base.embed_query(f"query: {text}")

    e5 = E5Embeddings(HuggingFaceEmbeddings(
        model_name="intfloat/multilingual-e5-large",
        encode_kwargs={"normalize_embeddings": True},
    ))
    ```

**Bài 3.3.** Chuyển `rag-lab` sang pgvector (hoặc Qdrant). Tắt chương trình, bật lại, xác nhận dữ liệu vẫn còn và không cần index lại.

**Bài 3.4.** Thêm trường `doc_type` với ít nhất 2 giá trị khác nhau vào tài liệu của bạn. Viết hàm `retrieve(query, doc_type=None)` có lọc tùy chọn.

**Bài 3.5 (mở rộng).** Dùng `text-embedding-3-small` với tham số `dimensions=512` (tính năng Matryoshka). So sánh hit@5 với bản 1536 chiều.

??? tip "Gợi ý"
    ```python
    from langchain_openai import OpenAIEmbeddings

    small = OpenAIEmbeddings(model="text-embedding-3-small", dimensions=512)
    ```

## Checklist

- [ ] Giải thích được bi-encoder và vì sao nó nhanh nhưng có giới hạn độ chính xác.
- [ ] Chọn model embedding dựa trên số liệu đo trên dữ liệu của mình.
- [ ] Giải thích được HNSW ở mức khái niệm và các tham số chính.
- [ ] `rag-lab` đã chạy trên vector database thật, dữ liệu không mất khi khởi động lại.
- [ ] Biết lọc metadata trong vector DB mình chọn và hiểu rủi ro của post-filter.

## Đọc thêm

- [Xây dựng semantic search](../../oss/python/langchain/knowledge-base.md): phần chọn embedding và vector store.
- [Danh sách tích hợp vector store](https://docs.langchain.com/oss/python/integrations/vectorstores) của LangChain.
- [MTEB Leaderboard](https://huggingface.co/spaces/mteb/leaderboard).

---

**Bài trước:** [Bài 2: Ingestion và Chunking](02-ingestion-chunking.md) · [Tổng quan](index.md) · **Bài tiếp theo:** [Bài 4: Retrieval nâng cao](04-retrieval-nang-cao.md)
