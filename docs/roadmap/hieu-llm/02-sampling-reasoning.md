# Bài 2: Sampling, reasoning và context

!!! abstract "Tóm tắt bài học"
    - **Mục tiêu:** hiểu cách model chọn token (sampling), cách model "suy nghĩ" trước khi trả lời (reasoning, adaptive thinking, effort), giới hạn của context window, và cách đọc stop reason.
    - **Thời lượng:** 3 đến 4 ngày.
    - **Yêu cầu trước:** [Bài 1](01-transformer-token.md).

## 1. Từ xác suất tới token: sampling

Ở Bài 1, model trả về một phân phối xác suất cho token tiếp theo. Chọn token nào từ phân phối đó gọi là **sampling** (lấy mẫu).

| Chiến lược | Cách chọn | Đặc điểm |
|---|---|---|
| Greedy | Luôn chọn token xác suất cao nhất | Ổn định nhưng dễ lặp lại, nhàm chán |
| Sampling thuần | Chọn ngẫu nhiên theo xác suất | Đa dạng nhưng đôi khi chọn token rất vô lý |
| **Temperature** | Làm "nhọn" hoặc "bẹt" phân phối trước khi chọn | Thấp: bảo thủ. Cao: sáng tạo, liều lĩnh |
| **Top-k** | Chỉ chọn trong k token cao nhất | Cắt bỏ đuôi vô lý |
| **Top-p** (nucleus) | Chỉ chọn trong nhóm token nhỏ nhất có tổng xác suất đạt p | Như top-k nhưng tự điều chỉnh theo độ "chắc chắn" |

Temperature chia điểm thô (logits) trước khi áp dụng softmax. Chạy thử để thấy tác động:

```python
import numpy as np

tokens = [" Hà", " thành", " Sài", " nơi", " một"]
logits = np.array([6.0, 3.5, 2.0, 1.5, 1.0])


def softmax(x: np.ndarray, temperature: float) -> np.ndarray:
    z = x / temperature
    e = np.exp(z - z.max())
    return e / e.sum()


for t in (0.2, 0.7, 1.0, 1.5):
    probs = softmax(logits, t)
    print(f"T={t}: " + "  ".join(f"{tok!r}={p:.2f}" for tok, p in zip(tokens, probs)))
```

Với `T=0.2`, " Hà" chiếm gần như 100%. Với `T=1.5`, các token khác có cơ hội đáng kể.

### 1.1 Tham số sampling trên API hiện nay

!!! warning "Model Claude thế hệ mới không còn nhận tham số sampling"
    Trên các model Claude mới nhất (Opus 5, Sonnet 5, Fable 5.1...), `temperature`, `top_p`, `top_k` **đã bị loại bỏ**: truyền vào sẽ nhận lỗi 400. Model tự điều chỉnh, và bạn kiểm soát độ "kỹ lưỡng" bằng tham số `effort` (mục 2.3). Các model cũ hơn như Haiku 4.5 và nhiều nhà cung cấp khác vẫn nhận `temperature`.

    Bài học rút ra: **đừng xây hệ thống phụ thuộc vào temperature = 0 để có output ổn định**. Hãy dùng structured output để ép định dạng, và eval để đo độ ổn định.

**LLM không hoàn toàn xác định (deterministic).** Ngay cả với temperature bằng 0 trên các model còn nhận tham số này, cùng một prompt vẫn có thể cho kết quả hơi khác nhau do cách phần cứng tính toán song song. Đừng viết test so khớp nguyên văn output của LLM.

## 2. Reasoning: để model suy nghĩ trước khi trả lời

### 2.1 Chain of thought

Vì model sinh từng token và mỗi token là "một bước tính toán", việc **viết ra các bước trung gian** giúp model giải được bài toán khó hơn. Đây là kỹ thuật **chain of thought**: yêu cầu model suy nghĩ từng bước trước khi đưa ra đáp án.

### 2.2 Reasoning model và adaptive thinking

Các model hiện đại được huấn luyện để tự suy nghĩ trong một khối **thinking** riêng trước khi trả lời. Với Claude, chế độ **adaptive thinking** để model tự quyết định có cần suy nghĩ không và suy nghĩ bao lâu, tùy độ khó của câu hỏi.

```python title="thinking_demo.py"
import anthropic

client = anthropic.Anthropic()

response = client.messages.create(
    model="claude-opus-5",
    max_tokens=16000,
    thinking={"type": "adaptive", "display": "summarized"},  # hiện bản tóm tắt suy nghĩ
    messages=[{
        "role": "user",
        "content": "Một cửa hàng giảm giá 20%, sau đó tăng giá 20% so với giá đã giảm. "
                   "Giá cuối so với giá gốc thay đổi bao nhiêu phần trăm?",
    }],
)

for block in response.content:
    if block.type == "thinking":
        print("[Tóm tắt suy nghĩ]\n", block.thinking, "\n")
    elif block.type == "text":
        print("[Trả lời]\n", block.text)

print(response.usage)
```

Những điểm cần biết:

- **Token suy nghĩ được tính phí như token output.** Suy nghĩ nhiều thì chất lượng tốt hơn với bài khó, nhưng đắt hơn và chậm hơn.
- Nội dung suy nghĩ thô của model không được trả về. Mặc định khối `thinking` có nội dung rỗng; đặt `"display": "summarized"` để nhận bản tóm tắt dễ đọc (hữu ích khi muốn hiển thị "đang suy nghĩ..." cho người dùng).
- `response.content` có thể chứa khối `thinking` **trước** khối `text`. Luôn lọc theo `block.type`, đừng giả định `content[0]` là câu trả lời.

### 2.3 Effort: điều chỉnh độ kỹ lưỡng

Tham số `effort` điều chỉnh mức độ model đầu tư vào một yêu cầu: suy nghĩ sâu đến đâu, gọi tool nhiều hay ít, trả lời dài hay ngắn.

```python
response = client.messages.create(
    model="claude-opus-5",
    max_tokens=16000,
    output_config={"effort": "low"},  # low | medium | high | xhigh | max
    messages=[{"role": "user", "content": "Phân loại cảm xúc: 'Giao hàng nhanh, đóng gói đẹp'"}],
)
```

| Effort | Phù hợp |
|---|---|
| `low` | Phân loại, trích xuất đơn giản, tác vụ số lượng lớn, cần độ trễ thấp |
| `medium` | Tác vụ thường ngày, tiết kiệm chi phí mà chất lượng vẫn đạt |
| `high` (mặc định) | Phần lớn tác vụ cần sự thông minh |
| `xhigh` | Lập trình, agent chạy nhiều bước |
| `max` | Khi độ chính xác quan trọng hơn chi phí |

!!! tip "Hạ effort trước khi đổi sang model nhỏ hơn"
    Để giảm chi phí, thử **model mạnh với effort thấp** trước khi chuyển sang model yếu hơn. Model mới ở effort thấp thường vẫn tốt hơn model cũ ở effort cao. Chọn effort theo từng loại tác vụ và **đo trên dữ liệu thật**, không đặt một mức chung cho mọi thứ.

## 3. Context window

**Context window** là tổng số token model xử lý được trong một request: system prompt, lịch sử hội thoại, tài liệu, định nghĩa tool, **cộng cả** phần output. Các model Claude hiện tại có context tới 1 triệu token.

Context lớn không có nghĩa là nên dùng hết:

| Vấn đề | Giải thích |
|---|---|
| **Chi phí** | Mỗi request trả tiền cho toàn bộ input. Hội thoại 100 lượt gửi lại cả 100 lượt mỗi lần |
| **Độ trễ** | Prompt dài hơn thì thời gian tới token đầu tiên dài hơn |
| **Chất lượng** | Thông tin quan trọng bị "chìm" giữa lượng lớn thông tin không liên quan. Nghiên cứu [Lost in the Middle](https://arxiv.org/abs/2307.03172) cho thấy thông tin ở giữa context dài dễ bị bỏ qua hơn |

Nguyên tắc: **đưa vào context đúng những gì cần, không hơn**. Đây là nội dung chính của [Context Engineering](../prompt-context/03-context-engineering.md). Với hội thoại rất dài, có các kỹ thuật như tóm tắt lịch sử, xóa kết quả tool cũ, hoặc compaction phía server.

## 4. `max_tokens` và stop reason

`max_tokens` là **giới hạn cứng** số token output (bao gồm cả token suy nghĩ). Model **không biết** giới hạn này; nó chỉ bị cắt ngang khi chạm tới.

Mỗi response có trường `stop_reason` cho biết vì sao model dừng. **Luôn kiểm tra nó**:

| `stop_reason` | Ý nghĩa | Cần làm gì |
|---|---|---|
| `end_turn` | Model trả lời xong tự nhiên | Dùng kết quả |
| `max_tokens` | **Bị cắt cụt** | Tăng `max_tokens`, dùng streaming, hoặc yêu cầu ngắn hơn. Đừng dùng output dở dang như thể nó hoàn chỉnh |
| `tool_use` | Model muốn gọi tool | Chạy tool, gửi kết quả lại (Bài 4) |
| `stop_sequence` | Gặp chuỗi dừng bạn đặt | Xử lý theo logic của bạn |
| `pause_turn` | Tool phía server chạy lâu, lượt bị tạm dừng | Gửi lại để model tiếp tục |
| `refusal` | Model từ chối vì lý do an toàn | Xem `stop_details`, xử lý phù hợp (Bài 3) |

```python
if response.stop_reason == "max_tokens":
    raise RuntimeError("Câu trả lời bị cắt cụt, cần tăng max_tokens")
```

!!! warning "Đặt `max_tokens` đủ rộng"
    Đặt `max_tokens` quá thấp để "tiết kiệm" thường phản tác dụng: câu trả lời bị cắt, phải gọi lại, tốn tiền gấp đôi. Với request không streaming, khoảng 16.000 là mức an toàn; với streaming có thể đặt cao hơn nhiều. Chỉ đặt thấp cho tác vụ chắc chắn ngắn (ví dụ phân loại). Muốn câu trả lời ngắn, hãy **yêu cầu trong prompt**.

## Bài tập

**Bài 2.1.** Chạy đoạn code softmax với temperature 0.1, 1, 5. Giải thích bằng lời vì sao temperature rất cao làm output vô nghĩa.

**Bài 2.2.** Chọn 3 bài toán: một câu dễ (phép cộng), một câu trung bình (bài toán giảm giá ở trên), một câu khó (bài logic nhiều bước). Chạy mỗi câu với effort `low`, `high`, `max`. Ghi lại: đúng hay sai, số token output, thời gian.

??? tip "Gợi ý"
    ```python
    import time

    for effort in ("low", "high", "max"):
        start = time.perf_counter()
        r = client.messages.create(
            model="claude-opus-5", max_tokens=32000,
            output_config={"effort": effort},
            messages=[{"role": "user", "content": question}],
        )
        answer = next(b.text for b in r.content if b.type == "text")
        print(effort, r.usage.output_tokens, f"{time.perf_counter() - start:.1f}s", answer[:80])
    ```
    Bạn sẽ thấy câu dễ đúng ở mọi mức, nhưng effort cao tốn nhiều token hơn vô ích. Đó là lý do nên chọn effort theo loại tác vụ.

**Bài 2.3.** Gửi yêu cầu "Viết một bài luận 2.000 chữ về lịch sử Hà Nội" với `max_tokens=500`. Kiểm tra `stop_reason`, và quan sát câu trả lời bị cắt ở đâu.

**Bài 2.4.** Chạy cùng một prompt "Viết một câu slogan cho quán cà phê" 5 lần. Các câu trả lời có giống nhau không? Điều đó ảnh hưởng thế nào tới cách bạn viết test?

## Checklist

- [ ] Giải thích được temperature và top-p, và biết model nào còn nhận tham số này.
- [ ] Dùng được adaptive thinking, hiểu token suy nghĩ cũng bị tính phí.
- [ ] Chọn effort theo loại tác vụ dựa trên đo đạc.
- [ ] Luôn kiểm tra `stop_reason` trước khi dùng kết quả.
- [ ] Hiểu vì sao không nên nhồi đầy context window.

## Đọc thêm

- [Models](../../oss/python/langchain/models.md): cấu hình model trong LangChain, bao gồm reasoning.
- [Streaming Reasoning Tokens](../../oss/python/langchain/frontend/reasoning-tokens.md): hiển thị quá trình suy nghĩ trên giao diện.
- [Lost in the Middle](https://arxiv.org/abs/2307.03172).

---

**Bài trước:** [Bài 1: LLM hoạt động thế nào](01-transformer-token.md) · [Tổng quan track](index.md) · **Bài tiếp theo:** [Bài 3: Claude API chuyên sâu](03-claude-api.md)
