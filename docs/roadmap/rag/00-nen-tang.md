# Bài 0: Nền tảng

!!! abstract "Tóm tắt bài học"
    - **Mục tiêu:** hiểu vì sao RAG tồn tại, nắm ba khái niệm nền tảng (token, embedding, tìm kiếm từ khóa) và chuẩn bị môi trường cho cả lộ trình.
    - **Thời lượng:** khoảng 1 tuần.
    - **Yêu cầu trước:** biết Python cơ bản, có API key của ít nhất một nhà cung cấp LLM.

## 1. Vì sao cần RAG?

LLM được huấn luyện trên một lượng dữ liệu khổng lồ, nhưng có hai giới hạn không thể tránh:

1. **Kiến thức bị đóng băng.** Model chỉ biết những gì có trong dữ liệu huấn luyện, tính đến một thời điểm cắt (knowledge cutoff). Nó không biết chính sách công ty bạn vừa ban hành tuần trước.
2. **Context hữu hạn.** Mỗi lần gọi, bạn chỉ đưa vào được một lượng văn bản giới hạn (context window). Dù context ngày càng dài, việc nhét toàn bộ 10.000 tài liệu vào mọi request vẫn quá chậm và quá đắt.

**RAG (Retrieval-Augmented Generation)** giải quyết cả hai: tại thời điểm người dùng hỏi, hệ thống **tìm** (retrieve) vài đoạn tài liệu liên quan nhất, rồi đưa chúng vào prompt để LLM **sinh** (generate) câu trả lời dựa trên đó.

### So sánh với các cách tiếp cận khác

| Cách tiếp cận | Làm gì | Ưu điểm | Nhược điểm |
|---|---|---|---|
| Prompt thuần | Chỉ hỏi LLM | Đơn giản nhất | Không biết dữ liệu riêng, dễ bịa |
| Long context | Nạp toàn bộ tài liệu vào prompt | Không cần hạ tầng tìm kiếm | Đắt, chậm, giới hạn dung lượng, khó phân quyền |
| **RAG** | Tìm đoạn liên quan rồi nạp vào prompt | Cập nhật tức thì, trích dẫn được nguồn, rẻ | Chất lượng phụ thuộc bước tìm kiếm |
| Fine-tuning | Huấn luyện thêm model trên dữ liệu riêng | Học được văn phong, định dạng | Tốn kém, khó cập nhật, không trích dẫn được nguồn |

!!! tip "Quy tắc nhớ nhanh"
    Fine-tuning dạy model **cách** trả lời (văn phong, định dạng). RAG cho model **biết** nội dung để trả lời. Hai kỹ thuật này bổ trợ nhau, không thay thế nhau.

## 2. Chuẩn bị môi trường

Cả lộ trình dùng chung một dự án tên `rag-lab`. Mình khuyến nghị dùng [uv](https://docs.astral.sh/uv/) để quản lý Python và thư viện.

```bash
uv init rag-lab
cd rag-lab
uv add langchain langchain-openai langchain-anthropic langchain-community \
       langchain-text-splitters pypdf numpy rank-bm25 python-dotenv
```

Tạo file `.env` ở thư mục gốc:

```bash
ANTHROPIC_API_KEY=sk-ant-...
OPENAI_API_KEY=sk-...          # dùng cho embedding
LANGSMITH_API_KEY=lsv2_...     # không bắt buộc, dùng để trace từ Bài 6
LANGSMITH_TRACING=true
```

!!! note "Về lựa chọn model"
    Lộ trình dùng Claude cho bước sinh câu trả lời và OpenAI cho embedding (Anthropic không cung cấp model embedding). Bạn có thể thay bằng bất kỳ nhà cung cấp nào LangChain hỗ trợ, chỉ cần đổi chuỗi tên model. Xem [Models](../../oss/python/langchain/models.md).

## 3. Token và context window

LLM không đọc chữ, mà đọc **token**: các mảnh từ có độ dài trung bình khoảng 3 đến 4 ký tự tiếng Anh. Tiếng Việt có dấu thường tốn **nhiều token hơn** tiếng Anh cho cùng một nội dung, điều này ảnh hưởng trực tiếp tới chi phí và lượng context bạn dùng được.

Chạy đoạn sau để xem số token thực tế:

```python
from dotenv import load_dotenv
from langchain.chat_models import init_chat_model

load_dotenv()
model = init_chat_model("anthropic:claude-sonnet-5")

for text in [
    "Employees are entitled to 12 days of annual leave.",
    "Nhân viên được nghỉ phép 12 ngày mỗi năm.",
]:
    response = model.invoke(f"Nhắc lại nguyên văn: {text}")
    print(text)
    print("  ->", response.usage_metadata)
```

`usage_metadata` cho biết `input_tokens` và `output_tokens`. Nhân với bảng giá của nhà cung cấp là ra chi phí mỗi request.

**Ba điều cần rút ra:**

- Mỗi chunk bạn đưa vào context đều **tốn tiền** và **làm chậm** câu trả lời. Retrieval tốt nghĩa là đưa vào ít nhưng đúng.
- Context dài không có nghĩa là model đọc kỹ tất cả. Nghiên cứu [Lost in the Middle](https://arxiv.org/abs/2307.03172) cho thấy thông tin nằm giữa context dài dễ bị bỏ qua.
- Với nhiều model, tham số `temperature` thấp (0 đến 0.3) giúp câu trả lời bám sát tài liệu hơn. Lưu ý: các model Claude thế hệ mới nhất (Sonnet 5, Opus 5) **không còn nhận** `temperature` và sẽ báo lỗi nếu bạn truyền vào; độ "kỹ" của câu trả lời được điều chỉnh bằng tham số `effort`. Model nhỏ như Haiku 4.5 vẫn nhận `temperature`. Xem thêm ở [track Hiểu LLM](../hieu-llm/02-sampling-reasoning.md).

## 4. Embedding và độ tương đồng

**Embedding** là một vector số thực (thường 384 đến 3072 chiều) biểu diễn ý nghĩa của một đoạn văn bản. Hai đoạn có nghĩa gần nhau sẽ có vector gần nhau, **kể cả khi không dùng chung từ nào**.

Độ "gần" thường đo bằng **cosine similarity**, tức cosin của góc giữa hai vector:

```text
cosine(a, b) = (a · b) / (‖a‖ × ‖b‖)
```

Giá trị gần 1 nghĩa là cùng hướng (rất giống nhau), gần 0 nghĩa là không liên quan.

```python
import numpy as np
from dotenv import load_dotenv
from langchain.embeddings import init_embeddings

load_dotenv()
embeddings = init_embeddings("openai:text-embedding-3-small")

sentences = [
    "Nhân viên được nghỉ phép 12 ngày mỗi năm.",
    "Mỗi năm người lao động có bao nhiêu ngày phép?",
    "Công ty hỗ trợ tiền gửi xe hàng tháng.",
    "Annual leave entitlement is twelve days.",
    "Giá vàng hôm nay tăng mạnh.",
]
vectors = np.array(embeddings.embed_documents(sentences))
print("Số chiều:", vectors.shape[1])

# Chuẩn hóa rồi nhân ma trận = cosine similarity của mọi cặp câu
normalized = vectors / np.linalg.norm(vectors, axis=1, keepdims=True)
similarity = normalized @ normalized.T

for i, s in enumerate(sentences):
    print(f"{similarity[0, i]:.3f}  {s}")
```

Bạn sẽ thấy câu 2 (hỏi về ngày phép, khác hẳn từ ngữ) và câu 4 (tiếng Anh) có điểm cao hơn nhiều so với câu 5. Đó chính là sức mạnh của **semantic search**.

## 5. Tìm kiếm từ khóa với BM25

Trước khi có embedding, công cụ tìm kiếm dùng **BM25**: chấm điểm tài liệu dựa trên việc từ trong câu hỏi xuất hiện bao nhiêu lần trong tài liệu, có điều chỉnh theo độ hiếm của từ (từ hiếm như "HĐ-2025-017" có trọng số cao hơn từ phổ biến như "của") và độ dài tài liệu.

```python
from rank_bm25 import BM25Okapi

corpus = [
    "Hợp đồng HĐ-2025-017 ký với công ty ABC về cung cấp thiết bị.",
    "Quy trình ký kết hợp đồng mua bán gồm ba bước.",
    "Nhân viên được nghỉ phép 12 ngày mỗi năm.",
]
tokenized = [doc.lower().split() for doc in corpus]
bm25 = BM25Okapi(tokenized)

for query in ["hđ-2025-017", "ngày nghỉ hằng năm"]:
    scores = bm25.get_scores(query.lower().split())
    print(query, "->", [round(s, 2) for s in scores])
```

Quan sát kết quả:

- Với mã hợp đồng `hđ-2025-017`, BM25 tìm chính xác ngay lập tức. Embedding thường **trượt** loại truy vấn này vì mã số không mang "ý nghĩa".
- Với "ngày nghỉ hằng năm", BM25 chỉ khớp được chữ "ngày" và "năm", trong khi embedding hiểu đây là câu hỏi về nghỉ phép.

| | BM25 (từ khóa) | Embedding (ngữ nghĩa) |
|---|---|---|
| Mã sản phẩm, số hiệu, tên riêng | Rất tốt | Kém |
| Từ đồng nghĩa, diễn đạt khác | Kém | Rất tốt |
| Đa ngôn ngữ | Không | Có (với model đa ngôn ngữ) |
| Chi phí | Gần như bằng 0 | Tốn chi phí embedding |

Vì mỗi bên mạnh một kiểu, hệ thống production thường **kết hợp cả hai** (hybrid search). Bạn sẽ làm điều này ở [Bài 4](04-retrieval-nang-cao.md).

!!! warning "Tiếng Việt và BM25"
    Tách từ bằng khoảng trắng (`split()`) làm "hợp đồng" thành hai từ rời "hợp" và "đồng", giảm độ chính xác. Bài 4 sẽ dùng thư viện tách từ tiếng Việt để khắc phục.

## Bài tập

**Bài 0.1.** Viết hàm `cosine_similarity(a, b)` chỉ dùng NumPy, kiểm tra kết quả khớp với ma trận `similarity` ở mục 4.

??? tip "Gợi ý lời giải"
    ```python
    def cosine_similarity(a: np.ndarray, b: np.ndarray) -> float:
        return float(a @ b / (np.linalg.norm(a) * np.linalg.norm(b)))

    assert abs(cosine_similarity(vectors[0], vectors[1]) - similarity[0, 1]) < 1e-6
    ```

**Bài 0.2.** Chọn 10 câu tiếng Việt thuộc 3 chủ đề khác nhau. Tính ma trận similarity và kiểm tra: các câu cùng chủ đề có điểm cao hơn câu khác chủ đề không? Có cặp nào bất ngờ không?

**Bài 0.3.** Đo số token của cùng một đoạn văn 500 chữ bằng tiếng Anh và tiếng Việt. Chênh lệch bao nhiêu phần trăm?

??? tip "Gợi ý lời giải"
    Gửi mỗi đoạn văn trong một request riêng, lấy `response.usage_metadata["input_tokens"]` trừ đi số token của phần prompt cố định. Thông thường tiếng Việt tốn nhiều token hơn đáng kể, nên khi ước tính chi phí cho người dùng Việt, đừng dùng số liệu benchmark tiếng Anh.

## Checklist

- [ ] Giải thích được cho người khác vì sao không nên nhét toàn bộ tài liệu công ty vào prompt.
- [ ] Phân biệt được khi nào dùng RAG, khi nào dùng fine-tuning.
- [ ] Giải thích được cosine similarity bằng hình học (góc giữa hai vector).
- [ ] Biết BM25 mạnh ở đâu và yếu ở đâu so với embedding.
- [ ] Dự án `rag-lab` đã chạy được, gọi được cả chat model và embedding model.

## Đọc thêm

- Trong site: [Models](../../oss/python/langchain/models.md), [Messages](../../oss/python/langchain/messages.md), [Retrieval](../../oss/python/langchain/retrieval.md).
- Bài báo gốc: [Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks](https://arxiv.org/abs/2005.11401).

---

[Tổng quan lộ trình](index.md) · **Bài tiếp theo:** [Bài 1: Naive RAG](01-naive-rag.md)
