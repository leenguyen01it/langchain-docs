# Bài 2: UX cho sản phẩm AI

!!! abstract "Tóm tắt bài học"
    - **Mục tiêu:** thiết kế trải nghiệm cho tính năng có đầu ra không chắc chắn: đặt kỳ vọng đúng, xử lý chờ đợi, thể hiện nguồn và độ tin cậy, giúp người dùng phát hiện và sửa sai, thu thập tín hiệu để cải tiến.
    - **Thời lượng:** khoảng 1 tuần.
    - **Yêu cầu trước:** [Bài 1](01-tu-duy-san-pham.md), [Nền tảng kỹ thuật, Bài 2](../nen-tang-ky-thuat/02-fastapi-streaming.md) (streaming).

## 1. Vì sao UX cho AI khác biệt?

Phần mềm truyền thống **xác định**: bấm nút A luôn ra kết quả B. Tính năng AI thì:

- **Có lúc sai**, và sai theo cách khó đoán, đôi khi rất tự tin.
- **Chậm hơn** nhiều so với thao tác thông thường.
- **Mỗi lần một khác**: cùng yêu cầu, kết quả khác nhau.

Mục tiêu của UX cho AI không phải che giấu những điều này, mà giúp người dùng **tin đúng mức**: đủ tin để dùng, đủ nghi ngờ để kiểm tra khi cần.

## 2. Các mẫu thiết kế cốt lõi

### 2.1 Đặt kỳ vọng đúng

- Nói rõ AI **làm được gì và không làm được gì**, ngay tại chỗ dùng, không giấu trong trang trợ giúp. Ví dụ dưới ô chat: "Trợ lý trả lời về sản phẩm, đơn hàng, đổi trả. Với khiếu nại, bạn sẽ được chuyển tới nhân viên."
- **Gợi ý mẫu** (prompt starters) cho người dùng mới: vừa hướng dẫn cách dùng, vừa định hướng vào những việc AI làm tốt.
- Đánh dấu rõ nội dung **do AI tạo ra**.

### 2.2 Xử lý chờ đợi

| Thời gian | Cách xử lý |
|---|---|
| Dưới 1 giây | Không cần chỉ báo |
| 1 đến 10 giây | **Streaming** chữ ngay khi có; nếu không stream được, chỉ báo đang xử lý |
| 10 giây đến vài phút (agent) | Hiện **từng bước** đang làm: "Đang tra đơn hàng...", "Đang đọc chính sách đổi trả..." |
| Lâu hơn (xử lý hàng loạt) | Chạy nền, cho người dùng làm việc khác, **thông báo** khi xong, hiện tiến độ (đã xong 120/500 sản phẩm) |

Hiện các bước của agent còn một lợi ích nữa: người dùng **hiểu được** AI đã dựa vào đâu, tăng độ tin tưởng hợp lý.

### 2.3 Thể hiện nguồn và độ tin cậy

- **Trích dẫn nguồn** (RAG): mỗi ý kèm liên kết tới tài liệu gốc, người dùng bấm vào để kiểm tra ([RAG, Bài 5](../rag/05-generation.md)).
- **Hiện dữ liệu đã dùng:** "Dựa trên: tên, chất liệu, 3 ảnh sản phẩm". Người dùng biết ngay nếu AI thiếu thông tin quan trọng.
- **Cẩn thận với con số phần trăm độ tin cậy:** "Độ tin cậy 87%" thường gây hiểu nhầm, vì điểm do model tự báo không được hiệu chỉnh. Tốt hơn là **hành vi khác nhau theo độ chắc chắn**: trường hợp chắc chắn thì trình bày bình thường, trường hợp không chắc thì nói rõ và đề nghị kiểm tra hoặc chuyển người.

### 2.4 Thiết kế cho lúc AI sai

Đây là phần quan trọng nhất, và hay bị bỏ qua nhất:

| Mẫu | Ví dụ |
|---|---|
| **Dễ sửa** | Bản nháp mô tả sản phẩm mở trong ô soạn thảo, sửa trực tiếp được, không phải một khối chữ chỉ đọc |
| **Tạo lại, có định hướng** | Nút "Viết lại" kèm lựa chọn: "ngắn hơn", "trang trọng hơn", "nhấn mạnh chất liệu" |
| **Hoàn tác** | Mọi thay đổi AI áp dụng (sửa hàng loạt mô tả, gắn tag) đều hoàn tác được |
| **Xem trước khi áp dụng** | Thay đổi hàng loạt hiện danh sách so sánh trước và sau, người dùng chọn áp dụng từng mục hoặc tất cả |
| **Lối thoát rõ ràng** | Chatbot luôn có nút "Gặp nhân viên"; khi AI không trả lời được, chủ động đề nghị |
| **Thất bại một cách lịch sự** | Lỗi hệ thống thì nói rõ và đề nghị thử lại, không hiện thông báo lỗi kỹ thuật |

### 2.5 Người dùng giữ quyền kiểm soát

- Hành động có hậu quả (đăng sản phẩm, gửi email cho khách, hoàn tiền) cần **xác nhận rõ ràng**, hiển thị chính xác điều sắp xảy ra.
- Cho phép **tắt** tính năng AI, hoặc điều chỉnh mức tự động.
- Cho phép **cấu hình** hành vi: giọng thương hiệu, những điều không được nhắc tới, độ dài mong muốn. Cấu hình này đi vào prompt.

### 2.6 Thu thập tín hiệu một cách tự nhiên

- Nút 👍/👎 nhỏ, không làm phiền, kèm ô lý do **tùy chọn** khi bấm 👎.
- **Tín hiệu ngầm** có giá trị hơn: bản nháp được đăng nguyên văn, được sửa nhiều, hay bị bỏ; người dùng bấm tạo lại bao nhiêu lần; người dùng chuyển sang gặp nhân viên.
- Ghi lại **nội dung sửa** của người dùng: đó là dữ liệu quý để cải thiện prompt và làm eval ([Evaluation, Bài 4](../evaluation/04-online-eval.md)).

## 3. Chat hay nhúng vào giao diện?

Không phải tính năng AI nào cũng nên là một khung chat.

| Dạng | Ví dụ | Phù hợp khi |
|---|---|---|
| **Khung chat** | Trợ lý trả lời khách hàng | Câu hỏi mở, đa dạng, cần hội thoại qua lại |
| **Nhúng tại chỗ** | Nút "Viết mô tả bằng AI" ngay cạnh ô mô tả sản phẩm | Tác vụ rõ ràng, gắn với một màn hình cụ thể |
| **Chạy nền, hiện kết quả** | Tag cảm xúc tự động trên danh sách đánh giá | Không cần người dùng khởi động |
| **Hàng loạt** | "Viết mô tả cho 200 sản phẩm chưa có mô tả" | Khối lượng lớn, cần xem trước và duyệt theo lô |

Tính năng **nhúng tại chỗ** thường được dùng nhiều hơn khung chat, vì người dùng không phải nghĩ nên hỏi gì: AI xuất hiện đúng lúc, đúng chỗ họ cần.

## 4. Thực hành: component bản nháp có streaming, sửa và feedback

Ví dụ React cho tính năng "Viết mô tả bằng AI" nhúng trong trang sửa sản phẩm. Backend là endpoint SSE như ở [Nền tảng kỹ thuật, Bài 2](../nen-tang-ky-thuat/02-fastapi-streaming.md#4-streaming-voi-server-sent-events).

```tsx title="AiDescriptionDraft.tsx"
import { useRef, useState } from "react";

type Status = "idle" | "streaming" | "done" | "error";

export function AiDescriptionDraft({ productId, onPublish }: {
  productId: string;
  onPublish: (text: string, meta: { edited: boolean; traceId: string }) => void;
}) {
  const [draft, setDraft] = useState("");
  const [status, setStatus] = useState<Status>("idle");
  const [traceId, setTraceId] = useState("");
  const original = useRef("");  // bản AI sinh ra, để biết người dùng có sửa không

  async function generate(tone?: string) {
    setDraft(""); setStatus("streaming");
    try {
      const res = await fetch("/api/descriptions/stream", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ productId, tone }),
      });
      const reader = res.body!.pipeThrough(new TextDecoderStream()).getReader();
      let buffer = "", text = "";
      while (true) {
        const { value, done } = await reader.read();
        if (done) break;
        buffer += value;
        const events = buffer.split("\n\n");
        buffer = events.pop()!;
        for (const raw of events) {
          const type = raw.match(/^event: (.*)$/m)?.[1];
          const data = JSON.parse(raw.match(/^data: (.*)$/m)?.[1] ?? "{}");
          if (type === "delta") { text += data.text; setDraft(text); }
          if (type === "done") setTraceId(data.trace_id);
          if (type === "error") throw new Error(data.message);
        }
      }
      original.current = text;
      setStatus("done");
    } catch {
      setStatus("error");
    }
  }

  return (
    <section aria-label="Mô tả do AI viết">
      {status === "idle" && <button onClick={() => generate()}>✨ Viết mô tả bằng AI</button>}

      {status === "error" && (
        <p>Chưa viết được mô tả lúc này. <button onClick={() => generate()}>Thử lại</button></p>
      )}

      {(status === "streaming" || status === "done") && (
        <>
          <p><small>Bản nháp do AI viết, dựa trên tên và thuộc tính sản phẩm. Hãy kiểm tra trước khi đăng.</small></p>
          {/* Ô soạn thảo: người dùng sửa trực tiếp */}
          <textarea value={draft} onChange={(e) => setDraft(e.target.value)}
                    disabled={status === "streaming"} rows={8} />
          {status === "done" && (
            <div>
              <button onClick={() => generate("ngắn gọn hơn")}>Ngắn hơn</button>
              <button onClick={() => generate("trang trọng hơn")}>Trang trọng hơn</button>
              <button onClick={() => generate()}>Viết lại</button>
              <button onClick={() => onPublish(draft, { edited: draft !== original.current, traceId })}>
                Dùng mô tả này
              </button>
            </div>
          )}
        </>
      )}
    </section>
  );
}
```

Những điểm UX trong component:

- Nhúng **tại chỗ**, một nút rõ ràng, không cần khung chat.
- **Streaming** vào ô soạn thảo; khóa ô trong lúc sinh, mở ra khi xong để sửa.
- Nhãn rõ ràng: nội dung do AI viết, dựa trên dữ liệu nào, cần kiểm tra.
- **Tạo lại có định hướng** thay vì chỉ một nút "thử lại".
- Lỗi được xử lý lịch sự, có nút thử lại.
- Khi đăng, gửi kèm `edited` và `traceId`: backend ghi lại tín hiệu "đăng nguyên văn hay đã sửa", gắn với trace để phân tích.

Trong Shopify app, thay các thẻ HTML bằng component của hệ thống thiết kế bạn đang dùng; các nguyên tắc giữ nguyên.

## 5. Checklist UX cho một tính năng AI

- [ ] Người dùng biết AI làm được gì, không làm được gì, ngay tại chỗ dùng.
- [ ] Nội dung do AI tạo được đánh dấu rõ.
- [ ] Có streaming hoặc hiển thị tiến trình phù hợp với thời gian chờ.
- [ ] Có nguồn, hoặc dữ liệu đầu vào được hiển thị, để người dùng kiểm chứng.
- [ ] Kết quả sửa được, tạo lại được, hoàn tác được.
- [ ] Hành động có hậu quả cần xác nhận, hiển thị chính xác điều sắp xảy ra.
- [ ] Có lối thoát (gặp người, làm thủ công) khi AI không giúp được.
- [ ] Lỗi được trình bày thân thiện, không lộ chi tiết kỹ thuật.
- [ ] Tín hiệu ngầm và feedback được ghi lại, gắn với trace.

## Bài tập

**Bài 2.1.** Chọn 3 sản phẩm AI bạn hay dùng. Đánh giá từng sản phẩm theo checklist ở mục 5. Sản phẩm nào xử lý "lúc AI sai" tốt nhất?

**Bài 2.2.** Dựng component ở mục 4 cùng backend SSE. Ghi lại vào database: đăng nguyên văn hay đã sửa, số lần tạo lại.

**Bài 2.3.** Thiết kế (phác thảo trên giấy hoặc Figma) luồng "viết mô tả hàng loạt cho 200 sản phẩm": chạy nền, tiến độ, xem trước và duyệt theo lô, áp dụng, hoàn tác.

**Bài 2.4.** Cho 3 người dùng thử tính năng, quan sát họ (không hướng dẫn). Ghi lại những chỗ họ lúng túng, những lúc họ tin AI quá mức hoặc quá nghi ngờ.

## Đọc thêm

- [Frontend: tổng quan](../../oss/python/langchain/frontend/overview.md), [Streaming Reasoning Tokens](../../oss/python/langchain/frontend/reasoning-tokens.md), [Markdown Messages](../../oss/python/langchain/frontend/markdown-messages.md)
- [People + AI Guidebook](https://pair.withgoogle.com/guidebook/)

---

**Bài trước:** [Bài 1: Tư duy sản phẩm AI](01-tu-duy-san-pham.md) · [Tổng quan track](index.md) · **Bài tiếp theo:** [Bài 3: Generative UI](03-generative-ui.md)
