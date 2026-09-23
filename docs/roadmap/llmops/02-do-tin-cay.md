# Bài 2: Độ tin cậy

!!! abstract "Tóm tắt bài học"
    - **Mục tiêu:** giữ ứng dụng hoạt động khi phụ thuộc bên ngoài (API LLM, tool, database) gặp sự cố: timeout, retry đúng cách, quản lý rate limit, fallback, circuit breaker, kiểm tra và sửa output, xuống cấp có kiểm soát.
    - **Thời lượng:** 4 đến 5 ngày.
    - **Yêu cầu trước:** [Bài 1](01-observability.md), [Hiểu LLM, Bài 3](../hieu-llm/03-claude-api.md#9-xu-ly-loi-va-retry).

## 1. Những thứ có thể hỏng

| Sự cố | Biểu hiện | Tần suất |
|---|---|---|
| Rate limit | HTTP 429 | Thường xuyên khi lưu lượng tăng |
| Nhà cung cấp quá tải | HTTP 529 (overloaded), 5xx | Thỉnh thoảng, theo đợt |
| Timeout, lỗi mạng | Kết nối treo, bị ngắt | Thỉnh thoảng |
| Sự cố diện rộng của nhà cung cấp | Mọi request lỗi trong nhiều phút | Hiếm, nhưng chắc chắn sẽ xảy ra |
| Output bị cắt | `stop_reason == "max_tokens"` | Tùy cấu hình |
| Từ chối | `stop_reason == "refusal"` | Hiếm với ứng dụng hợp lệ |
| Output sai định dạng hoặc sai logic nghiệp vụ | Parse lỗi, giá trị vô lý | Tùy tác vụ |
| Tool lỗi | API nội bộ, database chậm hoặc lỗi | Tùy hệ thống |

Nguyên tắc: **mọi phụ thuộc bên ngoài đều sẽ hỏng vào lúc nào đó**. Câu hỏi là hệ thống của bạn phản ứng thế nào.

## 2. Timeout

Không đặt timeout, một request treo có thể giữ tài nguyên (worker, kết nối) mãi mãi. Đặt timeout ở **mọi tầng**:

```python
import anthropic

client = anthropic.Anthropic(
    timeout=anthropic.Timeout(60.0, connect=5.0),  # tổng 60s, kết nối tối đa 5s
    max_retries=2,
)
```

- Output dài: dùng **streaming**. Timeout khi streaming áp dụng cho khoảng lặng giữa các đoạn dữ liệu, nên request dài vẫn chạy được mà không treo vô hạn.
- **Timeout tổng cho cả tác vụ** (ví dụ agent tối đa 90 giây), không chỉ cho từng lệnh gọi. Retry làm thời gian thực tế có thể lên tới `timeout × (số lần retry + 1)`.
- Timeout của server web, load balancer, proxy phải **dài hơn** timeout của ứng dụng, nếu không kết nối bị cắt trước khi ứng dụng kịp trả lỗi đẹp.

## 3. Retry đúng cách

### 3.1 Lỗi nào nên retry?

| Nên retry | Không nên retry |
|---|---|
| 429, 5xx, 529, timeout, lỗi mạng | 400 (request sai), 401, 403 (xác thực, quyền), 404 (sai model) |
| | `refusal`: retry nguyên văn thường cho kết quả tương tự |

### 3.2 Backoff có jitter

Retry ngay lập tức khi nhà cung cấp đang quá tải chỉ làm tình hình tệ hơn. Dùng **backoff tăng dần** (1s, 2s, 4s...) cộng **jitter** (thêm ngẫu nhiên) để hàng nghìn client không retry cùng một lúc.

SDK Anthropic đã làm điều này sẵn (mặc định 2 lần). Những lưu ý:

- **Đừng retry chồng nhiều tầng.** SDK retry 2 lần, code của bạn retry 3 lần, middleware retry thêm 3 lần: một lỗi thành 36 request. Chọn **một** tầng chịu trách nhiệm retry.
- **Tôn trọng header `retry-after`** khi nhà cung cấp gửi về.
- **Ngân sách retry:** khi tỉ lệ lỗi cao bất thường, giảm hoặc ngừng retry để không làm quá tải thêm (xem circuit breaker ở mục 6).

### 3.3 Idempotency

Retry an toàn với lệnh gọi LLM (chỉ tốn thêm tiền). Nhưng với **tool có tác dụng phụ** (tạo đơn, hoàn tiền, gửi email), retry có thể làm việc đó **hai lần**. Dùng khóa idempotency: mỗi thao tác có một ID duy nhất, hệ thống từ chối thực hiện lại cùng ID.

## 4. Quản lý rate limit

Nhà cung cấp giới hạn theo **số request mỗi phút** và **số token mỗi phút** (tách riêng input và output). Đọc trạng thái từ header phản hồi:

```python
raw = client.messages.with_raw_response.create(
    model="claude-opus-5", max_tokens=16000,
    messages=[{"role": "user", "content": "Xin chào"}],
)
for name in ("requests-remaining", "input-tokens-remaining", "output-tokens-remaining"):
    print(name, raw.headers.get(f"anthropic-ratelimit-{name}"))
message = raw.parse()
```

Chiến lược:

- **Giới hạn đồng thời phía client** (semaphore, [Nền tảng kỹ thuật, Bài 1](../nen-tang-ky-thuat/01-python-hien-dai.md#5-async-goi-nhieu-llm-cung-luc)).
- **Hàng đợi có ưu tiên:** request của người dùng đang chờ được ưu tiên hơn tác vụ nền.
- **Tách khối lượng lớn sang Batch API**: có hạn mức riêng, tách khỏi hạn mức của request thông thường, và rẻ hơn 50%.
- **Prompt caching** giúp giảm áp lực lên hạn mức token input.
- Theo dõi mức sử dụng và **xin nâng hạn mức trước** các đợt cao điểm (khuyến mãi, sự kiện).

## 5. Fallback

Khi đường chính không dùng được, có đường dự phòng:

| Loại fallback | Cách làm |
|---|---|
| **Model dự phòng** | Model chính quá tải thì chuyển sang model khác. LangChain: [`ModelFallbackMiddleware`](../../oss/python/langchain/middleware/built-in.md#model-fallback) |
| **Nền tảng dự phòng** | Cùng model Claude qua nhà cung cấp đám mây khác (Amazon Bedrock, Google Vertex AI) với client tương ứng |
| **Fallback khi bị từ chối** | Với một số model, API hỗ trợ tự chạy lại request trên model khác khi bị từ chối ([Hiểu LLM, Bài 3](../hieu-llm/03-claude-api.md#11-khi-model-tu-choi-refusal)) |
| **Câu trả lời dự phòng** | Trả kết quả đã cache, câu trả lời mẫu, hoặc thông báo "hệ thống bận" có hướng dẫn |
| **Chuyển cho con người** | Chatbot không trả lời được thì chuyển nhân viên, kèm lịch sử hội thoại |

!!! warning "Fallback cũng cần được eval"
    Model dự phòng có thể cho chất lượng khác, thậm chí cần prompt khác. Chạy eval trên đường dự phòng như với đường chính, và theo dõi tỉ lệ request đi qua fallback. Fallback chạy âm thầm 30% thời gian là một sự cố cần điều tra.

## 6. Circuit breaker

Khi một phụ thuộc đang hỏng hàng loạt, tiếp tục gọi nó chỉ làm chậm hệ thống (mỗi request chờ timeout) và làm phụ thuộc đó khó hồi phục hơn. **Circuit breaker** ngắt mạch: sau N lỗi liên tiếp, **ngừng gọi** trong một khoảng thời gian, chuyển thẳng sang fallback, rồi thử lại dè dặt.

```python title="app/circuit.py"
import time

import anthropic


class CircuitBreaker:
    def __init__(self, failure_threshold: int = 5, reset_after: float = 30.0):
        self.failure_threshold = failure_threshold
        self.reset_after = reset_after
        self.failures = 0
        self.opened_at: float | None = None

    def allow(self) -> bool:
        if self.opened_at is None:
            return True
        if time.monotonic() - self.opened_at >= self.reset_after:
            return True  # nửa mở: cho thử một request
        return False

    def record_success(self) -> None:
        self.failures, self.opened_at = 0, None

    def record_failure(self) -> None:
        self.failures += 1
        if self.failures >= self.failure_threshold:
            self.opened_at = time.monotonic()


primary = CircuitBreaker()


def answer(question: str) -> str:
    if primary.allow():
        try:
            result = call_primary_model(question)
            primary.record_success()
            return result
        except (anthropic.APIStatusError, anthropic.APIConnectionError):
            primary.record_failure()
    return call_fallback(question)  # model dự phòng, cache, hoặc thông báo bận
```

Trong hệ thống nhiều instance, trạng thái circuit breaker nên được chia sẻ (ví dụ qua Redis), hoặc dùng thư viện có sẵn.

## 7. Kiểm tra và sửa output

Request thành công chưa có nghĩa là output dùng được.

1. **Kiểm tra `stop_reason`** trước tiên: `max_tokens` (bị cắt), `refusal` (từ chối).
2. **Structured output** để loại bỏ lỗi định dạng ([Hiểu LLM, Bài 3](../hieu-llm/03-claude-api.md#6-structured-output)).
3. **Kiểm tra nghiệp vụ bằng code:** số tiền trong khoảng hợp lệ, ngày không ở quá khứ, mã sản phẩm có tồn tại.
4. **Sửa bằng phản hồi lỗi:** nếu kiểm tra thất bại, gọi lại một lần kèm thông báo lỗi cụ thể; model thường tự sửa được.

```python
def extract_with_validation(text: str, max_attempts: int = 2) -> Order:
    messages = [{"role": "user", "content": f"Trích xuất đơn hàng:\n{text}"}]
    for _ in range(max_attempts):
        order = client.messages.parse(
            model="claude-opus-5", max_tokens=16000, messages=messages, output_format=Order,
        ).parsed_output
        errors = validate_business_rules(order)  # trả về danh sách lỗi, rỗng nếu hợp lệ
        if not errors:
            return order
        messages += [
            {"role": "assistant", "content": order.model_dump_json()},
            {"role": "user", "content": f"Kết quả chưa hợp lệ: {errors}. Hãy sửa lại."},
        ]
    raise ValueError(f"Không trích xuất được đơn hàng hợp lệ: {errors}")
```

## 8. Agent chạy lâu: thực thi bền vững

Agent chạy 10 phút có thể gặp sự cố ở phút thứ 9. Chạy lại từ đầu vừa tốn tiền vừa có thể lặp lại tác dụng phụ. LangGraph lưu **checkpoint** sau mỗi bước, cho phép tiếp tục từ bước cuối cùng đã thành công. Xem [Persistence](../../oss/python/langgraph/persistence.md) và [Fault Tolerance](../../oss/python/langgraph/fault-tolerance.md).

## 9. Kiểm thử độ tin cậy

Đừng chờ sự cố thật mới biết hệ thống phản ứng ra sao. Chủ động **giả lập lỗi**:

- Client giả trả 429, 529, timeout theo tỉ lệ ngẫu nhiên (như client giả ở [Nền tảng kỹ thuật, Bài 4](../nen-tang-ky-thuat/04-test-docker-ci.md#3-api-test-voi-client-gia)).
- Tool giả chậm 30 giây hoặc trả lỗi.
- Kiểm tra: người dùng nhận thông báo gì? Có request nào treo mãi? Chi phí có tăng vọt vì retry? Cảnh báo có kích hoạt?

## Bài tập

**Bài 2.1.** Rà soát ứng dụng của bạn: liệt kê mọi phụ thuộc bên ngoài, timeout hiện tại, số tầng retry. Sửa những chỗ không có timeout hoặc retry chồng tầng.

**Bài 2.2.** Viết client giả trả lỗi 529 cho 50% request. Chạy 100 request qua ứng dụng, đo tỉ lệ thành công, độ trễ, số request thực tế gửi đi.

**Bài 2.3.** Thêm circuit breaker và fallback (model dự phòng hoặc thông báo bận). Chạy lại bài 2.2 với 100% lỗi trong 1 phút, rồi hết lỗi. Quan sát mạch ngắt và mở lại.

**Bài 2.4.** Thêm vòng kiểm tra nghiệp vụ và sửa lỗi cho một tác vụ trích xuất. Đo tỉ lệ output hợp lệ trước và sau.

## Checklist

- [ ] Mọi phụ thuộc có timeout; có timeout tổng cho mỗi tác vụ.
- [ ] Chỉ một tầng chịu trách nhiệm retry, có backoff và jitter, chỉ retry lỗi tạm thời.
- [ ] Tool có tác dụng phụ dùng khóa idempotency.
- [ ] Có fallback và circuit breaker; tỉ lệ đi qua fallback được theo dõi.
- [ ] Output được kiểm tra `stop_reason`, schema và quy tắc nghiệp vụ.
- [ ] Đã giả lập lỗi để kiểm chứng hành vi của hệ thống.

## Đọc thêm

- [Middleware có sẵn](../../oss/python/langchain/middleware/built-in.md): Model retry, Model fallback, Tool retry, Model call limit.
- [Fault Tolerance](../../oss/python/langgraph/fault-tolerance.md)
- Các mã lỗi: [MODEL_RATE_LIMIT](../../oss/python/langchain/errors/MODEL_RATE_LIMIT.md), [OUTPUT_PARSING_FAILURE](../../oss/python/langchain/errors/OUTPUT_PARSING_FAILURE.md)

---

**Bài trước:** [Bài 1: Observability](01-observability.md) · [Tổng quan track](index.md) · **Bài tiếp theo:** [Bài 3: Chi phí và độ trễ](03-chi-phi-do-tre.md)
