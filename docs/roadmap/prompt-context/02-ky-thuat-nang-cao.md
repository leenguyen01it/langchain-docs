# Bài 2: Kỹ thuật nâng cao

!!! abstract "Tóm tắt bài học"
    - **Mục tiêu:** vượt qua giới hạn của một prompt đơn lẻ bằng cách kết hợp nhiều lệnh gọi: chuỗi prompt, định tuyến, chạy song song, vòng lặp tự đánh giá và sửa; trích xuất dữ liệu an toàn; dùng chính LLM để cải thiện prompt.
    - **Thời lượng:** 4 đến 5 ngày.
    - **Yêu cầu trước:** [Bài 1](01-nguyen-tac-prompt.md).

Các kỹ thuật trong bài là những **workflow**: luồng xử lý do **code của bạn** điều khiển, mỗi bước là một lệnh gọi LLM. Chúng dễ đoán, dễ debug, dễ đo hơn agent, và giải quyết được phần lớn bài toán thực tế. Chỉ dùng agent khi luồng xử lý thực sự không thể định trước (xem [track Agents](../agents/01-agent-loop.md)).

Hàm tiện ích dùng xuyên suốt bài:

```python title="llm.py"
import anthropic
from pydantic import BaseModel

client = anthropic.Anthropic()
MODEL = "claude-opus-5"


def ask(prompt: str, system: str | None = None, effort: str = "high") -> str:
    kwargs = {"system": system} if system else {}
    response = client.messages.create(
        model=MODEL,
        max_tokens=16000,
        output_config={"effort": effort},
        messages=[{"role": "user", "content": prompt}],
        **kwargs,
    )
    return next(b.text for b in response.content if b.type == "text")


def ask_structured[T: BaseModel](prompt: str, schema: type[T], effort: str = "high") -> T:
    response = client.messages.parse(
        model=MODEL,
        max_tokens=16000,
        output_config={"effort": effort},
        messages=[{"role": "user", "content": prompt}],
        output_format=schema,
    )
    return response.parsed_output
```

(Cú pháp `def f[T: BaseModel]` cần Python 3.12 trở lên.)

## 1. Chuỗi prompt (prompt chaining)

Chia một tác vụ phức tạp thành **nhiều bước nhỏ**, output của bước trước là input của bước sau. Mỗi bước đơn giản hơn nên chính xác hơn, và bạn có thể kiểm tra kết quả **giữa các bước**.

Ví dụ: viết mô tả sản phẩm từ ghi chú lộn xộn của nhân viên kho, đảm bảo không bịa thông tin.

```python title="chain.py"
from pydantic import BaseModel, Field

from llm import ask, ask_structured


class ProductFacts(BaseModel):
    name: str
    material: str | None = Field(description="Chất liệu, null nếu ghi chú không nhắc tới")
    colors: list[str]
    sizes: list[str]
    features: list[str] = Field(description="Đặc điểm được nêu RÕ trong ghi chú")


class FactCheck(BaseModel):
    unsupported_claims: list[str] = Field(description="Các khẳng định trong mô tả KHÔNG có trong dữ kiện")
    ok: bool


def write_description(raw_notes: str) -> str:
    # Bước 1: trích xuất dữ kiện, tách biệt khỏi việc viết
    facts = ask_structured(
        f"Trích xuất thông tin sản phẩm từ ghi chú sau. Không suy đoán.\n\n<notes>\n{raw_notes}\n</notes>",
        ProductFacts,
        effort="low",
    )

    # Bước 2: viết, chỉ dựa trên dữ kiện đã trích xuất
    draft = ask(
        "Viết mô tả sản phẩm 80 đến 120 chữ, giọng điềm đạm, xưng 'bạn' với khách, "
        "chỉ dùng thông tin dưới đây.\n\n"
        f"<facts>\n{facts.model_dump_json(indent=2)}\n</facts>"
    )

    # Bước 3: kiểm tra chéo mô tả với dữ kiện
    check = ask_structured(
        f"<facts>\n{facts.model_dump_json()}\n</facts>\n\n<description>\n{draft}\n</description>\n\n"
        "Liệt kê các khẳng định trong mô tả không được dữ kiện hỗ trợ.",
        FactCheck,
    )
    if not check.ok:
        raise ValueError(f"Mô tả chứa thông tin không có căn cứ: {check.unsupported_claims}")
    return draft


print(write_description("ao so mi linen, mau be + trang, size S-XL, mac mat, ko nhan"))
```

Lợi ích so với một prompt duy nhất: bước 1 dùng effort thấp cho rẻ; nếu mô tả sai, bạn biết lỗi nằm ở bước trích xuất hay bước viết; bước 3 là một **cổng kiểm tra** (gate) tự động.

## 2. Định tuyến (routing)

Phân loại đầu vào trước, rồi chuyển tới **prompt chuyên biệt** (hoặc model phù hợp) cho từng loại. Mỗi prompt chuyên biệt ngắn hơn, tập trung hơn so với một prompt khổng lồ xử lý mọi trường hợp.

```python title="router.py"
from typing import Literal

from pydantic import BaseModel

from llm import ask, ask_structured


class Route(BaseModel):
    category: Literal["doi_tra", "van_chuyen", "tu_van_san_pham", "khieu_nai", "khac"]


PROMPTS = {
    "doi_tra": "Bạn xử lý yêu cầu đổi trả. Chính sách: <policy>...</policy>",
    "van_chuyen": "Bạn giải đáp về vận chuyển. Bảng phí và thời gian: <shipping>...</shipping>",
    "tu_van_san_pham": "Bạn tư vấn chọn size và phối đồ. Danh mục: <catalog>...</catalog>",
    "khieu_nai": "Bạn tiếp nhận khiếu nại. Xin lỗi chân thành, ghi nhận chi tiết, hẹn nhân viên liên hệ.",
    "khac": "Bạn là trợ lý chung của cửa hàng. Nếu không chắc, đề nghị khách để lại số điện thoại.",
}


def handle(message: str) -> str:
    route = ask_structured(f"Phân loại tin nhắn của khách:\n{message}", Route, effort="low")
    return ask(message, system=PROMPTS[route.category])
```

Bước phân loại là tác vụ đơn giản, phù hợp với effort thấp hoặc một model nhỏ. Đây cũng là nền tảng của mẫu **router** trong hệ thống nhiều agent (xem [Router](../../oss/python/langchain/multi-agent/router.md)).

## 3. Chạy song song

Hai dạng phổ biến:

- **Chia nhỏ (sectioning):** các phần độc lập chạy đồng thời, ví dụ đánh giá một bài viết theo 4 tiêu chí khác nhau, mỗi tiêu chí một lệnh gọi, rồi gộp kết quả.
- **Bỏ phiếu (voting):** chạy **cùng một** tác vụ nhiều lần, lấy kết quả đa số. Hữu ích cho quyết định quan trọng, ví dụ phát hiện nội dung vi phạm.

```python title="voting.py"
import asyncio
from collections import Counter
from typing import Literal

import anthropic
from pydantic import BaseModel

aclient = anthropic.AsyncAnthropic()


class Verdict(BaseModel):
    label: Literal["an_toan", "vi_pham"]


async def classify_once(text: str) -> str:
    r = await aclient.messages.parse(
        model="claude-opus-5", max_tokens=16000, output_format=Verdict,
        messages=[{"role": "user", "content": f"Bình luận sau có vi phạm quy định cộng đồng không?\n{text}"}],
    )
    return r.parsed_output.label


async def classify_with_vote(text: str, n: int = 3) -> tuple[str, float]:
    votes = await asyncio.gather(*(classify_once(text) for _ in range(n)))
    label, count = Counter(votes).most_common(1)[0]
    return label, count / n  # độ đồng thuận thấp: chuyển cho người kiểm duyệt
```

Độ đồng thuận (ví dụ chỉ 2/3 phiếu) là một tín hiệu hữu ích: trường hợp model "phân vân" nên được chuyển cho con người xem xét.

## 4. Vòng lặp tự đánh giá và sửa (evaluator-optimizer)

Một lệnh gọi **tạo** kết quả, một lệnh gọi khác **đánh giá** theo tiêu chí rõ ràng và đưa góp ý, lặp lại cho tới khi đạt hoặc hết số vòng cho phép. Hiệu quả khi có **tiêu chí đánh giá rõ ràng** và việc sửa theo góp ý thực sự cải thiện kết quả (viết nội dung marketing, dịch thuật, viết code).

```python title="refine.py"
from pydantic import BaseModel, Field

from llm import ask, ask_structured

RUBRIC = """1. Dưới 150 ký tự (hiển thị vừa màn hình thông báo điện thoại).
2. Có lời kêu gọi hành động cụ thể.
3. Không dùng từ ngữ gây áp lực quá mức ("cuối cùng", "duy nhất", "ngay lập tức").
4. Nhắc đúng mức giảm giá và thời hạn trong brief."""


class Review(BaseModel):
    passed: bool
    feedback: list[str] = Field(description="Góp ý cụ thể cho từng tiêu chí chưa đạt")


def write_push_notification(brief: str, max_rounds: int = 3) -> str:
    draft = ask(f"Viết một thông báo đẩy (push notification) theo brief:\n{brief}\n\nTiêu chí:\n{RUBRIC}")
    for _ in range(max_rounds):
        review = ask_structured(
            f"<brief>{brief}</brief>\n<rubric>{RUBRIC}</rubric>\n<draft>{draft}</draft>\n"
            "Đánh giá bản nháp theo từng tiêu chí.",
            Review,
        )
        if review.passed:
            return draft
        draft = ask(
            f"<brief>{brief}</brief>\n<draft>{draft}</draft>\n<feedback>{review.feedback}</feedback>\n"
            "Viết lại bản nháp, sửa theo góp ý. Chỉ trả về bản đã sửa."
        )
    return draft  # hết số vòng: trả bản tốt nhất hiện có, ghi log để người xem lại
```

Luôn có **giới hạn số vòng**. Với tiêu chí đo được bằng code (độ dài, có chứa từ khóa), hãy kiểm tra bằng code thay vì hỏi LLM: nhanh hơn, rẻ hơn, chính xác tuyệt đối.

## 5. Trích xuất dữ liệu an toàn

Trích xuất (hóa đơn, CV, hợp đồng, đơn hàng từ tin nhắn) là một trong những ứng dụng phổ biến nhất. Lỗi nguy hiểm nhất là model **tự điền** thông tin không có trong văn bản.

```python
from pydantic import BaseModel, Field


class OrderRequest(BaseModel):
    product: str | None = Field(description="Tên sản phẩm; null nếu không nhắc tới")
    quantity: int | None = Field(description="Số lượng; null nếu không nói rõ, KHÔNG mặc định là 1")
    phone: str | None = Field(description="Số điện thoại đúng như trong tin nhắn; null nếu không có")
    address: str | None
    evidence: list[str] = Field(description="Trích nguyên văn các đoạn tin nhắn dùng để điền thông tin")
    missing_fields: list[str] = Field(description="Các trường cần hỏi lại khách")
```

Các nguyên tắc:

- **Cho phép `null`** và nói rõ khi nào dùng. Nếu mọi trường đều bắt buộc, model buộc phải bịa.
- Yêu cầu **trích dẫn bằng chứng** (`evidence`), rồi kiểm tra bằng code rằng bằng chứng thực sự có trong văn bản gốc.
- Tách biệt **trích xuất** (đọc những gì có) với **suy luận** (đoán những gì không có). Nếu cần suy luận, để thành trường riêng có đánh dấu.
- Validate bằng code: số điện thoại đúng định dạng, số lượng dương.

## 6. Dùng LLM để cải thiện prompt

LLM rất giỏi viết và phê bình prompt. Một số cách dùng:

- **Viết bản nháp đầu tiên:** mô tả tác vụ, đối tượng, ví dụ đầu vào và đầu ra mong muốn; nhờ model viết system prompt.
- **Phân tích lỗi:** đưa prompt hiện tại và **các ví dụ model làm sai**, hỏi vì sao prompt dẫn tới lỗi đó và đề xuất sửa.
- **Sinh dữ liệu kiểm thử:** nhờ model viết 30 tin nhắn khách hàng đa dạng, kể cả trường hợp khó, viết tắt, cố tình gây nhiễu.

```python
critique = ask(f"""Đây là một system prompt và các trường hợp nó cho kết quả sai.

<prompt>
{current_prompt}
</prompt>

<failures>
{failures_with_expected_outputs}
</failures>

Phân tích nguyên nhân gốc của từng lỗi (prompt thiếu thông tin gì, mơ hồ ở đâu,
mâu thuẫn chỗ nào). Sau đó đề xuất bản prompt đã sửa. Giữ nguyên những phần
đang hoạt động tốt.""")
```

!!! warning "Luôn kiểm chứng bằng eval"
    Prompt do LLM đề xuất nghe có vẻ hợp lý nhưng không chắc đã tốt hơn. Chạy cả prompt cũ và mới trên cùng bộ dữ liệu và so sánh số liệu (xem [Bài 4](04-quan-ly-prompt.md)).

## 7. Khi nào dùng kỹ thuật nào?

| Tình huống | Kỹ thuật |
|---|---|
| Tác vụ có nhiều bước tuần tự, muốn kiểm tra giữa các bước | Chuỗi prompt |
| Đầu vào đa dạng, mỗi loại cần cách xử lý riêng | Định tuyến |
| Nhiều phần độc lập, cần nhanh | Chạy song song (chia nhỏ) |
| Quyết định quan trọng, cần độ tin cậy | Chạy song song (bỏ phiếu) |
| Có tiêu chí rõ, việc sửa theo góp ý cải thiện được chất lượng | Tự đánh giá và sửa |
| Luồng xử lý không thể định trước, cần tool linh hoạt | Agent ([track Agents](../agents/index.md)) |

## Bài tập

**Bài 2.1.** Chạy `chain.py` với 5 ghi chú sản phẩm tự viết (viết tắt, không dấu, thiếu thông tin). Cố tình sửa bước 2 để model bịa thêm "xuất xứ Việt Nam", kiểm tra bước 3 có phát hiện không.

**Bài 2.2.** Xây router ở mục 2 với 30 tin nhắn thử. Đo độ chính xác của bước phân loại. Loại tin nhắn nào hay bị phân loại sai?

**Bài 2.3.** Chạy `refine.py` với 3 brief khác nhau. Mỗi brief cần mấy vòng? Thay tiêu chí số 1 bằng kiểm tra `len(draft) < 150` bằng code.

**Bài 2.4.** Xây trình trích xuất `OrderRequest` từ tin nhắn đặt hàng qua chat. Viết code kiểm tra mỗi `evidence` có nằm trong tin nhắn gốc. Thử với 10 tin nhắn, trong đó có tin thiếu số điện thoại.

**Bài 2.5.** Lấy một prompt của bạn đang cho kết quả chưa tốt, thu thập 5 trường hợp sai, dùng kỹ thuật ở mục 6 để cải thiện. So sánh trước và sau trên 20 ví dụ.

## Checklist

- [ ] Biết chia tác vụ phức tạp thành chuỗi bước có kiểm tra giữa chừng.
- [ ] Dùng được định tuyến và chạy song song có giới hạn.
- [ ] Vòng lặp tự sửa luôn có giới hạn số vòng; tiêu chí đo được bằng code thì kiểm tra bằng code.
- [ ] Schema trích xuất cho phép `null` và yêu cầu bằng chứng.
- [ ] Mọi cải tiến prompt được kiểm chứng bằng số liệu.

## Đọc thêm

- [Building effective agents](https://www.anthropic.com/engineering/building-effective-agents) (Anthropic): các mẫu workflow và khi nào nên dùng agent.
- [Workflows và Agents](../../oss/python/langgraph/workflows-agents.md): cài đặt các mẫu này bằng LangGraph.

---

**Bài trước:** [Bài 1: Nguyên tắc viết prompt](01-nguyen-tac-prompt.md) · [Tổng quan track](index.md) · **Bài tiếp theo:** [Bài 3: Context engineering](03-context-engineering.md)
