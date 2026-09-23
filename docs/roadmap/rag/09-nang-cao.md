# Bài 9: Chủ đề nâng cao

!!! abstract "Tóm tắt bài học"
    - **Mục tiêu:** biết các hướng mở rộng của RAG, hiểu khi nào mỗi hướng đáng đầu tư, và có điểm khởi đầu để tự nghiên cứu sâu hơn.
    - **Thời lượng:** tùy chọn, học theo nhu cầu thực tế.
    - **Yêu cầu trước:** hoàn thành [Bài 1 đến Bài 8](index.md), đặc biệt là đã có eval runner ở [Bài 6](06-evaluation.md).

Các chủ đề trong bài này **độc lập** với nhau. Đừng học hết: hãy chọn theo vấn đề mà eval của bạn chỉ ra. Ví dụ, nếu các câu hỏi dạng "tổng hợp toàn bộ" luôn sai, hãy xem GraphRAG; nếu tài liệu nhiều biểu đồ, hãy xem multimodal RAG.

## 1. Long context hay RAG?

Context window của các model ngày càng lớn, đủ chứa hàng trăm trang. Vậy còn cần RAG không?

| Tiêu chí | Nạp thẳng toàn bộ tài liệu | RAG |
|---|---|---|
| Lượng dữ liệu | Vài chục đến vài trăm trang | Không giới hạn |
| Chi phí mỗi câu hỏi | Cao (trả tiền cho toàn bộ tài liệu mỗi lần, giảm được nhờ prompt caching) | Thấp |
| Độ trễ | Cao với context dài | Thấp |
| Câu hỏi cần suy luận xuyên suốt tài liệu | Tốt | Khó (chỉ thấy vài chunk) |
| Phân quyền theo tài liệu | Khó | Dễ (lọc khi retrieve) |
| Dữ liệu thay đổi liên tục | Phải nạp lại | Chỉ cập nhật phần thay đổi |
| Trích dẫn nguồn | Được, nhưng khó kiểm chứng | Tự nhiên, gắn với chunk |

**Kết hợp cả hai** thường là tốt nhất: dùng RAG để chọn **vài tài liệu** liên quan (thay vì vài chunk), rồi nạp **toàn bộ** các tài liệu đó vào context. Cách này đặc biệt hợp với hợp đồng, báo cáo, nơi mỗi tài liệu cần được đọc trọn vẹn.

```python
def retrieve_whole_documents(question: str, max_docs: int = 3) -> list[str]:
    """Dùng chunk để tìm tài liệu, nhưng trả về toàn văn tài liệu."""
    chunks = retrieve(question, k=10)
    doc_ids = list(dict.fromkeys(c.metadata["doc_id"] for c in chunks))[:max_docs]
    return [load_full_text(doc_id) for doc_id in doc_ids]  # đọc từ kho tài liệu gốc
```

## 2. GraphRAG

### 2.1 Vấn đề

RAG thông thường trả lời tốt câu hỏi **cục bộ** ("Điều 12 quy định gì?") nhưng kém với câu hỏi **toàn cục** hoặc **nhiều bước quan hệ**:

- "Các chủ đề chính trong 500 biên bản họp năm nay là gì?" (không chunk nào chứa câu trả lời)
- "Những dự án nào do người quản lý của anh Nam phụ trách?" (cần đi qua nhiều quan hệ)

### 2.2 Ý tưởng

1. Dùng LLM trích xuất **thực thể** (người, dự án, phòng ban, sản phẩm) và **quan hệ** giữa chúng từ tài liệu, tạo thành một **knowledge graph**.
2. (Với Microsoft GraphRAG) gom các thực thể thành **cộng đồng** (community) và tóm tắt từng cộng đồng.
3. Khi truy vấn: câu hỏi cục bộ đi theo quan hệ trong graph; câu hỏi toàn cục dùng các bản tóm tắt cộng đồng.

```python
# uv add langchain-experimental
from langchain_experimental.graph_transformers import LLMGraphTransformer

from rag.config import chat_model

transformer = LLMGraphTransformer(
    llm=chat_model,
    allowed_nodes=["NhanVien", "PhongBan", "DuAn"],
    allowed_relationships=["THUOC", "QUAN_LY", "PHU_TRACH"],
)
graph_documents = transformer.convert_to_graph_documents(chunks)

for gd in graph_documents[:2]:
    print(gd.nodes)
    print(gd.relationships)
# Lưu vào graph database như Neo4j, rồi truy vấn bằng Cypher hoặc để agent tự sinh truy vấn
```

!!! warning "Chi phí và độ phức tạp"
    GraphRAG tốn chi phí LLM lớn khi index (mỗi chunk một hoặc nhiều lệnh gọi), cần thêm graph database, và chất lượng phụ thuộc vào việc trích xuất thực thể. Chỉ đầu tư khi eval cho thấy **nhóm câu hỏi toàn cục hoặc nhiều bước quan hệ** là điểm yếu thực sự và quan trọng với người dùng.

Tham khảo: [Microsoft GraphRAG](https://github.com/microsoft/graphrag).

## 3. Multimodal RAG

Tài liệu thật chứa biểu đồ, sơ đồ, ảnh chụp màn hình, bảng dạng ảnh. Parse văn bản thuần sẽ bỏ qua toàn bộ thông tin này.

| Cách tiếp cận | Cách làm | Ưu | Nhược |
|---|---|---|---|
| **Mô tả ảnh thành văn bản** | Vision LLM viết mô tả chi tiết cho mỗi ảnh hoặc trang, index mô tả như văn bản | Dùng lại toàn bộ pipeline hiện có | Tốn chi phí khi index, mô tả có thể bỏ sót chi tiết |
| **Embed trực tiếp ảnh trang** (ColPali, ColQwen) | Model embed ảnh chụp cả trang, tìm bằng câu hỏi văn bản | Không cần OCR hay parse, giữ nguyên bố cục | Cần hạ tầng riêng, lưu trữ nặng hơn |
| **Multimodal embedding** | Model embed chung ảnh và văn bản vào một không gian | Tìm ảnh bằng văn bản và ngược lại | Chất lượng với tài liệu văn bản dày đặc còn hạn chế |

Cách thứ nhất là điểm khởi đầu dễ nhất:

```python title="rag/vision.py"
import base64

import pymupdf
from langchain_core.documents import Document

from rag.config import chat_model

DESCRIBE_PROMPT = """Mô tả chi tiết nội dung trang tài liệu này bằng tiếng Việt để phục vụ tìm kiếm.
- Chép lại toàn bộ văn bản, giữ cấu trúc heading.
- Với bảng: chuyển thành bảng Markdown.
- Với biểu đồ, sơ đồ: mô tả loại biểu đồ, các trục, số liệu chính và xu hướng."""


def describe_pdf_pages(path: str) -> list[Document]:
    docs = []
    with pymupdf.open(path) as pdf:
        for number, page in enumerate(pdf, start=1):
            png = page.get_pixmap(dpi=150).tobytes("png")
            response = chat_model.invoke([{
                "role": "user",
                "content": [
                    {"type": "text", "text": DESCRIBE_PROMPT},
                    {
                        "type": "image",
                        "base64": base64.b64encode(png).decode(),
                        "mime_type": "image/png",
                    },
                ],
            }])
            docs.append(Document(
                page_content=response.text,
                metadata={"source": path, "page": number, "modality": "page_image"},
            ))
    return docs
```

Để trả lời câu hỏi về biểu đồ chính xác hơn, bạn có thể lưu ảnh gốc của trang và **gửi kèm ảnh** cho LLM ở bước sinh câu trả lời, thay vì chỉ gửi đoạn mô tả.

Tham khảo: [ColPali](https://arxiv.org/abs/2407.01449). Xem thêm cách truyền ảnh trong [Messages](../../oss/python/langchain/messages.md).

## 4. RAG trên dữ liệu có cấu trúc

Câu hỏi "Doanh thu quý 3 của chi nhánh Hà Nội?" hay "Có bao nhiêu nhân viên phòng Kỹ thuật?" không nên trả lời bằng cách tìm chunk văn bản, mà bằng **truy vấn database**.

- **Text-to-SQL:** LLM sinh câu SQL từ câu hỏi, thực thi, rồi diễn giải kết quả. Xem [SQL Agent](../../oss/python/langchain/sql-agent.md) và [Custom SQL Agent](../../oss/python/langgraph/sql-agent.md).
- **Kết hợp:** agent có cả tool tìm tài liệu và tool SQL (như Bài 7), tự chọn nguồn phù hợp.
- **Bảng trong tài liệu:** với bảng lớn trong PDF, Excel, cân nhắc trích ra lưu vào SQL thay vì chunk thành văn bản.

!!! danger "Bảo mật Text-to-SQL"
    Luôn dùng tài khoản database **chỉ đọc**, giới hạn các bảng được phép truy cập, đặt timeout và giới hạn số dòng trả về. Với dữ liệu nhạy cảm, áp dụng row-level security ngay trong database.

## 5. Fine-tune embedding và reranker

Khi tài liệu dùng nhiều thuật ngữ chuyên ngành (y khoa, pháp lý, nội bộ công ty) mà model embedding chung không hiểu tốt, fine-tune trên dữ liệu của bạn có thể tăng đáng kể chất lượng retrieval.

Dữ liệu huấn luyện là các cặp **(câu hỏi, đoạn văn đúng)**. Bạn đã có sẵn cách sinh chúng ở [Bài 6](06-evaluation.md#23-nguon-cau-hoi) (`eval/synthesize.py`), chỉ cần sinh nhiều hơn (vài nghìn cặp).

```python title="finetune_embedding.py"
# uv add sentence-transformers datasets
import json

from datasets import Dataset
from sentence_transformers import SentenceTransformer, SentenceTransformerTrainer, losses

pairs = [json.loads(l) for l in open("train_pairs.jsonl", encoding="utf-8")]
train_dataset = Dataset.from_dict({
    "anchor": [p["question"] for p in pairs],
    "positive": [p["passage"] for p in pairs],
})

model = SentenceTransformer("BAAI/bge-m3")
# Mỗi câu hỏi được kéo gần đoạn văn đúng, đẩy xa các đoạn văn khác trong cùng batch
loss = losses.MultipleNegativesRankingLoss(model)

trainer = SentenceTransformerTrainer(model=model, train_dataset=train_dataset, loss=loss)
trainer.train()
model.save_pretrained("models/bge-m3-noi-bo")
```

!!! warning "Tách dữ liệu huấn luyện và đánh giá"
    **Không** dùng câu hỏi trong golden dataset để huấn luyện, nếu không điểm eval sẽ cao giả tạo. Sau khi fine-tune, chạy lại `compare_embeddings.py` (Bài 3) và `eval/run.py` (Bài 6) để xác nhận cải thiện là thật.

Reranker (cross-encoder) cũng fine-tune được theo cách tương tự với `CrossEncoder` của `sentence-transformers`.

## 6. Các kỹ thuật retrieval khác đáng biết

- **Late interaction (ColBERT):** thay vì nén cả đoạn văn thành một vector, giữ một vector cho **mỗi token**, so khớp chi tiết hơn. Chất lượng gần reranker nhưng vẫn tìm được trên tập lớn. Đổi lại tốn bộ nhớ hơn nhiều.
- **Sparse embedding học được (SPLADE, sparse của bge-m3):** giống BM25 nhưng trọng số từ khóa do model học, có mở rộng từ đồng nghĩa. Có thể thay BM25 trong hybrid search.
- **Late chunking:** embed cả tài liệu dài bằng model hỗ trợ context dài, rồi mới chia vector theo chunk. Mỗi chunk "biết" ngữ cảnh của cả tài liệu mà không cần gọi LLM như contextual retrieval.
- **Query routing theo độ khó:** câu hỏi dễ đi đường nhanh (2-Step RAG, model nhỏ), câu khó đi đường chậm (agent, model lớn). Tiết kiệm chi phí đáng kể khi phần lớn câu hỏi là câu dễ.

## 7. Cá nhân hóa và bộ nhớ dài hạn

Trợ lý có thể ghi nhớ thông tin về người dùng qua nhiều phiên (phòng ban, dự án đang làm, sở thích trình bày) để retrieval và câu trả lời phù hợp hơn. Xem [Long-term Memory](../../oss/python/langchain/long-term-memory.md) và [Stores](../../oss/python/langgraph/stores.md).

## 8. Từ RAG tới research agent

Với câu hỏi phức tạp cần tổng hợp từ hàng chục nguồn ("Soạn báo cáo so sánh chính sách phúc lợi của công ty với quy định pháp luật mới nhất"), một lần retrieve là không đủ. **Research agent** lập kế hoạch, chia nhỏ câu hỏi, giao cho các agent con tìm kiếm song song, rồi tổng hợp thành báo cáo có trích dẫn. Xem [Multi-agent](../../oss/python/langchain/multi-agent/index.md) và [Xây dựng Deep Agent từ đầu](../../oss/python/langchain/deep-agent-from-scratch.md).

## Bài tập

Chọn **một** hướng phù hợp nhất với điểm yếu mà eval của bạn chỉ ra:

**Bài 9.1 (Long context).** Cài đặt `retrieve_whole_documents` và so sánh với RAG theo chunk trên nhóm câu hỏi `multi_hop`. So sánh cả chất lượng và chi phí.

**Bài 9.2 (Multimodal).** Chọn một PDF có biểu đồ. Viết 5 câu hỏi về số liệu trong biểu đồ. So sánh pipeline văn bản thuần với pipeline có `describe_pdf_pages`.

**Bài 9.3 (Fine-tune).** Sinh 2.000 cặp câu hỏi và đoạn văn, fine-tune `bge-m3`, so sánh hit@5 trước và sau trên golden dataset.

**Bài 9.4 (GraphRAG).** Trích xuất graph từ một tập tài liệu có nhiều thực thể liên kết (sơ đồ tổ chức, danh sách dự án). Viết 5 câu hỏi cần đi qua ít nhất hai quan hệ, so sánh với RAG thông thường.

## Checklist

- [ ] Giải thích được khi nào nên nạp thẳng tài liệu, khi nào nên dùng RAG, và cách kết hợp hai cách.
- [ ] Biết GraphRAG giải quyết loại câu hỏi nào, và cái giá phải trả.
- [ ] Biết ít nhất một cách xử lý tài liệu có hình ảnh, biểu đồ.
- [ ] Đã thử ít nhất một hướng nâng cao và đo kết quả bằng eval.

## Đọc thêm

- [Microsoft GraphRAG](https://github.com/microsoft/graphrag)
- [ColPali: Efficient Document Retrieval with Vision Language Models](https://arxiv.org/abs/2407.01449)
- [Sentence Transformers: Training Overview](https://sbert.net/docs/sentence_transformer/training_overview.html)
- [Multi-agent](../../oss/python/langchain/multi-agent/index.md), [Long-term Memory](../../oss/python/langchain/long-term-memory.md)

---

**Bài trước:** [Bài 8: Production](08-production.md) · [Tổng quan](index.md)

!!! success "Hoàn thành lộ trình"
    Nếu đã đi hết 9 bài và làm các bài tập, bạn đã có một hệ thống RAG hoàn chỉnh: ingestion sạch, hybrid search có rerank, câu trả lời có trích dẫn, eval tự động, agent đa nguồn, phân quyền và API production. Bước tiếp theo là làm [dự án tổng kết](index.md#du-an-tong-ket) với dữ liệu thật và đưa vào portfolio.
