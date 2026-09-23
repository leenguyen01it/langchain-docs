# Bài 4: Transformer và Hugging Face

!!! abstract "Tóm tắt bài học"
    - **Mục tiêu:** tự cài đặt self-attention và một khối Transformer bằng PyTorch để hiểu tận gốc; phân biệt các kiến trúc encoder, decoder; sử dụng hệ sinh thái Hugging Face; biết các lựa chọn model cho tiếng Việt.
    - **Thời lượng:** khoảng 1 tuần.
    - **Yêu cầu trước:** [Bài 3](03-pytorch.md), [Hiểu LLM, Bài 1](../hieu-llm/01-transformer-token.md).

## 1. Self-attention từ con số 0

Nhắc lại ý tưởng ([Hiểu LLM, Bài 1](../hieu-llm/01-transformer-token.md#4-attention-moi-token-nhin-cac-token-khac)): mỗi token tạo ra **Query** (tôi tìm gì), **Key** (tôi có gì), **Value** (tôi đóng góp gì). Độ chú ý giữa token i và j là độ khớp giữa Query của i và Key của j.

```text
Attention(Q, K, V) = softmax(Q Kᵀ / √d) V
```

```python title="attention.py"
import math

import torch
from torch import nn
import torch.nn.functional as F


class CausalSelfAttention(nn.Module):
    def __init__(self, d_model: int, n_heads: int):
        super().__init__()
        assert d_model % n_heads == 0
        self.n_heads, self.d_head = n_heads, d_model // n_heads
        self.qkv = nn.Linear(d_model, 3 * d_model)   # chiếu ra Q, K, V cùng lúc
        self.out = nn.Linear(d_model, d_model)

    def forward(self, x: torch.Tensor) -> torch.Tensor:
        B, T, C = x.shape                              # batch, số token, số chiều
        q, k, v = self.qkv(x).split(C, dim=2)
        # Tách thành nhiều head: (B, n_heads, T, d_head)
        q, k, v = (t.view(B, T, self.n_heads, self.d_head).transpose(1, 2) for t in (q, k, v))

        scores = q @ k.transpose(-2, -1) / math.sqrt(self.d_head)   # (B, h, T, T)
        # Causal mask: token i chỉ được nhìn các token 0..i (không nhìn "tương lai")
        mask = torch.triu(torch.ones(T, T, dtype=torch.bool, device=x.device), diagonal=1)
        scores = scores.masked_fill(mask, float("-inf"))
        weights = F.softmax(scores, dim=-1)            # mỗi hàng tổng bằng 1
        y = weights @ v                                # trộn Value theo trọng số chú ý

        y = y.transpose(1, 2).contiguous().view(B, T, C)   # gộp các head
        return self.out(y)
```

Những điểm then chốt:

- **Chia cho √d**: không có bước này, tích vô hướng quá lớn khi số chiều cao, softmax trở nên "cực đoan" và khó huấn luyện.
- **Causal mask**: model sinh văn bản (GPT, Claude, Llama) chỉ được nhìn các token đứng trước, vì khi sinh, token tương lai chưa tồn tại.
- **Nhiều head**: mỗi head học một kiểu quan hệ khác nhau (ngữ pháp, tham chiếu, vị trí...).
- Ma trận `scores` có kích thước `T × T`: chi phí attention **tăng theo bình phương** độ dài chuỗi. Đó là một lý do context dài đắt đỏ.

### Một khối Transformer

```python title="attention.py (tiếp)"
class TransformerBlock(nn.Module):
    def __init__(self, d_model: int, n_heads: int):
        super().__init__()
        self.ln1, self.ln2 = nn.LayerNorm(d_model), nn.LayerNorm(d_model)
        self.attn = CausalSelfAttention(d_model, n_heads)
        self.mlp = nn.Sequential(
            nn.Linear(d_model, 4 * d_model), nn.GELU(), nn.Linear(4 * d_model, d_model)
        )

    def forward(self, x: torch.Tensor) -> torch.Tensor:
        x = x + self.attn(self.ln1(x))   # residual: cộng lại đầu vào
        x = x + self.mlp(self.ln2(x))
        return x


block = TransformerBlock(d_model=64, n_heads=4)
x = torch.randn(2, 10, 64)   # 2 chuỗi, mỗi chuỗi 10 token, mỗi token 64 chiều
print(block(x).shape)         # torch.Size([2, 10, 64])
```

Một LLM là: **embedding token** → **N khối như trên** chồng lên nhau (vài chục tới hơn trăm khối) → **lớp chiếu ra từ vựng** (logits cho mọi token). Toàn bộ kiến trúc chỉ có vậy; sức mạnh đến từ quy mô và dữ liệu.

## 2. Ba họ kiến trúc

| Kiến trúc | Attention | Ví dụ | Giỏi |
|---|---|---|---|
| **Encoder** | Hai chiều (nhìn cả trước và sau) | BERT, RoBERTa, PhoBERT, XLM-R | Hiểu văn bản: phân loại, trích xuất thực thể, **embedding**, reranker |
| **Decoder** | Một chiều (causal) | GPT, Claude, Llama, Qwen | **Sinh văn bản**, trò chuyện, suy luận, agent |
| **Encoder-decoder** | Encoder hai chiều, decoder một chiều | T5, BART | Biến đổi văn bản: dịch, tóm tắt |

Model embedding và reranker trong RAG thường là **encoder**; LLM bạn gọi qua API là **decoder**.

## 3. Hệ sinh thái Hugging Face

| Thành phần | Vai trò |
|---|---|
| **Hub** (huggingface.co) | Kho hàng trăm nghìn model, dataset mở |
| `transformers` | Tải và chạy model: `AutoTokenizer`, `AutoModel...`, `pipeline` |
| `datasets` | Tải, xử lý dataset hiệu quả |
| `sentence-transformers` | Embedding, reranker, fine-tune embedding |
| `peft`, `trl` | Fine-tune hiệu quả (LoRA), huấn luyện LLM (Bài 5) |
| `safetensors` | Định dạng lưu trọng số an toàn (không chạy code khi tải) |

### 3.1 Pipeline: dùng model trong vài dòng

```python
from transformers import pipeline

fill = pipeline("fill-mask", model="xlm-roberta-base")
for r in fill("Thủ đô của Việt Nam là <mask>."):
    print(f"{r['score']:.2f}  {r['token_str']}")
```

### 3.2 Embedding với sentence-transformers

```python
from sentence_transformers import SentenceTransformer

model = SentenceTransformer("BAAI/bge-m3")
vectors = model.encode(["giao hàng chậm quá", "đợi mãi không thấy hàng", "áo rất đẹp"],
                       normalize_embeddings=True)
print(vectors @ vectors.T)  # hai câu đầu có độ tương đồng cao
```

### 3.3 Fine-tune một encoder để phân loại

Với bài toán phân loại có vài nghìn mẫu, fine-tune một encoder đa ngôn ngữ thường vượt TF-IDF và cạnh tranh được với LLM, trong khi chạy nhanh và rẻ hơn nhiều:

```python title="finetune_encoder.py"
from datasets import load_dataset
from transformers import (
    AutoModelForSequenceClassification,
    AutoTokenizer,
    Trainer,
    TrainingArguments,
)

LABELS = ["khen", "hoi_dap", "khieu_nai"]
ds = load_dataset("csv", data_files="feedback.csv")["train"].train_test_split(test_size=0.2, seed=42)

name = "xlm-roberta-base"
tokenizer = AutoTokenizer.from_pretrained(name)
model = AutoModelForSequenceClassification.from_pretrained(name, num_labels=len(LABELS))


def preprocess(batch):
    enc = tokenizer(batch["text"], truncation=True, max_length=128)
    enc["labels"] = [LABELS.index(l) for l in batch["label"]]
    return enc


ds = ds.map(preprocess, batched=True, remove_columns=ds["train"].column_names)

trainer = Trainer(
    model=model,
    args=TrainingArguments(
        output_dir="out-classifier",
        num_train_epochs=3,
        per_device_train_batch_size=16,
        learning_rate=2e-5,
        eval_strategy="epoch",
    ),
    train_dataset=ds["train"],
    eval_dataset=ds["test"],
    processing_class=tokenizer,
)
trainer.train()
```

Learning rate khi fine-tune (2e-5) **nhỏ hơn nhiều** so với huấn luyện từ đầu, vì ta chỉ muốn điều chỉnh nhẹ những gì model đã học.

## 4. Model cho tiếng Việt

| Nhu cầu | Lựa chọn tham khảo | Ghi chú |
|---|---|---|
| Encoder tiếng Việt | PhoBERT (`vinai/phobert-base`) | Huấn luyện riêng cho tiếng Việt; **yêu cầu tách từ** trước (ví dụ bằng VnCoreNLP, underthesea) |
| Encoder đa ngôn ngữ | XLM-RoBERTa | Không cần tách từ, dùng ngay |
| Embedding | `BAAI/bge-m3`, `multilingual-e5` | Xem [RAG, Bài 3](../rag/03-embeddings-vector-db.md) |
| LLM mở | Qwen, Gemma, Llama (bản đa ngôn ngữ), và các model do cộng đồng Việt Nam fine-tune | Chất lượng tiếng Việt khác nhau nhiều, **tự đánh giá** trên tác vụ của bạn |

Luôn kiểm tra **giấy phép** (license) của model trước khi dùng cho sản phẩm thương mại.

## Bài tập

**Bài 4.1.** Chạy `CausalSelfAttention` với `T=5`, in ma trận `weights` của một head. Xác nhận các phần tử phía trên đường chéo bằng 0 và mỗi hàng tổng bằng 1.

**Bài 4.2.** Bỏ phép chia `√d`, khởi tạo `d_model=1024`, quan sát phân phối của `weights` (gần như one-hot?). Giải thích vì sao điều này gây khó khăn khi huấn luyện.

**Bài 4.3.** Fine-tune `xlm-roberta-base` trên dữ liệu phân loại ở Bài 2. So sánh F1 lớp khiếu nại với TF-IDF + LogReg, embedding + LogReg, và LLM.

**Bài 4.4 (mở rộng).** Làm theo [nanoGPT](https://github.com/karpathy/nanoGPT) của Andrej Karpathy: huấn luyện một GPT nhỏ trên một tập văn bản tiếng Việt (ví dụ tuyển tập truyện ngắn không còn bản quyền), sinh thử văn bản.

## Checklist

- [ ] Tự cài đặt được causal self-attention nhiều head, giải thích từng bước.
- [ ] Giải thích được vai trò của √d, causal mask, residual, LayerNorm.
- [ ] Phân biệt encoder, decoder, encoder-decoder và ứng dụng của từng loại.
- [ ] Dùng được `pipeline`, `sentence-transformers`, `Trainer` của Hugging Face.
- [ ] Biết các lựa chọn model cho tiếng Việt và lưu ý khi dùng.

## Đọc thêm

- [The Illustrated Transformer](https://jalammar.github.io/illustrated-transformer/)
- Andrej Karpathy: video "Let's build GPT: from scratch, in code, spelled out".
- [Hugging Face LLM Course](https://huggingface.co/learn)

---

**Bài trước:** [Bài 3: Deep learning với PyTorch](03-pytorch.md) · [Tổng quan track](index.md) · **Bài tiếp theo:** [Bài 5: Fine-tuning với LoRA](05-fine-tuning.md)
