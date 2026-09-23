# Bài 2: Chấm điểm và LLM-as-a-judge

!!! abstract "Tóm tắt bài học"
    - **Mục tiêu:** chọn đúng cách chấm cho từng loại tiêu chí, thiết kế giám khảo LLM đáng tin cậy, kiểm định giám khảo bằng nhãn của con người, và hiểu độ bất định của con số eval.
    - **Thời lượng:** 4 đến 5 ngày.
    - **Yêu cầu trước:** [Bài 1](01-tu-duy-eval.md), đã có danh sách loại lỗi ưu tiên.

## 1. Các cách chấm, từ rẻ tới đắt

| Cách chấm | Ví dụ | Ưu | Nhược |
|---|---|---|---|
| **Code** | So khớp chính xác, regex, kiểm tra JSON schema, "tool X có được gọi không", độ dài | Nhanh, rẻ, tuyệt đối nhất quán | Chỉ đo được tiêu chí khách quan |
| **LLM có đáp án tham chiếu** | "Câu trả lời có cùng ý với đáp án chuẩn không?" | Chấm được câu trả lời diễn đạt khác nhau | Cần đáp án cho từng ví dụ |
| **LLM không cần đáp án** | "Câu trả lời có hứa hẹn gì không có trong chính sách không?" | Chấm được output mở, cả trên production | Phải kiểm định cẩn thận |
| **So sánh cặp** | "Câu A hay câu B tốt hơn?" | Dễ quyết định hơn cho điểm tuyệt đối | Thiên lệch vị trí, chỉ cho kết quả tương đối |
| **Con người** | Chuyên gia chấm | Chính xác nhất | Chậm, đắt, không mở rộng được |

**Luôn ưu tiên code khi có thể.** Chỉ dùng LLM cho những tiêu chí cần hiểu ngữ nghĩa.

```python title="eval/code_checks.py"
import re


def uses_correct_pronoun(answer: str) -> bool:
    """Bot phải xưng 'Mộc', không xưng 'tôi' hay 'em'."""
    return not re.search(r"\b(tôi|em)\b", answer, flags=re.IGNORECASE)


def called_tool(trace: dict, tool_name: str) -> bool:
    return any(c["name"] == tool_name for c in trace.get("tool_calls", []))


def acknowledges_tool_error(trace: dict) -> bool | None:
    """Nếu có tool lỗi, câu trả lời phải thừa nhận. None nếu không có tool lỗi."""
    if not any(c.get("is_error") for c in trace.get("tool_calls", [])):
        return None  # không áp dụng
    return bool(re.search(r"(chưa|không) (tra|kiểm tra|tìm) được|gặp sự cố|thử lại", trace["output"]))
```

## 2. Thiết kế giám khảo LLM

### 2.1 Nguyên tắc

1. **Mỗi giám khảo một tiêu chí.** "Câu trả lời có bịa cam kết không?" thay vì "Câu trả lời tốt đến đâu?".
2. **Đạt hoặc không đạt** (nhị phân). Thang 1 đến 10 khó nhất quán, và "7 điểm" không nói được cần sửa gì.
3. **Định nghĩa rõ đạt và không đạt**, kèm **ví dụ** cho cả hai, đặc biệt là các trường hợp giáp ranh.
4. **Lập luận trước, kết luận sau** (trường `critique` đứng trước `passed`).
5. **Cung cấp đủ ngữ cảnh** mà giám khảo cần: chính sách, kết quả tool, đáp án tham chiếu.
6. **Dùng model mạnh** làm giám khảo, ít nhất trong giai đoạn đầu.

### 2.2 Ví dụ: giám khảo "bịa cam kết"

```python title="eval/judges.py"
import anthropic
from pydantic import BaseModel, Field

client = anthropic.Anthropic()


class Verdict(BaseModel):
    critique: str = Field(description="Phân tích ngắn gọn, nêu cụ thể cam kết nào có hoặc không có căn cứ")
    passed: bool


FABRICATION_JUDGE = """Bạn đánh giá câu trả lời của chatbot chăm sóc khách hàng.

Tiêu chí: câu trả lời KHÔNG được đưa ra cam kết mà không có căn cứ. Cam kết bao gồm:
thời gian giao hàng, mức hoàn tiền, khuyến mãi, quà tặng, ngoại lệ chính sách.
Một cam kết có căn cứ nếu nó xuất hiện trong <policy> hoặc trong <tool_results>.

- ĐẠT: mọi cam kết đều có căn cứ, hoặc câu trả lời không có cam kết nào.
- KHÔNG ĐẠT: có ít nhất một cam kết không có căn cứ.

<examples>
<example>
<tool_results>Đơn DH1024: Đang giao, dự kiến 15/03</tool_results>
<answer>Đơn của bạn đang giao, dự kiến tới ngày 15/03 ạ.</answer>
<verdict>ĐẠT. Ngày 15/03 có trong kết quả tool.</verdict>
</example>
<example>
<tool_results>Đơn DH1024: Đang giao</tool_results>
<answer>Đơn đang giao, bạn sẽ nhận trong 24h tới nhé.</answer>
<verdict>KHÔNG ĐẠT. "Trong 24h" không có trong kết quả tool hay chính sách.</verdict>
</example>
<example>
<answer>Mộc chưa tra được thời gian giao cụ thể, bạn chờ Mộc kiểm tra thêm nhé.</answer>
<verdict>ĐẠT. Không đưa ra cam kết.</verdict>
</example>
</examples>

<policy>
{policy}
</policy>

<tool_results>
{tool_results}
</tool_results>

<answer>
{answer}
</answer>"""


def judge_fabrication(answer: str, tool_results: str, policy: str) -> Verdict:
    response = client.messages.parse(
        model="claude-opus-5",
        max_tokens=16000,
        messages=[{
            "role": "user",
            "content": FABRICATION_JUDGE.format(policy=policy, tool_results=tool_results, answer=answer),
        }],
        output_format=Verdict,
    )
    return response.parsed_output
```

Để ý các ví dụ bao gồm cả trường hợp giáp ranh (câu từ chối cam kết vẫn **đạt**). Không có ví dụ này, giám khảo có thể đánh trượt câu trả lời thận trọng.

## 3. Kiểm định giám khảo

Một giám khảo LLM chưa kiểm định **không đáng tin**. Bạn cần biết nó đồng ý với con người tới đâu.

### 3.1 Quy trình

1. Lấy 100 đến 200 output, **con người chấm tay** đạt hoặc không đạt (người hiểu nghiệp vụ).
2. Chia thành tập **phát triển** (để sửa prompt giám khảo) và tập **kiểm tra** (chỉ dùng để báo cáo kết quả cuối).
3. Chạy giám khảo trên tập phát triển, xem các ca bất đồng, sửa prompt (thêm ví dụ, làm rõ định nghĩa). Lặp lại.
4. Khi đã hài lòng, chạy **một lần** trên tập kiểm tra và báo cáo.

### 3.2 Đo gì?

Đừng chỉ đo "tỉ lệ đồng ý". Nếu 90% output là đạt, một giám khảo **luôn nói đạt** cũng có 90% đồng ý, nhưng vô dụng. Hãy đo hai con số:

- **TPR** (tỉ lệ phát hiện đúng lỗi): trong các output con người chấm **không đạt**, giám khảo nói không đạt bao nhiêu phần trăm?
- **TNR** (tỉ lệ nhận đúng output tốt): trong các output con người chấm **đạt**, giám khảo nói đạt bao nhiêu phần trăm?

```python title="eval/calibrate.py"
def judge_agreement(human: list[bool], judge: list[bool]) -> dict:
    """True nghĩa là ĐẠT."""
    pairs = list(zip(human, judge))
    fails = [j for h, j in pairs if not h]
    passes = [j for h, j in pairs if h]
    return {
        "TPR (bắt được lỗi)": sum(not j for j in fails) / len(fails),
        "TNR (nhận đúng output tốt)": sum(passes) / len(passes),
        "số ca con người chấm không đạt": len(fails),
    }
```

Mục tiêu tham khảo: cả TPR và TNR trên khoảng 85% đến 90%. Tập kiểm định cần có **đủ ca không đạt** (ít nhất vài chục), nếu không TPR không đáng tin.

### 3.3 Ước lượng tỉ lệ lỗi thật

Khi đã biết TPR và TNR, bạn có thể hiệu chỉnh tỉ lệ đạt mà giám khảo báo cáo để ước lượng tỉ lệ đạt thật:

```python
def corrected_pass_rate(observed_pass_rate: float, tpr: float, tnr: float) -> float:
    # observed_fail = true_fail * TPR + (1 - true_fail) * (1 - TNR)
    observed_fail = 1 - observed_pass_rate
    true_fail = (observed_fail - (1 - tnr)) / (tpr + tnr - 1)
    return 1 - min(max(true_fail, 0.0), 1.0)
```

Ví dụ: giám khảo báo 80% đạt, với TPR = 0.9 và TNR = 0.95, tỉ lệ đạt thật ước tính khoảng 82%.

## 4. Các thiên lệch của giám khảo LLM

| Thiên lệch | Biểu hiện | Cách giảm |
|---|---|---|
| **Vị trí** | Trong so sánh cặp, hay chọn phương án đứng trước (hoặc sau) | Chấm hai lần, đổi thứ tự; chỉ tính khi hai lần nhất quán |
| **Độ dài** | Ưu ái câu trả lời dài, chi tiết, dù không đúng hơn | Rubric nói rõ độ dài không phải tiêu chí; kiểm tra tương quan điểm và độ dài |
| **Tự ưu ái** | Ưu ái output do chính model đó (hoặc cùng họ) sinh ra | Kiểm định với nhãn con người; cân nhắc giám khảo khác họ model |
| **Dễ dãi** | Cho đạt khi tiêu chí mơ hồ | Định nghĩa rõ, ví dụ không đạt cụ thể, đo TPR |
| **Bị output thao túng** | Output chứa câu kiểu "câu trả lời này hoàn toàn chính xác" | Đặt output trong thẻ, dặn giám khảo coi đó là dữ liệu cần đánh giá |

## 5. Độ bất định của con số eval

Với 50 ví dụ, tỉ lệ đạt 80% có khoảng tin cậy 95% vào khoảng từ 67% tới 89%. Nghĩa là "prompt mới đạt 82%, cũ đạt 78%" với 50 ví dụ **chưa nói lên điều gì**.

```python
import math


def confidence_interval(passed: int, total: int, z: float = 1.96) -> tuple[float, float]:
    """Khoảng tin cậy Wilson cho một tỉ lệ."""
    p = passed / total
    denom = 1 + z**2 / total
    center = (p + z**2 / (2 * total)) / denom
    half = z * math.sqrt(p * (1 - p) / total + z**2 / (4 * total**2)) / denom
    return center - half, center + half


print(confidence_interval(40, 50))    # khoảng (0.67, 0.89)
print(confidence_interval(400, 500))  # khoảng (0.76, 0.83)
```

Cách xử lý:

- **So sánh theo cặp** trên cùng ví dụ: đếm số ví dụ phiên bản mới sửa được và làm hỏng, thay vì so hai con số tổng.
- **Tăng số ví dụ** cho những quyết định quan trọng.
- **Chạy lặp lại** để thấy mức dao động do output không xác định.

## 6. Chi phí của eval

Giám khảo LLM tốn tiền: 5 giám khảo × 500 ví dụ = 2.500 lệnh gọi mỗi lần chạy eval. Cách tiết kiệm:

- Tiêu chí nào làm được bằng code thì dùng code.
- Dùng prompt caching cho phần cố định của prompt giám khảo (rubric, ví dụ, chính sách).
- Dùng Batch API (giảm 50%) cho eval không cần kết quả ngay.
- Khi đã kiểm định, thử giám khảo bằng model rẻ hơn hoặc effort thấp hơn, và **kiểm định lại**.

## Bài tập

**Bài 2.1.** Với 3 loại lỗi ưu tiên từ [Bài 1](01-tu-duy-eval.md), quyết định loại nào chấm bằng code, loại nào cần LLM. Cài đặt các hàm kiểm tra bằng code.

**Bài 2.2.** Viết một giám khảo LLM cho loại lỗi quan trọng nhất, theo đủ 6 nguyên tắc ở mục 2.1.

**Bài 2.3.** Tự chấm tay 100 output (đạt hoặc không đạt) cho tiêu chí đó. Chia 60 phát triển và 40 kiểm tra. Kiểm định và cải thiện giám khảo tới khi TPR và TNR trên tập phát triển đạt trên 85%, rồi báo cáo kết quả trên tập kiểm tra.

**Bài 2.4.** Thử nghiệm thiên lệch vị trí: viết giám khảo so sánh cặp, chấm 30 cặp theo cả hai thứ tự. Bao nhiêu phần trăm cặp cho kết quả không nhất quán?

## Checklist

- [ ] Ưu tiên chấm bằng code; chỉ dùng LLM cho tiêu chí cần hiểu ngữ nghĩa.
- [ ] Mỗi giám khảo LLM đo một tiêu chí, nhị phân, có ví dụ và lập luận trước kết luận.
- [ ] Giám khảo được kiểm định bằng nhãn con người, báo cáo TPR và TNR trên tập kiểm tra.
- [ ] Biết các thiên lệch chính và cách giảm.
- [ ] Không kết luận từ chênh lệch nhỏ trên dataset nhỏ.

## Đọc thêm

- [Evals](../../oss/python/langchain/test/evals.md): giám khảo LLM với `agentevals` và LangSmith.
- [RAG Bài 6](../rag/06-evaluation.md#41-llm-as-a-judge): giám khảo faithfulness và correctness cho RAG.

---

**Bài trước:** [Bài 1: Tư duy eval và phân tích lỗi](01-tu-duy-eval.md) · [Tổng quan track](index.md) · **Bài tiếp theo:** [Bài 3: Đánh giá agent](03-eval-agent.md)
