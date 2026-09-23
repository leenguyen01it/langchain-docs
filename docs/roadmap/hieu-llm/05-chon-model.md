# Bài 5: Chọn model

!!! abstract "Tóm tắt bài học"
    - **Mục tiêu:** chọn model bằng số liệu trên chính tác vụ của bạn thay vì theo bảng xếp hạng hay cảm tính; tính đúng chi phí trên mỗi tác vụ; biết khi nào nên dùng model mở chạy local.
    - **Thời lượng:** 3 đến 4 ngày.
    - **Yêu cầu trước:** [Bài 3](03-claude-api.md).

## 1. Bản đồ các model

| Nhóm | Ví dụ | Đặc điểm |
|---|---|---|
| **Model đóng qua API** | Claude (Anthropic), GPT (OpenAI), Gemini (Google) | Chất lượng cao nhất, không cần hạ tầng, trả tiền theo token, dữ liệu đi qua nhà cung cấp |
| **Model mở (open-weight)** | Llama, Qwen, Gemma, DeepSeek, Mistral | Tải trọng số về, tự chạy hoặc thuê nhà cung cấp chạy hộ; kiểm soát dữ liệu hoàn toàn, fine-tune được |
| **Model chuyên biệt** | Embedding, reranker, nhận dạng giọng nói, sinh ảnh | Làm một việc cụ thể, thường rẻ và nhanh hơn nhiều so với LLM đa năng |

Trong cùng một nhà cung cấp, model thường chia **tầng** theo năng lực và giá. Với Claude:

| Tầng | Model | Dùng cho |
|---|---|---|
| Mạnh nhất | Claude Fable 5.1 | Suy luận khó nhất, agent tự chủ chạy rất dài |
| Chủ lực | Claude Opus 5 | Mặc định cho phần lớn tác vụ đòi hỏi chất lượng: lập trình, agent, phân tích |
| Cân bằng | Claude Sonnet 5 | Khối lượng lớn, cần cân bằng chất lượng và chi phí |
| Nhanh, rẻ | Claude Haiku 4.5 | Phân loại, trích xuất đơn giản, định tuyến, tác vụ số lượng rất lớn |

## 2. Tiêu chí chọn model

| Tiêu chí | Câu hỏi cần trả lời |
|---|---|
| **Chất lượng trên tác vụ của bạn** | Trên 50 đến 100 ví dụ thật, model đúng bao nhiêu phần trăm? |
| **Chi phí trên mỗi tác vụ hoàn thành** | Bao gồm cả token suy nghĩ, số lượt gọi tool, số lần phải gọi lại |
| **Độ trễ** | Thời gian tới token đầu tiên (TTFT) và tốc độ sinh (token/giây) có đáp ứng trải nghiệm người dùng? |
| **Tính năng** | Tool use, structured output, vision, PDF, context dài, prompt caching, batch? |
| **Tiếng Việt** | Văn phong tự nhiên không? Hiểu từ địa phương, viết tắt, tiếng lóng? |
| **Dữ liệu và tuân thủ** | Dữ liệu được lưu ở đâu, bao lâu? Có đáp ứng yêu cầu của khách hàng, của luật? |
| **Vận hành** | Rate limit của tài khoản, độ ổn định, có nhà cung cấp dự phòng không? |

## 3. Benchmark: hữu ích nhưng có giới hạn

Các bảng xếp hạng công khai (bộ đề kiến thức, lập trình, toán, hoặc bình chọn của người dùng) giúp **lọc ứng viên**, nhưng không nên là căn cứ quyết định:

- **Không giống tác vụ của bạn.** Điểm cao ở bài toán olympic không nói gì về khả năng trả lời khách hàng bằng tiếng Việt theo đúng chính sách của cửa hàng.
- **Bão hòa và nhiễm dữ liệu.** Nhiều bộ đề đã bị các model gần như giải hết, hoặc câu hỏi đã lọt vào dữ liệu huấn luyện.
- **Phụ thuộc cách đo.** Cùng một model, prompt khác nhau cho điểm khác nhau đáng kể.

Quy tắc: **benchmark để chọn 3 đến 4 ứng viên, eval của bạn để chọn một**.

## 4. Tự so sánh model trên tác vụ của bạn

LangChain `init_chat_model` cho phép đổi model, thậm chí đổi nhà cung cấp, chỉ bằng một chuỗi. Rất tiện để so sánh:

```bash
uv add langchain langchain-anthropic langchain-ollama
```

```python title="compare_models.py"
import json
import statistics
import time
from typing import Literal

from langchain.chat_models import init_chat_model
from pydantic import BaseModel

# Dữ liệu có nhãn: 50 đến 100 phản hồi thật của khách hàng, gán nhãn bằng tay
dataset = [json.loads(line) for line in open("labeled_feedback.jsonl", encoding="utf-8")]
# Mỗi dòng: {"text": "...", "label": "negative"}


class Label(BaseModel):
    sentiment: Literal["positive", "neutral", "negative"]


PRICES = {  # USD mỗi 1 triệu token (input, output)
    "anthropic:claude-opus-5": (5.0, 25.0),
    "anthropic:claude-sonnet-5": (2.0, 10.0),
    "anthropic:claude-haiku-4-5": (1.0, 5.0),
    "ollama:qwen2.5:7b": (0.0, 0.0),  # chạy local: không tính phí token, nhưng tốn phần cứng
}

for model_name, (p_in, p_out) in PRICES.items():
    model = init_chat_model(model_name).with_structured_output(Label, include_raw=True)
    correct, latencies, cost = 0, [], 0.0
    for item in dataset:
        start = time.perf_counter()
        out = model.invoke(f"Phân loại cảm xúc của phản hồi khách hàng sau:\n{item['text']}")
        latencies.append(time.perf_counter() - start)
        if out["parsed"] and out["parsed"].sentiment == item["label"]:
            correct += 1
        usage = out["raw"].usage_metadata or {}
        cost += (usage.get("input_tokens", 0) * p_in + usage.get("output_tokens", 0) * p_out) / 1e6
    print(f"{model_name:30} accuracy={correct / len(dataset):.0%} "
          f"p50={statistics.median(latencies):.2f}s cost/1000 tác vụ=${cost / len(dataset) * 1000:.2f}")
```

Kết quả điển hình của loại thí nghiệm này: với tác vụ đơn giản như phân loại cảm xúc, model nhỏ đạt độ chính xác gần bằng model lớn với chi phí thấp hơn nhiều; với tác vụ phức tạp (suy luận nhiều bước, agent), khoảng cách chất lượng lớn và model mạnh hơn thường **rẻ hơn tính trên mỗi tác vụ hoàn thành**, vì làm đúng ngay từ đầu, ít phải gọi lại.

!!! tip "Đọc các câu sai, không chỉ nhìn con số"
    Hai model cùng đạt 90% có thể sai ở những chỗ rất khác nhau. Model A sai ở các câu mỉa mai, model B sai ở phản hồi có tiếng lóng. Loại lỗi nào nguy hiểm hơn với sản phẩm của bạn?

## 5. Chi phí trên mỗi tác vụ, không phải trên mỗi token

Giá mỗi token chỉ là một phần của bức tranh:

| Yếu tố | Ảnh hưởng |
|---|---|
| Token suy nghĩ | Tính như output. Effort cao thì nhiều token suy nghĩ hơn |
| Số lượt gọi | Agent có thể gọi model 5 đến 20 lần cho một tác vụ |
| Tỉ lệ thành công | Model rẻ nhưng sai 30% phải gọi lại hoặc chuyển cho người xử lý |
| Prompt caching | Có thể giảm 50% đến 90% chi phí input cho prompt lặp lại |
| Batch | Giảm 50% cho tác vụ không cần kết quả ngay |

Ví dụ: một tác vụ agent, model A (rẻ) cần trung bình 12 lượt và thành công 70%; model B (đắt gấp đôi mỗi token) cần 5 lượt và thành công 95%. Tính ra chi phí **trên mỗi tác vụ thành công**, model B có thể rẻ hơn.

!!! tip "Thử 'model mạnh, effort thấp' trước 'model yếu'"
    Trước khi hạ xuống model nhỏ hơn để tiết kiệm, hãy thử giữ model mạnh và hạ `effort` (xem [Bài 2](02-sampling-reasoning.md#23-effort-ieu-chinh-o-ky-luong)). Cách này thường giữ được chất lượng tốt hơn với mức tiết kiệm tương đương, và chỉ cần quản lý một model.

## 6. Định tuyến giữa nhiều model

Hệ thống thật thường dùng **nhiều model cho nhiều bước**:

- Model nhỏ: phân loại ý định, viết lại câu hỏi, trích xuất đơn giản, chấm điểm tài liệu trong RAG.
- Model chủ lực: câu trả lời cuối, agent, tác vụ cần suy luận.
- Model mạnh nhất: chỉ cho những yêu cầu khó mà eval chứng minh cần tới.

Lưu ý: mỗi model có cache riêng, và thêm model là thêm thứ cần eval, cần giám sát. Chỉ tách model khi số liệu cho thấy lợi ích rõ ràng.

## 7. Model mở và chạy local

### 7.1 Thử nhanh với Ollama

[Ollama](https://ollama.com/) chạy model mở trên máy cá nhân chỉ với vài lệnh:

```bash
ollama pull qwen2.5:7b
ollama run qwen2.5:7b "Viết một câu chào khách hàng bằng tiếng Việt"
```

Dùng từ Python qua LangChain: `init_chat_model("ollama:qwen2.5:7b")` như trong đoạn so sánh ở trên.

### 7.2 Khi nào nên tự chạy model?

| Nên cân nhắc tự chạy | Nên dùng API |
|---|---|
| Dữ liệu **không được phép** rời khỏi hạ tầng của bạn | Cần chất lượng cao nhất |
| Cần chạy offline, trên thiết bị | Khối lượng vừa phải hoặc biến động |
| Khối lượng rất lớn, ổn định, tác vụ đơn giản | Đội ngũ nhỏ, không có người vận hành GPU |
| Cần fine-tune sâu cho tác vụ hẹp | Cần tool use, agent phức tạp, context dài |

Chi phí ẩn của việc tự chạy: GPU (mua hoặc thuê), kỹ sư vận hành, tối ưu hiệu năng, cập nhật model, và thường là **chất lượng thấp hơn** model API hàng đầu. Xem chi tiết ở [track ML nền tảng, Bài 6](../ml-nen-tang/06-chay-model-local.md).

## 8. Model thay đổi liên tục

- Model mới ra mắt vài tháng một lần; model cũ bị **ngừng hỗ trợ** theo lịch công bố trước.
- Model mới có thể có **API khác** (ví dụ bỏ tham số `temperature`, đổi cách cấu hình thinking) và **hành vi khác** (prompt viết cho model cũ có thể không còn tối ưu).
- Vì vậy: ghi rõ model ID trong cấu hình (không rải rác trong code), có **bộ eval** để chạy lại khi nâng cấp, và đọc hướng dẫn chuyển đổi của nhà cung cấp.

## Bài tập

**Bài 5.1.** Gán nhãn tay 50 phản hồi khách hàng tiếng Việt thật (có thể lấy từ đánh giá sản phẩm công khai). Chạy `compare_models.py` với ít nhất 3 model. Lập bảng: độ chính xác, độ trễ p50, chi phí cho 1.000 tác vụ.

**Bài 5.2.** Với model tốt nhất ở bài 5.1, thử lại với `effort` thấp (với Claude: truyền tham số effort qua cấu hình model). Chất lượng và chi phí thay đổi thế nào?

**Bài 5.3.** Cài Ollama, chạy một model mở cỡ 7B đến 8B. Thử 10 câu hỏi tiếng Việt về chăm sóc khách hàng và so sánh chất lượng văn phong với Claude.

**Bài 5.4.** Viết một "báo cáo chọn model" ngắn (một trang) cho một tính năng giả định, như thể gửi cho quản lý: tiêu chí, ứng viên, kết quả đo, khuyến nghị, rủi ro.

## Checklist

- [ ] Biết các tầng model và đặc điểm của từng nhóm.
- [ ] Chọn model bằng eval trên dữ liệu của mình, benchmark chỉ để lọc ứng viên.
- [ ] Tính được chi phí trên mỗi tác vụ hoàn thành.
- [ ] Chạy được một model mở trên máy cá nhân.
- [ ] Cấu hình model ở một chỗ, có kế hoạch cho việc nâng cấp model.

## Đọc thêm

- [Models](../../oss/python/langchain/models.md): `init_chat_model` và danh sách nhà cung cấp.
- [Model fallback middleware](../../oss/python/langchain/middleware/built-in.md#model-fallback): tự chuyển sang model dự phòng khi lỗi.
- [Track ML nền tảng](../ml-nen-tang/index.md): fine-tune và tự chạy model.

---

**Bài trước:** [Bài 4: Tool use từ con số 0](04-tool-use.md) · [Tổng quan track](index.md) · **Track tiếp theo:** [Prompt & Context Engineering](../prompt-context/index.md)
