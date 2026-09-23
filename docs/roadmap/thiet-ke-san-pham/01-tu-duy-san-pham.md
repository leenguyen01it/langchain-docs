# Bài 1: Tư duy sản phẩm AI

!!! abstract "Tóm tắt bài học"
    - **Mục tiêu:** chọn đúng bài toán đáng giải bằng AI, quyết định mức độ tự động hóa phù hợp với chi phí của sai lầm, định nghĩa chỉ số thành công, đặc tả hành vi của tính năng AI bằng ví dụ, và ước lượng dự án khi kết quả không chắc chắn.
    - **Thời lượng:** 4 đến 5 ngày.
    - **Yêu cầu trước:** [Agents, Bài 1](../agents/01-agent-loop.md).

## 1. Bắt đầu từ vấn đề, không từ công nghệ

Câu hỏi sai: "Ta có thể thêm AI vào app ở đâu?". Câu hỏi đúng: "**Người dùng đang mất thời gian, tiền bạc hoặc cơ hội ở đâu**, và AI có phải cách tốt nhất để giải quyết không?"

Với chủ cửa hàng Shopify, ví dụ các vấn đề thật:

| Vấn đề | Tần suất | Mức đau | AI có hợp không? |
|---|---|---|---|
| Viết mô tả cho hàng trăm sản phẩm mới mỗi mùa | Cao | Cao (tốn nhiều giờ, hay bị bỏ qua) | **Rất hợp**: sinh văn bản, người duyệt nhanh |
| Trả lời khách hỏi đi hỏi lại về size, phí ship, đổi trả | Rất cao | Trung bình | **Hợp**: RAG trên chính sách, chuyển người khi cần |
| Đọc hàng nghìn đánh giá để biết sản phẩm có vấn đề gì | Trung bình | Trung bình | **Hợp**: phân loại, tóm tắt hàng loạt |
| Tính thuế, đối soát tiền | Cao | Cao | **Không hợp**: cần chính xác tuyệt đối, đã có quy tắc rõ ràng, dùng code |
| Quyết định giá bán | Thấp | Cao | **Chỉ hỗ trợ**: gợi ý kèm lý do, người quyết định |

### Khi nào AI là lựa chọn tốt?

- Tác vụ liên quan tới **ngôn ngữ, hình ảnh, dữ liệu không cấu trúc**, khó viết thành quy tắc.
- **Chấp nhận được** một tỉ lệ sai nhỏ, hoặc có cách phát hiện và sửa sai dễ dàng.
- Người dùng **kiểm tra kết quả nhanh hơn nhiều** so với tự làm (đọc duyệt một mô tả mất 20 giây, tự viết mất 10 phút).

### Khi nào không nên dùng AI?

- Có quy tắc rõ ràng, code giải quyết được chính xác.
- Sai một lần gây hậu quả nghiêm trọng và khó phát hiện.
- Khối lượng quá nhỏ, người dùng tự làm còn nhanh hơn đọc lại kết quả của AI.

## 2. Mức độ tự động hóa

Không phải tính năng AI nào cũng nên tự động hoàn toàn. Chọn mức theo **chi phí của sai lầm** và **mức độ tin cậy đã đo được**:

| Mức | Mô tả | Ví dụ | Khi nào |
|---|---|---|---|
| **1. Gợi ý** | AI đưa gợi ý, người tự làm | Gợi ý từ khóa SEO | Mới ra mắt, chưa đo được độ tin cậy |
| **2. Bản nháp** | AI làm, người **duyệt và sửa** trước khi dùng | Sinh mô tả sản phẩm, người bấm "Đăng" | Sai có hậu quả, nhưng duyệt nhanh |
| **3. Tự động có giám sát** | AI tự làm, người xem lại theo mẫu, có thể hoàn tác | Tự trả lời câu hỏi FAQ của khách, nhân viên xem log | Đã có số liệu độ tin cậy cao, sai dễ sửa |
| **4. Tự động hoàn toàn** | AI tự làm, không ai xem | Gắn nhãn phân loại đánh giá nội bộ | Sai gần như vô hại |

!!! tip "Bắt đầu thấp, nâng dần bằng số liệu"
    Ra mắt ở mức 2 (bản nháp), thu thập dữ liệu: người dùng sửa bao nhiêu phần trăm bản nháp, sửa nhiều hay ít. Khi tỉ lệ "đăng mà không sửa" đủ cao và ổn định, mới cân nhắc nâng lên mức 3 cho các trường hợp dễ. Chính người dùng sẽ cho bạn dữ liệu để quyết định.

## 3. Chỉ số thành công

Một tính năng AI cần ba lớp chỉ số:

| Lớp | Ví dụ với tính năng sinh mô tả sản phẩm |
|---|---|
| **Kinh doanh** (vì sao làm) | Thời gian merchant tiết kiệm; tỉ lệ sản phẩm có mô tả; tỉ lệ chuyển đổi của trang sản phẩm; tỉ lệ merchant giữ gói trả phí |
| **Sản phẩm** (người dùng có dùng không) | Tỉ lệ merchant dùng tính năng; tỉ lệ bản nháp được đăng; tỉ lệ đăng mà không sửa; mức độ sửa (số ký tự thay đổi) |
| **Mô hình** (AI làm tốt không) | Điểm eval: đúng thông tin sản phẩm, đúng giọng thương hiệu, không bịa; độ trễ; chi phí mỗi mô tả |

Sai lầm phổ biến là chỉ đo lớp dưới cùng. Điểm eval 95% vô nghĩa nếu merchant không dùng tính năng. Ngược lại, chỉ số kinh doanh tốt mà không có eval thì bạn không biết cải tiến ở đâu.

**Tín hiệu từ hành vi** (người dùng sửa gì, bỏ qua gì, bấm tạo lại bao nhiêu lần) thường trung thực và dồi dào hơn nhiều so với nút 👍/👎.

## 4. Đặc tả tính năng AI bằng ví dụ

Đặc tả phần mềm truyền thống mô tả "khi X thì làm Y". Với AI, cách đặc tả hiệu quả nhất là **tập ví dụ**: đầu vào tiêu biểu, đầu ra mong muốn, và đầu ra **không** chấp nhận được. Tập ví dụ này đồng thời là **bộ eval đầu tiên** của bạn.

```markdown title="spec-mo-ta-san-pham.md"
## Tính năng: Sinh mô tả sản phẩm

### Mục tiêu
Merchant có mô tả sản phẩm chất lượng trong dưới 1 phút thay vì 10 phút.

### Mức tự động: 2 (bản nháp, merchant duyệt trước khi đăng)

### Hành vi bắt buộc
- Chỉ dùng thông tin có trong dữ liệu sản phẩm (tên, thuộc tính, ghi chú của merchant).
- Theo giọng văn thương hiệu merchant đã cấu hình.
- 80 đến 150 chữ, có một đoạn mở và 3 đến 5 gạch đầu dòng lợi ích.

### Không chấp nhận
- Bịa chất liệu, xuất xứ, chứng nhận, cam kết bảo hành.
- Nhắc tên thương hiệu đối thủ.
- Tuyên bố y tế, sức khỏe không có căn cứ ("chữa", "trị", "giảm cân").

### Ví dụ
| Đầu vào | Đầu ra tốt | Đầu ra không đạt | Vì sao |
|---|---|---|---|
| Áo linen, không ghi xuất xứ | "...chất linen thoáng mát..." | "...linen nhập khẩu từ Pháp..." | Bịa xuất xứ |
| Kem dưỡng, ghi chú "dịu nhẹ" | "...kết cấu mỏng nhẹ..." | "...điều trị mụn hiệu quả..." | Tuyên bố y tế |

### Khi không đủ thông tin
Sinh mô tả ngắn hơn, kèm ghi chú cho merchant: "Thêm chất liệu để có mô tả tốt hơn".
```

Viết đặc tả dạng này **cùng với** người làm sản phẩm và người hiểu khách hàng, trước khi viết dòng code nào. Nó buộc cả nhóm thống nhất "tốt" nghĩa là gì.

## 5. Kiểm chứng trước khi xây

- **Thử bằng tay:** dùng giao diện chat của model, dán dữ liệu thật của vài merchant, xem kết quả có dùng được không. Mất một buổi chiều, tránh được hàng tuần xây nhầm.
- **Wizard of Oz:** với tính năng phức tạp, cho một nhóm nhỏ người dùng dùng thử trong khi **con người** (hoặc bạn với sự trợ giúp của LLM) đứng sau xử lý. Nếu người dùng không thấy giá trị ngay cả khi kết quả hoàn hảo, đừng xây.
- **Kiểm tra kinh tế:** chi phí AI mỗi lần dùng × số lần dùng dự kiến so với giá người dùng sẵn sàng trả. Xem các phép tính mẫu ở [Case study, Bài 1](../case-study/01-sinh-mo-ta-san-pham.md#2-uoc-luong-quy-mo-va-chi-phi).

## 6. Ước lượng dự án AI

Dự án AI khó ước lượng hơn phần mềm thông thường vì **không biết trước chất lượng đạt được**. Cách tiếp cận:

1. **Chia giai đoạn có điểm quyết định:**
   - Thăm dò (1 đến 2 tuần): prototype, bộ eval nhỏ, trả lời câu hỏi "có khả thi không, đạt khoảng bao nhiêu phần trăm".
   - Xây MVP (vài tuần): tính năng tối thiểu ở mức tự động thấp, ra mắt cho nhóm nhỏ.
   - Cải tiến (liên tục): vòng lặp eval, dữ liệu thật, nâng chất lượng.
2. **Cam kết theo thời gian và mục tiêu chất lượng, không cam kết tính năng hoàn hảo:** "Sau 2 tuần thăm dò, ta sẽ biết tính năng đạt khoảng 80% hay 95% và quyết định tiếp."
3. **Phần kỹ thuật "thường" chiếm phần lớn thời gian:** tích hợp, giao diện, dữ liệu, vận hành. Phần "gọi LLM" thường chỉ là một phần nhỏ.
4. **Luôn chừa thời gian cho eval và vòng sửa lỗi** sau khi có dữ liệu thật.

## Bài tập

**Bài 1.1.** Liệt kê 10 vấn đề của người dùng sản phẩm bạn đang làm (hoặc Shopify app của bạn). Chấm tần suất, mức đau, độ phù hợp với AI. Chọn 2 vấn đề đáng làm nhất.

**Bài 1.2.** Với vấn đề số 1, chọn mức độ tự động hóa và giải thích. Viết ba lớp chỉ số thành công.

**Bài 1.3.** Viết đặc tả theo mẫu ở mục 4, với ít nhất 10 ví dụ, trong đó có ít nhất 4 ví dụ "không đạt". Chuyển các ví dụ này thành bộ eval đầu tiên.

**Bài 1.4.** Thử bằng tay tính năng với dữ liệu thật của 3 người dùng (hoặc dữ liệu giả lập sát thực tế). Ghi lại: kết quả dùng được bao nhiêu phần trăm, phải sửa những gì.

## Checklist

- [ ] Chọn bài toán từ vấn đề của người dùng, có lý do vì sao AI phù hợp hơn cách khác.
- [ ] Chọn mức tự động hóa theo chi phí của sai lầm, có kế hoạch nâng mức dựa trên số liệu.
- [ ] Có chỉ số ở cả ba lớp: kinh doanh, sản phẩm, mô hình.
- [ ] Đặc tả tính năng bằng ví dụ, dùng luôn làm bộ eval.
- [ ] Kiểm chứng giá trị và tính kinh tế trước khi xây.

## Đọc thêm

- [Evaluation, Bài 1](../evaluation/01-tu-duy-eval.md): biến ví dụ thành eval.
- [People + AI Guidebook](https://pair.withgoogle.com/guidebook/) (Google PAIR).

---

[Tổng quan track](index.md) · **Bài tiếp theo:** [Bài 2: UX cho sản phẩm AI](02-ux-cho-ai.md)
