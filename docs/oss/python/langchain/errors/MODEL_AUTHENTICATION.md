# MODEL_AUTHENTICATION

!!! note "Ghi chú"
    Hiện chỉ dùng trong `langchainjs` (JavaScript/TypeScript).

Model provider của bạn đang từ chối cấp quyền truy cập vào dịch vụ của họ.

Lỗi này thường xảy ra khi có vấn đề với thông tin xác thực (authentication credentials) hoặc API key của bạn.

## Khắc phục sự cố

* Xác nhận API key hoặc thông tin xác thực của bạn chính xác và còn hợp lệ.
* Nếu dùng xác thực qua biến môi trường (environment-based), hãy kiểm tra:
    * Tên biến được viết đúng chính tả
    * Biến có chứa giá trị đã gán
    * Các package bên thứ ba như `dotenv` không cản trở việc nạp biến
* Nếu dùng proxy hoặc endpoint không chuẩn, đảm bảo provider tuỳ chỉnh của bạn không yêu cầu một scheme xác thực khác.
* Bỏ qua vấn đề biến môi trường bằng cách truyền credential một cách tường minh:

```python
from langchain_openai import ChatOpenAI

model = ChatOpenAI(api_key="YOUR_KEY_HERE")
```

---

!!! info "Kết nối tài liệu này"
    [Kết nối tài liệu này](https://docs.langchain.com/use-these-docs) với Claude, VSCode, và nhiều công cụ khác qua MCP để có câu trả lời theo thời gian thực.

!!! info "Đóng góp"
    [Chỉnh sửa trang này trên GitHub](https://github.com/langchain-ai/docs/edit/main/src/oss/langchain/errors/MODEL_AUTHENTICATION.mdx) hoặc [báo lỗi](https://github.com/langchain-ai/docs/issues/new/choose).
