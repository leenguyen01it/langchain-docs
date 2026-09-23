# Bài 4: Trách nhiệm và giao tiếp

!!! abstract "Tóm tắt bài học"
    - **Mục tiêu:** vận hành tính năng AI có trách nhiệm (minh bạch, quyền riêng tư, nội dung, sở hữu trí tuệ, giám sát của con người), viết tài liệu thiết kế cho tính năng AI, và trình bày kết quả eval, rủi ro, giới hạn cho người không chuyên.
    - **Thời lượng:** 3 đến 4 ngày.
    - **Yêu cầu trước:** [Bài 1](01-tu-duy-san-pham.md), [LLMOps, Bài 4](../llmops/04-bao-mat.md).

!!! warning "Đây không phải tư vấn pháp lý"
    Bài này nêu các vấn đề cần lưu ý khi làm sản phẩm AI. Quy định pháp luật (bảo vệ dữ liệu cá nhân, quảng cáo, bảo vệ người tiêu dùng, sở hữu trí tuệ) khác nhau theo quốc gia và thay đổi theo thời gian. Với sản phẩm thật, hãy tham khảo chuyên gia pháp lý và đọc kỹ điều khoản của nhà cung cấp model và nền tảng bạn phát hành ứng dụng.

## Phần A: Trách nhiệm

## 1. Minh bạch

- **Người dùng cuối cần biết đang nói chuyện với AI.** Chatbot trên cửa hàng nên giới thiệu là trợ lý tự động, và luôn có cách gặp người thật.
- **Nội dung do AI tạo nên được đánh dấu** trong giao diện quản trị (merchant biết mô tả nào do AI viết, đã duyệt hay chưa).
- **Nói rõ giới hạn** tại chỗ dùng ([Bài 2](02-ux-cho-ai.md#21-at-ky-vong-ung)): AI có thể sai, cần kiểm tra trước khi đăng.

## 2. Quyền riêng tư và dữ liệu

| Nguyên tắc | Áp dụng |
|---|---|
| **Tối thiểu hóa** | Chỉ gửi cho LLM dữ liệu cần thiết. Viết mô tả sản phẩm không cần thông tin khách hàng. Trả lời về đơn hàng chỉ cần đơn của người đang hỏi |
| **Mục đích rõ ràng** | Dữ liệu thu thập để phục vụ một tính năng không tự động được dùng cho mục đích khác (ví dụ huấn luyện model) nếu chưa được đồng ý |
| **Biết dữ liệu đi đâu** | Nắm chính sách của nhà cung cấp LLM: dữ liệu có được lưu không, bao lâu, xử lý ở khu vực nào, có dùng để huấn luyện không |
| **Lưu trữ có thời hạn** | Log, trace, lịch sử hội thoại có chính sách xóa; người dùng yêu cầu xóa thì xóa được ở mọi nơi (kể cả vector DB, cache, trace) |
| **Cách ly giữa khách hàng** | Dữ liệu của cửa hàng A không bao giờ xuất hiện trong câu trả lời cho cửa hàng B ([LLMOps, Bài 4](../llmops/04-bao-mat.md#5-du-lieu-nhay-cam)) |

Với ứng dụng phục vụ khách hàng ở Việt Nam, cần tìm hiểu các quy định về bảo vệ dữ liệu cá nhân hiện hành. Ứng dụng phục vụ merchant ở nhiều quốc gia (như Shopify app) còn phải tính tới quy định của các thị trường đó và yêu cầu về dữ liệu của nền tảng phân phối ứng dụng.

## 3. Nội dung và trách nhiệm pháp lý

Nội dung do AI tạo **được đăng dưới tên của merchant**, và merchant chịu trách nhiệm với khách hàng của họ. Tính năng của bạn cần giúp merchant tránh rủi ro:

| Rủi ro | Ví dụ | Biện pháp |
|---|---|---|
| **Quảng cáo sai sự thật** | Mô tả bịa chứng nhận, xuất xứ, công dụng | Chỉ dùng dữ liệu có sẵn; giám khảo kiểm tra thông tin không có căn cứ; merchant duyệt trước khi đăng |
| **Tuyên bố nhạy cảm** | Mỹ phẩm "trị mụn", thực phẩm "chữa bệnh" | Danh sách từ ngữ, loại tuyên bố bị cấm theo ngành hàng; cảnh báo cho merchant |
| **Cam kết thay merchant** | Chatbot hứa hoàn tiền, tặng quà | Chatbot không được cam kết ngoài chính sách đã cấu hình ([Evaluation, Bài 2](../evaluation/02-llm-judge.md#22-vi-du-giam-khao-bia-cam-ket)) |
| **Sở hữu trí tuệ** | Sao chép mô tả của đối thủ, dùng tên thương hiệu khác | Không đưa nội dung của bên khác vào prompt làm mẫu để sao chép; kiểm tra tên thương hiệu |
| **Nội dung có hại, phân biệt đối xử** | Gợi ý sản phẩm hoặc từ ngữ mang định kiến giới, vùng miền | Eval có ví dụ về định kiến; red teaming ([Evaluation, Bài 4](../evaluation/04-online-eval.md#7-red-teaming-tu-tan-cong-he-thong-cua-minh)) |

## 4. Giám sát của con người và trách nhiệm giải trình

- **Ai chịu trách nhiệm** khi AI sai? Xác định rõ trong nhóm: người sở hữu tính năng, người trực sự cố.
- **Con người trong vòng lặp** ở những chỗ hậu quả lớn: duyệt nội dung trước khi công khai, duyệt hành động tài chính.
- **Truy vết được:** mỗi nội dung, mỗi hành động của AI gắn với trace (model, phiên bản prompt, dữ liệu đầu vào), để khi có khiếu nại, bạn giải thích được chuyện gì đã xảy ra.
- **Có quy trình xử lý sự cố** và công tắc tắt khẩn cấp ([LLMOps, Bài 5](../llmops/05-trien-khai.md#6-xu-ly-su-co)).

## Phần B: Giao tiếp

## 5. Tài liệu thiết kế cho tính năng AI

Trước khi xây một tính năng AI đáng kể, viết một tài liệu thiết kế ngắn (2 đến 4 trang) để cả nhóm thống nhất. Mẫu:

```markdown title="design-doc-template.md"
# [Tên tính năng]

## 1. Vấn đề và mục tiêu
Người dùng nào, gặp vấn đề gì, đo bằng chỉ số nào (kinh doanh, sản phẩm, mô hình).

## 2. Phạm vi
Làm gì, KHÔNG làm gì. Mức độ tự động hóa và lý do.

## 3. Hành vi mong muốn
Đặc tả bằng ví dụ: tốt, không đạt, trường hợp biên (xem Bài 1).

## 4. Thiết kế giải pháp
Kiến trúc (sơ đồ), luồng dữ liệu, model và lý do chọn, prompt chính, tool.
Các phương án đã cân nhắc và vì sao không chọn.

## 5. Kế hoạch đánh giá
Dataset, tiêu chí, cách chấm, ngưỡng để ra mắt.

## 6. Chi phí và hiệu năng
Ước tính token, chi phí mỗi lần dùng và mỗi tháng, độ trễ mục tiêu.

## 7. Rủi ro và giảm thiểu
Chất lượng, bảo mật, quyền riêng tư, pháp lý, chi phí vượt dự kiến.

## 8. Kế hoạch ra mắt
Giai đoạn, nhóm người dùng thử, chỉ số theo dõi, điều kiện quay lại.

## 9. Câu hỏi còn mở
```

Các [case study](../case-study/index.md) ở track tiếp theo được viết theo cấu trúc gần giống mẫu này.

## 6. Trình bày kết quả eval cho người không chuyên

Quản lý, merchant, đội kinh doanh không cần biết "faithfulness 0,93". Họ cần biết: **dùng được không, sai thế nào, rủi ro ra sao, cần quyết định gì**.

| Cách nói kỹ thuật | Cách nói dễ hiểu |
|---|---|
| "Accuracy 94%, TPR của giám khảo 0,91" | "Trong 100 mô tả thử nghiệm, 94 cái dùng được ngay, 5 cái cần sửa vài chữ, 1 cái có lỗi nghiêm trọng" |
| "Hallucination rate giảm từ 6% xuống 1,5%" | "Trước đây cứ khoảng 17 mô tả có một cái bịa thông tin, giờ khoảng 70 cái mới có một. Merchant vẫn duyệt trước khi đăng nên lỗi không tới tay khách" |
| "p95 latency 4,2 giây" | "Hầu hết mô tả hiện ra trong 1 đến 2 giây, chậm nhất khoảng 4 giây" |

Nguyên tắc:

- **Dùng số đếm cụ thể** ("5 trên 100") thay vì phần trăm trừu tượng khi có thể.
- **Cho xem ví dụ thật:** một ví dụ tốt, một ví dụ sai điển hình. Người nghe hiểu ngay chất lượng ở mức nào.
- **Nói rõ lỗi nghiêm trọng nhất** và biện pháp đang có, đừng giấu.
- **So sánh với phương án hiện tại** (người tự làm mất bao lâu, sai bao nhiêu), không so với sự hoàn hảo.
- **Kết thúc bằng quyết định cần đưa ra:** "Đề xuất ra mắt cho 10% merchant ở chế độ bản nháp. Cần đồng ý: ngân sách khoảng X đô la mỗi tháng."

## 7. Giải thích giới hạn của AI cho người dùng

- Trong tài liệu hướng dẫn và màn hình giới thiệu tính năng: AI làm tốt gì, cần kiểm tra gì, dữ liệu được xử lý thế nào.
- Khi người dùng báo lỗi: cảm ơn, ghi nhận vào hàng đợi review, và nếu sửa được thì báo lại. Người dùng báo lỗi là nguồn dữ liệu eval quý nhất.
- Không hứa quá mức trong marketing ("AI viết mô tả hoàn hảo"). Kỳ vọng quá cao làm người dùng thất vọng ở lỗi đầu tiên.

## Bài tập

**Bài 4.1.** Rà soát một tính năng AI bạn đã làm theo mục 1 đến 4. Liệt kê rủi ro chưa được xử lý và biện pháp.

**Bài 4.2.** Viết tài liệu thiết kế theo mẫu ở mục 5 cho tính năng bạn chọn ở [Bài 1](01-tu-duy-san-pham.md).

**Bài 4.3.** Lấy kết quả eval của một dự án bạn đã làm (ví dụ ở track RAG hoặc Evaluation), viết một bản tóm tắt một trang cho quản lý không chuyên kỹ thuật, theo nguyên tắc ở mục 6. Nhờ một người không làm kỹ thuật đọc và hỏi lại những chỗ họ không hiểu.

## Checklist

- [ ] Người dùng cuối biết khi nào họ tương tác với AI; nội dung AI được đánh dấu.
- [ ] Dữ liệu gửi cho LLM được tối thiểu hóa; biết rõ chính sách dữ liệu của nhà cung cấp; có chính sách lưu trữ và xóa.
- [ ] Có biện pháp chống nội dung sai sự thật, tuyên bố nhạy cảm, cam kết ngoài chính sách.
- [ ] Xác định rõ người chịu trách nhiệm, mọi output AI truy vết được.
- [ ] Viết được tài liệu thiết kế và báo cáo eval dễ hiểu cho người không chuyên.

## Đọc thêm

- [Guardrails](../../oss/python/langchain/guardrails.md), [Human-in-the-loop](../../oss/python/langchain/human-in-the-loop.md)
- [LLMOps, Bài 4: Bảo mật](../llmops/04-bao-mat.md)

---

**Bài trước:** [Bài 3: Generative UI](03-generative-ui.md) · [Tổng quan track](index.md) · **Track tiếp theo:** [Case study thiết kế hệ thống](../case-study/index.md)
