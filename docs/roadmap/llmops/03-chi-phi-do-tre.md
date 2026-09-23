# Bài 3: Chi phí và độ trễ

!!! abstract "Tóm tắt bài học"
    - **Mục tiêu:** hiểu chi phí đến từ đâu, áp dụng các đòn bẩy tiết kiệm theo đúng thứ tự (miễn phí trước, đánh đổi sau), giảm độ trễ mà người dùng cảm nhận, và đặt ngân sách, hạn mức để không bao giờ bị bất ngờ.
    - **Thời lượng:** 4 đến 5 ngày.
    - **Yêu cầu trước:** [Bài 1](01-observability.md), đã có log token và chi phí từng lệnh gọi.

## 1. Đo trước khi tối ưu

Trước khi tối ưu, cần một **hồ sơ token** (token profile): chi phí đang nằm ở đâu.

```sql
SELECT feature,
       count(*)                                        AS calls,
       round(sum(cost_usd)::numeric, 2)                AS cost_usd,
       round(avg(input_tokens + cache_read_tokens))    AS avg_input,
       round(avg(output_tokens))                       AS avg_output,
       round(100.0 * sum(cache_read_tokens)
             / nullif(sum(input_tokens + cache_read_tokens + cache_write_tokens), 0), 1) AS cache_hit_pct
FROM llm_calls
WHERE created_at > now() - interval '7 days'
GROUP BY feature
ORDER BY cost_usd DESC;
```

Câu hỏi cần trả lời:

- **Tính năng nào tốn nhiều nhất?** Thường 1 đến 2 tính năng chiếm phần lớn chi phí. Tập trung vào đó.
- **Chi phí nằm ở input hay output?** Input lớn (prompt dài, lịch sử dài, tài liệu) hay output lớn (câu trả lời dài, suy nghĩ nhiều)?
- **Cache hit bao nhiêu?** Thấp bất thường là dấu hiệu cơ hội tiết kiệm lớn.
- **Agent tốn bao nhiêu lượt** cho mỗi tác vụ?

Và luôn tính **chi phí trên mỗi tác vụ hoàn thành**, không chỉ trên mỗi request: một request rẻ nhưng phải làm lại hai lần thì không rẻ.

## 2. Các đòn bẩy, theo thứ tự ưu tiên

Làm **các đòn bẩy miễn phí** (không ảnh hưởng chất lượng) trước, rồi mới tới **các đòn bẩy đánh đổi** (có thể ảnh hưởng chất lượng, phải kiểm chứng bằng eval).

### 2.1 Miễn phí: prompt caching

Thường là đòn bẩy lớn nhất. Phần prompt lặp lại (system prompt, định nghĩa tool, tài liệu, lịch sử hội thoại) đọc từ cache chỉ tốn khoảng 10% giá input.

- Sắp xếp prompt: **phần cố định trước, phần thay đổi sau**.
- Loại bỏ nội dung động khỏi phần đầu (thời gian hiện tại, ID request, dữ liệu người dùng trong system prompt).
- Giữ danh sách tool **ổn định và cùng thứ tự** giữa các request.
- Với hội thoại nhiều lượt và agent, cache cả lịch sử hội thoại để mỗi lượt mới chỉ trả giá đầy đủ cho phần mới thêm.
- **Kiểm chứng** bằng `cache_read_input_tokens`. Chi tiết: [Hiểu LLM, Bài 3](../hieu-llm/03-claude-api.md#8-prompt-caching).

### 2.2 Miễn phí: vệ sinh input

- **Cắt output của tool:** tool trả về 50 trường khi chỉ cần 5; trả cả trang HTML khi chỉ cần nội dung chính.
- **Giới hạn lịch sử:** tóm tắt hoặc xóa kết quả tool cũ ([Context engineering](../prompt-context/03-context-engineering.md#43-nen-giam-kich-thuoc-nhung-giu-y-chinh)).
- **RAG gọn:** rerank để đưa 4 chunk tốt thay vì 15 chunk vừa vừa.
- **Bỏ phần prompt thừa:** chỉ dẫn không còn tác dụng, ví dụ trùng lặp. Kiểm chứng bằng eval.

### 2.3 Miễn phí: vệ sinh vòng lặp agent

- Agent gọi tool thừa, tra đi tra lại cùng một thông tin? Cải thiện mô tả tool, gộp tool hay đi cùng nhau.
- Giới hạn số lượt, phát hiện vòng lặp.
- Tác vụ có quy trình cố định: dùng **workflow** thay vì agent ([Agents, Bài 1](../agents/01-agent-loop.md)).

### 2.4 Miễn phí: vệ sinh output

- Output đắt hơn input nhiều lần. Yêu cầu **đúng độ dài cần thiết**: "trả lời 2 đến 3 câu", "chỉ trả về nhãn".
- Dùng structured output cho tác vụ trích xuất, thay vì để model viết lời dẫn dài dòng.

### 2.5 Miễn phí: Batch API

Mọi tác vụ **không cần kết quả ngay** (xử lý dữ liệu hằng đêm, sinh mô tả hàng loạt, chạy eval) nên đi qua Batch API: **giảm 50%**. Xem [Hiểu LLM, Bài 3](../hieu-llm/03-claude-api.md#10-message-batches-xu-ly-hang-loat-re-hon-50).

### 2.6 Đánh đổi: effort

Hạ `effort` giảm token suy nghĩ và độ dài output. Nhiều tác vụ (phân loại, trích xuất, trò chuyện đơn giản) giữ nguyên chất lượng ở effort thấp; tác vụ lập trình, agent phức tạp thì cần effort cao. **Chọn theo từng tính năng, đo trên dữ liệu thật.**

### 2.7 Đánh đổi: chọn model và định tuyến

- Thử **model mạnh với effort thấp** trước khi hạ sang model nhỏ hơn.
- Tác vụ phụ trợ (phân loại ý định, viết lại câu hỏi, chấm điểm chunk) thường dùng tốt model nhỏ.
- Hệ thống nhiều model có chi phí ẩn: mỗi model có cache riêng, cần eval và giám sát riêng. Xem [Hiểu LLM, Bài 5](../hieu-llm/05-chon-model.md).

### 2.8 Tóm tắt

| Đòn bẩy | Loại | Mức tiết kiệm điển hình | Rủi ro chất lượng |
|---|---|---|---|
| Prompt caching | Miễn phí | Lớn với prompt dài lặp lại | Không |
| Vệ sinh input, output, vòng lặp | Miễn phí | Trung bình đến lớn | Thấp (kiểm chứng bằng eval) |
| Batch API | Miễn phí | 50% cho tác vụ không gấp | Không |
| Hạ effort | Đánh đổi | Trung bình | Có, cần eval |
| Đổi model, định tuyến | Đánh đổi | Lớn | Có, cần eval kỹ |

## 3. Độ trễ

### 3.1 Độ trễ nào quan trọng?

| Chỉ số | Ý nghĩa | Quan trọng với |
|---|---|---|
| **TTFT** (thời gian tới token đầu tiên) | Người dùng chờ bao lâu mới thấy phản hồi | Chat, giao diện tương tác |
| **Tốc độ sinh** (token/giây) | Chữ hiện ra nhanh hay chậm | Câu trả lời dài |
| **Tổng thời gian** | Từ lúc gửi tới lúc hoàn tất | API, tác vụ nền, agent |

### 3.2 Cách giảm

| Kỹ thuật | Tác động |
|---|---|
| **Streaming** | Giảm TTFT cảm nhận từ vài giây xuống khoảng một giây. Cải tiến trải nghiệm lớn nhất, gần như không tốn gì |
| **Prompt caching** | Giảm TTFT với prompt dài |
| **Output ngắn hơn** | Tổng thời gian tỉ lệ gần tuyến tính với số token output |
| **Effort thấp hơn** | Ít token suy nghĩ, bắt đầu trả lời sớm hơn |
| **Chạy song song** | Các bước độc lập (retrieve từ nhiều nguồn, gọi nhiều tool) chạy đồng thời |
| **Model nhỏ cho bước trung gian** | Viết lại câu hỏi, phân loại nhanh hơn |
| **Bỏ bước không cần thiết** | Mỗi lệnh gọi LLM thêm vài trăm mili giây trở lên. Bước rewrite, grading... có thực sự cải thiện eval? |
| **Tính trước** | Kết quả có thể chuẩn bị sẵn (tóm tắt tài liệu, gợi ý sản phẩm) thì tính trong tác vụ nền |
| **Hiển thị tiến trình** | Với agent chạy lâu, stream các bước ("Đang tra đơn hàng...") để người dùng biết hệ thống đang làm việc |

!!! tip "Chế độ tốc độ cao"
    Một số model cung cấp chế độ sinh nhanh hơn với giá cao hơn (ví dụ fast mode, đang ở dạng thử nghiệm, cho một số model Claude Opus). Chỉ cân nhắc khi độ trễ là yếu tố sống còn và các kỹ thuật ở trên đã được áp dụng.

## 4. Ngân sách và hạn mức

Không bao giờ để chi phí **không có giới hạn trên**.

| Cấp | Biện pháp |
|---|---|
| Mỗi request | Giới hạn độ dài input, `max_tokens` hợp lý, số lượt agent tối đa |
| Mỗi người dùng | Số request mỗi phút, số token mỗi ngày ([Nền tảng kỹ thuật, Bài 3](../nen-tang-ky-thuat/03-du-lieu-hang-doi.md#4-gioi-han-toc-o-va-ngan-sach-dung-chung)) |
| Mỗi khách hàng doanh nghiệp | Hạn mức theo gói dịch vụ; cảnh báo khi gần chạm |
| Toàn hệ thống | Giới hạn chi tiêu trên tài khoản nhà cung cấp; cảnh báo chi phí bất thường ([Bài 1](01-observability.md#52-canh-bao)) |
| Mỗi môi trường | API key riêng cho dev, staging, production, CI, mỗi key có giới hạn riêng |

!!! danger "Một vòng lặp có thể đốt hết ngân sách trong một đêm"
    Agent lặp vô tận, một script test chạy nhầm trên production, một bot spam API công khai của bạn: đều là những sự cố có thật. Giới hạn ở nhiều tầng và cảnh báo sớm là bắt buộc, không phải tùy chọn.

## Bài tập

**Bài 3.1.** Chạy truy vấn hồ sơ token ở mục 1 cho ứng dụng của bạn (hoặc dữ liệu giả lập). Xác định tính năng tốn nhất và nguồn chi phí chính (input, output, hay số lượt).

**Bài 3.2.** Áp dụng lần lượt các đòn bẩy miễn phí cho tính năng tốn nhất. Sau mỗi đòn bẩy, đo lại chi phí trung bình và chạy eval. Ghi vào bảng: đòn bẩy, chi phí trước và sau, chất lượng trước và sau.

**Bài 3.3.** Thử hạ effort cho 3 tính năng khác nhau. Tính năng nào giữ được chất lượng?

**Bài 3.4.** Đo TTFT và tổng thời gian của endpoint chat trước và sau khi bật streaming, prompt caching, và bỏ một bước trung gian không cần thiết.

## Checklist

- [ ] Có hồ sơ token theo tính năng; biết chi phí nằm ở đâu.
- [ ] Đã áp dụng các đòn bẩy miễn phí trước khi cân nhắc đánh đổi.
- [ ] Mọi đánh đổi (effort, model) được kiểm chứng bằng eval.
- [ ] Streaming cho mọi giao diện tương tác; đo TTFT.
- [ ] Có giới hạn chi phí ở mọi cấp và cảnh báo chi phí bất thường.

## Đọc thêm

- [Hiểu LLM, Bài 5: Chọn model](../hieu-llm/05-chon-model.md)
- [Context engineering](../prompt-context/03-context-engineering.md)
- [Models](../../oss/python/langchain/models.md), [Streaming](../../oss/python/langchain/streaming.md)

---

**Bài trước:** [Bài 2: Độ tin cậy](02-do-tin-cay.md) · [Tổng quan track](index.md) · **Bài tiếp theo:** [Bài 4: Bảo mật](04-bao-mat.md)
