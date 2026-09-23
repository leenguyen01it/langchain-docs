# Track: ML nền tảng & Fine-tuning

Phần lớn công việc của AI Engineer là dùng model có sẵn qua API. Nhưng hiểu nền tảng machine learning giúp bạn: đọc hiểu tài liệu và bài báo, biết vì sao embedding hoạt động, nhận ra khi nào một model phân loại nhỏ **tốt hơn và rẻ hơn** LLM, fine-tune model khi thực sự cần, và tự chạy model mở khi dữ liệu không được phép rời khỏi hạ tầng.

Track này **học song song** với các track khác, mỗi tuần vài giờ. Không cần hoàn thành trước khi làm RAG hay Agents.

| Bài | Nội dung | Thời lượng |
|---|---|---|
| [Bài 1: Toán cần thiết](01-toan-can-thiet.md) | Vector và ma trận, xác suất và softmax, cross-entropy, gradient descent tự viết bằng NumPy | 1 tuần |
| [Bài 2: ML cổ điển](02-ml-co-dien.md) | Học có giám sát, chia tập, overfitting, metric phân loại, scikit-learn, khi nào ML cổ điển thắng LLM | 1 tuần |
| [Bài 3: Deep learning với PyTorch](03-pytorch.md) | Tensor, autograd, `nn.Module`, vòng lặp huấn luyện, GPU, chống overfitting | 1 đến 2 tuần |
| [Bài 4: Transformer và Hugging Face](04-transformers-hf.md) | Tự cài đặt self-attention, hệ sinh thái Hugging Face, model tiếng Việt | 1 tuần |
| [Bài 5: Fine-tuning với LoRA](05-fine-tuning.md) | Khi nào nên fine-tune, chuẩn bị dữ liệu, LoRA và QLoRA, huấn luyện với TRL, đánh giá | 1 đến 2 tuần |
| [Bài 6: Chạy model local](06-chay-model-local.md) | Quantization, ước tính bộ nhớ, Ollama, vLLM, phục vụ adapter LoRA, so sánh chi phí | 1 tuần |

**Yêu cầu trước:** Python vững, toán phổ thông. Bài 3 trở đi nên có GPU (máy cá nhân, hoặc dịch vụ notebook miễn phí như Google Colab, Kaggle).

!!! tip "Học tới đâu là đủ?"
    Mục tiêu của AI Engineer **không phải** là nghiên cứu model mới. Đủ khi bạn: giải thích được embedding, attention, overfitting bằng lời của mình; đọc được code huấn luyện; fine-tune được một model nhỏ và đánh giá nó đúng cách; ra được quyết định "build hay buy" có căn cứ.

---

[Lộ trình AI Engineer](../index.md)
