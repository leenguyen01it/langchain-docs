# Bài 3: Generative UI

!!! abstract "Tóm tắt bài học"
    - **Mục tiêu:** hiểu phổ generative UI (controlled, declarative, open-ended), để agent trả về giao diện có cấu trúc thay vì chỉ văn bản, render an toàn ở frontend, và dựng thẻ phê duyệt cho hành động của agent.
    - **Thời lượng:** 4 đến 5 ngày.
    - **Yêu cầu trước:** [Bài 2](02-ux-cho-ai.md), [Agents, Bài 2](../agents/02-langchain-agents.md), React cơ bản.

## 1. Vượt ra ngoài khung chat

Khách hỏi "Áo nào hợp đi làm mùa hè, dưới 500 nghìn?". Một câu trả lời dạng văn bản liệt kê 3 sản phẩm kèm giá thì đọc được, nhưng **thẻ sản phẩm** có ảnh, giá, nút "Thêm vào giỏ" thì hữu ích hơn nhiều.

**Generative UI** là mọi cách để output của agent trở thành giao diện: thẻ, bảng, biểu mẫu, biểu đồ, nút hành động.

## 2. Phổ generative UI

| Cách tiếp cận | Agent quyết định | Bạn kiểm soát | Phù hợp |
|---|---|---|---|
| **Controlled** | **Khi nào** hiện component nào (qua tool call hoặc state) | Toàn bộ giao diện của từng component | Sản phẩm cần giao diện nhất quán, đúng thương hiệu. **Điểm khởi đầu nên chọn** |
| **Declarative** | Cách **ghép** các component từ một danh mục có sẵn | Danh mục component và thuộc tính cho phép | Dashboard, báo cáo linh hoạt |
| **Open-ended** | Gần như toàn bộ giao diện (HTML, ứng dụng nhỏ) | Rất ít, cần sandbox | Công cụ nội bộ, thử nghiệm |

Trang [Tổng quan Generative UI](../../oss/python/langchain/frontend/generative-ui-overview.md) giải thích chi tiết phổ này; các trang [Controlled](../../oss/python/langchain/frontend/controlled-generative-ui.md), [Declarative](../../oss/python/langchain/frontend/declarative-generative-ui.md), [Open-ended](../../oss/python/langchain/frontend/open-ended-generative-ui.md) hướng dẫn cài đặt với SDK frontend của LangChain.

## 3. Nguyên tắc quan trọng nhất: model chọn, hệ thống điền

!!! danger "Không để model tự viết dữ liệu quan trọng lên giao diện"
    Nếu model tự viết giá sản phẩm vào thẻ, nó có thể viết sai giá. Khách thấy giá 350.000đ, vào thanh toán thấy 450.000đ.

    Model chỉ nên trả về **định danh** (mã sản phẩm) và **nội dung diễn giải** (vì sao gợi ý). **Hệ thống** lấy dữ liệu thật (tên, giá, ảnh, tồn kho) từ database hoặc API để điền vào component.

## 4. Thực hành: trợ lý gợi ý sản phẩm trả về giao diện

### 4.1 Backend: định nghĩa các khối giao diện

```python title="shop/ui_blocks.py"
from typing import Literal, Union

from pydantic import BaseModel, Field


class TextBlock(BaseModel):
    type: Literal["text"]
    text: str


class ProductCards(BaseModel):
    type: Literal["product_cards"]
    product_ids: list[str] = Field(description="Mã sản phẩm lấy từ kết quả tool search_products, tối đa 4")
    reasons: list[str] = Field(description="Mỗi sản phẩm một câu ngắn giải thích vì sao phù hợp, cùng thứ tự")


class ComparisonTable(BaseModel):
    type: Literal["comparison_table"]
    product_ids: list[str]
    attributes: list[Literal["price", "material", "sizes", "colors"]]


class UIResponse(BaseModel):
    blocks: list[Union[TextBlock, ProductCards, ComparisonTable]]
```

Danh sách khối là một **danh mục đóng**: model chỉ được chọn trong số component bạn đã thiết kế. `attributes` cũng giới hạn bằng `Literal`.

### 4.2 Backend: agent trả về UIResponse

```python title="shop/recommend.py"
from langchain.agents import create_agent

from shop.catalog import get_products, search_products  # tool tìm kiếm, hàm lấy dữ liệu thật
from shop.ui_blocks import ProductCards, UIResponse

agent = create_agent(
    model="anthropic:claude-opus-5",
    tools=[search_products],
    response_format=UIResponse,
    system_prompt=(
        "Bạn là trợ lý tư vấn của cửa hàng thời trang. Dùng search_products để tìm sản phẩm. "
        "Trả lời bằng các khối giao diện: một khối text ngắn, sau đó product_cards cho gợi ý, "
        "và comparison_table nếu khách muốn so sánh. Chỉ dùng mã sản phẩm có trong kết quả tool."
    ),
)


def recommend(question: str) -> dict:
    result = agent.invoke({"messages": [{"role": "user", "content": question}]})
    response: UIResponse = result["structured_response"]

    # Hệ thống điền dữ liệu thật, đồng thời loại bỏ mã sản phẩm không tồn tại
    hydrated = []
    for block in response.blocks:
        if isinstance(block, ProductCards):
            products = get_products(block.product_ids)  # dict: id -> {name, price, image_url, in_stock}
            cards = [
                {**products[pid], "id": pid, "reason": reason}
                for pid, reason in zip(block.product_ids, block.reasons)
                if pid in products
            ]
            hydrated.append({"type": "product_cards", "cards": cards})
        else:
            hydrated.append(block.model_dump())
    return {"blocks": hydrated}
```

### 4.3 Frontend: registry component

```tsx title="BlockRenderer.tsx"
type Card = { id: string; name: string; price: number; image_url: string; in_stock: boolean; reason: string };
type Block =
  | { type: "text"; text: string }
  | { type: "product_cards"; cards: Card[] }
  | { type: "comparison_table"; product_ids: string[]; attributes: string[] };

const formatVnd = (n: number) => n.toLocaleString("vi-VN") + "đ";

function ProductCards({ cards }: { cards: Card[] }) {
  return (
    <div className="cards">
      {cards.map((c) => (
        <article key={c.id}>
          <img src={c.image_url} alt={c.name} />
          <h3>{c.name}</h3>
          <p>{formatVnd(c.price)}</p>
          <p><small>{c.reason}</small></p>
          <button disabled={!c.in_stock}>{c.in_stock ? "Thêm vào giỏ" : "Hết hàng"}</button>
        </article>
      ))}
    </div>
  );
}

const REGISTRY: Record<string, (props: any) => JSX.Element> = {
  text: ({ text }) => <p>{text}</p>,          // React tự escape: không render HTML thô
  product_cards: ProductCards,
  comparison_table: ComparisonTable,           // component tự viết, lấy dữ liệu thật theo product_ids
};

export function BlockRenderer({ blocks }: { blocks: Block[] }) {
  return (
    <>
      {blocks.map((block, i) => {
        const Component = REGISTRY[block.type];
        return Component ? <Component key={i} {...block} /> : null;  // khối lạ: bỏ qua
      })}
    </>
  );
}
```

Ba lớp an toàn: **danh mục đóng** ở schema, **điền dữ liệu thật và loại mã không tồn tại** ở backend, **registry chỉ render component đã biết** và không bao giờ render HTML thô ở frontend.

## 5. Component như tool (controlled)

Cách tiếp cận khác: mỗi component là một **tool** agent có thể gọi giữa cuộc hội thoại, frontend render component khi thấy tool call tương ứng. Ưu điểm: component xuất hiện **ngay khi** agent quyết định, xen kẽ với văn bản đang stream, thay vì chờ câu trả lời hoàn chỉnh. Xem [Controlled generative UI](../../oss/python/langchain/frontend/controlled-generative-ui.md) và [Tool Calling](../../oss/python/langchain/frontend/tool-calling.md).

Một biến thể là **headless tool**: tool được thực thi ở **trình duyệt** (mở hộp thoại chọn file, đọc vị trí, thao tác giỏ hàng phía client). Xem [Headless Tools](../../oss/python/langchain/frontend/headless-tools.md).

## 6. Thẻ phê duyệt: human-in-the-loop trên giao diện

Khi agent muốn thực hiện hành động nhạy cảm (tạo mã giảm giá, hoàn tiền), giao diện nên hiện một **thẻ phê duyệt** thay vì hỏi bằng văn bản:

```text
┌──────────────────────────────────────────────┐
│ Trợ lý muốn tạo mã giảm giá                  │
│ Mã: HE2026   Giảm: 15%   Hạn: 30/06/2026     │
│ Áp dụng: bộ sưu tập "Mùa hè"                 │
│                                              │
│ [ Duyệt ]   [ Sửa ]   [ Từ chối ]            │
└──────────────────────────────────────────────┘
```

- Hiển thị **chính xác** tham số sẽ được dùng, ở dạng dễ đọc (không phải JSON thô).
- **Sửa** cho phép người dùng chỉnh tham số trước khi duyệt.
- Backend dùng `HumanInTheLoopMiddleware` ([Agents, Bài 2](../agents/02-langchain-agents.md#5-human-in-the-loop-duyet-hoan-tien)); frontend dựng thẻ theo hướng dẫn [Human-in-the-Loop (frontend)](../../oss/python/langchain/frontend/human-in-the-loop.md#xay-dung-approvalcard).

## 7. Khi nào nên dùng generative UI?

| Nên | Không nên |
|---|---|
| Kết quả có cấu trúc tự nhiên: sản phẩm, đơn hàng, số liệu | Câu trả lời chủ yếu là giải thích, tư vấn bằng lời |
| Người dùng cần **hành động** tiếp: mua, duyệt, chọn | Chỉ để "trông đẹp hơn" |
| Dữ liệu cần chính xác tuyệt đối (giá, tồn kho): component lấy từ hệ thống | Component phức tạp mà model hay chọn sai; hãy thử một bố cục cố định trước |

## Bài tập

**Bài 3.1.** Dựng trợ lý gợi ý sản phẩm ở mục 4 với dữ liệu giả lập 30 sản phẩm. Thử câu hỏi cần gợi ý và câu hỏi cần so sánh.

**Bài 3.2.** Cố tình sửa system prompt cho model "bịa" một mã sản phẩm không tồn tại. Xác nhận backend loại bỏ nó và giao diện không hiện thẻ sai.

**Bài 3.3.** Thêm khối `order_status` (hiển thị tiến trình giao hàng) và làm cho agent dùng nó khi khách hỏi về đơn.

**Bài 3.4.** Làm theo trang [Human-in-the-Loop (frontend)](../../oss/python/langchain/frontend/human-in-the-loop.md), dựng thẻ phê duyệt cho tool `create_discount`.

## Checklist

- [ ] Phân biệt được controlled, declarative, open-ended, và chọn được cách phù hợp.
- [ ] Model chỉ chọn component và định danh; dữ liệu quan trọng do hệ thống điền.
- [ ] Danh mục component đóng; frontend chỉ render component đã đăng ký, không render HTML thô từ model.
- [ ] Hành động nhạy cảm có thẻ phê duyệt hiển thị rõ tham số.

## Đọc thêm

- [Frontend: tổng quan](../../oss/python/langchain/frontend/overview.md), [Tổng quan Generative UI](../../oss/python/langchain/frontend/generative-ui-overview.md)
- [Structured Output (frontend)](../../oss/python/langchain/frontend/structured-output.md), [Tích hợp giao diện](../../oss/python/langchain/frontend/integrations/overview.md)

---

**Bài trước:** [Bài 2: UX cho sản phẩm AI](02-ux-cho-ai.md) · [Tổng quan track](index.md) · **Bài tiếp theo:** [Bài 4: Trách nhiệm và giao tiếp](04-trach-nhiem-giao-tiep.md)
