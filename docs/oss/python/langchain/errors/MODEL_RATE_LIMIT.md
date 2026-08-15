# MODEL_RATE_LIMIT

!!! note "Ghi chú"
    Hiện tại lỗi này chỉ được dùng trong `langchainjs` (JavaScript/TypeScript).

Bạn đã vượt quá số lượng request tối đa mà nhà cung cấp model (model provider) cho phép trong một khoảng thời gian nhất định và đang tạm thời bị chặn.

Lỗi này xảy ra khi bạn gửi vượt quá số lượng request tối đa mà nhà cung cấp model của bạn cho phép trong một khung thời gian cụ thể, dẫn đến việc bị chặn tạm thời. Giới hạn này thường chỉ mang tính tạm thời và sẽ được gỡ bỏ sau khi giới hạn được reset.

## Khắc phục sự cố

Để khắc phục lỗi này, bạn có thể:

1. **Triển khai Rate Limiting**: Sử dụng một rate limiter để điều tiết tần suất request gửi tới model.
   Xem tài liệu [rate limiting](../models.md#rate-limiting).

2. **Triển khai Response Caching**: Sử dụng cơ chế cache response của model để giảm số request trùng lặp khi các câu truy vấn đầu vào lặp lại.

3. **Dùng nhiều nhà cung cấp (Provider)**: Phân phối request qua nhiều provider khác nhau nếu kiến trúc ứng dụng của bạn hỗ trợ cách tiếp cận này.

4. **Liên hệ nhà cung cấp**: Liên hệ với nhà cung cấp model của bạn để yêu cầu tăng giới hạn rate limit.
