# Bài 5: Hệ thống nhiều agent

!!! abstract "Tóm tắt bài học"
    - **Mục tiêu:** hiểu khi nào nhiều agent thực sự có lợi, nắm các mẫu (subagent, handoff, skills, router, custom workflow), xây một hệ thống subagent, và đánh giá được chi phí, độ phức tạp tăng thêm.
    - **Thời lượng:** 4 đến 5 ngày.
    - **Yêu cầu trước:** [Bài 2](02-langchain-agents.md), [Bài 3](03-langgraph.md).

## 1. Khi nào cần nhiều agent?

Nhiều agent **không** tự động tốt hơn một agent. Chúng thêm chi phí, độ trễ, và rất nhiều điểm có thể hỏng. Chỉ nên tách khi gặp một trong các vấn đề sau:

| Vấn đề của một agent | Nhiều agent giải quyết thế nào |
|---|---|
| **Quá nhiều tool** (20, 30 tool), agent chọn sai | Mỗi agent chuyên biệt giữ một nhóm tool nhỏ, rõ ràng |
| **Context bị quá tải** vì phải đọc nhiều tài liệu | Subagent đọc trong context riêng, chỉ trả về bản tóm tắt ([Context engineering, mục 4.4](../prompt-context/03-context-engineering.md#44-co-lap-tach-context-cho-tung-phan-viec)) |
| **Công việc song song được** (nghiên cứu 5 đối thủ) | Nhiều subagent chạy đồng thời, rút ngắn thời gian |
| **Các lĩnh vực cần prompt khác nhau hẳn** (bán hàng, kỹ thuật, kế toán) | Mỗi agent có system prompt, model, quy tắc riêng |
| **Nhiều đội cùng phát triển** | Mỗi đội sở hữu và phát hành agent của mình độc lập |

!!! warning "Cái giá của nhiều agent"
    - **Token:** hệ thống nhiều agent thường tiêu tốn nhiều token hơn đáng kể so với một agent cho cùng tác vụ.
    - **Mất thông tin khi chuyển giao:** agent chính chỉ truyền cho subagent một phần context; subagent chỉ trả về một bản tóm tắt. Chi tiết quan trọng có thể rơi rụng ở cả hai chiều.
    - **Khó debug và đánh giá:** lỗi có thể nằm ở agent nào, ở lượt chuyển giao nào.

    Hãy thử cải thiện **một agent** trước (bớt tool, viết lại mô tả, nén context). Chỉ tách khi eval chứng minh cần thiết.

## 2. Các mẫu thiết kế

| Mẫu | Cách hoạt động | Phù hợp |
|---|---|---|
| **Subagents** | Agent chính gọi các subagent như gọi tool, rồi tổng hợp kết quả | Nghiên cứu, tác vụ chia nhỏ và song song được |
| **Handoffs** | Quyền điều khiển cuộc hội thoại chuyển từ agent này sang agent khác | Chăm sóc khách hàng nhiều bộ phận, người dùng trò chuyện trực tiếp với agent chuyên môn |
| **Skills** | Một agent duy nhất, nạp thêm kiến thức và chỉ dẫn chuyên biệt khi cần | Nhiều lĩnh vực nhưng không cần tách agent thật sự |
| **Router** | Bước phân loại chuyển câu hỏi tới một hoặc nhiều agent chuyên biệt, rồi gộp kết quả | Nhiều nguồn tri thức tách biệt |
| **Custom workflow** | Tự thiết kế bằng LangGraph, kết hợp các mẫu trên và bước tất định | Quy trình nghiệp vụ phức tạp |

Trang [Multi-agent](../../oss/python/langchain/multi-agent/index.md) có bảng so sánh chi tiết các mẫu theo số lượt gọi model, số token, và khả năng tương tác với người dùng. Đọc kỹ phần này trước khi chọn.

## 3. Ví dụ: agent nghiên cứu với subagent

Tác vụ: "So sánh chính sách đổi trả và phí vận chuyển của 3 cửa hàng thời trang đối thủ". Agent chính lập kế hoạch và tổng hợp; mỗi subagent nghiên cứu một đối thủ trong context riêng.

```python title="research/agents.py"
import asyncio

from langchain.agents import create_agent
from langchain.tools import tool

# Công cụ tìm kiếm web phía server của Anthropic, khai báo trực tiếp như một tool
WEB_SEARCH = {"type": "web_search_20260209", "name": "web_search", "max_uses": 5}

researcher = create_agent(
    model="anthropic:claude-sonnet-5",  # worker: model rẻ hơn, đủ tốt cho việc đọc và tóm tắt
    tools=[WEB_SEARCH],
    system_prompt=(
        "Bạn nghiên cứu MỘT cửa hàng được giao. Tìm chính sách đổi trả và phí vận chuyển "
        "trên website chính thức của họ. Trả về tối đa 150 chữ, gồm: thời hạn đổi trả, điều kiện, "
        "phí vận chuyển, và URL nguồn cho mỗi thông tin. Nếu không tìm thấy, ghi rõ 'không tìm thấy'."
    ),
)


@tool
async def research_store(store_name: str) -> str:
    """Giao cho một trợ lý nghiên cứu chính sách đổi trả và vận chuyển của MỘT cửa hàng.
    Có thể gọi nhiều lần song song cho nhiều cửa hàng.

    Args:
        store_name: Tên cửa hàng cần nghiên cứu.
    """
    result = await researcher.ainvoke(
        {"messages": [{"role": "user", "content": f"Nghiên cứu cửa hàng: {store_name}"}]}
    )
    return result["messages"][-1].text  # chỉ bản tóm tắt quay về agent chính


lead = create_agent(
    model="anthropic:claude-opus-5",
    tools=[research_store],
    system_prompt=(
        "Bạn là trưởng nhóm phân tích cạnh tranh. Giao mỗi cửa hàng cho một lần gọi "
        "research_store (gọi song song), sau đó tổng hợp thành bảng so sánh kèm nhận xét. "
        "Không tự bịa thông tin mà trợ lý không tìm được."
    ),
)


async def main():
    result = await lead.ainvoke({"messages": [{
        "role": "user",
        "content": "So sánh chính sách đổi trả và phí ship của Routine, Coolmate và Owen",
    }]})
    print(result["messages"][-1].text)


asyncio.run(main())
```

Những điểm thiết kế quan trọng:

- **Mô tả nhiệm vụ cho subagent phải đầy đủ.** Subagent không thấy cuộc hội thoại của agent chính; nó chỉ biết những gì được truyền vào. Mô tả mơ hồ dẫn tới subagent làm trùng lặp hoặc làm sai việc.
- **Định dạng output của subagent rõ ràng và ngắn gọn**, có trích nguồn, để agent chính tổng hợp được và kiểm chứng được.
- **Model rẻ hơn cho worker**, model mạnh cho agent điều phối, là cách phổ biến để cân bằng chi phí.
- **Giới hạn** số subagent, số lượt của mỗi subagent (middleware ở [Bài 2](02-langchain-agents.md#3-tao-agent)).

Xem thêm các quyết định thiết kế (sync hay async, một tool cho mỗi subagent hay một tool dispatch chung) ở trang [Subagents](../../oss/python/langchain/multi-agent/subagents.md).

## 4. Ví dụ: handoff trong chăm sóc khách hàng

Với chatbot hỗ trợ nhiều bộ phận, người dùng nên trò chuyện **trực tiếp** với agent chuyên môn (không qua trung gian tóm tắt lại). Mẫu handoff chuyển quyền điều khiển: agent tiếp nhận xác định vấn đề rồi "bàn giao" cho agent đơn hàng hoặc agent kỹ thuật, kèm theo cấu hình tool và prompt tương ứng.

Làm theo hướng dẫn đầy đủ: [Handoffs: Customer Support](../../oss/python/langchain/multi-agent/handoffs-customer-support.md).

## 5. Đánh giá hệ thống nhiều agent

Ngoài chất lượng câu trả lời cuối, cần đo:

| Chỉ số | Vì sao |
|---|---|
| Tổng token và chi phí mỗi tác vụ | So với phương án một agent: lợi ích có xứng đáng? |
| Số lượt gọi subagent, số lượt trùng lặp | Phát hiện agent chính giao việc kém |
| Tỉ lệ subagent trả về "không tìm thấy" hoặc lỗi | Phát hiện nhiệm vụ mô tả mơ hồ hoặc tool thiếu |
| Độ trễ đầu cuối | Song song hóa có thực sự giúp nhanh hơn? |
| Thông tin bị mất qua chuyển giao | So chi tiết trong output của subagent với câu trả lời cuối |

Dùng tracing (LangSmith) để xem toàn bộ cây lệnh gọi: agent nào gọi agent nào, với input gì, trả về gì. Xem [track Evaluation, Bài 3](../evaluation/03-eval-agent.md).

## Bài tập

**Bài 5.1.** Chạy ví dụ nghiên cứu ở mục 3. Xem trace (bật LangSmith) để thấy các subagent chạy song song. Ghi lại tổng token và thời gian.

**Bài 5.2.** Viết phiên bản **một agent** có tool `web_search`, làm cùng tác vụ. So sánh chất lượng, token, thời gian với phiên bản nhiều agent. Phiên bản nào đáng dùng hơn cho tác vụ này?

**Bài 5.3.** Sửa system prompt của agent chính cho mơ hồ đi ("hãy nhờ trợ lý tìm thông tin"). Quan sát các subagent làm gì. Rút ra bài học về cách giao việc.

**Bài 5.4.** Làm theo hướng dẫn [Handoffs: Customer Support](../../oss/python/langchain/multi-agent/handoffs-customer-support.md) và điều chỉnh cho trợ lý cửa hàng của bạn.

## Checklist

- [ ] Nêu được các lý do chính đáng để dùng nhiều agent và cái giá phải trả.
- [ ] Phân biệt được subagent, handoff, skills, router, custom workflow.
- [ ] Nhiệm vụ giao cho subagent được mô tả đầy đủ, output ngắn gọn có trích nguồn.
- [ ] Có số liệu so sánh giữa phương án nhiều agent và một agent.

## Đọc thêm

- [Multi-agent: tổng quan](../../oss/python/langchain/multi-agent/index.md), [Subagents](../../oss/python/langchain/multi-agent/subagents.md), [Handoffs](../../oss/python/langchain/multi-agent/handoffs.md), [Skills](../../oss/python/langchain/multi-agent/skills.md), [Router](../../oss/python/langchain/multi-agent/router.md)
- [Xây dựng Deep Agent từ đầu](../../oss/python/langchain/deep-agent-from-scratch.md)
- [How we built our multi-agent research system](https://www.anthropic.com/engineering/multi-agent-research-system) (Anthropic)

---

**Bài trước:** [Bài 4: MCP](04-mcp.md) · [Tổng quan track](index.md) · **Track tiếp theo:** [Evaluation](../evaluation/index.md)
