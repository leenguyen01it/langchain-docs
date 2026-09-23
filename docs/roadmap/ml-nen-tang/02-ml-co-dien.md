# Bài 2: ML cổ điển

!!! abstract "Tóm tắt bài học"
    - **Mục tiêu:** nắm quy trình học có giám sát, chia tập dữ liệu đúng cách, nhận biết overfitting, đọc được các metric phân loại; huấn luyện model phân loại văn bản bằng scikit-learn và so sánh với LLM về chất lượng, chi phí, độ trễ.
    - **Thời lượng:** khoảng 1 tuần.
    - **Yêu cầu trước:** [Bài 1](01-toan-can-thiet.md).

## 1. Học có giám sát

**Học có giám sát** (supervised learning): cho model xem nhiều cặp (đầu vào, đáp án), model học quy luật để dự đoán đáp án cho đầu vào mới.

| Loại bài toán | Đầu ra | Ví dụ |
|---|---|---|
| Phân loại | Một nhãn trong tập cố định | Cảm xúc của phản hồi, loại yêu cầu hỗ trợ, email rác |
| Hồi quy | Một con số | Dự đoán doanh thu, thời gian giao hàng |

Ngoài ra có **học không giám sát** (không có đáp án: phân cụm khách hàng, giảm chiều dữ liệu) và **học tăng cường** (học qua thử và thưởng phạt, được dùng trong giai đoạn huấn luyện sau của LLM).

## 2. Chia tập dữ liệu

| Tập | Tỉ lệ gợi ý | Dùng để |
|---|---|---|
| **Train** | 70% | Huấn luyện model |
| **Validation** | 15% | Chọn model, chỉnh tham số |
| **Test** | 15% | Đánh giá **một lần cuối cùng**, báo cáo kết quả |

!!! danger "Rò rỉ dữ liệu (data leakage)"
    Nếu dữ liệu test "lọt" vào quá trình huấn luyện hoặc lựa chọn model, điểm số sẽ đẹp giả tạo và sụp đổ khi ra thực tế. Các dạng hay gặp: trùng lặp giữa train và test; dùng tập test để chọn tham số; với dữ liệu theo thời gian, dùng dữ liệu tương lai để dự đoán quá khứ. Nguyên tắc này áp dụng y hệt cho eval LLM ([Evaluation, Bài 1](../evaluation/01-tu-duy-eval.md#42-kich-thuoc-va-chia-tap)).

## 3. Overfitting và underfitting

| | Train | Validation | Nghĩa là |
|---|---|---|---|
| **Underfitting** | Kém | Kém | Model quá đơn giản, chưa học được quy luật |
| **Vừa khớp** | Tốt | Tốt | Mục tiêu |
| **Overfitting** | Rất tốt | Kém | Model "học thuộc" dữ liệu train, kể cả nhiễu, không tổng quát hóa |

Chống overfitting: thêm dữ liệu, model đơn giản hơn, **regularization** (phạt trọng số lớn), dừng sớm khi điểm validation ngừng tăng.

## 4. Metric phân loại

Với phản hồi khách hàng, giả sử lớp quan tâm là **"khiếu nại"** (cần xử lý gấp):

| | Model nói khiếu nại | Model nói không |
|---|---|---|
| **Thật sự là khiếu nại** | True Positive (TP) | False Negative (FN): **bỏ sót** |
| **Thật sự không phải** | False Positive (FP): **báo nhầm** | True Negative (TN) |

| Metric | Công thức | Trả lời câu hỏi |
|---|---|---|
| Accuracy | (TP + TN) / tổng | Đúng bao nhiêu phần trăm tổng thể |
| **Precision** | TP / (TP + FP) | Trong những gì model báo là khiếu nại, bao nhiêu là thật? |
| **Recall** | TP / (TP + FN) | Trong các khiếu nại thật, model bắt được bao nhiêu? |
| F1 | Trung bình điều hòa của precision và recall | Cân bằng cả hai |

!!! warning "Accuracy đánh lừa khi dữ liệu mất cân bằng"
    Nếu chỉ 3% phản hồi là khiếu nại, một model **luôn nói "không phải khiếu nại"** đạt accuracy 97% nhưng bỏ sót 100% khiếu nại. Luôn xem precision, recall của lớp quan trọng, và confusion matrix.

**Precision hay recall quan trọng hơn** tùy chi phí của từng loại sai. Bỏ sót khiếu nại (FN) khiến khách bỏ đi; báo nhầm (FP) chỉ tốn vài giây nhân viên đọc lại. Vậy nên ưu tiên recall cao.

## 5. Thực hành: phân loại phản hồi với scikit-learn

```bash
uv add scikit-learn pandas
```

```python title="classic_classifier.py"
import pandas as pd
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.linear_model import LogisticRegression
from sklearn.metrics import classification_report, confusion_matrix
from sklearn.model_selection import train_test_split
from sklearn.pipeline import make_pipeline

# feedback.csv: cột "text" và "label" (khen, hoi_dap, khieu_nai), vài trăm đến vài nghìn dòng
df = pd.read_csv("feedback.csv")
X_train, X_test, y_train, y_test = train_test_split(
    df["text"], df["label"], test_size=0.2, stratify=df["label"], random_state=42
)

model = make_pipeline(
    # TF-IDF với n-gram ký tự: chịu được viết tắt, sai chính tả, không dấu
    TfidfVectorizer(analyzer="char_wb", ngram_range=(2, 5), min_df=2, sublinear_tf=True),
    LogisticRegression(max_iter=1000, class_weight="balanced"),  # bù cho lớp hiếm
)
model.fit(X_train, y_train)

pred = model.predict(X_test)
print(classification_report(y_test, pred, digits=3))
print(confusion_matrix(y_test, pred, labels=["khen", "hoi_dap", "khieu_nai"]))
```

- **TF-IDF** biến văn bản thành vector: mỗi chiều là một cụm ký tự, giá trị cao nếu cụm đó xuất hiện nhiều trong văn bản này nhưng hiếm trong toàn bộ dữ liệu.
- **Logistic regression** là model tuyến tính học trọng số cho từng chiều; đơn giản, nhanh, dễ giải thích.
- `stratify` giữ tỉ lệ các lớp giống nhau giữa train và test; `class_weight="balanced"` giúp model không bỏ qua lớp hiếm.

### Cải tiến: embedding + model đơn giản

Thay TF-IDF bằng **embedding** từ một model embedding (xem [RAG, Bài 3](../rag/03-embeddings-vector-db.md)), rồi vẫn dùng logistic regression. Embedding nắm bắt ngữ nghĩa ("giao chậm quá" và "đợi mãi không thấy hàng" gần nhau), thường tăng chất lượng rõ rệt mà vẫn rẻ và nhanh.

```python
from langchain.embeddings import init_embeddings

embedder = init_embeddings("openai:text-embedding-3-small")
E_train = embedder.embed_documents(list(X_train))
E_test = embedder.embed_documents(list(X_test))

clf = LogisticRegression(max_iter=2000, class_weight="balanced").fit(E_train, y_train)
print(classification_report(y_test, clf.predict(E_test), digits=3))
```

## 6. ML cổ điển hay LLM?

So sánh trên bài toán phân loại phản hồi (con số minh họa; hãy tự đo trên dữ liệu của bạn):

| | TF-IDF + LogReg | Embedding + LogReg | LLM (zero-shot hoặc few-shot) |
|---|---|---|---|
| Cần dữ liệu có nhãn | Vài trăm đến vài nghìn | Vài trăm | Không cần, hoặc vài ví dụ |
| Chất lượng | Khá | Tốt | Tốt đến rất tốt, hiểu ngữ cảnh, mỉa mai |
| Chi phí mỗi 1 triệu phản hồi | Gần như bằng 0 | Chi phí embedding (thấp) | Cao hơn nhiều |
| Độ trễ | Dưới 1 ms | Chủ yếu là thời gian embed | Vài trăm ms đến vài giây |
| Đổi định nghĩa nhãn | Phải gán nhãn lại, huấn luyện lại | Như bên trái | Sửa prompt |
| Giải thích được | Có (trọng số của từng từ) | Hạn chế | Có thể yêu cầu model giải thích |

**Chiến lược thực tế:**

1. **Bắt đầu với LLM** để có sản phẩm nhanh, không cần dữ liệu.
2. Dùng chính LLM (kèm người kiểm tra) để **gán nhãn** hàng nghìn ví dụ.
3. Khi khối lượng lớn và nhãn ổn định, **huấn luyện model nhỏ** trên dữ liệu đó để giảm chi phí và độ trễ.
4. Kết hợp: model nhỏ xử lý các trường hợp chắc chắn (xác suất cao), chuyển trường hợp khó cho LLM.

## Bài tập

**Bài 2.1.** Thu thập hoặc sinh (bằng LLM, có người kiểm tra) 1.000 phản hồi tiếng Việt với 3 nhãn, trong đó "khiếu nại" chỉ chiếm 10%. Chạy `classic_classifier.py`. So sánh accuracy với recall của lớp khiếu nại.

**Bài 2.2.** Thử TF-IDF theo từ (`analyzer="word"`) và theo ký tự (`analyzer="char_wb"`) trên dữ liệu có nhiều viết tắt, không dấu. Cái nào tốt hơn? Vì sao?

**Bài 2.3.** So sánh ba phương án ở mục 6 trên cùng tập test: F1 của lớp khiếu nại, chi phí ước tính cho 1 triệu phản hồi, độ trễ trung bình.

**Bài 2.4.** Cài đặt chiến lược kết hợp: dùng `predict_proba` của logistic regression; nếu xác suất cao nhất dưới 0,8 thì gọi LLM. Bao nhiêu phần trăm phản hồi phải chuyển cho LLM? Chất lượng tổng thể thay đổi thế nào?

## Checklist

- [ ] Chia train, validation, test đúng cách và giải thích được rò rỉ dữ liệu.
- [ ] Nhận biết overfitting qua điểm train và validation.
- [ ] Đọc được confusion matrix, chọn precision hay recall theo bài toán.
- [ ] Huấn luyện được model phân loại văn bản bằng scikit-learn.
- [ ] Biết khi nào model nhỏ tốt hơn LLM, và cách kết hợp hai loại.

## Đọc thêm

- [scikit-learn User Guide](https://scikit-learn.org/stable/user_guide.html)
- [Hiểu LLM, Bài 5: Chọn model](../hieu-llm/05-chon-model.md)

---

**Bài trước:** [Bài 1: Toán cần thiết](01-toan-can-thiet.md) · [Tổng quan track](index.md) · **Bài tiếp theo:** [Bài 3: Deep learning với PyTorch](03-pytorch.md)
