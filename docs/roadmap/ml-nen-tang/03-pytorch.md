# Bài 3: Deep learning với PyTorch

!!! abstract "Tóm tắt bài học"
    - **Mục tiêu:** làm quen PyTorch: tensor, autograd, `nn.Module`, DataLoader; viết vòng lặp huấn luyện chuẩn có validation và dừng sớm; chạy trên GPU; hiểu các kỹ thuật chống overfitting.
    - **Thời lượng:** 1 đến 2 tuần.
    - **Yêu cầu trước:** [Bài 2](02-ml-co-dien.md).

```bash
uv add torch numpy scikit-learn
```

## 1. Tensor

Tensor là mảng nhiều chiều, giống mảng NumPy, nhưng chạy được trên **GPU** và **tự tính đạo hàm**.

```python
import torch

x = torch.tensor([[1.0, 2.0], [3.0, 4.0]])
print(x.shape, x.dtype)          # torch.Size([2, 2]) torch.float32
print(x @ x.T)                   # nhân ma trận

device = "cuda" if torch.cuda.is_available() else "mps" if torch.backends.mps.is_available() else "cpu"
x = x.to(device)                 # chuyển sang GPU nếu có
print(device)
```

Quy ước hình dạng hay gặp: `(batch, features)` cho dữ liệu dạng bảng, `(batch, seq_len, hidden)` cho văn bản trong Transformer.

## 2. Autograd: đạo hàm tự động

Ở [Bài 1](01-toan-can-thiet.md#52-tu-cai-at), bạn tính đạo hàm bằng tay. PyTorch làm việc đó tự động:

```python
w = torch.tensor(0.0, requires_grad=True)
b = torch.tensor(0.0, requires_grad=True)
x = torch.rand(100) * 10
y = 3 * x + 5 + torch.randn(100)

optimizer = torch.optim.SGD([w, b], lr=0.01)
for step in range(2000):
    loss = ((w * x + b - y) ** 2).mean()
    optimizer.zero_grad()   # xóa gradient cũ
    loss.backward()         # tính gradient của loss theo mọi tham số, tự động
    optimizer.step()        # cập nhật: tham_số -= lr * gradient

print(w.item(), b.item())   # xấp xỉ 3 và 5
```

Bốn dòng `loss`, `zero_grad`, `backward`, `step` là **trái tim của mọi vòng lặp huấn luyện**, từ model 2 tham số tới LLM hàng trăm tỉ tham số.

## 3. Xây model với `nn.Module`

Bài toán: phân loại phản hồi khách hàng (3 lớp) từ **embedding** đã tính ở [Bài 2](02-ml-co-dien.md#cai-tien-embedding-model-on-gian). Model là một mạng MLP nhỏ:

```python title="mlp.py"
import torch
from torch import nn


class FeedbackClassifier(nn.Module):
    def __init__(self, input_dim: int, hidden_dim: int = 256, num_classes: int = 3, dropout: float = 0.3):
        super().__init__()
        self.net = nn.Sequential(
            nn.Linear(input_dim, hidden_dim),   # x @ W + b
            nn.ReLU(),                          # hàm kích hoạt phi tuyến
            nn.Dropout(dropout),                # tắt ngẫu nhiên 30% nơ-ron khi huấn luyện
            nn.Linear(hidden_dim, num_classes), # ra 3 logits
        )

    def forward(self, x: torch.Tensor) -> torch.Tensor:
        return self.net(x)  # logits; softmax nằm trong hàm loss
```

- **Hàm kích hoạt phi tuyến** (ReLU) là thứ cho phép mạng học quy luật phức tạp. Không có nó, chồng bao nhiêu lớp tuyến tính cũng chỉ tương đương một lớp.
- `nn.CrossEntropyLoss` nhận **logits** và tự áp dụng softmax bên trong, ổn định số học hơn.

## 4. Vòng lặp huấn luyện chuẩn

```python title="train.py"
import numpy as np
import torch
from sklearn.model_selection import train_test_split
from torch import nn
from torch.utils.data import DataLoader, TensorDataset

from mlp import FeedbackClassifier

device = "cuda" if torch.cuda.is_available() else "cpu"
torch.manual_seed(42)

# embeddings.npy: (N, D) ; labels.npy: (N,) nhãn số 0, 1, 2
X = np.load("embeddings.npy").astype("float32")
y = np.load("labels.npy").astype("int64")
X_train, X_val, y_train, y_val = train_test_split(X, y, test_size=0.2, stratify=y, random_state=42)

train_loader = DataLoader(TensorDataset(torch.from_numpy(X_train), torch.from_numpy(y_train)),
                          batch_size=64, shuffle=True)
val_X, val_y = torch.from_numpy(X_val).to(device), torch.from_numpy(y_val).to(device)

model = FeedbackClassifier(input_dim=X.shape[1]).to(device)
loss_fn = nn.CrossEntropyLoss()
optimizer = torch.optim.AdamW(model.parameters(), lr=1e-3, weight_decay=1e-2)

best_val_loss, patience, bad_epochs = float("inf"), 5, 0
for epoch in range(100):
    model.train()  # bật dropout
    for xb, yb in train_loader:
        xb, yb = xb.to(device), yb.to(device)
        loss = loss_fn(model(xb), yb)
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()

    model.eval()  # tắt dropout khi đánh giá
    with torch.no_grad():
        val_logits = model(val_X)
        val_loss = loss_fn(val_logits, val_y).item()
        val_acc = (val_logits.argmax(dim=1) == val_y).float().mean().item()
    print(f"epoch {epoch:3d}  train_loss={loss.item():.3f}  val_loss={val_loss:.3f}  val_acc={val_acc:.3f}")

    # Dừng sớm: lưu model tốt nhất, dừng khi val_loss không cải thiện sau 5 epoch
    if val_loss < best_val_loss:
        best_val_loss, bad_epochs = val_loss, 0
        torch.save(model.state_dict(), "best_model.pt")
    else:
        bad_epochs += 1
        if bad_epochs >= patience:
            print("Dừng sớm")
            break

model.load_state_dict(torch.load("best_model.pt"))
```

| Thành phần | Vai trò |
|---|---|
| **Batch** (64 mẫu) | Tính gradient trên một nhóm nhỏ thay vì toàn bộ dữ liệu: nhanh hơn, vừa bộ nhớ GPU |
| **Epoch** | Một lượt đi qua toàn bộ dữ liệu train |
| **AdamW** | Optimizer phổ biến nhất hiện nay: tự điều chỉnh bước cho từng tham số, kèm weight decay |
| `model.train()` / `model.eval()` | Bật tắt các hành vi chỉ dùng khi huấn luyện (dropout) |
| `torch.no_grad()` | Không lưu thông tin tính gradient khi đánh giá: nhanh, ít bộ nhớ |

## 5. Chống overfitting

Quan sát log: nếu `train_loss` tiếp tục giảm mà `val_loss` bắt đầu tăng, model đang overfit.

| Kỹ thuật | Cách hoạt động |
|---|---|
| **Dừng sớm** | Dừng khi điểm validation ngừng cải thiện, giữ model tốt nhất |
| **Dropout** | Tắt ngẫu nhiên một phần nơ-ron mỗi bước, buộc mạng không phụ thuộc vào vài nơ-ron |
| **Weight decay** | Phạt trọng số lớn, giữ model "đơn giản" |
| **Thêm dữ liệu** | Hiệu quả nhất, nếu có thể |
| **Model nhỏ hơn** | Ít tham số hơn thì ít khả năng học thuộc |

## 6. GPU và bộ nhớ

- Model và dữ liệu phải **cùng thiết bị** (`.to(device)`). Lỗi "Expected all tensors to be on the same device" là lỗi phổ biến nhất của người mới.
- Hết bộ nhớ GPU ("CUDA out of memory"): giảm `batch_size`, dùng model nhỏ hơn, hoặc dùng độ chính xác thấp hơn (`bfloat16`).
- Không có GPU: Google Colab và Kaggle cung cấp GPU miễn phí có giới hạn thời gian, đủ cho các bài tập trong track này.

## Bài tập

**Bài 3.1.** Chạy lại ví dụ gradient descent của Bài 1 bằng autograd. So sánh kết quả với bản NumPy.

**Bài 3.2.** Huấn luyện `FeedbackClassifier` trên embedding của dữ liệu ở Bài 2. So sánh F1 của lớp khiếu nại với logistic regression. Mạng neural có tốt hơn không? Vì sao có thể không?

??? tip "Gợi ý"
    Với embedding chất lượng tốt và dữ liệu vài nghìn mẫu, logistic regression thường đã rất tốt; MLP có thể chỉ nhỉnh hơn một chút, hoặc overfit. Đây là bài học quan trọng: **model phức tạp hơn không tự động tốt hơn**, nhất là khi dữ liệu ít.

**Bài 3.3.** Tắt dropout và weight decay, tăng `hidden_dim` lên 2048, bỏ dừng sớm và chạy 100 epoch. Vẽ biểu đồ `train_loss` và `val_loss` theo epoch. Chỉ ra thời điểm bắt đầu overfit.

**Bài 3.4.** Thêm lớp trọng số cho `CrossEntropyLoss` (tham số `weight`) để bù cho lớp khiếu nại hiếm. Recall của lớp này thay đổi thế nào?

## Checklist

- [ ] Dùng được tensor, chuyển dữ liệu giữa CPU và GPU.
- [ ] Giải thích được bốn bước `loss`, `zero_grad`, `backward`, `step`.
- [ ] Viết được vòng lặp huấn luyện có validation, dừng sớm, lưu model tốt nhất.
- [ ] Nhận biết overfitting từ đường loss, biết các kỹ thuật chống.

## Đọc thêm

- [PyTorch Tutorials: Learn the Basics](https://pytorch.org/tutorials/beginner/basics/intro.html)
- [Dive into Deep Learning](https://d2l.ai/) (sách miễn phí, có code PyTorch)

---

**Bài trước:** [Bài 2: ML cổ điển](02-ml-co-dien.md) · [Tổng quan track](index.md) · **Bài tiếp theo:** [Bài 4: Transformer và Hugging Face](04-transformers-hf.md)
