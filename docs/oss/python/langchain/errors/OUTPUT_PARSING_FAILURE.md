# OUTPUT_PARSING_FAILURE

Một [output parser](https://reference.langchain.com/python/langchain_core/output_parsers/) không thể xử lý output của model như mong đợi.

!!! note "Ghi chú"
    Một số cấu trúc dựng sẵn (prebuilt) như các agent và chain kiểu cũ (legacy) của LangChain có thể sử dụng output parser nội bộ, vì vậy bạn có thể gặp lỗi này ngay cả khi không trực tiếp khởi tạo và sử dụng output parser.

## Khắc phục sự cố

* Cân nhắc dùng tool calling hoặc các kỹ thuật structured output khác nếu có thể, thay vì dùng output parser, để đảm bảo output trả về luôn parse được một cách đáng tin cậy.
* Thêm hướng dẫn định dạng (formatting instructions) rõ ràng, chi tiết hơn vào prompt.
* Nếu bạn đang dùng một model nhỏ hoặc kém năng lực hơn, hãy thử dùng một model mạnh hơn.
