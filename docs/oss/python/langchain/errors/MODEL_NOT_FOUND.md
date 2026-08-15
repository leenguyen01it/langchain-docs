# MODEL_NOT_FOUND

!!! note "Ghi chú"
    Hiện chỉ dùng trong `langchainjs` (JavaScript/TypeScript).

Tên model bạn chỉ định không được provider của bạn công nhận.

## Khắc phục sự cố

Để khắc phục lỗi này:

1. **Kiểm tra lại định danh model**: Kiểm tra kỹ chuỗi model bạn đang truyền vào. Đảm bảo chính tả và định dạng chính xác
2. **Kiểm tra cấu hình proxy/wrapper**: Nếu bạn đang dùng proxy hoặc host thay thế khác kèm model wrapper, xác nhận các tên model được phép không bị giới hạn hoặc thay đổi

Lỗi này thường bắt nguồn từ lỗi chính tả trong chuỗi tên model, hoặc do giới hạn áp đặt bởi dịch vụ proxy hoặc model wrapper nằm giữa code của bạn và API của provider.

---

!!! info "Kết nối tài liệu này"
    [Kết nối tài liệu này](https://docs.langchain.com/use-these-docs) với Claude, VSCode, và nhiều công cụ khác qua MCP để có câu trả lời theo thời gian thực.

!!! info "Đóng góp"
    [Chỉnh sửa trang này trên GitHub](https://github.com/langchain-ai/docs/edit/main/src/oss/langchain/errors/MODEL_NOT_FOUND.mdx) hoặc [báo lỗi](https://github.com/langchain-ai/docs/issues/new/choose).
