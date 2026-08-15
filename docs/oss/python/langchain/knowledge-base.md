# Xây dựng công cụ tìm kiếm ngữ nghĩa (semantic search) với LangChain

## Tổng quan

Xây dựng một công cụ tìm kiếm ngữ nghĩa trên một file PDF bằng [embeddings](https://docs.langchain.com/oss/python/integrations/embeddings) và [vector store](https://docs.langchain.com/oss/python/integrations/vectorstores) của LangChain. Dùng nó để truy xuất (retrieve) các đoạn văn bản tương tự với một truy vấn, sau đó gắn retriever vào [retrieval-augmented generation (RAG)](https://docs.langchain.com/oss/python/deepagents/retrieval) hoặc các luồng xử lý (workflow) LLM khác.

Hướng dẫn này bao gồm:

1. Tạo các đối tượng `Document` từ một file PDF.
2. Tạo embeddings.
3. Tải và chia nhỏ (split) một file PDF.
4. Lập chỉ mục (index) các chunk vào vector store và truy vấn theo độ tương đồng (similarity).
5. Bọc (wrap) store thành một retriever.

Hướng dẫn này cũng bao gồm một cách triển khai RAG tối giản trên nền công cụ tìm kiếm này.

### Khái niệm

Hướng dẫn này tập trung vào truy xuất văn bản (text retrieval) và bao gồm các khái niệm sau:

* [`Document`](https://reference.langchain.com/python/langchain-core/documents/base/Document)
* [Text splitters](https://docs.langchain.com/oss/python/integrations/splitters)
* [Embeddings](https://docs.langchain.com/oss/python/integrations/embeddings)
* [Vector store](https://docs.langchain.com/oss/python/integrations/vectorstores) và [retriever](https://docs.langchain.com/oss/python/integrations/retrievers)

## Cài đặt

### Cài đặt các dependency

Hướng dẫn này đọc một file PDF bằng package `pypdf`:

=== "pip"

    ```bash
    pip install pypdf
    ```

=== "conda"

    ```bash
    conda install pypdf -c conda-forge
    ```

=== "uv"

    ```bash
    uv add pypdf
    ```

Để biết thêm chi tiết, xem [Hướng dẫn cài đặt](install.md).

### Cấu hình LangSmith

Nhiều ứng dụng bạn xây dựng bằng LangChain sẽ chứa nhiều bước với nhiều lượt gọi LLM. Khi các ứng dụng này ngày càng phức tạp, việc có thể kiểm tra chính xác những gì đang diễn ra bên trong chain hoặc agent của bạn trở nên rất quan trọng. Cách tốt nhất để làm điều này là dùng [LangSmith](https://smith.langchain.com?utm_source=docs\&utm_medium=cta\&utm_campaign=langsmith-signup\&utm_content=oss-langchain-knowledge-base).

Sau khi đăng ký theo liên kết trên, hãy đảm bảo thiết lập các biến môi trường để bắt đầu ghi log trace:

```shell
export LANGSMITH_TRACING="true"
export LANGSMITH_API_KEY="..."
```

Trong notebook, bạn có thể thiết lập chúng bằng:

```python
import getpass
import os

os.environ["LANGSMITH_TRACING"] = "true"
os.environ["LANGSMITH_API_KEY"] = getpass.getpass()
```

## Tạo document

LangChain triển khai một abstraction (khái niệm trừu tượng) [`Document`](https://reference.langchain.com/python/langchain-core/documents/base/Document) đại diện cho một đơn vị văn bản kèm metadata liên quan. Nó có ba thuộc tính:

* `page_content`: một chuỗi (string) đại diện cho nội dung.
* `metadata`: một dict chứa metadata tuỳ ý.
* `id`: (tuỳ chọn) một chuỗi định danh cho document.

`metadata` có thể lưu nguồn gốc của document, mối quan hệ của nó với các document khác, và các thông tin khác. Một [`Document`](https://reference.langchain.com/python/langchain-core/documents/base/Document) riêng lẻ thường đại diện cho một chunk của một document lớn hơn.

Đoạn code sau tạo ra các document mẫu:

```python
from langchain_core.documents import Document

documents = [
    Document(
        page_content="Dogs are great companions, known for their loyalty and friendliness.",
        metadata={"source": "mammal-pets-doc"},
    ),
    Document(
        page_content="Cats are independent pets that often enjoy their own space.",
        metadata={"source": "mammal-pets-doc"},
    ),
]
```

## Tạo embeddings

Vector search lưu trữ các vector số gắn với văn bản. Embed một truy vấn (query) thành một vector cùng chiều (dimension), sau đó dùng các độ đo tương đồng (similarity metric, ví dụ cosine similarity) để tìm văn bản liên quan.

LangChain hỗ trợ embeddings từ [nhiều nhà cung cấp](https://docs.langchain.com/oss/python/integrations/embeddings/). Chọn một model để xác định cách văn bản được chuyển đổi thành vector số:

=== "OpenAI"

    ```shell
    pip install -U "langchain-openai"
    ```

    ```python
    import getpass
    import os

    if not os.environ.get("OPENAI_API_KEY"):
        os.environ["OPENAI_API_KEY"] = getpass.getpass("Enter API key for OpenAI: ")

    from langchain_openai import OpenAIEmbeddings

    embeddings = OpenAIEmbeddings(model="text-embedding-3-large")
    ```

=== "Azure"

    ```shell
    pip install -U "langchain-openai"
    ```

    ```python
    import getpass
    import os

    if not os.environ.get("AZURE_OPENAI_API_KEY"):
        os.environ["AZURE_OPENAI_API_KEY"] = getpass.getpass("Enter API key for Azure: ")

    from langchain_openai import AzureOpenAIEmbeddings

    embeddings = AzureOpenAIEmbeddings(
        azure_endpoint=os.environ["AZURE_OPENAI_ENDPOINT"],
        azure_deployment=os.environ["AZURE_OPENAI_DEPLOYMENT_NAME"],
        openai_api_version=os.environ["AZURE_OPENAI_API_VERSION"],
    )
    ```

=== "Google Gemini"

    ```shell
    pip install -qU langchain-google-genai
    ```

    ```python
    import getpass
    import os

    if not os.environ.get("GOOGLE_API_KEY"):
        os.environ["GOOGLE_API_KEY"] = getpass.getpass("Enter API key for Google Gemini: ")

    from langchain_google_genai import GoogleGenerativeAIEmbeddings

    embeddings = GoogleGenerativeAIEmbeddings(model="models/gemini-embedding-001")
    ```

=== "Google Vertex"

    ```shell
    pip install -qU langchain-google-vertexai
    ```

    ```python
    from langchain_google_vertexai import VertexAIEmbeddings

    embeddings = VertexAIEmbeddings(model="text-embedding-005")
    ```

=== "AWS"

    ```shell
    pip install -qU langchain-aws
    ```

    ```python
    from langchain_aws import BedrockEmbeddings

    embeddings = BedrockEmbeddings(model_id="amazon.titan-embed-text-v2:0")
    ```

=== "HuggingFace"

    ```shell
    pip install -qU langchain-huggingface
    ```

    ```python
    from langchain_huggingface import HuggingFaceEmbeddings

    embeddings = HuggingFaceEmbeddings(
        model_name="sentence-transformers/all-mpnet-base-v2",
        encode_kwargs={"normalize_embeddings": True},
    )
    ```

=== "Ollama"

    ```shell
    pip install -qU langchain-ollama
    ```

    ```python
    from langchain_ollama import OllamaEmbeddings

    embeddings = OllamaEmbeddings(model="llama3")
    ```

=== "Cohere"

    ```shell
    pip install -qU langchain-cohere
    ```

    ```python
    import getpass
    import os

    if not os.environ.get("COHERE_API_KEY"):
        os.environ["COHERE_API_KEY"] = getpass.getpass("Enter API key for Cohere: ")

    from langchain_cohere import CohereEmbeddings

    embeddings = CohereEmbeddings(model="embed-english-v3.0")
    ```

=== "MistralAI"

    ```shell
    pip install -qU langchain-mistralai
    ```

    ```python
    import getpass
    import os

    if not os.environ.get("MISTRALAI_API_KEY"):
        os.environ["MISTRALAI_API_KEY"] = getpass.getpass("Enter API key for MistralAI: ")

    from langchain_mistralai import MistralAIEmbeddings

    embeddings = MistralAIEmbeddings(model="mistral-embed")
    ```

=== "Nomic"

    ```shell
    pip install -qU langchain-nomic
    ```

    ```python
    import getpass
    import os

    if not os.environ.get("NOMIC_API_KEY"):
        os.environ["NOMIC_API_KEY"] = getpass.getpass("Enter API key for Nomic: ")

    from langchain_nomic import NomicEmbeddings

    embeddings = NomicEmbeddings(model="nomic-embed-text-v1.5")
    ```

=== "NVIDIA"

    ```shell
    pip install -qU langchain-nvidia-ai-endpoints
    ```

    ```python
    import getpass
    import os

    if not os.environ.get("NVIDIA_API_KEY"):
        os.environ["NVIDIA_API_KEY"] = getpass.getpass("Enter API key for NVIDIA: ")

    from langchain_nvidia_ai_endpoints import NVIDIAEmbeddings

    embeddings = NVIDIAEmbeddings(model="NV-Embed-QA")
    ```

=== "Voyage AI"

    ```shell
    pip install -qU langchain-voyageai
    ```

    ```python
    import getpass
    import os

    if not os.environ.get("VOYAGE_API_KEY"):
        os.environ["VOYAGE_API_KEY"] = getpass.getpass("Enter API key for Voyage AI: ")

    from langchain-voyageai import VoyageAIEmbeddings

    embeddings = VoyageAIEmbeddings(model="voyage-3")
    ```

=== "IBM watsonx"

    ```shell
    pip install -qU langchain-ibm
    ```

    ```python
    import getpass
    import os

    if not os.environ.get("WATSONX_APIKEY"):
        os.environ["WATSONX_APIKEY"] = getpass.getpass("Enter API key for IBM watsonx: ")

    from langchain_ibm import WatsonxEmbeddings

    embeddings = WatsonxEmbeddings(
        model_id="ibm/slate-125m-english-rtrvr",
        url="https://us-south.ml.cloud.ibm.com",
        project_id="<WATSONX PROJECT_ID>",
    )
    ```

=== "Fake"

    ```shell
    pip install -qU langchain-core
    ```

    ```python
    from langchain_core.embeddings import DeterministicFakeEmbedding

    embeddings = DeterministicFakeEmbedding(size=4096)
    ```

=== "Isaacus"

    ```shell
    pip install -qU langchain-isaacus
    ```

    ```python
    import getpass
    import os

    if not os.environ.get("ISAACUS_API_KEY"):
    os.environ["ISAACUS_API_KEY"] = getpass.getpass("Enter API key for Isaacus: ")

    from langchain_isaacus import IsaacusEmbeddings

    embeddings = IsaacusEmbeddings(model="kanon-2-embedder")
    ```

```python
vector_1 = embeddings.embed_query(documents[0].page_content)
vector_2 = embeddings.embed_query(documents[1].page_content)

assert len(vector_1) == len(vector_2)
print(f"Generated vectors of length {len(vector_1)}\n")
print(vector_1[:10])
```

```text
Generated vectors of length 1536

[-0.008586574345827103, -0.03341241180896759, -0.008936782367527485, -0.0036674530711025, 0.010564599186182022, 0.009598285891115665, -0.028587326407432556, -0.015824200585484505, 0.0030416189692914486, -0.012899317778646946]
```

Tiếp theo, lưu trữ embeddings trong một vector store hỗ trợ tìm kiếm tương đồng (similarity search) hiệu quả.

## Chọn một vector store

Các đối tượng [`VectorStore`](https://reference.langchain.com/python/langchain-core/vectorstores/base/VectorStore) của LangChain thêm văn bản và các đối tượng [`Document`](https://reference.langchain.com/python/langchain-core/documents/base/Document) vào một store rồi truy vấn chúng bằng các độ đo tương đồng. Chúng thường được khởi tạo với các model [embedding](https://docs.langchain.com/oss/python/integrations/embeddings) để chuyển văn bản thành vector số.

LangChain bao gồm các [tích hợp (integration)](https://docs.langchain.com/oss/python/integrations/vectorstores) với nhiều công nghệ vector store. Một số được hosted và cần credentials, một số chạy trên hạ tầng riêng (local hoặc bên thứ ba), và một số khác chạy in-memory cho các workload nhẹ. Chọn một vector store:

=== "In-memory"

    ```shell
    pip install -U "langchain-core"
    ```

    ```python
    from langchain_core.vectorstores import InMemoryVectorStore

    vector_store = InMemoryVectorStore(embeddings)
    ```

=== "Amazon OpenSearch"

    ```shell
    pip install -qU  boto3
    ```

    ```python
    from opensearchpy import RequestsHttpConnection

    service = "es"  # must set the service as 'es'
    region = "us-east-2"
    credentials = boto3.Session(
        aws_access_key_id="xxxxxx", aws_secret_access_key="xxxxx"
    ).get_credentials()
    awsauth = AWS4Auth("xxxxx", "xxxxxx", region, service, session_token=credentials.token)

    vector_store = OpenSearchVectorSearch.from_documents(
        docs,
        embeddings,
        opensearch_url="host url",
        http_auth=awsauth,
        timeout=300,
        use_ssl=True,
        verify_certs=True,
        connection_class=RequestsHttpConnection,
        index_name="test-index",
    )
    ```

=== "AstraDB"

    ```shell
    pip install -U "langchain-astradb"
    ```

    ```python
    from langchain_astradb import AstraDBVectorStore

    vector_store = AstraDBVectorStore(
        embedding=embeddings,
        api_endpoint=ASTRA_DB_API_ENDPOINT,
        collection_name="astra_vector_langchain",
        token=ASTRA_DB_APPLICATION_TOKEN,
        namespace=ASTRA_DB_NAMESPACE,
    )
    ```

=== "Chroma"

    ```shell
    pip install -qU langchain-chroma
    ```

    ```python
    from langchain_chroma import Chroma

    vector_store = Chroma(
        collection_name="example_collection",
        embedding_function=embeddings,
        persist_directory="./chroma_langchain_db",  # Nơi lưu dữ liệu cục bộ, bỏ đi nếu không cần
    )
    ```

=== "Milvus"

    ```shell
    pip install -qU langchain-milvus
    ```

    ```python
    from langchain_milvus import Milvus

    URI = "./milvus_example.db"

    vector_store = Milvus(
        embedding_function=embeddings,
        connection_args={"uri": URI},
        index_params={"index_type": "FLAT", "metric_type": "L2"},
    )
    ```

=== "MongoDB"

    ```shell
    pip install -qU langchain-mongodb
    ```

    ```python
    from langchain_mongodb import MongoDBAtlasVectorSearch

    vector_store = MongoDBAtlasVectorSearch(
        embedding=embeddings,
        collection=MONGODB_COLLECTION,
        index_name=ATLAS_VECTOR_SEARCH_INDEX_NAME,
        relevance_score_fn="cosine",
    )
    ```

=== "PGVector"

    ```shell
    pip install -qU langchain-postgres
    ```

    ```python
    from langchain_postgres import PGVector

    vector_store = PGVector(
        embeddings=embeddings,
        collection_name="my_docs",
        connection="postgresql+psycopg://...",
    )
    ```

=== "PGVectorStore"

    ```shell
    pip install -qU langchain-postgres
    ```

    ```python
    from langchain_postgres import PGEngine, PGVectorStore

    pg_engine = PGEngine.from_connection_string(
        url="postgresql+psycopg://..."
    )

    vector_store = PGVectorStore.create_sync(
        engine=pg_engine,
        table_name='test_table',
        embedding_service=embeddings
    )
    ```

=== "Pinecone"

    ```shell
    pip install -qU langchain-pinecone
    ```

    ```python
    from langchain_pinecone import PineconeVectorStore
    from pinecone import Pinecone

    pc = Pinecone(api_key=...)
    index = pc.Index(index_name)

    vector_store = PineconeVectorStore(embedding=embeddings, index=index)
    ```

=== "Qdrant"

    ```shell
    pip install -qU langchain-qdrant
    ```

    ```python
    from qdrant_client.models import Distance, VectorParams
    from langchain_qdrant import QdrantVectorStore
    from qdrant_client import QdrantClient

    client = QdrantClient(":memory:")

    vector_size = len(embeddings.embed_query("sample text"))

    if not client.collection_exists("test"):
        client.create_collection(
            collection_name="test",
            vectors_config=VectorParams(size=vector_size, distance=Distance.COSINE)
        )
    vector_store = QdrantVectorStore(
        client=client,
        collection_name="test",
        embedding=embeddings,
    )
    ```

## Tải và chia nhỏ một file PDF

Tải nội dung từ một file PDF, sau đó chia nhỏ (split) nó thành các chunk nhỏ hơn trước khi lập chỉ mục (index). Ví dụ này dùng [một file 10-K mẫu của Nike năm 2023](https://github.com/langchain-ai/langchain/blob/v0.3/docs/docs/example_data/nke-10k-2023.pdf).

```python
import pypdf
from langchain_core.documents import Document


# Dưới đây là một helper tối giản chỉ nhằm mục đích minh hoạ.
def load_pdf_pages(file_path: str) -> list[Document]:
    reader = pypdf.PdfReader(file_path)
    return [
        Document(
            page_content=page.extract_text() or "",
            metadata={"source": file_path, "page": i},
        )
        for i, page in enumerate(reader.pages)
    ]


file_path = "../example_data/nke-10k-2023.pdf"
docs = load_pdf_pages(file_path)
print(len(docs))
```

```text
107
```

Một trang thường quá thô (coarse) để dùng cho retrieval. Hãy chia nhỏ các trang hơn nữa để các đoạn văn bản liên quan không bị pha loãng bởi văn bản xung quanh. [`RecursiveCharacterTextSplitter`](https://reference.langchain.com/python/langchain-text-splitters/character/RecursiveCharacterTextSplitter) chia nhỏ đệ quy (recursively) theo các dấu phân cách phổ biến (như ký tự xuống dòng) cho đến khi mỗi chunk đạt kích thước mục tiêu. Đây là text splitter được khuyến nghị cho các use case văn bản thông thường.

Đặt `add_start_index=True` để mỗi phần chia (split) giữ một trường metadata `start_index` cho biết vị trí offset ký tự của nó trong document gốc.

```python
from langchain_text_splitters import RecursiveCharacterTextSplitter

text_splitter = RecursiveCharacterTextSplitter(
    chunk_size=1000, chunk_overlap=200, add_start_index=True
)
all_splits = text_splitter.split_documents(docs)

print(len(all_splits))
```

```text
516
```

## Lập chỉ mục document

Lập chỉ mục các chunk vào vector store:

```python
ids = vector_store.add_documents(documents=all_splits)
```

Hầu hết các tích hợp vector store cũng hỗ trợ kết nối tới một store đã có sẵn (ví dụ bằng client hoặc index name). Xem tài liệu của [tích hợp](https://docs.langchain.com/oss/python/integrations/vectorstores) cụ thể để biết chi tiết.

## Truy vấn vector store

Sau khi đã thêm document vào [`VectorStore`](https://reference.langchain.com/python/langchain-core/vectorstores/base/VectorStore), bạn có thể truy vấn nó:

* Đồng bộ (synchronously) và bất đồng bộ (asynchronously)
* Bằng truy vấn dạng chuỗi (string) và bằng vector
* Có và không có điểm số tương đồng (similarity score)
* Theo độ tương đồng (similarity) và [maximum marginal relevance](https://reference.langchain.com/python/langchain-core/vectorstores/base/VectorStore/max_marginal_relevance_search) (để cân bằng giữa độ tương đồng và tính đa dạng)

Các phương thức này thường trả về một danh sách các đối tượng [`Document`](https://reference.langchain.com/python/langchain-core/documents/base/Document).

### Tìm kiếm theo chuỗi (string)

Embeddings ánh xạ văn bản thành các vector dày đặc (dense vector) sao cho các nghĩa tương tự nằm gần nhau về mặt hình học. Điều này có nghĩa là bạn có thể truy xuất (retrieve) các đoạn văn bản liên quan bằng cách truyền vào một câu hỏi ngôn ngữ tự nhiên:

```python
results = vector_store.similarity_search(
    "How many distribution centers does Nike have in the US?"
)

print(results[0])
```

```python
page_content='direct to consumer operations sell products through the following number of retail stores in the United States:
U.S. RETAIL STORES NUMBER
NIKE Brand factory stores 213
NIKE Brand in-line stores (including employee-only stores) 74
Converse stores (including factory stores) 82
TOTAL 369
In the United States, NIKE has eight significant distribution centers. Refer to Item 2. Properties for further information.
2023 FORM 10-K 2' metadata={'page': 4, 'source': '../example_data/nke-10k-2023.pdf', 'start_index': 3125}
```

Truy vấn bất đồng bộ (async):

```python
results = await vector_store.asimilarity_search("When was Nike incorporated?")

print(results[0])
```

```python
page_content='Table of Contents
PART I
ITEM 1. BUSINESS
GENERAL
NIKE, Inc. was incorporated in 1967 under the laws of the State of Oregon. As used in this Annual Report on Form 10-K (this "Annual Report"), the terms "we," "us," "our,"
"NIKE" and the "Company" refer to NIKE, Inc. and its predecessors, subsidiaries and affiliates, collectively, unless the context indicates otherwise.
Our principal business activity is the design, development and worldwide marketing and selling of athletic footwear, apparel, equipment, accessories and services. NIKE is
the largest seller of athletic footwear and apparel in the world. We sell our products through NIKE Direct operations, which are comprised of both NIKE-owned retail stores
and sales through our digital platforms (also referred to as "NIKE Brand Digital"), to retail accounts and to a mix of independent distributors, licensees and sales' metadata={'page': 3, 'source': '../example_data/nke-10k-2023.pdf', 'start_index': 0}
```

### Trả về điểm số (score)

Bạn có thể trả về điểm số tương đồng (similarity score) cùng với các document. Ý nghĩa của điểm số này khác nhau tuỳ nhà cung cấp. Trong trường hợp này, điểm số là một độ đo khoảng cách (distance metric) tỉ lệ nghịch với độ tương đồng:

```python
# Lưu ý rằng mỗi nhà cung cấp triển khai điểm số khác nhau; điểm số ở đây
# là một độ đo khoảng cách tỉ lệ nghịch với độ tương đồng.

results = vector_store.similarity_search_with_score("What was Nike's revenue in 2023?")
doc, score = results[0]
print(f"Score: {score}\n")
print(doc)
```

```python
Score: 0.23699893057346344

page_content='Table of Contents
FISCAL 2023 NIKE BRAND REVENUE HIGHLIGHTS
The following tables present NIKE Brand revenues disaggregated by reportable operating segment, distribution channel and major product line:
FISCAL 2023 COMPARED TO FISCAL 2022
•NIKE, Inc. Revenues were $51.2 billion in fiscal 2023, which increased 10% and 16% compared to fiscal 2022 on a reported and currency-neutral basis, respectively.
The increase was due to higher revenues in North America, Europe, Middle East & Africa ("EMEA"), APLA and Greater China, which contributed approximately 7, 6,
2 and 1 percentage points to NIKE, Inc. Revenues, respectively.
•NIKE Brand revenues, which represented over 90% of NIKE, Inc. Revenues, increased 10% and 16% on a reported and currency-neutral basis, respectively. This
increase was primarily due to higher revenues in Men's, the Jordan Brand, Women's and Kids' which grew 17%, 35%,11% and 10%, respectively, on a wholesale
equivalent basis.' metadata={'page': 35, 'source': '../example_data/nke-10k-2023.pdf', 'start_index': 0}
```

### Tìm kiếm theo vector

Tự embed truy vấn, sau đó tìm kiếm bằng vector thu được:

```python
embedding = embeddings.embed_query("How were Nike's margins impacted in 2023?")

results = vector_store.similarity_search_by_vector(embedding)
print(results[0])
```

```python
page_content='Table of Contents
GROSS MARGIN
FISCAL 2023 COMPARED TO FISCAL 2022
For fiscal 2023, our consolidated gross profit increased 4% to $22,292 million compared to $21,479 million for fiscal 2022. Gross margin decreased 250 basis points to
43.5% for fiscal 2023 compared to 46.0% for fiscal 2022 due to the following:
*Wholesale equivalent
The decrease in gross margin for fiscal 2023 was primarily due to:
•Higher NIKE Brand product costs, on a wholesale equivalent basis, primarily due to higher input costs and elevated inbound freight and logistics costs as well as
product mix;
•Lower margin in our NIKE Direct business, driven by higher promotional activity to liquidate inventory in the current period compared to lower promotional activity in
the prior period resulting from lower available inventory supply;
•Unfavorable changes in net foreign currency exchange rates, including hedges; and
•Lower off-price margin, on a wholesale equivalent basis.
This was partially offset by:' metadata={'page': 36, 'source': '../example_data/nke-10k-2023.pdf', 'start_index': 0}
```

Tìm hiểu thêm:

* [API Reference](https://reference.langchain.com/python/langchain-core/vectorstores/base/VectorStore)
* [Tài liệu riêng cho từng tích hợp](https://docs.langchain.com/oss/python/integrations/vectorstores)

## Sử dụng retriever

Các đối tượng [`VectorStore`](https://reference.langchain.com/python/langchain-core/vectorstores/base/VectorStore) của LangChain không kế thừa (subclass) từ [`Runnable`](https://reference.langchain.com/python/langchain-core/runnables/base/Runnable). [Retriever](https://reference.langchain.com/python/langchain-core/retrievers/BaseRetriever) là các Runnable, nên chúng hỗ trợ các phương thức chuẩn như `invoke` và `batch` (cả đồng bộ lẫn bất đồng bộ).

Bạn cũng có thể xây dựng retriever từ vector store, và retriever cũng có thể bọc (wrap) các nguồn không phải vector (như các API bên ngoài).

Trong trường hợp này, tạo một retriever đơn giản mà không cần kế thừa `Retriever` bằng cách bọc `similarity_search`:

```python
from typing import List

from langchain_core.documents import Document
from langchain_core.runnables import chain


@chain
def retriever(query: str) -> List[Document]:
    return vector_store.similarity_search(query, k=1)


retriever.batch(
    [
        "How many distribution centers does Nike have in the US?",
        "When was Nike incorporated?",
    ],
)
```

```text
[[Document(metadata={'page': 4, 'source': '../example_data/nke-10k-2023.pdf', 'start_index': 3125}, page_content='direct to consumer operations sell products through the following number of retail stores in the United States:\nU.S. RETAIL STORES NUMBER\nNIKE Brand factory stores 213 \nNIKE Brand in-line stores (including employee-only stores) 74 \nConverse stores (including factory stores) 82 \nTOTAL 369 \nIn the United States, NIKE has eight significant distribution centers. Refer to Item 2. Properties for further information.\n2023 FORM 10-K 2')],
 [Document(metadata={'page': 3, 'source': '../example_data/nke-10k-2023.pdf', 'start_index': 0}, page_content='Table of Contents\nPART I\nITEM 1. BUSINESS\nGENERAL\nNIKE, Inc. was incorporated in 1967 under the laws of the State of Oregon. As used in this Annual Report on Form 10-K (this "Annual Report"), the terms "we," "us," "our,"\n"NIKE" and the "Company" refer to NIKE, Inc. and its predecessors, subsidiaries and affiliates, collectively, unless the context indicates otherwise.\nOur principal business activity is the design, development and worldwide marketing and selling of athletic footwear, apparel, equipment, accessories and services. NIKE is\nthe largest seller of athletic footwear and apparel in the world. We sell our products through NIKE Direct operations, which are comprised of both NIKE-owned retail stores\nand sales through our digital platforms (also referred to as "NIKE Brand Digital"), to retail accounts and to a mix of independent distributors, licensees and sales')]]
```

Vector store triển khai một phương thức `as_retriever` trả về một [`VectorStoreRetriever`](https://reference.langchain.com/python/langchain-core/vectorstores/base/VectorStoreRetriever). Các retriever này expose `search_type` và `search_kwargs` để chọn và tham số hoá (parameterize) các phương thức store bên dưới. Lặp lại ví dụ trên bằng:

```python
retriever = vector_store.as_retriever(
    search_type="similarity",
    search_kwargs={"k": 1},
)

retriever.batch(
    [
        "How many distribution centers does Nike have in the US?",
        "When was Nike incorporated?",
    ],
)
```

```text
[[Document(metadata={'page': 4, 'source': '../example_data/nke-10k-2023.pdf', 'start_index': 3125}, page_content='direct to consumer operations sell products through the following number of retail stores in the United States:\nU.S. RETAIL STORES NUMBER\nNIKE Brand factory stores 213 \nNIKE Brand in-line stores (including employee-only stores) 74 \nConverse stores (including factory stores) 82 \nTOTAL 369 \nIn the United States, NIKE has eight significant distribution centers. Refer to Item 2. Properties for further information.\n2023 FORM 10-K 2')],
 [Document(metadata={'page': 3, 'source': '../example_data/nke-10k-2023.pdf', 'start_index': 0}, page_content='Table of Contents\nPART I\nITEM 1. BUSINESS\nGENERAL\nNIKE, Inc. was incorporated in 1967 under the laws of the State of Oregon. As used in this Annual Report on Form 10-K (this "Annual Report"), the terms "we," "us," "our,"\n"NIKE" and the "Company" refer to NIKE, Inc. and its predecessors, subsidiaries and affiliates, collectively, unless the context indicates otherwise.\nOur principal business activity is the design, development and worldwide marketing and selling of athletic footwear, apparel, equipment, accessories and services. NIKE is\nthe largest seller of athletic footwear and apparel in the world. We sell our products through NIKE Direct operations, which are comprised of both NIKE-owned retail stores\nand sales through our digital platforms (also referred to as "NIKE Brand Digital"), to retail accounts and to a mix of independent distributors, licensees and sales')]]
```

`VectorStoreRetriever` hỗ trợ các search type gồm `"similarity"` (mặc định), `"mmr"` (maximum marginal relevance), và `"similarity_score_threshold"`. Dùng tuỳ chọn cuối để lọc document theo điểm số tương đồng.

Bạn có thể dùng retriever trong các ứng dụng phức tạp hơn như [retrieval-augmented generation (RAG)](https://docs.langchain.com/oss/python/deepagents/retrieval), kết hợp một câu hỏi với context truy xuất được vào trong một prompt cho LLM. Để tìm hiểu thêm về cách xây dựng ứng dụng dạng này, xem [hướng dẫn RAG](https://docs.langchain.com/oss/python/deepagents/rag).

## Bước tiếp theo

Bây giờ bạn đã biết cách xây dựng một công cụ tìm kiếm ngữ nghĩa trên một document PDF.

Để biết thêm thông tin, xem:

* [Các tích hợp embedding hiện có](https://docs.langchain.com/oss/python/integrations/embeddings)
* [Các tích hợp vector store hiện có](https://docs.langchain.com/oss/python/integrations/vectorstores)

Để biết thêm về RAG:

* [Tổng quan về retrieval](https://docs.langchain.com/oss/python/deepagents/retrieval)
* [RAG với Deep Agents](https://docs.langchain.com/oss/python/deepagents/rag)
* [Đánh giá một ứng dụng RAG](https://docs.langchain.com/langsmith/evaluate-rag-tutorial)

***

[Kết nối các tài liệu này](https://docs.langchain.com/use-these-docs) với Claude, VSCode, và nhiều công cụ khác qua MCP để có câu trả lời theo thời gian thực.

[Chỉnh sửa trang này trên GitHub](https://github.com/langchain-ai/docs/edit/main/src/oss/langchain/knowledge-base.mdx) hoặc [báo lỗi](https://github.com/langchain-ai/docs/issues/new/choose).
