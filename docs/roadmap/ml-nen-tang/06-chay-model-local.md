# Bài 6: Chạy model local

!!! abstract "Tóm tắt bài học"
    - **Mục tiêu:** ước tính được phần cứng cần thiết cho một model, hiểu quantization, chạy model mở bằng Ollama (thử nghiệm) và vLLM (phục vụ production), phục vụ adapter LoRA, và so sánh chi phí tự host với API một cách trung thực.
    - **Thời lượng:** khoảng 1 tuần.
    - **Yêu cầu trước:** [Bài 5](05-fine-tuning.md), [Hiểu LLM, Bài 5](../hieu-llm/05-chon-model.md#7-model-mo-va-chay-local).

## 1. Model cần bao nhiêu bộ nhớ?

Bộ nhớ GPU cho suy luận gồm hai phần chính:

```text
Bộ nhớ ≈ (số tham số × số byte mỗi tham số)  +  KV cache  +  phần phụ
```

| Độ chính xác | Byte mỗi tham số | Model 7B | Model 70B |
|---|---|---|---|
| FP16 / BF16 | 2 | khoảng 14 GB | khoảng 140 GB |
| INT8 | 1 | khoảng 7 GB | khoảng 70 GB |
| INT4 | 0,5 | khoảng 3,5 GB | khoảng 35 GB |

**KV cache** lưu kết quả tính toán của các token đã xử lý (xem [Hiểu LLM, Bài 1](../hieu-llm/01-transformer-token.md#he-qua-thuc-te-vi-sao-input-re-hon-output)), tăng theo **độ dài context × số request đồng thời**. Với context dài và nhiều người dùng cùng lúc, KV cache có thể chiếm nhiều bộ nhớ hơn cả trọng số model.

## 2. Quantization

**Quantization** (lượng tử hóa) lưu trọng số với ít bit hơn: 16 bit xuống 8 bit hoặc 4 bit. Model nhỏ hơn, chạy được trên phần cứng yếu hơn, thường nhanh hơn, đổi lại chất lượng giảm một chút (với 8 bit gần như không đáng kể; với 4 bit giảm nhẹ, tùy model và tác vụ).

| Định dạng | Dùng với | Đặc điểm |
|---|---|---|
| **GGUF** | llama.cpp, Ollama | Chạy tốt trên CPU, Mac (Apple Silicon), GPU tiêu dùng; nhiều mức (Q4, Q5, Q8...) |
| **AWQ, GPTQ** | vLLM và các server GPU | Tối ưu cho suy luận trên GPU |
| **FP8** | GPU thế hệ mới | Gần như không mất chất lượng, nhanh |

Luôn **đánh giá lại** chất lượng trên tác vụ của bạn sau khi lượng tử hóa, đặc biệt với tiếng Việt: model nhỏ lượng tử hóa mạnh có thể giảm chất lượng tiếng Việt rõ rệt hơn tiếng Anh.

## 3. Ollama: thử nghiệm nhanh

[Ollama](https://ollama.com/) là cách dễ nhất để chạy model mở trên máy cá nhân: tự tải model đã lượng tử hóa (GGUF), tự dùng GPU nếu có, cung cấp API HTTP.

```bash
ollama pull qwen2.5:7b
ollama run qwen2.5:7b "Viết câu chào khách hàng bằng tiếng Việt, thân thiện"
ollama ps     # xem model đang chạy và bộ nhớ sử dụng
```

Dùng từ Python qua LangChain:

```python
from langchain.chat_models import init_chat_model

local = init_chat_model("ollama:qwen2.5:7b")
print(local.invoke("Tóm tắt trong một câu: giao hàng trễ 3 ngày nhưng sản phẩm đẹp").text)
```

Nhờ interface chung của LangChain, bạn có thể thay model API bằng model local trong các ứng dụng RAG, agent đã viết mà gần như không sửa code (lưu ý model nhỏ có thể kém hơn nhiều khi dùng tool, suy luận nhiều bước).

## 4. vLLM: phục vụ production

Ollama phù hợp thử nghiệm và một người dùng. Để phục vụ **nhiều người dùng đồng thời** trên GPU, dùng server tối ưu thông lượng như [vLLM](https://docs.vllm.ai/):

- **Continuous batching:** gộp request của nhiều người dùng vào cùng lượt tính toán, tăng thông lượng nhiều lần so với xử lý lần lượt.
- **PagedAttention:** quản lý KV cache theo "trang" như bộ nhớ ảo, giảm lãng phí, phục vụ được nhiều request đồng thời hơn.
- **API tương thích chuẩn OpenAI:** nhiều thư viện và công cụ kết nối được ngay.

```bash
pip install vllm   # cần GPU NVIDIA, chạy trên Linux
vllm serve Qwen/Qwen2.5-7B-Instruct --max-model-len 8192
```

```python
from langchain_openai import ChatOpenAI

# vLLM cung cấp API theo chuẩn OpenAI tại cổng 8000
llm = ChatOpenAI(base_url="http://localhost:8000/v1", api_key="EMPTY", model="Qwen/Qwen2.5-7B-Instruct")
print(llm.invoke("Xin chào").text)
```

### Phục vụ adapter LoRA

Adapter đã fine-tune ở [Bài 5](05-fine-tuning.md) có thể được phục vụ cùng model gốc; một server phục vụ được nhiều adapter cho nhiều tác vụ:

```bash
vllm serve Qwen/Qwen2.5-0.5B-Instruct --enable-lora \
  --lora-modules mo-ta-san-pham=./out-mo-ta-san-pham
```

Khi gọi API, đặt `model="mo-ta-san-pham"` để dùng adapter.

### Đo hiệu năng

| Chỉ số | Ý nghĩa |
|---|---|
| TTFT | Thời gian tới token đầu tiên của một request |
| Token/giây mỗi request | Tốc độ người dùng cảm nhận |
| **Tổng token/giây** | Thông lượng của cả server, quyết định chi phí trên mỗi token |
| Số request đồng thời tối đa | Trước khi độ trễ tăng quá ngưỡng chấp nhận |

Đo bằng cách gửi tải tăng dần (10, 50, 100 request đồng thời) và ghi lại các chỉ số trên.

## 5. Tự host hay dùng API: tính toán trung thực

| Chi phí tự host | Thường bị bỏ quên |
|---|---|
| GPU (thuê theo giờ hoặc mua) | GPU chạy **24/7** kể cả lúc không có traffic, trừ khi có hệ thống tự co giãn |
| Kỹ sư vận hành | Cập nhật model, xử lý sự cố, giám sát, bảo mật |
| Hạ tầng phụ | Load balancer, lưu trữ model, dự phòng khi GPU lỗi |
| Chất lượng | Model mở vừa sức một GPU thường kém model API hàng đầu, dẫn tới tỉ lệ lỗi cao hơn, cần xử lý bằng con người nhiều hơn |

Cách so sánh:

1. Ước tính **tổng token mỗi tháng** (input và output) của tác vụ.
2. **API:** token × giá (đã tính prompt caching, batch nếu áp dụng được).
3. **Tự host:** (giá GPU mỗi giờ × số giờ chạy) + chi phí vận hành; kiểm tra thông lượng đo được ở mục 4 có đáp ứng được khối lượng và độ trễ yêu cầu không.
4. So sánh **chi phí trên mỗi tác vụ thành công**, không chỉ trên mỗi token, vì chất lượng khác nhau.

**Tự host thường hợp lý khi:** dữ liệu bắt buộc không rời hạ tầng; khối lượng rất lớn và **ổn định** trên tác vụ hẹp mà model nhỏ (có thể đã fine-tune) làm tốt; cần chạy offline hoặc trên thiết bị. Với phần lớn sản phẩm giai đoạn đầu, **API là lựa chọn kinh tế và nhanh hơn**.

## Bài tập

**Bài 6.1.** Ước tính bộ nhớ GPU cần để phục vụ một model 8B ở BF16 và INT4. Kiểm chứng bằng `ollama ps` (với bản lượng tử hóa) nếu máy bạn chạy được.

**Bài 6.2.** Chạy cùng một model ở hai mức lượng tử hóa khác nhau (ví dụ Q4 và Q8 trên Ollama). So sánh chất lượng trên 20 câu hỏi tiếng Việt và tốc độ sinh.

**Bài 6.3.** (Cần GPU NVIDIA hoặc máy thuê.) Chạy vLLM, đo tổng token/giây với 1, 10, 50 request đồng thời. Vẽ biểu đồ thông lượng và độ trễ theo số request đồng thời.

**Bài 6.4.** Phục vụ adapter LoRA từ Bài 5 bằng vLLM, gọi thử qua LangChain.

**Bài 6.5.** Viết bảng so sánh chi phí tự host và API cho tác vụ phân loại 2 triệu phản hồi mỗi tháng và tác vụ chatbot 50.000 hội thoại mỗi tháng. Kết luận cho từng tác vụ.

## Checklist

- [ ] Ước tính được bộ nhớ cần cho trọng số và hiểu vai trò của KV cache.
- [ ] Hiểu quantization và đánh giá lại chất lượng sau khi lượng tử hóa.
- [ ] Chạy được model bằng Ollama và dùng từ LangChain.
- [ ] Hiểu vì sao vLLM phục vụ nhiều người dùng hiệu quả; phục vụ được adapter LoRA.
- [ ] So sánh chi phí tự host và API một cách trung thực, trên mỗi tác vụ thành công.

## Đọc thêm

- [Ollama](https://ollama.com/), [vLLM](https://docs.vllm.ai/), [llama.cpp](https://github.com/ggml-org/llama.cpp)
- [Models](../../oss/python/langchain/models.md): kết nối LangChain với nhiều nhà cung cấp, kể cả model local.

---

**Bài trước:** [Bài 5: Fine-tuning với LoRA](05-fine-tuning.md) · [Tổng quan track](index.md)

!!! success "Hoàn thành lộ trình AI Engineer"
    Đi hết các track, bạn đã có đủ nền tảng để xây, đánh giá, vận hành và tối ưu sản phẩm AI thật. Hãy quay lại [trang tổng quan](../index.md#du-an-portfolio) để chọn dự án portfolio, và nhớ rằng kỹ năng quan trọng nhất vẫn là: **nhìn vào dữ liệu, đo trước khi tối ưu, bắt đầu đơn giản**.
