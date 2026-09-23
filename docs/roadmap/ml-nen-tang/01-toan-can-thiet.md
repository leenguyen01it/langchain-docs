# Bài 1: Toán cần thiết

!!! abstract "Tóm tắt bài học"
    - **Mục tiêu:** nắm đủ đại số tuyến tính, xác suất và giải tích để hiểu embedding, attention, hàm mất mát và cách model học; tự cài đặt gradient descent bằng NumPy.
    - **Thời lượng:** khoảng 1 tuần.
    - **Yêu cầu trước:** toán phổ thông, Python cơ bản.

Bài này ưu tiên **trực giác và code** hơn là chứng minh. Mỗi khái niệm đều gắn với chỗ nó xuất hiện trong AI.

## 1. Vector

Vector là một danh sách số. Trong AI, vector biểu diễn **mọi thứ**: một từ, một câu, một ảnh, một người dùng.

```python
import numpy as np

a = np.array([0.2, 0.9, 0.1])
b = np.array([0.3, 0.8, 0.0])

print(a + b)            # cộng từng phần tử
print(2 * a)            # nhân với một số
print(np.linalg.norm(a))  # độ dài (norm): căn bậc hai của tổng bình phương
```

### Tích vô hướng (dot product)

`a · b = a₁b₁ + a₂b₂ + ... + aₙbₙ`. Về hình học, nó bằng `‖a‖ × ‖b‖ × cos(góc giữa a và b)`: hai vector càng cùng hướng, tích càng lớn.

**Xuất hiện ở đâu:** cosine similarity trong tìm kiếm ngữ nghĩa ([RAG, Bài 0](../rag/00-nen-tang.md#4-embedding-va-o-tuong-ong)), độ "chú ý" giữa hai token trong attention (Bài 4).

```python
cosine = a @ b / (np.linalg.norm(a) * np.linalg.norm(b))
print(f"{cosine:.3f}")  # gần 1: rất giống nhau
```

## 2. Ma trận

Ma trận là bảng số. Nhân một vector với ma trận là **biến đổi** vector đó sang một không gian khác (xoay, co giãn, chiếu, đổi số chiều).

```python
W = np.random.randn(3, 2)   # ma trận 3x2: biến vector 3 chiều thành 2 chiều
x = np.array([0.2, 0.9, 0.1])
print(x @ W)                # vector 2 chiều

# Xử lý nhiều vector cùng lúc (một "batch"): mỗi hàng là một vector
X = np.random.randn(5, 3)   # 5 vector, mỗi vector 3 chiều
print((X @ W).shape)        # (5, 2)
```

**Xuất hiện ở đâu:** mỗi lớp của mạng neural về cơ bản là `output = activation(input @ W + b)`. Một LLM có hàng tỉ tham số chính là hàng tỉ con số nằm trong các ma trận `W` như vậy. GPU mạnh vì nhân ma trận cực nhanh.

## 3. Xác suất và softmax

Model phân loại và LLM đều trả về **phân phối xác suất**: các số không âm, tổng bằng 1.

**Softmax** biến một dãy điểm bất kỳ (logits) thành phân phối xác suất:

```text
softmax(zᵢ) = exp(zᵢ) / Σⱼ exp(zⱼ)
```

```python
def softmax(z: np.ndarray) -> np.ndarray:
    e = np.exp(z - z.max())  # trừ max để tránh tràn số
    return e / e.sum()


logits = np.array([2.0, 1.0, 0.1])
print(softmax(logits))  # [0.66, 0.24, 0.10]
```

**Xuất hiện ở đâu:** lớp cuối của LLM (chọn token tiếp theo, [Hiểu LLM, Bài 2](../hieu-llm/02-sampling-reasoning.md)), trọng số attention, model phân loại.

## 4. Hàm mất mát: cross-entropy

Model học bằng cách **giảm một con số đo mức sai**, gọi là hàm mất mát (loss). Với phân loại và LLM, đó là **cross-entropy**: `loss = -log(xác suất model gán cho đáp án đúng)`.

| Xác suất gán cho đáp án đúng | Loss |
|---|---|
| 0,99 | 0,01 (gần như không phạt) |
| 0,5 | 0,69 |
| 0,01 | 4,6 (phạt rất nặng) |

```python
def cross_entropy(probs: np.ndarray, correct_index: int) -> float:
    return float(-np.log(probs[correct_index]))


probs = softmax(np.array([2.0, 1.0, 0.1]))
print(cross_entropy(probs, 0))  # đoán đúng lớp 0 với xác suất 0.66: loss thấp
print(cross_entropy(probs, 2))  # đáp án thật là lớp 2: loss cao
```

**Pretraining LLM** chính là giảm cross-entropy của việc dự đoán token tiếp theo, trên hàng nghìn tỉ token. Chỉ số **perplexity** mà bạn hay gặp trong bài báo bằng `exp(loss trung bình)`: càng thấp, model càng "ít bất ngờ" trước văn bản thật.

## 5. Đạo hàm và gradient descent

### 5.1 Trực giác

Hãy tưởng tượng bạn đứng trên sườn đồi trong sương mù và muốn xuống thung lũng (loss thấp nhất). Bạn không thấy đường, nhưng cảm nhận được **độ dốc** dưới chân. Chiến lược: bước một bước nhỏ theo hướng dốc xuống, lặp lại.

- **Đạo hàm (gradient)** cho biết độ dốc: nếu tăng tham số một chút, loss tăng hay giảm, nhanh cỡ nào.
- **Gradient descent:** `tham_số_mới = tham_số_cũ - learning_rate × gradient`.
- **Learning rate** là độ dài bước: quá nhỏ thì học rất chậm, quá lớn thì nhảy qua lại không xuống được đáy.

### 5.2 Tự cài đặt

Bài toán: dự đoán giá một chiếc áo từ số lượng đã bán (giả lập), với model đường thẳng `y = w·x + b`.

```python title="gradient_descent.py"
import numpy as np

rng = np.random.default_rng(0)
x = rng.uniform(0, 10, size=100)
y = 3.0 * x + 5.0 + rng.normal(0, 1, size=100)  # quy luật thật: w=3, b=5, có nhiễu

w, b = 0.0, 0.0
learning_rate = 0.01

for step in range(2000):
    y_pred = w * x + b
    error = y_pred - y
    loss = (error ** 2).mean()          # mean squared error

    # Đạo hàm của loss theo w và b (tính tay bằng quy tắc chuỗi)
    grad_w = 2 * (error * x).mean()
    grad_b = 2 * error.mean()

    w -= learning_rate * grad_w
    b -= learning_rate * grad_b

    if step % 400 == 0:
        print(f"bước {step:4d}: loss={loss:.3f}  w={w:.3f}  b={b:.3f}")

print(f"Kết quả: w={w:.2f}, b={b:.2f} (quy luật thật: 3, 5)")
```

Model "học" được quy luật từ dữ liệu, chỉ bằng cách lặp lại: dự đoán, đo sai, tính độ dốc, điều chỉnh. Huấn luyện một LLM về bản chất là **cùng thuật toán này**, với hàng tỉ tham số thay vì hai, và đạo hàm được **tính tự động** (autograd, Bài 3) thay vì tính tay.

## 6. Bảng tra nhanh: toán nào ở đâu

| Khái niệm | Xuất hiện trong |
|---|---|
| Vector, tích vô hướng, cosine | Embedding, tìm kiếm ngữ nghĩa, attention |
| Nhân ma trận | Mọi lớp của mạng neural, chiếu Query/Key/Value |
| Softmax | Chọn token tiếp theo, trọng số attention, phân loại |
| Cross-entropy | Hàm mất mát khi huấn luyện LLM và model phân loại |
| Gradient descent | Mọi quá trình huấn luyện và fine-tuning |
| Phân phối xác suất, lấy mẫu | Temperature, top-p, sinh văn bản |
| Ma trận hạng thấp | LoRA (Bài 5) |

## Bài tập

**Bài 1.1.** Viết hàm tính cosine similarity cho **ma trận**: đầu vào gồm một vector câu hỏi và ma trận 1.000 vector tài liệu, trả về chỉ số top 5 tài liệu gần nhất. Không dùng vòng lặp Python.

??? tip "Gợi ý lời giải"
    ```python
    def top_k(query: np.ndarray, docs: np.ndarray, k: int = 5) -> np.ndarray:
        q = query / np.linalg.norm(query)
        d = docs / np.linalg.norm(docs, axis=1, keepdims=True)
        scores = d @ q
        return np.argsort(scores)[::-1][:k]
    ```

**Bài 1.2.** Trong `gradient_descent.py`, thử learning rate 0,0001, 0,01, 0,05. Chuyện gì xảy ra với loss? Giải thích bằng hình ảnh "xuống dốc".

**Bài 1.3.** Mở rộng `gradient_descent.py` thành **phân loại nhị phân** (logistic regression): dự đoán khách có mua lại không từ số lần mua trước đó, dùng hàm sigmoid và binary cross-entropy.

**Bài 1.4.** Tính perplexity của phân phối `softmax([2.0, 1.0, 0.1])` khi đáp án đúng lần lượt là mỗi lớp. Perplexity có ý nghĩa trực quan là gì?

## Checklist

- [ ] Giải thích được tích vô hướng và cosine similarity bằng hình học.
- [ ] Hiểu một lớp mạng neural là phép nhân ma trận cộng hàm kích hoạt.
- [ ] Tự viết được softmax, cross-entropy.
- [ ] Tự cài đặt gradient descent, giải thích được vai trò của learning rate.

## Đọc thêm

- 3Blue1Brown: loạt video "Essence of linear algebra" và "Neural networks".
- [Mathematics for Machine Learning](https://mml-book.github.io/) (sách miễn phí).

---

[Tổng quan track](index.md) · **Bài tiếp theo:** [Bài 2: ML cổ điển](02-ml-co-dien.md)
