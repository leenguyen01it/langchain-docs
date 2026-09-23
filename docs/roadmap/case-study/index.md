# Track: Case study thiết kế hệ thống

Các track trước dạy từng mảnh kỹ năng: RAG, agent, eval, vận hành. Công việc thật đòi hỏi **ghép các mảnh lại** cho một sản phẩm cụ thể, với ràng buộc cụ thể về quy mô, chi phí, độ trễ, bảo mật, và chọn giữa nhiều phương án đều "đúng". Track này luyện kỹ năng đó qua 5 case study, cũng là dạng câu hỏi phổ biến trong phỏng vấn AI Engineer.

| Case | Hệ thống | Kỹ năng trọng tâm |
|---|---|---|
| [1. Sinh mô tả sản phẩm](01-sinh-mo-ta-san-pham.md) | Shopify app sinh mô tả cho hàng nghìn cửa hàng | Đa tenant, realtime và batch, ước tính chi phí để định giá, hàng đợi công bằng |
| [2. Chatbot chăm sóc khách hàng](02-chatbot-cua-hang.md) | Widget chat trên cửa hàng, phục vụ khách của nhiều cửa hàng | RAG theo tenant, xác minh khách vãng lai, chống lạm dụng, chuyển người |
| [3. Trợ lý cho merchant](03-tro-ly-merchant.md) | Copilot trong trang quản trị: phân tích bán hàng, thực hiện thao tác | Tool có kiểu thay vì truy vấn tùy ý, số liệu chính xác, phê duyệt hành động |
| [4. Phân tích đánh giá hàng loạt](04-phan-tich-danh-gia.md) | Pipeline xử lý hàng triệu đánh giá mỗi tháng | Batch, thiết kế taxonomy, chưng cất sang model nhỏ, phát hiện vấn đề mới |
| [5. Tìm kiếm sản phẩm tiếng Việt](05-tim-kiem-san-pham.md) | Ô tìm kiếm hiểu ngôn ngữ tự nhiên, độ trễ dưới 300ms | Không có LLM trên đường nóng, làm giàu dữ liệu offline, hybrid search, đo relevance |

**Yêu cầu trước:** đã học [RAG](../rag/index.md), [Agents](../agents/index.md), [Evaluation](../evaluation/index.md), [LLMOps](../llmops/index.md). Có thể đọc song song với [Thiết kế sản phẩm & UX](../thiet-ke-san-pham/index.md).

## Khung thiết kế 7 bước

Mọi case đi theo cùng một khung. Dùng khung này cho dự án thật và cho phỏng vấn.

### Bước 1: Làm rõ yêu cầu

| Loại | Câu hỏi |
|---|---|
| **Chức năng** | Ai dùng? Làm được gì? Mức độ tự động (gợi ý, bản nháp, tự động)? |
| **Quy mô** | Bao nhiêu người dùng, tenant, request mỗi ngày? Cao điểm gấp bao nhiêu lần trung bình? |
| **Chất lượng** | Sai ở mức nào thì chấp nhận được? Loại sai nào **không** chấp nhận được? |
| **Độ trễ** | Người dùng đang chờ (tương tác) hay không (tác vụ nền)? Mục tiêu p95? |
| **Chi phí** | Ngân sách? Mô hình kinh doanh (miễn phí, thuê bao, tính theo lượt)? |
| **Ràng buộc** | Dữ liệu nhạy cảm? Giới hạn API của nền tảng? Ngôn ngữ? |

### Bước 2: Ước lượng quy mô

Tính nhanh bằng số tròn (back-of-the-envelope): số request, token mỗi request, chi phí mỗi tháng, request đồng thời lúc cao điểm. Mục đích không phải chính xác, mà là phát hiện sớm những thứ **sai vài bậc độ lớn** (ví dụ chi phí AI vượt giá bán).

### Bước 3: Kiến trúc tổng thể

Sơ đồ các thành phần và luồng dữ liệu. Phân biệt rõ **đường nóng** (người dùng đang chờ) và **đường nền** (xử lý offline, batch).

### Bước 4: Dữ liệu và đa tenant

Dữ liệu nào, lưu ở đâu, cập nhật thế nào, cách ly giữa các khách hàng ra sao.

### Bước 5: Chất lượng và eval

Tiêu chí, dataset, cách chấm, chỉ số online. Liên kết với đặc tả hành vi ([Thiết kế sản phẩm, Bài 1](../thiet-ke-san-pham/01-tu-duy-san-pham.md#4-ac-ta-tinh-nang-ai-bang-vi-du)).

### Bước 6: Chi phí và hiệu năng

Model cho từng bước, caching, batch, các đòn bẩy tối ưu ([LLMOps, Bài 3](../llmops/03-chi-phi-do-tre.md)).

### Bước 7: Rủi ro, vận hành và lộ trình

Bảo mật, lạm dụng, sự cố của phụ thuộc bên ngoài; giám sát; ra mắt theo giai đoạn.

## Bảng số liệu tham khảo

Dùng cho các phép ước lượng trong track. **Đây là giá trị tại thời điểm viết, chỉ dùng để luyện tập**; luôn kiểm tra giá và giới hạn hiện hành.

| Hạng mục | Giá trị dùng để ước lượng |
|---|---|
| Claude Opus 5 | 5 USD / 25 USD mỗi triệu token input / output |
| Claude Sonnet 5 | 2 USD / 10 USD |
| Claude Haiku 4.5 | 1 USD / 5 USD |
| Đọc từ prompt cache | Khoảng 10% giá input; chỉ áp dụng khi phần đầu prompt dài hơn ngưỡng tối thiểu của model (từ vài trăm tới vài nghìn token) |
| Batch API | Giảm 50% |
| Tiếng Việt | Khoảng 1 token cho mỗi 2 đến 3 ký tự (đo lại bằng `count_tokens` với dữ liệu của bạn) |
| Tốc độ sinh | Vài chục tới hơn một trăm token mỗi giây, tùy model và effort |
| Token suy nghĩ | Tính như output; ở effort thấp thường ít, ở effort cao có thể nhiều hơn phần trả lời |

!!! tip "Cách luyện tập hiệu quả"
    Đọc phần **yêu cầu** của mỗi case, **dừng lại**, tự thiết kế trong 45 đến 60 phút (viết ra giấy: ước lượng, sơ đồ, quyết định chính), rồi mới đọc phần thiết kế tham khảo. So sánh và ghi lại những gì bạn bỏ sót. Thiết kế tham khảo **không phải đáp án duy nhất**: nhiều quyết định có phương án khác hợp lý, miễn là bạn giải thích được đánh đổi.

---

[Lộ trình AI Engineer](../index.md)
