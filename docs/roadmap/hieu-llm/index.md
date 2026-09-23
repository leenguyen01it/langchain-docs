# Track: Hiểu LLM

Bạn không cần tự huấn luyện LLM để trở thành AI Engineer, nhưng cần hiểu **đủ sâu** để dự đoán khi nào model làm tốt, khi nào nó sai và vì sao. Người chỉ biết "gửi prompt, nhận câu trả lời" sẽ bế tắc ngay khi gặp lỗi đầu tiên: câu trả lời bị cắt cụt, chi phí tăng vọt, model bịa thông tin, hay tool bị gọi sai.

| Bài | Nội dung | Thời lượng |
|---|---|---|
| [Bài 1: LLM hoạt động thế nào](01-transformer-token.md) | Token, embedding, attention, dự đoán token tiếp theo, các giai đoạn huấn luyện, vì sao model "bịa" | 1 tuần |
| [Bài 2: Sampling, reasoning và context](02-sampling-reasoning.md) | Temperature và top-p, reasoning và adaptive thinking, effort, context window, stop reason | 3 đến 4 ngày |
| [Bài 3: Claude API chuyên sâu](03-claude-api.md) | Messages API, streaming, vision và PDF, structured output, đếm token, prompt caching, xử lý lỗi, batch | 1 tuần |
| [Bài 4: Tool use từ con số 0](04-tool-use.md) | Định nghĩa tool, tự viết vòng lặp agent, gọi tool song song, xử lý lỗi, tool runner | 4 đến 5 ngày |
| [Bài 5: Chọn model](05-chon-model.md) | Bản đồ các model, benchmark và giới hạn của nó, tự so sánh model, chi phí trên mỗi tác vụ, model mở và chạy local | 3 đến 4 ngày |

**Yêu cầu trước:** [track Nền tảng kỹ thuật](../nen-tang-ky-thuat/index.md), hoặc đã vững Python và async.

!!! note "Vì sao dùng SDK gốc thay vì LangChain?"
    Track này cố ý dùng **Anthropic SDK trực tiếp**. Hiểu API gốc giúp bạn biết framework (LangChain, LangGraph) đang làm gì bên dưới, debug được khi có lỗi, và dùng được những tính năng mới nhất ngay khi ra mắt. Các track sau ([RAG](../rag/index.md), [Agents](../agents/index.md)) dùng LangChain khi nó giúp code gọn hơn.

Sau track này, học tiếp [Prompt & Context Engineering](../prompt-context/index.md).

---

[Lộ trình AI Engineer](../index.md)
