# Bài 1: Observability

!!! abstract "Tóm tắt bài học"
    - **Mục tiêu:** trả lời được mọi câu hỏi "chuyện gì đã xảy ra với request này" và "hệ thống đang khỏe không" bằng trace, metric và log; thiết lập tracing với LangSmith; xây dashboard và cảnh báo cho ứng dụng LLM.
    - **Thời lượng:** 4 đến 5 ngày.
    - **Yêu cầu trước:** đã có một ứng dụng LLM chạy được.

## 1. Vì sao ứng dụng LLM cần observability đặc biệt?

Với API truyền thống, khi có lỗi bạn thường thấy exception và stack trace. Với ứng dụng LLM, phần lớn "lỗi" **không phải exception**: request trả về 200, nhưng câu trả lời sai, bịa, lạc đề, hoặc agent đi 25 bước thay vì 3. Để debug, bạn cần thấy **toàn bộ những gì model đã nhìn thấy và đã làm**.

| Trụ cột | Trả lời câu hỏi | Ví dụ trong ứng dụng LLM |
|---|---|---|
| **Trace** | Chuyện gì đã xảy ra với **một** request cụ thể? | Prompt thực tế, chunk retrieve được, từng tool call, câu trả lời |
| **Metric** | Hệ thống **nói chung** đang thế nào? | Độ trễ p95, chi phí mỗi giờ, tỉ lệ lỗi, tỉ lệ cache hit, tỉ lệ 👎 |
| **Log** | Sự kiện rời rạc | Lỗi kết nối, cảnh báo cấu hình |

## 2. Trace và span

Một **trace** là cây các **span**, mỗi span là một bước có thời gian bắt đầu, kết thúc, input, output và thuộc tính:

```text
POST /chat  (3.4s)
├── rewrite_query        claude-haiku-4-5   (0.4s, 180 → 25 token)
├── retrieve             pgvector + bm25    (0.08s, 8 chunk)
├── rerank               bge-reranker       (0.3s)
└── agent
    ├── model call #1    claude-opus-5      (1.1s, 4.200 → 90 token, cache_read 3.800)
    ├── tool: get_order  (0.05s)
    └── model call #2    claude-opus-5      (1.4s, 4.450 → 210 token, cache_read 4.200)
```

Nhìn trace này, bạn biết ngay: bước nào chậm, bước nào tốn, prompt cache có hoạt động không, và (khi mở từng span) model đã thấy gì ở mỗi bước.

## 3. Cần ghi lại những gì?

Với mỗi lệnh gọi LLM:

| Nhóm | Thuộc tính |
|---|---|
| Định danh | `trace_id`, `user_id` (hoặc mã đã ẩn danh), `tenant_id`, tính năng, phiên bản ứng dụng |
| Cấu hình | Model, phiên bản prompt, effort, tham số khác |
| Token | Input, output, cache read, cache write |
| Thời gian | Tổng độ trễ, thời gian tới token đầu tiên (TTFT) |
| Kết quả | `stop_reason`, lỗi (loại, mã), `request_id` của nhà cung cấp |
| Nội dung | Prompt và output (**có kiểm soát truy cập**, xem mục 6) |
| Chi phí | Tính từ token và bảng giá |

## 4. Tracing với LangSmith

### 4.1 Ứng dụng LangChain, LangGraph

Chỉ cần bật biến môi trường, mọi lệnh gọi model, tool, retriever, node trong graph đều được ghi lại tự động:

```bash
LANGSMITH_TRACING=true
LANGSMITH_API_KEY=lsv2_...
LANGSMITH_PROJECT=shop-assistant-prod
```

Gắn thêm metadata để lọc và phân tích:

```python
result = agent.invoke(
    {"messages": [...]},
    config={
        "configurable": {"thread_id": thread_id},
        "metadata": {"user_id": user_hash, "prompt_version": "v7", "feature": "support_chat"},
        "tags": ["production"],
    },
)
```

Xem [Observability](../../oss/python/langchain/observability.md) và [LangSmith Observability cho LangGraph](../../oss/python/langgraph/observability.md).

### 4.2 Code dùng SDK trực tiếp

Với code gọi Anthropic SDK trực tiếp, bọc client và đánh dấu các hàm bằng `@traceable` để có cây span đầy đủ:

```python
import anthropic
from langsmith import traceable
from langsmith.wrappers import wrap_anthropic

client = wrap_anthropic(anthropic.Anthropic())  # mọi lệnh gọi model được ghi thành span


@traceable(name="retrieve")
def retrieve(query: str) -> list[str]:
    ...


@traceable(name="answer_question", metadata={"feature": "faq"})
def answer_question(question: str) -> str:
    docs = retrieve(question)  # span con
    response = client.messages.create(  # span con, có token và độ trễ
        model="claude-opus-5",
        max_tokens=16000,
        messages=[{"role": "user", "content": f"{docs}\n\n{question}"}],
    )
    return next(b.text for b in response.content if b.type == "text")
```

### 4.3 Các lựa chọn khác

- **Langfuse:** mã nguồn mở, tự host được, tính năng tương tự.
- **OpenTelemetry:** chuẩn mở cho tracing. Có quy ước thuộc tính dành riêng cho GenAI (tên model, số token...). Phù hợp khi công ty đã có hạ tầng observability chung (Grafana, Datadog...). Nhiều nền tảng trên nhận dữ liệu OpenTelemetry.

Chọn công cụ nào không quan trọng bằng việc **bật tracing từ ngày đầu tiên**. Trace từ lúc phát triển giúp debug nhanh hơn nhiều, và trace production là nguồn dữ liệu cho eval ([track Evaluation](../evaluation/index.md)).

## 5. Dashboard và cảnh báo

### 5.1 Dashboard tối thiểu

| Nhóm | Biểu đồ |
|---|---|
| Lưu lượng | Số request theo giờ, theo tính năng |
| Độ trễ | TTFT và tổng độ trễ p50, p95, p99 |
| Lỗi | Tỉ lệ lỗi theo loại (429, 5xx, timeout), tỉ lệ `max_tokens`, tỉ lệ `refusal` |
| Chi phí | Chi phí theo giờ, theo tính năng, theo khách hàng lớn nhất; chi phí trung bình mỗi request |
| Hiệu quả | Tỉ lệ cache hit (`cache_read / tổng input`), số lượt trung bình của agent |
| Chất lượng | Tỉ lệ 👎, kết quả giám khảo online, tỉ lệ chuyển cho nhân viên |

Nguồn dữ liệu: bảng log lệnh gọi LLM ([Nền tảng kỹ thuật, Bài 3](../nen-tang-ky-thuat/03-du-lieu-hang-doi.md#2-ghi-log-chi-phi-tung-lenh-goi-llm)), nền tảng tracing, và kết quả eval online ([Evaluation, Bài 4](../evaluation/04-online-eval.md)).

### 5.2 Cảnh báo

| Cảnh báo | Điều kiện gợi ý | Thường do |
|---|---|---|
| Chi phí tăng đột biến | Chi phí giờ vừa qua gấp 3 lần trung bình cùng giờ tuần trước | Agent lặp vô tận, bị lạm dụng, prompt cache hỏng |
| Tỉ lệ lỗi cao | Hơn 5% request lỗi trong 10 phút | Sự cố nhà cung cấp, hết hạn mức, sai cấu hình |
| Độ trễ cao | p95 vượt ngưỡng trong 15 phút | Nhà cung cấp chậm, context phình to |
| Cache hit giảm mạnh | Giảm hơn một nửa so với hôm qua | Ai đó sửa system prompt, thêm nội dung động vào đầu prompt |
| Chất lượng giảm | Tỉ lệ 👎 hoặc tỉ lệ trượt giám khảo tăng rõ | Phiên bản mới có lỗi, dữ liệu nguồn thay đổi |

## 6. Quyền riêng tư trong trace

Trace chứa **toàn bộ nội dung hội thoại**: thông tin cá nhân của khách hàng, dữ liệu nội bộ. Hãy đối xử với hệ thống tracing như một database nhạy cảm:

- **Phân quyền truy cập:** chỉ những người cần thiết mới xem được nội dung.
- **Ẩn hoặc che dữ liệu nhạy cảm** trước khi gửi đi. LangSmith hỗ trợ ẩn input, output (ví dụ biến môi trường `LANGSMITH_HIDE_INPUTS`, `LANGSMITH_HIDE_OUTPUTS`) và tùy biến việc che dữ liệu.
- **Chính sách lưu trữ:** chỉ giữ nội dung đầy đủ trong thời gian cần thiết; metric và thuộc tính không nhạy cảm có thể giữ lâu hơn.
- **Tuân thủ:** kiểm tra yêu cầu của khách hàng và pháp luật (ví dụ quy định về bảo vệ dữ liệu cá nhân) về việc lưu và chuyển dữ liệu ra nước ngoài.

## Bài tập

**Bài 1.1.** Bật LangSmith cho một ứng dụng LangChain của bạn. Chạy 20 request, mở trace của request chậm nhất và tìm bước gây chậm.

**Bài 1.2.** Với một đoạn code dùng Anthropic SDK trực tiếp, thêm `wrap_anthropic` và `@traceable` để có cây span gồm ít nhất 3 cấp.

**Bài 1.3.** Xây dashboard (Grafana, Metabase, hoặc một trang Streamlit) từ bảng log lệnh gọi LLM, gồm ít nhất: request theo giờ, độ trễ p95, chi phí theo tính năng, tỉ lệ cache hit.

**Bài 1.4.** Cố tình phá prompt cache (chèn thời gian hiện tại vào đầu system prompt). Kiểm tra dashboard và cảnh báo có phát hiện không.

## Checklist

- [ ] Tracing được bật từ môi trường phát triển tới production.
- [ ] Mỗi lệnh gọi LLM ghi đủ: model, phiên bản prompt, token (cả cache), độ trễ, `stop_reason`, chi phí.
- [ ] Có dashboard về lưu lượng, độ trễ, lỗi, chi phí, hiệu quả, chất lượng.
- [ ] Có cảnh báo cho chi phí, lỗi, độ trễ, cache hit.
- [ ] Nội dung trace được bảo vệ: phân quyền, che dữ liệu nhạy cảm, chính sách lưu trữ.

## Đọc thêm

- [Observability](../../oss/python/langchain/observability.md), [LangSmith Studio](../../oss/python/langchain/studio.md)
- [LangSmith Observability cho LangGraph](../../oss/python/langgraph/observability.md)
- [OpenTelemetry](https://opentelemetry.io/)

---

[Tổng quan track](index.md) · **Bài tiếp theo:** [Bài 2: Độ tin cậy](02-do-tin-cay.md)
