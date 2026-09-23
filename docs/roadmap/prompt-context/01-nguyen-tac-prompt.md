# Bài 1: Nguyên tắc viết prompt

!!! abstract "Tóm tắt bài học"
    - **Mục tiêu:** nắm các nguyên tắc nền tảng giúp prompt cho kết quả tốt và ổn định: rõ ràng, có ngữ cảnh và lý do, có ví dụ, có cấu trúc, định rõ output; viết prompt tốt cho sản phẩm tiếng Việt.
    - **Thời lượng:** 4 đến 5 ngày.
    - **Yêu cầu trước:** [Hiểu LLM, Bài 3](../hieu-llm/03-claude-api.md).

## 1. Một ví dụ trước và sau

Tác vụ: viết mô tả sản phẩm cho cửa hàng thời trang online.

=== "Prompt yếu"

    ```text
    Viết mô tả sản phẩm cho áo sơ mi linen nam. Viết hay và hấp dẫn.
    ```

    Kết quả thường gặp: đoạn văn chung chung, sáo rỗng ("chất liệu cao cấp", "phong cách thời thượng"), độ dài tùy hứng, giọng văn không khớp thương hiệu, có thể bịa thông tin (thành phần vải, xuất xứ).

=== "Prompt tốt"

    ```text
    Bạn viết mô tả sản phẩm cho "Mộc", một thương hiệu thời trang nam tối giản,
    khách hàng là nhân viên văn phòng 25 đến 35 tuổi ở thành phố lớn.

    <brand_voice>
    Giọng văn: điềm đạm, chân thành, không phóng đại. Xưng "bạn" với khách hàng.
    Tránh các từ sáo rỗng như "đẳng cấp", "sang chảnh", "hot trend".
    </brand_voice>

    <product>
    Tên: Áo sơ mi linen cổ tàu
    Chất liệu: 100% linen
    Màu: be, trắng ngà, xanh rêu
    Form: regular fit
    Điểm nổi bật: thoáng mát, càng giặt càng mềm
    </product>

    Viết mô tả gồm:
    1. Một câu mở đầu nêu cảm giác khi mặc (tối đa 20 chữ).
    2. Ba gạch đầu dòng về lợi ích, mỗi dòng gắn với một đặc điểm trong <product>.
    3. Một câu gợi ý phối đồ.

    Chỉ dùng thông tin có trong <product>. Không tự thêm xuất xứ, giá hay cam kết
    nào khác, vì mô tả này được đăng công khai và phải đúng sự thật.
    ```

Prompt tốt dài hơn nhiều, và đó là điều bình thường. Nó trả lời những câu hỏi mà một người viết mới vào nghề sẽ cần hỏi: viết cho ai, giọng văn nào, dựa trên thông tin gì, định dạng ra sao, giới hạn ở đâu và **vì sao**.

## 2. Các nguyên tắc cốt lõi

### 2.1 Rõ ràng và cụ thể

Model không đọc được suy nghĩ của bạn. Hãy nói rõ:

- **Mục tiêu:** kết quả dùng để làm gì, cho ai đọc.
- **Tiêu chí thành công:** thế nào là một câu trả lời tốt.
- **Ràng buộc:** độ dài, định dạng, những gì không được làm.

Phép thử đơn giản: đưa prompt cho một đồng nghiệp **không biết gì về dự án**. Nếu họ phải hỏi lại, model cũng sẽ phải đoán.

### 2.2 Giải thích lý do

Nói **vì sao** giúp model tổng quát hóa đúng thay vì làm theo máy móc.

| Chỉ ra lệnh | Kèm lý do |
|---|---|
| "Không dùng emoji." | "Không dùng emoji, vì câu trả lời sẽ được đọc bằng giọng nói tổng hợp trên tổng đài." |
| "Trả lời ngắn." | "Trả lời trong 2 đến 3 câu, vì khách hàng đọc trên màn hình điện thoại trong khung chat nhỏ." |

Với lý do "đọc bằng giọng nói", model còn tự hiểu nên tránh cả ký hiệu đặc biệt, bảng biểu, đường dẫn, những điều bạn chưa kịp liệt kê.

### 2.3 Cho ví dụ (few-shot)

Ví dụ là cách mạnh nhất để truyền đạt định dạng và văn phong. Nguyên tắc:

- **3 đến 5 ví dụ** thường là đủ.
- **Đa dạng**: bao phủ các trường hợp khác nhau, kể cả trường hợp khó. Nếu mọi ví dụ đều giống nhau, model sẽ bắt chước cả những đặc điểm ngẫu nhiên.
- **Đặt trong thẻ** `<example>` để tách khỏi chỉ dẫn.

```text
Phân loại yêu cầu hỗ trợ vào một trong các nhóm: doi_tra, van_chuyen, thanh_toan, khac.

<examples>
<example>
<input>Áo bị rách chỉ ở vai, shop đổi giúp mình nhé</input>
<output>doi_tra</output>
</example>
<example>
<input>Mình chuyển khoản rồi mà đơn vẫn báo chưa thanh toán</input>
<output>thanh_toan</output>
</example>
<example>
<input>Shop có bán thẻ quà tặng không?</input>
<output>khac</output>
</example>
</examples>
```

### 2.4 Cấu trúc bằng thẻ XML

Khi prompt có nhiều phần (chỉ dẫn, tài liệu, ví dụ, dữ liệu người dùng), bọc mỗi phần trong một thẻ có tên rõ nghĩa: `<instructions>`, `<document>`, `<examples>`, `<customer_message>`. Lợi ích:

- Model phân biệt rõ đâu là chỉ dẫn, đâu là dữ liệu. Điều này cũng giúp **giảm rủi ro prompt injection** từ dữ liệu người dùng.
- Dễ tham chiếu: "Chỉ dùng thông tin trong `<product>`".
- Dễ tách phần output bằng code nếu yêu cầu model trả lời trong thẻ.

Không có tên thẻ "đặc biệt" nào; hãy đặt tên có nghĩa và dùng nhất quán.

### 2.5 Định rõ output

- Nói rõ **định dạng**: đoạn văn, gạch đầu dòng, JSON, bảng.
- Khi code cần đọc output, dùng **structured output** (xem [Hiểu LLM, Bài 3](../hieu-llm/03-claude-api.md#6-structured-output)) thay vì chỉ dặn "trả về JSON".
- Mô tả điều **muốn có** thay vì chỉ liệt kê điều **không muốn**: "Viết thành các đoạn văn liền mạch" hiệu quả hơn "Không dùng gạch đầu dòng".
- Văn phong của prompt ảnh hưởng tới văn phong output: prompt viết bằng đoạn văn tự nhiên có xu hướng cho output tự nhiên; prompt đầy markdown có xu hướng cho output đầy markdown.

### 2.6 Vai trò trong system prompt

Đặt vai trò và bối cảnh trong system prompt: "Bạn là chuyên viên tư vấn của một cửa hàng mỹ phẩm, am hiểu về da nhạy cảm". Vai trò cụ thể giúp model chọn đúng kiến thức, giọng văn và mức độ chi tiết.

### 2.7 Tài liệu dài: đặt trước, câu hỏi đặt sau

Khi đưa tài liệu dài (hợp đồng, báo cáo, nhiều trang) vào prompt:

- Đặt **tài liệu ở đầu**, **câu hỏi và chỉ dẫn ở cuối**.
- Bọc từng tài liệu trong thẻ kèm metadata: `<document source="hop-dong-2026.pdf">`.
- Với câu hỏi cần độ chính xác cao, yêu cầu model **trích dẫn đoạn liên quan trước**, rồi mới trả lời dựa trên đoạn trích đó.

### 2.8 Đừng "hét" vào model

Prompt cho model đời cũ thường đầy chữ in hoa, "BẮT BUỘC", "TUYỆT ĐỐI KHÔNG", lặp lại một quy tắc nhiều lần. Model hiện đại làm theo chỉ dẫn tốt hơn nhiều; nhấn mạnh quá mức khiến model **áp dụng quy tắc cả khi không nên** (ví dụ từ chối cả những yêu cầu hợp lệ). Hãy viết bình thường, giải thích lý do, và chỉ nhấn mạnh khi eval cho thấy model thực sự bỏ qua quy tắc đó.

### 2.9 Cho model không gian để suy nghĩ

Với tác vụ phức tạp (phân tích, suy luận nhiều bước), bật adaptive thinking (xem [Hiểu LLM, Bài 2](../hieu-llm/02-sampling-reasoning.md)). Với model không có chế độ này, yêu cầu model suy nghĩ trong thẻ `<thinking>` trước khi trả lời trong thẻ `<answer>`.

## 3. Viết prompt cho sản phẩm tiếng Việt

| Vấn đề | Cách xử lý trong prompt |
|---|---|
| **Xưng hô** | Nói rõ: "Xưng 'em', gọi khách là 'anh/chị'" hoặc "Xưng 'mình', gọi khách là 'bạn'". Không nói, model có thể đổi cách xưng hô giữa chừng |
| **Văn phong dịch** | Model đôi khi viết câu mang cấu trúc tiếng Anh ("Điều này là bởi vì..."). Cho 2 đến 3 ví dụ văn phong mong muốn, yêu cầu "viết tự nhiên như người Việt nói" |
| **Vùng miền** | Nếu cần, nói rõ: dùng từ miền Nam ("dạ", "nha") hay miền Bắc ("vâng", "nhé") |
| **Thuật ngữ** | Quy định rõ giữ nguyên hay dịch: "Giữ nguyên các thuật ngữ: COD, voucher, flash sale" |
| **Tiếng lóng, viết tắt của khách** | Cho ví dụ đầu vào có viết tắt ("sp", "ib", "ship cod") để model quen |
| **Viết prompt bằng ngôn ngữ nào?** | Prompt tiếng Anh hay tiếng Việt đều hoạt động tốt với model hiện đại. Điều quan trọng là nói rõ **ngôn ngữ output**. Hãy thử cả hai và đo trên dữ liệu của bạn |

## 4. Giải phẫu một system prompt hoàn chỉnh

```text
Bạn là trợ lý chăm sóc khách hàng của Mộc, thương hiệu thời trang nam tối giản.
Khách hàng liên hệ qua khung chat trên website, chủ yếu hỏi về sản phẩm,
đơn hàng, đổi trả.

<goals>
Giải quyết thắc mắc của khách nhanh và chính xác. Khi không chắc chắn,
chuyển cho nhân viên thay vì đoán, vì một câu trả lời sai về chính sách
có thể gây mất tiền cho khách và cho cửa hàng.
</goals>

<style>
Xưng "Mộc", gọi khách là "bạn". Trả lời 2 đến 4 câu, thân thiện, không
dùng emoji. Khách đọc trên điện thoại nên tránh bảng và đoạn văn dài.
</style>

<policies>
{policies}
</policies>

<escalation>
Chuyển cho nhân viên (gọi tool transfer_to_human) khi: khách muốn hoàn tiền,
khách bức xúc hoặc dọa đánh giá xấu, hoặc câu hỏi không có trong <policies>.
</escalation>
```

Các thành phần: **vai trò và bối cảnh**, **mục tiêu kèm lý do**, **văn phong**, **kiến thức** (được chèn vào), **quy tắc xử lý tình huống đặc biệt**. Phần thay đổi theo từng khách (thông tin đơn hàng, câu hỏi) đặt trong tin nhắn `user`, để system prompt giữ nguyên và **cache được**.

## Bài tập

**Bài 1.1.** Lấy prompt "yếu" ở mục 1, chạy 3 lần. Chạy prompt "tốt" 3 lần. So sánh: độ ổn định, mức độ bịa thông tin, độ khớp giọng văn.

**Bài 1.2.** Viết prompt phân loại yêu cầu hỗ trợ (mục 2.3) **không có ví dụ** và **có 5 ví dụ đa dạng**. Chạy trên 30 tin nhắn thật hoặc tự viết (có viết tắt, sai chính tả). So sánh độ chính xác.

**Bài 1.3.** Viết lại prompt sau cho tốt hơn, áp dụng ít nhất 4 nguyên tắc trong bài: *"Tóm tắt email này. KHÔNG ĐƯỢC dài quá. Phải có action items."*

??? tip "Gợi ý lời giải"
    ```text
    Bạn tóm tắt email công việc cho một trưởng nhóm bận rộn, người đọc bản
    tóm tắt trên điện thoại giữa các cuộc họp.

    <email>
    {email}
    </email>

    Viết tóm tắt gồm:
    - Một câu nêu mục đích chính của email.
    - Danh sách việc cần làm (nếu có), mỗi việc ghi rõ ai làm và hạn chót nếu
      email có nhắc tới. Nếu email không có việc cần làm, ghi "Không có việc cần làm".

    Tổng cộng không quá 80 chữ, vì người đọc chỉ có vài giây để lướt.
    ```
    Các nguyên tắc đã áp dụng: người đọc và mục đích, lý do cho giới hạn độ dài, định dạng cụ thể, xử lý trường hợp không có việc cần làm, thẻ XML cho dữ liệu, không dùng chữ in hoa.

**Bài 1.4.** Viết system prompt hoàn chỉnh cho chatbot của một cửa hàng bạn biết (hoặc cửa hàng Shopify của bạn). Nhờ một người khác đóng vai khách hàng khó tính, chat 10 lượt, ghi lại những chỗ bot trả lời chưa tốt, sửa prompt, lặp lại.

## Checklist

- [ ] Prompt nêu rõ mục tiêu, người đọc, định dạng output và lý do cho các ràng buộc.
- [ ] Dùng ví dụ đa dạng khi cần truyền đạt định dạng hoặc văn phong.
- [ ] Dùng thẻ XML để tách chỉ dẫn và dữ liệu.
- [ ] Tài liệu dài đặt trước, câu hỏi đặt sau.
- [ ] Prompt tiếng Việt quy định rõ xưng hô và văn phong.
- [ ] Không lạm dụng chữ in hoa và các từ nhấn mạnh.

## Đọc thêm

- [Prompt engineering overview](https://docs.claude.com/en/docs/build-with-claude/prompt-engineering/overview) của Anthropic.
- [Context Engineering](../../oss/python/langchain/context-engineering.md): phần System Prompt.

---

[Tổng quan track](index.md) · **Bài tiếp theo:** [Bài 2: Kỹ thuật nâng cao](02-ky-thuat-nang-cao.md)
